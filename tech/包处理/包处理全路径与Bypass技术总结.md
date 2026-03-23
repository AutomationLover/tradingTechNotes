# 数据包从网卡进入后的完整处理路径与 Bypass 技术总结

本文综合整理「包进入网卡后的完整处理流程」，从传统 Linux 内核路径出发，梳理 eBPF 各挂载点、XDP 插入位置，以及不同 bypass 技术在何时开始绕过内核。

---

## 1. 传统 Linux 内核收包路径概览

### 1.1 整体流程：从网线到用户态

```
NIC 硬件
    ↓
驱动 & DMA / IOMMU (IOVA)
    ↓
硬中断 → NAPI 轮询 (SoftIRQ)
    ↓
构造 skb → netif_receive_skb
    ↓
L2/L3/L4 协议栈 (eth_type_trans → ip_rcv → tcp_v4_rcv/udp_rcv)
    ↓
netfilter (iptables) hook 点
    ↓
Socket 接收队列 (sk_receive_queue)
    ↓
应用 recv() / read() → tcp_recvmsg / udp_recvmsg
```

### 1.2 关键阶段详解

#### 阶段一：硬件与 DMA

- **NIC RX 队列**：每个网卡有多个 RX 队列，每队列对应描述符 ring buffer
- **DMA**：网卡通过 DMA 将数据直接写入主内存，不经过 CPU
- **IOMMU/IOVA**：存在 IOMMU 时，驱动给 NIC 的是 IOVA，由 IOMMU 转成物理地址

#### 阶段二：硬中断 → NAPI → SoftIRQ

1. **硬中断**：NIC 收包、DMA 完成后触发 RX 中断
2. **NAPI 模式**：硬中断 handler 只做 `napi_schedule()`，禁用该队列中断，快速返回
3. **NET_RX_SOFTIRQ**：软中断被标记为 pending
4. **net_rx_action**：遍历 NAPI 队列，调用各驱动的 `poll()` 回调
5. **NAPI poll**：从 RX ring 批量取包、构造 skb、上送 `netif_receive_skb` 或 `napi_gro_receive`

#### 阶段三：协议栈入口 → Socket

- **netif_receive_skb**：L2 解析（`eth_type_trans`），按 protocol 分发
- **ip_rcv**：IP 层校验、路由查找（本机 vs 转发）
- **ip_local_deliver** → **tcp_v4_rcv** / **udp_rcv**：根据四元组查 sock，数据挂入 `sk_receive_queue`
- **recv()**：`sys_recvfrom` → `sock_recvmsg` → `tcp_recvmsg`，从 socket 队列取数据拷贝到用户态

### 1.3 发包路径（对照）

```
应用 send() / write()
    ↓
sys_sendto / sys_write → sock_sendmsg
    ↓
tcp_sendmsg / udp_sendmsg
    ↓
ip_queue_xmit → ip_local_out
    ↓
netfilter OUTPUT / POSTROUTING (iptables)
    ↓
dev_queue_xmit → qdisc (tc)
    ↓
驱动 TX → NIC DMA 发送
```

---

## 2. eBPF 挂载点与内核处理位置对照

### 2.1 收包路径上的挂载点地图

按「从 NIC 到应用」的顺序：

| 挂载点类型 | 内核位置 | 能看到什么 | 典型用途 |
|-----------|----------|------------|----------|
| **XDP** | 驱动 RX 最前面，NAPI poll 内、**创建 skb 之前** | raw packet，无 skb | 极早过滤、ACL、DDoS、入站时间戳、redirect 到 AF_XDP |
| **kprobe: napi_poll** | NET_RX_SOFTIRQ 中，由 net_rx_action 调用 | poll 上下文 | 测 poll 时长、batch size、NAPI 行为 |
| **tracepoint: irq:softirq_*** | 软中断进入/退出 | softirq 类型、CPU | 量化 NET_RX_SOFTIRQ 执行时间 |
| **TC ingress BPF** | 紧邻 netif_receive_skb 前后（clsact） | 已有 skb，有 __sk_buff | ACL、计数、标记、mirror、第二阶段时间戳 |
| **tracepoint: net:netif_receive_skb** | skb 进入协议栈时 | skb 元数据 | 全局 RX 流量统计 |
| **kprobe: netif_receive_skb** | 协议栈通用入口 | skb、协议信息 | 更细粒度函数级观测 |
| **kprobe: ip_rcv, tcp_v4_rcv, udp_rcv** | L3/L4 协议处理 | 协议层上下文 | 路由决策、特定协议行为 |
| **kprobe: tcp_recvmsg, udp_recvmsg** | socket 队列 → 用户缓冲 | sock、长度 | 测「socket 到用户态」延迟 |

