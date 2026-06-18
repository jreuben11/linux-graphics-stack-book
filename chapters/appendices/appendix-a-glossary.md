# Appendix A: Glossary

*A single alphabetical reference for the vocabulary used across all 19 parts of "The Linux Graphics Stack: From Kernel to Compositor, Browser, and Terminal." Each entry includes a chapter cross-reference and related terms. Terms that have distinct meanings at different layers of the stack (e.g., "fence") are disambiguated within a single entry. Abbreviations expand in parentheses after the term name and sort by their abbreviation.*

---

## A

### ABR (Adaptive Bitrate Streaming)
A delivery technique in which the server pre-encodes a video at multiple bitrate/resolution levels (renditions) and the client player selects which rendition to download for each segment based on observed network throughput and buffer state. The two dominant ABR protocols on Linux are HLS and MPEG-DASH; both rely on manifest files (M3U8 or MPD) to enumerate renditions. ABR algorithms range from simple throughput-based heuristics to reinforcement-learning approaches such as Pensieve and BOLA. A common misconception is that ABR requires a specialised CDN; standard HTTP object storage with byte-range support is sufficient.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **HLS (HTTP Live Streaming)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **M3U8 playlist**, **MPD (MPEG-DASH Media Presentation Description)**, **BOLA (Buffer Occupancy–based Lyapunov Algorithm)**, **Pensieve (RL-based ABR)**, **video segment**

---

### ACE (Adreno Compute Engine)
The Adreno Compute Engine is a dedicated asynchronous compute hardware block present on Qualcomm Adreno GPUs (A5xx and later) that executes compute shader workloads concurrently with the 3D graphics pipeline. Unlike the main SP (Shader Processor) array that handles both vertex/fragment shading and compute, the ACE enables the GPU to overlap rasterisation with GPGPU tasks such as post-processing, AI inference, and physics. On Linux, the `msm` DRM kernel driver exposes the ACE through its ring-buffer command submission model, and the Mesa Turnip Vulkan driver (`src/freedreno/vulkan/`) targets separate Vulkan compute queues backed by ACE hardware.

**Chapter(s):** Ch6 — ARM & Embedded GPU Drivers; Ch18 — Vulkan Drivers  
**Related:** **compute shader**, **compute unit (CU)**, **freedreno**, **wavefront (AMD thread group)**

---

### ACO (AMD Compiler for Optimizations)
ACO is AMD's in-tree Mesa shader compiler, living at `src/amd/compiler/`, that translates Mesa NIR into GCN/RDNA machine code entirely in C++ without depending on LLVM. Designed explicitly for lower compile latency and better register allocation than the LLVM path, ACO became the default backend for RADV in Mesa 20.2 and is used by radeonsi for compute. Its key passes include instruction selection (`isel`), register allocation (`ra`), and the spill/reload framework. A common misconception is that ACO replaces LLVM everywhere in Mesa — it does not; radeonsi's graphics path still uses LLVM, while only RADV and select compute paths use ACO by default.

**Chapter(s):** Ch15 — ACO: AMD's Optimising Compiler; Ch18 — Vulkan Drivers  
**Related:** **NIR**, **RADV**, **GCN ISA**, **RDNA ISA**, **register allocation (GPU)**, **LLVM (in Mesa context)**

---

### ACS (PCIe Access Control Services)
A PCIe capability that controls whether peer-to-peer transactions between PCIe endpoints are permitted or must be redirected through the root complex. When ACS is absent or misconfigured, multiple GPUs in a system may share the same IOMMU group, preventing independent passthrough to virtual machines. On consumer platforms with weak ACS support, all PCIe slots often end up in a single IOMMU group; the widely distributed `pcie_acs_override=` kernel parameter forces group splitting at the cost of reduced DMA isolation. Relevant kernel infrastructure lives in `drivers/pci/acs.c` and `drivers/iommu/`.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute  
**Related:** **IOMMU**, **PRIME**, **DMA-BUF**

---

### AGX (Apple Silicon GPU)
AGX refers to the GPU microarchitecture found in Apple Silicon SoCs (M1, M2, M3, and derivatives), as well as the open-source Mesa driver also named `agx` (sometimes called Asahi GPU driver), residing at `src/gallium/drivers/asahi/` and `src/asahi/`. The hardware uses a tile-based deferred rendering (TBDR) model with a proprietary ISA that was reverse-engineered by the Asahi Linux project. The Mesa AGX driver provides OpenGL ES and, increasingly, full Vulkan support (`hk` Vulkan driver) for Linux on Apple Silicon. Do not confuse the Mesa driver with Apple's own proprietary Metal/AGX driver stack; the two share no code.

**Chapter(s):** Ch6 — ARM & Embedded GPU Drivers; Ch18 — Vulkan Drivers  
**Related:** **Gallium3D**, **NIR**, **ISA (Instruction Set Architecture)**, **Turnip (Qualcomm Adreno Vulkan)**

---

### AHardwareBuffer
Android's public C API (introduced in Android 8.0 / API 26) for allocating GPU-accessible, cross-process shared memory buffers. `AHardwareBuffer_allocate()` accepts a descriptor specifying width, height, format, and usage flags (e.g., `AHARDWAREBUFFER_USAGE_GPU_FRAMEBUFFER | AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE`). Internally it wraps a Gralloc handle and can be exported as a POSIX file descriptor via `AHardwareBuffer_sendHandleToUnixSocket()` or imported into Vulkan via `VkImportAndroidHardwareBufferInfoANDROID` (the `VK_ANDROID_external_memory_android_hardware_buffer` extension). Do not confuse with the private `GraphicBuffer` C++ class used inside the Android framework itself — `AHardwareBuffer` is the stable NDK surface.

**Chapter(s):** Ch82 — Android HardwareBuffer and Cross-Process GPU Memory  
**Related:** **Gralloc**, **ANativeWindow**, **BufferQueue (Android)**, **IGraphicBufferProducer**

---

### AMF (AMD Advanced Media Framework)
The AMD Advanced Media Framework is a low-level C++ GPU-accelerated multimedia SDK that exposes AMD's hardware video encode (VCN, UVD/VCE) and decode blocks through a pipeline-based API. On Linux, AMF sits above the `amdgpu` kernel driver and communicates with encode/decode firmware via the `amdgpu_vce`/`amdgpu_uvd` or VCN ring buffers; the public header is `amf/core/Factory.h`. AMF is used by OBS Studio, FFmpeg (`-hwaccel amf`), and Handbrake as an alternative to VA-API for direct AMD hardware encode with lower overhead. Unlike VA-API, AMF is AMD-proprietary and requires the `amdgpu-pro` or ROCm driver stack; it is not part of Mesa.

**Chapter(s):** Ch72 — AMD FidelityFX & Tools; Ch26 — Hardware Video  
**Related:** **FidelityFX SDK**, **compute unit (CU)**, **lookahead (encoder)**

---

### ANativeWindow
The C interface (`<android/native_window.h>`) through which NDK code and HAL drivers acquire, queue, and dequeue graphics buffers destined for display in Android. `ANativeWindow` is a reference-counted opaque handle backed by a `Surface` object (a `BufferQueue` consumer+producer pair in `libgui`). Callers use `ANativeWindow_lock()` / `ANativeWindow_unlockAndPost()` for CPU rendering or `eglCreateWindowSurface(egl_display, config, anw, NULL)` to create an EGL rendering surface. The Vulkan equivalent is `vkCreateAndroidSurfaceKHR()` via `VK_KHR_android_surface`. Many graphics APIs on Android ultimately funnel frames through `ANativeWindow` into `SurfaceFlinger`.

**Chapter(s):** Ch83 — Vulkan on Android; Ch80 — SurfaceFlinger and the Android Display Pipeline  
**Related:** **BufferQueue (Android)**, **SurfaceFlinger**, **IGraphicBufferProducer**, **AHardwareBuffer**

---

### android::Fence
The Android C++ class (`frameworks/native/libs/ui/Fence.cpp`) wrapping a Linux sync file descriptor (`/sys/kernel/debug/sync`) to represent a GPU fence in the Android graphics stack. An `android::Fence` holds a single `int fd` pointing to a `sync_file` object created by DRM drivers via `SYNC_IOC_MERGE`; waiting on it blocks until all GPU work signalling that fence completes. In Android's buffer pipeline, `BufferQueue` attaches an `android::Fence` to each buffer slot: the producer sets a "release fence" when it finishes writing, and the consumer sets an "acquire fence". This is Android's analogue to the `dma_fence` / sync file mechanism used by Linux DRM (`DMA_BUF_IOCTL_EXPORT_SYNC_FILE`) and Wayland's `wp_linux_drm_syncobj_v1`. Do not confuse with Vulkan `VkFence`, which is a higher-level API-only primitive with no kernel backing.

**Chapter(s):** Ch75 — Explicit GPU Synchronisation; Ch83 — Vulkan on Android  
**Related:** **BufferQueue (Android)**, **ANativeWindow**, **AHardwareBuffer**, **IGraphicBufferProducer**

---

### ANV (Anv Vulkan Driver)
ANV (sometimes expanded as "Anvil") is Intel's Mesa Vulkan driver, located at `src/intel/vulkan/`, supporting Intel Gen 8 (Broadwell) through Xe2 (Battlemage) hardware. ANV implements the Vulkan WSI, pipeline compilation via the Intel EU ISA backend and BRW/ELK compiler, and a bindless heap design that maps descriptor sets to a GPU-visible buffer. On Gen 12+ it emits Gfx12 shader binary via the `elk_compiler`; on Xe2 it uses the new `brw_compiler` targeting Xe2 ISA. ANV ray tracing uses hardware-accelerated BVH traversal introduced on DG2/Arc. It is distinct from Intel's legacy GL driver `iris`, which shares the same compiler infrastructure but targets the OpenGL API.

**Chapter(s):** Ch18 — Vulkan Drivers; Ch56 — Ray Tracing on Linux; Ch50 — Vulkan Video Extensions  
**Related:** **BRW compiler (Intel EU ISA)**, **EU ISA (Intel Execution Unit ISA)**, **NIR**, **disk_cache (Mesa)**, **Fossilize**

---

