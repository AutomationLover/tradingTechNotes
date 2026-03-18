# Strace & Ltrace（课程第四节）笔记

https://www.udemy.com/course/learn-linux-user-space-debugging/learn/lecture/16684908#overview

## 1. Strace：系统调用级调试器

- **Strace = system call tracer**：跟踪进程发起的所有系统调用和收到的信号。
- 输出每一行都包含：
  - 系统调用名
  - 参数（已格式化）
  - 返回值（错误时附带 errno 对应的错误字符串）

心智模型：Strace 让你看到“用户态程序和内核之间的对话”。

---

## 2. Strace 的两种使用方式

1）直接用 Strace 启动程序：

strace ./prog arg1 arg2

- 程序在 Strace 之下运行，所有系统调用按顺序打印出来。
- Strace 输出默认写到 **stderr**，重定向要用 `2>`，例如：

strace ./prog 2&gt;trace.log

2）附加到已在运行的进程：

strace -p &lt;PID&gt;
strace -p &lt;PID&gt; -f   # 同时跟踪子进程

- 常用于线上/测试环境排查“已经在跑”的服务。
- Ctrl-C：只会停止 Strace，**不会杀掉被跟踪进程**（和“strace 直接启动进程”的情况相反）。

---

## 3. Strace 的典型用途

适合这些场景：

- 没有源码：只能从行为推断，Strace 让你看所有 system call。
- 程序表现异常，但：
  - 日志没有输出
  - 没有有用的 stdout/stderr
  - 端口不监听或不响应
- 安装器/脚本在某台机器上随机崩溃：看最后访问的文件/库。
- 想知道程序：
  - 打开了哪些配置文件/共享库
  - 哪个系统调用一直在卡（I/O、锁、网络等）

和 GDB 对比：  
GDB 看的是“代码 + 变量”；Strace 看的是“系统调用 + 资源访问”。

---

## 4. 从简单示例理解 Strace 输出

### 4.1 空程序为何也有一堆系统调用？

示例：`int main() { return 0; }`

strace ./a.out

你会看到：

- `execve()` 启动进程
- `brk()`, `mmap()` 等用于内存管理
- `open()` / `access()` 加载 C 标准库和动态链接器
- 如果是动态链接：大量 `open`, `mmap`, `close` 用来装载 `.so` 文件

这都是 **运行时启动代码 + glibc** 做的事情，不是你写的业务逻辑。

### 4.2 静态链接 vs 动态链接

- 默认：`gcc` 生成动态可执行文件，需要加载 glibc 等，所以 Strace 输出系统调用很多。
- 若用静态链接（讲师用的是 `-static` 思路）：加载动态库的系统调用明显减少。

心智模型：动态链接 = 运行前先“把库搬进来”；静态链接 = 全部打包，系统调用数量少。

### 4.3 `printf` 如何映射到系统调用

在有 `printf("Hello\n");` 的程序上 Strace：

- 可以看到：
  - `write(1, "Hello\n", 6) = 6`  
- 说明：`printf` 最终调用的是 `write` 系统调用，`1` 是 stdout 的 fd。

---

## 5. 常用选项与进阶用法

### 5.1 输出重定向和日志

- 默认输出到 stderr。
- 写入文件：

strace -o trace.log ./prog

- 或：

strace ./prog 2&gt;trace.log

便于后续用 `vimdiff` 等工具进行比较（讲师做了空程序 vs 带 printf 程序的 diff）。

### 5.2 信号和子进程

- Strace 会同时显示进程收到的信号，比如 `SIGALRM`、`SIGINT` 等。
- 对有 `fork()` 的程序，默认只跟踪父进程。
- 若希望同时跟踪子进程系统调用：

strace -f ./prog_fork

- 或 attach 时：

strace -p &lt;PID&gt; -f

- 使用 `-ff` 时，Strace 会为每个进程单独生成一个日志文件（典型命名后缀带 PID），便于区分父子。

### 5.3 统计模式（性能视角）

使用 `-c` 选项：

strace -c ./prog

- 输出总结表，包括每种 system call：
  - 调用次数
  - 总耗时
  - 平均耗时
  - 错误次数

可通过 `-c` 快速找出瓶颈 system call，比如大量 slow I/O。

`-c` 也可以和普通输出结合（`-c` + 加其他选项），既看明细又看统计。

### 5.4 时间戳和耗时

- `-t`：在每行前显示时间（秒级）。
- `-tt`：增加微秒级精度。
- `-ttt`：打印自 Epoch（1970）以来的秒数 + 微秒，用作“绝对时间戳”。

示例：

strace -tt ./prog

可以直观看每个系统调用发生时间间隔。

