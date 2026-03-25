# Linux Memory Management — Key Messages & Skills Summary

A concise summary of the main concepts, mechanisms, and skills covered in this Linux memory lesson.

https://www.udemy.com/course/memory-management-in-linux-kernel/learn/lecture/22162050#overview

---
# Linux 内存管理 — 知识要点总结

> 基于 Linux Memory Management 课程笔记的精简整理，偏概念与实战技能。

## 1. 物理地址空间（Physical Address Space）

- 物理地址空间是 CPU 能访问的全部地址范围。  
- 32 位系统：2³² = 4GB，可用于 RAM、BIOS、APIC、PCI、MMIO 设备等。  
- 实际布局包括：低 1MB、BIOS ROM、VGA、扩展 ROM、扩展内存等。  
- 调试命令：`cat /proc/iomem` 查看当前物理内存/设备映射。

## 2. 虚拟地址空间（Virtual Address Space）

- Linux 中所有“地址”本质上都是虚拟地址，真正访问物理内存要经过 MMU 翻译。  
- 每个进程都有独立的虚拟地址空间，用于隔离。  
- 32 位经典 3G/1G 划分：  
  - 0–3GB：用户态  
  - 3–4GB：内核态  
  - 内核通过 `PAGE_OFFSET`（如 x86 的 `0xC0000000`）来定义分界。  
- 64 位下常见布局：128TB 用户空间 + 128TB 内核空间，中间是非规范地址区域。  
- 内核与用户共享同一地址空间（高 1GB 或高 128TB）是为了避免每次系统调用都切换页表。

## 3. 页与页表（Pages & Page Tables）

- x86/ARM 默认页大小为 4KB，由 `PAGE_SIZE` 定义。  
- 虚拟页（Virtual Page）：虚拟地址空间被划分成固定大小的页，每一页用 `struct page` 表示。  
- 物理页帧（Page Frame）：物理内存也按页划分，用 PFN（Page Frame Number）标识。  
- 页表：负责虚拟页到物理页帧的映射。  
- 常用宏：`page_to_pfn()`、`pfn_to_page()`。  
- `struct page` 关键字段：  
  - `flags`：页状态（脏、锁定等）；  
  - `_count`：引用计数（空闲时为 -1）；  
  - `virtual`：对应虚拟地址（只适用于低端内存等场景）。  

## 4. 虚拟/物理地址转换（virt_to_phys / phys_to_virt）

- 头文件：`<asm/io.h>`。  
- 内核代码可以用：  
  - `phys_addr = virt_to_phys(virt_addr)`  
  - `virt_addr = phys_to_virt(phys_addr)`  
- 仅限内核虚拟地址（例如 `kmalloc()` 返回值），不能对用户态指针使用。  
- 用户空间指针（如 `malloc()` 返回值）必须通过系统调用由内核自行解析映射。

## 5. 用户态内存布局（User Space Memory Layout）

典型从低地址到高地址的布局：

- Text：程序代码段；  
- Initialized Data：已初始化的全局/静态变量，由 `exec` 从文件加载；  
- BSS：未初始化全局/静态变量，由 `exec` 负责清零；  
- Heap：堆，自底向上增长；  
- Stack：栈，自顶向下增长；  
- 命令行参数和环境变量在用户空间顶部附近。  

调试命令：`cat /proc/<pid>/maps` 查看进程虚拟地址空间布局。

## 6. 内核 vs 用户态内存管理

- 用户态使用“需求分页”（demand paging）：  
  - `malloc()` 只是预留虚拟地址空间；  
  - 真正的物理页在首次访问时通过缺页异常（page fault）分配。  
- 内核中的很多分配不是 demand-paged：  
  - 一旦成功 `kmalloc()`，就已经占用真实物理内存；  
  - 内核内存一般不会被换出到磁盘。  

| 维度 | 内核 | 用户态 |
| --- | --- | --- |
| 分配策略 | 立即占用物理内存 | 首次访问时映射物理页 |
| 换出 | 一般不被换出 | 可被换出到 swap |
| 缺页方式 | 通常不走 demand paging | 依赖缺页机制 |

