# ftrace：内核 tracing 基础设施

ftrace 是内核的基础设施，很多日常会用到的工具和框架都建立在 ftrace 之上。常见依赖关系大致可分为以下几类：

---

## 1. 内核自带工具和接口

最直接的是内核自带的 tracing 接口和配套工具，它们都以 ftrace 为底座：

### tracefs / `/sys/kernel/debug/tracing`

通过 `current_tracer`、`set_ftrace_filter`、`trace_pipe` 等文件启用和读取 ftrace，是所有 tracing 工具的汇合点。

### trace-cmd

Red Hat 出的 CLI 工具，将 ftrace 的各类文件操作封装起来：

- `trace-cmd record`、`trace-cmd report`、`trace-cmd stat` 等
- 用于采集和回放内核 trace，是用 ftrace 做系统级 trace 的主力工具之一

### kernelshark

图形界面（GTK/Qt）查看 trace-cmd 录到的 ftrace 数据，支持时间轴分析、事件过滤、CPU/任务视图对齐等。

### perf 的部分子系统

perf 主要基于 PMU/tracepoint，但部分事件和模式会复用 ftrace（例如早期的 function tracing、某些 sched/irq 事件），可以理解为与 ftrace 共用一部分 tracing 管道。

---

## 2. BCC / bpftrace 等 eBPF 工具中的「execsnoop 一类脚本」

早期的 `execsnoop`、`opensnoop`、`biolatency` 等脚本多基于 ftrace + kprobe 实现：

- 在 `do_execve()`、`vfs_read()`、block 层函数等上挂 ftrace/kprobe；
- 将事件写入 trace buffer；
- 用户态脚本从 `/sys/kernel/debug/tracing/trace_pipe` 读取并格式化输出。

之后逐渐迁移到 eBPF + tracepoint/kprobe，但思路一致，依然依赖内核 tracing 基础设施（ftrace + tracepoint）提供的数据流。

因此，像：

- BCC 工具集中的老版 `execsnoop`、`opensnoop`、`ext4slower` 等；
- 一些带 `-t`/`-p` 参数的小脚本；

要么直接操作 ftrace，要么复用 ftrace 的 tracepoint/kprobe 注册能力。

---

## 3. 系统级 tracing/分析工具

一些更大型的系统 tracing 工具，也在不同程度依赖 ftrace：

### LTTng / SystemTap（部分模式）

虽有自己的探针机制，但在某些内核事件上会复用 tracepoint / ftrace 的设施。

### Android systrace / AOSP 内的 tracing

早期就是把 ftrace 的输出包装成更直观的 UI（浏览器时间轴），大量用于 UI 卡顿、scheduler 等问题的调试。

---

## 总结

> **ftrace** 是内核层的「事件总线 + trace 缓冲区」，而 trace-cmd、kernelshark、早期的 execsnoop/opensnoop 等，都是不同形式的「ftrace 客户端」。  
>  
> 再往上一层，perf 的若干功能、BCC/bpftrace 中的部分工具以及更大的 tracing 框架，也都不同程度地建立在这套基础设施之上。


---

`execsnoop` 是一个建立在内核 tracing 基础设施之上的专用工具，而 `ftrace` 是这些基础设施里最早、最核心的一种。它们的关系可以概括为：

> ftrace 提供“内核事件追踪”的底层能力，  
> execsnoop 利用这些能力，专门用来追踪 exec() 调用。

---

## 1. ftrace 是什么

`ftrace`（Function Tracer）是 Linux 内核自带的 **通用内核追踪框架**，主要特点是：

- 支持在运行时跟踪内核函数调用、调度事件、中断等；
- 通过在函数入口插入“trace 桩”+ ring buffer 记录事件；
- 通过 `tracefs`/`debugfs` 暴露接口（通常在 `/sys/kernel/debug/tracing`）；
- 支持多种 tracer：`function`、`function_graph`、`sched`、`irqsoff`、`wakeup` 等。

它提供的是一种：**“在不改内核/不重启的前提下，动态观察内核行为”** 的能力。

---

## 2. execsnoop 是什么

