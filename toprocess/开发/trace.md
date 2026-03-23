# STAR 示例：2005 年左右的嵌入式 Linux 驱动开发 & 板子 Bring‑up 案例

## S（Situation：背景）

2005 年，我在一家做嵌入式通信设备的公司负责一块新 ARM9 SoC 板子的 Linux 移植和驱动开发。  
硬件刚从板厂回来，内核版本是 2.6.x，交叉工具链比较原始，没有成熟的 ftrace/perf 工具，主要依赖：

- 自己维护的 kernel patch
- 大量 `printk` 埋点
- 简单的自定义 trace 宏

项目关键里程碑是：在预定展会前完成板子 bring‑up（串口、网卡、Flash）并跑起来 demo 应用。  
但是在 bring‑up 过程中，系统启动经常“卡死”，串口无输出，偶尔能看到部分 init 信息，很难复现和定位。

---

## T（Task：任务）

我负责的任务是：

- 在没有成熟调试工具（无 JTAG、无 gdbstub、perf/ftrace 还不成熟）的条件下，
- 通过内核追踪“埋桩”的方式：
  - 找出系统启动卡死的具体阶段（bootloader → 内核 early init → 设备初始化）；
  - 确认是否是中断控制器/串口/定时器等底层驱动初始化异常；
  - 在不影响发布进度的前提下，给出稳定可复现的解决方案。

目标是：在两周内让系统稳定启动到 shell，demo 应用能稳定运行至少 24 小时不崩溃。

---

## A（Action：行动）

### 1. 细化“怀疑路径”，设计最小化追踪方案

结合硬件原理图和内核启动流程，我先圈定几个关键阶段：

- early printk / setup_console
- 时钟和中断控制器初始化
- 串口驱动 probe 和中断注册
- scheduler tick / timer 中断工作情况

考虑到 CPU 性能和串口带宽有限，我没有一股脑在所有路径 `printk`，而是设计了一套“分阶段打开”的埋桩方案：

- 使用自定义宏：

  - `BOOT_TRACE("stage_xxx: %d\n", step);`
  - 宏内部可以开关，使得可以通过内核命令行或编译开关控制是否启用。

- 把埋点放在：
  - `start_kernel()` 中几个关键 initcall 之前/之后；
  - 串口驱动的 `probe()`、`irq handler` 入口；
  - 定时器中断/调度相关路径的入口（只放少量计数和时间戳）。

这样既有足够信息，又尽量控制对时序的扰动。

### 2. 利用内核日志作为“事件源”，构造简易时间线

因为没有 ftrace、perf 等成熟框架，我把 `printk` 当成最原始的 event source：

- 给关键 `BOOT_TRACE` 日志打上单调递增序号和时间戳（jiffies）：
  - 例如：`BOOT[0012][123456] init_irq done`
- 每次启动失败后，通过串口采集完整 `dmesg` 输出（若能进入 shell），或者在 bootloader 支持的情况下，将 log 缓存到内存特定区域供重启后读取。

通过反复启动、比对多次日志，我构建了一条“典型启动路径时间线”，发现卡死前最后一条日志往往停在：

- 串口驱动中断启用后；
- 但 scheduler tick 相关的日志没有正常出现。

这暗示：中断子系统/串口中断配置存在问题，可能导致中断风暴或死锁。

### 3. 定位中断问题：精简版“中断追踪”

在确定问题集中在中断阶段后，我进一步细化埋桩：

- 在中断控制器初始化函数中：
  - 记录哪些中断被 enable/disable；
  - 打印相关寄存器配置（mask、priority）。
- 在通用中断入口（例如 `do_IRQ()` 或 SoC 相关中断入口）加轻量级计数：
  - 每种中断类型计数加一，超过一定次数打印一次统计；
  - 防止频繁 `printk` 直接拖死系统。

为了减少对时序的影响，这些埋点尽量只修改内存计数，不立即打印，只有在检测到“某中断计数异常高”时才输出统计信息。

通过这套简易“中断 trace”，在一次成功启动、但几秒后死机的场景中抓到了日志：

- 某个串口 RX 中断在短时间内触发了数量级远高于预期的次数；
- 同时收不到其他正常的 tick 中断。

结合硬件工程师的反馈，我们怀疑：

