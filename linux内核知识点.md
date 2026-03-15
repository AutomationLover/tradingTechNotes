# 针对 IMC 招聘的 Linux 内核知识点清单

基于那份 IMC Quantitative Microstructure Analyst / Performance Team 的 JD，Linux 内核这块如果想对标，重点可以按下面几个方向准备。

---

## 1. 网络栈 & 收发包路径（最高优先级）

围绕“从 NIC 到应用、再从应用到 NIC”的完整链路，理解：

### 1.1 驱动 + NAPI / 中断模型

- RX/TX ring buffer、DMA 的基本概念：网卡如何把包写到内存、如何从内存读出。
- 硬中断 → `NET_RX_SOFTIRQ` → `net_rx_action` → **NAPI poll** 的完整流程。
- 收包时批处理策略：
  - `net.core.netdev_budget` 等参数如何控制一次 poll 处理多少包。
  - 中断 coalescing（中断聚合）如何影响延迟与吞吐。
- `ksoftirqd` 在什么时候接手软中断，为什么会拉高 tail latency。

### 1.2 协议栈数据路径（尤其是 RX）

- 收包主路径：
  - `netif_receive_skb` / `__netif_receive_skb_core`
  - `ip_rcv` / `ip_local_deliver`
  - `tcp_v4_rcv` / `udp_rcv`
- `sk_buff` 的结构和生命周期：
  - 分配、引用计数、clone、free 对 cache 和延迟的影响。
- GRO / GSO / TSO / LRO：
  - 在栈中的位置；
  - 对 PPS / 吞吐 vs 延迟的 trade-off。

### 1.3 多队列 & 多核利用

- RX/TX 多队列机制：
  - **RSS / RPS / RFS / XPS** 的原理和配置方式；
  - 如何把某个 RX 队列 pinned 到特定 CPU（irq affinity、`/proc/irq/*/smp_affinity`）。
- 这些机制如何影响：
  - feed handler / trading engine 线程的 CPU 亲和性；
  - cache locality、CPU 负载均衡。

### 1.4 socket 层

- 发送路径：
  - `sys_sendto` / `sys_write`
  - `sock_sendmsg`
  - `tcp_sendmsg` / `udp_sendmsg`
- 接收路径：
  - `sys_recvfrom` / `sys_read`
  - `sock_recvmsg`
  - `tcp_recvmsg` / `udp_recvmsg`
- socket 队列：
  - `sk_receive_queue` / send queue；
  - backlog、半连接队列对行为的影响。

### 1.5 fast-path / bypass 技术位置

- 了解 DPDK、XDP、AF_XDP、Onload/VMA 等 roughly 绕过了哪些内核路径：
  - 不走 `netif_receive_skb` / `tcp_v4_rcv` 的场景；
  - 对 tcpdump、iptables、tc 这类工具的可见性影响。
- eBPF 在 XDP / TC / socket / cgroup hook 上能做什么：
  - 观测、过滤、限速、简单 LB。

---

## 2. 调度器、CPU 亲和性 & 中断（控制 jitter）

目标是能解释：为什么只有 P99 / P99.9 变差而中位数没变化。

### 2.1 调度基础

- CFS 调度原则、时间片、负载计算的大致概念。
- RT 调度类（SCHED_FIFO / SCHED_RR）的行为与风险（可能饿死其他线程）。
- 抢占点：
  - syscalls、软中断、内核临界区等大致有哪些位置会产生抢占。

### 2.2 CPU 亲和性与隔离

- `sched_setaffinity` / `taskset`：如何把关键线程固定在指定 CPU。
- `isolcpus`、`nohz_full`、`rcu_nocbs` 等 boot 参数对延迟的影响。
- 把“背景任务 / ksoftirqd / housekeeping 线程”赶到非关键核的策略。

### 2.3 中断与软中断

- 硬中断 vs 软中断：
  - 谁可以打断谁；
  - 谁不能睡眠；
  - 谁更危险（latency/jitter 角度）。
- `ksoftirqd`：
  - 在软中断干不完时接手；
  - 如何通过它的 CPU 使用率判断网络/调度是否出问题。
- 如何用：
  - `/proc/interrupts`、`/proc/softirqs`；
  - 加上 eBPF / tracepoint，看某核是否被 IRQ/softirq 打爆。

---

## 3. 内存子系统 & NUMA（关注延迟相关部分）

不需要成为 VM 专家，但要理清和 tail-latency 相关的点。

