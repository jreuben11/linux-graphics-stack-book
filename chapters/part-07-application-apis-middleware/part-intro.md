# Part VII — Application APIs & Middleware

The Linux graphics stack divides roughly into two halves: the lower half (Parts I–VI) builds up the substrate — kernel **DRM/KMS**, Mesa driver internals, **Wayland** compositor architecture, display pipelines, and GPU synchronisation primitives. This part begins the upper half. It is the layer where application-facing APIs — **Vulkan**, **EGL**, **OpenCL**, **VA-API**, **OpenXR**, **PipeWire** — are consumed by real software: games, media players, toolkits, ML inference engines, terminal emulators, and web browsers. The chapters here do not repeat what kernel internals exist; they explain exactly which kernel interfaces each API layer calls, how buffers cross subsystem boundaries without CPU copies, and what tradeoffs govern API choice for a given workload.

The unifying mechanism throughout this part is **DMA-BUF**: every major data flow — decoded video frames, captured camera images, rendered Vulkan swapchain images, PipeWire screen-capture streams — ultimately crosses subsystem boundaries as a **DMA-BUF** file descriptor carrying a **DRM format modifier**. Understanding how that mechanism is negotiated and what happens when it falls back to a CPU copy is the connective tissue that ties every chapter together.

## Chapters in This Part

**Chapter 24 — Vulkan and EGL for Application Developers** is the entry point for graphics application developers. It explains what happens beneath `vkCreateSwapchainKHR` on Linux — which kernel ioctls fire, how **linux-dmabuf** format and modifier negotiation proceeds on **Wayland**, and how timeline semaphores map onto **DRM sync objects** to close the explicit synchronisation loop from GPU to display. **EGL** is treated as a first-class citizen, not a legacy path, because it underpins headless rendering, **GBM**-backed surfaces, and **VA-API** interop.

**Chapter 25 — GPU Compute: OpenCL, CUDA, and ROCm** maps the fragmented Linux compute landscape: **OpenCL** ICD discovery, **rusticl** on **Gallium**, AMD **ROCm**/**HIP**, Intel **Level Zero**/**oneAPI**, and **Vulkan** compute pipelines. It covers the cross-API **DMA-BUF** interop that lets compute results feed graphics pipelines without CPU round-trips, and explains AMD's unified memory model and **Linux HMM** with concrete implications for **APU** platforms like the **Steam Deck**.

**Chapter 26 — Hardware Video: VA-API, V4L2, and libcamera** traces the complete hardware video pipeline from camera sensor or container to display. The chapter covers the **VA-API** surface model and driver dispatch, stateless **V4L2** codec drivers with the **Request API**, **GStreamer** zero-copy pipeline construction, **PipeWire** as a multimedia session layer, and **Vulkan Video** as the strategic forward API. The recurring theme is which API boundaries survive without a **DMA-BUF** export and where copies still occur.

**Chapter 27 — VR & AR: OpenXR and the Linux XR Stack** covers the **OpenXR** programming model from `XrInstance` creation through the **Monado** runtime's internal architecture. It explains how **Monado** acquires exclusive headset ownership via **DRM leasing** (`wp_drm_lease_v1`), drives direct-mode scan-out through **GBM** and `drmModeAtomicCommit`, and implements Asynchronous Timewarp as a **Vulkan** compute shader. Readers who understand Chapter 24 will see how **XrSwapchain** maps onto **Vulkan** swapchain concepts and how **V4L2** camera frames feed inside-out tracking.

**Chapter 38 — PipeWire: Unified Audio/Video Session Layer** explains the **SPA** plugin architecture, the node/link/port graph scheduler, and the **WirePlumber** session policy daemon in depth. It focuses on the three high-value data flows for graphics engineers: **Wayland** screen capture via `org.freedesktop.portal.ScreenCast`, zero-copy camera delivery from **V4L2** and **libcamera** sources, and GPU frame injection back into the graph as `SPA_DATA_DmaBuf` buffers. It also details **DRM format modifier** negotiation in **SPA** pods and the conditions under which the pipeline falls back to `SPA_DATA_MemFd`.

