## 2003 年常用的网络与驱动相关调试工具（以 Linux 为主）

下面分两块说：一块是 Socket/网络编程常用调试工具，一块是 Linux 驱动/内核相关的调试工具，并标出哪些今天还很常用。

---

## 2003 年的 Socket / 网络调试工具

### 当时常用的工具

- **tcpdump**  
  2003 年已经非常成熟，用来抓包、分析 TCP/UDP、协议头字段。  
  典型用法：
  - `tcpdump -i eth0`
  - `tcpdump -i eth0 tcp port 80`
  - `tcpdump -ni eth0 host 10.0.0.1`

- Ethereal / 早期 Wireshark  
  当时叫 Ethereal，是图形化抓包分析工具。  
  把 tcpdump 抓的 `pcap` 文件导入 Ethereal 分析，或直接用它抓。

- netstat / ss（当时主要是 netstat）  
  用来看连接状态、端口监听、路由表等：
  - `netstat -tnp`（TCP 连接 + 进程）
  - `netstat -an`（所有监听 + 连接）
  现在更多用 `ss`，但 2003 年 `netstat` 是主角。

- ping / traceroute  
  最基础的连通性排查工具：
  - `ping` 看丢包/延迟
  - `traceroute` 看路径、哪一跳有问题

- telnet / nc (netcat)  
  测试 TCP 端口连通性、手工发协议报文：
  - `telnet host 80`
  - `nc host 80`
  常用于「我服务到底在不在、能不能连」这种检查。

- strace（从用户态看系统调用）  
  对 Socket 程序很有用，可以看到进程在调用哪些 syscall，是否卡在 `connect`/`accept`/`read`等：
  - `strace -p <pid>`
  - `strace -f -o trace.log ./server`

### 哪些今天依然非常常用

几乎全部还在用，而且用法变化不大：

- `tcpdump` + 图形化 Wireshark：现在仍然是网络问题定位最基础的组合。
- `ping`、`traceroute`、`netstat`/`ss`、`telnet`/`nc`：今天你排查连不通、端口不通、路由不对，流程和当年很像。
- `strace`：排查阻塞在什么 syscalls、看错误码、看重试逻辑，仍然是首选。

可以说，如果你会用这些 2003 年的工具，再加上现在的 `ss`、`ip`、`curl`、`nmap`、`dig` 等，就够应付绝大多数「从用户态看网络」的问题了。

---

## 2003 年的 Linux 驱动 / 内核调试工具

### 当时的主力工具

- **printk + dmesg**  
  写驱动最常用的「printf 调试」：  
  - 内核里用 `printk(KERN_INFO "xxx\n");`
  - 用户态看 `dmesg` 输出  
  当时没有那么多 tracing 工具，大多数人严重依赖 printk。

- syslog / klogd  
  内核日志通过 syslog 或 klogd 收集，写到 `/var/log/messages` 等文件。  
  很多驱动问题就是看 `dmesg` + 系统日志综合分析。

- gdb + 远程调试（kgdb / gdb over serial）  
  2003 年也有给内核打 patch 用 kgdb、串口 gdb 远程调试内核的玩法，但部署比较麻烦，多用于嵌入式或专门搞 kernel dev 的人。  
  应用层用 gdb 调试用户态程序已经非常普遍。

- /proc 调试接口  
  那时大量驱动会在 `/proc` 下挂一些调试节点，打印状态、统计信息、寄存器内容等。  
  用户态 `cat /proc/xxx` 就能看到内部状态。

- early oprofile（性能分析）  
  oprofile 可以对 CPU 栈进行采样，用来做性能分析。2003 年已经出现，但工具链相对现在粗糙许多。

- vmstat / iostat / sar / top 等系统观察工具  
  用来从宏观上看 CPU 利用率、上下文切换、I/O 等情况，辅助判断「问题是不是在内核态/驱动层」。

### 哪些思想 & 工具今天还在

- **printk + dmesg** 依然是驱动调试「入门第一招」  
  虽然现在有 ftrace、perf、bpftrace 等高级工具，但 printk 仍然是最直观的方式，尤其是你要看初始化流程、错误路径时。

