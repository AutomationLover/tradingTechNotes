# GDB 调试速记（概念 + 常用命令 + 心智模型）

## 1. 什么是 GDB？

1986 年诞生 → 90 年代随 GNU 工具链在 Unix/Linux 圈流行 → 2000 年代以后基本成为开源 C/C++ 世界的默认调试器。

GDB 是 Linux 下最常用、最强大的用户态调试器，用来“在程序运行时拆开来看”。你可以用它在 C/C++ 等语言写的可执行程序里：

- 运行程序并在任意位置“刹车”（断点）
- 单步执行，观察每一步的代码路径
- 查看和修改变量、内存、寄存器
- 查看函数调用栈，定位是在哪一层逻辑出错
- 程序崩溃后配合 core dump 复盘“事发现场”，不一定要现场重现 bug

在 Linux 用户态调试体系中，大致分工是：

- GDB：逻辑错误、崩溃分析、单步跟代码
- Valgrind：内存泄漏、非法访问等内存问题
- Strace：系统调用层面的问题（文件、网络、权限等）
- Ltrace：库函数调用链问题（例如 libc 调用）

可以把 GDB 当作用户态调试的“瑞士军刀”，其他工具是补充视角。

---

## 2. GDB 的心智模型

可以用“看片 + 看现场 + 改现场”来理解：

1. 看片：  
   让程序在 GDB 里跑，用断点把关键场景“截屏”，然后单步往前走，看逻辑是不是按预期执行。

2. 看现场：  
   当程序在断点处停下时，GDB 就像一个“时间冻结”的现场，你可以查看：
   - 当前在哪一行代码（frame）
   - 是怎么一路调用过来的（backtrace）
   - 关键变量是什么值（print）
   - 内存和寄存器状态（x, info registers）

3. 改现场：  
   如果想验证某个假设，直接在 GDB 里修改变量、寄存器甚至跳转指令，再继续跑，看系统行为是否按你的猜想改变。

4. 事后勘察（core dump）：  
   程序崩溃产生 core 文件时，你可以在事后用 GDB 打开 core 文件，相当于回到崩溃瞬间的现场做取证，而不必再次重现 bug。

---

## 3. GDB 使用流程（从 0 到能查 bug）

### 3.1 编译阶段

为了让 GDB 能看懂源码和变量名，需要加入调试信息：

- 使用 `-g` 选项编译
- 不要开太激进的优化（大量 `-O2/-O3` 会让单步执行和源码对应变得困难）

例如：

gcc main.c -g -o main

---

### 3.2 启动 GDB

常见几种方式：

- 直接调试可执行文件：

gdb ./main

- 带参数运行程序：

gdb --args ./main arg1 arg2

- 附加到已运行的进程：

gdb -p <PID>

- 分析 core dump：

gdb ./main core

---

## 4. 常用命令（按场景整理）

### 4.1 启动与运行控制

- 启动程序：

run
run arg1 arg2

- 重新运行（会重新开始）：

run

- 继续执行到下一个断点或程序结束：

continue
或者简写：c

- 单步“到下一行”（不进入函数）：

next
简写：n

- 单步“进入函数内部”：

step
简写：s

- 从当前函数一次性跑到返回：

finish

- 立刻终止被调试程序：

kill

---

### 4.2 断点（breakpoints）

- 在某函数入口处设置断点：

break main
break my_func

- 在某个源文件的具体行：

break main.c:42

- 条件断点（例如当 i == 100 时才停）：

break main.c:42 if i == 100

- 列出当前所有断点：

info break

- 删除指定断点：

delete 1

- 暂时禁用某个断点：

disable 1

- 重新启用某个断点：

enable 1

---

### 4.3 查看调用栈和切换栈帧

- 查看当前调用栈：

backtrace
简写：bt

- 查看完整调用栈（包括更多信息）：

backtrace full

- 在不同栈帧之间切换（例如 frame 0, frame 1）：

frame 0
frame 1

（通常 frame 0 是当前函数，数字越大越靠近调用者）

---

### 4.4 变量与表达式

- 打印变量、表达式：

print x
print x + y
print *ptr

- 连续打印多个：

display x
display y

- 不要自动打印了：

undisplay

- 修改变量值：

set variable x = 10
set x = 10

---

### 4.5 内存查看

- 按地址查看内存，格式：`x/数量格式单位 地址`