## 7. 缺页异常（Page Fault）

- 触发条件：访问的虚拟地址尚未映射到物理页。  
- Minor fault：只涉及分配物理页并建立映射，不需要磁盘 I/O。  
- Major fault：需要从磁盘（例如 mmap 文件）加载数据，代价高。  
- 可以通过 `getrusage()` 中的 `ru_minflt`/`ru_majflt` 观测进程的缺页行为。

## 8. 内存域（Memory Zones，32 位视角）

- 低端内存（LOWMEM）：前 896MB，在内核启动时固定映射。  
  - “逻辑地址”（logical address）与物理地址差一个固定偏移。  
  - 常用宏：`__pa(address)` / `__va(address)` 做简单加减。  
- 主要 Zone：  
  - `ZONE_DMA`：低于 16MB，为老式 DMA 设备保留；  
  - `ZONE_NORMAL`：16MB–896MB，普通内核用途；  
  - `ZONE_HIGHMEM`：大于 896MB，只存在于 32 位系统，需要临时映射才可访问。  
- 高端内存（HIGHMEM）映射开销较大，所以内核倾向使用 LOWMEM 做内核数据结构。

64 位系统由于虚拟地址空间巨大，通常不需要 HIGHMEM 概念，只保留 DMA / DMA32 / Normal。

## 9. 伙伴系统（Buddy Allocator）

- 页分配器的底层算法，以 2^order 页为单位分配（order 0–10，对应 1–1024 页）。  
- 每个 order 一条空闲链表；分配时从对应 order 取块，不够就向上级拆分。  
- 释放时，如果“伙伴”空闲，会自动合并成更大块以减少碎片。  
- `cat /proc/buddyinfo` 可查看各 zone 各 order 的空闲块情况。

## 10. 内存分配分层（整体路径）

从内核模块视角：

- 小块/物理连续：`kmalloc()` → SLAB/SLUB 分配器 → 页分配器（buddy）。  
- 大块/不要求物理连续：`vmalloc()` → 直接走页分配器，建立非连续物理页的连续虚拟映射。

## 11. kmalloc 系列

- 头文件：`<linux/slab.h>`。  
- 原型：`void *kmalloc(size_t size, gfp_t flags)`。  
- 特性：  
  - 分配物理和虚拟上都连续的内存；  
  - 默认从 LOWMEM；  
  - 上限由 `MAX_ORDER` 和页大小决定，大约是 4MB 量级。  

常见 `GFP` 标志：

- `GFP_KERNEL`：普通上下文使用，可睡眠；  
- `GFP_ATOMIC`：中断上下文或持有自旋锁时使用，不可睡眠；  
- `GFP_USER`：用户态映射用途；  
- `GFP_HIGHUSER`：优先从 Highmem；  
- `GFP_DMA`：从 `ZONE_DMA` 分配，可 DMA。  

释放接口：`kfree(const void *ptr)`，只能用于 `kmalloc`/`kzalloc` 等分配的指针，且必须恰好释放一次。  
零初始化：`kzalloc()`。  
扩容：`krealloc()`。  
查询实际分配大小：`ksize()`（仅适用于 kmalloc/SLAB 对象）。  

特殊情况：`kmalloc(0)` 返回 `ZERO_SIZE_PTR`（非 NULL，不可解引用，但可传给 `kfree`）。

## 12. vmalloc

- 保证虚拟地址连续，但物理页不连续。  
- 从 HIGHMEM 区域分配，主要用于较大的内存需求且不依赖物理连续性的场景。  
- 本质是分配多页并建立映射，代价比 kmalloc 高（需要修改页表、刷新 TLB）。  
- 由于涉及睡眠和较重操作，不适合在中断上下文中使用。  
- 范围受 `VMALLOC_START`–`VMALLOC_END` 限制。

## 13. kmalloc vs vmalloc 总结