**Chapter 39 — Qt and GTK Rendering Pipelines** explains how the two dominant Linux GUI toolkits map their scene graphs to **Vulkan** and **OpenGL ES** on **Wayland**. It covers **Qt6**'s **QRhi** rendering hardware interface, the **Qt Quick** scene graph render loop, the **qsb** shader baker pipeline, **GTK4**'s **GskRenderer** unified GPU backend with its **GskVulkanRenderer** and **GskNglRenderer** paths, and how both toolkits add explicit GPU synchronisation via `wp_linux_drm_syncobj_v1` for **NVIDIA** compatibility. Font and text rendering — **FreeType**, **HarfBuzz**, **Pango**, and glyph atlas management — is introduced here and expanded in Chapter 47.

**Chapter 47 — Font and Text Rendering Pipeline** is a dedicated deep-dive into the text path that every toolkit, browser, and terminal relies on. It covers **FreeType 2** hinting modes, **HarfBuzz** OpenType shaping for complex scripts, **fontconfig** font matching, **Cairo** compositing, **Pango** layout, and glyph atlas strategies in **Qt**, **GTK4**, **Skia**, and **WebRender**. The chapter also explains why **LCD subpixel rendering** is disabled in composited **Wayland** sessions and how **OpenType variable fonts** challenge atlas cache-key design.

**Chapter 50 — Vulkan Video Extensions** covers the **Khronos** Vulkan Video extension family — `VK_KHR_video_queue`, `VK_KHR_video_decode_queue`, `VK_KHR_video_encode_queue`, and codec extensions for **H.264**, **H.265**, and **AV1** — from specification rationale through **RADV** and **ANV** driver implementation in Mesa to **FFmpeg** `hwaccel` integration. It explicitly compares **Vulkan Video** with **VA-API** on API complexity, render-pipeline integration, and hardware support, giving readers a principled basis for API selection.

**Chapter 76 — Modern Vulkan Extensions** documents the post-1.2 Vulkan extension landscape that defines contemporary engine and driver work: **VK_KHR_dynamic_rendering**, bindless descriptor indexing (`VK_EXT_descriptor_indexing`), mesh shaders (`VK_EXT_mesh_shader`), cooperative matrices (`VK_KHR_cooperative_matrix`), shader objects (`VK_EXT_shader_object`), graphics pipeline libraries, and extended dynamic state. It explains the extension promotion path from vendor-specific to `VK_EXT_` to `VK_KHR_` to core, and gives per-feature adoption status across **RADV**, **ANV**, and **NVK**.

**Chapter 96 — libcamera and the Linux Camera Stack** covers the unified camera abstraction that replaced a decade of fragmented SoC HALs. It explains **libcamera**'s **PipelineHandler** per-platform subclass model, the **IPA** sandbox for 3A algorithms, the **V4L2 Media Controller** graph configuration sequence, **FrameBuffer**/**DMA-BUF** integration, and how **PipeWire**'s camera portal makes frames available to WebRTC, **GStreamer**, and **OBS** through a single security gate. Hardware coverage includes **Raspberry Pi** (Unicam/PiSP), Intel **IPU3/CIO2**, and **Rockchip RKISP1/RKISP2**.

**Chapter 111 — Flatpak and Sandboxed Graphics Applications** is the chapter for application developers packaging GPU-accelerated software in **Flatpak** and for portal and compositor implementers who mediate that access. It explains why GPU sandboxing is structurally harder than CPU sandboxing: Mesa DRI drivers are architecture-specific shared objects whose ABI must version-match the host kernel driver, but **bubblewrap** establishes a separate mount namespace in which host library paths are absent. The chapter traces the complete data path from a Flatpak manifest's `finish-args` section through the bubblewrap namespace, the **GL extension** mounting mechanism (e.g., `org.freedesktop.Platform.GL.default`), **Vulkan ICD** discovery inside the sandbox, the **xdg-desktop-portal** ScreenCast and camera portals, and **VA-API** video decode under the sandbox. It closes with **Electron**/**CEF** nesting under Flatpak, a comparison with **Snap** and **AppImage**, and packaging best practices. Crucially, the chapter quantifies the security trade-offs: because the render node (`/dev/dri/renderD128`) must be bind-mounted into the sandbox, a compromised sandboxed application retains full GPU context access — the sandbox boundary stops at the kernel DRM interface, not at the GPU hardware.