- 串口中断触发条件配置不当；
- 或者中断清除时机错误，导致中断一直 pending，系统陷入忙于处理中断的状态。

### 4. 修改驱动 & 复用埋桩验证修复效果

针对上述怀疑，我对串口驱动做了两项修改：

- 调整中断触发条件（只在接收到完整字符/缓冲区一定深度时触发，而不是电平触发持续拉低）；
- 在中断 handler 中，明确按照芯片手册要求的顺序读/写寄存器清中断，而不是只读数据寄存器。

修改后我没有立即清理埋桩，而是保留关键 trace 宏，做了两轮验证：

- 连续冷启动数十次，确认日志中的启动阶段序列稳定、无卡死；
- 在运行 demo 应用、长时间压测串口收发时：
  - 中断计数在合理范围；
  - scheduler tick 和其他关键中断正常。

之后才逐步把冗余的 `printk` 删掉，只保留少量可以通过配置开关打开的 trace 宏，作为将来现场调试的预留手段。

---

## R（Result：结果）

通过这次“基于内核追踪埋桩”的排查：

- 在两周内定位并修复了串口中断配置/清除逻辑问题；
- 板子能稳定地从 bootloader 启动到 Linux shell，demo 应用在实验室连续稳定运行 72 小时无崩溃；
- 项目按时完成展会演示需求。

更重要的是，这次经历沉淀了一套在 **资源受限、工具链落后** 环境下的调试方法论：

- 把 `printk` + 自定义 trace 宏当成系统性的“事件源”，而不是零散打印；
- 用“最小化埋桩 + 构造时间线 + 逐步缩小范围”的方式代替无脑加日志；
- 在驱动和板级代码中预留可开关的 trace 能力，为后续现场问题快速诊断打基础。

如果放在今天看，这套方法其实就是早期版的“靠事件源和追踪框架做 root cause 分析”，只不过当年还没有现成的 ftrace/perf/eBPF，很多事情需要我们自己在内核里“搭一套简化的追踪基础设施”来完成。


- “宏内部可以开关”：

 ▫ 编译期：通过 ‎`CONFIG_BOOT_TRACE_DEBUG` 开 / 关整套功能。

 ▫ 运行期：通过内核命令行 ‎`boot_trace=1` 控制是否真的 ‎`printk`。


- 编译期开关 → 决定“有没有这段调试逻辑”。

- 运行期开关 → 决定“这段调试逻辑此刻是否生效”。



 
定义TRACE宏
```c
/* 1. 编译期开关：在 Kconfig 或 Makefile 里定义 CONFIG_BOOT_TRACE_DEBUG
 *    没定义时，BOOT_TRACE 直接变成空，完全不产生代码
 */
#ifdef CONFIG_BOOT_TRACE_DEBUG

/* 2. 运行期开关：用一个全局变量控制是否打印 */
static int boot_trace_enabled = 0;

/* 3. 内核参数：boot_trace=1 打开，boot_trace=0 关闭 */
static int __init boot_trace_param(char *str)
{
    /* 不关心具体值，只要给了就打开；
     * 如果你想严格解析 0/1，可以用 kstrtoint
     */
    boot_trace_enabled = 1;
    return 0;
}
early_param("boot_trace", boot_trace_param);

/* 4. 真正的宏：只有在编译期开启 + 运行期开启时才 printk */
#define BOOT_TRACE(fmt, ...)                                      \
    do {                                                          \
        if (boot_trace_enabled)                                   \
            printk(KERN_INFO "BOOT_TRACE: " fmt, ##__VA_ARGS__);  \
    } while (0)

#else  /* !CONFIG_BOOT_TRACE_DEBUG */

/* 5. 未开启调试时，宏直接是空，调用处不产生任何代码 */
#define BOOT_TRACE(fmt, ...)  do { } while (0)

#endif /* CONFIG_BOOT_TRACE_DEBUG */

```

使用
```c
void __init my_board_init(void)
{
    int step = 0;

    BOOT_TRACE("stage_%d: start board init\n", step++);

    init_clocks();
    BOOT_TRACE("stage_%d: clocks inited\n", step++);

    init_irqs();
    BOOT_TRACE("stage_%d: irqs inited\n", step++);

    init_serial();
    BOOT_TRACE("stage_%d: serial inited\n", step++);
}

```


