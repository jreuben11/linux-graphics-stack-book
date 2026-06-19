# Chapter 100: etnaviv: The Vivante GPU Open Driver

**Target audiences**: Embedded Linux driver developers working with NXP i.MX SoCs, SoC platform engineers integrating the Vivante GC-series GPU, developers building Wayland compositors or OpenGL ES applications on i.MX6/i.MX8 hardware, and systems engineers reverse-engineering proprietary GPU interfaces.

---

## Table of Contents

1. [Introduction: The Vivante GC-Series GPU Family](#1-introduction-the-vivante-gc-series-gpu-family)
2. [Reverse Engineering History](#2-reverse-engineering-history)
3. [Vivante GPU Architecture](#3-vivante-gpu-architecture)
4. [etnaviv Kernel Driver](#4-etnaviv-kernel-driver)
5. [Vivante ISA and Instruction Encoding](#5-vivante-isa-and-instruction-encoding)
6. [Mesa etnaviv Gallium Driver](#6-mesa-etnaviv-gallium-driver)
7. [Shader Compiler: NIR to Vivante ISA](#7-shader-compiler-nir-to-vivante-isa)
8. [GC7000, OpenCL, and the Vulkan Horizon](#8-gc7000-opencl-and-the-vulkan-horizon)
9. [Real-World Platforms](#9-real-world-platforms)
10. [Debugging and Contributing](#10-debugging-and-contributing)
11. [Integrations](#11-integrations)

---

## 1. Introduction: The Vivante GC-Series GPU Family

Vivante Corporation (later acquired by VeriSilicon) produced a family of small, power-efficient GPU IP cores sold to SoC vendors as licensable silicon. Unlike the better-known ARM Mali family, Vivante's GC-series cores were rarely documented publicly: no ISA specification, no programming guide. This opacity made reverse engineering the only path to an open-source driver, and that driver — **etnaviv** — is the result. Understanding etnaviv requires understanding not just the driver architecture but also the community effort that uncovered the hardware's behaviour from first principles.

The GC-series spans a wide capability range:

- **GC400 / GC320 / GC520** — 2D-only or minimal 3D cores without programmable shader units. GC400 handles 2D composition accelerated through a fixed-function pipeline. GC320 is the bitblt/stretch engine used alongside 3D cores on i.MX6. These do not support the etnaviv 3D path.
- **GC600** — a small 3D core deployed on Marvell ARMADA SoCs and i.MX8MM. Supports OpenGL ES 2.0 with programmable vertex and fragment shaders. Found in production on the i.MX8M Mini.
- **GC1000 / GC1000+** — mid-range OpenGL ES 2.0 core. Deployed on Amlogic S905 (AML S905X/X3 series), Ingenic JZ4780, and Rockchip RK3168.
- **GC2000** — full OpenGL ES 2.0 with a more capable shader engine, found on the **NXP i.MX6 Quad** (the dominant embedded Linux platform of the early 2010s) alongside a GC320 2D engine and GC355 vector graphics engine. The i.MX6Q SabreLite reference board uses this configuration.
- **GC7000 / GC7000L / GC7000Lite** — the current-generation 3D core, supporting **HALTI5** (Hardware Architecture Level Tiers) feature flags corresponding to OpenGL ES 3.x capabilities including geometry shaders, multiple render targets, and compute shaders. The GC7000Lite appears in the **NXP i.MX8MQ** (used in the **Librem 5** smartphone). The GC7000L appears in the **NXP i.MX8M Mini** (GC600 for 3D).
- **GC7000UL** — a newer GC7000 variant found in the **NXP i.MX8M Plus**, notable for its compute shader engine (enabling OpenCL 1.2 via etnaviv). Supports OpenGL ES 3.1/3.0. Hardware revision GC7000 r6204 (also called r6214 in later stepping). The UL variant has a single pixel pipe; the i.MX8MQ's GC7000L/Lite part has different rendering characteristics. Variant capability differences should be confirmed against hardware feature bitfields.

The driver stack lives across three upstream repositories:

- **etna_viv** (tools): [https://github.com/etnaviv/etna_viv](https://github.com/etnaviv/etna_viv) — register documentation, command-stream analysis tools, and the ISA disassembler.
- **Linux kernel**: `drivers/gpu/drm/etnaviv/` — the DRM kernel driver, mainlined in Linux 4.5.
- **Mesa**: `src/gallium/drivers/etnaviv/` — the Gallium3D userspace driver with NIR-based shader compiler.

This chapter focuses on the 3D-capable path: GC2000 through GC7000UL, using the etnaviv DRM kernel driver paired with the Mesa Gallium driver.

---

## 2. Reverse Engineering History

### Origins: Wladimir J. van der Laan and etna_viv

The etnaviv project began as **etna_viv**, started around 2012 by **Wladimir J. van der Laan** ([github.com/laanwj](https://github.com/laanwj/etna_viv)). The target hardware was the Vivante GC2000 embedded in the Freescale i.MX6, which was appearing in rapidly growing numbers of embedded Linux products but lacked any open-source 3D driver.

The methodology drew directly from the **Lima** project (which reverse-engineered the ARM Mali Utgard) and the **envytools** approach pioneered for NVIDIA hardware: intercept and decode the binary register writes that the proprietary GPU driver blob makes, then infer the hardware's programming model from those traces.

Concretely, the workflow was:

1. Run the proprietary **GalCore** driver (kernel module `galcore.ko`) on an i.MX6 device.
2. Intercept calls between the userspace EGL/GLES library (`libGAL.so`) and the kernel driver using `LD_PRELOAD`-based interception.
3. Capture GPU command buffers dumped into memory, then decode them offline using register maps.
4. Build register documentation from the decoded traces using the **rnndb rules-ng-ng XML** format (borrowed directly from envytools), producing `rnndb/isa.xml`, `rnndb/state.xml`, and `rnndb/state_3d.xml` — the same machine-readable register specs that were later included verbatim in the Linux kernel driver as `state.xml.h`, `state_3d.xml.h`, etc.

The etna_viv project also built the first user-space **shader assembler and disassembler** for the Vivante ISA, enabling validation of reverse-engineered instruction encoding by comparing disassembled blobs against expected GLSL semantics. Tools included `dump_cmdstream.py` (command-stream decoder), `fdr_dump_mem.py` (video memory extraction at specific execution points), and a **GDB plugin** for live GPU state inspection.

[Source: etna_viv README](https://github.com/etnaviv/etna_viv)

### Pengutronix: Christian Gmeiner and Lucas Stach

While van der Laan focused on tools and documentation, **Christian Gmeiner** and **Lucas Stach** at [Pengutronix](https://pengutronix.de/) took on the production-quality DRM kernel driver and Mesa Gallium driver. Pengutronix is a German embedded Linux consultancy with extensive experience upstreaming SoC drivers, and they became the primary long-term maintainers of the driver.

Lucas Stach submitted the **staging: etnaviv** patch series (111 patches) to the DRI mailing list in April 2015, and after review the driver moved out of staging and into `drivers/gpu/drm/etnaviv/` in **Linux 4.5** (released March 2016). This was a significant milestone: unlike many vendor GPU drivers that entered staging but never left, etnaviv completed the full upstreaming cycle relatively quickly.

[Source: XDC 2015 presentation, Stach/etnaviv](https://www.x.org/wiki/Events/XDC2015/Program/Stach_etnaviv.pdf)

A key advantage of etnaviv's methodology compared to some other reverse-engineering efforts was the quality of the register documentation inherited from etna_viv. Because the rnndb XML files described hardware registers in a structured, machine-readable form, they could be directly converted into C header files (`state.xml.h`) using the same toolchain that envytools uses for NVIDIA hardware. This meant the kernel driver's register access code was self-documenting and auditable against the captured traces, rather than consisting of opaque magic constants.

### No Public ISA Documentation

The Vivante GC-series cores have never had public ISA documentation or a programmer's reference manual. As of 2026, all knowledge of the instruction encoding, pipeline stages, and hardware quirks in etnaviv derives from:

- Command-stream captures from the proprietary `galcore.ko` driver
- Analysis of the proprietary shader compiler's output
- Empirical testing (rendering known patterns, comparing against expected output)
- Blog posts and presentations by the etnaviv developers themselves

This situation contrasts with freedreno (Qualcomm Adreno), which benefits from occasional reverse-engineering assistance from Qualcomm engineers, and Intel, which publishes hardware specifications. The absence of documentation means that hardware corner cases are discovered through CTS failures rather than specification study, and that GLES3 conformance has been slower to achieve than on platforms with public ISA documentation.

---

## 3. Vivante GPU Architecture

### Pipeline Stages

The Vivante GC2000 and GC7000 graphics pipeline follows a traditional architecture with the following fixed stages:

```
FE → VS → PA → SE → RA → PS → PE → RS
```

- **FE (Front End)**: A DMA engine that reads the command buffer from memory and dispatches state changes and draw calls. The FE processes VIVS_FE_* registers.
- **VS (Vertex Shader)**: Executes vertex shader programs. Driven by VBO data read via `VIVS_FE_VERTEX_STREAM_*` registers.
- **PA (Primitive Assembly)**: Assembles triangles/lines/points from VS output, performs clipping, and generates per-primitive data.
- **SE (Setup Engine)**: Calculates edge equations and other rasterisation parameters for each primitive.
- **RA (Rasterizer)**: The tile-based rasteriser. Walks tiles and generates fragments for pixels covered by primitives.
- **PS (Pixel Shader)**: Executes fragment shader programs per tile. On GC2000, the PS executes shaders for one 4×4 tile at a time.
- **PE (Pixel Engine)**: Performs per-fragment operations: alpha blending, depth test, stencil test.
- **RS (Resolve)**: A dedicated copy/format-convert engine that resolves the tiled internal framebuffer representation to linear scanout memory. The RS is also used for colour clear operations and texture upload format conversion.

[Source: etna_viv hardware documentation](https://github.com/etnaviv/etna_viv/blob/master/doc/hardware.md)

### Unified Shader ISA

One of the most significant architectural features of the Vivante GC-series is the **unified shader ISA**: vertex shaders and fragment shaders share exactly the same instruction set. There is no separate vertex shader ISA and fragment shader ISA. This means a single compiler backend can target both shader stages, which simplifies the Mesa driver considerably. (Compare with ARM Mali Utgard, which has completely different GP and PP instruction sets requiring two separate compilers in Lima.)

The execution model is **vec4 SIMD**: each shader instruction operates on four-component floating-point vectors. Instructions can read from three source operands with per-component swizzling and write to a destination with a component mask. Both vertex and pixel shaders execute through the same physical ALU hardware, though they run at different pipeline stages and draw from different register files.

### Tile-Based Rendering and Tiling Formats

Vivante GPUs use a tile-based internal representation for render targets and textures. Understanding the two tiling levels is essential for driver performance tuning:

**4×4 tile**: The fundamental unit of rasterisation. The PS processes one tile (16 pixels) at a time. Internal framebuffers and textures are stored with pixels arranged in 4×4 row-major tiles. A 1920×1080 render target has (1920/4) × (1080/4) = 129,600 tiles.

**64×64 supertile**: A hierarchical arrangement of 4×4 tiles into a 64×64-pixel macro-region. The GPU's DMA engines and RS unit prefer supertiled layout because it has much better cache locality for 2D spatial access patterns than either linear or simple tiled layout. The proprietary driver pads render buffers to multiples of 64 pixels in both dimensions. Supertiles follow a space-filling traversal order within the 64×64 block.

The three layouts used by the etnaviv Gallium driver are:

| Layout | etnaviv constant | Use case |
|--------|-----------------|----------|
| Linear | `ETNA_LAYOUT_LINEAR` | CPU-accessible buffers, import/export |
| Tiled | `ETNA_LAYOUT_TILED` | Textures, fast GPU access |
| Supertiled | `ETNA_LAYOUT_SUPER_TILED` | Render targets, best GPU performance |

The RS (Resolve) engine converts between these layouts at the end of each render pass, so the compositor can read a linear scanout buffer while the GPU renders to a supertiled intermediate.

### Memory Management Unit (MMU)

The GC2000 and later cores include a GPU-side **MMU** that provides a separate 32-bit virtual address space for each GPU context. The MMU uses a two-level page table: a directory of 1024 entries (PDE), each pointing to a page table of 1024 entries (PTE), giving 4 KB pages across a 4 GB virtual space.

The kernel driver manages MMU contexts through `etnaviv_iommu.c` and `etnaviv_iommu_v2.c`. `etnaviv_iommu_v2.c` handles the GC7000's improved MMU generation (IOMMU v2), which supports larger physical address spaces. The MMU is programmed through `VIVS_MMU_CONTROL` and related registers; the kernel initialises the page table base address in `VIVS_MMU_TABLE_ARRAY_PTR` before command submission.

On platforms with system IOMMU (e.g., i.MX8 using ARM SMMU), the driver can optionally use the system IOMMU in addition to the GPU-internal MMU, providing DMA isolation at the SoC level.

### Clock Domains

The Vivante GPU exposes two or three clock domains that the kernel driver controls independently:

- **Core clock** (`clk_core`): Drives the main 3D GPU logic including the shader ALUs.
- **Shader clock** (`clk_shader`): On some cores (GC2000 and later), the shader units run at a separate frequency that can be scaled for power management. `etnaviv_gpu_update_clock()` in `etnaviv_gpu.c` manages runtime frequency adjustment.
- **Bus clock** (`clk_bus`, `clk_reg`): AXI/AHB interface clocks for memory access. These are managed by the SoC clock framework and referenced in the device tree `clock-names`.

---

## 4. etnaviv Kernel Driver

### Source Tree Layout

The etnaviv kernel driver lives at `drivers/gpu/drm/etnaviv/` in the Linux kernel tree. The key source files are:

[Source: Linux kernel etnaviv directory](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/etnaviv)

```
drivers/gpu/drm/etnaviv/
├── etnaviv_drv.c        # Platform driver entry, drm_driver ops, device enumeration
├── etnaviv_drv.h        # Core structs: etnaviv_drm_private, IOCTL declarations
├── etnaviv_gem.c        # GEM object lifecycle: create, free, mmap
├── etnaviv_gem.h        # struct etnaviv_gem_object (shmem-backed)
├── etnaviv_gem_submit.c # IOCTL handler: DRM_IOCTL_ETNAVIV_GEM_SUBMIT
├── etnaviv_gem_prime.c  # DMA-BUF import/export (PRIME)
├── etnaviv_cmdbuf.c     # Command buffer (ring buffer) management
├── etnaviv_gpu.c        # Per-GPU state, clock management, reset, submit
├── etnaviv_gpu.h        # struct etnaviv_gpu
├── etnaviv_sched.c      # drm_gpu_scheduler integration
├── etnaviv_mmu.c        # GPU MMU address space management
├── etnaviv_iommu.c      # IOMMU v1 support
├── etnaviv_iommu_v2.c   # IOMMU v2 (GC7000 generation)
├── etnaviv_buffer.c     # Command stream packet construction helpers
├── etnaviv_cmd_parser.c # Command buffer validation (security)
├── etnaviv_hwdb.c       # Hardware database: known GPU model quirks
├── etnaviv_perfmon.c    # Performance monitor counters
├── etnaviv_dump.c       # GPU state dump on hang
├── state.xml.h          # Auto-generated register definitions (from rnndb)
├── state_3d.xml.h       # 3D pipeline register definitions
├── state_blt.xml.h      # BLT/RS unit register definitions
├── state_hi.xml.h       # High-level/global register definitions
└── cmdstream.xml.h      # Command stream packet type definitions
```

The `*.xml.h` header files are generated from the rnndb XML register database inherited from the etna_viv project. This is the same mechanism used by the Nouveau kernel driver (which generates `nvhw/*.xml.h` from envytools). The register database acts as a living specification: any hardware quirk discovered through testing is recorded here.

### Platform Driver and DRM Registration

`etnaviv_drv.c` implements the platform driver that matches the device tree `compatible = "vivante,gc"` nodes:

```c
/* drivers/gpu/drm/etnaviv/etnaviv_drv.c — simplified */
static const struct of_device_id etnaviv_of_match[] = {
    { .compatible = "vivante,gc" },
    { }
};

static struct platform_driver etnaviv_platform_driver = {
    .probe   = etnaviv_pdev_probe,
    .remove  = etnaviv_pdev_remove,
    .driver  = {
        .name           = "etnaviv",
        .of_match_table = etnaviv_of_match,
        .pm             = &etnaviv_pm_ops,
    },
};
```

Because a single SoC may contain multiple GPU cores (e.g., an i.MX6Q has a GC2000 3D core plus a GC320 2D core as separate device tree nodes), the driver uses a **component-based** binding via `component_add()`. A master driver aggregates all matched GPU nodes under one `drm_device`. The `drm_driver` ops table registered via `drm_dev_register()` hooks into the standard DRM infrastructure for file operations, IOCTL dispatch, and GEM shmem management.

### GEM Objects

Buffer allocation is handled by `etnaviv_gem.c`. The `struct etnaviv_gem_object` wraps a `drm_gem_shmem_object`:

```c
/* drivers/gpu/drm/etnaviv/etnaviv_gem.h */
struct etnaviv_gem_object {
    struct drm_gem_shmem_object base;  /* must be first */
    const struct etnaviv_gem_ops *ops;
    struct list_head    vram_node;
    struct mutex        lock;
    /* GPU virtual mappings: one per MMU context */
    struct list_head    mappings;
    dma_addr_t          paddr;         /* physical address if pinned */
};
```

Userspace allocates GEM objects via `DRM_IOCTL_ETNAVIV_GEM_NEW`, which returns a GEM handle. Objects are backed by anonymous shared memory (shmem) pages. GPU virtual addresses are assigned lazily when a buffer is first submitted to the GPU, at which point the driver allocates a range in the GPU MMU address space and installs page table entries.

### Command Buffer Submission

The central IOCTL is `DRM_IOCTL_ETNAVIV_GEM_SUBMIT`, implemented in `etnaviv_gem_submit.c` via `etnaviv_ioctl_gem_submit()`. The userspace structure is:

```c
/* include/uapi/drm/etnaviv_drm.h */
struct drm_etnaviv_gem_submit {
    __u32  fence;       /* out: fence sequence number */
    __u32  pipe;        /* in: GPU pipe (3D, 2D, VG) */
    __u32  exec_state;  /* in: initial execution state */
    __u32  nr_bos;      /* in: number of buffer objects */
    __u32  nr_relocs;   /* in: number of relocations */
    __u32  stream_size; /* in: command buffer size in bytes */
    __u64  bos;         /* in: ptr to drm_etnaviv_gem_submit_bo[] */
    __u64  relocs;      /* in: ptr to drm_etnaviv_gem_submit_reloc[] */
    __u64  stream;      /* in: ptr to command buffer data */
    __u32  flags;       /* in: ETNA_SUBMIT_* flags */
    __s32  fence_fd;    /* in/out: sync_file fence */
    __u64  pmrs;        /* in: performance monitor requests */
    __u32  nr_pmrs;
    __u32  pad;
};
```

The submit path performs the following steps:

1. **Validation**: `etnaviv_cmd_parser.c` walks every word in the command buffer, checking that each LOAD_STATE packet targets only permitted register ranges. This is the security boundary: untrusted userspace cannot write arbitrary GPU registers.
2. **BO mapping**: Each GEM handle in the `bos` array is resolved to an `etnaviv_gem_object`, mapped into the GPU MMU context if not already, and its GPU virtual address is patched into the command buffer at the relocation offsets.
3. **Fence setup**: If `fence_fd` is provided with `ETNA_SUBMIT_FENCE_FD_IN`, the submission waits on the incoming `sync_file` before scheduling. An outgoing fence is created and returned if `ETNA_SUBMIT_FENCE_FD_OUT` is set.
4. **Scheduler enqueue**: The submission is wrapped in a `drm_sched_job` and enqueued to the `drm_gpu_scheduler` instance in `etnaviv_sched.c`.

### GPU Scheduler Integration

`etnaviv_sched.c` wraps DRM's `drm_gpu_scheduler` (the same scheduler used by AMDGPU, Panfrost, Lima, and others). Each physical GPU has one scheduler instance. The scheduler provides:

- **Priority queues**: Multiple software queues per process context with priority inheritance.
- **Dependency tracking**: `drm_sched_job` depends on incoming fences; the scheduler dequeues jobs only when dependencies are satisfied.
- **Timeout and hang detection**: The scheduler monitors in-flight jobs with a configurable timeout. On timeout, it calls `etnaviv_sched_timedout_job()`, which triggers GPU recovery.

### GPU Reset and Hang Recovery

The `etnaviv_gpu_recover_hang()` function in `etnaviv_gpu.c` handles the GPU hang recovery path:

1. Save GPU debug state via `etnaviv_dump.c` (register snapshot, command buffer contents).
2. Disable GPU interrupts.
3. Assert GPU reset via `etnaviv_hw_reset()`, which writes the reset bit in `VIVS_HI_CLOCK_CONTROL`.
4. Re-initialise GPU state: reload MMU, reload command buffer ring, re-enable interrupts.
5. Complete all outstanding fence objects so blocked waiters are unblocked (with an error status).
6. Resume the scheduler.

### Capability Queries

Userspace discovers GPU capabilities via `DRM_IOCTL_ETNAVIV_GET_PARAM`:

```c
/* Key ETNAVIV_PARAM_* constants from include/uapi/drm/etnaviv_drm.h */
ETNAVIV_PARAM_GPU_MODEL           = 0x01  /* e.g. 0x7000 for GC7000 */
ETNAVIV_PARAM_GPU_REVISION        = 0x02  /* e.g. 0x6204 */
ETNAVIV_PARAM_GPU_FEATURES_0      = 0x03  /* hardware feature bitfield 0 */
/* ... FEATURES_1 through FEATURES_12 ... */
ETNAVIV_PARAM_GPU_SHADER_CORE_COUNT = 0x14
ETNAVIV_PARAM_GPU_PIXEL_PIPES       = 0x15
ETNAVIV_PARAM_GPU_INSTRUCTION_COUNT = 0x18
ETNAVIV_PARAM_GPU_NUM_CONSTANTS     = 0x19
ETNAVIV_PARAM_GPU_PRODUCT_ID        = 0x1c
```

The Mesa `etnaviv_screen.c` queries these at startup and builds an `etnaviv_specs` struct that all capability queries (`pipe_screen::get_param`, `pipe_screen::get_shader_param`) ultimately consult.

---

## 5. Vivante ISA and Instruction Encoding

### vec4 Instruction Format

The Vivante shader ISA uses a **128-bit instruction word** (four 32-bit words). Each instruction encodes a single vec4 operation in a fixed-field format. All instructions, whether vertex or fragment shader, use this same encoding — this is the unified ISA.

The key fields in a 128-bit instruction are:

| Field | Bits | Description |
|-------|------|-------------|
| Opcode | ~6 bits | Operation type (ADD, MUL, MAD, TEX, …) |
| Destination reg | ~6 bits | Temp register index |
| Write mask | 4 bits | Component enable: x, y, z, w |
| Source 0 | ~14 bits | Register index + swizzle + negate/absolute |
| Source 1 | ~14 bits | Register index + swizzle + negate/absolute |
| Source 2 | ~14 bits | Register index + swizzle + negate/absolute |
| Saturate | 1 bit | Clamp output to [0, 1] |
| Type | ~2 bits | Float/integer mode selector |

Source operands can address: temporary registers (t0–tN), uniform registers (u0–uN), input/varying registers, and constant registers. Component swizzle allows any permutation of `.xyzw` from the 4-component source vector to be applied before use.

[Source: etna_viv ISA documentation](https://github.com/laanwj/etna_viv/blob/master/doc/isa.md)

### Key Opcodes

The ISA includes the following core operations (opcode names taken from the etna_viv disassembler and rnndb ISA XML):

```
NOP                     — no operation
MOV   dst, src0         — register move
ADD   dst, src0, src1   — float add
MUL   dst, src0, src1   — float multiply
MAD   dst, src0, src1, src2  — fused multiply-add: dst = src0 * src1 + src2
DP3   dst.x, src0, src1 — 3-component dot product (result broadcast or .x)
DP4   dst.x, src0, src1 — 4-component dot product
DST   dst, src0, src1   — distance vector
MIN   dst, src0, src1   — component minimum
MAX   dst, src0, src1   — component maximum
RCP   dst, src0         — reciprocal
RSQ   dst, src0         — reciprocal square root
SQRT  dst, src0         — square root (GC7000+)
LOG   dst, src0         — base-2 logarithm
EXP   dst, src0         — base-2 exponential
FRC   dst, src0         — fractional part
TEXLD dst, src0, sampler — texture sample (src0 = coordinates)
BRANCH label            — conditional/unconditional branch
CALL  label             — subroutine call
RET                     — return from subroutine
KILL                    — discard fragment
```

The `TEXLD` instruction (also referred to as `TEX` in some documentation) takes a coordinate source and a sampler register index. Sampler state (filter mode, wrap mode, border colour) is programmed via `VIVS_TE_SAMPLER_*` registers in the state stream, not inline in the instruction.

### Assembly Example

A simple GLSL fragment shader:

```glsl
uniform sampler2D u_texture;
varying vec2 v_texcoord;
void main() {
    gl_FragColor = texture2D(u_texture, v_texcoord);
}
```

Compiles to approximately:

```asm
; Vivante ISA assembly (etna_viv assembler notation)
TEXLD  t0.xyzw, v0.xyyy, tex0   ; sample texture 0 at v0.xy -> t0
MOV    o0.xyzw, t0.xyzw          ; write result to output colour
```

### Comparison with Other Embedded GPU ISAs

Compared to the **Bifrost** ISA used in ARM Mali (covered in Chapter 90), the Vivante ISA is significantly simpler:

- **Bifrost** uses *clause* scheduling: the compiler packs multiple instruction tuples (up to 8) into a clause, managing register dependency via explicit scheduling. **Vivante** uses a simpler sequential issue model with no clause concept.
- **Bifrost** has separate texture, load/store, and arithmetic slots within a tuple. **Vivante** uses a single unified operation per 128-bit word.
- Both ISAs lack public documentation and were reverse-engineered.
- Both use NIR as the compiler IR in their Mesa implementations.

The Lima ISA (Mali-400/450), by contrast, has completely separate vertex (GP) and fragment (PP) ISAs. Vivante's unified ISA is closer to what NVIDIA uses internally, and simplifies the etnaviv shader compiler substantially.

---

## 6. Mesa etnaviv Gallium Driver

### Source Tree Overview

The Mesa Gallium driver lives at `src/gallium/drivers/etnaviv/`. As of Mesa 26.x (2026), the key files are:

```
src/gallium/drivers/etnaviv/
├── etnaviv_screen.c      # pipe_screen: capability queries, GPU detection
├── etnaviv_screen.h
├── etnaviv_context.c     # pipe_context: draw, state binding, flush
├── etnaviv_context.h
├── etnaviv_resource.c    # Texture/buffer allocation: etnaviv_resource struct
├── etnaviv_resource.h
├── etnaviv_state.c       # Blend/rasterizer/depth-stencil state → GPU registers
├── etnaviv_emit.c        # Command buffer packet emission
├── etnaviv_draw.c        # Draw call dispatch (etnaviv_draw_vbo)
├── etnaviv_shader.c      # Shader object: compile, cache, bind
├── etnaviv_compiler.c    # Legacy TGSI compiler (no longer default)
├── etnaviv_compiler_nir.c # NIR-based compiler (default since Mesa 21.3)
├── etnaviv_fence.c       # Gallium fence wrapping kernel fences
├── etnaviv_query.c       # Occlusion queries, timestamp queries
├── etnaviv_surface.c     # Framebuffer surface views
├── etnaviv_texture.c     # Sampler state, texture binding
├── etnaviv_translate.h   # Pipe format → Vivante format translation tables
├── etnaviv_tiling.c      # Tiling layout helpers
├── etnaviv_drm_public.h  # Public entry: etnaviv_drm_screen_create()
└── etnaviv_bo.h          # libdrm-etnaviv buffer object wrappers
```

### Screen Creation and Capability Reporting

`etnaviv_screen.c` implements `pipe_screen`. On startup via `etnaviv_drm_screen_create()`, the screen:

1. Queries `ETNAVIV_PARAM_GPU_MODEL` and `ETNAVIV_PARAM_GPU_REVISION` to identify the GPU.
2. Reads all `ETNAVIV_PARAM_GPU_FEATURES_*` bitfields and populates `struct etnaviv_specs` — the driver's internal capability descriptor.
3. Inspects hardware feature bits to enable/disable higher-level capabilities: OpenGL ES 3.x extensions, compute shaders, multiple render targets, etc.
4. Reports Gallium caps back to the state tracker via `etnaviv_screen_get_param()` and `etnaviv_screen_get_shader_param()`.

The `etnaviv_specs` struct drives nearly all conditional logic in the driver. Key fields include:

```c
/* From src/gallium/drivers/etnaviv/etnaviv_screen.h (simplified) */
struct etnaviv_specs {
    uint32_t  model;
    uint32_t  revision;
    uint32_t  halti;         /* HALTI level: 0=GLES2, 2=GLES3.0, 5=GLES3.1 */
    uint32_t  num_constants;
    uint32_t  max_registers;
    uint32_t  num_varyings;
    uint32_t  pixel_pipes;
    bool      has_shader_enhancements; /* GC2000+ feature */
    bool      has_texture_array;
    bool      has_msaa;
    bool      has_compute;   /* GC7000 compute shaders */
    /* ... many more feature flags ... */
};
```

The `halti` field (Hardware Architecture Level Tier Index) is the primary discriminator for GLES version support. HALTI 0–1 corresponds to OpenGL ES 2.0; HALTI 2 adds OpenGL ES 3.0 mandatory extensions; HALTI 5 (GC7000 r6204) corresponds to the feature set needed for OpenGL ES 3.1. **Note: the exact HALTI level to GLES version mapping should be verified against `etnaviv_screen.c` in the current Mesa source.**

### Resource Allocation

`etnaviv_resource.c` manages `struct etnaviv_resource`, which wraps a `pipe_resource`:

```c
struct etnaviv_resource {
    struct pipe_resource    base;
    struct etnaviv_bo      *bo;      /* GEM buffer object (libdrm-etnaviv) */
    enum etna_resource_addressing_mode addressing_mode;
    struct etnaviv_resource *texture; /* separate tiled copy for texture access */
    bool                    texture_is_rs_target; /* RS unit manages conversion */
    uint32_t                ts_bo;   /* tile status buffer (fast clear) */
    /* Per-level layout info */
    struct etnaviv_resource_level {
        uint32_t  offset;    /* byte offset in bo */
        uint32_t  stride;    /* row stride in bytes */
        uint32_t  layer_stride;
    } levels[ETNA_NUM_LOD];
};
```

Tiling layout selection during allocation follows this logic in `etnaviv_resource_create()`:

- Render targets: **supertiled** layout (`ETNA_LAYOUT_SUPER_TILED`) — best GPU performance.
- Textures: **tiled** layout (`ETNA_LAYOUT_TILED`) — GPU texture cache friendly.
- Staging buffers (CPU-accessible): **linear** layout (`ETNA_LAYOUT_LINEAR`).
- When a resource is used as both a render target and a texture, the driver maintains two GEM objects (a supertiled render target and a tiled texture copy) and uses the RS engine to convert between them at the end of each render pass.

### State Objects and Command Buffer Emission

Gallium state objects (blend state, rasterizer state, depth-stencil state) are created by functions in `etnaviv_state.c`. Each state object precomputes the register values for the Vivante hardware and stores them in a compact form. When the state object is bound, `etnaviv_emit.c` generates `LOAD_STATE` command buffer packets that write those register values to the GPU.

For example, blend state creation computes the values for `VIVS_PE_ALPHA_BLEND_COLOR`, `VIVS_PE_ALPHA_CONFIG`, and `VIVS_PE_COLOR_FORMAT` at `bind_blend_state` time, avoiding per-draw computation.

### Draw Call Dispatch

The central draw function `etnaviv_draw_vbo()` in `etnaviv_draw.c` (implementing `pipe_context::draw_vbo`) performs the following:

1. **Dirty state emission**: Check all dirty state flags; emit changed state as LOAD_STATE packets into the command buffer via `etnaviv_emit_state()`.
2. **Vertex buffer setup**: Emit `VIVS_FE_VERTEX_STREAM_BASE_ADDR`, `VIVS_FE_VERTEX_STREAM_CONTROL` packets for each active vertex buffer binding.
3. **Index buffer setup**: If indexed, emit `VIVS_FE_INDEX_STREAM_BASE_ADDR` and `VIVS_FE_INDEX_STREAM_CONTROL`.
4. **Draw command**: Emit the appropriate `DRAW_PRIMITIVES` or `DRAW_INDEXED_PRIMITIVES` command packet with primitive type and count.
5. **Flush tracking**: Mark relevant resources as GPU-modified.

### Context Flush and Submit

`etnaviv_flush()` (implementing `pipe_context::flush`) completes the current command buffer and submits it to the kernel via the `DRM_IOCTL_ETNAVIV_GEM_SUBMIT` IOCTL. The libdrm-etnaviv wrapper (`libdrm/etnaviv/`) provides the userspace API that the Mesa driver calls; this library manages command buffer ring management and the ioctl call itself.

---

## 7. Shader Compiler: NIR to Vivante ISA

### Historical Context: TGSI Then NIR

The etnaviv Mesa driver originally used a custom TGSI-based compiler. NIR support was added experimentally in **Mesa 19.2** (August 2019) by **Christian Gmeiner**, initially opt-in via `ETNA_MESA_DEBUG=nir`. Emma Anholt later completed the transition and the TGSI compiler backend was **removed entirely** in Mesa (merge request [!12889](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/12889)), making NIR the sole compilation path as of **Mesa 21.3** (November 2021).

[Source: Phoronix — Etnaviv Gallium3D Picks Up A NIR Compiler](https://www.phoronix.com/scan.php?page=news_item&px=Etnaviv-NIR-Compiler-Mesa-19.2)

The switch to NIR brought immediate benefits: shared optimisation passes with other Mesa drivers (common-subexpression elimination, dead-code elimination, algebraic simplifications), better correctness for edge cases, and a path toward OpenGL ES 3.x support through NIR lowering passes that can express ES3 semantics.

### Compilation Pipeline

Shader compilation in etnaviv (as of Mesa 26.x) follows these stages:

**Stage 1: GLSL → NIR** (handled by the Mesa common GLSL compiler)

The OpenGL state tracker compiles GLSL to NIR using `_mesa_glsl_compile_shader()`. The resulting NIR is in SSA form.

**Stage 2: NIR Lowering**

`etnaviv_compiler_nir.c` applies a sequence of NIR lowering passes before instruction selection:

```c
/* Simplified lowering sequence from etnaviv_compiler_nir.c */
NIR_PASS(_, nir, nir_lower_vars_to_ssa);
NIR_PASS(_, nir, nir_lower_io, nir_var_shader_in | nir_var_shader_out, ...);
NIR_PASS(_, nir, nir_lower_tex, &tex_options);
NIR_PASS(_, nir, nir_lower_alu_to_scalar, NULL, NULL);
NIR_PASS(_, nir, nir_lower_idiv, &idiv_options);
NIR_PASS(_, nir, nir_opt_algebraic);
NIR_PASS(_, nir, nir_opt_constant_folding);
NIR_PASS(_, nir, nir_opt_dce);   /* dead code elimination */
NIR_PASS(_, nir, nir_opt_cse);   /* common subexpression elimination */
```

`nir_lower_io` rewrites explicit I/O variable accesses to use `nir_intrinsic_load_input` / `nir_intrinsic_store_output` intrinsics with byte offsets, which maps directly to the Vivante input/output register file. `nir_lower_tex` handles texture operation lowering, including projective textures and shadow comparisons. `nir_lower_alu_to_scalar` breaks 2-, 3-, and 4-component ALU operations into scalar operations — necessary because although the Vivante ISA operates on vec4 registers, instruction selection still works scalar-at-a-time in the current compiler.

**Stage 3: Instruction Selection**

NIR ALU operations are mapped to Vivante opcodes in the instruction selection pass:

| NIR op | Vivante opcode |
|--------|---------------|
| `nir_op_fadd` | `ADD` |
| `nir_op_fmul` | `MUL` |
| `nir_op_ffma` | `MAD` |
| `nir_op_fdot3` | `DP3` |
| `nir_op_fdot4` | `DP4` |
| `nir_op_frcp` | `RCP` |
| `nir_op_frsq` | `RSQ` |
| `nir_op_fmin` | `MIN` |
| `nir_op_fmax` | `MAX` |
| `nir_op_mov` | `MOV` |
| `nir_texop_tex` | `TEXLD` |

Each selected instruction becomes an `etna_inst` struct (the driver's internal IR):

```c
/* From src/gallium/drivers/etnaviv/etnaviv_asm.h */
struct etna_inst {
    uint8_t    opcode;       /* INST_OPCODE_* from state_3d.xml.h */
    uint8_t    type;         /* INST_TYPE_F32, INST_TYPE_S32, ... */
    uint8_t    sat;          /* saturate output */
    uint8_t    skphp;        /* skip half-pixel (PS only) */
    struct etna_inst_dst dst;
    struct etna_inst_src src[3];
    uint8_t    tex;          /* texture unit index (TEX instructions only) */
};
```

**Stage 4: Register Allocation**

The compiler performs register allocation over the `etna_inst[]` array using a simple linear-scan allocator. Vivante GPUs expose a fixed number of temporary registers (`ETNAVIV_PARAM_GPU_REGISTER_MAX`), typically 64 on GC2000 and 64 on GC7000. The allocator must also satisfy the vec4 packing rules (four scalar values per register slot).

**Stage 5: Uniform and Varying Handling**

Uniforms are uploaded to the GPU via `VIVS_VS_UNIFORMS` (vertex shader) or `VIVS_PS_UNIFORMS` (pixel shader) state registers. The compiler records the uniform→register mapping, and `etnaviv_shader.c` uploads uniform values into the constant buffer state before each draw call that uses a different uniform configuration.

Varyings (outputs from the vertex shader consumed by the pixel shader) use a linking step: `etnaviv_link_shader()` matches VS output registers to PS input registers and generates the `VIVS_GL_VARYING_*` register state that the hardware interpolation engine uses.

**Stage 6: Binary Emission**

The final `etna_inst[]` array is serialised to 128-bit instruction words via `etna_assemble()` in `etnaviv_asm.c`. Each `etna_inst` is packed into four 32-bit words according to the Vivante instruction encoding, and the resulting byte array is uploaded to the GPU as a shader program via `VIVS_VS_INST_MEM` or `VIVS_PS_INST_MEM` state registers.

### EBC: A Modern Compiler Backend

In 2023, an alternative compiler backend called **EBC (Etnaviv Backend Compiler)** was presented at a Mesa development conference. EBC aims to replace the current instruction-selection pass with a more optimisation-friendly intermediate representation, enabling better register utilisation and targeting the more complex GC7000 feature set (including integer operations and compute shaders). As of 2026, EBC is under active development but not yet the default. [Note: EBC merge status should be verified against current Mesa main.]

---

## 8. GC7000, OpenCL, and the Vulkan Horizon

### GC7000 HALTI5 Capabilities

The GC7000 family represents a generational leap over GC2000. The HALTI5 feature level (exposed by GC7000 r6204 / r6214) enables:

- **OpenGL ES 3.0**: Integer types in shaders, transform feedback, multiple render targets, non-power-of-two textures without restrictions, floating-point render targets.
- **OpenGL ES 3.1**: Compute shaders, shader storage buffer objects (SSBO), atomic operations on image data, indirect draw.
- **Geometry shaders**: Present on GC7000UL and some GC7000L variants. (Note: not all GC7000 die sizes include geometry shader support — verify against hardware feature bits.)
- **MSAA**: 2x and 4x multisample anti-aliasing on GC7000.

Work toward GLES3 conformance on etnaviv continued actively into 2026, with Christian Gmeiner's blog posts documenting specific hardware quirks that required creative workarounds to pass the GLES3 CTS. The primary CI target is a GC7000 rev 6214 (HALTI5) device. [Source: Christian Gmeiner's blog, February 2026](https://christian-gmeiner.info/2026-02-20-gles3-on-etnaviv-fixing-the-hard-parts/)

### OpenCL Support via Rusticl

Compute shader support on GC7000 opened a path to **OpenCL**. The Mesa **Rusticl** frontend (a Rust-based OpenCL implementation, merged September 2022, targeting OpenCL 3.0) works with any Gallium driver that implements compute capabilities via `PIPE_CAP_COMPUTE`.

Tomeu Vizoso (formerly at Collabora) drove the initial work connecting etnaviv's compute shader path to Rusticl, targeting the **VeriSilicon VIPNano-QI NPU** — a Vivante-derived neural-network accelerator found in the Amlogic VIM3 board. The VIPNano-QI is closely related to the GC7000 compute engine. By late 2022, basic OpenCL kernel execution was working, with approximately 1,459 piglit CL tests passing on the VIM3 NPU. [Source: Collabora blog, December 2022](https://www.collabora.com/news-and-blog/blog/2022/12/15/machine-learning-with-etnaviv-and-opencl/)

The Rusticl path requires:

1. `etnaviv_screen` reporting `PIPE_CAP_COMPUTE = 1` (enabled when `etnaviv_specs.has_compute` is set, which requires a GC7000 or GC8000 class core).
2. NIR compute shaders to be lowered through `etnaviv_compiler_nir.c` with the compute-shader lowering passes enabled.
3. SSBO support (`PIPE_CAP_SHADER_BUFFER_OFFSET_ALIGNMENT`).

### Vulkan: Current Status

As of June 2026, **there is no merged Vulkan driver for etnaviv** in Mesa. The hardware (GC7000UL) is theoretically capable of supporting Vulkan 1.0 feature requirements (it has compute, multiple render targets, and the necessary blending modes), and NXP's proprietary driver stack claims Vulkan support. However, the reverse-engineering effort required to implement a correct Vulkan driver from scratch — without a public specification — is substantially larger than the GLES2 effort was. The etnaviv community has discussed Vulkan as a future goal, but no merge request or public branch targeting this exists in Mesa as of this writing.

This contrasts with **Panfrost**, which has the **PanVK** Vulkan driver in `src/panfrost/vulkan/` targeting Mali Bifrost and Valhall — but that effort benefited from a larger team (primarily Collabora) and more recent hardware with better-documented programming models. The etnaviv Vulkan situation is similar to the **freedreno** Vulkan driver (**Turnip**) in its early stages: a community waiting for a motivated team to lead the effort.

For embedded applications requiring Vulkan on i.MX8M Plus hardware, the NXP proprietary stack (`libvulkan_VIVANTE.so`) remains the only option as of 2026.

### Mesa CI and Conformance

etnaviv runs in Mesa's CI on actual hardware via **LAVA** (Linaro Automated Validation Architecture). The CI board is a GC7000 rev 6214 device. Tests run include:

- **dEQP-GLES2**: OpenGL ES 2.0 CTS. etnaviv passes the full GLES2 suite on GC2000 and GC7000 hardware.
- **dEQP-GLES3**: OpenGL ES 3.0 CTS. Partially passing as of mid-2026; work is ongoing.
- **Piglit**: Various Mesa-specific regression tests.
- **shader-db**: Shader compilation quality tracking.

---

## 9. Real-World Platforms

### NXP i.MX8MQ — Librem 5

The **Purism Librem 5** smartphone uses the **NXP i.MX8MQ** SoC, which contains a **Vivante GC7000Lite** GPU (HALTI5, single pixel pipe). The Librem 5 ships with **PureOS** (a Debian-based Linux distribution) and uses the **Phosh** desktop shell — a Wayland compositor built on GNOME Mobile and libhandy, running on top of **Mutter**.

The full graphics stack on the Librem 5 is:

```
Application (GTK4)
    ↓
Wayland (wl_surface, xdg_shell)
    ↓
Phosh/Mutter compositor
    ↓
Mesa etnaviv (OpenGL ES 2.0+, EGL + GBM)
    ↓
libdrm-etnaviv
    ↓
DRM/KMS: etnaviv kernel driver + DCSS display controller
    ↓
GC7000Lite GPU + LCDIF display engine
```

Power management is handled via devfreq and the standard Linux clock framework, with the i.MX8MQ SoC's PLL providing GPU clock scaling.

[Source: Purism community forum — Data from the Librem 5's GC7000Lite GPU](https://forums.puri.sm/t/data-from-the-librem-5s-vivante-gc7000lite-gpu/22226)

### NXP i.MX8M Plus — Developer Boards

The **NXP i.MX8M Plus EVK** (Evaluation Kit) and commercial modules such as the **Variscite DART-MX8M-PLUS** and **SolidRun i.MX8M Plus HummingBoard** use the GC7000UL (r6204), the primary GC7000 test platform for etnaviv developers alongside CI. This is the hardware that enabled OpenCL/compute work.

The GC7000UL (i.MX8MP) is actually reported as having a single pixel pipe and lacking the compression capabilities found in the i.MX8MQ's GC7000L/Lite variant. The key addition over GC2000 is the compute shader engine (HALTI5) and the OpenCL-capable compute pipeline. [Source: Phoronix, Etnaviv iMX8MP Support](https://www.phoronix.com/news/Etnaviv-iMX8MP-Support)

### NXP i.MX6Q SabreLite — Classic Reference Board

The i.MX6Q SabreLite with its **GC2000** (plus GC320 2D, GC355 VG) has been an etnaviv development platform since the project's inception. It remains the canonical GLES2 test target and the platform where van der Laan first reverse-engineered the Vivante command stream. Many long-time etnaviv developers still keep SabreLite boards for GLES2 regression testing.

### Device Tree Configuration

The device tree binding for etnaviv GPU cores uses `compatible = "vivante,gc"`. The hardware self-identifies through chip ID registers at fixed offsets, so a generic compatible string suffices for all GC variants. A typical i.MX8MM device tree node for the 3D GPU:

```dts
/* arch/arm64/boot/dts/freescale/imx8mm.dtsi */
gpu_3d: gpu@38000000 {
    compatible = "vivante,gc";
    reg = <0x0 0x38000000 0x0 0x8000>;
    interrupts = <GIC_SPI 3 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clk IMX8MM_CLK_GPU_AHB>,
             <&clk IMX8MM_CLK_GPU_BUS_ROOT>,
             <&clk IMX8MM_CLK_GPU3D_ROOT>;
    clock-names = "reg", "bus", "core";
    power-domains = <&pgc_gpu>;
};
```

[Source: Linux kernel — imx8mm GPU RFC patch](https://lkml.indiana.edu/hypermail/linux/kernel/2004.3/08070.html)

The i.MX8M Plus uses a similar node at `0x38000000` with the GC7000UL core. The `"vivante,gc"` compatible is the only compatible string required across all Vivante GC generations because the driver reads the GPU model and revision from hardware registers at probe time (`VIVS_HI_CHIP_MODEL` and `VIVS_HI_CHIP_REV`).

### Yocto Integration

For production i.MX8 builds using the Yocto Project, the meta-freescale layer provides the necessary recipes. A minimal image configuration adding etnaviv Mesa support:

```bash
# In local.conf or an image recipe
IMAGE_INSTALL:append = " mesa mesa-demos"

# Required kernel config
CONFIG_DRM_ETNAVIV=y
CONFIG_DRM_ETNAVIV_THERMAL=y

# libdrm with etnaviv support
PACKAGECONFIG:append:pn-libdrm = " etnaviv"
```

The Yocto `mesa` recipe defaults to including the etnaviv Gallium driver when building for i.MX targets. The GPU tools (register dumper, performance counter viewer) can be added via `etnaviv-gpu-tools` if that package exists in the BSP layer, though the official tools from the etnaviv project are not widely packaged.

---

## 10. Debugging and Contributing

### Debug Environment Variables

The etnaviv Mesa driver respects the `ETNA_MESA_DEBUG` environment variable (set in `etnaviv_screen.c`), which enables various debug output modes. Known flags include:

```bash
# Enable debug message output for GPU state changes
ETNA_MESA_DEBUG=msgs

# Dump command buffer contents as hex before submission
ETNA_MESA_DEBUG=dump

# Print compiler IR and generated ISA for each shader
ETNA_MESA_DEBUG=compiler

# Combine multiple flags
ETNA_MESA_DEBUG=msgs,compiler glxgears
```

Note that available flags change between Mesa releases; check `etnaviv_screen.c` in the Mesa source for the current list.

For system-level Gallium debugging, the standard `GALLIUM_HUD` overlay works with etnaviv:

```bash
# Display FPS and GPU load percentage as an on-screen overlay
GALLIUM_HUD=fps,GPU-load eglgears
```

The `GPU-load` query uses the etnaviv performance monitor counters (`etnaviv_perfmon.c`), which read hardware performance counter registers via `DRM_IOCTL_ETNAVIV_PM_QUERY_DOM` and `DRM_IOCTL_ETNAVIV_PM_QUERY_SIG`.

### ISA Disassembly

The etna_viv repository contains a disassembler for Vivante ISA that can decode shader binaries back to human-readable assembly. This is useful when debugging GPU hangs caused by incorrect shader code generation:

```bash
# Clone the tools repository
git clone https://github.com/etnaviv/etna_viv
cd etna_viv

# Disassemble a shader binary captured via ETNA_MESA_DEBUG=dump
python3 tools/disasm.py shader.bin
```

For kernel-level debugging, the etnaviv driver produces a GPU state dump on hang via `etnaviv_dump.c`. The dump captures all GPU registers and the contents of the current command buffer, which can be decoded using the rnndb register database.

### Kernel Driver Debugging

```bash
# Enable etnaviv kernel driver debug output via dynamic debug
echo "module etnaviv +p" > /sys/kernel/debug/dynamic_debug/control

# View GPU hang dumps
dmesg | grep etnaviv

# Inspect GPU state via debugfs
ls /sys/kernel/debug/dri/
cat /sys/kernel/debug/dri/0/etnaviv-gpu:38000000/info
```

### Testing Conformance

```bash
# Run the OpenGL ES 2.0 test suite against etnaviv
export EGL_PLATFORM=gbm
export MESA_LOADER_DRIVER_OVERRIDE=etnaviv
deqp-runner run --deqp deqp-gles2 \
    --caselist gles2-cases.txt \
    --baseline gles2-expected.txt

# Quick render sanity check
glxgears -display :0     # X11
eglgears               # Wayland/GBM
```

### Contributing

The etnaviv project accepts contributions through two channels:

**Kernel driver** (changes to `drivers/gpu/drm/etnaviv/`):
- Submit patches via email to the **dri-devel mailing list**: [dri-devel@lists.freedesktop.org](mailto:dri-devel@lists.freedesktop.org)
- CC: Lucas Stach ([l.stach@pengutronix.de](mailto:l.stach@pengutronix.de)) as the primary maintainer
- CC: Christian Gmeiner ([christian.gmeiner@gmail.com](mailto:christian.gmeiner@gmail.com))
- Follow the standard kernel patch format (`git format-patch`, `checkpatch.pl`)

**Mesa driver** (changes to `src/gallium/drivers/etnaviv/`):
- Submit merge requests at [gitlab.freedesktop.org/mesa/mesa](https://gitlab.freedesktop.org/mesa/mesa)
- CI must pass; the etnaviv LAVA CI board runs automatically on merge requests
- Tag `Christian Gmeiner` or `Lucas Stach` as reviewer for etnaviv-specific changes

**Hardware access**: Developers without access to i.MX hardware can use the **drm-shim** approach — the etnaviv kernel driver can be shimmed in Mesa's CI infrastructure to run the compiler without actual hardware. For full render testing, the recommended entry-level boards are the **NXP i.MX6Q SabreLite** (GC2000, widely available used) or the **NXP i.MX8M Plus EVK** (GC7000UL, the current primary development target).

The etnaviv GitHub organisation at [github.com/etnaviv](https://github.com/etnaviv) hosts the etna_viv tools repository and serves as the project's public face for historical documentation and tooling.

---

## 11. Integrations

This chapter connects to several other parts of the book:

**Chapter 1 — DRM Architecture**: etnaviv implements the full DRM stack: GEM buffer management (`etnaviv_gem.c`, shmem-backed), the DRM GPU scheduler (`etnaviv_sched.c`, via `drm_gpu_scheduler`), DRM-managed IOMMU contexts, sync_file/fence integration, and DRM PRIME for DMA-BUF import/export. The etnaviv kernel driver is a textbook example of a clean DRM driver built entirely from upstream DRM helpers.

**Chapter 5 — Mesa and Gallium3D**: etnaviv is a first-class Gallium3D driver. It implements `pipe_screen`, `pipe_context`, `pipe_resource`, `pipe_blend_state`, and the full draw pipeline via `etnaviv_draw_vbo()`. Its texture management (tiled/supertiled layout, RS-based resolve) illustrates the Gallium resource model in an embedded context. The `etnaviv_drm_screen_create()` function is the canonical entry point from the Mesa loader.

**Chapter 14 — NIR: The Common Shader IR**: The etnaviv compiler in `etnaviv_compiler_nir.c` is a good study of a real NIR consumer for a non-trivial ISA. It demonstrates the standard NIR lowering sequence (I/O lowering, texture lowering, ALU scalarisation), instruction selection from NIR ALU opcodes to a vendor ISA, and binary emission. The transition from TGSI to NIR (Mesa 19.2 → 21.3) and the removal of the TGSI backend in Mesa 22.x document how the Mesa community manages compiler infrastructure transitions.

**Chapter 90 — Open ARM GPU Drivers (Panfrost/Panthor/Lima)**: The reverse-engineering methodology for etnaviv is directly comparable to Lima and Panfrost. Key similarities: no public ISA documentation, command-stream interception for register tracing, rnndb-style register XML databases, NIR-based Mesa compiler. Key differences: etnaviv's unified shader ISA (one compiler for VS and FS) versus Lima's separate GP/PP compilers; etnaviv's absence of a firmware layer (unlike Panthor's mandatory CSF firmware); etnaviv's tile-based TBDR rendering via the RS resolve module versus Panfrost's per-tile fragment shading model. The Librem 5 — cited in both chapters — is a point of comparison: it uses etnaviv for 3D (Vivante GC7000Lite) alongside the same Phosh/Wayland stack discussed in the embedded compositor chapters.

**Chapter 99 — Automotive and Embedded Graphics**: NXP i.MX6 and i.MX8 are dominant SoC families in automotive instrument clusters and human-machine interfaces. etnaviv is the production open-source graphics driver for these platforms. The i.MX8M Plus (GC7000UL) is used in ADAS coprocessor boards, gateway ECUs, and infotainment systems. The discussion of Yocto integration and real-world board bring-up in this chapter complements the automotive Linux graphics stack discussion there.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