**Chapter 114 — OpenCV and GPU-Accelerated Computer Vision on Linux** covers OpenCV as the terminal consumer of nearly every API thread in this part. It explains the **cv::UMat** / **Transparent API (T-API)** dispatch mechanism that routes image operations to an **OpenCL** backend (via **rusticl**, **intel-compute-runtime**, or **pocl**) without changing calling code, the **cv::cuda::GpuMat** and **Stream** model for NVIDIA hardware, and **VA-API** decode integration for zero-copy camera-to-inference pipelines. The chapter traces how a camera frame delivered by **libcamera** or **V4L2** can traverse the entire stack — decoded by **VA-API**, passed as a **DMA-BUF** to an **OpenCL** kernel running on the same GPU, processed by OpenCV's T-API, and composited into a **Wayland** surface via **EGL** — without ever touching CPU memory. The **cv::dnn** backend selector and **ONNX** inference path are covered with concrete backend configuration examples for Intel, AMD, and NVIDIA hardware. Integration with **GStreamer** pipelines is examined in detail, as OpenCV's `cv::VideoCapture` GStreamer backend is the primary zero-copy ingestion path for production vision systems.

**Chapter 106 — Vulkan Memory Model** covers `VK_KHR_vulkan_memory_model`, the formal memory model that governs visibility and ordering of Vulkan memory operations across shader invocations, queues, and devices, explaining acquire/release semantics, availability/visibility chains, and how the model interacts with `VkBarrier2` and timeline semaphores.

**Chapter 127 — Mesh Shaders and Variable Rate Shading** documents `VK_EXT_mesh_shader` (task and mesh shader stages replacing the vertex pipeline) and `VK_KHR_fragment_shading_rate` (per-tile, per-primitive, and per-draw shading rate control), covering RADV, ANV, and NVK driver implementations and game-engine integration patterns.

**Chapter 133 — Vulkan Compute Queues and Async Compute** explains how to use Vulkan compute queues independently of the graphics queue, covering queue family selection, `VkSubmitInfo2` timeline semaphore chaining, async compute overlap with graphics work, and per-vendor queue topology on AMD (SDMA + ACE queues), Intel (CCS), and NVIDIA (copy engines).

**Chapter 135 — Vulkan Ray Tracing** provides a comprehensive reference for `VK_KHR_ray_tracing_pipeline`, `VK_KHR_acceleration_structure`, and `VK_KHR_ray_query`: acceleration structure lifecycle, shader binding table layout, ray generation / intersection / any-hit / closest-hit / miss shader stages, and RADV/ANV driver implementation details.

**Chapter 141 — Vulkan Cooperative Matrices** covers `VK_KHR_cooperative_matrix` — the Vulkan API for matrix-multiply-accumulate operations on GPU tensor cores — including matrix type layouts, supported element types, GLSL/SPIR-V cooperative matrix extensions, and integration with ML inference workloads on AMD RDNA3 and NVIDIA Tensor Cores.

**Chapter 148 — Vulkan Synchronisation Reference** is a comprehensive reference chapter for Vulkan synchronisation primitives: pipeline stages, access masks, image and buffer memory barriers (`VkImageMemoryBarrier2`, `VkBufferMemoryBarrier2`), events, timeline semaphores, and `VkFence`, with worked examples of common synchronisation patterns and validation layer guidance.

**Chapter 150 — EGL Architecture and DMA-BUF Interop** provides an in-depth treatment of the EGL API beyond the survey in Chapter 24: EGL extensions for DMA-BUF import/export (`EGL_EXT_image_dma_buf_import`, `EGL_EXT_image_dma_buf_import_modifiers`), the EGLDevice and EGLOutput APIs for headless and direct-to-display rendering, and surfaceless EGL contexts for compute and video workloads.

