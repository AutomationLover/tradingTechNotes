# DPDK、XDP、AF_XDP、Onload/VMA 绕过哪些内核路径，以及对 tcpdump/iptables/tc 的影响

这篇说明围绕两个问题：

1. DPDK、XDP、AF_XDP、Onload/VMA 大致绕过了 Linux 网络栈中的哪些路径（尤其是 `netif_receive_skb` / `tcp_v4_rcv`）？
2. 这些绕行对传统工具（tcpdump、iptables、tc）的可见性有什么影响？

---

## 1. 标准 Linux 网络栈：作为对比基线

收包路径（高度简化）：

- NIC → DMA → 中断 / NAPI poll  
  - netif_receive_skb / __netif_receive_skb_core  
  - ip_rcv / ip_local_deliver  
  - tcp_v4_rcv / udp_rcv  
  - 查找 socket → 将数据放入 socket 接收队列  
  - 应用层 recv() / read() → tcp_recvmsg / udp_recvmsg

发包路径（简化）：

- 应用 send() / write()  
  - sys_sendto / sock_sendmsg  
  - tcp_sendmsg / udp_sendmsg  
  - ip_queue_xmit / ip_local_out  
  - dev_queue_xmit  
  - 驱动 → NIC TX ring → 线缆

传统工具挂在这些位置上：

- tcpdump：通过 AF_PACKET / netdev 层抓 skb；
- iptables：通过 netfilter hooks（PREROUTING / INPUT / OUTPUT / POSTROUTING 等）处理 skb；
- tc：在 qdisc / clsact 上处理 skb（含 TC-BPF）。

---

## 2. DPDK：完全绕开 `netif_receive_skb` / `tcp_v4_rcv` 的用户态引擎

### 2.1 绕过的路径

收包：

- NIC → DPDK PMD（用户态驱动）  
  - NIC 队列 → 用户态 hugepage ring → 用户态轮询线程

不走：

- 不创建 `sk_buff`；  
- 不进入 `netif_receive_skb` / `ip_rcv` / `tcp_v4_rcv`；  
- 不进入 socket 层。

发包：

- 应用调用 DPDK API（例如 rte_eth_tx_burst）；  
  - 构造 mbuf → 直接通过 PMD 写入 NIC TX ring；

不走：

- `tcp_sendmsg` / `ip_queue_xmit` / `dev_queue_xmit` 等内核发送路径。

### 2.2 对 tcpdump / iptables / tc 的可见性

- tcpdump：
  - 传统 tcpdump 只看到通过 netdev / skb 的流量；
  - DPDK-only 流量没有 skb，不经过 netdev；
  - 结果：tcpdump 看不到 DPDK 引擎的流量，除非用 NIC SPAN/mirror 或在 DPDK 内部增加 capture。
- iptables：
  - 不走 netfilter，iptables 规则对 DPDK 流量完全无效。
- tc：
  - 不经过 qdisc，tc filter / tc BPF 也观察不到这些包。

EAL 是 DPDK 的环境抽象层，用来适配底层平台和完成初始化。
在 DPDK 里，EAL = Environment Abstraction Layer（环境抽象层）。可以直接把它理解成：DPDK 的“运行时内核 + 启动器”。

---

## 3. XDP：在 `netif_receive_skb` 之前的驱动级 hook

### 3.1 位置与绕过路径

XDP 程序挂在驱动 RX path 最前面：

- NIC → 驱动 RX → XDP 程序 → （决定 DROP / PASS / TX / REDIRECT）

具体动作影响：

- XDP_DROP：
  - 在驱动层直接丢包；
  - 不创建 skb；
  - 不走 `netif_receive_skb` / `ip_rcv` / `tcp_v4_rcv`；
- XDP_TX / XDP_REDIRECT：
  - 在驱动层直接回发或重定向到其他设备/queue/AF_XDP；
  - 同样不进入标准协议栈。
- XDP_PASS：
  - 让包进入 NAPI → netif_receive_skb → 协议栈 → socket（正常路径）。

### 3.2 对工具的可见性影响

- tcpdump：
  - 对 XDP_DROP / XDP_REDIRECT / XDP_TX 的包：  
    - 没有 skb，不会到 netdev 层；  
    - tcpdump 看不到；
  - 对 XDP_PASS 的包：
    - 正常栈流程，tcpdump 能看到。
- iptables / tc：
  - 只有 PASS 的流量会进入 netfilter / qdisc；
  - XDP 在入口提前丢/改的流量，iptables / tc 完全感知不到。
- eBPF：
  - 可以在 XDP hook 上观测自己的 DROP / REDIRECT；
  - 但是后面的 TC/kprobe/tracepoint 只看到 PASS 之后的流。

