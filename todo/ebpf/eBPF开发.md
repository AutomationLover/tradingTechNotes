# 如何开发 / 编写 eBPF 程序（整理版）

这份笔记对应问题：“如何开发 编写 eBPF 程序？”。  
按三块来总结：

- 按“开发方式/成熟度”分类
- 按“挂载点/用途”分类
- 通用开发步骤

---

## 一、按“开发方式 / 成熟度”分类

### 1. bcc + Python 风格：教学 & 快速原型

特点：你在 B 站课程里看到的那种模式。

- 内核侧 BPF 程序：
  - 用 C 写，但通常写在 Python 字符串里。
- 用户态：
  - 用 Python + bcc 库：
    - 编译内核态 C 为 BPF 字节码；
    - 加载程序、挂到 kprobe/tracepoint/socket/XDP 等 hook；
    - 从 BPF map 读数据、打印或聚合。

适用场景：

- 学习内核网络、kprobe、tracepoint；
- 临时排障、性能分析（类似 `tcpconnect`, `execsnoop` 那类工具）；
- PoC 级观测/监控。

优点：

- 上手快；
- Python 交互方便，适合调试和教学。

缺点：

- 依赖重（LLVM/clang、bcc 本身）；
- Python 常驻 + bcc 开销，不适合极端高性能、长期驻留数据平面。

---

### 2. libbpf / CO‑RE 风格：生产级 eBPF 应用

这是现在官方推荐的“正统”开发方式。

- BPF 程序：
  - 独立 `.bpf.c` 文件；
  - 每个 hook 用 SEC("...") 标记；
  - 用 clang 编译成 `.bpf.o`（ELF 格式的 BPF 对象）。
- 用户态：
  - 用 C（libbpf），或 Go（libbpf-go）、Rust（libbpf-rs/aya）：
    - 打开 ELF；
    - 加载 BPF 程序和 map；
    - attach 到 XDP、tc、kprobe、tracepoint、LSM、cgroup、socket 等；
    - 和 map/ringbuf 互动。

常见形态：

- 命令行工具（类似 bpftool、自研 perf 工具）；
- daemon/agent（如 Cilium agent、Falco、观测/安全守护进程）。

优点：

- 依赖小（内核 + libbpf 即可）；
- 支持 BPF CO‑RE（Compile Once – Run Everywhere）：
  - 同一 `.bpf.o` 能在多 kernel 版本上跑；
- 性能好，更适合生产。

缺点：

- 上手比 bcc 难一些：
  - 需要理解 ELF section、CO‑RE、libbpf API、BTF 等。

---

### 3. 高层语言封装：Go / Rust 等

不少项目为 libbpf 提供了封装：

- Go：
  - cilium/ebpf；
  - libbpf-go 等。
- Rust：
  - aya；
  - redbpf 等。

模式：

- BPF 程序大多仍用 C 写（也有全 Rust/BPF 的方案）；
- 用户态逻辑用 Go/Rust 写 loader / control-plane：
  - 加载 BPF；
  - attach；
  - 读写 map；
  - 暴露 HTTP/metrics 等。

适用场景：

- Cloud native / observability / 安全产品；
- 需要长期跑、且希望用户态代码更易维护（Go/Rust）。

---

## 二、按“挂载点 / 用途”分类

### 1. XDP / TC：fast path 数据平面

XDP（SEC("xdp")）：

- 挂在网卡驱动 RX 最前面；
- 在 skb 创建之前，对包做：
  - DROP；
  - PASS；
  - TX（回发）；
  - REDIRECT（到其他 netdev / cpumap / AF_XDP 等）。

适合：

- DDoS 防护、L2/L3 ACL；
- 简单 L4 LB；
- 粗粒度计数/采样；
- 要求极致 PPS 的早期决策。

TC BPF（SEC("tc")）：

- 挂在 qdisc 层（clsact / ingress / egress）；
- 已有 skb，可访问更多协议字段；
- 用途：
  - ACL / 分类 / 打标；
  - 带宽控制前后的观测；
  - 更细致的计数、时延采样。

典型组合模式：

- XDP：做“最前面、最简单、最热”的决策（drop/redirect）；
- TC：做后续稍复杂的处理（分流、限速、观测）。

---

### 2. kprobe / tracepoint / kfunc：内核函数与事件 hook

kprobe / kretprobe：

- 挂到任意内核函数入口/返回：
  - 示例：`tcp_v4_connect`, `tcp_sendmsg`, `napi_poll`, `do_sys_open` 等；
- 用于：
  - 记录调用参数、返回值；
  - 测量函数耗时（入口/返回时间戳）；
  - 分析网络、IO、调度等行为。

tracepoint / kfunc：

- 内核预定义的稳定 hook：
  - `net:netif_receive_skb`
  - `tcp:tcp_retransmit_skb`
  - `sched:sched_switch`
  - `irq:softirq_entry` 等；
- ABI 稳定，适合长期维护的 BPF 程序。

用途：

- 观测 / 性能分析 / 故障诊断；
- 不是直接改 datapath，而是旁路测量。

---

### 3. cgroup / socket / LSM：进程/容器/安全域控制

cgroup BPF：

- 挂到 cgroup 的 hook（如 `cgroup/connect4` 等）；
- 通过 cgroup（容器 / 服务 / pod）维度控制：
  - L3/L4 策略（允许/拒绝连接）；
  - 简单限速/计量。

socket 层 BPF：

- 挂在 socket-filter、socket send/recv hook 上；
- 做：
  - 透明代理；
  - L4 负载均衡；
  - 针对特定 socket 的观测/过滤。