常见格式说明：
- `数量`：要显示多少个单元
- `格式`：x=十六进制，d=十进制，c=字符，s=字符串
- `单位`：b=字节，h=2字节，w=4字节，g=8字节

示例：

x/16xb ptr      # 从 ptr 开始，显示 16 个字节，以十六进制格式
x/4xw &array    # 从 &array 开始，显示 4 个 4-byte 单元

---

### 4.6 源码浏览

- 显示当前行附近的源码：

list

- 显示指定文件某一行附近源码：

list main.c:42

- 在调试中随时查看当前所在行：

where             # 等价于 bt
或者
info line         # 更多源码行信息

---

### 4.7 和 core dump 联动

当程序因崩溃产生 core 文件时：

- 打开 core 文件：

gdb ./main core

- 进入后最关键的两步：

1）看调用栈：

backtrace

2）切到 frame 0 或相关 frame 看变量：

frame 0
print x
print *ptr

这相当于对“崩溃瞬间”的现场做取证。

---

## 5. 调试时的实用套路（心智模型落地）

下面整理几种你实际遇到 bug 时可以直接套用的套路。

### 5.1 程序逻辑不符合预期（没崩溃）

1. 在怀疑的逻辑前后打断点。
2. `run` 启动，跑到断点停下。
3. `print` 看关键变量的值是不是预期。
4. 用 `next`/`step` 按关键路径单步走。
5. 如有分支，优先跟向下游影响最大的那条。

目标：把“哪里开始不符合预期”具体化到某一行代码和变量变化。

---

### 5.2 程序崩溃（segfault 等）

1. 确保 `ulimit -c unlimited` 打开 core dump。
2. 程序崩溃后，通过 core 文件和二进制启动 GDB：

gdb ./main core

3. 进 GDB 后：

backtrace        # 先看调用栈
frame 0          # 通常是崩溃点
list             # 看当前源码
print 相关变量   # 确认哪个指针/索引有问题

4. 如果栈比较深，向上切换 frame，看上游函数传进来的参数是否异常。

目标：从“崩溃了”精确缩小到“哪次调用，哪个参数、哪个指针导致了崩溃”。

---

### 5.3 复杂业务逻辑难以脑补

1. 在关键接口函数入口处打断点（例如 RPC handler、状态机入口）。
2. `run`，每次请求触发时都会停在入口断点。
3. 查看本次调用的入参（print 参数），随后 `finish` 跑完这一轮。
4. 对比多次调用的行为，观察数据和状态演变。

目标：把复杂业务逻辑拆成一次次可观察的“调用事件”。

---

## 6. 小结

- 把 GDB 当成“时间冻结的运行现场”，可以随时看现场、改现场。
- 核心就是：`-g` 编译 → `gdb` 启动 → 断点 → 单步 → 看栈、看变量、看内存。
- 与 core dump 结合，可以对“历史事故”做事后勘察，无需现场重现问题。


-------

# GDB Part 2（上半节）笔记

## 1. GDB 是什么，用来干什么？

- GDB = GNU Debugger，用来调试**已编译的可执行文件**。
- 支持多种语言：汇编、C、C++ 等。
- 主要能力：
  - 逐步执行（single-step）
  - 设断点 / 条件断点
  - 查看 / 修改变量和内存
  - 查看调用栈（backtrace）
  - 分析崩溃（尤其是 segfault）

心智模型：GDB 让你在程序运行中“停下来、看一眼、再继续”。

---

## 2. 为 GDB 编译：`-g` 调试信息

- 普通编译：

  gcc file.c -o prog

- 调试版编译：

  gcc -g file.c -o prog

- `-g` 会在可执行文件中加入 **符号表信息**：
  - 变量名、函数名
  - 类型信息
  - 文件名+行号

课程里对比了加/不加 `-g` 的二进制大小，可以看到加 `-g` 后可执行文件明显变大。  
如果忘记加 `-g`，在 GDB 里会看到：

> “no debugging symbols found”

这时候只能看汇编，很不方便。

---

## 3. 启动 GDB 的几种方式

1）直接带可执行文件：

- `gdb ./prog`  
- 或者安静模式（不打印 banner）：

  gdb -q ./prog

2）先进 GDB 再装载文件：

- `gdb`  
  然后在 gdb 提示符里：

  file ./prog