### APC (Application Program Command, terminal)
An ANSI/ISO 6429 C1 control function introduced as a private-use container for terminal extensions. The APC string is framed by the two-character sequence `ESC _` (0x9F in 8-bit mode, or `\033_` in 7-bit) and terminated by `ST` (String Terminator, `ESC \`). Unlike DCS or OSC, APC has no standardised semantics defined by any major terminal standard, making it freely available for vendor extensions. The iTerm2 image protocol uses APC (`ESC _ Ga=…; ST`) as its framing mechanism, distinct from the Kitty Graphics Protocol which also uses APC framing. Terminal emulators that do not recognise a particular APC payload must silently ignore it.

**Chapter(s):** Ch43 — Terminal Pixel Protocols — Sixel, Kitty, and iTerm2  
**Related:** **DCS (Device Control String)**, **Kitty Graphics Protocol**, **Sixel**

---

### ASurfaceControl
Android NDK API (introduced in Android 10 / API 29) providing a subset of SurfaceFlinger's transaction model to native code without requiring Java. `ASurfaceControl_create()` creates a layer within the SurfaceFlinger hierarchy; `ASurfaceTransaction` batches atomic updates to position, z-order, crop, blend mode, and buffer content. Each `ASurfaceTransaction_setBuffer()` call attaches an `AHardwareBuffer` and an associated `android::Fence`. This API is the NDK equivalent of the Java `SurfaceControl` + `SurfaceControl.Transaction` pair used by `WindowManager`, exposing the same atomic commit semantics as KMS atomic ioctls on Linux. Common misconception: `ASurfaceControl` does not bypass SurfaceFlinger; it submits transactions to it.

**Chapter(s):** Ch80 — SurfaceFlinger and the Android Display Pipeline  
**Related:** **SurfaceFlinger**, **AHardwareBuffer**, **ANativeWindow**, **HWComposer (Android)**

---

### atomic modesetting
The KMS display-update model introduced in kernel 3.19 (drivers/gpu/drm/drm_atomic.c) that replaces the legacy `drmModeSetCrtc` / `drmModeSetPlane` approach. All display-pipeline changes — connector routing, CRTC modes, plane positions, properties — are batched into a single `struct drm_atomic_state` and submitted via `DRM_IOCTL_MODE_ATOMIC`. Drivers first validate the state with a `TEST_ONLY` flag; if validation passes, the commit is applied atomically on the next vblank. The key advantage over the legacy path is that partial-update failures cannot leave the hardware in an inconsistent state, and non-blocking commits allow the compositor to pipeline frame preparation. A common misconception is that "atomic" implies GPU-side atomicity; it strictly refers to the KMS object set being applied as a unit.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **KMS**, **CRTC**, **plane (DRM)**, **vblank**, **page flip**, **connector (DRM)**

---

### AV1
An open, royalty-free video codec specification published by the Alliance for Open Media (AOMedia) in 2018. AV1 is an AOMedia specification, not an ISO/IEC standard; ISO/IEC 23090-3 is VVC/H.266, not AV1. AV1 uses superblock coding units up to 128×128 pixels, a non-binary arithmetic entropy coder (derived from Daala's `daala_ec`), and film grain synthesis metadata. On Linux, reference software implementations are libaom (encoder/decoder), dav1d (decoder, optimised), SVT-AV1 (scalable encoder), and rav1e (Rust encoder). Hardware decode support is exposed via `VK_KHR_video_decode_av1` (Vulkan Video) and the `VAProfileAV1Profile0` VA-API profile. AV1 is not to be confused with its container: AV1 video is typically carried in ISOBMFF or WebM.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch50 — Vulkan Video Extensions; Ch26 — Hardware Video  
**Related:** **codec**, **H.264 / AVC**, **H.265 / HEVC**, **VP9**, **DCT (Discrete Cosine Transform)**, **entropy coding (CABAC/CAVLC)**, **NAL unit (H.264/H.265)**, **intra-prediction**, **inter-prediction**

---

### AVCodec / AVCodecContext (FFmpeg)
`AVCodec` is FFmpeg's read-only descriptor struct for a registered codec implementation; it carries the codec name, `AVCodecID`, supported pixel formats, and function pointers (`init`, `send_packet`, `receive_frame`, `close`). `AVCodecContext` is the per-instance mutable state allocated by `avcodec_alloc_context3()` and configured before `avcodec_open2()` is called; it holds bitrate, resolution, timebase, thread count, and a pointer back to the `AVCodec`. The two are distinct: `AVCodec` is shared among all instances of the same codec, while `AVCodecContext` owns the decode or encode state for one stream. Hardware acceleration attaches to `AVCodecContext` via `AVCodecContext.hw_device_ctx` (`AVHWDeviceContext`).

**Chapter(s):** Ch57 — FFmpeg Architecture and Programming  
**Related:** **FFmpeg**, **NVENC**, **NVDEC**, **V4L2 (Video4Linux2)**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**

---

## B

### B-frame
A bidirectionally predicted video frame that is compressed by referencing both a past reference frame and a future reference frame in decode order. Because B-frames exploit both forward and backward redundancy, they achieve higher compression ratios than P-frames at the same quality. In H.264/AVC the maximum B-frame distance is controlled by `num_ref_frames` and the encoder's `bf` parameter (x264 option `-b`); in HEVC, B-frames within a CTU hierarchy replace H.264's slice-level structure. A common operational point: B-frames increase encode latency by at least one reference-frame interval and are typically disabled for ultra-low-latency live streaming. B-frame slice type (`slice_type = B`) is defined in the H.264 bitstream syntax; the number of consecutive B-frames between reference frames is an encoder decision controlled at encode time.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch57 — FFmpeg Architecture and Programming  
**Related:** **I-frame**, **P-frame**, **IDR frame**, **inter-prediction**, **motion estimation**, **motion vector**, **H.264 / AVC**, **H.265 / HEVC**

---

### bgfx
bgfx is a cross-platform, rendering-API-agnostic C++ rendering library that abstracts Vulkan, OpenGL ES, Metal, DirectX 11/12, and WebGPU behind a single API through a command-based render-graph model. On Linux, bgfx selects either the Vulkan or OpenGL backend at runtime based on driver availability; its render-graph abstraction (`bgfx::Encoder`, `bgfx::ViewId`, transient buffers) is the primary concept connecting it to Ch84's render-graph patterns. bgfx is widely used in games and tools that must run on constrained or embedded hardware where a single-vendor Vulkan assumption is unsafe. Its GLSL-based BGFX shader language cross-compiles to SPIRV, GLSL, HLSL, and Metal via the bundled `shaderc` tool.

**Chapter(s):** Ch84 — bgfx and Render Graphs  
**Related:** **render graph**, **compute shader**, **Filament (rendering engine)**

---

### Bindless Resources
A descriptor model in which shaders index into large, unbounded arrays of descriptors — textures, buffers, or samplers — rather than binding a fixed set at draw time. In Vulkan this is enabled by `VK_EXT_descriptor_indexing` (promoted to Vulkan 1.2 as `VkPhysicalDeviceDescriptorIndexingFeatures`), which allows `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT` and `VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT`. The key benefit is eliminating per-draw descriptor set changes and enabling GPU-driven rendering where the shader selects resources via a material index. Bindless is distinct from push descriptors, which still bind a fixed set per draw.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch18 — Vulkan Drivers  
**Related:** **descriptor set**, **descriptor set layout**, **VkDescriptorPool**, **pipeline layout**

---

### BOLA (Buffer Occupancy–based Lyapunov Algorithm)
An ABR segment-selection algorithm that frames the bitrate adaptation problem as a utility-maximisation problem solved using Lyapunov optimisation theory. BOLA selects the rendition that maximises a weighted sum of video quality and buffer stability without needing explicit bandwidth estimation; it reacts to buffer occupancy level alone. Introduced in the 2016 IEEE INFOCOM paper by Spiteri et al. and later integrated into dash.js as the default ABR algorithm. BOLA-E and BOLA-FINITE are refinements that handle finite-duration content and bandwidth-estimation hysteresis. Unlike throughput-based algorithms, BOLA can remain stable under highly variable network conditions without oscillating between renditions.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **ABR (Adaptive Bitrate Streaming)**, **Pensieve (RL-based ABR)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **video segment**, **MPD (MPEG-DASH Media Presentation Description)**

---

### BPP (Bits Per Pixel)
The number of bits used to represent a single pixel in a framebuffer or wire encoding. On the display pipeline, BPP governs scanout bandwidth and is expressed through DRM format fourcc codes (`DRM_FORMAT_XRGB8888` = 32 bpp, `DRM_FORMAT_RGB565` = 16 bpp) defined in `include/uapi/drm/drm_fourcc.h`. In DSC compressed streams BPP refers to the *target* compressed output BPP, which may be fractional (e.g. 8.0 or 10.666). Do not confuse BPP with bit depth per channel: a 32-bpp XRGB8888 format has only 8 bits per colour channel, whereas a 30-bpp XRGB2101010 format has 10 bits per channel.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **framebuffer (DRM)**, **DSC (Display Stream Compression)**, **wide color gamut (WCG)**

---

### BRW compiler (Intel EU ISA)
The BRW (Broadwell) compiler is Intel's Mesa shader compiler for GEN/Xe ISA, residing at `src/intel/compiler/`. It consumes Mesa NIR and emits binary for Intel Execution Units, performing instruction scheduling, register allocation, and hardware workaround insertion. In Mesa 24+ the codebase was split into `brw_compiler` (targeting Xe2+) and `elk_compiler` (targeting Gen 4–12.5 via the legacy "Elk" path) to manage the diverging ISA. The compiler is shared between ANV (Vulkan) and `iris` (OpenGL), making it the single source of truth for Intel shader codegen in Mesa. Do not confuse BRW (the Mesa compiler) with Intel's downstream IGC (Intel Graphics Compiler) used in oneAPI/compute-runtime.

**Chapter(s):** Ch18 — Vulkan Drivers; Ch19 — OpenGL and Compatibility Drivers; Ch25 — GPU Compute  
**Related:** **ANV**, **EU ISA (Intel Execution Unit ISA)**, **NIR**, **LLVM (in Mesa context)**

---

### BT.2020
ITU-R Recommendation BT.2020, the colour primaries and signal encoding standard for Ultra High Definition Television (UHDTV) and the container space for HDR10, HLG, and Dolby Vision content. Its wide-gamut primaries cover approximately 75 % of visible colour, significantly exceeding BT.709 and sRGB. The transfer function for SDR-within-BT.2020 is similar to BT.709's, but HDR mastered in BT.2020 typically uses the PQ (ST 2084) or HLG transfer function. In KMS, BT.2020 is expressed via the `DRM_COLOR_YCBCR_BT2020` coefficient enum and the `hdr_output_metadata` property on connectors (`include/uapi/drm/drm_mode.h`). Wide-gamut display panels advertise BT.2020 coverage in EDID extension blocks.

**Chapter(s):** Ch3 — Advanced Display Features; Ch26 — Hardware Video; Ch53 — Display Calibration and colord  
**Related:** **BT.709**, **HDR10**, **HLG (Hybrid Log-Gamma)**, **PQ (Perceptual Quantizer / ST 2084)**, **wide color gamut (WCG)**, **EDID**

---

### BT.709
ITU-R Recommendation BT.709, the colour primaries and transfer function standardised for HDTV and the dominant colour space of SDR video and most sRGB-targeted content. Its primaries define a triangular gamut in CIE xy covering approximately 35 % of visible colour. The transfer function is a gamma-approximately-2.2 curve (with a linear toe segment). In the kernel, BT.709 matrix coefficients appear as `DRM_COLOR_YCBCR_BT709` in `include/uapi/drm/drm_mode.h` and are referenced by the `wp_color_representation_v1` Wayland protocol for YCbCr video surfaces handed to the compositor. BT.709 primaries are a subset of BT.2020, so content produced in BT.2020 must be tone-mapped or gamut-mapped when displayed on BT.709 panels.

**Chapter(s):** Ch3 — Advanced Display Features; Ch26 — Hardware Video; Ch53 — Display Calibration and colord  
**Related:** **BT.2020**, **sRGB**, **HDR10**, **wide color gamut (WCG)**, **CTM (Color Transform Matrix)**

---

### BufferQueue (Android)
The core producer–consumer pipeline in the Android graphics stack, implemented in `frameworks/native/libs/gui/BufferQueue.cpp`. A `BufferQueue` manages a ring of `GraphicBuffer` slots (typically 2–64 slots) between one producer (`IGraphicBufferProducer`) and one consumer (`IGraphicBufferConsumer`). Producers call `dequeueBuffer()` to obtain a free slot, render into it, then call `queueBuffer()` with a fence; consumers call `acquireBuffer()` and, after compositing, `releaseBuffer()`. `SurfaceFlinger` is the most common consumer; app windows via `Surface` are the most common producers. The slot count, format, and dimensions are negotiated at connect time via `setDefaultBufferSize()` and `setDefaultBufferFormat()`.

**Chapter(s):** Ch80 — SurfaceFlinger and the Android Display Pipeline; Ch82 — Android HardwareBuffer and Cross-Process GPU Memory  
**Related:** **IGraphicBufferProducer**, **SurfaceFlinger**, **ANativeWindow**, **android::Fence**

---

### BVH (Bounding Volume Hierarchy)
A Bounding Volume Hierarchy is a tree-based spatial acceleration structure where each internal node bounds the geometry of all of its children, enabling ray-scene intersection tests to skip large portions of the scene through hierarchical culling. In GPU hardware ray tracing, the BVH is split into a Bottom-Level AS (BLAS, per-mesh) and a Top-Level AS (TLAS, per-scene of instances); the `VkAccelerationStructureKHR` object in Vulkan wraps both levels, and NVIDIA's RT Cores (Ampere/Ada) and AMD's RDNA2+ intersection hardware accelerate BLAS traversal in fixed-function units. In OptiX, BVHs are built via `optixAccelComputeMemoryUsage` + `optixAccelBuild`, with compaction via `OPTIX_BUILD_FLAG_ALLOW_COMPACTION` + `optixAccelCompact` reducing memory by 50–70%.

**Chapter(s):** Ch56 — Ray Tracing on Linux; Ch67 — OptiX 9; Ch70 — RTX Kit  
**Related:** **ray tracing hardware (RT Core / XMX)**, **ReSTIR / RTXDI**, **RTXGI**, **OptiX**

---

## C

### CAS (Contrast Adaptive Sharpening)
Contrast Adaptive Sharpening is a spatial image-sharpening filter developed by AMD that applies a variable-strength edge enhancement based on local contrast — high contrast edges receive less sharpening to avoid ringing, while low-contrast regions receive more. CAS is available as an open-source GLSL/HLSL compute shader in the FidelityFX SDK (`ffx_cas.h`) and is GPU-vendor-agnostic, running on any hardware supporting compute shaders. Unlike FSR (which adds upscaling), CAS operates purely in the spatial domain on a native-resolution image. On Linux, CAS is integrated into Mesa's `vkBasalt` post-processing layer and can be toggled via `ENABLE_VK_LAYER_vkBasalt=1`.

**Chapter(s):** Ch72 — AMD FidelityFX & Tools; Ch29 — Upscaling, Effects & Overlays  
**Related:** **FidelityFX SDK**, **FSR 4**, **compute shader**

---

### CMAF (Common Media Application Format)
An MPEG/ISO standard (ISO/IEC 23000-19) that defines a common packaging layer for both HLS and MPEG-DASH by restricting fragmented MP4 (fMP4) to a single compatible track format. A CMAF track is a fragmented ISOBMFF file where each `moof`+`mdat` pair constitutes a chunk; multiple chunks aggregate into a segment. By sharing a single set of CMAF tracks, content providers can eliminate the cost of encoding separate TS-wrapped HLS segments and DASH MP4 segments. CMAF is the enabling technology for low-latency DASH and LL-HLS, both of which deliver chunks rather than complete segments. CMAF does not dictate the manifest format: HLS uses M3U8 playlists and DASH uses MPD files to reference the same underlying CMAF tracks.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **HLS (HTTP Live Streaming)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **LL-HLS (Low-Latency HLS)**, **M3U8 playlist**, **MPD (MPEG-DASH Media Presentation Description)**, **video segment**

---

### codec
Short for *coder-decoder*; in the video domain the term is overloaded across three distinct layers. At the **specification** layer, a codec is a standard defining a compressed bitstream format (e.g. ISO/IEC 14496-10 for H.264). At the **software implementation** layer, a codec is a library that encodes or decodes that bitstream (e.g. x264, libvpx, dav1d, libaom). At the **hardware** layer, a codec is a fixed-function silicon block that implements the specification (e.g. AMD VCN, Intel Quick Sync, NVIDIA NVENC/NVDEC). These three layers are independent: the spec governs bitstream interoperability; different implementations of the same spec are not required to produce bit-identical output. On Linux, the kernel exposes hardware codec blocks via V4L2 or VA-API; FFmpeg and GStreamer wrap both hardware and software implementations behind a uniform `AVCodec`/`GstElement` API.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch57 — FFmpeg Architecture and Programming; Ch58 — GStreamer: Pipeline-Based Multimedia  
**Related:** **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **VP9**, **AVCodec / AVCodecContext (FFmpeg)**, **V4L2 (Video4Linux2)**, **NVENC**, **NVDEC**

---

### color LUT
A Look-Up Table that maps input colour values to output colour values during the KMS colour management pipeline. The kernel exposes per-CRTC LUTs through the `DEGAMMA_LUT`, `GAMMA_LUT`, and (in newer drivers) intermediate `CTM` atomic properties; the table entries are `struct drm_color_lut` (defined in `include/uapi/drm/drm_mode.h`) with 16-bit fixed-point red, green, and blue components. The degamma LUT linearises nonlinear input (e.g. sRGB encoded content), the CTM matrix then transforms primaries, and the gamma LUT re-encodes for the display. Driver-specific LUT sizes are reported via the `DEGAMMA_LUT_SIZE` and `GAMMA_LUT_SIZE` properties. LUTs can implement arbitrary tone mapping for HDR-to-SDR or SDR-to-HDR conversions.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features; Ch53 — Display Calibration and colord  
**Related:** **degamma**, **CTM (Color Transform Matrix)**, **KMS**, **HDR10**, **sRGB**

---

### color-management-v1 (Wayland protocol)
The `wp_color_management_v1` Wayland protocol extension (freedesktop.org `wayland-protocols`, staging) that lets clients describe the colour space and HDR metadata of their surfaces and lets the compositor apply correct colour transforms before scanout. Clients bind `wp_color_management_surface_v1` to a `wl_surface` and set an image description created from an ICC profile (`wp_image_description_creator_icc_v1`) or a parametric primaries + EOTF triple. The compositor converts the surface from the declared colour space into the display's output colour volume, programming the KMS `DEGAMMA_LUT`, `CTM`, and `GAMMA_LUT` blob properties accordingly. Distinct from the legacy `wl_output` colour-hint approach; this protocol is the long-term replacement adopted by Mutter (GNOME 48) and KWin (Plasma 6.1).

**Chapter(s):** Ch46 — Wayland Protocol Ecosystem; Ch74 — HDR and Wide Colour Gamut  
**Related:** **wl_surface**, **wl_compositor**, **MuraCorrection**, **zwp_linux_dmabuf_v1**

---

### compute shader
A compute shader is a GPU program stage that executes general-purpose parallel computation outside the fixed-function graphics pipeline, without rasterisation, vertex processing, or fragment output. In Vulkan, compute shaders are dispatched via `vkCmdDispatch` or `vkCmdDispatchIndirect` on a compute-capable `VkQueue`; the SPIR-V entry point uses `OpEntryPoint ExecutionModel GLCompute`. On AMD hardware, compute shaders are scheduled to CUs (Compute Units) in wavefronts of 32 or 64 lanes; on NVIDIA hardware, in warps of 32 threads. In Mesa, the SPIR-V compute shader enters NIR via `spirv_to_nir` and is compiled by ACO or LLVM before submission to the kernel driver.

**Chapter(s):** Ch25 — GPU Compute; Ch14 — NIR: Mesa's Shader IR; Ch15 — ACO  
**Related:** **compute unit (CU)**, **warp (NVIDIA thread group)**, **wavefront (AMD thread group)**, **CUDA stream**

---

### compute unit (CU)
A Compute Unit is AMD's term for the fundamental shader multiprocessor block in GCN and RDNA architectures, analogous to NVIDIA's Streaming Multiprocessor (SM). Each CU contains four SIMD32 (RDNA) or four SIMD16 (GCN) vector ALU arrays, a scalar ALU, a scheduler, L1 instruction and data caches, LDS (Local Data Share) shared memory, and a set of VGPRs and SGPRs. At RDNA3 (RX 7000 series), a Compute Unit contains two WGPs (Work Group Processors), each with two SIMD32 units. The `amdgpu` kernel driver reports the number of CUs per shader array via `amdgpu_gfx_gpu_info`; ROCm exposes them via `hipDeviceGetAttribute(HIP_DEVICE_ATTRIBUTE_MULTIPROCESSOR_COUNT, ...)`.

**Chapter(s):** Ch5 — x86 GPU Drivers; Ch25 — GPU Compute; Ch48 — ROCm and ML  
**Related:** **wavefront (AMD thread group)**, **compute shader**, **warp (NVIDIA thread group)**

---

### confidential computing (GPU)
Confidential computing in the GPU context refers to hardware and firmware mechanisms that protect GPU workloads from inspection or tampering by the host OS, hypervisor, or other tenants, extending CPU-based TEE (Trusted Execution Environment) protection into GPU memory. On AMD GPUs, this is achieved via SEV-SNP combined with AMD's Secure Encrypted Virtualisation extensions for the GPU (MI300 and newer); NVIDIA's H100 and later implement GPU Confidential Computing via the CC mode flag in `nvidia-smi`, locking GPU memory behind hardware encryption and exposing isolated GPU contexts to guest VMs. On Linux, VFIO (`/dev/vfio/`) and the IOMMU subsystem are prerequisite infrastructure for GPU confidential VM deployment.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute; Ch5 — x86 GPU Drivers  
**Related:** **SEV-SNP (AMD)**, **IOMMU group**, **vfio (Virtual Function I/O)**, **VRAM**

---

### connector (DRM)
A `struct drm_connector` (defined in `include/drm/drm_connector.h`) that represents a physical display output port — HDMI, DisplayPort, DVI, eDP, DSI, and so on. A connector reports its status (connected / disconnected / unknown) and exposes EDID data read from the attached display. In the atomic KMS object graph a connector links to an encoder which feeds a CRTC; one CRTC can drive several connectors simultaneously for clone mode. Connectors carry atomic properties including `CRTC_ID`, `link-status`, `Content Protection`, `HDR_OUTPUT_METADATA`, `VRR_CAPABLE`, and the colour management property set. Connectors must not be confused with encoders: a connector is the logical output port, while an encoder converts digital signals into the wire format.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **encoder (DRM)**, **CRTC**, **KMS**, **EDID**, **atomic modesetting**, **DDC (Display Data Channel)**

---

### Cook-Torrance BRDF
The Cook-Torrance BRDF (Bidirectional Reflectance Distribution Function) is the industry-standard microfacet specular shading model used in physically based rendering, formulated as `f_r = D(h)·F(v,h)·G(l,v,h) / (4·(n·l)·(n·v))` where D is the normal distribution function (typically GGX/Trowbridge-Reitz), F is the Fresnel term (Schlick approximation), and G is the geometric shadowing/masking term. All three terms take microfacet roughness as input, making it energy-conserving at high roughness. In practice, the Cook-Torrance specular term is combined with a Lambertian diffuse term and driven by `metallic` and `roughness` material parameters following the Disney/Unreal PBR metallic-roughness model adopted by glTF 2.0 and Filament. GLSL implementations appear in Filament's `shading_model_standard.fs`.

**Chapter(s):** Ch83 — Filament; Ch64 — glTF 2.0; Ch56 — Ray Tracing on Linux  
**Related:** **PBR (Physically Based Rendering)**, **IBL (Image-Based Lighting)**, **GI (Global Illumination)**, **glTF 2.0**

---

### CRF (Constant Rate Factor)
A rate control mode in x264, x265, and libaom that targets a constant perceptual quality level rather than a fixed bitrate; the encoder allocates more bits to complex scenes and fewer to simple ones, allowing file size to vary freely. CRF is a logarithmic quality scale (lower = higher quality): x264's default is 23, x265's is 28, libaom's `--end-usage=q` uses a quantizer parameter (CQP) analagously. CRF is an encoder implementation concept, not a codec bitstream concept; the H.264 or HEVC bitstream produced carries no indication that CRF was used. A common misconception is that CRF is the same as Constant Bitrate (CBR) or Variable Bitrate (VBR); it is neither — it is a perceptual quality target.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch57 — FFmpeg Architecture and Programming  
**Related:** **codec**, **rate-distortion optimization**, **PSNR (Peak Signal-to-Noise Ratio)**, **VMAF (Video Multi-Method Assessment Fusion)**, **H.264 / AVC**, **H.265 / HEVC**, **NVENC**

---

### CRTC
A `struct drm_crtc` (defined in `include/drm/drm_crtc.h`) representing one independent display timing engine — historically named after the CRT controller chip that generated the horizontal and vertical sync signals. Each CRTC drives one display scanout pipeline: it consumes one or more planes, composites them in hardware, and clocks the result out through an encoder/connector chain. The number of CRTCs bounds the maximum number of independent displays a GPU can drive simultaneously. In atomic modesetting, CRTC properties include the active `MODE`, `ACTIVE` flag, `GAMMA_LUT`, `DEGAMMA_LUT`, `CTM`, and VRR/HDR extensions. A common misconception is that each CRTC corresponds to a physical monitor; in MST configurations a single CRTC may feed a virtual connector.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **KMS**, **plane (DRM)**, **encoder (DRM)**, **connector (DRM)**, **atomic modesetting**, **vblank**, **mode (display mode)**

---

### CTM (Color Transform Matrix)
A 3×3 colour transformation matrix applied by the display hardware between the degamma and gamma LUT stages of the KMS colour pipeline. Represented in the kernel as `struct drm_color_ctm` in `include/uapi/drm/drm_mode.h`, containing nine S31.32 fixed-point coefficients that transform from one set of colour primaries to another (e.g. wide-gamut BT.2020 content to the panel's native primaries). Set via the `CTM` atomic property on a CRTC. Not all hardware implements a hardware CTM; drivers that lack it may emulate it by folding the matrix into the gamma LUT. The CTM must be distinguished from the `wp_color_representation_v1` coefficient set, which encodes YCbCr-to-RGB conversion for video buffers upstream of the CRTC pipeline.

**Chapter(s):** Ch3 — Advanced Display Features; Ch53 — Display Calibration and colord  
**Related:** **color LUT**, **degamma**, **CRTC**, **wide color gamut (WCG)**, **BT.2020**

---

### CUDA
CUDA (Compute Unified Device Architecture) is NVIDIA's proprietary parallel computing platform and programming model that exposes GPU SMs (Streaming Multiprocessors) as a grid of thread blocks, each block subdivided into warps of 32 threads sharing L1/shared memory. The CUDA software stack on Linux consists of `libcuda.so` (the driver API, versioned against the kernel module, entry point `cuInit`) and `libcudart.so` (the runtime API, entry point `cudaInitDevice`); mismatched versions cause `CUDA_ERROR_INVALID_PTX`. CUDA kernels are written in CUDA C++ (an extension of C++), compiled via `nvcc` to PTX intermediate assembly and then to SASS (device-specific binary) by the JIT compiler in the driver. On Linux, CUDA requires either the proprietary NVIDIA kernel module or `nvidia-open` (R525+, Ch9).

**Chapter(s):** Ch66 — CUDA Runtime, Streams, and NVRTC  
**Related:** **CUDA stream**, **cuGraph**, **NVRTC**, **SASS (NVIDIA shader assembly)**, **tensor core / XMX**, **warp (NVIDIA thread group)**

---

### CUDA stream
A CUDA stream is an ordered, asynchronous GPU work queue in which operations (kernel launches, memory copies, and events) are guaranteed to execute in issue order relative to each other, but different streams may overlap on the GPU. The default stream 0 imposes implicit synchronisation barriers between all CUDA calls from all streams in the same process, which is often undesirable; `cudaStreamCreateWithFlags(stream, cudaStreamNonBlocking)` creates a stream that does not synchronise with stream 0. Stream creation, synchronisation, and ordering are managed by `cudaStreamCreate`, `cudaStreamSynchronize`, and `cudaStreamWaitEvent`; stream priorities (0 = default, negative = higher priority) are set via `cudaStreamCreateWithPriority`. CUDA streams are conceptually analogous to Vulkan `VkQueue` submission queues.

**Chapter(s):** Ch66 — CUDA Runtime, Streams, and NVRTC  
**Related:** **CUDA**, **cuGraph**, **NVRTC**, **compute shader**

---

### cuGraph
cuGraph (CUDA Graphs, accessed via `cudaGraph_t`) is a CUDA API for pre-recording a directed acyclic graph of GPU operations — kernel launches (`cudaGraphAddKernelNode`), memory copies (`cudaGraphAddMemcpyNode`), and dependencies — that can then be instantiated (`cudaGraphInstantiate`) and repeatedly launched (`cudaGraphLaunch`) with minimal CPU overhead. Stream capture (`cudaStreamBeginCapture` → work → `cudaStreamEndCapture`) is the ergonomic way to record a graph from existing stream-based code. Once instantiated, graph parameters can be updated without reinstantiation via `cudaGraphExecKernelNodeSetParams`. cuGraph is distinct from the RAPIDS cuGraph graph analytics library; both use the `cuGraph` namespace prefix in different contexts, which is a common source of confusion.

**Chapter(s):** Ch66 — CUDA Runtime, Streams, and NVRTC  
**Related:** **CUDA**, **CUDA stream**, **NVRTC**

---

## D

### DASH (Dynamic Adaptive Streaming over HTTP)
An ISO/IEC standard (ISO/IEC 23009-1) for adaptive bitrate streaming over plain HTTP that exposes a Media Presentation Description (MPD) XML manifest describing video/audio renditions and their segment URLs. Unlike HLS, DASH is container-agnostic; in practice CMAF-packaged fMP4 segments are used for maximum CDN interoperability. A DASH player fetches and parses the MPD, selects an Adaptation Set and Representation for each period, then requests segments using template URLs or SegmentBase/SegmentList addressing. The DASH-IF Interoperability Points constrain the standard to a testable subset for live and VOD profiles. Low-Latency DASH (CTE or chunked transfer) further reduces glass-to-glass latency by delivering incomplete segments as CMAF chunks.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery; Ch57 — FFmpeg Architecture and Programming  
**Related:** **ABR (Adaptive Bitrate Streaming)**, **MPD (MPEG-DASH Media Presentation Description)**, **CMAF (Common Media Application Format)**, **HLS (HTTP Live Streaming)**, **video segment**, **BOLA (Buffer Occupancy–based Lyapunov Algorithm)**

---

### DCP (Display Co-Processor)
Apple Silicon's dedicated firmware-controlled display controller, separate from the AGX GPU, that manages the full KMS pipeline on M-series Macs running Asahi Linux. DCP runs proprietary RTKit firmware and exposes its functionality through an Apple-specific mailbox IPC protocol rather than standard MMIO registers. The Linux kernel driver for DCP is maintained under `drivers/gpu/drm/apple/` in the Asahi Linux kernel tree (not yet in Linus's tree as of kernel 6.10); a simpler pre-DCP display pipe driver (`drivers/gpu/drm/adp/`) handles the Touch Bar display on older Apple hardware that did reach mainline. DCP's architecture diverges from conventional DRM CRTC drivers because mode-setting commands are sent as serialised RTKit messages rather than register writes.

**Chapter(s):** Ch6 — ARM & Embedded GPU Drivers  
**Related:** **KMS**, **CRTC**, **atomic modesetting**, **connector (DRM)**

---

### DCS (Device Control String)
An ANSI/ISO 6429 C1 control function used to send data to a terminal device. The DCS string begins with `ESC P` (or 0x90 in 8-bit mode) followed by optional parameter bytes, an intermediate byte, and a data string, terminated by String Terminator `ST` (`ESC \`). DCS is the framing mechanism for the Sixel image protocol: a Sixel stream is `ESC P <params> q <sixel-data> ST`. DCS is also used by the DECRQSS (Request Status String) and DECCRA (Copy Rectangular Area) sequences in VT220+ terminals. Kitty Graphics Protocol does not use DCS; it uses APC instead, which avoids conflicts with the Sixel DCS payload namespace.

**Chapter(s):** Ch43 — Terminal Pixel Protocols — Sixel, Kitty, and iTerm2  
**Related:** **Sixel**, **APC (Application Program Command, terminal)**, **Kitty Graphics Protocol**

---

### DCT (Discrete Cosine Transform)
A linear transform that converts spatial-domain pixel residuals into frequency-domain coefficients, concentrating energy in the low-frequency components so that high-frequency coefficients can be quantized aggressively. In H.264 a 4×4 integer approximation of the DCT is applied to luma and chroma residual blocks after motion compensation; in H.265 the transform unit (TU) may be 4×4, 8×8, 16×16, or 32×32. AV1 introduces multiple transform types including DST-7 and identity transforms alongside the DCT. The quantization step that follows DCT determines perceptual quality: a larger quantizer step drops more high-frequency detail, reducing bitrate. After entropy coding, the decoder reverses quantization via IDCT to reconstruct the residual. The DCT is distinct from the DWT (discrete wavelet transform) used in JPEG 2000.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **macroblock**, **entropy coding (CABAC/CAVLC)**, **transform coding**, **intra-prediction**, **inter-prediction**, **rate-distortion optimization**

---

### DDC (Display Data Channel)
A two-wire I²C bus (typically running at 100 kHz) embedded in HDMI and DisplayPort cables that allows the GPU to read EDID data from the connected monitor and to send control commands via the Display Data Channel Command Interface (DDC/CI). The kernel reads DDC via `drm_do_get_edid()` in `drivers/gpu/drm/display/drm_dp_helper.c` (for DP) or through the generic I²C adapter registered on the connector. On DisplayPort the equivalent mechanism is the AUX channel rather than DDC; HDMI sources use DDC pin 5 (SCL) and pin 6 (SDA) of the connector. The primary use of DDC in the kernel is EDID retrieval at hotplug detect time.

**Chapter(s):** Ch2 — KMS: The Display Pipeline  
**Related:** **EDID**, **connector (DRM)**, **DisplayPort**, **hot-plug detect (HPD)**

---

### Dear ImGui
An immediate-mode graphical user interface library (C++, header-only core) widely used in GPU tools and game development for in-process debug overlays, profilers, and editors. "Immediate mode" means the UI is rebuilt every frame from scratch in CPU code — there is no retained widget tree. ImGui renders via a backend that issues Vulkan, OpenGL, or other API draw calls; the Vulkan backend uses a single `VkDescriptorPool`, `VkPipeline`, and vertex/index `VkBuffer` pair managed per-frame. It is the UI layer used by tools such as RenderDoc, Nsight, and Tracy's companion viewer. The canonical repository is [https://github.com/ocornut/imgui](https://github.com/ocornut/imgui).

**Chapter(s):** Ch30 — Debugging & Profiling  
**Related:** **Tracy (profiler)**, **VkPipeline**, **VkCommandBuffer**

---

### DeepStream (NVIDIA)
NVIDIA's GStreamer-based SDK for GPU-accelerated video analytics and inference pipelines, running on Jetson SoCs and discrete datacenter GPUs. DeepStream provides a set of proprietary GStreamer plugins (`nvstreammux`, `nvinfer`, `nvtracker`, `nvmsgbroker`) that ingest RTSP/RTMP camera streams via NVDEC hardware decode, run TensorRT inference in the decoded frame domain via CUDA, and emit structured metadata downstream. The core buffer type is `NvBufSurface`, a wrapper around CUDA-accessible DMA-BUF memory that avoids CPU copies across GStreamer element boundaries. DeepStream is not a replacement for GStreamer but a proprietary extension layer above it; pipeline graph construction follows the standard `gst_element_link` / `gst_pad_link` API.

**Chapter(s):** Ch59 — NVIDIA DeepStream SDK; Ch58 — GStreamer: Pipeline-Based Multimedia  
**Related:** **GStreamer**, **NVDEC**, **NVENC**, **FFmpeg**, **PipeWire (multimedia)**, **V4L2 (Video4Linux2)**

---

### degamma
The first colour-pipeline stage in KMS, which applies a lookup table to linearise the nonlinear (gamma-encoded) pixel values arriving from a framebuffer before they are processed by the CTM and gamma stages. Set via the `DEGAMMA_LUT` atomic property on a CRTC; the table is a `struct drm_color_lut[]` array stored as a blob property. For sRGB content the degamma LUT approximates an inverse sRGB transfer function (roughly gamma 2.2), converting encoded values to linear light. For HDR content mastered with PQ encoding the degamma LUT encodes the inverse ST 2084 EOTF. Omitting the degamma step when applying a CTM on nonlinear data produces incorrect colour mathematics; this is a common bug in early HDR compositor implementations.

**Chapter(s):** Ch3 — Advanced Display Features; Ch53 — Display Calibration and colord  
**Related:** **color LUT**, **CTM (Color Transform Matrix)**, **sRGB**, **PQ (Perceptual Quantizer / ST 2084)**, **CRTC**

---

### Descriptor Set
A Vulkan object (`VkDescriptorSet`) that binds a group of resources — uniform buffers, sampled images, storage buffers, samplers — to shader binding points. Descriptor sets are allocated from a `VkDescriptorPool` according to a `VkDescriptorSetLayout` that declares the binding slots and resource types. Multiple sets (up to `maxBoundDescriptorSets`, at least 4) can be bound simultaneously at different set indices, allowing frequent-change resources (per-draw) to occupy a high-numbered set while infrequent resources (per-frame, per-material) stay in lower-numbered sets. Unlike OpenGL's implicit binding state, Vulkan descriptor sets make resource binding explicit and composable.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch25 — GPU Compute  
**Related:** **descriptor set layout**, **VkDescriptorPool**, **pipeline layout**, **bindless resources**, **push constant**

---

### Descriptor Set Layout
A Vulkan object (`VkDescriptorSetLayout`) that describes the shape of a descriptor set: the binding indices, descriptor types (`VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`, `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`, etc.), and shader stage visibility of each slot. Created once and shared across many `VkDescriptorSet` allocations, it is the schema against which the driver builds hardware descriptor table formats. Descriptor set layouts are composed into a `VkPipelineLayout`; the layout must match the `layout(set=N, binding=M)` declarations in SPIR-V shaders. Mismatches cause `VK_ERROR_INCOMPATIBLE_DRIVER` or validation errors.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **descriptor set**, **pipeline layout**, **VkPipeline**, **SPIR-V cooperative matrix (VK_KHR_cooperative_matrix)**

---

### disk_cache (Mesa)
Mesa's disk cache is a persistent, on-disk store for compiled shader binaries and pipeline objects, implemented in `src/util/disk_cache.c`. When a shader is compiled, Mesa serialises the resulting driver binary (post-NIR, post-ACO/BRW) into a keyed cache entry under `~/.cache/mesa_shader_cache/` (XDG-compliant). On subsequent runs, if the cache key — derived from the shader source hash, driver version, and hardware identifiers — matches, Mesa skips recompilation and loads the binary directly. This is distinct from Fossilize (which serialises Vulkan pipeline state for replay) and from the kernel-level GPU command buffer caches. Disabling it (via `MESA_SHADER_CACHE_DISABLE=true`) forces full recompilation and is useful for shader compiler regression testing.

**Chapter(s):** Ch12 — The Mesa Loader and Driver Dispatch; Ch15 — ACO: AMD's Optimising Compiler; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **shader cache**, **Fossilize**, **ACO**, **NIR**, **shader-db**

---

### DisplayPort
A royalty-free digital display interface standardised by VESA that carries video, audio, and auxiliary data over a main link of one to four differential lanes. Key Linux-relevant features include Multi-Stream Transport (MST) for daisy-chaining multiple monitors, Display Stream Compression (DSC) for high-resolution/high-refresh outputs over limited bandwidth, and the AUX channel (a low-bandwidth bidirectional sideband used for EDID, link training, and DPCD register access). Kernel helpers live in `drivers/gpu/drm/display/drm_dp_helper.c` and `drm_dp_mst_topology.c`. DisplayPort Adaptive-Sync is the VESA specification underlying AMD FreeSync and NVIDIA G-Sync Compatible on Linux.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **MST (Multi-Stream Transport)**, **DSC (Display Stream Compression)**, **DDC (Display Data Channel)**, **EDID**, **FreeSync**, **G-Sync**, **VRR (Variable Refresh Rate)**

---

### DLSS 4
DLSS 4 (Deep Learning Super Sampling 4) is NVIDIA's transformer-model–based neural rendering technology introduced for Ada Lovelace and Blackwell GPUs at CES 2025, succeeding the CNN-based DLSS 2/3. Its super-resolution component uses a vision transformer with self-attention over a temporal history buffer of previous frames to reconstruct high-frequency detail suppressed in sub-native rendering. DLSS 4 extends frame generation to Multi Frame Generation (MFG), producing 1–4 (DLSS 4) or 1–6 (DLSS 4.5, June 2026) synthetic intermediate frames per rendered frame using an optical flow estimator and inpainting network. Applications integrate DLSS 4 through the NGX Vulkan API: `NVSDK_NGX_VULKAN_Init`, `NVSDK_NGX_VULKAN_CreateFeature1`, and `NVSDK_NGX_VULKAN_EvaluateFeature`; required Vulkan extensions include `VK_NV_optical_flow` and `VK_NV_low_latency2`.

**Chapter(s):** Ch68 — DLSS 4, Neural Rendering, and Frame Generation  
**Related:** **Frame Generation (NVIDIA)**, **tensor core / XMX**, **TensorRT**, **NRD (NVIDIA Real-time Denoisers)**, **FSR 4**

---

### DMA fence
A kernel synchronisation primitive (`struct dma_fence`, defined in `include/linux/dma-fence.h`, implemented in `drivers/dma-buf/dma-fence.c`) representing a point in time when an asynchronous GPU or DMA operation completes. Renamed from `struct fence` in kernel 4.10 to avoid namespace collisions. A `dma_fence` has a context (an integer identifying the timeline), a sequence number, a signalled flag, and a list of callbacks invoked when the fence fires. In the DRM context, fences are the primitive underlying both implicit synchronisation (via `dma_resv`) and explicit synchronisation (via `drm_syncobj` and `sync_file`). Do not confuse with a Vulkan `VkFence`, which is a GPU-to-CPU signalling object at the API level and is typically backed by a `dma_fence` in Mesa drivers.

**Chapter(s):** Ch4 — GPU Memory Management; Ch3 — Advanced Display Features  
**Related:** **DMA-resv**, **drm_syncobj**, **sync file**, **implicit synchronisation**, **explicit synchronisation**, **fence (DRM)**

---

### DMA-BUF
A kernel framework (`drivers/dma-buf/dma-buf.c`) for sharing memory buffers across different kernel drivers and subsystems without copying. A `struct dma_buf` is exported by one driver (e.g. a GPU driver creating a GEM object) as a file descriptor via `DRM_IOCTL_PRIME_HANDLE_TO_FD`, then imported by another driver (e.g. a V4L2 camera, a VA-API codec, or a Wayland compositor) via `dma_buf_get()` / `DRM_IOCTL_PRIME_FD_TO_HANDLE`. DMA-BUF carries an attached `struct dma_resv` (reservation object) which holds implicit fences tracking read and write usage across all importers/exporters. Userspace shares DMA-BUF fds through `linux-dmabuf-unstable-v1` in Wayland and through `EGL_EXT_image_dma_buf_import` in EGL. Do not confuse with GEM handles, which are process-local integers, not transferable file descriptors.

**Chapter(s):** Ch4 — GPU Memory Management; Ch20 — Wayland Protocol Fundamentals; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **GEM (Graphics Execution Manager)**, **DMA-resv**, **DMA fence**, **implicit synchronisation**, **explicit synchronisation**, **PRIME**, **GBM (Generic Buffer Manager)**

---

### DMA-resv
A `struct dma_resv` (defined in `include/linux/dma-resv.h`, formerly called `reservation_object`, renamed ~5.4) that embeds into every `struct dma_buf` and serves as the container for implicit fence tracking on a shared buffer. It holds one exclusive write fence and a list of shared read fences. When a GPU job writes to a DMA-BUF it installs an exclusive fence; GPU jobs that only read the buffer install shared fences. Drivers and subsystems that consume a DMA-BUF (e.g. the KMS scanout path, V4L2 capture) call `dma_resv_wait()` to block until all prior GPU work completes before accessing the buffer. DMA-resv is the kernel mechanism behind implicit synchronisation; moving to explicit synchronisation means bypassing `dma_resv` in favour of `drm_syncobj` timelines passed explicitly through Wayland or Vulkan.

**Chapter(s):** Ch4 — GPU Memory Management; Ch3 — Advanced Display Features  
**Related:** **DMA-BUF**, **DMA fence**, **implicit synchronisation**, **explicit synchronisation**, **drm_syncobj**

---

### drm_syncobj
A `struct drm_syncobj` (implemented in `drivers/gpu/drm/drm_syncobj.c`) — a kernel-managed container that holds a `dma_fence` and is exposed to userspace as an integer handle via `DRM_IOCTL_SYNCOBJ_CREATE`. Unlike a `sync_file` fd, a syncobj is mutable: its internal fence can be replaced via `DRM_IOCTL_SYNCOBJ_SIGNAL` or updated to a new point by importing a fence from a completed GPU submission. Timeline syncobjs (introduced alongside `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_FOR_SUBMIT`) attach a `dma_fence_chain` to implement monotonically increasing sequence points, which directly backs Vulkan timeline semaphores. Syncobjs are the kernel mechanism the `linux-drm-syncobj-v1` Wayland protocol extension exports to compositor clients for explicit frame synchronisation.

**Chapter(s):** Ch3 — Advanced Display Features; Ch4 — GPU Memory Management; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **DMA fence**, **sync file**, **timeline semaphore**, **explicit synchronisation**, **linux-drm-syncobj-v1**, **DMA-resv**

---

### DSC (Display Stream Compression)
A visually lossless compression standard (VESA Display Stream Compression 1.2) applied to the DisplayPort or HDMI video stream between the GPU and the monitor to reduce link bandwidth requirements. DSC enables 4K 144 Hz, 8K, or high-bit-depth HDR resolutions that would otherwise exceed available lane bandwidth. The GPU encoder breaks the image into slices, compresses each slice independently using a modified predictive coding algorithm, and the monitor decompresses in real time. Linux kernel DSC helpers reside in `drivers/gpu/drm/display/drm_dsc_helper.c`. Individual GPU drivers (i915, amdgpu, nouveau) configure DSC parameters via their display engines. Lossless in practice means the standard targets a rate-distortion budget perceptually indistinguishable from uncompressed at the specified BPP target.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **DisplayPort**, **BPP (Bits Per Pixel)**, **connector (DRM)**, **MST (Multi-Stream Transport)**

---

### DXC (DirectX Shader Compiler)
DXC is Microsoft's open-source HLSL compiler, hosted at `https://github.com/microsoft/DirectXShaderCompiler`, built on an LLVM/Clang fork. It compiles HLSL source to DXIL (DirectX Intermediate Language, an LLVM IR dialect) or optionally to SPIR-V via the `spirv-export` backend (`-spirv` flag). On Linux, DXC is used by VKD3D-Proton and DXVK to translate DirectX 12/11 shaders to SPIR-V for submission to Mesa Vulkan drivers. DXC is distinct from the older `fxc` compiler (which emitted DXBC) and from `glslang` (which handles GLSL/HLSL to SPIR-V). The SPIR-V emitted by DXC enters the standard Mesa NIR front end identically to SPIR-V from any other source.

**Chapter(s):** Ch28 — Windows Compatibility; Ch61 — SPIR-V Ecosystem in Depth  
**Related:** **HLSL**, **SPIR-V**, **SPIRV-Tools**, **spirv-cross**, **glslang**, **NIR**

---

### Dynamic Rendering (VK_KHR_dynamic_rendering)
A Vulkan 1.3 core feature (originally `VK_KHR_dynamic_rendering`) that eliminates the need to create `VkRenderPass` and `VkFramebuffer` objects before recording rendering commands. Instead, `vkCmdBeginRendering` takes a `VkRenderingInfo` struct naming the color and depth/stencil `VkImageView` attachments inline. This simplifies the API for renderers that do not benefit from tiled subpass merging (common on mobile), allows attachment formats to be specified at command-recording time, and reduces object proliferation. The trade-off is that tile-based GPU drivers may lose subpass-merge optimizations that explicit render pass subpasses could exploit.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **render pass**, **subpass**, **VkRenderPass**, **VkFramebuffer**, **frame graph (render graph pattern)**

---

## E

### EDID
Extended Display Identification Data — a 128-byte (or longer, with 256-byte CEA extension blocks) binary structure stored in a monitor's EEPROM and read by the GPU over DDC or the DP AUX channel. EDID encodes the monitor's manufacturer, supported resolutions and refresh rates, maximum pixel clock, colour space coverage (including BT.2020 / DCI-P3 primaries via HDR static metadata Type 1 in CEA-861-H), and HDCP capability. The kernel parses EDID in `drivers/gpu/drm/drm_edid.c` and uses it to populate connector modes and capabilities exposed to userspace via `DRM_IOCTL_MODE_GETCONNECTOR`. A missing or corrupt EDID causes KMS to fall back to a safe-modes list; the `drm.edid_firmware` kernel parameter allows overriding with a file-based EDID blob.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **DDC (Display Data Channel)**, **connector (DRM)**, **mode (display mode)**, **HDR metadata**, **hot-plug detect (HPD)**

---

### EGL (Embedded/Extensible Graphics Library)
A platform-integration API defined by Khronos that sits between OpenGL ES / OpenGL / Vulkan rendering contexts and the native window system. On Linux, EGL provides `eglGetDisplay` (accepting `EGLNativeDisplayType`, commonly a `wl_display *` or a `gbm_device *`), `eglCreateContext`, `eglCreateWindowSurface`, and `eglSwapBuffers`. Mesa implements EGL in `src/egl/`; the Wayland backend (`platform_wayland.c`) allocates GBM-backed buffers and passes them to the compositor via the `linux-dmabuf` Wayland protocol. EGL is not a rendering API itself — it does not issue draw calls — but is the mandatory surface and context management layer for OpenGL ES and the recommended path for Vulkan WSI surface creation on embedded platforms.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch12 — The Mesa Loader and Driver Dispatch; Ch34 — ANGLE — WebGL on Linux  
**Related:** **VkSwapchainKHR**, **EGLSyncKHR**, **GLES (OpenGL ES)**, **VkInstance**, **swapchain**

---

### EGLSyncKHR
An EGL synchronization object (`EGLSyncKHR`, from `EGL_KHR_fence_sync` and `EGL_ANDROID_native_fence_sync`) that wraps a GPU timeline point in the EGL API. `eglCreateSyncKHR(EGL_SYNC_FENCE_KHR)` inserts a fence into the current command stream; `eglClientWaitSyncKHR` blocks the CPU until the GPU passes that point. The Android variant (`EGL_SYNC_NATIVE_FENCE_ANDROID`) exports the sync to a Linux `sync_file` file descriptor via `eglDupNativeFenceFDANDROID`, enabling interop with the kernel explicit sync framework (`DRM_SYNCOBJ`). In Mesa's Wayland EGL platform, `EGLSyncKHR` is the EGL-visible surface of the same `drm_syncobj` mechanism used by `zwp_linux_explicit_synchronization_v1`.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch3 — Advanced Display Features; Ch46 — The Evolving Wayland Protocol Ecosystem  
**Related:** **EGL (Embedded/Extensible Graphics Library)**, **VkFence (binary)**, **VkSemaphore**, **explicit sync**

---

### encoder (DRM)
A `struct drm_encoder` (defined in `include/drm/drm_encoder.h`) representing the signal-encoding block between a CRTC and a connector — for example, a TMDS encoder for HDMI, a LVDS encoder for a laptop panel, or a DP PHY for DisplayPort. Encoders are a legacy abstraction: their primary role was to model hardware that physically serialises the digital video bus into a wire-level signal. In modern atomic KMS drivers most of the useful work has moved into bridges (`drm_bridge`) and connector helpers, and the encoder struct is often a thin wrapper satisfying the KMS object graph requirement (CRTC → encoder → connector). An encoder can serve multiple connectors in clone mode (same content on multiple outputs), but each connector can attach to only one encoder at a time.

**Chapter(s):** Ch2 — KMS: The Display Pipeline  
**Related:** **CRTC**, **connector (DRM)**, **KMS**, **atomic modesetting**

---

### entropy coding (CABAC/CAVLC)
The final lossless compression stage in a block-based video encoder that maps quantized transform coefficients and syntax elements to a compact bitstream. H.264 defines two entropy coding modes: Context-Adaptive Binary Arithmetic Coding (CABAC) and Context-Adaptive Variable-Length Coding (CAVLC). CABAC achieves roughly 10–15% better compression by adapting a binary arithmetic model using per-symbol context states; it is mandatory in H.264 Main and High profiles. CAVLC is a Huffman-variant with lower computational cost; it is required in Baseline profile (used for real-time mobile encoding). H.265/HEVC mandates CABAC exclusively. AV1 uses a non-binary arithmetic coder (derived from Daala's `daala_ec`), which allows multi-symbol coding and higher throughput than binary CABAC; AV1 does not use ANS despite early proposals to do so.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **DCT (Discrete Cosine Transform)**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **NAL unit (H.264/H.265)**, **macroblock**, **transform coding**

---

### EU ISA (Intel Execution Unit ISA)
The Intel Execution Unit (EU) ISA is the native machine code executed by Intel GPU shader processors, spanning from Gen 4 (Broadwell era) through Xe2 (Battlemage). Each EU is a SIMD vector processor running SIMD8/SIMD16/SIMD32 lanes; the ISA includes send instructions for memory and sampler access, branch/flow control, math instructions, and typed/untyped surface messages. The ISA is not publicly documented by Intel but is reverse-engineered and implemented in Mesa's BRW/ELK compiler and in Intel's IGC. Gen 12.5 (Alchemist/Arc) introduced a new SIMD32-first threading model; Xe2 added further ISA changes requiring the `brw_compiler` split. Understanding EU ISA is necessary for writing Mesa shader compiler passes or analysing Intel GPU shader assembly output.

**Chapter(s):** Ch18 — Vulkan Drivers; Ch19 — OpenGL and Compatibility Drivers  
**Related:** **BRW compiler (Intel EU ISA)**, **ANV**, **NIR**, **register allocation (GPU)**

---

### explicit synchronisation
A GPU synchronisation discipline in which the producer of a buffer passes a fence object *explicitly* to the consumer, and the consumer waits on that fence before accessing the buffer, rather than relying on the kernel to track fences implicitly via `dma_resv`. On Linux, explicit sync is implemented through `drm_syncobj` handles exchanged between driver command-submission ioctls and, at the Wayland level, through the `linux-drm-syncobj-v1` protocol extension (acquire and release points on `zwp_linux_buffer_release_v1`). The motivation was NVIDIA's proprietary driver, which could not safely participate in `dma_resv`-based implicit sync, causing tearing or stalls when compositors imported NVIDIA buffers. Vulkan mandates explicit synchronisation at the API level via semaphores and fences; driver implementations typically translate these to `drm_syncobj` operations.

**Chapter(s):** Ch3 — Advanced Display Features; Ch20 — Wayland Protocol Fundamentals; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **implicit synchronisation**, **drm_syncobj**, **DMA-resv**, **linux-drm-syncobj-v1**, **sync file**, **DMA fence**, **timeline semaphore**

---

## F

### fence (DRM)
In DRM/KMS contexts, "fence" is an overloaded term. (1) At the kernel level, a DRM fence is a `dma_fence` signalled when a GPU job or display operation completes, and is the primitive behind both implicit and explicit sync. (2) In the KMS page-flip path, a fence may be attached to a `drm_plane_state.fence` to delay scanout until the GPU finishes rendering the new framebuffer. (3) In Vulkan, a `VkFence` is a GPU-to-CPU sync object (not a GPU-to-GPU primitive); Mesa drivers typically back it with a `dma_fence` polled from CPU. (4) In Android, "sync fence" historically referred to a `sync_file` fd wrapping a `dma_fence`. These four usages share the word but operate at different abstraction layers; conflating them is a common source of confusion.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features; Ch4 — GPU Memory Management  
**Related:** **DMA fence**, **sync file**, **drm_syncobj**, **implicit synchronisation**, **explicit synchronisation**, **page flip**

---

### FFmpeg
A comprehensive open-source multimedia framework providing libraries (`libavformat`, `libavcodec`, `libavfilter`, `libavdevice`, `libswscale`, `libswresample`, `libpostproc`) and the `ffmpeg`, `ffprobe`, and `ffplay` CLI tools. `libavcodec` wraps software codec implementations (x264, x265, libaom, libvpx) and hardware acceleration backends (VA-API, VDPAU, V4L2 M2M, NVENC/NVDEC, Vulkan). `libavformat` handles container muxing/demuxing across MP4, MKV, TS, HLS, DASH, SRT, and RTMP. The API uses a send/receive model: `avcodec_send_packet()` pushes compressed data, `avcodec_receive_frame()` retrieves decoded `AVFrame` objects. Hardware acceleration is configured via `AVHWDeviceContext` attached to an `AVCodecContext`; the Vulkan hwaccel path uses `AVVkFrame` for zero-copy interop with Vulkan Video.

**Chapter(s):** Ch57 — FFmpeg Architecture and Programming; Ch58 — GStreamer: Pipeline-Based Multimedia  
**Related:** **AVCodec / AVCodecContext (FFmpeg)**, **GStreamer**, **NVENC**, **NVDEC**, **V4L2 (Video4Linux2)**, **HLS (HTTP Live Streaming)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **SRT (Secure Reliable Transport)**

---

### FidelityFX SDK
The FidelityFX SDK is AMD's open-source (MIT) collection of rendering effect and upscaling algorithms packaged as GPU-vendor-neutral GLSL/HLSL compute shader libraries, distributed via `github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK`. Each effect (FSR, CAS, DENOISER, SSSR, SPD, DOF, Hybrid Reflections, etc.) is packaged as a header-only or thin-API effect component callable from any Vulkan or DirectX 12 application. On Linux, integration requires the Vulkan backend (`ffx_api_vk.h`) and passes `VkDevice`, `VkPhysicalDevice`, and queue handles to `ffxCreateContext`. The SDK also includes the FidelityFX Shader Compiler (`FidelityFX-SC`) that cross-compiles HLSL→GLSL→SPIR-V for portable deployment. RGP and the FidelityFX Frame Interpolation overlay are companion tools distributed alongside the SDK.

**Chapter(s):** Ch72 — AMD FidelityFX & Tools  
**Related:** **FSR 4**, **CAS**, **RGP (Radeon GPU Profiler)**, **compute shader**

---

### FIFO mode (Wayland presentation)
The `wp_fifo_v1` Wayland protocol (wayland-protocols, staging) that enforces strict first-in, first-out frame ordering for a `wl_surface`, ensuring a submitted frame is displayed before any later frame. Without `wp_fifo_v1`, a Wayland compositor may skip intermediate frames when the application renders faster than the display refresh (analogous to Vulkan's `VK_PRESENT_MODE_MAILBOX_KHR`). With `wp_fifo_v1`, the compositor must honour every submitted frame in order, equivalent to `VK_PRESENT_MODE_FIFO_KHR`. The protocol adds a `wp_fifo_v1.set_barrier()` / `wp_fifo_v1.wait_barrier()` roundtrip to pace the client, avoiding runaway buffer queuing. It is the Wayland complement to `wp_presentation` timing feedback for latency-sensitive but tearing-intolerant clients.

**Chapter(s):** Ch46 — Wayland Protocol Ecosystem  
**Related:** **wl_surface**, **wp_viewport**, **wp_linux_drm_syncobj_surface_v1**, **zwp_linux_dmabuf_v1**

---

### Filament (rendering engine)
Filament is Google's open-source, production-quality physically based rendering engine for Android, Linux, macOS, and web, designed to deliver high-fidelity PBR rendering with predictable resource budgets. Its rendering backend abstracts Vulkan, OpenGL ES, Metal, and WebGPU through a `DriverApi` interface; on Linux the Vulkan backend is the primary target, allocating resources via VMA and submitting command buffers to Mesa Vulkan drivers (RADV, ANV). Filament's material system uses a custom domain-specific language compiled offline to GLSL/SPIR-V shaders via `matc`; materials implement the Cook-Torrance BRDF with clearcoat, anisotropy, and cloth extensions. The `filamat` compiler emits backend-specific shader variants and bakes them into `.filamat` binary material archives.

**Chapter(s):** Ch83 — Filament; Ch18 — Vulkan Drivers  
**Related:** **PBR (Physically Based Rendering)**, **cook-torrance BRDF**, **IBL (Image-Based Lighting)**, **render graph**, **bgfx**

---

### Fossilize
Fossilize is a Vulkan pipeline state serialisation library and toolset (`https://github.com/ValveSoftware/Fossilize`) developed by Valve. It records `VkPipeline`, `VkRenderPass`, `VkDescriptorSetLayout`, and associated shader modules into a portable `.foz` database, allowing later replay to pre-warm Mesa's disk cache without re-running the application. Steam uses Fossilize to distribute pre-compiled shader databases so games avoid first-run stutter. The `fossilize-replay` tool can stress-test pipeline compilation in parallel and is widely used by RADV and ANV developers for regression testing. Fossilize operates at the Vulkan API level above Mesa, but its output feeds directly into Mesa's `disk_cache` via normal `vkCreateGraphicsPipelines` calls during replay.

**Chapter(s):** Ch16 — Mesa's Vulkan Common Infrastructure; Ch28 — Windows Compatibility; Ch30 — Debugging & Profiling  
**Related:** **disk_cache (Mesa)**, **shader cache**, **RADV**, **ANV**, **SPIR-V**

---

### Frame Generation (NVIDIA)
NVIDIA Frame Generation (part of DLSS 3 and 4) is a technique that synthesises additional rendered frames between actual engine-rendered frames using GPU-resident optical flow data and an AI inpainting network, increasing displayed frame rate without proportionally increasing render workload. The optical flow estimator computes a dense per-pixel motion field from consecutive rendered frames using dedicated Optical Flow hardware (Ampere+). Frame Generation is integrated via the NGX SDK (`NVSDK_NGX_VULKAN_CreateFeature1` with `NVSDK_NGX_Feature_FrameGeneration`); it requires `VK_NV_optical_flow` and `VK_NV_low_latency2` Vulkan extensions and a Turing or later GPU (though full multi-frame generation requires Ada Lovelace or newer). A common misconception is that Frame Generation reduces actual render latency — it increases displayed framerate but does not reduce input-to-photon latency, which NVIDIA Reflex addresses separately.

**Chapter(s):** Ch68 — DLSS 4, Neural Rendering, and Frame Generation; Ch29 — Upscaling, Effects & Overlays  
**Related:** **DLSS 4**, **tensor core / XMX**, **XeSS**, **FSR 4**

---

### Frame Graph (render graph pattern)
An architectural pattern for GPU renderers in which all render passes, their resource inputs and outputs, and inter-pass dependencies are declared as a directed acyclic graph before any command recording begins. The frame graph then performs automatic render target lifetime management, layout transitions, barrier insertion, and pass culling (removing passes whose outputs are unused). Popularized by Yuriy O'Donnell's GDC 2017 talk on Frostbite and later adopted in engines such as Bevy's `RenderGraph` and Godot 4's `RenderingDevice` pipeline. In Vulkan the frame graph translates to `VkRenderPass`/`VkSubpassDependency` or `VK_KHR_dynamic_rendering` with explicit `vkCmdPipelineBarrier2` calls.

**Chapter(s):** Ch40 — Bevy and wgpu; Ch41 — Godot 4 RenderingDevice; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **render pass**, **subpass**, **dynamic rendering (VK_KHR_dynamic_rendering)**, **VkRenderPass**, **pipeline layout**

---

### framebuffer (DRM)
A `struct drm_framebuffer` (defined in `include/drm/drm_framebuffer.h`, created via `DRM_IOCTL_MODE_ADDFB2`) that wraps one or more GEM object handles and describes how the GPU and scanout hardware should interpret the pixel data: width, height, pixel format (DRM fourcc), pitch in bytes, and a DRM format modifier encoding the tiling layout or hardware compression state. A framebuffer is attached to a plane (`drm_plane_state.fb`) and presented to the display by a CRTC. Multiple framebuffers can reference the same underlying GEM allocation; the driver's `drm_framebuffer_funcs.create_handle` allows userspace to import a framebuffer's buffer by PRIME fd. Note that a DRM framebuffer is not an OpenGL framebuffer object (`GL_FRAMEBUFFER`); they are distinct concepts at entirely different layers.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch4 — GPU Memory Management  
**Related:** **GEM (Graphics Execution Manager)**, **plane (DRM)**, **CRTC**, **scan-out**, **PRIME**, **GBM (Generic Buffer Manager)**

---

### freedreno
freedreno is the open-source Mesa Gallium3D driver for Qualcomm Adreno GPUs (A2xx through A7xx), implementing OpenGL ES and OpenGL on top of the `msm` DRM kernel driver. Its source lives in `src/gallium/drivers/freedreno/`; the driver submits GPU command streams via `msm_gem_submit` ioctls wrapped by the `libmsm` userspace library. freedreno also provides the OpenGL ES backend underneath the Mesa Turnip Vulkan driver (`src/freedreno/vulkan/`), sharing the Adreno compiler (`src/compiler/isaspec/`, `ir3/`) and register database between both drivers. A key design feature is the `ir3` intermediate representation (analogous to Mesa NIR but Adreno-specific), into which SPIR-V and GLSL are translated before ISA codegen.

**Chapter(s):** Ch19 — OpenGL and Compatibility Drivers; Ch6 — ARM & Embedded GPU Drivers  
**Related:** **ACE (Adreno Compute Engine)**, **compute unit (CU)**, **compute shader**

---

### FreeSync
AMD's implementation of VESA Adaptive-Sync for DisplayPort and HDMI, branded as FreeSync (with tiered FreeSync Premium and FreeSync Premium Pro designations). It enables Variable Refresh Rate (VRR) operation where the monitor's refresh period tracks the GPU's frame production rate within a supported range (e.g. 48–144 Hz), eliminating tearing without the latency of V-sync. In the Linux kernel VRR/FreeSync support is enabled through the `VRR_CAPABLE` and `FREESYNC` connector properties and the `VRR_ENABLED` CRTC property in `drivers/gpu/drm/amd/display/`. FreeSync is not an NVIDIA feature; NVIDIA hardware uses G-Sync or the separate G-Sync Compatible mode (which uses the same VESA Adaptive-Sync electrical standard). FreeSync Premium Pro additionally requires HDR support.

**Chapter(s):** Ch3 — Advanced Display Features  
**Related:** **VRR (Variable Refresh Rate)**, **G-Sync**, **atomic modesetting**, **connector (DRM)**, **DisplayPort**

---

### FSR 4 (FidelityFX Super Resolution 4)
FSR 4 is AMD's fourth-generation spatial and temporal AI-accelerated upscaling algorithm, released in 2025 targeting RDNA4 (RX 9000 series) GPUs that include dedicated AI accelerators (matrix units). Unlike FSR 2/3 which are GPU-vendor-neutral temporal algorithms, FSR 4 leverages machine learning inference (a transformer-based spatial reconstruction network) and therefore delivers highest quality on hardware with tensor-like compute capability. It is distributed as part of the FidelityFX SDK; the Vulkan integration uses `ffxFsr4ContextCreate` and dispatches via compute shaders targeting `VK_KHR_cooperative_matrix` on supported hardware. On older RDNA hardware, FSR 3 (spatial-temporal) remains the recommended path. A common misconception is that FSR 4 is purely a resolution upscaler — it also includes temporal anti-aliasing.

**Chapter(s):** Ch72 — AMD FidelityFX & Tools; Ch29 — Upscaling, Effects & Overlays  
**Related:** **FidelityFX SDK**, **CAS**, **DLSS 4**, **XeSS**, **tensor core / XMX**

---

## G

### G-Sync
NVIDIA's proprietary Variable Refresh Rate technology, originally requiring a dedicated NVIDIA G-Sync module inside the monitor that communicated with the GPU over a proprietary protocol rather than VESA Adaptive-Sync. G-Sync Compatible is a later certification that allows VRR on VESA Adaptive-Sync monitors with NVIDIA GPUs. On Linux, G-Sync hardware monitors are not supported by Nouveau due to the proprietary protocol; NVIDIA's proprietary driver added VRR support for G-Sync Compatible (Adaptive-Sync) panels. In the kernel DRM layer, both FreeSync and G-Sync Compatible resolve to the same `VRR_ENABLED` CRTC property mechanism; the branding difference reflects validation and feature tiers rather than kernel API differences.

**Chapter(s):** Ch3 — Advanced Display Features  
**Related:** **VRR (Variable Refresh Rate)**, **FreeSync**, **connector (DRM)**, **DisplayPort**

---

### Gallium3D
Gallium3D is Mesa's internal architecture for state trackers (API frontends) and pipe drivers (GPU backends), defined by the `pipe_context`, `pipe_screen`, and `pipe_resource` interfaces in `src/gallium/include/pipe/`. A single state tracker — e.g., the OpenGL/ES state tracker `mesa/state_tracker`, or `rusticl` for OpenCL — compiles to NIR and calls into the pipe driver, which translates NIR to hardware ISA. GPU pipe drivers include `radeonsi` (AMD), `iris` (Intel Gen 8+), `nouveau` (NVIDIA), `panfrost` (ARM Mali), and `asahi` (Apple AGX). Gallium3D is not used by Vulkan drivers, which bypass it entirely in favour of the direct NIR-to-ISA path and Mesa's Vulkan common layer.

**Chapter(s):** Ch13 — Gallium3D: The OpenGL State Tracker; Ch17 — Software Renderers; Ch19 — OpenGL and Compatibility Drivers  
**Related:** **NIR**, **Mesa**, **disk_cache (Mesa)**, **LLVM (in Mesa context)**, **ACO**

---

### Gamescope
A nested Wayland compositor developed by Valve, used as the session compositor on Steam Deck and as a game-framing tool on desktop Linux. Gamescope runs as a Wayland client of the host compositor (or directly on DRM/KMS as a standalone compositor) and presents game windows as Wayland surfaces that it then composites — applying upscaling (FSR 1/2/3, NIS), colour management, and HDR tone-mapping — before submitting the final frame. It uses `wlroots` primitives internally and exploits KMS plane promotion to hand off unscaled game buffers directly to display hardware overlays, bypassing GPU compositing when possible. On Steam Deck, Gamescope controls HDMI/DisplayPort output via atomic KMS commits and manages VRR (FreeSync) negotiation.

**Chapter(s):** Ch22 — Production Compositors; Ch29 — Upscaling, Effects and Overlays  
**Related:** **wlroots**, **liftoff (KMS composition library)**, **zwp_linux_dmabuf_v1**, **MangoHUD**

---

### GBM (Generic Buffer Manager)
A userspace library (`src/gbm/` in the Mesa repository) that provides a portable API for allocating GPU-tiled, GPU-compatible buffers suitable for EGL surface creation and KMS scan-out. `gbm_device_create()` opens a DRM render node; `gbm_bo_create()` or `gbm_surface_create()` allocates a buffer object backed by a GEM allocation in the driver. GBM buffer objects can be exported as DMA-BUF fds for sharing with Wayland compositors via the `linux-dmabuf-unstable-v1` protocol. GBM is not a kernel subsystem — it lives entirely in userspace (Mesa) and issues DRM ioctls to the kernel. It is effectively a thin portability shim around the driver-specific GEM allocation ioctl; the GBM API is defined in `include/gbm.h` and each Mesa driver provides a backend under `src/gbm/backends/dri/`.

**Chapter(s):** Ch4 — GPU Memory Management; Ch20 — Wayland Protocol Fundamentals; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **GEM (Graphics Execution Manager)**, **DMA-BUF**, **framebuffer (DRM)**, **scan-out**, **KMS**

---

### GCN ISA
GCN (Graphics Core Next) ISA is AMD's shader instruction set architecture introduced with the Southern Islands (HD 7000) series in 2012 and used through RDNA 1 (RX 5000 series, 2019). GCN introduced a scalar ALU alongside the vector ALU, explicit wavefront-level programming (64-thread wavefronts), and a unified memory model that underpins ROCm/HIP. In Mesa, the ACO and LLVM-AMDGPU backends both emit GCN machine code for older AMD hardware; the `amdgcn-amd-amdhsa` LLVM target triple identifies GCN. GCN ISA is distinct from RDNA ISA — RDNA 1/2/3/4 brought 32-thread waves, wave32/wave64 duality, and new instruction encodings. Do not use "GCN" generically to mean all AMD GPU ISA; RDNA hardware should be referred to by its own ISA name.

**Chapter(s):** Ch15 — ACO: AMD's Optimising Compiler; Ch19 — OpenGL and Compatibility Drivers; Ch48 — ROCm and Machine Learning on Linux GPUs  
**Related:** **ACO**, **RDNA ISA**, **RADV**, **LLVM (in Mesa context)**, **register allocation (GPU)**

---

### GEM (Graphics Execution Manager)
The primary kernel GPU buffer object framework, introduced in kernel 2.6.28 for Intel's i915 driver and since adopted by virtually all DRM drivers. A GEM object (`struct drm_gem_object`, defined in `include/drm/drm_gem.h`) represents a kernel-managed GPU-accessible memory allocation. Userspace references GEM objects by driver-local integer *handles* obtained via driver-specific allocation ioctls (e.g. `DRM_IOCTL_I915_GEM_CREATE`, `DRM_IOCTL_AMDGPU_GEM_CREATE`); handles are process-local and cannot be passed to other processes directly. Cross-process sharing is done via PRIME: a GEM object is exported as a DMA-BUF fd with `DRM_IOCTL_PRIME_HANDLE_TO_FD`. TTM is an alternative, lower-level memory manager that GEM objects may use as their backing store.

**Chapter(s):** Ch4 — GPU Memory Management; Ch1 — DRM Architecture & the Driver Model  
**Related:** **TTM (Translation Table Manager)**, **DMA-BUF**, **PRIME**, **GBM (Generic Buffer Manager)**, **framebuffer (DRM)**

---

### GI (Global Illumination)
Global Illumination (GI) refers to rendering algorithms that simulate indirect light transport — light that reaches a surface after bouncing off one or more other surfaces — as opposed to direct illumination which considers only the primary light-to-surface path. In real-time GPU rendering, GI is approximated via screen-space techniques (SSGI, SSAO), precomputed irradiance probes (RTXGI's SHaRC and NRC), or stochastic path tracing (ReSTIR). Hardware ray tracing (RT Cores, Ch56) accelerates GI by enabling efficient secondary ray casts; the RTXGI SDK provides SHaRC and NRC implementations on top of `VK_KHR_ray_tracing_pipeline`. Offline renderers such as Blender Cycles achieve GI via unbiased Monte Carlo path tracing using OptiX or CUDA kernels.

**Chapter(s):** Ch70 — RTX Kit; Ch56 — Ray Tracing on Linux; Ch42 — Blender  
**Related:** **RTXGI**, **SHaRC**, **Neural Radiance Cache (NRC)**, **IBL (Image-Based Lighting)**, **ReSTIR / RTXDI**, **BVH**

---

### GLES (OpenGL ES)
OpenGL for Embedded Systems, a subset of OpenGL defined by Khronos for resource-constrained and mobile hardware. GLES 2.0 (2007) removed the fixed-function pipeline and required programmable shaders; GLES 3.0 (2012) added transform feedback, instanced rendering, and uniform buffer objects; GLES 3.2 (2015) added geometry and tessellation shaders, making it nearly equivalent to OpenGL 4.3 core profile. On Linux, GLES contexts are created via EGL (not GLX). Mesa implements GLES via the same Gallium3D state tracker as desktop OpenGL; ANGLE translates GLES to Vulkan, Metal, or D3D for portability. A common misconception is that GLES is "old" — GLES 3.2 covers Vulkan 1.0-era features.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch34 — ANGLE — WebGL on Linux; Ch13 — Gallium3D: The OpenGL State Tracker  
**Related:** **OpenGL**, **EGL (Embedded/Extensible Graphics Library)**, **VkPipeline**, **SPIR-V cooperative matrix (VK_KHR_cooperative_matrix)**

---

### GLSL (OpenGL Shading Language)
GLSL is the C-like shading language defined by the OpenGL specification for writing vertex, fragment, geometry, tessellation, and compute shaders. In Mesa, GLSL source is parsed and compiled to NIR by the `glsl` front end (`src/compiler/glsl/`), which handles GLSL versions 1.10 through 4.60 and GLSL ES 1.00/3.20. Unlike SPIR-V, GLSL retains driver-side compilation at runtime, meaning each driver sees the raw GLSL text. Common misconceptions: GLSL and HLSL are syntactically similar but semantically distinct (e.g., matrix multiplication order, coordinate conventions); GLSL is not WGSL. The `glslang` reference compiler can optionally pre-validate GLSL and emit SPIR-V, but Mesa's own front end is authoritative for OpenGL conformance.

**Chapter(s):** Ch13 — Gallium3D: The OpenGL State Tracker; Ch14 — NIR: Mesa's Shader Intermediate Representation; Ch19 — OpenGL and Compatibility Drivers  
**Related:** **NIR**, **glslang**, **SPIR-V**, **WGSL**, **HLSL**

---

### glslang
glslang is the Khronos reference compiler for GLSL and HLSL, hosted at `https://github.com/KhronosGroup/glslang`. It parses GLSL (including Vulkan-flavour GLSL with `GL_KHR_vulkan_glsl` extensions) and HLSL, performs semantic validation, and emits SPIR-V. In Vulkan toolchains it is typically invoked via the `glslangValidator` CLI tool or the `glslang` C++ library API. `shaderc` wraps `glslang` as its GLSL/HLSL front end. Unlike Mesa's internal GLSL front end, `glslang` targets cross-vendor Vulkan/SPIR-V; it is also used to translate GLSL shaders in DXVK's Wine layer. Do not confuse `glslang` with Mesa's internal GLSL compiler — the two are separate codebases, and Mesa does not use `glslang` internally for OpenGL.

**Chapter(s):** Ch14 — NIR: Mesa's Shader Intermediate Representation; Ch24 — Vulkan and EGL for Application Developers; Ch61 — SPIR-V Ecosystem in Depth  
**Related:** **GLSL**, **SPIR-V**, **shaderc**, **SPIRV-Tools**, **DXC**, **HLSL**

---

### glTF 2.0
glTF 2.0 (GL Transmission Format 2.0) is a Khronos open standard (June 2017) JSON-based 3D asset format designed for efficient runtime delivery of GPU-ready geometry, materials, animations, and scene graphs. A `.gltf` file references binary `.bin` buffers for mesh data (vertex/index buffers directly uploadable via `VkBuffer`) and `.ktx2` textures; the `.glb` container packages all three into one binary file. The glTF 2.0 PBR material model uses `baseColorFactor`/`metallicRoughnessTexture` following the Disney metallic-roughness workflow, mapping directly to Cook-Torrance BRDF parameters. Extensions such as `KHR_materials_clearcoat`, `KHR_lights_punctual`, and `KHR_draco_mesh_compression` are ratified Khronos extensions; `EXT_mesh_gpu_instancing` exposes GPU instancing metadata.

**Chapter(s):** Ch64 — glTF 2.0  
**Related:** **PBR (Physically Based Rendering)**, **cook-torrance BRDF**, **IBL (Image-Based Lighting)**, **OpenUSD / USD**

---

### Gralloc (Android)
The Android Graphics Allocator HAL (`hardware/interfaces/graphics/allocator`), responsible for allocating physically contiguous, GPU-accessible, optionally hardware-protected memory buffers shared across processes. Gralloc 4.0 (IAllocator 2.0 / IMapper 4.0 HIDL/AIDL, Android 11+) allocates buffers with explicit usage flags (CPU read/write, GPU render target, hardware composer overlay, video decoder output) and returns opaque handles. Drivers implement Gralloc as a vendor HAL (e.g., Arm's `gralloc_arm` for Mali, `libgbm`-based on Mesa-backed devices). Gralloc is not a Gallium-layer concept; it exists entirely within Android's HAL layer and has no direct equivalent on desktop Linux, where GBM fills a similar but different role.

**Chapter(s):** Ch82 — Android HardwareBuffer and Cross-Process GPU Memory; Ch80 — SurfaceFlinger and the Android Display Pipeline  
**Related:** **AHardwareBuffer**, **HWComposer (Android)**, **BufferQueue (Android)**, **ANativeWindow**

---

### Graphics Pipeline Library (VK_EXT_graphics_pipeline_library)
A Vulkan extension (promoted to an important optimization path in Vulkan 1.3 ecosystem) that allows a `VkPipeline` to be created in independently linkable stages — vertex input, pre-rasterization shaders, fragment shader, and fragment output — rather than requiring all state upfront. The individual `VkPipeline` library stages can be compiled ahead of time and then fast-linked into a final executable pipeline at draw time. This substantially reduces stutter caused by late pipeline compilation in games, and is exploited by DXVK and VKD3D-Proton to front-load shader compilation. It does not replace `VkPipelineCache`; both are used together.

**Chapter(s):** Ch18 — Vulkan Drivers; Ch28 — Windows Compatibility; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **VkPipeline**, **pipeline cache**, **VkPipelineCache**, **shader object (VK_EXT_shader_object)**, **VkShaderModule**

---

### GStreamer
A pipeline-based open-source multimedia framework for Linux (and other platforms) in which media processing is expressed as a directed graph of `GstElement` objects linked by `GstPad` connections. Data flows as `GstBuffer` objects carrying `GstCaps` format negotiation metadata; zero-copy GPU paths use `GstDmaBufAllocator` to share DMA-BUF file descriptors across elements without CPU readback. Key plugin sets include `gst-plugins-base`, `gst-plugins-good`, `gst-plugins-bad`, `gst-plugins-ugly`, and vendor extensions such as DeepStream (NVIDIA) and `gstreamer-vaapi` (Mesa VA-API bridge). The `gst-libav` plugin wraps `libavcodec` codecs as GStreamer elements. GStreamer's SPA-node abstraction is analogous to—but separate from—PipeWire's SPA node, though PipeWire ships `pipewiresrc`/`pipewiresink` elements for interoperability.

**Chapter(s):** Ch58 — GStreamer: Pipeline-Based Multimedia; Ch59 — NVIDIA DeepStream SDK  
**Related:** **FFmpeg**, **DeepStream (NVIDIA)**, **PipeWire (multimedia)**, **V4L2 (Video4Linux2)**, **NVDEC**, **NVENC**

---

### GuC (Graphics Micro Controller, Intel)
The GuC is an embedded ARM-based microcontroller present on Intel Gen9+ GPUs (i915 and Xe drivers) that handles GPU command submission scheduling, power management, and — on Gen12+ — context save/restore for GPU preemption. Firmware is loaded from `/lib/firmware/i915/` (e.g., `adlp_guc_70.bin`) by `intel_guc_fw_init_early` in the i915 driver; GuC scheduling (`intel_guc_submission_enable`) replaces the legacy execlist submission model. On the Xe kernel driver (used for Intel Arc and Meteor Lake iGPU), GuC is mandatory for all command submission (`xe_guc.c`). A common misconception is that GuC replaces the HuC — they coexist: GuC handles graphics scheduling, HuC handles video encode/decode.

**Chapter(s):** Ch5 — x86 GPU Drivers; Ch51 — GPU Power Management  
**Related:** **HuC (Video Micro Controller, Intel)**, **Level Zero (Intel)**, **IOMMU group**

---

## H

### H.264 / AVC
The video codec specified in ITU-T H.264 and ISO/IEC 14496-10 (MPEG-4 AVC), published in 2003 and the most widely deployed codec on Linux today. H.264 organises video into slices of macroblocks (16×16 luma samples); intra-predicted slices use spatial prediction while inter-predicted slices use motion vectors against reference frames stored in the Decoded Picture Buffer (DPB). The bitstream is packaged in NAL units. Key profiles: Baseline (no B-frames, CAVLC only), Main, High (CABAC, B-frames, 8×8 transform). Software implementations: x264 (encoder), OpenH264 (Cisco, encoder/decoder), FFmpeg's built-in h264 decoder (libavcodec), and the JM reference software. Note: libde265 is an H.265/HEVC decoder, not an H.264 decoder. Hardware implementations on Linux: AMD VCN, Intel Quick Sync (MFX), NVIDIA NVENC/NVDEC, exposed via VA-API (`VAProfileH264High`) and V4L2 M2M.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch26 — Hardware Video; Ch50 — Vulkan Video Extensions  
**Related:** **H.265 / HEVC**, **AV1**, **codec**, **NAL unit (H.264/H.265)**, **IDR frame**, **I-frame**, **B-frame**, **P-frame**, **macroblock**, **entropy coding (CABAC/CAVLC)**, **intra-prediction**, **inter-prediction**, **NVENC**, **NVDEC**, **V4L2 (Video4Linux2)**

---

### H.265 / HEVC
The video codec specified in ITU-T H.265 and ISO/IEC 23008-2 (High Efficiency Video Coding), published in 2013 as the successor to H.264. HEVC replaces H.264's macroblock with a Coding Tree Unit (CTU) hierarchy that allows coding units up to 64×64 samples, enabling better large-area prediction and reduced overhead for 4K/UHD content. HEVC mandates CABAC entropy coding exclusively and adds Sample Adaptive Offset (SAO) and adaptive loop filtering. Software implementations: x265 (encoder), libde265 (decoder). Hardware: AMD VCN-2+, Intel Quick Sync Gen9+, NVIDIA NVENC/NVDEC (Maxwell+), exposed via VA-API (`VAProfileHEVCMain`) and Vulkan Video (`VK_KHR_video_decode_h265`). Royalty concerns led major browser vendors to prefer AV1; HEVC remains dominant in broadcast and OTT delivery.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch26 — Hardware Video; Ch50 — Vulkan Video Extensions  
**Related:** **H.264 / AVC**, **AV1**, **codec**, **NAL unit (H.264/H.265)**, **IDR frame**, **I-frame**, **macroblock**, **entropy coding (CABAC/CAVLC)**, **DCT (Discrete Cosine Transform)**, **NVENC**, **NVDEC**

---

### HDCP (High-bandwidth Digital Content Protection — GPU security context)
HDCP is a content protection protocol (HDCP 1.4 and 2.3) mandated by content providers for protected media playback over HDMI and DisplayPort. In the Linux kernel, HDCP is implemented within the DRM/KMS layer via `drm_hdcp.c` and per-driver callbacks (`drm_connector_helper_funcs.hdcp_enable`); the user-space interface is the KMS connector property `Content Protection` (values: `Undesired`, `Desired`, `Enabled`). From a GPU security perspective, HDCP links to the broader protected content pipeline: on Intel, the HuC firmware (Ch5) participates in HDCP 2.3 authentication; on AMD, the PSP (Platform Security Processor) handles key exchange (`amdgpu_hdcp.c`). Wayland compositors must propagate `Content Protection` property changes from Vulkan/EGL surfaces to KMS; this is handled by the `wp_drm_lease_v1` and related protocols.

**Chapter(s):** Ch3 — Advanced Display Features; Ch55 — GPU Containers and Cloud Compute; Ch5 — x86 GPU Drivers  
**Related:** **GuC**, **HuC (Video Micro Controller, Intel)**, **vfio (Virtual Function I/O)**, **IOMMU group**

---

### HDR metadata
Descriptor data accompanying an HDR video signal that informs the display of the mastering conditions and content characteristics so the display can correctly tone-map the signal to its own peak luminance. Two categories exist: *static metadata* (SMPTE ST.2086 + CTA-861.3), which specifies mastering display primaries, white point, min/max luminance, MaxCLL, and MaxFALL once per stream; and *dynamic metadata* (SMPTE ST.2094-10 / HDR10+, and Dolby Vision), which provides per-scene or per-frame tone-mapping guidance. In KMS, static HDR metadata is delivered through the `HDR_OUTPUT_METADATA` connector property as a `struct hdr_output_metadata` blob. Wayland clients signal HDR intent via `wp_color_management_v1` surface colour descriptions. Linux compositors as of 2025 primarily support static (HDR10) metadata; dynamic metadata standards are under active development in Mesa and the kernel.

**Chapter(s):** Ch3 — Advanced Display Features; Ch26 — Hardware Video  
**Related:** **HDR10**, **MaxCLL / MaxFALL**, **PQ (Perceptual Quantizer / ST 2084)**, **HLG (Hybrid Log-Gamma)**, **connector (DRM)**, **BT.2020**

---

### HDR10
The baseline static HDR format defined by the Consumer Technology Association (CTA-861.3) and standardised as SMPTE ST.2084 / ST.2086. HDR10 mandates BT.2020 colour primaries, the PQ (ST 2084) transfer function, 10-bit pixel depth, and static metadata (SMPTE ST.2086 mastering display primaries, MaxCLL, MaxFALL) delivered in the HDMI InfoFrame or DP SDPs. In KMS, the `HDR_OUTPUT_METADATA` connector property carries a `struct hdr_output_metadata` (`include/uapi/drm/drm_mode.h`) with a nested `struct hdr_metadata_infoframe` containing these values. HDR10 does not support dynamic per-scene metadata (unlike HDR10+); compositors set the static metadata once per swapchain surface and the GPU transmits it in every frame's InfoFrame. Most Linux GPU drivers support HDR10 output; end-to-end compositor HDR10 pipelines became usable from kernel 6.3 onward.

**Chapter(s):** Ch3 — Advanced Display Features; Ch26 — Hardware Video; Ch53 — Display Calibration and colord  
**Related:** **PQ (Perceptual Quantizer / ST 2084)**, **BT.2020**, **HDR metadata**, **MaxCLL / MaxFALL**, **wide color gamut (WCG)**, **connector (DRM)**

---

### HLG (Hybrid Log-Gamma)
A backward-compatible HDR transfer function (ITU-R BT.2100) jointly developed by the BBC and NHK for broadcast HDR. HLG uses a logarithmic curve for high-luminance signal values and a linear-gamma curve for the bottom third of the signal range, allowing SDR displays to render HLG content without metadata (the gamma segment maps to acceptable SDR brightness) while HDR displays apply the full HLG OOTF. HLG does not use static mastering metadata or MaxCLL/MaxFALL values, which distinguishes it from HDR10. In Linux graphics, HLG surfaces require an appropriate EOTF annotation in the compositor's colour management pipeline; Mesa's colour management layer recognises HLG via the `VK_EXT_swapchain_colorspace` extension enum `VK_COLOR_SPACE_BT2020_LINEAR_EXT` and related values. The KMS colour pipeline must apply the HLG OOTF in the degamma LUT for correct display.

**Chapter(s):** Ch3 — Advanced Display Features; Ch26 — Hardware Video  
**Related:** **HDR metadata**, **HDR10**, **PQ (Perceptual Quantizer / ST 2084)**, **BT.2020**, **degamma**, **wide color gamut (WCG)**

---

### HLS (HTTP Live Streaming)
Apple's adaptive bitrate streaming protocol, originally specified in RFC 8216, that delivers video as a sequence of short segments (historically MPEG-2 TS, now CMAF fMP4) referenced by M3U8 playlist files. A master playlist enumerates renditions (variant streams) with bandwidth and resolution tags; each rendition has a media playlist listing segment URLs. HLS is supported by `ffmpeg`'s `hls` muxer (`-f hls`) and GStreamer's `hlssink2` element. Low-Latency HLS (LL-HLS) extends the protocol with partial segment delivery (`#EXT-X-PART`) and HTTP/2 server push, reducing glass-to-glass latency below 2 seconds. Despite its origin at Apple, HLS is the most widely deployed ABR protocol on Linux CDN infrastructure.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery; Ch57 — FFmpeg Architecture and Programming  
**Related:** **ABR (Adaptive Bitrate Streaming)**, **M3U8 playlist**, **LL-HLS (Low-Latency HLS)**, **CMAF (Common Media Application Format)**, **video segment**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **FFmpeg**, **GStreamer**

---

### HLSL (High-Level Shader Language)
HLSL is Microsoft's C-like shading language for DirectX, used to write shaders for all DirectX shader stages. On Linux, HLSL appears primarily in the Wine/DXVK/VKD3D-Proton compatibility layer, where DirectX games ship HLSL source or pre-compiled DXBC/DXIL bytecode that must be translated to SPIR-V for Mesa Vulkan drivers. Translation paths include DXC (`-spirv`), `vkd3d-shader` (which can consume DXBC/DXIL and emit SPIR-V), and `glslang`'s HLSL front end. HLSL uses column-major matrix convention and registers (`t0`, `s0`, `b0`) whereas GLSL uses locations/bindings; these differences must be mapped carefully during translation. Slang, a newer language from NVIDIA, is designed to be a superset-of-HLSL for cross-platform shader development.

**Chapter(s):** Ch28 — Windows Compatibility; Ch61 — SPIR-V Ecosystem in Depth  
**Related:** **DXC**, **SPIR-V**, **glslang**, **Slang (NVIDIA shading language)**, **GLSL**, **WGSL**

---

### hot-plug detect (HPD)
A hardware signal on HDMI (pin 19) and DisplayPort (pin 18 of the 20-pin connector) that the monitor asserts when it is connected and powered. The GPU monitors this pin and generates an interrupt when the signal changes state. In the Linux DRM layer the interrupt triggers `drm_helper_hpd_irq_event()` which re-reads connector status and EDID, then delivers a `uevent` to udev so compositors and display managers can update the available mode list. Spurious HPD interrupts (HPD glitches) from long or poor-quality cables can cause unnecessary mode re-negotiations; some hardware exposes an HPD filter configuration. For DP MST hubs the HPD event only triggers topology re-enumeration; individual downstream port changes are discovered via DPCD reads.

**Chapter(s):** Ch2 — KMS: The Display Pipeline  
**Related:** **connector (DRM)**, **EDID**, **DDC (Display Data Channel)**, **DisplayPort**, **MST (Multi-Stream Transport)**

---

### HuC (Video Micro Controller, Intel)
The HuC is a dedicated ARM-based video microcontroller on Intel Gen9+ GPUs that offloads video encode/decode rate-control and quality-optimisation algorithms from the host CPU to firmware running on the GPU. Firmware is loaded from `/lib/firmware/i915/` (e.g., `tgl_huc_7.9.3.bin`) by `intel_huc_fw_init` in the i915/Xe driver; HuC enable status can be verified via `/sys/kernel/debug/dri/0/i915_huc_load_status`. When HuC is active, VA-API encode operations (via `libva-intel-media-driver`) use firmware-side rate control, improving encoding efficiency by 15–30% over CPU rate control. HDCP 2.3 authentication on Gen11+ also requires HuC for the key exchange protocol.

**Chapter(s):** Ch5 — x86 GPU Drivers; Ch26 — Hardware Video  
**Related:** **GuC (Graphics Micro Controller, Intel)**, **Level Zero (Intel)**, **HDCP**

---

### HWComposer (Android)
The Hardware Composer HAL (`hardware/interfaces/graphics/composer`) that SurfaceFlinger uses to hand off display layer composition to dedicated display hardware, bypassing GPU compositing when possible. For each frame, SurfaceFlinger submits a list of layers (`HwcLayer`) to the HWC HAL; the HAL marks each layer as either `DEVICE` (composed by display hardware overlay planes) or `CLIENT` (must be composited by SurfaceFlinger on the GPU into a client-target buffer). HWC2/HWC3 (Android 11+) use HIDL/AIDL interfaces; HWC3 aligns more closely with the DRM/KMS atomic plane model. Common misconception: HWComposer is not a GPU driver; it programs display controller hardware, analogous to KMS atomic commits on Linux.

**Chapter(s):** Ch80 — SurfaceFlinger and the Android Display Pipeline  
**Related:** **SurfaceFlinger**, **Gralloc**, **AHardwareBuffer**, **ANativeWindow**

---

### HWUI (Android)
Android's hardware-accelerated 2D UI rendering library (`frameworks/base/libs/hwui`), which translates View draw calls into a GPU command stream. HWUI builds a display list from `Canvas` operations, differencing it frame-to-frame (partial invalidation), then converts retained display list nodes into OpenGL ES or Vulkan draw calls. Since Android 12, HWUI defaults to a Vulkan backend (`SkiaVulkan`) using Skia's GPU module; the legacy OpenGL ES backend (`SkiaGL`) remains for older devices. HWUI manages its own texture atlas for bitmaps and glyphs, submitting the final composited frame as an `ANativeWindow` buffer to `SurfaceFlinger`. Do not confuse with `SurfaceFlinger`'s own compositor; HWUI renders app content, while SurfaceFlinger composites app layers.

**Chapter(s):** Ch81 — HWUI and the Android View Rendering Pipeline  
**Related:** **SurfaceFlinger**, **ANativeWindow**, **BufferQueue (Android)**, **Gralloc**

---

## I

### I-frame
An intra-coded video frame that is compressed using only spatial prediction within the frame itself, with no reference to other frames; it serves as a random-access point in a Group of Pictures (GOP). I-frames use intra-prediction of luma and chroma samples from already-decoded neighbouring blocks in the same frame, followed by DCT and entropy coding of the residual. Because I-frames carry full spatial information, they are substantially larger than P-frames or B-frames. An important distinction: not all I-frames are IDR frames. Open-GOP I-frames may be referenced by B-frames from a prior GOP; only IDR frames guarantee a clean Decoded Picture Buffer reset and a true seek point. Encoders typically place I-frames every 2–10 seconds, controlled by the keyframe interval parameter.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **IDR frame**, **P-frame**, **B-frame**, **intra-prediction**, **macroblock**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**

---

### IBL (Image-Based Lighting)
Image-Based Lighting is a technique that derives scene illumination from a high-dynamic-range panoramic environment map (typically an equirectangular or cubemap image), enabling realistic and physically consistent ambient and specular contributions without placing explicit light sources. In PBR pipelines, IBL splits the reflectance integral into a diffuse irradiance convolution (stored as a low-resolution irradiance cubemap or spherical harmonics) and a specular pre-filtered environment map combined with the BRDF LUT (split-sum approximation by Brian Karis). On GPU, the pre-filtering pass is a compute shader operating over cubemap mip levels; Filament's `IBL baker` (`tools/cmgen`) generates the pre-filtered maps offline. In glTF 2.0, IBL is specified via the `KHR_lights_image_based` extension (`imageBasedLight` node property).

**Chapter(s):** Ch83 — Filament; Ch64 — glTF 2.0; Ch56 — Ray Tracing on Linux  
**Related:** **PBR (Physically Based Rendering)**, **cook-torrance BRDF**, **GI (Global Illumination)**, **glTF 2.0**

---

### IDR frame
An Instantaneous Decoder Refresh frame in H.264/AVC and H.265/HEVC: an I-frame that additionally triggers a flush of the Decoded Picture Buffer (DPB) and resets all reference frame lists. After an IDR, the decoder may not use any frame decoded before the IDR as a reference; this makes IDR frames the only guaranteed clean random-access entry points in a bitstream. In H.264 the IDR is a specific NAL unit type (nal_unit_type == 5); in H.265 IDR frames are NAL types 19 (IDR_W_RADL) and 20 (IDR_N_LP). The critical misconception: an I-frame is not always an IDR frame. Open-GOP encoding inserts I-frames that are NOT IDR frames so that preceding B-frames can reference forward into the new GOP, improving compression. HLS and DASH segment boundaries should coincide with IDR frames, not merely I-frames.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **I-frame**, **B-frame**, **P-frame**, **NAL unit (H.264/H.265)**, **H.264 / AVC**, **H.265 / HEVC**, **video segment**, **SEI (Supplemental Enhancement Information, H.264)**

---

### IGraphicBufferProducer
The Android Binder interface (`frameworks/native/libs/gui/include/gui/IGraphicBufferProducer.h`) that the producer side of a `BufferQueue` exposes over IPC. Its key methods are `dequeueBuffer()` (returns a slot index and a fence the producer must wait for before writing), `queueBuffer()` (returns the filled buffer to the consumer along with a fence, timestamp, and crop rect), `connect()` / `disconnect()` to register a producer API (CPU, OpenGL ES, media, camera), and `requestBuffer()` to materialise a `GraphicBuffer` from a slot index. `Surface` in Java and `ANativeWindow` in NDK both wrap `IGraphicBufferProducer` under the hood. This interface is internal to Android's framework; NDK code should prefer `ANativeWindow` or `AHardwareBuffer`.

**Chapter(s):** Ch80 — SurfaceFlinger and the Android Display Pipeline; Ch82 — Android HardwareBuffer and Cross-Process GPU Memory  
**Related:** **BufferQueue (Android)**, **ANativeWindow**, **SurfaceFlinger**, **android::Fence**

---

### imgui (see Dear ImGui)
See **Dear ImGui**. The project's canonical short name is `imgui`; the repository and header are `imgui.h`. The term "ImGui" is also used generically for the immediate-mode GUI paradigm, but in Linux graphics tooling it almost always refers to ocornut's library specifically.

**Chapter(s):** Ch30 — Debugging & Profiling  
**Related:** **Dear ImGui**, **Tracy (profiler)**

---

### implicit synchronisation
A GPU synchronisation model in which the kernel automatically tracks in-flight GPU work on a `dma_buf` via the `dma_resv` reservation object, and consumers (such as the KMS scanout engine or another GPU job) automatically wait for in-flight fences without any explicit fence passing in userspace. All classic OpenGL and the majority of X11/Wayland GL stacks rely on implicit sync: the application renders, the kernel records the rendering fence on the GEM/DMA-BUF object, and the compositor's KMS page-flip waits for that fence automatically. Implicit sync breaks down with NVIDIA's proprietary driver (which manages GPU timelines internally and cannot register fences on `dma_resv` objects owned by other drivers), leading to the push for explicit synchronisation via `drm_syncobj`. The move to explicit sync is now the preferred path for new Wayland and Vulkan code.

**Chapter(s):** Ch3 — Advanced Display Features; Ch4 — GPU Memory Management; Ch20 — Wayland Protocol Fundamentals  
**Related:** **DMA-resv**, **explicit synchronisation**, **DMA fence**, **DMA-BUF**, **drm_syncobj**

---

### inter-prediction
A compression technique that reconstructs a block of the current frame by referencing previously decoded frames (or future frames for B-frames), transmitting only a motion vector and a residual rather than full pixel data. In H.264, inter-prediction operates on macroblocks (16×16) and sub-partitions down to 4×4 luma samples; in H.265, on coding units within a CTU; in AV1, on coding blocks up to 128×128 with compound prediction modes. The motion vector points to the best-matching region in a reference frame found during motion estimation at encode time. Inter-prediction typically achieves far higher compression than intra-prediction for regions of slow-moving video. Inter-prediction is the mechanism; motion estimation is the encoder-side algorithm that computes the motion vectors.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **intra-prediction**, **motion estimation**, **motion vector**, **I-frame**, **P-frame**, **B-frame**, **macroblock**, **DCT (Discrete Cosine Transform)**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**

---

### intra-prediction
A spatial prediction technique that reconstructs a block's pixels by extrapolating from already-decoded neighbouring blocks within the same frame, without referencing other frames. H.264 defines 9 intra-prediction modes for 4×4 luma blocks and 4 modes for 16×16 luma blocks; H.265 defines 35 total modes: 33 angular/directional modes (modes 2–34) plus DC (mode 1) and Planar (mode 0); AV1 defines 13 directional modes plus additional compound and recursive filtering modes. The residual between the prediction and the actual block is then DCT-coded and entropy-coded. Intra-prediction is the exclusive prediction mode in I-frames and IDR frames, and is used for individual macroblocks or coding units within P-frames and B-frames when inter-prediction would be suboptimal. Intra-prediction quality determines the baseline quality of keyframes and thus segment switching quality in ABR.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **inter-prediction**, **I-frame**, **IDR frame**, **macroblock**, **DCT (Discrete Cosine Transform)**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**

---

### IOMMU
The Input-Output Memory Management Unit — a hardware unit (Intel VT-d, AMD-Vi / IOMMU) that provides virtual address translation for DMA operations issued by PCIe devices such as GPUs. The IOMMU enforces that a GPU's DMA engine can only access memory regions explicitly mapped for it, providing isolation between devices and preventing a compromised or buggy GPU from reading arbitrary host RAM. In Linux, `drivers/iommu/` implements IOMMU support; GPU drivers call `dma_map_*` APIs which internally program the IOMMU page tables via `iommu_map()`. For GPU passthrough to virtual machines, VFIO (`drivers/vfio/`) uses the IOMMU to assign a dedicated IOMMU domain to the VM, combined with PCIe ACS to isolate the device group. The `CONFIG_IOMMU_API` kernel option must be enabled for VFIO GPU passthrough to function.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute; Ch4 — GPU Memory Management  
**Related:** **ACS (PCIe Access Control Services)**, **DMA-BUF**, **GEM (Graphics Execution Manager)**, **TTM (Translation Table Manager)**

---

### IOMMU group
An IOMMU group is the smallest unit of isolation enforced by the system IOMMU (Input-Output Memory Management Unit) — all PCI devices within the same group must be managed together because they share a translation domain (DMA address space) and cannot be independently assigned to different virtual machines or user-space drivers. On Linux, IOMMU groups are exposed via `/sys/kernel/iommu_groups/N/devices/`; the `vfio-pci` driver requires that all devices in a group are bound to VFIO before any can be passed through to a VM. For GPU pass-through, the GPU and its companion HDMI audio function must be in the same IOMMU group (or different groups); ACS (Access Control Services) PCIe capability can split combined groups. IOMMU group membership is a hardware topology property that cannot be changed in software.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute; Ch5 — x86 GPU Drivers  
**Related:** **vfio (Virtual Function I/O)**, **confidential computing (GPU)**, **SEV-SNP (AMD)**, **VRAM**

---

### ION allocator
A contiguous DMA buffer allocator that originated in the Android kernel tree (`drivers/staging/android/ion/`) to provide a unified API for allocating physically contiguous or IOMMU-mapped buffers shared across camera, codec, GPU, and display hardware on Android SoCs. ION exported buffers as DMA-BUF fds, making it the forerunner of the generalised DMA-BUF sharing model. ION was staged out of the mainline kernel by kernel 5.11 (its staging entry was removed in 5.11, the subsystem itself around 5.12) and replaced by the DMA-BUF Heaps framework (`drivers/dma-buf/heaps/`), which provides per-heap character devices (`/dev/dma_heap/system`, `/dev/dma_heap/cma`) with a simpler `DMA_HEAP_IOCTL_ALLOC` interface. Code still referencing ION in out-of-tree Android BSP kernels is using a deprecated API; new code should use DMA-BUF heaps.

**Chapter(s):** Ch4 — GPU Memory Management; Ch6 — ARM & Embedded GPU Drivers  
**Related:** **DMA-BUF**, **GEM (Graphics Execution Manager)**, **IOMMU**

---

### ISA (Instruction Set Architecture)
In the GPU context, ISA refers to the binary instruction encoding and semantic specification for a GPU shader processor, analogous to x86 or ARM ISA on CPUs. Each GPU family has its own ISA: AMD has GCN and RDNA; Intel has its EU ISA (Gen 4–12.5, then Xe2+); NVIDIA has SASS (the native ISA) and PTX (a virtual ISA); Qualcomm Adreno has its own ISA reverse-engineered for Turnip/freedreno. Mesa shader compilers (ACO, BRW, LLVM-AMDGPU, etc.) are responsible for lowering NIR to the target ISA. Unlike CPU ISAs, GPU ISAs are typically proprietary and undocumented by the vendor, requiring reverse engineering for open-source driver development.

**Chapter(s):** Ch14 — NIR: Mesa's Shader Intermediate Representation; Ch15 — ACO: AMD's Optimising Compiler; Ch18 — Vulkan Drivers  
**Related:** **GCN ISA**, **RDNA ISA**, **EU ISA (Intel Execution Unit ISA)**, **SASS (NVIDIA ISA)**, **NVIR / PTX**, **ACO**, **NIR**

---

## K

### Kitty Graphics Protocol
A terminal graphics protocol designed by Kovid Goyal for the Kitty terminal emulator, framed using APC (`ESC _` … `ESC \`). Unlike Sixel, which is a streaming raster format, the Kitty protocol sends raw image data (encoded in base64) in chunked payloads, referencing images by a 32-bit numeric ID assigned by the terminal. Images can be transmitted via direct base64 in-band, via shared POSIX memory (`t=s`), via temporary files (`t=t`), or via DMA-BUF file descriptors (`t=d`) for zero-copy GPU-resident delivery. A separate placement command (`a=p`) positions a previously uploaded image at a cell coordinate, enabling efficient re-display without re-transmission. The protocol is supported by Kitty, WezTerm, foot, and Ghostty.

**Chapter(s):** Ch43 — Terminal Pixel Protocols — Sixel, Kitty, and iTerm2; Ch45 — Terminal Integration with the Compositor Stack  
**Related:** **APC (Application Program Command, terminal)**, **Sixel**, **DCS (Device Control String)**

---

### KMS (Kernel Mode Setting)
The Linux kernel subsystem that manages the display pipeline — taking over mode-setting from userspace or the BIOS and performing it inside the kernel to eliminate flicker during driver load, enable early boot graphics, and enforce security boundaries. KMS exposes a set of abstract objects (CRTC, encoder, connector, plane, framebuffer) via DRM ioctls; historically set via `DRM_IOCTL_MODE_SETCRTC` (legacy path) and now via `DRM_IOCTL_MODE_ATOMIC` (atomic path). The KMS API is defined primarily in `include/uapi/drm/drm_mode.h` and implemented in `drivers/gpu/drm/drm_crtc.c`, `drm_atomic.c`, and related files. Wayland compositors operate exclusively as KMS clients via libDRM; X.org's modesetting DDX also uses KMS. KMS should not be confused with "modesetting" (the act); KMS is the subsystem, modesetting is the operation.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features; Ch1 — DRM Architecture & the Driver Model  
**Related:** **CRTC**, **plane (DRM)**, **connector (DRM)**, **encoder (DRM)**, **atomic modesetting**, **modesetting**, **framebuffer (DRM)**, **vblank**

---

### KTX2 (Khronos Texture 2.0)
A container file format (`.ktx2`) defined by the Khronos Group for storing GPU textures, including compressed formats, mipmaps, array layers, and cubemap faces, in a layout suitable for direct upload to GPU memory. KTX2 mandates Basis Universal supercompression as its primary codec, enabling runtime transcoding to platform-native formats (BC7, ASTC, ETC2) via `libktx`'s `ktxTexture2_TranscodeBasis`. Unlike KTX1, KTX2 stores data in mip-major order (matching GPU memory layouts) and uses a `VkFormat` identifier rather than the legacy OpenGL format enums. It is the standard texture container for glTF 2.0 (`KHR_texture_basisu`) and is integrated into Vulkan workflows via `VkBufferImageCopy` upload.

**Chapter(s):** Ch63 — KTX2, Basis Universal, and GPU Texture Compression; Ch64 — glTF 2.0 — The 3D Asset Pipeline Standard  
**Related:** **VkImage**, **VkBuffer**, **SPIR-V cooperative matrix (VK_KHR_cooperative_matrix)**, **VkImageView**

---

## L

### Level Zero (Intel)
Level Zero is Intel's low-level GPU compute and media API (`ze_*` prefix) that provides direct, explicit control over Intel GPU resources analogous to what Vulkan provides for graphics. It is the primary programming model for Intel's oneAPI stack and underlies the `OpenCL` and SYCL runtime implementations for Intel GPUs (`igc` compiler, Intel Compute Runtime). On Linux, Level Zero communicates with the kernel via the `i915` or `Xe` DRM driver's compute engine (`drm_i915_gem_execbuffer2` or Xe's native submission path); `libze_loader.so` dispatches to `libze_intel_gpu.so`. Intel XeSS (Ch71) uses Level Zero for its AI super-sampling inference pass. Level Zero differs from Vulkan compute in that it exposes sub-device partitioning, allowing each GPU tile on Intel Ponte Vecchio (PVC) to be addressed independently via `ZE_DEVICE_TYPE_GPU` sub-devices.

**Chapter(s):** Ch71 — Intel Xe and Arc; Ch62 — SYCL 2020  
**Related:** **GuC (Graphics Micro Controller, Intel)**, **HuC (Video Micro Controller, Intel)**, **XeSS (Intel Xe Super Sampling)**, **compute shader**

---

### libdisplay-info
A C library (`gitlab.freedesktop.org/emersion/libdisplay-info`) for parsing and interpreting EDID (Extended Display Identification Data) and DisplayID binary blobs retrieved from monitors. It provides structured access to monitor capabilities including supported timings, colour primaries, HDR static metadata (CEA-861.3 HDR Metadata block), and HDMI/DisplayPort vendor-specific extensions, without requiring callers to manually decode EDID byte offsets. Wayland compositors such as `wlroots`-based compositors and KWin use `libdisplay-info` to expose accurate display colour volume information for `wp_color_management_v1` image descriptions and to validate HDR capability before enabling HDR scanout modes. Not to be confused with `xf86-video-*` DDC/CI code paths, which are X11-specific.

**Chapter(s):** Ch74 — HDR and Wide Colour Gamut; Ch21 — Building Compositors with wlroots  
**Related:** **color-management-v1 (Wayland protocol)**, **MuraCorrection**, **wlroots**, **liftoff (KMS composition library)**

---

### liftoff (KMS composition library)
A C library (`gitlab.freedesktop.org/emersion/libliftoff`) that automatically assigns Wayland compositor layers to DRM/KMS hardware overlay planes using a best-fit algorithm, falling back to GPU compositing only for layers that no plane can accommodate. The compositor registers each layer with properties (buffer, format, zpos, alpha, transform) and calls `liftoff_output_apply()`, which attempts KMS atomic test commits to probe hardware plane compatibility, caching successful assignments. This removes the need for compositors to hand-code plane assignment heuristics. `Gamescope` and experimental wlroots backends use `libliftoff`. Common misconception: liftoff is not a GPU rendering library; it operates purely at the DRM plane assignment layer and calls no GL or Vulkan APIs.

**Chapter(s):** Ch21 — Building Compositors with wlroots; Ch22 — Production Compositors  
**Related:** **Gamescope**, **wlroots**, **zwp_linux_dmabuf_v1**, **wp_linux_drm_syncobj_surface_v1**

---

### linux-drm-syncobj-v1
A Wayland protocol extension (defined in the `wayland-protocols` repository as `linux-drm-syncobj-unstable-v1.xml`) that exposes DRM syncobj timeline points to Wayland compositor clients, enabling fully explicit synchronisation between the GPU driver and the compositor. A client imports a DRM syncobj fd into the compositor via `zwp_linux_drm_syncobj_manager_v1`, then attaches acquire and release timeline points to each `wl_buffer` submission. The compositor waits on the acquire point before scanning out the buffer and signals the release point when the display is done with it. This protocol directly solves the NVIDIA implicit-sync problem: NVIDIA's driver can signal the acquire syncobj point when rendering completes, without requiring `dma_resv` participation. Kernel backing is provided by `drivers/gpu/drm/drm_syncobj.c`.

**Chapter(s):** Ch3 — Advanced Display Features; Ch20 — Wayland Protocol Fundamentals; Ch46 — The Evolving Wayland Protocol Ecosystem  
**Related:** **drm_syncobj**, **explicit synchronisation**, **DMA fence**, **implicit synchronisation**, **timeline semaphore**

---

### LL-HLS (Low-Latency HLS)
An extension to standard HLS defined in the rfc8216bis draft (draft-pantos-hls-rfc8216bis, intended to supersede RFC 8216) and Apple's developer documentation that reduces glass-to-glass latency to 1–3 seconds by delivering sub-segment *partial segments* (chunks) before the full segment is complete. The M3U8 media playlist uses `#EXT-X-PART` tags to expose partial URIs as soon as each CMAF chunk is written; HTTP/2 server push and `_HLS_push` blocking playlist requests allow the player to receive chunks without polling. LL-HLS requires IDR-aligned chunk boundaries and CMAF packaging; it is incompatible with legacy MPEG-2 TS segments. FFmpeg's `hls` muxer supports LL-HLS via `-hls_flags low_latency`; GStreamer exposes it through `hlssink3`. LL-HLS complements but does not replace WebRTC or SRT for sub-second latency use cases.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **HLS (HTTP Live Streaming)**, **CMAF (Common Media Application Format)**, **M3U8 playlist**, **video segment**, **IDR frame**, **ABR (Adaptive Bitrate Streaming)**, **FFmpeg**, **GStreamer**

---

### LLVM (in Mesa context)
In Mesa, LLVM provides both a shader compiler backend and infrastructure utilities. The `src/amd/llvm/` directory contains the AMDGPU LLVM backend integration used by `radeonsi` (OpenGL) and optionally by RADV and ROCm; it compiles NIR (after lowering to LLVM IR) to GCN/RDNA machine code via the `amdgcn-amd-amdhsa` or `amdgcn-mesa-mesa3d` target. LLVM is also used by the `llvmpipe` software renderer (`src/gallium/drivers/llvmpipe/`) to JIT-compile shaders to CPU code. The critical distinction is that ACO is Mesa's preferred RADV backend specifically because it avoids LLVM's compile-time overhead; `radeonsi` still defaults to LLVM. Using LLVM for Mesa shaders requires matching LLVM version (≥15 as of Mesa 24), and Mesa's LLVM integration is isolated from any system LLVM used by other tools.

**Chapter(s):** Ch15 — ACO: AMD's Optimising Compiler; Ch17 — Software Renderers; Ch19 — OpenGL and Compatibility Drivers; Ch48 — ROCm and Machine Learning on Linux GPUs  
**Related:** **ACO**, **NIR**, **GCN ISA**, **RDNA ISA**, **Gallium3D**, **RADV**

---

### LMEM (Local Memory)
GPU-local VRAM (Video RAM) directly attached to the GPU die and accessed at full GPU memory bandwidth — as opposed to SMEM (system memory) accessed over PCIe. In discrete GPUs (AMD RDNA/CDNA, NVIDIA Ampere+, Intel Xe Arc), LMEM is the primary allocation target for framebuffers, textures, and GPU-local data structures for maximum performance. In the Linux kernel, i915/Xe drivers use the term "LMEM" explicitly in their TTM region definitions (`drivers/gpu/drm/i915/gem/i915_gem_lmem.c`); amdgpu uses `AMDGPU_GEM_DOMAIN_VRAM`; nouveau/NVK uses TTM's VRAM region. Mesa driver memory heaps labelled `DEVICE_LOCAL` in Vulkan correspond to LMEM. On integrated GPUs with unified memory architecture (UMA), LMEM does not exist as a separate pool; all memory is shared system RAM.

**Chapter(s):** Ch4 — GPU Memory Management; Ch5 — x86 GPU Drivers  
**Related:** **TTM (Translation Table Manager)**, **GEM (Graphics Execution Manager)**, **PRIME render offload**

---

### lookahead (encoder)
Lookahead in a video encoder is the practice of buffering and pre-analysing a queue of N future input frames before deciding the encoding parameters (quantizer, bitrate allocation, B-frame structure) for the current frame, enabling more accurate rate control and improved visual quality at the same bitrate. On GPU hardware encoders, lookahead is implemented in firmware: NVIDIA NVENC supports `enableLookahead = 1` with `lookaheadDepth` up to 32 frames in `NV_ENC_INITIALIZE_PARAMS`; AMD VCN exposes it through AMF via `AMF_VIDEO_ENCODER_HEVC_LTR_MODE` parameters; Intel QSV exposes it via `mfxExtCodingOption3.LookAheadDepth`. In FFmpeg, NVENC lookahead is enabled via `-rc:v vbr -lookahead_depth 32`. Lookahead increases encoder latency by up to N frames and GPU memory usage, making it unsuitable for low-latency streaming but beneficial for VOD encoding.

**Chapter(s):** Ch57 — FFmpeg Architecture; Ch59 — NVIDIA DeepStream SDK  
**Related:** **AMF (AMD Advanced Media Framework)**, **CUDA stream**, **NvFBC (NVIDIA Framebuffer Capture)**

---

## M

### M3U8 playlist
The UTF-8 playlist file format used by HLS, defined in RFC 8216, whose name derives from MP3 URL playlist (`.m3u`) with UTF-8 encoding. Two playlist types serve different roles: a *master playlist* lists available renditions (`#EXT-X-STREAM-INF` tags with BANDWIDTH, RESOLUTION, CODECS attributes) and the URLs of their respective *media playlists*. A *media playlist* lists individual segment URLs with durations (`#EXTINF`), optional byte-range specifications, and, for LL-HLS, partial segment (`#EXT-X-PART`) and rendition-report tags. `#EXT-X-KEY` carries AES-128 or sample-AES DRM key URI information. Conforming M3U8 playlists must not contain Windows-style CRLF line endings; a common server misconfiguration that causes player failures. FFmpeg generates M3U8 files via the `hls` muxer.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery; Ch57 — FFmpeg Architecture and Programming  
**Related:** **HLS (HTTP Live Streaming)**, **LL-HLS (Low-Latency HLS)**, **ABR (Adaptive Bitrate Streaming)**, **video segment**, **CMAF (Common Media Application Format)**, **MPD (MPEG-DASH Media Presentation Description)**

---

### macroblock
The fundamental coding unit in H.264/AVC and earlier MPEG codecs: a 16×16 block of luma samples paired with two 8×8 chroma blocks (in 4:2:0 format). A macroblock may be coded as intra (I-MB), predicted (P-MB), or bidirectionally predicted (B-MB), and may be further partitioned into 8×8, 8×4, 4×8, or 4×4 sub-macroblocks for motion compensation. Macroblocks within a slice are processed in raster-scan order, enabling row-level parallelism via Wavefront Parallel Processing in H.265. H.265/HEVC replaces the macroblock with the Coding Tree Unit (CTU) hierarchy, and AV1 uses superblocks; neither calls their unit a "macroblock." Using "macroblock" as a generic synonym for coding unit across all codecs is a common inaccuracy.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **intra-prediction**, **inter-prediction**, **DCT (Discrete Cosine Transform)**, **motion vector**, **entropy coding (CABAC/CAVLC)**

---

### MangoHUD
An open-source Vulkan and OpenGL overlay library (`github.com/flightlessmango/MangoHud`) that displays real-time GPU/CPU performance metrics — frame time, FPS, GPU temperature, VRAM usage, CPU frequency — as an in-game HUD. MangoHUD injects itself as a Vulkan layer (registered via `VkLayerProperties` in `/usr/share/vulkan/implicit_layer.d/`) or as an OpenGL wrapper (`LD_PRELOAD`), intercepting present calls to measure frame timing. It reads GPU performance counters from DRM sysfs (`/sys/class/drm/card0/device/gpu_busy_percent`) and vendor-specific interfaces. It is commonly used with `MANGOHUD=1` environment variable and is integrated with Gamescope and Steam's performance overlay on Steam Deck.

**Chapter(s):** Ch29 — Upscaling, Effects and Overlays; Ch54 — Linux Input Stack  
**Related:** **Gamescope**, **zwp_linux_dmabuf_v1**, **liftoff (KMS composition library)**

---

### MaxCLL / MaxFALL
Two static HDR10 content light level metrics defined in CTA-861.3. **MaxCLL** (Maximum Content Light Level) is the highest luminance in nits of any single pixel in the entire video stream. **MaxFALL** (Maximum Frame-Average Light Level) is the highest average luminance across all pixels in any single frame of the stream. Both values are encoded as unsigned 16-bit integers in the CTA HDR Static Metadata Descriptor Type 1 and are conveyed in the HDMI HDR InfoFrame or DP HDR SDP. In the Linux kernel they appear as `max_cll` and `max_fall` fields inside `struct hdr_metadata_infoframe` nested within `struct hdr_output_metadata` in `include/uapi/drm/drm_mode.h`. Tone-mapping algorithms in compositors and displays use MaxCLL and MaxFALL to scale HDR content to the display's actual peak luminance capability without clipping.

**Chapter(s):** Ch3 — Advanced Display Features; Ch26 — Hardware Video  
**Related:** **HDR10**, **HDR metadata**, **PQ (Perceptual Quantizer / ST 2084)**, **connector (DRM)**, **BT.2020**

---

### MDL (NVIDIA Material Definition Language)
NVIDIA MDL is a domain-specific language for describing physically based surface and volume materials in a vendor-neutral, mathematically precise way, allowing the same material to render identically across multiple rendering backends (RTX Renderer, OptiX, iray, Chaos V-Ray). MDL describes materials as a composition of BSDFs (e.g., `df::diffuse_reflection_bsdf`, `df::microfacet_ggx_vcavities_bsdf`) combined via mixing and layering operators; the compiled form is a `.mdlm` shared library. The MDL SDK (`libmdl_sdk.so`, `IMdl_compiler`, `IMdl_backend`) parses, validates, and compiles MDL to PTX or GLSL for GPU execution. In Omniverse, MDL materials are embedded in USD via the `mdl_material` schema (Ch69) and the `OmniSurface` and `OmniHair` MDL material graphs.

**Chapter(s):** Ch69 — NVIDIA Omniverse, OpenUSD, and the RTX Renderer  
**Related:** **OptiX**, **Omniverse (NVIDIA)**, **OpenUSD / USD**, **PBR (Physically Based Rendering)**

---

### Mesa
Mesa is the open-source implementation of OpenGL, OpenGL ES, Vulkan, OpenCL, and related GPU APIs on Linux (and other platforms), hosted at `https://gitlab.freedesktop.org/mesa/mesa`. It consists of API state trackers (OpenGL, EGL, GLX, Vulkan ICDs), the Gallium3D pipe driver architecture, the NIR shader IR, and a collection of GPU-specific driver backends covering AMD, Intel, NVIDIA (Nouveau), Qualcomm, ARM Mali, Apple AGX, and others. Mesa does not include kernel GPU drivers — those live in the Linux kernel's `drivers/gpu/drm/` tree; Mesa is the user-space side that communicates with the kernel via DRM render node ioctls. A common misconception is that Mesa equals OpenGL; Mesa also provides the primary Vulkan implementation used by most Linux GPU drivers.

**Chapter(s):** Ch12 — The Mesa Loader and Driver Dispatch; Ch13 — Gallium3D: The OpenGL State Tracker; Ch14 — NIR: Mesa's Shader Intermediate Representation  
**Related:** **Gallium3D**, **NIR**, **RADV**, **ANV**, **disk_cache (Mesa)**, **ACO**

---

### mesh shader (hardware perspective)
A mesh shader is a programmable GPU pipeline stage that replaces the fixed-function vertex/tessellation/geometry pipeline with two new compute-like stages: the Task shader (optionally culls and generates mesh shader work groups) and the Mesh shader (generates indexed primitives directly in a thread-group, outputting vertices and primitive indices into a shared mesh output). On AMD RDNA2+ hardware, mesh shaders execute on the same CU array as compute shaders and are submitted via `VK_EXT_mesh_shader` draw commands (`vkCmdDrawMeshTasksEXT`); on NVIDIA Turing+, they use dedicated warp-level mesh output buffers. In Mesa, RADV implements `VK_EXT_mesh_shader` with ACO backend support (`src/amd/vulkan/radv_cmd_buffer.c`); ANV implements it for Gen12.5+ (`src/intel/vulkan/anv_cmd_buffer.c`). Mesh shaders are used for GPU-driven rendering where the CPU never touches per-draw primitive counts.

**Chapter(s):** Ch18 — Vulkan Drivers; Ch56 — Ray Tracing on Linux  
**Related:** **compute shader**, **compute unit (CU)**, **warp (NVIDIA thread group)**, **wavefront (AMD thread group)**

---

### mode (display mode)
A `struct drm_display_mode` (defined in `include/drm/drm_modes.h`) that fully describes a display timing: pixel clock in kHz, horizontal and vertical active resolution, front porch, sync pulse width, back porch, sync polarity, interlace flag, and computed refresh rate. Modes are enumerated by parsing EDID (via `drm_edid_to_modes()` in `drivers/gpu/drm/drm_edid.c`) and added to a connector's mode list. The compositor or display manager selects a mode and sets it via KMS; in atomic modesetting the chosen mode is encoded as a blob property on the CRTC (`MODE_ID`). Modes also carry flags for reduced blanking (CVT-RB), double-scan, and stereo. The term "display mode" should not be confused with "modesetting" (the act of configuring the pipeline) or with a render mode (windowed vs. fullscreen) at the application layer.

**Chapter(s):** Ch2 — KMS: The Display Pipeline  
**Related:** **EDID**, **CRTC**, **connector (DRM)**, **modesetting**, **atomic modesetting**, **VRR (Variable Refresh Rate)**

---

### modesetting
The act of configuring the KMS display pipeline — selecting a display mode, connecting CRTC/encoder/connector chains, allocating a framebuffer, and initiating scanout — as opposed to the KMS subsystem itself. Historically done by the X server or BIOS (userspace mode setting, UMS); kernel mode setting (KMS) moved this into the kernel. The X.org modesetting DDX (`xf86-video-modesetting`) is a KMS-based X driver that performs modesetting generically via libDRM rather than hardware-specific code. The DRM modesetting API provides both a legacy path (`DRM_IOCTL_MODE_SETCRTC`) and the preferred atomic path (`DRM_IOCTL_MODE_ATOMIC`). In common usage, "modesetting" as an adjective often refers to the generic DRM modesetting driver or to the `DRM_CAP_DUMB_BUFFER` path used for simple framebuffer output.

**Chapter(s):** Ch1 — DRM Architecture & the Driver Model; Ch2 — KMS: The Display Pipeline  
**Related:** **KMS (Kernel Mode Setting)**, **atomic modesetting**, **mode (display mode)**, **CRTC**, **framebuffer (DRM)**

---

### Monado (OpenXR runtime)
The open-source reference implementation of the OpenXR 1.x runtime, developed at Collabora under the freedesktop.org umbrella ([https://gitlab.freedesktop.org/monado/monado](https://gitlab.freedesktop.org/monado/monado)). Monado provides the `XrInstance`, `XrSession`, and `XrSwapchain` objects that OpenXR applications call; it drives HMD displays via DRM direct mode (bypassing the desktop compositor), allocates swapchain images via GBM or Vulkan, and integrates SLAM-based tracking from V4L2 cameras. The IPC layer allows multiple OpenXR clients to share a single Monado service process. Monado is not the only Linux OpenXR runtime — SteamVR also ships an OpenXR runtime — but is the primary open-source reference.

**Chapter(s):** Ch27 — VR & AR  
**Related:** **OpenXR**, **VkSwapchainKHR**, **EGL (Embedded/Extensible Graphics Library)**, **VA-API**

---

### MOQT (Media over QUIC Transport)
An IETF-standardised (draft-ietf-moq-transport) transport protocol that delivers live and on-demand media over QUIC streams, designed to achieve sub-second latency at scale without the head-of-line blocking problems of HTTP/1.1-based ABR. MOQT defines a publish/subscribe model: producers publish *objects* organised into *groups* on *tracks*; subscribers receive objects in application-defined priority order. Unlike WebRTC (peer-to-peer) or HLS/DASH (HTTP pull), MOQT supports relay-tree fan-out for millions of simultaneous viewers. Twitch's WISH/WarpDrive and Meta's Whip-over-QUIC are precursor deployments. MOQT is transport-only; codec framing (e.g. LOC for H.264/HEVC/AV1) is defined in companion specifications. Linux integration is via user-space QUIC libraries (msquic, quic-go, aioquic).

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **ABR (Adaptive Bitrate Streaming)**, **WebRTC**, **SRT (Secure Reliable Transport)**, **HLS (HTTP Live Streaming)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **video segment**

---

### motion estimation
The encoder-side algorithm that, for each block or macroblock in the current frame, searches a reference frame for the region that most closely matches, producing a motion vector and a residual difference signal. Common search strategies include full-search (exhaustive but costly), diamond search, hexagonal search, and hierarchical/multi-resolution search. The quality metric used during search is typically Sum of Absolute Differences (SAD) or Sum of Squared Differences (SSD) for speed, and Rate-Distortion Optimisation (RDO) for quality. Motion estimation is purely an encoder concern; the bitstream carries only motion vectors and residuals, not the intermediate search results. Hardware encoders (NVENC, AMD AMF, Intel Quick Sync) implement motion estimation in fixed-function silicon, operating at high speed but with fewer search candidates than software encoders.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch57 — FFmpeg Architecture and Programming  
**Related:** **motion vector**, **inter-prediction**, **macroblock**, **rate-distortion optimization**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **NVENC**, **B-frame**, **P-frame**

---

### motion vector
A pair of horizontal and vertical displacement values that describes the offset from a block in the current frame to its best-matching predictor region in a reference frame, as determined by motion estimation at encode time. In H.264 motion vectors are coded in units of quarter-luma-sample precision (quarter-pel); in H.265, quarter-pel with optional 1/8-pel for certain modes; in AV1, eighth-pel. Motion vectors are entropy-coded differentially relative to a predicted MV from neighbouring blocks (MVP: Motion Vector Prediction). In AV1 the WARPMV compound mode applies an affine warp transformation rather than translational motion vectors. On the decoder side, motion vectors direct reference frame sample copying in the motion compensation step. Large motion vectors indicate fast camera or object motion.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **motion estimation**, **inter-prediction**, **macroblock**, **B-frame**, **P-frame**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**

---

### MPD (MPEG-DASH Media Presentation Description)
The XML manifest file defined in ISO/IEC 23009-1 that describes all available representations (video, audio, subtitle tracks) for a MPEG-DASH stream. An MPD is structured as `MPD` → `Period` → `AdaptationSet` → `Representation` → `SegmentTemplate` / `SegmentBase` / `SegmentList`. The `SegmentTemplate` element uses `$Number$` or `$Time$` template variables to construct segment URLs without enumerating each segment individually, enabling live streams. The `Representation` element carries codec (`codecs` attribute using RFC 6381 MIME type strings), bandwidth, width, and height. DASH clients parse the MPD to build a rendition list and feed it to their ABR algorithm. The MPD's `availabilityStartTime` and `timeShiftBufferDepth` attributes define DVR window semantics for live streams.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **DASH (Dynamic Adaptive Streaming over HTTP)**, **ABR (Adaptive Bitrate Streaming)**, **CMAF (Common Media Application Format)**, **video segment**, **M3U8 playlist**, **BOLA (Buffer Occupancy–based Lyapunov Algorithm)**

---

### MST (Multi-Stream Transport)
A DisplayPort 1.2+ capability allowing a single DP output to drive multiple independent video streams by multiplexing them on the link using time-division slots. A daisy-chain or hub of MST branch devices splits the streams to individual downstream monitors. In the Linux kernel, MST is implemented in `drivers/gpu/drm/display/drm_dp_mst_topology.c`, which builds a topology graph of branch and sink nodes discovered via DPCD reads on the AUX channel. Each downstream monitor appears as a separate `drm_connector` in KMS, with its own mode list; a KMS "virtual CRTC" is assigned to each. MST complicates dynamic CRTC allocation since the total available link bandwidth must be shared across all streams. HPD events on an MST hub trigger full topology re-enumeration rather than a simple single-connector re-probe.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **DisplayPort**, **connector (DRM)**, **CRTC**, **hot-plug detect (HPD)**, **DSC (Display Stream Compression)**

---

### MuraCorrection
A display uniformity correction technique that compensates for per-panel brightness and colour non-uniformity ("mura" — Japanese for "blotchiness") arising from manufacturing variation in OLED and LCD panels. The compositor or display driver stores a per-pixel correction LUT derived from factory calibration measurements and applies it as a post-processing pass — typically as an additional KMS `GAMMA_LUT` or a custom shader pass — before scanout. On Linux, KWin (Plasma 6.x) exposes mura correction as a KMS blob property applied via atomic commit; `libdisplay-info` assists in reading panel-specific calibration metadata from EDID vendor extensions. Not to be confused with gamma correction or colour management: mura correction addresses spatial uniformity, while colour management addresses colorimetric accuracy.

**Chapter(s):** Ch74 — HDR and Wide Colour Gamut; Ch53 — Display Calibration and colord  
**Related:** **color-management-v1 (Wayland protocol)**, **libdisplay-info**, **wl_compositor**

---

## N

### naga (Rust shader translator)
Naga is a pure-Rust shader translation library (`https://github.com/gfx-rs/wgpu/tree/trunk/naga`, part of the `wgpu` workspace) that parses WGSL, SPIR-V, and GLSL and emits SPIR-V, GLSL, HLSL, MSL, and WGSL. In the `wgpu` GPU abstraction (used by Firefox WebGPU and the Bevy game engine), naga serves as the counterpart to Tint in Chrome's Dawn — both translate WGSL to SPIR-V for submission to Mesa Vulkan drivers. Naga operates at the IR level on its own `Module`/`Function`/`Expression` representation, distinct from NIR or SPIR-V. Unlike Tint, naga is not a reference implementation; it is a production component in Gecko's WebGPU and in Bevy's render pipeline.

**Chapter(s):** Ch40 — Bevy and wgpu — A Rust-Native Vulkan Client; Ch52 — Firefox and WebRender; Ch35 — Dawn and WebGPU  
**Related:** **WGSL**, **SPIR-V**, **wgpu (Rust WebGPU/Vulkan)**, **Tint (WGSL compiler)**, **NIR**, **spirv-cross**

---

### NAL unit (H.264/H.265)
A Network Abstraction Layer unit: the fundamental packaging element of H.264/AVC (ITU-T H.264 Annex B) and H.265/HEVC bitstreams. Each NAL unit carries a 1-byte (H.264) or 2-byte (H.265) header specifying the `nal_unit_type` followed by the RBSP (Raw Byte Sequence Payload). In H.264, key types include: SPS (Sequence Parameter Set, type 7), PPS (Picture Parameter Set, type 8), IDR slice (type 5), non-IDR slice (type 1), and SEI (type 6). In H.265, SPS/PPS/VPS occupy types 33/34/32 and SEI types 39/40. In byte-stream format (Annex B) NAL units are delimited by start codes (`0x00 0x00 0x01`); in AVCC/HVCC format (used in MP4 containers) a length prefix replaces the start code. FFmpeg's `libavcodec` transparently handles both formats via the `h264_mp4toannexb` and `hevc_mp4toannexb` bitstream filters.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **H.264 / AVC**, **H.265 / HEVC**, **IDR frame**, **SEI (Supplemental Enhancement Information, H.264)**, **entropy coding (CABAC/CAVLC)**, **FFmpeg**

---

### Neural Radiance Cache (NRC)
The Neural Radiance Cache is an online-trained neural network GI technique (Müller et al., SIGGRAPH 2021) that learns the radiance field of the current scene during rendering, compressing irradiance into a multi-resolution hash encoding followed by a small MLP (4 hidden layers, 64 neurons, ReLU). At each frame, unresolved path suffixes (rays that would require additional bounces) are terminated and the query is routed to the NRC network, which predicts the accumulated radiance. Training uses the short-path radiance estimates that hit lights directly, forming a self-supervised online loss. In the RTXGI SDK, NRC is implemented as a CUDA/Vulkan hybrid: `rtxgi::NRC::Context`, `SharcGetCachedRadiance`-equivalent queries dispatched via `vkCmdTraceRaysKHR`, and Adam optimiser weight updates on each frame via CUDA.

**Chapter(s):** Ch70 — RTX Kit; Ch67 — OptiX 9  
**Related:** **RTXGI**, **SHaRC**, **GI (Global Illumination)**, **ReSTIR / RTXDI**, **CUDA**

---

### NIR (New Intermediate Representation)
NIR is Mesa's canonical in-memory shader intermediate representation, defined in `src/compiler/nir/`, used across all Mesa GPU drivers and API front ends as the common compilation currency. Shaders arriving as GLSL, SPIR-V, or HLSL are all lowered to NIR early in the Mesa pipeline; NIR is then optimised (dead code elimination, constant folding, algebraic simplifications via `nir_opt_*` passes) before being lowered to hardware ISA by driver backends such as ACO, BRW, or the LLVM AMDGPU backend. NIR represents programs as SSA-form functions with typed variables, intrinsics for I/O, and explicit phi nodes. The key architectural decision is that NIR is the single handoff point between API state trackers and GPU-specific code, enabling optimisation passes to benefit all drivers simultaneously.

**Chapter(s):** Ch14 — NIR: Mesa's Shader Intermediate Representation; Ch15 — ACO: AMD's Optimising Compiler; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **ACO**, **SPIR-V**, **GLSL**, **Gallium3D**, **BRW compiler (Intel EU ISA)**, **LLVM (in Mesa context)**

---

### NRD (NVIDIA Real-time Denoisers)
NRD is NVIDIA's open-source (MIT, `github.com/NVIDIAGameWorks/RayTracingDenoiser`) Vulkan-based denoising SDK that implements three production-grade spatiotemporal denoising algorithms: REBLUR, RELAX, and SIGMA. NRD accepts G-buffer inputs (`IN_MV` motion vectors, `IN_NORMAL_ROUGHNESS` Oct-packed, `IN_VIEWZ` linear depth, `IN_DIFF_RADIANCE_HITDIST`, `IN_SPEC_RADIANCE_HITDIST`) plus noisy radiance, and outputs filtered radiance suitable for temporal accumulation; textures must have pre-divided albedo. The `NrdIntegration` helper class (`BeginFrame`, `SetCommonSettings`, `SetDenoiserSettings`, `Denoise`) manages the multi-pass Vulkan dispatch (pre-pass, history fix, blur, post-blur, split-screen). NRD feeds into DLSS Super Resolution and is an alternative to the heavier Ray Reconstruction approach.

**Chapter(s):** Ch70 — RTX Kit; Ch68 — DLSS 4  
**Related:** **REBLUR**, **RELAX**, **SIGMA (NRD denoiser)**, **DLSS 4**, **ReSTIR / RTXDI**

---

### NVDEC
NVIDIA's fixed-function hardware video decode engine, present on Maxwell-generation and later discrete GPUs and Jetson SoCs. NVDEC is distinct from CUDA cores and shader processors; it is a dedicated silicon block that runs independently of the 3D/compute pipeline. Supported codecs vary by GPU generation: Pascal adds VP9; Ampere adds AV1 decode (Turing does not support AV1 decode). On Linux, NVDEC is accessed via two paths: the `nvdec` and `cuvid` hwaccel backends in FFmpeg (e.g. `h264_cuvid`, `hevc_cuvid` decoder names) using NVIDIA's Video Codec SDK; and via the VDPAU compatibility layer (legacy). NVDEC does not expose a V4L2 or VA-API interface natively; Mesa and the open driver ecosystem use VA-API on AMD/Intel instead. NVDEC output is CUDA-accessible device memory, enabling zero-copy pipelines to CUDA inference (DeepStream).

**Chapter(s):** Ch66 — CUDA Runtime, Streams, and NVRTC; Ch57 — FFmpeg Architecture and Programming; Ch59 — NVIDIA DeepStream SDK  
**Related:** **NVENC**, **FFmpeg**, **DeepStream (NVIDIA)**, **codec**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **V4L2 (Video4Linux2)**

---

### NVENC
NVIDIA's fixed-function hardware video encode engine, present on Kepler-generation and later discrete GPUs and Jetson SoCs. Like NVDEC, NVENC is dedicated silicon separate from CUDA shader cores. On Linux, NVENC is accessed via NVIDIA's Video Codec SDK (`nvEncodeAPI.h`) and exposed through FFmpeg as hardware encoder names: `h264_nvenc`, `hevc_nvenc`, `av1_nvenc` (Ada Lovelace+). GStreamer accesses NVENC via the `nvh264enc`/`nvhevcenc` elements in `gst-plugins-bad`. Rate control modes supported include CBR, VBR, CQP, and (on newer GPUs) a lookahead mode that buffers frames for better rate decisions. A common misconception: NVENC quality is lower than software x264 at equivalent bitrates due to fewer motion estimation candidates; this gap has narrowed significantly on Turing/Ampere hardware.

**Chapter(s):** Ch66 — CUDA Runtime, Streams, and NVRTC; Ch57 — FFmpeg Architecture and Programming; Ch59 — NVIDIA DeepStream SDK  
**Related:** **NVDEC**, **FFmpeg**, **GStreamer**, **codec**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **rate-distortion optimization**, **CRF (Constant Rate Factor)**

---

### NvFBC (NVIDIA Framebuffer Capture)
NvFBC (NVIDIA Framebuffer Capture) is a proprietary NVIDIA API for high-performance, low-latency capture of the GPU's framebuffer output — specifically designed for game streaming and remote desktop applications. NvFBC (`libnvidia-fbc.so`, `NvFBCCreateInstance`, `NVFBC_SESSION_HANDLE`) operates at the display engine level, capturing the composed final image after all overlays are merged, avoiding the high CPU cost of X11 `XShmGetImage` or DMA readbacks. On Linux with Wayland, NvFBC requires the proprietary NVIDIA driver and captures via an NVFBC-aware compositor path; Mesa and Wayland-native compositors do not support NvFBC. Sunshine, the open-source game streaming server, uses NvFBC as its preferred NVIDIA capture path on Linux.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute  
**Related:** **Sunshine (game streaming)**, **CUDA stream**, **NVRTC**

---

### NVIR / PTX (NVIDIA Parallel Thread Execution)
PTX (Parallel Thread Execution) is NVIDIA's virtual ISA and intermediate representation for GPU compute programs, defined by NVIDIA in the PTX ISA specification. PTX is the output of the CUDA compiler (`nvcc`/`nvrtc`) and is JIT-compiled at runtime by the NVIDIA driver to SASS (the native hardware ISA). PTX uses a RISC-like, infinite-register, predicated execution model with explicit memory space qualifiers (`global`, `shared`, `local`). In the open-source context, PTX does not directly enter Mesa — NVK (the open Vulkan driver) receives SPIR-V and compiles it via the `nvk` compiler to NVIDIA's native ISA using reverse-engineered knowledge; PTX is relevant to CUDA workloads handled by the NVIDIA proprietary driver. "NVIR" is sometimes used informally for NVIDIA's intermediate representations but is not an official term.

**Chapter(s):** Ch25 — GPU Compute; Ch66 — CUDA Runtime, Streams, and NVRTC; Ch10b — NVK: Building a Vulkan Driver from Scratch  
**Related:** **SASS (NVIDIA ISA)**, **ISA (Instruction Set Architecture)**, **SPIR-V**, **NIR**

---

### NVRTC
NVRTC (NVIDIA Runtime Compilation) is a CUDA library (`libnvrtc.so`) that compiles CUDA C++ source code to PTX at application runtime, enabling dynamic kernel generation without a full CUDA Toolkit installation. The core API sequence is `nvrtcCreateProgram` (wrap source string), `nvrtcAddNameExpression` (preserve C++ symbol mangling), `nvrtcCompileProgram` (compile with options including `-arch=compute_90`), `nvrtcGetPTX` (retrieve PTX string) → `cuModuleLoadData` + `cuKernelGetFunction` (load and bind). NVRTC 13.x (CUDA 13.x) adds C++23 support; header-only CUDA C++ headers (`libcudacxx`) allow deployment without a full toolkit. NVRTC is the compilation backbone for OptiX 9 shader pipelines (`optixModuleCreate` accepts PTX) and for inline RTXNS neural shader inference.

**Chapter(s):** Ch66 — CUDA Runtime, Streams, and NVRTC; Ch67 — OptiX 9  
**Related:** **CUDA**, **CUDA stream**, **OptiX**, **SASS (NVIDIA shader assembly)**, **RTXNS (Neural Shaders)**

---

## O

### Omniverse (NVIDIA)
NVIDIA Omniverse is a real-time 3D collaboration and simulation platform built on OpenUSD as the universal scene description format and on the NVIDIA RTX Renderer as the default rendering engine. On Linux, Omniverse runs as a headless Kit SDK process (`kit-kernel`, extension-based Python+C++ framework) or as an interactive application; the RTX Renderer (`omni.rtx.renderer` extension) supports RTX Real-Time 2.0 mode (interactive, DLSS-accelerated) and RTX Interactive (progressive path tracing). Key Kit API entry points: `omni.kit.app` event loop, `UsdContext::Open` (load scene), `omni.usd.stage` USD stage management, `omni.rtx.settings.core` renderer configuration. NVIDIA recommends Ubuntu 22.04/24.04 with Ada Lovelace or newer GPU and driver ≥ 535.

**Chapter(s):** Ch69 — NVIDIA Omniverse, OpenUSD, and the RTX Renderer  
**Related:** **OpenUSD / USD**, **MDL (NVIDIA Material Definition Language)**, **DLSS 4**, **OptiX**, **PhysX 5**, **WARP (NVIDIA Python GPU kernel)**

---

### OpenCL
An open, Khronos-defined API and C-based kernel language for heterogeneous parallel compute across CPUs, GPUs, DSPs, and FPGAs. The host API (`cl_platform_id`, `cl_device_id`, `cl_context`, `cl_command_queue`, `cl_kernel`) abstracts device discovery and command submission; OpenCL C kernels compile to an intermediate representation (SPIR-V since OpenCL 2.1) and then to device ISA. On Linux, `rusticl` (Mesa) provides an OpenCL 3.0 ICD over Gallium3D pipe drivers (same hardware paths as radeonsi/iris), and Intel's `intel-compute-runtime` provides OpenCL over Level Zero. ROCm's `ROCm-OpenCL-Runtime` targets AMD compute hardware. A common misconception is that OpenCL is deprecated — OpenCL 3.0 (2020) is actively maintained, though SYCL and HIP are favored for new code.

**Chapter(s):** Ch25 — GPU Compute; Ch62 — SYCL 2020 and Portable Heterogeneous Compute  
**Related:** **SYCL 2020**, **VkPipeline**, **SPIR-V cooperative matrix (VK_KHR_cooperative_matrix)**, **VMA (Vulkan Memory Allocator)**

---

### OpenGL
The long-standing Khronos cross-platform API for 2D and 3D rendering, covering versions 1.0 (1992) through 4.6 (2017, which added SPIR-V shader ingestion). OpenGL is a stateful API with a global context binding, fixed-function compatibility features in the compatibility profile, and implicit driver-managed synchronization. On Linux, OpenGL contexts are created via GLX (X11) or EGL (Wayland/GBM). Mesa implements the OpenGL core and compatibility profiles via the Gallium3D state tracker (`src/mesa/state_tracker/`); NIR is Mesa's internal shader IR. OpenGL is no longer receiving new major features — Vulkan is Khronos's low-overhead successor — but OpenGL 4.6 remains widely used, and the compatibility profile is required for legacy and Windows-ported applications.

**Chapter(s):** Ch13 — Gallium3D: The OpenGL State Tracker; Ch19 — OpenGL and Compatibility Drivers; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **GLES (OpenGL ES)**, **EGL (Embedded/Extensible Graphics Library)**, **VkPipeline**, **SPIR-V cooperative matrix (VK_KHR_cooperative_matrix)**

---

### OpenUSD / USD
OpenUSD (Universal Scene Description) is an open-source (Apache 2.0) 3D scene description and interchange framework originally developed by Pixar and now standardised under the Alliance for OpenUSD (AOUSD Core Specification 1.0, December 2025, Linux Foundation). USD represents scenes as a hierarchy of `UsdPrim` objects on a `UsdStage` with typed schemas (`UsdGeomMesh`, `UsdLuxSphereLight`, `UsdShadeMaterial`, `UsdPhysicsRigidBodyAPI`) composed from multiple layers via the arc system (references, inherits, specialises, payload arcs, variantSets). The C++ USD SDK (25.02+) exposes `UsdStage::Open`, `UsdPrim::GetAttribute`, and `UsdRelationship` for programmatic scene authoring; the Python bindings (`pxr.Usd`, `pxr.UsdGeom`) are widely used in DCCs. On Linux, USD is foundational to NVIDIA Omniverse, Blender's USD import/export addon, and Hydra-based rendering pipelines.

**Chapter(s):** Ch69 — NVIDIA Omniverse, OpenUSD, and the RTX Renderer; Ch64 — glTF 2.0  
**Related:** **Omniverse (NVIDIA)**, **MDL (NVIDIA Material Definition Language)**, **PhysX 5**, **glTF 2.0**

---

### OpenVX
A Khronos standard for accelerated computer vision workloads, defining a graph-based API (`vx_graph`, `vx_node`, `vx_image`, `vx_tensor`) that enables offline compilation of vision pipelines for heterogeneous hardware. OpenVX 1.3 added neural network tensor operations. On Linux, implementations include AMD's ROCm OpenVX and NVIDIA's VisionWorks (legacy). OpenVX is distinct from OpenCL and Vulkan compute: its graph model allows drivers to perform cross-node optimizations such as buffer fusion and layout selection. In the Linux graphics stack context it appears primarily in embedded and automotive vision pipelines; desktop GPU workloads have largely migrated to Vulkan compute or CUDA/ROCm.

**Chapter(s):** Ch65 — Vulkan Safety Critical and OpenVX  
**Related:** **OpenCL**, **SYCL 2020**, **VkPipeline**

---

### OpenXR
The Khronos cross-vendor API standard for VR and AR (XR) applications, defining runtime-agnostic interfaces for session management (`xrCreateSession`), swapchain image acquisition (`xrAcquireSwapchainImage`), space and pose tracking (`XrSpace`, `XrPosef`), action input, and compositor layer submission (`xrEndFrame` with `XrCompositionLayerProjection`). On Linux, OpenXR runtimes include Monado (open-source reference) and SteamVR. The OpenXR Vulkan graphics binding (`XR_KHR_vulkan_enable2`) lets applications submit rendered `VkImage` swapchain images directly to the runtime compositor. OpenXR replaces the fragmented OculusVR SDK / OpenVR / OSVR APIs with a single portable interface.

**Chapter(s):** Ch27 — VR & AR  
**Related:** **Monado (OpenXR runtime)**, **VkSwapchainKHR**, **VkImage**, **VkSemaphore**

---

### OptiX
OptiX is NVIDIA's GPU-accelerated ray tracing SDK built on top of CUDA that provides a programmable ray tracing pipeline — ray generation, intersection, any-hit, closest-hit, miss, and callable shader stages — with hardware BVH traversal accelerated by RT Cores. The core API types are `OptixDeviceContext` (wraps a `CUcontext`), `OptixAccelStructure` (BVH handle), `OptixPipeline` (linked shader program), and `OptixShaderBindingTable` (SBT, maps ray types to shader records). As of OptiX 9.1 (January 2026, requiring R590+), new features include Cooperative Vectors for per-shader neural inference (`optixCoopVecMatMul`) and the Clusters/Megageometry API for massive dynamic BVH construction. OptiX shaders compile from CUDA C++ via NVRTC → PTX → `optixModuleCreate`; they share CUDA device memory with Vulkan via `VK_KHR_external_memory_fd`.

**Chapter(s):** Ch67 — OptiX 9 — NVIDIA's Ray Tracing Framework  
**Related:** **BVH**, **ray tracing hardware (RT Core / XMX)**, **CUDA**, **NVRTC**, **NRD (NVIDIA Real-time Denoisers)**

---

## P

### P-frame
A predictively coded video frame that is compressed by referencing one or more previously decoded frames in display order; it carries motion vectors and DCT-coded residuals for macroblocks or coding units that differ from their reference. P-frames require less data than I-frames because they only encode differences, but more than B-frames because they cannot reference future frames. In H.264, P-slices allow each macroblock to reference any single frame from the reference picture list L0. In H.265, a P-frame is effectively a B-frame with only L0 references active. In AV1 terminology, LAST_FRAME/GOLDEN_FRAME/ALTREF_FRAME reference frames correspond conceptually to P-frame and B-frame references. P-frames between I-frames define GOP structure alongside any B-frames, directly affecting seek granularity and ABR segment size.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **I-frame**, **B-frame**, **IDR frame**, **inter-prediction**, **motion vector**, **macroblock**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**

---

### page flip
An atomic KMS operation that updates the framebuffer displayed by a CRTC on the next vblank interval, implemented via `DRM_IOCTL_MODE_PAGE_FLIP` (legacy) or an atomic commit updating the primary plane's `FB_ID` property (preferred). The driver schedules the buffer pointer update to coincide with the start of the vertical blanking period so the switch is invisible to the viewer, avoiding tearing. On completion, the kernel sends a `DRM_EVENT_FLIP_COMPLETE` event (polled via `drmHandleEvent()`) to notify the compositor that the previous buffer is no longer displayed and can be reused or released. An in-flight fence on the new framebuffer's underlying DMA-BUF (implicit sync) or a syncobj acquire point (explicit sync) delays the flip until GPU rendering completes.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **CRTC**, **framebuffer (DRM)**, **vblank**, **scan-out**, **implicit synchronisation**, **explicit synchronisation**, **DMA fence**

---

### PBR (Physically Based Rendering)
Physically Based Rendering is a rendering methodology that models light-matter interaction using energy-conserving BRDFs grounded in physical optics, enabling consistent appearance of materials across lighting conditions. The two dominant parametric PBR workflows are metallic-roughness (Disney/Unreal, adopted by glTF 2.0 and Filament) and specular-glossiness (PBR-SpecGloss extension); both drive Cook-Torrance microfacet models with GGX distribution, Schlick Fresnel, and Smith geometry terms. On the GPU, a PBR fragment shader samples albedo/metallic/roughness texture maps, evaluates the BRDF, and adds indirect lighting via IBL (pre-filtered envmap + BRDF LUT) or GI probes. A common misconception is that PBR requires ray tracing — rasterisation-based PBR with IBL is standard in real-time engines.

**Chapter(s):** Ch83 — Filament; Ch64 — glTF 2.0; Ch56 — Ray Tracing on Linux  
**Related:** **cook-torrance BRDF**, **IBL (Image-Based Lighting)**, **GI (Global Illumination)**, **glTF 2.0**, **MDL (NVIDIA Material Definition Language)**

---

### Pensieve (RL-based ABR)
A reinforcement learning-based ABR algorithm introduced in the 2017 ACM SIGCOMM paper "Neural Adaptive Video Streaming with Pensieve" by Mao, Netravali, and Alizadeh at MIT. Pensieve trains a neural network policy (using asynchronous advantage actor-critic, A3C) on simulated network traces to select video renditions; the policy input is recent throughput history, buffer level, chunk sizes, and remaining video duration. The trained policy outperforms classical throughput-based and BOLA approaches in average bitrate and rebuffering ratio on held-out traces. Pensieve runs client-side as a JavaScript module in the browser or as a native C++ policy served from a prediction endpoint. It is not yet deployed in production ABR stacks; it serves as the primary academic baseline for learned ABR research. The RL policy is trained offline; real-time inference is extremely lightweight.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **ABR (Adaptive Bitrate Streaming)**, **BOLA (Buffer Occupancy–based Lyapunov Algorithm)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **HLS (HTTP Live Streaming)**, **VMAF (Video Multi-Method Assessment Fusion)**

---

### PhysX 5
PhysX 5 (Apache 2.0, `github.com/NVIDIA-Omniverse/PhysX`, current 5.6.1) is NVIDIA's GPU-accelerated physics simulation library supporting rigid body dynamics (`PxRigidDynamic`), finite-element soft bodies (`PxSoftBody`), position-based dynamics (PBD) for cloth and fluids (`PxPBDParticleSystem`), and articulations (`PxArticulationReducedCoordinate`). GPU simulation is activated by `PxSceneDesc::flags |= PxSceneFlag::eENABLE_GPU_DYNAMICS`, with the GPU broadphase via `PxBroadPhaseType::eGPU` and a CUDA context manager (`PxCudaContextManager`). PhysX 5 integrates with OpenUSD via `UsdPhysicsRigidBodyAPI` and the `PhysxSchema` extension; USDRT Fabric provides a zero-copy geometry update path (`omni.warp.from_fabric`). In Omniverse, PhysX 5 runs co-located with the RTX Renderer on the same CUDA context, enabling GPU-driven simulation→render coupling without CPU readback.

**Chapter(s):** Ch69 — NVIDIA Omniverse, OpenUSD, and the RTX Renderer  
**Related:** **Omniverse (NVIDIA)**, **OpenUSD / USD**, **CUDA**, **WARP (NVIDIA Python GPU kernel)**

---

### Pipeline Cache
A Vulkan mechanism (`VkPipelineCache`, `vkCreatePipelineCache`) for storing and reusing the compiled binary results of pipeline creation across runs of an application. Internally, a pipeline cache stores driver-private blobs keyed by a hash of the pipeline state and SPIR-V. Applications serialize the cache to disk via `vkGetPipelineCacheData` and reload it on next launch via `vkCreatePipelineCache` with the stored data, avoiding recompilation. Vulkan mandates that caches from different devices or driver versions must be silently ignored (validated by a UUID in the cache header). Mesa's cross-driver disk cache (`src/util/disk_cache.c`) integrates with `VkPipelineCache` for all Mesa Vulkan drivers.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch16 — Mesa's Vulkan Common Infrastructure; Ch12 — The Mesa Loader and Driver Dispatch  
**Related:** **VkPipelineCache**, **VkPipeline**, **graphics pipeline library (VK_EXT_graphics_pipeline_library)**, **shader object (VK_EXT_shader_object)**

---

### Pipeline Layout
A Vulkan object (`VkPipelineLayout`) that defines the complete interface between shader resources and the pipeline: the ordered list of `VkDescriptorSetLayout` objects (one per set index) and any push constant ranges. A pipeline layout is required when creating any `VkPipeline` and when binding descriptor sets (`vkCmdBindDescriptorSets`) or writing push constants (`vkCmdPushConstants`). Two pipeline layouts are "compatible for set N" if their descriptor set layout and push constant declarations at indices 0..N are identical, allowing descriptor sets bound under one pipeline to remain valid when switching to a compatible pipeline. Misunderstanding pipeline layout compatibility is a common source of validation errors.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **descriptor set layout**, **descriptor set**, **push constant**, **VkPipeline**, **VkDescriptorPool**

---

### PipeWire (multimedia)
A low-latency graph-based media server for Linux that unifies audio (replacing PulseAudio and JACK) and video (replacing V4L2 userspace routing) under a single daemon and API. PipeWire's core abstraction is the SPA (Simple Plugin API) node: plugins expose SPA ports with negotiated format caps, and the PipeWire daemon constructs a processing graph analogous to GStreamer pipelines. For video, PipeWire handles screen capture (via the `xdg-desktop-portal` screencast interface), camera input, and inter-process video sharing, using DMA-BUF for zero-copy GPU frame passing. GStreamer integrates via `pipewiresrc` and `pipewiresink` elements; JACK clients connect via the PipeWire-JACK compatibility layer. PipeWire is the recommended video session layer for Wayland compositors; `wp_linux_drm_syncobj` synchronisation is used to maintain frame timing across the SPA graph.

**Chapter(s):** Ch38 — PipeWire and the Video Session Layer; Ch58 — GStreamer: Pipeline-Based Multimedia  
**Related:** **GStreamer**, **V4L2 (Video4Linux2)**, **FFmpeg**, **DeepStream (NVIDIA)**

---

### plane (DRM)
A `struct drm_plane` (defined in `include/drm/drm_plane.h`) representing a hardware layer within a CRTC's compositor engine that can independently scan out a framebuffer at a specified position, scale, and blend mode. DRM defines three plane types: *primary* (the base full-screen layer, mandatory for each CRTC), *overlay* (additional hardware layers for video or UI compositing), and *cursor* (a small hardware-composited sprite for the mouse pointer). Planes expose atomic properties including `FB_ID`, `CRTC_ID`, `SRC_*` (source crop), `CRTC_*` (destination rectangle), `rotation`, `alpha`, `pixel_blend_mode`, `IN_FENCE_FD` (explicit sync), and format/modifier constraints. Using hardware planes for video decode output (zero-copy YUV display) or browser quads avoids GPU compositing overhead.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **CRTC**, **framebuffer (DRM)**, **scan-out**, **atomic modesetting**, **explicit synchronisation**

---

### PQ (Perceptual Quantizer / ST 2084)
The electro-optical transfer function (EOTF) standardised as SMPTE ST 2084, designed to encode luminance values from 0.001 to 10,000 nits in a 10-bit (or higher) signal by matching human contrast-sensitivity (the Barten model). PQ is the transfer function mandated for HDR10 and HDR10+ content (used with BT.2020 primaries). Unlike gamma or HLG, PQ is an absolute transfer function: the encoded value corresponds to an absolute nit level, not a display-relative percentage. In the Linux display pipeline, PQ content must be linearised by applying the inverse PQ curve (ST 2084 EOTF) in the KMS degamma LUT before CTM colour transformation. Mesa's Vulkan drivers expose PQ via `VK_COLOR_SPACE_HDR10_ST2084_EXT` in `VK_EXT_swapchain_colorspace`.

**Chapter(s):** Ch3 — Advanced Display Features; Ch26 — Hardware Video; Ch53 — Display Calibration and colord  
**Related:** **HDR10**, **HDR metadata**, **HLG (Hybrid Log-Gamma)**, **BT.2020**, **degamma**, **wide color gamut (WCG)**

---

### PRIME
The DRM mechanism for sharing GEM buffer objects between different GPU drivers, implemented via DMA-BUF file descriptors. `DRM_IOCTL_PRIME_HANDLE_TO_FD` exports a GEM handle from one driver (e.g. a dedicated GPU) as a DMA-BUF fd; `DRM_IOCTL_PRIME_FD_TO_HANDLE` imports it into another driver (e.g. the integrated GPU driving the display). PRIME enables multi-GPU render offload: the discrete GPU renders a scene, exports the result via PRIME, and the integrated GPU scans it out without a CPU copy. The PRIME infrastructure is implemented in `drivers/gpu/drm/drm_prime.c`. Do not confuse PRIME the buffer-sharing mechanism with PRIME render offload (the composited multi-GPU configuration) — the latter uses the former as its buffer-transfer substrate.

**Chapter(s):** Ch4 — GPU Memory Management; Ch49 — Multi-GPU and PRIME Render Offload  
**Related:** **DMA-BUF**, **GEM (Graphics Execution Manager)**, **PRIME render offload**, **IOMMU**, **GBM (Generic Buffer Manager)**

---

### PRIME render offload
A multi-GPU display configuration in which a discrete GPU (e.g. NVIDIA or AMD dGPU) renders frames for applications, and an integrated GPU (iGPU) handles the display output and compositor. The discrete GPU exports rendered framebuffers via PRIME/DMA-BUF; the compositor (e.g. Sway, GNOME Shell/Mutter) imports them and scans them out through the iGPU's KMS/display engine. On Wayland+Vulkan stacks this is managed by the `DRI_PRIME` environment variable (Mesa) or Wayland's `zwp_linux_drm_syncobj_v1` pairing. The `xrandr --setprovideroffloadsink` command implements the equivalent for X11. A common pitfall is forgetting that the PRIME copy crosses a PCIe bus: zero-copy is only possible if both GPUs share an IOMMU domain or support peer-to-peer DMA. Kernel support is in `drivers/gpu/drm/drm_prime.c` and driver-specific p2p paths.

**Chapter(s):** Ch49 — Multi-GPU and PRIME Render Offload  
**Related:** **PRIME**, **DMA-BUF**, **GBM (Generic Buffer Manager)**, **IOMMU**, **ACS (PCIe Access Control Services)**, **scan-out**

---

### PSNR (Peak Signal-to-Noise Ratio)
A pixel-fidelity metric used to quantify the distortion introduced by lossy video compression, defined as `10 × log₁₀(MAX² / MSE)` where MAX is the maximum pixel value (e.g. 255 for 8-bit) and MSE is the mean squared error between the original and reconstructed frames. PSNR is expressed in decibels (dB); values above 40 dB are generally considered transparent for broadcast content. PSNR is computationally cheap and widely used in codec benchmark tooling (e.g. `ffmpeg -i orig.yuv -i encoded.yuv -filter_complex psnr`). Its main limitation: it correlates poorly with human perceptual quality for structured artefacts such as ringing, blocking, and colour banding. VMAF was developed by Netflix specifically to address PSNR's perceptual limitations. PSNR remains the standard metric for hardware encoder conformance testing.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **VMAF (Video Multi-Method Assessment Fusion)**, **CRF (Constant Rate Factor)**, **rate-distortion optimization**, **codec**, **FFmpeg**

---

### Push Constant
A small block of per-draw shader constants (maximum 128 bytes guaranteed; queried via `VkPhysicalDeviceLimits.maxPushConstantsSize`) that is written directly into the command buffer via `vkCmdPushConstants` and visible to shaders as `layout(push_constant) uniform PushConstants { ... }`. Push constants are the lowest-latency mechanism for passing per-draw data such as transform matrices or draw IDs to shaders, bypassing descriptor sets entirely. Unlike uniform buffers, push constants have no backing `VkBuffer` — the driver embeds them into the command stream or a special register file. They are distinct from specialization constants, which are compile-time constants baked at `vkCreateGraphicsPipelines` time.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch25 — GPU Compute  
**Related:** **pipeline layout**, **descriptor set**, **VkCommandBuffer**, **VkPipeline**

---

## R

### RADV (RADEON Vulkan)
RADV is AMD's Vulkan driver implemented entirely within Mesa, located at `src/amd/vulkan/`. It targets GCN through RDNA4 hardware via the `amdgpu` kernel driver and uses ACO (or optionally LLVM) as its shader compiler backend. RADV was originally developed by Valve and Bas Nieuwenhuizen outside AMD, becoming the de-facto standard AMD Vulkan driver on Linux and achieving full Vulkan 1.3 conformance. RADV supports ray tracing on RDNA2+ via the `VK_KHR_ray_tracing_pipeline` extension and Vulkan Video encode/decode on VCN hardware. It is distinct from AMD's proprietary `amdvlk` ICD (which is open-source but not in-Mesa) — RADV is preferred on Linux because it ships with Mesa and receives distro updates.

**Chapter(s):** Ch18 — Vulkan Drivers; Ch15 — ACO: AMD's Optimising Compiler; Ch56 — Ray Tracing on Linux; Ch50 — Vulkan Video Extensions  
**Related:** **ACO**, **GCN ISA**, **RDNA ISA**, **NIR**, **disk_cache (Mesa)**, **Fossilize**

---

### rate-distortion optimization
The encoder decision process that minimises a Lagrangian cost function `J = D + λ·R`, where D is distortion (e.g. SSD between original and reconstructed pixels) and R is rate (bits consumed), and λ is a Lagrange multiplier determined by the target quality or quantizer. For each coding mode candidate (macroblock type, partition size, motion vector, transform, quantizer step), the encoder evaluates J and selects the mode with the lowest cost. Rate-distortion optimisation (RDO) is the mechanism behind CRF, VBR, and quality-based rate control; it is what separates high-quality software encoders (x264, x265, libaom) from fast fixed-function hardware encoders that use heuristic shortcuts. In H.265 and AV1, RDO operates recursively over the CTU/superblock quad-tree, making it computationally expensive but enabling superior compression.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **CRF (Constant Rate Factor)**, **PSNR (Peak Signal-to-Noise Ratio)**, **VMAF (Video Multi-Method Assessment Fusion)**, **motion estimation**, **DCT (Discrete Cosine Transform)**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **NVENC**

---

### ray tracing hardware (RT Core / XMX)
RT Cores are dedicated fixed-function ray-BVH intersection test units present on NVIDIA GPUs from Turing (RTX 20xx) onward; they run BVH traversal and ray-triangle intersection independently of the shader SMs, offloading the most expensive ray tracing operation from programmable compute. On AMD RDNA2+, comparable dedicated intersection hardware accelerates `VK_KHR_ray_tracing_pipeline` workloads without being branded separately; RDNA3 adds hardware-accelerated ray-box and ray-triangle tests. Intel Arc (Xe HPG, Gen12.7+) includes dedicated Ray Tracing Units (RTUs). XMX (Xe Matrix Extensions) are Intel's tensor/matrix multiply units on Arc GPUs used for AI workloads and XeSS inference — they are not ray tracing units; the name collision with "RT Core" in some marketing is a source of confusion. In Vulkan, all three are exposed uniformly via `VK_KHR_acceleration_structure` and `VK_KHR_ray_tracing_pipeline`.

**Chapter(s):** Ch56 — Ray Tracing on Linux; Ch67 — OptiX 9; Ch71 — Intel Xe and Arc  
**Related:** **BVH**, **OptiX**, **ReSTIR / RTXDI**, **tensor core / XMX**, **XeSS (Intel Xe Super Sampling)**

---

### RDNA ISA
RDNA (Radeon DNA) ISA is AMD's shader instruction set architecture introduced with RDNA 1 (Navi10, RX 5700, 2019), superseding GCN. RDNA reduced the wave size from 64 to 32 threads (wave32), introduced a new instruction encoding, improved the scalar/vector split, and added dedicated rasteriser improvements. RDNA 2 added hardware ray tracing accelerators; RDNA 3 introduced a compute unit split (dual-issue shader array); RDNA 4 (RX 9000 series) added further ML/AI acceleration units. In Mesa, ACO and the LLVM AMDGPU backend both support RDNA ISA, selected by the target GPU's `gfx` generation string (e.g., `gfx1100` for RDNA 3). Do not conflate RDNA (a graphics-compute ISA) with CDNA (AMD's compute-only Instinct ISA), which shares heritage but has diverged significantly.

**Chapter(s):** Ch15 — ACO: AMD's Optimising Compiler; Ch18 — Vulkan Drivers; Ch48 — ROCm and Machine Learning on Linux GPUs  
**Related:** **GCN ISA**, **ACO**, **RADV**, **LLVM (in Mesa context)**, **register allocation (GPU)**

---

### REBLUR
REBLUR (REcurrent BLUr) is one of NRD's three denoising algorithms, designed for general diffuse and specular indirect lighting. It implements an adaptive-radius spatiotemporal filter where the spatial blur radius is inversely proportional to the accumulated sample count in the temporal history buffer — recently invalidated regions (disocclusions, fast motion) receive wider blurring while stable regions approach the noise-free limit. REBLUR accepts per-pixel hit distance alongside radiance to enable depth-aware filtering. It is the recommended NRD algorithm for non-ReSTIR sampling strategies; RELAX is preferred when input samples have strong temporal correlation (e.g., ReSTIR temporal reuse) because REBLUR's temporal filter amplifies correlated bias. The REBLUR shader set is configured via `NrdIntegrationDesc.denoiser = REBLUR_DIFFUSE_SPECULAR`.

**Chapter(s):** Ch70 — RTX Kit  
**Related:** **NRD (NVIDIA Real-time Denoisers)**, **RELAX**, **SIGMA (NRD denoiser)**, **ReSTIR / RTXDI**

---

### register allocation (GPU)
GPU register allocation is the compiler phase that maps an unlimited set of virtual registers (as used in NIR or SSA form) onto the finite physical register file of the GPU. Unlike CPUs, GPU physical register files are large but shared across all simultaneously resident wavefronts on a compute unit (e.g., AMD RDNA has 256 VGPRs and 106 SGPRs per SIMD; each additional register used reduces occupancy). Mesa's ACO implements its own graph-colouring and linear-scan register allocator (`src/amd/compiler/aco_register_allocation.cpp`) tuned for AMD hardware's wave-level constraints, including SGPR/VGPR split, vector/scalar separation, and spill costs that consider LDS (local data share) usage. Intel's BRW compiler similarly manages SIMD-width-aware register allocation for EU hardware. Poor register allocation is a primary cause of GPU shader performance regressions.

**Chapter(s):** Ch15 — ACO: AMD's Optimising Compiler; Ch14 — NIR: Mesa's Shader Intermediate Representation  
**Related:** **ACO**, **BRW compiler (Intel EU ISA)**, **GCN ISA**, **RDNA ISA**, **NIR**, **LLVM (in Mesa context)**

---

### RELAX
RELAX is NRD's temporally stable denoising algorithm specifically tuned for ReSTIR-sampled inputs, which exhibit strong inter-frame correlation that other denoisers misinterpret as signal. RELAX maintains separate diffuse and specular history buffers and applies an A-Trous (à-trous) wavelet spatial filter rather than an adaptive-radius blur, making it robust to the correlated noise patterns produced by spatiotemporal reservoir resampling. The key difference from REBLUR is that RELAX uses a roughness-aware specular lobe clamping strategy to suppress fireflies from ReSTIR's high-variance samples. RELAX is configured in NRD via `NrdIntegrationDesc.denoiser = RELAX_DIFFUSE_SPECULAR`; it is the algorithm used in production games shipping RTXDI such as Cyberpunk 2077.

**Chapter(s):** Ch70 — RTX Kit  
**Related:** **NRD (NVIDIA Real-time Denoisers)**, **REBLUR**, **SIGMA (NRD denoiser)**, **ReSTIR / RTXDI**

---

### render graph (see also: frame graph, Ch16 / Part IV)
A render graph (also called a frame graph) is a directed acyclic graph data structure that describes a GPU frame's sequence of render passes as nodes (each with declared input/output textures and buffers as edges), enabling an engine or middleware layer to automatically manage resource lifetimes, scheduling, barrier insertion, and pass culling. In bgfx, the render graph is implicit in the `bgfx::ViewId` submission order and transient buffer system. In Bevy/wgpu, the `RenderGraph` resource (`bevy::render::render_graph::RenderGraph`) stores nodes as trait objects. The Linux graphics-stack relevance is that Mesa's Vulkan common infrastructure (Ch16) provides render-pass lowering — transforming Vulkan's explicit `VkRenderPass`/subpass graph into driver-specific execution, which is the hardware-level analogue of the engine-level render graph concept. See also the Mesa/ACO frame graph discussion in Ch15–16.

**Chapter(s):** Ch84 — bgfx and Render Graphs; Ch16 — Mesa's Vulkan Common Infrastructure; Ch40 — Bevy and wgpu  
**Related:** **bgfx**, **Filament (rendering engine)**, **compute shader**

---

### Render Pass
In Vulkan, a `VkRenderPass` object describing the attachments (color, depth/stencil, resolve) used during rendering and the subpasses that consume them, including load/store operations (`VK_ATTACHMENT_LOAD_OP_CLEAR`, `VK_ATTACHMENT_STORE_OP_STORE`) and implicit layout transitions between subpasses. Render passes are a key optimization primitive for tile-based GPU architectures (ARM Mali, Qualcomm Adreno) where declaring subpass dependencies allows the driver to keep intermediate results in on-chip tile memory without round-tripping to DRAM. In OpenGL the equivalent concept does not exist — framebuffer binding and draw calls are stateful and implicit. `VK_KHR_dynamic_rendering` provides an alternative that avoids creating `VkRenderPass` objects.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **subpass**, **VkRenderPass**, **VkFramebuffer**, **dynamic rendering (VK_KHR_dynamic_rendering)**, **frame graph (render graph pattern)**

---

### RenderDoc
RenderDoc is an open-source, cross-platform GPU frame capture and debugging tool (`github.com/baldurk/renderdoc`) that captures a complete snapshot of a single rendered frame — all draw calls, state, resources, shaders, and outputs — at the Vulkan, OpenGL, or D3D API boundary. On Linux, RenderDoc works as a `LD_PRELOAD`-style Vulkan layer (`VK_LAYER_RENDERDOC_Capture`) injected into the target application process; frames are captured via the in-app overlay or programmatically via `RENDERDOC_API_1_6_0`. The captured `.rdc` file can be opened in the RenderDoc UI to inspect per-draw pipeline state, texture contents, shader debugger (stepping SPIR-V), and event timing. A common misconception is that RenderDoc can capture CUDA or compute-only workloads — it captures graphics API calls only; for CUDA profiling, NSight or NvPerf is appropriate.

**Chapter(s):** Ch30 — Debugging & Profiling  
**Related:** **RGP (Radeon GPU Profiler)**, **compute shader**, **CUDA**

---

### ReSTIR / RTXDI
ReSTIR (Reservoir-based Spatiotemporal Importance Resampling, Bitterli et al., SIGGRAPH 2020) is a Monte Carlo estimator for direct illumination that uses Weighted Reservoir Sampling (WRS) to maintain per-pixel light sample reservoirs that are reused temporally (across frames with motion vectors) and spatially (across neighbouring pixels with MIS-weighted visibility), dramatically improving sample efficiency under millions of dynamic lights. RTXDI v3.0 (`github.com/NVIDIAGameWorks/RTXDI`) is NVIDIA's open-source Vulkan/Slang implementation providing `RTXDI_SampleLightsForSurface`, `RTXDI_TemporalResampling`, and `RTXDI_SpatialResampling` shader functions over a `RtxdiContext` C++ host object. RTXDI v3.0 also includes ReSTIR PT (path tracing suffix reuse with reconnection shift mapping and Jacobian correction). Production references include Cyberpunk 2077 PT mode and Alan Wake 2.

**Chapter(s):** Ch70 — RTX Kit; Ch56 — Ray Tracing on Linux  
**Related:** **BVH**, **NRD (NVIDIA Real-time Denoisers)**, **RTXGI**, **ray tracing hardware (RT Core / XMX)**

---

### RGP (Radeon GPU Profiler)
The Radeon GPU Profiler (RGP) is AMD's open-source frame-level GPU performance analysis tool (`github.com/GPUOpen-Tools/radeon_gpu_profiler`) that captures hardware-level pipeline stalls, barrier costs, wave occupancy, and shader execution timelines for Vulkan and DirectX 12 applications on RDNA/GCN hardware. RGP captures are taken via the companion Radeon Developer Panel (RDP) or the AMD Developer Driver; on Linux, the `amdgpu` kernel driver's performance counter infrastructure (`drm_perfmon`) provides the counter data exposed to RGP. RGP's shader timeline view shows individual wavefront dispatch times, VGPR and SGPR occupancy, and barrier stalls — information not available from GPU timestamps alone. RGP is part of the GPUOpen tool suite and is distributed alongside the FidelityFX SDK.

**Chapter(s):** Ch72 — AMD FidelityFX & Tools; Ch30 — Debugging & Profiling  
**Related:** **RenderDoc**, **FidelityFX SDK**, **wavefront (AMD thread group)**, **compute unit (CU)**

---

### RTXGI
RTXGI (RTX Global Illumination SDK, `github.com/NVIDIAGameWorks/RTXGI`) is NVIDIA's open-source Vulkan SDK that provides two real-time indirect lighting algorithms: SHaRC (Spatially Hashed Radiance Cache) and NRC (Neural Radiance Cache). SHaRC maintains a GPU hash map with `uint64_t` spatial keys indexed by world-space position and surface normal; `SharcGetCachedRadiance` and `SharcUpdateCache` are the shader-side query and update functions, costing ~1–3 ms/frame at 1080p. NRC trains a multi-resolution hash encoding + 4-layer MLP online each frame (~4–8 ms), suitable for enclosed environments with rich indirect lighting. Both algorithms build on `VK_KHR_ray_tracing_pipeline` and can be combined with RTXDI for direct lighting, with NRD for denoising, and with DLSS for final upscale.

**Chapter(s):** Ch70 — RTX Kit; Ch56 — Ray Tracing on Linux  
**Related:** **SHaRC**, **Neural Radiance Cache (NRC)**, **ReSTIR / RTXDI**, **NRD (NVIDIA Real-time Denoisers)**, **GI (Global Illumination)**

---

### RTXNS (Neural Shaders)
RTXNS (RTX Neural Shaders SDK, v1.3, `github.com/NVIDIAGameWorks/RTXNS`) is NVIDIA's open-source Vulkan SDK that enables small neural network inference (MLPs) directly inside any Vulkan shader using the `VK_NV_cooperative_vector` extension, which exposes tensor core matrix-multiply operations (`coopVecMatMulNV`, `coopVecActivation`) as GLSL/SPIR-V built-ins. The `rtxns::NetworkLayout` C++ class configures GPU weight buffers; a Slang `NeuralNetwork<inputW, hiddenW, outputW, layers, Activation>` template generates SPIR-V Cooperative Vector instructions. Weight formats include FP16, E4M3 FP8 (Ada Lovelace+), and INT8. Linux requirements: NVIDIA driver ≥ 572.16, Ada Lovelace+ GPU, and verification via `vulkaninfo | grep VK_NV_cooperative_vector`. RTXNS is distinct from Slang's general differentiable shading — it is specifically optimised for inference on Cooperative Vector hardware.

**Chapter(s):** Ch70 — RTX Kit; Ch67 — OptiX 9  
**Related:** **tensor core / XMX**, **RTXNTC (Neural Texture Compression)**, **Slang**, **CUDA**, **OptiX**

---

### RTXNTC (Neural Texture Compression)
RTXNTC (RTX Neural Texture Compression SDK, v0.5, `github.com/NVIDIAGameWorks/RTXNTC`) replaces traditional block compression (BC7, BC6H) with a learned multi-layer perceptron encoding that compresses texture sets at up to 7× lower VRAM usage than BC7 at equivalent or higher perceptual quality. Textures are encoded offline via `rtxntc-encode --onnx --quality --format E4M3` into a `.ntc` file containing a 4-layer × 32-neuron MLP with FP8 (E4M3) weights plus a multi-resolution UV hash encoding feature grid. At runtime, `rtxns::SampleTexture` (compiling to Cooperative Vector SPIR-V via RTXNS, Ch70) performs inference to reconstruct texel values. A scalar fallback path is available for pre-Ada GPUs but at degraded performance. RTXNTC is particularly relevant for large open-world games where texture VRAM budgets are the binding constraint.

**Chapter(s):** Ch70 — RTX Kit  
**Related:** **RTXNS (Neural Shaders)**, **tensor core / XMX**, **CUDA**, **VRAM**

---

## S

### SASS (NVIDIA ISA)
SASS (Shader ASSembly) is NVIDIA's native GPU machine code ISA — the actual binary executed by NVIDIA GPU shader processors (SMs, Streaming Multiprocessors). SASS is architecture-specific: `sm_86` for Ampere, `sm_89` for Ada Lovelace, `sm_90` for Hopper, each with distinct instruction encodings. NVIDIA does not publicly document SASS; it is the output of the proprietary PTXAS assembler (which JIT-compiles PTX) or the closed-source `cubin` compiler. In the open-source NVK Vulkan driver (`src/nouveau/compiler/`), the `nak` (NVIDIA Architecture Kernel) compiler translates Mesa NIR to SASS for the corresponding Turing/Ampere hardware. SASS is distinct from PTX — PTX is portable across architectures and JIT-compiled, while SASS is architecture-specific binary that runs directly.

**Chapter(s):** Ch10b — NVK: Building a Vulkan Driver from Scratch; Ch25 — GPU Compute; Ch66 — CUDA Runtime, Streams, and NVRTC  
**Related:** **NVIR / PTX**, **ISA (Instruction Set Architecture)**, **NIR**, **SPIR-V**

---

### scan-out
The continuous process by which the display engine reads pixel data from a framebuffer in GPU memory and transmits it to the monitor over the display link, one scanline at a time, synchronised to the display's pixel clock and sync signals. The GPU's display controller begins a new frame at the start of each vertical active period, reading from the address stored in the CRTC's plane register. During vertical blanking (vblank) no active pixel data is transmitted, giving the CPU/GPU a safe window to update the framebuffer pointer (page flip) without tearing. Scan-out imposes bandwidth requirements on the memory subsystem; high-resolution HDR 10-bit 144 Hz displays require tens of GB/s of memory bandwidth just for scan-out, which is why display controller DMA ports are typically high-priority in the memory arbiter.

**Chapter(s):** Ch2 — KMS: The Display Pipeline  
**Related:** **framebuffer (DRM)**, **CRTC**, **vblank**, **page flip**, **plane (DRM)**, **BPP (Bits Per Pixel)**

---

### SDL3 GPU API
The high-level GPU abstraction layer introduced in SDL 3.0 (`SDL_GPUDevice`, `SDL_GPUCommandBuffer`, `SDL_GPURenderPass`), providing a portable rendering API over Vulkan, Metal, and D3D12 backends. SDL3 GPU is positioned as a simpler alternative to raw Vulkan for game developers who want cross-platform GPU access without writing backend-specific code. On Linux, the SDL3 Vulkan backend translates `SDL_GPUGraphicsPipeline` to `VkPipeline`, `SDL_GPUTexture` to `VkImage`, and `SDL_GPUSampler` to `VkSampler`. Unlike wgpu or Dawn, SDL3 GPU does not target the browser and does not enforce a safe subset of GPU features — it exposes the full capabilities of the underlying API.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkPipeline**, **VkSwapchainKHR**, **VkCommandBuffer**, **VMA (Vulkan Memory Allocator)**

---

### SEI (Supplemental Enhancement Information, H.264)
A category of H.264/AVC NAL unit (type 6) that carries metadata not required for correct decoding but useful for display, timing, or content-level signalling. Defined payloads include: buffering period, picture timing (PTS/DTS), pan-scan rectangle, filler payload, user data unregistered, recovery point, and stereo video info. H.265/HEVC also defines SEI messages (NAL types 39 and 40 for prefix and suffix SEI), and H.266/VVC extends them further; SEI is not unique to H.264 despite the chapter title. In practical usage, HDMI HDR metadata (SMPTE ST 2086 mastering display colour volume, MaxCLL/MaxFALL) is commonly carried as SEI messages in HEVC bitstreams. FFmpeg can inject and extract SEI payloads via the `h264_metadata` and `hevc_metadata` bitstream filters.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch57 — FFmpeg Architecture and Programming  
**Related:** **NAL unit (H.264/H.265)**, **H.264 / AVC**, **H.265 / HEVC**, **IDR frame**, **FFmpeg**

---

### SEV-SNP (AMD)
AMD SEV-SNP (Secure Encrypted Virtualisation — Secure Nested Paging) is a hardware security extension for AMD EPYC CPUs (Milan/Genoa and newer) that combines memory encryption (SEV), VM-isolation from the hypervisor (SEV-ES, encrypted register state), and integrity protection via Reverse Map Table (RMP) enforcing that each physical page is owned by exactly one VM — preventing hypervisor-level memory tampering. On Linux, the KVM hypervisor (`arch/x86/kvm/svm/sev.c`) implements SEV-SNP guest creation via `KVM_SEV_SNP_LAUNCH_*` ioctls; the guest kernel verifies the attestation report via `sev_guest` driver. In the GPU confidential computing context, AMD MI300 GPUs can operate inside SEV-SNP VMs with encrypted GPU memory when combined with the ROCm Trusted Execution Environment (TEE) stack, preventing the host from reading AI model weights or inference data.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute; Ch5 — x86 GPU Drivers  
**Related:** **confidential computing (GPU)**, **IOMMU group**, **vfio (Virtual Function I/O)**

---

### shader cache
Shader cache refers generically to any mechanism that stores compiled GPU shader binaries to avoid redundant recompilation. In Mesa, the primary shader cache is `disk_cache` (`~/.cache/mesa_shader_cache/`), keyed on driver version, GPU id, and shader IR hash. At the Vulkan API level, `VkPipelineCache` provides an application-level cache that drivers can serialise to disk; Mesa's Vulkan drivers serialise pipeline cache objects using the same underlying `disk_cache`. Fossilize extends this to enable offline pre-warming. On NVIDIA proprietary driver, the shader cache lives at `~/.nv/GLCache/`. The distinction matters for debugging: disabling Mesa disk cache (`MESA_SHADER_CACHE_DISABLE=1`) does not disable any Vulkan pipeline cache the application maintains, and vice versa.

**Chapter(s):** Ch12 — The Mesa Loader and Driver Dispatch; Ch16 — Mesa's Vulkan Common Infrastructure; Ch30 — Debugging & Profiling  
**Related:** **disk_cache (Mesa)**, **Fossilize**, **shader-db**, **RADV**, **ANV**

---

### Shader Object (VK_EXT_shader_object)
A Vulkan extension providing `VkShaderEXT` objects that decouple individual shader stages from the monolithic pipeline compilation step of `vkCreateGraphicsPipelines`. Each shader stage (vertex, fragment, compute, mesh, etc.) is compiled independently via `vkCreateShadersEXT` and bound at draw time with `vkCmdBindShadersEXT`. The driver may still link stages internally when they are first used together, but the application no longer manages `VkPipeline` objects. This eliminates the combinatorial pipeline state explosion in large rendering engines and is directly complementary to `VK_EXT_graphics_pipeline_library`. Shader objects are incompatible with render passes created via the legacy `VkRenderPass` path when mixing with non-object pipelines.

**Chapter(s):** Ch18 — Vulkan Drivers; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkPipeline**, **graphics pipeline library (VK_EXT_graphics_pipeline_library)**, **VkShaderModule**, **pipeline cache**

---

### shader-db
shader-db is a Mesa developer tool and CI regression-tracking system (`https://gitlab.freedesktop.org/mesa/shader-db`) that maintains a corpus of real-world GLSL shaders harvested from games and applications. It is used to measure the quality of Mesa shader compiler output (instruction counts, register pressure, estimated cycles) before and after compiler changes, catching performance regressions before they reach users. The `run` script compiles each shader through the Mesa driver under test and reports statistics. In Mesa CI and developer workflow, shader-db runs are mandatory for changes to ACO, the BRW compiler, or NIR optimisation passes. Shader-db does not test functional correctness — that is the role of `piglit` and the Khronos CTS.

**Chapter(s):** Ch15 — ACO: AMD's Optimising Compiler; Ch31 — Conformance & Regression Testing; Ch32 — Contributing to the Linux Graphics Stack  
**Related:** **ACO**, **NIR**, **RADV**, **disk_cache (Mesa)**, **BRW compiler (Intel EU ISA)**

---

### shaderc
shaderc is Google's open-source shader compiler library and CLI tool (`https://github.com/google/shaderc`), providing a C/C++ API and `glslc` CLI that wraps `glslang` and `SPIRV-Tools`. Its primary role is GLSL/HLSL → SPIR-V compilation with optional optimisation passes from `spirv-opt`. shaderc is used by Dawn (Chrome's WebGPU implementation), Vulkan SDK toolchains, and many Vulkan application frameworks to pre-compile shaders at build time rather than at runtime. The `glslc` CLI mimics GCC/Clang argument conventions (e.g., `-fshader-stage=`, `-I` for includes, `-O` for optimisation level). shaderc does not perform driver-specific compilation — its output is standard SPIR-V, which is then handed to Mesa via `vkCreateShaderModule`.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch35 — Dawn and WebGPU; Ch61 — SPIR-V Ecosystem in Depth  
**Related:** **glslang**, **SPIR-V**, **SPIRV-Tools**, **Tint (WGSL compiler)**, **DXC**

---

### SHaRC
SHaRC (Spatially Hashed Radiance Cache) is a real-time indirect illumination technique from RTXGI that stores irradiance in a GPU-resident hash map keyed by quantised world-space position and surface normal, updated each frame via path-traced radiance queries and accumulated with exponential moving average blending. The shader API consists of `SharcGetCachedRadiance(hashMap, positionW, normalW, roughness, out radiance)` for cache lookups and `SharcUpdateCache(hashMap, positionW, normalW, radiance)` for update writes, executed within a `vkCmdTraceRaysKHR` dispatch. SHaRC costs approximately 1–3 ms per frame at 1080p on Ada Lovelace and is optimised for open-world scenes where probe-based GI fails due to scale. It differs from NRC in that SHaRC is a pure data structure with no neural network, making it deterministic and easier to debug.

**Chapter(s):** Ch70 — RTX Kit  
**Related:** **RTXGI**, **Neural Radiance Cache (NRC)**, **GI (Global Illumination)**, **ReSTIR / RTXDI**

---

### SIGMA (NRD denoiser)
SIGMA is NRD's specialised shadow denoiser algorithm, designed to filter penumbra noise from stochastic ray-traced hard and soft shadows without the aggressive temporal smearing that general-purpose denoisers apply to shadow edges. SIGMA applies a 5×5 cross-bilateral filter in screen space, using linear depth (`IN_VIEWZ`) and hit distance to detect shadow boundaries and preserve sharp contact shadows while smoothing noisy penumbras. Unlike REBLUR and RELAX which handle diffuse/specular indirect lighting, SIGMA processes a dedicated `IN_SHADOWDATA` input channel containing the ray-traced shadow mask and hit distance. The algorithm is configured in NRD via `NrdIntegrationDesc.denoiser = SIGMA_SHADOW`.

**Chapter(s):** Ch70 — RTX Kit  
**Related:** **NRD (NVIDIA Real-time Denoisers)**, **REBLUR**, **RELAX**, **ReSTIR / RTXDI**

---

### Sixel
A DEC terminal graphics format, originally designed for the VT240 (1983), that encodes raster images as streams of 6-bit "sixels" (six vertically stacked pixels per character cell row) within a DCS string: `ESC P <P1> ; <P2> ; <P3> q <sixel-data> ST`. The three DCS parameters specify pixel aspect ratio, background select, and colour palette mode. Each sixel data byte represents a column of 6 pixels for the currently selected colour, with `#n;2;r;g;b` commands defining up to 256 palette entries. Modern terminal emulators including xterm, foot, mlterm, and WezTerm support Sixel; the `libsixel` library provides an encoder/decoder. Sixel's main limitation is its 256-colour palette ceiling, which the Kitty Graphics Protocol overcomes by transmitting raw RGB(A) pixel data.

**Chapter(s):** Ch43 — Terminal Pixel Protocols — Sixel, Kitty, and iTerm2  
**Related:** **DCS (Device Control String)**, **Kitty Graphics Protocol**, **APC (Application Program Command, terminal)**

---

### Slang
Slang is NVIDIA's open-source shading language (`github.com/shader-slang/slang`) that extends HLSL with generics, interfaces (allowing polymorphic shader dispatch without virtual functions), automatic differentiation (`[Differentiable]` attribute for gradient-based rendering and neural network training), and Cooperative Vector support for tensor core inference. The Slang compiler (`slangc`) emits SPIR-V, PTX, DXIL, GLSL, WGSL, or Metal from a single shader source. In the Linux graphics context, Slang is used extensively in RTXDI, RTXGI, RTXNS, and RTXNTC shader code, and in NVIDIA's Falcor research framework. Slang is also under evaluation by the Mesa project as a potential portable shading language targeting SPIR-V — cross-reference the Mesa/NIR shader pipeline discussion in Ch14. Importantly, Slang is a distinct project from GLSL or HLSL and should not be conflated with either.

**Chapter(s):** Ch70 — RTX Kit; Ch14 — NIR: Mesa's Shader IR; Ch69 — NVIDIA Omniverse  
**Related:** **RTXNS (Neural Shaders)**, **RTXGI**, **tensor core / XMX**, **compute shader**

---

### Smithay (Rust compositor library)
A modular Rust library (`github.com/smithay/smithay`) providing building blocks for writing Wayland compositors, analogous to `wlroots` but in safe Rust. Smithay provides a Wayland protocol dispatcher built on `wayland-server` crate, DRM/KMS backend helpers (`SmithayDrmBackend`), GBM buffer allocation, libinput event handling, and implementations of core Wayland protocols (`xdg-shell`, `linux-dmabuf`, `wp_linux_drm_syncobj_v1`). The COSMIC desktop's `cosmic-comp` compositor is the primary production consumer of Smithay. Unlike wlroots, Smithay imposes no specific architecture; compositors compose library components explicitly. Smithay's DRM backend exercises the same KMS atomic commit and GBM allocation paths as wlroots, and its Rust-first design aligns with the Nova/DRM-Rust driver initiative.

**Chapter(s):** Ch22 — Production Compositors; Ch21 — Building Compositors with wlroots  
**Related:** **wlroots**, **wl_compositor**, **zwp_linux_dmabuf_v1**, **linux-drm-syncobj-v1 (Wayland protocol)**

---

### SPIR-V
SPIR-V (Standard Portable Intermediate Representation — Vulkan) is the Khronos binary shader intermediate language used as the primary shader input format for Vulkan and OpenCL 2.1+. SPIR-V is a typed, SSA-form, binary IR with a fixed-width 32-bit word encoding; it represents shader programs as `OpFunction`/`OpLabel`/`OpPhi` structures with typed instructions for arithmetic, memory, and control flow. In Mesa, SPIR-V enters via `vkCreateShaderModule` and is translated to NIR by `src/compiler/spirv/spirv_to_nir.c`. SPIR-V is vendor-neutral — GLSL, HLSL, WGSL, and Slang can all compile to SPIR-V, making it the universal interchange format for the Vulkan ecosystem. Do not confuse SPIR-V with SPIR (an older OpenCL binary format based on LLVM IR) — they share a name prefix but are unrelated binary formats.

**Chapter(s):** Ch14 — NIR: Mesa's Shader Intermediate Representation; Ch16 — Mesa's Vulkan Common Infrastructure; Ch61 — SPIR-V Ecosystem in Depth  
**Related:** **NIR**, **SPIRV-Tools**, **spirv-cross**, **glslang**, **Tint (WGSL compiler)**, **naga**, **shaderc**

---

### SPIR-V Cooperative Matrix (VK_KHR_cooperative_matrix)
A Vulkan extension and corresponding SPIR-V capability (`CooperativeMatrixKHR`) enabling groups of shader invocations within a subgroup or larger scope to collectively perform matrix multiply-accumulate (MMA) operations on matrix fragments, mapping directly to hardware tensor core or matrix engine instructions (NVIDIA Tensor Cores, AMD WMMA, Intel XMX). The extension defines `OpCooperativeMatrixMulAddKHR` in SPIR-V; the host API queries supported matrix type/dimension/scope combinations via `vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR`. This is the Vulkan equivalent of CUDA's `wmma` API or ROCm's `rocwmma`. It is primarily used in machine learning inference shaders and is foundational to GPU-accelerated transformers running through Vulkan compute.

**Chapter(s):** Ch25 — GPU Compute; Ch61 — SPIR-V Ecosystem in Depth; Ch48 — ROCm and Machine Learning on Linux GPUs  
**Related:** **SYCL 2020**, **OpenCL**, **VkPipeline**, **VkCommandBuffer**

---

### spirv-cross
spirv-cross is a Khronos-associated open-source library (`https://github.com/KhronosGroup/SPIRV-Cross`) that decompiles SPIR-V binary back to human-readable high-level shading languages: GLSL, HLSL, MSL (Metal Shading Language), and ESSL. It is widely used for debugging (inspecting driver-consumed SPIR-V), for cross-platform shader portability (porting Vulkan shaders to Metal/iOS via MSL), and in tools like RenderDoc's shader viewer. In Mesa debugging workflows, `spirv-cross -V` on a captured SPIR-V blob can reveal what the driver actually received before NIR translation. spirv-cross does not optimise SPIR-V and is not a forward compiler (GLSL → SPIR-V); that is `glslang` or `shaderc`. It also does not produce NIR — only Mesa's internal SPIR-V-to-NIR pass does that.

**Chapter(s):** Ch30 — Debugging & Profiling; Ch61 — SPIR-V Ecosystem in Depth; Ch34 — ANGLE — WebGL on Linux  
**Related:** **SPIR-V**, **SPIRV-Tools**, **glslang**, **NIR**, **shaderc**

---

### SPIRV-Tools
SPIRV-Tools is the Khronos reference toolset for working with SPIR-V binaries, hosted at `https://github.com/KhronosGroup/SPIRV-Tools`. It provides: `spirv-as`/`spirv-dis` (assembler/disassembler), `spirv-val` (validator per the SPIR-V specification), `spirv-opt` (an optimiser applying SPIR-V-level transformation passes such as dead code elimination, inlining, and constant folding), and `spirv-link` (linker for SPIR-V modules). In Mesa, `spirv-val` is run in debug builds to catch specification violations before NIR translation. The `spirv-opt` passes can be applied before driver submission to reduce driver-side work, and shaderc integrates `spirv-opt` behind its `-O` flag. SPIRV-Tools is distinct from `spirv-cross` (which decompiles to high-level languages) and from `glslang` (which compiles to SPIR-V).

**Chapter(s):** Ch61 — SPIR-V Ecosystem in Depth; Ch30 — Debugging & Profiling; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **SPIR-V**, **spirv-cross**, **glslang**, **shaderc**, **NIR**

---

### sRGB
The standard RGB colour space defined by IEC 61966-2-1, the implicit colour space of most web content, windowed GUI applications, and SDR display pipelines. sRGB uses the BT.709 colour primaries, a D65 white point, and a piecewise transfer function approximating gamma 2.2 (with a linear segment below ~0.04 nits). In the KMS colour pipeline, sRGB-encoded framebuffers are linearised by the degamma LUT before colour management operations and re-encoded by the gamma LUT for output. The `DRM_FORMAT_XRGB8888` scanout format implies sRGB encoding unless a format modifier or plane property overrides it. A misconception is that sRGB and BT.709 are identical: they share primaries but sRGB defines the encoding for computer monitors while BT.709 defines encoding for broadcast video, and their transfer functions differ slightly in the dark-tone region.

**Chapter(s):** Ch3 — Advanced Display Features; Ch53 — Display Calibration and colord  
**Related:** **BT.709**, **degamma**, **color LUT**, **wide color gamut (WCG)**, **CTM (Color Transform Matrix)**

---

### SRT (Secure Reliable Transport)
An open-source UDP-based transport protocol developed by Haivision and now governed by the SRT Alliance (srtalliance.org), designed for reliable low-latency video contribution over unpredictable internet links. SRT uses selective ARQ (automatic repeat request) retransmission with a configurable latency budget, AES-128/256 encryption for content security, and a caller/listener/rendezvous connection model. Unlike RTMP (TCP, prone to head-of-line blocking) and WebRTC (peer-to-peer, complex ICE/STUN), SRT targets professional broadcast contribution and live streaming ingest from encoders to streaming servers. On Linux, `libsrt` is the reference implementation; FFmpeg supports SRT via `srt://` protocol URLs (e.g. `ffmpeg -i srt://host:port`) and GStreamer via `srtsrc`/`srtsink` elements in `gst-plugins-bad`. mediamtx and SRS are common open-source SRT relay server implementations.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery; Ch57 — FFmpeg Architecture and Programming  
**Related:** **FFmpeg**, **GStreamer**, **WebRTC**, **HLS (HTTP Live Streaming)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **MOQT (Media over QUIC Transport)**

---

### Subpass
A subdivision within a Vulkan `VkRenderPass` that reads from a subset of attachments as input attachments (`layout(input_attachment_index=N) uniform subpassInput`) and writes to a subset as color/depth outputs. Subpasses are connected by `VkSubpassDependency` structs declaring `srcSubpass`/`dstSubpass` execution and memory dependencies. The key purpose is enabling tile-based GPU drivers to keep G-buffer data from a geometry subpass in on-chip tile memory and consume it in a lighting subpass without writing to DRAM — a technique that can halve memory bandwidth on mobile hardware. On desktop GPUs, subpass dependencies reduce to standard pipeline barriers. `VK_KHR_dynamic_rendering` does not natively support subpass-style on-chip merging.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **render pass**, **VkRenderPass**, **VkFramebuffer**, **dynamic rendering (VK_KHR_dynamic_rendering)**

---

### Sunshine (game streaming)
Sunshine is an open-source (MIT, `github.com/LizardByte/Sunshine`), self-hosted game streaming server for Linux (and Windows/macOS) that implements the NVIDIA GameStream protocol, making it compatible with NVIDIA Shield clients and the Moonlight open-source client. On Linux, Sunshine captures the desktop via KMS/DRM (`/dev/dri/cardN` CRTC framebuffer readback), NvFBC (NVIDIA, proprietary driver only), or PipeWire/Wayland screencopy; it encodes the captured frames using NVENC (NVIDIA), VAAPI (AMD/Intel), or software encoders (libx264/libx265) and transmits them over RTSP with NVIDIA's video streaming protocol. The Wayland capture path uses the `zwlr_screencopy_manager_v1` protocol where supported; for NvFBC, the proprietary NVIDIA driver is mandatory.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute  
**Related:** **NvFBC (NVIDIA Framebuffer Capture)**, **lookahead (encoder)**, **AMF (AMD Advanced Media Framework)**

---

### SurfaceFlinger
Android's system compositor service (`frameworks/native/services/surfaceflinger`), running as a privileged system process that composites all application and system UI layers into the final display frames. SurfaceFlinger receives `GraphicBuffer`s from app windows via `BufferQueue`, passes the layer list to `HWComposer` for hardware plane assignment, GPU-composites any `CLIENT`-marked layers using Skia or OpenGL ES into a client-target buffer, and then submits the final frame to the display via `HWComposer`. SurfaceFlinger manages VSync timing, display power states, and screen rotation. It is the Android equivalent of a Wayland compositor: apps are producers, SurfaceFlinger is the consumer and display controller.

**Chapter(s):** Ch80 — SurfaceFlinger and the Android Display Pipeline  
**Related:** **HWComposer (Android)**, **BufferQueue (Android)**, **HWUI (Android)**, **Gralloc**

---

### Swapchain
The collection of presentable images that an application renders into and submits to the display system. In Vulkan, `VkSwapchainKHR` (from `VK_KHR_swapchain`) owns N images (typically 2–3 for double/triple buffering); the application acquires an available image via `vkAcquireNextImageKHR`, renders into it, then presents it via `vkQueuePresentKHR`. On Linux/Wayland, Mesa's WSI layer (`src/vulkan/wsi/wsi_common_wayland.c`) backs swapchain images with GBM-allocated DMA-BUF objects passed to the compositor via `linux-dmabuf`. In EGL the analogous concept is the EGL surface with `eglSwapBuffers`. A Wayland swapchain reports `currentExtent == UINT32_MAX` because the compositor controls surface size; applications must query `wl_surface.preferred_buffer_scale`.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch12 — The Mesa Loader and Driver Dispatch  
**Related:** **VkSwapchainKHR**, **EGL (Embedded/Extensible Graphics Library)**, **VkSemaphore**, **VkFence (binary)**

---

### SYCL 2020
A royalty-free Khronos standard for single-source C++17 heterogeneous programming, allowing GPU kernels and host code to coexist in the same `.cpp` translation unit. SYCL 2020 added unified shared memory (USM), parallel reductions, group algorithms, and `sycl::ext::oneapi::experimental` extensions for graph APIs. On Linux, Intel's DPC++ (oneAPI) implements SYCL via an LLVM-based compiler that targets SPIR-V for GPU execution, routing through Level Zero or OpenCL ICDs. AMD's HIP SYCL adapter and the open-source `hipSYCL`/`AdaptiveCpp` project extend SYCL to AMDGPU via ROCm. SYCL is frequently confused with OpenCL — SYCL is a C++ abstraction layer; OpenCL is the underlying runtime it may target.

**Chapter(s):** Ch62 — SYCL 2020 and Portable Heterogeneous Compute; Ch25 — GPU Compute  
**Related:** **OpenCL**, **SPIR-V cooperative matrix (VK_KHR_cooperative_matrix)**, **VkPipeline**

---

### sync file
A Linux kernel synchronisation fd (`struct sync_file`, defined in `include/linux/sync_file.h`, implemented in `drivers/dma-buf/sync_file.c`) that wraps one or more `dma_fence` objects and is exported to userspace as a file descriptor. Created by `sync_file_create()` and exported via the `SYNC_IOC_MERGE` ioctl, or directly from a GPU command-submission ioctl that returns an out-fence fd. A sync file fd can be polled (`select`/`epoll`) to block until the underlying fence signals; it can also be inspected via `SYNC_IOC_FILE_INFO` to enumerate its constituent fences. Android's native "sync fence" mechanism is built on `sync_file`. In the DRM explicit sync path, GPU drivers return a sync file fd as an out-fence on plane commits (`OUT_FENCE_PTR` atomic property) and accept one as an in-fence (`IN_FENCE_FD` plane property).

**Chapter(s):** Ch3 — Advanced Display Features; Ch4 — GPU Memory Management; Ch20 — Wayland Protocol Fundamentals  
**Related:** **DMA fence**, **drm_syncobj**, **explicit synchronisation**, **fence (DRM)**, **plane (DRM)**

---

## T

### TBDR (Tile-Based Deferred Rendering)
A GPU rendering architecture in which the screen is divided into small tiles (typically 16×16 or 32×32 pixels) and geometry processing is completed for the entire frame before rasterisation begins, allowing each tile to be rendered entirely in fast on-chip tile memory without repeatedly fetching from and writing to main GPU memory (LMEM). This defers all fragment shading to a second pass once the visibility (HSR — Hidden Surface Removal) of every triangle per tile is known, minimising overdraw and memory bandwidth. TBDR is the dominant architecture for mobile and embedded GPUs: Apple AGX (used with the DCP display subsystem on Asahi Linux), ARM Mali (Midgard, Bifrost, Valhall), Imagination PowerVR, and Qualcomm Adreno all use TBDR or TBDR-derived designs. The Linux kernel's GPU drivers for these (panfrost, asahi, pvr, msm) must be aware of tile-memory flush semantics when synchronising render passes via DMA fences.

**Chapter(s):** Ch6 — ARM & Embedded GPU Drivers  
**Related:** **DCP (Display Co-Processor)**, **DMA fence**, **GEM (Graphics Execution Manager)**, **scan-out**

---

### tensor core / XMX
Tensor cores are dedicated matrix-multiply-accumulate (MMA) units on NVIDIA GPUs (Volta onward) that perform D = A×B + C on small matrices (e.g., 16×16×16 for FP16 on Ampere) in a single clock cycle, providing an order-of-magnitude throughput improvement over scalar FP32 for AI inference and training. In CUDA, tensor cores are accessed via WMMA (Warp Matrix Multiply-Accumulate) intrinsics (`nvcuda::wmma::mma_sync`) or via cuBLAS/cuDNN/CUTLASS. In Vulkan, they are exposed through `VK_NV_cooperative_matrix` and, for inference specifically, `VK_NV_cooperative_vector` (used by RTXNS). Intel's XMX (Xe Matrix Extensions) are the equivalent matrix accelerator units on Intel Arc GPUs, used for XeSS inference and oneAPI/Level Zero workloads; they are accessed via the `intel_sub_group_*` extensions in OpenCL/Level Zero or `VK_KHR_cooperative_matrix` in Vulkan.

**Chapter(s):** Ch66 — CUDA Runtime, Streams, and NVRTC; Ch71 — Intel Xe and Arc; Ch70 — RTX Kit  
**Related:** **CUDA**, **DLSS 4**, **XeSS (Intel Xe Super Sampling)**, **RTXNS (Neural Shaders)**, **warp (NVIDIA thread group)**

---

### TensorRT
TensorRT is NVIDIA's closed-source deep learning inference optimisation library (`libnvinfer.so`) that takes a trained neural network (from ONNX, TensorFlow, or PyTorch) and produces a GPU-architecture-specific optimised engine plan (`.plan` file) for deployment. The build pipeline: ONNX model → `trtexec --onnx --saveEngine --fp16 --timingCacheFile` (or C++ `IBuilder::buildSerializedNetwork`) → serialised engine. The runtime API: `ICudaEngine::deserialize`, `IExecutionContext::setTensorAddress`, `IExecutionContext::enqueueV3(cudaStream)`. TensorRT applies kernel fusion, FP16/INT8 quantisation, and layer-level optimisation specific to the target GPU SM version; engine plans are not portable across GPU generations. On Linux, TensorRT interoperates with Vulkan via `VK_KHR_external_memory` for zero-copy tensor→image transfers, enabling hybrid Vulkan rasterisation + TensorRT inference pipelines.

**Chapter(s):** Ch68 — DLSS 4, Neural Rendering, and Frame Generation  
**Related:** **CUDA**, **CUDA stream**, **DLSS 4**, **tensor core / XMX**, **NVRTC**

---

### timeline semaphore
A Vulkan synchronisation primitive (`VkSemaphore` with `VkSemaphoreTypeCreateInfo.semaphoreType = VK_SEMAPHORE_TYPE_TIMELINE`, introduced in Vulkan 1.2 via `VK_KHR_timeline_semaphore`) that carries a monotonically increasing 64-bit integer counter rather than a binary signal/wait state. Signal and wait operations specify a counter value; multiple CPU and GPU operations can signal and wait on different counter values of the same semaphore without resetting it. In Mesa drivers (RADV, ANV, NVK), Vulkan timeline semaphores are backed by DRM syncobj timelines (`drm_syncobj` with `dma_fence_chain` points), implemented in `drivers/gpu/drm/drm_syncobj.c`. Do not define timeline semaphores by analogy with OpenGL fences or classic binary semaphores; the counter-based model is qualitatively different and enables wait-before-signal patterns.

**Chapter(s):** Ch3 — Advanced Display Features; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **drm_syncobj**, **DMA fence**, **explicit synchronisation**, **linux-drm-syncobj-v1**, **sync file**

---

### Tint (WGSL compiler)
Tint is the reference WGSL (WebGPU Shading Language) compiler developed as part of the Dawn project, located at `src/tint/` within `https://dawn.googlesource.com/dawn`. It parses WGSL source, constructs a typed AST, lowers it to Tint IR, and then emits SPIR-V (for Vulkan), HLSL (for D3D12), MSL (for Metal), or GLSL (for OpenGL/ES fallback). In Chromium, Tint is invoked as part of the Dawn shader compilation pipeline before `vkCreateShaderModule` is called; the emitted SPIR-V enters Mesa NIR identically to any other SPIR-V. Tint implements WGSL's strict portability restrictions (no undefined behaviour, no implicit type conversions) which makes its output more predictable but requires more explicit WGSL source. Tint is distinct from naga, which is the analogous WGSL compiler in Firefox's wgpu stack.

**Chapter(s):** Ch35 — Dawn and WebGPU; Ch14 — NIR: Mesa's Shader Intermediate Representation  
**Related:** **WGSL**, **SPIR-V**, **naga**, **Dawn**, **NIR**, **shaderc**

---

### Tracy (profiler)
An open-source, low-overhead frame profiler for C++ applications ([https://github.com/wolfpld/tracy](https://github.com/wolfpld/tracy)) that captures CPU zones, GPU timestamps, memory allocations, and frame markers. Tracy operates as an always-on instrumented profiler rather than a sampling profiler: the application links `TracyClient.cpp` and annotates code with `ZoneScoped` macros; a separate GUI server (`tracy`) receives and visualizes the data over TCP in real time. GPU timing on Vulkan is captured via `VkQueryPool` timestamp queries with `TracyVkCtx`. Tracy is commonly used alongside RenderDoc for frame-level analysis and is the primary profiler integrated into Bevy and many indie game engines on Linux.

**Chapter(s):** Ch30 — Debugging & Profiling  
**Related:** **Dear ImGui**, **VkCommandBuffer**, **VkQueue**

---

### transform coding
The compression technique in which a block of spatial-domain pixel residuals is mapped to a frequency-domain coefficient representation via a linear transform (typically DCT or DST), followed by quantization and entropy coding of the resulting sparse coefficient array. The theoretical motivation is energy compaction: natural video content has most of its spatial energy concentrated in low-frequency components, so a frequency-domain representation allows the encoder to aggressively discard high-frequency detail with minimal perceptual impact. After transform and quantization, the dominant coding cost shifts from raw pixel values to run-length and level coding of the coefficient scan order. Transform coding is the lossy core of all block-based video codecs (H.264, HEVC, AV1, VP9); lossless modes bypass quantization but retain the transform for coefficient prediction benefits.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs  
**Related:** **DCT (Discrete Cosine Transform)**, **entropy coding (CABAC/CAVLC)**, **macroblock**, **intra-prediction**, **inter-prediction**, **rate-distortion optimization**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**

---

### TTM (Translation Table Manager)
The kernel GPU memory manager (`drivers/gpu/drm/ttm/`) that provides a generic framework for managing GPU virtual address space, VRAM placement, and buffer object eviction across driver boundaries. TTM defines `struct ttm_buffer_object` with memory placement policies (system RAM, VRAM/LMEM, CPU-accessible aperture), an LRU eviction mechanism that migrates buffers from VRAM to system RAM under memory pressure, and GPU virtual address space management via `ttm_resource_manager`. Most major DRM drivers use TTM as the backing memory manager for GEM objects: amdgpu, nouveau, i915 (for LMEM), xe, radeon, vmwgfx. Simple drivers (vc4, panfrost) use the lighter-weight GEM CMA or shmem helpers instead. TTM is internal kernel infrastructure; userspace interacts with GEM handles and never calls TTM APIs directly.

**Chapter(s):** Ch4 — GPU Memory Management; Ch5 — x86 GPU Drivers  
**Related:** **GEM (Graphics Execution Manager)**, **LMEM (Local Memory)**, **DMA-BUF**, **GBM (Generic Buffer Manager)**

---

### Turnip (Qualcomm Adreno Vulkan)
Turnip is the open-source Mesa Vulkan driver for Qualcomm Adreno GPUs (A6xx and later), located at `src/freedreno/vulkan/`. It is developed primarily by Collabora and Google engineers using a reverse-engineered understanding of the Adreno ISA and command stream, without access to Qualcomm hardware documentation. Turnip works in conjunction with the `msm` DRM kernel driver (`drivers/gpu/drm/msm/`) and uses the `ir3` compiler (`src/freedreno/ir3/`) to translate NIR to Adreno shader binary. As a tile-based deferred renderer, Turnip implements `gmem` (GPU local memory for tile storage) and `sysmem` (system memory fallback) rendering paths, exposed via custom Vulkan extensions. Do not confuse Turnip with the freedreno OpenGL driver (`src/gallium/drivers/freedreno/`) — they share the `ir3` compiler but are separate API implementations.

**Chapter(s):** Ch6 — ARM & Embedded GPU Drivers; Ch18 — Vulkan Drivers  
**Related:** **NIR**, **Gallium3D**, **SPIR-V**, **ISA (Instruction Set Architecture)**, **ANV**, **RADV**

---

## V

### V4L2 (Video4Linux2)
The Linux kernel API for video capture, output, and memory-to-memory (M2M) codec devices, defined in `include/uapi/linux/videodev2.h`. V4L2 exposes devices as character files under `/dev/video*`; applications use `ioctl(VIDIOC_*)` calls to negotiate formats (`VIDIOC_S_FMT`), allocate buffers (`VIDIOC_REQBUFS`), and queue/dequeue frames (`VIDIOC_QBUF`/`VIDIOC_DQBUF`). The *stateless codec* path (introduced ~4.16) implements request API (`/dev/media*`) to associate compressed input and raw output buffers with a per-frame control set; the kernel driver performs the actual hardware decode. The *stateful codec* path (older) treats the codec as a black box. GStreamer's `v4l2h264dec`, `v4l2hevcenc`, and `v4l2src` elements use V4L2; FFmpeg accesses V4L2 via the `v4l2_m2m` hwaccel. V4L2 is the primary kernel codec API on ARM SoCs (RPi, Jetson) and Chromebooks.

**Chapter(s):** Ch26 — Hardware Video; Ch58 — GStreamer: Pipeline-Based Multimedia  
**Related:** **codec**, **GStreamer**, **FFmpeg**, **H.264 / AVC**, **H.265 / HEVC**, **AV1**, **NVDEC**, **PipeWire (multimedia)**

---

### VA-API
The Video Acceleration API, a Linux/BSD open-source API (`libva`, MIT-licensed) providing a vendor-neutral interface for hardware video decode, encode, and processing. The API surface covers `VADisplay`, `VAContext`, `VASurface`, `VABuffer`, and `vaPutSurface`; codec-specific parameter structures (`VAPictureParameterBufferH264`, `VAEncSequenceParameterBufferHEVC`, etc.) pass codec state to the hardware. Drivers are loadable shared libraries (`iHD_drv_video.so` for Intel, `radeonsi_drv_video.so` for AMD via Mesa). VA-API surfaces are internally DMA-BUF objects and can be exported (`vaExportSurfaceHandle` with `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2`) for zero-copy import into Vulkan or direct compositor display via `linux-dmabuf`. VA-API is distinct from Vulkan Video: both target hardware codecs but have incompatible APIs and different hardware support matrices.

**Chapter(s):** Ch26 — Hardware Video; Ch50 — Vulkan Video Extensions  
**Related:** **VkImage**, **VkSwapchainKHR**, **EGL (Embedded/Extensible Graphics Library)**, **OpenXR**

---

### vblank
The vertical blanking interval — the period between the end of the last active scanline of one frame and the start of the first active scanline of the next frame, during which the display's electron beam (on CRTs) or the timing controller (on LCDs) resets to the top of the screen. In DRM/KMS, vblank events are tracked by `drivers/gpu/drm/drm_vblank.c`; drivers call `drm_crtc_handle_vblank()` from their vblank interrupt handler. Userspace waits for vblank via `DRM_IOCTL_WAIT_VBLANK` or receives `DRM_EVENT_VBLANK` / `DRM_EVENT_FLIP_COMPLETE` events. Vblank intervals are the synchronisation point for page flips (avoiding tearing) and for the Wayland `wp_presentation` protocol's presentation timestamps. With VRR enabled, the vblank interval is variable; drivers report this via the `VRR_CAPABLE` property.

**Chapter(s):** Ch2 — KMS: The Display Pipeline; Ch3 — Advanced Display Features  
**Related:** **page flip**, **scan-out**, **CRTC**, **VRR (Variable Refresh Rate)**, **KMS**

---

### vfio (Virtual Function I/O)
VFIO is a Linux kernel framework (`drivers/vfio/`) that allows safe userspace access to physical hardware devices, primarily for PCI device pass-through to virtual machines, by routing DMA operations through the IOMMU. A device is handed to VFIO by binding its kernel driver to `vfio-pci`: `echo <BDF> > /sys/bus/pci/devices/<BDF>/driver/unbind && echo vfio-pci > /sys/bus/pci/devices/<BDF>/driver_override && echo <BDF> > /sys/bus/pci/drivers/vfio-pci/bind`. The VFIO device file (`/dev/vfio/<group>`) is then opened by a VMM (QEMU/KVM via `-device vfio-pci`) or by user-space applications for direct hardware access. All devices in the same IOMMU group must be bound to VFIO simultaneously. VFIO is the foundation for GPU pass-through to VMs and for GPU confidential computing deployments on Linux.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute; Ch5 — x86 GPU Drivers  
**Related:** **IOMMU group**, **confidential computing (GPU)**, **SEV-SNP (AMD)**

---

### VGEM
A virtual GEM driver (`drivers/gpu/drm/vgem/vgem_drv.c`) that creates a software-only DRM device (`/dev/dri/renderD*`) with no physical GPU behind it. VGEM provides a GEM allocator backed by anonymous memory pages and supports DMA-BUF export/import, allowing it to act as a neutral buffer-exchange intermediary between GPU drivers that cannot directly import each other's native GEM objects. VGEM is used extensively in `igt-gpu-tools` test cases that need to pass buffers between two different GPU drivers without a real multi-GPU setup, and in Google Chrome's SurfaceTexture path on Chrome OS for cross-driver zero-copy sharing. VGEM does not support rendering; it is purely a buffer allocation and sharing facility.

**Chapter(s):** Ch1 — DRM Architecture & the Driver Model; Ch31 — Conformance & Regression Testing  
**Related:** **GEM (Graphics Execution Manager)**, **DMA-BUF**, **PRIME**

---

### video segment
A fixed-duration media file or byte range that constitutes the unit of delivery in HLS and MPEG-DASH adaptive bitrate streaming. In legacy HLS deployments, segments are MPEG-2 Transport Stream files (`.ts`) of 2–10 seconds; modern HLS and DASH use CMAF-packaged fragmented MP4 files (`.m4s`). Segment duration directly determines switching latency: shorter segments allow faster quality adaptation but increase HTTP request overhead and origin server load. IDR frame alignment is required at every segment boundary to ensure a player switching renditions mid-stream can decode from the first frame of the new segment without referencing frames from a previous rendition. In LL-HLS, a segment is further subdivided into *partial segments* (chunks) for sub-2-second delivery while the full segment remains the unit referenced by CDN caches.

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **HLS (HTTP Live Streaming)**, **DASH (Dynamic Adaptive Streaming over HTTP)**, **ABR (Adaptive Bitrate Streaming)**, **CMAF (Common Media Application Format)**, **M3U8 playlist**, **MPD (MPEG-DASH Media Presentation Description)**, **IDR frame**, **LL-HLS (Low-Latency HLS)**

---

### VirtIO-GPU
A paravirtualised GPU device specification (`virtio-spec`, device ID 16) that provides 2D/3D graphics acceleration inside virtual machines via a virtio transport, removing the need to pass through physical GPU hardware. The guest kernel driver (`virtio_gpu.ko`) implements the DRM/KMS interfaces — exposing GEM objects and KMS connectors — so guest Mesa drivers function normally. The host side is implemented by QEMU (`hw/display/virtio-gpu.c`), which relays 3D commands to the host GPU via `virgl` (Virgil 3D, an OpenGL-over-virtio translation layer) or the newer `venus` protocol (Vulkan-over-virtio, using `VK_EXT_external_memory_host`). VirtIO-GPU supports DMA-BUF export from guest GEM objects, enabling zero-copy display of guest framebuffers into host compositor windows.

**Chapter(s):** Ch55 — GPU Containers and Cloud Compute; Ch6 — ARM GPU Drivers  
**Related:** **wl_drm (legacy)**, **zwp_linux_dmabuf_v1**, **wl_buffer**, **wlroots**

---

### vk-bootstrap
A small open-source C++ helper library ([https://github.com/charles-lunarg/vk-bootstrap](https://github.com/charles-lunarg/vk-bootstrap)) that reduces the boilerplate of Vulkan initialization — `VkInstance`, `VkPhysicalDevice`, `VkDevice`, and `VkSwapchainKHR` creation — to a builder-pattern API. `vkb::InstanceBuilder` selects Vulkan version and enables validation layers; `vkb::PhysicalDeviceSelector` picks the best GPU by device type and feature requirements; `vkb::DeviceBuilder` creates the logical device; `vkb::SwapchainBuilder` creates the swapchain with sensible defaults. It does not replace VMA or abstract Vulkan beyond initialization. It is widely used in tutorials, prototypes, and projects such as the `vkguide.dev` tutorial that targets Linux as the primary platform.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkInstance**, **VkPhysicalDevice**, **VkDevice**, **VkSwapchainKHR**, **Vulkan loader**

---

### VkBuffer
A Vulkan object representing a linear array of bytes in device memory, used for vertex data, index data, uniform data, storage buffers, indirect draw arguments, ray tracing acceleration structure scratch space, and more. A `VkBuffer` is created with `vkCreateBuffer` specifying a size and a `VkBufferUsageFlags` bitmask; it must be bound to a `VkDeviceMemory` allocation (or a VMA-managed block) via `vkBindBufferMemory` before use. Unlike `VkImage`, a `VkBuffer` has no layout state, no tiling, and no format — it is a raw byte range. Shaders access buffers via descriptor set bindings (`VK_DESCRIPTOR_TYPE_STORAGE_BUFFER`, `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`) or, for small amounts, as push constants.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkImage**, **VkDescriptorPool**, **descriptor set**, **VMA (Vulkan Memory Allocator)**, **push constant**

---

### VkCommandBuffer
The primary Vulkan object for recording GPU commands. A `VkCommandBuffer` is allocated from a `VkCommandPool` and records `vkCmd*` calls — render pass commands, draw/dispatch calls, pipeline barriers, copy operations, and debug labels. Command buffers are not executed until submitted to a `VkQueue` via `vkQueueSubmit`/`vkQueueSubmit2`. Command buffers can be primary (directly submittable) or secondary (recorded inline into a primary via `vkCmdExecuteCommands`). Recording is single-threaded per command buffer, but multiple command buffers can be recorded in parallel on different CPU threads and submitted together. Command buffers are reset (via `vkResetCommandBuffer` or pool reset) for reuse; they do not automatically synchronize with the GPU.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkQueue**, **VkSubmitInfo**, **VkFence (binary)**, **VkSemaphore**, **render pass**

---

### VkDescriptorPool
A Vulkan object (`VkDescriptorPool`) from which `VkDescriptorSet` allocations are carved out. Created with `vkCreateDescriptorPool` specifying maximum counts of each descriptor type (e.g., `maxSets=100`, `VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER: 50`) and optionally `VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT` to allow individual set freeing. Without that flag, all sets must be freed at once via `vkResetDescriptorPool`. Pool sizing is application-managed — underestimating counts causes `VK_ERROR_OUT_OF_POOL_MEMORY`. `VMA` does not manage descriptor pools; libraries such as `vk-bootstrap` help with initial setup but descriptor pool management typically requires application-specific strategies (e.g., one pool per frame, growable pools).

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **descriptor set**, **descriptor set layout**, **VkDevice**, **pipeline layout**

---

### VkDevice
The Vulkan logical device object, created by `vkCreateDevice` from a `VkPhysicalDevice` with a list of requested queue families (`VkDeviceQueueCreateInfo`), enabled features (`VkPhysicalDeviceFeatures2`), and extension names. `VkDevice` is the central object from which almost all other Vulkan objects are created: `vkCreateBuffer`, `vkCreateImage`, `vkAllocateMemory`, `vkCreateCommandPool`, `vkCreatePipeline`, etc. A single application may create multiple logical devices from the same or different physical devices (e.g., for multi-GPU), but objects from different devices cannot be shared. `VkDevice` is analogous to an OpenGL context in scope but is not thread-local: Vulkan commands are explicitly dispatched to queues.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkPhysicalDevice**, **VkInstance**, **VkQueue**, **VkCommandBuffer**

---

### VkFence (binary)
A CPU-GPU synchronization primitive (`VkFence`) that has two states: unsignaled and signaled. A fence is passed to `vkQueueSubmit` and transitions to signaled when the submitted command buffer completes GPU execution. The CPU waits on a fence via `vkWaitForFences` (blocking) or polls via `vkGetFenceStatus`. Fences cannot be used for GPU-to-GPU synchronization — that is the role of `VkSemaphore`. A binary fence is reset to unsignaled via `vkResetFences` before reuse. Crucially, binary `VkFence` is distinct from `VkSemaphore` (GPU-to-GPU sync, binary or timeline) and from EGL's `EGLSyncKHR` (which wraps the same kernel primitive but is exposed through the EGL API). There is no "timeline fence" in Vulkan; timeline semantics are only in `VkSemaphore`.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkSemaphore**, **VkSubmitInfo**, **VkCommandBuffer**, **EGLSyncKHR**

---

### VkFramebuffer
A Vulkan object (`VkFramebuffer`) that binds specific `VkImageView` objects to the attachment slots declared in a `VkRenderPass`. Created with `vkCreateFramebuffer` referencing a `VkRenderPass` and an array of `VkImageView` handles. The framebuffer specifies the actual pixel dimensions (`width`, `height`, `layers`) for the render pass execution. A framebuffer is tied to a specific render pass — it cannot be used with a different pass even if the attachment count and formats happen to match. With `VK_KHR_dynamic_rendering`, `VkFramebuffer` objects are not needed; attachments are specified inline in `VkRenderingInfo`. Framebuffer objects are lightweight and do not hold pixel data — that lives in the bound `VkImage` memory.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkRenderPass**, **VkImageView**, **render pass**, **dynamic rendering (VK_KHR_dynamic_rendering)**

---

### VkImage
A Vulkan object representing a multidimensional array of texels with a specific format (`VkFormat`), extent, mip levels, array layers, and tiling (`VK_IMAGE_TILING_OPTIMAL` or `VK_IMAGE_TILING_LINEAR`). Unlike `VkBuffer`, a `VkImage` has layout state (`VkImageLayout`) that must be transitioned via `vkCmdPipelineBarrier` or render pass implicit transitions before use in different operations (sampling, storage, color attachment, transfer). Optimal tiling uses a driver-defined swizzled layout opaque to the CPU; linear tiling is row-major and CPU-accessible but restricted in capability. `VkImage` objects back swapchain images, render targets, textures, and VA-API/DMA-BUF imports (`VK_EXT_external_memory_dma_buf`).

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch50 — Vulkan Video Extensions  
**Related:** **VkImageView**, **VkBuffer**, **VkFramebuffer**, **VkSampler**, **VMA (Vulkan Memory Allocator)**

---

### VkImageView
A Vulkan object (`VkImageView`) that describes how a `VkImage` is interpreted for a specific use: the `VkImageViewType` (1D, 2D, 3D, cube, array variants), `VkFormat` (may reinterpret the image's format within compatibility classes), component swizzle, and `VkImageSubresourceRange` (selected mip levels and array layers). Shaders and framebuffers operate on `VkImageView`, never directly on `VkImage`. A single `VkImage` can have multiple views — for example, a cube map image may have a `VK_IMAGE_VIEW_TYPE_CUBE` view for sampling and individual `VK_IMAGE_VIEW_TYPE_2D` views per face for render-target use. Views are lightweight objects and do not copy data.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkImage**, **VkFramebuffer**, **VkSampler**, **descriptor set**

---

### VkInstance
The root Vulkan object, created by `vkCreateInstance` with application metadata, requested API version, enabled instance-level layers, and instance-level extensions (`VK_KHR_surface`, `VK_KHR_wayland_surface`, `VK_EXT_debug_utils`). `VkInstance` loads and initializes the Vulkan loader, which enumerates available ICDs (Mesa, NVIDIA proprietary) and layers. All subsequent physical device enumeration (`vkEnumeratePhysicalDevices`) is done through the instance. A process typically creates one `VkInstance`; creating multiple is permitted but uncommon. The instance is distinct from `VkDevice` — instance-level calls deal with enumeration and WSI surface creation, while device-level calls handle rendering.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch12 — The Mesa Loader and Driver Dispatch  
**Related:** **VkPhysicalDevice**, **VkDevice**, **Vulkan loader**, **Vulkan layers**, **vk-bootstrap**

---

### VkPhysicalDevice
A handle (`VkPhysicalDevice`) representing one physical GPU or accelerator visible to the Vulkan instance. Queried via `vkEnumeratePhysicalDevices`; exposes properties (`VkPhysicalDeviceProperties2` — vendor ID, device name, driver version, limits), features (`VkPhysicalDeviceFeatures2` — geometry shaders, descriptor indexing, etc.), memory heaps and types (`VkPhysicalDeviceMemoryProperties`), and queue families (`VkQueueFamilyProperties`). An application inspects `VkPhysicalDevice` to select the best GPU (preferring `VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU`) and to check extension and feature support before creating a `VkDevice`. On a multi-GPU system (PRIME laptop), multiple `VkPhysicalDevice` handles are returned.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch49 — Multi-GPU and PRIME Render Offload  
**Related:** **VkInstance**, **VkDevice**, **vk-bootstrap**

---

### VkPipeline
A Vulkan object (`VkPipeline`) encapsulating the complete GPU execution state for a draw or dispatch: shader stages (`VkShaderModule` + entry point), fixed-function state (vertex input, rasterization, depth/stencil, blending for graphics), pipeline layout (descriptor and push constant interface), and render pass compatibility. Created by `vkCreateGraphicsPipelines` or `vkCreateComputePipelines`; creation may trigger shader compilation and is potentially expensive. Pipelines are immutable after creation and bound via `vkCmdBindPipeline`. The cost of pipeline creation is why `VkPipelineCache`, `VK_EXT_graphics_pipeline_library`, and `VK_EXT_shader_object` exist — each addresses a different aspect of reducing this overhead.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **VkShaderModule**, **pipeline cache**, **VkPipelineCache**, **pipeline layout**, **graphics pipeline library (VK_EXT_graphics_pipeline_library)**, **shader object (VK_EXT_shader_object)**

---

### VkPipelineCache
The Vulkan object (`VkPipelineCache`) passed to `vkCreateGraphicsPipelines` or `vkCreateComputePipelines` to reuse previously compiled pipeline binaries. The cache is opaque to the application; the driver populates it with compiled shader binaries and pipeline state binaries keyed internally. Applications serialize the cache blob (`vkGetPipelineCacheData`) to disk and reload it on subsequent runs. Multiple caches can be merged via `vkMergePipelineCaches`. The cache header contains a driver UUID and device UUID so that stale caches from different drivers or hardware are silently discarded. Mesa's on-disk shader cache (`MESA_SHADER_CACHE_DIR`, default `~/.cache/mesa_shader_cache`) is the backing store for Mesa Vulkan drivers' pipeline cache blobs.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch12 — The Mesa Loader and Driver Dispatch  
**Related:** **pipeline cache**, **VkPipeline**, **graphics pipeline library (VK_EXT_graphics_pipeline_library)**

---

### VkQueue
A Vulkan object representing a stream of GPU work within a queue family (graphics, compute, transfer, video decode, etc.). Queues are retrieved from a `VkDevice` via `vkGetDeviceQueue`; they are not created, only obtained. Work is submitted via `vkQueueSubmit` (Vulkan 1.0/1.1) or `vkQueueSubmit2` (Vulkan 1.3 / `VK_KHR_synchronization2`), passing `VkSubmitInfo` structs that chain command buffers with wait and signal semaphores. A single queue family may expose multiple queues for parallel command submission from different CPU threads; submissions to the same queue are ordered; cross-queue ordering requires semaphores. `vkQueuePresentKHR` is the special submission call for swapchain presentation.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkCommandBuffer**, **VkSubmitInfo**, **VkSemaphore**, **VkFence (binary)**, **VkDevice**

---

### VkRenderPass
The Vulkan object (`VkRenderPass`, created by `vkCreateRenderPass` or `vkCreateRenderPass2`) that formally declares all attachment formats, sample counts, load/store operations, and subpass structure for a rendering operation. It is paired with a `VkFramebuffer` (which provides the actual `VkImageView` bindings) and a `VkPipeline` (which must be compatible with the render pass and subpass index). Render pass creation allows the driver to determine optimal memory layouts and, on tile-based hardware, plan on-chip memory allocation across subpasses. With `VK_KHR_dynamic_rendering`, applications can bypass `VkRenderPass` objects entirely; the tradeoff is potential loss of tile-merge optimizations on mobile hardware.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch16 — Mesa's Vulkan Common Infrastructure  
**Related:** **render pass**, **subpass**, **VkFramebuffer**, **VkImageView**, **dynamic rendering (VK_KHR_dynamic_rendering)**

---

### VkSampler
A Vulkan object (`VkSampler`) encoding texture sampling parameters: magnification/minification filters (`VK_FILTER_LINEAR`, `VK_FILTER_NEAREST`), mipmap mode (`VK_SAMPLER_MIPMAP_MODE_LINEAR`), address modes per axis (`VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE`, etc.), anisotropy level, LOD bias, border color, and optional depth comparison function for shadow samplers. Samplers are created once and reused; they are bound into descriptor sets via `VK_DESCRIPTOR_TYPE_SAMPLER` or `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`. Unlike OpenGL where sampler state is attached to a texture object, Vulkan separates the image data (`VkImage`), view (`VkImageView`), and sampling parameters (`VkSampler`) into three independent objects.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkImageView**, **VkImage**, **descriptor set**, **VkDescriptorPool**

---

### VkSemaphore
A Vulkan GPU-to-GPU synchronization primitive. Binary semaphores (`VkSemaphore` with default `VkSemaphoreType::BINARY`) have signal/wait pairs: one queue submission signals the semaphore, another waits on it before executing — enabling producer/consumer ordering across queues or between rendering and presentation (`vkQueuePresentKHR` waits on the render-complete semaphore). Timeline semaphores (`VK_SEMAPHORE_TYPE_TIMELINE`, Vulkan 1.2) generalize this to a monotonically increasing 64-bit counter: submissions signal to or wait for specific counter values, enabling multi-frame in-flight without per-frame semaphore arrays. Both are distinct from `VkFence`, which is CPU-visible; semaphores are GPU-only. On Linux, timeline semaphores map to kernel DRM sync objects (`drm_syncobj`).

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch3 — Advanced Display Features  
**Related:** **VkFence (binary)**, **VkSubmitInfo**, **VkQueue**, **EGLSyncKHR**, **swapchain**

---

### VkShaderModule
A Vulkan object (`VkShaderModule`) wrapping a SPIR-V binary blob, created by `vkCreateShaderModule` with a pointer to the SPIR-V word stream. The shader module is referenced by name (entry point string, typically `"main"`) and stage (`VkShaderStageFlagBits`) in `VkPipelineShaderStageCreateInfo` during pipeline creation. The driver may or may not compile to ISA at `vkCreateShaderModule` time — this is implementation-defined; actual compilation often defers to `vkCreateGraphicsPipelines`. In Vulkan 1.3+, `VK_EXT_shader_object` supersedes shader modules for many use cases by compiling shaders into standalone `VkShaderEXT` objects. Shader modules are consumed and then typically destroyed after pipeline creation.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch14 — NIR: Mesa's Shader Intermediate Representation  
**Related:** **VkPipeline**, **shader object (VK_EXT_shader_object)**, **SPIR-V cooperative matrix (VK_KHR_cooperative_matrix)**, **pipeline cache**

---

### VkSubmitInfo
The struct (`VkSubmitInfo` for Vulkan 1.0–1.2; `VkSubmitInfo2` for Vulkan 1.3 / `VK_KHR_synchronization2`) passed to `vkQueueSubmit` that packages one or more `VkCommandBuffer` handles with wait semaphore arrays (and their pipeline stage masks) and signal semaphore arrays. With `VkSubmitInfo2`, each semaphore is wrapped in `VkSemaphoreSubmitInfo` which additionally carries a 64-bit value for timeline semaphore use and a `VkPipelineStageFlags2` bitmask with finer granularity than the original `srcStageMask`. Multiple `VkSubmitInfo` structs can be batched in a single `vkQueueSubmit` call for efficiency. The optional `VkFence` parameter of `vkQueueSubmit` is signaled when all command buffers in the batch complete.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkQueue**, **VkCommandBuffer**, **VkSemaphore**, **VkFence (binary)**

---

### VkSwapchainKHR
The Vulkan object (`VkSwapchainKHR`, from `VK_KHR_swapchain`) managing the ring of presentable images between the application and the windowing system. Created by `vkCreateSwapchainKHR` with a `VkSurfaceKHR` (platform-specific surface handle), image count, format, present mode (`VK_PRESENT_MODE_MAILBOX_KHR`, `VK_PRESENT_MODE_FIFO_KHR`), and extent. On Linux/Wayland, Mesa creates GBM-backed DMA-BUF images and shares them with the compositor via `wl_buffer`. The application cycles: `vkAcquireNextImageKHR` (get an available image index) → render → `vkQueuePresentKHR` (return image for display). Present modes determine frame pacing: FIFO is vsync-locked; MAILBOX allows triple-buffering without tearing.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch12 — The Mesa Loader and Driver Dispatch  
**Related:** **swapchain**, **VkSemaphore**, **VkFence (binary)**, **EGL (Embedded/Extensible Graphics Library)**, **VkImage**

---

### VMA (Vulkan Memory Allocator)
An open-source C++ library by AMD GPUOpen ([https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)) that manages `VkDeviceMemory` allocations on behalf of applications, implementing suballocation, memory type selection, and defragmentation. The key convenience is `VMA_MEMORY_USAGE_AUTO` (introduced in VMA 3.0): instead of manually selecting a Vulkan memory type index, the application specifies its usage intent (CPU-to-GPU upload, GPU-only, GPU-to-CPU readback) and VMA selects the optimal heap — using `DEVICE_LOCAL` for pure GPU resources, `HOST_VISIBLE | HOST_COHERENT` for staging buffers, and `HOST_VISIBLE | DEVICE_LOCAL` (ReBAR/SAM) when available. `VMA_MEMORY_USAGE_AUTO` eliminates one of the most error-prone aspects of raw Vulkan.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkBuffer**, **VkImage**, **VkDevice**, **SDL3 GPU API**

---

### VMAF (Video Multi-Method Assessment Fusion)
A perceptual video quality metric developed by Netflix and open-sourced at `github.com/Netflix/vmaf`, trained on human opinion scores (MOS) to predict perceived quality better than PSNR. VMAF combines three elementary metrics—Visual Information Fidelity (VIF), Additive Distortion Metric (ADM/DLM), and temporal Motion—using a Support Vector Machine (SVR) regressor trained on Netflix streaming content. Scores range 0–100; a VMAF score of 93+ is generally considered perceptually transparent at 1080p. On Linux, VMAF is integrated into FFmpeg via `libvmaf` (`-filter_complex vmaf`) and into ffmpeg-quality-metrics tooling. VMAF scores are model-, resolution-, and encoding-profile-dependent: scores should not be compared across different VMAF model versions (e.g. vmaf_v0.6.1 vs. vmaf_4k_v0.6.1 vs. VMAF NEG).

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery  
**Related:** **PSNR (Peak Signal-to-Noise Ratio)**, **CRF (Constant Rate Factor)**, **rate-distortion optimization**, **Pensieve (RL-based ABR)**, **FFmpeg**, **codec**

---

### volk
volk is a meta-loader for the Vulkan API (`https://github.com/zeux/volk`), implemented as a single-header C library that dynamically loads all Vulkan function pointers from the Vulkan loader (`libvulkan.so`) at runtime using `vkGetInstanceProcAddr`/`vkGetDeviceProcAddr`. This avoids linking against the Vulkan loader's stub library, reducing dispatch overhead because calls go directly to the ICD (e.g., Mesa RADV or ANV) without an extra indirection through the loader dispatch table. volk is widely used in game engines (Vulkan Memory Allocator, Sascha Willems' examples) and reduces boilerplate for Vulkan function pointer management. It is distinct from the Vulkan loader itself (`vulkan-loader`/`libvulkan.so`) — volk is a per-application utility, not a system component.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers; Ch40 — Bevy and wgpu — A Rust-Native Vulkan Client  
**Related:** **RADV**, **ANV**, **SPIR-V**, **wgpu (Rust WebGPU/Vulkan)**

---

### VP9
An open, royalty-free video codec developed by Google and published in 2013, used extensively on YouTube and in WebM containers. VP9 uses a superblock structure (up to 64×64 luma samples), multi-reference frame prediction, and a boolean arithmetic coder (a probability-based binary arithmetic coder derived from VP8's entropy coder). VP9 profiles differ by bit depth and chroma subsampling: Profile 0 (8-bit 4:2:0), Profile 1 (8-bit 4:2:2/4:4:4), Profile 2 (10/12-bit 4:2:0), Profile 3 (10/12-bit 4:2:2/4:4:4); VP9 does not use ANS. VP9 does not use NAL units; bitstream framing uses a superframe index at the end of each packet. Software implementations: libvpx (Google's reference encoder/decoder). Hardware decode support on Linux: VA-API (`VAProfileVP9Profile0`), V4L2 M2M (Rockchip, RPi), and NVDEC (Pascal+). VP9 has been largely superseded by AV1 for new deployments but remains widely deployed due to its hardware support breadth. A misconception: VP9 and AV1 are not interchangeable; VP9 bitstreams cannot be decoded by AV1 decoders.

**Chapter(s):** Ch60 — Video Compression Algorithms: DCT, Motion Estimation, and Modern Codecs; Ch26 — Hardware Video  
**Related:** **AV1**, **codec**, **H.264 / AVC**, **H.265 / HEVC**, **entropy coding (CABAC/CAVLC)**, **NVDEC**, **V4L2 (Video4Linux2)**

---

### VRAM
VRAM (Video Random Access Memory) refers to the dedicated on-package or on-card memory directly accessible by the GPU's memory controllers at high bandwidth, distinct from system (CPU) RAM. Modern GPU VRAM uses GDDR6/GDDR6X (discrete consumer GPUs) or HBM2e/HBM3 (data centre GPUs like H100/MI300X); bandwidth ranges from 300 GB/s (GDDR6, RX 6600) to 3.35 TB/s (HBM3, H100 SXM). In Linux kernel drivers, VRAM is managed via TTM (`drivers/gpu/drm/ttm/`) memory heaps; in Mesa, VRAM is the `DEVICE_LOCAL` `VkMemoryHeap` type. Applications should not conflate VRAM capacity with total addressable memory in unified architectures (AMD APU, Apple Silicon) where CPU and GPU share a single LPDDR pool. `nvidia-smi` and `rocm-smi` report VRAM usage; NVML exposes it via `nvmlDeviceGetMemoryInfo`.

**Chapter(s):** Ch4 — GPU Memory Management; Ch25 — GPU Compute; Ch66 — CUDA Runtime  
**Related:** **compute unit (CU)**, **IOMMU group**, **RTXNTC (Neural Texture Compression)**

---

### VRR (Variable Refresh Rate)
A display technology that allows the monitor's refresh rate to vary dynamically between a minimum and maximum value (e.g. 48–144 Hz) in synchrony with the GPU's frame delivery rate, eliminating both tearing (from tearing without V-sync) and latency/stuttering (from V-sync). Standardised by VESA as Adaptive-Sync for DisplayPort and by HDMI Forum for HDMI 2.1 VRR. In the Linux kernel, VRR support is implemented via the `VRR_CAPABLE` connector property (read-only, set from EDID or panel data), the `VRR_ENABLED` CRTC property (set by the compositor to activate VRR), and the `FREESYNC` property on AMD hardware (`drivers/gpu/drm/amd/display/`). The compositor controls VRR per-CRTC; game engines and video players benefit automatically when VRR is active. On NVIDIA hardware under the proprietary driver, VRR for G-Sync Compatible monitors uses the same kernel mechanism.

**Chapter(s):** Ch3 — Advanced Display Features  
**Related:** **FreeSync**, **G-Sync**, **connector (DRM)**, **CRTC**, **vblank**, **DisplayPort**

---

### Vulkan Layers
Software interception layers that sit between the application's Vulkan calls and the Vulkan loader or ICD. Layers are enabled at `VkInstance` creation time (instance layers) or implicitly via environment variables (`VK_INSTANCE_LAYERS`, `VK_LAYER_PATH`). The loader chains layers in order; each layer's `vkGetInstanceProcAddr` wraps the next layer or ICD. Common layers on Linux include `VK_LAYER_KHRONOS_validation` (validation), `VK_LAYER_MESA_overlay` (FPS/GPU stats HUD), `VK_LAYER_MANGOHUD` (MangoHud overlay), and `VK_LAYER_VKBASALT` (post-processing). The layer mechanism is the foundation for debugging, profiling overlays, and API translation shims such as DXVK's `d3d11.dll`-to-Vulkan path.

**Chapter(s):** Ch30 — Debugging & Profiling; Ch29 — Upscaling, Effects & Overlays  
**Related:** **Vulkan loader**, **Vulkan validation layers**, **VkInstance**, **VkDevice**

---

### Vulkan Loader
The shared library (`libvulkan.so.1` on Linux) provided by the Khronos Group and packaged by distributions, responsible for dispatching Vulkan calls to the correct ICD (e.g., `radeon_icd.x86_64.json` for RADV, `nvidia_icd.json` for NVIDIA). The loader reads JSON ICD manifests from `/usr/share/vulkan/icd.d/` and layer manifests from `/usr/share/vulkan/explicit_layer.d/`. It implements `vkGetInstanceProcAddr` and `vkGetDeviceProcAddr`, builds the dispatch table, and chains enabled layers. On multi-GPU systems, the loader enables selecting or filtering physical devices. Applications link against `libvulkan.so` (the loader), not against driver libraries directly, allowing ICD substitution without recompilation.

**Chapter(s):** Ch12 — The Mesa Loader and Driver Dispatch; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VkInstance**, **Vulkan layers**, **Vulkan validation layers**, **vk-bootstrap**

---

### Vulkan Memory Allocator (see VMA)
See **VMA (Vulkan Memory Allocator)**. The library's canonical name in GPUOpen documentation is "Vulkan Memory Allocator"; its header is `vk_mem_alloc.h` and its namespace prefix is `vma` / `VMA_`.

**Chapter(s):** Ch24 — Vulkan and EGL for Application Developers  
**Related:** **VMA (Vulkan Memory Allocator)**, **VkBuffer**, **VkImage**

---

### Vulkan Validation Layers
A specific Vulkan layer (`VK_LAYER_KHRONOS_validation`, provided by the `vulkan-validation-layers` package or the `Vulkan-ValidationLayers` GitHub repository) that inserts comprehensive API usage checking between the application and the driver. The validation layer checks parameter validity, object lifetime, synchronization hazards (via `VK_EXT_synchronization2`), image layout correctness, pipeline layout compatibility, and render pass conformance. Errors are reported via `VK_EXT_debug_utils` callbacks. The validation layer has substantial CPU overhead and must not be enabled in production; it is the first diagnostic tool to enable when debugging Vulkan crashes or incorrect rendering. It is separate from GPU-assisted validation (`VK_VALIDATION_FEATURE_ENABLE_GPU_ASSISTED_EXT`), which catches shader errors at runtime.

**Chapter(s):** Ch30 — Debugging & Profiling; Ch24 — Vulkan and EGL for Application Developers  
**Related:** **Vulkan layers**, **Vulkan loader**, **VkInstance**, **VkDevice**

---

## W

### WARP (NVIDIA Python GPU kernel)
NVIDIA Warp (BSD-3, `pip install warp-lang`, `github.com/NVIDIA/warp`) is a Python framework for GPU-accelerated simulation and spatial computing that JIT-compiles Python-like `@wp.kernel` functions to PTX via LLVM at first invocation. Warp kernels use `wp.tid()` for thread indexing, `wp.atomic_add` for parallel reductions, and typed arrays (`wp.array(dtype=wp.vec3, device="cuda")`) for GPU memory. A key feature is automatic differentiation: kernels decorated with `@wp.kernel` can be differentiated via `wp.Tape` for gradient-based physics simulation and neural rendering. Warp integrates with OpenUSD Fabric via `omni.warp.from_fabric` for zero-copy geometry update in Omniverse. Note: NVIDIA Warp (`warp-lang`) is entirely unrelated to Microsoft's WARP software rasteriser (D3D11 CPU fallback) or to the NVIDIA GPU hardware "warp" (32-thread SIMT group).

**Chapter(s):** Ch69 — NVIDIA Omniverse, OpenUSD, and the RTX Renderer  
**Related:** **CUDA**, **PhysX 5**, **Omniverse (NVIDIA)**, **warp (NVIDIA thread group)**

---

### wavefront (AMD thread group)
A wavefront is AMD's equivalent of NVIDIA's warp — the fundamental SIMT execution unit scheduled to a CU's SIMD array. On GCN architecture, a wavefront consists of 64 lanes (threads) executing in SIMD16 groups across 4 cycles; on RDNA (GCN successor), the default is 32-lane waves (Wave32), matching NVIDIA's warp size, with optional Wave64 mode for compatibility. Each CU on RDNA3 can hold up to 16 wavefronts in flight (8 per SIMD32 unit); switching between wavefronts hides VRAM latency. In Mesa's ACO compiler, the Wave32/Wave64 mode is selected per pipeline (`src/amd/compiler/aco_interface.cpp`) and affects register allocation strategy. In HIP/ROCm, wavefront size is queried via `hipDeviceGetAttribute(hipDeviceAttributeWarpSize, ...)`, returning 32 or 64.

**Chapter(s):** Ch15 — ACO: AMD's Optimising Compiler; Ch25 — GPU Compute; Ch48 — ROCm and ML  
**Related:** **compute unit (CU)**, **warp (NVIDIA thread group)**, **RGP (Radeon GPU Profiler)**, **compute shader**

---

### WebRTC
A suite of W3C/IETF standards enabling real-time peer-to-peer audio, video, and data communication directly between browsers and native applications without a relay server for media. WebRTC's media plane uses RTP/RTCP over DTLS-SRTP (encrypted UDP); the signalling plane is application-defined (commonly WebSocket + SDP exchange). ICE (Interactive Connectivity Establishment) with STUN and TURN servers handles NAT traversal and candidate gathering. Preferred codecs are VP8/VP9/AV1 for video and Opus for audio; H.264 is supported via OpenH264 or hardware paths. On Linux, GStreamer's `webrtcbin` element implements the full WebRTC stack; `libwebrtc` is Google's reference implementation used in Chrome. WebRTC targets sub-300 ms glass-to-glass latency, making it distinct from HLS/DASH (seconds latency) and SRT (hundreds of ms latency).

**Chapter(s):** Ch60b — Video Streaming Protocols and Adaptive Bitrate Delivery; Ch58 — GStreamer: Pipeline-Based Multimedia  
**Related:** **SRT (Secure Reliable Transport)**, **MOQT (Media over QUIC Transport)**, **GStreamer**, **FFmpeg**, **H.264 / AVC**, **AV1**, **VP9**

---

### wgpu (Rust WebGPU/Vulkan)
wgpu is a Rust implementation of the WebGPU API surface, hosted at `https://github.com/gfx-rs/wgpu`, providing safe Vulkan, Metal, D3D12, and OpenGL backends via a cross-platform GPU abstraction. On Linux, wgpu's primary backend uses `ash` (Vulkan FFI bindings) to call Mesa Vulkan drivers (RADV, ANV, NVK). wgpu includes the `naga` shader translator for WGSL → SPIR-V, `wgpu-core` (the portable API implementation), and `wgpu-hal` (the hardware abstraction layer per backend). It is used as the GPU abstraction in Firefox's WebGPU implementation and in the Bevy game engine. wgpu is architecturally parallel to Dawn (Chrome's WebGPU implementation) but written in Rust and not a Khronos or W3C reference implementation. Do not confuse wgpu (the Rust library) with WebGPU (the W3C specification).

**Chapter(s):** Ch40 — Bevy and wgpu — A Rust-Native Vulkan Client; Ch52 — Firefox and WebRender; Ch35 — Dawn and WebGPU  
**Related:** **naga**, **WGSL**, **SPIR-V**, **RADV**, **ANV**, **volk**, **Tint (WGSL compiler)**

---

### WGSL (WebGPU Shading Language)
WGSL (WebGPU Shading Language) is the W3C-specified shading language for the WebGPU API, defined at `https://www.w3.org/TR/WGSL/`. It is a strongly-typed, memory-safe language with no pointer arithmetic, explicit address spaces (`storage`, `uniform`, `workgroup`, `private`, `function`), and mandatory portability — all undefined behaviour present in GLSL/HLSL is eliminated. WGSL is the only shader language accepted by the WebGPU API; browsers compile it to backend ISAs via Tint (Chrome/Dawn) or naga (Firefox/wgpu), which emit SPIR-V for the Vulkan path. WGSL is syntactically distinct from GLSL and HLSL; it uses `fn`, `var`, `let`, `struct` keywords and `@` attributes for binding decorations. WGSL shaders cannot be submitted directly to Mesa — they must first be compiled to SPIR-V.

**Chapter(s):** Ch35 — Dawn and WebGPU; Ch40 — Bevy and wgpu — A Rust-Native Vulkan Client; Ch52 — Firefox and WebRender  
**Related:** **Tint (WGSL compiler)**, **naga**, **SPIR-V**, **GLSL**, **HLSL**, **wgpu (Rust WebGPU/Vulkan)**

---

### wide color gamut (WCG)
A display capability and content characteristic referring to colour gamuts wider than sRGB/BT.709, most commonly DCI-P3 (approximately 25 % more colours than sRGB) or BT.2020 (approximately 75 % of all visible colours). WCG displays advertise their native primaries via EDID extended colour metadata (VCDB / HDR Static Metadata block in CEA-861-H). In KMS, the compositor configures the CTM and LUTs to map application colour spaces into the display's native WCG gamut. For Linux, end-to-end WCG support requires a colour-managed compositor (GNOME Shell with the `wp_color_management_v1` protocol as of 2025), WCG-aware Mesa paths for Vulkan swapchains, and a kernel colour pipeline with sufficient precision (12-bit LUTs). WCG is distinct from HDR: a WCG display can be SDR (DCI-P3 at 400 nits) and a high-luminance display can be sRGB-only.

**Chapter(s):** Ch3 — Advanced Display Features; Ch53 — Display Calibration and colord  
**Related:** **BT.2020**, **sRGB**, **CTM (Color Transform Matrix)**, **HDR10**, **color LUT**, **EDID**

---

### wlroots
A modular C library (`gitlab.freedesktop.org/wlroots/wlroots`) providing compositor infrastructure: DRM/KMS backend (atomic commits, GBM allocation), Vulkan renderer, libinput event handling, protocol implementations (xdg-shell, linux-dmabuf, wp_linux_drm_syncobj_v1, screencopy), and XWayland integration. Compositors such as Sway, Hyprland, river, and labwc are built directly on top of wlroots. The library enforces a clear separation: the backend abstracts hardware (DRM, headless, X11), the renderer draws compositor decorations (via Vulkan or GLES2), and protocol modules handle Wayland wire encoding. wlroots is not a complete compositor by itself; it is a toolkit that requires a compositor author to wire together backends, renderers, and protocol handlers. Version 0.18+ targets Wayland-native explicit sync exclusively.

**Chapter(s):** Ch21 — Building Compositors with wlroots  
**Related:** **wl_compositor**, **zwp_linux_dmabuf_v1**, **Smithay (Rust compositor library)**, **liftoff (KMS composition library)**

---

### wl_buffer
The core Wayland object (`wl_buffer` interface in `wayland.xml`) representing a single frame of pixel data that a client attaches to a `wl_surface` for display. A `wl_buffer` is created from backing memory via allocator-specific factory protocols: `wl_shm_pool.create_buffer()` for shared memory (CPU-accessible), or `zwp_linux_dmabuf_v1.create_immed()` / `zwp_linux_dmabuf_params.create()` for DMA-BUF GPU-resident buffers. After calling `wl_surface.attach(buffer)` and `wl_surface.commit()`, the compositor owns the buffer until it sends `wl_buffer.release`, signalling that the client may reuse it. A `wl_buffer` is a lightweight protocol object; the actual pixel memory is held by the underlying `wl_shm_pool` or DMA-BUF fd, not by the `wl_buffer` itself.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals  
**Related:** **wl_surface**, **zwp_linux_dmabuf_v1**, **wl_compositor**, **wp_linux_drm_syncobj_surface_v1**

---

### wl_compositor
The core Wayland global object (interface `wl_compositor` in `wayland.xml`) that clients use to create `wl_surface` and `wl_region` objects. It is the entry point for surface management: `wl_compositor.create_surface()` returns a new bare surface with no role, and `wl_compositor.create_region()` returns a region used for input or opaque-area hints. `wl_compositor` itself does not perform compositing; it is a factory object. The name is sometimes confused with the compositor application as a whole — the distinction is that `wl_compositor` is the Wayland protocol interface name, while a "Wayland compositor" (e.g., Sway, KWin, Mutter) is the host process implementing the entire display server. Version 5 (Wayland 1.22) added `wl_surface.offset()` for sub-pixel positioning.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals  
**Related:** **wl_surface**, **wl_buffer**, **wlroots**, **xdg_shell**

---

### wl_drm (legacy)
A Mesa-internal Wayland protocol extension (`src/egl/wayland/wayland-drm/wayland-drm.xml`) used before `zwp_linux_dmabuf_v1` was standardised, enabling EGL on Wayland to share DRM buffer objects (GEM handles) as `wl_buffer`s. Clients advertised a DRM device node name (`wl_drm.authenticate`) and created `wl_buffer`s from GEM global names (`wl_drm.create_buffer`). The `wl_drm` protocol is now deprecated; `zwp_linux_dmabuf_v1` supersedes it with proper DMA-BUF file descriptor semantics, support for format modifiers, and multi-plane formats. Mesa still contains `wl_drm` code for compatibility with very old compositors, but all current compositors (wlroots, Mutter, KWin) require `zwp_linux_dmabuf_v1`. Common misconception: `wl_drm` is not a DRM/KMS kernel interface — it is a Wayland protocol only.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals; Ch12 — Mesa WSI and EGL Platform  
**Related:** **zwp_linux_dmabuf_v1**, **wl_buffer**, **wl_compositor**, **wlroots**

---

### wl_surface
The fundamental Wayland object representing a rectangular region of pixels that the compositor places on screen. Created via `wl_compositor.create_surface()`, a `wl_surface` has no inherent role until one is assigned by a higher-level protocol: `xdg_toplevel` for application windows, `xdg_popup` for menus, `wl_subsurface` for video overlays. The `wl_surface` double-buffer commit model means all state changes (`attach`, `damage`, `set_opaque_region`, `set_input_region`, `set_buffer_scale`, `set_buffer_transform`) are accumulated and applied atomically on `wl_surface.commit()`. The compositor signals buffer release via `wl_buffer.release`. Frame synchronisation callbacks are requested with `wl_surface.frame()`, which fires after the compositor has presented the committed buffer.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals  
**Related:** **wl_buffer**, **wl_compositor**, **xdg_toplevel**, **wp_viewport**

---

### wp_linux_drm_syncobj_surface_v1
The per-surface object created by the `wp_linux_drm_syncobj_v1` protocol that attaches DRM timeline sync object acquire and release points to individual `wl_surface` commits. For each commit, the client sets an acquire timeline point (the compositor must wait for this GPU signal before reading the buffer) via `wp_linux_drm_syncobj_surface_v1.set_acquire_point()` and a release timeline point (the compositor signals this when done with the buffer) via `set_release_point()`. This replaces the implicit synchronisation previously embedded in DMA-BUF kernel metadata and is required for correct multi-queue GPU architectures, particularly NVIDIA GPUs under Wayland. It extends the surface-level commit model of `wl_surface` without modifying the core protocol.

**Chapter(s):** Ch75 — Explicit GPU Synchronisation; Ch46 — Wayland Protocol Ecosystem  
**Related:** **linux-drm-syncobj-v1 (Wayland protocol)**, **wl_surface**, **zwp_linux_dmabuf_v1**, **wl_buffer**

---

### wp_viewport
The `wp_viewport` Wayland protocol (wayland-protocols, stable) that allows clients to specify a source crop rectangle and destination size for a `wl_surface`, enabling GPU-side scaling and cropping without requiring the client to resize its buffer. `wp_viewporter.get_viewport(surface)` creates a `wp_viewport` object; `wp_viewport.set_source(x, y, w, h)` defines the sub-rectangle of the buffer to sample (in fixed-point buffer coordinates), and `wp_viewport.set_destination(w, h)` sets the size the compositor should scale the result to in surface-local coordinates. This is used for video playback surfaces (avoiding buffer reallocation on window resize), HiDPI fractional scaling, and cursor scaling. It is distinct from `wl_surface.set_buffer_scale()`, which only handles integer scale factors.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals; Ch46 — Wayland Protocol Ecosystem  
**Related:** **wl_surface**, **wl_buffer**, **wl_compositor**, **zwp_linux_dmabuf_v1**

---

## X

### xdg-desktop-portal
A D-Bus service framework (`github.com/flatpak/xdg-desktop-portal`) that provides sandboxed applications (Flatpak, Snap, containerised apps) with controlled access to compositor and system services via well-defined portal interfaces, without requiring direct Wayland socket or filesystem access. Portals include `org.freedesktop.portal.ScreenCast` (screen capture via PipeWire), `org.freedesktop.portal.RemoteDesktop` (input injection), `org.freedesktop.portal.FileChooser`, and `org.freedesktop.portal.OpenURI`. Backend implementations (e.g., `xdg-desktop-portal-gnome`, `xdg-desktop-portal-wlr`) translate portal D-Bus calls into compositor-specific Wayland protocol calls. The screen-cast portal backend uses `ext-image-copy-capture-v1` or compositor screencopy protocols to deliver DMA-BUF frames into a PipeWire stream consumed by OBS, browsers, or video conferencing apps.

**Chapter(s):** Ch23 — Legacy, Sandboxed App, and XWayland Support; Ch46 — Wayland Protocol Ecosystem  
**Related:** **xdg-portal**, **xdg_shell**, **wl_surface**, **zwp_linux_dmabuf_v1**

---

### xdg-portal
An informal collective name for the `xdg-desktop-portal` system and its companion portal backends (e.g., `xdg-desktop-portal-gnome`, `xdg-desktop-portal-wlr`, `xdg-desktop-portal-kde`). The term is often used interchangeably with `xdg-desktop-portal` but technically refers to the broader ecosystem of D-Bus portal interfaces, their specifications (published at `flatpak.github.io/xdg-desktop-portal`), and the compositor-specific backend implementations. Each portal backend translates abstract D-Bus portal calls into the compositor's native protocol: the `wlr` backend uses `wlr-screencopy-unstable-v1` for screen capture while the GNOME backend uses `meta-screencopy` (transitioning to `ext-image-copy-capture-v1`). The term "xdg-portal" does not refer to the `xdg_shell` Wayland protocol.

**Chapter(s):** Ch23 — Legacy, Sandboxed App, and XWayland Support; Ch46 — Wayland Protocol Ecosystem  
**Related:** **xdg-desktop-portal**, **xdg_shell**, **wlroots**, **Smithay (Rust compositor library)**

---

### xdg_popup
A `xdg-shell` protocol role (`xdg_popup` interface in `xdg-shell.xml`) for transient, short-lived surfaces such as right-click context menus, tooltips, and dropdown menus. An `xdg_popup` must be created with a parent surface (either `xdg_toplevel` or another `xdg_popup`) and a `xdg_positioner` describing the desired placement relative to that parent's geometry, with gravity, constraint adjustments, and anchor point. The compositor may reposition the popup to keep it on-screen (constraint adjustment), and must dismiss all popups in a grab chain when the user clicks outside them (via `xdg_popup.dismiss`). Popups must be destroyed in reverse creation order; attempting to destroy a parent popup before its children is a protocol error.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals  
**Related:** **xdg_shell**, **xdg_toplevel**, **wl_surface**, **wl_compositor**

---

### xdg_shell
The Wayland protocol extension (`xdg-shell.xml`, wayland-protocols stable) that defines the standard window management roles for application surfaces. It provides `xdg_wm_base` as the global factory object, `xdg_surface` as a role-capable wrapper around a `wl_surface`, and two concrete roles: `xdg_toplevel` (application windows) and `xdg_popup` (transient menus/tooltips). `xdg_shell` replaced the older `wl_shell` protocol (removed from wlroots 0.15+). The protocol defines a configure/ack loop: the compositor sends `xdg_toplevel.configure()` with suggested size and state (maximised, fullscreen, activated), the client must respond with `xdg_surface.ack_configure()` before committing its next buffer at that size, ensuring atomic size transitions. Applications must implement this loop correctly to avoid protocol errors.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals  
**Related:** **xdg_toplevel**, **xdg_popup**, **wl_surface**, **wl_compositor**

---

### xdg_toplevel
The `xdg-shell` role (interface `xdg_toplevel` in `xdg-shell.xml`) for standard application windows that participate in the compositor's window management. `xdg_toplevel` exposes requests for setting title, app-id, minimum/maximum size, and for initiating interactive move/resize (delegating pointer grab handling to the compositor). Server decorations may be negotiated via the `zxdg_decoration_manager_v1` protocol (KWin, Mutter) or handled by the client (CSD). The compositor sends `configure` events with proposed size, state flags (`maximized`, `fullscreen`, `resizing`, `activated`, `tiled_*`), and capabilities; the client acknowledges with `ack_configure` and commits a buffer at the negotiated size. `xdg_toplevel.wm_capabilities` (added in xdg-shell version 5) allows the compositor to advertise which window management operations it supports.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals  
**Related:** **xdg_shell**, **xdg_popup**, **wl_surface**, **xdg-desktop-portal**

---

### XeSS (Intel Xe Super Sampling)
XeSS (Xe Super Sampling) is Intel's AI-accelerated temporal upscaling technology that reconstructs a high-resolution output from sub-native rendered frames using a transformer-based neural network. XeSS has two runtime paths: `XeSS DP4a` (dot product accumulate 4a) for non-Intel hardware (any GPU supporting `VK_KHR_shader_integer_dot_product`) using INT8 inference, and `XeSS XMX` for Intel Arc GPUs leveraging the dedicated Xe Matrix Extensions for higher-quality INT8 and BF16 inference. The Vulkan integration uses the `libxess.so` library: `xessD3D12CreateContext`/`xessVKCreateContext`, `xessGetInputResolution`, and `xessExecute`. XeSS inputs are the same as FSR/DLSS: colour buffer, depth, motion vectors, exposure; jitter is applied using Halton sequences via `xessSetJitterScale`. XeSS is open-source for the integration layer but the neural network weights are proprietary.

**Chapter(s):** Ch71 — Intel Xe and Arc  
**Related:** **tensor core / XMX**, **Level Zero (Intel)**, **DLSS 4**, **FSR 4**, **Frame Generation (NVIDIA)**

---

## Z

### zwp_linux_dmabuf_v1
The `zwp_linux_dmabuf_v1` Wayland protocol extension (wayland-protocols, stable) that allows clients to create `wl_buffer` objects backed by Linux DMA-BUF file descriptors, enabling zero-copy sharing of GPU-resident buffers between clients and the compositor. Clients enumerate supported formats and DRM format modifiers via `zwp_linux_dmabuf_v1.get_default_feedback()` or per-surface feedback (`get_surface_feedback()`), then create buffers with `zwp_linux_dmabuf_params.create()`, specifying fd, offset, stride, and modifier per plane. The compositor imports the DMA-BUF into its GPU as a texture without CPU copies. Version 4 (wayland-protocols 1.31) added the format-modifier feedback mechanism, replacing the older per-format event enumeration. This protocol supersedes the legacy `wl_drm` extension.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals; Ch46 — Wayland Protocol Ecosystem  
**Related:** **wl_buffer**, **wl_surface**, **wl_drm (legacy)**, **linux-drm-syncobj-v1 (Wayland protocol)**

---

### zwp_relative_pointer_v1
The `zwp_relative_pointer_v1` Wayland protocol extension (wayland-protocols, stable) that delivers raw, unaccelerated pointer motion deltas to a client, independent of the pointer's position on screen. It is used alongside `zwp_pointer_constraints_v1` (specifically `zwp_locked_pointer_v1`) for first-person game and 3D viewport navigation: the pointer is locked in place while the application receives `zwp_relative_pointer_v1.relative_motion` events carrying `dx`, `dy`, `dx_unaccel`, and `dy_unaccel` values in wl_fixed_t format (signed 24.8 fixed-point). `dx`/`dy` carry libinput-accelerated deltas; `dx_unaccel`/`dy_unaccel` carry raw hardware deltas before pointer acceleration. A client must hold a `wl_pointer` seat capability to create a `zwp_relative_pointer_v1` object via the factory global `zwp_relative_pointer_manager_v1`.

**Chapter(s):** Ch20 — Wayland Protocol Fundamentals; Ch54 — Linux Input Stack  
**Related:** **wl_compositor**, **wl_surface**, **wlroots**, **Gamescope**

---

