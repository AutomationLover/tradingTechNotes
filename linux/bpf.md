用户态通过文件描述符访问 BPF map（用 ‎`bpf()` 或 libbpf/bcc 帮你封装），map 则在 BPF 程序里用 C 结构定义，由内核分配内存。

真正的数据存放并不是在你的进程地址空间，而是在：

- 内核为这个 map 对象单独分配的内存（比如 kmalloc、vmalloc、per-CPU 区等）；

- 你只能通过 map 的 fd + syscalls 间接访问，不能直接拿到裸指针。

---

从工程视角看，“**eBPF 体系**”就是：在 Linux 内核里用一套安全可验证的字节码和运行时，把“内核可编程”系统化标准化的一整套机制和生态。

可以拆成四块来记。

## 1. 核心：eBPF 虚拟机 + Verifier

内核里有一个小型 64 位虚拟机：

- 10 个 64-bit 寄存器 `R0–R9`，`R10` 为 frame pointer。  
- 自己的一套指令集（算术、跳转、内存访问、调用 helper 等）。  
- eBPF 程序本身是字节码，由 clang/llvm 从 “BPF C” 编出来。

为了安全，所有 eBPF 程序加载进内核前都必须过 **Verifier**：

- 检查程序必然终止（不能有无限循环）。  
- 检查所有内存访问有边界（map / 堆栈 / packet）。  
- 检查调用链深度、helper 使用是否合法。  

这保证了“你可以在内核里跑动态代码，但不会搞崩内核”。

## 2. 结构：Program + Hook + Map + Helper

整个 eBPF 体系抽象非常统一，基本四个概念：

1. **Program**  
   - 一段 eBPF 字节码，对应具体类型：XDP、tc、kprobe、tracepoint、cgroup、LSM 等。  
   - 每种类型规定了“在什么事件被触发”“参数是什么”。

2. **Hook / Attach 点**  
   - Program 附着到内核的某个事件点上：  
     - 网络：XDP、tc ingress/egress、socket。  
     - 内核事件：kprobe/kretprobe、tracepoint。  
     - 用户态：uprobe/uretprobe。  
     - 安全：LSM hook。  
   - 事件发生 → 内核调用对应的 eBPF Program。

3. **Map**  
   - 内核态/用户态共享的 key–value 存储：hash、array、per-CPU、LRU、ringbuf 等。  
   - eBPF 程序用 map 存状态、统计、缓存，用户态程序也可以通过同一个 map 读写。

4. **Helper 函数**  
   - 内核暴露的 API，eBPF 程序通过 `bpf_call` 调用：  
     - 操作 map（lookup/update/delete）。  
     - 获取当前 pid/tgid/cgroup/netns 等元数据。  
     - 读写 packet、重定向、封包解包（XDP/tc）。  
     - 输出事件到 perf ringbuf / ringbuf map。  

这四块组合起来，就能在各种内核路径上做观测、过滤、修改和策略执行。

## 3. 用户态：bpf() 系统调用 + 工具栈

内核通过一个统一的 `bpf()` syscall 暴露 eBPF 能力：

- 加载 program（`BPF_PROG_LOAD`）。  
- 创建 / 操作 map（`BPF_MAP_CREATE/UPDATE_ELEM/LOOKUP_ELEM`）。  
- attach/detach program 到具体 hook（近年还有 BPF link 抽象）。  

在此之上，形成了完整的工具栈：

- **libbpf**：官方 C 库，直接对接 bpf() + BTF + CO-RE。  
- **bpftool**：命令行管理/调试（列 program/map、反汇编、pin 等）。  
- **BCC / bpftrace**：高级封装（Python/Lua DSL、类似 awk 的 tracing 语言）。  
- 高层项目：Cilium、Falco、Katran、bcc-tools、各种 observability/安全/网络项目。

## 4. 应用版图：从抓包到“内核可编程平台”

在 eBPF 体系上，现在主要几大类应用：

- **网络数据面**  
  - XDP：L2/L3 极早期丢包、过滤、防 DoS、load balancer。  
  - tc-BPF：L3/L4 方向的调度/整形/封装（Geneve/VXLAN 等）。  
  - CNI：Cilium 等，用 eBPF 做 datapath、policy、load balancing。

- **可观测性 / Tracing**  
  - kprobe/tracepoint/uprobe → bpftrace / BCC → 一行命令看任意函数调用、延迟分布。  
  - 系统性能分析、应用调用链、内核内部行为可视化。

- **安全 / 策略**  
  - LSM BPF：在安全钩子里做细粒度访问控制。  
  - 审计、入侵检测（Falco）、runtime policy enforcement。

- **其他**  
  - cgroup 限流、资源记账。  
  - 自定义协议处理、特殊场景的数据面逻辑。

本质上：**eBPF 把“写 C 改内核、写模块”升级成“写受控小程序挂 hook”**，并用 VM + Verifier + map + helper + bpf() 这整套体系，标准化了“内核内的可编程扩展层”。

> 一句话总结：  
> eBPF 体系 = 以 eBPF VM 为核心，配合 Verifier、Program 类型与 Hook、Map、Helper、bpf() syscall 以及整套工具/生态，构成的 Linux 通用内核可编程平台。