- `-T`：显示每个系统调用的耗时（系统调用执行时间）。
- 可以组合使用：

strace -tt -T ./prog

心智模型：`-t` 系列看“什么时候调用”，`-T` 看“每次 system call 花了多久”。

### 5.5 只跟踪感兴趣的系统调用（过滤）

通过 `-e trace=` 过滤：

- 只看打开/关闭文件相关调用：

strace -e trace=open,close ./prog

- 只看读写文件：

strace -e trace=read,write ./prog

- 排除某些调用：

strace -e trace=\!open,\!close,\!read,\!write ./prog

（注意需要引号和转义，有关 `!` 要放在 shell 能接受的引号里。）

- 使用 **系统调用组**：

strace -e trace=file ./prog     # 所有文件相关系统调用  
strace -e trace=memory ./prog   # 内存相关（mmap/brk 等）  
strace -e trace=signal ./prog   # 信号相关 system call  
strace -e trace=ipc,network ./prog  # IPC 或网络相关

这对大程序非常重要，可以避免巨量“噪声”。

### 5.6 附加到脚本和长期运行进程

- 对 shell 脚本：

strace ./script.sh

Strace 会显示脚本中调用的可执行程序（echo、cp 等）的系统调用，而不是 shell 内建细节。

- 对长期运行的进程（如 `while` + `sleep`）：

  - 先用 `&` 放到后台，再 `strace -p PID`。
  - 输出可见 `write`（对应 `printf`）和 `nanosleep`（对应 `sleep`）。
  - Ctrl-C 只停 Strace，不停进程；需要 `fg` 再 Ctrl-C 才结束进程（讲师现场演示）。

### 5.7 字符串展示长度

默认 Strace 对字符串参数只打印前 32~40 个字符。  
使用 `-s` 可扩大显示长度，例如：

strace -s 80 ./prog

可完整看到长字符串内容，比如长日志行或结构序列化内容。

---

## 6. ltrace：库调用级调试器

- **ltrace = library call tracer**：跟踪的是 **库函数调用**，而不是内核系统调用。
- 典型用途：
  - 查看程序使用了哪些 glibc 函数（`malloc`, `printf`, `fopen` 等）。
  - 查找某个崩溃发生在什么库函数之后（比如 `strcpy` 越界）。
  - 调试自己的共享库：是否被调用、参数如何、返回值如何。

心智模型：  
- Strace：程序和内核之间的 syscall 流。  
- Ltrace：程序和库之间的函数调用流。

### 6.1 基本用法

和 Strace 类似：

ltrace ./prog

常见输出：

- `malloc(…) = 0x...`
- `fopen("test", "wb") = 0x...`
- `fwrite(...) = 100`
- `fclose(0x...) = 0`

讲师例子中用 `ltrace` 看到：`printf`、`fopen`、`fwrite`、`fclose` 等 glibc 调用顺序。

也支持：

- `-p PID`：附加到运行中的进程。
- `-c`：统计模式，汇总调用次数、耗时等。
- 某些 `-e` 类似过滤（只看特定函数），与 Strace 思路接近。

### 6.2 ltrace 在定位崩溃中的作用

示例：代码里错误地做了 `p+1` 后再 `free(p+1)`，导致 `free` 崩溃。

用 ltrace：

ltrace ./prog

可看到在调用 `free` 时传入的是错误地址，且崩溃就发生在 `free()` 调用位置，从而锁定 bug 在“库调用层”。

---

## 7. Strace vs Ltrace：分工一览

- Strace：
  - 关注 syscall 层（文件、网络、进程、内存、信号等）。
  - 不显示库函数名，只显示最终进入内核的 system call。
  - 适用于：无源码、部署故障、权限/路径问题、I/O 阻塞、网络/文件访问等。

- Ltrace：
  - 关注库函数调用（glibc 及自定义共享库）。
  - 看应用调用了哪些库函数、用的什么参数。
  - 适用于：库调用顺序、调用参数错误、崩溃位于库函数内部等。

两者可以结合使用：  
先用 Strace 看 “syscall 层是否正常”，再用 Ltrace 看 “库调用层是否合理”。

---

## 8. 本节课的心智模型

这一节实际在补齐 GDB 之外的“黑盒调试工具”：

- GDB：有源码、有调试信息时的“白盒单步调试”。
- Strace：不知道内部实现、只看 syscalls+signals 的“黑盒行为分析”。
- Ltrace：不知道内部实现、只看库调用的“黑盒 API 级分析”。

在没有源码或不方便重编译的场景下，Strace/Ltrace 提供一条 **零侵入** 的调试路径，让你通过“观察程序与系统/库的对话”来推断问题所在。