`execsnoop` 是一个现成的 **“监控进程 exec() 行为”的工具**，典型输出包括：

- 进程名（PCOMM）  
- PID / PPID  
- 返回值（RET）  
- 完整命令行（ARGS）

用处是在系统中发现：

- 大量短命进程频繁被拉起（例如某应用频繁 exec 外部命令）；
- top/ps 看不到高 CPU 进程，但整体 CPU 很高（短命进程在刷）；
- 想要知道“到底是谁在不停地启动这些命令”。

---

## 3. 早期关系：execsnoop 基于 ftrace/kprobe 实现

早期版本的 `execsnoop` 是 **直接基于 ftrace / kprobe** 实现的：

- 利用 kprobe/ftrace 挂在内核的 `do_execve()` / `do_execveat()` 等函数上；
- 每当有进程 exec 新程序时，收集当前进程名、PID/PPID、参数等；
- 把这些事件推入 trace buffer，再由用户态脚本从 `trace_pipe` 读出并格式化。

也就是说：

> 早期的 execsnoop = “在 exec 相关内核函数上启用 ftrace/kprobe，并把事件打印成人类可读日志”。

本质上是 **针对 exec 事件场景化封装的 ftrace 客户端**。

---

## 4. 现在的关系：可用 ftrace，也可以用 eBPF/tracepoint

随着 eBPF 普及，现代的 execsnoop 实现更多是基于：

- `tracepoint:sched:sched_process_exec` + eBPF；
- BCC/bpftrace 提供的高级封装脚本。

但本质没变：

- 底层仍依赖 **内核的 tracing 机制**（tracepoint/kprobe/ftrace）；
- execsnoop 只是一个“专门盯 exec 的工具视图”。

---

## 5. 一句话总结

- **ftrace**：内核内建的通用 tracing 引擎，提供追踪函数/事件的能力；
- **execsnoop**：利用 ftrace/tracepoint/eBPF 等 tracing 能力，专门展示 exec() 事件的工具。

你可以把它们的关系理解为：

> ftrace / tracepoint / eBPF = 跟踪引擎  
> execsnoop = 基于这些引擎，专门监控 exec() 的一个“应用层工具”。



---

# ftrace 简介

`ftrace`（Function Tracer）是 Linux 内核自带的 **内核级跟踪与分析框架**，用来在运行时观察内核函数的调用行为、延迟和调度情况。

它的几个关键点：

## 1. 核心定位

ftrace 的核心作用是：在 **不改内核源码、不重启系统** 的前提下，动态地追踪内核内部行为。典型用途包括：

- 跟踪某个内核函数是否被调用、调用频率如何  
- 分析系统卡顿、延迟高时，是哪个路径/函数耗时严重  
- 观察调度、抢占、软中断等内核时序行为  
- 为 perf / eBPF 等更高层工具提供底层 trace 基础

## 2. 工作原理（概念层面）

- 内核在构建时编译进 ftrace 支持（`CONFIG_FUNCTION_TRACER` 等选项）。
- 对可被跟踪的函数插入一个“桩”（通常是 `mcount`/`_fentry_`），运行时可以启用或禁用。
- 当你通过接口开启某种 tracer 时，这些桩会把执行信息写入一个环形缓冲区（ring buffer）。
- 用户空间通过 `debugfs` 或 `tracefs`（一般挂载在 `/sys/kernel/debug/tracing`）读取这些 trace 记录。

这让内核可以做到“平时几乎无开销，有需要时打开详细跟踪”。

## 3. 常用能力与 tracer 类型

ftrace 本身支持多种 tracer 模式，例如：

- `function`：记录哪些函数被调用（函数级 trace）
- `function_graph`：记录函数调用的层级结构和耗时（调用图 + 延迟）
- `sched`：跟踪调度事件（任务切换等）
- `irqsoff` / `preemptoff`：追踪关中断 / 关抢占导致的延迟
- `wakeup`：追踪唤醒延迟

通过组合不同 tracer，可以定位：

- 谁把 CPU 占用太久（长临界区）
- 哪个内核路径导致调度延迟
- 哪些函数调用频率异常之高

