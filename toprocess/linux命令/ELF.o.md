# ELF In Deep — Lesson Summary

A comprehensive summary of the ELF (Executable and Linkable Format) course covering binary structure, compilation, sections, segments, and practical tools.

https://www.udemy.com/course/elf-in-deep/learn/lecture/42675682#overview

---

# ELF — 关键知识点总结

> 基于 “ELF In Deep” 笔记做的精简版复盘，偏概念 + 实战工具。

---

## 1. ELF 是什么，有什么用？

ELF（Executable and Linkable Format）是一种通用二进制文件格式，用在：

- 可执行文件（binaries）：`execve()` 用它构造进程地址空间；
- 动态库（`.so`）：运行时由动态链接器加载；
- 目标文件（`.o`）：链接器的输入；
- 核心转储（core dump）：程序崩溃时的内存快照；
- 内核模块：由内核加载。

主要特性：

- 支持大端/小端，32/64 位架构；
- 支持动态链接、共享库；
- 支持位置无关代码（PIC）。

其他系统的对应格式：macOS 用 Mach-O；Windows 用 PE（Portable Executable）。

---

## 2. 从源码到 ELF：编译流水线

完整流水线：

Source(.c) → 预处理(`gcc -E`) → 编译成汇编(`cc1`) → 汇编成 .o(`as`) → 链接成 ELF(`ld`)

常用命令：

- `gcc userspace.c -o userspace`：一把梭生成用户态可执行；
- `objdump -d -M intel userspace.o`：反汇编 `.o`；
- `file ./userspace`：识别文件类型。

在 bare-metal 场景中：

- 链接器先产出 ELF；
- 再用 `objcopy -O binary add.elf add.bin` 转成“纯字节流”镜像（从指定物理地址开始的连续字节）。

---

## 3. Bare-metal 场景关键点

以 ARM + QEMU 为例：

- 使用 `gcc-arm-none-eabi` 工具链 + `qemu-system-arm` 做实验；
- 链接脚本控制 `.text` `.data` `.bss` `.rodata` 在 Flash / RAM 中的具体布局：
  - `.text`、`.rodata` 通常在 Flash；
  - `.data`、`.bss` 在 RAM，`.bss` 由启动代码清零；
- 多文件程序中，跨文件符号通过：
  - 汇编器先标记为“未解析”；
  - 链接时统一解析；
  - `.global` 让标签对其他文件可见（C 里的 `extern` 等价物）。

这部分让你看到：**ELF + 链接脚本 + 启动代码** 一起决定了裸机环境里代码/数据实际落在什么物理地址。

---

## 4. ELF 整体结构：Header + Sections + Segments

一个 ELF 文件从高层看，有四个部分：

1. ELF 头（ELF header）：全局信息（魔数、位宽、端序、文件类型、入口地址等）；
2. Section Header Table：节头表，按 **section 视角** 给链接器用；
3. Program Header Table：程序头表，按 **segment 视角** 给 loader/内核用；
4. 对应各个 section 的实际内容（代码、数据、符号等）。

关键区别：

- **链接期**：主要看 section 视图（如何合并 `.o`、做重定位、生成可执行/so）；
- **运行期**：主要看 segment 视图（如何映射 PT_LOAD 段到进程虚拟内存）。

---

## 5. ELF Header 关键信息

通过 `readelf -h` 可以看到 ELF 头，几个重要字段：

- `e_ident`（16 字节）：
  - magic：`7f 45 4c 46`（`0x7f + "ELF"`）；
  - class：32 位（1）/64 位（2）；
  - data：小端（1）/大端（2）；
  - OS/ABI：系统 ABI 扩展；
- `e_type` 文件类型：
  - `REL`：可重定位对象（.o）；
  - `EXEC`：可执行文件；
  - `DYN`：共享对象（.so/PIE）；
  - `CORE`：core dump；
- `e_machine`：目标架构（ARM/x86 等）；
- `e_entry`：入口地址（入口指令虚拟地址）；
- `e_phoff` / `e_phnum`：program header 表的偏移和条数；
- `e_shoff` / `e_shnum`：section header 表的偏移和条数。