- kmalloc：物理 + 虚拟连续，适合 DMA/硬件驱动、频繁小块分配、性能敏感路径。  
- vmalloc：虚拟连续、物理不连续，适合需要大块内存且不要求物理连续的内核组件（如大数组、缓冲区）。

## 14. 物理连续内存的利弊

优势：

- 很多设备只能使用物理地址做 DMA，必须物理连续；  
- 对 TLB 友好，大块映射可减少 TLB miss。  

劣势：

- 大块物理连续空间难以在系统运行一段时间后找到，容易导致分配失败或碎片严重。

## 15. 内核栈与栈使用分析

- 每个进程拥有用户栈和内核栈，内核栈大小固定且较小（如 x86_64 上的 16KB）。  
- 中断使用单独的中断栈，而非进程的内核栈，以防栈溢出。  
- 可通过 `CONFIG_FRAME_WARN` 在编译时设置函数栈帧大小告警阈值。  
- `checkstack.pl` 脚本配合 `objdump` 可静态分析某个模块中每个函数的大致栈使用情况（仅静态估计，不处理递归）。

## 16. 实战技能点

- 读懂 `/proc/iomem` 和 `/proc/<pid>/maps`，理解系统和进程的内存布局。  
- 使用 `getconf PAGESIZE` 确认系统页大小。  
- 使用 `cat /proc/buddyinfo` 观察各 zone 各阶空闲情况，辅助分析碎片。  
- 使用 `getrusage()` 的 `ru_minflt` / `ru_majflt` 分析缺页开销。  
- 内核中正确选择 `virt_to_phys()`/`phys_to_virt()` 并避免用于用户地址。  
- 内核模块中根据上下文选择合适的分配接口与 GFP 标志（kmalloc vs vmalloc、GFP_KERNEL vs GFP_ATOMIC）。  
- 使用 `checkstack.pl` 评估函数栈使用，配合 `CONFIG_FRAME_WARN` 控制定点函数的栈深。
---

## 1. Physical Address Space

- **Definition**: The full range of memory addresses the CPU can access.
- **32-bit systems**: 2³² = 4 GB address space.
- **Contents**: RAM, BIOS, APIC, PCI, and other memory-mapped I/O devices.
- **Layout**: Low memory (0–1 MB), BIOS ROM, VGA, expansion ROMs, extended memory.
- **Command**: `cat /proc/iomem` — shows the current memory map for physical devices.

---

## 2. Virtual Address Space

- **Principle**: On Linux, every address is virtual; none points directly to RAM.
- **Translation**: MMU (Memory Management Unit) translates virtual → physical.
- **Per-process**: Each process has its own virtual address space (isolation).
- **3G/1G split** (32-bit): User space (0–3 GB) + kernel space (3–4 GB).
- **`PAGE_OFFSET`**: Kernel config that defines the split (e.g. x86: `0xC0000000`, ARM: `0x80000000`).
- **64-bit**: 128 TB user space, ~16M TB non-canonical, 128 TB kernel space.
- **Why kernel shares address space**: Avoids switching address spaces on every syscall.

---

## 3. Pages and Page Tables

- **Page size**: 4096 bytes (4 KB) on x86/ARM; defined by `PAGE_SIZE`.
- **Virtual page**: Fixed-length block of virtual memory; represented by `struct page`.
- **Page frame**: Fixed-length block of physical memory; identified by PFN (Page Frame Number).
- **Page table**: Maps virtual pages to physical frames.
- **Macros**: `page_to_pfn()`, `pfn_to_page()`.
- **Command**: `getconf PAGESIZE` or `getconf PAGE_SIZE`.

### `struct page` (from `<linux/mmtypes.h>`)

- `flags`: Page status (dirty, locked, etc.).
- `_count`: Reference count; -1 when free.
- `virtual`: Virtual address of the page.

---

## 4. Virtual ↔ Physical Address Translation