两种最终效果一样：GDB 加载这个可执行文件的符号信息。

---

## 4. 启动 / 退出程序：`run`、`continue`、`quit`

### 启动程序：`run` / `r`

- 在 GDB 里第一次启动程序要用：

  run

- 简写：

  r

- 再次 `run` 会**重新开始整个程序**，而不是从当前行继续。

### 继续执行：`continue` / `c`

- 当程序在断点处停下时，用：

  continue

- 简写：

  c

- 作用：从当前停点一直跑到：
  - 下一个断点，或
  - 程序结束

区别心智模型：

- `run`：从头重新跑一次。
- `continue`：从当前停住的地方往下跑，直到下一次停下。

### 退出 GDB：`quit` / `q`

- `quit` 或 `q`：退出调试器。

---

## 5. 断点：设、查、删

### 5.1 设置断点：`break` / `b`

两种常见方式：

- 按函数名：

  break main
  break factorial

- 按行号：

  break 12
  break 6

- 简写：

  b main
  b 12

断点的本质：告诉 GDB“程序跑到这里就停一下”。

### 5.2 查看断点：`info breakpoints`

- `info breakpoints`：列出所有断点：
  - 断点编号（1、2、3…）
  - 位置（文件:行号或函数）
  - 是否启用
  - 被命中次数

### 5.3 删除断点：`delete`

- 删除某个断点：

  delete 1

- 删除所有断点：

  delete

---

## 6. 查看源码：`list`

- `list`：默认显示当前点附近的 10 行。
- 指定起始行：

  list 1      # 从第 1 行开始
  list 4,8    # 显示第 4 到第 8 行

- 按函数名：

  list main

心智模型：`list` = 在 GDB 里“翻源码”。

---

## 7. 单步执行：`next` / `step` / `finish`

### 7.1 `next`（不进入函数内部）

- `next` / 简写 `n`：
  - 执行当前行，如果这一行调用了函数，只会“整行执行完”，停在下一行。
  - 不会进入被调用函数内部。

适合：不关心某个函数内部，只看当前函数的控制流。

### 7.2 `step`（进入函数内部）

- `step` / 简写 `s`：
  - 执行当前行，如果有函数调用，会**跳进函数内部**，停在被调函数第一行。
  - 对库函数（如 `printf`）若没有调试符号，会进入 glibc 代码，通常你会用 `finish` 跳出来。

适合：要细看某个函数内部的逻辑时使用。

### 7.3 `finish`（跑完当前函数）

- `finish`：
  - 一口气跑完**当前函数**，回到调用处。
  - 常用在：
    - 已经确认函数内部没有问题，只想快速回到调用点。
    - 从 `printf` 等库函数的内部跳回用户代码。

### 7.4 重复执行上一条命令：回车键

- 在 GDB 中按 Enter，会重复执行上一条命令。
  - 如果上一条是 `next`，多次回车就相当于不断单步执行。
  - 如果上一条是 `print loop`，回车会重复打印 `loop`。

---

## 8. 命令缩写与帮助：`help`

- 常用缩写：
  - `run` → `r`
  - `continue` → `c`
  - `break` → `b`
  - `next` → `n`
  - `step` → `s`
  - `quit` → `q`
- 获取帮助：

  help
  help run
  help break

可以在 GDB 里随时查命令说明和用法。

---

## 9. 带参数程序的调试：传递 `argv`

课程里举了有命令行参数的程序（比如控制 loop 次数），传参有三种方式：

1）在启动 GDB 时指定：

- `gdb --args ./prog 100`

  - `--args` 后面的都当作被调程序的参数。
  - 进 GDB 后直接 `run` 即可。

2）在 GDB 内，用 `run` 带参数：

- 启动 GDB：`gdb ./prog`
- 在 gdb 里：

  run 100

3）使用 `set args`：

- 在 gdb 里：

  set args 100 200
  run

心智模型：

- `--args`：一次搞定，最常用。
- `run <args>`：临时想改参数快用。
- `set args`：需要频繁 `run` 同一组参数时好用。

---

## 10. 用 GDB 调试 segfault：`backtrace` / `frame` / `print`

课程给了一个示例程序：`malloc` 超大内存（约 2GB，`1 << 31`），然后用 `fgets` 写入，导致 segfault。

调试步骤：

