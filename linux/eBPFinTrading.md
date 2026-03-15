# eBPF 怎么用在低延迟交易（HFT）中？

问题是：**ebpf怎么用在 低延迟 trading中？**

核心结论：eBPF 更适合做 **超轻量观测、过滤和简单风控**，挂在内核 fast‑path 旁边，而不是直接取代 DPDK/FPGA/Onload 这种“主数据平面”。

下面按几个典型用法说明。

---

## 1. 心态：不要把 eBPF 放在“撮合核心路径”上扛所有逻辑

严肃 HFT 的主数据路径通常是：

- FPGA NIC 上做 L2/L3/L4 + feed decode 的一部分；
- 或者 Solarflare Onload / Mellanox VMA / 自研 DPDK 引擎；
- 单核 pin、busy‑poll、hugepage、NUMA 绑核，一切围绕 **低 jitter** 调优。

在这种架构下：

- eBPF 不适合在“几十纳秒级”的主线里做复杂逻辑；
- 更合适的角色是“**在内核/驱动侧做轻量观测和防护**”，你可以把它当成“可编程内核 sidecar”。

---

## 2. 用 XDP / TC 做 L2/L3 级过滤和流量整形

### 场景：前置过滤、DDoS 防护、管理面隔离

在机房中，你常需要在“很靠前”的位置做：

- 行情 feed 清洗：
  - 丢弃无关 multicast 组/心跳等；
- 管理面/监控流量的限制：
  - 防止管理面 burst 干扰交易内核；
- 简单 ACL / DDoS 防护。

**XDP 用法：**

- 挂在交易 NIC 的 RX path 上：
  - 根据源 IP/port/VLAN/multicast group 做 `XDP_DROP`；
  - 或 `XDP_REDIRECT` 到别的 queue / NIC（例如把管理面引流到另一块网卡）；
- 能力：
  - 在 skb 创建前、还没进协议栈就能快速“截流”。

**TC ingress 用法：**

- 在 ingress qdisc 上挂 BPF 程序：
  - 基于 `struct __sk_buff` 的 5‑tuple、DSCP、VLAN 做分类/计数/过滤；
- 比 XDP 晚一点（已经有 skb），但做 ACL 更方便。

**收益：**

- 让“真正的行情/订单流”跑在更干净的 RX 路径上；
- 减少软中断和 NAPI poll 的负载，对 tail latency 有直接帮助。

---

## 3. 用 kprobe / tracepoint 做极轻量延迟 & 抖动剖析

目标：**理解 NIC → 核心线程这条链路各段的时间分布，不打扰主路径。**

可以在以下位置挂 eBPF：

- `napi_poll`（kprobe/kretprobe）：
  - 测量每轮 poll 的耗时；
  - 统计 batch size（一次 poll 处理多少包）；
- `netif_receive_skb` / tracepoint `net:netif_receive_skb`：
  - 记录“skb 进入协议栈”的时间；
- `tcp_recvmsg`（如果你有走内核栈的控制/辅助流）：
  - 记录从 socket 队列到用户缓冲的时间；
- `sched:sched_switch`、`irq:softirq_entry/exit` tracepoint：
  - 观察调度/软中断行为是否在延迟 spike 时异动。

使用方式：

- 只做**抽样**（例如每 N 个包采样一次）；
- 用 per‑CPU map 更新极少字段：
  - 时间戳、队列 ID、CPU ID、简单计数；
- 用户态守护进程低频拉取并画时间序列。

这样可以做到：

- 把 tick‑to‑trade 拆成若干段（NIC→XDP→NAPI→应用→send→NIC）；
- 在不污染 hot path 的前提下，定位是哪一段偶尔引入了几十到几百微秒的 jitter。

---

## 4. 用 eBPF 做 pre‑trade 风控 / 限速的“内核侧预过滤”

有些简单的风控逻辑可以前移到内核，以减轻撮合线程压力：

- 按 account / strategy / host 针对连接数、下单速率做粗限速；
- 对明显违规流量（过高频率或异常模式）在内核侧直接丢弃/拒绝。

可用挂载点：

- cgroup/connect4 / sendmsg4：
  - 针对某个 cgroup（容器/策略进程）做连接级策略：允许/拒绝；
- socket-layer BPF：
  - 对某些 socket 的 send/recv 进行轻量检查；
- TC egress BPF：
  - 对某些 flow 做速率限制（policing/shaping）。

注意：

- 在 eBPF 里只做**非常简单**的判断和计数；
- 重逻辑仍在应用/风控引擎里；
- 目标是“挡掉完全不合规的流量”，而不是把整个规则引擎塞进 BPF 程序。

---

## 5. 用 eBPF 监控基础设施健康（NAPI/softirq/IRQ/CPU）

低延迟交易中，很多问题来自基础设施的微小变化，例如：

- irq 绑核被改坏；
- `netdev_budget` 调整导致 NAPI 行为变化；
- softirq/ksoftirqd 在某个核上负载异常。

eBPF 可以作为“机房神经系统”的一部分：

- 周期性收集：
  - 每个 RX 队列的 PPS / 数据量；
  - `napi_poll` 耗时
