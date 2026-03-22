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