1）编译加 `-g`，在 GDB 里 `run` 程序，输入后触发 segfault。
2）GDB 输出：

   Program received signal SIGSEGV

3）查看调用栈：

   backtrace      # 简写 bt

   可以看到多层 frame，从 glibc 往上到 `main`。

4）切换到自己的 frame（比如 frame 3）：

   frame 3

5）`list` 看出错行：

   list 10

6）`print` 看关键变量：

   print buff

   发现指针是 0（NULL）。

7）回退到分配处：

   `buff` 来自 `malloc(1 << 31)`，理论上申请 2GB，`malloc` 失败时返回 NULL。

8）结论：`malloc` 失败返回 NULL，程序没做检查，随后对 NULL 写导致 segfault。  
   正确做法：检查 `malloc` 返回值，如为 NULL 则报错退出，不再传给 `fgets`。

心智模型：  
- segfault → `bt` 找栈 → `frame` 切到自己函数 → `list` 看行 → `print` 看指针 → 找根因。

---

## 11. 检查 / 修改变量和内存：`print` / `ptype` / `x`

### 11.1 `print`：看变量

- 基本用法：

  print i
  p i

- 可以打印表达式：

  print i + 1
  print *ptr
  print &i
  print sizeof(i)

- 改变格式：

  p/x i    # hex
  p/o i    # octal
  （没有直接的二进制格式，二进制用 `x` 来看）

### 11.2 `ptype`：看类型

- `ptype i`：显示 `i` 的类型（如 `int`）。
- `ptype &i`：显示 `int *`。
- `ptype main`：显示 `int main(void)`。

### 11.3 `x`：examine memory（按地址看内存）

- 语法：`x/<count><format><unit> <address>`
  - `count`：要显示的元素个数
  - `format`：`x`(hex)、`d`(decimal)、`c`(char)、`s`(string)
  - `unit`：`b`(1 byte)、`h`(2 bytes)、`w`(4 bytes)、`g`(8 bytes)

例子（对 `int i = 1337`）：

- `x/x &i`：以 4 字节单位、十六进制显示一个值。
- `x/4xb &i`：以字节为单位，显示 4 个字节的十六进制值，可以看 little-endian 布局。

对数组：

- 有 `int arr[3] = {1,2,3}`：
  - `p arr`：打印指针值（数组退化）。
  - `x/12xb arr`：查看 3 个整数的 12 字节内存布局。

### 11.4 打印字符串：`x/s`

- 对 `char *msg = "Hello World";`：
  - `x/s msg`：从 `msg` 指向的地址起，以 C 字符串方式打印，直到 `'\0'`。

---

## 12. 修改变量：`set`（简单提及）

课程示例：

- 在 GDB 内修改字符串：

  set msg = "Linux"

然后继续执行：

- `next`，`printf("%s\n", msg);` 会打印 `Linux` 而不是原来的字符串。

同样可以修改整数等基本类型。  
心智模型：`set` 让你在不重新编译的情况下“实验性修改现场”。

---

## 13. 小结：这一半节 GDB 内容的结构

本段视频核心在于建立一套 GDB 的基础使用“肌肉记忆”：

1. 编译：`gcc -g`，保证有符号。
2. 启动：`gdb -q ./prog`。
3. 断点：`b main` / `b <line>`，`info breakpoints`。
4. 运行：`run`（首次），之后用 `c` 继续。
5. 单步：`n` 不进函数，`s` 进函数，`finish` 跑完函数。
6. 崩溃分析：`bt` → `frame` → `list` → `print`。
7. 检查状态：`print` / `ptype` / `x`，必要时用 `set` 修改变量。
8. 命令缩写 + 回车重复上一命令，提高操作效率。

后续课程会在此基础上接上 coredump、Valgrind、strace/ltrace 等工具，形成完整的用户态调试工具链。


--------

# GDB Part 2（下半节）完整笔记

## 1. 调用栈与 frame（延续）

这一段开头先巩固了调用栈 / frame 的概念，并补充了几个配套命令：

- `backtrace` / `bt`：列出当前调用栈的所有 frame（0 是当前，越大越“上层调用者”）。
- `frame N`：切换到第 N 个 frame，再用 `list`、`info locals`、`print` 就是针对这个 frame。
- `info frame`：显示当前 frame 的底层信息：
  - 当前帧在栈上的地址
  - 当前指令指针（IP）和返回地址
  - 参数列表和局部变量在栈上的位置