- gdb 调用户态程序的使用场景没有变化  
  对网络服务，今天还是会用 gdb + core dump 看崩溃、死锁、死循环。

- `/proc` 风格的调试接口概念延续到今天  
  只是现在更推荐 `debugfs` / `sysfs` 等更规范的接口，但「写驱动时暴露一些 debug 信息给用户态」这件事本身没变。

- oprofile 的思路被 perf 等继承  
  perf 本质上是 oprofile 那一系演进而来，所以 2003 年那套「采样栈、观察热点」的性能分析思路现在仍适用，只是工具更强大。

---

## 今天额外常见、但 2003 刚萌芽甚至不存在的东西（对比一下）

给你一个时间感，大概哪些是「2003 几乎没有、今天你会常用」的：

- 内核 / 驱动方向：
  - ftrace / tracepoints / trace-cmd
  - perf (perf record / perf report)
  - eBPF / bpftrace / BCC
  - systemtap（2003 年刚起步，后来才慢慢成熟）

- 网络方向：
  - iproute2 (`ip addr` / `ip route` / `ip link`)，2003 已有但很多人还在用 `ifconfig` + `route`。
  - curl、httpie 这类 HTTP 调试工具今天非常常用，当年更多是自己写小程序或用 telnet 手撸协议。
  - 高层框架自带的 debug/metrics（nginx 的 stub_status、Envoy/Istio metrics、Prometheus/Grafana 等）在 2003 都还不存在或刚刚萌芽。

---

## 小结：2003 的工具学了，今天仍然很值

从你的学习目标（NAPI、驱动、内核网络性能调优）来看，当年那批工具和方法有几个价值点：

1. 低层工具（tcpdump、printk、gdb、strace）到今天依然是「最后的底线」，永远不会过时。  
2. 2003 年的调试方式逼着你「从协议和内核视角」看问题，有利于你把 XDP / eBPF / NAPI 这些新东西和传统路径对照起来。  
3. 只要把这些老工具和今天的 perf / ftrace / bpftrace 联动起来，你的知识图谱会非常完整，从 2003 一直铺到现在的云原生时代。

如果你愿意下一步我可以帮你把「针对网络性能问题」的调试流程整理成一张从 2003 老工具到现代工具的对照表，比如：先 tcpdump/ss → 再 strace/gdb → 再 perf/ftrace/bpftrace 等，对应你现在在学的内容。  

---

## 用「2003 老方法」对比现在的 NAPI（尽量用很简单的说法）

先假设场景都是「网卡收包 → 内核 → 上层应用」。

### 2003 年传统老方法（无 NAPI 的典型模型）

可以简单理解为：**“谁来数据谁就不停按门铃”**。

- 网卡收到每个包，都触发一次硬中断（IRQ）。  
- 内核的中断处理函数（硬中断上半部）被频繁唤醒：
  - 从网卡 DMA buffer 里把包搬进内核网络栈；
  - 做完就返回。
- 如果流量很大，就会出现：
  - 中断次数非常多（interrupt storm），CPU 大量时间花在「响应中断 + 切换上下文」上；
  - CPU 还没处理完前一批包，又来了新的中断；
  - 导致实际用于「真正处理包 / 做业务逻辑」的时间反而变少。

直观一点：  
2003 的典型做法 = **「每来一个快递，就按一次门铃叫你下楼取件」**。  
快递多的时候，你一整天都在上楼下楼，效率很低。

### 现在的 NAPI 模型（高负载时的“混合模式”）

NAPI 的核心想法可以简化成一句话：  
**“流量小就按门铃，流量大就直接改成你定时下楼扫一遍快递柜”**。

- 流量小时：  
  - 跟老方法差不多，用中断通知收包，延迟低（低负载延迟友好）。