辅助工具：`hexdump -C`、`file`。

---

## 6. Sections：链接视角

sections 是链接器的工作单元，用来描述“代码/数据/符号”的分类。

常见 section：

- `.text`：
  - 机器码；
  - 类型：`SHT_PROGBITS`；
  - 标志：`SHF_EXECINSTR | SHF_ALLOC`。
- `.data`：
  - 已初始化的全局/静态变量；
  - `SHT_PROGBITS` + `SHF_ALLOC | SHF_WRITE`。
- `.bss`：
  - 未初始化数据（运行时清零）；
  - 类型 `SHT_NOBITS`（文件中不占空间）+ 可写+可装载。
- `.rodata`：
  - 只读常量（字符串字面量等）；
  - 只读+可装载。
- `.symtab` / `.dynsym`：
  - 符号表：前者是“全符号”，后者是动态链接用的子集；
- `.strtab` / `.dynstr`：
  - 符号名字符串表；
- `.shstrtab`：
  - section 名称字符串表（section header 的 `sh_name` 从这里索引）；
- `.debug_*`：
  - DWARF 调试信息；
- 以及 `.init`、`.fini`、`.interp`、`.plt`、`.got` 等专用节。

节头表（section header table）中：

- 每条记录描述一个节的名字、类型、flags、文件偏移、大小等；
- `e_shoff` 指向表起始；`e_shnum` 是表项数量。

常用命令：

- `readelf -S <file>`：列出所有节；
- `readelf -x .section <file>`：dump 某个具体节。

---

## 7. 与 Sections 相关的实用工具

- `size <file>`：显示 `.text` / `.data` / `.bss` / 总大小；
- `strings -d <file>`：打印数据段里的可打印字符串；
- `nm <file>` / `nm -D <file>`：
  - 列出符号（静态 / 动态）；
- `strip <file>`：
  - 去掉符号/调试信息，减小体积但不利调试；
- `addr2line -e <elf> <addr>`：
  - 把运行时地址映射回“源文件:行号”；
- `sstrip <file>`：
  - 粗暴去掉 section header，只保留程序头；程序还能跑，但调试器/objdump 很难用。

---

## 8. Segments：加载视角（Program Header Table）

Program Header Table 描述“段”（segment）：

- 每个 entry 描述一个 segment 对应的文件区间与内存映射；
- 由 loader/内核在 `execve` 时使用。

常见 `p_type`：

- `PT_LOAD`：
  - 可装载段（代码和数据），要映射进进程地址空间；
- `PT_PHDR`：
  - 程序头自身位置；
- `PT_INTERP`：
  - 动态链接器路径（例如 `/lib64/ld-linux-x86-64.so.2`）；
- `PT_NOTE`：
  - 额外注释信息（不一定加载）。

关键字段：

- `p_offset`：该段在文件中的偏移；
- `p_vaddr`：该段映射的虚拟地址；
- `p_filesz`：文件中大小；
- `p_memsz`：映射后内存大小（`memsz > filesz` 的部分就是 BSS）；
- `p_flags`：读/写/执行权限（`PF_R`、`PF_W`、`PF_X`）。

查看命令：`readelf -l <file>`。

---

## 9. Section vs Segment：两种视角

- **Linker 视角（sections）**：
  - 合并 `.o` 的 `.text/.data`；
  - 生成/合并符号表；
  - 重定位。
- **Loader 视角（segments）**：
  - 只关心 `PT_LOAD` 段把哪些字节映射到进程虚拟地址空间；
  - 每个 segment 往往包含多个 section，例如：
    - 一个可执行段包含 `.text + .rodata`；
    - 一个可写段包含 `.data + .bss`。

不是所有 section 都映射进内存，比如 `.debug_*`、`.symtab` 只给工具用，不给运行时用。

内核加载 ELF（`fs/binfmt_elf.c`）的大致步骤：

