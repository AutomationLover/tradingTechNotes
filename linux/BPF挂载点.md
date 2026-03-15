# 跟网络/延迟相关的关键 BPF 挂载点及其在内核中的位置

本文按“从 NIC 到应用”的路径，介绍常用的、与网络性能和延迟分析高度相关的 BPF 挂载点：

- XDP
- TC ingress / egress
- kprobe / kretprobe（napi_poll、netif_receive_skb、tcp_sendmsg、tcp_recvmsg 等）
- tracepoint（net:/sched:/irq:/softirq: 等）

重点说明它们在内核中的位置、能看到什么、适合做什么。

---

## 1. XDP：驱动 RX 最前面的入口（raw packet，尚无 skb）

### 1.1 在收包路径中的位置

收包大致路径：

- NIC → DMA → 驱动 RX → NAPI poll → 创建 skb → netif_receive_skb → ip_rcv → tcp_v4_rcv → socket → 应用 recv()

XDP 挂在：

- 驱动 RX 处理的最前面；
- 在 NAPI poll 逻辑里，一拿到 DMA 好的数据，就先跑 XDP 程序；
- 此时 **尚未创建 `struct sk_buff`**，只有 raw packet buffer。

位置总结：

- XDP 在“创建 skb 之前、进入协议栈之前”，是软件路径最靠前的 hook。

### 1.2 能看到 / 能做什么

XDP sees：

- 原始 L2/L3/L4 头部和 payload；
- 没有 `struct sk_buff`、没有 socket 信息。

适合做：

- 极靠前的过滤：
  - ACL / DDoS 粗过滤；
  - 黑/白名单；
- 粗粒度统计 / 采样：
  - 按 IP/port/5-tuple 做计数；
  - 简单打点；
- 简单 L4 逻辑：
  - 简单 L4 LB；
  - redirect 到 AF_XDP；
- 延迟分析：
  - 作为“内核第一次看到这个包”的时间点，用 `bpf_ktime_get_ns()` 打入站时间戳。

---

## 2. TC ingress / egress：在 skb 层、接近设备的流量控制点

### 2.1 在路径中的位置

有了 skb 之后，路径变成：

- RX：
  - NIC → 驱动/NAPI → 创建 skb →（可选）TC ingress / clsact → netif_receive_skb → ip_rcv → tcp_v4_rcv → socket
- TX：
  - 应用 send → tcp_sendmsg → ip_local_out → dev_queue_xmit → qdisc / TC egress → 驱动 → NIC

对应挂载点：

- **TC ingress BPF**：
  - 挂在 ingress qdisc（通常是 `clsact` 的 ingress side）；
  - 触发点紧邻 `netif_receive_skb` 前后。
- **TC egress BPF**：
  - 挂在 egress qdisc（`dev_queue_xmit` 之后）；
  - 在 skb 被排队到某个设备队列之前/之中执行。

### 2.2 能看到 / 能做什么

此时已经有 `struct __sk_buff`：

- 可获取：
  - skb 长度、ifindex、mark 等；
  - 使用 helper 读取 L2/L3/L4 字段（MAC/IP/port 等）；
- 可以：
  - 做计数/统计；
  - ACL（丢包/放行）；
  - mirror / redirect；
  - 修改 skb 某些字段（在合理范围内）。

延迟/观测用途：

- **TC ingress**：
  - 第二个靠前的观测点（比 XDP 晚一点，但信息更多）；
  - 可以：
    - 在 skb 上打收到时间戳；
    - 按队列/设备/5-tuple 统计流量；
    - 看哪些包在 ingress 就被 drop/redirect。
- **TC egress**：
  - 发送方向上靠近设备的观测点；
  - 可以：
    - 打“准备发出”时间戳；
    - 结合 qdisc 状态估算排队时延。

与 XDP 区别：

- XDP：性能/位置更“硬”，但只看 raw packet，语义较弱；
- TC：稍靠后，有 skb，对流分类/统计友好。

---