- 一旦流量大到某个阈值：  
  - 网卡先用中断把 CPU「叫醒」一次；
  - 驱动在中断里 **关闭后续接收中断**；
  - 把这个网卡加入一个 poll 列表；
  - 后面由内核在软中断 / NAPI poll 里「主动轮询」网卡接收队列，一次性把很多包取出来；
  - 当包处理得差不多（队列快空了），驱动再重新开启中断。

这样做有几个简单直观的差别：

1. **中断数量**  
   - 老方法：高流量时中断次数 ≈ 收到的包数，疯狂打断 CPU。  
   - NAPI：高流量时「先中断一次 → 后面主要靠轮询」，中断次数大幅减少。

2. **CPU 利用方式**  
   - 老方法：CPU 被中断频繁抢占，一直在「被动」地被唤醒处理少量数据。  
   - NAPI：CPU 在高负载时变成「主动一次处理一大批包」，更像批处理，效率更高。

3. **缓存和批处理效果**  
   - 老方法：每次中断只处理少量包，cache 命中和批处理效果都不稳定。  
   - NAPI：一次 poll 处理一批包，cache 利用率好、每包的 amortized 开销更小。

4. **延迟特性**  
   - 老方法：在低负载时，包一来就中断，延迟很低；高负载时被中断和上下文切换压垮，整体反而变差。  
   - NAPI：低负载仍然用中断（保持低延迟），高负载转成轮询（牺牲一点延迟换吞吐和稳定性）。

5. **驱动里的逻辑形态**  
   - 老方法：驱动里的重点在「中断 handler 里收包」，下半部也以中断为主线。  
   - NAPI：驱动必须实现 `poll` 回调，并在合适的时机 `napi_schedule` / `napi_complete`，显式控制：
     - 什么时候关接收中断；
     - 什么时候轮询处理多少包；
     - 什么时候再打开中断。

---

### 一句话总结差异（方便你写在笔记里）

- 2003 年传统方式：  
  **“纯中断驱动收包：每个包都按门铃，流量大就中断风暴。”**

- NAPI 模式：  
  **“中断 + 轮询混合：低负载用门铃，高负载关门铃，改成你定期下楼一次收一大批包。”**

本质上就是：  
**从“每包一次中断”，升级成“先用中断触发，再用轮询批量干活”，把 CPU 从中断风暴中解救出来。**


---

## 2003 年左右的 Socket 编程关键技术（以 Linux 为主）

可以把 2003 年的 Socket 编程理解成「经典 UNIX 网络编程」：C 语言 + BSD Sockets + 简单并发模型。

### 当时比较关键的技术点（2003）

在 2003 年的 Linux（2.4 → 早期 2.6 内核）上：

- **BSD Sockets API**  
  核心系统调用基本就是今天还在用的这一套：  
  `socket`、`bind`、`listen`、`connect`、`accept`、`send`、`recv`、`sendto`、`recvfrom`、`getsockopt`、`setsockopt`。  
  地址族主要是 IPv4（`AF_INET`），IPv6（`AF_INET6`）已经有，但用得不算特别广。

- 阻塞 I/O + 简单并发模型  
  当时常见的两类服务端写法：
  1. 每个连接一个进程/线程：`accept` 之后 `fork` 或 `pthread_create`。
  2. 少量 worker 进程/线程，每个处理多个连接（有点像早期 prefork 模式）。

- 基于 `select` / `poll` 的事件循环  
  想在单线程里管理多路连接，基本就用：
  - `select` + `fd_set`
  - 或 `poll` + `struct pollfd` 数组  
  这是「可移植」写法的主流方案。

- Linux 特有的 **epoll 刚刚登场**  
  `epoll` 当时是「新玩意」+「高性能 Linux 特性」：
  - `epoll_create`、`epoll_ctl`、`epoll_wait`  
  主要用来解决 `select`/`poll` 在大量 FD 场景下的性能问题（C10k 问题刚开始被广泛讨论）。

- 基本 TCP/UDP 行为  
  三次握手、四次挥手、TIME_WAIT、backlog、Nagle 算法、`SO_REUSEADDR` 等等，已经是写服务必须懂的基础。

