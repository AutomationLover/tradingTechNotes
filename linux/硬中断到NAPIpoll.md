# Linux 内核中 “硬中断 → NET_RX_SOFTIRQ → net_rx_action → NAPI poll” 的完整流程

这条链路描述的是：**网卡收包后，先用一次硬中断“叫醒”CPU，然后在软中断 `NET_RX_SOFTIRQ` 中由 `net_rx_action` 调度各个 NAPI 实例的 `poll()` 函数，批量从网卡队列拉包并交给协议栈处理。**

下面按时间顺序拆开说明。

---

## 1. 网卡收包 & 触发硬中断（Hard IRQ）

1. NIC 收到线上数据帧，将数据通过 DMA 写入主内存中的 RX ring buffer。
2. 满足条件（有新包、达到门限等）时，网卡向 CPU 触发 **接收中断**（RX interrupt）。
3. CPU 当前正在执行其他代码，被中断打断，跳到对应中断向量，执行网卡驱动注册的 **硬中断处理函数（IRQ handler）**。

在 **NAPI 模式** 下，硬中断 handler 只做极少量工作：

- 不在这里跑完整收包逻辑；
- 主要做两件事：
  - 调用 `napi_schedule()` / `__napi_schedule()`，标记对应的 NAPI 实例需要处理；
  - 临时关闭或抑制后续 RX 中断，避免 “interrupt per packet” 的中断风暴；
- 然后快速返回，把后续重活留给软中断。

状态此时可以理解为：

> 包已经在内存，CPU 被提醒“有网络事件”，但具体处理延后到软中断阶段。

---

## 2. 标记并触发 `NET_RX_SOFTIRQ`（网络接收软中断）

`napi_schedule()` 在把 NAPI 实例加入队列的同时，会：

- 将当前 CPU 的 **`NET_RX_SOFTIRQ` 标记为 pending**；
- 告诉内核：“这个 CPU 上有网络接收任务要处理。”

软中断不会立刻执行，而是在适当的时机统一处理，比如：

- 从硬中断返回到普通内核上下文时；
- 内核检查 softirq 标志时；
- 如果软中断压力大，部分工作会由内核线程 `ksoftirqd/N` 接手执行。

此时整体状态：

> “`NET_RX_SOFTIRQ` 已经被标记，等待 CPU 在合适机会进入网络接收软中断的处理逻辑。”

---

## 3. 处理 `NET_RX_SOFTIRQ`：调用 `net_rx_action`

当 CPU 准备处理软中断时，会依次处理所有 pending 的 softirq 类型，其中包括：

- `NET_RX_SOFTIRQ`（网络接收软中断）

对应的处理函数是：

- `net_rx_action()`

`net_rx_action()` 的职责可以总结为：

> “我是网络接收 softirq 的调度器，要把本 CPU 上所有被 `napi_schedule()` 过的 NAPI 对象拉出来，按照预算依次调用它们的 `poll()` 函数，批量收包。”

内部大致流程：

1. 遍历当前 CPU 的 NAPI 队列（这些 `struct napi_struct` 都是之前在硬中断里被标记过的）。
2. 对每个 NAPI：
   - 调用其 `poll()` 回调函数，传入一个 `budget`（这次最多处理多少包）。
3. 如果某个 NAPI 在本轮 poll 中完成了队列处理：
   - 调用 `napi_complete()`，表示这轮活干完，可以重新开启该设备的 RX 中断。
4. 如果还剩很多包未处理且已耗尽 budget：
   - 保留 NAPI 状态，留待下一轮软中断或交由 `ksoftirqd` 继续处理，避免软中断占满 CPU。

---

## 4. NAPI poll：在软中断上下文里批量收包

每个 NAPI 实例（`struct napi_struct`）都与一个或多个 RX 队列关联，并有一个驱动提供的 `poll()` 回调，例如：

- `xxx_napi_poll(struct napi_struct *napi, int budget)`

在 `net_rx_action()` 调用某个 NAPI 的 `poll()` 时，典型行为：

1. 从对应的 RX ring / completion 队列中，批量取出已 DMA 完成的包描述符。
2. 为每个包构造或填充 `sk_buff`：
   - 指定数据指针和长度；
   - 填充协议字段、校验信息等。
3. 将 `skb` 递交到上层协议栈入口，例如：
   - `napi_gro_receive()`（走 GRO 聚合）；
   - 或直接 `netif_receive_skb()`。
4. 累计已处理包数：
   - 达到 `budget` 或暂时没有更多包，就返回；
   - 通过返回值/内部状态告诉 `net_rx_action()` 是否还有剩余工作。

当 `poll()` 把当前队列处理完时，通常会：

- 调用 `napi_complete(napi)`：
  - 告诉内核和驱动：“这轮 NAPI 轮询结束，我现在空闲，可以重新启用 RX 硬中断。”

这样下次新流量到来时，仍会由硬中断触发新一轮 softirq + NAPI poll。

---

## 5. 一句话串起来的完整流程

从 NIC 收包到协议栈处理的关键“中断 + 轮询”路径可以压缩成：

1. **网卡收包 + DMA → 硬中断**  
   - NIC 把包写到 RX ring，触发 RX 中断；  
   - 硬中断 handler 调用 `napi_schedule()` 标记 NAPI，禁止进一步 RX 中断，快速返回。

2. **软中断标记 → 进入 `NET_RX_SOFTIRQ`**  
   - `NET_RX_SOFTIRQ` 被标记为 pending；  
   - CPU 在合适时机处理 softirq，进入 `net_rx_action()`。

3. **`net_rx_action()` 调度 NAPI poll**  
   - 遍历所有 pending 的 NAPI 实例；  
   - 为每个 NAPI 调用其 `poll()` 回调，在给定 `budget` 内批量收包。

4. **NAPI poll 批量收包并上送协议栈**  
   - 从 RX ring 批量取包，构造 skb；  
   - 交给上层协议栈（如 `ip_rcv` / `tcp_v4_rcv` 等）；  
   - 队列清空后调用 `napi_complete()`，重新允许 RX 中断。

因此，“硬中断 → NET_RX_SOFTIRQ → net_rx_action → NAPI poll” 本质是 Linux 收包时的：

> **“一次中断唤醒 + 在软中断上下文中轮询批量处理”** 模式，  
> 既避免了每包一个硬中断的中断风暴，又通过配额控制和批处理兼顾吞吐与延迟。