## 3. kprobe / kretprobe：插在关键函数入口/出口的“细粒度钉子”

kprobe/kretprobe 可以挂到任意内核函数（入口/返回），在网络路径上特别关注这几个：

- `napi_poll`
- `netif_receive_skb` / `__netif_receive_skb_core`
- `tcp_sendmsg` / `udp_sendmsg`
- `tcp_recvmsg` / `udp_recvmsg`

### 3.1 在路径中的典型位置

按收发路径插点：

- `napi_poll`：
  - 在 `NET_RX_SOFTIRQ` 中，由 `net_rx_action` 调用；
  - 位置：从 RX 队列批量拉包、构造 skb、上交协议栈前。
- `netif_receive_skb`：
  - 所有 skb 进入上层协议栈前的公共入口；
  - 在 TC ingress/clsact 旁边。
- `tcp_sendmsg`：
  - 应用 send/write → sys_sendto/sys_write → sock_sendmsg → **tcp_sendmsg**；
  - 位置：TCP 发送入口，socket → IP 层之前。
- `tcp_recvmsg`：
  - 应用 recv/read → sys_recvfrom/sys_read → sock_recvmsg → **tcp_recvmsg**；
  - 位置：内核从 `sk_receive_queue` 把数据拷入用户缓冲的入口。

### 3.2 能看到 / 能做什么

挂 kprobe / kretprobe 后，可以：

- 访问函数参数：
  - 比如 `struct sock *sk`, `struct sk_buff *skb`, 长度等；
- 记录时间戳：
  - 入口记录 `start_ts`，返回时记录 `end_ts`，计算耗时；
- 读出额外上下文：
  - 当前 CPU、current task、cgroup、socket 关联信息等；
- 写入 BPF map：
  - 用 key（flow_id / tid / sock pointer）聚合统计。

典型延迟分解：

- `napi_poll` 入口/出口：
  - 测每轮 poll 的耗时和处理包数，分析 NAPI/softirq jitter。
- `netif_receive_skb`：
  - 联合 XDP/TC ingress 时间戳，测“从驱动到协议栈入口”的延迟。
- `tcp_sendmsg`：
  - 测应用发起 send 到离开 socket 层之间的时间，观察 socket 层瓶颈。
- `tcp_recvmsg`：
  - 测“数据从 socket 队列到用户缓冲”的时间，理解 socket 收数据延迟。

---

## 4. tracepoint：面向 net/sched/irq/softirq 的系统级“埋点”

tracepoint 是内核预定义的稳定 hook，适合用 BPF 做长期观测。

对网络/延迟特别有用的子系统：

- `net:*`
- `sched:*`
- `irq:*` / `softirq:*`

### 4.1 net: 子系统（网络事件）

典型 tracepoint：

- `net:netif_receive_skb`：
  - skb 进入协议栈时触发；
  - 类似在 `netif_receive_skb` 上的官方 hook。
- `net:net_dev_queue` / `net:net_dev_xmit`：
  - 发送路径上，包进入设备队列/开始发送时触发。
- `tcp:tcp_retransmit_skb`：
  - TCP 重传时触发。

用途：

- 全局统计：
  - 每个设备的收发 PPS、drops；
  - TCP 重传行为；
- 比 kprobe 更稳定，用于长期 metrics / 告警。

### 4.2 sched: 子系统（调度行为）

关键 tracepoint：

- `sched:sched_switch`：
  - 每次调度器切换当前任务时触发；
  - 包含：prev_pid/next_pid、优先级、CPU 等信息。

用途：

- 跟踪 feed handler / trading engine 线程是否被频繁抢占；
- 将“延迟尾巴升高”的时间段对齐到调度行为上：
  - 比如延迟 P99 拉高时，是否发生大量 sched_switch；
  - CPU 是否突然跑了其它 heavy 任务。

### 4.3 irq:/softirq: 子系统（中断与软中断）

常见 tracepoint：