**Chapter 152 — Rust GPU Ecosystem: wgpu, ash, and gpu-allocator** surveys the Rust-language GPU programming landscape on Linux: the `wgpu` safe GPU abstraction crate, `ash` raw Vulkan bindings, `gpu-allocator` for Vulkan/D3D12 memory management, `naga` shader compiler IR, and how Rust GPU crates integrate with the Mesa Vulkan driver stack.

**Chapter 154 — GPU-Driven Rendering** covers the architectural shift from CPU-driven draw call submission to GPU-driven pipelines where the GPU itself culls and dispatches geometry: indirect draw commands (`vkCmdDrawIndirect`, `vkCmdDrawIndexedIndirectCount`), GPU culling with compute shaders, meshlet-based rendering with `VK_EXT_mesh_shader`, and multi-draw indirect best practices on RDNA and Xe hardware.

**Chapter 157 — Vulkan Descriptor Binding Models** compares and contrasts the four Vulkan descriptor binding approaches — classic descriptor sets, push descriptors (`VK_KHR_push_descriptor`), descriptor buffers (`VK_EXT_descriptor_buffer`), and bindless/bindful hybrid models — with per-vendor performance characteristics and guidance on when to use each.

**Chapter 165 — Vulkan Video: Hardware Encode and Decode** covers the `VK_KHR_video_queue`, `VK_KHR_video_decode_queue`, `VK_KHR_video_encode_queue`, and codec extension family for H.264, H.265, AV1, and VP9 on Linux, with RADV and ANV implementation details, FFmpeg hwaccel integration, and a comparison with VA-API for decode pipeline selection.

**Chapter 173 — VK\_EXT\_shader\_object: Pipeline-Free Shader Binding in Vulkan** covers the `VkShaderEXT` object introduced by `VK_EXT_shader_object` (ratified EXT, revision 1, 2023): how shaders are compiled independently of pipeline state via `vkCreateShadersEXT`, bound per-stage with `vkCmdBindShadersEXT`, and serialised for instant reload via `vkGetShaderBinaryDataEXT`. The chapter explains why the extension requires extensive dynamic state (`VK_EXT_extended_dynamic_state` family) and must be used inside `VkRenderingInfo` (i.e. requires `VK_KHR_dynamic_rendering`). Driver implementation status is documented: RADV enabled by default in Mesa 24.1 (`src/amd/vulkan/radv_shader_object.c`), NVK added support in February 2024, ANV landed in Mesa 25.3-devel. The chapter includes a complete before/after migration example converting a static graphics pipeline to shader objects, and performance guidance from the spec's conformance floor (≤150% CPU cost vs. fully-static pipelines).

## How the Chapters Interrelate