- 安全性与健壮性基础  
  比如：
  - 处理「短读/短写」（read/write 只读/写了部分数据）
  - 处理 `EAGAIN` / `EWOULDBLOCK` / `EINTR`
  - 避免 C 语言里的缓冲区溢出  

这些在当时就属于「写网络服务必须过的一关」。

### 哪些部分今天依然是主流

其实 Socket 这块很多东西几乎没变，主要变化在「规模」和「封装」：

- **BSD Sockets API 基本原样保留到今天**  
  现在的 Linux 上，你还是用这一套系统调用。  
  一份 2003 年写得还不错的 C 语言 Socket 服务端代码，今天通常可以几乎不改就编译运行。

- TCP/UDP 基本原理几乎没变化  
  三次握手、拥塞控制、TIME_WAIT、MTU/分片这些，今天还是网络工程和性能调优的核心概念。

- **epoll 从小众变成主流**  
  当年是「新特性」，现在是高性能服务器标配：
  - nginx、HAProxy 等高性能服务直接用 epoll 风格事件循环。
  - 各种语言的异步运行时（Go、Rust、Java NIO、libuv/Node.js）本质上都是对 epoll/kqueue/IOCP 的封装。

- 非阻塞 I/O + 事件驱动模型更重要  
  将 FD 设为 non-blocking，配合事件循环处理大量连接，这种设计在「C10k / C100k / C100 万连接」时代更关键。

- 健壮性与安全要求只增不减  
  以前要考虑短读/短写、错误处理，现在还要加上：
  - TLS 到处都是（加密握手、证书校验）
  - DoS/连接洪泛防护、限流、超时策略等

总结一下：如果你今天拿一本 2003 年的 Linux 网络编程书看，里面讲的大部分概念到现在都还是对的，只不过需要脑补/补充：epoll、现代的异步编程模式、TLS、云原生环境等新内容。

---

## 2003 年左右的 Linux 驱动开发关键技术

2003 年正好是 Linux 2.4 → 2.6 的大迁移阶段，这是驱动层面非常大的一个转折点：新的驱动模型、更好的电源管理、网络里的 NAPI 等都在这个时期出现。

### 当时比较关键的技术点（2003）

- 内核版本 & 驱动模型变化  
  - 2.4：驱动模型比较原始，大量使用 `/proc`，抽象层次较少。
  - 2.5/2.6：引入新的驱动模型，引入 `struct device`、`struct bus_type` 等抽象，以及基于 `sysfs` 的设备树。  
    写驱动的人必须非常关心自己目标内核版本，是 2.4 还是 2.6。

- 可加载内核模块（LKM）  
  写 `.ko` 模块，典型框架：
  - `module_init` / `module_exit`
  - `MODULE_LICENSE`、`MODULE_AUTHOR` 等宏  
  加载卸载用 `insmod`、`rmmod`、`modprobe`。

- 常见驱动类型  
  大致三大类：
  1. 字符设备驱动：实现 `/dev/xxx`，通过 `file_operations` 提供 `open`/`read`/`write`/`ioctl` 等。
  2. 块设备驱动：接老的 block layer API。
  3. 网络驱动：基于 `struct net_device`，实现回调，比如 `open`、`stop`、`hard_start_xmit` 等。

- 中断处理 + 下半部机制  
  典型流程：
  - 用 `request_irq` 申请中断，`free_irq` 释放。
  - 中断上半部（hard IRQ handler）尽量短，只做必要工作。
  - 通过「下半部」做耗时处理，例如：
    - `tasklet`
    - `softirq`
    - 更早期已废弃的 bottom-half 机制  
  网络方向上，NAPI 机制刚出现，用来缓解高吞吐 NIC 带来的中断风暴。

- 内存与 DMA 操作  
  为设备分配/映射内存，典型操作：
  - 使用 `ioremap`、`readl`、`writel` 等访问 MMIO 寄存器。
  - DMA API：`pci_alloc_consistent`、`pci_map_single` 等。  
  并且要理解 cache 一致性、对齐、DMA-safe 缓冲区等问题。

