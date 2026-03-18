# QEMU RISC-V 64 Simulation Guide on Ubuntu 24.04

> A complete step-by-step guide to simulating a RISC-V 64-bit core using QEMU on Ubuntu 24.04 —
> covering bare-metal firmware development and booting a Linux kernel.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites & System Setup](#prerequisites--system-setup)
3. [Install QEMU with RISC-V Support](#install-qemu-with-risc-v-support)
4. [Install the RISC-V Cross-Compiler Toolchain](#install-the-risc-v-cross-compiler-toolchain)
5. [Part 1 — Bare-Metal Simulation](#part-1--bare-metal-simulation)
   - [Understanding the virt Machine](#understanding-the-virt-machine)
   - [Write a Bare-Metal Program](#write-a-bare-metal-program)
   - [Linker Script](#linker-script)
   - [Compile & Link](#compile--link)
   - [Run on QEMU](#run-on-qemu)
   - [Debugging with GDB](#debugging-with-gdb)
6. [Part 2 — Linux on RISC-V](#part-2--linux-on-risc-v)
   - [Option A: Use a Pre-built Image (Quickstart)](#option-a-use-a-pre-built-image-quickstart)
   - [Option B: Build Linux from Source](#option-b-build-linux-from-source)
   - [Build BusyBox Root Filesystem](#build-busybox-root-filesystem)
   - [Build OpenSBI Firmware](#build-opensbi-firmware)
   - [Run Linux on QEMU](#run-linux-on-qemu)
7. [QEMU Machine Flags Reference](#qemu-machine-flags-reference)
8. [Troubleshooting](#troubleshooting)
9. [Summary Cheatsheet](#summary-cheatsheet)

---

## Overview

| Mode | What runs | Tooling needed |
|---|---|---|
| **Bare-metal** | Your own firmware/ELF directly on virt hardware | Cross-GCC, linker script, QEMU |
| **Linux** | OpenSBI → Linux kernel → BusyBox rootfs | Cross-GCC, Linux source, BusyBox, OpenSBI, QEMU |

QEMU's `qemu-system-riscv64` emulates a **`virt`** machine — a synthetic board with:
- A 64-bit RISC-V hart (hardware thread)
- UART at `0x10000000` (NS16550A compatible)
- CLINT/PLIC interrupt controllers
- VirtIO devices, PCIe, and configurable RAM

---

## Prerequisites & System Setup

### Update your system first

```bash
sudo apt update && sudo apt upgrade -y
```

### Core build dependencies

```bash
sudo apt install -y \
  build-essential \
  git \
  curl \
  wget \
  python3 \
  python3-pip \
  ninja-build \
  pkg-config \
  libglib2.0-dev \
  libpixman-1-dev \
  libslirp-dev \
  flex \
  bison \
  bc \
  cpio \
  unzip \
  rsync \
  file \
  gdb-multiarch
```

> **`gdb-multiarch`** is the GDB binary that supports RISC-V targets on an x86-64 host.

---

## Install QEMU with RISC-V Support

### Option A — APT (simplest, Ubuntu 24.04 ships QEMU 8.x)

```bash
sudo apt install -y qemu-system-misc
```

`qemu-system-misc` includes `qemu-system-riscv64` and `qemu-system-riscv32`.

Verify:

```bash
qemu-system-riscv64 --version
# QEMU emulator version 8.x.x
```

### Option B — Build from source (latest QEMU 9.x)

Use this if you need newer machine features or a more recent QEMU version.

```bash
# Extra deps for source build
sudo apt install -y \
  libglib2.0-dev \
  libfdt-dev \
  libpixman-1-dev \
  zlib1g-dev \
  libslirp-dev \
  libcap-ng-dev \
  libattr1-dev \
  libaio-dev \
  liburing-dev

# Clone and build
git clone https://gitlab.com/qemu-project/qemu.git
cd qemu
git checkout v9.2.0   # or latest stable tag

./configure \
  --target-list=riscv64-softmmu,riscv32-softmmu \
  --enable-slirp \
  --prefix=/usr/local

make -j$(nproc)
sudo make install
```

Verify:

```bash
qemu-system-riscv64 --version
```

---

## Install the RISC-V Cross-Compiler Toolchain

You need a cross-compiler that runs on your x86-64 Ubuntu host but produces RISC-V 64-bit binaries.

### Option A — APT prebuilt (fastest)

```bash
sudo apt install -y \
  gcc-riscv64-linux-gnu \
  g++-riscv64-linux-gnu \
  binutils-riscv64-linux-gnu
```

Check:

```bash
riscv64-linux-gnu-gcc --version
# riscv64-linux-gnu-gcc (Ubuntu ...) 13.x.x
```

> This toolchain targets Linux (with glibc). For bare-metal you can use it with `-ffreestanding` and a custom linker script, or build the bare-metal specific toolchain below.

### Option B — Bare-metal toolchain (newlib / no OS)

```bash
sudo apt install -y gcc-riscv64-unknown-elf
```

If not available via APT, build from the RISC-V GNU toolchain repo:

```bash
git clone https://github.com/riscv-collab/riscv-gnu-toolchain.git
cd riscv-gnu-toolchain

sudo apt install -y autoconf automake autotools-dev curl python3 \
  libmpc-dev libmpfr-dev libgmp-dev gawk texinfo libtool patchutils \
  zlib1g-dev libexpat-dev libglib2.0-dev

./configure --prefix=/opt/riscv --with-arch=rv64gc --with-abi=lp64d
make -j$(nproc)

# Add to PATH
echo 'export PATH=/opt/riscv/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash
riscv64-unknown-elf-gcc --version
```

---

## Part 1 — Bare-Metal Simulation

### Understanding the virt Machine

When QEMU starts with `-machine virt`, the memory map is:

| Address | Device |
|---|---|
| `0x00001000` | Boot ROM (QEMU puts reset vector here) |
| `0x02000000` | CLINT (timer/software interrupts) |
| `0x0c000000` | PLIC (external interrupts) |
| `0x10000000` | UART0 (NS16550A) |
| `0x80000000` | DRAM starts here |

Your bare-metal program must be linked to start at `0x80000000`.

---

### Write a Bare-Metal Program

Create a project directory:

```bash
mkdir -p ~/riscv-baremetal && cd ~/riscv-baremetal
```

#### `start.S` — Minimal assembly startup

```asm
# start.S
.section .text.start
.global _start

_start:
    # Set up stack pointer — top of a 64KB stack we reserve in BSS
    la   sp, stack_top

    # Clear BSS section
    la   t0, _bss_start
    la   t1, _bss_end
clear_bss:
    bge  t0, t1, bss_done
    sd   zero, 0(t0)
    addi t0, t0, 8
    j    clear_bss

bss_done:
    # Jump to C main
    call main

    # Hang if main returns
hang:
    j hang

# Reserve 64 KB stack in BSS
.section .bss
.align 4
stack_bottom:
    .skip 65536
stack_top:
```

#### `uart.h` — UART helper

```c
// uart.h
#ifndef UART_H
#define UART_H

#include <stdint.h>

#define UART0_BASE  0x10000000UL
#define UART_THR    (*(volatile uint8_t *)(UART0_BASE + 0x00))  // Transmit Holding Register
#define UART_LSR    (*(volatile uint8_t *)(UART0_BASE + 0x05))  // Line Status Register
#define UART_LSR_TX_EMPTY  0x20

static inline void uart_putc(char c) {
    while (!(UART_LSR & UART_LSR_TX_EMPTY));
    UART_THR = c;
}

static inline void uart_puts(const char *s) {
    while (*s) uart_putc(*s++);
}

#endif
```

#### `main.c` — Hello World from bare metal

```c
// main.c
#include "uart.h"

void main(void) {
    uart_puts("=== RISC-V 64 Bare-Metal on QEMU ===\r\n");
    uart_puts("Hello from bare metal!\r\n");

    // Count to 5 and print
    for (int i = 1; i <= 5; i++) {
        uart_putc('0' + i);
        uart_putc('\r');
        uart_putc('\n');
    }

    uart_puts("Done. Halting.\r\n");

    // QEMU exit via SiFive test device (MMIO at 0x100000)
    // Write 0x5555 to trigger PASS exit
    volatile uint32_t *qemu_exit = (volatile uint32_t *)0x100000;
    *qemu_exit = 0x5555;

    while (1); // fallback hang
}
```

---

### Linker Script

```ld
/* link.ld */
OUTPUT_ARCH(riscv)
ENTRY(_start)

MEMORY {
    RAM (rwx) : ORIGIN = 0x80000000, LENGTH = 128M
}

SECTIONS {
    . = 0x80000000;

    .text : {
        *(.text.start)     /* startup code first */
        *(.text*)
        *(.rodata*)
    } > RAM

    .data : {
        *(.data*)
    } > RAM

    .bss : {
        _bss_start = .;
        *(.bss*)
        *(COMMON)
        _bss_end = .;
    } > RAM

    . = ALIGN(8);
    _end = .;
}
```

---

### Compile & Link

Using the Linux cross-compiler with `-ffreestanding`:

```bash
CROSS=riscv64-linux-gnu

# Compile assembly startup
${CROSS}-gcc -march=rv64gc -mabi=lp64d \
    -ffreestanding -nostdlib \
    -O0 -g -gdwarf-2 \
    -c start.S -o start.o

# Compile C code
${CROSS}-gcc -march=rv64gc -mabi=lp64d  -mcmodel=medany\
    -ffreestanding -nostdlib -O2 \
    -c main.c -o main.o

# Link
${CROSS}-gcc -march=rv64gc -mabi=lp64d -mcmodel=medany\
    -ffreestanding -nostdlib \
    -T link.ld \
    -o baremetal.elf \
    start.o main.o

# Optional: inspect the ELF
${CROSS}-objdump -d baremetal.elf | head -60
${CROSS}-readelf -h baremetal.elf
```

If using the `riscv64-unknown-elf-gcc` toolchain, replace the prefix accordingly:

```bash
CROSS=riscv64-unknown-elf
```

---

### Run on QEMU

```bash
qemu-system-riscv64 \
  -machine virt \
  -cpu rv64 \
  -m 128M \
  -nographic \
  -bios none \
  -kernel baremetal.elf
```

**Flag explanation:**

| Flag | Meaning |
|---|---|
| `-machine virt` | Use the generic virtual RISC-V board |
| `-cpu rv64` | 64-bit RISC-V hart |
| `-m 128M` | 128 MB of RAM |
| `-nographic` | Redirect UART to your terminal (no GUI) |
| `-bios none` | No OpenSBI — QEMU jumps directly to your ELF |
| `-kernel baremetal.elf` | Your ELF binary (loaded at 0x80000000) |

**Expected output:**

```
=== RISC-V 64 Bare-Metal on QEMU ===
Hello from bare metal!
1
2
3
4
5
Done. Halting.
```

Press `Ctrl+A` then `X` to exit QEMU.

---

### Debugging with GDB

Open two terminals.

**Terminal 1 — Start QEMU in debug mode:**

```bash
qemu-system-riscv64 \
  -machine virt \
  -cpu rv64 \
  -m 128M \
  -nographic \
  -bios none \
  -kernel baremetal.elf \
  -s -S
```

> `-s` opens GDB server on port `1234`. `-S` freezes CPU at startup until GDB connects.

**Terminal 2 — Connect GDB:**

```bash
gdb-multiarch baremetal.elf
```

Inside GDB:

```
(gdb) set architecture riscv:rv64
(gdb) target remote localhost:1234
(gdb) break main
(gdb) continue
(gdb) layout src
(gdb) stepi 
(gdb) info registers
(gdb) x/10i $pc
```
Note: step has a shortcut - "si" to execute one riscv instruction at a time. 

### Useful command for GDB while debugging. 

```

# Step one instruction
(gdb) stepi

# Step one instruction but don't descend into calls (step over)
(gdb) nexti

# Disassemble around current PC
(gdb) disassemble $pc,+40

# Watch a memory address for changes
(gdb) watch *(uint64_t*)0x80001000

# Read a CSR by its address number (e.g. mstatus = 0x300)
(gdb) info registers mstatus

# Print a variable
(gdb) print my_variable

# Hex dump memory (8 words of 8 bytes each)
(gdb) x/8gx 0x80000000

# Enable TUI mode (split: source on top, gdb prompt below)
(gdb) tui enable
(gdb) layout split   # source + disassembly
(gdb) layout regs    # adds register panel

# Refresh TUI if display glitches
(gdb) refresh
```

---

### TUI Layout Reference

Once inside GDB TUI (`tui enable`), cycle through layouts:
```
layout src        → source code only
layout asm        → disassembly only
layout split      → source + disassembly
layout regs       → adds register panel on top
Ctrl+X A          → toggle TUI on/off
Ctrl+L            → redraw if the screen corrupts

---

## Part 2 — Linux on RISC-V

Linux on RISC-V requires three components:

```
┌─────────────────────────────────────────────┐
│  QEMU virt machine                          │
│                                             │
│  ┌─────────────┐  boots  ┌───────────────┐  │
│  │  OpenSBI    │ ──────► │  Linux Kernel │  │
│  │  (M-mode)   │         │  (S-mode)     │  │
│  └─────────────┘         └───────┬───────┘  │
│                                  │ mounts   │
│                              ┌───▼──────┐   │
│                              │ initramfs│   │
│                              │ (BusyBox)│   │
│                              └──────────┘   │
└─────────────────────────────────────────────┘
```

- **OpenSBI** — SBI (Supervisor Binary Interface) firmware, runs in M-mode
- **Linux kernel** — runs in S-mode
- **BusyBox rootfs** — minimal userspace packed as initramfs

---

## Option A: Use a Pre-built Image (Quickstart)

The fastest way to boot Linux on RISC-V QEMU is to use a pre-built Debian/Ubuntu cloud image.

```bash
# Install required tools
sudo apt install -y qemu-system-misc opensbi u-boot-qemu

# Download a pre-built RISC-V Debian image
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-nocloud-riscv64.qcow2

# Boot it
qemu-system-riscv64 \
  -machine virt \
  -cpu rv64 \
  -m 1G \
  -smp 2 \
  -nographic \
  -bios /usr/lib/riscv64-linux-gnu/opensbi/generic/fw_jump.elf \
  -kernel /usr/lib/u-boot/qemu-riscv64_smode/uboot.elf \
  -drive file=debian-12-nocloud-riscv64.qcow2,format=qcow2 \
  -append "root=/dev/vda1 rw console=ttyS0" \
  -device virtio-blk-device,drive=hd0 \
  -drive file=debian-12-nocloud-riscv64.qcow2,if=none,id=hd0 \
  -netdev user,id=net0 \
  -device virtio-net-device,netdev=net0
```

Login with user `root` (no password on nocloud images).

---

## Option B: Build Linux from Source

### Step 1 — Additional build dependencies

```bash
sudo apt install -y \
  libncurses-dev \
  libssl-dev \
  libelf-dev \
  dwarves \
  zstd \
  device-tree-compiler
```

### Step 2 — Set environment variables

```bash
export CROSS_COMPILE=riscv64-linux-gnu-
export ARCH=riscv
```

Add these to `~/.bashrc` to persist:

```bash
echo 'export CROSS_COMPILE=riscv64-linux-gnu-' >> ~/.bashrc
echo 'export ARCH=riscv'                        >> ~/.bashrc
source ~/.bashrc
```

---

### Build BusyBox Root Filesystem

BusyBox provides a minimal shell and userspace utilities.

```bash
mkdir -p ~/riscv-linux && cd ~/riscv-linux

# Clone BusyBox
git clone https://git.busybox.net/busybox --depth=1 --branch=1_36_stable
cd busybox

# Configure for static build (no shared libs needed in initramfs)
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- menuconfig
```

In menuconfig, navigate to:
- **Settings → Build Options → [*] Build static binary (no shared libs)** → Enable

```bash
# Build BusyBox
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- install
# Installs to ./_install/
```

#### Create the initramfs directory structure

```bash
cd ~/riscv-linux
mkdir -p initramfs/{bin,sbin,etc,proc,sys,dev,tmp,usr/bin,usr/sbin}

# Copy BusyBox install tree
cp -a busybox/_install/* initramfs/

# Create init script
cat > initramfs/init << 'EOF'
#!/bin/sh
mount -t proc  none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev 2>/dev/null || mknod /dev/null c 1 3

echo ""
echo "=============================="
echo " RISC-V Linux on QEMU"
echo "=============================="
echo ""

exec /bin/sh
EOF
chmod +x initramfs/init

# Pack as cpio initramfs
cd initramfs
find . | cpio -oH newc | gzip > ../initramfs.cpio.gz
cd ..
echo "initramfs built: $(du -sh initramfs.cpio.gz)"
```

---

### Build OpenSBI Firmware

OpenSBI is the RISC-V SBI implementation that runs in machine mode and passes control to the kernel.

```bash
cd ~/riscv-linux

git clone https://github.com/riscv-software-src/opensbi.git --depth=1
cd opensbi

make CROSS_COMPILE=riscv64-linux-gnu- \
     PLATFORM=generic \
     FW_PIC=y \
     -j$(nproc)

# The firmware ELF we need:
ls build/platform/generic/firmware/fw_jump.elf
```

---

### Build the Linux Kernel

```bash
cd ~/riscv-linux

git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git \
    --depth=1 --branch v6.8
cd linux

# Use the default RISC-V config
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- defconfig

# (Optional) customise
# make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- menuconfig

# Build — this takes 10–30 minutes depending on your CPU
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)
```

Output files:

```
arch/riscv/boot/Image          ← uncompressed kernel image
arch/riscv/boot/Image.gz       ← gzip compressed
```

---

### Run Linux on QEMU

```bash
cd ~/riscv-linux

qemu-system-riscv64 \
  -machine virt \
  -cpu rv64 \
  -m 512M \
  -smp 2 \
  -nographic \
  -bios opensbi/build/platform/generic/firmware/fw_jump.elf \
  -kernel linux/arch/riscv/boot/Image \
  -initrd initramfs.cpio.gz \
  -append "root=/dev/ram rw console=ttyS0 earlycon=sbi"
```

**Flag explanation:**

| Flag | Meaning |
|---|---|
| `-bios fw_jump.elf` | OpenSBI firmware in M-mode |
| `-kernel Image` | Linux kernel in S-mode |
| `-initrd initramfs.cpio.gz` | BusyBox root filesystem |
| `-append "..."` | Kernel command-line arguments |
| `-smp 2` | Simulate 2 harts (CPU cores) |
| `console=ttyS0` | Route kernel console to QEMU's stdio |
| `earlycon=sbi` | Early boot console via SBI calls |

**Expected boot sequence:**

```
OpenSBI v1.x
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 ...
[    0.000000] Linux version 6.8.0 (...)
[    0.000000] Machine model: riscv-virtio,qemu
...
[    0.xxxxxx] Run /init as init process
==============================
 RISC-V Linux on QEMU
==============================

/ #
```

You now have a shell. Try:

```sh
uname -a
cat /proc/cpuinfo
cat /proc/meminfo
ls /
```

Press `Ctrl+A` then `X` to exit QEMU.

---

## QEMU Machine Flags Reference

### CPU variants

```bash
# List all supported RISC-V CPUs
qemu-system-riscv64 -machine virt -cpu help

# Common choices
-cpu rv64                  # Generic RV64GC
-cpu sifive-u54            # SiFive U54 (RV64GC, in-order)
-cpu sifive-u34            # SiFive U34 (RV32GC)
-cpu thead-c906            # T-Head C906
```

### Memory and SMP

```bash
-m 256M                    # 256 MB RAM
-m 1G                      # 1 GB RAM
-smp 1                     # 1 hart
-smp 4                     # 4 harts (cores)
```

### Storage (persistent disk)

```bash
# Create a virtual disk image
qemu-img create -f qcow2 rootdisk.qcow2 8G

# Attach it
-drive file=rootdisk.qcow2,if=none,id=hd0,format=qcow2 \
-device virtio-blk-device,drive=hd0
```

### Networking

```bash
# User-mode NAT (simple, no config needed)
-netdev user,id=net0 \
-device virtio-net-device,netdev=net0

# Host port forwarding (SSH on host port 2222 → guest port 22)
-netdev user,id=net0,hostfwd=tcp::2222-:22 \
-device virtio-net-device,netdev=net0
```

### Serial / UART options

```bash
-nographic                 # UART → stdio (most common for headless)
-serial mon:stdio          # UART + QEMU monitor on stdio
-serial pty                # Create a PTY, useful for scripting
```

### Machine inspection

```bash
# List all virt machine properties
qemu-system-riscv64 -machine virt,help

# Dump device tree blob
qemu-system-riscv64 -machine virt,dumpdtb=virt.dtb -nographic -bios none; \
  dtc -I dtb -O dts virt.dtb -o virt.dts && head -80 virt.dts
```

---

## Troubleshooting

### QEMU exits immediately with no output

- Check you used `-nographic` (without it, QEMU tries to open a GUI window).
- Ensure `-bios none` is set for bare-metal (otherwise QEMU tries to load OpenSBI and fails).
- Verify your ELF entry point is at `0x80000000`:  
  `riscv64-linux-gnu-readelf -h baremetal.elf | grep Entry`

### `Illegal instruction` in bare-metal

- Ensure you compiled with the correct march/mabi matching your CPU:  
  `-march=rv64gc -mabi=lp64d` for the default `rv64` CPU.
- The `virt` machine's default CPU supports the G extension set (IMAFD + Zicsr + Zifencei).

### Linux kernel panics with `VFS: Unable to mount root`

- Make sure your `initramfs.cpio.gz` was built correctly and referenced with `-initrd`.
- Confirm the cpio archive has a valid `/init` executable at the root.

### OpenSBI prints `Domain0 Next Address : 0x0000000000000000`

- This means OpenSBI can't find the kernel. Ensure `-kernel` points to the raw `Image` file, not a compressed `.gz`.

### Cross-compiler not found

```bash
# Verify toolchain is installed
which riscv64-linux-gnu-gcc
# If missing:
sudo apt install gcc-riscv64-linux-gnu
```

### Slow build (Linux kernel)

```bash
# Use all available cores
make -j$(nproc)
# Check core count
nproc
```

---

## Summary Cheatsheet

```bash
# ── INSTALL ──────────────────────────────────────────────────────
sudo apt install -y qemu-system-misc gcc-riscv64-linux-gnu \
  g++-riscv64-linux-gnu binutils-riscv64-linux-gnu gdb-multiarch \
  device-tree-compiler libncurses-dev libssl-dev libelf-dev dwarves

# ── BARE-METAL ───────────────────────────────────────────────────
# Compile
riscv64-linux-gnu-gcc -march=rv64gc -mabi=lp64d \
  -ffreestanding -nostdlib -T link.ld \
  -o baremetal.elf start.S main.c

# Run
qemu-system-riscv64 -machine virt -cpu rv64 -m 128M \
  -nographic -bios none -kernel baremetal.elf

# Debug
qemu-system-riscv64 -machine virt -cpu rv64 -m 128M \
  -nographic -bios none -kernel baremetal.elf -s -S
# (in another terminal)
gdb-multiarch baremetal.elf -ex "target remote :1234"

# ── LINUX ────────────────────────────────────────────────────────
# Run (pre-built OpenSBI + kernel + initramfs)
qemu-system-riscv64 -machine virt -cpu rv64 -m 512M -smp 2 \
  -nographic \
  -bios opensbi/build/platform/generic/firmware/fw_jump.elf \
  -kernel linux/arch/riscv/boot/Image \
  -initrd initramfs.cpio.gz \
  -append "root=/dev/ram rw console=ttyS0 earlycon=sbi"

# Exit QEMU: Ctrl+A then X
```

---

## Further Reading

- [QEMU RISC-V documentation](https://www.qemu.org/docs/master/system/target-riscv.html)
- [RISC-V GNU Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain)
- [OpenSBI repository](https://github.com/riscv-software-src/opensbi)
- [Linux RISC-V architecture docs](https://www.kernel.org/doc/html/latest/arch/riscv/index.html)
- [RISC-V ISA Specification](https://riscv.org/technical/specifications/)
- [BusyBox](https://busybox.net/)

## Appendix

---

## Inspecting Registers in Bare-Metal RISC-V on QEMU

---

### Method 1: GDB via QEMU's Built-in GDB Stub

This gives full register visibility — GPRs, CSRs, and FP registers.

**Launch QEMU in debug mode:**

```bash
qemu-system-riscv64 -machine virt -cpu rv64 -m 128M \
  -nographic -bios none -kernel baremetal.elf \
  -s -S
```

**Connect GDB in a second terminal:**

```bash
gdb-multiarch baremetal.elf
```

**Inside GDB:**

```gdb
(gdb) set architecture riscv:rv64
(gdb) target remote localhost:1234

# All general-purpose registers (x0–x31 + pc)
(gdb) info registers

# A single register
(gdb) info registers sp
(gdb) print $a0

# ALL registers including FP and CSRs
(gdb) info all-registers
```

---

### Method 2: QEMU Monitor

The QEMU monitor is a built-in shell accessible while the VM is running — no GDB needed.

**Launch with a split UART + monitor:**

```bash
qemu-system-riscv64 -machine virt -cpu rv64 -m 128M \
  -nographic -bios none -kernel baremetal.elf \
  -serial mon:stdio
```

Switch to the monitor with `Ctrl+A C`, then:

```
(qemu) info registers         # GPRs + PC + key CSRs
(qemu) info registers -a      # ALL registers including all CSRs
(qemu) x /10i $pc             # Disassemble 10 instructions at PC
(qemu) xp /8xw 0x80000000     # Hex dump memory at address
```

Switch back to the serial console with `Ctrl+A C` again. Exit QEMU with `Ctrl+A X`.

---

### Method 3: Read CSRs Directly in C Code

You can read any CSR from bare-metal C and print it over UART — useful for runtime
inspection without a debugger attached.

#### `csr.h` — CSR read/write macros

```c
#ifndef CSR_H
#define CSR_H

#include <stdint.h>

// Generic CSR read — works for any CSR name
#define CSR_READ(csr) ({                            \
    uint64_t _val;                                  \
    asm volatile ("csrr %0, " #csr : "=r"(_val));  \
    _val;                                           \
})

// Generic CSR write
#define CSR_WRITE(csr, val) \
    asm volatile ("csrw " #csr ", %0" :: "r"((uint64_t)(val)))

// Snapshot all 32 GPRs into a uint64_t[32] array
#define READ_GPRS(gprs) asm volatile (  \
    "sd x0,   0(%0)\n"  "sd x1,   8(%0)\n"  \
    "sd x2,  16(%0)\n"  "sd x3,  24(%0)\n"  \
    "sd x4,  32(%0)\n"  "sd x5,  40(%0)\n"  \
    "sd x6,  48(%0)\n"  "sd x7,  56(%0)\n"  \
    "sd x8,  64(%0)\n"  "sd x9,  72(%0)\n"  \
    "sd x10, 80(%0)\n"  "sd x11, 88(%0)\n"  \
    "sd x12, 96(%0)\n"  "sd x13,104(%0)\n"  \
    "sd x14,112(%0)\n"  "sd x15,120(%0)\n"  \
    "sd x16,128(%0)\n"  "sd x17,136(%0)\n"  \
    "sd x18,144(%0)\n"  "sd x19,152(%0)\n"  \
    "sd x20,160(%0)\n"  "sd x21,168(%0)\n"  \
    "sd x22,176(%0)\n"  "sd x23,184(%0)\n"  \
    "sd x24,192(%0)\n"  "sd x25,200(%0)\n"  \
    "sd x26,208(%0)\n"  "sd x27,216(%0)\n"  \
    "sd x28,224(%0)\n"  "sd x29,232(%0)\n"  \
    "sd x30,240(%0)\n"  "sd x31,248(%0)\n"  \
    :: "r"(gprs) : "memory")

#endif
```

#### `print.h` — Minimal hex printer over UART (no printf needed)

```c
#ifndef PRINT_H
#define PRINT_H

#include "uart.h"
#include <stdint.h>

static void print_hex64(uint64_t val) {
    uart_puts("0x");
    for (int i = 60; i >= 0; i -= 4) {
        uint8_t nibble = (val >> i) & 0xF;
        uart_putc(nibble < 10 ? '0' + nibble : 'a' + nibble - 10);
    }
}

static void print_reg(const char *name, uint64_t val) {
    uart_puts("  ");
    uart_puts(name);
    uart_puts(" = ");
    print_hex64(val);
    uart_puts("\r\n");
}

#endif
```

#### `main.c` — Dump GPRs + CSRs at runtime

```c
#include "uart.h"
#include "csr.h"
#include "print.h"
#include <stdint.h>

static const char *gpr_names[32] = {
    "zero","ra","sp","gp","tp",
    "t0","t1","t2","s0","s1",
    "a0","a1","a2","a3","a4","a5","a6","a7",
    "s2","s3","s4","s5","s6","s7","s8","s9","s10","s11",
    "t3","t4","t5","t6"
};

void dump_gprs(void) {
    uint64_t gprs[32];
    READ_GPRS(gprs);
    uart_puts("\r\n=== General Purpose Registers ===\r\n");
    for (int i = 0; i < 32; i++)
        print_reg(gpr_names[i], gprs[i]);
}

void dump_csrs(void) {
    uart_puts("\r\n=== Machine-Mode CSRs ===\r\n");

    uart_puts("-- Machine Info --\r\n");
    print_reg("mvendorid", CSR_READ(mvendorid));
    print_reg("marchid  ", CSR_READ(marchid));
    print_reg("mimpid   ", CSR_READ(mimpid));
    print_reg("mhartid  ", CSR_READ(mhartid));

    uart_puts("-- Machine Status --\r\n");
    print_reg("mstatus  ", CSR_READ(mstatus));
    print_reg("misa     ", CSR_READ(misa));

    uart_puts("-- Trap Handling --\r\n");
    print_reg("mtvec    ", CSR_READ(mtvec));
    print_reg("mscratch ", CSR_READ(mscratch));
    print_reg("mepc     ", CSR_READ(mepc));
    print_reg("mcause   ", CSR_READ(mcause));
    print_reg("mtval    ", CSR_READ(mtval));

    uart_puts("-- Interrupts --\r\n");
    print_reg("mie      ", CSR_READ(mie));
    print_reg("mip      ", CSR_READ(mip));

    uart_puts("-- Counters --\r\n");
    print_reg("mcycle   ", CSR_READ(mcycle));
    print_reg("minstret ", CSR_READ(minstret));
    print_reg("time     ", CSR_READ(time));

    uart_puts("-- Physical Memory Protection --\r\n");
    print_reg("pmpcfg0  ", CSR_READ(pmpcfg0));
    print_reg("pmpaddr0 ", CSR_READ(pmpaddr0));
}

void decode_mstatus(void) {
    uint64_t ms = CSR_READ(mstatus);
    uart_puts("\r\n=== mstatus decoded ===\r\n");

    uart_puts("  MIE  (M-mode global interrupt enable) : ");
    uart_putc(((ms >>  3) & 1) ? '1' : '0'); uart_puts("\r\n");
    uart_puts("  SIE  (S-mode global interrupt enable) : ");
    uart_putc(((ms >>  1) & 1) ? '1' : '0'); uart_puts("\r\n");
    uart_puts("  MPIE (previous MIE)                   : ");
    uart_putc(((ms >>  7) & 1) ? '1' : '0'); uart_puts("\r\n");

    uart_puts("  MPP  (previous privilege mode)        : ");
    switch ((ms >> 11) & 0x3) {
        case 3:  uart_puts("Machine\r\n");    break;
        case 1:  uart_puts("Supervisor\r\n"); break;
        default: uart_puts("User\r\n");
    }

    uart_puts("  FS   (FP unit state)                  : ");
    const char *fs_str[] = {"Off","Initial","Clean","Dirty"};
    uart_puts(fs_str[(ms >> 13) & 0x3]); uart_puts("\r\n");
}

void decode_misa(void) {
    uint64_t isa = CSR_READ(misa);
    uart_puts("\r\n=== misa decoded (ISA extensions) ===\r\n");
    for (int i = 0; i < 26; i++) {
        if (isa & (1UL << i)) {
            uart_puts("  Extension ");
            uart_putc('A' + i);
            uart_puts(" supported\r\n");
        }
    }
    uart_puts("  MXL (machine XLEN): ");
    switch ((isa >> 62) & 0x3) {
        case 1:  uart_puts("32-bit\r\n");  break;
        case 2:  uart_puts("64-bit\r\n");  break;
        default: uart_puts("128-bit\r\n");
    }
}

void main(void) {
    uart_puts("\r\n*** RISC-V Register Inspector ***\r\n");
    dump_gprs();
    dump_csrs();
    decode_mstatus();
    decode_misa();
    uart_puts("\r\nDone.\r\n");
    while (1);
}
```

**Build with `-O0`** so the compiler does not optimise away or reorder register contents:

```bash
riscv64-linux-gnu-gcc -march=rv64gc -mabi=lp64d \
  -ffreestanding -nostdlib -O0 \
  -T link.ld -o inspector.elf \
  start.S main.c

qemu-system-riscv64 -machine virt -cpu rv64 -m 128M \
  -nographic -bios none -kernel inspector.elf
```

---

### Key CSR Reference

| CSR | Privilege | Purpose |
|---|---|---|
| `mhartid` | M | Hardware thread ID — which core you are on |
| `misa` | M | ISA extensions active (I, M, A, F, D, C…) |
| `mstatus` | M | Privilege level, interrupt enable state, FP state |
| `mtvec` | M | Base address of trap/interrupt handler |
| `mepc` | M | PC saved when the last trap occurred |
| `mcause` | M | Cause of the last trap |
| `mtval` | M | Faulting address or bad instruction word |
| `mie` | M | Interrupt enable bits (per source) |
| `mip` | M | Interrupt pending bits (per source) |
| `mcycle` | M | Cycle counter since reset |
| `minstret` | M | Instructions-retired counter |
| `pmpcfg0..3` | M | Physical memory protection configuration |
| `satp` | S | Page-table base address + paging mode (Sv39/Sv48) |
| `sstatus` | S | Supervisor-mode status register |
| `sepc` / `scause` | S | Supervisor trap PC and cause |

---

### Tips

- **`mcause` bit 63**: set means interrupt, cleared means synchronous exception. The lower bits are the cause code — e.g. `0x2` = illegal instruction, `0xB` = machine external interrupt.
- **Cycle counting**: read `mcycle` before and after a block of code to measure elapsed cycles — no OS timer required.
- **`mhartid` in multi-core**: with `-smp 4`, each hart reads a unique value from `mhartid`. Use this in `start.S` to branch only hart 0 into `main` and park the rest.
- **Privilege traps on CSR access**: reading a machine-mode CSR from S-mode or U-mode triggers an illegal instruction exception. Only read M-mode CSRs while running in M-mode (which is the default in bare-metal with `-bios none`).

