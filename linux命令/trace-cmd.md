# trace-cmd 常用命令与用法速查

基于 ftrace 的 CLI 工具，核心思路：**用 trace-cmd 控制内核 tracing，再把结果保存为二进制文件，最后格式化查看/分析**。

---

## 1. 查看支持的 tracer 和 event

### 1.1 列出所有 tracer

trace-cmd list -t

常见 tracer 示例：
- function
- function_graph
- irqsoff
- preemptoff
- wakeup
- nop（空 tracer）

### 1.2 列出所有 event

trace-cmd list -e

按 subsystem 列：
- sched
- irq
- net
- block
- syscalls
- 等等

也可以只看某个 subsystem 的 events，例如：
- trace-cmd list -e sched

---

## 2. 查看当前 tracing 状态

### 2.1 基本统计信息

trace-cmd stat

查看：
- 当前 tracer
- 缓冲区大小
- 各 CPU 状态等

（等价于去翻 /sys/kernel/tracing 里的一堆文件的摘要版）

---

## 3. 在线启动 / 停止 tracing

### 3.1 启动 tracing

trace-cmd start [选项]

常见组合示例：

1. 启动一个 function tracer：
   - trace-cmd start -p function

2. 启动 function_graph tracer：
   - trace-cmd start -p function_graph

3. 只追踪某个进程：
   - trace-cmd start -p function -P <pid>

4. 只追踪某些函数：
   - trace-cmd start -p function -l <func1> -l <func2>

常用选项：
- -p：选择 tracer（对应 current_tracer）
- -P：PID 过滤（只 trace 指定进程）
- -l：function 过滤（只 trace 某些函数）
- -n：排除函数（negative filter）

### 3.2 停止 tracing

trace-cmd stop

停止往 ring buffer 写入，但数据还留在 buffer 中。

---

## 4. 显示 / 清理 trace 缓冲区

### 4.1 显示当前 trace（文本）

trace-cmd show

作用类似 cat /sys/kernel/tracing/trace，但格式更友好一些。

### 4.2 重置 / 清空 trace

trace-cmd reset

等价做的事情大致包括：
- 禁用 tracer（设置成 nop）
- 清空 ring buffer
- 清除 filter 等配置

是「回到干净初始状态」的快捷方式。

---

## 5. 记录到文件并离线分析

### 5.1 记录 trace 到文件

trace-cmd record [选项] -- <命令>

典型用法：

1. 记录整个系统某段时间：
   - trace-cmd record -p function

2. 只记录某个命令执行期间：
   - trace-cmd record -p function -- ./your_program

3. 限制 tracer、event、PID 等：
   - trace-cmd record -p function_graph -P <pid>
   - trace-cmd record -e sched:sched_switch
   - trace-cmd record -e net:netif_receive_skb

记录完会生成：
- trace.dat

### 5.2 把 trace.dat 转成可读文本

trace-cmd report

或者指定文件：
- trace-cmd report -i trace.dat

常配合 grep / less / awk 做进一步分析，例如：
- trace-cmd report | grep your_function

### 5.3 从内核缓冲区提取到 trace.dat

trace-cmd extract

场景：
- 你已经用 trace-cmd start / stop 控制过 tracing；
- 数据在内核 ring buffer 中；
- 想把当前缓冲区内容保存成 trace.dat，方便离线查看。

---

## 6. 常用过滤示例（Function / PID）

### 6.1 只追踪某个函数

trace-cmd start -p function -l tcp_v4_connect

或者 record 模式：

trace-cmd record -p function -l tcp_v4_connect -- ./client

### 6.2 排除噪音函数

trace-cmd start -p function -n schedule

表示追踪所有函数调用，但排除 schedule。

### 6.3 只追踪某个 PID

trace-cmd start -p function -P 12345

或者：

trace-cmd record -p function -P 12345 -- ./your_program

---

## 7. 与 ftrace 基本文件操作的对照关系

- current_tracer
  - trace-cmd start -p XXX（设置 tracer）
  - trace-cmd reset（恢复 nop）
- set_ftrace_filter / set_ftrace_notrace
  - trace-cmd start -l / -n（函数过滤）
- set_ftrace_pid
  - trace-cmd start -P（PID 过滤）
- trace / trace_pipe
  - trace-cmd show（一次性 dump）
  - trace-cmd record / report（结构化记录 + 离线分析）

---

## 8. 快速常用命令小抄

1. 查看可用 tracer：
   - trace-cmd list -t

2. 查看可用 events：
   - trace-cmd list -e

3. 看当前状态：
   - trace-cmd stat

4. 在线启停：
   - trace-cmd start -p function
   - trace-cmd stop
   - trace-cmd show
   - trace-cmd reset

5. 记录 + 离线分析：
   - trace-cmd record -p function -- ./app
   - trace-cmd report

6. 从当前内核缓冲区导出：
   - trace-cmd extract