- 同步原语  
  使用：
  - `spin_lock` / `spin_lock_irqsave`
  - 信号量 / mutex
  - 原子操作  
  以及清楚中断上下文 VS 进程上下文的限制（中断上下文不能睡眠等）。

- 配置 & 用户态接口  
  - 通过 `ioctl` 提供设备特有的控制接口。
  - 大量使用 `/proc` 暴露调试/状态信息。
  - 当时 `sysfs` 刚开始被采用，用于更结构化地暴露设备属性。

### 哪些部分今天依然是主流

Linux 驱动开发的「核心思想」变化不大，主要是具体 API 的演进和清理。

仍然非常重要的部分包括：

- **模块化驱动与生命周期管理**  
  直到今天，`module_init` / `module_exit`、引用计数、资源清理顺序等仍是驱动开发的基本功。

- **设备模型、总线和 sysfs**  
  2.5/2.6 引入的新驱动模型，实际上成为后面长期稳定的基础：
  - `struct device`
  - `struct bus_type`
  - `struct device_driver`
  - 通过 sysfs 暴露设备属性  
  如果理解了当年 2.6 的驱动模型，现在的内核在概念层面是延续这套设计的。

- 字符设备 & 网络驱动的基本模式  
  字符设备：实现 `file_operations`，给用户态提供一个 `/dev/xxx` 接口。  
  网络驱动：`struct net_device` + 一组 `ndo_*` 回调。名字和细节变了不少，但「这个设备是一个 net_device，上面挂一堆回调」的模式是同一套。

- 中断、软中断以及 NAPI 的思想  
  - 「中断上半部尽快返回，耗时工作放到下半部/其他上下文」这个设计理念完全没变。
  - NAPI 风格的「高负载时改用轮询」仍然是现代 NIC 驱动的核心模式。  
  虽然具体 API 演进（tasklet 走向废弃，更多使用工作队列、线程化中断等），但「中断 → 延迟执行」的分层仍是主线。

- DMA 和 MMIO 的基本模型  
  映射设备寄存器、管理 DMA 缓冲区的原理并没有变化，只是 API 更统一、更安全一些。

- 并发控制和内存模型  
  什么时候用 spinlock、什么时候用 mutex、什么时候用 RCU，这些仍是现代驱动和内核开发中最重要的能力之一。  
  对「能不能睡眠」「当前上下文是否可以阻塞」的判断规则也和 2003 年本质一致。

变化较大的地方包括：

- 很多 2.4 / 早期 2.6 的具体 API、结构体字段现在已经废弃或大改（老的 net_device 回调、老 block layer、老式 bottom half 等）。
- `/proc` 现在不太鼓励随便挂自定义东西；更推荐用 `sysfs`、`configfs`。
- 电源管理（suspend/resume）、热插拔（hotplug）在现在的驱动里比当年重要得多，相关框架也成熟很多。

---

## Socket & 驱动这两块从 2003 到今天的共通主题

如果你把「Socket 编程」和「驱动开发」当成一个整体来看，有几类 **跨越 20 年** 都很值钱的核心能力：

- 对 Linux 网络栈的整体视角  
  从用户态的 `socket` 调用，一路到 TCP/UDP/IP 协议栈，再到 qdisc / net_device，再到 NIC 驱动、中断、DMA，打通这一条链，在今天做云网络 / 性能调优都非常有价值。

- 对并发与异步的理解  
  用户态：非阻塞 I/O、事件循环（epoll）。  
  内核态：中断、软中断、NAPI、workqueue、各种锁与内存模型。  
  这些概念 2003 年就已经存在，现在只是进一步工程化和抽象化。

- 性能思维  
  系统调用开销、上下文切换、缓冲区大小、零拷贝、batch 处理、interrupt vs polling、offload 能力等，从 2003 一直到现在都是高性能网络/驱动设计的关键词。

- 清晰的用户态 / 内核态边界意识  
  哪些逻辑放在用户态，哪些逻辑必须下沉到内核态，该用 syscalls 还是 netlink、还是字符设备 / sysfs 暴露控制接口，这些设计问题在今天依然是架构决策的关键点。

