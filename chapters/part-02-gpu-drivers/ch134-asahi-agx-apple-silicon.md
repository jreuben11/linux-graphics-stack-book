# Chapter 134: Apple Silicon AGX GPU — Asahi Linux

> **Part**: Part II — GPU Drivers (extended reference)
> **Audience**: Driver developers interested in reverse-engineering methodology (primary); GPU architecture enthusiasts; embedded/ARM Linux developers; anyone targeting Apple M-series hardware with Linux. This chapter complements [Chapter 73](ch73-asahi-apple-silicon.md), which introduced the Asahi project and covered its broad architecture. Chapter 134 goes deeper into the AGX ISA, the RTKit firmware protocol, the Honeykrisp Vulkan driver internals, the unified memory architecture implications, and the 2025–2026 status including Vulkan 1.4 conformance and M3/M4 reverse engineering challenges.
> **Status**: First draft — 2026-06-19

---

## Table of Contents

- [1. Introduction — The Asahi Linux Project and Its Place in History](#1-introduction--the-asahi-linux-project-and-its-place-in-history)
- [2. Apple AGX Hardware Architecture — TBDR Deep Dive](#2-apple-agx-hardware-architecture--tbdr-deep-dive)
- [3. The AGX Instruction Set Architecture](#3-the-agx-instruction-set-architecture)
- [4. Reverse-Engineering Methodology](#4-reverse-engineering-methodology)
- [5. The Asahi Kernel DRM Driver — Rust Implementation](#5-the-asahi-kernel-drm-driver--rust-implementation)
- [6. AGX Firmware and the RTKit Protocol](#6-agx-firmware-and-the-rtkit-protocol)
- [7. Mesa Asahi Driver — Compiler and Gallium](#7-mesa-asahi-driver--compiler-and-gallium)
- [8. Honeykrisp — The Vulkan Driver](#8-honeykrisp--the-vulkan-driver)
- [9. The Unified Memory Architecture — Implications for the Driver Stack](#9-the-unified-memory-architecture--implications-for-the-driver-stack)
- [10. Performance, Conformance, and Current Status](#10-performance-conformance-and-current-status)
- [Integrations](#integrations)
- [References](#references)

---

## 1. Introduction — The Asahi Linux Project and Its Place in History

**Asahi Linux** ([asahilinux.org](https://asahilinux.org/)) is the project bringing a first-class Linux experience to Apple Silicon Macs — computers built around Apple's M-series system-on-chip designs (M1, M2, M3, M4 and their Pro/Max/Ultra variants). The project was founded by Hector Martin in late 2020 following Apple's announcement of the M1. Its central challenge was immediately apparent: Apple Silicon hardware carries no publicly documented register-level GPU interface, no published ISA for its GPU shader processors, and no kernel DRM driver that could be ported or borrowed.

The Apple AGX GPU (Apple Graphics eXtended, sometimes also referred to internally as the "Apple G-series" by the Asahi team, e.g., G13 for M1-era and G14 for M2-era variants) is Apple's custom tile-based deferred renderer, entirely undocumented at every layer: hardware registers, shader ISA, firmware binary interface, and power management protocol. The Asahi GPU driver is the result of a multi-year reverse-engineering effort led primarily by Alyssa Rosenzweig (compiler and OpenGL driver), Asahi Lina (kernel Rust driver), and Dougall Johnson (ISA reverse engineering), with contributions from dozens of open-source collaborators.

The milestones achieved to date make this one of the most technically significant open-source GPU driver efforts in Linux history:

- **December 2022**: Initial OpenGL 2.1 / OpenGL ES 2.0 drivers shipped in Asahi Linux. [Source: "GPU drivers now in Asahi Linux", December 2022](https://asahilinux.org/2022/12/gpu-drivers-now-in-asahi-linux/)
- **February 2024**: OpenGL 4.6 and OpenGL ES 3.2 conformance certified — the first conformant driver for Apple hardware on any OS for these standards. [Source: "Conformant OpenGL® 4.6 on the M1", February 2024](https://asahilinux.org/2024/02/conformant-gl46-on-the-m1/)
- **June 2024**: Vulkan 1.3 conformance achieved in approximately four weeks of intensive development on the new "Honeykrisp" driver. [Source: "Vulkan 1.3 on the M1 in 1 month", June 2024](https://asahilinux.org/2024/06/vk13-on-the-m1-in-1-month/)
- **Late 2024**: OpenCL 3.0 and Vulkan 1.4 conformance, plus sparse binding enabling VKD3D-Proton DX12 game support. Honeykrisp became the only conformant Vulkan 1.4 driver for Apple hardware on any operating system — surpassing what Apple's own MoltenVK achieves.
- **October 2024**: AAA gaming on Asahi Linux announced: titles including Control, The Witcher 3, Cyberpunk 2077, and Fallout 4 playable via FEX (x86 emulation) + Wine + DXVK/VKD3D-Proton. [Source: "AAA gaming on Asahi Linux", October 2024](https://asahilinux.org/2024/10/aaa-gaming-on-asahi-linux/)
- **2025**: DRM Native Context implementation merged into upstream virglrenderer; Mesa fork retired; UAPI accepted into mainline kernel. [Source: "Progress Report: Linux 6.16", August 2025](https://asahilinux.org/2025/08/progress-report-6-16/)
- **2026**: The `drm/asahi` kernel driver (~21,000 lines of Rust) continues its upstreaming review. M3/M4 GPU support under active reverse engineering. [Source: "Progress Report: Linux 6.19", February 2026](https://asahilinux.org/2026/02/progress-report-6-19/)

This chapter examines the AGX hardware architecture, the ISA, the reverse-engineering methodology, the kernel DRM driver, the RTKit firmware protocol, the Mesa compiler and Gallium driver, the Honeykrisp Vulkan driver, the unified memory architecture, and the current project status. Readers who have not read [Chapter 73](ch73-asahi-apple-silicon.md) are encouraged to start there for a broader overview; this chapter assumes that context and goes deeper on selected technical areas.

---

## 2. Apple AGX Hardware Architecture — TBDR Deep Dive

### GPU Generations and SoC Variants

Apple's AGX GPU family spans multiple generations, each tied to specific SoC chip IDs that the Asahi driver refers to by their internal Apple identifiers:

| GPU Generation | Chip ID | Product | Clusters | Cores/Cluster | Total Cores |
|---|---|---|---|---|---|
| G13G | T8103 | M1 | 1 | 8 | 8 |
| G13S | T6000 | M1 Pro (16-core GPU) | 2 | 8 | 16 |
| G13C | T6001 | M1 Max | 4 | 8 | 32 |
| G13D | T6002 | M1 Ultra (two dies) | 8 | 8 | 64 |
| G14G | T8112 | M2 | 1 | 10 | 10 |
| G14S | T6020 | M2 Pro | 2 | 10 | 20 |
| G14C | T6021 | M2 Max | 4 | 10 | 40 |
| G14D | T6022 | M2 Ultra | 8 | 10 | 80 |
| G15/G16 | T8122/T6030+ | M3/M4 series | varies | varies | varies |

M3 and M4 series GPUs (G15/G16 variants) introduce hardware-accelerated ray tracing, mesh shaders, and Dynamic Caching — significantly different GPU architectures that require fresh reverse engineering. As of mid-2026, M3 machines boot to software-rendered desktops but lack AGX GPU acceleration. [Source: Asahi Linux Progress Report 6.19](https://asahilinux.org/2026/02/progress-report-6-19/)

The `DRM_IOCTL_ASAHI_GET_PARAMS` IOCTL returns `struct drm_asahi_params_global`, which exposes `gpu_generation`, `gpu_variant`, `num_clusters_total`, `num_cores_per_cluster`, `num_frags_per_cluster`, and `core_masks[]` — the complete topology description needed by userspace to allocate optimal tiling parameters and work dispatch sizes. [Source: `include/uapi/drm/asahi_drm.h`](https://github.com/torvalds/linux/blob/master/include/uapi/drm/asahi_drm.h)

### Tile-Based Deferred Rendering Architecture

AGX implements TBDR (Tile-Based Deferred Rendering), an architecture with PowerVR lineage. The rendering pipeline has two mandatory phases per frame:

**Phase 1: Tile Accelerator (TA) / Vertex Pass.** The VDM (Vertex Data Master) fetches vertex index and attribute data, dispatching vertex shaders to the USC (Unified Shader Cores). Post-transform vertex positions are fed to the PPP (Primitive Processing Pipeline) for clipping, culling, and primitive sorting. The PPP writes per-tile primitive lists into the TVB (Tiled Vertex Buffer), also called the Parameter Buffer (PB) — a large allocation in system RAM that accumulates all geometry touching each tile in the framebuffer. Tile size is typically 32×32 pixels on AGX, though the driver negotiates this through `utile_width_px` and `utile_height_px` fields.

**Phase 2: ISP / Fragment Pass.** For each tile, the ISP (Image Synthesis Processor) reads the tile's primitive list from the TVB, rasterizes triangles, and dispatches fragment shaders on USCs. Fragment outputs accumulate in the **tilebuffer** — a small region of on-chip SRAM, sized to hold one tile's worth of color, depth, and stencil samples. At the completion of each tile, a special USC program called the **EoT (End-of-Tile) program** stores the tilebuffer contents to the render target in system RAM. At the start of each tile, a **background (BG) program** loads the prior render target contents (for `VK_ATTACHMENT_LOAD_OP_LOAD`) or writes a clear color (for `VK_ATTACHMENT_LOAD_OP_CLEAR`) into the tilebuffer.

This two-program model is exposed directly in the kernel UAPI via `struct drm_asahi_bg_eot`, which is embedded in `struct drm_asahi_cmd_render`. The kernel does not generate these programs; userspace Mesa compiles them as USC binaries and passes their GPU virtual addresses to the kernel, which forwards them verbatim to firmware. This is architecturally significant: the driver has full control of the tilebuffer load/store pipeline by controlling two tiny but critical shader programs per render pass.

**Partial Renders.** If the TVB fills during geometry submission (a common occurrence on scenes with dense overlapping geometry), the firmware must perform a **partial render**: it flush the completed tiles' tilebuffer contents to a temporary system-RAM surface, empties the TVB, and resumes geometry submission. When the tile is later re-entered for the remainder of the scene, the `partial_bg` program reloads the already-rendered content. The `struct drm_asahi_cmd_render` carries both `bg`/`eot` and `partial_bg`/`partial_eot` fields for this reason. Partial renders are entirely firmware-managed; the userspace driver cannot predict or control when they occur, only provide the programs that handle them correctly.

### Unified Shader Cores and Compute Model

AGX uses unified shader cores (USC): vertex, fragment, geometry-emulation, tessellation-emulation, and compute shaders all execute on the same pool of ALUs. There is no separate fixed-function vertex engine or fragment engine distinct from the shader ALU fabric.

Compute shaders on AGX do not run through the TA/ISP pipeline. A separate hardware unit called the CDM (Compute Dispatch Master) dispatches compute work to USCs, with a parallel fast path that bypasses the tiling infrastructure entirely. This is reflected in the UAPI by the existence of `struct drm_asahi_cmd_compute` alongside `struct drm_asahi_cmd_render`, each submitted independently in the same `DRM_IOCTL_ASAHI_SUBMIT` call.

The CDM fast path matters enormously for Mesa's geometry shader and tessellation emulation strategy. Because AGX lacks hardware geometry shaders and hardware tessellation shaders with the full semantics required by OpenGL/Vulkan, Mesa emulates them as compute dispatches that write into a vertex buffer, which is then consumed by the VDM for the actual TA pass. This two-dispatch strategy — CDM dispatch (geometry/tess emulation) followed by VDM dispatch (tiling) — is orchestrated entirely in userspace.

---

## 3. The AGX Instruction Set Architecture

The AGX ISA was reverse-engineered by Dougall Johnson through iterative analysis of compiled Metal shader binaries. The authoritative reference is documented at [dougallj.github.io/applegpu](https://dougallj.github.io/applegpu/docs.html) and the [applegpu GitHub repository](https://github.com/dougallj/applegpu). This section summarises the key architectural properties that are essential for understanding the Mesa compiler backend.

### Instruction Encoding

AGX instructions have **variable length in 2-byte multiples**, ranging from 2 to 12 bytes. An L-bit in the instruction encoding indicates whether trailing bytes are omitted: if the L-bit is zero, the final 2 or 4 bytes of the instruction encoding are omitted and should be read as zero. This encoding choice compresses instruction streams for common cases while allowing the full encoding for instructions requiring many operands. All encodings are little-endian.

This contrasts sharply with conventional RISC ISAs (ARM64, RISC-V) that use fixed-width instructions, and with Intel GEN (EU) instructions that also use variable width. The variable-length encoding means the AGX disassembler (`agxdecode`, maintained in-tree in Mesa at `src/asahi/disassembler/`) must decode one instruction at a time without being able to seek to a known instruction boundary.

### Register File

Each SIMD-group of 32 threads has access to:

- **128 general-purpose registers (GPRs)** per thread, each storing a 32-bit value. Registers are named `r0` through `r127`. They can be accessed as full 32-bit (`r0`), low 16-bit (`r0l`), or high 16-bit (`r0h`). Most ALU operations support both 32-bit and 16-bit (half-precision) operand widths.
- **256 uniform registers** (`u0`–`u255`), 32 bits each, shared identically across all 32 threads in the SIMD-group. Uniform registers carry values broadcast from the CPU — buffer base addresses, grid dimensions, push constants, and similar per-dispatch constants. The Mesa compiler lowers NIR uniform loads to uniform register reads, keeping them out of the per-thread register file.
- **Special registers**: `r0l` doubles as the execution mask stack depth register. `r1` is the link register for function calls within shaders.

The 128-register file is substantially larger than what older mobile GPU ISAs provide (ARM Mali's Bifrost ISA offers 64 registers per thread, for instance). A larger register file supports more in-flight work without register spilling, but means the register allocator must be more careful about register pressure across control flow boundaries.

### Register Cache and Cache Hints

The GPU maintains a **register cache** that keeps recently-used GPR values quickly accessible. The ISA operand encoding includes two optional hints:

- **Cache hint**: indicates the operand is likely to be reused soon, prioritising it for retention in the register cache.
- **Discard hint**: indicates the source register will not be read again after this instruction, allowing the register cache to invalidate it without writeback.

The Mesa compiler generates these hints during instruction selection and register allocation — the discard hint is particularly valuable for reducing register pressure across large register files.

### Execution Masking and Control Flow

AGX uses an **execution mask stack** rather than separate branch instructions. The 32-bit mask in `r0l` tracks which of the 32 threads in the SIMD-group are currently active. Control flow instructions push and pop the execution mask:

- `if_*cmp` compares a value per-thread and pushes a new mask retaining only threads where the condition holds, while remembering the prior mask.
- `else_*cmp` inverts the mask (active = previously inactive, inactive = previously active) to execute the else branch.
- `while_*cmp` loops back while any thread in the current mask remains active.
- `pop_exec` pops the mask stack, restoring the pre-`if` active set.
- `jmp_exec_none` skips the next instruction if no threads are active (fast exit from dead regions).
- `jmp_exec_any` loops back if any thread is still active (loop continuation).

This execution-mask model means the compiler never generates explicit branch-to-address instructions for shader control flow — all divergence is handled by mask manipulation. The only true jumps are at the exit of loops (`jmp_exec_any`) and dead-code elimination (`jmp_exec_none`). This simplifies instruction scheduling at the cost of executing both sides of a divergent branch (masked threads still consume pipeline bandwidth).

### Arithmetic Instructions

The ISA supports the full set of ALU operations needed for a modern GPU shader ISA:

```
# Integer arithmetic (from applegpu ISA reference)
iadd    r0, r1, r2         # add (with optional shift)
imadd   r0, r1, r2, r3    # multiply-add: r0 = r1*r2 + r3
# Bitfield operations
bfi     r0, r1, r2, #pos, #width  # bit field insert
extr    r0, r1, r2, #lsb          # extract from pair of regs

# Floating-point (supports fp16 and fp32 operands)
fmadd   r0, r1, r2, r3    # fused multiply-add: r0 = r1*r2 + r3
fadd    r0, r1, r2         # add
fmul    r0, r1, r2         # multiply

# Transcendentals
rcp     r0, r1             # reciprocal
rsqrt   r0, r1             # reciprocal square root
log2    r0, r1             # log base 2
exp2    r0, r1             # 2^x
sin_pt_1  r0, r1           # sine (phase 1 of 2-phase evaluation)
sin_pt_2  r0, r1           # sine (phase 2 — combines with pt_1 result)
```

The two-phase sine/cosine evaluation (`sin_pt_1`/`sin_pt_2`) is an unusual feature requiring the compiler to emit a paired instruction sequence rather than a single transcendental. The Mesa compiler has a dedicated NIR lowering pass for this. [Source: dougallj.github.io/applegpu](https://dougallj.github.io/applegpu/docs.html)

### Memory Instructions

```c
// Device (global) memory — with format conversion
device_load   r0, r1, #offset, format=unorm8x4   // load + unpack
device_store  r0, r1, #offset, format=f32x4       // pack + store

// Threadgroup (shared) memory
threadgroup_load   r0, r1, #offset
threadgroup_store  r0, r1, #offset
threadgroup_barrier                                 // sync threads

// Texture operations
texture_sample  r0, tex=0, smp=0, coords=r4       // sample
texture_load    r0, tex=0, lod=0, coords=r4        // load (no sampler)
```

The `device_load` instruction supports rich format conversion in hardware — reading `unorm8`, `snorm8`, `rgb10a2`, `rg11b10f`, and similar packed formats directly without a separate unpack step. This eliminates format conversion instructions for common vertex attribute and texture formats, a notable efficiency over ISAs that require explicit unpacking.

### SIMD Collective Operations

AGX supports sub-group (quad and SIMD-group) collective operations:

```
icmp_ballot   r0, r1, cond     # bitfield: which threads satisfy cond
fcmp_ballot   r0, r1, cond
simd_shuffle  r0, r1, r2       # permute across SIMD group
simd_shuffle_down r0, r1, #delta  # shift-down within group
```

These map directly to Vulkan's `VK_KHR_shader_subgroup_*` extension operations, which Honeykrisp exposes. The `icmp_ballot` instruction in particular enables `subgroupBallot()` in SPIR-V/GLSL.

### Float64 (Double Precision)

AGX G13/G14 has no native fp64 ALU. OpenGL requires `GL_ARB_gpu_shader_fp64`; Honeykrisp/Vulkan requires `VkPhysicalDeviceFeatures::shaderFloat64`. The Mesa driver emulates fp64 using pairs of fp32 registers with software double-precision arithmetic — slower than native but compliant. This is disclosed via `VkPhysicalDeviceVulkan11Features::storageInputOutput16` and the absence of `VkPhysicalDeviceFeatures::shaderFloat64` in the Vulkan device features. [Source: Mesa Asahi documentation](https://docs.mesa3d.org/drivers/asahi.html)

---

## 4. Reverse-Engineering Methodology

Understanding how Asahi's engineers reverse-engineered AGX is as important for the driver developer audience as the driver itself. The methodology produced a production-quality stack from zero documentation in approximately three years — a pace that exceeded all prior expectations for GPU reverse engineering.

### The Fundamental Challenge

Apple's GPU does not expose a hardware-register-level programming interface to the CPU. Unlike conventional discrete GPUs (where the CPU writes command packets directly to a memory-mapped FIFO that feeds fixed-function hardware), the AGX GPU is managed entirely by a firmware running on the ASC (Apple Silicon Coprocessor) embedded in the SoC. The CPU driver writes data structures into shared memory and sends doorbell messages; the ASC firmware reads those structures and programs the actual GPU hardware. This means that CPU-level MMIO tracing (which works well for simpler peripherals) reveals almost nothing about GPU operation — the interesting GPU registers are written by the ASC firmware, not the CPU.

### m1n1: The Bare-Metal Hypervisor

For the non-GPU SoC peripherals (NVMe/ANS, SMC, USB, display DCP), the Asahi team used **m1n1** ([github.com/AsahiLinux/m1n1](https://github.com/AsahiLinux/m1n1)) — a bare-metal hypervisor for the M1 that runs below macOS. m1n1 acts as the system bootloader for Asahi Linux but can also boot macOS as a guest, intercepting and logging MMIO accesses transparently.

The m1n1 Python framework (`proxyclient/`) enables live interaction over USB serial:

```python
# Connect to the M1 via USB serial
from m1n1.setup import *

# Read a 32-bit register from a known physical address
p.read32(0x23b700000)  # example GPU MMIO base

# Trap MMIO region and log accesses
u.hv.map_mmio_trace(0x23b700000, 0x100000, handler)

# Load a hypervisor tracer module before booting macOS
# proxyclient/hv/trace_dcp.py — trace Display Coprocessor
```

[Source: m1n1 Hypervisor documentation](https://asahilinux.org/docs/sw/m1n1-hypervisor/)

The `proxyclient/hv/` directory in m1n1 contains Python tracer modules that pre-configure the hypervisor before macOS boots. For GPU peripherals, the tracer logs RTKit mailbox messages, capturing the doorbell traffic between the CPU driver and the ASC firmware. However, because the GPU command data lives in shared memory (not in mailbox payloads), this only reveals the control-plane signalling, not the actual GPU command buffers.

### IOKit Interception — The Key GPU Technique

For the GPU itself, Alyssa Rosenzweig developed a different approach that operates at the macOS userspace level. The macOS Metal driver is a userspace library (`MetalAGX.framework`) that makes IOKit calls into the kernel's AGX kernel extension (`IOGraphicsFamily`/`AGXAccelerator`). The key IOKit entrypoints are:

1. **Memory allocation** — allocates GPU-visible shared memory via `IOSurface` or `IOMemoryDescriptor`
2. **Command buffer creation** — fills in the GPU command structure in allocated memory
3. **Command buffer submission** — submits to the firmware via an IOKit `connectClient:` call that triggers an ASC doorbell

By intercepting these three calls using `DYLD_INSERT_LIBRARIES` (macOS equivalent of Linux's `LD_PRELOAD`), the team could dump the entire contents of every GPU-visible memory region at the moment of submission — capturing a complete binary snapshot of the command stream for a known, controlled draw call. [Source: Alyssa Rosenzweig, Dissecting the Apple M1 GPU](https://alyssarosenzweig.ca/blog/asahi-gpu-part-1.html)

This technique is preserved in Mesa as the `wrap` library (`src/asahi/lib/wrap.c`), built with `-Dtools=asahi`. The dumped binary data is processed by `agxdecode` (`src/asahi/decode/`), which decodes the command stream into human-readable form as the engineers progressively understood the encoding.

### ISA Reverse Engineering

Dougall Johnson's ISA reverse engineering operated at a lower level: examining the binary output of the Metal shader compiler (`metallib` format) for minimal, carefully crafted shader programs. By writing a shader that performs one specific arithmetic operation and examining which bits in the binary change when the operation or operands change, he deduced the instruction encoding field by field.

```metal
// Minimal Metal shader for ISA probing
kernel void test_add(device float* out, uint id [[thread_position_in_grid]]) {
    out[id] = out[id] + 1.0;  // produces: fmadd instruction
}
```

Compiling this with `xcrun -sdk macosx metal -c test.metal -o test.air && xcrun -sdk macosx metallib test.air -o test.metallib` and extracting the binary with `metallib-extract` (a tool the team wrote) yields the raw USC binary. Systematic variation of the shader source — changing the constant, the operation type, the register source — revealed the ISA encoding incrementally.

The complete documented ISA is at [dougallj.github.io/applegpu](https://dougallj.github.io/applegpu/docs.html). The `applegpu` Python package provides a disassembler/assembler and an instruction interpreter (used for differential testing of the Mesa compiler). Mesa's AGX compiler backend references this documentation directly in comments citing specific ISA sections.

### The Iterative Fuzzing Cycle

For firmware data structures (the "firmware ABI" — the packed C structs that the kernel driver writes into shared memory for the ASC to read), the methodology was iterative binary fuzzing:

1. Intercept a known draw call and capture the binary command buffer.
2. Modify one bit (or one byte) at a known offset.
3. Resubmit the modified buffer.
4. Observe the outcome: GPU crash, wrong output, or expected output.
5. Deduce the field's purpose from its effect.

Over hundreds of such iterations, with `agxdecode` automating the structure layout annotation, the team documented over 70 distinct firmware struct types. Each struct's layout was encoded in Python (for the `proxyclient` tools) and later in Rust (for the kernel driver's `fw/` directory). Changes in macOS firmware versions sometimes changed struct layouts silently, which is why the Rust driver versioned its firmware struct definitions across the supported firmware versions.

---

## 5. The Asahi Kernel DRM Driver — Rust Implementation

The AGX kernel DRM driver lives in `drivers/gpu/drm/asahi/` in the Asahi Linux kernel fork ([github.com/AsahiLinux/linux, branch `gpu/rust-wip`](https://github.com/AsahiLinux/linux/tree/gpu/rust-wip/drivers/gpu/drm/asahi)). As of 2025, the UAPI header has been accepted into mainline Linux at `include/uapi/drm/asahi_drm.h`; the driver itself (~21,000 lines of Rust) is undergoing the upstream review process. It was written in Rust — making it the first DRM driver in the Linux kernel written in that language.

### Key Source Files

The driver is structured as follows:

| File | Purpose |
|---|---|
| `driver.rs` | Platform driver entry, DRM device registration, IOCTL dispatch |
| `gpu.rs` | `GpuManager` trait: per-firmware-version GPU management |
| `file.rs` | Per-open-file IOCTL handler (`submit`, `vm_create`, `gem_create`, etc.) |
| `gem.rs` | GEM buffer object (`gem::Object`), folio-backed system memory |
| `mmu.rs` | UAT (GPU MMU): VM contexts, page table allocation, TTBAT management |
| `pgtable.rs` | ARM64-compatible GPU page table manipulation (16KiB pages) |
| `channel.rs` | RTKit channel ring buffers: CPU-to-GPU and GPU-to-CPU messages |
| `workqueue.rs` | Firmware work queue management |
| `microseq.rs` | Firmware microsequence instruction builder |
| `initdata.rs` | GPU initialization data structures sent to firmware at boot |
| `event.rs` | GPU completion event and DMA fence infrastructure |
| `crashdump.rs` | AGX firmware crash log capture and decoding |
| `buffer.rs` | TVB (Tiled Vertex Buffer) / Parameter Buffer lifecycle management |
| `alloc.rs` | GPU-side slab allocator for firmware objects |
| `mem.rs` | Typed wrapper for firmware-ABI memory regions |
| `regs.rs` | MMIO register definitions (GPU and ASC) |
| `slotalloc.rs` | Fixed-capacity slot allocator for firmware context handles |
| `queue/` | Command queue state machine, submission logic |
| `fw/` | Typed Rust structs matching firmware ABI (`fw::job`, `fw::event`, etc.) |
| `hw/` | Per-chip constants: `t8103.rs`, `t8112.rs`, `t600x.rs`, `t602x.rs` |

[Source: AsahiLinux/linux driver directory](https://github.com/AsahiLinux/linux/tree/gpu/rust-wip/drivers/gpu/drm/asahi)

### Why Rust

The decision to write the kernel driver in Rust rather than C was driven by the nature of the firmware ABI. Over 70 distinct packed struct types, many with version-dependent layouts across different firmware releases, would be error-prone in C where a wrong field offset is a runtime memory corruption. Rust's type system allows each firmware struct to be expressed with compile-time size assertions and field-type guarantees:

```rust
// drivers/gpu/drm/asahi/fw/job.rs (simplified)
#[repr(C)]
pub(crate) struct GpuJob {
    pub(crate) unk_0: u32,
    pub(crate) cmd_type: u32,
    pub(crate) vm_slot: u32,
    pub(crate) num_cmds: u32,
    pub(crate) cmds_ptr: U64,
    // ... 70+ more fields
}
// Compile-time size check to catch layout regressions
const _: () = assert!(core::mem::size_of::<GpuJob>() == 0x3c0);
```

If a firmware update changes the struct size, the compile-time assertion fails immediately — a regression that would be a silent runtime crash in C. Asahi Lina cited this as the primary motivation in the RFC patch series. [Source: LKML, RFC PATCH 00/18, March 2023](https://lkml.iu.edu/hypermail/linux/kernel/2303.0/07151.html)

Beyond struct safety, writing the driver in Rust forced the creation of high-quality Rust abstractions for the DRM subsystem — `drm::device::Device<T>`, GEM object wrappers, DMA fence wrappers, and the GPU scheduler integration — that serve as templates for future Rust DRM drivers.

### The DRM IOCTL Interface

The Asahi UAPI defines 11 IOCTLs:

```c
// include/uapi/drm/asahi_drm.h
#define DRM_ASAHI_GET_PARAMS      0x00
#define DRM_ASAHI_VM_CREATE       0x01
#define DRM_ASAHI_VM_DESTROY      0x02
#define DRM_ASAHI_GEM_CREATE      0x03
#define DRM_ASAHI_GEM_MMAP_OFFSET 0x04
#define DRM_ASAHI_GEM_BIND_OBJECT 0x05
#define DRM_ASAHI_VM_BIND         0x06
#define DRM_ASAHI_QUEUE_CREATE    0x07
#define DRM_ASAHI_QUEUE_DESTROY   0x08
#define DRM_ASAHI_SUBMIT          0x09
#define DRM_ASAHI_GET_TIME        0x0a
```

The UAPI design follows the pattern established by Intel Xe and Panthor: **explicit VM management** (userspace controls the GPU virtual address space) and **explicit synchronization** (no implicit buffer tracking; all sync via DRM sync objects). This is architecturally cleaner than the implicit sync model used by older drivers like `amdgpu` in GFX9 and earlier, and aligns Asahi with the modern driver model that Vulkan mandates.

A typical render submission sequence:

```c
// 1. Query device capabilities
struct drm_asahi_params_global params = {};
ioctl(fd, DRM_IOCTL_ASAHI_GET_PARAMS, &params);
// params.gpu_generation, .num_clusters_total, etc.

// 2. Create a GPU virtual address space
struct drm_asahi_vm_create vm = {};
ioctl(fd, DRM_IOCTL_ASAHI_VM_CREATE, &vm);
uint32_t vm_id = vm.vm_id;

// 3. Allocate a GEM buffer
struct drm_asahi_gem_create gem = {
    .size    = 4096,
    .flags   = DRM_ASAHI_GEM_WRITEBACK,
};
ioctl(fd, DRM_IOCTL_ASAHI_GEM_CREATE, &gem);

// 4. Bind it into the GPU VA space
struct drm_asahi_gem_bind_op bind_op = {
    .op      = DRM_ASAHI_BIND_OP_BIND,
    .handle  = gem.handle,
    .flags   = DRM_ASAHI_BIND_READ | DRM_ASAHI_BIND_WRITE,
    .addr    = 0x200000000,  // chosen GPU VA
    .range   = 4096,
};
struct drm_asahi_vm_bind bind = {
    .vm_id       = vm_id,
    .num_binds   = 1,
    .binds.array = &bind_op,
};
ioctl(fd, DRM_IOCTL_ASAHI_VM_BIND, &bind);

// 5. Create a submission queue
struct drm_asahi_queue_create queue = {
    .vm_id    = vm_id,
    .priority = 1,
};
ioctl(fd, DRM_IOCTL_ASAHI_QUEUE_CREATE, &queue);

// 6. Submit a render command
struct drm_asahi_cmd_render render_cmd = { /* ... */ };
struct drm_asahi_submit submit = {
    .queue_id  = queue.queue_id,
    .in_syncs  = &in_sync,
    .out_syncs = &out_sync,
    .cmds      = &render_cmd_header,
    .num_cmds  = 1,
};
ioctl(fd, DRM_IOCTL_ASAHI_SUBMIT, &submit);
```

[Source: `include/uapi/drm/asahi_drm.h`](https://github.com/torvalds/linux/blob/master/include/uapi/drm/asahi_drm.h)

### The UAT — GPU MMU

AGX uses an MMU called the UAT (Unified Address Translator) that is structurally identical to the ARM64 CPU MMU:

- 40-bit GPU output address space (42-bit for M1 Ultra with its two-die package)
- 16 KiB pages (not the 4 KiB typical on x86/discrete GPU)
- ARM64-compatible multi-level page table format (L1/L2/L3)
- Up to 64 active GPU VM contexts

The driver maintains a TTBAT (Translation Table Base Address Table) — a firmware-visible array of page table base pointers, one per VM context. When a new VM is created via `DRM_IOCTL_ASAHI_VM_CREATE`, the kernel allocates a slot in the TTBAT, sets up an ARM64-format page directory, and updates the TTBAT entry. The firmware uses the TTBAT slot index as the VM identifier in work submissions.

The 16 KiB page size is an Apple system-wide requirement (all Apple Silicon platforms, CPU and GPU alike, use 16 KiB pages). This has an important implication: any imported DMA-BUF from a non-Apple driver that uses 4 KiB pages must be remapped at the GPU side in 16 KiB chunks, potentially wasting up to 12 KiB per allocation. Mesa handles this transparently.

---

## 6. AGX Firmware and the RTKit Protocol

### The ASC Coprocessor and RTKit

The AGX GPU's command scheduling, preemption, power management, fault recovery, and performance monitoring do not run on the main ARM64 application processor. Instead, they run on the ASC (Apple Silicon Coprocessor) — a separate ARM64 CPU core embedded in the SoC alongside the GPU. Apple's RTKit RTOS (Real-Time OS) runs on the ASC. The main CPU driver communicates with the ASC exclusively via shared memory and a small set of doorbell (mailbox) registers.

The Linux `apple-rtkit` driver (`drivers/soc/apple/rtkit.c`) provides the abstraction layer over the Apple ASC mailbox hardware. RTKit messages are 64-bit values delivered over endpoint (EP) channels. Each endpoint represents a service; the GPU driver uses dedicated endpoints for initialization, command submission, and power management. [Source: `drivers/soc/apple/rtkit.c`](https://github.com/torvalds/linux/blob/master/drivers/soc/apple/rtkit.c)

### RTKit Message Format

An RTKit message consists of:
- An **endpoint ID** (`u8`) identifying the service
- A **64-bit payload** whose meaning is endpoint-specific

The GPU uses several endpoint channels:

```
EP_GPU_CONTROL  = 0x20   # GPU initialization and power control
EP_GPU_WORK     = 0x21   # Work submission acknowledgement
EP_GPU_FAULT    = 0x22   # GPU fault notifications
EP_CRASH        = 0xfe   # Firmware crash log
```

All actual GPU work data is passed via shared memory — the EP_GPU_WORK endpoint only carries small notification messages that say "work at VA 0xXXXXXXXX is now complete" rather than carrying the work data itself.

### Firmware Initialization Sequence

The GPU initialization sequence in `gpu.rs` / `initdata.rs`:

1. **Load firmware**: The AGX firmware binary (`gpu.macho`, a MachO format binary) is loaded from the macOS firmware partition via `request_firmware()`. The driver uses `apple-rtkit` to transfer it to the ASC's address space.

2. **Construct initdata**: `initdata.rs` builds a large hierarchical data structure containing:
   - Pointers to channel ring buffers (CPU-to-GPU and GPU-to-CPU channels)
   - Power/DVFS state tables
   - MMIO mapping lists the firmware needs direct access to
   - UAT (MMU) configuration including TTBAT pointer
   - Heap allocator state for firmware-side buffer management

3. **Send BOOT message**: A single RTKit message on `EP_GPU_CONTROL` carries the physical address of the initdata structure. The firmware reads the structure, configures itself, and sends a `BOOT_DONE` message in reply.

4. **Wait for BOOT_DONE**: The kernel blocks (with timeout) waiting for the firmware `BOOT_DONE` acknowledgement. Timeout indicates firmware crash or incompatibility.

```rust
// gpu.rs (simplified)
fn start_firmware(&self, initdata_pa: u64) -> Result {
    self.rtkit.send_message(
        EP_GPU_CONTROL,
        GPU_MSG_BOOT | (initdata_pa << 8),
    )?;
    // Block waiting for BOOT_DONE
    self.boot_wait.wait_timeout(BOOT_TIMEOUT_MS)?;
    Ok(())
}
```

### Firmware Work Submission

Once initialized, GPU work is submitted by:

1. **Building a job structure** in shared memory (the `fw::job::GpuJob` Rust struct)
2. **Writing the job VA** into the work channel ring buffer in `channel.rs`
3. **Ringing the doorbell** register (`DOORBELL_REG` in `regs.rs`) to notify the ASC that work is available
4. **The ASC reads the ring buffer**, extracts the job VA, reads the job structure from shared memory, and programs the AGX GPU hardware directly
5. **Completion interrupt**: when the GPU finishes, the ASC writes a completion stamp to a shared memory location and sends an RTKit message on `EP_GPU_WORK`
6. **DMA fence signalling**: `event.rs` handles the completion notification, calling `drm_fence_signal()` to unblock waiting Vulkan/OpenGL commands

The **microsequence** is a separate small instruction stream within the job structure that the ASC executes as a simple virtual machine before and after submitting work to the GPU:

```
// microseq.rs instruction types (simplified enum)
pub enum Instr {
    Start3D(StartParams),       // Begin 3D render pass
    StartTA(StartParams),       // Begin tile accelerator pass
    StartCompute(StartParams),  // Begin compute dispatch
    Timestamp { dst: VA },      // Record GPU timestamp
    WaitForIdle,                // Synchronization barrier
    Finish,                     // Signal completion
}
```

A typical render microsequence is: `StartTA → Timestamp → WaitForIdle → Timestamp → Start3D → Timestamp → WaitForIdle → Timestamp → Finish`. The timestamps at each stage are written to GPU-visible memory locations specified in the command, enabling `drm_asahi_timestamps` to report per-stage timing to userspace.

### Firmware ABI Versioning and Bugs

The AGX firmware is a closed-source binary that changes with every macOS update. The driver must be compatible with the specific firmware version shipped on the user's macOS partition. The Asahi kernel driver handles this by having per-generation firmware versions in `hw/t8103.rs`, `hw/t8112.rs`, etc., and using conditional struct layouts in `fw/` for fields that changed across versions.

Known firmware bugs require workarounds. One category: when the TVB overflows and a partial render occurs, a timing window in the firmware can produce a race condition in the overflow handler that corrupts the parameter buffer. The Mesa driver works around this by pre-allocating a larger TVB than strictly necessary for the known scene complexity. Another category: the firmware's DVFS (Dynamic Voltage and Frequency Scaling) governor can occasionally request an unsupported P-state during workloads with rapid transitions; the driver intercepts and clamps these requests.

GPU faults (out-of-bounds memory access, shader timeout, etc.) are reported via `EP_GPU_FAULT` with a fault code. The driver logs the fault, captures a crash dump via `crashdump.rs` (which reads a crash log structure the firmware fills in a dedicated shared-memory region), and resets the GPU by restarting the firmware.

---

## 7. Mesa Asahi Driver — Compiler and Gallium

### Repository Structure

The Mesa Asahi driver lives in `src/asahi/` and `src/gallium/drivers/asahi/` within the Mesa repository ([gitlab.freedesktop.org/mesa/mesa](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/asahi)). Key subdirectories:

```
src/asahi/
├── compiler/          # AGX NIR compiler backend → AGX ISA
│   ├── agx_compile.c  # Top-level: NIR → AGX binary
│   ├── agx_isel.c     # Instruction selection
│   ├── agx_ra.c       # Register allocation
│   ├── agx_pack.h     # ISA instruction packing (generated)
│   └── agx_nir_*.c    # NIR lowering passes
├── decode/            # agxdecode: command stream disassembler
├── disassembler/      # AGX ISA disassembler
├── lib/               # Shared AGX utilities, wrap library (macOS)
│   └── wrap.c         # macOS IOKit intercept library
└── vulkan/            # Honeykrisp Vulkan driver (hk_*)

src/gallium/drivers/asahi/
├── agx_state.c        # agx_context, state tracker
├── agx_draw.c         # agx_draw_vbo — draw call entry point
├── agx_batch.c        # Batch management
└── agx_pipe.c         # Gallium pipe interface
```

### The NIR Compiler Backend

The AGX compiler backend takes NIR (Mesa's intermediate representation, see [Chapter 14](../../chapters/part-03-mesa/ch14-nir.md)) as input and produces AGX ISA binary. The pipeline:

```
GLSL/SPIR-V
    ↓  (nir_validate, nir_lower_*)
NIR (SSA form)
    ↓  (agx_preprocess_nir)
AGX-lowered NIR
    ↓  (agx_isel)
AGX SSA IR
    ↓  (agx_ra)
AGX allocated IR (registers assigned)
    ↓  (agx_pack)
AGX ISA binary
```

Key NIR lowering passes specific to AGX:

- **`agx_nir_lower_varyings.c`**: lowers vertex shader outputs (`gl_Position`, varyings) to AGX `st_var` instructions; lowers fragment shader inputs to `ldcf` (load coefficient) and `iter` (interpolate) instructions. The hardware performs interpolation in a separate "Varying Processing" unit that reads coefficient data written by `st_var`.

- **`agx_nir_lower_uniforms.c`**: lowers NIR uniforms and push constants to AGX uniform register reads (`u0`–`u255`).

- **`agx_nir_lower_images.c`**: lowers image loads/stores to `device_load`/`device_store` with appropriate format specifiers.

- **`agx_nir_lower_robustness.c`**: implements buffer robustness (GL 4.6 `GL_KHR_robustness`) without hardware bounds-checking hardware — uses unsigned-minimum clamping: `index = umin(index, buffer_size - 1)` with a single `imadd` instruction.

- **`agx_nir_lower_control_flow.c`**: lowers structured control flow (if/else, loops) to execution mask push/pop sequences.

### Register Allocation

The AGX register allocator (`agx_ra.c`) operates on AGX SSA IR after instruction selection. The register file has 128 GPRs per thread, but the allocator must account for:

1. **Register file fragmentation** from the 16-bit/32-bit dual-width register model (a 32-bit register `r0` and its halves `r0l`/`r0h` are the same storage).
2. **Predefined registers**: `r0l` (execution mask), `r1` (link register) cannot be allocated freely.
3. **SIMD-group register pressure**: a higher thread count per SIMD-group uses fewer registers per thread (the hardware partitions the 128 registers across active SIMD-groups).

The allocator uses a standard SSA-based graph coloring approach but with the AGX-specific constraint that the starting register of a multi-component vector must be aligned to the vector width (a `vec4` must start at an even-numbered register for the 32-bit components).

### `agx_pack.h` — Generated ISA Packing

Similar to Intel's `genxml` and the Panfrost `pan_pack.h`, the AGX compiler uses a generated header `agx_pack.h` that provides C macros for constructing each instruction's binary encoding from named fields:

```c
// src/asahi/compiler/agx_pack.h (generated)
agx_pack(out, FMADD, cfg) {
    cfg.saturate   = false;
    cfg.dest       = 0;   // r0
    cfg.src0       = 1;   // r1
    cfg.src1       = 2;   // r2
    cfg.src2       = 3;   // r3
    cfg.size       = AGX_SIZE_32;
}
```

The generator reads the ISA description (derived from Dougall Johnson's documentation) and produces the macro pack/unpack infrastructure, ensuring that the instruction encoding in the compiler stays in sync with the disassembler (`agxdecode`).

---

## 8. Honeykrisp — The Vulkan Driver

### Architecture and Name

"Honeykrisp" is the name for the Asahi Linux Vulkan driver for Apple Silicon, implemented in `src/asahi/vulkan/` in Mesa. The name comes from a variety of apple — Asahi Lina's cat is also named Honeykrisp — continuing a tradition of cat-named Mesa drivers (NVK was named after Lina's NVK driver, and the naming convention persisted).

Honeykrisp was not written from scratch. Faith Ekstrand's NVK driver (the open-source Vulkan driver for NVIDIA GPUs) provided the structural foundation. NVK's architecture is modern, clean, and designed to implement the full Vulkan specification without portability waivers — an ideal starting point. The key structs — `hk_device`, `hk_queue`, `hk_cmd_buffer`, `hk_shader`, `hk_descriptor_set` — are renamed and adapted NVK counterparts.

### Four-Week Timeline to Vulkan 1.3 Conformance

The extraordinary speed of Honeykrisp development (Vulkan 1.3 conformance in approximately four weeks, June 2024) was possible because:

1. **NVK foundation** provided all the Vulkan bookkeeping and command buffer recording infrastructure.
2. **The AGX compiler backend** (NIR → AGX ISA) already existed and was tested via OpenGL 4.6.
3. **The kernel UAPI** (explicit VM, explicit sync) was designed with Vulkan in mind.

The Honeykrisp developer (Alyssa Rosenzweig) posted a detailed day-by-day log: April 3 (descriptor lowering + compute), April 6 (graphics pipeline + dynamic state), April 18 (DRM format modifiers for WSI), April 24 (YCbCr), April 25 (GPU query copies via compute "witchcraft"), April 26-27 (border color emulation + conformance). The final `vkcts` run showed 686,930 passes and 0 failures. [Source: "Vulkan 1.3 on the M1 in 1 month"](https://asahilinux.org/2024/06/vk13-on-the-m1-in-1-month/)

### Dynamic State Strategy

One of Honeykrisp's most significant architectural decisions is its approach to pipeline state. Most Vulkan drivers (RadV, ANV, TURNIP) compile pipeline state into shader preambles at `vkCreateGraphicsPipeline` time. This produces fast draws but causes **pipeline compile stutter**: the first time a pipeline is used in a game scene, the driver must spend milliseconds compiling a new binary, causing frame hitches.

Honeykrisp treats almost all Vulkan pipeline state as **dynamic**. Instead of baking state into the shader binary, it uses four strategies per state type:

| State type | Strategy |
|---|---|
| Blend state, depth test | Conditional code inserted into shader |
| Vertex input layout | Precompiled variants (one per vertex format combo) |
| Descriptor layout | Indirection table (GPU reads descriptor via a pointer) |
| Shader prologues/epilogs | AMD-style split: vertex fetch prolog + main body + FS output epilog |

This approach increases CPU overhead per draw call (more setup per submit) but eliminates shader stutter entirely — a trade-off that benefits gaming workloads where frame pacing matters more than raw throughput at typical draw rates.

### Background/End-of-Tile Programs — TBDR Vulkan Mapping

The most architecturally unique aspect of Honeykrisp is its mapping of Vulkan render pass `loadOp`/`storeOp` to AGX BG/EoT programs.

Vulkan specifies per-attachment load and store operations at render pass begin and end:
- `VK_ATTACHMENT_LOAD_OP_CLEAR` → tilebuffer must be cleared to a clear value
- `VK_ATTACHMENT_LOAD_OP_LOAD` → tilebuffer must be loaded from prior render target contents
- `VK_ATTACHMENT_STORE_OP_STORE` → tilebuffer must be written back to render target
- `VK_ATTACHMENT_STORE_OP_DONT_CARE` → tilebuffer need not be stored (bandwidth saving)

On AGX, these operations are implemented as tiny USC programs:

- **BG program**: synthesised by Honeykrisp to implement `LOAD_OP_*` — either a clear (writes solid color to the tilebuffer) or a load (reads from the render target into the tilebuffer).
- **EoT program**: synthesised by Honeykrisp to implement `STORE_OP_*` — either a store (writes tilebuffer to the render target) or a discard (does nothing).

These programs are USC binaries compiled by `libagx` at runtime when the render pass is created. They are passed to the kernel in `struct drm_asahi_bg_eot` fields of `struct drm_asahi_cmd_render`. The kernel does not inspect their content; it forwards them to firmware, which dispatches them as part of the hardware tile loop.

This design means that operations like MSAA resolve, render target format conversion, and depth/stencil layout transitions can all be implemented as EoT program variants — the tilebuffer-resident representation is the "free" transformation point, happening after fragment shading but before the write back to system RAM.

### Descriptor Model

Honeykrisp implements a **bindless descriptor model** compatible with the AGX hardware. AGX shaders access buffers and textures via handles stored in a GPU-visible descriptor heap. Honeykrisp uses `VK_EXT_descriptor_indexing` to expose this model to Vulkan applications. All descriptors are stored in a large buffer mapped into the GPU address space; shaders load handles from this buffer using `device_load` instructions at descriptor set offsets.

The descriptor heap is allocated once per VkDevice and managed by Honeykrisp using a free-list allocator. When a descriptor set is bound, Honeykrisp does not copy descriptors into the command buffer; instead, it writes a pointer to the descriptor set's heap region into a uniform register. Shader code generated by Honeykrisp accesses descriptors via indexed loads through this pointer.

### Geometry Shaders and Tessellation — Compute Emulation

AGX hardware geometry shaders exist but are constrained: the M1's tessellator cannot implement the full OpenGL/Vulkan TCS/TES semantics (e.g., it cannot handle all primitive topologies with variable output vertex counts). Honeykrisp emulates geometry shaders and tessellation as compute dispatches:

1. A **CDM compute dispatch** runs the geometry shader / tessellation evaluation shader, writing amplified geometry to a scratch buffer.
2. The output scratch buffer is then consumed by a **VDM draw** as vertex data for the TA pass.

This two-dispatch approach adds CPU overhead (two IOCTL submits per draw with geometry/tessellation) but achieves full correctness. The compute dispatch is synthesised by the NIR compiler as a transformed version of the geometry/tessellation shader with write-to-buffer outputs replacing the normal vertex outputs.

### Hardware Border Color Issue

AGX hardware does not support all Vulkan border color semantics. Specifically, `VK_BORDER_COLOR_INT_OPAQUE_BLACK` in 16-bit integer formats returns incorrect values from the native sampler. Honeykrisp implements a software workaround: when a sampler requires an incompatible border color, the compiler injects a fixup code sequence after the `texture_sample` instruction that conditionally replaces the result with the correct border color value when the texture coordinate is out of range. [Source: Vulkan 1.3 blog post](https://asahilinux.org/2024/06/vk13-on-the-m1-in-1-month/)

### Sparse Binding — VK_EXT_buffer_device_address and Sparse Textures

Sparse binding (`VK_EXT_sparse_binding`, `VK_SPARSE_MEMORY_BIND`) was a critical feature for VKD3D-Proton (the DirectX 12 to Vulkan translation layer). It allows a large virtual buffer or texture to be backed by non-contiguous physical memory, enabling streaming allocation of large assets.

The AGX hardware supports sparse binding via a hardware feature called **soft faults**: when the GPU accesses an unmapped page, instead of hard-faulting and crashing, it returns zero (for reads) or discards (for writes). The Asahi UAPI exposes this via `DRM_ASAHI_FEATURE_SOFT_FAULTS` and the `DRM_ASAHI_BIND_SINGLE_PAGE` bind flag (which maps a single physical page across a large VA range — the sparse "null mapping" case). Mesa 25.1 added sparse binding support, unlocking VKD3D-Proton DX12_0 capability. [Source: AAA gaming blog post](https://asahilinux.org/2024/10/aaa-gaming-on-asahi-linux/)

### Running Honeykrisp

On an Asahi Linux system with the appropriate Mesa build:

```bash
# Check Vulkan device
MESA_LOADER_DRIVER_OVERRIDE=honeykrisp vulkaninfo 2>/dev/null | grep -E "apiVersion|deviceName"

# Run a Vulkan application
MESA_VK_DEVICE_SELECT=1 vkcube

# OpenGL via Zink on top of Honeykrisp
MESA_LOADER_DRIVER_OVERRIDE=zink GALLIUM_DRIVER=zink glxinfo | grep "OpenGL version"

# Gaming via FEX (x86 emulation) + Wine + DXVK
FEX_ROOTFS=/path/to/rootfs FEXBash wine game.exe
```

---

## 9. The Unified Memory Architecture — Implications for the Driver Stack

### No Discrete VRAM

Apple Silicon SoCs use a unified memory architecture (UMA) where the CPU and GPU share a single pool of physical LPDDR5/LPDDR5X DRAM. There is no discrete VRAM, no PCI-Express memory controller, and no distinction between CPU-accessible and GPU-accessible memory. Every allocation made via `DRM_IOCTL_ASAHI_GEM_CREATE` is in the same physical memory pool accessible to both the CPU and GPU.

From the Vulkan memory model perspective, this means:

```c
// vulkaninfo on Asahi Linux — memory heaps
Heap 0:
  Size:   16 GB (or 8/24/32/192 GB depending on model)
  Flags:  DEVICE_LOCAL | HOST_VISIBLE | HOST_COHERENT | HOST_CACHED

// Memory types — there is only one heap
Type 0: DEVICE_LOCAL | HOST_VISIBLE | HOST_COHERENT | HOST_CACHED
```

Unlike discrete GPU systems (where `HOST_VISIBLE` and `DEVICE_LOCAL` are on separate heaps), on AGX they are the same heap. `VkBuffer` allocations with `MEMORY_PROPERTY_DEVICE_LOCAL_BIT` and `HOST_VISIBLE_BIT` are on the same physical memory — there is no staging buffer copy required for CPU-written GPU data.

### Zero-Copy CPU ↔ GPU

The zero-copy implication is profound for performance:

- **Vertex buffers**: An application can write vertex data directly to a `HOST_VISIBLE | DEVICE_LOCAL` buffer and submit a draw immediately — no staging copy needed.
- **Uniform buffers**: Similarly, uniform buffer updates are a CPU write to shared memory; no GPU copy.
- **Read-back**: GPU compute results can be read by the CPU without a copy or cache flush.

The memory is cache-coherent between CPU and GPU. There is no need for `VkMappedMemoryRange::flush` / `invalidate` calls on AGX — the memory type is `HOST_COHERENT`, meaning the CPU cache is always coherent with the GPU. The Mesa driver does not emit any cache management instructions for CPU↔GPU data sharing.

### DMA-BUF Sharing — Cross-Engine Zero-Copy

Despite the single physical memory pool, multiple hardware engines on the SoC (GPU, ISP image signal processor, display DCP, video encoder/decoder) have separate MMUs and separate driver instances. Cross-engine buffer sharing still uses DMA-BUF ([Chapter 75](../../chapters/part-07-sync/ch75-dmabuf.md)):

```c
// Camera ISP → GPU zero-copy pipeline
int camera_fd = open("/dev/video0", O_RDWR);
// VIDIOC_QUERYBUF + VIDIOC_EXPBUF to get DMA-BUF fd
int dmabuf_fd = export_camera_buffer(camera_fd);

// Import into Vulkan as external memory
VkImportMemoryFdInfoKHR import_info = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = dmabuf_fd,
};
```

Because both the ISP and the GPU draw from the same physical pool, the DMA-BUF import is truly zero-copy — it maps the same physical pages into the GPU VA space without any copy. On a discrete GPU system, a DMA-BUF import from a camera driver would typically still require a CPU-side copy unless both drivers support direct system-memory access. On AGX, this copy is structurally eliminated.

### Display Pipeline and DCP

The internal display is driven by the DCP (Display Coprocessor), which runs its own firmware on an ASC. The `drivers/gpu/drm/apple/` kernel driver provides DRM/KMS on top of the DCP's remote procedure call interface. The display driver does not share code with the AGX GPU driver at the kernel level — they are separate DRM devices — but they share DMA-BUF objects for scanout buffers.

For external displays connected via USB-C (which require coordination between the DCP, the DPXBAR, the ATCPHY USB-C PHY, and the ACE power delivery controller), Asahi Linux achieved USB-C display support in late 2025 via the "fairydust" branch, enabling full external display support for the first time. [Source: Asahi Linux five-year milestone announcement](https://news.lavx.hu/article/asahi-linux-hits-five-year-milestone-with-breakthrough-usb-c-display-support)

Wayland compositors work well on Asahi Linux:

```bash
# Sway (tiling WM)
sway

# GNOME Shell / Mutter
XDG_SESSION_TYPE=wayland gnome-shell

# KDE Plasma / KWin
startplasma-wayland

# Testing headless (CI)
weston --backend=headless-backend.so
```

### Memory Bandwidth and Shared Pressure

The UMA advantage (zero-copy) comes with a corresponding constraint: CPU and GPU compete for the same memory bandwidth. The M2 Ultra provides approximately 800 GB/s of memory bandwidth (200 GB/s per LPDDR5X channel × 4 channels); this is shared among all SoC engines.

GPU workloads that are bandwidth-limited (large framebuffer reads, high-resolution rendering) can be starved by concurrent CPU workloads and vice versa. The TBDR architecture partially mitigates this: by keeping intermediate fragment data in the on-chip tilebuffer rather than system RAM, the ISP pass accesses system RAM only at the BG/EoT tile boundaries — dramatically reducing bandwidth for opaque geometry rendering compared to an immediate-mode GPU.

Profiling memory bandwidth on Asahi Linux:

```bash
# Monitor GPU memory bandwidth via sysfs (from gpu.rs DVFS counters)
cat /sys/kernel/debug/dri/0/agx_stats

# perf-based profiling (with Asahi perf kernel support)
perf stat -e cycles,cache-misses ./your_gpu_application
```

---

## 10. Performance, Conformance, and Current Status

### Conformance Achieved

As of mid-2026, the Asahi Mesa stack is conformant to:

| Standard | Version | Date |
|---|---|---|
| OpenGL | 4.6 | February 2024 |
| OpenGL ES | 3.2 | February 2024 |
| OpenCL | 3.0 | 2024 |
| Vulkan | 1.4 | Late 2024 |

Honeykrisp is the only conformant Vulkan 1.4 driver for Apple hardware on any operating system. Apple's own MoltenVK achieves Vulkan 1.3 with extensions but not full 1.4 conformance. [Source: Khronos conformance database]

The OpenGL 4.6 / OpenGL ES 3.2 achievement was particularly notable because these standards require features that AGX hardware does not natively support, all of which were emulated:

- **Geometry shaders**: emulated as CDM compute dispatches
- **Tessellation shaders**: emulated as CDM compute dispatches (hardware tessellator too limited)
- **Transform feedback**: emulated as a compute-augmented geometry pass
- **Buffer robustness**: software bounds-clamping via `umin` instruction
- **SPIR-V ingestion**: full SPIR-V → NIR → AGX pipeline

### Performance Metrics

Selected performance figures from Asahi Linux:

- **Buffer clear throughput (M1 Ultra)**: 355 GB/s for 16-byte aligned buffers using AGX-optimized `libagx_fill` implementation. [Source: Progress Report 6.19](https://asahilinux.org/2026/02/progress-report-6-19/)
- **Buffer copy throughput (Vulkan, Mesa 25.2+)**: 30%–2× speedup over previous implementation after replacing CPU memcpy with GPU copy shaders for aligned buffer operations.
- **Vulkan draw call throughput (`vkoverhead`)**: ~100 million draws per second on M1, despite the dynamic-state-first architecture that adds per-draw CPU overhead.
- **4K rendering**: Xonotic and Quake run at 4K 60fps on M1 systems.
- **AAA gaming**: Control, The Witcher 3, Cyberpunk 2077 and Fallout 4 are playable (alpha quality) via FEX + Wine + DXVK/VKD3D-Proton. Most titles require 16 GB RAM due to x86 emulation overhead.

### Linux vs macOS Performance Parity

As a general observation, the Asahi driver achieves approximately 80–95% of macOS Metal performance on equivalent compute-bound workloads. The remaining gap comes from:

1. **Firmware optimization**: Apple's macOS AGX firmware is tuned for Metal's submission model; the Asahi driver uses the same firmware but generates slightly different submission patterns.
2. **Driver overhead**: Metal's user-mode driver has years of per-workload tuning; Honeykrisp optimises for correctness first.
3. **DVFS tuning**: Apple's macOS power management firmware targets different thermal envelopes from the Linux `asahi_freq` governor defaults.

For bandwidth-bound workloads (large buffer copies, texture streaming), the UMA architecture means Linux and macOS performance are nearly identical — the bottleneck is memory bandwidth, which is hardware-fixed.

### Missing Features (Mid-2026)

| Feature | Status |
|---|---|
| M3/M4 GPU (G15/G16) | Reverse engineering in progress; no GPU acceleration |
| `VK_EXT_mesh_shader` | In development (hardware tessellator partially mapped) |
| Ray tracing (`VK_KHR_ray_tracing_pipeline`) | Not supported — M3/M4 only; hardware reverse engineering pending |
| `VK_KHR_video_decode` / `VK_KHR_video_encode` | Not exposed (video codec hardware not yet reverse-engineered) |
| Sparse textures (virtual texturing) | Supported since Mesa 25.1 (via `DRM_ASAHI_FEATURE_SOFT_FAULTS`) |
| `VK_EXT_external_memory_host` | Under development (CPU pointer import as GPU buffer) |
| Hardware performance counters | Partial — firmware exposes limited counters via RTKit |
| DirectX via VKD3D-Proton | DX12_0 working; DX12_1 (mesh shaders, ray tracing) pending M3+ |

### Upstreaming Status

- **`include/uapi/drm/asahi_drm.h`**: merged into mainline Linux kernel (ahead of the driver itself — a first for the DRM subsystem).
- **`drivers/gpu/drm/asahi/`**: ~21,000 lines of Rust, under upstream review. The review process is lengthy given the driver's size and the novelty of Rust in DRM.
- **`drivers/gpu/drm/apple/`** (DCP display driver): upstream review ongoing in parallel.
- **Mesa Gallium and Honeykrisp**: fully upstream in Mesa. The Asahi Mesa fork has been retired as of Mesa 25.2.
- **`drivers/soc/apple/rtkit.c`**: upstreamed. The RTKit abstraction is used by the NVMe ANS driver and other Apple Silicon drivers already in mainline.
- **M3/M4 support**: not yet in any release; requires fresh reverse engineering of the G15/G16 GPU generation.

### Power Management

AGX frequency scaling on Asahi Linux is managed via the `apple_gpu_freq` driver, which communicates with Apple's PMU firmware to select P-states (performance/voltage states). The GPU frequency is exposed via:

```bash
# GPU frequency control
ls /sys/class/devfreq/    # Apple GPU devfreq device
cat /sys/class/devfreq/*/cur_freq    # Current frequency in Hz
cat /sys/class/devfreq/*/available_frequencies

# CPU frequency (Apple's cpufreq driver)
cat /sys/devices/system/cpu/cpufreq/policy*/cpuinfo_max_freq
```

The GPU and CPU share a thermal power budget managed by Apple's System Management Controller (SMC) firmware. Under sustained GPU load, the CPU frequency may be throttled to keep total SoC power within the thermal design point — a characteristic of the UMA thermal model that differs from discrete GPU systems where the GPU and CPU have independent thermal limits.

---

## Integrations

This chapter connects to the following chapters across the book:

**[Chapter 8 — Nouveau (NVIDIA Open Driver)](../../chapters/part-02-gpu-drivers/ch08-nouveau.md)**: The closest analogy to Asahi's reverse-engineering methodology. Nouveau used `LD_PRELOAD` wrappers and `mmt-replay` for FIFO capture; Asahi used `DYLD_INSERT_LIBRARIES` and IOKit interception. Both projects document undocumented ISAs (envytools for NVIDIA; dougallj/applegpu for AGX). The fundamental difference: Nouveau targets a FIFO-based command processor that the CPU programs directly, while Asahi targets a firmware-mediated ASC that the CPU never programs directly.

**[Chapter 9 — Lima/Panfrost (ARM Open Drivers)](../../chapters/part-02-gpu-drivers/ch09-panfrost.md)**: Panfrost and Asahi share code in Mesa's `src/` tree — in particular, compiler infrastructure patterns and the `pan_` utility libraries. Panfrost also reverse-engineered ARM Mali ISAs; the experience of the Panfrost team (Alyssa Rosenzweig was also a Panfrost contributor before Asahi) directly informed Asahi's approach to ISA documentation.

**[Chapter 10 — Nova (RISC-V GPU Rust Driver)](../../chapters/part-02-gpu-drivers/ch10-nova.md)**: Nova is another DRM driver written in Rust, for NVIDIA's open-firmware GPUs. Where Asahi is a reverse-engineering effort with a closed firmware binary, Nova targets NVIDIA's open GSP firmware with published interfaces. The comparison illustrates how Rust in DRM addresses different problems: for Asahi, compile-time correctness over an undocumented ABI; for Nova, memory-safe abstraction over a newly-published spec.

**[Chapter 14 — NIR (Mesa Intermediate Representation)](../../chapters/part-03-mesa/ch14-nir.md)**: The AGX compiler backend (`src/asahi/compiler/`) uses NIR as its input IR. All the NIR lowering passes described in this chapter — varying lowering, uniform lowering, image lowering, robustness, control flow — are NIR transformation passes. Understanding NIR is prerequisite for understanding the AGX compiler.

**[Chapter 73 — Asahi Linux Introduction](ch73-asahi-apple-silicon.md)**: The companion introductory chapter covering the same project. Chapter 73 provides the broad overview; this chapter (134) goes deeper on the ISA, firmware protocol, Honeykrisp Vulkan driver internals, and UMA implications.

**[Chapter 75 — DMA-BUF](../../chapters/part-07-sync/ch75-dmabuf.md)**: DMA-BUF is the mechanism enabling zero-copy cross-engine sharing on Asahi Linux — camera ISP to GPU, GPU to display DCP. On the UMA architecture, DMA-BUF sharing means sharing the same physical pages, making it structurally zero-copy in a way discrete GPU systems cannot match.

**[Chapter 119 — Zink (OpenGL on Vulkan)](../../chapters/part-04-opengl/ch119-zink.md)**: Zink was the initial path to higher OpenGL versions on AGX before the Gallium driver reached OpenGL 4.6. Even today, `GALLIUM_DRIVER=zink` can run OpenGL on top of Honeykrisp, giving a second implementation path for OpenGL applications on Apple Silicon.

**[Chapter 129 — GPU Firmware Loading](../../chapters/part-11-advanced/ch129-gpu-firmware.md)**: The AGX firmware (`gpu.macho`) is loaded via `request_firmware()`, sharing the Linux firmware loading infrastructure with other GPU drivers. The specifics of Apple's RTKit boot protocol make it a more complex firmware loading sequence than typical — covering firmware loading, memory mapping, RTKit IPC handshake, and boot-done acknowledgement.

---

## References

- [Asahi Linux — Project Home](https://asahilinux.org/)
- [Asahi Linux Blog](https://asahilinux.org/blog/)
- ["GPU drivers now in Asahi Linux" (December 2022)](https://asahilinux.org/2022/12/gpu-drivers-now-in-asahi-linux/)
- ["Tales of the M1 GPU" (November 2022)](https://asahilinux.org/2022/11/tales-of-the-m1-gpu/)
- ["Conformant OpenGL® 4.6 on the M1" (February 2024)](https://asahilinux.org/2024/02/conformant-gl46-on-the-m1/)
- ["Vulkan 1.3 on the M1 in 1 month" (June 2024)](https://asahilinux.org/2024/06/vk13-on-the-m1-in-1-month/)
- ["AAA gaming on Asahi Linux" (October 2024)](https://asahilinux.org/2024/10/aaa-gaming-on-asahi-linux/)
- ["Progress Report: Linux 6.16" (August 2025)](https://asahilinux.org/2025/08/progress-report-6-16/)
- ["Progress Report: Linux 6.19" (February 2026)](https://asahilinux.org/2026/02/progress-report-6-19/)
- [Apple GPU (AGX) — Asahi Linux Documentation](https://asahilinux.org/docs/hw/soc/agx/)
- [Dougall Johnson — Apple GPU ISA Documentation](https://dougallj.github.io/applegpu/docs.html)
- [applegpu GitHub — ISA disassembler/assembler](https://github.com/dougallj/applegpu)
- [m1n1 Bootloader/Hypervisor](https://github.com/AsahiLinux/m1n1)
- [AsahiLinux/linux — Rust DRM Driver](https://github.com/AsahiLinux/linux/tree/gpu/rust-wip/drivers/gpu/drm/asahi)
- [`include/uapi/drm/asahi_drm.h` — Linux mainline UAPI](https://github.com/torvalds/linux/blob/master/include/uapi/drm/asahi_drm.h)
- [`drivers/soc/apple/rtkit.c` — Apple RTKit driver](https://github.com/torvalds/linux/blob/master/drivers/soc/apple/rtkit.c)
- [Mesa Asahi Driver Documentation](https://docs.mesa3d.org/drivers/asahi.html)
- [Mesa Asahi Source Tree](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/asahi)
- [Asahi Linux UAPI v5 patch series (March 2025)](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20250326-agx-uapi-v5-1-04fccfc9e631@rosenzweig.io/)
- [Rust for Linux — Apple AGX GPU Driver](https://rust-for-linux.com/apple-agx-gpu-driver)
- [AsahiLinux/gpu — Archived early research repository](https://github.com/AsahiLinux/gpu)
- [Asahi Linux Wiki — SW:AGX driver notes](https://leo3418.github.io/asahi-wiki-build/swagx-driver-notes/)
