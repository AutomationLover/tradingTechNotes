# Coredump & Valgrind（课程第三节）笔记

https://www.udemy.com/course/learn-linux-user-space-debugging/learn/lecture/16684906#overview 

## 1. Core dump：概念与用途

- **Core dump 定义**：程序崩溃时，由内核生成的一份“内存快照”文件。内容包括：
  - 代码段、数据段
  - 堆（heap）、栈（stack）
  - 所有 CPU 寄存器

- 主要用途：
  - 事后分析崩溃原因，不需要当场复现 bug
  - 嵌入式场景：目标板资源有限，可把 core 拷到 host 机上用 GDB 分析

心智模型：core = 某一时刻整个进程地址空间 + 寄存器状态的“冻结照片”。

---

## 2. 如何生成 core dump

1. 默认很多系统不生成 core，因为进程的 core 文件大小限制为 0。
2. 查看进程资源限制（包括 core 文件大小）：

   ulimit -a

   如果 `core file size` 为 0，则不会生成 core。

3. 启用 core 生成：

   ulimit -c unlimited

4. 再次运行有 bug 的程序（例如对 NULL 解引用写入），程序崩溃后当前目录会出现 `core` 文件。

---

## 3. 用 GDB 分析 core dump（崩溃后调试）

命令格式：

gdb ./binary core

进入 GDB 后：

- 直接打印出崩溃的信号类型和出错的源代码行号  
- 然后可以照常使用 GDB 命令：
  - `info locals`：查看局部变量（比如看到指针为 NULL）
  - `info registers`：看寄存器
  - `backtrace` / `bt`：看调用栈，切换 frame 分析

心智模型：GDB + core = 回到崩溃瞬间做“事故现场勘察”。

---

## 4. 运行中程序如何生成 core

### 4.1 使用 GDB attach 生成 core（不中断或仅短暂停顿进程）

场景：程序长时间运行（sleep / 守护进程等），需要某一时刻的现场快照。

步骤：

1. 程序正常运行，先通过 ps 等获取 PID。
2. 用 GDB 附着：

   gdb -p &lt;PID&gt;

3. 在 GDB 里执行：

   generate-core-file &lt;可选文件名&gt;
   detach
   quit

结果：进程继续运行（仅调试期间短暂停顿），磁盘上生成一个 core 文件，例如 `core.&lt;PID&gt;`。  
之后用：

gdb ./binary core.&lt;PID&gt;

查看当时的栈、局部变量等。

### 4.2 用信号 kill -SIGABRT 生成 core（会终止进程）

场景：程序 hang 住、占着 CPU 或一直在 ps 里，但看起来“卡死”。

步骤：

1. 找到 PID，清理目录中旧 core（方便区分）。
2. 发送 abort 信号：

   kill -SIGABRT &lt;PID&gt;

3. 进程收到 SIGABRT 后崩溃退出，内核生成 core 文件。
4. 用 GDB 打开 core，确认它卡在什么位置、哪个函数的哪一行。

### 4.3 程序内部调用 abort() 生成 core

在代码里直接调用：

abort();

效果类似 SIGABRT：程序异常终止并生成 core，这相当于代码中内置“一键 dump”。

---

## 5. C 常见内存问题分类

课程列出了典型内存错误类型：

1. 未初始化变量
   - 局部变量默认是垃圾值，行为不可预测（有时真有时假）。
2. 越界访问（buffer overflow / underflow）
   - 写越界：向数组 / malloc buffer 的末尾之后写数据。
   - 写下溢：向开始位置之前写数据。
   - 读越界 / 读下溢：读取越界的内容，典型如 `%s` 打印没有 `'\0'` 的数组，读到别的变量。
3. use-after-free（释放后继续使用指针）
   - 又称“悬空指针”（dangling pointer）。
4. use-after-return
   - 返回局部变量地址，函数退出后继续使用。
5. double free
   - 同一块内存多次 free。
6. memory leak
   - malloc 后没有 free，内存“丢失”：一次性泄漏几 KB 可以忽略，但在循环里持续泄漏，很快吃光内存。

补充：  
- 示例中用 `free` 命令和 `free | grep Mem` 比较前后 used memory，验证持续泄漏时系统内存使用确实上升。

---

## 6. GCC 与 Valgrind 的分工

- GCC（尤其加 `-Wall`）可以在编译期发现部分问题：
  - 未初始化局部变量的使用
  - use-after-return（返回局部变量地址）

- 但 GCC 无法检测很多运行期错误，例如：
  - 堆上越界访问（读/写）
  - double free
  - use-after-free
  - 动态内存泄漏

所以需要运行期工具（Valgrind 等）来补充。

---

## 7. Valgrind / Memcheck 使用与能力

Valgrind 定位：

- 是一个“调试 / 分析工具包装器”，内部包含多个工具：
  - 其中 **Memcheck** 专门用于内存错误检测（泄漏、越界、UAF 等）。

使用步骤：

1. 编译时包含调试信息：

   gcc -g your.c -o your_prog

2. 用 Valgrind 运行程序（选用 Memcheck 工具）：

   valgrind --tool=memcheck --leak-check=yes ./your_prog

示例中演示了 Memcheck 检测能力：

- 内存泄漏（definitely lost / possibly lost），并给出 malloc 的行号。
- 写越界 / 读越界（栈/堆都可以），提示 invalid read/write，指出：
  - 哪一行做了越界访问
  - 哪一行做的分配
- 未初始化变量使用：
  - 提示使用未初始化值发生在哪一行
  - 可以用 `--track-origins=yes` 追溯这个未初始化值最初来自哪里。
- double free：
  - 提示 invalid free，指出 malloc/free 次数不匹配。
- use-after-free：
  - 当释放后继续写/读，提示 invalid write/read of size N，并指出 free 的位置。

---

## 8. Valgrind 的优缺点

优点：

- 几乎覆盖常见动态内存错误（越界、UAF、double free、leak、未初始化使用等）。
- 报告中附带栈信息和行号，定位非常方便。

缺点：

- 性能开销大：程序在 Valgrind 下会变慢约 10–30 倍。
- 内存开销大：Valgrind 会以自己的 malloc 替换系统 malloc，用额外结构记录元信息。

定位：  
适合开发 / 测试环境做内存体检，不适合在线上常驻。

---

## 9. 这一节的整体结构心智模型

本节内容在搭一个“Linux 用户态内存与崩溃调试工具链”的框架：

- **core dump + GDB**：用来做崩溃后的现场还原和运行中 snapshot。
- **内存 bug 分类**：未初始化、越界、UAF、use-after-return、double free、leak 等，明确“都有例子、都要小心”。
- **GCC**：编译期第一道防线。
- **Valgrind / Memcheck**：运行期深度检测动态内存问题。
- 为后续 Electric Fence、strace/ltrace 等工具打基础，形成一个完整的“用户态调试工具箱”。