The part has a clear dependency spine. **Chapter 24** must be read before most others: it establishes the **Vulkan** device model, the **EGL**/**GBM** context creation path, **linux-dmabuf** modifier negotiation, and **DRM sync objects**. Every subsequent chapter that sends frames to the display or imports **DMA-BUF** handles — Chapters 25, 26, 27, 38, 39, 50, 76, 111, and 114 — builds directly on those foundations.

**Chapter 25** (compute) provides a survey of the full compute landscape — **OpenCL** ICD discovery, **ROCm**/**HIP** overview, **Level Zero**/**oneAPI**, and Vulkan compute — establishing **DMA-BUF** interop patterns between compute, video, and graphics APIs. The deep treatment of the AMD ML stack (HIP, MIOpen, PyTorch ROCm, amdkfd, MI300X) has moved to **Chapter 48 in Part XX (AI/ML Inference)**. Readers interested in the AMD compute stack should read Chapter 25 here first, then Chapter 48 in Part XX. **Chapter 114** (OpenCV) builds on Chapter 25's compute landscape and, for DNN inference on ROCm, requires Chapter 48 (Part XX).

**Chapter 26** (video) and **Chapter 50** (Vulkan Video) are explicitly linked. Chapter 26 covers **VA-API** as the established production path; Chapter 50 covers **Vulkan Video** as its strategic successor, and its comparison section (§8) directly references Chapter 26's surface model. The **PipeWire** session layer section of Chapter 26 overlaps intentionally with **Chapter 38**, which expands the **SPA** internals; Chapter 38 is the definitive reference, Chapter 26's §7 is a summary oriented toward video pipeline authors. **Chapter 114** depends on Chapter 26 for VA-API decode, Chapter 38 for PipeWire-delivered camera frames, and Chapter 96 for the libcamera integration.

**Chapter 27** (OpenXR/VR) depends on Chapter 24 for **Vulkan** swapchain concepts, and on Parts II–III for **DRM leasing** and atomic modesetting mechanics. Its tracking pipeline (§6) uses **V4L2** camera capture in the same mode described in Chapter 26 §5.

**Chapters 39 and 47** form a natural pair on the toolkit rendering side: Chapter 39 covers **Qt6** and **GTK4** scene graph and GPU submission paths, introducing the text pipeline at a high level; Chapter 47 then provides the complete **FreeType**/**HarfBuzz**/**Pango** reference that underpins what Chapter 39 describes. Browser and terminal developers reading Chapter 47 will find the atlas management and **Wayland** subpixel sections directly relevant to **Skia**, **WebRender**, **Ghostty**, and **foot**.

**Chapter 76** (modern Vulkan extensions) can be read independently as a reference chapter, but it is most useful after Chapter 24 has established the core **Vulkan** device and pipeline model. Features like **VK_KHR_cooperative_matrix** tie back to ML workloads covered in Part XX; mesh shaders and pipeline libraries are relevant to the toolkit rendering paths in Chapter 39.

**Chapter 96** (libcamera) depends on the **V4L2 Media Controller** concepts introduced in Chapter 26 §5, and its **PipeWire** integration section (§8) is a direct consumer of the session model established in Chapter 38. It stands as a self-contained deep-dive for embedded and camera-oriented developers.

**Chapter 111** (Flatpak) is best read after Chapter 24, which establishes the Vulkan ICD loader and the EGL/GBM context model that the GL extension mechanism replicates inside the sandbox, and after Chapter 38, whose PipeWire portal model (the ScreenCast and camera portals) is the authoritative access path for screen capture and camera within the Flatpak security boundary. It also intersects Chapter 26 (VA-API decode under the sandbox) and Chapter 96 (libcamera frames through the portal). Readers packaging vision applications in Flatpak will need Chapter 111 before applying the patterns in Chapter 114.

**Chapter 114** (OpenCV) is designed as an integration chapter — it synthesises the camera, compute, video, and EGL/OpenGL threads from Chapters 24, 25, 26, 38, 96, and Part XX's Chapter 48 into a single end-to-end vision pipeline narrative. It can be read after any of those chapters and will reinforce how the individual API pieces compose. Its Flatpak packaging section cross-references Chapter 111 directly.

The shared data structures that thread through every chapter are `drm_prime` **DMA-BUF** file descriptors, **DRM format modifiers** (the 64-bit values like `DRM_FORMAT_MOD_LINEAR` or `AMD_FMT_MOD_*`), **DRM sync objects** (`drm_syncobj` / timeline semaphores), and **VkImage** as the universal GPU-resident buffer type. Understanding how these four primitives are negotiated, exported, imported, and synchronised across API boundaries is the practical skill this part conveys.

```mermaid
graph TD
    Ch24["Ch24: Vulkan & EGL\n(foundation)"]
    Ch25["Ch25: GPU Compute\n(OpenCL / ROCm survey / Vulkan compute)"]
    Ch26["Ch26: Hardware Video\n(VA-API / V4L2 / GStreamer)"]
    Ch27["Ch27: VR & AR\n(OpenXR / Monado)"]
    Ch38["Ch38: PipeWire\n(session layer)"]
    Ch39["Ch39: Qt & GTK\n(toolkit rendering)"]
    Ch47["Ch47: Font & Text\n(FreeType / HarfBuzz / Pango)"]
    Ch50["Ch50: Vulkan Video\n(VK_KHR_video_*)"]
    Ch76["Ch76: Modern Vulkan Extensions\n(dynamic rendering / mesh shaders)"]
    Ch96["Ch96: libcamera\n(ISP / PipelineHandler / IPA)"]
    Ch111["Ch111: Flatpak Graphics\n(sandbox / GL extension / portals)"]
    Ch114["Ch114: OpenCV GPU Vision\n(T-API / UMat / cv::dnn / GStreamer)"]
    Ch48_XX["Ch48 (Part XX): ROCm & ML\n(amdkfd / HIP / PyTorch)"]

    Ch24 --> Ch25
    Ch24 --> Ch26
    Ch24 --> Ch27
    Ch24 --> Ch39
    Ch24 --> Ch50
    Ch24 --> Ch76
    Ch24 --> Ch111
    Ch24 --> Ch114
    Ch25 --> Ch48_XX
    Ch25 --> Ch114
    Ch26 --> Ch38
    Ch26 --> Ch50
    Ch26 --> Ch96
    Ch26 --> Ch114
    Ch38 --> Ch96
    Ch38 --> Ch111
    Ch38 --> Ch114
    Ch39 --> Ch47
    Ch48_XX --> Ch114
    Ch96 --> Ch114
    Ch111 --> Ch114
```

## Prerequisites and What Comes Next

Readers should arrive here having read Parts I–III: the **DRM/KMS** substrate (Part I), Mesa driver architecture and **NIR**/**Gallium** internals (Part II), and the **Wayland** compositor stack including buffer import paths and **linux-dmabuf** protocol (Part III). Familiarity with Parts IV–VI (display pipeline, synchronisation, and HDR) is helpful for the presentation-timing and explicit-sync discussions in Chapters 24, 39, and 50 but is not strictly required. Parts VIII–X build directly on this part: gaming and compatibility layers (**DXVK**, **VKD3D-Proton**, **Steam Play**) assume the compute, video, and modern Vulkan extension knowledge established here, and the browser chapters (Part X) rely on the **EGL**, **Vulkan Video**, **PipeWire**, and font pipeline foundations laid in Chapters 24, 26, 38, and 47. Chapter 111's Flatpak coverage is directly relevant to Part VIII's discussion of how Steam and Proton games are packaged and sandboxed on the Steam Deck. Chapter 114's vision pipeline patterns are a prerequisite for any robotics or automotive chapters that appear in later parts.