LSM BPF：

- 挂在 Linux Security Module hook 上；
- 实现安全策略 / 审计（文件访问、exec、connect 等）。

---

### 4. uprobes / USDT：用户态函数 hook

uprobes / uretprobes：

- 类似 kprobe，但作用在用户态 ELF 的函数；
- 可在你的 feed handler / trading engine / 自研库函数入口/返回打钩子；
- 常用于：
  - 应用层性能分析；
  - 和内核观测结合做端到端 tracing。

---

## 三、通用开发步骤

无论是 bcc 还是 libbpf，写 eBPF 程序的基本流程类似：

### 步骤 1：选择挂载点（hook）

先画清楚你关心的路径，然后选 hook：

- 网络收包：
  - XDP、TC ingress；
  - `napi_poll`, `netif_receive_skb` kprobe / tracepoint；
- 网络发包：
  - TC egress；
  - `tcp_sendmsg`, `dev_queue_xmit` kprobe；
- 连接行为：
  - cgroup/connect hook；
  - `tcp_v4_connect` kprobe；
- 应用行为：
  - uprobes / USDT（如某个策略函数、数据库函数等）。

思路：**先画图（data path / code path），再选挂载点。**

---

### 步骤 2：编写 BPF 程序（C）

- 使用 BPF helper API：
  - `bpf_map_lookup_elem`, `bpf_map_update_elem`, `bpf_ktime_get_ns` 等；
- 使用 SEC("...") 指定 section（挂载点）：
  - `SEC("xdp")`
  - `SEC("tc")`
  - `SEC("kprobe/tcp_sendmsg")`
  - `SEC("tracepoint/net/netif_receive_skb")`
- 遵守 verifier 约束：
  - 循环必须有界；
  - 不越界访问内存；
  - 指针必须可验证；
  - 不能调用阻塞函数。

示意（文本形式）：

SEC("kprobe/tcp_sendmsg")
int handle_tcp_sendmsg(struct pt_regs *ctx) {
    __u64 ts = bpf_ktime_get_ns();
    // 使用 ts 和参数更新 map
    return 0;
}

---

### 步骤 3：定义 BPF map（共享数据结构）

在 BPF C 文件中定义 map，用于：

- BPF 程序内部共享状态；
- BPF 和用户态共享数据。

bcc 风格（宏）：

BPF_HASH(conn_stats, struct key_t, u64);

libbpf / CO‑RE 风格（内嵌 map 定义）：

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, struct key_t);
    __type(value, struct value_t);
} conn_stats SEC(".maps");

选择 map 类型：

- HASH / LRU_HASH：按 key 聚合；
- ARRAY / PERCPU_ARRAY：索引访问、per-CPU 计数；
- RINGBUF / PERF_EVENT_ARRAY：事件流。

---

### 步骤 4：编译

bcc：

- 运行时由 bcc 调用 clang/LLVM 编译；
- BPF C 作为字符串传给 `BPF(text=...)`。

libbpf：

- 构建时编译为 BPF 对象：

clang -target bpf -O2 -g -c prog.bpf.c -o prog.bpf.o

- `.bpf.o` 是 ELF 格式的 BPF 对象：
  - 包含程序、map、BTF 等 section；
  - CO‑RE 依赖 BTF 信息做结构偏移适配。

---

### 步骤 5：加载 & attach

bcc（Python）示意：

from bcc import BPF

b = BPF(text=bpf_program_text)
b.attach_kprobe(event="tcp_sendmsg", fn_name="handle_tcp_sendmsg")

libbpf（C/Go/Rust）示意逻辑：

- 打开对象：
  - bpf_object__open_file("prog.bpf.o", ...);
- 加载：
  - bpf_object__load(obj);
- attach 到某个 hook：
  - XDP：bpf_program__attach_xdp();
  - kprobe：bpf_program__attach_kprobe();
  - TC：tc + bpf offload 组合等。

loader 负责：

- 调用 `bpf()` syscall 创建 map / prog；
- 注册到指定 hook 上；
- 处理加载失败、回滚、cleanup。

---

### 步骤 6：用户态消费数据 / 控制行为

用户态逻辑一般做两件事：

1. **读数据 / 观测：**

   - 定期或按需从 map 中拉取统计信息：
     - 例如计数器、直方图、key → metrics；
   - 从 ringbuf/perf event 中读取事件流：
     - 比如每次 EBPF 程序 emit 的延迟样本、错误事件等。

2. **写配置 / 控制：**

   - 往 map 写入配置：
     - 黑/白名单；
     - 动态策略参数（阈值、开关等）；
   - 作为轻量级 control-plane。

然后可以：

- 将数据导出到 TSDB（Prometheus、InfluxDB 等）；
- 用 Grafana 画曲线；
- 或直接在 CLI 中输出用于 debug。

---

## 四、小结：从 demo 到实战的脑图

- 开发方式：
  - bcc：学习和快速实验；
  - libbpf + CO‑RE：生产级、长期运行；
  - Go/Rust 封装：写 agent 更舒服。
- 挂载点：
  - XDP / TC：数据平面；
  - kprobe / tracepoint / uprobes：关键函数和事件；
  - cgroup / socket / LSM：策略、安全、多租户。
- 流程：
  - 画路径 → 选 hook → 写 BPF + map → 编译 → 加载 & attach → 用户态消费/控制。

对于你当前的学习项目（Linux 内核网络 + eBPF + HFT），可以先用 bcc 在 kprobe/tracepoint 上多写几个观测工具，把路径和数据打点跑顺，再逐步迁移到 libbpf/CO‑RE 版本，作为“生产级”的实现。