如果你现在回头系统看一遍 2003 那一代的 Socket 编程和 Linux 驱动教程，再对照现代内核文档和高性能网络实践，其实很容易把历史演进和现在的最佳实践串在一起，这对你理解 XDP、NAPI、eBPF 等后来的技术会有非常大的帮助。


---


## Socket programming around 2003 (Linux focus)

You can roughly think of 2003-era socket programming as “classic UNIX networking”: C + BSD sockets + simple concurrency models.

### What was key in 2003

On Linux in 2003 (kernel 2.4 → early 2.6):

- **BSD sockets API**  
  Core calls: `socket`, `bind`, `listen`, `connect`, `accept`, `send`, `recv`, `sendto`, `recvfrom`, `getsockopt`, `setsockopt`.  
  Mostly IPv4 (`AF_INET`), with growing but less common IPv6 (`AF_INET6`).

- **Blocking I/O + simple concurrency models**  
  Two common patterns:
  1. One process/thread per connection: using `fork` or `pthread_create` after `accept`.
  2. A small number of worker processes handling many connections.

- **select/poll-based event loops**  
  For handling many FDs with a single thread: `select` and `poll` were the standard portable mechanisms.  
  You might see:
  - `select` with `fd_set`
  - `poll` with `struct pollfd` arrays

- **epoll just arriving (Linux-specific)**  
  `epoll` was new and considered “advanced”/Linux-only:
  - `epoll_create`, `epoll_ctl`, `epoll_wait`
  It was specifically introduced to solve the scalability problems of `select/poll` for thousands of connections.

- **Basic TCP/UDP behavior**  
  Understanding handshake, TIME_WAIT, backlog, Nagle’s algorithm, `SO_REUSEADDR`, etc., was already essential.

- **Security and robustness basics**  
  Handling partial reads/writes, EAGAIN/EWOULDBLOCK, signal interruptions (EINTR), and avoiding buffer overflows in C.

### What is *still* common and popular today

Most of the conceptual stack hasn’t changed; what changed is scale and tooling:

- **BSD sockets API is almost identical**  
  The function set and semantics are still the same on modern Linux.  
  If you wrote a well-structured C server in 2003, it likely compiles today with minimal changes.

- **TCP/UDP fundamentals are unchanged**  
  Three-way handshake, congestion control basics, TIME_WAIT, MTU/fragmentation, etc., are still critical.

- **epoll is now mainstream**  
  In 2003 it was “cutting edge”. Today:
  - High-performance servers (nginx, HAProxy, etc.) rely on epoll-style event loops.
  - Many languages’ async runtimes (Go, Rust, Java NIO, libuv for Node.js) are thin wrappers over epoll/kqueue/IOCP.

- **Nonblocking I/O + event-driven design**  
  The classic pattern of setting sockets nonblocking and using an event loop is even *more* important now (C10k/C100k/C1M problems).

- **Security & robustness concerns are the same, but stricter**  
  You still must handle partial I/O, errors, timeouts; now you also think about TLS everywhere, DoS resistance, rate limiting, etc.

So: a lot of the mental model you’d learn from a 2003 Linux socket programming book still transfers almost 1:1; you’d just add epoll, async patterns, TLS, and “cloud-native” tooling on top.

---

## Linux driver programming around 2003

Around 2003, Linux moved from 2.4 to 2.6, which was a *big* transition: new driver model, better power management, NAPI for networking, etc.

### What was key in 2003

- **Kernel version and driver model**  
  - 2.4 series: older driver model, heavy use of `/proc`, fewer abstractions.
  - 2.5/2.6 series (around 2002–2003): new driver model with `struct device`, `struct bus_type`, `sysfs` for representing devices in userspace.  
  Driver writers had to care about which kernel they targeted.

- **Loadable kernel modules**  
  Writing `.ko` modules with:
  - `module_init` / `module_exit`
  - `MODULE_LICENSE`, `MODULE_AUTHOR`, etc.
  Loading/unloading via `insmod`, `rmmod`, `modprobe`.

