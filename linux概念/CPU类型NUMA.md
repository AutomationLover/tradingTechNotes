# SMP 与其他处理模式

## 1. UP（单处理器）与 SMP（对称多处理）

### UP（Uniprocessor）

- 整个系统只有一个 CPU/逻辑处理器。
- 任意时刻只会有一条执行路径在跑内核代码。
- 仍然有并发：比如当前正在跑内核代码时被中断打断，但**不会**出现“两个 CPU 同时修改同一块内存”的情况。
- Linux 在 `CONFIG_SMP` 关闭时，一些锁（例如自旋锁）会退化成空操作或简单检查，因为从硬件角度看不会有跨 CPU 并发。

### SMP（Symmetric MultiProcessing）

- 有多个地位对等的 CPU/核心，**共享同一份物理内存**。
- 任意 CPU 都可以运行任意进程、执行任意内核代码、响应中断。
- 内核必须处理：
  - 多个 CPU 并发访问同一数据结构；
  - 同一个 IRQ 可以打到不同 CPU（通过 `/proc/interrupts` 和 `/proc/irq/*/smp_affinity` 来观察和控制）。
- 为此内核引入：
  - 自旋锁、RCU、原子操作、per‑CPU 变量等同步机制；
  - 调度器和中断子系统都要考虑 CPU 亲和性和负载均衡。

现代 x86_64/ARM 服务器与 PC 基本都是 “多核 + 超线程 + SMP” 的组合。

---

## 2. SMP 内部的几种硬件形态

虽然统称 SMP，但硬件实现可以是不同形态，对内核来说都表现为“多个 CPU + 共享内存”。

### 多核（Multi-core）

- 一个物理封装里有多个核心（core），每个核心在内核视角里是一个 CPU。
- 通常共享某些缓存层级（L2/L3）和内存控制器。

### SMT / Hyper-Threading

- 一个核心同时暴露多个逻辑 CPU（硬件线程），共享执行单元与缓存。
- 在 `/proc/cpuinfo` 里看到的 cpu0/cpu1 可能是一个物理 core 上的两个硬件线程。
- 对内核而言，仍当作多个 SMP CPU 调度。

### NUMA（Non‑Uniform Memory Access）

- 整体仍是 SMP（所有 CPU 都能访问全部物理内存），但**访问不同内存区域的延迟不同**。
- 典型双路/多路服务器：每路 CPU 有自己的本地内存，访问本地内存延迟低，访问远端节点需要跨互连总线。
- Linux 在此基础上引入：
  - Node 概念、NUMA zone；
  - `numactl`、`/sys/devices/system/node` 等接口；
  - 内存分配尽量“本地化”，调度器考虑 CPU/内存拓扑。

可以简单记：NUMA 是“有拓扑和延迟差异的 SMP”。

---

## 3. 与 SMP 对比的其他模式

### AMP（Asymmetric MultiProcessing，非对称多处理）

- 有多个 CPU/核，但**角色不对等**：
  - 一些核跑通用 OS（如 Linux）和业务代码；
  - 另一些核跑 RTOS、裸机程序或专用固件，用于特定任务（音频处理、DSP、通信协议等）。
- 各 CPU 可能有各自独立内存或部分共享内存，通过 mailbox、共享内存、消息队列通信。
- 与 SMP 区别：
  - SMP：所有 CPU 都能跑同一个内核和同一套任务调度，地位对称；
  - AMP：CPU 之间职责固定、不互相抢任务，甚至不一定跑同一套 OS。

很多嵌入式 SoC（ARM 大核 + 小核/DSP/MCU 组合）常见 AMP 场景：Linux 跑在 A 核上，旁边的 M 核跑裸机控制程序。

### MPP / Cluster（大规模并行 / 集群）

- 多台完整的**整机**（每台机子内部可能是 SMP），通过网络互联：
  - 每台机器有自己的 CPU、内存、设备；
  - 机器之间不共享物理内存，只能通过网络传输数据。
- 上层通过 MPI、分布式存储、消息队列等实现协同。
- 对比 SMP：
  - SMP：多个 CPU 共享同一块物理内存，内核靠锁/RCU 保护共享数据；
  - Cluster/MPP：多机分布式，每台机器独立运行 OS，靠网络协议协作。

---

## 4. Linux 内核视角：为什么 “是否 SMP” 很重要

### 编译时维度（CONFIG_SMP）

- 未开启 `CONFIG_SMP`：
  - 内核假定只有 1 个 CPU；
  - 很多自旋锁、per‑CPU 机制会被优化简化甚至直接去掉；
  - 并发模型仅包含“进程/线程 + 中断”，没有“多 CPU”这一层。

- 开启 `CONFIG_SMP`：
  - 引入真正的自旋锁实现、自适应锁、RCU、调度器中的多 CPU 支持；
  - IRQ 子系统支持 `smp_affinity`、`ksoftirqd/N`、跨 CPU IPI；
  - 内核各子系统都要考虑并发访问和 cache ping‑pong。

### 运行时维度（你能看到的）

- `/proc/interrupts` 有多列 CPU 计数，就说明在 SMP；可以看到某个 IRQ 如何在多个 CPU 之间分布。
- `/proc/irq/*/smp_affinity` / `smp_affinity_list` 用来“绑核”，比如把网卡中断 pin 到某个 core。
- 调度器会在多个 CPU 之间迁移进程，负载均衡；NUMA 系统还会考虑节点亲和。
- 你在笔记里研究的：
  - 自旋锁；
  - per‑CPU 数据结构；
  - RCU；
  - 中断 affinity；
  - workqueue/ksoftirqd 绑核；
  
  本质上都是围绕 **“在 SMP（特别是多核 + NUMA）上如何又快又安全地并发”** 展开的。

---

## 5. 从实际应用场景来记（网络 / HFT / 内核调优视角）

- 在 **UP** 上：
  - 没有跨 CPU 并发，同一时刻只有一个执行路径跑内核；
  - 锁竞争只是逻辑上的，并不会有真实的 cache 双向抖动。
- 在 **SMP（多核）** 上：
  - 网卡中断、softirq、ksoftirqd、用户线程可以在不同核上同时抢同一个队列或 socket；
  - 需要考虑：
    - 合理配置中断/队列的 `smp_affinity`，做到 “RX queue/softirq/用户线程 同核”；
    - 用 per‑CPU 队列减少跨核共享状态；
    - 用自旋锁/RCU 避免数据竞争。
- 在 **NUMA + SMP** 上：
  - 还要关注“CPU + 内存”是否在同一个 NUMA 节点：
    - 绑定进程到特定 NUMA node；
    - 使用 node‑local 内存分配策略；
    - 将 NIC 中断和处理线程绑在同一 node，减少远程内存访问。

可以把核心记成一句话：

> SMP = 多核共享内存，是 Linux 内核里所有锁、自旋、RCU、per‑CPU、IRQ 亲和、NUMA 调优的基础模式；  
> UP/AMP/Cluster 则是从“是否共享内存、CPU 是否对称”不同维度上的其他处理模式。
