# dmesg 视频总结（14. Dmesg in deep）

## 1. dmesg 的作用与内核 ring buffer

Linux 内核在启动早期还没有根文件系统，不能把日志写到普通文件中，于是会把日志（例如 printk 输出）写入一个 ring buffer（环形缓冲区）。等系统用户空间起来后，像 systemd-journald 等日志进程会定期读取这个 ring buffer，把内容写入 `/var/log/...` 等日志文件（不同发行版路径不同）。

dmesg 命令的作用就是 读取 / 控制 内核的这个 ring buffer：

- 默认行为：打印当前 ring buffer 的内容到标准输出（stdout）。
- 还可以清空 ring buffer、去掉时间戳、按 log level 过滤、转换为可读时间等。

如果没有 ring buffer，那么在文件系统可用之前的所有内核日志都会丢失；借助 ring buffer，可以在系统启动之后通过 dmesg 把这些日志取出来。

---

## 2. dmesg 的常用选项（视频中讲到的）

下面所有“代码/命令”都用 txt 形式表示（不是嵌套的 markdown 代码块），方便你在 Obsidian 里自由复制。

### 2.1 默认输出：打印全部 ring buffer

dmesg

### 2.2 打印并清空 ring buffer（需要 root，-c）

小写 -c：先打印再清空：

dmesg -c

行为：

- 把当前内核 ring buffer 的内容打印到终端；
- 然后清空 ring buffer。

清空后再次执行：

dmesg

几乎看不到内容，直到系统有新的内核日志产生。

### 2.3 只清空不打印（需要 root，-C）

大写 -C：只清空，不打印任何内容：

dmesg -C

行为：

- 不打印当前任何日志；
- 直接把 ring buffer 清空。

清空后：

dmesg

输出为空或很少，直到有新日志。

### 2.4 时间戳显示与人类可读时间（-T）

内核在 ring buffer 里保存的是相对时间戳（启动后经过的时间）。dmesg 可以控制是否显示时间戳以及显示格式。

把内核时间戳转换为人类可读格式：

dmesg -T

时间戳会变成类似：

[Fri Mar 20 22:15:30 2026] ...

（视频中还提到通过选项控制 timestamps 的显示与否，不同发行版 / 版本语法略有不同，但核心是“可以开关和格式化时间戳”。）

### 2.5 按 log level 过滤（-l）

printk 支持不同严重级别（log level），例如：

emerg, alert, crit, err, warn, notice, info, debug

dmesg 可以只显示某些级别：

只看 info 级别：

dmesg -l info

只看 error 级别：

dmesg -l err

同时看多个级别（用逗号分隔）：

dmesg -l err,crit,alert,emerg

也可以与 -T 组合，例如：

dmesg -T -l err,crit,alert,emerg

这样既只看错误/严重级别，又带可读时间戳。

### 2.6 组合多个选项

常见组合示例：

只看 info 级别，带可读时间：

dmesg -T -l info

只看严重错误，带可读时间：

dmesg -T -l err,alert

核心：-T 控制时间格式，-l 控制级别，可以随需求自由组合。

---

## 3. 如何用 dmesg 查看 printk 输出

你在 kernel module 或内核代码中写的 printk，最终都会进入内核 ring buffer。dmesg 是最直接的查看方式。

### 3.1 在模块中使用 printk（示例，txt）

示例 C 代码：

static int __init my_init(void)
{
    printk(KERN_INFO "my_module: init called\n");
    return 0;
}

static void __exit my_exit(void)
{
    printk(KERN_INFO "my_module: exit called\n");
}

module_init(my_init);
module_exit(my_exit);

加载模块：

insmod my_module.ko

卸载模块：

rmmod my_module

### 3.2 用 dmesg 查看 printk 输出的基本方法（txt）

1) 直接查看全部内核日志：

dmesg

然后配合 grep 查找模块相关输出：

dmesg | grep my_module

2) 带人类可读时间戳：

dmesg -T

只看与模块相关的行：

dmesg -T | grep my_module

3) 只看最近几行，快速定位：

dmesg -T | tail

或：

dmesg -T | tail -n 50

---

## 4. 调试时常用的“清空 + 重新观察”流程（txt）

调试模块时，为了只看某一次操作产生的 printk 输出，可以使用“先清空再观察”的模式。

步骤 1：清空 ring buffer（root）

sudo dmesg -C

此时 buffer 被清空。

步骤 2：执行你要测试的操作

例如重新加载模块：

insmod my_module.ko

或者运行某个会触发 printk 的路径。

步骤 3：查看这次操作产生的日志

查看全部日志：

dmesg -T

只看与模块相关的日志：

dmesg -T | grep my_module

因为之前已清空 ring buffer，这里的内容基本只包含这一次操作产生的日志，干净易读。

---

## 5. 配合 log level 使用 dmesg 过滤 printk（txt）

如果在 printk 中使用不同 log level，例如：

printk(KERN_INFO "my_module: info message\n");
printk(KERN_ERR  "my_module: error happened\n");

可以只看 error 级别的日志：

dmesg -T -l err | grep my_module

或者看一部分严重级别：

dmesg -T -l err,crit,alert,emerg | grep my_module

这样 info / debug 等低优先级信息不会干扰你定位问题。

---

## 6. 实战可记住的 dmesg 常用命令（速查，txt）

查看所有内核日志（包括你的 printk）：
dmesg

带可读时间戳：
dmesg -T

清空 ring buffer（root），然后做实验：
sudo dmesg -C

只看最新日志：
dmesg -T | tail

只看和你模块相关的内容：
dmesg -T | grep my_module

只看错误级别及更严重的日志：
dmesg -T -l err,crit,alert,emerg
