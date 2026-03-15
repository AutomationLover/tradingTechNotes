通常某种组合比较典型：

- AF_INET + SOCK_STREAM + 0 → TCP；

- AF_INET + SOCK_DGRAM + 0 → UDP；

- AF_INET + SOCK_RAW + IPPROTO_ICMP → 原始 ICMP；

- AF_UNIX + SOCK_STREAM → Unix 域流 socket；

- AF_UNIX + SOCK_DGRAM → Unix 域 datagram socket；

- AF_UNIX + SOCK_SEQPACKET → Unix 域 seqpacket。