- `irq:softirq_entry` / `irq:softirq_exit`：
  - 软中断进入/退出时触发；
  - 标记 softirq 类型（如 NET_RX_SOFTIRQ）。
- `irq:irq_handler_entry` / `irq:irq_handler_exit`：
  - 硬中断 handler 进入/退出时触发。

用途：

- 量化每个 CPU 上：
  - `NET_RX_SOFTIRQ` 执行时间；
  - 其他 softirq（TIMER_SOFTIRQ 等）的占比；
- 识别：
  - 某些时段 `NET_RX_SOFTIRQ` 异常偏高；
  - 某些 CPU 上软中断完全由 ksoftirqd 接管（说明压力过大）。

---

## 5. 收包路径上的“挂载点地图”（从 NIC 到应用）

把上述挂载点按 RX 路径顺序串起来：

1. **NIC / 驱动入口：**
   - **XDP**：
     - 位置：驱动 RX 最前面（raw packet，无 skb）；
     - 用途：极早的过滤/计数/入站时间戳。

2. **NAPI / 软中断：**
   - **kprobe/kretprobe：`napi_poll`**：
     - 位置：`NET_RX_SOFTIRQ` 中，由 `net_rx_action` 调度；
     - 用途：测 poll 时长、batch size，分析 NAPI 行为和 softirq 抖动。
   - **tracepoint：`irq:softirq_entry/exit`**：
     - 全局 softirq 行为。

3. **skb 创建 → netif_receive_skb：**
   - **TC ingress BPF**：
     - 位置：紧邻 netif_receive_skb 前后（clsact/ingress）；
     - 用途：ACL/计数/标记，第二阶段时间戳。
   - **tracepoint：`net:netif_receive_skb`**：
     - 官方稳定 hook，可做全局 RX 流量统计。
   - **kprobe：`netif_receive_skb`**（可选）：
     - 更细粒度的函数级观测。

4. **L3/L4 协议栈：**
   - **kprobe：`ip_rcv`, `tcp_v4_rcv`, `udp_rcv`**：
     - 用于分析特定协议行为、路由决策。
   - **tracepoint：`tcp:tcp_retransmit_skb` 等**：
     - TCP 重传等特定事件。

5. **socket 层 / 应用交互：**
   - **kprobe/kretprobe：`tcp_sendmsg`, `udp_sendmsg`**：
     - 位置：应用 send → 协议栈入口；
     - 用于测量发送路径上的 socket 层耗时。
   - **kprobe/kretprobe：`tcp_recvmsg`, `udp_recvmsg`**：
     - 位置：socket 队列 → 用户缓冲；
     - 用于测量“数据从 socket 队列到用户态”的延迟。

6. **系统行为背景：**
   - **tracepoint：`sched:sched_switch`**：
     - 全局调度行为；
   - **tracepoint：`irq:*`, `softirq:*`**：
     - 硬/软中断执行情况。

---

## 6. 综合：它们在延迟/性能分析中的角色分工

- **XDP：**
  - 最早的入口点；
  - 适合做“是否进系统”和“入站时间戳”的粗粒度判定。

- **TC ingress / egress：**
  - 在 skb 层，贴近设备；
  - 可做更丰富分类/计数；
  - ingress/egress 时间戳可用于分析排队/整形对延迟的影响。

- **kprobe / kretprobe：**
  - 在关键函数入口/出口打点；
  - 是延迟路径分段（NAPI、协议栈、socket）的主要工具。

- **tracepoint（net/sched/irq/softirq）：**
  - 提供系统级视角；
  - 帮你将“延迟异常窗口”和“调度/中断行为”对齐，诊断根因。

用一句面向 HFT/性能工程的总结：

> XDP/TC 给你流级、设备级的包路径钩子；  
> kprobe 给你函数级的精细时间戳；  
> tracepoint 则在 net/sched/irq 层面提供全局 context。  
> 把这三类数据拼起来，你就能从 NIC 到线程，把一条 feed/订单链路的延迟拆成清晰的几段。
