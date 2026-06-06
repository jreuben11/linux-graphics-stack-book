# Appendix C: Kernel and Mesa Feature Version Matrix

**Audience**: All readers — systems and driver developers, graphics application developers, and browser and web platform engineers.

This appendix provides a consolidated at-a-glance reference for when major Linux graphics stack features landed in the Linux kernel and in Mesa, giving readers a single source of truth for the question: "what do I actually need to run this?" Feature availability in the Linux graphics stack is split across two independently versioned codebases — a new Mesa release alone is not sufficient if the host kernel is too old to expose the required DRM interfaces — and this split is a persistent source of confusion when diagnosing capability gaps or planning deployment targets. By pairing feature landing versions with the kernel and Mesa versions shipped by major distributions, this appendix bridges the gap between upstream versioning and the real-world systems readers deploy on.

---

## Table of Contents

1. [Introduction and How to Read the Tables](#1-introduction-and-how-to-read-the-tables)
2. [Kernel DRM/KMS Feature Landing Table](#2-kernel-drmkms-feature-landing-table)
3. [Mesa Feature Landing Table](#3-mesa-feature-landing-table)
4. [Feature Cross-Reference: Kernel-Gated Mesa Features](#4-feature-cross-reference-kernel-gated-mesa-features)
5. [Distribution Shipping Versions Reference Table](#5-distribution-shipping-versions-reference-table)
6. [Integrations](#integrations)
7. [References](#references)

---

## 1. Introduction and How to Read the Tables

The Linux graphics stack exposes its capabilities along three independent versioning axes: the **Linux kernel version** (which determines what DRM/KMS interfaces, synchronization primitives, and display properties are available to userspace), the **Mesa version** (which determines what userspace driver logic, shader compilers, and Vulkan/OpenGL extensions are implemented), and, for display-related features, the **display server or compositor version** (which determines whether the software that consumes those kernel and Mesa interfaces actually exercises them). Some features require all three axes to align simultaneously before they are useful.

### Versioning Conventions

Throughout these tables, kernel versions are expressed as `X.Y`, corresponding to the first stable release in which a feature became available for production use. When a feature merged during a `-rc` development cycle, the version attributed is the final stable release (for example, a patch merged in `v5.17-rc1` is listed as `5.17`). Mesa versions are expressed as `X.Y`, omitting the patch-level `.Z` suffix since patch releases contain bug fixes only, not new feature enablement. Where a feature was present but gated behind a runtime environment variable or debug flag before a later release, that later release — when the feature was enabled by default — is used as the "available" version.

### The Kernel-Gated Concept

A critical pattern in this stack is the **kernel-gated feature**: Mesa may fully implement a capability, but that capability either falls back to a slower path or is disabled entirely unless the running kernel exposes the underlying DRM interface. The most common examples are synchronization primitives (where Mesa needs `drm_syncobj` timeline points to implement Vulkan timeline semaphores or the Wayland explicit sync protocol) and display properties (where Mesa or a compositor needs a specific KMS connector or CRTC property to exist before it can expose HDR output, VRR, or wide-gamut color management). Section 4 collects all kernel-gated dependencies in one place for operational convenience.

### Column Definitions

- **Feature**: Canonical short name, plus the upstream identifier (ioctl name, KMS property name, Mesa driver flag, or Vulkan extension name) in monospace where applicable.
- **Kernel**: First stable kernel release containing the production-ready interface. `N/A` when the feature is purely a Mesa/userspace concern.
- **Mesa**: First Mesa release with the implementation enabled by default. `N/A` when Mesa-independent.
- **Notes**: Hardware requirements (e.g., "RDNA2+"), conditional status, or significant caveats.

---

## 2. Kernel DRM/KMS Feature Landing Table

The features below are grouped by functional area. Within each group, entries are ordered chronologically by kernel version.

### 2.1 Display Connectivity

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| DRI3 (`DRI3Open`, `DRI3PixmapFromBuffers`) | 3.15 | N/A | Requires X.Org Server 1.15+. Replaces DRI2 shared-memory path; passes GPU buffer fds directly between GL client and X server. |

**DRI3** was introduced in Linux v3.15 and X.Org Server 1.15 as the successor to DRI2. Under DRI2, the display server and GL client shared memory segments, requiring a copy whenever a rendered buffer was handed off. DRI3 allows the client to pass a DMA buffer file descriptor directly to the server, completely eliminating the copy. This is the foundation on which zero-copy rendering in X11 applications rests. While DRI3 itself is a protocol between userspace components, the kernel side simply needs to expose the DRM prime (dma-buf) file descriptor infrastructure present since v3.5; v3.15 is the version at which X.Org adopted DRI3 as its default path.

### 2.2 Display Pipeline

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| Atomic modesetting (KMS atomic) | 4.2 | N/A | Prerequisite for all modern compositors: Weston, Mutter, KWin, sway. Replaces the legacy `set_crtc` single-call path. |
| DRM format modifiers (`DRM_FORMAT_MOD_*`) | 4.14 | N/A | Standardises vendor tiling/compression layout descriptors for dma-buf cross-device buffer sharing. |
| VRR/FreeSync (`VRR_ENABLED` KMS property) | 5.0 | N/A | Adds `VRR_ENABLED` connector property; driver-side CRTC logic to drive variable refresh via EDID VRR or FreeSync metadata. |
| HDR output metadata (`HDR_OUTPUT_METADATA` KMS property) | 5.2 | N/A | Adds `HDR_OUTPUT_METADATA` blob property on connectors. Required for OS-level HDR pipeline. Available on amdgpu from v5.2; standardised connector property from v5.17. |
| KMS color management pipeline (per-plane/CRTC LUT, CTM) | 6.8–6.12 | N/A | Extended KMS properties for compositor-driven wide-gamut and HDR color management. First plane-level properties landed in 6.8; baseline usability for distro adoption reached approximately 6.12. |

**KMS atomic modesetting** merges the previously separate `set_crtc`, `set_plane`, and `set_property` codepaths into a single atomic transaction that either commits entirely or rolls back. The kernel-side infrastructure appeared in v3.19 via `drm_atomic.h`, but the flag `DRIVER_ATOMIC` was set on all atomic-capable drivers — including i915, amdgpu, and vc4 — by v4.2, making that the practical availability version for compositor authors. All modern Linux compositors (Mutter for GNOME, KWin for KDE Plasma, sway for wlroots-based environments, Weston as the reference implementation) require atomic modesetting for reliable operation of VRR, overlay planes, and non-blocking commit pipelines. [Source: `include/drm/drm_atomic.h`, `Documentation/gpu/drm-kms.rst`](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html)

**DRM format modifiers** solve a problem that emerges in cross-device buffer sharing: two subsystems may agree on a pixel format (e.g., `DRM_FORMAT_ARGB8888`) but disagree on the memory layout, because GPUs tile framebuffers and apply proprietary lossless compression schemes. The modifier is a 64-bit value whose upper 8 bits identify the hardware vendor and whose lower 56 bits encode the exact layout. Without modifiers, dma-buf sharing can only safely use a linear (uncompressed, row-major) layout, sacrificing the memory-bandwidth savings of tiling. With modifiers, the producer of a buffer can advertise its exact layout and the consumer can either accept it or negotiate a fallback. The kernel infrastructure was added in v4.14 through the `DRM_FORMAT_MOD_*` constants in `include/uapi/drm/drm_fourcc.h`.

**HDR output metadata** was first implemented on amdgpu connectors in v5.2 but was extended to a generic KMS connector blob property in v5.17. [Source: phoronix.com/news/Linux-5.3-AMDGPU-HDR](https://www.phoronix.com/news/Linux-5.3-AMDGPU-HDR). The `HDR_OUTPUT_METADATA` blob contains HDR10 static metadata (maximum luminance, colour primaries, transfer function indicator) that the display controller forwards to the panel via HDMI or DisplayPort InfoFrames. A compositor that wants to signal HDR content to the display must set this property; without it the display applies its default tone mapping.

### 2.3 Synchronization

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| Explicit sync objects (`drm_syncobj`, `DRM_IOCTL_SYNCOBJ_*`) | 4.12 | N/A | Introduces GPU sync object primitives for explicit GPU-CPU and GPU-GPU synchronization; foundation for Vulkan semaphore import/export and timeline semaphores. |
| In-fence / out-fence on atomic commits | 4.13 | N/A | Attaches DMA fences to atomic KMS commits; enables compositor to pipeline GPU render with scanout without CPU-side `poll()` or busy-wait. |
| `drm_syncobj` timeline points | 5.2 | N/A | Extends `drm_syncobj` to support timeline (monotonically increasing) signalling; required for Vulkan timeline semaphores and for the Wayland `wp_linux_drm_syncobj_v1` explicit sync protocol. |
| Wayland explicit sync (`wp_linux_drm_syncobj_v1` kernel side) | 6.6 / 6.8 | N/A | Timeline syncobj are the kernel mechanism required by the protocol. The protocol itself requires kernel ≥ 6.6 for the timeline point infrastructure, but stable production use requires ≥ 6.8 for bug-fixing commits addressing races. |
| ntsync (NT Synchronization Primitives driver) | 6.14 | N/A | Provides keyed events, mutexes, semaphores matching Windows NT semantics for Wine/Proton. The device node appeared partially in 6.10 but was marked `BROKEN`; fully enabled in 6.14 and enabled by default in Wine 11.0 / Proton 11.0. |

**`drm_syncobj`** was introduced in v4.12 as a new DRM-level synchronization primitive that userspace can create, signal, wait on, and pass across process boundaries. Before `drm_syncobj`, GPU-to-CPU synchronization relied on implicit DMA fences embedded in GEM buffer objects, which are invisible to Vulkan-style APIs that expect explicit synchronization control. The `DRM_IOCTL_SYNCOBJ_CREATE`, `DRM_IOCTL_SYNCOBJ_WAIT`, and `DRM_IOCTL_SYNCOBJ_SIGNAL` ioctls in `include/uapi/drm/drm.h` form the base API. Vulkan drivers (RADV, ANV, turnip) map `VkSemaphore` and `VkFence` objects onto `drm_syncobj` handles. [Source: `drivers/gpu/drm/drm_syncobj.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c)

**Timeline points** (v5.2) extend `drm_syncobj` from a binary signal/reset model to a monotonically increasing 64-bit counter, directly matching the semantics of Vulkan timeline semaphores (`VkSemaphoreTypeTimeline`). This extension also provides the kernel mechanism required by the Wayland `wp_linux_drm_syncobj_v1` protocol, in which a Wayland client passes an acquire point (a timeline value the compositor must wait for before scanning a buffer) and a release point (a value the compositor signals when it finishes reading the buffer). The net effect is that the compositor never needs to insert a CPU-side fence wait between the client's GPU render completion and the scanout, eliminating the source of stuttering and tearing on multi-GPU or multi-queue setups.

**ntsync** provides a kernel-level emulation of Windows NT synchronization primitives (mutexes, semaphores, keyed events) that Wine and Proton use to synchronise Windows game threads. Before ntsync, Wine used `esync` (eventfd-based) or `fsync` (futex-based) userspace implementations that required busy-spinning or large numbers of eventfds for highly contended workloads. The ntsync device (`/dev/ntsync`) first appeared in v6.10 but was compiled with `BROKEN` so it would not build in default configurations. The complete, production-ready driver was merged for v6.14 and adopted as the default synchronization backend in Wine 11.0 and Proton 11.0. [Source: GamingOnLinux — NTSYNC lands in Linux 6.14](https://www.gamingonlinux.com/2025/01/ntsync-for-proton-wine-now-in-linux-kernel-6-14-that-should-make-many-steamos-users-happy/)

### 2.4 Infrastructure

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| DRM GPU scheduler (`drm_sched`, `gpu_scheduler` module) | 4.15 | N/A | Generic driver-agnostic work queue and priority scheduling for GPU command submission. Adopted by amdgpu, panfrost, lima, etnaviv, v3d, and others. |

**The DRM GPU scheduler** (`drivers/gpu/drm/scheduler/`) provides a reusable kernel-side abstraction for queuing GPU jobs, prioritising them across multiple hardware queues, and handling reset/recovery. Before the generic scheduler, each driver that needed job scheduling (amdgpu, radeon) maintained its own ad-hoc queue. The scheduler exposes `drm_sched_job`, `drm_sched_entity`, and `drm_sched_rq` structures that represent individual GPU operations, the software context submitting them, and the hardware ring they target, respectively. Priority scheduling (high, normal, low, kernel) was added subsequently in v5.x. The scheduler is now the standard submission mechanism in virtually all DRM drivers that manage GPU command rings. [Source: `drivers/gpu/drm/scheduler/sched_main.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/scheduler/sched_main.c)

---

## 3. Mesa Feature Landing Table

The features below are grouped by functional area: compiler infrastructure, drivers and extensions, new drivers/frontends, and integration libraries.

### 3.1 Compiler Infrastructure

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| NIR as default IR (replaces TGSI in all major drivers) | N/A | 20.0 | NIR development began in Mesa 13; Mesa 20.0 completed the transition to NIR as the primary IR for radeonsi, r600, iris, nouveau, and others. TGSI retained for legacy paths only. |
| ACO compiler backend for RADV (graphics shaders) | N/A | 20.2 | Valve-funded backend replacing LLVM for the RADV Vulkan driver's graphics pipeline. Significantly faster compile times and competitive codegen. Opt-in via `RADV_PERFTEST=aco` in Mesa 19.3–20.1. |
| ACO backend for compute shaders (RADV) | N/A | 22.1 | ACO coverage extended from graphics to compute shaders; previously RADV compute used LLVM even when graphics used ACO. |

**NIR (New IR)** was conceived as a replacement for TGSI (Trivial Shader IR) to provide a more structured, SSA-based intermediate representation better suited to modern optimisation passes and backends. NIR development began around Mesa 13.0 with initial adoption in i965 and vc4. By Mesa 20.0, all major drivers had completed their migration: radeonsi, r600, iris (Intel), nouveau, and others. TGSI paths remain in the tree for historical compatibility but receive no active development. NIR is now the universal IR through which shaders pass on their way from GLSL/SPIR-V parsing to backend code generation. [Source: `src/compiler/nir/` in Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/compiler/nir)

**ACO (AMD Compiler Open)** was initially proposed in a Mesa-dev RFC in July 2019 by Valve engineers who needed faster shader compilation for games. The backend compiles NIR directly to AMD ISA without going through LLVM IR, removing the overhead of LLVM's general-purpose optimisation pipeline. ACO became opt-in via `RADV_PERFTEST=aco` in Mesa 19.3 and was promoted to the default in Mesa 20.2. Compile times for typical game shaders improved by a factor of two to five compared to the LLVM path, and generated code quality was competitive with or superior to LLVM on most workloads. [Source: Mesa 20.2 release notes; Phoronix](https://www.phoronix.com/news/Mesa-20.2-Released)

### 3.2 RADV Driver Extensions

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| Ray tracing in RADV (`VK_KHR_ray_tracing_pipeline`, `VK_KHR_acceleration_structure`) | N/A | 21.3 | Experimental; requires RDNA2+ (GFX 10.3+, RX 6000 series). Opt-in via `RADV_PERFTEST=rt` in 21.3; promoted to non-experimental in subsequent releases. |
| Mesh shaders in RADV (`VK_EXT_mesh_shader`) | N/A | 22.2 | RDNA2+ only. Replaces the traditional vertex/geometry pipeline with a compute-like programmable geometry amplification stage. |
| Graphics Pipeline Library in RADV (`VK_EXT_graphics_pipeline_library`) | N/A | 23.0 | Allows partial pipeline pre-compilation and link-time assembly; reduces pipeline stutter in games by deferring final linking to draw time. |
| Vulkan Video decode in RADV (`VK_KHR_video_decode_queue`, H.264/H.265) | N/A | 23.1 | Hardware-accelerated video decode via Vulkan API on RDNA hardware. Initial implementation covers H.264 (`VK_EXT_video_decode_h264`) and H.265 (`VK_EXT_video_decode_h265`). AV1 decode added in Mesa 24.1. |

**Ray tracing in RADV** (Mesa 21.3) landed as an experimental feature enabled via the `RADV_PERFTEST=rt` environment variable. The implementation covers `VK_KHR_acceleration_structure` (the BLAS/TLAS construction pipeline) and `VK_KHR_ray_tracing_pipeline` (the ray generation, closest-hit, miss, and any-hit shader stage). At launch the implementation was functional but not highly optimised; the shader binding table handling and traversal shader compilation have been progressively improved in subsequent releases. Hardware support is limited to RDNA2 (Navi 21, RX 6000 series) and later, as RDNA1 lacks the dedicated BVH traversal hardware. [Source: Phoronix — Mesa 21.3 RADV Ray-Tracing](https://www.phoronix.com/news/Mesa-21.3-RADV-Ray-Tracing)

**Graphics Pipeline Library (GPL)** solves a well-known problem in Vulkan application performance: pipeline state objects must be fully compiled to GPU machine code before first use, and this compilation can take hundreds of milliseconds per pipeline, causing visible frame-rate hitches ("shader compilation stutter") during gameplay. `VK_EXT_graphics_pipeline_library` allows an application (or driver) to split a pipeline into independently compiled parts (vertex input, pre-rasterisation, fragment output, and fragment shader), compile each asynchronously, and link them at draw time with minimal linking overhead. Mesa 23.0 introduced the RADV implementation of GPL, which has since become a critical path for game compatibility through VKD3D-Proton. [Source: docs.mesa3d.org/drivers/radv.html](https://docs.mesa3d.org/drivers/radv.html)

**Vulkan Video decode** (Mesa 23.1) implemented hardware-accelerated decode of H.264 and H.265 video streams through the Khronos Vulkan Video extension family. The RADV implementation targets RDNA-class hardware (TONGA through NAVI2x at launch) and exposes `VK_KHR_video_queue`, `VK_KHR_video_decode_queue`, `VK_EXT_video_decode_h264`, and `VK_EXT_video_decode_h265`. AV1 hardware decode (`VK_KHR_video_decode_av1`) followed in Mesa 24.1. [Source: airlied.blogspot.com/2022/12/vulkan-video-decoding-radv-status.html](https://airlied.blogspot.com/2022/12/vulkan-video-decoding-radv-status.html)

### 3.3 NVK (NVIDIA Open Vulkan Driver)

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| NVK (non-conformant, merged into Mesa) | Requires kernel ≥ 6.7 (nouveau GSP) | 23.3 | Merged into Mesa 23.3 as an experimental, non-conformant Vulkan driver. Requires the NVIDIA open GPU kernel modules (`nvidia.ko` from NVIDIA open-gpu-kernel-modules or the in-tree nouveau with GSP firmware). |
| NVK (Vulkan 1.3 conformant, built by default) | Requires kernel ≥ 6.7 | 24.1 | Declared Vulkan 1.3 conformant; built by default on x86/x86_64. Covers Turing (RTX 20xx) through Ada Lovelace (RTX 40xx) and later. OpenGL 4.6 conformant when paired with Zink. |
| NVK (Vulkan 1.4 conformant, Maxwell/Pascal/Volta) | Requires kernel ≥ 6.7 | 25.x | Vulkan 1.4 conformance extended back to Maxwell, Pascal, and Volta microarchitectures as of early 2025. |

**NVK** is the first fully open-source Vulkan driver for NVIDIA GPU hardware. It lives in the Mesa repository at `src/nouveau/vulkan/` and shares NIR-based compiler infrastructure with the rest of Mesa. The driver depends on the NVIDIA GSP (Graphics System Processor) firmware path in the kernel, which was stabilised in the nouveau driver for v6.7. Mesa 24.1 represents the milestone at which NVK moved from experimental to production-ready: it passed the Vulkan 1.3 conformance test suite (CTS) and was enabled in default builds. [Source: Phoronix — Mesa NVK Vulkan 1.3 Conformant](https://www.phoronix.com/news/Mesa-NVK-Vulkan-Conformant); [Source: docs.mesa3d.org/drivers/nvk.html](https://docs.mesa3d.org/drivers/nvk.html)

### 3.4 New Drivers and Frontends

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| Lavapipe (software CPU Vulkan, `VK_DRIVER_ID_MESA_LLVMPIPE`) | N/A | 21.0 | CPU-based Vulkan 1.1 implementation (later Vulkan 1.3 conformant) built on LLVMpipe Gallium infrastructure. Used for CI, headless testing, and GPU-less fallback. |
| Zink (OpenGL-over-Vulkan Gallium driver) | N/A | 21.0 | Gallium driver that translates OpenGL calls to Vulkan; enables OpenGL on Vulkan-only devices. Full OpenGL 4.6 + ES 3.2 reached in Mesa 21.3. Replaces legacy Nouveau GL on NVIDIA as of Mesa 25.1. |
| rusticl (OpenCL 3.0 state tracker in Rust) | N/A | 22.3 | Rust-written OpenCL 3.0 state tracker targeting NIR-based Gallium drivers. Replaces the abandoned Clover. Clover was removed entirely in Mesa 25.2. |

**Lavapipe** (originally called "Vallium" in early development) is Mesa's software Vulkan rasteriser, built on the same LLVMpipe infrastructure that powers software OpenGL. It compiles Vulkan shaders via NIR and LLVM to native host CPU instructions, making it useful for continuous integration (where GPU hardware is unavailable), headless compute, and as a ground-truth reference implementation for debugging. Lavapipe shipped as part of Mesa 21.0 and subsequently achieved Vulkan 1.2 conformance in 2022 and Vulkan 1.3 conformance thereafter. [Source: Mesa (computer graphics) — Wikipedia](https://en.wikipedia.org/wiki/Mesa_(computer_graphics))

**Zink** translates the OpenGL API to Vulkan at the Gallium state tracker level. Each GL draw call, texture bind, or render target switch is translated to a corresponding sequence of Vulkan commands directed at whatever Vulkan driver is present on the system. This makes OpenGL available on devices that only expose a Vulkan driver (useful for NVK and, in the future, for new GPU drivers that only implement Vulkan). As of Mesa 25.1, Zink+NVK became the default OpenGL path for NVIDIA hardware on nouveau, replacing the older Nouveau Gallium GL driver. [Source: 9to5linux — Mesa 25.1 to replace Nouveau driver with Zink/NVK](https://9to5linux.com/mesa-25-1-to-replace-nouveau-driver-with-zink-nvk-by-default-for-nvidia-gpus)

**rusticl** was introduced in Mesa 22.3 as a replacement for Clover, the long-stagnant OpenCL Gallium state tracker. rusticl is written in Rust and targets OpenCL 3.0 conformance, reusing the NIR compiler infrastructure for kernel compilation and the Gallium pipe interface for device abstraction. Clover had not seen active development for several years and lacked support for modern compute features. The Clover code was fully removed from the Mesa tree in Mesa 25.2. [Source: nullr0ute.com — Getting started with OpenCL using Mesa/rusticl](https://nullr0ute.com/2023/12/getting-started-with-opencl-using-mesa-rusticl/)

### 3.5 Integration

| Feature | Kernel | Mesa | Notes |
|---|---|---|---|
| GLVND support (`libGL` dispatch via `libglvnd`) | N/A | 18.x | Integration with the GL Vendor-Neutral Dispatch library; allows Mesa and proprietary drivers to coexist on the same system. `MESA_LOADER_DRIVER_OVERRIDE` becomes available. |

**GLVND (GL Vendor-Neutral Dispatch)** solves the longstanding problem on Linux where installing a proprietary OpenGL driver would overwrite or conflict with Mesa's `libGL.so`. With GLVND, there is a single `/usr/lib/libGL.so` that dispatches to the correct vendor implementation at runtime. Mesa integrated GLVND support in the Mesa 18.x era; most distributions now ship Mesa against GLVND by default. [Source: Mesa (computer graphics) — Wikipedia](https://en.wikipedia.org/wiki/Mesa_(computer_graphics))

---

## 4. Feature Cross-Reference: Kernel-Gated Mesa Features

This section is the most operationally critical for deployment planning. Each entry documents a case where a Mesa feature requires a specific minimum kernel version to function, even if the Mesa version is new enough. Where applicable, the fallback behaviour when the kernel requirement is not met is noted.

> **Reading note**: In each entry below, the pattern is: "Even if you install Mesa X.Y, you need kernel Y.Z for this feature to activate."

| Mesa Feature | Minimum Kernel Required | Reason | Fallback When Kernel Is Too Old |
|---|---|---|---|
| Atomic compositor (Mutter, KWin, sway, Weston) | 4.2 | KMS atomic API (`DRIVER_ATOMIC`). Without it, compositors cannot perform non-blocking commits, overlay planes, or reliable VRR. | Compositor may fall back to legacy modeset (missing VRR, overlays, and non-blocking commits) or fail to start on some drivers. |
| Vulkan semaphore import/export (`VK_KHR_external_semaphore_fd`) | 4.12 | `drm_syncobj` backing for `VkSemaphore`/`VkFence`. Without `drm_syncobj`, drivers cannot implement interop semaphores. | Vulkan drivers disable the `VK_KHR_external_semaphore_fd` and `VK_KHR_external_fence_fd` extensions. |
| In-fence/out-fence compositor pipelining | 4.13 | DMA fences on atomic commits. Without in-fences, the compositor must insert a CPU-side wait before each commit. | Compositor inserts synchronous wait; noticeable latency increase, especially under load. |
| DRM format modifier negotiation (dma-buf sharing) | 4.14 | `DRM_FORMAT_MOD_*` constants and AddFB2 modifier support. Without modifiers, cross-device buffer sharing is restricted to linear layouts. | Buffers shared between GPU and display engine use linear (uncompressed) layout; memory bandwidth increases significantly. |
| VRR/FreeSync in compositor | 5.0 | `VRR_ENABLED` KMS connector property. The compositor sets this property to enable variable refresh; the kernel must expose it. | VRR silently unavailable regardless of Mesa or compositor version. |
| Vulkan timeline semaphores (`VkSemaphoreTypeTimeline`) | 5.2 | `drm_syncobj` timeline points. Required for Vulkan 1.2 timeline semaphore semantics. | Vulkan driver reports `timelineSemaphore` feature as unsupported; applications relying on `VK_SEMAPHORE_TYPE_TIMELINE` will fail feature checks. |
| HDR compositing pipeline | 5.2 / 5.17 | `HDR_OUTPUT_METADATA` KMS connector blob property. Compositors must set this property to push HDR10 metadata to the display. First amdgpu-specific in 5.2; generic KMS property standardised in 5.17. | Display receives no HDR metadata InfoFrame; applies default SDR or display-internal HDR tone mapping rather than OS-directed mapping. |
| NVK driver (NVIDIA open Vulkan) | 6.7 (nouveau with GSP firmware) | NVK requires the nouveau kernel driver to be using the NVIDIA GSP (Graphics System Processor) firmware interface, which became stable in 6.7. | NVK does not initialise; `VK_ERROR_INCOMPATIBLE_DRIVER` or driver load failure at `vkCreateInstance`. |
| Wayland explicit sync (`wp_linux_drm_syncobj_v1`) | 6.6 (minimum); 6.8 (stable) | Protocol passes `drm_syncobj` timeline points from Wayland client to compositor. Kernel 6.6 contains the timeline point infrastructure; kernel 6.8 contains critical bug-fixing commits resolving race conditions. | Compositor and clients fall back to implicit sync. On multi-GPU (e.g., iGPU display + dGPU render) setups, implicit sync can cause tearing, dropped frames, or corruption. |
| ntsync (Wine/Proton NT synchronization) | 6.14 | `ntsync` kernel module (`/dev/ntsync` device). Partial implementation in 6.10 was marked `BROKEN` and not compiled in default configs. | Wine/Proton automatically detect absence of `/dev/ntsync` and fall back to `fsync` (futex-based) or `esync` (eventfd-based) implementations. These are functional but less efficient under heavy contention. |

---

## 5. Distribution Shipping Versions Reference Table

This section maps each major distribution release to the kernel and Mesa versions it ships at general availability (GA), then derives which features from Sections 2 and 3 are available out of the box. Note that distributions typically receive security patch updates to the shipped kernel version throughout their support lifetime, but the kernel minor version (and thus the feature set) changes only through HWE/backport/upgrade paths noted below.

### Summary Table

| Distribution | GA Kernel | GA Mesa | Notable Available Features | Notable Missing Features (at GA) |
|---|---|---|---|---|
| Ubuntu 22.04 LTS (Jammy) | 5.15 | 22.0 | Atomic KMS, DRI3, format modifiers, VRR, `drm_syncobj`, timeline semaphores, DRM GPU scheduler, ACO (RADV default), NIR, Lavapipe, Zink, GLVND | HDR metadata (needs 5.17+), explicit sync protocol (needs 6.6+/6.8+), ntsync (needs 6.14+), NVK (needs 6.7+), GPL in RADV (needs Mesa 23.0+), Vulkan Video (needs Mesa 23.1+) |
| Ubuntu 24.04 LTS (Noble) | 6.8 | 24.0 | All above + HDR metadata, NVK (conformant), GPL in RADV, Vulkan Video | Explicit sync protocol (needs 6.8+ — GA kernel satisfies minimum but see note), ntsync (needs 6.14+) |
| Fedora 38 | 6.2 | 23.0 | Atomic KMS, format modifiers, VRR, `drm_syncobj`, timeline semaphores, ACO, NIR, Lavapipe, Zink, GPL in RADV (first release), RADV ray tracing | HDR metadata fully available (5.17+: yes), explicit sync (needs 6.6+: no), ntsync (needs 6.14+: no), NVK (needs 6.7+: no) |
| Fedora 39 | 6.5 | 23.2 | All Fedora 38 features + HDR metadata stable, RADV mesh shaders | Explicit sync (needs 6.6+: no), ntsync (needs 6.14+: no) |
| Fedora 40 | 6.8 | 24.0 | All above + explicit sync (kernel 6.8 satisfies stable requirement), NVK conformant, Vulkan Video with AV1 (Mesa 24.1) | ntsync (needs 6.14+: no) |
| Debian 12 (Bookworm) | 6.1 | 22.3.6 | Atomic KMS, format modifiers, VRR, `drm_syncobj`, ACO, NIR, Lavapipe, Zink, rusticl (first release), RADV ray tracing | HDR metadata (6.1 kernel: present at 5.2+ so available), explicit sync (needs 6.6+/6.8+: no), GPL in RADV (needs Mesa 23.0+: no), Vulkan Video (needs Mesa 23.1+: no), ntsync (needs 6.14+: no), NVK (needs 6.7+: no) |
| SteamOS 3.5/3.6 | 6.1 / 6.5 | 23.x / 24.1 | Full RADV feature set including ACO, GPL, ray tracing; Valve-patched kernel with additional gaming optimizations | ntsync in 6.1/6.5 era (N/A); added in SteamOS 3.7.x era with kernel 6.11+ |
| SteamOS 3.7.x | 6.11 | 24.x | All prior features; ntsync available at 6.14 only — 6.11 still lacks it | ntsync (ships in 6.14; 6.11 has fsync fallback) |

### Per-Distribution Notes

**Ubuntu 22.04 LTS (Jammy Xerus)** shipped with kernel 5.15 LTS and Mesa 22.0 at GA in April 2022. The 5.15 kernel fully supports the VRR/FreeSync `VRR_ENABLED` property (5.0+), `drm_syncobj` and timeline points (4.12 / 5.2), and DRM format modifiers (4.14). It ships Mesa 22.0, which includes ACO as the default RADV compiler backend, NIR as the universal IR, Lavapipe, Zink, and GLVND. However, 22.04 GA misses HDR output metadata (requires 5.17; arriving with the HWE kernel), the `wp_linux_drm_syncobj_v1` Wayland explicit sync protocol (requires 6.6/6.8), and ntsync (requires 6.14). Users who need newer kernel features on 22.04 can enable the **HWE (Hardware Enablement) stack**, which tracks a newer kernel: the 22.04 HWE eventually tracks kernel 6.5, giving HDR metadata support, but still falls short of the explicit sync requirement (6.8). For newer Mesa, the community **oibaf** and **kisak** PPAs provide Mesa packages backported to Ubuntu LTS. [Package tracker: packages.ubuntu.com/jammy/mesa-vulkan-drivers](https://packages.ubuntu.com/jammy/mesa-vulkan-drivers)

**Ubuntu 24.04 LTS (Noble Numbat)** shipped with kernel 6.8 and Mesa 24.0 at GA in April 2024. The 6.8 kernel satisfies the stable requirement for Wayland explicit sync (kernel ≥ 6.8 for bug-fix commits) and provides HDR output metadata, `drm_syncobj` timeline points, and DRM GPU scheduler. Mesa 24.0 ships with NVK as a non-experimental (though not yet default-built) Vulkan driver, RADV GPL, and Vulkan Video H.264/H.265. The point release 24.04.2 (shipped with kernel 6.11) and later 24.04.3 (kernel 6.14) bring ntsync support to users on the HWE track. Note that Ubuntu 24.04 GA Mesa 24.0 does not include the Vulkan Video AV1 decode that arrived in Mesa 24.1; the oibaf/kisak PPAs or Flatpak runtimes can provide this. [Release notes: documentation.ubuntu.com/release-notes/24.04/](https://documentation.ubuntu.com/release-notes/24.04/)

**Fedora 38** shipped with kernel 6.2 and Mesa 23.0 in April 2023. This is the first Fedora release to include RADV's `VK_EXT_graphics_pipeline_library` implementation (Mesa 23.0), which substantially reduces pipeline compilation stutter in Vulkan games through VKD3D-Proton. The 6.2 kernel misses Wayland explicit sync (requires 6.6+) and ntsync (requires 6.14). Fedora does not hold Mesa at GA; it receives Mesa updates through COPR repositories and upstream package updates throughout the release lifetime. [Package tracker: packages.fedoraproject.org/pkgs/mesa](https://packages.fedoraproject.org/pkgs/mesa)

**Fedora 39** shipped with kernel 6.5 and Mesa 23.2 in November 2023. The 6.5 kernel provides VRR (5.0+), HDR metadata (5.17+), timeline syncobj (5.2+), and the initial plumbing for explicit sync (6.5 is below 6.6 minimum; the protocol requires 6.6). ntsync remains unavailable. Mesa 23.2 includes RADV mesh shaders (`VK_EXT_mesh_shader`) and improvements to the ray tracing pipeline.

**Fedora 40** shipped with kernel 6.8 and Mesa 24.0.5 in April 2024, including experimental HDR support in GNOME Plasma. The 6.8 kernel satisfies both the minimum explicit sync kernel requirement and the stable requirement. ntsync remains absent until Fedora 41 era (kernel 6.14). [GamingOnLinux — Fedora Linux 40 is officially out](https://www.gamingonlinux.com/2024/04/fedora-linux-40-is-officially-out-now/)

**Debian 12 (Bookworm)** shipped in June 2023 with kernel 6.1 LTS and Mesa 22.3.6. Despite being a modern Debian release, the combination presents significant gaps for advanced graphics use cases. The 6.1 kernel predates the explicit sync timeline (requires 6.6/6.8) and ntsync (requires 6.14). Mesa 22.3 misses RADV GPL (requires Mesa 23.0) and Vulkan Video decode (requires Mesa 23.1). The `debian-backports` repository provides newer Mesa packages, and users can install `linux-image-amd64` from backports to obtain a newer kernel (kernel 6.6 LTS is available in Debian backports). [Package tracker: packages.debian.org/source/bookworm/mesa](https://packages.debian.org/source/bookworm/mesa)

**SteamOS 3.x (Holo)** is Valve's gaming-focused distribution for the Steam Deck and other devices, based on Arch Linux. Unlike the distributions above, SteamOS does not hold Mesa at a fixed version; Mesa is updated aggressively as new releases become available. SteamOS 3.5 shipped with kernel 6.1 (Valve-patched with gaming-specific additions) and Mesa 23.x. SteamOS 3.6 (preview) moved to kernel 6.5 and Mesa 24.1 — the release that includes NVK as conformant. SteamOS 3.7.x (stable 2024–2025) runs kernel 6.11. The ntsync driver, required for full Wine 11.0 / Proton 11 performance, requires kernel 6.14; Valve enabled ntsync in SteamOS 3.7.20 Beta, corresponding to a Valve-patched kernel 6.14. Users on fsync/esync fallback lose ntsync's performance advantage but retain full functionality. [SteamOS Getting Started wiki](https://github.com/ValveSoftware/SteamOS/wiki/Getting-Started)

### Notable Gaps Summary

The following gaps are the most commonly encountered in real-world deployments:

1. **Ubuntu 22.04 + GA kernel (5.15)**: VRR is available, HDR metadata is not (requires 5.17). Even with the HWE kernel upgraded to 6.5, explicit sync (requires 6.8) and ntsync (requires 6.14) remain unavailable without a full kernel version upgrade outside the HWE track.

2. **Debian 12 (kernel 6.1, Mesa 22.3)**: HDR metadata property is present (5.2+) but explicit sync protocol, RADV GPL, Vulkan Video, NVK, and ntsync are all absent. Upgrading Mesa via backports provides GPL and Vulkan Video but cannot provide explicit sync or ntsync without a kernel upgrade.

3. **Any system on kernel 6.10 expecting ntsync**: The `ntsync` driver was merged with `BROKEN` status in 6.10 and does not build in default configurations. Wine and Proton on 6.10 automatically fall back to fsync. Only 6.14+ provides a fully operational ntsync driver.

4. **NVK on any kernel below 6.7**: The NVIDIA GSP firmware path in the nouveau driver stabilised in 6.7. Mesa 24.1 builds NVK by default but NVK will fail to initialise on kernels below 6.7.

---

## Integrations

This appendix cross-references the following chapters:

- **Chapter 1 (DRM Architecture)** — The DRM ioctl namespace (`DRM_IOCTL_*`), GEM object lifecycle, and driver capability flags discussed there provide the implementation context for the kernel-side features tabulated in Section 2.
- **Chapter 4 (KMS and Atomic Modesetting)** — The `drm_atomic.h` API, plane composition, and the non-blocking commit pipeline are the implementation layer behind the atomic modesetting and in-fence/out-fence entries in Section 2.
- **Chapter 5 (DMA-BUF and Explicit Sync)** — The `drm_syncobj` timeline points and `wp_linux_drm_syncobj_v1` protocol entries in Section 2 and Section 4 correspond directly to the kernel-side and Wayland-side APIs detailed in Chapter 5.
- **Chapter 9 (Mesa Compiler Stack: NIR and ACO)** — The NIR and ACO entries in Section 3 are the high-level summary of the compiler architecture covered in depth there.
- **Chapter 12 (RADV: AMD Vulkan Driver)** — The RADV ray tracing, mesh shader, GPL, and Vulkan Video entries in Section 3 correspond to the RADV feature implementation covered in Chapter 12.
- **Chapter 14 (NVK: NVIDIA Open Vulkan Driver)** — The NVK entries in Section 3 and the kernel-gated GSP firmware requirement in Section 4 are the version-matrix view of the architecture described in Chapter 14.
- **Chapter 18 (OpenCL and Compute on Linux)** — The rusticl and GLVND entries in Section 3 correspond to the compute stack architecture described there.
- **Chapter 22 (Wayland Compositing and Protocols)** — The `wp_linux_drm_syncobj_v1` protocol entry and the VRR/HDR display property entries in Sections 2 and 4 map to the compositor integration topics covered in Chapter 22.
- **Appendix A (Glossary)** — Technical terms used throughout this appendix (NIR, ACO, TGSI, drm_syncobj, GLVND, GPL, GSP) are defined in the glossary.
- **Appendix B (Build and Test Environment Setup)** — Appendix B covers how to build Mesa and kernel development versions that expose features ahead of what distributions ship.

---

## References

1. Linux Kernel DRM/KMS Documentation — https://www.kernel.org/doc/html/latest/gpu/drm-kms.html
2. Linux Kernel DRM UAPI Reference — https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html
3. Mesa 3D Documentation Index — https://docs.mesa3d.org/
4. Mesa RADV Driver Documentation — https://docs.mesa3d.org/drivers/radv.html
5. Mesa NVK Driver Documentation — https://docs.mesa3d.org/drivers/nvk.html
6. Mesa Zink Driver Documentation — https://docs.mesa3d.org/drivers/zink.html
7. Mesa LLVMpipe/Lavapipe Documentation — https://docs.mesa3d.org/drivers/llvmpipe.html
8. `wp_linux_drm_syncobj_v1` Wayland Protocol — https://wayland.app/protocols/linux-drm-syncobj-v1
9. KWin Wayland Explicit Sync MR — https://invent.kde.org/plasma/kwin/-/merge_requests/4693
10. Phoronix — Mesa 20.2 Released With RADV ACO By Default — https://www.phoronix.com/news/Mesa-20.2-Released
11. Phoronix — Mesa NVK Vulkan Driver Vulkan 1.3 Conformant — https://www.phoronix.com/news/Mesa-NVK-Vulkan-Conformant
12. Phoronix — Mesa 21.3 RADV Ray-Tracing — https://www.phoronix.com/news/Mesa-21.3-RADV-Ray-Tracing
13. GamingOnLinux — ntsync in Linux 6.14 — https://www.gamingonlinux.com/2025/01/ntsync-for-proton-wine-now-in-linux-kernel-6-14-that-should-make-many-steamos-users-happy/
14. GamingOnLinux — Mesa NVK now Vulkan 1.4 conformant on Maxwell/Pascal/Volta — https://www.gamingonlinux.com/2025/04/mesa-nvk-nvidia-vulkan-driver-now-vulkan-1-4-conformant-on-maxwell-pascal-and-volta-gpus/
15. GamingOnLinux — Fedora 40 is out — https://www.gamingonlinux.com/2024/04/fedora-linux-40-is-officially-out-now/
16. 9to5Linux — Mesa 25.1 to replace Nouveau with Zink/NVK by default — https://9to5linux.com/mesa-25-1-to-replace-nouveau-driver-with-zink-nvk-by-default-for-nvidia-gpus
17. nullr0ute blog — Getting started with OpenCL using Mesa/rusticl — https://nullr0ute.com/2023/12/getting-started-with-opencl-using-mesa-rusticl/
18. Airlied blog — Vulkan Video decoding RADV status — https://airlied.blogspot.com/2022/12/vulkan-video-decoding-radv-status.html
19. Ubuntu 24.04 LTS Release Notes — https://documentation.ubuntu.com/release-notes/24.04/
20. Debian 12 Mesa package tracker — https://packages.debian.org/source/bookworm/mesa
21. Fedora package tracker (Mesa) — https://packages.fedoraproject.org/pkgs/mesa
22. SteamOS Getting Started Wiki — https://github.com/ValveSoftware/SteamOS/wiki/Getting-Started
23. Collabora blog — DRM format modifiers — https://www.collabora.com/news-and-blog/blog/2017/02/09/notes-on-drm-format-modifiers/
24. Melissa Wen blog — AMD KMS Color API developments — https://melissawen.github.io/blog/2025/05/19/drm-info-with-kms-color-api
25. Mesa LLVMpipe ACO RFC (mesa-dev mailing list, July 2019) — https://lists.freedesktop.org/archives/mesa-dev/2019-July/221006.html
26. `drivers/gpu/drm/drm_syncobj.c` (Linux kernel source) — https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c
27. `drivers/gpu/drm/scheduler/sched_main.c` (Linux kernel source) — https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/scheduler/sched_main.c

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