- **Header**: `#include <asm/io.h>`
- **Functions**:
  - `phys_addr = virt_to_phys(virt_addr);`
  - `virt_addr = phys_to_virt(phys_addr);`
- **Scope**: Only for kernel virtual addresses (e.g. from `kmalloc`), **not** user-space addresses.
- **User space**: `virt_to_phys()` must not be used on user-space pointers (e.g. from `malloc`).

---

## 5. User Space Memory Layout

Typical layout (low → high):

| Region | Description |
|--------|-------------|
| Text | Program code |
| Initialized data | Read from program file by `exec` |
| BSS (uninitialized data) | Zeroed by `exec` |
| Heap | Grows upward |
| Stack | Grows downward |
| Command-line args & env | Top of user space |

- **Command**: `cat /proc/<pid>/maps` — shows process memory map.

---

## 6. Kernel vs User Memory Management

| Aspect | Kernel | User Space |
|--------|--------|------------|
| Paging | Not demand-paged | Demand-paged (lazy) |
| Allocation | Real physical memory at allocation time | Physical pages mapped on first access |
| Discarding | Never paged out | Can be paged out |

- **User space**: `malloc()` reserves address range; physical pages are allocated on first access (page fault).

---

## 7. Page Faults

- **Trigger**: Access to a page that is not yet mapped.
- **Minor fault**: Kernel allocates a physical page and maps it.
- **Major fault**: Page is backed by a file (e.g. `mmap`); involves disk I/O.
- **Cost**: Major faults are much more expensive than minor faults.
- **Demo**: Use `getrusage()` and `ru_majflt` / `ru_minflt` to observe page faults.

---

## 8. Memory Zones (32-bit)

### Low Memory (LOWMEM) — first 896 MB

- Permanently mapped at boot.
- **Logical addresses**: Virtual addresses that translate to physical by subtracting a fixed offset.
- **Macros**: `__pa(address)`, `__va(address)`.

### Zones

1. **ZONE_DMA**: Below 16 MB — for DMA-capable devices.
2. **ZONE_NORMAL**: 16 MB–896 MB — normal kernel use.
3. **ZONE_HIGHMEM**: Above 896 MB — only on 32-bit.

### High Memory (HIGHMEM) — top 128 MB of kernel space

- Used to temporarily map physical memory above 896 MB.
- Mapping is created and torn down on demand; slower than low memory.
- **64-bit**: No HIGHMEM; address space is large enough.

---

## 9. Memory Zones (64-bit)

- **DMA**: Low 16 MB (legacy).
- **DMA32**: Low 4 GB — for devices that only DMA to low 4 GB.
- **Normal**: RAM above ~4 GB.
- **HighMem**: Not used on 64-bit.

---

## 10. Buddy Allocator

- **Unit**: Blocks of 2^order pages (order 0–10, `MAX_ORDER = 11`).
- **Lists**: 11 free lists, one per order.
- **Allocation**: Take a block from the right list; if empty, split a larger block.
- **Freeing**: Merge buddies when both are free.
- **Command**: `cat /proc/buddyinfo` — shows free blocks per zone and order.

---

## 11. Memory Allocation Hierarchy

```
Kernel Module
     │
     ├── kmalloc ──────► SLAB allocator ──► Page allocator
     │
     └── vmalloc ──────► Page allocator (direct)
              │
              ▼
         Main Memory
```

---

## 12. kmalloc Family

### `kmalloc()`

- **Header**: `#include <linux/slab.h>`
- **Prototype**: `void *kmalloc(size_t size, int flags);`
- **Properties**: Physically and virtually contiguous; from LOWMEM (unless HIGHMEM flag).
- **Max size**: ~4 MB (2²² bytes) on x86/ARM with 4 KB pages and `MAX_ORDER = 11`.
- **Formula**: `KMALLOC_MAX_SIZE = 1UL << (MAX_ORDER + PAGE_SHIFT - 1)`.

### Common flags