1. 读取 ELF header + program header 表；
2. 映射所有 `PT_LOAD` 段到新进程虚拟空间；
3. 若存在 `PT_INTERP`，启动动态链接器（ld-linux）；
4. 最终跳转到入口点（可执行文件或解释器入口）。

---

## 10. `_start`、`main` 与启动流程

用户看到的 `main()` 并不是真正的 ELF 入口。真正入口是 `_start`：

- `_start` 通常从 `crt1.o` 提供；
- 在 ELF header 中 `e_entry` 指向 `_start`；
- `_start` 会调用 `__libc_start_main()`，后者负责：
  - 安全/uid 检查；
  - 调用全局构造函数；
  - 调用 `main()`；
  - 调用 `exit()` 结束进程。

典型流程：

`execve()` → 内核解析 ELF、建立地址空间 → `start_thread` 把 PC 设为 `_start` → `_start` → `__libc_start_main(main, ...)` → `main()` → `exit()`。

构建方式：

- `gcc -o a a.c`：动态链接可执行；
- `gcc -static -o a a.c`：静态可执行（所有依赖打包进 ELF）；
- `-nostartfiles`、`-nostdlib`：
  - 禁用默认运行时，自己提供 `_start`（常见于内核、bootloader、裸机固件）。

---

## 11. Minimal ELF：运行 vs 调试

对“能否运行”来说：

- 内核加载 ELF **只需要 ELF 头 + Program Header Table**；
- Section Header Table 完全可以删除（`sstrip` 的做法），程序仍然可以运行。

但对“能否友好调试/分析”来说：

- `gdb`、`objdump` 等工具大量依赖 section header 和调试节；
- 一旦 section headers 被删，只剩 segment，运行 OK，但 debug 极其困难。

这点非常关键：  
**Loader 看的是 program header（segments），  
Linker/调试器看的是 section header（sections）。**

---

## 12. ELF 常用命令速查表

Inspection（结构查看）：

- `readelf -h <file>`：ELF 头；
- `readelf -S <file>`：节列表；
- `readelf -l <file>`：段列表；
- `readelf -n <file>`：note；
- `readelf -p .interp <file>`：动态链接器路径；
- `file <file>`：识别 ELF 类型和架构。

Symbols & Disasm：

- `nm <file>` / `nm -D <file>`：符号；
- `objdump -d <file>`：反汇编；
- `objdump -s -j .text <file>`：dump `.text`。

Conversion & Manipulation：

- `objcopy -O binary in.elf out.bin`：ELF → 裸 bin；
- `strip <file>`：去掉符号/调试信息；
- `addr2line -e <elf> <addr>`：地址 → 源代码行。

---

这一整套内容，把 ELF 从“格式名词”变成了一条完整链路：

- 编译/链接如何生产 ELF；
- ELF 内部结构（header、sections、segments）怎么组织；
- 内核如何用 segments 加载可执行；
- 工具链（readelf/objdump/nm/addr2line/strip/objcopy）如何借助 sections 做调试和分析。

---

## 1. Introduction

### What is ELF?

**ELF** stands for **Executable and Linkable Format**. It is a versatile file format used for:

| Type | Description |
|------|-------------|
| **Binaries** | Output of the linker; used by `exec` to create a process memory image |
| **Libraries** | `.so` files used by dynamic linkers at runtime |
| **Core dumps** | Created when a program terminates with a fault |
| **Object code** | `.o` files targeted by the linker to create executables |
| **Kernel modules** | Loaded by the kernel |

**Key features:**
- Supports big-endian and little-endian
- 32-bit and 64-bit architectures
- Dynamic linking (shared libraries at runtime)
- Position-independent code (PIC)

**Other OS formats:** macOS uses Mach-O; Windows uses Portable Executable (PE).

---

### Why Learn ELF?

- **OS understanding** — Inner workings of how operating systems load and run programs
- **Software development** — Debugging when things go wrong
- **DFIR** — Digital forensics and incident response after security breaches
- **Malware research** — Binary analysis and reverse engineering

---

### Compilation Process

