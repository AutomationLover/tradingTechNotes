通常某种组合比较典型：

- AF_INET + SOCK_STREAM + 0 → TCP；

- AF_INET + SOCK_DGRAM + 0 → UDP；

- AF_INET + SOCK_RAW + IPPROTO_ICMP → 原始 ICMP；

- AF_UNIX + SOCK_STREAM → Unix 域流 socket；

- AF_UNIX + SOCK_DGRAM → Unix 域 datagram socket；

- AF_UNIX + SOCK_SEQPACKET → Unix 域 seqpacket。


在你当前的学习/工作场景里（Linux 内核网络、eBPF、HFT）最常用的组合：

- IPv4 TCP：

 ▫ ‎`socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)`（或第三个参数 0）

- IPv4 UDP（单播/多播行情）：

 ▫ ‎`socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)`（或 0）

- AF_PACKET 抓包（tcpdump 那类工具的底层）：

 ▫ ‎`socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL))`

- Unix 域本地通信（低延迟本机 IPC）：

 ▫ ‎`socket(AF_UNIX, SOCK_STREAM, 0)`