---

## Part Roadmap Summary

*Synthesised from the Roadmap sections of this part's chapters.*

### Near-term (6–12 months)

- **Vulkan Roadmap 2026 baseline adoption across Mesa.** The Khronos Roadmap 2026 milestone (published alongside Vulkan 1.4.340) mandates `VK_KHR_fragment_shading_rate`, timeline semaphores, multi-draw indirect, shader draw parameters, host-image copies, and higher descriptor limits as a conformance baseline. RADV, ANV, and NVK are expected to meet this milestone during the Mesa 26.x cycle, which directly affects chapters covering Vulkan WSI (Ch24), modern extensions (Ch76), mesh shaders/VRS (Ch127), compute queues (Ch133), and descriptor binding (Ch157).

- **Vulkan Video codec family completion.** VP9 decode (`VK_KHR_video_decode_vp9`) has landed in RADV and ANV; AV1 encode (`VK_KHR_video_encode_av1`) is shipping in RADV and in progress for ANV. The `VK_KHR_video_encode_quantization_map` and `VK_KHR_video_encode_intra_refresh` extensions are rolling out across Mesa drivers. GStreamer `vkvideo` elements and the FFmpeg Vulkan hwaccel path are stabilising to cover the full H.264/H.265/AV1/VP9 matrix (Ch50, Ch165).

- **EGL/DMA-BUF stack modernisation.** Mesa 25.2 deprecated `EGL_WL_bind_wayland_display` and the pre-DMA-BUF `wl_drm` path; DRI2 code removal is in progress. The `wp_linux_drm_syncobj_v1` explicit-sync Wayland protocol is being completed across compositors (Mutter, KWin, wlroots). NVIDIA's `egl-wayland2` library, built entirely on DMA-BUF, is being adopted. These changes consolidate the DMA-BUF/syncobj interop path that underpins Ch24, Ch150, and Ch38.

- **Mesa 26.x driver feature advances.** Mesa 26.1 NIR/URB mesh-shader unification improves cross-driver sharing. `VK_EXT_descriptor_heap` (published in Vulkan 1.4.340) is entering early bring-up in RADV and ANV. `VK_EXT_present_timing` was merged for RADV, ANV, NVK, PanVK, and Turnip WSI backends in Mesa 26.1, enabling precise present scheduling. NVK's `VK_EXT_shader_object` support arrived in early 2024; ANV landed in Mesa 25.3.