1. **Pre-processing** — `gcc -E`
2. **Compiling** — Assembly file creation (`cc1`)
3. **Assembling** — Object file creation (`as`)
4. **Linking** — Combines object files into executable (`ld`)

```
Source (.c) → Preprocessor → Compiler → Assembler → Linker → Executable
```

**Key commands:**
- `gcc userspace.c -o userspace` — Full compilation
- `objdump -d -M intel userspace.o` — Disassemble object file
- `file ./userspace` — Identify file type

---

## 2. Bare Metal Programming

### ARM Lab Setup

- **QEMU** — Emulates ARM machines (e.g., Connex/Gumstix board)
- **Toolchain:** `gcc-arm-none-eabi`, `qemu-system-arm`

### Opcodes & Machine Code

- **Opcodes** — Machine language instructions specifying operations
- **Instruction** = opcode + operand(s)
- Machine code can be written via: switches, hex editor, assembler, or high-level language

### Assembly Basics

- Format: `label: instruction @ comment`
- Labels reference memory locations (e.g., for branch instructions)
- Assembler directives start with `.` (period)

### Generating Binary Files

- Linker output is ELF; for bare metal, convert to raw binary
- `objcopy -O binary add.elf add.bin` — Converts ELF to consecutive bytes
- Binary format: consecutive bytes from a specific memory address

### Symbol Resolution

- Multi-file programs: assembler marks cross-file references as "unresolved"
- Linker resolves these when combining object files
- `.global` directive makes labels visible to other files (like `extern` in C)

### Data in RAM & Startup Code

- Linker scripts control section placement (`.text`, `.data`, `.bss`, `.rodata`)
- `.bss` placed after `.data` in RAM; symbols mark start/end
- `.rodata` placed after `.text` in Flash

---

## 3. ELF Structure

### Four Main Parts

1. **ELF header** — Always present; conveys basic binary info
2. **Section header table** — Sections for linker
3. **Program header table** — Segments for loader
4. **Object code** — Actual instructions and data

**Link time:** Section headers used to create executable  
**Runtime:** Program headers used to load into memory

---

### ELF Header

**Key information:**
- Target processor (ARM, x86, etc.)
- File type (executable, shared library, relocatable)
- Entry point (address of first instruction)

**e_ident (16 bytes):**
- **Magic:** `7f 45 4c 46` (ELF)
- **Class:** 32-bit (01) or 64-bit (02)
- **Data:** Little-endian (01) or big-endian (02)
- **Version:** Currently 01
- **OS/ABI:** OS-specific extensions
- **ABI version, padding**

**Type (e_type):**
- `REL` (1) — Relocatable (object file)
- `EXEC` (2) — Executable
- `DYN` (3) — Shared object
- `CORE` (4) — Core dump

**Other fields:** `e_machine`, `e_entry`, `e_phoff`, `e_shoff`, `e_phnum`, `e_shnum`, etc.

**Commands:** `readelf -h <file>`, `hexdump -C`, `file <file>`

---

### Building Executables

| Type | Command |
|------|---------|
| Dynamic executable | `gcc -o output source.c` |
| Static executable | `gcc -static -o output source.c` |
| Shared library | `gcc -fPIC -c -o lib.o lib.c` then `ld -shared -soname lib.so.1 -o lib.so.1.0 -lc lib.o` |
| Core dump | `ulimit -c unlimited` |

---

### _start and Program Entry

- **`_start`** — OS entry point (from `crt1.o`)
- **`main`** — Programmer's entry point
- `_start` calls `__libc_start_main()` which:
  - Checks uid/security
  - Runs constructors/destructors
  - Calls `main()`
  - Calls `exit()` with return value

**Flow:** `execve()` → kernel parses ELF → creates process → `start_thread` → userspace at `_start`

**Options:** `-nostartfiles`, `-nostdlib` — For kernels/firmware without standard runtime

---

## 4. ELF Sections

### Sections vs Segments

| View | Used by | Purpose |
|------|---------|---------|
| **Sections** | Linker | Merge code/data, relocation, symbol tables |
| **Segments** | Loader | Load code/data into process memory |

