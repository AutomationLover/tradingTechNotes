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
 