### 2.2 发包路径上的挂载点

| 挂载点类型 | 内核位置 | 典型用途 |
|-----------|----------|----------|
| **kprobe: tcp_sendmsg, udp_sendmsg** | 应用 send → 协议栈入口 | 测 socket 层发送耗时 |
| **tracepoint: net:net_dev_queue / net_dev_xmit** | dev_queue_xmit 前后 | 发送流量统计 |
| **TC egress BPF** | dev_queue_xmit 之后，qdisc 阶段 | 标记、计数、排队时延分析 |

### 2.3 关键结论

- **XDP**：最靠前，在 skb 创建之前，是「内核第一次看到这个包」的点
- **TC ingress/egress**：有 skb，贴近设备，对流分类和统计更友好
- **kprobe/kretprobe**：函数级钉子，用于延迟分段（NAPI、协议栈、socket）
- **tracepoint (net/sched/irq/softirq)**：系统级稳定 hook，用于全局 context 对齐

---

## 3. XDP 插入位置详解

### 3.1 精确插入点

```
NIC → DMA 写入 RX ring
    ↓
驱动 RX 处理开始
    ↓
NAPI poll 从 RX ring 取描述符，得到 DMA 完成的 raw buffer
    ↓
★★★★★ XDP 程序在此处执行 ★★★★★  ← 尚未创建 skb
    ↓
XDP_DROP / XDP_TX / XDP_REDIRECT → 不进入后续路径
XDP_PASS → 继续创建 skb → netif_receive_skb → 协议栈
```

### 3.2 XDP 返回值与路径

| 返回值 | 后续路径 |
|--------|----------|
| **XDP_DROP** | 直接丢包，不创建 skb，不进入 netif_receive_skb / ip_rcv / tcp_v4_rcv |
| **XDP_TX** | 在驱动层直接回发，不进入协议栈 |
| **XDP_REDIRECT** | 重定向到其他设备/队列/AF_XDP，不进入协议栈 |
| **XDP_PASS** | 正常进入 NAPI → netif_receive_skb → 完整内核栈 |

### 3.3 XDP 模式

- **Native XDP**：在驱动 RX 路径最前面执行，性能最高
- **Generic XDP**：在 netif_receive_skb 之后、skb 已创建时执行，兼容性好
- **Offload XDP**：程序卸载到网卡硬件执行

---

## 4. 不同 Bypass 技术的「开始绕过」时机

### 4.1 标准 Linux 栈（基线）

所有包经过：`netif_receive_skb` → `ip_rcv` → `tcp_v4_rcv` / `udp_rcv` → socket → `tcp_recvmsg`。  
tcpdump、iptables、tc、大多数 eBPF hook 均可见。

### 4.2 XDP：在 netif_receive_skb 之前的「选择性绕过」

| 场景 | 绕过时机 | 绕过内容 |
|------|----------|----------|
| XDP_DROP | 驱动 RX、创建 skb 之前 | 不创建 skb，不进 netif_receive_skb / ip_rcv / tcp_v4_rcv |
| XDP_TX / XDP_REDIRECT | 同上 | 驱动层直接回发或重定向，不进协议栈 |
| XDP_PASS | 无绕过 | 正常走完整内核栈 |

**对工具影响**：DROP/REDIRECT/TX 的包无 skb，tcpdump / iptables / tc 不可见；PASS 的包仍可见。

### 4.3 AF_XDP：XDP + 用户态 ring 的轻量 bypass

- **绕过时机**：在 XDP 层，通过 `XDP_REDIRECT` 把特定流导向 AF_XDP socket 的 umem ring
- **绕过内容**：被 redirect 的包不创建 skb，不经过 netif_receive_skb / ip_rcv / tcp_v4_rcv / socket
- **数据路径**：NIC → 驱动 RX → XDP (REDIRECT) → 用户态 AF_XDP ring → 用户态应用
- **对工具影响**：AF_XDP 流量 tcpdump / iptables / tc 不可见；PASS 流量仍可见

