# BrackOS x86-64 Kernel

BrackOS is an experimental, freestanding x86-64 operating-system kernel written primarily in C, with assembly for boot and interrupt entry. It boots through GRUB using the Multiboot2 protocol and provides a command-line shell, basic graphics, storage, and a small user-process environment.

## What’s included

- **Boot and CPU setup:** Multiboot2 entry code switches the processor from 32-bit startup into 64-bit long mode. The kernel initializes its own GDT, TSS, interrupt table, and page tables.
- **Memory management:** Physical page-frame allocation, a kernel heap, virtual address spaces for user processes, and page-level protections. The kernel sets up protections including a null-page guard, a kernel stack guard page, and write/execute separation where supported.
- **Interrupts and devices:** Keyboard and mouse input, a periodic timer, CMOS real-time clock, serial output, ATA disk I/O, and optional APIC interrupt-controller support discovered through ACPI tables.
- **Console and graphics:** A VGA-style text console, mirrored to COM1 when available. If GRUB supplies a usable 32-bit framebuffer, the console can use it; framebuffer drawing, a pointer cursor, and simple windows are also provided.
- **Shell and utilities:** Commands for system information, time and memory reporting, input history, process management, filesystem operations, and launching programs.
- **User programs:** The kernel can load x86-64 ELF executables, including static PIE images with relative relocations. Programs can be embedded in the kernel or loaded from storage. The syscall layer provides a subset of Linux x86-64-style calls for console I/O, files, memory, process creation, and timing.
- **Recovery:** Boot diagnostics and a limited safe-mode display are included.

## Storage

The kernel has two distinct filesystem paths:

- **FAT32 on the first IDE disk (`A:`):** Used by the shell’s `ls`, `cd`, `mkdir`, `nf`, `rm`, `rmdir`, and `paper` commands. The FAT32 implementation includes sector caching and file operations using FAT 8.3 names.
- **BRKF on the second IDE disk (`/`):** BRKF is BrackOS’s own filesystem, implemented as a portable C library and integrated with the kernel. It supports directories, files, links, POSIX-shaped metadata, checksums, and an `fsck` checker. It backs the user-program VFS when mounted.

Use `brkf help` in the shell for the BRKF command list. Formatting the second disk erases it; the shell requires the explicit confirmation `brkf format yes`.

## Building and running

The `x64/Makefile` describes a Linux or WSL2 (Ubuntu/Debian) build environment. Its listed dependencies are:

```sh
sudo apt install build-essential nasm binutils grub-pc-bin grub-common xorriso mtools qemu-system-x86
```

The Makefile defines these targets:

```sh
make          # build kernel.bin
make iso      # create kernel.iso
make run      # create the ISO and start it in QEMU
make clean    # remove generated build artifacts
```

The `run` target launches QEMU with 512 MB of memory and the boot ISO. The source list in the Makefile at this revision appears not to include many modules referenced by `kernel.c` (including process, virtual-memory, syscall, and BRKF modules). Check and update that list if the build reports missing symbols or objects.

## Source layout

| Path | Purpose |
| --- | --- |
| `src/boot.s`, `src/linker.ld` | Multiboot2 entry, CPU-mode transition, initial page tables, and kernel image layout |
| `src/kernel.c` | Kernel initialization, shell command dispatch, and main loops |
| `src/idt.c`, `src/idt_stubs.s`, `src/trap.h` | Exceptions, hardware interrupts, and saved trap frames |
| `src/gdt.c`, `src/paging.c`, `src/pmm.c`, `src/vmm.c`, `src/heap.c` | CPU tables and memory management |
| `src/sched.c`, `src/process.c`, `src/elf.c`, `src/syscall.c`, `src/vfs.c` | Scheduling, ELF loading, user processes, syscalls, and the user-facing VFS |
| `src/blockdev.c`, `src/fat32.c`, `src/fs_shell.c`, `src/paper.c` | IDE block devices, FAT32, shell filesystem commands, and the text editor |
| `brkf/brkf.c`, `brkf/brkf.h`, `brkf/brkf_port.h` | Portable BRKF filesystem implementation and kernel integration interface |
| `brkf/brkf_specs.txt` | BRKF on-disk format specification |
| `src/fb.c`, `src/font.c`, `src/window.c`, `src/cursor.c`, `src/gui_commands.c` | Framebuffer drawing and basic GUI components |
| `src/keyboard.c`, `src/mouse.c`, `src/input.c` | Input drivers and the shared input-event queue |
| `src/acpi.c`, `src/apic.c`, `src/timer.c`, `src/rtc.c`, `src/serial.c` | Platform, timer, clock, and diagnostic support |
| `grub/grub.cfg` | GRUB menu entry for the kernel |

## Scope

BrackOS is an experimental kernel, not a general-purpose desktop operating system. Hardware support and the syscall and filesystem interfaces are deliberately limited. The BRKF specification explicitly lists journaling, compression, encryption, and extents as outside the scope of its version 1 format.