---

### Common Sections

| Section | Content | Type | Flags |
|---------|---------|------|-------|
| **.text** | Executable machine code | SHT_PROGBITS | SHF_EXECINSTR, SHF_ALLOC |
| **.data** | Initialized global/static variables | SHT_PROGBITS | SHF_ALLOC, SHF_WRITE |
| **.bss** | Uninitialized data (zeroed at runtime) | SHT_NOBITS | SHF_ALLOC, SHF_WRITE |
| **.rodata** | Read-only data (e.g., string literals) | SHT_PROGBITS | SHF_ALLOC |
| **.symtab** | Symbol table (all symbols) | — | — |
| **.dynsym** | Dynamic linking symbols | — | ALLOC (loaded at runtime) |
| **.strtab** / **.dynstr** | String tables for symbol names | SHT_STRTAB | — |
| **.shstrtab** | Section header string table (section names) | SHT_STRTAB | — |

**Other sections:** `.init`, `.fini`, `.interp`, `.plt`, `.got`, `.debug_*`

---

### Section Header Table

- Each section has: name, type, flags, size, offset
- `e_shoff` — Offset of section header table
- `e_shnum` — Number of entries
- Section names stored in `.shstrtab` (indexed by `sh_name`)

**Commands:** `readelf -S`, `readelf -x .section_name`

---

### Tools for Sections

| Tool | Purpose |
|------|---------|
| `size` | List section sizes and total |
| `strings -d` | Print strings from data section |
| `nm` | List symbols (`nm -D` for dynamic) |
| `strip` | Remove symbols (reduces size, hinders debugging) |
| `addr2line` | Map addresses to source file:line |
| `sstrip` | Remove section headers (keeps program header; breaks gdb/objdump) |

---

### Debugging & Notes

- **DWARF** — Debug info format (`.debug_*` sections) with `-g`
- **Note sections:** `.note.gnu.build-id`, `.note.gnu.property`, `.note.ABI-tag`
- **dumpelf** — Dump ELF structures as C structs (`pax-utils` package)

---

## 5. ELF Segments

### Program Header Table

- Describes how to load the executable into memory
- Located after ELF header (`e_phoff`)
- One entry per segment

**Key segment types (p_type):**
- **PT_LOAD** — Load into memory
- **PT_PHDR** — Program header location
- **PT_INTERP** — Interpreter path (e.g., dynamic linker)
- **PT_NOTE** — Informational (not necessarily loaded)

**Flags (p_flags):** `PF_R` (read), `PF_W` (write), `PF_X` (execute)

**Fields:** `p_offset`, `p_vaddr`, `p_filesz`, `p_memsz` (BSS when memsz > filesz)

---

### Section vs Segment View

- **Loader** uses segment view (program headers)
- **Linker** uses section view (section headers)
- Segments map multiple sections; not all sections go into segments

**Kernel loading (fs/binfmt_elf.c):**
1. Load ELF header and program header table
2. Load PT_LOAD segments
3. Check for interpreter (dynamic linker)
4. Transfer control to executable or interpreter

---

## 6. Minimal ELF

- Section headers are **optional** for execution
- Only program header table is required for OS to load and run
- `sstrip` removes section headers; binary still runs
- Tools like `gdb` and `objdump` need section headers for full functionality

---

## Quick Reference — Useful Commands

```bash
# Inspection
readelf -h <file>      # ELF header
readelf -S <file>      # Sections
readelf -l <file>      # Segments
readelf -n <file>      # Notes
readelf -p .interp      # Interpreter path
file <file>            # File type

# Symbols & disassembly
nm <file>              # Symbols
objdump -d <file>      # Disassemble
objdump -s -j .text    # Dump section

# Conversion & manipulation
objcopy -O binary in.elf out.bin   # ELF → raw binary
strip <file>           # Remove symbols
addr2line -e <elf> <addr>  # Address to source line
```

---

*Summary generated from ELFInDeep course materials.*
