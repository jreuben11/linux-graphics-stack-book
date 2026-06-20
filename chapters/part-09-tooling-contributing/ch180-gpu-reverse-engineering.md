# Chapter 180 — GPU Reverse Engineering: Tools, Methodology, and Case Studies

**Target audiences:** Systems and driver developers working on open-source GPU drivers for undocumented hardware; contributors to Nouveau, Panfrost, etnaviv, Asahi, freedreno/turnip, and other reverse-engineered drivers; kernel hackers who want to understand how register maps, command-stream formats, and ISAs are discovered from scratch.

---

## Table of Contents

1. [Why GPU Reverse Engineering Matters](#1-why-gpu-reverse-engineering-matters)
2. [The Landscape of RE-Derived Drivers](#2-the-landscape-of-re-derived-drivers)
3. [What GPU RE Actually Means](#3-what-gpu-re-actually-means)
4. [envytools: The NVIDIA RE Toolkit](#4-envytools-the-nvidia-re-toolkit)
   - [4.1 nva: Direct MMIO Access Tools](#41-nva-direct-mmio-access-tools)
   - [4.2 rnndb: The XML Register Database](#42-rnndb-the-xml-register-database)
   - [4.3 envydis and envyas: ISA Disassembly and Assembly](#43-envydis-and-envyas-isa-disassembly-and-assembly)
5. [valgrind-mmt: Capturing the Proprietary Command Stream](#5-valgrind-mmt-capturing-the-proprietary-command-stream)
   - [5.1 How valgrind-mmt Works](#51-how-valgrind-mmt-works)
   - [5.2 demmt: Decoding the Trace](#52-demmt-decoding-the-trace)
6. [Methodology: MMIO Probing](#6-methodology-mmio-probing)
7. [Command-Stream Capture Across Drivers](#7-command-stream-capture-across-drivers)
   - [7.1 Nouveau: mmiotrace and valgrind-mmt](#71-nouveau-mmiotrace-and-valgrind-mmt)
   - [7.2 freedreno: libwrap, cffdump, and the .rd Format](#72-freedreno-libwrap-cffdump-and-the-rd-format)
   - [7.3 etnaviv: viv_interpose and libvivhook](#73-etnaviv-viv_interpose-and-libvivhook)
   - [7.4 Mesa u_trace: Generic GPU Tracing](#74-mesa-u_trace-generic-gpu-tracing)
8. [ISA Reverse Engineering](#8-isa-reverse-engineering)
   - [8.1 nvdisasm: NVIDIA's Own Disassembler](#81-nvdisasm-nvidias-own-disassembler)
   - [8.2 LLVM-Based Disassemblers](#82-llvm-based-disassemblers)
   - [8.3 intel_aubdump: Intel Command Capture](#83-intel_aubdump-intel-command-capture)
9. [Firmware Protocol Reverse Engineering](#9-firmware-protocol-reverse-engineering)
   - [9.1 Ghidra for GPU Firmware](#91-ghidra-for-gpu-firmware)
   - [9.2 Differential Firmware Analysis](#92-differential-firmware-analysis)
10. [Case Study: Nouveau and envytools](#10-case-study-nouveau-and-envytools)
11. [Case Study: Panfrost and the Mali Job Descriptor Format](#11-case-study-panfrost-and-the-mali-job-descriptor-format)
12. [Case Study: Asahi Linux and the Apple AGX Firmware](#12-case-study-asahi-linux-and-the-apple-agx-firmware)
13. [Case Study: etnaviv and the Vivante GC Series](#13-case-study-etnaviv-and-the-vivante-gc-series)
14. [Legal Considerations](#14-legal-considerations)
15. [Integrations](#15-integrations)

---

## 1. Why GPU Reverse Engineering Matters

A GPU driver cannot be written from first principles. Unlike a software algorithm where the specification is mathematics, a GPU driver must program specific undocumented registers in precisely the right sequence, with precisely the right values, or the hardware silently produces wrong results, hangs, or triggers a firmware fault. Without a hardware reference manual — which GPU vendors almost never release publicly for their consumer products — the only source of ground truth is the vendor's own proprietary driver binary.

Reverse engineering is the discipline of recovering that ground truth by observing what the binary driver does to the hardware and building up a symbolic model of the hardware from those observations. The goal is not to reproduce the binary driver — that would invite copyright liability — but to understand the hardware well enough to write an independent implementation from scratch.

The need for this discipline is not hypothetical. Every major GPU driver on Linux outside the vendor-supplied proprietary modules was built, at least in part, through reverse engineering:

- **Nouveau** (NVIDIA): All GPU generations from NV04 through Ada Lovelace, with register maps built by tracing the proprietary `nvidia.ko` driver.
- **Panfrost/Panthor** (ARM Mali Midgard, Bifrost, Valhall/CSF): Job descriptor formats and shader ISAs recovered from the vendor's Android kernel driver and firmware binary.
- **etnaviv** (Vivante GC series): Command-stream format and ISA recovered by intercepting the proprietary Vivante EGL driver on i.MX SoCs.
- **freedreno/turnip** (Qualcomm Adreno): Command-stream format recovered from the Android kgsl driver; Adreno ISA partially documented by Qualcomm and partially recovered via RE.
- **Asahi (agx)** (Apple AGX M-series): GPU firmware protocol and shader ISA recovered by tracing macOS driver interactions and using the m1n1 hypervisor.

Without this work, none of these GPUs would have an open-source driver. The hardware would be permanently locked to the vendor's proprietary software stack.

---

## 2. The Landscape of RE-Derived Drivers

Before going into tools and methodology, it is useful to understand what each project targets and how complete its knowledge base is:

| Driver | Hardware | Kernel location | Primary RE technique |
|---|---|---|---|
| Nouveau | NVIDIA NV04–AD102 | `drivers/gpu/drm/nouveau/` | mmiotrace, valgrind-mmt, envytools |
| Panfrost | Mali Midgard, Bifrost | `drivers/gpu/drm/panfrost/` | panwrap LD_PRELOAD, panloader |
| Panthor | Mali Valhall/CSF | `drivers/gpu/drm/panthor/` | CSF firmware protocol analysis |
| etnaviv | Vivante GC800–GC7000 | `drivers/gpu/drm/etnaviv/` | viv_interpose LD_PRELOAD, libvivhook |
| freedreno | Adreno A2xx–A7xx | `drivers/gpu/drm/msm/` | libwrap LD_PRELOAD, cffdump, .rd files |
| agx (Asahi) | Apple AGX G13–G19 | `drivers/gpu/drm/asahi/` | m1n1 hypervisor tracing, macOS API RE |

The level of vendor cooperation varies widely. ARM became significantly more cooperative with the Panfrost project after 2020, eventually providing some official documentation. Qualcomm has partially published Adreno documentation. Apple provides no documentation. NVIDIA's 2022 open-gpu-kernel-modules release provided register name headers for newer GPUs but not architectural documentation.

---

## 3. What GPU RE Actually Means

A GPU presents multiple layers of abstraction to the driver, and each layer has to be independently reverse-engineered:

**MMIO register space.** The GPU exposes control registers to the CPU via a PCI BAR (Base Address Register), typically BAR0 for the main control space. These registers select GPU functional units (graphics engine, display engine, video decode, clock control, memory controller) and control their operation. The number of registers is large — a modern NVIDIA GPU has thousands of distinct register addresses in BAR0 — and most are not self-documenting. RE methodology must discover which address controls which behaviour.

**Command stream format.** The CPU does not directly program the 3D/compute engine registers in real time. Instead, it prepares a command buffer (called a pushbuffer in NVIDIA terminology, indirect buffer or IB in AMD/Adreno parlance, command list in Apple AGX terminology) and passes its DMA address to the GPU. The GPU's command processor reads the buffer and programs the 3D engine registers from it. The command buffer has its own encoding — a protocol of opcode/value pairs, DMA addresses for textures and vertex buffers, and synchronisation primitives. This protocol must be understood from scratch.

**Shader ISA.** The GPU runs shaders — vertex, fragment, compute — as programs on its shader cores. The GPU's shader instruction set is a proprietary ISA that must be decoded from compiled binaries extracted from the blob driver. Without ISA documentation, the driver cannot write a shader compiler.

**Firmware protocol.** Modern GPUs run one or more ARM Cortex-M (or proprietary Falcon) microcontrollers for command scheduling, power management, and hardware fault handling. The host driver communicates with this firmware through shared memory mailboxes. The message protocol — structure layouts, sequence numbers, timeout expectations, error codes — must be reverse-engineered from the firmware binary or from traces of host-to-firmware message traffic.

These four layers are largely independent, and different tools are used for each. A command-stream trace shows you the command format but says nothing about register semantics; an MMIO trace shows register writes but not shader code; a firmware binary contains no 3D command structures.

---

## 4. envytools: The NVIDIA RE Toolkit

The primary toolkit for NVIDIA GPU reverse engineering is **envytools** [Source](https://github.com/envytools/envytools), hosted at GitHub and described as "tools for people envious of NVIDIA's blob driver." It is primarily maintained by Marcelina Kościelnicka and has contributions from dozens of Nouveau developers over more than fifteen years. envytools covers three distinct needs: direct hardware access (`nva/` tools), a symbolic register database (`rnndb/`), and ISA disassembly/assembly (`envydis/`, `envyas`).

### 4.1 nva: Direct MMIO Access Tools

The `nva/` subdirectory contains a set of small utilities for directly reading and writing GPU MMIO registers from userspace. These tools require a kernel module (`nva.ko`, built alongside them) that `ioremap()`s the GPU's BAR0 at load time and exports a character device at `/dev/nva0` (or `/dev/nvidiaX` for some configurations) through which userspace can issue read/write requests.

The key tools are:

- **`nvalist`** — Lists available NVIDIA GPUs visible to the nva kernel module, showing PCI IDs and the codename detected.
- **`nvapeek <addr>`** — Reads the 32-bit MMIO register at `<addr>` within BAR0.
- **`nvapeek <addr> <count>`** — Reads a range of `count` bytes of registers starting at `<addr>`.
- **`nvapoke <addr> <value>`** — Writes a 32-bit `<value>` to the register at `<addr>`.
- **`nvafuzz <addr> [<end>]`** — Writes random values to a register or range in an infinite loop; used to fuzz an unknown register space for unexpected side effects.
- **`nvawatch <addr> [-t]`** — Polls the register at `<addr>` in a tight loop and prints its value every time it changes; with `-t`, adds a timestamp and elapsed time since the last change. Useful for discovering which registers change during a display mode set or context switch.
- **`nvascan <addr> <end>`** — Probes each register address in a range to determine which respond meaningfully (many addresses return all-zeros or all-ones on unimplemented registers).
- **`nvagetbios`** — Extracts the GPU VBIOS (Video BIOS) image from the card. The VBIOS is an x86 real-mode binary that initialises the GPU before the operating system loads, and it contains tables of PLL coefficients, thermal thresholds, voltage levels, and sometimes partial register documentation.
- **`nvafakebios`** — Uploads a modified VBIOS image for testing purposes.
- **`nvadownload` / `nvaupload`** — Transfer raw data to and from GPU VRAM, enabling extraction of firmware images or GPU-side data structures.
- **`nvafucstart`** — Uploads and executes Falcon microcontroller microcode. The Falcon is NVIDIA's proprietary embedded processor that runs on units including PDAEMON (power management), PGRAPH CTX (context save/restore), PMU, and the video processor.

These tools are indispensable during early hardware bring-up. When a new NVIDIA GPU generation appears, the first task is to `nvapeek` BAR0 at known register addresses from prior generations, observe which respond as expected, and identify what has moved or changed.

```bash
# Build nva tools (requires nva.ko, built separately)
# From envytools build directory:

# List available GPUs
./nva/nvalist
# Output: 0: NV162 [TU102] (PCI 0000:01:00.0)

# Read NV_PMC_BOOT_0 (always at BAR0+0x0000) to identify GPU revision
./nva/nvapeek 0x0000
# Output: 0x00000000: 162000a1

# Monitor PLL lock bit (NVIDIA G80: NV_PRAMDAC_NVPLL_COEFF at 0x680500)
./nva/nvawatch 0x680500 -t
# Prints value and elapsed ms each time it changes

# Fuzz unknown register 0x10000–0x10100 for side effects
./nva/nvafuzz 0x10000 0x10100
```

> **Note:** `nva.ko` requires direct BAR0 access via `ioremap()`. On modern kernels with lockdown enabled (Secure Boot active), `ioremap()` of physical PCI BAR addresses by out-of-tree modules may be restricted. This is why envytools RE work is typically done on development machines with Secure Boot disabled.

### 4.2 rnndb: The XML Register Database

The `rnndb/` directory is the accumulated knowledge base of the envytools project. It is a collection of XML files describing every characterised NVIDIA GPU register — its BAR0 address, the width and semantics of each bitfield, which GPU families it applies to, and (where known) what it does.

The schema is defined by the `rules-ng-ng` XML format, with the `headergen2` tool generating C header files from the XML. These headers are used directly by the Nouveau kernel driver — `drivers/gpu/drm/nouveau/` contains the generated `nvhw/` headers. The access macros `nvkm_rd32()` and `nvkm_wr32()`, defined in `include/nvkm/core/device.h`, wrap `ioread32_native()` and `iowrite32_native()` against the `ioremap()`'d BAR0 base, using these symbolic names.

```xml
<!-- rnndb/graph/nv50_pgraph.xml (simplified excerpt) -->
<!-- Source: https://github.com/envytools/envytools/blob/master/rnndb/graph/nv50_pgraph.xml -->
<database xmlns="http://nouveau.freedesktop.org/
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://nouveau.freedesktop.org/ rules-ng.xsd">
<domain name="NV_PGRAPH" bare="yes" prefix="NV_PGRAPH">
  <!-- Discovered by mmiotrace of nvidia.ko on G80 hardware -->
  <reg32 offset="0x0400" name="INTR">
    <bitfield name="NOTIFY" pos="0"/>
    <bitfield name="MISSING_HW" pos="4"/>
    <bitfield name="BUFFER_NOTIFY" pos="16"/>
  </reg32>
  <reg32 offset="0x0404" name="INTR_EN"/>
  <reg32 offset="0x0408" name="NSOURCE"/>
</domain>
</database>
```

The rnndb coverage is strongest for NV50 (G80, Tesla architecture) through Maxwell (GM200) generations, where a decade of active tracing has produced near-complete register maps. For Turing (TU102) and later, rnndb coverage is partial; the 2022 open-gpu-kernel-modules release by NVIDIA provided generated register headers under MIT licence that the Nouveau project can reference, partially reducing the need for new RE on modern GPUs [Source](https://github.com/NVIDIA/open-gpu-kernel-modules).

The `demmt` tool in `demmt/` consumes traces produced by `valgrind-mmt` (see next section) and decodes MMIO addresses against the rnndb, printing a human-readable annotation for each register access:

```
# demmt decoding a valgrind-mmt trace of glxgears on G80
NV_PGRAPH_INTR <- 0x00000001     # NOTIFY interrupt pending
NV_PGRAPH_INTR_EN -> 0xfffffffe  # Disable NOTIFY interrupt enable
NV_FIFO_CACHES <- 0x00000001     # Enable FIFO caches
```

### 4.3 envydis and envyas: ISA Disassembly and Assembly

`envydis` is a multi-target disassembler for NVIDIA GPU shader ISAs and the Falcon microcontroller ISA [Source](https://github.com/envytools/envytools/tree/master/envydis). Given a raw binary blob (extracted from a compiled shader or GPU firmware), `envydis` outputs human-readable assembly. `envyas` is the complementary assembler, accepting the same syntax and producing binary output — essential for writing test shaders to probe ISA behaviour.

The ISA targets supported by envydis correspond to NVIDIA's major GPU generations:

- **`nv50`** — NV50/G80 (Tesla) unified shader architecture introduced in 2006
- **`gf100`** — GF100/Fermi, the first CUDA-capable architecture with a new scalar ISA
- **`gk110`** — GK110/Kepler, adding warp shuffle and CUDA Dynamic Parallelism instructions
- **`gm107`** — GM107/Maxwell, the 8-bit integer extensions and improved scheduling
- **`falcon`** — The Falcon RISC-like microcontroller ISA, used for firmware on PGRAPH CTX, PDAEMON, PMU, SEC, and video decode units across all modern NVIDIA GPUs

```bash
# Disassemble a raw GF100 (Fermi) shader binary extracted from the blob
envydis -m gf100 shader.bin

# Typical output (simplified):
# 0x00000000: 0x00000000a0000001  nop
# 0x00000008: 0xfc0007e04003c000  mov b32 $r0 c0[0x0]   # load from const buffer
# 0x00000010: 0x20000000fc800201  fadd $r0 $r0 $r1

# Assemble a test shader and write to binary
envyas -m gf100 -o test.bin test.s
```

For shader ISAs of Volta (GV100) and later generations, NVIDIA's own `nvdisasm` tool (bundled in the CUDA SDK) is more complete than envydis, as the ISA was significantly redesigned and reverse engineering coverage in envydis is incomplete for these newer targets.

---

## 5. valgrind-mmt: Capturing the Proprietary Command Stream

`valgrind-mmt` is a fork of the Valgrind dynamic instrumentation framework, maintained in the `envytools/valgrind` repository [Source](https://github.com/envytools/valgrind). It adds a Valgrind tool client called `mmt` (mmap tracer) that intercepts `mmap()` calls made by the application being traced and installs write-monitoring on the mapped memory regions.

The name `mmt` reflects its core mechanism: it traces `mmap`-based memory accesses rather than raw MMIO accesses. The NVIDIA proprietary driver communicates with the GPU's command processor by `mmap()`ing a region of GPU command buffer memory into the application's address space and writing PM4-style pushbuffer commands directly into it via normal store instructions. By instrumenting those stores, `valgrind-mmt` captures the complete pushbuffer stream being sent to the GPU — the command format, the method-value pairs, the DMA addresses of texture and vertex data — without needing kernel-level MMIO tracing.

### 5.1 How valgrind-mmt Works

Under normal execution, the proprietary driver's pushbuffer writes are just store instructions to userspace addresses. Valgrind runs the application under a JIT-compiled simulation environment (the Valgrind core), and `valgrind-mmt` instruments every store instruction that falls within a region registered as a GPU command buffer. The instrumentation records the address, value, and width of each write.

Additionally, `valgrind-mmt` intercepts `ioctl()` calls to the NVIDIA driver device files (`/dev/nvidiaX`), capturing the ioctl number and arguments. The combination of mmap writes and ioctl calls constitutes a complete picture of the host-side protocol between the userspace driver and the kernel module.

```bash
# Capture a valgrind-mmt trace of glxgears using the proprietary nvidia driver
# Requires: valgrind-mmt fork built (not upstream Valgrind)

valgrind --tool=mmt --mmt-trace-nvidia-ioctls \
         --log-file=glxgears-trace.bin.log \
         glxgears

# The binary log is then decoded by demmt from envytools:
./demmt/demmt -l glxgears-trace.bin.log
```

The `--mmt-trace-nvidia-ioctls` flag tells the tool to also capture NVIDIA-specific ioctl() calls. Without this flag, only pushbuffer writes (mmap writes) are captured; with it, the resource-manager ioctl traffic is also included, showing how the driver allocates memory, creates channels, and sets up the channel object hierarchy.

### 5.2 demmt: Decoding the Trace

`demmt` (in `envytools/demmt/`) reads the binary trace produced by `valgrind-mmt` and produces human-readable output. It performs several decoding steps:

1. It identifies the GPU from the NVIDIA device ioctl traffic and selects the appropriate rnndb register set.
2. It reconstructs the pushbuffer method stream, decoding each method-value pair against the NV50 (or later) FIFO method table in rnndb.
3. For memory address fields in commands (texture base, vertex buffer base, framebuffer base), it attempts to correlate them with the allocation records captured from ioctl() calls, printing symbolic names.
4. For shader program addresses, if a shader binary has been extracted separately (via `nvadownload` or by instrumenting `cuModuleLoad()`), `demmt` can call `envydis` inline to disassemble the shader code in context.

The result is a trace like:

```
# demmt output (G80, simplified)
NVRM ioctl: 0x2080014b (NV01_DEVICE_0) -> handle 0xbeef0001
NVRM ioctl: 0x20801f01 (NV50_CHANNEL_IND_GPFIFO) -> handle 0xbeef0002
mmap: channel pushbuf at 0x7f4400000000 (size 0x200000)
...
[pushbuf] NV50_3D.CLEAR_VALUE_DEPTH = 0xffffffff
[pushbuf] NV50_3D.CLEAR_VALUE_COLOR[0] = 0x00000000
[pushbuf] NV50_3D.CLEAR_BUFFERS = COLOR0|DEPTH
[pushbuf] NV50_3D.VP_ADDRESS_HIGH = 0x00000001
[pushbuf] NV50_3D.VP_ADDRESS_LOW  = 0x40000000   # shader @ 0x140000000
  --> envydis (gf100): fadd $r0 $r1 $r2 ...
```

This output is the raw material from which Nouveau developers fill in rnndb entries and understand the command format expected by the G80 PGRAPH engine.

---

## 6. Methodology: MMIO Probing

When no prior rnndb entry exists for a register — either because the GPU is new or because the register controls an undocumented feature — the discovery process follows a systematic MMIO probing methodology:

**Step 1: Identify the BAR0 base address.** The PCI subsystem exposes the BAR addresses via `/sys/bus/pci/devices/<bdf>/resource`. The first readable numeric region is typically BAR0.

```bash
# Find the BAR0 base address for the first NVIDIA GPU
cat /sys/bus/pci/devices/0000:01:00.0/resource
# Line 0 = BAR0: 0x00000000f6000000 0x00000000f7ffffff 0x0000000000040200
# BAR0 base = 0xf6000000, size = 0x2000000 (32 MiB)
```

**Step 2: Read baseline register values.** Before touching anything, `nvapeek` a range of interest to establish the baseline state. This is especially important for status and interrupt registers that clear on read.

**Step 3: Trigger a known GPU operation** and watch which registers change using `nvawatch` on suspected addresses, or capture a before/after snapshot with `nvapeek <range>`. Operations that produce visible output — a display mode change, a clear-colour operation, a triangle draw — are easiest to correlate.

**Step 4: Write candidate values and observe effects.** `nvapoke` a suspected register with known field encodings from similar registers, then observe whether the display, a GPU compute result, or another observable register changes as expected.

**Step 5: Cross-reference with VBIOS.** The VBIOS extracted by `nvagetbios` sometimes contains tabular data using the same register addresses used during hardware initialisation. Parsing the VBIOS data tables (done by the `nvbios` tool in envytools) can reveal register names that correspond to PLL coefficients, thermal limits, and clock domain controls that are used during POST.

**Step 6: Build the rnndb entry.** Once a register's purpose, address, bitfield structure, and valid values are understood, an XML entry is added to the appropriate rnndb file. This is the final deliverable of the probing cycle — transforming an empirical observation into a symbolic, documented register definition that future driver code can use.

The kernel's built-in `mmiotrace` infrastructure (enabled via `CONFIG_MMIOTRACE` and accessed through `ftrace`) provides an alternative to the nva tools when the proprietary driver itself needs to be traced against real hardware interactions. It works by intercepting `ioremap()` calls from the driver module and setting up page-protection traps so that each register access triggers a page fault that the trace infrastructure records before replaying the access [Source](https://www.kernel.org/doc/html/latest/trace/mmiotrace.html).

```bash
# Enable mmiotrace in the kernel
echo 1 > /sys/kernel/debug/tracing/events/mmiotrace/enable
# Load the proprietary driver (which calls ioremap on BAR0)
modprobe nvidia
# Run an application, then read the trace
cat /sys/kernel/debug/tracing/trace > mmio-trace.txt
```

The `demmio` tool (separate from `demmt`) parses mmiotrace output and decodes register addresses against rnndb, producing the same symbolic output as demmt but from kernel-level captures rather than userspace pushbuffer traces.

---

## 7. Command-Stream Capture Across Drivers

Each open-source GPU driver has developed its own command-stream capture toolchain, adapted to the specific way the proprietary vendor driver submits work to the GPU. The common pattern is either kernel-level interception (reading the submitted IBs through debugfs or sysfs before they execute) or userspace interposition (LD_PRELOAD or Valgrind instrumentation).

### 7.1 Nouveau: mmiotrace and valgrind-mmt

Nouveau uses both mmiotrace (for initial GPU bring-up, to understand what BAR0 register sequences the blob driver executes) and valgrind-mmt (for understanding the command-buffer protocol the userspace driver sends to the kernel channel). The workflow is described in the Nouveau wiki under MmioTraceDeveloper [Source](https://nouveau.freedesktop.org/MmioTraceDeveloper.html).

For decoding Nouveau-specific pushbuffers, the `nouveau_trace` facility in the Nouveau kernel driver can dump submitted IBs to a debugfs file, which can then be decoded against the NV50 pushbuffer method tables in rnndb.

### 7.2 freedreno: libwrap, cffdump, and the .rd Format

The freedreno stack for Adreno GPUs uses a different approach: an LD_PRELOAD library called `libwrap.so` (and `libwrapfake.so` for emulation) that intercepts the KGSL (Kernel Graphics Support Layer) driver ioctls and captures command buffers before they are submitted [Source](https://github.com/freedreno/freedreno/wiki/Reverse-engineering-tools).

Captured data is written to `.rd` (re-dump) files — a simple binary format with typed sections (type, length, value) containing GPU register writes, IB contents, and texture/vertex buffer snapshots. The `.rd` files compress well and are the standard interchange format for sharing GPU issues in the freedreno community.

```bash
# Capture a command stream from the blob Adreno driver using libwrap
LD_PRELOAD=/path/to/libwrap.so \
    WRAP_GPU_ID=630 WRAP_GMEM_SIZE=512k \
    glmark2-es2 2>&1 | gzip > trace.rd.gz

# Decode the .rd file with cffdump
./util/cffdump trace.rd.gz
# With --summary for a condensed view showing state at each draw call:
./util/cffdump --summary trace.rd.gz

# For kernel-side capture (Mesa freedreno driver, not blob):
cat /sys/kernel/debug/dri/0/rd > cmdstream.rd &
# Enable full buffer snapshots for replay:
echo Y > /sys/module/msm/parameters/rd_full
glmark2-es2
kill %1
```

`cffdump` understands Adreno PM4 packets, links in `librnn` from envytools for register decode, and calls the freedreno shader disassembler to show inline shader disassembly at draw call boundaries [Source](https://github.com/freedreno-zz/freedreno/blob/master/util/cffdump.c). The XML register definitions for Adreno GPUs (analogous to rnndb but for Qualcomm hardware) live in `src/freedreno/registers/` in Mesa [Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/freedreno/registers).

For reproducing GPU faults from field reports, the `replay` tool in `src/freedreno/drm-shim/` reconstructs a GPU submission from a `.rd` file using the MSM kernel driver, and `crashdec` decodes GPU core dumps from `/sys/devices/virtual/devcoredump/`.

### 7.3 etnaviv: viv_interpose and libvivhook

The etnaviv project (for Vivante GC-series GPUs found in NXP/Freescale i.MX SoCs) used two main interception mechanisms during its initial reverse-engineering phase:

- **`viv_interpose.so`**: An LD_PRELOAD library that intercepts Vivante driver calls (the `galcore` kernel driver ioctl interface) and captures command buffers.
- **`libvivhook`**: A hook library that allowed feature-bit overrides on a running system with the binary blob driver, enabling developers to disable specific hardware features and observe which rendering operations broke — a form of differential analysis to isolate which GPU feature controlled which rendering behaviour.

Captured streams were decoded by `dump_cmdstream.py` [Source](https://github.com/etnaviv/etna_viv), which understood the Vivante command stream format and output human-readable PM4-style decodes. The Vivante ISA was reverse-engineered by the same project using `tools/disasm.py` and `tools/asm.py` — a Python-based disassembler/assembler pair that was the foundation for the etnaviv shader compiler in Mesa.

> **Note: needs verification** — The precise current status of `viv_interpose` and `libvivhook` as maintained tools vs. archived artifacts from the initial RE phase should be confirmed in the etna_viv repository.

### 7.4 Mesa u_trace: Generic GPU Tracing

Mesa provides a driver-agnostic GPU tracing framework called `u_trace` (in `src/util/u_trace.h`) that is used by freedreno, panfrost, Iris, and other drivers for performance tracing rather than reverse engineering [Source](https://docs.mesa3d.org/u_trace.html). `u_trace` can output traces in human-readable text or in JSON format compatible with `chrome://tracing`. It operates by inserting GPU timestamp queries at trace points defined by macro, making it useful for identifying the GPU-side timing of draws and compute dispatches.

While `u_trace` is a tracing tool for open-source drivers rather than a reverse-engineering tool for proprietary ones, it serves the same debugging purpose once the initial RE work is done: helping developers understand what the GPU is doing and where time is being spent.

---

## 8. ISA Reverse Engineering

Recovering the GPU shader ISA is the most technically demanding part of GPU reverse engineering, because the ISA is a custom VLIW or RISC-like instruction set with no public documentation, and the instructions interact with the hardware's register file, special-function units, and memory subsystem in ways that are difficult to infer from I/O observations alone.

The general methodology is:

1. **Extract a shader binary.** Intercept the proprietary driver's shader compilation call (e.g., `glShaderBinary()`, `cuModuleLoad()`, or a custom ioctl) using an LD_PRELOAD hook and write the GPU binary to disk.
2. **Enumerate instruction encodings.** Feed trivial shaders (constant output, identity transform, single arithmetic operations) through the proprietary compiler and examine the resulting binaries. Each binary differs by exactly the instruction(s) needed to implement the trivial shader, making individual instruction encodings identifiable.
3. **Discover register conventions.** Map which instruction fields correspond to source and destination register indices by producing shaders that move values between specific registers and observing which fields change.
4. **Discover control flow encoding.** Compile shaders with branches and loops to identify jump offset fields, predicate register encoding, and loop count encoding.
5. **Discover special-function encoding.** Compile shaders using `sin`, `cos`, `rcp`, `rsqrt`, `log2`, `exp2` and isolate the special-function instruction encoding.

### 8.1 nvdisasm: NVIDIA's Own Disassembler

For Volta (GV100) and later NVIDIA architectures, NVIDIA bundles `nvdisasm` with the CUDA SDK [Source](https://docs.nvidia.com/cuda/cuda-binary-utilities/index.html). `nvdisasm` is the authoritative disassembler for NVIDIA shader ISAs from SM50 (Maxwell) through SM100 (Blackwell):

```bash
# Disassemble a CUDA cubin file
nvdisasm --binary SM89 mykernel.cubin    # SM89 = Ada Lovelace

# Also available: cuobjdump for host+device binaries
cuobjdump --dump-sass myprogram         # SASS = Shader Assembly

# Output includes control code annotations:
# /*0008*/  @P0 FADD R0, R0, R1 ;       # Conditional float add
# /*0018*/      EXIT ;
```

`nvdisasm` is particularly useful for Nouveau developers working on newer architectures because envydis ISA coverage for post-Maxwell generations is incomplete. The CUDA SDK is freely downloadable from developer.nvidia.com without NVIDIA account requirements.

For Kepler (GK104–GK210) shaders, `envydis -m gk110` and `cuobjdump` both work, with envydis being more useful for cross-referencing against the Nouveau driver's shader encoding in `src/nouveau/compiler/`.

### 8.2 LLVM-Based Disassemblers

LLVM includes GPU-targeting backends that can be repurposed as disassemblers:

- **AMDGPU:** `llvm-objdump` with the AMDGPU target disassembles GCN and RDNA shader binaries: `llvm-objdump -d --mattr=+gfx1100 shader.elf`
- **SPIRV-Cross / spirv-dis:** For SPIR-V intermediate representation, `spirv-dis` (from SPIRV-Tools) disassembles SPIR-V binaries to a human-readable textual format, useful for tracing Vulkan/OpenCL shader compilation chains.

For Adreno ISA (Qualcomm), the `qrisc-disasm` tool in freedreno's tree disassembles SQE (shader queue engine) firmware binaries.

### 8.3 intel_aubdump: Intel Command Capture

For Intel i915/Xe GPUs, `intel_aubdump` (from intel-gpu-tools) captures an application's i915 GEM execbuffer2 submissions to an AUB (Auburn Utility Binary) file [Source](https://manpages.debian.org/testing/intel-gpu-tools/intel_aubdump.1.en.html). The AUB format logs GPU workloads in the form they would be submitted to Intel Graphics Hardware, enabling replay using the GPU simulator or analysis in tools like Intel's Graphics Frame Analyzer.

```bash
# Launch glxgears and capture all GPU submissions to an AUB file
intel_aubdump --output=glxgears.aub glxgears

# Override PCI ID for simulating a different GPU generation
intel_aubdump --device 0x9a49 --output=sim.aub myapp  # TGL
```

AUB capture was especially important during the early development of the Intel XeHPC and Xe2 architectures, where command format changes needed to be understood before hardware was publicly available.

---

## 9. Firmware Protocol Reverse Engineering

Modern GPUs delegate substantial functionality to embedded ARM Cortex-M microcontrollers or proprietary processors (NVIDIA's Falcon). The host driver communicates with this firmware through shared memory — a region of GPU-accessible DRAM where the CPU writes request structures and the firmware writes response structures. Reverse-engineering this protocol requires different techniques than MMIO or command-stream analysis.

### 9.1 Ghidra for GPU Firmware

Ghidra (the NSA's open-source RE framework, available at https://ghidra-sre.org/) is the primary tool for static analysis of GPU firmware binaries. GPU firmware images are typically ARM Cortex-M (thumb2) binaries for the GPU's management processor, or custom ISA binaries for NVIDIA Falcon processors.

For ARM Cortex-M firmware (Mali CSF firmware, Adreno GMU firmware, NVIDIA PMU):

1. **Extract the firmware image.** On Linux, firmware blobs are loaded by the driver from `/lib/firmware/` and passed to the GPU via `request_firmware()`. The loaded binary can be captured with `nvadownload` or by instrumenting `request_firmware()`.
2. **Load in Ghidra with the correct architecture.** Select ARM Cortex-M, thumb2 mode. Define the memory map: firmware load address (often read from hardware registers or the driver source), stack region, and MMIO-mapped peripheral addresses.
3. **Recover symbol names.** GPU firmware images sometimes retain a `.symtab` section, or error strings (ASCII) can be cross-referenced to identify function names from log message format strings.
4. **Trace the IPC protocol.** Find the firmware's main dispatch loop (typically polling a mailbox register or a shared-memory ring buffer) and trace the handler for each message type. Document the structure layout at each message type's handler entry.

For NVIDIA Falcon firmware, `envydis -m falcon` disassembles the Falcon bytecode before bringing it into Ghidra.

### 9.2 Differential Firmware Analysis

When firmware binaries are updated, comparing consecutive versions reveals changed protocol structures without requiring full binary analysis:

```bash
# Diff two Falcon firmware versions at binary level using radiff2 (from radare2)
radiff2 -s gsp_tu10x.bin.old gsp_tu10x.bin.new

# More structured: compare function boundaries using bindiff (after Ghidra analysis)
# Areas of change indicate modified or new message types
```

Changes in consecutive firmware versions tend to be localised: a new message type adds a handler function and a corresponding enum value; a bug fix changes one function. By binary-diffing firmware versions and focusing RE effort on changed regions, the analysis time for a full firmware update can be reduced substantially.

For the Asahi Linux project, firmware comparison was particularly important for the Apple AGX firmware because Apple ships firmware updates via macOS, and comparing versions was the fastest way to identify new protocol messages introduced in each macOS release.

---

## 10. Case Study: Nouveau and envytools

The Nouveau/envytools case illustrates the full RE pipeline from raw hardware to upstream kernel driver. The following narrative describes the process as it played out for the G80 (NV50, "Tesla") architecture from roughly 2007 to 2011.

**Phase 1: Establishing BAR0 ownership.** The first challenge was accessing BAR0 registers directly from userspace. Early Nouveau work used `/dev/mem` (readable on pre-lockdown kernels) to mmap the BAR0 physical address. The `nva.ko` module was developed to provide a safer, more permanent mechanism.

**Phase 2: Boot-up register sequence.** Running `nvagetbios` extracted the VBIOS, which revealed PLL coefficient tables indexed by the GPU revision in NV_PMC_BOOT_0. The `nvbios` tool parsed these tables, giving the first symbolic register names for the clock domain registers.

**Phase 3: mmiotrace of the blob.** Using the kernel's `mmiotrace` facility (which Pekka Paalanen developed specifically for Nouveau work and upstreamed in Linux 2.6.31), Nouveau developers captured the full BAR0 register sequence produced by loading `nvidia.ko` and running the blob driver through a rendering workload. The raw addresses and values were matched against the VBIOS-derived names and extended by pattern matching: registers whose addresses were 4 bytes apart and whose values followed a predictable pattern were likely consecutive array entries.

**Phase 4: PGRAPH command stream.** `valgrind-mmt` was used to capture the pushbuffer stream produced by the blob's OpenGL implementation on G80. `demmt` decoded the stream against the NV50 method table (bootstrapped from the NV40 table as a starting point), revealing the method-value pairs needed to set up a draw call: viewport, scissor, render target address, texture bindings, vertex program address, and draw primitive.

**Phase 5: rnndb population.** Each discovered register or method became an rnndb XML entry. The `headergen2` tool generated `nv50_defs.h` from the XML, and the Nouveau kernel driver began replacing raw hex constants with symbolic names. This is the transition from "we know what the hardware does" to "we can write maintainable driver code that does it."

**Phase 6: Shader ISA.** The G80 vertex and fragment shader ISA (NV50 "VP3"/"GP4" ISA) was recovered by Marcelina Kościelnicka using the technique of compiling trivial GLSL shaders through the blob, extracting the GPU binaries, and systematically cataloguing instruction encodings. This work became the `nv50_ir` shader compiler in Nouveau/Mesa — a complete from-scratch GPU compiler built entirely from RE-derived ISA documentation.

The rnndb XML for NV50 today covers the full register space as understood — thousands of entries representing fifteen years of accumulated observation [Source](https://github.com/envytools/envytools/tree/master/rnndb).

---

## 11. Case Study: Panfrost and the Mali Job Descriptor Format

ARM Mali Midgard and Bifrost GPUs use a job-based execution model: the driver builds a job descriptor (a data structure in shared memory) describing what work the GPU should do, and submits its physical address to the GPU's Job Manager. Reverse-engineering the job descriptor format was the central challenge for the Panfrost project.

**The panwrap approach.** Alyssa Rosenzweig bootstrapped the RE work in 2017 using an LD_PRELOAD library called `panwrap` (later also called `panloader`) that intercepted calls to the ARM Mali vendor kernel driver (`mali.ko`, also known as the Midgard DDK or KBASE driver). The KBASE driver exposes ioctls through `/dev/mali0` for job submission, memory allocation, and GPU configuration. `panwrap` interposed these ioctls, capturing:

- The job descriptor structures before submission (revealing their layout)
- The memory regions allocated for job ring buffers and shared resources
- The fragment/vertex/tiler job chaining structures

The KBASE ioctl interface is the proprietary ARM kernel driver interface; it is not the same as the mainline Panfrost DRM interface that was later upstreamed.

```bash
# Capture Mali job descriptor trace using panwrap (Android development board)
# panwrap.so must be compiled for the target's ABI

LD_PRELOAD=/data/local/tmp/panwrap.so glmark2-es2

# panwrap dumps job descriptors to stderr, e.g.:
# FRAGMENT JOB @ 0xaaaab0001000:
#   [0x00] VERTEX_ARRAY = 0xaaaab0002000
#   [0x08] VARYING_LIST = 0xaaaab0003000
#   [0x10] TEXTURE_DESCRIPTOR = 0xaaaab0004000
```

**Shader ISA RE.** Connor Abbott reverse-engineered the Mali Bifrost shader ISA using the same enumerate-trivial-shaders methodology: compile minimal GLSL through the vendor compiler, extract the GPU binary, and systematically catalog the instruction encoding. This work produced the Bifrost ISA documentation used by the `bifrost_compile` shader compiler in Mesa [Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/panfrost/compiler).

**Partial ARM cooperation.** After Panfrost was accepted into the mainline Linux kernel (5.2, 2019) and demonstrated robust Mali Midgard support, ARM became significantly more cooperative. ARM provided unofficial clarifications on job descriptor semantics, confirmed or denied hypotheses about render state encoding, and eventually published some ISA documentation for Valhall. The Panthor driver (for Mali Valhall/CSF) benefited from ARM's increased transparency about the CSF (Command Stream Frontend) firmware protocol [Source](https://lwn.net/Articles/953784/).

**XDC 2018.** The methodology was publicly presented by Alyssa Rosenzweig at XDC 2018, where the slide deck described panwrap's approach and the job descriptor structures discovered for Midgard — an important moment in establishing Panfrost as a serious open-source effort with a reproducible methodology rather than a binary-copying project [Source](https://xdc2018.x.org/slides/Panfrost-XDC_2018.pdf).

---

## 12. Case Study: Asahi Linux and the Apple AGX Firmware

The Apple AGX GPU case is the most complex firmware RE effort in the Linux graphics ecosystem, because the Apple GPU's architecture is unique: all communication between the host driver and the GPU hardware passes through an Apple-proprietary firmware (the ASC — Apple System Coprocessor), meaning the host driver does not program GPU registers directly but instead sends structured messages to the firmware via shared memory.

**m1n1 hypervisor as the primary RE tool.** The Asahi Linux project developed `m1n1`, a hypervisor that runs below macOS on Apple Silicon hardware and intercepts all communication between the macOS kernel and the hardware peripherals, including the GPU ASC [Source](https://github.com/AsahiLinux/gpu). Asahi Lina wrote a GPU tracer for the m1n1 Python framework that:

1. Intercepted all reads and writes to the GPU ASC shared memory region
2. Captured the structure and timing of host-to-firmware messages (submission pipes, device control messages)
3. Captured firmware-to-host response messages (event messages, fault reports)

The m1n1 framework's Python scripting interface allowed rapid iteration: a new message type could be identified, a Python struct definition written, and the tracer updated to decode future messages of that type — all without rebooting or rebuilding a kernel module.

**Complexity of the initialization sequence.** One of the most striking findings from the AGX RE work was the sheer complexity of the GPU firmware initialization protocol. The initialization structure sent to the firmware on startup had approximately 1,000 fields across dozens of nested data structures — far more than any prior GPU firmware protocol encountered in Nouveau or Panfrost work. Many fields were discovered empirically: observing that changing a field from its macOS-provided value caused firmware crashes narrowed down which fields the firmware validated.

**Prototype Python driver.** Alyssa Rosenzweig's unusual approach to building the AGX userspace driver was to first reverse-engineer the macOS GPU driver's userspace API (the `IOAccelContext2` interface) sufficiently to allocate memory and submit commands to the GPU, then write a Python prototype driver using this interface. This allowed the userspace rendering pipeline to be developed and tested against real hardware before the Linux kernel driver existed [Source](https://asahilinux.org/2022/11/tales-of-the-m1-gpu/).

**drm-shim with embedded Python interpreter.** The Mesa driver development methodology was equally novel: the team embedded a Python interpreter inside `drm-shim` (Mesa's fake DRM kernel layer for testing), allowing the rendering pipeline to be prototyped and debugged without requiring a working kernel driver. Only once the Mesa driver was substantially working was the kernel driver (written in Rust by Asahi Lina) developed.

**AGX shader ISA.** Dougall Johnson reverse-engineered the Apple AGX shader instruction set architecture independently, producing the `dougallj/applegpu` documentation [Source](https://github.com/dougallj/applegpu). The ISA is a SIMD design with 32 lanes, with instruction encoding documented in JSON format that the `agx` compiler backend in Mesa reads for instruction selection and scheduling.

**Legal separation.** The Asahi project maintains strict procedural separation between the reverse engineering work (conducted in dedicated `#asahi-re` IRC channels, with binary disassembly restricted to trusted contributors) and the driver implementation work, to produce results that are functionally equivalent to a clean-room approach [Source](https://asahilinux.org/copyright/).

---

## 13. Case Study: etnaviv and the Vivante GC Series

The Vivante GC-series GPU (found in NXP/Freescale i.MX6, Marvell Armada, and other SoCs) presented a different RE challenge: the GPU was not widely used in desktop systems, making the userspace tools less critical, but the embedded nature of the hardware made kernel-level tracing difficult.

Christian Gmeiner and Lucas Stach led the etnaviv RE effort, which produced both a DRM kernel driver (in mainline since Linux 4.1) and a Mesa Gallium driver [Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/etnaviv).

**etna_viv: The RE repository.** The `etna_viv` repository [Source](https://github.com/etnaviv/etna_viv) contains the tools and documentation from the RE phase. The primary capture mechanism was `viv_interpose.so`, an LD_PRELOAD library intercepting the `galcore` kernel driver ioctl interface. Like Panfrost's panwrap, this captured job submissions to the GPU before they were dispatched.

**rnndb format for Vivante.** Following the envytools convention, the Vivante GPU state maps, ISA, and command stream format were documented in rnndb-compatible XML files in `etna_viv/rnndb/`. This choice allowed the `rules-ng-ng` toolchain (and specifically `librnn` from envytools) to be reused for decoding captured command streams and for generating the C header constants used in the etnaviv kernel driver.

**libvivhook: Feature bit manipulation.** An unusual tool in the etnaviv RE toolkit was `libvivhook`, which could override hardware feature bits as seen by the blob driver. The Vivante GPU exposes its hardware capabilities through feature registers read during driver initialisation. By using libvivhook to mask specific feature bits, the developer could tell the blob driver that a GPU feature was absent and observe which rendering paths it took instead — enabling a form of differential analysis that isolated which command-stream opcodes corresponded to which rendering features.

**Shader ISA.** The Vivante shader ISA was reverse-engineered using `tools/disasm.py`, a Python disassembler that decoded the ISA encoding discovered by the enumerate-trivial-shaders methodology. This Python disassembler was later replaced by a C implementation and became the basis for the etnaviv NIR shader compiler in Mesa.

---

## 14. Legal Considerations

GPU reverse engineering operates in a complex legal landscape. The key principles are:

**Interoperability exception.** In the United States, Section 1201(f) of the Digital Millennium Copyright Act (DMCA) explicitly permits reverse engineering of software "for the sole purpose of achieving interoperability of an independently created computer program with other programs." EU Directive 2009/24/EC (the Computer Programs Directive), Article 6, provides similar protection. These provisions have been upheld in landmark cases including *Sega Enterprises v. Accolade* (9th Circuit, 1992) and *Sony Computer Entertainment v. Connectix* (9th Circuit, 2000), establishing that observing software behavior for the purpose of achieving interoperability is not copyright infringement.

**Clean-room methodology.** The classical clean-room approach — used by Phoenix Technologies to build an IBM-compatible BIOS — involves strictly separating the RE team (who document what the hardware does) from the implementation team (who write the driver based only on that documentation, never having seen the proprietary code). This provides the strongest legal protection because independent creation is a complete defense against copyright infringement claims.

Most open-source GPU drivers do not apply strict clean-room separation because the overhead is prohibitive for volunteer projects. However, the *absence* of clean-room methodology does not make the driver illegal: reverse engineering for interoperability is permitted regardless of whether the same person both reverse engineers and implements.

**GPL compatibility of RE-derived drivers.** Drivers written by observing hardware behavior are entirely original works authored by the driver developers. They are not derivative works of the proprietary driver binary under copyright law, even if they produce the same observable hardware effects. The GPL kernel driver license applies to the driver authors' own source code, which is independently copyrightable.

**NVIDIA's stance on Nouveau.** NVIDIA's EULA for the proprietary driver prohibits reverse engineering. However, the DMCA's Section 1201(f) interoperability exception takes precedence over EULAs for the purpose of enabling interoperability, and no legal action has been brought against the Nouveau project in its fifteen-year history despite direct competition with the proprietary driver.

**ARM's semi-cooperation with Panfrost.** ARM's response to Panfrost evolved from initial silence to active cooperation. After Panfrost was upstreamed in Linux 5.2 (2019), ARM hired developers to contribute to the open-source driver and eventually provided official documentation for certain aspects of the Valhall GPU architecture. This is a different model from strict RE: the open-source driver incorporated ARM's officially provided information alongside the originally RE-derived knowledge. ARM's cooperation does not retroactively affect the legality of the initial RE work.

**The Asahi model.** Asahi Linux's approach — documented in their copyright policy — takes the legally careful position of treating RE-derived knowledge and driver implementation as functionally equivalent to clean-room results, while maintaining procedural separations within the project [Source](https://asahilinux.org/copyright/). Binary disassembly is restricted to trusted contributors who document their findings as clean specifications; other developers implement from those specifications without examining the binary directly.

**What RE cannot fix: patent exposure.** The interoperability exception addresses copyright only. GPU implementations may be covered by hardware patents. Copyright law does not protect register encodings (facts about hardware are not copyrightable), but patent law may protect specific circuit designs or algorithms. In practice, patent claims against open-source GPU drivers have not materialized, partly because the drivers enable wider use of the hardware vendor's own products.

---

## 15. Integrations

This chapter provides the methodological foundation for understanding several other chapters:

- **[Chapter 7 (Reverse Engineering NVIDIA: History and Methodology)](../../part-03-nouveau-story/ch07-reverse-engineering-nvidia.md)** — Covers the historical narrative of Nouveau's RE work in detail, including the political context of the Aalto University incident and NVIDIA's 2022 open-gpu-kernel-modules release. This chapter provides the tools and techniques that chapter describes in use.

- **[Chapter 8 (Nouveau Kernel Driver)](../../part-03-nouveau-story/ch08-nouveau-kernel-driver.md)** — The Nouveau kernel driver's `nvhw/` headers are generated from rnndb XML by `headergen2`. The `nvkm_rd32()`/`nvkm_wr32()` access macros and the symbolic register constants throughout `drivers/gpu/drm/nouveau/` are the direct product of the valgrind-mmt → demmt → rnndb pipeline described in this chapter.

- **[Chapter 90 (Panfrost, Panthor, and Lima)](../../part-02-gpu-drivers/ch90-panfrost-panthor-lima.md)** — The Panfrost driver's job descriptor format and shader ISA understanding were established by the panwrap/panloader methodology described in Section 11. The Panthor driver's CSF firmware protocol is the product of the firmware RE approaches described in Section 9.

- **[Chapter 73 (Asahi Linux and Apple Silicon)](../../part-02-gpu-drivers/ch73-asahi-apple-silicon.md)** — The AGX driver's command list format and firmware protocol understanding came from the m1n1 hypervisor tracing described in Section 12. The Rust kernel driver and the Mesa AGX compiler backend are built on the RE-derived documentation.

- **[Chapter 160 (freedreno and turnip: Adreno on Linux)](../../part-02-gpu-drivers/ch160-freedreno-turnip-adreno.md)** — The freedreno `.rd` file format, `cffdump`, and `libwrap` capture methodology described in Section 7.2 underpin the freedreno/turnip development workflow for reproducing GPU faults and understanding command-stream format.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