---

## 4. AF_XDP：XDP + 用户态 ring 的轻量 bypass

### 4.1 位置与绕过路径

AF_XDP 是一个 socket 类型，通常配合 XDP 使用：

- NIC → 驱动 RX → XDP 程序  
  - 对特定流：XDP_REDIRECT 到 AF_XDP 队列（用户态 ring）；  
  - 对其他流：XDP_PASS，走普通内核栈。

被重定向到 AF_XDP 的包：

- 在 XDP 层就“拐弯”进入用户态共享 ring buffer；
- 不会创建 skb，不经过 `netif_receive_skb` / `ip_rcv` / `tcp_v4_rcv` / socket。

### 4.2 对工具的可见性影响

- tcpdump：
  - AF_XDP redirect 的包没有 skb，tcpdump 看不到；
  - PASS 的流量仍然可见。
- iptables / tc：
  - 对 AF_XDP 流量不起作用（不进 netfilter / qdisc）；
  - 仅对 PASS 流量生效。
- eBPF：
  - 可以在 XDP programs 和 AF_XDP 周边的 BPF hook 看见这类流；
  - 后面的 TC/tracepoint 仍然只能看到进入协议栈的流。

---

## 5. Onload / VMA：用户态 TCP/UDP 栈的 kernel-bypass

### 5.1 大致原理与绕过路径

Onload（Solarflare）和 VMA（Mellanox）是一类“用户态 TCP/UDP 栈”：

- 应用仍然写 POSIX socket API（socket/connect/send/recv 等）；
- Onload/VMA 的用户态库 hook 这些调用；
- 通过 NIC 驱动和 kernel-bypass 通道，把数据直接从 NIC 队列接到用户态栈：
  - 自己维护 TCP/UDP 状态机；
  - 不走 `tcp_v4_rcv` / `tcp_sendmsg` 等内核协议栈。

对被它们管理的 socket：

- 收包：NIC → 用户态库（Onload/VMA）→ 应用；
- 发包：应用 → 用户态库 → NIC；
- 标准内核栈在这些连接上的参与非常有限。

### 5.2 对工具的可见性影响

- tcpdump：
  - AF_PACKET 在 netdev/skb 层，只能看到走内核栈的流量；
  - Onload/VMA 的 fast path 流量不会经由 skb；
  - 根据实现，有可能完全看不到这些包，需使用：
    - NIC 的 SPAN/mirror；
    - 或 Onload/VMA 本身提供的调试/capture 功能。
- iptables / tc：
  - 这些 bypass 连接不会经过 netfilter 和 qdisc；
  - iptables / tc 无法对它们施加策略（ACL/限速等）。
- eBPF：
  - kprobe/tracepoint 在 `tcp_sendmsg` / `tcp_v4_rcv` 上看不到这些流；
  - 只能在更底层（NAPI/IRQ）和用户态（uprobes）观察。

---

## 6. 总结：绕过路径与工具可见性的整体对比

可以用如下视角来记住：

- **标准 Linux 栈：**
  - 所有包走 `netif_receive_skb` / `ip_rcv` / `tcp_v4_rcv` / `tcp_sendmsg`；
  - tcpdump / iptables / tc / 大多数 eBPF hook 都能看到并控制。

- **XDP：**
  - 在 `netif_receive_skb` 之前：
    - 对 DROP/REDIRECT/TX 的包：完全绕开后续协议栈和传统工具；
    - 对 PASS 的包：仍走完整内核栈，工具可见。

- **AF_XDP：**
  - 用 XDP 将特定流重定向到用户态 ring；
  - 这些流不创建 skb，不进入 `netif_receive_skb` / `tcp_v4_rcv`；
  - tcpdump / iptables / tc 对这部分流量不可见。

- **DPDK：**
  - 完整收发都在用户态 PMD 内；
  - 完全不走 netdev/skb，不进协议栈；
  - 所有基于内核栈的工具对 DPDK-only 流量统统看不到。

- **Onload / VMA：**
  - 为特定 socket 提供用户态 TCP/UDP 栈；
  - 这些 socket 的流量不经过内核 TCP/UDP 协议栈；
  - 对标准内核工具来说非常“隐形”，需要专门的集成或旁路监控。

一句话收尾：

> DPDK / Onload/VMA 是“尽量完全绕开内核栈”的 bypass；  
> XDP / AF_XDP 则是在更靠前的位置给你一个开关，让你选择：哪些流依然走 `netif_receive_skb` / `tcp_v4_rcv` 和传统工具，哪些流直接走侧道到用户态。
