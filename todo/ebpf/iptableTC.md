# iptables 和 tc 的关系，以及它们在内核中的位置

这篇说明回答两个问题：

1. iptables 和 tc 分别在内核的什么位置？
2. 它们之间的关系是什么，各自负责哪一层的事情？

简化理解：

- **iptables（netfilter）**：在 IP 层 / 传输层决定“这个包要不要放行、要不要改地址/端口”等策略。  
- **tc（qdisc）**：在设备层附近决定“这个包走哪条队列、以什么顺序/速率发出去”。

---

## 1. iptables 在内核中的位置：netfilter 钩子

iptables 实际是 netfilter 框架的用户态前端。用户写的规则（filter/nat/mangle 等）会被内核安装到不同的 netfilter hook 上。

以 IPv4 为例，一个包在内核中大致会经过这些 netfilter 钩子：

### 1.1 收包方向（从 NIC 进来）

路径简化：

- NIC → NAPI / netif_receive_skb → ip_rcv

在 ip_rcv 之后有：

- `NF_INET_PRE_ROUTING`（PREROUTING 链）  
  - 常见用途：基于目的 IP/端口做 DNAT、早期过滤。

然后根据路由决策：

- 如果目的地是本机：
  - `NF_INET_LOCAL_IN`（INPUT 链）  
    - 对送给本机 socket 的流量做防火墙检查。
- 如果需要转发：
  - `NF_INET_FORWARD`（FORWARD 链）  
    - 对转发流量做过滤。

### 1.2 发包方向（本机发出）

路径简化：

- 应用 send() → tcp_sendmsg/udp_sendmsg → ip_queue_xmit / ip_local_out

在 ip_local_out 之后有：

- `NF_INET_LOCAL_OUT`（OUTPUT 链）  
  - 对本机发出的流量做过滤、改字段等。

路由确定出口设备后：

- `NF_INET_POST_ROUTING`（POSTROUTING 链）  
  - 常见用途：SNAT/MASQUERADE 等。

最终：

- ip_finish_output → dev_queue_xmit → 驱动 → NIC

**总结：**

iptables 在这些 hook 上进行处理：

- 放行 (`ACCEPT`) / 丢弃 (`DROP`)；
- NAT（SNAT/DNAT/MASQUERADE）；
- 修改 IP/TOS/TTL 等。

它位于 **IP 层和传输层之间**，在协议栈中段做“策略和安全决策”。

---

## 2. tc 在内核中的位置：qdisc / 流量控制层

tc 是 Linux **流量控制（Traffic Control）/ 队列调度（qdisc）** 的用户态前端。

### 2.1 egress（发送方向）

在数据从内核协议栈到达设备前，路径大致是：

- 应用 send() / write()  
  - tcp_sendmsg / udp_sendmsg → ip_queue_xmit / ip_local_out  
  - ip_output / ip_finish_output → dev_queue_xmit(skb)

在 `dev_queue_xmit(skb)` 这里：

- skb 会被交给对应网卡的 **qdisc（队列规则）**：
  - 默认可能是 `pfifo_fast`、`fq_codel`、`fq` 等；
  - 你可以用 tc qdisc / tc class / tc filter 自定义：
    - 队列结构（PFIFO、HTB、TBF、FQ…）
    - 流量分类（tc filter / TC-BPF）
    - 整形（shaping）、限速（policing）。

随后：

- qdisc 决定发送顺序和速率；
- 驱动从 TX ring 把包送到 NIC。

### 2.2 ingress（接收方向）

接收方向上的 tc ingress：

- 在 `netif_receive_skb` 调用链中，如果配置了 ingress qdisc（比如 `clsact`），会调用 ingress qdisc；
- 你可以在 ingress 上用 tc filter / TC-BPF 做：
  - 丢包 / 放行；
  - 标记（打 tc index / skb mark 等）；
  - mirror / redirect 到其他设备。

注意：

- ingress qdisc 不能真正“排队”（包已经到达了），更多是用于过滤和观测；
- 真正的队列/整形主要发生在 egress（发送方向）。

**总结：**

tc 在 **设备层附近（L2/L3 之间）** 控制：

- 包如何排队；
- 走哪条子队列/class；
- 是否整形/限速/优先级调度。

---

## 3. iptables 和 tc 的关系与差异

可以用一句话来区分两者：

> **iptables 决定“这个包能不能进/出系统、长什么样”；  
> tc 决定“这个包在某个设备上以什么方式排队和发送”。**

### 3.1 工作层级

- **iptables / netfilter：**
  - 主要在 L3（IP） + L4（TCP/UDP）层工作；
  - 基于源/目的 IP、端口、协议、连接状态等匹配；
  - 做：
    - 安全策略 / ACL；
    - NAT（SNAT/DNAT/MASQUERADE）；
    - 基本包头修改（TTL/TOS 等）。

- **tc / qdisc：**
  - 主要在设备层（接近 L2）附近工作；
  - 按接口/队列来设计排队规则；
  - 做：
    - 队列调度（PFIFO、HTB、FQ、CQ 等）；
    - 整形 / 限速；
    - 优先级区分。

### 3.2 在发包路径上的先后顺序

以本机发出的 TCP 包为例（简化）：

1. 应用 `send()`；
2. 内核：
   - `tcp_sendmsg` → `ip_queue_xmit` → `ip_local_out`；
3. netfilter：
   - **OUTPUT 链（iptables）**：对本机发出的包做过滤/修改；
4. 路由 / 选出口设备；
5. netfilter：
   - **POSTROUTING 链（iptables）**：做 SNAT 等；
6. qdisc / tc：
   - `dev_queue_xmit` → **qdisc（tc）**：排队、整形、决定发送顺序；
7. 驱动 → NIC。

所以：**iptables 先做策略/NAT，tc 后做排队/带宽控制**。

### 3.3 在工程中的配合方式

在实际系统（尤其是 HFT/云网络）中：

- iptables / nftables 常用于：
  - 防火墙 / ACL；
  - NAT（SNAT/DNAT/端口映射）；
  - 连接追踪（stateful firewall）；
- tc 常用于：
  - 带宽整形（限制某条线路或某类业务的速率）；
  - 按业务/队列区分优先级，保证某些流低延迟；
  - 收集统计（使用 TC-BPF 计数、打标签）。

---

## 4. 在内核路径上的直观位置图

从收发路径角度，可以用这样一张“逻辑图”来记：

### 收包方向（ingress）

- NIC → 驱动 / NAPI → netif_receive_skb  
  - （可选）tc ingress qdisc（clsact/ingress）  
  - ip_rcv / ip_local_deliver  
    - netfilter PREROUTING / INPUT / FORWARD 链（iptables）  
  - 进入 socket → 应用

### 发包方向（egress）

- 应用 send/write → tcp_sendmsg / udp_sendmsg  
  - ip_queue_xmit / ip_local_out  
  - netfilter OUTPUT 链（iptables）  
  - 路由 / 选设备  
  - netfilter POSTROUTING 链（iptables）  
  - dev_queue_xmit  
    - qdisc / tc egress（排队、整形）  
  - 驱动 → NIC

**结论：**

- iptables/netfilter：挂在 IP 层几处关键 hook 上，主要负责“安全策略 / NAT / 过滤”。  
- tc/qdisc：挂在设备发送/接收前后（主要是 egress），负责“排队 / 整形 / 队列选择”。

两者在路径上前后相邻，但关注点完全不同：一个偏“逻辑/安全”，一个偏“排队/性能”。  
