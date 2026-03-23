# RX/TX 多队列机制与 RSS / RPS / RFS / XPS

本文总结的是：

- RX/TX 多队列的整体思路；
- RSS / RPS / RFS / XPS 的原理和配置方式；
- 如何把 RX 队列 pin 到特定 CPU（IRQ 亲和性）；
- 它们对 feed handler / trading engine 的 CPU 亲和性、cache locality 和负载均衡的影响。

---

## 1. RX/TX 多队列的基本图景

现代 NIC 通常有多个 RX/TX 队列（Q0, Q1, Q2, Q3…）：

- 每个队列有自己的 RX/TX ring；
- 每个队列可以有独立的中断号和 NAPI 实例；
- 不同队列可以分配给不同 CPU 处理，实现多核并行收发。

后面的 RSS/RPS/RFS/XPS，本质都是在回答一个问题：

> 这个包 / 这个发送请求应该走哪条队列，由哪个 CPU 处理？

---

## 2. RSS / RPS / RFS / XPS：四个机制的原理

### 2.1 RSS（Receive Side Scaling）——硬件按流散列收包

**原理：**

- 网卡对每个入站包基于 2/4/5 元组（源/目的 IP、端口、协议）计算一个 hash；
- 通过 indirection table 把 hash 映射到某个 RX 队列；
- 每个 RX 队列的中断和 NAPI poll 可以绑到不同 CPU。

**效果：**

- 同一流的包落在同一个 RX 队列（保持顺序）；
- 流量分散到多个 RX 队列/CPU，多核并行收包。

**配置要点：**

- 使用 ethtool 查看/配置：
  - ethtool -n ethX / -N / -X（不同版本参数略有区别）
- 在 /proc/interrupts 里可以看到：
  - 每个 RX 队列对应的 IRQ：如 `eth0-TxRx-0`, `eth0-TxRx-1`。

---

### 2.2 RPS（Receive Packet Steering）——软件层的收包分流

**原理：**

- 包先到达某个 CPU（由当前 RX 队列 / IRQ 决定）；
- 在 `netif_receive_skb` 一类路径上，内核对 skb 算 hash；
- 根据 `/sys/class/net/ethX/queues/rx-N/rps_cpus` 的 CPU mask，将 skb 转发到目标 CPU 的接收队列，由该 CPU 的 NET_RX_SOFTIRQ 处理。

**效果：**

- 即便硬件队列数有限，也能在软件层把收包负载分散到更多 CPU；
- 类似“软件版 RSS”。

**配置要点：**

- 每个 RX 队列都有一个 `rps_cpus` 文件：
  - /sys/class/net/ethX/queues/rx-N/rps_cpus
- 写入 CPU mask（十六进制）即可，比如：
  - `echo 4 > rps_cpus` 表示允许 CPU2 处理该队列包。

---

### 2.3 RFS（Receive Flow Steering）——按应用 CPU 做流亲和收包

RPS 只按 hash 均匀分流，不知道哪个 CPU 正在跑这个 flow 的应用。RFS 在 RPS 之上加入了“应用位置”的信息。

**原理：**

- 内核维护 flow → socket → last_cpu 的映射；
- 收到新包时：
  - 先用 hash 找 flow；
  - 查 flow 映射，找出最近一次处理该 flow 的 CPU；
  - 优先把这个 flow 后续的包 steer 到该 CPU。

**效果：**

- 收包 softirq、协议栈处理、应用 `recv()` 都集中在同一 CPU 上；
- 极大提升 cache locality，减少跨核唤醒和 cache miss；
- 对“每个连接/流绑定一个线程”的模型尤其友好。

**配置要点：**

- 总的 flow 表大小：
  - `/proc/sys/net/core/rps_sock_flow_entries`
- 每个 RX 队列独立的 flow entry 数：
  - `/sys/class/net/ethX/queues/rx-N/rps_sock_flow_entries`

RFS 是对 RPS 的增强：先启用 RPS，再配 RFS 让它按“应用 CPU”做更聪明的 steer。

---

### 2.4 XPS（Transmit Packet Steering）——发送时按 CPU 选队列

**原理：**

- 在发送路径（`dev_queue_xmit` 之前）：
  - 内核根据 XPS 配置，为当前 skb 选择一个 TX 队列；
  - 通常按当前 CPU 或与 socket 亲和的 CPU 来选。

**效果：**

- 将发包操作尽可能分散到多个 TX 队列；
- 减少多个 CPU 抢同一个 TX 队列锁的竞争；
- 提升发送侧 cache locality 和多核并行度。

