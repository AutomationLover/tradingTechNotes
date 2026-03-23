# 从 NAPI poll 到 socket：完整路径说明

问题是：**NAPI poll 之后，数据包是如何一路到达 socket 的？**

简化回答：**NAPI poll 从网卡队列拉出包后，把每个包包装成 skb 交给协议栈（`netif_receive_skb` → `ip_rcv` → `tcp_v4_rcv` / `udp_rcv`），协议栈再把数据放进对应 socket 的接收队列，最后应用通过 `recv()` / `read()` 从 socket 里把数据读出来。**

下面按顺序拆开。

---

## 1. 从 NAPI poll 到协议栈入口

在 `net_rx_action()` 里调用某个 NAPI 实例的 `poll()` 时，驱动实现的 `poll()` 回调（如 `xxx_napi_poll`）会做：

1. **从 RX ring 批量取包**
   - NAPI poll 从 NIC 的 RX ring / completion 队列里，取出已经 DMA 好的数据包描述符。

2. **构造 `sk_buff`（skb）**
   - 为每个数据包分配或复用一个 `struct sk_buff`；
   - - 对每个从网卡队列里拿到的包，我们需要一个 skb 来描述它。
在简单实现里，每个包都新分配一个 skb 和数据缓冲区；
在高性能实现里，会尽可能从事先准备好的 skb/页池里复用对象，
只在必须的时候（旧对象还在被用、格式不对等）才真的向内核要新的内存。
   - 设置数据指针、长度、协议相关的 metadata（校验标志、队列号等）。
  





3. **把 skb 交给协议栈入口**
   - 如果启用 GRO：
     - 调用类似 `napi_gro_receive(napi, skb)`，让 GRO 尝试将同一流的多个 TCP 报文聚合；
   - 或直接上送：
     - 调用 `netif_receive_skb(skb)` / `netif_receive_skb_core(skb)`。

到这一步为止：  
**NAPI poll 已经完成“从网卡队列搬运到通用网络栈”的工作，包进入协议栈处理路径。**

---

## 2. `netif_receive_skb` → IP 层 → TCP/UDP 层

从 `netif_receive_skb` 开始，就是 Linux 标准协议栈：

1. **L2 处理与分发：`netif_receive_skb` / `__netif_receive_skb_core`**
   - 解析二层头（以太网），通过 `eth_type_trans` 判断上层协议类型：
     - IPv4 / IPv6 / ARP 等；
   - 根据 `skb->protocol` 把 skb 分发到相应协议处理函数。

2. **IPv4 例子：`ip_rcv`**
   - 进入 `ip_rcv(skb)`：
     - 做版本、头长、校验和、TTL 等检查；
     - 进行路由查找：决定本机处理还是转发。
   - 对于本机地址：
     - 走 `ip_local_deliver(skb)`；
     - 最终到 `ip_local_deliver_finish(skb)`，根据 `skb->protocol` 再分发给上层协议（TCP/UDP 等）。

3. **分发到 TCP 或 UDP：**

   - **TCP：`tcp_v4_rcv(skb)`**
     - 根据四元组（源/目的 IP + 源/目的端口）在哈希表中查找对应 `struct sock`