配合使用方法类似：

1. `bt` 看出 `main -> func1 -> func2` 这样的调用链。
2. `frame 1` 切到 `func1`，`info locals` / `print n` 看局部变量。
3. `frame 0` 切回 `func2`，`print i` 看另一个函数的局部。

---

## 2. `info` 系列命令总结

课程在这里系统过了一遍 `info` 相关命令：

- `info functions`：所有函数（包含 C 库）。
- `info variables`：所有全局 / static 变量（所有源文件和库）。
- `info locals`：当前 frame 的局部变量。
- `info args`：当前 frame 的参数列表。
- `info breakpoints`：所有断点和 watchpoint。
- `info registers` / `info reg`：
  - 列出所有寄存器：名字、十六进制值、十进制值。
  - 如 `info reg rax`、`info reg rsp` 只看某一个。

心智模型：  
- “看函数上下文” → `info args` / `info locals`。  
- “看调用关系” → `bt` / `frame` / `info frame`。  
- “看 CPU 现场” → `info registers`。  

---

## 3. `register` 关键字与寄存器观察

讲师用一个小例子讲清楚 `register` 局部变量在 GDB 中的表现：

- 代码中：

  - `register int y;`
- 在 GDB 中：
  - `info locals` 能看到 `y` 这个变量。
  - 但 `print &y` 会失败：**register 变量没有可用的内存地址**（按 C 语义，它被放在寄存器，而不是内存）。
  - 通过 `info registers` 可以看到某个寄存器（如 `rbx`）的值随 `y` 的变化而变化。

概念上是正确的，实际现代编译器会自己决定寄存器分配，`register` 更多是“建议”，但 GDB 这个例子帮你建立直觉：  
“有些变量压根在寄存器里，没有稳定的内存地址可以取”。

---

## 4. 条件断点（完整）

在上半节已经提到条件断点，这一段用两个完整例子复习了两种写法：

1）已有断点，后加条件：

- 先设断点：

  break 27

- 查看断点编号：

  info breakpoints    # 比如该断点是 3 号

- 添加条件（只在 i == 5 时停）：

  condition 3 i == 5

2）一开始就带条件：

- 例如在行 11 上，只在 `i == 400` 时停：

  break 11 if i == 400

效果：for 循环跑到 `i == 400` 时才第一次停，前 0–399 次迭代不会干扰调试。

---

## 5. Watchpoint：监视变量读写

这部分是断点的“变量视角”版本：

- 普通断点：绑在“某一行代码”上。
- watchpoint：绑在“某个表达式/变量的值”上。

三种命令：

1. `watch expr`  
   - 当 `expr` 被**写入（值改变）** 时停下。
   - 典型例如：`watch x` → 只要 `x` 被修改，就会显示 old value / new value 并停下。

2. `rwatch expr`  
   - 当 `expr` 被**读** 时停下。
   - 常用在追查“谁在读这个危险指针 / 密钥”等。

3. `awatch expr`  
   - 当 `expr` 被**读或写**时都停下。

注意点：

- 表达式必须在当前作用域里（比如 watch x 之前必须先 `run` 到有 `x` 的时候，否则 “no symbol x in current context”）。
- watchpoint 也出现在 `info breakpoints` 里，只是 type 不同。
- 单步 `next` 时 watchpoint 的提示可能不明显；通常在 `continue` 下更有用，它会打印 old/new 值，比如：
  - `Old value = 0`
  - `New value = 10`

启用 / 禁用：

- 禁用 watchpoint（编号 2）：

  disable 2

- 重新启用：

  enable 2

---

## 6. TUI 模式：GDB 内看源码界面

讲师介绍了 GDB 自带的文本 UI 模式（类似简易 IDE）：

- 启动时直接用：

  gdb -tui ./prog

- 或在 GDB 内切换：

  Ctrl+X, A        # 开启/关闭 TUI 模式

TUI 模式特点：

- 终端上方展示源码，当前执行行高亮。
- 下方是 GDB 命令行。
- 可以正常使用 `break`、`next`、`step`、`continue` 等。
- 若显示乱掉，用：

  Ctrl+L          # 重新 repaint 屏幕

适合：在纯终端环境下，也能直观看到“当前停在哪一行”。

---

