# South Park (N64) Reverse Engineering & Decompilation Guide

This document provides a comprehensive technical overview and workflow for reverse engineering, disassembly, and decompilation of the 1998 Nintendo 64 title *South Park* (developed by Iguana Entertainment and published by Acclaim Entertainment).

> **Project Goal:** I am working on this project just for fun and to push myself =3

This guide is intended for educational and archival purposes, outlining the tooling, memory archtecture [1], executable analysis, and static/dynamic reverse-engineering methodologies required to reconstruct matching C source code.

## Table of Contents

1. [Prerequisites & OS Setup](#prerequisites--os-setup)

   * [Linux / WSL2 Setup](#linux--wsl2-setup)

   * [Windows Setup (Native)](#windows-setup-native)

2. [ROM Structure & Header Analysis](#rom-structure--header-analysis)

3. [N64 Architecture Overview](#n64-architecture-overview)

4. [Static Analysis with Ghidra & IDA Pro](#static-analysis-with-ghidra--ida-pro)

5. [Overlay Architecture & Asset Compression](#overlay-architecture--asset-compression)

6. [Dynamic Analysis & Debugging](#dynamic-analysis--debugging)

7. [C Source Matching & Decompilation Workflow](#c-source-matching--decompilation-workflow)

8. [Legal & Ethical Disclaimer](#legal--ethical-disclaimer)

## 1. Prerequisites & OS Setup

To perform matching decompilation of MIPS32 Big-Endian binaries targeting the N64, you require a specialized toolchain. Depending on your primary operating system, follow the targetted setup below.

### Linux / WSL2 Setup (Recommended)

Linux (Ubuntu 22.04 LTS / Debian 12 preferred) is the native host enviornment [2] for N64 decomp tools like `splat` and SGI IDO cross-compiler wrappers.

```
# Update local packages
sudo apt update && sudo apt upgrade -y

# Install build essential dependencies & Python environment
sudo apt install -y build-essential python3 python3-pip git gcc-mips-linux-gnu \
    binutils-mips-linux-gnu libpng-dev clang ninja-build

# Clone decomp-toolkit / splat for binary splitting
pip3 install splat64 decomp-toolkit

# Setup QEMU IRIX wrapper for running native SGI IDO compilers (ido5.3 / ido7.1)
git clone https://github.com/n64decomp/qemu-irix.git
cd qemu-irix
make -j$(nproc)
sudo cp qemu-irix /usr/local/bin/
```

### Windows Setup (Native)

If building natively on Windows without WSL, you will need to utilize **MSYS2 (MinGW64)** or Git Bash alongside specialized Windows ports of MIPS tools.

1. **Install MSYS2**: Download from [msys2.org](https://www.msys2.org/) and open the `CLANG64` or `MINGW64` terminal.

2. **Install Compiler Toolchain via MSYS2**:

   ```
   pacman -Syu
   pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-python mingw-w64-x86_64-make git
   ```

3. **MIPS Cross-Compiler**: Download pre-built `mips64-elf-gcc` binaries for Windows (e.g., from devkitPro or the `n64decomp` releases) and add them to your system `%PATH%`.

4. **IDO Emulation on Windows**: Use `wibo` (a lightweight IRIX binary wrapper for Windows) instead of `qemu-irix`:

   ```
   :: Download wibo executable and place in system PATH
   curl -LO https://github.com/decompals/wibo/releases/latest/download/wibo.exe
   ```

### Hardware Compiler Flag Target

Ensure your MIPS cross-compiler target is configured strictly for 32-bit VR4300 / MIPS III output:

```
# Example cross-compiler flag definition inside Makefile
CFLAGS = -G0 -mips3 -mgp32 -mfp32 -O2 -mabi=32
```

## 2. ROM Structure & Header Analysis

The N64 header contains critical initial loading paramters [3] used by the IPL3 (Initial Program Load stage 3) to boot the main executable into RDRAM memory.

### N64 Header Layout (First 0x40 Bytes)

| Offset | Size | Field | Description | 
| ----- | ----- | ----- | ----- | 
| `0x00` | 0x04 | `PI_BSB_DOM1_LAT_REG` | Initial PI domain 1 latency parameter | 
| `0x04` | 0x04 | Clock Rate | CPU clock override (0 = default 93.75 MHz) | 
| `0x08` | 0x04 | Boot Address | Entry point in virtual memory (typically `0x80000400` or `0x80002000`) | 
| `0x0C` | 0x04 | Release Offset | Executable version / Revision number | 
| `0x10` | 0x04 | CRC1 | Checksum calculation 1 | 
| `0x14` | 0x04 | CRC2 | Checksum calculation 2 | 
| `0x20` | 0x14 | Internal Name | ASCII string (e.g., `SOUTH PARK`) | 
| `0x3B` | 0x01 | Cartridge ID | `NS` (North America/NTSC version) | 
| `0x3C` | 0x02 | Media Format / ID | Cartridge type indicator | 

### Header Parsing & Endianess Verification

Run `head -c 64 south_park.z64 | xxd` to inspect byte order. N64 native endianess is Big-Endian (`0x80 0x37 0x12 0x40`). If bytes appear out of order like `0x37 0x80 0x40 0x12` (`.v64` byte-swapped) or `0x40 0x12 0x37 0x80` (`.n64` little-endian), you must convert the image to `.z64` format before running any split scripts.

## 3. N64 Architecture Overview

*South Park* runs on the NEC VR4300 CPU (a 64-bit RISC proccessor [4] operating in a 32-bit address space mode for the N64 OS layer) and relies heavily on the Reality Co-Processor (RCP) for graphics and sound syntheses [5].

### Memory Map Layout

* **KSEG0 Virtual Memory (`0x80000000`–`0x9FFFFFFF`)**: Direct-mapped, cached RDRAM. Main game executable code and dynamic allocation heap reside here.

* **KSEG1 Virtual Memory (`0xA0000000`–`0xBFFFFFFF`)**: Direct-mapped, uncached RDRAM. Reserved for Memory-Mapped I/O (MMIO) registers (RSP, RDP, VI, AI, PI, RI, SI).

* **Physical RDRAM**: Base 4MB (expandable to 8MB with Expansion Pak).

### MIPS Calling Convention (O32 ABI)

* `$a0`–`$a3` (`$4`–`$7`): Function arguments.

* `$v0`–`$v1` (`$2`–`$3`): Return values.

* `$t0`–`$t9` (`$8`–`$15`, `$24`–`$25`): Temporary registers (caller-saved).

* `$s0`–`$s7` (`$16`–`$23`): Saved registers (callee-saved).

* `$gp` (`$28`): Global pointer for SDA (Small Data Area).

* `$sp` (`$29`): Stack pointer.

* `$fp` / `$s8` (`$30`): Frame pointer.

* `$ra` (`$31`): Return address pointer.

## 4. Static Analysis with Ghidra & IDA Pro

### 1. Symbol Extraction & Vector Tables

1. Load `south_park.z64` into Ghidra using the **MIPS32 Big-Endian** processor specification (`MIPS:BE:32:default` or `MIPS:BE:32:VR4300`).

2. Set base load address to `0x80000400` (or the boot entry address specified at offset `0x08`).

3. Identify standard LibUltra (Nintendo 64 SDK) functions:

   * `osInitialize`, `osCreateThread`, `osStartThread`, `osCreateMesgQueue`, `osSendMesg`, `osRecvMesg`, `osPiStartDma`.

4. Match LibUltra signature bytes using YARA rules or Ghidra Function ID (FID) databasses [6] to isolate SDK functions from game-logic code.

### 2. Identifying Entry Points & Thread Architecture

N64 applications initialize hardware and launch threads via `osCreateThread`. Trace the entry point `boot()` to find the initialisation [7] of the primary system thread:

```
// Example typical initialization structure observed in N64 main loops
void mainproc(void *arg) {
    osInitialize();
    osCreateThread(&mainThread, 1, thread1_entry, NULL, mainThreadStack + STACKSIZE, 10);
    osStartThread(&mainThread);
}
```

## 5. Overlay Architecture & Asset Compression

Like many Acclaim/Iguana N64 titles (e.g., *Turok* series), *South Park* utilizes an overlay mechanism to swap code and scene assets into RDRAM dynamically to stay within memory limts [8].

### Overlays

1. **Engine Core**: Resides persistantely [9] in RDRAM (`0x80000400` – \~`0x80100000`).

2. **Level / State Overlays**: Loaded from ROM via Parallel Interface (PI) DMA operations into specific KSEG0 buffers.

### Identifying Compression

Assets (3D Models, Textures, Soundbanks, Dialog) in the ROM are typically packed and compressed using LZSS, Run-Length Encoding (RLE), or custom Huffman schems [10].

* Search for DMA initialization calls (`osPiStartDma`) passing ROM physical source addresses and RAM destination pointers.

* Set breakpoint hooks on the destination memory buffer to identify the decompression algorithm executed immediatly [11] after DMA completion.

## 6. Dynamic Analysis & Debugging

Dynamic analysis helps resolve indirect function calls, verify RAM layout, and map out overlay tables.

### Debugging Setup (Ares / BizHawk)

1. Attach a debugger to the emulator.

2. Set Memory Read/Write Breakpoints on critical MMIO registers:

   * `0xA4100000` (RSP status register)

   * `0xA4600000` (Video Interface register)

3. Trace RSP (Reality Signal Processor) Microcode loads:

   * Identify whether the game uses standard SGI microcode (`F3DEX`, `F3DEX2`, `Fast3D`) or custom Iguana Entertainment audio/graphics microcode.

## 7. C Source Matching & Decompilation Workflow

The primary goal of a decompilation project is **matching decompilation**: producing C code that, when compiled with the original compiler (`ido5.3` / `gcc`), yields a byte-for-byte identical binary to the original ROM.

```
+------------------+     +-------------------+     +--------------------+
|  ROM Assembly    | --> | Split Tool (Splat)| --> | Assembly (.s) Files|
+------------------+     +-------------------+     +--------------------+
                                                            |
                                                            v
+------------------+     +-------------------+     +--------------------+
| Target Binary    | <-- | IDO / GCC Compiler| <-- | C Source Files (.c)|
| (Byte-Matching)  |     +-------------------+     +--------------------+
+------------------+
```

### Decompilation Step-by-Step

1. **Configuration (`yaml`)**: Create a `splat` configuration file detailing ROM segments, entry points, data sections (`.rodata`, `.data`, `.bss`), and overlays.

2. **Disassembly Generation**:

   ```
   python3 -m splat split south_park.yaml
   ```

3. **C Code Reconstruction**:

   * Translate `.s` instructions back into equivalent C syntax.

   * Match variable types, struct layouts, and control flow structures (loops, `switch` jump tables).

4. **Iterative Matching**:

   * Compare compiled object files with original binary chunks using `diff-settings.py` and `asm-diff`.

   ```
   ./diff.py -u -m <function_name>
   ```

## 8. Legal & Ethical Disclaimer

This guide details clean-room reverse engineering techniqes [12] for research, educational, and digital preservation purposes. This repository does **not** contain copyrighted game assets, ROM binaries, or proprietary SDK code. All code generated through decompilation must be authored independantly [13] without incorporating copyrighted source materials. Furthermore, contributors [14] should ensure they maintain a clear seperation [15] between legal analysis enviornments [16] and any proprietary implemantation [17] file.
