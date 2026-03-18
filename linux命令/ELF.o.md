# ELF In Deep — Lesson Summary

A comprehensive summary of the ELF (Executable and Linkable Format) course covering binary structure, compilation, sections, segments, and practical tools.

https://www.udemy.com/course/elf-in-deep/learn/lecture/42675682#overview

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
