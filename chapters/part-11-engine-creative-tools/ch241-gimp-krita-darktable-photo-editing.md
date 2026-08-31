# Chapter 241: GIMP, Krita, and darktable — GPU-Accelerated Raster and RAW Photo Editing on Linux

**Target audiences:** Graphics application developers evaluating GPU compute integration patterns for creative desktop tools; Linux desktop developers working on GEGL, Krita, or darktable; photography and imaging engineers who need to understand how these applications actually reach the GPU (and how often they don't); systems developers comparing OpenCL-as-accelerator against GPU-native compute-graph architectures.

---

## Table of Contents

1. [Overview: Three Applications, Three GPU Strategies](#1-overview-three-applications-three-gpu-strategies)
2. [GEGL — GIMP's Node-Based Processing Core](#2-gegl--gimps-node-based-processing-core)
   - [2.1 GeglBuffer, Tiling, and babl](#21-geglbuffer-tiling-and-babl)
   - [2.2 OpenCL in GEGL: Opt-In, and Off by Default](#22-opencl-in-gegl-opt-in-and-off-by-default)
3. [GIMP 3.0: GTK3, Wayland, and the New Plugin API](#3-gimp-30-gtk3-wayland-and-the-new-plugin-api)
4. [Krita's Canvas: KisOpenGLCanvas2 and Instant Preview](#4-kritas-canvas-kisopenglcanvas2-and-instant-preview)
   - [4.1 Two Kinds of Tiles: KisTiledDataManager vs. KisTextureTileInfoPool](#41-two-kinds-of-tiles-kistileddatamanager-vs-kistexturetileinfopool)
5. [darktable's Per-Module OpenCL Pixel Pipeline](#5-darktables-per-module-opencl-pixel-pipeline)
   - [5.1 dt_tiling and VRAM-Constrained Processing](#51-dt_tiling-and-vram-constrained-processing)
   - [5.2 Diagnosing OpenCL: darktable-cltest and Device Priority](#52-diagnosing-opencl-darktable-cltest-and-device-priority)
6. [RawTherapee: The Deliberate CPU-Only Counterpoint](#6-rawtherapee-the-deliberate-cpu-only-counterpoint)
7. [vkdt: A Vulkan-Native Compute-Graph Pipeline](#7-vkdt-a-vulkan-native-compute-graph-pipeline)
8. [RAW Decoding as the Shared CPU-Bound Layer: RawSpeed and LibRaw](#8-raw-decoding-as-the-shared-cpu-bound-layer-rawspeed-and-libraw)
- [Roadmap](#roadmap)
9. [Integrations](#9-integrations)

---

## 1. Overview: Three Applications, Three GPU Strategies

GIMP, Krita, and darktable form the core of Linux's free-software raster and RAW photo-editing stack, alongside RawTherapee and the newer vkdt. Unlike Blender (Ch42) or the game engines covered elsewhere in Part XI, none of these applications is built around a GPU-resident scene graph as its primary execution model. Instead, each occupies a different point on a spectrum between "GPU as an optional per-operation accelerator" and "GPU as the processing substrate":

- **GIMP** runs its editing pipeline through GEGL, a node-based image-processing graph where individual operations can carry an OpenCL kernel — but OpenCL remains disabled by default even in GIMP 3.0 (2025), for reliability reasons discussed in §2.2.
- **Krita** keeps its brush engines and tile-compositing logic entirely CPU-bound, using the GPU (via `KisOpenGLCanvas2`) only to display the result and to smooth out live brush feedback through a lower-resolution proxy layer.
- **darktable** ships a genuine per-module OpenCL implementation for most of its image-operation (`iop`) plugins, with automatic tiling to fit large images through limited VRAM — the closest of the three to real GPU-accelerated processing, though CPU fallback is always present.
- **RawTherapee** deliberately has no GPU path at all, relying on OpenMP-multithreaded CPU code across its entire pipeline.
- **vkdt**, an experimental Vulkan-native rewrite of darktable's processing model, replaces the "OpenCL-as-accelerator" pattern with a compute graph where every stage — RAW decode through display — runs as Vulkan compute shaders scheduled from a single command buffer.

Reading these five architectures side by side is instructive precisely because they disagree about where GPU acceleration belongs in an image editor, and each choice was made for identifiable engineering reasons rather than by default.

## 2. GEGL — GIMP's Node-Based Processing Core

GIMP's editing pipeline since the 2.10 series has been built on **GEGL** (Generic Graphics Library), a graph-based image-processing framework maintained alongside GIMP. GEGL replaced the older direct `GimpPixelRgn` region-access API with a lazily evaluated graph of `GeglNode` objects; each node wraps a GEGL "operation" (`gegl:gaussian-blur`, `gegl:npd`, `gegl:brightness-contrast`, and so on), and the graph is only actually computed for the region and resolution a consumer requests. [Source](https://gegl.org/)

### 2.1 GeglBuffer, Tiling, and babl

Image data flowing through the graph is stored in `GeglBuffer`, a tiled, mipmapped buffer type designed for out-of-core processing — images larger than available RAM can be worked on because a `GeglBuffer` can swap its tiles to disk rather than requiring the whole image resident in memory. Internally a `GeglBuffer` is backed by `GeglTile` objects; the default tile size used throughout GEGL's own operations is **128×64 pixels**, and all tiles within one buffer are kept at a uniform size, with pixel data stored linearly inside each tile. [Source](https://gegl.org/architecture.html)

Format and color-space conversion between nodes is handled by **babl**, a small sister library that describes pixel formats (8-bit integer RGB through full-float CMYK, premultiplied or not, a range of color spaces) and provides conversion paths between them, with SIMD-optimized fast paths where available. When a `GeglBuffer` region is fetched in a format that already matches its internal storage format, babl's conversion degenerates to a `memcpy`; otherwise it performs the requested colorspace/format conversion on the way out. [Source](https://gegl.org/babl/)

### 2.2 OpenCL in GEGL: Opt-In, and Off by Default

GEGL operations can optionally ship an OpenCL kernel alongside their CPU implementation, and GIMP exposes a Preferences → Image Processing toggle to enable OpenCL use. This has been true since the GIMP 2.10 series. What has *not* changed by GIMP 3.0 (2025) is the default state of that toggle: OpenCL support in GIMP has remained **disabled by default since GIMP 2.10.22**, specifically because of robustness problems in OpenCL driver implementations encountered in the field. The GIMP 3.0 release notes describe continued work in this area but stop short of flipping the default: "Some most needed work has been done on making OpenCL implementations more robust though it is still disabled by default." [Source](https://www.gimp.org/release-notes/gimp-3.0.html)

This makes GIMP's GPU story narrower than it might first appear from GEGL's architecture alone: the node graph and per-operation OpenCL kernels exist, but the out-of-the-box experience for essentially all GIMP users, even on 3.0, is a fully CPU-executed GEGL graph. This is a meaningfully different posture from darktable's per-module OpenCL pipeline (§5), where OpenCL is auto-detected and used by default whenever a compatible device and driver are found.

## 3. GIMP 3.0: GTK3, Wayland, and the New Plugin API

GIMP 3.0, released in 2025, is built on **GTK3** rather than the GTK2 toolkit that GIMP 2.x shipped for over a decade. The GTK3 migration is largely orthogonal to GEGL/OpenCL processing — it is a UI-toolkit change — but it has two effects relevant to a graphics-stack reader:

- **Native Wayland support.** GIMP 3.0 runs natively under Wayland compositors as well as X11, rather than requiring XWayland; the release notes state this plainly: "GIMP 3.0 now runs natively on Wayland (though you can still run it on X11 as well)!" [Source](https://www.gimp.org/release-notes/gimp-3.0.html) This also brought improved HiDPI display scaling and tablet-input handling, both of which are consumed through GTK3's own input and scaling infrastructure rather than anything GEGL-specific.
- **CSS-based theming**, replacing GIMP's older theme format.

The canvas itself is still composited through GTK's own drawing surface rather than a dedicated OpenGL/Vulkan canvas widget — GIMP's GTK3 move did not add a GPU-accelerated canvas display path comparable to Krita's `KisOpenGLCanvas2` (§4). The GPU-relevant work in 3.0 is confined to GEGL's per-operation OpenCL kernels (§2.2), not to canvas presentation.

Independently of the toolkit change, GIMP 3.0 replaced the Script-Fu/Python-2-only plugin model with a **GObject-Introspection-based plugin API**, opening the extension surface to any language with GI bindings. Supported languages listed in the 3.0 release notes are Python 3 (replacing the old `python-fu` binding), JavaScript, Lua (marked experimental due to instabilities), Vala, Script-Fu (TinyScheme), and C, with `GimpProcedureDialog` auto-generating a settings UI from a plugin's declared parameters. [Source](https://www.gimp.org/release-notes/gimp-3.0.html) GIMP 3.0 also introduced **non-destructive editing (NDE)** for GEGL filters: rather than immediately merging a filter's result into the active layer, filters "stay active" and remain re-editable through an effects list until the user explicitly merges or discards them, with the effect stack preserved across save/load in native XCF files. [Source](https://www.gimp.org/release-notes/gimp-3.0.html)

## 4. Krita's Canvas: KisOpenGLCanvas2 and Instant Preview

Krita separates canvas *display* from image *processing* more sharply than GIMP does. The display layer is abstracted behind `KisCanvas2`, which delegates to one of two concrete backends: `KisOpenGLCanvas2`, an OpenGL-accelerated canvas requiring at least OpenGL 3.0, or `KisQPainterCanvas`, a software-rendered fallback used when a suitable OpenGL context is unavailable. [Source](https://docs.krita.org/en/general_concepts/colors/color_managed_workflow.html)

The brush engines and layer-compositing logic underneath this display layer remain CPU-bound — Krita does not run its stroke computation as shader code. What the OpenGL canvas *does* provide beyond simple display is **Instant Preview** (also called Level-of-Detail or "LOD" strokes internally), a feature originally funded through a 2015 Kickstarter campaign. When a brush stroke would be too slow to render live at full resolution — a large, heavily anti-aliased brush on a big canvas, for example — Krita renders the visible feedback against a lower-resolution proxy of the canvas while the full-resolution stroke computes in the background, swapping in the final result once ready. Instant Preview provides no benefit when the working zoom level already matches the canvas's native display resolution, since there is then no lower-resolution proxy to fall back to. [Source](https://docs.krita.org/en/reference_manual/preferences/display_settings.html)

### 4.1 Two Kinds of Tiles: KisTiledDataManager vs. KisTextureTileInfoPool

Krita's codebase contains two distinct notions of "tile" that are easy to conflate but serve different layers of the system:

- **`KisTiledDataManager`** is Krita's core pixel-storage mechanism: every paint layer's pixel data is held in a grid of fixed-size tiles, reported at **64×64 pixels**, which allows Krita to allocate, swap, and undo memory in bounded chunks rather than reallocating whole-layer buffers on every edit. This is the data-model analog of GEGL's `GeglTile` (§2.1), and it is entirely independent of whether the OpenGL or QPainter canvas backend is active.
- **`KisTextureTileInfoPool`** is a separate, GPU-facing pooling mechanism used only by `KisOpenGLCanvas2` to manage chunks of pixel data being uploaded to OpenGL textures for canvas display, with a default pooled tile-size parameter of **256**. This pool exists purely to make texture-upload allocation efficient for the display path; it has no bearing on how pixel data is stored or edited in the underlying `KisTiledDataManager`.

The two tiling schemes solve different problems — memory-bounded editable pixel storage versus efficient GPU texture upload — and their differing tile sizes (64×64 vs. a pooled granularity of 256) should not be read as inconsistency; they belong to different subsystems that happen to share the word "tile."

## 5. darktable's Per-Module OpenCL Pixel Pipeline

darktable's processing architecture centers on the **pixelpipe** (`dt_dev_pixelpipe`), a chain of **IOP** ("image operation") modules through which a RAW or other source image flows on its way to the screen or to an export. Each IOP module — exposure, white balance, denoising, sharpening, and dozens of others — is darktable's basic plugin unit, and each one is independently chainable and reorderable in the pipeline. Where GIMP's GEGL operations *optionally* carry an OpenCL kernel behind a global, off-by-default toggle, darktable's IOP modules more commonly ship a paired CPU (`.c`) and OpenCL (`.cl`) implementation, and OpenCL is auto-detected and used by default whenever darktable finds a usable device and driver at startup — not an opt-in setting the user must discover. darktable maintains separate pixelpipe instances for full-resolution export/processing versus the lower-resolution preview shown while editing, so preview responsiveness and final-quality output are computed through independently sized paths. [Source](https://docs.darktable.org/usermanual/4.6/en/special-topics/opencl/)

### 5.1 dt_tiling and VRAM-Constrained Processing

Because darktable routinely processes images from RAW sensors whose full resolution can exceed the VRAM available on a modest GPU, the pixelpipe includes an automatic tiling layer, the `dt_tiling` function family (invoked through `dt_dev_pixelpipe_process_tiling()`), which determines an optimal tile size for a given module and image and splits the OpenCL computation into overlapping tiles that are each processed and stitched back together — allowing GPU-accelerated modules to complete even when the full image would not fit in device memory in one pass. More recent darktable releases added a faster internal OpenCL tiling mode that specifically improves throughput on smaller-VRAM cards, where the earlier tiling implementation's overhead was proportionally more expensive. [Source](https://docs.darktable.org/usermanual/4.6/en/special-topics/opencl/)

### 5.2 Diagnosing OpenCL: darktable-cltest and Device Priority

darktable ships a standalone diagnostic binary, **`darktable-cltest`**, whose sole job is to check whether a usable OpenCL environment is present and print the same debug information that `darktable -d opencl` would emit at startup, then exit — a quick way to confirm OpenCL device detection without launching the full application. [Source](https://manpages.debian.org/testing/darktable/darktable-cltest.1.en.html)

On multi-GPU systems, device selection is controlled by the `opencl_device_priority` configuration parameter (default `*/!0,*/*/!0,*`): darktable scans available OpenCL devices in priority order and takes the first free one for each pixelpipe instance it needs to schedule, and a device can be prefixed with `+` to force GPU-only processing for that device (suspending rather than falling back to CPU when no permitted device is free). [Source](https://docs.darktable.org/usermanual/4.6/en/special-topics/opencl/multiple-devices/)

## 6. RawTherapee: The Deliberate CPU-Only Counterpoint

RawTherapee, a mature FOSS RAW processor comparable in scope to darktable, has **no OpenCL or other GPU-acceleration path** anywhere in its pipeline — every module, from demosaicing through the tone-curve and detail stages, runs on the CPU, parallelized with OpenMP across available cores. This is not an oversight or a stalled roadmap item; it is a standing architectural choice, made and maintained despite past developer interest in GPU acceleration. RawTherapee is a useful case study precisely because it is the counterexample among Linux RAW processors: a project with comparable feature scope to darktable that concluded predictable CPU multithreading was preferable to the driver and OpenCL-implementation fragmentation risk that GPU acceleration would introduce, a tradeoff darktable's own tiling and device-priority machinery (§5.1–§5.2) exists specifically to manage.

## 7. vkdt: A Vulkan-Native Compute-Graph Pipeline

**vkdt** (`github.com/hanatos/vkdt`) is an experimental, Vulkan-native raw-photography pipeline built as a from-scratch successor to darktable's processing model rather than a fork of darktable itself; its project tagline describes it as a "raw photography workflow that sucks less." [Source](https://github.com/hanatos/vkdt) Where darktable treats OpenCL as a per-module accelerator layered onto a CPU-first pipelpipe (§5), vkdt inverts the relationship: its processing model is a generic node graph — a directed acyclic graph where each node can have multiple inputs and outputs — compiled via topological sort directly into a single Vulkan command buffer. Every processing stage, not just a subset of accelerated modules, is implemented as GLSL/Vulkan compute or graphics work, executed from that one scheduled command buffer rather than dispatched module-by-module through a CPU-orchestrated OpenCL loop.

This architecture is what makes vkdt capable of workloads that sit outside darktable's typical single-still-image editing use case: because the whole graph is GPU-resident and cheap to re-execute, vkdt supports real-time animation and timelapse workflows, multi-frame image alignment, and highlight inpainting as graph operations rather than as separate offline tools. vkdt's ray-query-based lens simulation is discussed as a Vulkan ray-tracing case study in Ch135. Within this chapter, vkdt stands as the concrete example of the architectural step GIMP and Krita have not taken: a full compute-graph GPU pipeline replacing "accelerate select operations" with "the pipeline *is* the GPU program."

## 8. RAW Decoding as the Shared CPU-Bound Layer: RawSpeed and LibRaw

Beneath all of the GPU-acceleration differences above, RAW decoding itself is a layer every one of these applications keeps on the CPU. darktable's primary decoder is **RawSpeed** (`github.com/darktable-org/rawspeed`), a decode-only library — it does not extract metadata or embedded thumbnails, perform demosaicing, or do anything beyond turning a manufacturer's RAW container and its vendor-specific (often lossless-JPEG-based) compression into raw sensor data, including vendor-specific decode paths for the many mutually incompatible RAW formats camera makers ship. RawSpeed v3 measured 20–40%+ faster than v1 on modern high-megapixel files — for example, roughly 2× faster decode of Sony A7R IV files — but the speed gain is entirely within the CPU decode step; RawSpeed has no GPU path. Because RawSpeed does not support every camera darktable needs to read (Fujifilm's GFX 50S II being one cited example), darktable does not rely on RawSpeed exclusively: it falls back to **LibRaw** for cameras RawSpeed cannot decode, rather than presenting a single one-library-per-application mapping. [Source](https://github.com/darktable-org/rawspeed)

LibRaw itself (used directly by RawTherapee and as darktable's fallback) is likewise a CPU-only decode library. The practical upshot is that "decode on the CPU, then accelerate downstream processing however each application chooses to" is the one architectural pattern all five applications in this chapter share — GIMP's off-by-default OpenCL, Krita's display-only GPU use, darktable's per-module OpenCL, RawTherapee's pure-CPU pipeline, and vkdt's full Vulkan compute graph all begin from the same CPU-bound RAW-decode step before their designs diverge.

## Roadmap

### Near-term (6-12 months)

- **GIMP's post-3.2 direction points away from OpenCL, not toward expanding it.** Speaking at FOSDEM 2026, GIMP developer Ondřej Míchal described GEGL's existing OpenCL support (§2.2) as never having worked "in any meaningful manner" even before it was hidden from recent releases, and framed hardware acceleration for image operations as a priority built on "modern" GPU APIs rather than OpenCL, whose vendor support has been declining. No specific API (Vulkan or otherwise) or implementation timeline has been committed to publicly yet. [Source: FOSDEM 2026 coverage via LavX News](https://news.lavx.hu/article/gimp-s-post-3-2-roadmap-hardware-acceleration-cmyk-support-and-more)
- **darktable narrowed its OpenCL device support in 5.6.0 rather than expanding it.** The legacy AMD-APP OpenCL driver was blacklisted after developer Jens-Hanno Schwalm found it "had been blacklisted because leading to problems regularly" — the kind of driver-specific instability that §5.2's `darktable-cltest` diagnostic and device-priority mechanism exist to surface. Affected users can manually re-enable the driver via configuration, but there is no committed plan to restore default support. [Source: discuss.pixls.us — OpenCL GPU acceleration disabled in 5.6.0](https://discuss.pixls.us/t/opencl-gpu-acceleration-disabled-in-5-6-0/58805)

### Medium-term (1-3 years)

- **Krita's GPU-canvas work is headed toward mobile/QML, not a desktop Vulkan renderer.** The 2026 Krita roadmap describes continuing Alvin Wong's 2025 prototype of embedding an OpenGL-based canvas inside a QML application, targeting Krita's mobile build, rather than any new rendering backend for `KisOpenGLCanvas2` (§4) on desktop. The parallel Qt6 port ships alongside a Qt5 build starting with Krita 5.3/6.0 (March 24, 2026), but no Vulkan adoption is mentioned. [Source: Krita — 2026 Roadmap](https://krita.org/en/posts/2026/roadmap-2026/)
- **If GIMP's GPU-API pivot lands on Vulkan, it would still not converge with vkdt's architecture.** Even a Vulkan-based GIMP canvas, per the near-term item above, is aimed at accelerating zoom/pan/composite previews — the same category of display-only acceleration Krita already does via OpenGL (§4) — not at replacing GEGL's node graph with a compute-graph pixel pipeline the way vkdt (§7) replaces darktable's pixelpipe entirely. The two GPU-API adoptions, if both happen, would remain architecturally distinct.

### Long-term

- **RawTherapee's CPU-only position (§6) shows no sign of changing.** A GitHub issue requesting OpenCL support has been open on the RawTherapee repository since 2014 with no maintainer commitment to implement it, consistent with this chapter's framing of RawTherapee as a deliberate architectural counterpoint rather than a project trailing behind darktable by oversight. [Source: RawTherapee/RawTherapee issue #1678](https://github.com/Beep6581/RawTherapee/issues/1678)
- **vkdt is likely to remain the outlier architecture, not a preview of where the others are heading.** Krita's and GIMP's 2026 roadmaps both point toward incremental, display-layer GPU acceleration rather than pipeline-wide rewrites, and darktable's own 5.6.0 change tightened its existing OpenCL-as-accelerator model rather than loosening it in vkdt's direction. Barring a change in any of these projects' stated priorities, the OpenCL-as-accelerator pattern (§2, §5) is likely to remain the default for GIMP and darktable's main branch, with vkdt continuing as Johannes Hanika's experimental, feature-reduced alternative rather than an eventual replacement.

## 9. Integrations

- **Ch101 (Color Science and ICC Profile Pipeline)** — the lcms2 color transforms that GIMP, Krita, and darktable all call downstream of whichever pipeline architecture (GEGL, `KisOpenGLCanvas2`, or the darktable pixelpipe) produced their working-space pixel data.
- **Ch53 (Display Calibration and colord)** — how each application's final output is mapped to a calibrated display via `_ICC_PROFILE` atom consumption, independent of the GPU-acceleration model covered here.
- **Ch134 (OpenCL on Linux)** — darktable's OpenCL device selection, `dt_tiling`, and `darktable-cltest` diagnostics (§5) as this book's primary desktop-application case study for OpenCL, contrasted with GEGL's narrower, off-by-default use of the same API (§2.2).
- **Ch135 (Vulkan Ray Tracing)** — vkdt's ray-query-based lens simulation (§7), the same project referenced there for its `VK_KHR_ray_query` usage.
- **Ch42 (Blender — GPU Compositing and Rendering)** — a GPU-accelerated node-graph pipeline and OpenColorIO integration that goes further than GIMP's or Krita's GPU use, offering a useful contrast for readers coming from Blender's architecture.
- **Ch4 (GPU Memory Management)** — the VRAM constraints that darktable's `dt_tiling` (§5.1) works around by splitting OpenCL work into overlapping tiles.
- **Ch242 (DaVinci Resolve — Professional Color Grading and the Linux GPU Pipeline)** — a sibling Part XI chapter covering a full multi-backend (CUDA/OpenCL/Vulkan) GPU compute pipeline across an entire NLE, in contrast to this chapter's per-filter and per-module acceleration models.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
