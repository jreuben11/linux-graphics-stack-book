# Chapter 194: Cross-Stack Integration — Protocols, Synchronisation, and the Coordination Layer

> **Part**: Part VI — The Display Stack
> **Audience**: Systems developers and graphics application developers who want to understand how the Linux graphics stack's many independent components are made to cooperate correctly and efficiently; contributors planning work that spans multiple layers of the stack
> **Status**: First draft — 2026-06-22

## Table of Contents

- [Overview](#overview)
- [1. The Fragmentation Problem](#1-the-fragmentation-problem)
- [2. The Wayland Protocol as Coordination Bus](#2-the-wayland-protocol-as-coordination-bus)
- [3. Zero-Copy Buffer Transport: DMA-BUF and Format Modifiers](#3-zero-copy-buffer-transport-dma-buf-and-format-modifiers)
- [4. Explicit GPU Synchronisation](#4-explicit-gpu-synchronisation)
- [5. Color Management End-to-End](#5-color-management-end-to-end)
- [6. Frame Pacing and Tearing Control](#6-frame-pacing-and-tearing-control)
- [7. Overlay Plane Promotion and Zero-Copy Scanout](#7-overlay-plane-promotion-and-zero-copy-scanout)
- [8. Mesa NIR: Driver Unification at the Shader IR Layer](#8-mesa-nir-driver-unification-at-the-shader-ir-layer)
- [9. Cross-Stack Debugging and Observability](#9-cross-stack-debugging-and-observability)
- [10. Roadmap](#10-roadmap)
- [Integrations](#integrations)

---

## Overview

The Linux graphics stack is structurally the opposite of a monolith. A rendered frame traverses a minimum of six independent software layers — kernel DRM/KMS, userspace Mesa, EGL or Vulkan WSI, the Wayland protocol, the compositor, and the application toolkit — each maintained by a separate community, each carrying its own abstractions and release cadence, and each largely unaware of the others' internal state. On top of this, hardware introduces GPU drivers (open and proprietary), display controller firmware, panel timing, and connector hardware as additional independently-specified pieces.

This structure is not accidental: the separation of concerns that makes the Linux graphics stack easy to extend and replace (swap Mesa for a proprietary driver; swap KWin for Sway; swap GTK for Qt) is also what creates the deficiencies. When two layers cannot agree on buffer lifetime, synchronisation point, color profile, or frame timing without explicit negotiation, that negotiation must happen somewhere, and the places where it historically did not happen are exactly where bugs, inefficiency, and incompatibility have concentrated.

This chapter examines the deficiency landscape and the mechanisms the Linux graphics community has developed to address it — primarily through the Wayland protocol ecosystem, which has emerged as the coordination bus that lets independent layers negotiate the contracts they need to cooperate correctly. It also covers Mesa NIR (which solves a parallel intra-stack fragmentation problem at the shader compilation layer), DMA-BUF (the zero-copy buffer transport primitive), and the cross-stack debugging tools that let developers trace a frame across all layers.

Chapters 74 and 75 cover HDR/wide color gamut and explicit GPU synchronisation in depth; this chapter treats those topics at the integration level — how each fits into the broader coordination story — and cross-references accordingly.

---

## 1. The Fragmentation Problem

### 1.1 What Fragmentation Actually Costs

The Linux graphics stack's modularity introduces costs in five concrete areas:

**Latency and pipeline stalls.** A correctly pipelined frame never waits: the compositor begins importing the new buffer at the earliest possible moment after the application GPU work completes, and KMS scans it out at the next vblank without stalling. In practice, without explicit contracts, the compositor cannot safely import a buffer until it has stalled the GPU pipeline to verify the previous owner has finished — adding one to two frame intervals of latency per frame under load.

**Redundant copies.** Without a shared zero-copy buffer primitive, each layer boundary defaults to a `memcpy`. A video decoder writing to a VA-API surface, a texture sampler reading it, a compositor blending it, and a KMS scanout displaying it could in principle touch the same DRM GEM buffer throughout; without format negotiation, each hands a new CPU copy to the next. In practice, `glTexImage2D` with a CPU pointer — the fallback path — copies once per frame per layer boundary.

**Color-space blindness.** For most of the Linux graphics stack's history, every layer assumed sRGB for 8-bit surfaces and nothing for 10-bit surfaces. A wide-gamut application writing scRGB or DCI-P3 values had no way to declare that intent; the compositor composited them as if they were sRGB; the display received no color metadata. The result was systematically wrong color on any display with a wide-gamut panel or HDR mode.

**Implicit synchronisation hazards.** The legacy model required every layer to guess when the previous layer's GPU work was done. The common heuristic was `glFinish()` or a pipeline stall: wait until the GPU is idle, then hand off the buffer. On hardware with implicit fence propagation (most AMD, Intel, recent ARM Mali), a looser version worked; on NVIDIA, which did not expose implicit fences to the Wayland compositor path until the open-driver era, it failed regularly, producing corruption, incorrect frame order, or visual tearing.

**Debuggability gaps.** Tracing a dropped frame requires correlation across `WAYLAND_DEBUG` protocol events, Mesa GPU timestamps, DRM event timestamps, compositor frame records, and kernel `ftrace` GPU scheduler events — six independent data streams with no shared frame identifier or unified timeline.

### 1.2 Why These Are Cross-Layer Problems

Each of the above is a deficiency that cannot be fixed unilaterally. Fixing copy overhead requires that the producer (app or decoder), the compositor, and the kernel DRM layer all agree on the same buffer handle type, tiling modifier, and lifetime semantics. Fixing synchronisation requires that the GPU driver expose fence primitives, that Mesa's WSI layer export those primitives to the Wayland surface, that the protocol carry them, and that the compositor consume them. No single layer can fix any of these alone.

This is the core insight that shaped the last five years of Linux graphics development: **cross-layer improvements require explicit contracts, and those contracts need a standardisation mechanism**. That mechanism turned out to be the Wayland protocol extension process.

---

## 2. The Wayland Protocol as Coordination Bus

### 2.1 Protocol Extensions as API Contracts

Wayland's core protocol (`wayland.xml`) defines surfaces, buffers, inputs, and outputs at the most primitive level. It intentionally does not specify buffer formats, GPU synchronisation, color profiles, or frame timing — these are left to **protocol extensions** (`*.xml` files in `wayland-protocols`, vendor namespaces, or staging protocols in the `wayland-protocols` repository).

This extensibility is the key: when two layers need a new contract, the appropriate authors draft a Wayland protocol extension, submit it for review in `wayland-protocols`, and once ratified it becomes the standard mechanism that compositor and toolkit authors implement independently. The protocol is the specification; implementations are independent.

```
wayland-protocols repository:
  stable/          ← ratified, can be relied upon
    xdg-shell/
    linux-dmabuf/
    presentation-time/
    ...
  staging/         ← implementation-ready, not yet stable
    linux-drm-syncobj/
    color-management/
    fifo/
    tearing-control/
    ...
  unstable/        ← early drafts, may change
    ...
```

[Source: https://gitlab.freedesktop.org/wayland/wayland-protocols]

The progression from `unstable` → `staging` → `stable` corresponds roughly to: "draft", "implemented and tested in at least two compositors and two clients", "guaranteed stable ABI". The `staging` tier exists precisely to let the ecosystem test and iterate before freezing the wire format.

### 2.2 The Role of xdg-desktop-portal

Not all cross-layer contracts are between app and compositor. Some involve three-party coordination: app ↔ portal ↔ system service. **`xdg-desktop-portal`** is a D-Bus service that provides a compositor-agnostic interface for privileged operations — screen capture, file picker, camera access — where the implementation is compositor-specific but the API is stable.

```
App (any toolkit)
  │  D-Bus call: org.freedesktop.portal.ScreenCast.CreateSession
  ▼
xdg-desktop-portal (running as user service)
  │  compositor-specific backend: xdg-desktop-portal-gnome / xdg-desktop-portal-wlr
  ▼
Compositor: captures frame via Wayland protocol (wlr_screencopy / ext_image_copy_capture)
  │  returns PipeWire stream node ID to portal
  ▼
App: connects to PipeWire stream → receives DMA-BUF frames
```

This pattern — a stable portal API hiding a compositor-specific implementation — solves the "every compositor needs a separate integration" problem for operations that require compositor trust.

### 2.3 Protocol Ratification as Community Coordination

The Wayland protocol review process serves a second function beyond technical specification: it forces the multiple parties (Mesa developers, compositor authors, toolkit developers, GPU driver developers) to agree on semantics before they write code. The `wp_linux_drm_syncobj_v1` protocol, for example, was co-developed by GPU driver authors (to ensure the kernel syncobj interface was fit for purpose), Mesa WSI engineers (to ensure EGL/Vulkan surface code could export the right primitives), and compositor authors (KWin, Mutter, wlroots) simultaneously. The protocol text served as the common reference. This is distinct from how kernel uAPIs or Mesa internal APIs are designed — those are unilateral.

---

## 3. Zero-Copy Buffer Transport: DMA-BUF and Format Modifiers

### 3.1 DMA-BUF as the Universal Buffer Primitive

**DMA-BUF** is a Linux kernel mechanism for sharing GPU (and other) buffer objects across process and subsystem boundaries using file descriptors. Introduced in Linux 3.3, it is now the foundation of every zero-copy path in the Linux graphics stack.

```
Producer (app / GPU / video decoder)
  │  gbm_bo_get_fd() / drmPrimeHandleToFD() / vaExportSurfaceHandle()
  ├── DMA-BUF fd (integer file descriptor referencing a GEM BO)
  │
  │  sendmsg() over Unix socket / Wayland protocol attachment
  ▼
Consumer (compositor / another GPU / display engine)
  │  drmPrimeFDToHandle() / eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)
  └── Same physical memory, no copy
```

The fd is the serialisable reference: it can be sent over Unix sockets (and therefore over the Wayland protocol), across process boundaries, and to the kernel's KMS subsystem for direct scanout. When the last holder closes the fd, the buffer is released — reference-counted by the kernel.

### 3.2 The Format Modifier Problem

DMA-BUF alone is insufficient: GPUs tile memory (arrange pixels in Z-order or vendor-specific patterns) and compress it (DCC on AMD, CCS on Intel, AFBC on ARM) to reduce memory bandwidth. A DMA-BUF exported from an AMD GPU in `AMD_FMT_MOD_DCC` tiled format cannot be directly consumed by Intel's display engine — the layout is unknown.

**DRM format modifiers** (introduced in Linux 4.10, standardised via `DRM_FORMAT_MOD_*` constants in `drm_fourcc.h`) solve this by attaching a 64-bit modifier token to every buffer that describes its memory layout:

```c
/* A DMA-BUF with a tiling modifier */
struct drm_format_modifier_blob {
    uint32_t count_formats;    /* number of format entries */
    uint32_t count_modifiers;  /* number of modifier entries */
    /* followed by format array and modifier array */
};

/* Example modifier constants */
DRM_FORMAT_MOD_LINEAR              /* 0ULL: row-major, no tiling */
DRM_FORMAT_MOD_INVALID             /* -1ULL: unknown/opaque */
AMD_FMT_MOD_DCC                    /* AMD delta color compression */
I915_FORMAT_MOD_Yf_TILED_CCS      /* Intel Y-tiled with CCS */
AFBC_FORMAT_MOD_BLOCK_SIZE_16x16  /* ARM AFBC 16×16 superblock */
```

Format modifier negotiation flows through the `zwp_linux_dmabuf_v1` / `wp_linux_dmabuf_v1` Wayland protocol: the compositor advertises which `(format, modifier)` pairs it can import; the application selects one; the GPU allocates a GBM buffer in that format; the DMA-BUF fd carries the modifier as metadata. The compositor's EGL or Vulkan import path (`eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)` with `EGL_DMA_BUF_PLANE0_MODIFIER_*` attributes) uses the modifier to pass the layout to Mesa, which configures the texture unit accordingly. [Source: https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import_modifiers.txt]

The `VK_EXT_drm_format_modifier` Vulkan extension (`VkPhysicalDeviceImageDrmFormatModifierInfoEXT`, `VkImageDrmFormatModifierExplicitCreateInfoEXT`) brings the same negotiation into the Vulkan path, allowing `VkImage` objects to be allocated in compositor-compatible tiled layouts and exported as DMA-BUFs without format conversion. [Source: https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_drm_format_modifier.html]

### 3.3 GBM: The Buffer Allocation Library

**GBM** (Generic Buffer Management, `libgbm`) is Mesa's userspace API for allocating DRM GEM buffers in GPU-native formats. It is the allocator that closes the loop between format modifier negotiation and buffer allocation:

```c
struct gbm_device *gbm = gbm_create_device(drm_fd);

/* Allocate a buffer in a modifier negotiated with the compositor */
uint64_t modifiers[] = { AMD_FMT_MOD_DCC, DRM_FORMAT_MOD_LINEAR };
struct gbm_bo *bo = gbm_bo_create_with_modifiers2(gbm,
    width, height, GBM_FORMAT_ARGB8888,
    modifiers, ARRAY_SIZE(modifiers),
    GBM_BO_USE_RENDERING | GBM_BO_USE_SCANOUT);

/* Export as DMA-BUF fd for compositor submission */
int fd = gbm_bo_get_fd(bo);
uint64_t modifier = gbm_bo_get_modifier(bo);
```

`gbm_bo_create_with_modifiers2()` allocates in the first modifier from the list that the driver supports. The resulting `gbm_bo` carries the actual modifier chosen, which is then passed to the compositor as `wp_linux_dmabuf_v1` buffer parameters.

---

## 4. Explicit GPU Synchronisation

This section summarises the cross-layer story; Chapter 75 covers `wp_linux_drm_syncobj_v1` implementation in depth.

### 4.1 The Implicit Sync Failure Mode

The legacy implicit sync model requires compositors to call `eglClientWaitSync()` or stall the GL pipeline after receiving a client buffer — waiting for the GPU to finish rendering before the compositor touches the buffer. On AMD and Intel hardware, driver-level fence propagation makes this mostly invisible; on NVIDIA (proprietary driver) and on multi-GPU configurations, there was no reliable fence propagation path through the Wayland protocol, and compositors had to either stall completely or risk displaying partially-rendered frames.

The NVIDIA proprietary driver on Wayland suffered from this for years: the compositor could not know when NVIDIA's rendering was done, so it inserted a `glFinish()` stall — serialising what should be pipelined work and causing frame rate drops and stuttering. [Source: https://www.phoronix.com/news/NVIDIA-Wayland-Explicit-Sync]

### 4.2 The Cross-Layer Solution

`wp_linux_drm_syncobj_v1` (Wayland protocol) carries explicit DRM timeline syncobj acquire and release points alongside each `wl_surface_commit`. The cross-layer implementation chain:

```
DRM kernel: timeline syncobj (drm/syncobj.c)
  DRM_IOCTL_SYNCOBJ_CREATE / DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL
  │
Mesa EGL/Vulkan WSI: EGL_ANDROID_native_fence_sync, VK_KHR_external_semaphore_fd
  mesa/src/egl/drivers/dri2/platform_wayland.c
  │  exports timeline point alongside DMA-BUF
  │
Wayland: wp_linux_drm_syncobj_surface_v1.set_acquire_point()
  │
Compositor (KWin 6.1+, Mutter 46+, wlroots):
  consumes acquire point, waits before compositing
  │
KMS: atomic_commit with implicit fence from compositor GPU work
  │
Display
```

Each layer contributes one piece: the kernel exposes the syncobj API; Mesa exports the fence; the protocol carries it; the compositor consumes it. No single layer could fix this; the protocol made the contract explicit. The fix landed in Mesa 24.1, KWin 6.1, and Mutter 46 (GNOME 46) in mid-2024. [Source: https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/linux-drm-syncobj/linux-drm-syncobj-v1.xml]

---

## 5. Color Management End-to-End

This section covers the coordination story; Chapter 74 covers HDR and wide color gamut in depth.

### 5.1 The sRGB Assumption

Every layer of the pre-2024 Linux graphics stack implicitly assumed sRGB for 8-bit surfaces. Wide-gamut displays (DCI-P3, BT.2020) existed but the stack had no way to communicate primaries or transfer functions between layers:

- The application rendered in a color space with no way to declare it to EGL or the Wayland compositor.
- The compositor blended surfaces in whatever color space the framebuffer happened to be in.
- KMS could set `DRM_MODE_COLORSPACE_*` on CRTC output properties, but nothing fed color metadata from the application through the stack to the CRTC.

The result was systematic color errors: saturated colors clipped at the sRGB gamut boundary; HDR content tone-mapped incorrectly or not at all; wide-gamut panels displaying the wrong primaries.

### 5.2 The Cross-Layer Color Management Stack

`wp_color_management_v1` (ratified 2025) defines a surface-level color contract: applications attach a **color description** (primaries, transfer function, optionally a full ICC profile) to their `wl_surface`. The compositor uses that description when blending surfaces into the compositor framebuffer, and passes the resulting framebuffer color metadata to KMS for display color transform application.

The cross-layer chain:

```
Application
  │  wp_color_manager_v1.create_color_description(primaries=BT2020, tf=ST2084_PQ)
  │  wp_color_management_surface_v1.set_color_description(surface, color_desc)
  │
Wayland protocol: color description attached to wl_surface
  │
Compositor (KWin, Mutter)
  │  reads surface color description
  │  transforms during composition (tonemap HDR → SDR output, or blend in shared HDR space)
  │  GskVulkanRenderer / KWin scene graph: compositor-side color transform shaders
  │
KMS DRM output
  │  drm_color_mgmt: CRTC color LUT (DRM_PROP_DEGAMMA / DRM_PROP_GAMMA / DRM_PROP_CTM)
  │  drm_hdr_output_metadata: HDR_OUTPUT_METADATA blob (MaxCLL, MaxFALL, EOTF)
  │  sent to display over HDMI/DP HDR metadata InfoFrame
  │
Display: applies EOTF, renders in correct primaries
```

This requires kernel DRM color pipeline support (the 3×3 CTM matrix, gamma LUTs, and HDR metadata CRTC properties), a compositor that implements the color math, and a display that honours the HDR metadata. The protocol is the coordinating mechanism that lets these independent implementations find each other.

The `colord` system service (Ch53) sits adjacent to this: it maintains system-wide ICC profiles from display calibration measurements and feeds them to the compositor's color management path.

---

## 6. Frame Pacing and Tearing Control

### 6.1 The Frame Pacing Problem

In the legacy Wayland model, an application renders and calls `wl_surface_commit()`. The compositor may present that frame at the next vblank, or it may skip it if a newer frame arrives before the vblank. From the application's perspective there is no feedback: it cannot know whether its frame was presented or dropped, or when to start rendering the next frame to minimise latency without causing a miss.

The `wp_presentation` protocol (stable, `wayland-protocols/stable/presentation-time/`) added accurate presentation timestamps — apps can measure actual displayed-at times. But it did not add frame scheduling: the question of *when the compositor will consume the next commit* remained unanswered.

This creates a dilemma:
- **Commit too early**: compositor queues two frames, presents the first and drops the second — wasted GPU work.
- **Commit too late**: compositor presents at the wrong vblank — one frame of extra latency.

### 6.2 wp_fifo_v1: Ordered Frame Delivery

`wp_fifo_v1` (staging, in `wayland-protocols/staging/fifo/`) addresses commit ordering. The core primitive is `wp_fifo_v1.set_barrier()` + `wp_fifo_v1.wait_barrier()`: an application that wants its frames delivered in order (i.e., no compositor frame-skipping) can mark a commit as a barrier that the compositor must wait for before applying the next commit.

```c
struct wp_fifo_manager_v1 *fifo_mgr = /* bind from registry */;
struct wp_fifo_v1 *fifo = wp_fifo_manager_v1_get_fifo(fifo_mgr, surface);

/* Commit frame N */
wl_surface_damage_buffer(surface, 0, 0, INT32_MAX, INT32_MAX);
wl_surface_attach(surface, buffer_n, 0, 0);
wp_fifo_v1_set_barrier(fifo);      /* "don't apply the next commit until this one is shown" */
wl_surface_commit(surface);

/* Commit frame N+1 */
wl_surface_attach(surface, buffer_n1, 0, 0);
wp_fifo_v1_wait_barrier(fifo);     /* "wait for the previous barrier before applying this" */
wl_surface_commit(surface);
```

The effect is that the compositor presents `buffer_n`, waits for that presentation, then applies and presents `buffer_n1` — FIFO delivery, not "latest wins". This is the primitive that video players and applications needing smooth playback require; without it, a compositor under load will skip frames and disrupt video cadence. [Source: https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/fifo/fifo-v1.xml]

### 6.3 wp_tearing_control_v1: Explicit Tearing Opt-In

`wp_tearing_control_v1` (staging) allows applications to opt out of the compositor's tear-free guarantee for individual surfaces:

```c
struct wp_tearing_control_manager_v1 *mgr = /* bind */;
struct wp_tearing_control_v1 *ctrl =
    wp_tearing_control_manager_v1_get_tearing_control(mgr, surface);

/* Allow the compositor to present this surface immediately without vblank sync */
wp_tearing_control_v1_set_presentation_hint(ctrl,
    WP_TEARING_CONTROL_V1_PRESENTATION_HINT_ASYNC);
```

With `ASYNC` hint, the compositor is permitted (but not required) to display the surface mid-frame — tearing but with minimal latency. The primary use case is competitive games where input latency matters more than tearing, running inside a compositor (rather than fullscreen exclusive). The compositor can choose to honour the hint only when the surface is fullscreen and the hardware supports direct scanout, giving zero-copy async presentation. [Source: https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/tearing-control/tearing-control-v1.xml]

### 6.4 wp_presentation and Frame Timing

`wp_presentation` (stable) provides per-commit presentation timestamps and flags. On a Wayland compositor that implements it, the application receives a `presented` event carrying the exact kernel KMS timestamp at which the buffer was scanned out:

```c
struct wp_presentation *pres = /* bind */;

/* After each commit, request a feedback object */
struct wp_presentation_feedback *fb =
    wp_presentation_feedback(pres, surface);

/* Listener receives 'presented' or 'discarded' event */
static void feedback_presented(void *data,
    struct wp_presentation_feedback *fb,
    uint32_t tv_sec_hi, uint32_t tv_sec_lo, uint32_t tv_nsec,
    uint32_t refresh, uint32_t seq_hi, uint32_t seq_lo,
    uint32_t flags) {
    /* tv_sec_hi/lo + tv_nsec: CLOCK_MONOTONIC presentation time */
    /* refresh: target refresh interval in nanoseconds */
    /* flags: WP_PRESENTATION_FEEDBACK_KIND_VSYNC | _ZERO_COPY | _HW_COMPLETION | _HW_CLOCK */
}
```

The `WP_PRESENTATION_FEEDBACK_KIND_ZERO_COPY` flag in the response tells the application that its buffer was scanned out directly (no compositor copy) — the highest-quality presentation path. `WP_PRESENTATION_FEEDBACK_KIND_HW_CLOCK` indicates the timestamp came from the KMS hardware clock, not a software estimate.

---

## 7. Overlay Plane Promotion and Zero-Copy Scanout

### 7.1 KMS Overlay Planes

KMS (Kernel Mode Setting) exposes **overlay planes** (`drm_plane` objects of type `DRM_PLANE_TYPE_OVERLAY`) that allow multiple buffers to be composed by the display hardware at scan time, without the compositor GPU doing any alpha blending. The display controller reads from two (or more) DMA-BUF sources simultaneously and blends them in hardware at the pixel clock rate.

When a compositor can assign a Wayland surface to an overlay plane, the result is **zero-copy direct scanout**: the app's DMA-BUF is handed directly to KMS, bypassing compositor GPU work entirely. This saves power (no compositor draw call), reduces latency (one fewer step), and can enable HDR or higher refresh rates for individual surfaces (e.g., a game running at 144 Hz inside a 60 Hz desktop).

### 7.2 GskSubsurfaceNode and Compositor Plane Promotion

GTK4 (§4, Ch39) exposes `GskSubsurfaceNode` — a render node that tells the GTK4 Vulkan/GL renderer "this content should be a Wayland subsurface, not composited into the parent surface". The compositor, on receiving a `wl_subsurface` from a GTK4 application, can attempt to assign the subsurface to an overlay plane via KMS atomic commit.

The protocol that enables this is **`xdg_toplevel_icon`** and, more relevantly for GPU surfaces, the combination of `wp_linux_drm_syncobj_v1` (explicit sync) and `linux-dmabuf-v1` modifiers on the subsurface: the compositor can attempt an atomic plane assignment with the DMA-BUF if:

1. The buffer's format modifier matches a modifier supported by the overlay plane.
2. Explicit sync acquire/release points are carried (so the plane promotion doesn't race with app rendering).
3. The overlay plane geometry fits the compositor's constraints (position, z-order, scaling limits).

If the KMS `TEST_ONLY` atomic commit fails, the compositor falls back to GPU compositing. This "optimistic plane assignment" pattern allows subsurfaces — including video overlays, WebKit web surfaces (Ch193), and terminal graphics (Ch45) — to be promoted to zero-copy hardware overlay without the application having to know.

### 7.3 ext_image_copy_capture: Unified Screen Capture

The proliferation of compositor-specific screen capture APIs (`wlr_screencopy_v1` for wlroots, `zcosmos_screencopy_v2` for Cosmic, `_wlr_output_management_v1`, and compositor-proprietary D-Bus methods) was itself a fragmentation problem: every screen capture tool needed separate compositor backends.

`ext_image_copy_capture_v1` (in `wayland-protocols/staging/ext-image-copy-capture/`) standardises capture: a client creates a `ext_image_copy_capture_session_v1` for an output or toplevel, requests frames into DMA-BUF-backed `wp_buffer` objects, and receives frame events. Compositors can implement one protocol instead of four.

From a cross-stack perspective, `ext_image_copy_capture` closes the loop between the Wayland compositor and `xdg-desktop-portal`'s screencast backend: the portal can implement its PipeWire screen-capture path on top of `ext_image_copy_capture` rather than compositor-specific protocols, giving a stable path for remote desktop, recording, and accessibility tools across all compositors. [Source: https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/ext-image-copy-capture/ext-image-copy-capture-v1.xml]

---

## 8. Mesa NIR: Driver Unification at the Shader IR Layer

### 8.1 The Pre-NIR State

Before NIR, each Mesa driver maintained its own shader IR:
- `GLSL IR` — Mesa's original intermediate representation, consumed by the GLSL compiler
- `TGSI` — Gallium3D's Tungsten Graphics Shader Infrastructure, the original Gallium IR
- `ARB programs` — legacy ARB_vertex_program / ARB_fragment_program IR

Each driver had its own GLSL→TGSI or GLSL→driver-IR lowering path, its own set of optimisation passes (dead code elimination, constant folding, loop unrolling), and its own register allocator. Sharing optimisations between drivers was structurally difficult: a fix in the `i915` pass didn't help `radeonsi`; a new optimisation in `r600` wasn't available to `freedreno`.

### 8.2 NIR as the Universal Lowering Target

**NIR** (New IR, introduced in Mesa 10.5, now the default for all Mesa drivers) is a single SSA (Static Single Assignment) IR that all Mesa state trackers lower to before handing work to driver backends. [Source: https://docs.mesa3d.org/nir.html]

```
Application (GLSL / SPIR-V / HLSL via DXVK)
  │
  glslang / spirv_to_nir() / tgsi_to_nir()
  │
NIR — unified SSA IR
  │  shared optimisation passes:
  │    nir_lower_vars_to_ssa()     — phi node insertion
  │    nir_opt_algebraic()         — algebraic simplification
  │    nir_opt_dead_cf()           — dead control flow elimination
  │    nir_lower_phis_to_scalar()  — vectorisation
  │    nir_opt_loop_unroll()       — loop unrolling
  │    nir_lower_samplers()        — texture access lowering
  │    nir_opt_gcm()               — global code motion
  │  ... ~80 shared passes as of Mesa 24
  │
Driver backend
  ├── ACO (AMD — Mesa 20+, replaces LLVM for GFX8+)
  ├── BRW (Intel Xe, iris)
  ├── NAK (NVIDIA NVK — Rust, replaces NVIR)
  ├── Panfrost (ARM Mali)
  ├── Turnip (Qualcomm Adreno)
  ├── Lima (ARM Mali400)
  └── ... (all Gallium drivers)
```

[Source: https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/compiler/nir]

The cross-stack benefit is dramatic: a bug fix in `nir_opt_algebraic` (e.g., an incorrect fold of a negation) applies to every driver simultaneously. A new optimisation (e.g., better divergence analysis for subgroup operations) is written once and benefits all targets. The ACO backend for AMD, the BRW backend for Intel, and the NAK backend for NVIDIA all start from the same NIR input with the same ~80 passes already applied — only the final ISA-specific lowering and register allocation differ.

### 8.3 SPIR-V as the Cross-API Entry Point

NIR also solves the cross-API fragmentation problem at the entry point. Instead of every API (OpenGL, Vulkan, OpenCL, Metal via MoltenVK, D3D via DXVK/VKD3D) feeding its own IR into each driver:

```
OpenGL GLSL    → glslang    → SPIR-V → spirv_to_nir() → NIR → driver
Vulkan SPIR-V             → SPIR-V → spirv_to_nir() → NIR → driver
OpenCL C       → Clang      → SPIR-V → spirv_to_nir() → NIR → driver
DXVK (D3D11)   → dxvk-spirv→ SPIR-V → spirv_to_nir() → NIR → driver
VKD3D (D3D12)  → vkd3d-spirv→SPIR-V → spirv_to_nir() → NIR → driver
```

SPIR-V + `spirv_to_nir()` is the integration point. Any language that can produce SPIR-V (via `glslang`, `Clang`/`spirv-llvm`, `DirectXShaderCompiler`, or custom toolchains) automatically works with every Mesa driver. This is the mechanism that makes DXVK (Ch104), VKD3D-Proton, and zink (OpenGL-over-Vulkan) possible without per-driver porting work.

---

## 9. Cross-Stack Debugging and Observability

Debugging a problem that spans multiple layers requires tools that each layer exposes. There is no single unified profiler, but the combination of per-layer tools can reconstruct a frame's journey.

### 9.1 Per-Layer Debug Environment Variables

Each layer exposes a debug interface via environment variables:

```bash
# Wayland protocol trace (all protocol messages)
WAYLAND_DEBUG=1 myapp

# Mesa GPU debug (shader dumps, pipeline state, draw calls)
MESA_DEBUG=all,flush myapp        # all debug output + explicit flushes
MESA_GLSL_CACHE_DISABLE=1        # disable shader cache (force recompilation)
MESA_SHADER_DUMP_PATH=/tmp/shaders myapp  # dump GLSL/SPIR-V/ISA to files

# GTK4 renderer selection and debug
GSK_RENDERER=vulkan myapp
GTK_DEBUG=frames myapp           # frame timing output

# DRM/KMS event tracing
sudo cat /sys/kernel/debug/dri/0/state   # current DRM atomic state
sudo cat /sys/kernel/debug/dri/0/framebuffer  # active framebuffers

# Linux perf + GPU events
perf stat -e gpu/cycles/ myapp

# Wayland compositor debug (compositor-specific)
KWIN_EXPLICIT_SYNC=1 myapp       # force explicit sync path in KWin
MUTTER_DEBUG_PAINT=1             # Mutter frame paint debug
```

### 9.2 RenderDoc: Multi-API Frame Capture

**RenderDoc** (open-source, `https://renderdoc.org`) is the primary cross-API GPU frame debugger for Linux. It intercepts the Vulkan, OpenGL, and OpenGL ES APIs at the driver dispatch layer and captures all GPU commands, buffer contents, and texture data for a single frame, providing a full replay and inspection capability.

RenderDoc on Wayland requires the application to use the `VK_LAYER_RENDERDOC_Capture` Vulkan layer (injected via `VK_INSTANCE_LAYERS=VK_LAYER_RENDERDOC_Capture`); the capture file contains enough state to fully replay the frame offline. It does not capture the compositor's own rendering — it sees the application's submissions to the Vulkan/GL API but not what the compositor does with the resulting `wl_buffer`. For compositor-level capture, `apitrace` with the compositor's rendering API is the closest available tool.

### 9.3 Frame Timing Correlation

The `wp_presentation` protocol (§6.4) provides per-frame KMS presentation timestamps on the application side. On the compositor side, **KWin** and **Mutter** expose frame timing via compositor-specific mechanisms:

- KWin: `KWin::EffectsHandler::renderTargetScale()` and the `RenderLayer` frame stats; the `kwin_wayland --xwayland` process logs DRM pageflip timestamps via kernel tracepoints.
- Mutter: `MetaCompositorView` frame stats and the `MetaKmsCrtc` DRM pageflip callback timestamps; exposed in GNOME's experimental `org.gnome.Shell.Eval` for compositor state inspection.

**`drm_monitor`** (part of `libdrm`'s `util/` tools) listens for DRM `vblank` and `pageflip` events and timestamps them with `CLOCK_MONOTONIC`, providing a ground truth for when frames actually hit the display.

Correlating these three clocks (app `wp_presentation` timestamp, compositor frame record, `drm_monitor` pageflip) with a shared `CLOCK_MONOTONIC` base gives a complete picture of where latency originates: in application rendering, in compositor queuing, or in display controller timing.

### 9.4 Linux perf and GPU Tracepoints

The Linux kernel `ftrace` infrastructure exposes GPU scheduler events on AMD (`amdgpu_cs_ioctl`, `amdgpu_sched_run_job`), Intel (`i915_gem_request_submit`, `i915_request_queue`), and to a lesser extent other drivers:

```bash
# Trace AMD GPU job submission and completion
sudo trace-cmd record -e amdgpu:amdgpu_cs_ioctl \
                      -e amdgpu:amdgpu_sched_run_job \
                      -p function myapp
sudo trace-cmd report
```

Combined with `perf` hardware performance counters, this provides timestamps for when GPU work was submitted, when it started executing on the hardware, and when it completed — enabling measurement of GPU pipeline fill and idle time across the DRM scheduler, Mesa command stream, and application rendering loop.

---

## 10. Roadmap

### Near-term (6–12 months)

- **`ext_image_copy_capture` stabilisation and portal migration**: `ext_image_copy_capture_v1` is currently staging; ratification is expected once KWin and Mutter implementations mature. The `xdg-desktop-portal` screencast backend is scheduled to migrate to it from `wlr_screencopy` (for wlroots compositors) and GNOME-specific APIs, unifying screen capture across all Wayland compositors. Note: compositor implementation status requires verification for mid-2026 state.
- **`wp_fifo_v1` + VRR interaction**: The interaction between `wp_fifo_v1` ordered delivery and variable refresh rate displays is an open design question: on a VRR display, "present in FIFO order" and "present at the frame's natural refresh interval" may conflict. Protocol clarification is expected as compositors implement both features simultaneously.
- **`wp_color_management_v1` toolkit integration**: The `wp_color_management_v1` protocol is ratified (2025); GTK4 and Qt6 bindings are under active development. GTK's `GdkWaylandSurface` integration is tracked in GNOME GitLab; Qt's `QWaylandWindow` integration is tracked in the qtwayland repository. Full end-to-end HDR on a calibrated display — from `wp_color_management_v1` surface submission through compositor tonemap through KMS `drm_hdr_output_metadata` — is the milestone that closes the HDR story.
- **Mesa NIR divergence analysis improvements**: Mesa 24+ introduced initial subgroup divergence analysis in NIR for better GPU thread wavefront management; follow-on work to extend this to loop divergence and memory access divergence is tracked in the Mesa NIR optimisation backlog, benefiting all drivers simultaneously.

### Medium-term (1–3 years)

- **Compositor-bypass direct scanout protocol**: A proposed Wayland extension (no stable name as of mid-2026) would allow privileged compositor clients (games, media players) to request direct KMS scanout — submitting a DMA-BUF directly to KMS without any compositor involvement, while retaining Wayland input event delivery. This eliminates the last copy/stall in the fullscreen path without requiring the app to open `/dev/dri/card0` directly.
- **Unified GPU tracing infrastructure**: There is growing interest (from RenderDoc, Mesa, and compositor communities) in a unified GPU frame tracing format that correlates application API calls, Mesa GPU commands, DRM scheduler events, and compositor frame records in a single timeline — analogous to Perfetto on Android. No concrete standardisation effort exists as of mid-2026; this is a significant gap in debuggability.
- **Cross-layer power management coordination**: Current DVFS (dynamic voltage/frequency scaling) decisions in the DRM power management layer (`amdgpu_dpm`, `i915_pm`) are made without knowledge of frame deadlines from the compositor. A mechanism for the compositor to communicate expected frame intervals and GPU workload targets to the kernel power governor would enable earlier frequency scaling with fewer latency spikes.

### Long-term

- **Wayland as the universal graphics IPC layer**: The trajectory of Wayland protocol extensions (color, sync, capture, fifo, plane promotion) suggests a possible future where the Wayland protocol is the *complete* cross-layer negotiation mechanism: every contract between app, compositor, and kernel is carried by an explicit Wayland protocol message. This would make Wayland not just a display protocol but a full graphics IPC bus — the Linux equivalent of what `CoreGraphics` + `CoreAnimation` + `IOSurface` are on macOS.
- **NIR for non-Mesa targets**: SPIR-V + NIR has proven so effective that non-Mesa GPU stacks (notably the NVIDIA proprietary driver's CUDA compiler) are exploring NIR as an intermediate stage for OpenGL/Vulkan shader compilation. If NIR became a multi-vendor standard IR, cross-vendor shader analysis tools and optimisation pass sharing would become practical.

---

## Integrations

This chapter draws together themes that run through the entire book. Every other chapter that touches a layer boundary implicitly depends on the mechanisms described here:

- **Chapter 2 (KMS: The Display Pipeline)**: The KMS atomic API and overlay plane model are the kernel side of the DMA-BUF zero-copy scanout path and the `drm_hdr_output_metadata` color management endpoint described in §3 and §5.

- **Chapter 14 (NIR)**: The unified NIR IR described in §8 is the subject of Chapter 14 in full: shader lowering passes, optimisation pipeline, and the `spirv_to_nir()` entry point.

- **Chapter 16 (Mesa Vulkan Common Infrastructure)**: Mesa's Vulkan common layer implements the EGL/Vulkan WSI side of the explicit sync chain (§4): `eglCreateSyncKHR`, `EGL_ANDROID_native_fence_sync`, and `VK_KHR_external_semaphore_fd` are the Mesa-side primitives that produce the timeline syncobj points carried by `wp_linux_drm_syncobj_v1`.

- **Chapter 20 (Wayland Protocol Fundamentals)**: The core protocol mechanics — `wl_registry`, `wl_proxy`, event loops, wire format — described in Chapter 20 are the substrate on which all the extension protocols in §2 operate.

- **Chapter 46 (The Evolving Wayland Protocol Ecosystem)**: Chapter 46 covers the `wayland-protocols` repository and the extension authoring/ratification process in depth; this chapter references that process as the coordination mechanism without duplicating the protocol-authoring detail.

- **Chapter 74 (HDR and Wide Color Gamut on Linux)**: The `wp_color_management_v1` cross-layer chain summarised in §5 is the subject of Chapter 74's full treatment: kernel DRM color pipeline, KMS HDR metadata properties, compositor tonemap shaders, and panel HDR mode negotiation.

- **Chapter 75 (Explicit GPU Synchronisation)**: The `wp_linux_drm_syncobj_v1` cross-layer chain summarised in §4 is covered in full in Chapter 75, including the kernel DRM syncobj API, Mesa WSI export, protocol wire format, and compositor import.

- **Chapter 39 (Qt and GTK GPU Rendering)**: The `GskSubsurfaceNode` plane promotion path (§7.2) and `GdkFrameClock` / `wp_presentation` integration (§6.4) are the toolkit-side consumers of the frame pacing and zero-copy scanout mechanisms described here.

- **Chapter 45 (Terminal Integration with the Compositor Stack)**: Terminal emulators that support the Kitty graphics protocol or Sixel use DMA-BUF subsurfaces (Ch45) as the mechanism for zero-copy image display — the same overlay plane promotion path described in §7.

- **Chapter 104 (DXVK and VKD3D-Proton)**: DXVK and VKD3D rely on `spirv_to_nir()` as their Mesa integration point (§8.3): D3D bytecode → vendor SPIR-V → NIR → driver ISA is the complete compilation chain.

---

## References

- wayland-protocols repository: [https://gitlab.freedesktop.org/wayland/wayland-protocols](https://gitlab.freedesktop.org/wayland/wayland-protocols)
- `wp_linux_drm_syncobj_v1` protocol: [https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/linux-drm-syncobj/linux-drm-syncobj-v1.xml](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/linux-drm-syncobj/linux-drm-syncobj-v1.xml)
- `wp_color_management_v1` protocol: [https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/color-management/color-management-v1.xml](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/color-management/color-management-v1.xml)
- `wp_fifo_v1` protocol: [https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/fifo/fifo-v1.xml](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/fifo/fifo-v1.xml)
- `wp_tearing_control_v1` protocol: [https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/tearing-control/tearing-control-v1.xml](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/tearing-control/tearing-control-v1.xml)
- `ext_image_copy_capture_v1` protocol: [https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/ext-image-copy-capture/ext-image-copy-capture-v1.xml](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/ext-image-copy-capture/ext-image-copy-capture-v1.xml)
- `EGL_EXT_image_dma_buf_import_modifiers`: [https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import_modifiers.txt](https://www.khronos.org/registry/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import_modifiers.txt)
- `VK_EXT_drm_format_modifier`: [https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_drm_format_modifier.html](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_drm_format_modifier.html)
- Mesa NIR documentation: [https://docs.mesa3d.org/nir.html](https://docs.mesa3d.org/nir.html)
- Mesa NIR source tree: [https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/compiler/nir](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/compiler/nir)
- NVIDIA explicit sync on Wayland (Phoronix): [https://www.phoronix.com/news/NVIDIA-Wayland-Explicit-Sync](https://www.phoronix.com/news/NVIDIA-Wayland-Explicit-Sync)
- RenderDoc GPU debugger: [https://renderdoc.org](https://renderdoc.org)
- xdg-desktop-portal: [https://github.com/flatpak/xdg-desktop-portal](https://github.com/flatpak/xdg-desktop-portal)
- GBM API reference (Mesa): [https://mesa.freedesktop.org/dbdocs/gbm/](https://mesa.freedesktop.org/dbdocs/gbm/)
- Linux DRM format modifiers (drm_fourcc.h): [https://cgit.freedesktop.org/drm/drm-tip/tree/include/uapi/drm/drm_fourcc.h](https://cgit.freedesktop.org/drm/drm-tip/tree/include/uapi/drm/drm_fourcc.h)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