- **Basic driver types**  
  1. Character drivers (implementing `file_operations` for `/dev/xxx`)
  2. Block drivers (old block layer APIs)
  3. Network drivers (`struct net_device` with callbacks like `open`, `stop`, `hard_start_xmit`, etc.)

- **Interrupt handling + bottom halves**  
  Handling hardware interrupts via:
  - `request_irq`, `free_irq`
  - Top half (quick ISR) and bottom halves via:
    - `tasklet`s
    - `softirq`s
    - (older bottom-half mechanisms being phased out)
  NAPI was introduced to mitigate interrupt storms in high-throughput NICs.

- **Memory and DMA**  
  Mapping device memory and buffers:
  - `ioremap`, `readl`, `writel` for MMIO
  - DMA APIs: `pci_alloc_consistent`, `pci_map_single`, etc.  
  Plus knowledge of cache coherency and alignment.

- **Synchronization primitives**  
  Using:
  - `spin_lock`, `spin_lock_irqsave`
  - semaphores and mutexes
  - atomic operations  
  Understanding interrupt context vs process context was crucial.

- **Configuration & user-space interfaces**  
  - `ioctl` for device-specific control
  - `/proc` entries (somewhat abused)
  - Early use of `sysfs` for structured device attributes

### What is *still* common and popular today

The core ideas of Linux driver programming are very similar; the *APIs* evolved, and some mechanisms were cleaned up or deprecated.

What’s still fundamental:

- **Kernel modules and lifecycle**  
  `module_init` / `module_exit`, reference counting, and careful teardown are still essential.

- **Device models, buses, and sysfs**  
  The driver model introduced around 2.5/2.6 became the stable foundation:
  - `struct device`, `struct bus_type`, `struct device_driver`
  - sysfs as the main way to expose device attributes  
  If you understand the 2.6 driver model, you are conceptually aligned with current kernels.

- **Character and network drivers as basic building blocks**  
  Character drivers with `file_operations` and network drivers with `struct net_device` and `ndo_*` ops are still standard patterns, even though function names and fields changed.

- **Interrupts, softirqs, and NAPI concepts**  
  - The idea of splitting work between hard IRQ and deferred processing still applies.
  - For NICs, NAPI-style polling under high load is still the core model.
  Even though the exact APIs changed (tasklets being deprecated, more work moved into threaded IRQs, workqueues), the pattern “short top half, heavy work bottom-half/context” remains.

- **DMA and MMIO handling**  
  The fundamental ideas of mapping device memory and using DMA-safe buffers are unchanged; only the helper functions and best practices have been refined.

- **Locking and concurrency**  
  `spinlock` vs mutex vs RCU etc. remain central topics. The rules about what you can do in interrupt context vs process context are the same.

What changed or became “legacy”:

- Many specific APIs and struct fields from 2.4/early 2.6 are now deprecated or significantly refactored (old network driver callbacks, old block layer APIs, older interrupt bottom-half mechanisms like tasklets).
- `/proc` is less used for driver configuration; `sysfs` and `configfs` are preferred.
- Power management (suspend/resume) and hotplug support are much more important today than in typical 2003 tutorials.

---

## Common themes that bridge 2003 → today

Across both socket programming and Linux driver development, a few **conceptual** skills have remained very valuable:

- Understanding the **Linux networking stack** end-to-end: from `socket` in userspace, through TCP/UDP/IP, through qdisc and `net_device`, down to NIC driver, interrupts, and DMA.
- Being comfortable with **concurrency and asynchrony**: event loops (epoll), interrupts, bottom halves, locking, and memory ordering.
- Being able to reason about **performance**: syscalls, context switches, buffer sizes, zero-copy, interrupt vs polling, batching.
- Knowing **where user space ends and kernel space begins**, and how they interact (syscalls, `ioctl`, netlink, character devices, sysfs).

If you study materials from the 2003 era but keep an eye on modern APIs (epoll, current `net_device`/NAPI interfaces, sysfs usage), you’ll get both historical context and skills directly useful on current kernels.
