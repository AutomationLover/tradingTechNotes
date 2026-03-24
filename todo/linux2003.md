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