### 4.4 DPDK：完全用户态 PMD，最早 bypass

- **绕过时机**：从「网卡收包」那一刻起，即不进入内核网络栈
- **绕过内容**：整条路径在用户态完成
  - 收包：NIC → DPDK PMD（用户态驱动）→ 用户态 hugepage ring → 轮询线程
  - 发包：应用 → DPDK API → mbuf → PMD → NIC TX ring
- **不经过**：不创建 skb、不经过 netif_receive_skb / ip_rcv / tcp_v4_rcv / tcp_sendmsg / dev_queue_xmit
- **对工具影响**：tcpdump / iptables / tc 对 DPDK-only 流量完全不可见，需 NIC SPAN 或 DPDK 内部 capture

### 4.5 Onload / VMA：用户态 TCP/UDP 栈的 bypass

- **绕过时机**：对「被管理的 socket」而言，从 NIC 收包 / 发包即绕过内核协议栈
- **绕过内容**：
  - 应用仍用 POSIX socket API（socket/connect/send/recv）
  - 用户态库 hook 这些调用，通过 NIC 驱动与 kernel-bypass 通道直接对接
  - 自己维护 TCP/UDP 状态机
- **不经过**：tcp_v4_rcv / tcp_sendmsg 等内核协议栈
- **对工具影响**：fast path 流量无 skb，tcpdump / iptables / tc 基本不可见，需 NIC SPAN 或厂商调试功能

### 4.6 Bypass 时机对比图

```
                    传统路径                     Bypass 开始点
                    ─────────                    ──────────────

NIC 收包
    │
    ▼
DMA → RX ring
    │
    ▼
硬中断 → NAPI poll
    │
    ├──────────────────────────────────────────► DPDK：此处即完全用户态，不进内核
    │
    ▼
驱动取包，准备构造 skb
    │
    ├──────────────────────────────────────────► XDP (DROP/TX/REDIRECT)：此处执行后即绕过
    │                                              AF_XDP：XDP REDIRECT 在此处拐入用户态
    ▼
创建 skb
    │
    ▼
netif_receive_skb / TC ingress
    │
    ├──────────────────────────────────────────► Onload/VMA：对管理的 socket，数据不经过此处
    │                                              （实际 bypass 更早，在 NIC 到用户态库之间）
    ▼
ip_rcv → tcp_v4_rcv → socket
    │
    ▼
应用 recv()
```

---

## 5. 小结

1. **传统内核路径**：NIC → DMA → 中断/NAPI → skb → netif_receive_skb → L2/L3/L4 → netfilter → socket → 用户态。
2. **eBPF 挂载点**：XDP 最前（无 skb）→ TC ingress（有 skb）→ kprobe/tracepoint 分布在 NAPI、协议栈、socket 各层。
3. **XDP 插入点**：驱动 RX 处理最前面，NAPI poll 拿到 raw packet 后、创建 skb 之前。
4. **Bypass 时机**：
   - **DPDK**：最早，从网卡起即完全用户态；
   - **XDP (DROP/TX/REDIRECT)**：在驱动层、skb 创建前；
   - **AF_XDP**：同 XDP，通过 REDIRECT 拐入用户态 ring；
   - **Onload/VMA**：对管理的 socket，从 NIC 到用户态库的 fast path 不经过内核协议栈。

---

## 参考文章（本仓库 linux 目录）

- `网卡收发包.md`：硬件、DMA、NAPI 整体视角
- `硬中断到NAPIpoll.md`：硬中断 → SoftIRQ → NAPI poll 详细流程
- `NAPIpoll到socket.md`：从 NAPI 到协议栈、socket 的完整路径
- `收发包路径.md`：tcp_sendmsg / tcp_recvmsg / sys_sendto / sys_recvfrom 的调用链
- `BPF挂载点.md`：网络相关 BPF 挂载点与内核位置
- `kernel-bypass.md`：DPDK、XDP、AF_XDP、Onload/VMA 的绕过路径与工具可见性
- `iptableTC.md`：iptables 与 tc 在内核中的位置与关系