## 7. 日志记录：`set logging`

想把 GDB 交互过程保存到文件里，方便回顾 / 分享时，可以用 logging：

- 开启日志：

  set logging on

  - 默认输出到当前目录的 `gdb.txt`（注意课程里老师打错命令曾覆盖源码，这是个坑）。

- 关闭日志：

  set logging off

- 改变日志文件名：

  set logging file my-gdb.log

之后你在 GDB 里的命令和输出就会被追加到这个文件里。

---

## 8. 附加到已经在跑的进程：`attach` / `detach`

场景：某个程序已经在后台跑了（比如一个循环里 `sleep` 的小服务），想在不重启它的情况下调试一下。

步骤：

1. 在 shell 里启动程序，例如：

   ./prog &    # 后台跑

2. 在 GDB 里查 PID，可以用：

   (gdb) shell ps -f | grep prog

3. 用 PID attach：

   attach <PID>

   - 需要 root 权限时，要 `sudo gdb`。

4. attach 后进程会停住，这时可以像普通调试一样：
   - `bt`、`frame`、`info locals`、`break` 等。
   - 甚至可以设条件断点：例如“当 i == 40 时停下”。

5. 调试结束，用：

   detach

   - 进程继续运行，GDB 和它断开。

心智模型：`attach` 用于“临时接管一个已经运行的进程”。

---

## 9. 看汇编：`disassemble`

要在 GDB 中看某个函数的汇编，可以用：

- 针对当前函数：

  disassemble

- 或指定函数名：

  disassemble main

输出包括每条指令地址、opcode 和汇编语句。  
配合单步：

- 先 `break main` → `run` 停在 main。
- `disassemble main` 看整体。
- `next` / `step` 单步后再 `disassemble main` 看指令执行到哪。

适合：

- 做体系结构 / 性能分析，验证编译器生成的代码。
- 调试优化、内联、奇怪 crash 等 low-level 问题。

---

## 10. `start` 命令：`break main` + `run` 的快捷方式

之前习惯的启动姿势是：

- `gdb ./prog`
- `break main`
- `run`

课程介绍了一个简化命令：

- `start`

等价于：

- 在 `main` 第一行自动设置一个断点，并运行程序直到这里停下。

之后你可以直接 `next` / `step` 漫游 main，而不必手动设置 main 的断点。

---

## 11. `command`：给断点挂一组自动执行的 GDB 命令

场景：每次 hit 到某个断点时，都想自动打印一些变量，而不是每次手动打 `print`。

用法：

1. 设置断点，例如：

   break 14

2. 绑定命令：

   command 1     # 1 是断点编号
   print loop
   end

3. 之后每次断点 1 被命中时，GDB 会自动执行 `print loop`（可以多条命令）。

可以配合条件断点和 logging 做“调试脚本”。

---

## 12. 多源文件时的断点写法（Q&A 补充）

最后的问答中讲师补充了一点多文件项目的写法：

- 当工程有多个 `.c` / `.cpp` 文件时，可以明确指定文件名：

  - 按行号下断：

    break file.c:42

  - 按函数名：

    break file.c:my_func

避免在重名函数 / 多个翻译单元中模糊匹配。

---

## 13. GDB 全系列要点到这里基本齐活

结合 GDB Part 1 + Part 2，GDB 这一块的整体能力矩阵大概是：

- 基础：
  - `-g` 编译、`gdb -q ./prog`
  - `run` / `continue` / `quit`
- 断点与单步：
  - `break` / `info breakpoints` / `delete`
  - `next` / `step` / `finish`
  - `start` 作为快捷启动
- 调用栈与 frame：
  - `bt` / `frame` / `info frame`
- 检查状态：
  - `print` / `ptype` / `x` / `info locals` / `info args`
  - `info registers`
- 条件与变量驱动：
  - 条件断点：`break ... if expr` / `condition N expr`
  - watchpoint：`watch` / `rwatch` / `awatch`
  - `command N ... end` 在断点自动执行一组命令
- 进阶：
  - TUI：`gdb -tui` 或 Ctrl+X,A
  - 日志：`set logging on/off`, `set logging file`
  - attach：`attach PID` / `detach`
  - 汇编：`disassemble`

这些构成了用户态 C 程序调试时几乎所有常用手段，为后面的 coredump / valgrind / strace / ltrace 铺好了基础。

