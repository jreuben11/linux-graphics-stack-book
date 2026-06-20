# Chapter 160: Freedreno and Turnip: Open-Source Qualcomm Adreno Driver

**Target audiences**: Developers building Linux-on-ARM systems with Qualcomm SoCs (Snapdragon X Elite, RB5, Dragonboard); driver developers studying Turnip as the most capable open-source mobile Vulkan driver; and embedded engineers running Wayland compositors on Adreno hardware.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Qualcomm Adreno GPU Architecture](#qualcomm-adreno-gpu-architecture)
3. [Adreno TBDR: Binning, VSC, and LRZ](#adreno-tbdr-binning-vsc-and-lrz)
4. [GMU: Graphics Management Unit](#gmu-graphics-management-unit)
5. [Freedreno: The OpenGL Gallium Driver](#freedreno-the-opengl-gallium-driver)
6. [Turnip: The Vulkan Driver](#turnip-the-vulkan-driver)
7. [UBWC and AFBC: Bandwidth Compression on Adreno](#ubwc-and-afbc-bandwidth-compression-on-adreno)
8. [Adreno 7xx and 8xx: Newer Generations](#adreno-7xx-and-8xx-newer-generations)
9. [Freedreno vs. Turnip vs. Zink](#freedreno-vs-turnip-vs-zink)
10. [Compiler: ir3 and the Adreno ISA](#compiler-ir3-and-the-adreno-isa)
11. [Kernel Driver: msm_drm](#kernel-driver-msm_drm)
12. [GPU Memory: IOMMU and SMMU](#gpu-memory-iommu-and-smmu)
13. [Performance Counters and Debugging Tools](#performance-counters-and-debugging-tools)
14. [Upstream Development Workflow](#upstream-development-workflow)
15. [Qualcomm Snapdragon X Elite on Linux](#qualcomm-snapdragon-x-elite-on-linux)
16. [Roadmap](#roadmap)
17. [Integrations](#integrations)

---

## Introduction

Qualcomm Adreno GPUs power the dominant Android platform (Snapdragon SoCs) and, with the Snapdragon X series, are entering the Linux laptop market. The open-source Adreno driver stack consists of:

- **Freedreno** — Mesa Gallium3D driver (OpenGL, OpenGL ES)
- **Turnip** — Mesa Vulkan driver, the highest-performing open-source mobile Vulkan driver
- **ir3** — the shared shader compiler (NIR → Adreno IR → a3xx/a5xx/a6xx/a7xx ISA)
- **msm_drm** — Linux kernel DRM driver for Qualcomm display and GPU
- **Zink** — OpenGL over Vulkan; when paired with Turnip, provides a competitive full-desktop OpenGL path

Turnip is particularly impressive: it supports Vulkan 1.3, ray tracing on A7xx (Adreno 700 series), mesh shaders, and competitive Vulkan performance versus the proprietary Qualcomm driver on Android. It is used in production on postmarketOS, Arch Linux ARM, and Qualcomm's own reference Linux images.

This chapter covers the complete open-source Adreno stack in depth: the Tile-Based Deferred Rendering (TBDR) architecture with its binning pass, the Graphics Management Unit (GMU) power firmware, bandwidth compression via UBWC, newer A7xx/A8xx generation support, the choice between Freedreno/Turnip/Zink, and the tooling for profiling and debugging Adreno hardware on Linux.

Sources: [freedreno/turnip Mesa source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/freedreno) | [msm_drm kernel](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/msm)

---

## Qualcomm Adreno GPU Architecture

### GPU Families

| Series | SoC Examples | Architecture Notes | Turnip Vulkan |
|---|---|---|---|
| A3xx | SD 800/801 | Scalar ISA, legacy | Not supported |
| A5xx | SD 820/835 | a5xx ISA, 128-wide waves | 1.0 (experimental) |
| A6xx | SD 845–888 | UCHE L2, GMEM 1 MB, LRZ v1 | 1.3 (full support) |
| A7xx | SD 8 Gen 2/3, X Elite | 2 MB GMEM, concurrent binning, wave64, RT | 1.3 + RT + mesh |
| A8xx | SD 8 Elite, X2 Elite | Gen 8 architecture, VRS | 1.3 + VRS (Mesa 26+) |

### Unified Compute Architecture

Adreno A6xx+ uses a SIMD architecture of 128 threads per wave on A6xx (wave128) and 64 threads per lane across a 2048-bit vector on A7xx (wave64). Unlike Mali's quad-based Bifrost, Adreno is warp-based, similar in spirit to NVIDIA's SIMT model.

Key A6xx hardware blocks:
- **SP (Shader Processor)**: runs vertex/fragment/compute shaders; four SPs on A650
- **UCHE (Unified Cache Hierarchy)**: 512 KB L2 cache shared between SP and TP; sits behind vertex fetch, VSC writes, texture L1, LRZ, and storage image accesses
- **TP (Texture Processor)**: texture filtering and L1 texture cache (16 KB per SP)
- **RB (Render Backend)**: render output unit, depth/stencil logic, CCU
- **CP (Command Processor)**: parses type-4 and type-7 command stream packets from the ring buffer
- **VSC (Visibility Stream Compressor)**: hardware unit that records per-tile primitive visibility during the binning pass

The A7xx generation (Adreno 730 on Snapdragon 8 Gen 2, Adreno 740/750 on Snapdragon 8 Gen 3) doubles the GMEM to 2 MB, doubles the VSC pipe count to 32, and introduces a split command processor (BR/BV) enabling concurrent binning. [Source: Chips and Cheese A7xx analysis](https://chipsandcheese.com/p/inside-snapdragon-8-gen-1s-igpu-adreno-gets-big)

### Tile-Based Rendering (GMEM)

Adreno uses GMEM (on-chip Graphics Memory) for tile rendering, avoiding expensive DRAM round-trips for intermediate render results:

```
Render pass start:
  1. Binning pass (vertex-only): VSC records tile occupancy → visibility stream
  2. LRZ pass: records worst-case (farthest) Z per LRZ block in low-res Z buffer
Per tile:
  3. Load: gmem_load copies system-memory attachment data → GMEM (if loadOp=LOAD)
  4. Render: all fragment shading reads/writes GMEM (fast on-chip SRAM)
  5. Store: gmem_store copies GMEM → system-memory (if storeOp=STORE)
```

This is Qualcomm's version of TBDR. Turnip and Freedreno are both architected around this model: the render pass abstraction maps directly onto the GMEM/binning lifecycle. [Source: TurnIP tiled rendering deep-dive](https://deepwiki.com/sailfishos-mirror/mesa/3.3.1-turnip-vulkan-driver-and-tiled-rendering)

---

## Adreno TBDR: Binning, VSC, and LRZ

### The Binning Pass

Before fragment shading begins, Adreno executes a vertex-only binning pass. The CP emits a lightweight draw call that runs only the vertex shader (no fragment stage). The output of each vertex is tested against each tile's bounding box; the VSC hardware records a bitmask (the visibility stream) of which primitives fall within each tile.

The visibility stream tells the GPU, for each tile: "only draw primitives 3, 7, 12, ..." — avoiding redundant fragment processing. This is Adreno's answer to PowerVR's HSR (Hidden Surface Removal) and ARM Mali's forward pixel kill: the binning pass eliminates invisible geometry at the per-tile granularity.

Turnip implements the binning pass in `tu_cmd_buffer.c` and `tu_autotune.c`. The VSC buffers that hold the visibility stream must be allocated per-command-buffer with conservative sizing; if the VSC overflows its buffer, the driver doubles the pipe pitch for subsequent command buffers:

```c
/* src/freedreno/vulkan/tu_cmd_buffer.c */
static void
tu6_emit_binning_pass(struct tu_cmd_buffer *cmd, struct tu_cs *cs)
{
   const struct tu_framebuffer *fb = cmd->state.framebuffer;

   /* Configure VSC pipe count (16 on A6xx, 32 on A7xx): */
   tu6_emit_vsc(cmd, cs);

   /* Emit vertex-only draw for the binning pass: */
   tu_cs_emit_pkt7(cs, CP_SET_VISIBILITY_OVERRIDE, 1);
   tu_cs_emit(cs, 1);  /* override: all visible for binning */

   tu_cs_emit_pkt7(cs, CP_SET_MODE, 1);
   tu_cs_emit(cs, 1);  /* binning mode */

   /* Re-emit draw calls; fragment stage disabled: */
   tu6_draw_common(cmd, cs, false, 0);
}
```

[Source: Mesa freedreno tu_cmd_buffer.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/freedreno/vulkan/tu_cmd_buffer.c)

### VSC: Visibility Stream Compressor

The VSC is the hardware unit that performs the binning. It maintains 16 (A6xx) or 32 (A7xx) "pipes", each covering a range of tiles. During the binning pass, as each triangle is rasterised at low resolution, the VSC writes into its pipe data buffers:

- **Prim buffer**: a per-tile bitmask of which primitives are visible
- **Size buffer**: primitive counts used to validate buffer bounds

The driver configures the VSC by writing base addresses for each pipe into `VSC_PIPE_DATA_PRIM_BASE` (A6xx) or via `CP_SET_PSEUDO_REG` (A7xx). During the render pass, the CP reads the visibility stream and skips draw calls that have no visible primitives in the current tile.

### LRZ: Low Resolution Z

LRZ (Low Resolution Z) is Adreno's hierarchical Z-buffer. It is a separate, lower-resolution depth buffer stored at 1/8th the dimensions of the full-res depth buffer (one LRZ block covers an 8×8 pixel region). Each LRZ block stores the farthest (worst-case) Z value in that region.

LRZ works in two phases:

**LRZ write (binning pass)**: The vertex shader outputs Z values. The binning pass records, for each LRZ block, the farthest Z among all rasterised vertices. This builds a conservative max-Z map.

**LRZ read (render pass)**: As fragments arrive, the hardware tests each fragment's Z against the LRZ block's max-Z value. If `z_frag > lrz_max` for a back-to-front operation, the fragment is trivially rejected before even running the fragment shader. This is equivalent to a zero-overhead depth prepass.

The performance benefit is substantial: fragments rejected by LRZ never reach the SP, saving both ALU cycles and memory bandwidth. Igalia benchmarks showed 13–16% performance improvement from enabling LRZ on certain workloads. [Source: Igalia LRZ blog post](https://blogs.igalia.com/siglesias/2021/04/19/low-resolution-z-buffer-support-on-turnip/)

LRZ is disabled in several conditions where it cannot safely predict visibility:

- Fragment shaders that write to SSBOs or perform atomic operations (side effects preclude early kill)
- `discard` (demote) instructions in the fragment shader
- Depth comparison direction switching (GREATER vs LESS) — invalidates the min/max interpretation
- Blending enabled with depth writes
- Depth bounds testing with a non-trivial range

Turnip tracks LRZ state per draw call:

```c
/* src/freedreno/vulkan/tu_lrz.c */
enum tu_lrz_force_disable_mask {
   TU_LRZ_FORCE_DISABLE_LRZ    = BIT(0),  /* LRZ itself */
   TU_LRZ_FORCE_DISABLE_WRITE  = BIT(1),  /* LRZ writes only */
};

static void
tu6_emit_lrz(struct tu_cmd_buffer *cmd, struct tu_cs *cs)
{
   const struct tu_lrz_state *lrz = &cmd->state.lrz;

   tu_cs_emit_regs(cs,
      A6XX_GRAS_LRZ_CNTL(
         .enable       = lrz->enabled,
         .lrz_write    = lrz->write,
         .dir_write    = lrz->dir_write,
         .z_test_enable = lrz->z_test_enable,
      ));
}
```

A650+ introduces **LRZ fast clear** (`enable_lrz_fast_clear`): the LRZ buffer is cleared by the GPU in a single pass without per-pixel work, further reducing render pass setup overhead.

A7xx enables **early-LRZ-late-depth** mode: even when a fragment shader contains `discard`, LRZ can reject fragments that fail the coarse Z test, with the full per-sample depth test applied later. This was unavailable on A6xx and accounts for part of the A7xx performance uplift in depth-heavy scenes.

### CCU: Color Cache Unit

The CCU (Color Cache Unit) is the render backend's L1 cache for color and depth attachments. In GMEM mode, the CCU carves out a portion of GMEM for its own cache lines; in SYSMEM (bypass) mode it uses a different offset. Transitions between GMEM and SYSMEM modes require explicit CCU flush/invalidate sequences:

```c
/* Flush CCU before switching modes: */
tu_cs_emit_pkt7(cs, CP_EVENT_WRITE, 1);
tu_cs_emit(cs, PC_CCU_FLUSH_COLOR_TS);
tu_cs_emit_pkt7(cs, CP_EVENT_WRITE, 1);
tu_cs_emit(cs, PC_CCU_FLUSH_DEPTH_TS);
tu_cs_emit_pkt7(cs, CP_WAIT_FOR_IDLE, 0);
```

[Source: Mesa tu_cmd_buffer.c CCU handling](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/freedreno/vulkan/tu_cmd_buffer.c)

---

## GMU: Graphics Management Unit

### What is the GMU?

Present on Adreno A6xx and later, the Graphics Management Unit (GMU) is a dedicated embedded microcontroller (ARM Cortex-M3 class) that sits alongside the GPU core. It runs a separate firmware blob and takes responsibility for GPU power and clock management, offloading those duties from the host CPU.

Before the GMU, the CPU had to spin-wait for GPU power domain transitions — expensive in both latency and energy. The GMU enables asynchronous power management: the CPU can issue a "set frequency" request and return immediately, while the GMU negotiates the voltage/clock change with the PMIC and system power controller.

Key GMU firmware files loaded by the kernel at runtime:
- `a660_gmu.bin` — GMU firmware for Adreno A660 (Snapdragon 888)
- `a690_gmu.bin` — Adreno 690 (Snapdragon 8cx Gen 3)
- `a730_gmu.bin` — Adreno 730 (Snapdragon 8 Gen 2)

[Source: linux/drivers/gpu/drm/msm/adreno/a6xx_gmu.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/msm/adreno/a6xx_gmu.c)

### GMU Power States

The GMU manages three primary idle states for the GPU:

| State | Description | GX Rail |
|---|---|---|
| ACTIVE | Normal GPU operation | On |
| IFPC | Integrated Frame Power Collapse | Off (GPU powered down) |
| SLUMBER | Deep idle; GMU itself may sleep | Off |

State transitions follow a strict sequence. When the GPU becomes idle, the driver requests IFPC via an OOB (out-of-band) notification; the GMU gates the GPU clock, collapses the GX (GPU execution) power domain, and acknowledges. On the next GPU work submission, the CPU sets an "OOB set" bit and waits for the GMU to wake the GPU before the ring buffer write proceeds:

```c
/* drivers/gpu/drm/msm/adreno/a6xx_gmu.c */
int a6xx_gmu_set_oob(struct a6xx_gmu *gmu, enum a6xx_gmu_oob_state state)
{
    int ret;
    u32 val;
    int request, ack;

    request = gmu->legacy ? state : (28 - state * 2);
    ack     = gmu->legacy ? (state + 16) : (31 - state * 2);

    gmu_write(gmu, REG_A6XX_GMU_HOST2GMU_INTR_SET, BIT(request));

    ret = gmu_poll_timeout(gmu,
        REG_A6XX_GMU_GMU2HOST_INTR_INFO, val,
        val & BIT(ack), 10, 10000);
    if (ret)
        dev_err(gmu->dev, "OOB set timeout for %s\n",
                a6xx_gmu_oob_names[state]);

    gmu_write(gmu, REG_A6XX_GMU_GMU2HOST_INTR_CLR, BIT(ack));
    return ret;
}
```

Before entering slumber, the driver sends `HFI_H2F_MSG_PREPARE_SLUMBER` to give the GMU firmware time to drain pending power transactions:

```c
static int a6xx_gmu_notify_slumber(struct a6xx_gmu *gmu)
{
    struct a6xx_hfi_msg_prep_slumber msg = {
        .bw   = 0,
        .freq = 0,
    };
    return a6xx_hfi_send_prepared_slumber(gmu, &msg);
}
```

### HFI: Host Firmware Interface

The CPU communicates with the GMU firmware through the HFI (Host Firmware Interface), a pair of shared-memory circular queues: one host-to-firmware (H2F) and one firmware-to-host (F2H). The queue structures are mapped at a physical address known to both parties:

```c
/* drivers/gpu/drm/msm/adreno/a6xx_hfi.c */
struct a6xx_hfi_queue_header {
    u32 read_index;
    u32 write_index;
    u32 size;         /* in u32 words */
    u32 rx_request;
    u32 dropped;
};

struct a6xx_hfi_queue {
    struct a6xx_hfi_queue_header *header;
    u32  *data;
    u32   history[HFI_HISTORY_SZ];
    u32   history_idx;
    spinlock_t lock;
    atomic_t   seqnum;
};
```

Key HFI message types exchanged at boot and runtime:

| Message ID | Direction | Purpose |
|---|---|---|
| `HFI_H2F_MSG_INIT` | H2F | GMU initialization |
| `HFI_H2F_MSG_PERF_TABLE` | H2F | Upload GPU frequency/voltage table |
| `HFI_H2F_MSG_BW_TABLE` | H2F | Upload memory bandwidth table |
| `HFI_H2F_MSG_GX_BW_PERF_VOTE` | H2F | Request clock/bandwidth level change |
| `HFI_H2F_MSG_PREPARE_SLUMBER` | H2F | Notify GMU before power collapse |

The initialization sequence (`a6xx_hfi_start`) uploads the performance and bandwidth tables, enables IFPC (Integrated Frame Power Collapse) and ACD (Adaptive Clock Distribution), then starts the GMU core execution. Once started, the GMU handles all runtime frequency scaling without further CPU intervention — the CPU simply votes for a performance level and the GMU executes the transition asynchronously.

```c
/* a6xx_hfi_start: upload tables and start GMU */
int a6xx_hfi_start(struct a6xx_gmu *gmu, int boot_state)
{
    int ret;

    ret = a6xx_hfi_send_perf_table(gmu);
    if (!ret) ret = a6xx_hfi_send_bw_table(gmu);
    if (!ret) ret = a6xx_hfi_send_acd(gmu, gmu->acd_enabled);
    if (!ret) ret = a6xx_hfi_send_ifpc(gmu, true);
    if (!ret) ret = a6xx_hfi_send_start(gmu);
    return ret;
}
```

[Source: linux a6xx_hfi.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/msm/adreno/a6xx_hfi.c)

### GMU Firmware Loading

The GMU firmware is loaded by `a6xx_gmu_fw_load()` at GPU power-on. Modern firmware uses a block-based layout: ITCM (instruction TCM), DTCM (data TCM), and optionally instruction/data cache. The loader places each block at the appropriate base address, which differs by generation (A6xx uses `0x10004000` for DTCM on newer variants). After writing the firmware, `a6xx_gmu_fw_start()` releases the Cortex-M3 reset vector and the GMU firmware begins executing its HFI message handler loop.

The two interrupt lines wired to the GMU are:
- **`a6xx_gmu_irq`**: handles GMU watchdog expiration and AHB bus errors (indicates firmware crash)
- **`a6xx_hfi_irq`**: raised by GMU firmware to signal that an F2H queue message is ready (e.g., an ACK for a previous H2F request)

---

## Freedreno: The OpenGL Gallium Driver

### Source Structure

```
mesa/src/gallium/drivers/freedreno/
├── freedreno_context.c  # pipe_context implementation
├── freedreno_resource.c # BO allocation, tiling, UBWC
├── freedreno_screen.c   # pipe_screen, capabilities
├── a6xx/
│   ├── fd6_draw.c       # draw call emission for A6xx
│   ├── fd6_gmem.c       # GMEM (tile) management
│   ├── fd6_texture.c    # texture descriptors
│   └── fd6_zsa.c        # depth/stencil state
```

### GMEM Binning Pass

Freedreno implements the Adreno binning (tiling) pass:

```c
/* freedreno/a6xx/fd6_gmem.c: emit binning pass */
static void fd6_emit_tile_init(struct fd_batch *batch)
{
    /* Emit VS-only binning pass: */
    fd6_vsc_update_sizes(batch);
    OUT_PKT4(ring, REG_A6XX_VPC_SO_OVERRIDE, 1);
    OUT_RING(ring, A6XX_VPC_SO_OVERRIDE_SO_DISABLE);
    /* Emit binning draw: */
    fd6_draw_emit(batch, ring, DI_PT_POINTLIST_PSIZE,
        IGNORE_VISIBILITY, 0, batch->draw_count);
}
```

The binning pass is a vertex-only render that records which tiles each primitive touches, enabling per-tile fragment processing. [Source: Mesa fd6_gmem.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/gallium/drivers/freedreno/a6xx/fd6_gmem.c)

### Autotune: GMEM vs SYSMEM Selection

Not every render pass fits efficiently within GMEM. For very large framebuffers (e.g., a 4K surface with 4 MSAA samples), the tile size becomes too small to amortize the binning overhead. Freedreno and Turnip both implement an "autotune" algorithm that profiles recent render passes and decides whether GMEM or SYSMEM (direct DRAM access, bypassing tiling) yields better performance.

The `TU_AUTOTUNE_ALGO` environment variable selects the algorithm:
- `bandwidth` (default): estimates SYSMEM vs GMEM memory bandwidth
- `profiled`: uses GPU timestamp queries to measure actual pass duration
- `prefer_gmem` / `prefer_sysmem`: force one mode (useful for debugging regressions)

---

## Turnip: The Vulkan Driver

### Architecture Overview

```
mesa/src/freedreno/vulkan/
├── tu_device.c          # VkDevice, VkPhysicalDevice
├── tu_cmd_buffer.c      # VkCommandBuffer recording
├── tu_pipeline.c        # VkPipeline compilation
├── tu_renderpass.c      # VkRenderPass, GMEM load/store
├── tu_image.c           # VkImage, tiling, UBWC
├── tu_descriptor_set.c  # VkDescriptorSet
├── tu_shader.c          # shader compilation (NIR → ir3 binary)
├── tu_lrz.c             # LRZ state tracking and emission
└── tu_autotune.c        # GMEM vs SYSMEM decision engine
```

### Vulkan Version Support

Turnip achieves near-complete Vulkan 1.3 on A6xx+ and is the most capable open-source mobile Vulkan driver. On A7xx it additionally exposes ray tracing and mesh shader extensions:

```bash
# Check Turnip Vulkan support:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json \
    vulkaninfo --summary
# Output: Vulkan 1.3 on Adreno (TM) 740 ...
```

```c
/* tu_physical_device.c: capability flags */
VKFEATURE(vk12, bufferDeviceAddress);
VKFEATURE(vk12, descriptorIndexing);
VKFEATURE(vk12, timelineSemaphore);
VKFEATURE(vk13, dynamicRendering);
VKFEATURE(vk13, synchronization2);
VKFEATURE(vk13, maintenance4);
```

[Source: Mesa tu_physical_device.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/freedreno/vulkan/tu_physical_device.c)

### GMEM-Aware Render Pass

Turnip's render pass implementation is tightly coupled to GMEM. A Vulkan render pass with `loadOp=DONT_CARE` and `storeOp=DONT_CARE` avoids all DRAM traffic — the entire render executes on-chip:

```c
/* tu_renderpass.c: decide GMEM vs SYSMEM */
static enum tu_gmem_layout tu_pick_gmem_layout(
    const struct tu_render_pass *pass,
    const struct tu_framebuffer *fb)
{
    uint32_t gmem_bytes_needed = 0;
    for (uint32_t i = 0; i < pass->attachment_count; i++) {
        gmem_bytes_needed +=
            pass->attachments[i].cpp * fb->width * fb->height / tile_count;
    }
    if (gmem_bytes_needed > phys_dev->gmem_size)
        return TU_GMEM_LAYOUT_BYPASS;  /* too big: fall back to SYSMEM */
    return TU_GMEM_LAYOUT_FULL_COLOR;
}
```

SYSMEM mode (bypass GMEM) is used when attachments are too large for GMEM. Performance degrades significantly — prefer keeping render target sizes within GMEM capacity. For reference, A6xx has 1 MB GMEM and A7xx has 2 MB; a 1080p RGBA8 color + D24S8 depth framebuffer fits in A6xx GMEM, but 4K with MSAA does not.

### Command Stream Packets

Adreno's command processor uses type-4 and type-7 packets:

```c
/* freedreno/common/freedreno_pm4.h */
#define CP_TYPE4_PKT  0x40000000
#define CP_TYPE7_PKT  0x70000000

/* Emit a register write (type-4 packet): */
#define OUT_PKT4(ring, reg, cnt) \
    OUT_RING(ring, CP_TYPE4_PKT | ((cnt) << 0) | \
             (((reg) >> 2) << 8))

/* Draw call (type-7 packet): */
#define OUT_PKT7(ring, opcode, cnt) \
    OUT_RING(ring, CP_TYPE7_PKT | ((cnt) << 0) | ((opcode) << 8))
```

[Source: Mesa freedreno_pm4.h](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/freedreno/common/freedreno_pm4.h)

### VK_EXT_physical_device_drm

Turnip exposes `VK_EXT_physical_device_drm`, which allows applications and compositors to correlate a Vulkan physical device with a DRM device file (e.g., `/dev/dri/renderD128`). This is essential on multi-GPU systems (discrete + integrated) and on Snapdragon platforms where the display DRM device and the GPU DRM device may have different minor numbers.

The extension reports the DRM render and primary device node minor numbers:

```c
/* Query DRM device node minor for Turnip physical device: */
VkPhysicalDeviceDrmPropertiesEXT drm_props = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DRM_PROPERTIES_EXT,
};
VkPhysicalDeviceProperties2 props2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
    .pNext = &drm_props,
};
vkGetPhysicalDeviceProperties2(phys_dev, &props2);
/* drm_props.renderMinor == 128 → /dev/dri/renderD128 */
```

A compositor using this to select the correct GPU for rendering (e.g., to match the display's DRM primary node) can compare `drm_props.renderMinor` against the render node it opened for KMS. Without this extension, the compositor has to probe device node paths heuristically. On Snapdragon X Elite, where the DPU's DRM primary node and the GPU's render node share the same MSM DRM driver instance, the minor numbers are typically the same; on systems with a discrete GPU (e.g., a Thunderbolt eGPU), this distinction matters for zero-copy buffer sharing. [Source: VK_EXT_physical_device_drm spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_physical_device_drm.html)

---

## UBWC and AFBC: Bandwidth Compression on Adreno

### UBWC: Qualcomm's Proprietary Compression

UBWC (Universal Bandwidth Compression) is Qualcomm's lossless render target compression format, available on Adreno A5xx and later. It reduces memory bandwidth by compressing render target tiles: each 256-byte tile has a 4-byte metadata word in a separate flag buffer indicating whether the tile data is compressed. On a cache miss, the hardware decompresses on the fly.

UBWC is applied to color attachments, depth attachments, and scan-out framebuffers. The Mesa Freedreno driver added UBWC support for A6xx in Mesa 19.1 via approximately 200 lines of code in `freedreno_resource.c` — wiring up compressed BO allocation and the DRM scan-out modifier:

```c
/* freedreno/freedreno_resource.c */
static bool
fd_resource_set_usage(struct pipe_screen *pscreen,
                      struct pipe_resource *prsc,
                      enum pipe_resource_usage usage)
{
    struct fd_resource *rsc = fd_resource(prsc);
    /* Enable UBWC for render targets on A5xx+: */
    if (screen->info->a6xx.has_ubwc &&
        (prsc->bind & PIPE_BIND_RENDER_TARGET)) {
        rsc->layout.ubwc = true;
    }
    return true;
}
```

[Source: Phoronix UBWC for Freedreno](https://www.phoronix.com/news/Freedreno-UBWC-A6XX) | [Mesa MR: freedreno UBWC scanout](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/20892)

UBWC is distinct from AFBC: UBWC is Qualcomm-specific and uses a different block layout and metadata scheme; AFBC is ARM's format used on Mali GPUs. On Adreno SoCs, UBWC is the primary bandwidth compression format for GPU-produced render targets; Adreno GPU hardware does not natively produce AFBC-encoded render targets. However, Adreno display controllers (DPU) on certain SoCs can scan out AFBC-encoded framebuffers produced by a co-processor such as the ISP or a codec engine — in that case the AFBC modifier is carried in the DMA-BUF and negotiated through KMS.

For Turnip, UBWC support on Vulkan images is controlled via `TU_DEBUG=noubwc` to disable it for debugging:

```bash
# Disable UBWC (for debugging compression regressions):
TU_DEBUG=noubwc vulkan_app

# Also disable LRZ for comparison:
TU_DEBUG=nolrz vulkan_app
```

### UBWC Memory Layout

UBWC stores image data in two planes: the primary colour plane (laid out in vendor-defined macro-tiles) and a metadata plane holding one 4-byte status word per 256-byte block. Status values indicate:

- `0x00` — raw (uncompressed block)
- `0x01`–`0xFF` — compressed size in 64-byte units (a fully compressed block occupies 64 bytes)

The metadata plane base address is passed separately to the hardware. The Turnip image allocation code in `tu_image.c` aligns and sizes both planes and stores their offsets in `tu_image.layout`:

```c
/* src/freedreno/vulkan/tu_image.c */
static void
tu_image_layout_init(struct tu_image *image)
{
    struct fdl_layout *layout = &image->layout[0];

    /* UBWC metadata plane follows immediately after the colour plane,
     * aligned to the UBWC metadata granularity (4 KB on A6xx): */
    image->ubwc_offset = ALIGN(layout->size, 4096);
    image->ubwc_size   = fdl_ubwc_size(layout);
    image->bo_size     = image->ubwc_offset + image->ubwc_size;
}
```

[Source: Mesa tu_image.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/freedreno/vulkan/tu_image.c)

### DRM Format Modifiers and Compositor Integration

Turnip exposes UBWC to compositors and importers via `VK_EXT_image_drm_format_modifier`. This extension allows a Vulkan image to be created with an explicit DRM format modifier — the modifier encodes the compression layout that producers and consumers must agree on.

For UBWC on Adreno, the relevant modifier is:
```
DRM_FORMAT_MOD_QCOM_COMPRESSED
```

defined in `include/uapi/drm/drm_fourcc.h`. A compositor (Weston, Wayfire, or wlroots-based) negotiating a zero-copy buffer share with Turnip will:

1. Query available modifiers via `VkDrmFormatModifierPropertiesEXT`
2. Include `DRM_FORMAT_MOD_QCOM_COMPRESSED` in its `zwp_linux_dmabuf_v1` format list
3. Allocate a DMA-BUF with the UBWC modifier via Turnip
4. Import the DMA-BUF into the display hardware's KMS plane (the DPU supports UBWC scan-out natively)

This gives a zero-copy path from GPU render target → KMS display plane, with UBWC active throughout — the GPU writes UBWC-compressed pixels, the display controller reads them without decompression, and the DRAM bandwidth saving (typically 20–40% for typical render targets) is preserved end-to-end.

[Source: VK_EXT_image_drm_format_modifier spec](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html) | [Mesa 24.0 release notes](https://docs.mesa3d.org/relnotes/24.0.0.html)

### AFBC on Adreno: Import, Not Export

ARM's AFBC (Arm Framebuffer Compression) uses `DRM_FORMAT_MOD_ARM_AFBC_*` modifiers defined in `include/uapi/drm/drm_fourcc.h`. Unlike UBWC, which is both produced and consumed by the Adreno GPU, AFBC on Adreno SoCs appears only in the *import* direction: video decoder output (e.g., from the Qualcomm video firmware) or images from a Mali GPU on a heterogeneous SoC may arrive as AFBC-encoded DMA-BUFs. The Adreno display controller on some SoC families can scan out these AFBC buffers directly — avoiding a decompression step — but the GPU's shader units do not write AFBC.

This is the key architectural distinction:

| Format | GPU writes? | GPU reads (texture)? | DPU scan-out? |
|---|---|---|---|
| Linear | Yes | Yes | Yes |
| UBWC | Yes (A5xx+) | Yes (A5xx+) | Yes (A5xx+) |
| AFBC | No | No (no hardware decode) | Yes (select SoCs) |

For display-only zero-copy of camera or codec output, a wlroots/Weston compositor can negotiate `DRM_FORMAT_MOD_ARM_AFBC_BLOCK_SIZE_16x16 | DRM_FORMAT_MOD_ARM_AFBC_SPARSE` through `zwp_linux_dmabuf_v1`, and the DPU will scan it out unmodified. The GPU (Turnip) is not involved in this path — it is a KMS-level concern only.

When a Vulkan application needs to *texture from* an AFBC buffer, the buffer must first be decompressed (e.g., by a blit operation or by the codec engine itself writing a linear or UBWC fallback). There is no `VK_EXT_image_drm_format_modifier` path for AFBC on Adreno that allows the GPU shader units to sample an AFBC image directly. Note: this may change with future Adreno generations; verify against the modifier list exposed by `vkGetPhysicalDeviceImageFormatProperties2` on the target device.

[Source: Linux kernel AFBC documentation](https://www.infradead.org/~mchehab/rst_conversion/gpu/afbc.html) | [drm_fourcc.h AFBC modifiers](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_fourcc.h)

---

## Adreno 7xx and 8xx: Newer Generations

### A7xx Architecture Highlights

The Adreno A7xx family (Adreno 730 on Snapdragon 8 Gen 2; Adreno 740/750 on Snapdragon 8 Gen 3; Adreno 830 on Snapdragon X Elite) introduces several architecture changes visible to the driver:

**Wave64 shaders**: A7xx increases the vector width from 128-thread waves (A6xx) to 64-wide thread groups operating on 2048-bit vectors. This doubles per-instruction throughput for compute workloads but increases sensitivity to branch divergence within a wave. The ir3 compiler adjusts register pressure estimates and instruction scheduling for the wider wave.

**Concurrent binning (BR/BV split)**: A7xx splits the Command Processor into two independent threads: BR (Binning Renderer) and BV (Binning Visual). The binning pass for frame N can now execute in parallel with the rendering pass for frame N-1. Turnip enables this with `TU_ONCHIP_BARRIER` and `TU_ONCHIP_CB_BV_TIMESTAMP` onchip memory synchronisation tokens to coordinate the BR and BV threads without a full GPU stall.

**2 MB GMEM**: Doubled from A6xx's 1 MB, allowing larger render targets to stay in GMEM without falling back to SYSMEM. A 1440p RGBA8 + D24S8 surface now fits in GMEM on A7xx.

**Tile max dimensions**: A7xx expands the maximum tile size from 1024×1008 (A6xx) to 1728×1728, improving rendering efficiency for tall/wide render targets.

**32 VSC pipes**: Double the A6xx count, accommodating larger scenes with more tiles and more primitives per tile.

### Ray Tracing on A7xx

Turnip's ray tracing journey for A7xx followed a staged rollout. The XDC 2024 talk "Ray Tracing for Adreno GPUs on Turnip" (Connor Abbott) laid out the architectural approach. [Source: XDC 2024 RT slides](https://indico.freedesktop.org/event/6/contributions/302/)

**`VK_KHR_ray_query`** — accelerated inline ray queries from any shader stage — merged into Mesa 25.0 (January 2025), authored by Connor Abbott in 25 patches. This is enabled for **Adreno 740 and newer**. Ray query allows compute and fragment shaders to trace rays inline against an acceleration structure without the full pipeline machinery. [Source: Phoronix Turnip VK_KHR_ray_query](https://www.phoronix.com/news/Mesa-TURNIP-VK_KHR_ray_query)

**`VK_KHR_acceleration_structure`** — BVH building via compute shader — is the prerequisite; it was merged alongside ray_query. Acceleration structure build and update commands are encoded as compute dispatches on the Adreno CP.

**`VK_KHR_ray_tracing_pipeline`** — full ray tracing pipeline with ray generation, any-hit, closest-hit, miss, and intersection shader stages — is a separate extension requiring deeper integration with the CP's indirect dispatch mechanism. As of Mesa 26.0, this extension is in progress; the ir3 compiler has gained new instruction encodings for the `raya` (ray acceleration) ALU instructions, but conformance submission for the full pipeline extension is still pending. Note: verify current status against Mesa HEAD before relying on this for production.

The A7xx hardware implements BVH traversal acceleration units distinct from the shader processors. These units accept a ray descriptor (origin, direction, tmin, tmax, flags) and a BVH root pointer and return the closest intersection — the shader provides only the ray parameters and reads back the result, without looping over BVH nodes in software. This is the same model as NVIDIA's RT cores, and is why `VK_KHR_ray_query` lands first: it requires fewer changes to the command stream protocol than `VK_KHR_ray_tracing_pipeline` which needs dedicated shader stage switching packets.

```c
/* Conceptual ir3 ray query emission (simplified): */
/* rayQueryInitializeEXT(rq, as, flags, mask, origin, tmin, dir, tmax) */
raya.init r0.x, c0.xyz, c3.x, c4.xyz, c7.x  /* set ray params */
raya.trav r0.x                               /* hardware BVH traversal */
/* rayQueryGetIntersectionTEXT(rq, committed) → result in r1.x */
```

### Mesh Shaders on A7xx

A7xx supports `VK_EXT_mesh_shader` (task and mesh shader stages). The ir3 compiler gained task/mesh shader support, enabling GPU-driven geometry culling pipelines — relevant for the increasing number of game engines (Nanite, etc.) that rely on mesh shaders for high-geometry rendering.

### A8xx / Adreno Gen 8 (Snapdragon X2 Elite, 8 Elite)

Turnip support for the Adreno Gen 8 GPU (A8xx, found in Snapdragon X2 Elite and Snapdragon 8 Elite) was merged into Mesa 26.0 in 2026, paired with Linux 6.19 kernel enablement in the MSM driver. [Source: Phoronix Mesa 26.0 Adreno Gen 8](https://www.phoronix.com/news/Mesa-26.0-Adreno-Gen-8-Graphics)

The Gen 8 initial support ships with Variable Rate Shading (VRS, `VK_KHR_fragment_shading_rate`) disabled while stability issues are resolved. VRS allows the fragment shader invocation rate to be reduced for background or low-priority regions, trading visual quality for bandwidth and fill-rate savings — particularly relevant on SoCs where DRAM bandwidth is the primary bottleneck. Enabling VRS on Gen 8 is an active development priority.

---

## Freedreno vs. Turnip vs. Zink

Understanding which driver to use for a given workload is important for Adreno Linux deployments.

### Freedreno (OpenGL via Gallium)

Freedreno is the Gallium3D driver implementing OpenGL ES 3.2 and desktop OpenGL 4.5 directly on Adreno hardware. It has the broadest GPU generation support — A2xx through A7xx — and is the only option for older Adreno 3xx/4xx GPUs that predate Turnip.

Use Freedreno when:
- Targeting legacy hardware (Adreno 3xx/4xx/5xx)
- Running OpenGL ES workloads on embedded systems (robotics, automotive) where Mesa is built with a minimal feature set
- Debugging GL-level rendering issues with `FD_MESA_DEBUG` flags

Known limitation: Freedreno's desktop OpenGL path has lagged behind Turnip in terms of active development investment; for GL 4.x on modern A6xx/A7xx, Zink-on-Turnip often yields better performance and correctness.

### Turnip (Native Vulkan)

Turnip is the primary driver for modern Adreno hardware. It provides:
- Full Vulkan 1.3 on A6xx+
- Ray tracing and mesh shaders on A7xx
- Best raw Vulkan performance (competitive with the proprietary Qualcomm driver)
- Correct GMEM utilisation and LRZ for optimal bandwidth

Use Turnip for:
- All Vulkan applications on A6xx+ hardware
- Game engines with Vulkan backends (Godot, Unreal Engine 5, Vulkan-native)
- The Snapdragon X Elite laptop platform

### Zink (OpenGL over Vulkan, using Turnip as backend)

Zink is the Mesa Gallium driver that translates OpenGL calls into Vulkan API calls. When Turnip is available, Zink uses Turnip as its Vulkan backend, forming a Zink→Turnip stack.

This combination delivers OpenGL 4.6 capability on any hardware where Turnip runs. The Turnip driver was selected by default for Valve's Steam Frame VR headset, with Zink-on-Turnip providing the OpenGL path needed by OpenVR/SteamVR. [Source: Mesa 2025 highlights](https://www.phoronix.com/news/Mesa-2025-Highlights)

The advantage of Zink-on-Turnip over native Freedreno for GL:
- Inherits Turnip's GMEM optimisations (LRZ, UBWC, concurrent binning)
- Kept current with Turnip's A7xx/A8xx generation support
- OpenGL 4.6 feature set, vs. 4.5 from Freedreno

Trade-off: Zink adds a translation layer overhead (GL state → Vulkan objects). For latency-sensitive applications, native Freedreno has slightly lower CPU overhead. For throughput-bound workloads, Zink-on-Turnip is generally faster.

```bash
# Force Zink (OpenGL via Turnip's Vulkan):
MESA_LOADER_DRIVER_OVERRIDE=zink \
    MESA_GL_VERSION_OVERRIDE=4.6 \
    glxgears

# Native Freedreno GL:
MESA_LOADER_DRIVER_OVERRIDE=kmsro \
    glxgears
```

[Source: Phoronix Turnip+Zink OpenGL 4.6](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/vulkan/1337059-turnip-vulkan-driver-now-works-with-zink-for-opengl-4-6-approaching-vulkan-1-3)

### Driver Selection Matrix

| Workload | Recommended Driver | Notes |
|---|---|---|
| Vulkan app, A6xx/A7xx | Turnip | Best performance |
| OpenGL app, A6xx/A7xx | Zink → Turnip | GL 4.6 via Vulkan |
| OpenGL ES app, A3xx/A4xx | Freedreno | Only option |
| OpenGL ES embedded, A5xx | Freedreno | Lower overhead |
| Wayland compositor | Turnip + EGL | UBWC zero-copy via DMA-BUF |
| Steam games (Proton) | Turnip (DXVK → Vulkan) | DXVK uses Vulkan directly |

---

## Compiler: ir3 and the Adreno ISA

### ir3 Compiler Pipeline

```
GLSL/WGSL/SPIR-V → NIR → ir3 NIR passes → ir3 IR → register allocation → a6xx binary
```

```c
/* freedreno/ir3/ir3_compiler_nir.c */
struct ir3_shader_variant *ir3_shader_create_variant(
    struct ir3_shader *shader,
    const struct ir3_shader_key *key,
    bool keep_ir)
{
    struct ir3_context *ctx = ir3_context_init(shader->compiler, shader, key);
    ir3_nir_lower_variant(ctx->v, shader->nir);
    emit_instructions(ctx);             /* NIR → ir3 IR */
    ir3_optimize_loop(ctx->compiler, ctx->ir);
    ir3_ra(ctx->v);                     /* register allocation */
    assemble(ctx->v);                   /* emit binary */
    return ctx->v;
}
```

[Source: Mesa ir3_compiler_nir.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/freedreno/ir3/ir3_compiler_nir.c)

### ir3 Instruction Set

ir3 (Adreno IR) is a RISC-like 64-bit instruction set with explicit sync bits:

```
; Example a6xx fragment shader instructions:
; vec4 = texture(sampler, uv):
(sy)(ss)sam.s2en.mode6.base0 (f32)(xyzw)r0.x, r0.x, r1.x, 0, 0

; mul: r0.x = r1.x * r2.x (float):
mul.f r0.x, r1.x, r2.x

; add: r0.x = r1.x + c0.x (with const):
add.f r0.x, r1.x, c0.x

; kill (discard fragment):
kill r0.x
```

The `(sy)` and `(ss)` prefixes are hardware synchronisation markers: `(sy)` stalls until all previous memory reads complete; `(ss)` stalls until all texture fetches complete. The ir3 compiler inserts these conservatively and then optimises them away where the hardware's scoreboard can guarantee ordering.

### Half-Precision (fp16)

A6xx and A7xx natively execute fp16 (half-precision) instructions at 2× throughput. The ir3 compiler promotes arithmetic to fp16 where safe (the `nir_lower_alu_to_scalar` pass, combined with ir3's own `ir3_nir_lower_io_to_scalar`, handles this):

```bash
# Enable aggressive fp16:
IR3_SHADER_DEBUG=nofp16   # disable fp16 (for debugging precision issues)
IR3_SHADER_DEBUG=verbose  # print NIR before/after passes
IR3_SHADER_DEBUG=disasm   # print final assembly
```

---

## Kernel Driver: msm_drm

### MSM DRM Architecture

```
drivers/gpu/drm/msm/
├── msm_drv.c              # DRM driver registration
├── adreno/
│   ├── a6xx_gpu.c         # A6xx GPU ops
│   ├── a6xx_gmu.c         # Graphics Management Unit (power/clock)
│   ├── a6xx_hfi.c         # Host Firmware Interface queue management
│   └── a6xx_ringbuffer.c  # command ring
├── disp/
│   ├── dpu1/              # Display Processing Unit (Snapdragon DPU)
│   └── mdp5/              # older MDP5 display
└── msm_gem.c              # GEM BO management
```

### Ring Buffer Command Submission

MSM uses a ring buffer (not a job queue) for GPU command submission:

```c
/* msm/adreno/a6xx_ringbuffer.c */
void a6xx_rb_submit(struct msm_gpu *gpu, struct msm_gem_submit *submit,
    struct msm_ringbuffer *ring)
{
    /* Write CP_INDIRECT_BUFFER packet pointing to submit's IB: */
    OUT_PKT7(ring, CP_INDIRECT_BUFFER_PFE, 3);
    OUT_RING(ring, lower_32_bits(submit->iova));
    OUT_RING(ring, upper_32_bits(submit->iova));
    OUT_RING(ring, submit->nr_cmds);

    /* Write fence value to scratch register on completion: */
    OUT_PKT7(ring, CP_EVENT_WRITE, 4);
    OUT_RING(ring, CACHE_FLUSH_TS | CP_EVENT_WRITE_0_IRQ);
    OUT_RING(ring, lower_32_bits(rbmemptr(ring, fence)));
    OUT_RING(ring, upper_32_bits(rbmemptr(ring, fence)));
    OUT_RING(ring, submit->seqno);
}
```

[Source: linux a6xx_ringbuffer.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/msm/adreno/a6xx_ringbuffer.c)

---

## GPU Memory: IOMMU and SMMU

### SMMU (System MMU)

Qualcomm SoCs use the ARM SMMU v2/v3 for GPU IOVA (I/O Virtual Address) mapping:

```c
/* msm/msm_iommu.c */
struct msm_iommu {
    struct msm_mmu base;
    struct iommu_domain *domain;
};

static int msm_iommu_map(struct msm_mmu *mmu, uint64_t iova,
    struct sg_table *sgt, size_t len, int prot)
{
    struct msm_iommu *iommu = to_msm_iommu(mmu);
    return iommu_map_sgtable(iommu->domain, iova, sgt, prot);
}
```

All GPU buffers (BOs) get an IOVA through the SMMU. GPU commands reference IOVAs, not physical addresses — enabling full memory isolation between GPU contexts and between GPU and other DMA masters.

### Protected Mode (Secure World)

A6xx supports a protected execution mode for DRM content protection (video decode in the secure world). The GPU's SMMU is configured to restrict access to secure BOs from non-secure contexts.

---

## Performance Counters and Debugging Tools

### fdperf: Live Hardware Counter Monitoring

`fdperf` is an ncurses-based terminal tool that reads Adreno hardware performance counters in real time, similar to `intel_gpu_top` for Intel hardware. It is built from `mesa/src/freedreno/fdperf/` and reads the counters via the `freedreno_perfcntrs` kernel interface.

Key counter groups available on A6xx:

| Group | Counter Examples | What to Watch |
|---|---|---|
| SP (Shader Processor) | `sp/fs_stage_full_ALU_instructions`, `sp/busy_cycles` | ALU utilisation, stall cycles |
| UCHE (L2 Cache) | `uche/vbif_read_beats_TP`, `uche/read_requests_TP` | L2 hit rate, texture bandwidth |
| LRZ | `lrz/visible_prim_after_lrz`, `lrz/full_8x8_tiles_killed` | LRZ kill rate |
| VSC | `vsc/overflow` | Binning overflow events |
| RB (Render Backend) | `rb/2D_fillrate`, `rb/3D_fillrate` | Fill rate |
| CCU | `ccu/busy_cycles`, `ccu/stall_cycles` | Cache stalls |

```bash
# Build fdperf from Mesa source:
meson setup build/ -Dtools=freedreno
ninja -C build/ fdperf

# Run live counter display:
sudo ./fdperf
# Press 'g' to cycle counter groups, 'q' to quit
```

The `AMD_performance_monitor` OpenGL extension (for Freedreno) and `VK_KHR_performance_query` (for Turnip) expose these same counters programmatically from user-space.

### VK_KHR_performance_query Integration

Turnip implements `VK_KHR_performance_query`, giving Vulkan applications direct access to the hardware performance counters defined in the `freedreno_perfcntrs` XML database. A profiling session looks like:

```c
/* 1. Enumerate available performance counters: */
uint32_t counter_count;
vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR(
    phys_dev, queue_family, &counter_count, NULL, NULL);

VkPerformanceCounterKHR *counters =
    calloc(counter_count, sizeof(*counters));
vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR(
    phys_dev, queue_family, &counter_count, counters, NULL);

/* 2. Create a query pool with selected counters: */
uint32_t enabled[] = { LRZ_VISIBLE_PRIM_IDX, SP_BUSY_CYCLES_IDX };
VkQueryPoolPerformanceCreateInfoKHR perf_info = {
    .sType = VK_STRUCTURE_TYPE_QUERY_POOL_PERFORMANCE_CREATE_INFO_KHR,
    .queueFamilyIndex   = 0,
    .counterIndexCount  = ARRAY_SIZE(enabled),
    .pCounterIndices    = enabled,
};
/* 3. Wrap draws with vkCmdBeginQuery/vkCmdEndQuery and read results. */
```

The counter indices correspond to the hardware groups defined in `src/freedreno/registers/adreno/`. On A6xx, there are 512 countable events spread across 16 groups; Turnip exposes all of them via `VK_KHR_performance_query`.

[Source: Mesa fdperf source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/freedreno/fdperf) | [Mesa freedreno driver docs](https://docs.mesa3d.org/drivers/freedreno.html)

### Command Stream Capture and cffdump

The primary debugging tool for Adreno command streams is `cffdump`, which decodes `.rd` (re-dump) binary trace files. It links in `librnn` from the Nouveau envytools project for register name resolution and calls the freedreno disassembler for inline shader disassembly.

Capturing a command stream trace:

```bash
# Enable kernel command stream capture:
echo Y | sudo tee /sys/module/msm/parameters/rd_full
sudo cat /sys/kernel/debug/dri/0/rd > trace.rd &

# Run the application:
./myapp

# Stop capture:
kill %1

# Decode with cffdump:
cffdump trace.rd

# For compressed traces:
cffdump trace.rd.gz
```

For hang debugging, use the kernel's devcoredump support:

```bash
# After a GPU hang, decode the crash dump:
crashdec -f /sys/class/devcoredump/devcd0/data > crash.txt
# The output identifies "ESTIMATED CRASH LOCATION" in the command stream
```

cffdump supports Lua scripting for custom analysis. For example, to count draw calls per shader combination:

```bash
cffdump --script my_analysis.lua trace.rd
```

[Source: freedreno reverse engineering tools wiki](https://github.com/freedreno/freedreno/wiki/Reverse-engineering-tools) | [Mesa freedreno docs](https://docs.mesa3d.org/drivers/freedreno.html)

### Command Stream Replay and rddecompiler

For reproducing driver bugs in isolation, the `replay` tool can re-execute a captured `.rd` trace file:

```bash
# Replay a captured command stream:
./replay test.rd

# Replay only frames 10–20:
./replay --first=10 --last=20 test.rd

# Decompile command stream to editable C source (A6xx+):
rddecompiler -s cmdstream_5 example.rd > repro.cc
# Edit repro.cc to isolate the bug, then:
./replay --override=5 repro.cc
```

This workflow is invaluable for bisecting rendering bugs: capture a frame with a visual artifact, replay it, edit the decompiled command stream to remove draw calls until the artifact disappears — isolating the offending draw.

### Turnip Breadcrumbs for Hang Debugging

For GPU hang debugging, Turnip supports breadcrumb markers: the driver writes monotonically increasing sequence numbers to a UDP socket before executing each command buffer section. When the GPU hangs, the last received breadcrumb number identifies the command buffer location:

```bash
# Terminal 1: listen for breadcrumbs
nc -lkvup $PORT | stdbuf -o0 xxd -pc -c 4 | \
    awk '{printf("%u\n", "0x" $0)}'

# Terminal 2: run application with breadcrumbs enabled
TU_BREADCRUMBS=$IP:$PORT,break=-1:0 ./vulkan_app

# After the hang, the last printed number identifies the location.
# Re-run with a specific breakpoint to stop just before the crash:
TU_BREADCRUMBS=$IP:$PORT,break=$LAST:0 ./vulkan_app
```

### Perfetto GPU Tracing on Snapdragon

Qualcomm Snapdragon hardware supports [Perfetto](https://perfetto.dev/) GPU counter tracing. On Android, Qualcomm's AGI (Adreno GPU Profiler / Android GPU Inspector) builds on this. On Linux (DragonBoard, Snapdragon X Elite), Perfetto can be configured to sample GPU counters directly:

```bash
# Perfetto config for GPU counter sampling (Linux/Snapdragon):
# (Save as gpu_trace.cfg)
buffers { size_kb: 65536 }
data_sources {
  config {
    name: "gpu.counters.msm_kgsl"
    gpu_counter_config {
      counter_period_ns: 1000000
      counter_ids: 1   # SP busy cycles
      counter_ids: 12  # UCHE read requests
    }
  }
}

# Run trace:
perfetto --config gpu_trace.cfg --out trace.pb --duration-ms 5000

# Visualise at https://ui.perfetto.dev/
```

Note: the Perfetto GPU counter data source name and counter ID mapping for the upstream msm_drm kernel driver may differ from the Android KGSL driver. Verify against your kernel version.

### Debug Environment Variables

```bash
# Freedreno OpenGL:
FD_MESA_DEBUG=perf,log,gmem,noubwc app

# Turnip Vulkan:
TU_DEBUG=perf,cs,nolrz,noubwc,gmem app

# Force sysmem (bypass GMEM tiling):
TU_DEBUG=sysmem app

# Force binning pass even for small scenes:
TU_DEBUG=forcebin app

# ir3 compiler debugging:
IR3_SHADER_DEBUG=verbose,disasm app 2>&1 | head -200

# MSM kernel debug:
echo 'module msm +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
dmesg | grep msm_gpu

# Stale register detection (Turnip):
TU_DEBUG_STALE_REGS_RANGE=0x00000c00,0x0000be01 \
TU_DEBUG_STALE_REGS_FLAGS=cmdbuf,renderpass \
    ./vulkan_app
```

---

## Upstream Development Workflow

### Adding Support for a New Adreno Generation

The process for upstreaming a new Adreno GPU generation (e.g., adding A8xx support) follows a well-established pattern:

1. **Register headers**: The `envytools`-style register headers in `src/freedreno/registers/adreno/` are updated. These XML files define every hardware register name, bit field, and enum value for the new generation. The `rnn-header-gen` tool generates C headers from the XML.

2. **Device database**: `src/freedreno/common/freedreno_devices.py` gains a new entry for the GPU chip ID, GMEM size, and generation-specific feature flags. Running `python3 freedreno_devices.py` regenerates the `freedreno_devices.h` C header.

3. **Kernel enablement**: New `struct adreno_info` entries are added to `drivers/gpu/drm/msm/adreno/adreno_device.c`, associating the chip ID with GPU ops, firmware paths, and quirks.

4. **Turnip/Freedreno per-gen code**: A new `a8xx/` subdirectory may be added (or the existing `a7xx/` extended) with generation-specific draw emission, GMEM configuration, LRZ setup, and register writes.

5. **CI coverage**: A new CI job is added to `.gitlab-ci.yml` targeting hardware in the freedesktop.org CI farm.

### Freedesktop CI with Qualcomm Hardware

The Mesa project runs continuous integration on real Qualcomm hardware. Qualcomm registered a GitLab runner for an RB5 (Snapdragon 888, Adreno 660) device hosted in Qualcomm labs. Additional SM8250 (Snapdragon 865) and SM8550 (Snapdragon 8 Gen 2) runners have been added. [Source: LKML drm/ci SM8250 GitLab runner](https://lkml.rescloud.iu.edu/2311.1/01550.html)

Every merge request to Mesa that touches `src/freedreno/` or `src/gallium/drivers/freedreno/` automatically triggers:
- Freedreno OpenGL conformance (GLES CTS) on RB5 hardware
- Turnip Vulkan conformance (Vulkan CTS) on RB5 hardware
- `deqp-vk` and `deqp-gles31` suites

This hardware CI catches regressions before they reach the main branch — critical for a driver that ships in production on Qualcomm's reference Linux images and postmarketOS.

### The cffdump Reverse Engineering Workflow

When bringing up a new Adreno generation, the primary reverse-engineering approach is:

1. **Capture from the proprietary driver on Android**: Using the kernel's `rd` capture interface or `kgsl`-level hooks, capture `.rd` traces of the proprietary Qualcomm OpenGL ES driver executing known-good test cases.

2. **Decode with cffdump**: Compare the proprietary driver's register writes against the open driver's output. Discrepancies identify missing or incorrect register programming in the open driver.

3. **Test with replay**: Use `replay` to isolate minimal test cases.

4. **Upstreaming**: Once the new register sequences are understood, they are encoded in the XML register database and the per-generation driver code, submitted as a Mesa merge request.

---

## Qualcomm Snapdragon X Elite on Linux

### Platform Context

The Snapdragon X Elite (X1E80100) is a Windows-on-ARM laptop platform that shipped in 2024 (Dell XPS 13, Lenovo ThinkPad X13s, etc.). Linux support via the `linux-next` tree includes:

- **msm_drm** — kernel DRM driver for the display pipeline (DPU1)
- **Freedreno/Turnip** — Mesa drivers for the Adreno 830 GPU
- **iris V4L2 decoder** — hardware H.264/H.265 decode, landed in Linux 6.15

```bash
# Check Adreno on Snapdragon X Elite:
lspci | grep -i qualcomm       # or: lsmod | grep msm
vulkaninfo --summary | grep -i adreno
cat /sys/kernel/debug/dri/0/gpu_busy
```

### Current Status (2026)

As of kernel 6.19 and Mesa 26.0:
- Adreno 830 on X Elite: Vulkan 1.3 via Turnip (working)
- Adreno Gen 8 (X2 Elite, 8 Elite): Turnip support merged Mesa 26.0
- Display output via DPU1 (working; panel link, DisplayPort)
- Hardware video decode via `iris` V4L2 driver (landed Linux 6.15)
- Suspend/resume — in progress; runtime PM functional, suspend-to-RAM stabilising
- Variable Rate Shading on Gen 8 — disabled pending stability work

The Snapdragon X Elite ships with a firmware-signed UEFI and SecureBoot; Turnip running without the proprietary Qualcomm graphics stack requires the open-source `a660_gmu.bin` firmware loaded from `/lib/firmware/qcom/`. On Debian/Ubuntu, the `firmware-qcom` package provides these blobs.

---

## Roadmap

### Near-term (6–12 months)

- **Variable Rate Shading (VRS) on A8xx**: Initial Gen 8 support ships with VRS disabled while developers resolve stability issues; enabling it is an active near-term goal. [Source](https://news.lavx.hu/article/adreno-gen-8-vulkan-support-merged-into-mesa-26-0-for-snapdragon-x2-elite-linux-graphics)
- **Qualcomm iris video decoder**: The `iris` V4L2 video decoder driver (H.264/H.265 decode on Snapdragon SoCs) landed for Linux 6.15. Integration with VA-API for hardware-accelerated decode in Firefox and Chromium is ongoing. [Source](https://www.phoronix.com/news/Linux-6.15-Media-Subsystem)
- **Suspend/resume on Snapdragon X Elite**: Power management (suspend-to-RAM, runtime PM) is in active upstream patch series. Note: exact landing kernel version needs verification.
- **Vulkan 1.4 conformance for Turnip**: Turnip is tracking the Vulkan 1.4 specification; conformance submission expected as extension coverage completes. Note: needs verification.

### Medium-term (1–3 years)

- **Ray tracing maturation on A7xx/A8xx**: Turnip has initial RT support on A7xx; the goal is full `VK_KHR_ray_tracing_pipeline` and `VK_KHR_ray_query` conformance and RT enablement on A8xx. Note: needs verification of exact extension conformance status per generation.
- **Mesh shaders and task shaders**: `VK_EXT_mesh_shader` enablement on A7xx+ is in progress; full support requires ir3 compiler work for the mesh/task shader stages.
- **ir3 compiler performance**: Ongoing NIR-level optimisations (instruction scheduling, register allocation tuning for A7xx wave sizes) to close the performance gap with the proprietary Qualcomm driver on SPEC and gaming benchmarks.
- **FastRPC and DSP offload integration**: Qualcomm's FastRPC mechanism (for DSP-accelerated AI/ML) needs deeper Linux integration to support GPU–Hexagon DSP interleaved compute workloads. Note: needs verification.

### Long-term

- **OpenCL/Compute on Turnip**: Adding a RustiCL compute path using the ir3 compiler backend to enable OpenCL workloads on Adreno GPUs for AI inference and GPGPU. Note: speculative; upstream interest needed.
- **Vulkan video extensions**: Integration of `VK_KHR_video_decode_*` with the `iris` firmware interface would enable GPU-accelerated video decode directly through Vulkan. Note: needs verification of feasibility.
- **Rust contributions to ir3/Turnip**: Following the `nova` Rust NVIDIA kernel driver trend, there is speculative interest in using Rust for safety-critical parts of Turnip or ir3; no official RFC exists. Note: speculative.

---

## Integrations

- **Ch01 (DRM Architecture)** — msm_drm registers as a DRM driver; it uses `drm_sched` for GPU job scheduling and implements DRM KMS for the Qualcomm DPU display
- **Ch11 (DMA-BUF)** — Freedreno/Turnip export GEM BOs as DMA-BUF FDs with UBWC modifier; camera capture (V4L2 on Qualcomm ISP) and display (DPU1 scan-out) share zero-copy UBWC buffers with the GPU
- **Ch14 (Gallium3D)** — Freedreno is a Gallium3D driver; shares the `pipe_context`/`pipe_screen` interface with RadeonSI, Iris, Panfrost
- **Ch15 (NIR)** — ir3 compiles from NIR; all NIR optimisation passes (fp16 promotion, scalarisation, constant folding) apply to Adreno as well as AMD/Intel
- **Ch19 (Vulkan)** — Turnip is a Mesa Vulkan driver using the `vk_common` framework (`vk_device`, `vk_command_buffer`) shared with RADV, ANV, Panvk
- **Ch20/21 (Wayland/wlroots)** — Turnip exports UBWC DMA-BUF surfaces to Wayland compositors via `zwp_linux_dmabuf_v1`; the DPU1 KMS driver scans out UBWC framebuffers directly
- **Ch159 (Panfrost/Mali)** — Compare ARM Mali (Panfrost/Panthor) vs Qualcomm Adreno (Freedreno/Turnip) open-source maturity; both targets TBDR architectures and use NIR-based compilers with per-generation ISA backends
- **Ch169 (Snapdragon X Elite Linux)** — Chapter 169 covers the full system perspective of Linux on Snapdragon X Elite; this chapter covers the GPU driver stack component in depth