- **Application middleware updates.** PipeWire 1.6.x continues stabilisation with an atomic transaction API, ACP-to-UCM migration, and `filter-graph` ONNX/FFmpeg backends. Qt 6.12 (September 2026) adds indirect rendering in `QRhi` and `VK_EXT_device_fault` exposure. GTK 4.21+ is refactoring `GskRenderer` for SVG filter and COLRv1 support. HarfBuzz's `hb_gpu_paint_t` API is graduating from experimental status. libcamera 0.7.x is adding multi-threaded software ISP debayering and GPU-ISP (GLES 2.0) offload. OpenCV 5.x is ramping up GPU inference in its new graph DNN engine.

- **Rust GPU ecosystem stabilisation.** `wgpu` is targeting a stable 1.0 API; `naga` is consolidating into the `gfx-rs/wgpu` monorepo. `rust-gpu` is under community stewardship after Embark Studios handoff, focusing on SPIR-V backend stability. ROCm 8.0 (TheRock build system, mid-2026) adds gfx1151 support. `io_uring` `DRM_IOCTL_*` uring_cmd landing is tracked for Linux 6.13–6.15.

### Medium-term (1–3 years)

- **Vulkan Video replacing VA-API as the primary hardware video API.** GStreamer `vkvideo` elements (`vkh264dec`, `vkav1enc`, etc.) are expected to supersede `vaapi*` elements once encode completeness and sustained performance parity are achieved. Chromium is working toward replacing its VA-API GPU process hwaccel with native Vulkan Video. The Zink VA-API-on-Vulkan-Video bridge aims to serve legacy VA-API applications from Vulkan Video drivers. NVK Vulkan Video decode support is expected within one to two Mesa release cycles (Ch26, Ch50, Ch165).

- **Zero-copy pipeline maturation from sensor to display.** The architectural ambition across libcamera, PipeWire, V4L2, and Vulkan Video is a single DMA-BUF-backed buffer traversing ISP, codec, compositor, and KMS without CPU copies. Near-gaps include PipeWire–Vulkan video format-conversion nodes (Vulkan SPA plugin hardening), PipeWire camera portal v2 with richer control negotiation, and deeper integration of libcamera Layers plugin for burst HDR and ZSL. PipeWire's `filter-graph` module (FFmpeg/ONNX backends, 1.6) is the foundation for in-graph ML video filtering (Ch26, Ch38, Ch96).

- **Cooperative matrices and ML inference on the open-source Vulkan stack.** `VK_KHR_cooperative_matrix` is maturing on RDNA4 (RADV) and Xe-HPG (ANV); `VK_NV_cooperative_matrix2` landed in RADV 25.2 targeting FSR4 and VKD3D-Proton. Sub-byte precision (`FP8`, `INT4`, `cl_khr_cooperative_matrix`) and SPIR-V type extensions are in Khronos discussion, driven by LLM inference workloads. AMD UDNA (CDNA/RDNA convergence, ~2027) and Intel Xe2 DPAS improvements will shape the ISA targets for Ch25, Ch133, Ch141.

- **GPU-driven rendering and descriptor binding modernisation.** `VK_EXT_device_generated_commands` (DGC) is shipping in production drivers; GPU work graphs (`VK_AMDX_shader_enqueue` with mesh nodes) are being tracked for cross-vendor KHR promotion. `VK_EXT_descriptor_heap` — replacing the descriptor-set subsystem with a flat GPU-visible heap — is seeking KHR ratification and potential mandating in a future Roadmap milestone. VKD3D-Proton is an early adopter. Indirect Execution Sets and subgroup reconvergence guarantees (Roadmap 2026) improve GPU-driven culling in Bevy, Godot 4, and open-source engines (Ch127, Ch133, Ch154, Ch157).

