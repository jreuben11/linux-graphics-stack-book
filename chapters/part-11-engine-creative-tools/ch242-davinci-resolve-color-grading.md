# Chapter 242: DaVinci Resolve — Professional Color Grading and the Linux GPU Pipeline

**Target audiences:** Graphics application developers studying how a professional, closed-source NLE structures GPU compute across decode, compositing, and color; Linux desktop and workstation engineers deploying DaVinci Resolve on RHEL/Rocky-family and community-patched distributions; imaging engineers comparing Resolve's multi-backend (CUDA/OpenCL) architecture against the OpenCL-as-accelerator and Vulkan-native models covered in Ch241.

---

## Table of Contents

1. [Overview: A Professional NLE's Multi-Backend GPU Architecture](#1-overview-a-professional-nles-multi-backend-gpu-architecture)
2. [Linux Distribution Support: The RHEL/Rocky-Centric .run Installer](#2-linux-distribution-support-the-rhelrocky-centric-run-installer)
3. [GPU Compute Backends: CUDA, OpenCL, and the Absence of Vulkan](#3-gpu-compute-backends-cuda-opencl-and-the-absence-of-vulkan)
   - [3.1 GPU Configuration and Multi-GPU Assignment](#31-gpu-configuration-and-multi-gpu-assignment)
4. [Free vs. Studio: GPU-Feature Gating](#4-free-vs-studio-gpu-feature-gating)
5. [Color Page: DaVinci Wide Gamut, ACES, and OpenColorIO in Fusion](#5-color-page-davinci-wide-gamut-aces-and-opencolorio-in-fusion)
6. [Fusion Page: A GPU Compute-Graph Compositor (and Where It Falls Back to CPU)](#6-fusion-page-a-gpu-compute-graph-compositor-and-where-it-falls-back-to-cpu)
7. [ResolveFX and the Neural Engine: Shader Effects vs. AI-Driven Effects](#7-resolvefx-and-the-neural-engine-shader-effects-vs-ai-driven-effects)
8. [Video Codec Support on Linux: ProRes, and the Hardware Decode Gap](#8-video-codec-support-on-linux-prores-and-the-hardware-decode-gap)
9. [Render Cache Architecture: Smart Cache, User Cache, and GPU Contention](#9-render-cache-architecture-smart-cache-user-cache-and-gpu-contention)
10. [Integrations](#10-integrations)

---

## 1. Overview: A Professional NLE's Multi-Backend GPU Architecture

DaVinci Resolve is Blackmagic Design's professional non-linear editor, colorist tool, and compositor, and it is one of the few applications covered in Part XI that is closed-source. It is included in this book because its GPU architecture is instructive by contrast with Ch241's GIMP/Krita/darktable survey: rather than one application choosing one GPU-acceleration posture, Resolve is a single application whose different pages (Edit, Color, Fusion, Deliver) each lean on GPU compute differently, all funneled through a shared, user-selectable compute backend — CUDA on NVIDIA, OpenCL on AMD and Intel. Unlike the OpenCL-as-per-module-accelerator pattern in darktable (Ch241 §5) or the fully GPU-native compute graph of vkdt (Ch241 §7), Resolve's architecture is closer to "the whole application is GPU-resident, and the backend is swappable," with the Color and Fusion pages differing sharply in how completely they actually achieve that.

This chapter also corrects a common assumption worth stating plainly up front: Resolve does **not** expose a Vulkan GPU processing mode. Its GPU Processing Mode setting on Linux is a choice between **CUDA** and **OpenCL** (Metal is macOS-only), and public release notes and community documentation through DaVinci Resolve 20.x show no Vulkan backend having been added. [Source](https://wiki.archlinux.org/title/DaVinci_Resolve) Readers coming from Mesa's RADV Vulkan driver work on Linux should not expect Resolve to route AMD compute through it — AMD GPUs on Linux go through OpenCL.

## 2. Linux Distribution Support: The RHEL/Rocky-Centric .run Installer

Blackmagic Design ships DaVinci Resolve for Linux as a self-extracting `.run` installer (`chmod +x DaVinci_Resolve_<version>_Linux.run && sudo ./DaVinci_Resolve_<version>_Linux.run -i`), with no `.deb` or `.rpm` package provided directly. Formal Blackmagic support is narrow: officially supported configurations center on the RHEL/CentOS/Rocky Linux 8.x family (Rocky Linux 8.6 has been cited as a specific supported baseline) paired with modern NVIDIA GPUs, plus Ubuntu 22.04 LTS. [Source](https://forum.blackmagicdesign.com/viewtopic.php?f=21&t=194581)

In practice, the Linux Resolve community runs it far more broadly than that support matrix through unofficial patch layers: Fedora, Arch, and Debian/Mint-derived installs are common, and community tooling such as `davincibox` packages Resolve with the necessary library shims for non-RHEL distributions in a container-like wrapper. [Source](https://github.com/zelikos/davincibox/blob/main/README.md) None of this is Blackmagic-supported; it is the same "runs, with effort, outside the vendor's tested matrix" situation familiar from other proprietary Linux creative and CAD software.

## 3. GPU Compute Backends: CUDA, OpenCL, and the Absence of Vulkan

DaVinci Resolve's GPU Processing Mode is configured in Preferences → System → Memory and GPU. On Linux, the practical choice is binary: NVIDIA GPU owners select **CUDA**, and AMD (and Intel) GPU owners select **OpenCL** — the same OpenCL API used as the per-module accelerator in darktable (Ch241 §5), here instead driving effects processing, scaling, and color transforms across the entire application. [Source](https://vagon.io/gpu-guide/how-to-use-gpu-on-davinci-resolve-studio-17) There is no third, Vulkan-based option; despite long-standing community requests on the Blackmagic forums for a Vulkan backend (motivated by Linux AMD users wanting to bypass ROCm/OpenCL driver issues), no such backend has shipped as of DaVinci Resolve 20.x. [Source](https://forum.blackmagicdesign.com/viewtopic.php?f=21&t=111279)

### 3.1 GPU Configuration and Multi-GPU Assignment

On a single-GPU system, that one GPU handles both the UI/display compositing and the compute-heavy effects/color processing. With two or more GPUs installed, Resolve exposes a **"Use Display GPU For Compute"** checkbox to opt the display GPU into shared compute duty rather than reserving it purely for UI rendering, and a **GPU Selection Mode** (Auto or Manual) governs which installed GPUs participate in compute at all — Manual mode surfaces an explicit per-GPU checklist. [Source](https://vagon.io/gpu-guide/how-to-use-gpu-on-davinci-resolve-studio-17) This manual, per-GPU assignment model is one of the more explicit multi-GPU configuration surfaces among Linux creative applications; most of the apps in Ch241 have no equivalent multi-GPU load-distribution UI at all.

## 4. Free vs. Studio: GPU-Feature Gating

DaVinci Resolve ships as a free edition and a paid **Studio** edition (a one-time license), and the split is substantially a GPU-feature split rather than a purely feature-count split:

- **Multi-GPU support is Studio-only.** The free edition is restricted to a single GPU; only Studio can distribute compute work across two or more installed GPUs via the mechanism in §3.1. [Source](https://www.storyblocks.com/resources/tutorials/davinci-resolve-free-vs-studio)
- **Noise reduction quality is GPU-gated.** The free edition's noise reduction runs on the CPU; Studio moves noise reduction to the GPU and adds temporal and spatial modes (analyzing multiple frames rather than one) that the free edition does not offer at all. [Source](https://www.storyblocks.com/resources/tutorials/davinci-resolve-free-vs-studio)
- **GPU-accelerated H.264/H.265 hardware encode is Studio-only**, alongside full HDR grading (Dolby Vision) and the broader ResolveFX/Neural Engine library discussed in §7. [Source](https://www.storyblocks.com/resources/tutorials/davinci-resolve-free-vs-studio)

The practical effect is that free-edition Resolve on Linux is closer to a single-GPU, CPU-assisted tool, while Studio is the version that actually exercises the multi-GPU and GPU-resident-effects architecture this chapter describes.

## 5. Color Page: DaVinci Wide Gamut, ACES, and OpenColorIO in Fusion

Resolve's Color page applies primary and secondary color correction, LUTs, and grades as GPU-executed operations against the timeline's working color space. Resolve 17 introduced **DaVinci Wide Gamut** and **DaVinci Intermediate**, an internal working color space and gamma designed to unify grading across footage originating in different camera-vendor log formats (S-Log3, LogC, Blackmagic RAW, and others) without clipping highlight or shadow detail during intermediate exposure adjustments; Project Settings' Color Management panel exposes this as **"DaVinci YRGB Color Managed"**, with the timeline color space set to DaVinci Wide Gamut Intermediate. Working in these wide spaces requires 32-bit RGBA image processing precision — the default 8-bit pipeline does not carry enough headroom. [Source](https://blog.frame.io/2024/01/08/color-management-nodes-davinci-resolve/)

Separately, Resolve's Color Science dropdown offers **ACES** color management (ACEScc/ACEScct), with DaVinci Resolve 20 adding ACES 2.0 support in beta. [Source](https://sharktacos.github.io/OpenColorIO-configs/docs/Resolve.html) General-purpose OpenColorIO config import for the Color page itself is not a standard feature — it has been a recurring community feature request on the Blackmagic forums rather than shipped functionality. [Source](https://forum.blackmagicdesign.com/viewtopic.php?f=33&t=133121) Where OCIO does appear as a first-class citizen is one page over, in Fusion.

## 6. Fusion Page: A GPU Compute-Graph Compositor (and Where It Falls Back to CPU)

The Fusion page is Resolve's node-based compositor — originally the standalone Fusion Studio product, now integrated as a page alongside Edit, Color, and Deliver. Each node in a Fusion composition represents an operation (color correction, masking, keying, tracking, particles, text) connected in a directed graph, and Blackmagic states Fusion "was architected from the ground up to exploit GPU parallel processing," with compositing running primarily on the graphics card. [Source](https://www.blackmagicdesign.com/products/davinciresolve/fusion)

This GPU-first framing has a real limit, though: it applies cleanly to 2D compositing, but **3D rendering within Fusion shifts almost entirely back to the CPU**. Even within Fusion's particle system — one of its GPU-oriented subsystems — only the final `pRender` node ever executes on the GPU, and only when configured in 3D mode; every upstream node feeding into it is computed on the CPU. [Source](https://jayaretv.com/fusion/list-of-fusion-nodes-that-are-gpu-accelerated/) Fusion is therefore best understood as a GPU-accelerated 2D compositing graph with a CPU-bound 3D subsystem bolted on, rather than a uniformly GPU-resident compute graph in the way vkdt's pipeline is (Ch241 §7).

Fusion is also where **OpenColorIO** is exposed directly to users: dedicated OCIO nodes in the Effects Library's Color category apply OpenColorIO color-space transforms within a composition, and Fusion's ACES color management is itself implemented through the OCIO framework — a more literal, config-driven use of OCIO than the Color page's built-in ACES/DaVinci Wide Gamut options in §5. [Source](https://www.steakunderwater.com/VFXPedia/__man/Resolve18-6/DaVinciResolve18_Manual_files/part1916.htm)

## 7. ResolveFX and the Neural Engine: Shader Effects vs. AI-Driven Effects

Resolve Studio ships **over 100 GPU- and CPU-accelerated ResolveFX** — blurs, lens flares, film grain, image restoration, and stylize effects — implemented as conventional shader-style GPU compute against the active backend (CUDA or OpenCL). [Source](https://www.blackmagicdesign.com/products/davinciresolve/studio) Layered on top of these is the **DaVinci Neural Engine**, Blackmagic's umbrella term for deep-learning-driven features: facial recognition and **Face Refinement** (skin retouching, eye-bag removal, subtle relighting), **Magic Mask** (click-guided object/person segmentation and tracking, generating rotoscoping masks from a few user strokes), **Speed Warp** (optical-flow-based retiming that additionally uses the Neural Engine to detect and correct areas where flow-based interpolation would look "rubbery"), and Smart Reframe/SuperScale. [Source](https://vagon.io/blog/davinci-resolve-neural-engine-guide)

The distinction that matters architecturally is that ResolveFX are general GPU shader/compute passes running on whichever backend is selected in §3, while Neural Engine features run trained models — Blackmagic markets a "300% speed boost" for Neural Engine features attributable to hardware-specific acceleration paths (e.g., NVIDIA RTX/Tensor Core acceleration where available), falling back to general GPU compute on hardware without dedicated tensor/AI-accelerator paths. [Source](https://vagon.io/blog/davinci-resolve-neural-engine-guide) Neural Engine features and full ResolveFX are Studio-exclusive (§4); the free edition's effects processing stays on the CPU-heavier, non-AI subset.

## 8. Video Codec Support on Linux: ProRes, and the Hardware Decode Gap

For years, ProRes **encoding** outside licensed Apple hardware was effectively unavailable on Windows and Linux. Blackmagic added ProRes encode support to Resolve on both Windows and Linux in **version 19.1.4**, released March 21, 2025. [Source](https://www.pugetsystems.com/blog/2025/03/25/prores-encoding-in-davinci-resolve-19-1-4/) ProRes RAW **decoding** followed later, arriving simultaneously across macOS, Windows, and Linux in **Resolve 20.2**, and camera-RAW decoding including ProRes RAW works identically in the free and Studio editions. [Source](https://www.pugetsystems.com/blog/2025/03/25/prores-encoding-in-davinci-resolve-19-1-4/)

Hardware-accelerated H.264/H.265 support on Linux is comparatively weak and worth flagging as a real limitation rather than glossing over it: Resolve on Linux supports **NVENC hardware encode** for NVIDIA GPUs, but hardware **decode** of H.264/H.265 on Linux is, per community reports, effectively unsupported in current Resolve builds — a gap relative to Windows and macOS, where hardware decode is available (subject to the usual bit-depth and chroma-subsampling caveats that constrain hardware decode generally). [Source](https://www.pugetsystems.com/labs/articles/what-h-264-and-h-265-hardware-decoding-is-supported-in-davinci-resolve-studio-2122/) Linux Resolve users working with H.264/H.265 camera-native footage therefore commonly transcode to an intermediate codec (ProRes or DNxHR) before editing, precisely to route around this decode gap rather than relying on the NVDEC/VA-API hardware paths that other Linux media applications in this book (Ch26) use directly.

## 9. Render Cache Architecture: Smart Cache, User Cache, and GPU Contention

Resolve's render cache bakes computationally expensive effects, grades, and timeline operations into new intermediate media, which plays back in real time in place of the original source whenever the live GPU pipeline can't keep up. Two modes are available: **Smart Cache**, which automatically caches clip formats, grading operations, and effects Resolve judges too processor-intensive for real-time playback with no user configuration required; and **User Cache**, which caches only clips/effects the user explicitly marks, plus a configurable auto-cache list (transitions, composites, Fusion effects) set in the Master Settings of Project Settings. [Source](https://www.steakunderwater.com/VFXPedia/__man/Resolve18-6/DaVinciResolve18_Manual_files/part250.htm) Cache generation is not instantaneous by default — Resolve waits roughly five seconds after playback stops (or begins caching live during playback) before writing cache media, and cache files should be directed to fast local storage since cache read/write competes with the same disk I/O as source media playback. [Source](https://beginnersapproach.com/davinci-resolve-render-cache/)

This cache system exists specifically because caching and live GPU compute contend for the same device: when the on-screen GPU status indicator goes red — signaling that Timeline effects, Color-page grading, or processor-intensive source media cannot be composited in real time on the currently assigned GPU(s) — enabling Smart or User Cache is the standard remedy, trading upfront GPU/disk time for guaranteed real-time playback later. On the multi-GPU configurations available in Studio (§3.1), cache generation and live color/Fusion GPU work can be spread across separate GPUs specifically to avoid this contention, which is part of why Resolve's explicit per-GPU assignment model in §3.1 matters in practice and not just as a configuration curiosity.

## 10. Integrations

- **Ch108 (ROCm and HIP)** — the AMD compute stack that Resolve's Linux OpenCL backend runs on top of; §3 notes the long-standing (and still unfulfilled) community request for a Vulkan alternative to this OpenCL/ROCm dependency.
- **Ch134 (OpenCL on Linux)** — Resolve's OpenCL GPU Processing Mode (§3) alongside darktable's per-module OpenCL pipeline from Ch241 §5, as two different points on the OpenCL-in-a-Linux-creative-app spectrum: whole-application backend selection here, versus per-module `.cl` kernels there.
- **Ch66 (CUDA Runtime, Streams, and NVRTC)** — the NVIDIA compute path selected via GPU Processing Mode (§3) for Resolve's effects, color, and (Studio-only) noise-reduction processing.
- **Ch26 (Hardware Video)** — the NVDEC/VA-API decode paths that other Linux media applications use directly, contrasted with Resolve's comparatively weak Linux H.264/H.265 hardware decode support (§8), which pushes many Linux colorists toward ProRes/DNxHR transcoding as a workaround.
- **Ch101 (Color Science and ICC Profile Pipeline)** — the ICC/OCIO-adjacent color-transform concepts underlying DaVinci Wide Gamut, ACES, and the OCIO nodes exposed directly in Fusion (§5–§6).
- **Ch241 (GIMP, Krita, and darktable — GPU-Accelerated Raster and RAW Photo Editing on Linux)** — the sibling Part XI chapter on per-filter/per-module GPU acceleration models, in contrast to this chapter's whole-application, user-selectable CUDA/OpenCL backend spanning an entire NLE, and to vkdt's fully GPU-native compute graph, which Fusion's CPU-bound 3D subsystem (§6) does not match.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
