用户态通过文件描述符访问 BPF map（用 ‎`bpf()` 或 libbpf/bcc 帮你封装），map 则在 BPF 程序里用 C 结构定义，由内核分配内存。
