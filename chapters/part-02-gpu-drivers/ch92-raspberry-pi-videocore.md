# Chapter 92: The Raspberry Pi GPU Stack — VideoCore and V3D

> **Part**: Part II — GPU Drivers
> **Audience**: Embedded Linux developers, hobbyist hardware hackers, IoT and kiosk developers, education platform developers
> **Status**: First draft — 2026-06-18

## Table of Contents

- [Scope](#scope)
- [1. Introduction: Hardware Generations and the Road to Openness](#1-introduction-hardware-generations-and-the-road-to-openness)
- [2. VideoCore IV Architecture (VC4 / Pi 1–3)](#2-videocore-iv-architecture-vc4--pi-13)
- [3. VideoCore VI and the V3D Driver (Pi 4)](#3-videocore-vi-and-the-v3d-driver-pi-4)
- [4. V3D Shader Compiler: NIR to VIR to QPU Binary](#4-v3d-shader-compiler-nir-to-vir-to-qpu-binary)
- [5. KMS and Display on Raspberry Pi](#5-kms-and-display-on-raspberry-pi)
- [6. Multimedia and Camera: V4L2 and libcamera](#6-multimedia-and-camera-v4l2-and-libcamera)
- [7. Hardware Video Decode](#7-hardware-video-decode)
- [8. Raspberry Pi OS Graphics Stack](#8-raspberry-pi-os-graphics-stack)
- [9. V3DV: Vulkan Driver for Raspberry Pi](#9-v3dv-vulkan-driver-for-raspberry-pi)
- [10. Pi in Production: Kiosk, Signage, and Robotics](#10-pi-in-production-kiosk-signage-and-robotics)
- [11. Integrations](#11-integrations)
- [References](#references)

---

## Scope

This chapter traces the complete Linux graphics stack on Raspberry Pi hardware, from the QPU instruction set of the original VideoCore IV through the V3D kernel driver and Mesa Gallium/Vulkan drivers that power modern Pi boards. Embedded Linux developers will find the kernel driver internals (vc4, v3d, HVS, V4L2 M2M codecs) covered in depth. Hobbyist hardware hackers will appreciate the history of reverse engineering that preceded the open-source drivers. IoT and kiosk developers will find the production deployment section — Weston, cage, labwc, Chromium on Wayland — directly actionable. Education platform developers targeting Raspberry Pi as a Vulkan or OpenGL ES platform will find the conformance history and GLES capability per generation clearly laid out.

---

## 1. Introduction: Hardware Generations and the Road to Openness

Raspberry Pi boards span nearly fifteen years of **Broadcom** SoC evolution, and understanding which GPU is in which board is essential before reading a single kernel header.

| Board | SoC | GPU core | V3D revision |
|---|---|---|---|
| Pi 1, Zero, Zero W | BCM2835 | VideoCore IV | VC4 |
| Pi 2 (rev 1.1+) | BCM2836 | VideoCore IV | VC4 |
| Pi 3, Zero 2 W | BCM2837 | VideoCore IV | VC4 |
| Pi 4, CM4 | BCM2711 | VideoCore VI / V3D 4.2 | V3D 4.2 |
| Pi 5, CM5 | BCM2712 | VideoCore VII / V3D 7.1 | V3D 7.1 |

The **BCM2835** through **BCM2837** family shares the same **VideoCore IV** GPU block. In Broadcom terminology "**VideoCore IV**" names the complete multimedia processor — an ARM-adjacent processor running firmware that manages display, camera, and audio, in addition to the 3D acceleration unit. The 3D acceleration portion is driven by **QPU**s (Quad Processor Units) — 16-way **SIMD** vector processors, each with an add-ALU and a multiply-ALU pipeline that together execute two floating-point operations per cycle. Every Pi 1–3 uses the **vc4** kernel **DRM** driver and the **vc4** **Mesa** Gallium driver for **OpenGL ES**.

**VideoCore IV** uses a **tile-based deferred rendering** (**TBDR**) architecture in which a frame is divided into 64×64 pixel tiles, processed in a binning pass (coordinate shader + primitive binning into a **Tile Control List**) followed by a rendering pass (full vertex and fragment shading into an on-chip tile buffer). The **vc4** kernel driver, living at **drivers/gpu/drm/vc4/**, exposes the **DRM_IOCTL_VC4_SUBMIT_CL** submission interface accepting a **drm_vc4_submit_cl** structure; buffer objects are **CMA** (Contiguous Memory Allocator)-backed because **VideoCore IV** lacks an **MMU**. The **Mesa** **vc4** Gallium driver at **src/gallium/drivers/vc4/** compiles **TGSI** shaders to **QPU** binary and submits binning and render control lists via that **IOCTL**, supporting **GLES 2.0** on Pi 1–3.

The **BCM2711** in Pi 4 introduced a new and substantially more capable 3D block, called **V3D 4.2** by Broadcom. This block is distinct from the **VideoCore VI** firmware processor: the firmware still runs on a **VideoCore VI** core managing boot and thermal policy, but the 3D accelerator communicates directly with the Linux kernel through the **v3d** **DRM** driver, completely bypassing the firmware blob for rendering. This architectural split — firmware-managed peripherals separate from a directly-driven 3D accelerator — was the enabling condition for a fully open-source 3D driver. **V3D 4.2** adds a hardware **MMU** with a two-level page table providing per-context **GPU** virtual address spaces, a Compute Shader Dispatch (**CSD**) unit for **OpenGL ES** compute shaders and **Vulkan** compute, and a **Texture Formatting Unit** (**TFU**) for format conversion and mipmap generation in fixed-function hardware without shader involvement. The **v3d** kernel driver at **drivers/gpu/drm/v3d/** exposes four hardware queues — bin, render, **TFU**, and **CSD** — each with its own submit **IOCTL** and integrated with the **DRM GPU scheduler**. The **Mesa** **v3d** Gallium driver at **src/gallium/drivers/v3d/** is **NIR**-based throughout and supports **GLES 3.1** on Pi 4 and Pi 5.

The **BCM2712** in Pi 5 advances to **V3D 7.1**, the same architectural family but with wider **QPU** slices and improved memory bandwidth. The Pi 5 also introduces the **RP1** southbridge chip (connected to **BCM2712** over **PCIe** 2.0 x4), which takes over USB, Gigabit Ethernet, **MIPI** camera/display interfaces, and GPIO from the main SoC, leaving **BCM2712** free to focus on CPU/GPU work. The Pi 5 display path uses an updated **HVS** (informally **HVS7**) with higher bandwidth and 4K pixel format support. [Source](https://www.raspberrypi.com/news/rp1-the-silicon-controlling-raspberry-pi-5-i-o-designed-here-at-raspberry-pi/)

The **V3D** shader compiler, shared between the Gallium **v3d** driver and the Vulkan **v3dv** driver, lives at **src/broadcom/compiler/** and implements a pipeline that lowers **Mesa** **NIR** through **V3D**-specific passes (**v3d_nir_lower_io()**, **v3d_nir_lower_txf_ms()**, **v3d_nir_lower_image_load_store()**, **v3d_nir_lower_scratch()**, **nir_opt_sink()**) into **VIR** (V3D Intermediate Representation), then instruction-schedules **VIR** to hide texture lookup latency, performs graph-colouring register allocation, and emits 64-bit **QPU** binary words. Fragment shaders can run in 2-thread or 4-thread mode to hide **TMU** (Texture and Memory Unit) latency via hardware thread interleaving. Uniforms are consumed via the **LDUNIF** FIFO from a per-shader **uniforms BO** rather than from an indexed constant buffer hardware block.

Display across all Pi generations is managed by the **vc4** **DRM** driver, which registers the **HVS** (Hardware Video Scaler) as a global compositor accepting up to eight **DRM** planes, **Pixel Valve** (**PV**) blocks as **drm_crtc** objects, and encoders including **vc4_hdmi**, **vc4_dsi** (**MIPI DSI** for the official touchscreen), **vc4_vec** (composite video), and **vc4_dpi** (parallel display). The **KMS** atomic modeset path drives these through **vc4_hvs_atomic_check()** and **vc4_hvs_atomic_flush()**. **HDMI** audio is implemented via the internal **MAI** (Multi-channel Audio Interconnect) bus and exposed as an **ALSA** PCM device.

On the multimedia and camera side, the chapter covers both the legacy **MMAL** (Multimedia Abstraction Layer) over **VCHI** (VideoCore Host Interface) path — which powered the deprecated **raspistill**/**raspivid** tools and the **bcm2835_v4l2** wrapper — and the modern **libcamera** stack. **libcamera**'s **PipelineHandlerRPi** drives **Unicam** (the **CSI-2** D-PHY receiver) and the hardware **ISP** (Image Signal Processor) directly, with 3A algorithms running in the **IPA** (Image Processing Algorithm) plugin at **src/ipa/rpi/**.

Hardware video decode on Pi 4 and Pi 5 uses the **V4L2** Memory-to-Memory (**M2M**) framework via the **bcm2835-codec** driver at **drivers/staging/vc04_services/bcm2835-codec/**, which exposes **H.264**, **HEVC**, **MJPEG**, and other codecs as **/dev/video** nodes. **FFmpeg**'s **v4l2m2m** decoder family and **MPV** integrate directly with this driver. A zero-copy decode-to-display path is possible by exporting decoded **NV12** frames as **DMA-BUF** file descriptors, importing them into **Mesa** **v3d** via **EGL_EXT_image_dma_buf_import** as **EGLImage** objects, and compositing via a **Wayland** surface or a direct **DRM** overlay plane using the **HVS**'s native **NV12** support.

The **Raspberry Pi OS** graphics stack has evolved from **X11** + **Openbox** through **Wayfire** (the default Wayland compositor in Pi OS Bookworm, 2023) to **labwc** (the current default from November 2024). **Mesa** on Pi 4/5 exposes **libEGL.so** (**EGL 1.5**), **libGLESv2.so** (**GLES 3.1**), and the **vulkan-broadcom** package installs the **V3DV** Vulkan **ICD** (**libvulkan_broadcom.so**). GPU performance is commonly benchmarked with **GLmark2**.

The **V3DV** Vulkan driver at **src/broadcom/vulkan/** was developed primarily by **Igalia** and achieved **Vulkan 1.0** Khronos conformance in 2020, **Vulkan 1.2** in 2022, and **Vulkan 1.3** in **Mesa 24.3** (November 2024) for both **V3D 4.2** and **V3D 7.1**. Key driver structures include **v3dv_device**, **v3dv_cmd_buffer** (which accumulates a **BCL** and **RCL**), and **v3dv_pipeline**. Draw calls translate to **Control List** (**CL**) packet emission; compute dispatches submit **CSD** jobs via **DRM_IOCTL_V3D_SUBMIT_CSD**. The driver supports **VK_KHR_synchronization2** for integration with modern Vulkan applications and game engines.

In production environments the chapter covers kiosk and digital signage deployments using **Weston**'s kiosk shell, the **cage** single-app **Wayland** compositor (built on **wlroots**), and **Chromium** with **Ozone** Wayland support and **VA-API** hardware video decode via a **VA-API**→**V4L2** translation layer. Thermal management via **DVFS** (Dynamic Voltage and Frequency Scaling) and overclocking through **/boot/firmware/config.txt** are critical for production thermal budgets. The **Compute Module 4** (**CM4**) and **Compute Module 5** (**CM5**) form factors enable custom carrier boards for industrial **HMI**, machine vision, and edge AI use cases. **ROS 2** robotics pipelines on Pi combine **libcamera** capture, **OpenCV** with **GStreamer** **v4l2src**, **v4l2video10h264dec** hardware decode, and **OpenGL ES** or **Vulkan** compute for perception acceleration.

### A History of Openness

When the original Pi shipped in 2012, its GPU was driven entirely by a closed firmware blob (**start.elf**) loaded from the SD card boot partition by the **VideoCore** before ARM was even released from reset. The firmware exposed a **VCHI** (**VideoCore Host Interface**) IPC channel over which the ARM-side **MMAL** (**Multimedia Abstraction Layer**) API communicated. 3D rendering was available only through a proprietary **EGL**/**GLES** library distributed in **/opt/vc/**.

The first crack in that wall was Herman Hermitage's reverse engineering of the **VideoCore IV** ISA starting in 2012. The [videocoreiv](https://github.com/hermanhermitage/videocoreiv) project documented the assembly language, **QPU** pipeline, and control list format, providing the foundation that later driver work depended on.

Eric Anholt (then at Intel, later at Broadcom/Raspberry Pi) wrote the open-source **vc4** kernel **DRM** driver and accompanying **Mesa** Gallium driver, merged into Linux 4.4 and **Mesa** 10.6 respectively. For the first time, a Pi could render **OpenGL ES** using standard, firmware-free kernel interfaces. Eric subsequently authored the **v3d** kernel driver and **Mesa** driver for Pi 4, completing the transition to a fully open graphics stack. **Igalia** then led the development of the **v3dv** Vulkan driver, achieving Khronos conformance in 2020.

---

## 2. VideoCore IV Architecture (VC4 / Pi 1–3)

### QPU: The Shader Processor

The VideoCore IV QPU (Quad Processor Unit) is a 16-way SIMD vector processor. Despite being named "Quad Processor", the 4 in its name refers to groups of 4 SIMD lanes forming a "quad" — the hardware processes 16 lanes simultaneously. [Source](https://docs.broadcom.com/doc/12358545)

Each QPU has two ALU pipelines running in parallel per clock cycle: an add-ALU and a multiply-ALU. The instruction encoding captures both operations in a single 64-bit instruction word, allowing up to two floating-point operations per QPU per cycle at peak throughput. A QPU running at 400 MHz (typical Pi 3 clock) therefore achieves 6.4 GFLOPS (16 lanes × 2 ops × 400 MHz) in the vector float domain.

The QPU ISA encodes instructions in 64-bit words with the following major instruction classes:
- **ALU instructions**: encode the add-ALU opcode, multiply-ALU opcode, source register files A and B, destination register, and a write signal field.
- **Load immediate instructions**: load a 32-bit immediate into a QPU register.
- **Branch instructions**: relative and absolute branches with a mandatory three-instruction branch delay slot.
- **Semaphore instructions**: 16 hardware semaphores for QPU-to-QPU synchronisation within a tile.

The QPU register file is 32 entries of 32-bit vectors, split into register file A and register file B. Uniforms (constant parameters uploaded from the CPU) are read via a dedicated uniform FIFO: the shader reads `QPU_R_UNIFORM` to consume successive uniform values. Texture lookups are performed by writing a texture coordinate to `QPU_W_TMU0_S` (and optionally T, R, B channels), then reading the result from `QPU_R4` after an instruction latency.

### VC4 Tiled Rendering Pipeline

VideoCore IV uses a **tile-based deferred rendering** architecture. A frame is divided into 64×64 pixel tiles. Rendering proceeds in two passes.

**Binning pass (coordinate shader + primitive binning)**: The GPU runs a reduced vertex shader called the **coordinate shader** that computes only clip-space positions (not varyings). A primitive assembly unit sorts each triangle into the tile bins it overlaps. The binner writes a **Tile Control List** (TCL) per tile: a compact instruction stream describing which primitives touch that tile.

**Rendering pass**: For each tile, the renderer reads the TCL, re-runs the full **vertex shader** (computing varyings), then runs the **fragment shader** (rasterisation, texture sampling, blending) for each covered pixel. The tile buffer (64×64 pixels of colour, depth, and stencil) is stored in a dedicated on-chip tile buffer SRAM. After tile completion, the tile is flushed to main memory (SDRAM) via DMA.

This architecture dramatically reduces memory bandwidth: colour and depth reads/writes during shading go to fast on-chip SRAM rather than SDRAM. Only tile load (fast-clear or background blit) and tile store (blending with framebuffer) require SDRAM access.

### vc4 Kernel DRM Driver

The kernel driver lives at [`drivers/gpu/drm/vc4/`](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/vc4). Key source files:

- `vc4_drv.c` — DRM driver registration, `vc4_dev` device structure
- `vc4_bo.c` — Buffer Object (BO) management via CMA (Contiguous Memory Allocator); VC4 lacks an MMU so requires physically contiguous buffers
- `vc4_cl.c` — Control List helpers: `vc4_cl` structure wrapping a BO for the binner or render control list, macros for emitting CL packets
- `vc4_gem.c` — GEM submission path: `vc4_exec_info` tracking a submitted frame
- `vc4_render_cl.c` — Kernel-side render control list generation (the render CL is generated by the kernel to avoid security validation complexity)
- `vc4_validate.c` — Binner CL validation: the user submits a binner CL which must be validated to prevent GPU-accessible memory escapes
- `vc4_v3d.c` — QPU engine: cache flush, reset, performance counter setup

The key IOCTL is `DRM_IOCTL_VC4_SUBMIT_CL`, accepting a `drm_vc4_submit_cl` structure:

```c
/* Source: include/uapi/drm/vc4_drm.h */
struct drm_vc4_submit_cl {
    __u64 bin_cl;           /* Pointer to binner control list */
    __u64 shader_rec;       /* Array of shader records */
    __u64 uniforms;         /* Uniforms array (pointer to per-shader uniform data) */
    __u64 bo_handles;       /* GEM handle array for BOs referenced in this job */
    __u32 bin_cl_size;
    __u32 shader_rec_size;
    __u32 shader_rec_count;
    __u32 uniforms_size;
    __u32 bo_handle_count;
    __u16 width;            /* Render width in pixels */
    __u16 height;           /* Render height in pixels */
    __u8  min_x_tile;
    __u8  min_y_tile;
    __u8  max_x_tile;
    __u8  max_y_tile;
    struct drm_vc4_submit_rcl_surface color_read;
    struct drm_vc4_submit_rcl_surface color_write;
    struct drm_vc4_submit_rcl_surface zs_read;
    struct drm_vc4_submit_rcl_surface zs_write;
    /* ... in/out fences, perf counters ... */
};
```

### Mesa vc4 Gallium Driver

The Mesa userspace driver lives at `src/gallium/drivers/vc4/`. It implements the Gallium3D `pipe_driver_descriptor` interface. The shader compiler takes TGSI (the old Gallium shader IR) and converts it to VC4 IR, then emits QPU binary. The driver assembles binner and render control lists in userspace (the binner CL is validated by the kernel; the render CL is regenerated by the kernel for security), packages them into `drm_vc4_submit_cl`, and submits via the IOCTL.

GLES 2.0 is supported on VC4 (the Pi 1–3 maximum). The vc4 driver does not support GLES 3.0 or compute.

---

## 3. VideoCore VI and the V3D Driver (Pi 4)

### V3D 4.2 Architecture

The BCM2711's V3D 4.2 block represents a significant architectural evolution over VideoCore IV. Key differences:

- **MMU**: V3D 4.x has a two-level page table MMU providing a 4GB GPU virtual address space. Buffer objects use `shmem`-backed memory rather than CMA, since physically contiguous memory is no longer required. The MMU provides isolation between GPU client contexts.
- **Multiple QPU slices**: The V3D 4.2 in BCM2711 has 4 QPUs per slice; the V3D 7.1 in BCM2712 has 12 QPUs total across its wider slice count.
- **Dedicated compute**: A Compute Shader Dispatch (CSD) unit handles OpenGL ES compute shaders and Vulkan compute, dispatching work items independently of the graphics pipeline.
- **Texture Formatting Unit (TFU)**: A fixed-function unit for format conversion, mipmap generation, and YUV-to-RGB conversion without CPU involvement.
- **Separate drivers**: V3D has its own kernel DRM driver (`drivers/gpu/drm/v3d/`) entirely separate from the vc4 driver, and its own Mesa Gallium driver (`src/gallium/drivers/v3d/`).

### v3d Kernel DRM Driver

The kernel driver lives at [`drivers/gpu/drm/v3d/`](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/v3d). The [kernel documentation](https://docs.kernel.org/gpu/v3d.html) describes the overall architecture. Key source files:

- `v3d_drv.c` — DRM driver init, `v3d_dev` device structure, IOCTL table
- `v3d_bo.c` — GEM buffer object management; `v3d_bo` wraps a `drm_gem_shmem_object` (no CMA required due to MMU)
- `v3d_mmu.c` — Page table management for the V3D MMU; `v3d_mmu_set_page_table()`, `v3d_mmu_insert_ptes()`
- `v3d_gem.c` — Job submission and reference tracking; `v3d_job` base structure
- `v3d_sched.c` — DRM GPU scheduler integration; one `drm_gpu_scheduler` per hardware queue (bin, render, TFU, CSD)
- `v3d_irq.c` — Interrupt handler; completion signalling for bin, render, TFU, and CSD done events
- `v3d_perfmon.c` — Performance counter infrastructure

#### Job Types

V3D exposes four job types, each with its own hardware queue and submit IOCTL:

```c
/* Source: drivers/gpu/drm/v3d/v3d_drv.h */
enum v3d_queue {
    V3D_BIN,      /* Binning (tiling) pass */
    V3D_RENDER,   /* Rendering pass */
    V3D_TFU,      /* Texture Formatting Unit */
    V3D_CSD,      /* Compute Shader Dispatch */
    V3D_CACHE_CLEAN, /* L2C cache clean (internal) */
};
```

A complete draw call submission consists of a **bin job** followed by a **render job**, with an explicit `drm_sched_job_add_dependency()` call making the render job depend on the bin job's fence. This replaces VC4's approach of submitting a single combined job. The kernel documentation states: "Dependencies between bin and render are managed using `drm_sched_job_add_dependency()`, instead of having clients submit jobs using the hardware's semaphores to interlock between them." [Source](https://docs.kernel.org/gpu/v3d.html)

The **TFU job** submits a format conversion operation (e.g., linear-to-tiled texture conversion, or mipmap chain generation) entirely in hardware without shader involvement. The **CSD job** dispatches compute workloads, supplying a workgroup count in X/Y/Z dimensions.

The bin pipeline can run out of memory when binning large or complex scenes. The driver handles the binner out-of-memory interrupt dynamically by allocating additional memory and restarting the binner, invisible to userspace.

#### V3D MMU and Buffer Management

Unlike VC4's flat CMA buffers, V3D uses a per-context page table. The MMU supports up to 4GB of GPU virtual address space per context (V3D 3.x used a flat single-level page table; V3D 4.x and 7.x use two-level tables). Buffer objects are imported into the GPU virtual address space at job submission time by inserting PTEs into the context's page table. This enables buffer sharing (DMA-BUF import from V4L2 decoded frames) and proper GPU memory isolation between processes.

### Mesa v3d Gallium Driver

`src/gallium/drivers/v3d/` implements Gallium3D for V3D. Unlike the vc4 driver which used TGSI, the v3d driver is NIR-based throughout. The driver supports GLES 3.1 on Pi 4/5, including compute shaders via the CSD path. Key files:

- `v3d_context.c` — `v3d_context` creation and destruction
- `v3d_program.c` — Shader compilation entry points; `v3d_uncompiled_shader` (TGSI/NIR in), `v3d_compiled_shader` (QPU binary out)
- `v3d_uniforms.c` — Uniform buffer handling; uploading uniform data to BOs for the QPU uniform FIFO
- `v3d_job.c` — `v3d_job` for Gallium (render command accumulation)
- `v3d_blit.c` — TFU-accelerated blit and mipmap generation

---

## 4. V3D Shader Compiler: NIR to VIR to QPU Binary

The V3D compiler pipeline is shared between the Gallium v3d driver and the Vulkan v3dv driver. It lives in `src/broadcom/compiler/` and converts Mesa NIR into V3D VIR (V3D Intermediate Representation), then schedules and register-allocates VIR to produce QPU binary. [Source](https://blogs.igalia.com/itoral/2021/03/16/improving-performance-of-the-v3d-compiler-for-opengl-and-vulkan/)

### Entry Points

Three top-level entry points cover the three shader types:

```c
/* Source: src/broadcom/compiler/v3d_compiler.h */
struct v3d_compile *v3d_compile_vs(const struct v3d_compiler *compiler,
                                   struct v3d_vs_key *key,
                                   struct v3d_uncompiled_shader *shader_state,
                                   nir_shader *s);

struct v3d_compile *v3d_compile_fs(const struct v3d_compiler *compiler,
                                   struct v3d_fs_key *key,
                                   struct v3d_uncompiled_shader *shader_state,
                                   nir_shader *s);

struct v3d_compile *v3d_compile_cs(const struct v3d_compiler *compiler,
                                   struct v3d_key *key,
                                   struct v3d_uncompiled_shader *shader_state,
                                   nir_shader *s);
```

Each function runs a sequence of NIR lowering passes, lowers to VIR, schedules instructions, allocates registers, and emits QPU binary.

### NIR Lowering Passes

Before lowering to VIR the compiler applies several V3D-specific NIR passes:

- `v3d_nir_lower_io()` — Maps GLSL in/out variables to QPU register-file accesses, varying reads, and VPM (Vertex Pipeline Memory) reads for vertex attributes.
- `v3d_nir_lower_txf_ms()` — Lowers multi-sample texture fetches to V3D's texture fetch mechanism.
- `v3d_nir_lower_image_load_store()` — Lowers image load/store operations to the V3D's TMU (Texture and Memory Unit) instructions.
- `v3d_nir_lower_scratch()` — Lowers shader scratch memory usage to stack-allocated TMU reads/writes.
- `nir_opt_sink()` — A general Mesa pass that moves instruction definitions closer to their uses, reducing register live ranges. This pass was found by Igalia to be particularly effective on V3D, cutting total instruction counts by ~8% on representative shader databases.

### VIR: V3D Intermediate Representation

VIR is an SSA-based IR specific to the V3D compiler. Each VIR instruction maps closely to a QPU instruction or a short sequence. Key VIR opcodes include:

| VIR opcode | QPU mapping |
|---|---|
| `V3D_QPU_A_FADD` | Floating-point add (add ALU) |
| `V3D_QPU_M_FMUL` | Floating-point multiply (mul ALU) |
| `V3D_QPU_A_MOV` | Register move |
| `V3D_QPU_M_SMUL24` | 24-bit integer multiply |
| `V3D_QPU_A_FMIN` / `FMAX` | Float min/max |
| `LDTMU` | Load from Texture and Memory Unit (texture result read-back) |
| `LDUNIF` | Load next uniform value |
| `LDI` | Load immediate |

VIR also tracks texture operation setup: `V3D_QPU_SIG_WRTMUC` (write TMU coordinate) signals and `V3D_QPU_SIG_LDTMU` (load TMU result) form pairs that bound the texture latency window the scheduler must respect.

### Instruction Scheduling and Register Allocation

The instruction scheduler builds a dependency graph from VIR SSA data flow and known QPU latencies. The key latency to hide is the texture lookup latency — typically 8–20 QPU cycles between writing TMU coordinates and reading the result. The scheduler moves independent instructions into that latency window.

Fragment shaders can run in **2-thread** or **4-thread** mode. In multi-threaded fragment mode, the QPU hardware interleaves multiple logical fragment shader threads to hide texture latency: while one thread waits for a texture result, the QPU executes instructions from another thread. The compiler's thread count selection is governed by register pressure: a shader using many temporary registers cannot support 4-thread mode (which halves the register file per thread). The compiler may recompile at lower thread count if register spilling is needed. [Source](https://blogs.igalia.com/itoral/2021/03/16/improving-performance-of-the-v3d-compiler-for-opengl-and-vulkan/)

After scheduling, a graph-coloring register allocator maps VIR temporaries to QPU register file entries. Register spills are handled via the TMU scratch path (`v3d_nir_lower_scratch()`).

### QPU Binary Emission

The final pass (`vir_to_qpu.c`) converts scheduled VIR to 64-bit QPU instruction words. Each word encodes:

```
[63:58] signal field (WRTMUC, LDTMU, LDUNIF, etc.)
[57:54] conditions / pack
[53:46] add-ALU opcode
[45:38] mul-ALU opcode
[37:32] add destination (QPU_W_* write signal)
[31:26] mul destination
[25:20] add source a
[19:14] add source b
[13: 8] mul source a
[ 7: 0] mul source b + register file selects
```

On V3D 7.1 (Pi 5) the instruction encoding is the same architectural family but with extended opcode fields and additional write signals for the wider QPU slices.

### Uniform Buffer Handling

Uniforms are not stored in a separate constant buffer hardware block; instead, each shader BO contains a **uniforms BO** that is uploaded alongside the QPU binary. The QPU reads uniforms sequentially from the LDUNIF FIFO. The compiler tracks uniform reads in the VIR stream and the driver uploads the correct uniform values in order at each draw call. This design means the CPU must re-upload uniforms on every draw if they change (there is no indexed constant buffer hardware).

---

## 5. KMS and Display on Raspberry Pi

### VC4 Display Pipeline Architecture

The vc4 DRM driver handles display for all Pi 1-5 boards. Even on Pi 4 and Pi 5 — where the 3D driver is `v3d` — display is still managed by `vc4` because the Hardware Video Scaler (HVS) and pixel valves are part of the VideoCore display subsystem common across all Broadcom Pi SoCs. [Source](https://docs.kernel.org/gpu/vc4.html)

The vc4 display pipeline has three layers:

1. **HVS (Hardware Video Scaler)**: A global compositor that accepts up to eight display layers (DRM planes). For each active layer, the HVS performs scaling, pixel format conversion, and colorspace conversion at system clock rate, producing a stream of composited pixels. The HVS has multiple output FIFOs, one per active CRTC.

2. **Pixel Valve (PV)**: Each CRTC is a Pixel Valve that generates HSYNC, VSYNC, and pixel clock timing, consuming the HVS output FIFO and feeding an encoder.

3. **Encoders**: `vc4_hdmi` (HDMI 0 and HDMI 1), `vc4_dsi` (DSI0 single-lane, DSI1 four-lane for the official Raspberry Pi touchscreen), `vc4_vec` (composite video on Pi 1-3), `vc4_dpi` (DPI parallel display).

Key kernel functions:

```c
/* Source: drivers/gpu/drm/vc4/vc4_hvs.c */
/* Allocate HVS display list memory for a CRTC's planes */
int vc4_hvs_atomic_check(struct drm_device *dev, struct drm_atomic_state *state);

/* Write display list entries into HVS DLIST SRAM at atomic flush */
void vc4_hvs_atomic_flush(struct drm_crtc *crtc,
                          struct drm_atomic_state *state);

/* Source: drivers/gpu/drm/vc4/vc4_plane.c */
/* Compute HVS element state for a plane configuration */
int vc4_plane_atomic_check(struct drm_plane *plane,
                           struct drm_atomic_state *state);
```

The atomic modeset path follows the standard DRM atomic workflow (see Ch2). At `atomic_check` time, the driver computes what HVS display list entries would be needed for each plane, verifying hardware limits (scale factor limits, layer count). At `atomic_flush`, the CRTC triggers the HVS to switch to the new display list at the next vertical blank.

### HDMI Audio

HDMI audio on vc4 is implemented inside the HDMI IP block using an internal MAI (Multi-channel Audio Interconnect) bus. The driver exposes an ALSA PCM device. IEC-60958 infoframes and ELD (EDID-Like Data for audio) are managed by the `vc4_hdmi` subdriver. [Source](https://docs.kernel.org/gpu/vc4.html)

### DSI Display

BCM2835 has two MIPI DSI controllers: DSI0 (single data lane) and DSI1 (four data lanes). The `vc4_dsi.c` driver implements `drm_bridge` and `drm_panel` support. The official Raspberry Pi DSI touchscreen panels connect to DSI1. The Pi 5 DSI interface is routed through the RP1 southbridge's MIPI DSI transmitter rather than directly from BCM2712.

### Pi 5 Display: HVS7 and RP1

The BCM2712 in Pi 5 uses an updated HVS (informally called HVS7) with higher bandwidth and support for 4K pixel formats. The vc4 driver gained Pi 5 support in kernel 6.1 (via Raspberry Pi downstream) and was upstreamed over the 6.x kernel series.

The Pi 5 supports two HDMI outputs at up to 4K@60 Hz simultaneously, and DSI 0/1 for display panels. The `dtoverlay` system in [`config.txt`](https://www.raspberrypi.com/documentation/computers/config_txt.html) selects which display outputs are active at boot. The RP1's PIO peripheral is documented separately from the GPU display pipeline but can drive GPIO-connected display panels via the `vc4-kms-dpi-*` device tree overlays.

---

## 6. Multimedia and Camera: V4L2 and libcamera

### Legacy Path: MMAL over VCHI

Before libcamera, Raspberry Pi camera and multimedia ran through MMAL (Multi-Media Abstraction Layer), a Broadcom-designed API that tunneled commands over VCHI (VideoCore Host Interface) IPC to the VideoCore firmware. The firmware ran the ISP, camera, and codec engines in hardware on behalf of the ARM. The ARM-side `raspicam` tools (`raspistill`, `raspivid`) used MMAL exclusively. This path is deprecated on Pi OS Bookworm and newer boards.

The `bcm2835_v4l2` driver exposed a V4L2 capture device (`/dev/video0`) that wrapped MMAL internally, providing V4L2 semantics to applications like Motion or GStreamer while still calling into firmware under the hood.

### libcamera on Raspberry Pi

libcamera is an open-source C++ camera stack that replaced MMAL for camera capture. [Source](https://libcamera.org/) The Pi-specific pipeline handler is `PipelineHandlerRPi` (in the libcamera source at `src/libcamera/pipeline/rpi/`), which drives two hardware blocks directly:

- **Unicam**: The CSI-2 D-PHY receiver in BCM2835/BCM2711. The first Unicam instance supports two CSI-2 data lanes; the second supports four. Unicam DMA-copies raw sensor frames to memory. The kernel driver for Unicam lived in the Raspberry Pi downstream tree through kernel 6.6; upstreaming work using the mainline V4L2 subdev API began in 2024. [Source](https://lists.libcamera.org/pipermail/libcamera-devel/2024-March/040711.html)

- **ISP (Image Signal Processor)**: A hardware ISP in BCM2711 performs debayering, Auto White Balance (AWB), Color Correction Matrix (CCM), lens shading correction, and gamma correction. The ISP driver is at `drivers/staging/vc04_services/bcm2835-isp/` in the Raspberry Pi kernel tree.

The libcamera request-response API models the camera pipeline:

```cpp
// Source: libcamera public API — libcamera/camera.h
libcamera::CameraManager *cm = new libcamera::CameraManager();
cm->start();

std::shared_ptr<libcamera::Camera> camera = cm->get("platform/...unicam");
camera->acquire();

libcamera::CameraConfiguration *config =
    camera->generateConfiguration({libcamera::StreamRole::StillCapture});
camera->configure(config);

// Allocate FrameBuffers and create Requests
libcamera::FrameBufferAllocator *allocator =
    new libcamera::FrameBufferAllocator(camera);
allocator->allocate(config->at(0));

libcamera::Request *request = camera->createRequest();
request->addBuffer(&config->at(0), buffer);
camera->start();
camera->queueRequest(request);
```

The IPA (Image Processing Algorithm) plugin for Raspberry Pi (`src/ipa/rpi/`) runs the 3A (auto-exposure, auto-white-balance, auto-focus) algorithms in userspace, communicating with the kernel ISP driver via the libcamera IPA interface. The ISP tuning parameters (lens shading tables, CCM matrices, gamma curves) are stored per-camera-module in JSON tuning files.

The user-facing capture applications (`libcamera-still`, `libcamera-vid`, `rpicam-still`, `rpicam-vid`) replaced the deprecated `raspistill`/`raspivid` tools in Raspberry Pi OS Bullseye and later.

---

## 7. Hardware Video Decode

### V4L2 Memory-to-Memory Framework

Hardware video decode on Pi 4 and Pi 5 uses the V4L2 Memory-to-Memory (M2M) framework. An M2M device exposes two queues: an OUTPUT queue (compressed input) and a CAPTURE queue (decoded output). The application writes compressed packets to the OUTPUT queue and reads decoded frames from the CAPTURE queue.

### bcm2835-codec Driver

The `bcm2835-codec` driver (at `drivers/staging/vc04_services/bcm2835-codec/` in the Raspberry Pi kernel tree) is the V4L2 M2M driver that uses the VideoCore firmware's codec engines via VCHI/MMAL under the hood. It exposes multiple `/dev/video*` nodes: [Source](https://github.com/raspberrypi/linux/blob/rpi-6.6.y/drivers/staging/vc04_services/bcm2835-codec/bcm2835-v4l2-codec.c)

- `/dev/video10` — H.264 decoder
- `/dev/video11` — H.264 encoder
- `/dev/video12` — ISP (image signal processor format conversion)
- `/dev/video18` / `/dev/video31` — HEVC decoder, MJPEG decoder (device numbers may vary with kernel version; verify with `v4l2-ctl --list-devices`)

**Note: HEVC and VP9 decode support on Pi 4 requires the relevant MMAL component in the VideoCore firmware; not all firmware versions or configurations expose all codecs. Verify with `v4l2-ctl -d /dev/videoN --list-formats-out` on your target board.**

### FFmpeg and MPV Integration

FFmpeg supports V4L2 M2M through its `v4l2m2m` decoder family:

```bash
# H.264 hardware decode on Pi 4
ffmpeg -hwaccel v4l2m2m -vcodec h264_v4l2m2m -i input.mp4 -f null -

# Playback with MPV hardware decode
mpv --hwdec=v4l2m2m video.mp4
```

### Zero-Copy Display Path

A particularly important path for video playback combines hardware decode with OpenGL ES display without a CPU-side copy:

1. V4L2 M2M decoder exports decoded frames as DMA-BUF file descriptors (NV12 format).
2. The NV12 DMA-BUF is imported into Mesa v3d via `EGLImage` using `EGL_EXT_image_dma_buf_import`.
3. The EGLImage becomes an OpenGL ES texture (NV12 with YUV-to-RGB conversion either in shader or via HVS plane).
4. The V3D GPU composites the decoded video onto the display via a Wayland surface or direct DRM overlay plane.

This path avoids any CPU-side pixel copy, keeping the decoded video data on the GPU/display path throughout. The vc4 HVS can natively display NV12 on a hardware overlay plane, making it possible to display decoded video without the 3D GPU at all.

### Pi 5 Video Decode

The BCM2712 in Pi 5 includes improved hardware codec support with better H.265/HEVC throughput. The same `bcm2835-codec` driver is used (updated for Pi 5 in the Raspberry Pi kernel fork at `rpi-6.6.y` and later). FFmpeg with `v4l2m2m` works identically on Pi 5.

---

## 8. Raspberry Pi OS Graphics Stack

### Compositor Evolution

Raspberry Pi OS has gone through three compositor generations:

1. **X11 + Openbox (through Bullseye)**: The traditional X11 stack using the `fbdev` or early KMS backend.
2. **Wayfire (Bookworm, 2023)**: When Pi OS Bookworm was released in October 2023, Wayland became the default for Pi 4 and Pi 5, using Wayfire as the compositor. Wayfire is a wlroots-based Wayland compositor with a plugin architecture. Older Pi boards (Pi 2/3) initially retained X11.
3. **labwc (November 2024 onwards)**: Raspberry Pi switched the default Wayland compositor from Wayfire to labwc — "a wlroots-based, lightweight, minimalist Wayland compositor" — after finding Wayfire's development direction increasingly incompatible with their hardware requirements. labwc provides an openbox-compatible layout model on Wayland. This release also brought Wayland to all Pi models. [Source](https://www.raspberrypi.com/news/a-new-release-of-raspberry-pi-os/)

### EGL and OpenGL ES

Mesa on Pi 4/5 exposes:
- `libEGL.so` — EGL 1.5 via Mesa's EGL implementation
- `libGLESv2.so` — GLES 3.1 (Pi 4 via v3d 4.2, Pi 5 via v3d 7.1)
- `libGLESv1_CM.so` — GLES 1.1 (legacy)

On Pi 2/3 the vc4 Mesa driver exposes GLES 2.0 only. The GLES 3.0 gap on vc4 is a design constraint: the QPU architecture lacks the integer and transform-feedback features that GLES 3.0 requires.

OpenGL ES 3.1 on Pi 4 includes compute shaders (dispatched via the V3D CSD unit), image load/store, and shader storage buffer objects (SSBOs). These are critical for modern multimedia and AI inference workloads.

### Vulkan

The `vulkan-broadcom` Mesa package installs the V3DV Vulkan ICD (Installable Client Driver). The ICD JSON points to `libvulkan_broadcom.so`:

```bash
# Verify Vulkan on Pi 4/5
vulkaninfo --summary 2>/dev/null | grep -E "apiVersion|GPU"
# Expected on Pi 5: Vulkan 1.3, GPU: V3D 7.1
```

### Performance Reference

GLmark2 is the standard OpenGL ES benchmark used for Pi GPU comparisons. Results are highly configuration-dependent (clock speed, memory frequency, thermal throttling):

| Board | GLmark2 (approximate, default config) |
|---|---|
| Pi 3 (vc4) | ~100–180 |
| Pi 4 (v3d 4.2) | ~500–750 |
| Pi 5 (v3d 7.1) | ~1800–2500 |

These are illustrative estimates across community reports; actual scores vary significantly with overclocking, cooling, and software configuration. [Source](https://www.raspberrypi.com/news/benchmarking-raspberry-pi-5/)

---

## 9. V3DV: Vulkan Driver for Raspberry Pi

### Driver Location and History

The V3DV driver lives at [`src/broadcom/vulkan/`](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/broadcom/vulkan) in the Mesa repository. It was developed primarily by Igalia under contract from Raspberry Pi Ltd. The conformance history:

- **Vulkan 1.0**: First Khronos conformance achieved October 2020 (Mesa 20.3) — the milestone that made V3DV an official Khronos member implementation.
- **Vulkan 1.2**: Conformance achieved and announced in August 2022.
- **Vulkan 1.3**: Conformance added in Mesa 24.3 (November 2024), covering both Raspberry Pi 4 (V3D 4.2) and Raspberry Pi 5 (V3D 7.1). [Source](https://9to5linux.com/mesa-24-3-open-source-graphics-stack-adds-vulkan-1-3-conformance-for-v3dv)

Mesa 26.1 (May 2026) added further extensions including `VK_KHR_present_id`, `VK_KHR_present_wait`, and `VK_EXT_hdr_metadata` to V3DV.

### Key Data Structures

```c
/* Source: src/broadcom/vulkan/v3dv_device.h */
struct v3dv_device {
   struct vk_device vk;              /* Vulkan device base */
   struct v3dv_physical_device *pdevice;
   struct v3d_device_info devinfo;   /* Hardware revision info */

   struct v3dv_queue *queue;         /* Default graphics+compute queue */
   struct v3dv_bo_cache bo_cache;    /* BO recycling cache */

   int fd;                           /* DRM device fd */
   struct v3d_simulator_file *sim_file; /* NULL on real hardware */
};

struct v3dv_cmd_buffer {
   struct vk_command_buffer vk;
   struct v3dv_device *device;

   struct v3dv_cl bcl;  /* Binning Control List being recorded */
   struct v3dv_cl rcl;  /* Render Control List */
   struct v3dv_cl indirect; /* Indirect CL for state changes */

   /* Current render pass state */
   struct v3dv_render_pass *pass;
   struct v3dv_framebuffer *framebuffer;
   uint32_t subpass_idx;
};

struct v3dv_pipeline {
   struct vk_object_base base;
   struct v3dv_device *device;

   struct v3dv_pipeline_stage *vs;   /* Vertex shader compiled stage */
   struct v3dv_pipeline_stage *fs;   /* Fragment shader compiled stage */
   struct v3dv_pipeline_stage *cs;   /* Compute shader compiled stage */

   /* V3D hardware attribute format descriptors */
   uint32_t va_count;
   struct v3d_vs_prog_data *vs_prog_data;
   struct v3d_fs_prog_data *fs_prog_data;
};
```

### Control List Generation for Draw Calls

Vulkan draw calls translate to Control List (CL) packet emission. The V3DV driver inherits the same CL packet format from the hardware spec (shared with the kernel driver). When `vkCmdDraw()` is recorded:

1. The driver emits state-change packets to the BCL (Binning Control List): viewport, scissor, vertex attribute format, shader state record pointer.
2. A `PRIM_LIST` packet triggers the hardware binner for the draw.
3. At `vkCmdEndRenderPass()`, the driver finalises the BCL and generates the RCL (Render Control List) describing tile clear operations, tile load, tile render, and tile store packets.
4. On queue submit (`vkQueueSubmit()`), the driver submits a bin job and a render job via `DRM_IOCTL_V3D_SUBMIT_CL`, with the render job depending on the bin job's completion fence.

For compute dispatches (`vkCmdDispatch()`), the driver instead submits a CSD job via `DRM_IOCTL_V3D_SUBMIT_CSD`.

### V3DV Hardware Constraints and Workarounds

V3D is a unified-memory GPU: it shares LPDDR4/LPDDR5 SDRAM with the CPU. This means:

- **No dedicated VRAM**: Allocation pressure is shared with the CPU. Large framebuffers, textures, and buffer objects compete with the OS for system memory.
- **Occupancy-limited compute**: The QPU's register file limits thread occupancy on compute workloads. Shaders with high register usage cannot fill the QPU pipelines.
- **Push constant limitations**: V3D does not have a dedicated push constant hardware block; push constants are implemented via uniform buffer uploads before each draw.

The V3DV driver implements several workarounds for hardware errata and missing features. For example, certain blending modes require shader-based emulation on V3D 4.2 that are accelerated in hardware on V3D 7.1. The `v3d_device_info` structure (populated at driver init from the V3D hub identification register) selects between hardware-version-specific code paths.

### VK_KHR_synchronization2

V3DV supports `VK_KHR_synchronization2`, which is required by many modern Vulkan applications and game engines. The driver maps Vulkan pipeline stage flags to V3D job completion fences and uses the DRM scheduler's fence infrastructure for synchronisation.

---

## 10. Pi in Production: Kiosk, Signage, and Robotics

### Kiosk and Digital Signage

Raspberry Pi boards are widely deployed as digital signage and kiosk terminals. The standard production stack:

**Weston kiosk shell**: Weston (the reference Wayland compositor) includes a kiosk shell (`weston --shell=kiosk`) that displays a single maximised application without desktop chrome. Suitable for simple signage with a dedicated application.

**cage**: A minimal single-app Wayland compositor:

```bash
# Run a single browser in kiosk mode
cage chromium-browser -- --ozone-platform=wayland --kiosk https://example.com
```

`cage` uses wlroots and the vc4/v3d DRM backend, making it compatible with all Pi 4/5 hardware.

**Chromium on Wayland**: Raspberry Pi OS Bookworm includes Chromium with Ozone Wayland support:

```bash
chromium-browser --ozone-platform=wayland \
                 --enable-features=VaapiVideoDecodeLinuxGL \
                 --use-gl=egl \
                 --kiosk https://dashboard.example.com
```

The `--enable-features=VaapiVideoDecodeLinuxGL` flag enables VA-API hardware video decode within Chromium on Pi 4/5, using the V4L2 M2M backend via a VA-API → V4L2 translation layer.

### Thermal Management and Overclocking

For production deployments, thermal management is critical. The Pi 5 includes an active cooling fan header; the Pi 4 relies on heatsink and optional fan HAT. Monitor temperatures:

```bash
# Read SoC temperature
cat /sys/class/thermal/thermal_zone0/temp
# Returns value in millidegrees Celsius (e.g., 52000 = 52°C)

# Watch temperature continuously
watch -n 1 vcgencmd measure_temp
```

GPU overclocking is configured in `/boot/firmware/config.txt`:

```ini
# Pi 5 example — use with active cooling only
gpu_freq=1000        # Default 910 MHz, max varies by bin
over_voltage=2       # Slight voltage increase for stability
```

The `force_turbo=1` flag disables DVFS (Dynamic Voltage and Frequency Scaling) and keeps the CPU/GPU at maximum clocks continuously; it triggers a warranty-void flag in the OTP fuses and is not recommended for production thermal budgets.

### Raspberry Pi Compute Module in Production

The Compute Module 4 (CM4) and Compute Module 5 (CM5) provide the BCM2711 and BCM2712 SoCs on a SODIMM-style module for custom carrier boards. Production uses include:

- Industrial HMI (Human-Machine Interface) panels using DSI display
- Thin-client terminals using HDMI output
- Machine vision systems combining the camera ISP with OpenCV or TensorFlow Lite inference
- Edge AI devices leveraging GLES 3.1 compute shaders or Vulkan compute for neural network inference

### ROS 2 and Robotics

ROS 2 (Robot Operating System 2) runs on Raspberry Pi. GPU-accelerated perception pipelines combine:

- **libcamera** for camera capture through the ISP
- **OpenCV** with a GStreamer `v4l2src` pipeline for V4L2 capture
- **GStreamer** with `v4l2video10h264dec` for hardware H.264 decode
- **OpenGL ES compute** or **Vulkan compute** for point cloud processing or SLAM acceleration

```bash
# Example GStreamer pipeline: camera → hardware encode → RTP stream
gst-launch-1.0 libcamerasrc ! \
  video/x-raw,format=NV12,width=1920,height=1080,framerate=30/1 ! \
  v4l2h264enc ! h264parse ! rtph264pay ! \
  udpsink host=192.168.1.100 port=5000
```

---

## 11. Integrations

The Raspberry Pi graphics stack touches nearly every layer described in this book:

- **[Ch1 (DRM/KMS Architecture)](../part-01-kernel-layer/ch01-drm-architecture.md)**: The `vc4` and `v3d` DRM drivers implement the full DRM driver interface — `drm_driver`, `drm_device`, GEM buffer management, and the DRM render node (`/dev/dri/renderD128`). The HVS display pipeline is registered as a set of `drm_crtc`, `drm_encoder`, and `drm_connector` objects.

- **[Ch2 (KMS Atomic Modeset)](../part-01-kernel-layer/ch02-kms-atomic-modeset.md)**: The vc4 HVS implements atomic modeset via `vc4_hvs_atomic_check()` and `vc4_hvs_atomic_flush()`. The HVS display list is the Pi-specific analogue of a hardware plane state machine; understanding atomic modeset is prerequisite to understanding HVS layer programming.

- **[Ch5 (Mesa Gallium3D Architecture)](../part-04-mesa-architecture/ch05-gallium3d.md)**: Both `vc4` (TGSI-based) and `v3d` (NIR-based) are Gallium3D `pipe_screen` implementations. The Gallium state tracker manages OpenGL ES state; the driver provides the pipe CSO (Compiled Shader Object) and draw call execution.

- **[Ch13 (GPU Memory Management and GEM)](../part-01-kernel-layer/ch13-gpu-memory-gem.md)**: The vc4 driver uses CMA-backed GEM objects (no MMU); the v3d driver uses shmem-backed GEM with its own MMU page tables. Both drivers implement `drm_gem_object_funcs` for mmap, prime export, and reference counting.

- **[Ch14 (NIR: Mesa's Common IR)](../part-04-mesa-architecture/ch14-nir.md)**: The v3d compiler is entirely NIR-based. NIR lowering passes (`v3d_nir_lower_io`, `v3d_nir_lower_txf_ms`, `nir_opt_sink`) adapt the common IR to V3D hardware constraints before VIR emission.

- **[Ch21 (wlroots and the Wayland Compositor Stack)](../part-06-display-stack/ch21-wlroots.md)**: Both labwc (current Pi OS default) and cage (kiosk deployments) are wlroots-based. wlroots's DRM backend uses the vc4/v3d DRM driver for atomic modeset and the v3d GBM backend for EGL surface allocation.

- **[Ch42 (Terminal Graphics: Sixel and Kitty)](../part-12-terminal-graphics/ch42-sixel-kitty.md)**: Terminal pixel graphics protocols (Sixel, Kitty Graphics Protocol) are increasingly used on Pi-based interactive kiosk and educational setups. The rendering path terminates at an OpenGL ES texture blit through the v3d driver.

- **[Ch57 (FFmpeg and the Linux Multimedia Stack)](../part-13-video-streaming/ch57-ffmpeg.md)**: FFmpeg's `v4l2m2m` decoders and encoders use the bcm2835-codec driver. The zero-copy decode→display path (DMA-BUF from V4L2 M2M to EGL image to v3d texture) is the Pi-optimised multimedia playback pipeline.

- **[Ch90 (Panfrost and Lima: Open Embedded GPU Drivers)](../part-02-gpu-drivers/ch90-panfrost-lima.md)**: Panfrost (Mali Midgard/Bifrost) and Lima (Mali Utgard) are the closest architectural peers to vc4/v3d: all are open-source drivers for embedded tile-based GPUs without discrete VRAM. Comparing the tile-binning and ISA designs across these families reveals the design space for embedded tiled renderers.

---

## References

- Broadcom VideoCore IV Architecture Guide: [docs.broadcom.com/doc/12358545](https://docs.broadcom.com/doc/12358545)
- Herman Hermitage, VideoCore IV reverse engineering: [github.com/hermanhermitage/videocoreiv](https://github.com/hermanhermitage/videocoreiv)
- Herman Hermitage, VideoCore IV QPU: [github.com/hermanhermitage/videocoreiv-qpu](https://github.com/hermanhermitage/videocoreiv-qpu)
- Linux kernel vc4 driver: [github.com/torvalds/linux/tree/master/drivers/gpu/drm/vc4](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/vc4)
- Linux kernel v3d driver: [github.com/torvalds/linux/tree/master/drivers/gpu/drm/v3d](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/v3d)
- Kernel documentation, drm/vc4: [docs.kernel.org/gpu/vc4.html](https://docs.kernel.org/gpu/vc4.html)
- Kernel documentation, drm/v3d: [docs.kernel.org/gpu/v3d.html](https://docs.kernel.org/gpu/v3d.html)
- Mesa V3D driver documentation: [docs.mesa3d.org/drivers/v3d.html](https://docs.mesa3d.org/drivers/v3d.html)
- Mesa V3DV Vulkan driver source: [gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/broadcom/vulkan](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/broadcom/vulkan)
- Mesa 24.3 release notes (Vulkan 1.3 for V3DV): [docs.mesa3d.org/relnotes/24.3.0.html](https://docs.mesa3d.org/relnotes/24.3.0.html)
- V3DV Vulkan 1.2 conformance announcement: [raspberrypi.com/news/vulkan-update-version-1-2-conformance-for-raspberry-pi-4](https://www.raspberrypi.com/news/vulkan-update-version-1-2-conformance-for-raspberry-pi-4/)
- Igalia V3D compiler performance blog: [blogs.igalia.com/itoral/2021/03/16/improving-performance-of-the-v3d-compiler-for-opengl-and-vulkan](https://blogs.igalia.com/itoral/2021/03/16/improving-performance-of-the-v3d-compiler-for-opengl-and-vulkan/)
- V3DV Vulkan Mesa driver announcement (Igalia): [igalia.com/2020/10/20/v3dv-becomes-an-official-Vulkan-Mesa-driver.html](https://www.igalia.com/2020/10/20/v3dv-becomes-an-official-Vulkan-Mesa-driver.html)
- Raspberry Pi processor documentation: [raspberrypi.com/documentation/computers/processors.html](https://www.raspberrypi.com/documentation/computers/processors.html)
- RP1 southbridge announcement: [raspberrypi.com/news/rp1-the-silicon-controlling-raspberry-pi-5-i-o-designed-here-at-raspberry-pi](https://www.raspberrypi.com/news/rp1-the-silicon-controlling-raspberry-pi-5-i-o-designed-here-at-raspberry-pi/)
- Raspberry Pi OS Bookworm (Wayfire Wayland default): [raspberrypi.com/news/bookworm-the-new-version-of-raspberry-pi-os](https://www.raspberrypi.com/news/bookworm-the-new-version-of-raspberry-pi-os/)
- Raspberry Pi OS labwc switch (November 2024): [raspberrypi.com/news/a-new-release-of-raspberry-pi-os](https://www.raspberrypi.com/news/a-new-release-of-raspberry-pi-os/)
- libcamera project: [libcamera.org](https://libcamera.org/)
- libcamera Unicam upstream patches (March 2024): [lists.libcamera.org/pipermail/libcamera-devel/2024-March/040711.html](https://lists.libcamera.org/pipermail/libcamera-devel/2024-March/040711.html)
- bcm2835-codec V4L2 driver: [github.com/raspberrypi/linux/blob/rpi-6.6.y/drivers/staging/vc04_services/bcm2835-codec/bcm2835-v4l2-codec.c](https://github.com/raspberrypi/linux/blob/rpi-6.6.y/drivers/staging/vc04_services/bcm2835-codec/bcm2835-v4l2-codec.c)
- Raspberry Pi 5 benchmarking: [raspberrypi.com/news/benchmarking-raspberry-pi-5](https://www.raspberrypi.com/news/benchmarking-raspberry-pi-5/)
- Raspberry Pi config.txt reference: [raspberrypi.com/documentation/computers/config_txt.html](https://www.raspberrypi.com/documentation/computers/config_txt.html)
