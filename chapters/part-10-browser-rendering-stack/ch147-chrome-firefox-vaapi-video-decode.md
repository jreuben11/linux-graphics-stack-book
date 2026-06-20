# Chapter 147: Chrome and Firefox Hardware Video Decode via VA-API

**Part**: X — The Browser Rendering Stack  
**Primary audience**: Browser and web platform engineers who need to understand how Chrome and Firefox offload video decoding to the GPU via VA-API on Linux; systems and driver developers debugging VA-API integration, sandbox policies, or zero-copy NV12 compositing failures in browser environments.

---

## Table of Contents

1. [Why Hardware Video Decode Matters](#1-why-hardware-video-decode-matters)
2. [The VA-API Foundation](#2-the-va-api-foundation)
3. [Chrome's VA-API Architecture — Two Generations](#3-chromes-va-api-architecture--two-generations)
4. [VaapiVideoDecoder in Depth](#4-vaapivideodecoder-in-depth)
5. [Zero-Copy Path: VASurface to the Compositor](#5-zero-copy-path-vasurface-to-the-compositor)
6. [Out-of-Process Video Decode (OOP-VD)](#6-out-of-process-video-decode-oop-vd)
7. [Chrome on Wayland: DMA-BUF and linux-dmabuf](#7-chrome-on-wayland-dma-buf-and-linux-dmabuf)
8. [Firefox's VA-API Architecture](#8-firefoxs-va-api-architecture)
9. [Firefox VA-API in Depth](#9-firefox-va-api-in-depth)
10. [Codec Coverage: AV1, HEVC, VP9, and 10-Bit Formats](#10-codec-coverage-av1-hevc-vp9-and-10-bit-formats)
11. [NVIDIA: No Native VA-API in Chrome, nvidia-vaapi-driver for Firefox](#11-nvidia-no-native-va-api-in-chrome-nvidia-vaapi-driver-for-firefox)
12. [Verification and Debugging](#12-verification-and-debugging)
13. [Practical Configuration](#13-practical-configuration)
14. [Integrations](#14-integrations)
15. [References](#15-references)

---

## 1. Why Hardware Video Decode Matters

The scale argument is straightforward: decoding 4K H.264 in software saturates multiple CPU cores on a modern x86-64 processor, while the same stream decoded in fixed-function GPU silicon — available on virtually every Intel, AMD, and NVIDIA chip shipped since around 2017 — consumes a fraction of the power through dedicated video engine hardware. Multiply across the web as a distribution medium and the savings in carbon, battery life, and thermal headroom are significant. For a laptop user streaming 4K YouTube, the difference between software and hardware video decode can account for a substantial portion of battery runtime. Note: specific power and battery figures vary widely by SoC generation and workload; benchmark measurements from platform-specific sources (Intel Power Gadget, AMD µProf) should be consulted for precise values.

Beyond power, there is a latency and quality argument. Hardware video engines often run at higher bit depths and offer more precise deblocking and film grain synthesis than software paths at equivalent CPU budgets. AV1 film grain synthesis in particular — a randomised, seed-driven per-frame operation required for perceptual fidelity — was explicitly designed with GPU execution in mind; the AV1 bitstream carries dedicated film grain parameter syntax elements. Software AV1 decoding at 4K resolution still challenges even high-end desktop CPUs at playback rate.

The Linux browser story for hardware video decode has a complicated timeline. Both Chrome and Firefox shipped Linux VA-API support substantially later than their Windows and macOS equivalents, and the path was littered with driver compatibility regressions, sandbox policy gaps, and zero-copy infrastructure that simply did not exist until Wayland adoption matured. The major milestones:

- **2021 (Chrome 91)**: `--enable-features=VaapiVideoDecoder` flag added for experimental VA-API decode; only worked reliably with Intel `iHD` driver.
- **2022 (Firefox 101)**: VA-API hardware decode enabled by default for Intel and AMD hardware in Firefox.
- **2023 (Firefox 115)**: Intel GPU VA-API explicitly enabled as a stable ESR milestone.
- **2024 (Chrome 131/132)**: Feature flag renamed from `VaapiVideoDecodeLinuxGL` to `AcceleratedVideoDecodeLinuxGL`; enabled by default for Linux GL configurations. HEVC 10-bit (`VAProfileHEVCMain10`) and VP9 Profile 2 10-bit (`P010` format) support landed.
- **2025 (Chrome 143)**: Hardware video decode enabled by default on Wayland (the `AcceleratedVideoDecoder` feature). Firefox 136 extended AMD GPU VA-API to default-on status.
- **2025 (Mesa 26.1)**: Unified video decode implementation shared between RadeonSI (Gallium/VA-API) and RADV (Vulkan Video), both backed by AMD VCN hardware.

The late arrival of default-on hardware decode in Chrome on Linux relative to Windows is primarily attributable to three factors: the absence of a stable zero-copy path between VA-API surfaces and the Wayland compositor before `zwp_linux_dmabuf_v1` was widely deployed; incomplete OOP-VD sandbox policies for AMD and NVIDIA hardware; and the incompatibility of the `i965` Intel driver — which powered the original VA-API path — with the Vulkan compositing stack Chrome subsequently adopted.

Thermal throttling is a less visible but practically important reason organisations care about this at scale. A server-side transcoding or viewing cluster that soft-decodes high-resolution streams will throttle CPU frequency under sustained load, causing decode frame-rate drops that manifest as buffering stalls even at adequate network bandwidth. On client laptops, sustained CPU decode at 4K creates enough heat to trigger fan activity, reducing user experience. The industry's shift from H.264 to VP9 (YouTube 2015) and then to AV1 (YouTube/Netflix 2018–2020) has progressively increased decode complexity with each codec generation. AV1 is substantially more computationally demanding to decode in software than H.264 at the same resolution; the exact multiple depends on implementation and hardware, but the difference is large enough that smooth software AV1 4K playback at 60fps is impractical on battery-powered devices, making hardware acceleration necessary rather than optional.

---

## 2. The VA-API Foundation

VA-API (Video Acceleration API) is a vendor-neutral C API defined by Intel and standardised via the `libva` library. It exposes fixed-function video decode and encode engines on GPU hardware as a pipeline of:

1. A **profile** (`VAProfile`) — the codec and profile level, such as `VAProfileH264High`.
2. An **entrypoint** (`VAEntrypoint`) — the operation type: `VAEntrypointVLD` for variable-length decode, `VAEntrypointEncSlice` for encode, `VAEntrypointVideoProc` for post-processing.
3. A **context** (`VAContextID`) — a hardware decode session bound to a display, config, and surface pool.
4. **Buffers** (`VABufferID`) — typed data blobs carrying picture parameters, IQ matrices, slice parameters, and compressed bitstream data.
5. **Surfaces** (`VASurfaceID`) — GPU-allocated output buffers in a tiled, vendor-specific format (NV12 for 8-bit YUV 4:2:0, P010 for 10-bit).

[Source: `va/va.h`, libva, https://github.com/intel/libva/blob/master/va/va.h]

The minimal decode sequence for a single H.264 frame looks like this:

```c
/* media/gpu/vaapi/vaapi_wrapper.cc — simplified decode sequence */

// 1. Query driver support and create config
VAStatus status = vaCreateConfig(va_display,
    VAProfileH264High, VAEntrypointVLD,
    NULL, 0, &va_config_id);

// 2. Allocate a pool of output surfaces
vaCreateSurfaces(va_display, VA_RT_FORMAT_YUV420,
    coded_width, coded_height,
    surfaces, num_surfaces, NULL, 0);

// 3. Create a decode context bound to those surfaces
vaCreateContext(va_display, va_config_id,
    coded_width, coded_height, VA_PROGRESSIVE,
    surfaces, num_surfaces, &va_context_id);

// 4. Per-frame decode:
vaBeginPicture(va_display, va_context_id, surfaces[current_idx]);

// Submit picture parameters (H.264: VAPictureParameterBufferH264)
vaCreateBuffer(va_display, va_context_id,
    VAPictureParameterBufferType,
    sizeof(VAPictureParameterBufferH264), 1,
    &pic_param, &pic_param_buf);
vaRenderPicture(va_display, va_context_id, &pic_param_buf, 1);

// Submit IQ matrix
vaRenderPicture(va_display, va_context_id, &iq_matrix_buf, 1);

// Submit each slice (params + compressed data)
vaRenderPicture(va_display, va_context_id, &slice_param_buf, 1);
vaRenderPicture(va_display, va_context_id, &slice_data_buf, 1);

vaEndPicture(va_display, va_context_id);

// Wait for hardware completion
vaSyncSurface(va_display, surfaces[current_idx]);

// 5. Export decoded surface as DMA-BUF for the compositor
VADRMPRIMESurfaceDescriptor desc;
vaExportSurfaceHandle(va_display, surfaces[current_idx],
    VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2,
    VA_EXPORT_SURFACE_SEPARATE_LAYERS | VA_EXPORT_SURFACE_READ_ONLY,
    &desc);
// desc.objects[0].fd  → luma DRM BO file descriptor
// desc.objects[1].fd  → chroma DRM BO file descriptor (AMD disjoint case)
// desc.layers[0].drm_format → DRM_FORMAT_R8  (Y plane, 8-bit)
// desc.layers[1].drm_format → DRM_FORMAT_GR88 (UV plane, 8-bit NV12)
```

[Source: `va/va_drmcommon.h`, libva, https://github.com/intel/libva/blob/master/va/va_drmcommon.h]

The `VADRMPRIMESurfaceDescriptor` structure is the bridge between the VA-API world and the DRM/DMA-BUF world. When `VA_EXPORT_SURFACE_SEPARATE_LAYERS` is requested on an NV12 surface, the descriptor returns one DRM object per plane: layer 0 is `DRM_FORMAT_R8` (luma), layer 1 is `DRM_FORMAT_GR88` (chroma). This is the data structure that both Chrome and Firefox use to import a decoded frame into their respective compositor pipelines without a CPU-side copy.

There are two sets of VA-API driver backends relevant to Linux browsers:

- **Intel iHD** (`libva-intel-media-driver`, `intel-media-driver` package) — the current Intel VA-API driver for Gen 9 (Skylake) through Xe. Exposes `VAProfileAV1Profile0` on Gen12+ (Tiger Lake, Rocket Lake, Alder Lake, and newer). Required by Chrome's `VaapiVideoDecoder`; the legacy `i965` driver is no longer supported by Chrome's active decode path.
- **AMD RadeonSI/Gallium** — Mesa's `src/gallium/frontends/va/` implements VA-API via the gallium frontend. Set `LIBVA_DRIVER_NAME=radeonsi`. As of Mesa 26.1, RadeonSI and RADV share a unified VCN hardware decode implementation for H.264, H.265, VP9, and AV1. [Source: Mesa 26.1 release notes, https://docs.mesa3d.org/relnotes/26.1.0.html]

Driver selection is controlled by the `LIBVA_DRIVER_NAME` environment variable. If unset, `libva` queries the DRM device for the PCI vendor ID and selects a default driver name. Both Chrome and Firefox open the DRM render node (`/dev/dri/renderD128`) to obtain the `VADisplay` rather than using the X11 display path, ensuring compatibility on Wayland where no X display is available.

---

## 3. Chrome's VA-API Architecture — Two Generations

Chrome's Linux VA-API story spans two distinct architectures, separated by a significant refactor completed around Chrome 116 (mid-2023).

### Generation 1: VaapiVideoDecodeAccelerator (VAVDA)

The original VA-API path in Chrome was `VaapiVideoDecodeAccelerator` (`media/gpu/vaapi/vaapi_video_decode_accelerator.h`), which implemented the `media::GpuVideoDecodeAccelerator` interface. VAVDA had several structural problems:

- It depended on `VABufferInfo` API exposed only by the Intel `i965` driver, not by `iHD` or Mesa drivers.
- It ran inside the **GPU process**, sharing the GPU process's sandbox with the GL compositor.
- It did not interoperate with the Vulkan compositing path that Chrome adopted from approximately Chrome 92 onward.
- It performed a copy for AMD surfaces because it assumed single-BO NV12 layout (Intel layout), failing silently for AMD's disjoint two-BO NV12.

VAVDA was **removed from the Chromium codebase around Chrome 116**. Any documentation or tutorials referencing `VaapiVideoDecodeAccelerator` or the `i965` Intel driver requirement are describing this removed architecture. The file `media/gpu/vaapi/vaapi_video_decode_accelerator.cc` no longer exists in the main branch.

### Generation 2: VaapiVideoDecoder (Current)

The current architecture centres on `VaapiVideoDecoder`, which implements `media::VideoDecoderMixin`. This class:

- Runs in the dedicated **video utility process** under OOP-VD (see Section 6).
- Delegates bitstream parsing to per-codec `AcceleratedVideoDecoder` subclasses.
- Interfaces with `libva` exclusively through `VaapiWrapper`.
- Exports decoded surfaces as DMA-BUF and communicates them to the GPU process via Mojo, backed by `OzoneImageBacking`.

The feature flag history for this generation:
- Chrome 91–130: `--enable-features=VaapiVideoDecoder` (experimental).
- Chrome 131: Flag renamed to `AcceleratedVideoDecodeLinuxGL`.
- Chrome 132: Enabled by default on Linux with GL compositing.
- Chrome 143: `AcceleratedVideoDecoder` enabled by default on Wayland.

A separate flag, `AcceleratedVideoDecodeLinuxZeroCopyGL`, gates the zero-copy path where the decoded NV12 is imported as an EGLImage without a CPU round-trip. When the zero-copy flag is disabled, Chrome falls back to uploading the decoded plane data via shared memory — correct but expensive. As of Chrome 143 on Wayland, both flags are enabled by default.

The `chrome://gpu` page's "Video Acceleration Information" section lists every VA-API profile Chrome detected at startup. The `chrome://media-internals` page's `video_decoder` field for a playing video will show `"VaapiVideoDecoder"` for hardware decode or `"FFmpegVideoDecoder"` for software fallback — the fastest way to confirm hardware decode is active.

---

## 4. VaapiVideoDecoder in Depth

### VaapiWrapper: The libva Binding Layer

`VaapiWrapper` (`media/gpu/vaapi/vaapi_wrapper.h`) is Chrome's encapsulation of the `libva` C API. It is responsible for:

- Obtaining the `VADisplay` via `vaGetDisplayDRM(drm_fd)`, where `drm_fd` is a file descriptor for `/dev/dri/renderD128` (or render nodes 129–137 if the first is unavailable).
- Managing the lifecycle of `VAConfigID`, `VAContextID`, and `VASurfaceID` objects.
- Submitting `VABuffer` objects to hardware via `vaRenderPicture()` and flushing with `vaEndPicture()`.
- Exporting decoded surfaces via `vaExportSurfaceHandle()` to produce `VADRMPRIMESurfaceDescriptor`.
- Implementing driver-specific workarounds — for example, AMD Stoney Ridge has surface format limitations that require special-casing.

```cpp
// media/gpu/vaapi/vaapi_wrapper.cc (simplified VADisplay acquisition)

// VADisplayStateHandle — singleton, reference-counted
VADisplay va_display = vaGetDisplayDRM(drm_fd_.get());
// drm_fd_ is opened from /dev/dri/renderD128
// libva 2.x: vaGetDisplayDRM returns the display without opening X11
if (!vaDisplayIsValid(va_display))
    return false;
int major, minor;
vaInitialize(va_display, &major, &minor);
```

[Source: `media/gpu/vaapi/vaapi_wrapper.cc`, Chromium, https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_wrapper.cc]

`VaapiWrapper` exposes a factory method `CreateForVideoCodec(profile, ...)` that takes a `media::VideoCodecProfile`, maps it to the corresponding `VAProfile`, and creates a new wrapper bound to that profile's decode config. It performs profile availability checking via `vaQueryConfigEntrypoints()` at creation time and returns `nullptr` if the driver does not support the requested profile.

`ExportVASurfaceAsNativePixmapDmaBuf()` is the key output method — it calls `vaExportSurfaceHandle()` with `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` and wraps the resulting `VADRMPRIMESurfaceDescriptor` into a `NativePixmapDmaBuf` (a Chrome abstraction over a multi-plane DMA-BUF). This `NativePixmapDmaBuf` is then passed to the `OzoneImageBacking` system.

### Codec-Specific Delegates

Each supported codec has a delegate class that implements `AcceleratedVideoDecoder` and is responsible for parsing the codec's bitstream headers and building the appropriate VA-API buffer structures:

- `H264VaapiVideoDecoderDelegate` (`media/gpu/vaapi/h264_vaapi_video_decoder_delegate.cc`) — fills `VAPictureParameterBufferH264`, `VAIQMatrixBufferH264`, and `VASliceParameterBufferH264`. It maintains a 16-entry reference picture buffer: `pic_param.ReferenceFrames[16]`.
- `VP9VaapiVideoDecoderDelegate` — supports VP9 Profile 0 (8-bit) and Profile 2 (10-bit): profile selection condition is `(profile == 0 && bit_depth == 8) || (profile == 2 && bit_depth == 10)`.
- `AV1VaapiVideoDecoderDelegate` (`media/gpu/vaapi/av1_vaapi_video_decoder_delegate.cc`) — fills `VADecPictureParameterBufferAV1` via `FillAV1PictureParameter()`, segment information via `FillSegmentInfo()`, film grain parameters via `FillFilmGrainInfo()`, and submits tile group data via `FillAV1SliceParameters()`.
- `VP8VaapiVideoDecoderDelegate` — H.264-era codec still used by some WebRTC streams.
- `HEVCVaapiVideoDecoderDelegate` — for `VAProfileHEVCMain` and `VAProfileHEVCMain10` on Intel iHD hardware.

[Source: `media/gpu/vaapi/`, Chromium, https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/]

The decode submission pattern, illustrated for H.264, is: `SubmitBuffer(VAPictureParameterBufferType, ...)` → `SubmitBuffer(VAIQMatrixBufferType, ...)` → for each slice: `SubmitBuffer(VASliceParameterBufferType, ...)` + `SubmitBuffer(VASliceDataBufferType, ...)` → `ExecuteAndDestroyPendingBuffers()` (which calls `vaEndPicture()`). This corresponds exactly to the libva decode sequence shown in Section 2.

### State Machine and Surface Lifecycle

`VaapiVideoDecoder` runs a state machine with four states: `kUninitialized`, `kWaitingForInput`, `kDecoding`, and `kError`. Decode tasks are queued as `base::queue<DecodeTask>` entries. An internal `output_frames_` map tracks the lifetime relationship between `VASurface` objects and their corresponding `VideoFrame` wrappers, ensuring that the `VASurface` is not recycled to the pool while a `VideoFrame` backed by it is still live in the compositor pipeline.

---

## 5. Zero-Copy Path: VASurface to the Compositor

The "zero-copy" claim in hardware video decode means the decoded pixels never pass through the CPU between the video engine and the display controller. The chain is:

```
[VA-API video engine]
       ↓ VASurface (tiled GPU memory, vendor-specific layout)
       ↓ vaExportSurfaceHandle() → VADRMPRIMESurfaceDescriptor
       ↓ DMA-BUF file descriptor(s) (one per plane, or one shared)
       ↓ NativePixmapDmaBuf (Chrome abstraction)
       ↓ OzoneImageBacking (SharedImage backed by the NativePixmap)
       ↓ EGLImage via EGL_EXT_image_dma_buf_import
       ↓ GL texture / Vulkan VkImage (in GPU process compositor)
       ↓ KMS scanout or Wayland compositor
```

### OzoneImageBacking

`OzoneImageBacking` (`gpu/command_buffer/service/shared_image/ozone_image_backing.h`) is described in its source comment as: *"Implementation of SharedImageBacking that uses a NativePixmap created via an Ozone surface factory. The memory associated with the pixmap can be aliased by both GL and Vulkan."*

[Source: `gpu/command_buffer/service/shared_image/ozone_image_backing.h`, Chromium, https://source.chromium.org/chromium/chromium/src/+/main:gpu/command_buffer/service/shared_image/ozone_image_backing.h]

`OzoneImageBacking::BeginAccess()` and `EndAccess()` implement the synchronisation contract: the compositor calls `BeginAccess()` with a fence that must be signalled before reading, and calls `EndAccess()` to release the backing back to the video decoder. This maps to kernel `dma_fence` implicit synchronisation between the VA-API producer and the EGL/Vulkan consumer — the kernel ensures ordering without requiring explicit semaphore signals from user space.

On Wayland, the Ozone platform factory calls `WaylandSurfaceFactory::CreateNativePixmapFromHandle()` to wrap the DMA-BUF into a Wayland-presentable pixmap. On X11, `X11SurfaceFactory::CreateNativePixmapFromHandle()` is used instead.

### Intel vs. AMD Plane Layout Difference

This is a subtle but historically important incompatibility. Intel `iHD` exports NV12 as a **single DRM buffer object** with the luma plane at offset 0 and the chroma plane at a computed offset within the same fd. The `VADRMPRIMESurfaceDescriptor` has `num_objects == 1`.

AMD Mesa exports NV12 as **two disjoint DRM buffer objects** — separate fds for luma and chroma. The `VADRMPRIMESurfaceDescriptor` has `num_objects == 2`. Chrome's original zero-copy path assumed single-BO layout and silently fell back to a copy for AMD. A patch in Chrome's Vulkan backend added disjoint support by using `VK_IMAGE_CREATE_DISJOINT_BIT` with per-plane `vkBindImageMemory2()` calls — treating each DMA-BUF as a separate `VkDeviceMemory` import. This is why hardware decode zero-copy worked on Intel years before it worked reliably on AMD.

[Source: `VA_EXPORT_SURFACE_SEPARATE_LAYERS` documentation, `va/va_drmcommon.h`, https://github.com/intel/libva/blob/master/va/va_drmcommon.h]

### DMABuf Video Frame Mapper

For the GL (non-Vulkan) compositor path, `VaapiDmabufVideoFrameMapper` (`media/gpu/vaapi/vaapi_dmabuf_video_frame_mapper.cc`) implements the import:

```cpp
// media/gpu/vaapi/vaapi_dmabuf_video_frame_mapper.cc (simplified)
// "All the planes are stored in the same buffer, VAImage.va_buffer"
// (comment from source, applies to Intel single-BO case)

// Creates an EGLImage from DMA-BUF fds using EGL_EXT_image_dma_buf_import:
// EGL_DMA_BUF_PLANE0_FD_EXT  → luma fd
// EGL_DMA_BUF_PLANE0_OFFSET_EXT → 0
// EGL_DMA_BUF_PLANE0_PITCH_EXT  → luma stride
// EGL_DMA_BUF_PLANE1_FD_EXT  → chroma fd (same fd as luma on Intel)
// EGL_DMA_BUF_PLANE1_OFFSET_EXT → chroma plane offset
// EGL_DMA_BUF_PLANE1_PITCH_EXT  → chroma stride
```

The result is wrapped via `WrapExternalYuvDataWithLayout()` into a `VideoFrame` that references the EGLImage. A destruction observer ensures the source `VASurface` remains live until the mapped frame is destroyed.

---

## 6. Out-of-Process Video Decode (OOP-VD)

### Architecture

OOP-VD moves the VA-API decode logic out of the GPU process into a dedicated **video utility process**. The key motivation was isolation: a driver crash inside a video codec path should not kill the GPU process (which also hosts the compositor). A secondary motivation was sandboxing: the video utility process can be given exactly the permissions required for VA-API decode — access to the DRM render node, the ability to allocate GEM buffers, the ability to mmap decoded surfaces — without the broader permissions the GPU compositor requires.

The video utility process runs `VaapiVideoDecoder` directly. The decoded `NativePixmapDmaBuf` — carrying one or two DMA-BUF file descriptors — is transferred to the GPU process via Mojo. The GPU process then creates an `OzoneImageBacking` from the imported `NativePixmap` and makes it available to the Viz compositor as a `SharedImage` mailbox.

[Source: `media/mojo/services/gpu_mojo_media_client.cc`, Chromium, https://source.chromium.org/chromium/chromium/src/+/main:media/mojo/services/gpu_mojo_media_client.cc]

### Sandbox Policy

`media/gpu/sandbox/hardware_video_decoding_sandbox_hook_linux.cc` implements the OOP-VD sandbox policy. Key details:

```cpp
// media/gpu/sandbox/hardware_video_decoding_sandbox_hook_linux.cc

// Allows /dev/dri/renderD128 through /dev/dri/renderD137
void AllowAccessToRenderNodes(BrokerFilePermission* permissions) {
    // Intel VA-API: read-only access is sufficient
    // AMD VA-API: read-write access required, plus preload of
    //   libvulkan_radeon.so and the Vulkan ICD JSON file
    // V4L2: /dev/video-dec* and /dev/media-dec*
}
```

Intel VA-API requires only read-only render node access (`COMMAND_OPEN | COMMAND_STAT | COMMAND_READLINK`). AMD VA-API requires **read-write** render node access and additionally preloads `libvulkan_radeon.so` because the Mesa radeonsi VA-API driver internally uses Vulkan for some operations.

A notable FD limit increase: commit `696a8f7d22bf` raised the video utility process's file descriptor limit to 8192, with the commit message explaining: *"video decoding of many video streams can use thousands of FDs"* — each concurrent 4K stream can consume hundreds of DMA-BUF file descriptors.

[Source: `media/gpu/sandbox/hardware_video_decoding_sandbox_hook_linux.cc`, Chromium, https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/sandbox/hardware_video_decoding_sandbox_hook_linux.cc]

### Process Lifecycle

`GpuMojoMediaClient::CreateVideoDecoder()` in `media/mojo/services/gpu_mojo_media_client.cc` is guarded by `#if BUILDFLAG(ALLOW_OOP_VIDEO_DECODER)`. When the OOP-VD build flag is active and hardware decode is available, `CreatePlatformVideoDecoder()` selects `VaapiVideoDecoder`. The IPC channel between the video utility process and the GPU process uses `GpuChannel` + `MediaGpuChannelManager`, with Mojo interfaces carrying DMA-BUF fds as `mojo::PlatformHandle` wrappers.

The lifetime model for a decoded NV12 surface is: the video utility process holds the `VASurface` alive via `VaapiVideoDecoder`'s `output_frames_` map; it exports the surface as a `NativePixmapDmaBuf`; the DMA-BUF fds are sent to the GPU process via Mojo; the GPU process creates an `OzoneImageBacking` referencing those fds; the `OzoneImageBacking` holds a reference to the `NativePixmapDmaBuf`; when the GPU process's compositor is done with the frame, it calls `EndAccess()` and releases its reference; when the `NativePixmapDmaBuf` refcount drops to zero, the DMA-BUF fds are closed and the video utility process is notified that the `VASurface` may be returned to the surface pool. This lifecycle is managed without explicit reference counting messages — the kernel's DMA-BUF reference counting (tracked by the `dma_buf` struct in the kernel) handles it automatically: the surface's underlying GEM buffer object is freed only when all fd references across all processes are closed.

---

## 7. Chrome on Wayland: DMA-BUF and linux-dmabuf

When Chrome runs with `--ozone-platform=wayland` (the default on most Wayland desktops since Chrome 108+), the decoded NV12 surface follows a display path that differs from the X11 EGLImage path:

1. `VaapiVideoDecoder` exports the `VASurface` as a `NativePixmapDmaBuf` (two DMA-BUF fds for NV12).
2. An `OzoneImageBacking` is created in the GPU process, backed by the `NativePixmap`.
3. The Viz compositor selects this NV12 SharedImage as an overlay candidate (video overlay promotion).
4. The Ozone/Wayland backend calls `WaylandSurfaceFactory::CreateNativePixmapFromHandle()` and submits the DMA-BUF to the Wayland compositor via the `zwp_linux_dmabuf_v1` protocol:

```
zwp_linux_dmabuf_params_v1.add(fd_luma, 0, offset_y, stride_y, modifier_hi, modifier_lo)
zwp_linux_dmabuf_params_v1.add(fd_chroma, 1, offset_uv, stride_uv, modifier_hi, modifier_lo)
zwp_linux_dmabuf_params_v1.create_immed(width, height, DRM_FORMAT_NV12, flags)
→ wl_buffer
```

The Wayland compositor (KWin, Mutter, sway, etc.) receives a `wl_buffer` backed by GPU memory that has never been touched by the CPU since the hardware video engine wrote it. On a compositing desktop running KMS with overlay plane support, the compositor can further promote this buffer directly to a KMS plane, skipping the compositing GPU pass entirely.

[Source: `zwp_linux_dmabuf_v1` protocol, https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml]

The `AcceleratedVideoDecodeLinuxZeroCopyGL` feature flag separately gates the zero-copy path for the GL compositor (X11 and XWayland). With this flag disabled, the decoded NV12 is copied via shared memory to an upload buffer and then uploaded to a GL texture — correct, but consuming one CPU copy per frame. Both `AcceleratedVideoDecodeLinuxGL` and `AcceleratedVideoDecodeLinuxZeroCopyGL` are enabled by default as of Chrome 143 on Wayland.

---

## 8. Firefox's VA-API Architecture

Firefox's hardware video decode architecture differs from Chrome's in a fundamental way: **Firefox drives libva through FFmpeg** (`libavcodec` with `AV_HWDEVICE_TYPE_VAAPI`), rather than calling libva directly. This means Firefox's VA-API path is tightly coupled to FFmpeg's hardware acceleration infrastructure, with Firefox extracting the decoded `AVFrame`'s underlying `VASurface` via a descriptor export call.

### FFmpegVideoDecoder

`FFmpegVideoDecoder<LIBAV_VER>` (`dom/media/platforms/ffmpeg/FFmpegVideoDecoder.h/.cpp`) inherits from `FFmpegDataDecoder<LIBAV_VER>` and `DecoderDoctorLifeLogger`. Key members relevant to VA-API:

```cpp
// dom/media/platforms/ffmpeg/FFmpegVideoDecoder.h (selected members)

AVBufferRef* mVAAPIDeviceContext;   // hw device context for AV_HWDEVICE_TYPE_VAAPI
VADisplay   mDisplay;               // the underlying VADisplay
UniquePtr<VideoFramePool<LIBAV_VER>> mVideoFramePool;  // surface pool
```

[Source: `dom/media/platforms/ffmpeg/FFmpegVideoDecoder.h`, Firefox/Gecko, https://searchfox.org/mozilla-central/source/dom/media/platforms/ffmpeg/FFmpegVideoDecoder.h]

VA-API initialisation in `FFmpegVideoDecoder` creates an `AVHWDeviceContext`:

```cpp
// dom/media/platforms/ffmpeg/FFmpegVideoDecoder.cpp — VA-API init (simplified)

mVAAPIDeviceContext = mLib->av_hwdevice_ctx_alloc(AV_HWDEVICE_TYPE_VAAPI);
// Scope guard for cleanup on init failure...

AVHWDeviceContext* hwctx  = (AVHWDeviceContext*)mVAAPIDeviceContext->data;
AVVAAPIDeviceContext* vactx = (AVVAAPIDeviceContext*)hwctx->hwctx;

// Use Firefox's singleton VADisplay
vactx->display        = VADisplayHolder::GetSingleton()->GetDisplay();
vactx->driver_quirks  = 0;

mLib->av_hwdevice_ctx_init(mVAAPIDeviceContext);
```

[Source: `dom/media/platforms/ffmpeg/FFmpegVideoDecoder.cpp`, Firefox/Gecko, https://searchfox.org/mozilla-central/source/dom/media/platforms/ffmpeg/FFmpegVideoDecoder.cpp]

`VADisplayHolder::GetSingleton()` is a Firefox singleton that holds the `VADisplay`. It opens the DRM render node to obtain the display, and registers a `VAAPIDisplayReleaseCallback` for cleanup on shutdown.

Firefox's accelerated codec list is queried via `GetAcceleratedFormats()` / `IsFormatAccelerated(AVCodecID)`, which checks for VA-API profile support before selecting a hardware decoder.

### GMP Architecture vs. Platform Decoder

Firefox's media stack has two decoder selection paths:
1. **GMP (Gecko Media Plugins)** — used for encrypted media (EME/Widevine). VA-API is not available through the GMP path; hardware decode for encrypted content uses a different mechanism on Linux. The GMP process sandbox is even more restrictive than the RDD process sandbox, and no VA-API path exists through it.
2. **Platform decoder** — used for non-encrypted media. This is where `FFmpegVideoDecoder` with VA-API hardware acceleration lives. The `FFmpegVideoDecoder` is selected by `PDMFactory::CreatePDM()` when it determines the system supports `AV_HWDEVICE_TYPE_VAAPI` for the requested codec.

The `media.hardware-video-decoding.enabled` preference in `about:config` controls whether the platform decoder attempts hardware acceleration. On Firefox 101+, this defaults to `true` for Intel and AMD hardware. The `media.ffmpeg.vaapi.enabled` preference provides a more granular control specifically for the FFmpeg VA-API path; if `media.hardware-video-decoding.enabled` is `true` but `media.ffmpeg.vaapi.enabled` is `false`, Firefox will not attempt VA-API even on capable hardware.

The key architectural difference from Chrome is that Firefox does not maintain its own codec bitstream parsers for the hardware path. FFmpeg's `libavcodec` handles H.264 NAL unit parsing, VP9 superframe parsing, and AV1 OBU parsing. The hardware accelerator (`AV_HWDEVICE_TYPE_VAAPI`) is configured as the `hw_device_ctx` on the `AVCodecContext`, and FFmpeg calls back into VA-API to allocate and decode into `AVFrame` objects whose data pointers reference VA-API surfaces rather than CPU memory. This contrasts with Chrome, where codec parsers (H264Parser, VP9RawBitstream, etc.) are Chrome's own and are shared between the VA-API and Vulkan Video backends.

### RDD Process: Firefox's Equivalent of OOP-VD

Firefox runs media decoding in the **RDD (Remote Data Decoder) process**, its equivalent of Chrome's OOP-VD video utility process. The RDD process sandbox was originally incompatible with VA-API because the seccomp policy blocked the `ioctl` calls needed for DRM render node access.

A series of bugs tracked the fix: bugs 1683808, 1743926, 1751363, 1759109, 1769182 on Bugzilla. The meta tracking bug 1743926 "[meta] RDD VA-API sandbox fixes" covers the full fix history. `MOZ_DISABLE_RDD_SANDBOX=1` is an environment variable that disables the RDD sandbox entirely — useful for isolating whether a hardware decode failure is sandbox-related, but **a security risk** and not suitable for production use.

[Source: Mozilla Bugzilla bug 1743926, https://bugzilla.mozilla.org/show_bug.cgi?id=1743926]

---

## 9. Firefox VA-API in Depth

### DMABufSurface

`DMABufSurface` (`widget/gtk/DMABufSurface.h/.cpp`) is Firefox's abstraction for GPU memory accessible via DMA-BUF. There are two subtypes: `SURFACE_RGBA` for compositor textures and `SURFACE_YUV` for video decode outputs.

`DMABufSurfaceYUV` handles `VA_FOURCC_NV12`, `VA_FOURCC_I420`, `VA_FOURCC_P010` (10-bit NV12), and `VA_FOURCC_P016` formats. When a VA-API frame is decoded, `GetVAAPISurfaceDescriptor(VADRMPRIMESurfaceDescriptor*)` exports the surface descriptor, and `CreateImageVAAPI(int64_t, int64_t, int64_t)` creates a `DMABufSurfaceYUV` from the decoded frame.

```cpp
// widget/gtk/DMABufSurface.cpp (UpdateYUVData, simplified)
// Imports a VA-API decoded surface into Firefox's DMABuf infrastructure:

bool DMABufSurfaceYUV::UpdateYUVData(VADRMPRIMESurfaceDescriptor* desc) {
    mSurfaceFormat = desc->fourcc;  // e.g., VA_FOURCC_NV12
    for (uint32_t i = 0; i < desc->num_layers; i++) {
        mDmabufFds[i]      = desc->objects[desc->layers[i].object_index[0]].fd;
        mStrides[i]        = desc->layers[i].pitch[0];
        mOffsets[i]        = desc->layers[i].offset[0];
        mDrmFormats[i]     = desc->layers[i].drm_format;
        mDrmModifiers[i]   = desc->objects[desc->layers[i].object_index[0]]
                               .drm_format_modifier;
    }
    mWidth   = desc->width;
    mHeight  = desc->height;
    return true;
}
```

[Source: `widget/gtk/DMABufSurface.cpp`, Firefox/Gecko, https://searchfox.org/mozilla-central/source/widget/gtk/DMABufSurface.cpp]

Per-plane DMA-BUF fds are stored directly. Surfaces are recycled via `mCanRecycle`, and a global refcount enables cross-process sharing between the RDD process and the compositor process.

### IPC: SurfaceDescriptorDMABuf

The decoded `DMABufSurfaceYUV` is serialised for IPC via `SurfaceDescriptorDMABuf` (defined in `SurfaceDescriptor.ipdlh`). This is the wire format that moves from the RDD process to Firefox's compositor (parent) process. The compositor-side texture is represented by `WaylandDMABUFTextureHostOGL`, which creates an `EGLImage` from the received DMA-BUF file descriptors.

On Wayland, WebRender's `TextureHost` backed by `SurfaceDescriptorDMABuf` is the zero-copy NV12 compositing path. The luma and chroma planes are imported as separate `EGLImage` objects via `EGL_EXT_image_dma_buf_import`, then bound as GL textures for the NV12→RGB shader in WebRender's video compositing pass.

### Fence Synchronisation

Firefox uses `EGLSyncKHR` objects for GPU-GPU synchronisation between the VA-API producer and the EGL compositor consumer:

```cpp
// widget/gtk/DMABufSurface.cpp (fence synchronisation, simplified)
// FenceSet() creates an EGLSyncKHR after VA-API decode completes
// FenceWait() blocks the compositor's EGL context until the fence signals
DMABufSurface::FenceSet();   // creates EGLSyncKHR, stores fd via GetSyncFd()
DMABufSurface::FenceWait();  // EGLClientWaitSyncKHR in compositor thread
```

`SetSemaphoreFd()` provides an alternative path for more advanced GPU-GPU coordination using semaphore file descriptors. The EGLSync approach is simpler but adds per-frame latency if the compositor thread blocks; the semaphore path allows pipelined operation.

[Source: `widget/gtk/DMABufSurface.h`, Firefox/Gecko, https://searchfox.org/mozilla-central/source/widget/gtk/DMABufSurface.h]

---

## 10. Codec Coverage: AV1, HEVC, VP9, and 10-Bit Formats

### AV1

AV1 hardware decode support arrived on Linux later than on other platforms due to both hardware and software gaps:

- **Intel Gen12 (Tiger Lake, 2020+)**: Intel Media Driver 2020.3 added `VAProfileAV1Profile0` on Gen12. AV1 4:2:0 8-bit and 10-bit, up to 8K. Requires `iHD` driver; `i965` has no AV1 support.
- **AMD RX 6000 (Navi 2x, RDNA2, 2020+)**: VCN 3.0 hardware AV1 decode. Accessible via `LIBVA_DRIVER_NAME=radeonsi` on Mesa 21.1+.

In Chrome, `AV1VaapiVideoDecoderDelegate` submits AV1 tile groups via `FillAV1SliceParameters()`. Each tile group is a separate `VASliceParameterBufferAV1` + `VASliceDataBufferType` pair. Film grain synthesis (if present in the stream) is submitted via `FillFilmGrainInfo()` populating `VAFilmGrainStructAV1` in the picture parameter buffer — the hardware video engine performs film grain synthesis rather than the CPU.

In Firefox, FFmpeg's `libavcodec` AV1 VA-API decoder (codec ID `AV_CODEC_ID_AV1`) handles the same tile group submission via the `av1_vaapi` hwaccel decoder. The `AVCodecHWConfig` for `AV_HWDEVICE_TYPE_VAAPI` is checked via `IsFormatAccelerated(AV_CODEC_ID_AV1)`.

### VP9 and 10-Bit Formats

VP9 Profile 0 (8-bit YUV 4:2:0) has been widely supported on Intel and AMD VA-API for several years. VP9 Profile 2 (10-bit YUV 4:2:0) requires `P010` surface format. Chrome's delegate confirms support with: `(profile == 0 && bit_depth == 8) || (profile == 2 && bit_depth == 10)`. Chrome landed `P010` NV12 10-bit import support for the zero-copy path in 2024.

### HEVC (H.265)

HEVC hardware decode on Linux VA-API is complicated by codec patent licensing:

- **Intel**: `VAProfileHEVCMain` and `VAProfileHEVCMain10` available on Intel iHD with Gen 9+ (Skylake+). Chrome 107+ officially supports HEVC hardware decode on supported platforms; Linux/VA-API HEVC (`VAProfileHEVCMain10`) landed in Chrome's `HEVCVaapiVideoDecoderDelegate` in 2024.
- **AMD**: HEVC VA-API decode supported on Mesa RadeonSI since Mesa 20.x for GCN and RDNA1+ hardware.
- **Firefox**: HEVC is typically not shipped in Firefox Linux binaries due to patent concerns. Distribution-built Firefox packages may enable it, but it is not in Mozilla's official Linux builds.

### Vulkan Video — Future Direction

Khronos Vulkan Video (`VK_KHR_video_decode_queue`, `VK_KHR_video_decode_h264`, `VK_KHR_video_decode_av1`) offers a codec-specific video decode API inside the Vulkan command stream. A Chromium design document published in December 2025 positions Vulkan Video as a future **complementary** backend to VA-API, not a replacement. The integration requires approximately 2,200 lines of conversion code to translate Chromium's internal parser structs (`H264SPS`, `H265VPS`, etc.) into Vulkan's `StdVideo*` structs defined in the `vulkan_video_codec_*` headers. The zero-copy path for Vulkan Video decoding is simpler than for VA-API: decoded frames are `VkImage` objects in the DPB (Decoded Picture Buffer) image array, importable directly into Vulkan swapchain composition without the VA-API → DMA-BUF → EGLImage → Vulkan texture chain.

Note: as of this writing, no Chrome stable version ships Vulkan Video as a hardware decode path on Linux. The transition planning is in progress.

[Source: Vulkan Video extensions, Khronos Registry, https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_video_decode_queue.html]

### Chrome Codec Status Verification

`vainfo` on the system shows which profiles and entrypoints are available:

```bash
vainfo --display drm --device /dev/dri/renderD128
# Intel iHD output excerpt:
# VAProfileAV1Profile0            :	VAEntrypointVLD
# VAProfileH264High               :	VAEntrypointVLD
# VAProfileHEVCMain               :	VAEntrypointVLD
# VAProfileHEVCMain10             :	VAEntrypointVLD
# VAProfileVP9Profile0            :	VAEntrypointVLD
# VAProfileVP9Profile2            :	VAEntrypointVLD
```

[Source: `vainfo`, libva-utils, https://github.com/intel/libva-utils]

`chrome://gpu` → "Video Acceleration Information" shows Chrome's view of available profiles after applying the driver blocklist. A discrepancy between `vainfo` output and `chrome://gpu` typically indicates a Chrome blocklist entry for that driver/hardware combination.

---

## 11. NVIDIA: No Native VA-API in Chrome, nvidia-vaapi-driver for Firefox

### Chrome on NVIDIA

Chrome does not have a VA-API decode path for NVIDIA GPUs on Linux. The Chrome GPU blocklist explicitly disables VA-API for NVIDIA hardware; the underlying reason is that NVIDIA's proprietary driver does not ship a VA-API library, and Chrome's `VaapiWrapper` requires a working VA-API driver. The `--enable-features=VaapiOnNvidiaGPUs` Chrome flag exists for experimental testing but is not a supported path.

NVIDIA hardware video decode in Chrome on Linux is accessible via:
1. **Vulkan Video** — a future path; Khronos published a Chromium Vulkan Video integration design document in December 2025 positioning it as a complementary backend. Note: needs verification of shipped Chrome version.
2. **Software decode** — `FFmpegVideoDecoder` falls back to libavcodec CPU decode.
3. **GStreamer backend** — the `gst-nv-video-dec` GStreamer element uses NVIDIA's NVDEC, but this is not Chrome's video decode path; Chrome does not use GStreamer.

### Firefox on NVIDIA via nvidia-vaapi-driver

The `nvidia-vaapi-driver` project (https://github.com/elFarto/nvidia-vaapi-driver) is a third-party VA-API implementation that wraps NVIDIA's NVDEC via the NVIDIA kernel driver. It is supported by Firefox but **explicitly unsupported** by Chrome.

Requirements for `nvidia-vaapi-driver` with Firefox:
- `nvidia-drm.modeset=1` kernel module parameter.
- `LIBVA_DRIVER_NAME=nvidia` (supported on libva ≥ 2.20).
- `MOZ_DISABLE_RDD_SANDBOX=1` — required on older Firefox builds where the RDD sandbox blocked VA-API; may not be needed on Firefox 136+.
- NVIDIA proprietary driver 470+ or 500+.

```bash
# /etc/environment or session startup:
LIBVA_DRIVER_NAME=nvidia
NVD_BACKEND=direct          # preferred; bypasses EGL, accesses NVDEC directly
                             # EGL backend regressed in driver 525+
```

Supported codecs via `nvidia-vaapi-driver`: AV1, H.264, HEVC, VP8, VP9, MPEG-2, VC-1. Not supported: MPEG-4 Part 2, JPEG decode.

Firefox verification:
```bash
MOZ_LOG="FFmpegVideo:5" firefox 2>&1 | grep -i vaapi
# Logging at level 5 for the FFmpegVideo module shows VA-API init, profile selection,
# and per-frame decode operations.
```

Note: The environment variable `MOZ_VA_DEBUG` is sometimes referenced in older documentation, but the correct logging mechanism for `FFmpegVideoDecoder` is `MOZ_LOG="FFmpegVideo:5"`.

---

## 12. Verification and Debugging

A systematic verification and debugging workflow for hardware video decode on Linux:

### Step 1: Verify Driver and Profile Support

```bash
# List VA-API profiles and entrypoints:
vainfo --display drm --device /dev/dri/renderD128

# Force AMD Mesa driver:
LIBVA_DRIVER_NAME=radeonsi vainfo

# Force Intel iHD driver:
LIBVA_DRIVER_NAME=iHD vainfo

# Force legacy Intel driver (deprecated, no AV1, Chrome incompatible):
LIBVA_DRIVER_NAME=i965 vainfo
```

Note: the command is `vainfo` (from the `libva-utils` package), not `va-info`. There is no tool called `va-info`.

### Step 2: Verify Chrome Hardware Decode

```bash
# Launch Chrome with hardware decode explicitly enabled:
google-chrome --enable-features=AcceleratedVideoDecodeLinuxGL

# Or for Wayland:
google-chrome --ozone-platform=wayland \
    --enable-features=AcceleratedVideoDecoder,AcceleratedVideoDecodeLinuxGL
```

Then navigate to `chrome://gpu` and check the "Video Acceleration Information" section. Navigate to `chrome://media-internals` while a video plays and check the `video_decoder` field:
- `"VaapiVideoDecoder"` — hardware decode active.
- `"FFmpegVideoDecoder"` — software decode (hardware unavailable or disabled).

### Step 3: Verify GPU Utilisation

```bash
# Intel: check Video engine utilisation
intel_gpu_top
# "Video" engine percentage > 0 during playback = hardware decode active

# AMD/Intel/NVIDIA:
nvtop
```

### Step 4: Enable VA-API Tracing

```bash
# Trace all libva API calls (per-thread log files):
LIBVA_TRACE=/tmp/libva.log google-chrome --enable-features=AcceleratedVideoDecodeLinuxGL

# Set verbosity level (0=quiet, 1=errors, 2=verbose):
LIBVA_MESSAGING_LEVEL=2 google-chrome
```

`LIBVA_TRACE` creates per-thread log files at the given path prefix. Each `va*()` call is logged with its parameters and return status, which makes it possible to diagnose `VA_STATUS_ERROR_*` failures without stepping through the Chrome binary.

### Step 5: Firefox Debugging

```bash
# Firefox VA-API debug logging:
MOZ_LOG="FFmpegVideo:5" firefox 2>&1 | tee /tmp/firefox-vaapi.log

# about:support → "HARDWARE_VIDEO_DECODING" shows current status
# about:config → media.hardware-video-decoding.enabled (must be true)

# If VA-API fails due to RDD sandbox (older Firefox builds):
MOZ_DISABLE_RDD_SANDBOX=1 firefox  # security risk, debug only
```

### Common Failure Modes

| Symptom | Likely Cause | Remedy |
|---|---|---|
| `chrome://gpu` shows no codec support | Wrong `LIBVA_DRIVER_NAME` or missing driver | Install `libva-intel-media-driver` (Intel) or Mesa (AMD); set `LIBVA_DRIVER_NAME` |
| Video plays but GPU Video engine stays at 0% | Zero-copy path active but GPU engine not tracked by tool | Check `chrome://media-internals` for `VaapiVideoDecoder` confirmation |
| Blank video / green frames | EGL import failure; AMD disjoint plane issue | Ensure Mesa ≥ 21.1 for AMD; check `LIBVA_MESSAGING_LEVEL=2` output |
| Firefox RDD crash / no hardware decode | RDD sandbox blocking render node access | Firefox 136+: sandbox should be fixed; older: try `MOZ_DISABLE_RDD_SANDBOX=1` temporarily |
| `vaExportSurfaceHandle` returns `VA_STATUS_ERROR_UNIMPLEMENTED` | Driver too old or missing DRM_PRIME_2 support | Update Mesa / Intel Media Driver |

---

## 13. Practical Configuration

### AMD: System-Level VA-API Setup

```bash
# /etc/environment (or ~/.profile)
LIBVA_DRIVER_NAME=radeonsi

# Verify Mesa version (Mesa 21.1+ required for stable AMD AV1 decode):
glxinfo | grep "OpenGL version"
vainfo
```

AMD on Mesa does not require explicit `LIBVA_DRIVER_NAME=radeonsi` on most distributions because Mesa auto-detects AMD via the PCI vendor ID. However, if `nvidia-vaapi-driver` is also installed on a multi-GPU system, the explicit variable prevents conflicts.

### Intel: Prefer iHD over i965

```bash
# /etc/environment
LIBVA_DRIVER_NAME=iHD

# Install on Debian/Ubuntu:
sudo apt install intel-media-va-driver-non-free  # for Gen 11 (Ice Lake)+
# or:
sudo apt install intel-media-va-driver            # open variant, Gen 9+
```

The `i965` driver was historically the default on many distributions. Chrome's current `VaapiVideoDecoder` requires `iHD`. If both are installed and the wrong one is selected, `chrome://gpu` will show "Software only, hardware acceleration unavailable" even though hardware is capable.

### Flatpak and Snap: Device Access

Flatpak-sandboxed browsers (Flathub Chrome, Firefox) run with restricted filesystem access. Without explicit DRM permission, `/dev/dri/renderD128` is inaccessible:

```bash
# Check current permissions:
flatpak info --show-permissions com.google.Chrome

# Grant DRI access:
flatpak override --user --device=dri com.google.Chrome
flatpak override --user --device=dri org.mozilla.firefox

# Verify inside Flatpak (run as the Flatpak user):
flatpak run --command=vainfo com.google.Chrome
```

Snap-packaged browsers similarly require the `hardware-observe` and optionally `opengl` snap interfaces; check with `snap connections <name>`.

### Wayland vs. X11 Considerations

On **Wayland** (native, `--ozone-platform=wayland`): the zero-copy NV12 path via `zwp_linux_dmabuf_v1` is fully supported in Chrome 143+ and Firefox 101+. Hardware decode is enabled by default. The compositor must support `linux-dmabuf-v1` with the `zwp_linux_dmabuf_feedback_v1` extension for format/modifier negotiation to work correctly.

On **XWayland** (X11 applications running under Wayland): the EGLImage import path via `EGL_EXT_image_dma_buf_import` is used. This generally works but may have issues with DRM format modifier negotiation depending on the Mesa version and the specific compositor's XWayland DMA-BUF forwarding support.

On **X11 native**: the EGLImage path is used with `X11SurfaceFactory`. Hardware decode works but may not support overlay plane promotion; the decoded frame is composited into the X11 window by the GPU rather than scanned out directly by KMS.

### Multi-GPU Systems

On systems with both an integrated GPU (Intel/AMD iGPU) and a discrete GPU, VA-API will typically use whichever device `libva` selects via `/dev/dri/renderD128` (usually the first detected device). Chrome and Firefox both open the render node by index; to force a specific GPU, set `DRI_PRIME=1` (Mesa) to select the secondary GPU, or use `LIBVA_DRM_DEVICE=/dev/dri/renderD129` if supported by the libva version.

---

## Roadmap

### Near-term (6–12 months)

- **Firefox 153 Vulkan Video decode (July 2026)**: Mozilla has resolved Bug 2021722 — "Add Vulkan Video path to FFmpegVideoDecoder" — for the Firefox 153 release (scheduled July 21, 2026). Contributed by NVIDIA engineer Tymur Boiko, this adds a native `VK_KHR_video_decode_queue` path alongside the existing `FFmpeg`/VA-API route, finally giving NVIDIA users first-class hardware decode without the `nvidia-vaapi-driver` workaround. [Source](https://bugzilla.mozilla.org/show_bug.cgi?id=2021722) [Source](https://linuxiac.com/firefox-153-lands-vulkan-video-decode-support/)
- **Chrome hardware video encode on Linux (default-on)**: Chrome 143 enabled hardware decode by default on Wayland, but `AcceleratedVideoEncoder` still requires an explicit flag as of mid-2026. The near-term expectation is that VA-API AV1 and H.264 encode will be promoted to default-on once the OOP-VD sandbox policy for encode is validated across Intel, AMD, and NVIDIA hardware. [Source](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/gpu/vaapi.md)
- **Chromium Vulkan Video decode meta-issue**: Issue 40600458 tracks Vulkan Video (`VK_KHR_video_decode_queue`) integration into Chromium's media stack as a path parallel to VA-API, motivated by the goal of eliminating the non-Vulkan code path and reducing platform-specific surface area. [Source](https://issues.chromium.org/issues/40600458)
- **10-bit (P010) zero-copy path on AMD in Chrome**: Issue 349428388 tracks gaps in 10-bit P010 zero-copy decode for Chrome's Linux VA-API decoder; `DRM_FORMAT_P010` modifier negotiation with `zwp_linux_dmabuf_feedback_v1` is the outstanding piece. Note: needs verification for exact target Chrome version.
- **Firefox VA-API hardware encode for WebRTC**: Hardware encode (send-side) via VA-API or Vulkan Video for Firefox's WebRTC stack is not yet implemented on Linux as of this writing. Near-term work is focused on enabling it for AV1 and H.264, aligning with the Vulkan Video decode infrastructure landed in Firefox 153. [Source](https://bugzilla.mozilla.org/show_bug.cgi?id=1210727)

### Medium-term (1–3 years)

- **Chromium dropping the non-Vulkan (VA-API-direct) code path**: The Chromium GPU team's stated direction is to consolidate on a single Vulkan-backed video pipeline, using `VK_KHR_video_decode_queue` / `VK_KHR_video_encode_queue` for all hardware decode and encode operations on Linux, and retaining VA-API only as the underlying driver ABI where the Mesa or proprietary driver exposes it (i.e., Vulkan Video extensions implemented on top of libva by the driver). This collapses the current two-pipeline architecture (`VaapiVideoDecoder` vs. a future `VulkanVideoDecoder`). [Source](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/linux/hw_video_decode.md)
- **Unified Mesa VCN backend for VA-API and Vulkan Video**: Mesa 26.1 shipped a shared AMD VCN backend used by both the `radeonsi` Gallium VA-API frontend and the `radv` Vulkan Video implementation. Future Mesa releases are expected to extend this to encode (AV1, H.264/H.265) and to deepen VCN4 (RDNA 3) feature coverage including AV1 film grain synthesis in hardware. [Source](https://docs.mesa3d.org/relnotes/26.1.0.html)
- **Intel Arc / Xe2 VA-API and Vulkan Video maturity**: The Intel `iHD` driver and the emerging `intel-media-driver` Vulkan Video path (`VK_INTEL_*` extensions) are expected to reach feature parity for AV1 encode, HEVC encode, and VP9 encode on Xe2-based hardware, enabling Chrome and Firefox to enable encode-side hardware acceleration by default for Intel discrete GPUs. Note: needs verification against Intel Media Driver release schedule.
- **`zwp_linux_dmabuf_feedback_v1` adoption by all major compositors**: The `feedback` extension, which allows the compositor to communicate the preferred DRM format/modifier set for zero-copy import, is required for reliable 10-bit (P010/NV21) zero-copy video in both Chrome and Firefox. KDE Plasma and GNOME Mutter are expected to complete full `feedback` support, removing the last fallback-to-copy cases in browser video pipelines. [Source](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml)
- **WebCodecs hardware encode on Linux**: Chrome's WebCodecs `VideoEncoder` API on Linux currently falls back to software (libvpx, libaom, libx264). Hardware encode support via the VA-API `VAEntrypointEncSlice` path is a stated goal, tracked in the Chromium issue tracker but without a committed timeline. Firefox's WebCodecs encode path faces the same dependency on VA-API encode sandbox policies. Note: needs verification for tracking issue numbers.

### Long-term

- **Full Vulkan Video replace of VA-API in the browser**: The long-term architectural direction for both Chrome and Firefox is a single `VK_KHR_video_*` path that eliminates `libva` as a direct browser dependency. VA-API would remain relevant only as an IPC between the Mesa Vulkan driver implementation and the underlying kernel video engine, invisible to the browser. This converges with the Mesa unified VCN backend strategy and the Khronos Vulkan Video roadmap for adding AV1 encode extensions. [Source](https://www.khronos.org/vulkan/chrome-video/vulkan_video_integration.html)
- **Hardware AV1 encode default-on for WebRTC in browsers**: As NVIDIA Hopper/Ada, AMD RDNA 3/4, and Intel Xe2 hardware AV1 encoders mature and their Linux driver support stabilises, the expectation is that Chrome and Firefox will enable hardware AV1 encode by default for WebRTC video conferencing on Linux — reducing CPU load and enabling higher-quality video calls at equivalent bitrates compared to software libaom.
- **Sandboxing improvements for multi-codec and multi-GPU decode**: The current OOP-VD sandbox (`hardware_video_decoding_sandbox_hook_linux.cc`) allowlists specific render-node device paths and libva symbol sets. Long-term, the direction is towards capability-based sandbox policies that do not require per-codec allowlist entries, enabling new codecs (future VVC/H.266 hardware decode, future codec generations) to be picked up automatically without sandbox policy updates in each browser release. Note: needs verification against Chromium sandbox team public discussions.
- **Tighter integration between browser video pipelines and HDR/wide-gamut display stacks**: As HDR display support (via `drm_connector` HDR metadata, `DRM_FORMAT_P010` with BT.2020 colorspace, and Wayland `wp_color_management_v1`) matures in compositors, browser video pipelines will need to propagate BT.2020/PQ colorspace metadata from decoded frames through the compositor overlay path to the display without tone-mapping to SDR. This is a joint roadmap item across the VA-API decode layer (this chapter), the Wayland compositor stack (Ch27), and the Vulkan/EGL colorspace APIs. Note: needs verification against Wayland color management protocol status.

---

## 14. Integrations

**Chapter 26 (Hardware Video: VA-API, VDPAU, V4L2, GStreamer)**: Chapter 26 covers the VA-API API layer in detail — `vaCreateConfig`, `vaCreateContext`, `vaBeginPicture`, `vaEndPicture`, and `vaExportSurfaceHandle`. The `VADRMPRIMESurfaceDescriptor` and `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2` described there are the exact data structures Chrome's `VaapiWrapper::ExportVASurfaceAsNativePixmapDmaBuf()` and Firefox's `GetVAAPISurfaceDescriptor()` use as the DMA-BUF bridge. The Mesa gallium VA-API frontend (`src/gallium/frontends/va/`) described in Ch26 is the AMD driver both browsers call. Read Ch26 for the full libva API reference; this chapter covers the browser-specific integration layer above it.

**Chapter 33 (Chromium's Multi-Process GPU Architecture)**: `VaapiVideoDecoder` runs in the **video utility process**, not the GPU process. The GPU process hosts `OzoneImageBacking` and the Viz/Skia compositor. Mojo IPC (`GpuChannel` / `MediaGpuChannelManager`) carries `mojo::PlatformHandle`-wrapped DMA-BUF fds between the video utility process and the GPU process. The sandbox architecture described in Ch33 applies to the GPU process; the video utility process has its own, narrower sandbox policy defined in `hardware_video_decoding_sandbox_hook_linux.cc`.

**Chapter 36 (The Chromium Compositor — CC and Viz)**: Decoded video frames become `SharedImage` objects backed by `OzoneImageBacking` (Ch147) and are submitted to Viz as overlay candidates. The overlay promotion logic in Viz selects NV12 `SharedImage` quads for direct KMS plane assignment, bypassing the GPU compositing pass. The `BeginAccess()` / `EndAccess()` fence contract on `OzoneImageBacking` is the interface between the video decode pipeline (this chapter) and the Viz compositor pipeline (Ch36).

**Chapter 37 (Skia and 2D Rendering)**: When video decode is not promoted to a KMS overlay, the decoded NV12 `SharedImage` is composited into the output frame by Skia Ganesh or Graphite. Skia's `SkImage` wrapping a hardware-decoded NV12 texture goes through an NV12→RGB colour conversion shader in Skia's GPU backend. This path shares the same `SharedImage` mailbox infrastructure used by canvas 2D and CSS `<image>` elements.

**Chapter 52 (Firefox and WebRender)**: The `SurfaceDescriptorDMABuf` IPC type (Ch147, Section 9) is consumed by WebRender's `TextureHost` implementation. `WaylandDMABUFTextureHostOGL` in the Firefox GPU process creates the EGLImage from DMA-BUF planes and presents the NV12 texture to WebRender's video compositing pass. The zero-copy NV12 compositing in WebRender on Wayland described in Ch52 uses `DMABufSurfaceYUV` (this chapter) as its pixel source.

**Chapter 38 (PipeWire and the Video Session Layer)**: PipeWire screen capture delivers frames as `wl_drm` / `zwp_linux_dmabuf_v1` buffers through the same DMA-BUF sharing chain as hardware video decode. A captured screen frame and a decoded video frame both end up as `SurfaceDescriptorDMABuf` / `OzoneImageBacking` instances backed by DRM-PRIME fds. The DMA-BUF fd lifetime and the EGLSync fence synchronisation model are identical between the two paths.

**Chapter 146 (WebCodecs and Browser Hardware Acceleration)**: Chrome's WebCodecs `VideoDecoder` API on Linux uses `VaapiVideoDecoder` as its hardware backend — the same `VaapiWrapper`, the same OOP-VD sandbox hook, and the same `OzoneImageBacking`-based zero-copy path. The `ExportVASurfaceAsNativePixmapDmaBuf()` export path is shared between `<video>` element decode and WebCodecs `VideoFrame` output. Firefox's WebCodecs implementation similarly reuses the `FFmpegVideoDecoder` VA-API path.

**Chapter 60b (Video Streaming Protocols and Adaptive Bitrate Delivery)**: Chrome WebRTC's video receive path uses the same `VaapiVideoDecoder` infrastructure for incoming RTP H.264, VP9, and AV1 streams. An RTP-packetised video stream is depacketised by the WebRTC `VideoReceiveStream`, reassembled into frames, and dispatched to the same `media::VideoDecoder` dispatch mechanism that `<video>` element decoding uses — which on Linux selects `VaapiVideoDecoder` for hardware-capable codecs. Firefox WebRTC similarly reuses the `FFmpegVideoDecoder` with `AV_HWDEVICE_TYPE_VAAPI` for incoming RTP video. The GStreamer `webrtcbin → vaapih264dec` zero-copy path referenced in Ch60b is the GStreamer-native analogue of the same hardware decode integration. Note: Firefox VA-API hardware encode for WebRTC send paths is not implemented via libva on Linux as of this writing; only receive-side decode benefits from VA-API acceleration.

---

## 15. References

- libva API reference: https://github.com/intel/libva/blob/master/va/va.h
- libva DRM common types (`VADRMPRIMESurfaceDescriptor`): https://github.com/intel/libva/blob/master/va/va_drmcommon.h
- libva-utils (`vainfo`): https://github.com/intel/libva-utils
- Intel Media Driver (iHD): https://github.com/intel/media-driver
- Mesa gallium VA frontend: https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/va
- Chromium `VaapiVideoDecoder`: https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_video_decoder.h
- Chromium `VaapiWrapper`: https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/vaapi/vaapi_wrapper.cc
- Chromium OOP-VD sandbox hook: https://source.chromium.org/chromium/chromium/src/+/main:media/gpu/sandbox/hardware_video_decoding_sandbox_hook_linux.cc
- Chromium `OzoneImageBacking`: https://source.chromium.org/chromium/chromium/src/+/main:gpu/command_buffer/service/shared_image/ozone_image_backing.h
- Firefox `FFmpegVideoDecoder`: https://searchfox.org/mozilla-central/source/dom/media/platforms/ffmpeg/FFmpegVideoDecoder.h
- Firefox `DMABufSurface`: https://searchfox.org/mozilla-central/source/widget/gtk/DMABufSurface.h
- Mozilla RDD VA-API sandbox meta bug 1743926: https://bugzilla.mozilla.org/show_bug.cgi?id=1743926
- nvidia-vaapi-driver: https://github.com/elFarto/nvidia-vaapi-driver
- `zwp_linux_dmabuf_v1` protocol: https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml
- `EGL_EXT_image_dma_buf_import` extension specification: https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import.txt
- Mesa 26.1 release notes (unified VCN backend): https://docs.mesa3d.org/relnotes/26.1.0.html
- Linux kernel GPU documentation: https://www.kernel.org/doc/html/latest/gpu/