### 3.1 页分配与 SLUB

- 基本内存分配方式：`kmalloc` / `vmalloc` 大致区别。
- 高频路径里避免频繁分配/释放带来的：
  - page fault；
  - SLUB 慢路径；
  - 对 cache 的冲击。

### 3.2 NUMA

- NUMA node 的概念；
- 本地 vs 远程内存访问的延迟差；
- NIC PCIe 插槽在哪个 NUMA 节点、线程/进程跑在哪个 node 上：
  - 如何用 `numactl`、cgroups 来 colocate 进程与 NIC 以及内存。

### 3.3 Huge pages & TLB

- 为什么 hugepage 减少 TLB miss，对 tail-latency 很关键；
- 在 DPDK / Onload / 自研引擎中 hugepage 几乎是标配；
- 如何观察 TLB miss / page fault 对延迟的影响（perf + eBPF）。

---

## 4. 文件 & I/O 接口（相对次要，但要看得懂）

JD 里提到“memory, network, and file I/O”，在 HFT 风格环境中：

- 核心撮合路径一般不用磁盘做同步；
- 但日志、监控、持久化的 I/O 行为如果绑错核/配置不当，照样会影响延迟。

需要理解：

- VFS 基本模型：文件描述符、page cache、writeback 大概怎么工作；
- `read` / `write` / `fsync` 在内核里可能怎样阻塞；
- 如何用 eBPF / tracepoint：
  - 看哪些线程在做大量 sync I/O；
  - 某段时间是否有大规模 writeback 干扰。

---

## 5. 内核可观测性接口：perf + eBPF hooks

JD 点名了 eBPF，意味着希望你能用它们当“显微镜”。

### 5.1 基本工具

- perf：
  - 查看 CPU 占用、hotspots、cache miss、TLB miss；
- tracepoint / ftrace：
  - net:netif_receive_skb
  - sched:sched_switch
  - irq:* 等事件；
- BPF maps：
  - hash、array、per-CPU map、ringbuf；
  - 如何在不影响性能的前提下用它们收集指标。

### 5.2 关键 BPF 挂载点（跟网络/延迟相关）

- XDP：
  - 极靠前的包过滤 / drop / 粗统计；
- TC ingress / egress：
  - 在 skb 上做计数、ACL、时间戳采样；
- kprobe / kretprobe：
  - napi_poll、netif_receive_skb、tcp_sendmsg、tcp_recvmsg 等关键函数；
- tracepoint：
  - net:*、sched:*、softirq:* 等 tracepoint 用于全局系统行为观测。

### 5.3 写“低成本” BPF 的意识

- 使用 per-CPU map 避免锁竞争；
- 做抽样，避免全量 trace；
- 避免在热路径中使用 `bpf_trace_printk` 或频繁 perf event 输出；
- 精简数据结构，以“能回答问题”为目标而不是“全都抓”。

---

## 6. 与 kernel-bypass / FPGA 的接口认知

JD 还提到：

- ultra-low latency environments；
- kernel network bypass technologies。

这部分对于 Linux 内核知识的要求是：

- 能清楚地对比：

  - 标准内核网络栈 + NAPI  
  vs  
  - DPDK / Onload / VMA / FPGA pipeline

- 知道它们绕过了哪些内核路径：
  - 哪些地方不再走 `netif_receive_skb`、`tcp_v4_rcv` 等；
  - 标准工具（tcpdump、iptables、tc、eBPF TC hook）在哪些情况下看不到 bypass 流量。
- 明白在这种架构里，内核栈仍然负责：
  - 管理面、监控流量；
  - 其他非极限延迟的业务流；
  - eBPF 在这些流量上的观测依然关键（避免干扰 hot cores、监控整体健康状态）。

---

## 7. 面试时可以总结成的一句话

在面试里，你可以把上面的内容浓缩成类似这样的定位：

> 在 Linux 内核方面，我重点掌握了网络收发路径（驱动/NAPI、中断、协议栈、socket 层）、调度与 NUMA 对 tail-latency 的影响，以及如何用 perf、kprobe/tracepoint、XDP/TC 这些 eBPF hook 在不干扰系统行为的前提下，对低延迟交易环境中的内核性能和 jitter 做精细观测和诊断。同时，我理解 DPDK / XDP / kernel-bypass / FPGA 在栈中的位置和取舍，而不是只停留在概念上。

这句背后的每一块，就是上面清单里的那些具体知识点。
