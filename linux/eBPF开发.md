# 如何开发 / 编写 eBPF 程序（分类总结版）

这份笔记对应的问题是：“如何开发 编写 eBPF 程序？请分类总结”。

下面按“开发方式/成熟度”、“挂载点/用途”、“通用开发步骤”三大块来整理。

---

## 一、按“开发方式 / 成熟度”分类

### 1. bcc + Python 风格：教学 & 快速原型

特点：你视频里看到的那种形式。

- 内核侧 BPF 程序：
  - 用 C 写，但作为字符串嵌在 Python 程序里。
- 用户态：
  - 用 Python（或 Lua）+ bcc 库：
    - 编译内核态 C 为 BPF 字节码；
    - 加载程序、挂载到 kprobe/tracepoint/socket/XDP 等；
    - 从 BPF map 读数据、打印或聚合。

适用场景：

- 学习 eBPF、内核网络、kprobe/tracepoint；
- 临时排障、性能分析（`execsnoop`, `tcpconnect` 一类）；
- PoC 级观测/监控。

优点：

- 上手快、Python 友好；
- 适合作教学和一次性工具。

缺点：

- 依赖重（LLVM/clang、bcc）；
- Python 常驻 + bcc overhead，不适合极致性能和长期驻留生产 datapath。

---

### 2. libbpf / CO-RE 风格：生产级 eBPF 应用

这是现在官方推荐的主路线。

- BPF 程序：
  - 用独立的 `.bpf.c` 文件写；
  - 每个 hook 用 `SEC("...")` 注解；
  - 编译成 ELF 对象（`.bpf.o`）。
- 用户态：
  - 用 C（libbpf）、Go（libbpf-go）、Rust（libbpf-rs/aya）：
    - 打开 ELF；
    - 加载 BPF 程序和 map；
    - attach 到 XDP、tc、kprobe、tracepoint、LSM、cgroup、socket 等；
    - 处理 map / ringbuf 中的输出数据。

常见形态：

- 命令行工具（类似 bpftool / 自研 perf 工具）；
- 常驻 agent（如 Cilium agent、Falco、观测/安全守护进程）。

优点：

- 依赖小（内核 + libbpf）；
- 支持 BPF CO‑RE（Compile Once – Run Everywhere），同一个 .o 可以在多 kernel 上跑；
- 性能好，更适合生产环境。

缺点：

- 学习曲线比 bcc 略陡：需要理解 section、CO‑RE、libbpf API 等。

---

### 3. 高层语言封装（Go / Rust / 其它）

不少项目为 libbpf/eBPF 提供了更高层封装：

- Go：
  - cilium/ebpf；
  - libbpf-go 等。
- Rust：
  - aya；
  - redbpf 等。

模式：

- BPF 程序多数仍是 C（也有全 Rust/BPF 的尝试）；
- 用户态逻辑用 Go/Rust 实现加载、attach、读 map/写 map、暴露指标等；
- 适合你在 cloud networking / observability 里写“长期跑的 agent / sidecar”。

---

## 二、按“挂载点 / 用途”分类

### 1. XDP / TC：数据平面流量处理（fast path）

**XDP（`SEC("xdp")`）**

- 挂在网卡驱动 RX 最前端；
- 能在 skb 创建前对包做：
  - DROP；
  - PASS；
  - TX（重发）；
  - REDIRECT（到其他接口 / AF_XDP / cpumap 等）。

适用场景：

- DDoS scrub、防火墙粗过滤；
- 简单 L4 负载均衡；
- 计费/采样、计数器；
- 需要极致 PPS、很低 L3/L4 逻辑复杂度。

**TC BPF（`SEC("tc")`）**

- 挂在 qdisc 层（clsact / ingress / egress）；
- 已经有 skb，可以访问更多协议信息；
- 适合：
  - 更接近 L3/L4 的处理；
  - 流量分类、ACL、打标；
  - 细粒度观测（统计/时间戳）。

典型组合：

- XDP 做“最前面