## 4. 基本使用方式（概览）

ftrace 的典型接口在 `/sys/kernel/debug/tracing`（或 `/sys/kernel/tracing`），常见文件包括：

- `current_tracer`：选择当前 tracer 类型（如 `function`, `function_graph`）
- `set_ftrace_filter`：设置要跟踪的函数白名单
- `set_ftrace_notrace`：设置黑名单
- `trace` / `trace_pipe`：读取跟踪结果（`trace_pipe` 支持持续输出）
- `tracing_on`：开关 trace

例如（概念示意）：

```bash
# 选择函数调用图 tracer
echo function_graph > /sys/kernel/debug/tracing/current_tracer

# 只跟踪某个函数（例如 do_sys_open）
echo do_sys_open > /sys/kernel/debug/tracing/set_ftrace_filter

# 开始跟踪
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 读取 trace
cat /sys/kernel/debug/tracing/trace

# 停止跟踪
echo 0 > /sys/kernel/debug/tracing/tracing_on

---

# execsnoop 和 ftrace 的关系

## 1. 各自是什么

- **ftrace（Function Tracer）**  
  Linux 内核自带的通用 tracing 框架，可以在运行时跟踪：
  - 内核函数调用（function / function_graph）
  - 调度事件（sched）
  - 中断、关中断/关抢占导致的延迟等  
  提供的是底层能力：在指定 hook 上记录事件到 ring buffer，再通过 `/sys/kernel/debug/tracing` 读出来。

- **execsnoop**  
  一个“现成的工具脚本”，专门用来 **实时监控进程的 exec 行为**，输出：
  - 新进程名（`PCOMM`）
  - PID / PPID
  - 返回值（RET）
  - 完整命令行参数（ARGS）

  它解决的就是你在案例里遇到的那种短命进程问题：  
  “有大量短时进程在不停 exec，但 top、ps 一眼看不到是谁”。

## 2. 早期关系：execsnoop 基于 ftrace 实现

最早 Brendan Gregg 那批 `execsnoop` 是 **直接用 ftrace 的 function tracing / kprobe 能力** 做的：

- 在内核的 `do_execve()` / `do_execveat()` 等路径上挂 ftrace/kprobe；
- 每当有进程 exec 新程序时，收集进程信息和参数；
- 将事件写到 trace buffer，再由用户空间脚本读出，格式化成你看到的 `PCOMM PID PPID RET ARGS` 输出。

所以，从实现角度看，早期的 execsnoop 本质上就是：

> “在 exec 相关内核函数上开启 ftrace/kprobe → 把信息打印成人类可读的 exec 日志”

这就是两者最直接的关系：**execsnoop 是站在 ftrace 这类内核追踪基础设施之上的“应用层工具”**。

## 3. 现在的关系：可以基于 ftrace，也可以基于 eBPF

随着 eBPF 普及，很多 `execsnoop` 变体已经改为基于 eBPF / tracepoint：

- 有些版本用 `tracepoint:sched:sched_process_exec` + eBPF program 捕获 exec 事件；
- 还有 bcc/bpftrace 版 `execsnoop`，完全在 BPF 上实现，不再直接操纵 ftrace 文件接口。

但本质没变：

- ftrace、kprobe、tracepoint、eBPF 都是 **内核事件捕获机制**；
- execsnoop 是一个“选定了 exec 这个事件、做好解析和展示”的专用工具。

可以这么归纳今天的状态：

- **ftrace**：老牌通用 tracing 引擎，perf、部分 eBPF 工具都可以利用它；
- **execsnoop**：一个“用内核 tracing 机制（早期主要是 ftrace，现在多为 tracepoint+eBPF）监控 exec()”的工具。

## 4. 一句话总结

- ftrace 提供的是“**怎么追踪内核函数/事件**”的底层能力；
- execsnoop 则回答“**我想看所有 exec()，帮我把 PID/PPID/ARGS 打出来**”，是一个基于这些能力封装好的成品工具。

你在实战里可以把它们看成：  
ftrace / eBPF / tracepoint = 跟踪引擎  
execsnoop = 针对 exec 事件的一个“现成的视图”。