**配置要点：**

- 每个 TX 队列有一个 `xps_cpus` 文件：
  - `/sys/class/net/ethX/queues/tx-N/xps_cpus`
- 写 CPU mask，例如：
  - queue 0 → CPU0/1；
  - queue 1 → CPU2/3。

---

## 3. 把 RX 队列 pin 到特定 CPU：IRQ 亲和性 & smp_affinity

多队列只解决了“硬件上有多条路”，还需要通过 **IRQ 亲和性** 明确哪个队列由哪个 CPU 处理。

### 3.1 查看 IRQ 和队列映射

- `cat /proc/interrupts` 可以看到类似：

  - 32:  1000  eth0-TxRx-0  
  - 33:  2000  eth0-TxRx-1  

这些 IRQ 行通常标明了队列号（如 `eth0-TxRx-0` 对应 RX/TX queue 0）。

### 3.2 设置每个 IRQ 的 CPU 亲和性

- 对某个 IRQ（例如 32），配置文件是：

  - `/proc/irq/32/smp_affinity`

- 文件内容是一个十六进制 CPU mask：
  - `1` → CPU0  
  - `2` → CPU1  
  - `4` → CPU2  
  - `8` → CPU3 等

**示例：让 RX 队列 0 的中断只在 CPU2 上处理**

- 假设 queue0 使用 IRQ 32：

  - `echo 4 > /proc/irq/32/smp_affinity`

配合 RSS：

- RSS 决定“某个流 → queue0”；  
- IRQ 亲和性决定“queue0 → CPU2”；  
- 于是某些流就固定由 CPU2 的 NAPI poll/softirq 处理。

---

## 4. 对 feed handler / trading engine 的影响

在 HFT / feed handler / trading engine 的环境中，这些机制的目标很直接：

> 让同一条流的收包、中断处理、协议栈处理和业务线程执行 **尽可能在同一 CPU 上完成**，同时避免某个核被打爆。

### 4.1 CPU 亲和性：线程和队列一一对应

理想配置（举例）：

- 某条交易所 X 的行情流：
  - 通过 RSS 落在 RX queue 0；
  - RX queue 0 的 IRQ → CPU2（通过 smp_affinity 配好）；
  - feed handler 线程 → CPU2（通过 `sched_setaffinity`/`taskset` 固定）。

这样：

- NAPI poll / NET_RX_SOFTIRQ / `tcp_v4_rcv` 等收包路径在 CPU2 上跑；
- 应用线程也在 CPU2 上解析 feed；
- 不需要在 CPU 之间搬 `skb` 或跨核唤醒，提升 cache locality、减少 jitter。

RFS 会进一步把“正在这个 CPU 上 `recv()` 的 flow”的包 steer 回这个 CPU，即便 RSS 的初始分配不完美。

### 4.2 cache locality：减少跨核迁移

如果不做上述配置，很容易出现：

- RX 队列中断在 CPU0 上；  
- 应用线程在 CPU3 上：

结果是：

- 收包和协议栈处理在 CPU0，数据进入 socket 队列时在 CPU0 的 cache 里；
- CPU3 被唤醒来 `recv()`，从它的角度看数据不在本地 cache，需要跨核访问；
- 频繁跨核导致 cache miss 和不可预测的延迟尾巴。

通过 RSS/RPS/RFS/XPS + IRQ affinity + 线程 CPU 亲和：

- 你尽量把同一流相关的工作锁定在同一个或一组固定 CPU 上；
- 减少 cache 抖动和跨核迁移。

### 4.3 CPU 负载均衡 vs 专核

在 HFT 环境：

- 关键核心（跑 critical feed handler / matching）通常被精心隔离：  
  - 只接收需要的队列中断；  
  - 只跑特定进程；  
  - 其他系统任务（ksoftirqd、后台 daemons）尽量挪到“非关键核”。
- 多队列 + RSS/RPS/XPS 负责：
  - 把非关键流量分散到其他核；
  - 避免单个核心软中断/中断爆掉；
  - 让关键核只处理自己关心的那部分流量。

---

## 5. 一句话总结

**多队列 + RSS/RPS/RFS/XPS + IRQ 亲和性 + 线程 CPU 亲和性** 的组合核心目标是：

> 让“某条流”的收包、协议栈处理和业务线程尽可能在同一 CPU 上执行，既避免单核被打爆，又最大化 cache 命中率和延迟可预测性。

这正是 HFT feed handler / trading engine 调优时常见的“queue0 ↔ CPU2 ↔ 线程 A”那种目标状态背后的机制组合。