- **Toolkit and font rendering evolution.** GTK5 API planning is underway (Vulkan 1.2+ baseline, possible GSK redesign). Both Qt's `QRhi` and GTK's GSK renderer are wiring up `wp_color_management_v1` for HDR wide-gamut surfaces. HarfBuzz GPU paint integration and variable COLRv1 full-stack support require coordinated FreeType, HarfBuzz, and toolkit atlas updates. Fractional-scale-aware SDF glyph caching is needed as `wp_fractional_scale_v1` becomes the norm (Ch39, Ch47).

- **Sandboxing and GPU virtualisation hardening.** The Venus/virglrenderer virtio-GPU approach is maturing toward driver-agnostic sandbox graphics without loading host userspace drivers. A per-capability GPU access daemon is planned to replace coarse `--device=dri` grants. `systemd-appd` and Varlink portal IPC are expected to improve Flatpak nested sandboxing. XDG portal v2 camera API and richer GPU manifest declarations will follow (Ch111). OpenCV 5.x T-API gains a native libcamera backend, Vulkan compute DNN backend operator expansion, and PipeWire DMA-BUF camera source integration (Ch114).

- **OpenXR/VR spatial computing and ML tracking.** Monado is gaining OpenXR Spatial Entities (`XR_EXT_plane_detection`, `XR_EXT_spatial_anchor`), persistent SLAM mapping via Basalt VIO, and Android/standalone-device deployment. GPU-accelerated ML inference (ONNX/TFLite via Vulkan compute) for hand and eye tracking is on the medium-term roadmap. DRM leasing improvements are needed for USB4/Thunderbolt headsets (Ch27).

### Long-term

- **Unified explicit synchronisation spanning all Linux GPU subsystems.** The `drm_syncobj` timeline model underpins Vulkan's timeline semaphores, `wp_linux_drm_syncobj_v1`, and `EGL_ANDROID_native_fence_sync`, but camera (V4L2), hardware video codecs (VA-API), and media subsystems still carry independent implicit-fence models. The architectural goal is a single kernel timeline primitive spanning KMS, V4L2, Vulkan, and media — with multi-process timeline FD export (`drm_syncobj` timeline semantics across process boundaries) and a unified `drm_sched` arbiter scheduling Vulkan queues, CUDA streams, and V4L2/VA-API sessions with unified priority and preemption semantics (Ch24, Ch133, Ch148).

- **Vulkan as the universal GPU substrate — compute, video, and inference convergence.** The long-term trajectories of Ch25 (compute), Ch50/Ch165 (video), Ch76/Ch157 (modern extensions), and Ch141 (cooperative matrices) converge on Vulkan compute as the cross-vendor fallback for ML inference and video processing, with `VK_KHR_cooperative_matrix` providing matrix-MMA acceleration and Vulkan Video removing VA-API as an intermediary. CXL 3.x-attached HBM exposed as DMA-BUF heaps and NPU/GPU convergence under the DRM/accel subsystem are the furthest-horizon items.

- **EGL and pipeline-model deprecation.** EGL's role is narrowing to OpenGL ES, VA-API interop, and legacy X11 surfaces as Vulkan WSI and Zink (OpenGL-on-Vulkan) mature; new applications are expected to use Vulkan `VK_KHR_video_queue` rather than EGL + VA-API. Similarly, `VK_EXT_shader_object` — already default in RADV and shipped in ANV/NVK — may become the primary shader-binding model in a future Vulkan Roadmap milestone, with `VkPipeline` receding to a compatibility layer (Ch150, Ch173). The Rust GPU ecosystem's long-term goal of safe, zero-overhead Vulkan abstractions (bridging `ash` and `wgpu`) and rust-gpu as a first-class shader authoring path aligns with this direction (Ch152).

- **XR, foveated rendering, and compositor convergence.** OpenXR 2.0 or a major revision is accumulating spatial-mesh types, standardised ML tracking hooks, and a formal passthrough API. VRS driven by calibrated eye-tracking data (`VK_KHR_fragment_shading_rate` fed by on-chip neural inference) and OpenXR standardisation of per-layer rate control are expected for consumer headsets. The long-term proposal of a unified compositor handling both desktop Wayland windows and XR layers in a single process would eliminate the DRM-lease handoff entirely (Ch27, Ch127).
