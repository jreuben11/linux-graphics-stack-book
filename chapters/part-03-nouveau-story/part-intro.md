# Part III — The Open NVIDIA Stack

**Audiences:** Systems and driver developers; graphics application developers who need to understand
what makes NVIDIA hardware behave differently from AMD or Intel under open-source drivers.

---

## Table of Contents

1. [A Cautionary Tale — and a Triumph](#a-cautionary-tale--and-a-triumph)
2. [Key Concepts](#key-concepts)
   - [MMIO Registers and the BAR0 Window](#mmio-registers-and-the-bar0-window)
   - [Pushbuffers and the Host FIFO](#pushbuffers-and-the-host-fifo)
   - [PCI BAR Addressing and DMA Coherency](#pci-bar-addressing-and-dma-coherency)
   - [VBIOS, P-states, and PLL Clock Trees](#vbios-p-states-and-pll-clock-trees)
   - [Nouveau UAPI](#nouveau-uapi)
   - [Linux Auxiliary Bus](#linux-auxiliary-bus)
   - [Bindless Global Descriptor Buffer](#bindless-global-descriptor-buffer)
3. [The Narrative Arc of Part III](#the-narrative-arc-of-part-iii)
4. [Chapter Dependency Diagram](#chapter-dependency-diagram)
5. [Where Part III Fits in the Book](#where-part-iii-fits-in-the-book)

---

## A Cautionary Tale — and a Triumph

For the better part of two decades — from the first `nouveau` commits in 2006 through the GSP-RM
open-source release in 2022 — the NVIDIA graphics subsystem on Linux was the most capable GPU
hardware with the least capable open-source driver. Every other GPU class covered in this book
— Intel integrated graphics, AMD Radeon, Qualcomm Adreno, ARM Mali — eventually reached a state
where the open driver matched or approached proprietary performance. NVIDIA did not. The gap was
not the fault of the engineers who wrote Nouveau; it was the direct consequence of a business
decision to withhold hardware documentation while shipping a monolithic binary kernel module that
locked out any co-development.

The community response, sustained over 17 years, was systematic reverse engineering. Engineers
captured MMIO traffic from the proprietary driver using the `mmiotrace` infrastructure
[Source](https://www.kernel.org/doc/html/latest/trace/mmiotrace.html), decoded the register
sequences by hand, and built the Envytools suite — an XML register database (`rnndb/`), a
disassembler for NVIDIA's proprietary microcontroller ISAs (`envydis`), and a capture-analysis
tool (`demmt`) — to turn raw observations into symbolic kernel knowledge
[Source](https://github.com/envytools/envytools). The result was a kernel driver (`nouveau`) that
ran on every NVIDIA GPU since 2004 but could not reclock the GPU to its rated performance because
doing so required VBIOS P-state tables and signed PMU firmware that the community had never seen.

Then, in May 2022, NVIDIA opened the nvidia-open kernel module
[Source](https://developer.nvidia.com/blog/nvidia-releases-open-source-gpu-kernel-modules/), and in
the months that followed, Nouveau acquired the ability to boot using NVIDIA's own GSP-RM firmware,
unlocking reclocking and modern display support on Turing and later GPUs. Simultaneously, Faith
Ekstrand (then at Collabora) began building NVK — a clean-slate Vulkan driver for NVIDIA hardware
inside Mesa — that reached Vulkan 1.0 conformance in 2023 and has since become the recommended
path for open-source NVIDIA rendering [Source](https://www.collabora.com/news-and-blog/news-and-events/introducing-nvk.html).
The contrast with RADV (AMD's open Vulkan driver, written with full hardware documentation and
AMD engineering support, reaching conformance in roughly two years) illustrates the cost of
documentation: NVK required roughly the same calendar time as RADV, but only became possible
*after* the firmware documentation gap was partially closed by GSP-RM.

Part III tells this story in seven chapters: from the methodology of reverse engineering (Ch. 7),
through the nvkm kernel driver architecture (Ch. 8), the GSP-RM firmware turning point (Ch. 9),
the Nova clean-sheet Rust driver (Ch. 10a), the NVK Vulkan driver (Ch. 10b), the NAK Rust shader
compiler that forms the shader-compilation half of the NVK stack (Ch. 118), and finally the
display, reclocking, and power management picture that ties the hardware to the Linux power
management framework (Ch. 11). It is not merely a "how does the driver work" account — it is a
record of how an open driver was built despite systematic obstacles, and what the architecture
looks like now that those obstacles have partly lifted.

---

## Key Concepts

The following concepts are specific to NVIDIA hardware and the Nouveau/NVK approach. Later
chapters assume familiarity with them.

### MMIO Registers and the BAR0 Window

A **Memory-Mapped I/O (MMIO) register** is a GPU control register exposed at a fixed physical
address. The kernel maps this address into virtual address space with `ioremap()`, producing a
pointer through which register reads and writes generate PCIe transactions to the GPU. NVIDIA GPUs
expose their primary register file through **BAR0** (Base Address Register 0), a 16 MB or 32 MB
PCIe aperture that maps the NV_PBUS, NV_PFIFO, NV_PGRAPH, NV_PMC, and related register blocks.
Nouveau accesses this window through `nvif_rd32()` / `nvif_wr32()` which ultimately call
`ioread32()` / `iowrite32()` on the BAR0 mapping
[Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/nouveau/include/nvif/io.h).
A second aperture, **BAR1**, maps a configurable window into GPU VRAM that the CPU can read and
write directly — important for buffer allocation strategies where avoiding PCIe round-trips
matters. Chapter 7 explains how `mmiotrace` captures every BAR0 access the proprietary driver
makes, and Chapter 8 shows how those captured sequences are encoded as register operations in the
`nvkm` object model.

### Pushbuffers and the Host FIFO

NVIDIA GPUs do not accept individual register writes as rendering commands. Instead, the driver
assembles a **pushbuffer**: a contiguous block of GPU-accessible memory containing a stream of
*methods* — 32-bit words encoding (subchannel ID, method offset, data) tuples — that the GPU's
**host FIFO (PFIFO)** engine reads and dispatches to the appropriate hardware engine (graphics,
compute, copy). This is NVIDIA's equivalent of AMD's ring buffer or Intel's batchbuffer. The
driver submits work by writing the pushbuffer's GPU virtual address and length into a channel
control structure, then doorbell-ringing the PFIFO to schedule it.

In Nouveau, the pushbuffer mechanism is managed through `nvkm_chan` (channel) objects and the
`drm_sched` GPU scheduler [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/nouveau/nvkm/engine/fifo/). The
modern userspace interface — `DRM_IOCTL_NOUVEAU_EXEC` — accepts an array of push buffer
descriptors from NVK and submits them as a single scheduler job. Understanding the pushbuffer
model is essential for Chapter 8 (how nvkm manages channels) and Chapter 10b (how NVK builds
command streams before submitting them).

### PCI BAR Addressing and DMA Coherency

PCIe **Base Address Registers (BARs)** are the physical address windows through which the CPU
accesses GPU resources. Beyond BAR0 (MMIO registers) and BAR1 (VRAM aperture), NVIDIA GPUs
expose a BAR2 aperture used for privileged register access (the DP/PRIV region). GPU drivers
declare a **DMA mask** (`dma_set_mask_and_coherent(dev, DMA_BIT_MASK(40))` or 64-bit equivalent)
to tell the kernel which address bits the GPU's DMA engine can use when writing into system RAM.

**DMA coherency** on PCIe is asymmetric: the GPU can write to system RAM via posted PCIe writes
that are not guaranteed to be visible to the CPU until the CPU executes a memory barrier (a fence)
or reads a completion register. This is why Nouveau's fence infrastructure exists: the GPU writes
a *fence value* to a system RAM location as the final operation in a pushbuffer, and the CPU
polls that location (or sleeps on it via `drm_syncobj` timeline semantics) to know when GPU work
is complete. Chapter 4 covers the DMA-BUF and implicit-sync fence infrastructure; Chapter 8
explains how Nouveau implements these abstractions.

### VBIOS, P-states, and PLL Clock Trees

The **Video BIOS (VBIOS)** is a firmware blob stored in a SPI flash chip on the GPU PCB and
shadowed into BAR0 at a fixed offset (typically `0x619xxx` on pre-Turing GPUs). It contains
tables describing the GPU's **performance states (P-states)**: P0 is maximum performance, P8 (or
higher) is idle/minimum. Each P-state entry records target frequencies for the GPU's clock
domains — GPC clock (shader clock), memory clock, and display clocks — as well as voltage targets
and fan speed parameters.

Clock generation uses **Phase-Locked Loop (PLL)** circuits. Each PLL multiplies a reference
oscillator (typically 27 MHz) by a rational factor encoded as a (M, N, P) coefficient triple.
Computing the correct PLL coefficients for a target frequency, writing them to the appropriate
BAR0 registers, and verifying phase lock is a multi-step sequence that requires knowing the
frequency limits of each PLL (found in the VBIOS). Because this table was never documented,
Nouveau could only reclock NV50 and Fermi GPUs (where the PLL sequence was fully reverse-engineered)
and could not boost Maxwell and later GPUs to their rated clocks without signed PMU firmware.
Chapter 11 covers reclocking in detail; Chapter 9 explains how GSP-RM finally resolves this gap.

### Nouveau UAPI

Nouveau exposes its kernel interface to userspace via DRM ioctls defined in
[`include/uapi/drm/nouveau_drm.h`](https://github.com/torvalds/linux/blob/master/include/uapi/drm/nouveau_drm.h).
The legacy ioctls (`DRM_IOCTL_NOUVEAU_GETPARAM`, `DRM_IOCTL_NOUVEAU_GEM_NEW`,
`DRM_IOCTL_NOUVEAU_GEM_PUSHBUF`) support the old Mesa GL driver. NVK uses a newer UAPI:
`DRM_IOCTL_NOUVEAU_VM_BIND` for GPU virtual address space management (allocating and mapping BO
ranges without a per-submission address fixup), and `DRM_IOCTL_NOUVEAU_EXEC` for command
submission with explicit `drm_syncobj` timeline acquire/release points. This VM_BIND-style UAPI
mirrors the direction taken by `i915` (for `xe`) and `amdgpu`; it permits the userspace driver to
manage a persistent GPU virtual address space and submit GPU work with explicit synchronisation
rather than relying on the kernel to emit implicit fence completions. Chapter 8 covers both the
old and new UAPI; Chapter 10b shows how NVK uses the new UAPI.

### Linux Auxiliary Bus

The **Linux Auxiliary Bus** (`drivers/base/auxiliary.c`) is a kernel mechanism for a parent
driver to create child sub-devices that bind their own separate drivers
[Source](https://www.kernel.org/doc/html/latest/driver-api/auxiliary_bus.html). A parent calls
`auxiliary_device_register()` to publish a named sub-device; a child driver calls
`auxiliary_driver_register()` and the kernel's probe machinery matches and binds them. NVIDIA uses
this in `nvidia-open` to split the kernel module into functional units; Nouveau uses it to
register the `nvkm_gsp` GSP-RM sub-device (which loads the GSP firmware and owns the RPC
channel) as an auxiliary child of the top-level `nvkm_device`. In Nova, the split between
`nova-core` (GSP firmware management, auxiliary device provider) and `nova-drm` (DRM interface,
auxiliary driver consumer) is the central architectural seam. Chapter 9 explains the GSP-RM use
of the auxiliary bus in depth; Chapter 10a (Nova) is built around it.

### Bindless Global Descriptor Buffer

Vulkan descriptor sets on most hardware are allocated from a pool, laid out in a format the driver
and hardware agree on, and bound to the pipeline before a draw call. NVIDIA's hardware supports a
**bindless** model where the shader directly indexes into a **global descriptor table** stored in
GPU VA space — a single large buffer in which each descriptor slot holds a (GPU VA, size, format)
tuple that the hardware can dereference without explicit descriptor set binding. NVK exploits this
to implement Vulkan descriptor sets efficiently: rather than copying descriptor data into a
hardware-specific layout at vkUpdateDescriptorSets time, NVK writes descriptors into a
heap-allocated slot in the global buffer and encodes the buffer address as a 64-bit push constant.
This eliminates descriptor set copies during `vkCmdBindDescriptorSets` for many workloads and
is a significant architectural divergence from AMD RADV, which must program hardware descriptor
tables via userspace ring buffer writes. Chapter 10b examines this in detail within NVK's
descriptor set implementation.

### What Is Now Open — and What Remains Proprietary

The 2022 nvidia-open release and the subsequent NVK/Nova/NAK work represent a genuine and significant shift. But the boundary between open and closed is precise, and understanding it exactly matters for anyone building production systems on NVIDIA hardware.

**What is now open (source-available and community-maintainable):**

- **nvidia-open host code**: The `nvidia-open` kernel module published at [github.com/NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules) is the host-side RPC client — the code that runs on the CPU, constructs GSP-RM RPC calls, and manages the DMA rings. It is licensed under MIT/GPL dual licence. This is what lets Nouveau (and Nova) communicate with GSP-RM without reverse engineering the RPC protocol.
- **NVK**: Mesa's Vulkan driver (`src/nouveau/vulkan/`) is fully open source, written from scratch by Faith Ekstrand and contributors. It holds Vulkan 1.4 conformance on Mesa 25.x for Turing (TU) through Ada (AD) GPUs, with Hopper (GH) and Blackwell (GB) support under development.
- **NAK (Nvidia Awesome Kompiler)**: Mesa's Rust shader compiler backend (`src/nouveau/compiler/`) that compiles NIR to NVIDIA SASS ISA for Maxwell through Hopper. Fully open, merged in Mesa 24.0.
- **Nova**: The Rust reimplementation of the kernel driver (`drivers/gpu/drm/nova/`), merged in Linux 6.15. `nova-core` manages GSP-RM communication; `nova-drm` provides the DRM interface. Both are GPL and actively maintained in-tree.
- **Envytools / rnndb**: The register database, microcontroller disassemblers, and ISA documentation accumulated over 17 years of reverse engineering. These remain the foundation of the SASS ISA knowledge that NAK encodes.

**What remains proprietary (binary-only, not open source):**

- **GSP-RM firmware binary** (`gsp_tu10x.bin`, `gsp_ga10x.bin`, `gsp_ad10x.bin`, etc.): This is a signed ARM Falcon / RISC-V binary blob loaded via `request_firmware()`. The firmware runs the entire GPU resource manager on the GPU's system processor — handling memory management, power transitions, and fault recovery. It is distributed via the linux-firmware repository under a restricted licence that permits redistribution but not modification. **Without this blob, no NVK, Nova, or nvidia-open frame reaches the screen on Turing and later hardware.** The firmware is signed by NVIDIA and cannot be replaced or modified by the community.
- **VBIOS**: The Video BIOS ROM stored on GPU flash is still a closed binary. On GSP-RM-based GPUs, the VBIOS is loaded and consumed by GSP-RM directly; the host driver does not need to parse it directly, but the content is not documented.
- **Display PHY microcode**: The firmware for NVIDIA's display engine PLL/PHY units (the `hdmi_gsp_*.bin` blobs for HDMI 2.1 on Ampere+ hardware) is a separate signed binary. Without it, HDMI output at 4K/120Hz is not available. This is distinct from the main GSP-RM blob.
- **Hardware register and ISA documentation for unreleased generations**: NVIDIA still does not publish register-level documentation or hardware architecture manuals. The Envytools rnndb database covers generations through Ampere (GA10x); Hopper (GH100) and Blackwell (GB200) ISA details are known only through the GSP-RM open-source code's embedded #defines and through NAK's reverse-engineered instruction encoding tables.

**Practical consequence**: The open NVIDIA stack on Linux requires exactly one proprietary artefact — the GSP-RM firmware blob — for all display and GPU functionality on Turing (2018) and later hardware. Pre-Turing GPUs (Maxwell, Pascal) can in principle function without GSP-RM, but face the historical reclocking limitations documented in Chapter 11.

### RT-Cores and Hardware Ray Tracing on NVIDIA

NVIDIA introduced dedicated ray tracing hardware units — **RT-Cores** — with the Turing architecture (TU102, 2018). RT-Cores are fixed-function BVH traversal accelerators integrated into each SM. When a shader executes a ray tracing instruction, the SM dispatches the ray to the RT-Core hardware, which walks the two-level BVH (`TLAS` → `BLAS` tree) and returns the nearest intersection without consuming shader execution cycles on the main ALU units. This is the hardware that underpins the Vulkan `VK_KHR_ray_tracing_pipeline` and `VK_KHR_acceleration_structure` extensions.

**Status on the open stack:**

- **NVK exposes `VK_KHR_ray_tracing_pipeline` on Turing+ GPUs** (confirmed in Mesa 24.2+). The extension requires hardware support for BVH traversal, shader binding tables, and callable shaders — all of which are routed through GSP-RM when NVK submits work.
- **`VK_KHR_acceleration_structure`** (BVH build/query) is also supported. NVK compiles the BVH build shaders through NAK and submits the BVH construction as a compute dispatch; the actual traversal hardware is invoked at ray dispatch time.
- **The RT-Core microarchitecture is not publicly documented.** NVIDIA has not published the internal register interface for the RT-Core units, the BVH node format constraints beyond what the Vulkan spec requires, or the performance characteristics of the traversal hardware. Functional access is available through the standard Vulkan ray tracing API — which NVK implements correctly — but low-level tuning (e.g., optimal BVH node size, hardware traversal batch depth) relies on inference from performance measurements rather than architectural documentation.
- **Ampere (GA10x, 2020) and Ada Lovelace (AD10x, 2022)** added 2nd and 3rd generation RT-Cores with improved throughput. All three generations are accessible through the same Vulkan API surface; differences are observable only through performance, not through API capability flags.

Chapter 10b examines the NVK acceleration structure implementation; Chapter 135 covers Vulkan ray tracing across all open and proprietary Linux drivers including RADV (AMD RDNA2+) and RADV+ACO vs NVK+NAK compile-time differences.

---

## Chapters in This Part

**Chapter 7 — Reverse Engineering NVIDIA: History and Methodology** explains how `mmiotrace` captures MMIO traffic, how Envytools turns captured traces into a symbolic register database, and why 17 years of reverse engineering was insufficient to fully close the reclocking gap.

**Chapter 8 — The Nouveau Kernel Driver: nvkm Architecture** covers the `nvkm_device` / `nvkm_subdev` / `nvkm_engine` object hierarchy, Falcon microcontroller abstractions, channel and pushbuffer management, `drm_sched` integration, and the full DRM UAPI surface.

**Chapter 9 — GSP-RM, Firmware, and the nvidia-open Connection** explains what changed in 2022: NVIDIA's GPU System Processor runs a complete resource manager (GSP-RM) exposing an RPC interface over dual 256 KB ring queues, and how Nouveau acquired GSP-RM support shortly after.

**Chapter 10a — Nova: The Rust NVIDIA Kernel Driver** examines the clean-sheet Rust reimplementation introduced in Linux 6.15, whose nova-core / nova-drm split across the Linux Auxiliary Bus encodes the GSP-RM trust boundary in Rust's type system.

**Chapter 10b — NVK: Building a Vulkan Driver from Scratch** covers the Mesa Vulkan driver for NVIDIA hardware: the SPIR-V → NIR → NAK shader compilation pipeline, the bindless descriptor heap, and VM_BIND command submission.

**Chapter 118 — NAK: The Rust Shader Compiler for NVIDIA GPUs** documents the Nvidia Awesome Kompiler — Mesa's first GPU compiler backend written in Rust — covering its NIR-to-SASS lowering pipeline, register allocation, and ISA encoding for Maxwell through Hopper.

**Chapter 11 — Display, Reclocking, and Power Management** closes the part by covering NVIDIA display engine reclocking, P-state tables, PLL clock trees, and the power management picture that varies significantly by GPU generation.

## The Narrative Arc of Part III

The seven chapters form a progression from epistemology to architecture to application:

**Chapter 7 — Reverse Engineering NVIDIA: History and Methodology** establishes where the
knowledge in everything that follows comes from. It is a chapter about method: how `mmiotrace`
works, how Envytools turns captured traces into a symbolic register database, how that database
becomes the header constants referenced throughout `nvkm`. It is also a chapter about limits: why
17 years of reverse engineering was insufficient to fully close the reclocking gap, and what that
tells us about the fundamental information asymmetry between a hardware vendor and an open-source
community without hardware documentation.

**Chapter 8 — The Nouveau Kernel Driver: nvkm Architecture** is the structural centre of the
part. It covers the `nvkm_device` / `nvkm_subdev` / `nvkm_engine` object hierarchy, Falcon
microcontroller abstractions, channel and pushbuffer management, `drm_sched` integration, and
the full DRM UAPI surface. The kernel abstractions Chapter 8 defines — channels, fences,
`drm_gpuvm` mappings — are the interfaces that Chapters 9, 10a, 10b, and 11 all build upon.

**Chapter 9 — GSP-RM, Firmware, and the nvidia-open Connection** is the pivot. It explains what
changed in 2022: NVIDIA's GPU System Processor runs a complete resource manager (GSP-RM) on an
embedded Falcon v6 core, exposing an RPC interface over dual 256 KB ring queues. The nvidia-open
release opened the host-side RPC client code; Nouveau acquired GSP-RM support shortly after.
Chapter 9 also draws the honest boundary between what is now open and what remains in signed
firmware blobs, and compares the trust model to AMD PSP and Intel HuC/GuC.

**Chapter 10a — Nova: The Rust NVIDIA Kernel Driver** examines the clean-sheet reimplementation
introduced in Linux 6.15. Nova treats GSP-RM as the sole hardware authority and encodes that
trust boundary in Rust's type system — `FirmwareObject<F, Loaded>` typestate, `PinInit`-based
pinned initialisation, `bounded_enum!`-validated hardware constants. The nova-core / nova-drm
split across the Linux Auxiliary Bus is the architectural statement: the two components have
different stability contracts and can evolve independently.

**Chapter 10b — NVK: Building a Vulkan Driver from Scratch** covers the Mesa counterpart. NVK
is what a Vulkan driver looks like when its authors can choose freely — no legacy GL state tracker
to preserve, no hardware workarounds inherited from a prior generation. The SPIR-V → NIR → NAK
(Nouveau Assembler Kit) shader compilation pipeline, the bindless descriptor heap, and VM_BIND
command submission are designed together rather than retrofitted. NVK also provides a worked
example of how to use Mesa's Vulkan common infrastructure, valuable to anyone writing a new Mesa
Vulkan driver.

**Chapter 118 — NAK: The Rust Shader Compiler for NVIDIA GPUs** is the shader-compilation half
of the NVK story. NVK hands compiled shaders off to NAK — the Nvidia Awesome Kompiler — which
ingests NIR, applies NVIDIA-specific optimization and lowering passes, allocates registers across
NVIDIA's banked GPR file and (on Turing+) the uniform register file, schedules instructions for
latency hiding, and emits binary SASS for Maxwell through Hopper ISAs. NAK is Mesa's first GPU
compiler backend written in Rust, merged in Mesa 24.0 (February 2024), and its clean
SSA-throughout design replaced the legacy nv50_ir C++ backend that could not correctly target
Turing or later GPU generations. The chapter covers NAK's IR design, its NIR-to-SASS lowering
pipeline (including `nak_compile_shader()` at the NVK call boundary), register allocation
correctness improvements over nv50_ir, Rust-in-Mesa FFI mechanics, and NAK's influence on
subsequent Mesa Rust compiler efforts such as KRAID (ARM Mali Valhall).

**Chapter 11 — Display, Reclocking, and Power Management** closes the part by anchoring the
hardware concerns that affect every NVIDIA user on Linux: can the display engine drive the
connected monitor correctly, can the GPU clock up to its rated performance, and does it idle
efficiently when not under load? The answer varies significantly by GPU generation — fully yes on
Turing+ with GSP-RM, partially on Maxwell/Pascal, and fully reverse-engineered only on NV50/Fermi.
Chapter 11 explains why the boundary falls where it does.

---

## Chapter Dependency Diagram

The diagram below shows the reading dependencies among Part III chapters. An arrow from A to B
means B builds on concepts or interfaces introduced in A.

```mermaid
graph TD
    Ch7["Ch. 7 — Reverse Engineering Methodology<br/>(mmiotrace, Envytools, rnndb)"]
    Ch8["Ch. 8 — Nouveau Kernel Driver: nvkm<br/>(nvkm_device, channels, UAPI)"]
    Ch9["Ch. 9 — GSP-RM Firmware<br/>(RPC interface, nvidia-open)"]
    Ch10a["Ch. 10a — Nova: Rust Kernel Driver<br/>(nova-core / nova-drm, Auxiliary Bus)"]
    Ch10b["Ch. 10b — NVK: Vulkan Driver<br/>(SPIR-V → NIR → NAK, VM_BIND)"]
    Ch118["Ch. 118 — NAK: Rust Shader Compiler<br/>(NIR → SASS, register allocation, ISA encoding)"]
    Ch11["Ch. 11 — Display, Reclocking, Power<br/>(P-states, PLL, DRM KMS)"]

    Ch7 --> Ch8
    Ch8 --> Ch9
    Ch9 --> Ch10a
    Ch9 --> Ch10b
    Ch8 --> Ch10b
    Ch10b --> Ch118
    Ch8 --> Ch11
    Ch9 --> Ch11
```

Ch. 118 (NAK) depends on Ch. 10b (NVK) because NAK is the shader-compilation backend that NVK
invokes via `nak_compile_shader()`; understanding NVK's pipeline layout and descriptor model
provides the necessary context for why NAK's NIR ingestion interface is designed the way it is.
Readers focused specifically on compiler internals may read Ch. 118 after Ch. 7 (for ISA
background from Envytools) and before Ch. 10b, treating NVK as the consumer rather than the
prerequisite.

---

## Where Part III Fits in the Book

Part III occupies the same tier as Part II (GPU Drivers) in the kernel stack, but at a finer
granularity: where Part II surveys multiple GPU vendor drivers at a structural level, Part III
traces one vendor's driver in depth precisely because that driver's history is the most
illuminating case study in the cost of closed hardware documentation. The contrast with AMD (whose
RADV driver was written with hardware specifications and reached conformance quickly) and Intel
(whose open driver predates Mesa itself) is drawn explicitly in Chapter 7 and revisited in Chapter
10b.

Parts IV and V (Mesa Architecture, Mesa GPU Drivers) consume the kernel and Mesa interfaces this
part defines: NVK sits within the Part V Vulkan driver layer, and the GPU UAPI surface from
Chapter 8 is the kernel interface those Mesa chapters assume exists. NAK (Ch. 118) is also the
shader-compiler half of that driver layer — the NIR-to-SASS pipeline it defines is the NVIDIA
analogue of the NIR-to-ISA backends covered for AMD (ACO) and Intel (brw/elk) in Part V. Part VI
(Display Stack) relies on the `drm_syncobj` explicit-sync infrastructure explained in Chapters
9–10, which is what finally enabled first-class NVIDIA Wayland support. The power management
picture in Chapter 11 provides context for the profiling and tuning chapters in Part IX. NAK's
Rust-in-Mesa integration pattern (Chapter 118) is also a cross-cutting reference for Part IV
(Mesa Architecture) chapters that address the Mesa build system and compiler infrastructure.

Readers arriving directly at Part III should have read Parts I and II, particularly the DRM/KMS
architecture (Ch. 1–2), GPU memory management (Ch. 4), and the structural GPU driver survey
(Ch. 5). Those chapters establish the `drm_device`, `drm_gem_object`, `drm_sched`, and
`drm_syncobj` abstractions that nvkm, Nova, and NVK all build upon.

---

*Part III spans Chapters 7–11 and Chapter 118, totalling seven chapters and approximately 90–120 pages.*

---

## Part Roadmap Summary

*Synthesised from the Roadmap sections of this part's chapters.*

### Near-term (6–12 months)

- **Hopper and Blackwell enablement across the entire stack.** The 62-patch series adding Nouveau GSP-RM support for GH100, GB10x, and GB20x GPUs (using R570-series signed firmware blobs from `linux-firmware.git`) is the dominant theme across Ch. 7, 8, 9, and 11. Nova's v12+ Hopper/Blackwell enablement patch series targets Linux 7.3–7.4. NVK conformance testing on Hopper-class hardware follows once the kernel side stabilises.
- **GSP-RM as the default path on Turing+.** Linux 6.18 makes `NvGspRm=1` the default for all Turing and later GPUs, bringing automatic reclocking and display engine initialisation via firmware to a much wider installed base without manual kernel parameters (Ch. 9, Ch. 11).
- **Nova reaching functional usability.** The `DRM_NOVA_EXEC` command-submission ioctl and `drm_gpuvm`-backed GPUVM integration (Linux 7.2 preparation already merged) are the immediate gates for Nova transitioning from an infrastructure skeleton to a driver that runs real GPU workloads. `drm_syncobj` timeline support follows, enabling Wayland explicit-sync participation (Ch. 10a).
- **NAK compiler quality improvements.** Register allocation quality is the primary performance gap between NVK and the proprietary driver; incremental improvements ship each Mesa release (Mesa 26.x). Delta Color Compression (DCC) for render targets — expected to halve memory bandwidth on supported hardware — requires coordinated NAK and nvkm changes and was in late draft at XDC 2025 (Ch. 118, Ch. 10b).
- **Vulkan extension backlog.** `VK_KHR_maintenance10`, `VK_EXT_discard_rectangles`, DLSS via `VK_NVX_binary_import`, and a Zink/OpenGL regression triage following the Mesa 25.1 default switch to NVK+Zink are active near-term items for NVK (Ch. 10b, Ch. 118).
- **Nova `fwctl` interface and voltage-scaling exposure.** March 2026 patches add a `fwctl` driver to `nova-core` enabling userspace GSP RPC commands; parallel work exposes `nvkm_volt` GPU voltage as a writable hwmon attribute on Kepler/early Maxwell hardware (Ch. 11).

### Medium-term (1–3 years)

- **Nova superseding Nouveau for Turing+ hardware.** The architectural direction agreed by Red Hat, NVIDIA, and the Nouveau maintainers is that Nova's clean-sheet Rust implementation replaces nvkm's Turing+ code paths, while Nouveau retains the pre-Turing (NV50 through Volta) installed base in maintenance mode. NVK's NVKMD abstraction layer is explicitly designed so the same Mesa Vulkan userspace can target either kernel driver without structural changes (Ch. 8, Ch. 9, Ch. 10a, Ch. 10b, Ch. 11).
- **Nova `nova-drm` KMS and display support.** After compute/render stabilisation, display engine support (NVDisplay, HDMI/DP SOR, atomic KMS commit) is the next major Nova feature area, eventually porting or rewriting the `dispnv50/` codebase in Rust (Ch. 10a, Ch. 11).
- **Ray tracing and hardware-accelerated video.** `VK_KHR_ray_tracing_pipeline` and `VK_KHR_acceleration_structure` require both NAK instruction encoders for RT-specific SASS opcodes and NVK BVH management — progress is contingent on further reverse engineering of RT Core internals or NVIDIA disclosure. NVENC/NVDEC via GSP-RM RPC is a parallel track, with partial documentation available in `open-gpu-doc` (Ch. 10b, Ch. 118, Ch. 9).
- **Rust-in-Mesa compiler cross-pollination.** KRAID (ARM Mali Valhall Rust compiler, merged June 2026) modelled its IR on NAK; shared register-allocator and NIR-lowering infrastructure is expected to be factored into a common Rust crate, reducing duplication between the two backends and establishing Rust as the default pattern for new Mesa compiler backends (Ch. 118).
- **Advanced display and power management features.** HDR full pipeline support (requiring the Wayland `color-management-v1` protocol, Mesa NVK colour management, and nouveau KMS coordination), UHBR10/13.5/20 (DP 2.0) link training on Ada, GPU Boost/per-application clock hinting via GSP RPC, and improved D3cold resume latency for PRIME multi-GPU laptops are all active medium-term development areas (Ch. 11).

### Long-term

- **Nova as the reference Rust DRM driver.** Alongside the ARM Tyr driver, Nova is positioned in `drm-rust-next` as one of two canonical Rust DRM driver implementations. The architectural choices it encodes — `FirmwareObject<F, S>` typestate, `bounded_enum!`-validated MMIO, auxiliary bus split — are expected to become the conventions for future Rust DRM drivers across GPU vendors, documented in `Documentation/gpu/nova/guidelines.rst`.
- **Open compute ecosystem without proprietary toolchain.** The Tinygrad Mesa NIR backend (merged January 2026) makes NAK the critical path for free-software NVIDIA ML compute. Long-term, NAK coverage of cooperative matrix, Hopper collective-memory (`HGMMA`, `BGMMA`, `ACQBULK`), and PTX semantics not currently reachable from NIR could enable a CUDA-compatible compute path and Rusticl/OpenCL integration on NVK — reducing dependency on the proprietary CUDA runtime (Ch. 118, Ch. 10b, Ch. 9).
- **Knowledge preservation for pre-Turing hardware.** As GSP-RM becomes the baseline for all post-2018 NVIDIA GPUs, the Envytools rnndb and the 17 years of mmiotrace-derived register knowledge for NV50 through Volta represent irreplaceable documentation of hardware the vendor will never officially document. Ensuring rnndb accuracy and discoverability — including eventual Blackwell entries backfilled from open-gpu-doc headers — is a long-horizon community responsibility independent of any feature work (Ch. 7).
- **Unified open-source power management and GSP firmware transparency.** The long-term vision is a coherent voltage/clock/thermal/fan abstraction spanning Kepler through Ada, with GSP-RM handling firmware-signed operations and a clean kernel sysfs/hwmon API exposed to userspace. Competitive and regulatory pressure may eventually motivate NVIDIA to open-source GSP-RM firmware source or publish signing infrastructure for community builds, though no concrete signal for this exists as of 2026 (Ch. 9, Ch. 11).
