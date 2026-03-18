# Linux Memory Management — Key Messages & Skills Summary

A concise summary of the main concepts, mechanisms, and skills covered in this Linux memory lesson.

https://www.udemy.com/course/memory-management-in-linux-kernel/learn/lecture/22162050#overview

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
