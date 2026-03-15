# 什么是 tick‑to‑trade 链路

“tick‑to‑trade 链路”通常是指**从行情变化那一刻开始，到你真正把订单送出到交易所之间的整条路径**，以及这段路径上的总用时和各个分段的延迟。

在 HFT / 量化交易语境里，可以理解为：

> Tick（行情变化） → 系统感知 → 策略决策 → 风控检查 → 构造订单 → 订单到达交易所入口  
> Tick Update → System Perception → Strategy Logic → Pre-Trade Risk Check → Order Construction → Exchange Gateway  
> 这一整段时间 = tick‑to‑trade latency。

---

## 1. tick‑to‑trade 链路中的典型阶段

具体实现会有差异，但大体可以拆成以下几个阶段：

### 1. Tick 在交易所产生

- 交易所撮合引擎匹配了新的交易、更新了订单簿；
- 生成一条新的行情消息（tick），通过其市场数据系统往外广播：
  - 多播 UDP（如 ITCH/ITCH-like）；
  - 或 TCP/专有二进制协议等。

### 2. Tick 到达你机房的 NIC

- 这条行情消息从交易所网络一路传输；
- 抵达你服务器 NIC 时，网卡通常会有一个硬件时间戳（如果启用了 PTP/HW timestamp）：
  - 记为 `T_md_rx`；
- 这是“**tick 到达你机房边界**”的时间点。

### 3. 从 NIC 到 feed handler

- 包被 NIC 写入 RX ring，经过：
  - 驱动 RX → NAPI poll → 可能的 XDP / AF_XDP / DPDK / Onload/VMA 等；
- 最终，负责行情的进程（feed handler）在用户态看到这条 tick：
  - 完成协议解码（ITCH/FIX/自研）；
  - 更新本地订单簿视图。

这一段包含：

- 物理线路尾巴；
- NIC → 内核/用户态 bypass 路径；
- 内核中的 NAPI/softirq 行为。

是 tick‑to‑trade 链路中的“**网络 + 内核接收**”部分。

### 4. 策略决策与风控

- 策略引擎基于当前订单簿状态和新 tick：
  - 计算是否要下单、下多少、用什么价格；
- 可能经过一系列 pre‑trade 风控：
  - per‑account 限额；
  - notional/volume 限制；
  - 市价保护、价差检查等；
- 这一段是**纯应用逻辑的计算时间**。

### 5. 构造并发送订单

- 根据选择的协议（FIX/ITCH/专有）构造订单报文；
- 通过：

  - 内核 TCP/IP 栈（普通 socket）；
  - 或 Onload/VMA 用户态栈；
  - 或 DPDK/AF_XDP 等 kernel‑bypass 方式；

  将订单发往交易所网关。

这对应内核中的：

- `tcp_sendmsg` / `udp_sendmsg` / DPDK TX / AF_XDP TX；
- `ip_queue_xmit` / `ip_local_out` / `dev_queue_xmit` / 驱动 → NIC。

### 6. 订单到达交易所入口

- 包在网络上传输；
- 到达交易所入口路由 / 网关 / Session 层；
- **tick‑to‑trade 的终点**通常就定义在“订单到达交易所边界”这一刻（有些团队会以 ACK 收到为终点，这属于不同定义）。

---

## 2. tick‑to‑trade 延迟的分段测量（示意）

在工程实践中，会把 tick‑to‑trade 拆成若干段，分别测量。结合前面你学的 eBPF/XDP/DPDK，可以大致这样划分：

1. **Tick‑to‑NIC：**  
   - 交易所在内部产生 tick → 包到你 NIC 的时间；  
   - 主要取决于机房物理距离、对方系统架构；
   - 你通常只能看到 `T_md_rx` 这端，另一端由交易所定义。

2. **NIC‑to‑feed handler（内核接收 + 驱动）**：  
   - NIC → XDP / DPDK / Onload / NAPI poll → feed handler 线程；  
   - eBPF 可在：
     - XDP、`napi_poll`、`netif_receive_skb` 等位置打时间戳。

3. **feed handler‑to‑decision（策略/风控）**：  
   - 行情解析、订单簿更新 → 策略计算 → 风控  
   - 纯应用内部逻辑。

4. **decision‑to‑send（应用发单入口）**：  
   - 策略决定下单 → 调用 socket send / DPDK TX / Onload send；  
   - eBPF 可在 `tcp_sendmsg` / `udp_sendmsg` 等入口打点。

5. **send‑to‑exchange（order‑to‑exchange RTT 的一半）**：  
   - 从你这边发出订单包 → 到达交易所入口；  
   - 可以从 NIC / 抓包的 TX 时间戳 + 对端测量推算。

tick‑to‑trade latency 可以近似表示为上述几段的总和。

---

## 3. 总结：tick‑to‑trade 链路的意义

- 对策略而言：tick‑to‑trade 越短，越有机会在别人之前排队或抢 liquidity；  
- 对工程而言：你需要知道这条链路上的每个环节：
  - 哪一段是网络/内核问题（NAPI/softirq/队列/调度）；
  - 哪一段是应用逻辑/风控慢；
  - 哪一段是外部（交易所/链路）因素。

你前面学的：

- XDP / AF_XDP / DPDK / Onload/VMA；
- NAPI poll / netif_receive_skb / tcp_sendmsg / tcp_recvmsg；
- eBPF 的各种挂载点（XDP、TC、kprobe、tracepoint）；

本质上就是给你一套“沿着 tick‑to‑trade 链路布满小探针”的工具，帮助你把整个链路从黑箱拆成若干可度量的阶段。