| Flag | Use case |
|------|----------|
| `GFP_KERNEL` | Normal context; may sleep — not for interrupt context |
| `GFP_ATOMIC` | Interrupt context; never sleeps |
| `GFP_USER` | User-space allocations |
| `GFP_HIGHUSER` | From HIGHMEM |
| `GFP_DMA` | From ZONE_DMA |

### `kfree()`

- **Prototype**: `void kfree(const void *ptr);`
- **Rules**: Free exactly once; only use on pointers from `kmalloc()`.
- **Leaks**: Kernel memory is never freed automatically.

### Related functions

- **`kzalloc()`**: Like `kmalloc()` but zeroes memory.
- **`krealloc()`**: Resize memory allocated by `kmalloc()`.
- **`ksize()`**: Returns actual allocated size (may be larger than requested).
- **`ksize()` scope**: Only for `kmalloc()` / `kmem_cache_alloc()`; not for `vmalloc()`.

### Special cases

- **`kmalloc(0)`**: Returns `ZERO_SIZE_PTR`; non-NULL but faults on dereference; safe to pass to `kfree()`.

---

## 13. vmalloc

- **Properties**: Virtually contiguous, **not** physically contiguous.
- **Zone**: Always from HIGHMEM.
- **Use**: Large allocations when physical contiguity is not required.
- **Limit**: Up to physical RAM size; bounded by `VMALLOC_START`–`VMALLOC_END`.
- **Overhead**: Higher than `kmalloc` (page table changes, TLB invalidation).
- **Context**: Cannot be used in interrupt context.

---

## 14. kmalloc vs vmalloc

| Aspect | kmalloc | vmalloc |
|--------|---------|---------|
| Physical contiguity | Yes | No |
| Virtual contiguity | Yes | Yes |
| Zone | LOWMEM | HIGHMEM |
| DMA/hardware | Yes | No |
| Interrupt context | Yes (with `GFP_ATOMIC`) | No |
| Allocator | SLAB → Page | Page |
| Overhead | Lower | Higher |
| Typical use | Small, DMA, hardware | Large, no physical contiguity |

---

## 15. Physically Contiguous Memory

**Advantages**

1. Many devices cannot use virtual addresses.
2. Single large page mapping reduces TLB overhead.

**Disadvantage**

- Hard to find large contiguous blocks; fragmentation.

---

## 16. Kernel Stack

- **Per process**: User stack (user space) + kernel stack (kernel space).
- **Size**: Fixed and small (e.g. 16 KB on x86_64 since Linux 3.15).
- **Reason**: Saves memory; avoids need for two contiguous pages.
- **Interrupts**: Use separate interrupt stacks, not the process kernel stack.

---

## 17. Stack Usage Tools

- **`CONFIG_FRAME_WARN`**: Emits a warning when a function’s stack frame exceeds a threshold (default 1024 bytes).
- **`checkstack.pl`**: Lists stack usage per function (biggest first).
  - Example: `objdump -D hello.ko | perl ~/linux-5.2.8/scripts/checkstack.pl`
  - Limitation: Static analysis only; does not handle recursion.

---

## 18. Practical Skills

1. **Inspect memory layout**: `cat /proc/iomem`, `cat /proc/<pid>/maps`
2. **Check page size**: `getconf PAGESIZE`
3. **Inspect buddy allocator**: `cat /proc/buddyinfo`
4. **Measure page faults**: `getrusage()` with `ru_majflt` and `ru_minflt`
5. **Kernel address translation**: `virt_to_phys()` / `phys_to_virt()` for kernel addresses only
6. **Stack analysis**: `checkstack.pl` on kernel modules

---

## 19. Quick Reference: Header Files

| Purpose | Header |
|---------|--------|
| Address translation | `<asm/io.h>` |
| `struct page` | `<linux/mmtypes.h>` |
| kmalloc/kfree | `<linux/slab.h>` |
| vmalloc | `<linux/vmalloc.h>` |
| vmalloc range | `<asm/pgtable.h>` |

---

*Summary generated from the LinuxMem lesson materials (day12 & day13).*
