用户态通过文件描述符访问 BPF map（用 ‎`bpf()` 或 libbpf/bcc 帮你封装），map 则在 BPF 程序里用 C 结构定义，由内核分配内存。

真正的数据存放并不是在你的进程地址空间，而是在：

- 内核为这个 map 对象单独分配的内存（比如 kmalloc、vmalloc、per-CPU 区等）；

- 你只能通过 map 的 fd + syscalls 间接访问，不能直接拿到裸指针。

