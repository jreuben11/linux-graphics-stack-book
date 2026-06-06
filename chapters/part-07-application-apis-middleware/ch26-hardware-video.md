# Chapter 26: Hardware Video Decode/Encode

**Part VII — Application APIs and Middleware**

**Audiences targeted**: Application developers building video pipelines with VA-API, GStreamer, and PipeWire; systems developers who need to understand how VA-API drivers map onto Mesa/Gallium internals and kernel DRM/V4L2 infrastructure.

Hardware-accelerated video is one of the oldest and least-understood segments of the Linux graphics stack. Despite VA-API's introduction in 2009, the user-facing APIs remain inconsistent, the zero-copy pipeline connecting decode to display is underused, and the V4L2 stateless codec interface has brought a new generation of SoC hardware support that intersects in non-obvious ways with the older VA-API model. This chapter maps the entire pipeline from camera capture or media container to pixels on screen, with emphasis on the DMA-BUF exchange points that connect each subsystem. The zero-copy imperative is a recurring thread: at every API boundary we examine whether GPU memory crosses a CPU copy and how to avoid that cost.

---

## Table of Contents

1. [VA-API: Architecture and Surface Model](#1-va-api-architecture-and-surface-model)
2. [VA-API Drivers: Mesa and Vendor Implementations](#2-va-api-drivers-mesa-and-vendor-implementations)
3. [VA-API Encode Pipeline](#3-va-api-encode-pipeline)
4. [VDPAU: NVIDIA's Video Path and Its Limitations](#4-vdpau-nvidias-video-path-and-its-limitations)
5. [V4L2: Kernel Video Capture and Stateless Codec Interface](#5-v4l2-kernel-video-capture-and-stateless-codec-interface)
6. [GStreamer: Pipeline Construction and Zero-Copy Video](#6-gstreamer-pipeline-construction-and-zero-copy-video)
7. [PipeWire: The Linux Multimedia Session Manager](#7-pipewire-the-linux-multimedia-session-manager)
8. [Practical: Hardware-Accelerated Transcoding Pipeline](#8-practical-hardware-accelerated-transcoding-pipeline)
9. [libcamera: Modern Camera Stack for Linux](#9-libcamera-modern-camera-stack-for-linux)
10. [Vulkan Video: The Strategic Hardware Video API](#10-vulkan-video-the-strategic-hardware-video-api)
11. [Integrations](#integrations)
12. [References](#references)

---

## 1. VA-API: Architecture and Surface Model

VA-API (Video Acceleration API) is a vendor-neutral, open-source API for hardware-accelerated video decode, encode, and video processing on Linux. Originally developed by Intel in 2009 and open-sourced as `libva`, it provides an ICD-style dispatch layer that routes API calls to hardware-specific driver plugins. The core library `libva.so` loads the appropriate driver plugin at runtime from `/usr/lib/dri/`; the driver is selected by the environment variable `LIBVA_DRIVER_NAME` or by querying the DRM device for a hint. On AMD hardware the driver name resolves to `radeonsi` and the plugin is `radeonsi_drv_video.so`; on Intel it resolves to `iHD` and loads `iHD_drv_video.so` for Gen 8+ or `i965_drv_video.so` for older hardware; on NVIDIA with the open-source wrapper it loads `nvidia_drv_video.so`.

The capability discovery sequence follows a three-step protocol. After obtaining a `VADisplay` from `vaGetDisplayDRM("/dev/dri/renderD128")`, the application calls:

```c
/* va/va.h — libva 2.x */
VAStatus vaInitialize(VADisplay dpy,
                      int *major_version,
                      int *minor_version);

VAStatus vaQueryConfigProfiles(VADisplay dpy,
                               VAProfile *profile_list,
                               int *num_profiles);

VAStatus vaQueryConfigEntrypoints(VADisplay dpy,
                                  VAProfile profile,
                                  VAEntrypoint *entrypoints,
                                  int *num_entrypoints);
```

`vaQueryConfigProfiles` fills a caller-allocated array with the codec profiles supported by the driver — for example, `VAProfileH264Main`, `VAProfileHEVCMain`, `VAProfileAV1Profile0`. For each profile the application then calls `vaQueryConfigEntrypoints` to learn which operations are available: `VAEntrypointVLD` for accelerated decode, `VAEntrypointEncSlice` for full hardware encode, `VAEntrypointEncSliceLP` for low-power encode, and `VAEntrypointVideoProc` for post-processing. The `vainfo` command-line tool is the practical way to inspect this matrix on a live system.

The central object in VA-API is the `VASurface` — an opaque GPU-resident buffer holding exactly one video frame in the hardware driver's native format. Applications never directly allocate GPU memory; they call `vaCreateSurfaces`, which allocates surfaces through the driver:

```c
VAStatus vaCreateSurfaces(VADisplay dpy,
                           unsigned int format,   /* VA_RT_FORMAT_YUV420, etc. */
                           unsigned int width,
                           unsigned int height,
                           VASurfaceID *surfaces,
                           unsigned int num_surfaces,
                           VASurfaceAttrib *attrib_list,
                           unsigned int num_attribs);
```

The `format` parameter selects the chroma subsampling family (`VA_RT_FORMAT_YUV420`, `VA_RT_FORMAT_YUV420_10BPC`, `VA_RT_FORMAT_RGB32`). The pixel format within that family is further constrained by `VASurfaceAttrib` entries: `VA_SURFACE_ATTRIB_PIXEL_FORMAT` pins the exact fourcc (for example, `VA_FOURCC_NV12`, `VA_FOURCC_P010`, `VA_FOURCC_AYUV`); `VA_SURFACE_ATTRIB_USAGE_HINT` signals the intended use to the driver (`VA_SURFACE_ATTRIB_USAGE_HINT_DECODER`, `VA_SURFACE_ATTRIB_USAGE_HINT_DISPLAY`, `VA_SURFACE_ATTRIB_USAGE_HINT_ENCODER`). The hint matters for tiling: hardware codecs often decode into tiled GPU memory layouts that are more efficient for the fixed-function decode engine. The display controller can scan out tiled surfaces on modern Intel and AMD hardware (using DRM format modifiers), but an element downstream that reads the surface as a simple linear array will trigger a detiling copy inside the driver.

A `VAConfig` records the codec profile and entrypoint pair plus any driver-specific configuration attributes. A `VAContext` binds a `VAConfig` to a specific output resolution and a set of `VASurface` render targets. Together they represent one decode or encode pipeline instance:

```c
VAStatus vaCreateConfig(VADisplay dpy,
                         VAProfile profile,
                         VAEntrypoint entrypoint,
                         VAConfigAttrib *attrib_list,
                         int num_attribs,
                         VAConfigID *config_id);

VAStatus vaCreateContext(VADisplay dpy,
                          VAConfigID config_id,
                          int picture_width,
                          int picture_height,
                          int flag,
                          VASurfaceID *render_targets,
                          int num_render_targets,
                          VAContextID *context);
```

The decode submission loop uses `VABuffer` objects to carry the bitstream and codec metadata from the application to the hardware. The key buffer types are: `VASliceDataBuffer` (the raw encoded slice bytes), `VAPictureParameterBuffer` (per-frame reference picture list, flags, POC counts), `VASliceParameterBuffer` (slice offsets and lengths within the slice data buffer), `VAIQMatrixBuffer` (quantization matrix for H.264/HEVC), and `VAProcPipelineParameterBuffer` (for post-processing). The decode submission sequence is:

```c
vaBeginPicture(dpy, context, render_target);          /* bind output surface */
vaRenderPicture(dpy, context, buffers, num_buffers);  /* attach param/slice bufs */
vaEndPicture(dpy, context);                           /* submit to hardware */
vaSyncSurface(dpy, render_target);                    /* CPU-side blocking wait */
```

`vaEndPicture` is non-blocking; it enqueues the decode job to the hardware command queue. `vaSyncSurface` blocks until the hardware reports completion. For pipelined use (the common case), the application overlaps the next frame's `vaBeginPicture`/`vaRenderPicture`/`vaEndPicture` calls while waiting only when it needs to read the decoded pixels.

### The vaExportSurfaceHandle DMA-BUF Export

The critical function for connecting VA-API to the rest of the graphics stack is `vaExportSurfaceHandle`. It exports the underlying GPU allocation as one or more DMA-BUF file descriptors, which can then be imported into Vulkan, EGL, a Wayland compositor, or a GStreamer downstream element without any CPU copy:

```c
/* va/va.h — libva 2.x */
VAStatus vaExportSurfaceHandle(VADisplay dpy,
                               VASurfaceID surface_id,
                               uint32_t mem_type,         /* VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2 */
                               uint32_t flags,            /* VA_EXPORT_SURFACE_READ_ONLY | VA_EXPORT_SURFACE_COMPOSED_LAYERS */
                               void *descriptor);         /* out: VADRMPRIMESurfaceDescriptor* */
```

The `mem_type` must be `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2`; older `DRM_PRIME` support exports a single fd without modifier information and is deprecated. The `flags` field controls layering: `VA_EXPORT_SURFACE_COMPOSED_LAYERS` returns the entire surface as a single layer with all planes, while `VA_EXPORT_SURFACE_SEPARATE_LAYERS` splits each plane into its own layer with its own format (useful for accessing Y and UV planes of NV12 separately as `DRM_FORMAT_R8` and `DRM_FORMAT_GR88`).

The output is a `VADRMPRIMESurfaceDescriptor`:

```c
/* va/va_drmcommon.h */
typedef struct {
    uint32_t fourcc;            /* overall format, e.g. DRM_FORMAT_NV12 */
    uint32_t width;
    uint32_t height;
    uint32_t num_objects;       /* number of DMA-BUF fds */
    struct {
        int fd;                 /* DMA-BUF file descriptor */
        uint32_t size;
        uint64_t drm_format_modifier;
    } objects[4];
    uint32_t num_layers;
    struct {
        uint32_t drm_format;    /* format for this layer */
        uint32_t num_planes;
        uint32_t object_index[4]; /* which object fd each plane uses */
        uint32_t offset[4];
        uint32_t pitch[4];
    } layers[4];
} VADRMPRIMESurfaceDescriptor;
```

A typical NV12 surface on Intel or AMD exports as one DMA-BUF fd (one object), one layer with two planes (Y at offset 0, UV at `width*height`), and a `drm_format_modifier` that encodes the tiling mode. A multi-planar surface on some hardware may export with separate fds per plane. The caller must close each `fd` after importing it downstream. The `drm_format_modifier` value must be communicated to the importing API — it is the key that allows a Vulkan driver or EGL implementation to map the pixels correctly without detiling.

---

## 2. VA-API Drivers: Mesa and Vendor Implementations

### Mesa Gallium VA-API State Tracker

Mesa's VA-API support lives in `src/gallium/frontends/va/` and is the open-source implementation for AMD and Intel (legacy) hardware. The state tracker translates VA-API calls into Gallium `pipe_video_codec` operations, defined in `src/gallium/include/pipe/p_video_codec.h`. The key interface functions are `create_video_codec`, `destroy`, `begin_frame`, `decode_bitstream`, `end_frame`, and `flush`. Each Gallium pipe driver that supports video implements these callbacks using hardware-specific firmware packets.

For AMD hardware, the `radeonsi` Gallium driver implements `pipe_video_codec` using the VCN (Video Core Next) hardware block present on GCN5 (Vega) and all RDNA generations. VCN generational capabilities:

- **VCN 1.0** (Vega, Raven Ridge): H.264, HEVC, VP9 decode; H.264, HEVC encode
- **VCN 2.0** (Navi 10, Renoir): adds H.264/HEVC encode improvements; JPEG decode
- **VCN 3.0** (RDNA2, Navi 21, Van Gogh): adds AV1 decode hardware; first VCN with AV1 support
- **VCN 4.0** (RDNA3, Navi 31): adds AV1 encode hardware; Mesa 23.1+ exposes AV1 encode through `VAEntrypointEncSlice` with `VAProfileAV1Profile0`

The `radeonsi` VA driver is loaded as `radeonsi_drv_video.so` and accesses the VCN hardware through the `amdgpu` kernel DRM driver using firmware command submission via the `AMDGPU_HW_IP_VCN_DEC` and `AMDGPU_HW_IP_VCN_ENC` queue types. The `RADEON_FEEDBACK_BUFFER_SIZE` parameter exposes hardware feedback for multi-slice AV1 decode, where the hardware must report the number of slices processed per frame.

### Intel VA-API Drivers

Intel ships two VA-API driver generations. The legacy `libva-intel-driver` (providing `i965_drv_video.so`) covers Pre-Broadwell hardware (Gen 7 and earlier) and is being phased out of active development. The current driver is `intel-media-driver` (providing `iHD_drv_video.so`), covering Gen 8 (Broadwell) and newer, including Tigerlake, Alder Lake, Raptor Lake, and Intel Arc (Alchemist). `iHD_drv_video.so` supports AV1 decode from Tigerlake (Gen 12) onward, and AV1 encode from Arc (Xe HPG) onward.

Intel's oneVPL (Video Processing Library) is a higher-level media processing API that wraps `iHD` under the hood. It provides GPU-accelerated decode, encode, and VPP (video pre/post processing) through a unified interface independent of VA-API details, but its Linux path still dispatches through `libva` for hardware access.

### NVIDIA VA-API

NVIDIA does not ship a native VA-API driver. Instead, the open-source `nvidia-vaapi-driver` project (maintained by Xavier Chantry, distributed as `nvidia_drv_video.so`) wraps NVDEC hardware access through a combination of the NVIDIA VDPAU library and, more recently, direct Vulkan Video calls. The wrapper translates `vaInitialize`, `vaCreateContext`, `vaBeginPicture`, etc. into NVDEC operations. H.264 and HEVC decode work reliably with NVIDIA driver 525 or newer; VP9 decode is supported on Maxwell+; AV1 decode on Ampere+. The wrapper does not support encode because NVENC does not have a VA-API-compatible interface.

The `LIBVA_DRIVER_NAME` and `LIBVA_DRIVERS_PATH` environment variables override the automatic driver selection. Setting `LIBVA_DRIVER_NAME=radeonsi` and `LIBVA_DRIVERS_PATH=/usr/lib/dri` is the standard way to force a specific driver. `vainfo` queries and prints the driver name, API version, and the full profile/entrypoint matrix — it is the first diagnostic tool to reach for when hardware acceleration is not working.

### NVIDIA's Forward Path: Vulkan Video

NVIDIA's proprietary Vulkan driver exposes `VK_KHR_video_decode_h264`, `VK_KHR_video_decode_h265`, and `VK_KHR_video_decode_av1` extensions on Maxwell+ hardware. This is the strategic forward path for NVIDIA hardware decode on Linux. The `nvidia-vaapi-driver` will eventually be superseded for NVIDIA hardware by applications and middleware that use Vulkan Video directly, bypassing the VA-API translation layer. Section 10 covers the Vulkan Video programming model in full.

---

## 3. VA-API Encode Pipeline

Hardware encode in VA-API uses the same object model as decode — `VAConfig`, `VAContext`, `VASurface` — but with encode-specific entrypoints and buffer types. The encode entrypoints are `VAEntrypointEncSlice` (full hardware encode with rate control support) and `VAEntrypointEncSliceLP` (low-power encode using a dedicated power-efficient engine, available on Intel TGL+ and some AMD RDNA3 configurations).

Rate control is configured through `VAEncMiscParameterBuffer` buffers carrying `VAEncMiscParameterRateControl` structures. The primary rate control modes are `VA_RC_CBR` (constant bit rate, for live streaming), `VA_RC_VBR` (variable bit rate, for file encoding), and `VA_RC_CQP` (constant quantization parameter, for quality-driven offline encoding). The `VAEncMiscParameterRateControl` structure carries `bits_per_second`, `target_percentage`, `window_size`, and `initial_qp` fields. Multiple `VAEncMiscParameterBuffer` buffers are attached per frame to configure HRD (hypothetical reference decoder) parameters, frame rate, and quality level independently.

Reference frame management uses `VAEncPictureParameterBuffer` (codec-specific variant), which contains the reference picture list entries. For H.264, this is `VAEncPictureParameterBufferH264` with `CurrPic`, `ReferenceFrames[16]`, and slice-level fields. The decoded picture buffer (DPB) surfaces are the same `VASurface` objects used as render targets; the encoder reads from them for inter-prediction. H.264 encode parameter flow:

1. `VAEncSequenceParameterBufferH264` — SPS-level parameters (profile, level, resolution, GOP structure)
2. `VAEncPictureParameterBufferH264` — per-picture DPB references, frame type (I/P/B)
3. `VAEncSliceParameterBufferH264` — slice type, QP delta, macroblock range
4. `VAEncMiscParameterBuffer` (rate control, HRD, frame rate) — attached as separate render buffers

AV1 hardware encode is available from RDNA3 (VCN 4.0) on AMD and from Intel Arc (Xe HPG) onward. Mesa 23.1 exposed `VAProfileAV1Profile0` with `VAEntrypointEncSlice` for radeonsi. The AV1 encode parameter structure `VAEncSequenceParameterBufferAV1` carries the sequence OBU fields, while `VAEncPictureParameterBufferAV1` contains the frame header fields including the tile configuration.

Retrieving encoded data uses the `VACodedBuffer` mechanism. After `vaEndPicture` and `vaSyncSurface`, the application maps the coded buffer:

```c
VACodedBufferSegment *segment;
vaMapBuffer(dpy, coded_buf, (void**)&segment);
while (segment) {
    fwrite(segment->buf, 1, segment->size, output);
    segment = segment->next;  /* linked list for multi-segment frames */
}
vaUnmapBuffer(dpy, coded_buf);
```

The non-blocking alternative is `vaQuerySurfaceStatus`, which returns `VASurfaceRendering` while the encode is in progress and `VASurfaceReady` when complete. For low-latency screen recording the `VAEntrypointEncSliceLP` path with `VA_RC_CBR` and short GOP (all-intra or I+P without B-frames) minimises pipeline depth; the LP engine completes frames faster at the cost of slightly lower quality per bit.

---

## 4. VDPAU: NVIDIA's Video Path and Its Limitations

VDPAU (Video Decode and Presentation API for Unix) was introduced by NVIDIA in 2008 and predates VA-API by about a year. Its design philosophy is display-centric rather than decode-centric: the API includes a complete post-processing pipeline (`VdpVideoMixer`) and presentation queue (`VdpPresentationQueue`) alongside the core decode objects. The primary VDPAU objects are `VdpDevice` (the hardware context), `VdpDecoder` (a decode pipeline instance, analogous to `VAContext`), `VdpVideoSurface` (a decoded frame surface, analogous to `VASurface`), `VdpOutputSurface` (a post-processed display-ready surface), and `VdpPresentationQueue` (manages frame scheduling for display).

NVIDIA's VDPAU driver supports H.264, HEVC, VP9, and AV1 decode on Maxwell and newer GPUs. `VdpVideoMixer` provides GPU-accelerated deinterlacing (temporal and temporal-spatial), noise reduction, edge enhancement, and inverse telecine — all running entirely on the GPU without CPU involvement. The output of the mixer is a `VdpOutputSurface` that is directly presented via `VdpPresentationQueue::Display`.

The fundamental limitation is that `VdpPresentationQueue` targets an X11 window (`VdpPresentationQueueTarget` created via `vdp_presentation_queue_target_create_x11`). There is no native Wayland backend. On Wayland desktops, VDPAU-based video players (older MPV or VLC configurations) fall back through the `libvdpau-va-gl` wrapper, which translates VDPAU calls to either VA-API (with hardware accelerated decode) or OpenGL (for software rendering of the presentation queue). This translation is lossy and introduces latency because the `VdpVideoSurface` must be copied to a `VASurface` or an OpenGL texture at the API boundary.

OpenGL interop via `VdpauInitNV`, `vdpauRegisterVideoSurfaceNV`, and `vdpauMapSurfacesNV` allows a VDPAU-decoded surface to be accessed as an OpenGL texture without a CPU copy — but this path requires an OpenGL context on the same X11 display, reinforcing the X11 dependency.

Intel and AMD have never shipped VDPAU drivers; the API is exclusively an NVIDIA-on-X11 technology. For new applications on Wayland, the correct NVIDIA acceleration path is VA-API via `nvidia-vaapi-driver` for current hardware, and Vulkan Video for forward-looking code. VDPAU remains in active use only in legacy X11 deployments with NVIDIA hardware and in the video player code of MPV/VLC, where the VDPAU path is still enabled when the X11 VDPAU backend is detected.

---

## 5. V4L2: Kernel Video Capture and Stateless Codec Interface

V4L2 (Video for Linux 2) is the kernel subsystem for video capture, output, and codec offload. Its scope extends well beyond cameras: it is also used for digital TV capture, memory-to-memory format conversion, and — crucially for embedded Linux — hardware video decode on SoC codecs. V4L2 devices appear as `/dev/videoN` nodes; camera ISP and codec hardware with multiple components additionally expose `/dev/mediaN` nodes through the Media Controller framework.

### V4L2 Buffer Model and DMA-BUF

The V4L2 buffer lifecycle uses four ioctls: `VIDIOC_REQBUFS` (allocate buffer queue slots), `VIDIOC_QUERYBUF` (retrieve per-buffer parameters), `VIDIOC_QBUF` (hand a buffer to the hardware), and `VIDIOC_DQBUF` (receive a completed buffer from the hardware). The memory type, specified in `v4l2_requestbuffers.memory`, determines buffer ownership:

- `V4L2_MEMORY_MMAP`: kernel allocates buffers; application maps them with `mmap` using offsets from `VIDIOC_QUERYBUF`
- `V4L2_MEMORY_DMABUF`: application provides DMA-BUF fds for each buffer slot; the kernel driver uses them directly
- `V4L2_MEMORY_USERPTR`: application provides userspace pointers; deprecated and rarely used for new code

For DMA-BUF export of MMAP-allocated buffers, `VIDIOC_EXPBUF` returns a DMA-BUF fd for a given buffer index:

```c
struct v4l2_exportbuffer expbuf = {
    .type  = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .index = i,
    .flags = O_RDONLY,
};
ioctl(fd, VIDIOC_EXPBUF, &expbuf);
int dmabuf_fd = expbuf.fd;
```

This fd can be imported into Vulkan, EGL, or another V4L2 device using `V4L2_MEMORY_DMABUF` — enabling zero-copy camera pipelines.

### Stateful vs. Stateless Codecs

V4L2 codec devices come in two architectural flavours that have profoundly different programming models, and confusing them is a frequent source of bugs.

A **stateful codec** driver internalises the entire decode state machine. The application feeds NAL units (for H.264/HEVC) or OBUs (for AV1) to the OUTPUT queue as contiguous byte streams, and the driver handles bitstream parsing, DPB reference tracking, output reordering, and error concealment internally. The application receives decoded frames from the CAPTURE queue in display order, ready for immediate display. Stateful codecs are found on Samsung Exynos MFC hardware, Mediatek Vcodec, and some older SoC codecs. They are easy to use but inflexible: the application cannot inspect or control individual decode parameters.

A **stateless codec** driver is a thin hardware submission layer. The kernel driver exposes raw hardware registers as V4L2 controls; the application (or a userspace library) is responsible for parsing the bitstream, maintaining the DPB reference list, and supplying every per-frame decode parameter explicitly via `VIDIOC_S_EXT_CTRLS`. The kernel does no bitstream parsing and maintains no cross-frame state. Stateless drivers are found on Rockchip RKVDEC, Allwinner Cedrus, Hantro, Broadcom (Raspberry Pi), and — since kernel 6.2 — Intel IPU stateless paths. The design philosophy is that bitstream parsing is a well-solved userspace problem (libraries like ffmpeg and libvpx), while hardware submission is the kernel's job.

### The V4L2 Request API

The Request API is the mechanism that makes stateless codecs work. A *request* is a kernel object representing one atomic decode job — exactly one compressed frame plus all its metadata. It is allocated from the media device:

```c
int media_fd = open("/dev/media0", O_RDWR);
int request_fd;
ioctl(media_fd, MEDIA_IOC_REQUEST_ALLOC, &request_fd);
```

Controls are attached to the request by setting `v4l2_ext_controls.which = V4L2_CTRL_WHICH_REQUEST_VAL` and `.request_fd = request_fd` before calling `VIDIOC_S_EXT_CTRLS`. Buffers are attached by setting `V4L2_BUF_FLAG_REQUEST_FD` and `v4l2_buffer.request_fd = request_fd` in the `VIDIOC_QBUF` call. The request is then submitted atomically:

```c
ioctl(request_fd, MEDIA_REQUEST_IOC_QUEUE, NULL);
```

Completion is detected by `poll(request_fd, POLLIN)`, which becomes readable when the hardware finishes. After polling, `VIDIOC_DQBUF` retrieves the decoded frame from the CAPTURE queue. The request fd can be reused for the next frame by calling `MEDIA_REQUEST_IOC_REINIT` before reattaching controls and buffers.

H.264 stateless decode uses these V4L2 controls (stable since kernel 5.11):

```c
struct v4l2_ctrl_h264_sps     sps;        /* sequence parameter set */
struct v4l2_ctrl_h264_pps     pps;        /* picture parameter set */
struct v4l2_ctrl_h264_decode_params dp;   /* per-frame POC, reference list */
struct v4l2_ctrl_h264_scaling_matrix sm;  /* quantisation matrix */
```

AV1 stateless controls (`v4l2_ctrl_av1_sequence`, `v4l2_ctrl_av1_frame`) were merged in kernel 6.6 (October 2023).

### Rockchip RKVDEC — A Complete Worked Example

The Rockchip RKVDEC (Video Decoder Engine) illustrates the stateless Request API on real SoC hardware. RKVDEC is present on RK3399 and RK3588; the RK3399 RKVDEC driver was in staging since kernel 5.10. The RK3588 uses a distinct hardware block (VDPU381) rather than RKVDEC2; VDPU381 support was merged into the mainline kernel in kernel 6.15 (early 2026), adding H.264 and HEVC decode via the stateless V4L2 interface. AV1 decode on RK3588 via VDPU381 is targeted for a future kernel release.

A complete H.264 stateless decode loop on RK3399:

```c
/* Step 1: Open the media controller and codec video node */
int media_fd = open("/dev/media0", O_RDWR | O_CLOEXEC);
int video_fd = open("/dev/video0", O_RDWR | O_CLOEXEC);

/* Step 2: Set OUTPUT format to H264_SLICE (stateless slice input) */
struct v4l2_format fmt = {
    .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
    .fmt.pix_mp = {
        .pixelformat = V4L2_PIX_FMT_H264_SLICE,
        .width       = coded_width,
        .height      = coded_height,
    },
};
ioctl(video_fd, VIDIOC_S_FMT, &fmt);

/* Step 3: Set CAPTURE format to NV12 (decoded output) */
fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
fmt.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_NV12;
ioctl(video_fd, VIDIOC_S_FMT, &fmt);

/* Step 4: Allocate OUTPUT buffers (slice data, MMAP) */
struct v4l2_requestbuffers reqbufs = {
    .count  = 8,
    .type   = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
    .memory = V4L2_MEMORY_MMAP,
};
ioctl(video_fd, VIDIOC_REQBUFS, &reqbufs);

/* Step 5: Allocate CAPTURE buffers as DMA-BUF (decoded frames, zero-copy) */
reqbufs.type   = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
reqbufs.memory = V4L2_MEMORY_DMABUF;
reqbufs.count  = NUM_DPB_SLOTS + 2;  /* DPB surfaces + display buffers */
ioctl(video_fd, VIDIOC_REQBUFS, &reqbufs);

/* Per-frame decode loop */
for each frame {
    /* Step 6: Allocate a fresh request */
    int req_fd;
    ioctl(media_fd, MEDIA_IOC_REQUEST_ALLOC, &req_fd);

    /* Step 7: Attach H.264 controls to the request */
    struct v4l2_ext_controls ctrls = {
        .which      = V4L2_CTRL_WHICH_REQUEST_VAL,
        .request_fd = req_fd,
        .count      = 4,
        .controls   = (struct v4l2_ext_control[]) {
            { .id = V4L2_CID_STATELESS_H264_SPS,           .ptr = &sps },
            { .id = V4L2_CID_STATELESS_H264_PPS,           .ptr = &pps },
            { .id = V4L2_CID_STATELESS_H264_DECODE_PARAMS, .ptr = &decode_params },
            { .id = V4L2_CID_STATELESS_H264_SCALING_MATRIX,.ptr = &scaling_matrix },
        },
    };
    ioctl(video_fd, VIDIOC_S_EXT_CTRLS, &ctrls);

    /* Step 8: Queue the compressed slice data to OUTPUT with request_fd */
    struct v4l2_buffer out_buf = {
        .type    = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
        .memory  = V4L2_MEMORY_MMAP,
        .index   = output_slot,
        .flags   = V4L2_BUF_FLAG_REQUEST_FD,
        .request_fd = req_fd,
    };
    ioctl(video_fd, VIDIOC_QBUF, &out_buf);

    /* Step 9: Queue a DMA-BUF capture buffer for the decoded frame */
    struct v4l2_buffer cap_buf = {
        .type    = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
        .memory  = V4L2_MEMORY_DMABUF,
        .index   = capture_slot,
        .m.planes[0].m.fd = dpb_dmabuf_fds[capture_slot],
    };
    ioctl(video_fd, VIDIOC_QBUF, &cap_buf);

    /* Step 10: Submit the request atomically */
    ioctl(req_fd, MEDIA_REQUEST_IOC_QUEUE, NULL);

    /* Step 11: Wait for completion */
    struct pollfd pfd = { .fd = req_fd, .events = POLLIN };
    poll(&pfd, 1, -1);

    /* Step 12: Dequeue the decoded NV12 DMA-BUF */
    ioctl(video_fd, VIDIOC_DQBUF, &cap_buf);
    /* cap_buf.m.planes[0].m.fd is now a ready NV12 DMA-BUF — pass to display */

    ioctl(req_fd, MEDIA_REQUEST_IOC_REINIT, NULL);  /* reuse for next frame */
}
```

The CAPTURE buffers are NV12 DMA-BUF fds backed by the same GEM/DMA-BUF infrastructure as the display driver. They can be imported directly into a Wayland compositor via `zwp_linux_dmabuf_v1`, into a GStreamer `waylandsink`, or into an EGL image — the pixel data never crosses a CPU boundary from decode completion to display.

Verify support with: `v4l2-ctl -d /dev/video0 --list-formats-out` (should show `H264_SLICE`), and use the GStreamer pipeline `gst-launch-1.0 filesrc location=video.h264 ! h264parse ! v4l2slh264dec ! waylandsink` for a complete zero-copy playback path.

### V4L2 and GStreamer

The GStreamer `v4l2codecs` plugin (`gst-plugins-bad`) provides stateless decoder elements: `v4l2slh264dec`, `v4l2slhevcdec`, `v4l2slvp9dec`, and `v4l2slav1dec` (since GStreamer 1.22). Each element owns a request-submission thread, a DPB surface pool backed by `V4L2_MEMORY_DMABUF`, and a bitstream parser feeding per-frame controls. Chromium's V4L2VDA (V4L2 Video Decode Accelerator) on ChromeOS uses the same pattern: userspace DPB management, direct Request API usage, and DMA-BUF export of decoded frames.

---

## 6. GStreamer: Pipeline Construction and Zero-Copy Video

GStreamer is a plugin-based multimedia framework using a pipeline graph model. Elements (source, filter, sink) connect via pads, and the caps negotiation mechanism ensures every pair of connected pads agrees on buffer format before streaming begins. The bus delivers asynchronous messages (errors, state changes, EOS) to the application.

### VA-API Elements

The VA-API GStreamer elements originate from the `gstreamer-vaapi` project (now merged into the main GStreamer repository). The decode elements follow the pattern `vaapi{codec}dec`: `vaapih264dec`, `vaapih265dec`, `vaapivp9dec`, `vaapiv9dec`, `vaapiav1dec`. The `vaapidecodebin` is a convenience bin containing a decoder, a queue, and optionally `vaapipostproc`; it was demoted to NONE rank in newer GStreamer releases in favour of the individual decoder elements, which have cleaner DMA-BUF negotiation.

`vaapipostproc` performs GPU-accelerated colour conversion, scaling, and deinterlacing using VA-API `VAEntrypointVideoProc`. It accepts `VASurface` DMA-BUF inputs and produces DMA-BUF outputs, preserving the zero-copy path. `vaapisink` can display VA-API surfaces directly: on X11 it uses VA-API's `vaPutSurface`; on Wayland it uses `zwp_linux_dmabuf_v1` after calling `vaExportSurfaceHandle` — the fd goes directly into the compositor without CPU copying.

### Zero-Copy Buffer Passing

GStreamer zero-copy rests on three mechanisms:

1. **`GstDmaBufAllocator`**: a GstAllocator subclass that creates `GstMemory` objects backed by DMA-BUF fds. Any element using this allocator for its output buffers participates in zero-copy with any downstream element that understands DMA-BUF input.

2. **`memory:DMABuf` caps feature**: a caps feature string that elements declare to advertise DMA-BUF capability. During caps negotiation, if upstream declares `memory:DMABuf` and downstream accepts it, GStreamer routes the DMA-BUF buffers directly. If any element in the chain lacks DMA-BUF support, the caps negotiation falls back to system memory and GStreamer inserts a CPU copy at that boundary.

3. **`GstDrmFormatModifierMeta`** (GStreamer 1.24+): a buffer metadata type carrying the DRM format modifier from the allocating element to the importing element, enabling correct tiled/compressed buffer import.

To verify that zero-copy is actually happening in a running pipeline, insert a pad probe and inspect the buffer's memory type:

```c
/* Inside a pad probe callback */
guint n_mem = gst_buffer_n_memory(buf);
for (guint i = 0; i < n_mem; i++) {
    GstMemory *mem = gst_buffer_peek_memory(buf, i);
    if (gst_is_dmabuf_memory(mem)) {
        int fd = gst_dmabuf_memory_get_fd(mem);
        g_print("Buffer %u: DMA-BUF fd=%d\n", i, fd);
    }
}
```

A complete zero-copy decode-to-display pipeline on an AMD or Intel system:

```bash
gst-launch-1.0 filesrc location=video.mp4 \
    ! qtdemux \
    ! vaapih264dec \
    ! waylandsink sync=false
```

The `vaapih264dec` element outputs VA-API `VASurface` DMA-BUF buffers; `waylandsink` uses `zwp_linux_dmabuf_v1` to submit them to the Wayland compositor. The decoded pixels never reach main memory between decode and compositor compositing.

A hardware transcode pipeline:

```bash
gst-launch-1.0 filesrc location=input.mp4 \
    ! qtdemux \
    ! vaapih264dec \
    ! vaapihevcenc \
    ! matroskamux \
    ! filesink location=output.mkv
```

`vaapihevcenc` consumes the VA-API decoded surface directly as its input, using it as the encode source — no format conversion or CPU copy required when both decode and encode share the same `VADisplay`.

### Screen Recording via PipeWire

Screen recording that combines PipeWire screen capture with hardware encode:

```bash
gst-launch-1.0 pipewiresrc \
    ! video/x-raw,format=BGRA \
    ! vaapih264enc \
    ! mp4mux \
    ! filesink location=screenrec.mp4
```

The `pipewiresrc` element receives frames from a PipeWire screen capture node as DMA-BUF buffers; `vaapih264enc` ingests those buffers directly as VA-API surfaces if the caps negotiation confirms `memory:DMABuf` is supported end-to-end.

Debugging: set `GST_DEBUG=3` for verbose caps negotiation logs; `gst-inspect-1.0 vaapih264dec` shows element properties and pad templates; `GST_DEBUG_DUMP_DOT_DIR=/tmp/gst` writes pipeline graph `.dot` files after state changes that can be visualised with Graphviz.

---

## 7. PipeWire: The Linux Multimedia Session Manager

PipeWire is a multimedia session manager and IPC framework that replaced PulseAudio as the standard audio daemon on Fedora 34+, Ubuntu 22.10+, and most modern Linux desktops. Its architecture goes well beyond audio: it provides JACK-compatible low-latency pro-audio, and — critically for this chapter — serves as the standard broker for screen capture and camera access on Wayland desktops. Applications do not talk to V4L2 cameras or compositor screencopy APIs directly; they connect to a PipeWire node and receive buffers brokered through the PipeWire daemon.

### The Graph Model

PipeWire organises media flow as a graph of *nodes*, *ports*, and *links*. A node is a source or sink of media — a camera node, a screencopy source, an application playback sink. Each node has typed input and output *ports*. *Links* are directed connections between ports established by the session manager (wireplumber). The PipeWire daemon acts as the graph router; individual media nodes can live in separate processes and share memory via DMA-BUF without the data passing through the daemon process.

The key API objects from a client perspective: `pw_main_loop` wraps an `epoll` fd and drives all callbacks; `pw_context` owns the graph description and the connection to the PipeWire daemon socket; `pw_core` is the client's live connection to the daemon (a single client can have multiple cores on different sockets).

### SPA Plugin Architecture

PipeWire's internal hardware backends are SPA (Simple Plugin API) plugins loaded at runtime from `/usr/lib/spa-0.2/`. The SPA node interface drives processing: `spa_node_process()` executes one graph iteration, pulling or pushing one batch of buffers. Capabilities and format parameters are encoded in `spa_pod` objects — a binary type-length-value format analogous in role to GStreamer caps but structured as a compact binary encoding that can carry optional fields without version negotiation.

The V4L2 SPA plugin (`libspa-v4l2.so`) wraps a V4L2 camera device as a SPA source node, handling `V4L2_MEMORY_DMABUF` buffer allocation internally. The DRM screencopy SPA plugin connects to the compositor's screencopy protocol (`zwlr_screencopy_manager_v1` on wlroots, `ext_image_copy_capture_v1` on newer compositors) and feeds frames into a PipeWire source node.

### pw_stream for Screen Capture and Camera Consumers

`pw_stream` is the high-level client API for connecting to a PipeWire source or sink node. The essential pattern for a screen capture consumer:

```c
#include <pipewire/pipewire.h>
#include <spa/param/video/format-utils.h>

/* Format negotiation: build a SPA pod offering BGRA + DMA-BUF */
uint8_t params_buf[1024];
struct spa_pod_builder b = SPA_POD_BUILDER_INIT(params_buf, sizeof(params_buf));
struct spa_pod *params[1];

params[0] = spa_pod_builder_add_object(&b,
    SPA_TYPE_OBJECT_Format, SPA_PARAM_EnumFormat,
    SPA_FORMAT_mediaType,       SPA_POD_Id(SPA_MEDIA_TYPE_video),
    SPA_FORMAT_mediaSubtype,    SPA_POD_Id(SPA_MEDIA_SUBTYPE_raw),
    SPA_FORMAT_VIDEO_format,    SPA_POD_Id(SPA_VIDEO_FORMAT_BGRA),
    SPA_FORMAT_VIDEO_modifier,
        SPA_POD_CHOICE_ENUM_Long(3,
            DRM_FORMAT_MOD_LINEAR,
            DRM_FORMAT_MOD_LINEAR,        /* preferred */
            I915_FORMAT_MOD_X_TILED),     /* acceptable alternative */
    SPA_FORMAT_VIDEO_size,      SPA_POD_CHOICE_RANGE_Rectangle(
                                    &SPA_RECTANGLE(1920,1080),
                                    &SPA_RECTANGLE(1,1),
                                    &SPA_RECTANGLE(7680,4320)),
    SPA_FORMAT_VIDEO_framerate, SPA_POD_CHOICE_RANGE_Fraction(
                                    &SPA_FRACTION(60,1),
                                    &SPA_FRACTION(1,1),
                                    &SPA_FRACTION(240,1)));

/* Create stream and connect */
struct pw_stream *stream = pw_stream_new(core, "screen-capture", 
    pw_properties_new(
        PW_KEY_MEDIA_TYPE,     "Video",
        PW_KEY_MEDIA_CATEGORY, "Capture",
        PW_KEY_MEDIA_ROLE,     "Screen",
        NULL));

pw_stream_connect(stream,
    PW_DIRECTION_INPUT,
    portal_node_id,              /* obtained from xdg-desktop-portal */
    PW_STREAM_FLAG_AUTOCONNECT,
    params, 1);
```

When format negotiation completes, the `on_param_changed` callback fires with the confirmed format pod. Parse it to extract the agreed modifier:

```c
static void on_param_changed(void *data, uint32_t id, const struct spa_pod *param) {
    if (id != SPA_PARAM_Format || param == NULL) return;

    struct spa_video_info_raw info;
    spa_format_video_raw_parse(param, &info);
    /* info.format, info.size.width, info.size.height are now confirmed */

    /* Check for negotiated modifier */
    const struct spa_pod_prop *mod_prop =
        spa_pod_find_prop(param, NULL, SPA_FORMAT_VIDEO_modifier);
    if (mod_prop) {
        uint64_t modifier;
        spa_pod_get_long(&mod_prop->value, (int64_t*)&modifier);
        /* modifier is the DRM format modifier for this stream — use it when
           importing the DMA-BUF into EGL or Vulkan */
    }
}
```

When a frame arrives, the `on_process` callback dequeues the buffer:

```c
static void on_process(void *data) {
    struct pw_buffer *b = pw_stream_dequeue_buffer(stream);
    if (!b) return;

    struct spa_buffer *buf = b->buffer;
    if (buf->datas[0].type != SPA_DATA_DmaBuf) {
        /* Fallback: shared memory — copy required */
        pw_stream_queue_buffer(stream, b);
        return;
    }

    int dmabuf_fd = buf->datas[0].fd;
    uint32_t width   = /* from confirmed format */;
    uint32_t height  = /* from confirmed format */;

    /* Import into EGL: */
    EGLint attribs[] = {
        EGL_WIDTH,                      width,
        EGL_HEIGHT,                     height,
        EGL_LINUX_DRM_FOURCC_EXT,       DRM_FORMAT_ARGB8888,
        EGL_DMA_BUF_PLANE0_FD_EXT,      dmabuf_fd,
        EGL_DMA_BUF_PLANE0_OFFSET_EXT,  0,
        EGL_DMA_BUF_PLANE0_PITCH_EXT,   buf->datas[0].chunk->stride,
        EGL_DMA_BUF_PLANE0_MODIFIER_LO_EXT, (uint32_t)(negotiated_modifier & 0xFFFFFFFF),
        EGL_DMA_BUF_PLANE0_MODIFIER_HI_EXT, (uint32_t)(negotiated_modifier >> 32),
        EGL_NONE
    };
    EGLImage img = eglCreateImageKHR(egl_display, EGL_NO_CONTEXT,
                                     EGL_LINUX_DMA_BUF_EXT, NULL, attribs);
    /* Use img as a GL texture or pass to encoder */

    pw_stream_queue_buffer(stream, b);  /* return buffer to producer */
}
```

When both sides negotiate `SPA_DATA_DmaBuf` with a specific modifier, the DMA-BUF fd is imported directly into GPU memory as an EGL image or a Vulkan image — the compositor's framebuffer never reaches CPU RAM in the capture-to-encode path.

### DMA-BUF Modifier Negotiation

The `SPA_FORMAT_VIDEO_modifier` field in the format pod carries a DRM format modifier choice. If the consumer includes a `SPA_POD_CHOICE_ENUM_Long` listing acceptable modifiers and the producer agrees on one, both sides commit to that modifier. If modifiers are not negotiated (the pod omits the modifier field), PipeWire may fall back to `SPA_DATA_MemFd` (shared memory with CPU copy). For GPU-accelerated consumers (encoders, display elements), the modifier negotiation step is mandatory to ensure the buffer can be imported without detiling.

### wireplumber: Session and Policy Manager

PipeWire is policy-free by design. wireplumber is the daemon that implements session management policy as Lua scripts under `/usr/share/wireplumber/`. Policy rules control which nodes are linked, which applications have access to which devices, and default sink/source selection. For screen capture, wireplumber enforces portal-granted permissions: when an application receives a PipeWire node id via `org.freedesktop.portal.ScreenCast`, wireplumber verifies the access token before allowing the link. This means screen capture on Wayland requires user consent via a compositor dialog but needs no elevated privileges — the security boundary is the portal token validated by wireplumber.

Diagnostics: `wpctl status` prints the current graph; `wpctl inspect <id>` shows node properties; `pw-dump` outputs the full graph as JSON.

### The xdg-desktop-portal Screen Capture Flow

The complete Wayland screen capture path, end-to-end:

1. Application calls `org.freedesktop.portal.ScreenCast.CreateSession` then `SelectSources` and `Start` via D-Bus.
2. The portal implementation (`xdg-desktop-portal-wlr`, `xdg-desktop-portal-gnome`, or `xdg-desktop-portal-kde`) calls the compositor's screencopy protocol on the application's behalf.
3. The portal creates a PipeWire source node connected to the screencopy output; it returns a `pipewire_node_id` and a `restore_token` to the application over D-Bus.
4. The application opens a `pw_stream` and calls `pw_stream_connect` with the `pipewire_node_id` as the target; wireplumber links the screencopy source to the application's input stream.
5. Frames arrive as `SPA_DATA_DmaBuf` fds in the `on_process` callback.

The complete path — compositor GPU framebuffer → DRM DMA-BUF → PipeWire screencopy node → application DMA-BUF fd → encoder or display — involves zero CPU copies when DMA-BUF modifier negotiation succeeds. OBS Studio uses this path on Wayland via `xdg-desktop-portal` and PipeWire screen capture.

---

## 8. Practical: Hardware-Accelerated Transcoding Pipeline

### Use Case: H.264 4K to AV1 on AMD or Intel GPU

Hardware transcoding is one of the highest-value use cases for VA-API. The goal is to keep the entire decode-encode-output pipeline on the GPU, with the CPU involved only for I/O and demuxing.

#### ffmpeg VA-API Path

```bash
ffmpeg -hwaccel vaapi \
       -hwaccel_device /dev/dri/renderD128 \
       -hwaccel_output_format vaapi \
       -i input.mp4 \
       -vf 'scale_vaapi=w=3840:h=2160' \
       -c:v av1_vaapi \
       -rc_mode VBR \
       -b:v 8M \
       -maxrate 12M \
       output.mkv
```

Flag-by-flag breakdown: `-hwaccel vaapi` enables VA-API accelerated decode; `-hwaccel_device /dev/dri/renderD128` selects the render node (check `vainfo --display drm` to find the right one); `-hwaccel_output_format vaapi` instructs ffmpeg to keep the decoded frames in VA-API surface memory rather than copying to system memory; `scale_vaapi` performs GPU-side scaling; `av1_vaapi` is the VA-API AV1 encoder (requires VCN 4.0 on AMD or Intel Arc). Without `-hwaccel_output_format vaapi`, ffmpeg would copy each decoded frame to system memory before feeding the encoder, negating the hardware decode benefit.

Verify AV1 encode availability before invoking:

```bash
vainfo 2>/dev/null | grep -i av1
```

Expected output on an RDNA3 system: `VAProfileAV1Profile0: VAEntrypointEncSlice`.

#### GStreamer Equivalent

```bash
gst-launch-1.0 filesrc location=input.mp4 \
    ! qtdemux \
    ! vaapih264dec \
    ! vaapiav1enc bitrate=8000 \
    ! matroskamux \
    ! filesink location=output.mkv
```

The GStreamer pipeline is more transparent about the zero-copy path: `vaapih264dec` outputs VA-API surfaces; `vaapiav1enc` consumes them as-is because both elements share the same `VADisplay`. When running GStreamer with `GST_DEBUG=vaapi*:4`, the log shows "Using DMABuf" when the surface is shared without copy.

### Identifying the Bottleneck

On a balanced 4K 60fps pipeline the limiting factor is usually encode throughput, not decode. Measure encode queue depth with `vaQuerySurfaceStatus` polling: a queue that drains slowly indicates the encode engine is saturated. For latency-sensitive screen recording, switch to `VAEntrypointEncSliceLP` (the low-power encode engine), which trades compression efficiency for lower latency.

### Common Errors

`VA_STATUS_ERROR_UNSUPPORTED_PROFILE` typically means the profile/entrypoint combination was not checked against `vainfo` before context creation. `VA_STATUS_ERROR_RESOLUTION_NOT_SUPPORTED` occurs when the codec's maximum resolution (from `vaGetConfigAttributes` with `VAConfigAttribMaxPictureWidth`) is exceeded. For debugging, `vaDeriveImage` followed by `vaMapBuffer` allows inspecting a decoded surface's pixels directly from C — useful for confirming the decoder is producing correct output before chasing display issues.

---

## 9. libcamera: Modern Camera Stack for Linux

### The V4L2 Media Controller Problem

Modern camera subsystems on ARM SoCs are not single-device pipelines. A typical ISP (Image Signal Processor) chain consists of: a MIPI CSI-2 sensor, a CSI-2 receiver block, an ISP (pixel pipeline including demosaicing, colour correction, noise reduction), a scaler, and a DMA output engine. Each of these is a separate hardware block. A single V4L2 `/dev/videoN` node cannot represent this topology: there is no standard way to configure sensor crop regions, ISP colour matrix coefficients, or scaler ratios through raw V4L2 ioctls. The Media Controller API was designed to fill this gap.

### The Media Controller API

The Media Controller kernel API exposes the ISP pipeline as a graph of *entities*, *pads*, and *links*, accessible through `/dev/mediaN`:

- `media_entity`: represents one hardware component (sensor, CSI-2 receiver, ISP, scaler, DMA engine)
- `media_pad`: a typed input/output connector on an entity; pads carry media bus formats (MIPI CSI-2 format codes, Bayer patterns, bit depths)
- `media_link`: a directed connection between two pads, with enable/disable control via `MEDIA_IOC_SETUP_LINK`

The full topology is retrieved in a single ioctl: `MEDIA_IOC_G_TOPOLOGY` fills a `media_v2_topology` struct with arrays of all entities, pads, interfaces, and links. The `media-ctl` command-line tool wraps these ioctls:

```bash
# List the topology of an RKISP1 pipeline on RK3399
media-ctl -d /dev/media0 -p

# Set the sensor output format on pad 0
media-ctl -d /dev/media0 \
    --set-v4l2 '"ov5695 4-0036":0[fmt:SBGGR10_1X10/2592x1944]'

# Enable the link from sensor to CSI-2 receiver
media-ctl -d /dev/media0 --links '"ov5695 4-0036":0 -> "rockchip-sy-mipi-dphy":0[1]'
```

V4L2 subdevice nodes (`/dev/v4l-subdevN`) accept per-entity format and selection rectangle configuration via `VIDIOC_SUBDEV_S_FMT` and `VIDIOC_SUBDEV_S_SELECTION`. These ioctls carry a pad index so the same subdevice can have different formats on its sink and source pads (expressing the format transformation performed by the hardware block).

### libcamera Architecture

libcamera is a C++ userspace library that abstracts the complexity of ISP pipeline configuration. Its fundamental abstraction is the *pipeline handler* — a per-SoC class that encodes knowledge of one hardware ISP topology. Pipeline handlers live at `src/libcamera/pipeline/` in the libcamera source tree: `rkisp1`, `ipu3`, `rpi/vc4`, `simple` (for generic V4L2-only cameras), and `qcom_camss` (in progress).

The pipeline handler `match()` method is called at startup to determine if the handler applies to the detected Media Controller topology. `configure()` programs the ISP pipeline by calling the equivalent of the `media-ctl` and `VIDIOC_SUBDEV_S_FMT` ioctls internally. `start()` begins streaming; `stop()` flushes and halts.

libcamera's buffer model: a `FrameBuffer` is backed by one or more DMA-BUF fds. Applications allocate `FrameBuffer` pools via `FrameBufferAllocator`, then create `Request` objects associating buffers with camera streams. Completed requests arrive via a signal callback:

```cpp
#include <libcamera/libcamera.h>
using namespace libcamera;

std::shared_ptr<Camera> camera = /* obtain from CameraManager */;
std::unique_ptr<CameraConfiguration> config =
    camera->generateConfiguration({ StreamRole::VideoRecording });

config->at(0).pixelFormat = formats::NV12;
config->at(0).size = { 1920, 1080 };
camera->configure(config.get());

FrameBufferAllocator allocator(camera);
allocator.allocate(config->at(0).stream());

camera->requestCompleted.connect([](Request *req) {
    const FrameBuffer *buf = req->buffers().begin()->second;
    /* buf->planes()[0].fd is a DMA-BUF fd — import into Vulkan or pass to encoder */
    int dmabuf_fd = buf->planes()[0].fd.get();
    /* ... */
    req->reuse(Request::ReuseBuffers);
    camera->queueRequest(req);
});

camera->start();
```

`buf->planes()[0].fd.get()` returns the raw DMA-BUF file descriptor. The camera never writes to CPU-mapped memory; the pixel data travels from the ISP output DMA engine directly to the GPU-importable buffer.

### IPA Plugins

libcamera's IPA (Image Processing Algorithm) plugins implement per-sensor 3A (auto exposure, auto white balance, auto focus) and lens shading correction in a sandboxed isolated process. The IPA receives statistics from the ISP hardware (histogram, auto-focus statistics) and writes back tuning parameters (exposure time, analogue gain, colour correction matrix). The sandbox uses a Mojo-style IPC channel to prevent a compromised IPA from affecting the application. IPA source lives at `src/ipa/`; separate modules exist for `rkisp1`, `ipu3`, and `rpi`.

### Platform-Specific Details

**Raspberry Pi** (`rpi/vc4`): the pipeline handler issues mailbox commands to the VideoCore IV VPU firmware for ISP statistics and tuning. The Unicam CSI-2 receiver is a V4L2 subdevice. The IPA handles tuning for OV5647, IMX219, IMX477, and IMX708 sensors. libcamera on RPi is the standard camera stack for Raspberry Pi OS.

**Rockchip RKISP1** (`rkisp1`): present on RK3399; libcamera configures the ISP via V4L2 subdev ioctls. The IPA implements RKISP1-specific 3A. RK3588's RKISP2 is supported from libcamera 0.3 (2024).

**Intel IPU3** (`ipu3`): present on 7th–10th gen Intel laptops. The pipeline handler configures both the ImgU (image processing unit) and CIO2 (MIPI CSI-2 receiver) subdevices. Stable since libcamera 0.1 on mainline kernel.

### libcamera to GPU: Zero-Copy Vulkan Import

A `FrameBuffer` DMA-BUF fd from libcamera can be imported directly into a Vulkan image:

```cpp
/* After requestCompleted callback */
int dmabuf_fd = buf->planes()[0].fd.get();

VkExternalMemoryImageCreateInfo ext_img_info = {
    .sType       = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
VkImageDrmFormatModifierExplicitCreateInfoEXT mod_info = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_EXPLICIT_CREATE_INFO_EXT,
    .drmFormatModifier = DRM_FORMAT_MOD_LINEAR,  /* from ISP config */
    .drmFormatModifierPlaneCount = 2,            /* NV12: Y + UV */
    .pPlaneLayouts = (VkSubresourceLayout[]) {
        { .offset = 0,           .rowPitch = 1920 },
        { .offset = 1920*1080,   .rowPitch = 1920 },
    },
};
VkImageCreateInfo img_info = {
    .sType     = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext     = &ext_img_info,   /* chain mod_info for tiled inputs */
    .format    = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,  /* NV12 */
    .extent    = { 1920, 1080, 1 },
    .tiling    = VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT,
    .usage     = VK_IMAGE_USAGE_SAMPLED_BIT,
};
vkCreateImage(device, &img_info, NULL, &image);

VkImportMemoryFdInfoKHR import_info = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = dmabuf_fd,
};
VkMemoryAllocateInfo alloc_info = { .pNext = &import_info, /* ... */ };
vkAllocateMemory(device, &alloc_info, NULL, &memory);
vkBindImageMemory(device, image, memory, 0);
/* image is now a Vulkan-accessible view of the camera frame — run compute shaders on it */
```

This pattern is used by Monado (OpenXR runtime) for camera-based tracking and by any GPU compute pipeline that processes live camera input.

---

## 10. Vulkan Video: The Strategic Hardware Video API

### Why Vulkan Video

VA-API's hardware implementation is per-driver and under-specified: the VA-API specification does not mandate internal command buffer formats, synchronisation semantics, or error recovery behaviour. This leads to driver-specific bugs and capability gaps — a `vaEndPicture` implementation on Intel may behave differently from the same call on AMD when slices are split across multiple buffers. Vulkan Video is Khronos's response: a formally specified, cross-vendor API for hardware video decode and encode that uses the same object model, synchronisation primitives, command buffer architecture, and memory management as Vulkan graphics and compute.

The video decode extensions were promoted from `EXT` to `KHR` in Vulkan specification version 1.3.238 (December 2022). Video encode extensions (`VK_KHR_video_encode_h264`, `VK_KHR_video_encode_h265`, `VK_KHR_video_encode_av1`) were promoted to KHR in Vulkan 1.3.275 (2024).

### The Vulkan Video Object Model

**`VkVideoSessionKHR`** is the stateful codec context. It is created with `vkCreateVideoSessionKHR`, specifying the video profile, picture format, maximum coded extent, DPB array size, and queue family index. The video profile is codec-specific: `VkVideoDecodeH264ProfileInfoKHR` for H.264, `VkVideoDecodeH265ProfileInfoKHR` for H.265, `VkVideoDecodeAV1ProfileInfoKHR` for AV1.

**`VkVideoSessionParametersKHR`** holds per-session codec parameter sets that persist across frames: SPS/PPS for H.264, VPS/SPS/PPS for H.265, sequence header for AV1. These are created once per stream and referenced from every decode command.

**DPB management**: the application allocates a pool of `VkImage` objects as DPB (Decoded Picture Buffer) slots at session creation time. Each DPB image is identified in decode commands via `VkVideoPictureResourceInfoKHR`, which wraps a `VkImageView`. The DPB image array can be backed by either a single layered image or separate images per slot (if `VK_VIDEO_CAPABILITY_SEPARATE_REFERENCE_IMAGES_BIT_KHR` is supported).

**Video command buffers**: video operations are recorded into standard `VkCommandBuffer` objects using video-specific commands:

```c
/* Minimal H.264 decode command buffer (pedagogical) */
VkVideoBeginCodingInfoKHR begin = {
    .sType            = VK_STRUCTURE_TYPE_VIDEO_BEGIN_CODING_INFO_KHR,
    .videoSession     = video_session,
    .videoSessionParameters = session_params,  /* SPS/PPS */
    .referenceSlotCount = num_ref_slots,
    .pReferenceSlots  = ref_slots,             /* active DPB entries */
};
vkCmdBeginVideoCodingKHR(cmd_buf, &begin);

VkVideoDecodeH264PictureInfoKHR h264_pic = {
    .sType         = VK_STRUCTURE_TYPE_VIDEO_DECODE_H264_PICTURE_INFO_KHR,
    .pStdPictureInfo = &std_picture_info,  /* StdVideoDecodeH264PictureInfo */
    .sliceCount    = num_slices,
    .pSliceOffsets = slice_offsets,
};
VkVideoDecodeInfoKHR decode_info = {
    .sType          = VK_STRUCTURE_TYPE_VIDEO_DECODE_INFO_KHR,
    .pNext          = &h264_pic,
    .srcBuffer      = bitstream_buf,       /* VkBuffer with H.264 NAL data */
    .srcBufferOffset = 0,
    .srcBufferRange = nal_size,
    .dstPictureResource = { .imageViewBinding = output_image_view },
    .pSetupReferenceSlot = &setup_slot,    /* DPB slot for this frame */
    .referenceSlotCount  = num_ref_slots,
    .pReferenceSlots     = ref_slots,
};
vkCmdDecodeVideoKHR(cmd_buf, &decode_info);
vkCmdEndVideoCodingKHR(cmd_buf, &(VkVideoEndCodingInfoKHR){ .sType = ... });
```

The command buffer is submitted to a `VkQueue` supporting `VK_QUEUE_VIDEO_DECODE_BIT_KHR`. Synchronisation uses the standard `VkFence`/`VkSemaphore` model, allowing the decode queue to synchronise with the graphics or compute queue without special-casing.

### Mesa/RADV Vulkan Video Implementation

RADV (the open-source AMD Vulkan driver in Mesa) implements Vulkan Video decode using the VCN hardware block via the `amdgpu` kernel DRM driver. The implementation lives in `src/amd/vulkan/radv_video.c` in the Mesa source tree.

- H.264 and H.265 decode: Mesa 23.0+, RDNA1+
- AV1 decode: Mesa 24.1+, RDNA2+ (VCN 3.0+)

The Intel ANV (Anv Vulkan) driver has Vulkan Video H.264 and H.265 decode in active development targeting Tigerlake+ using the HCP hardware pipeline. AV1 decode on Intel via ANV is targeted for a future Mesa release.

NVIDIA's proprietary Vulkan driver exposes `VK_KHR_video_decode_h264`, `VK_KHR_video_decode_h265`, and `VK_KHR_video_decode_av1` on Maxwell+, making it the most complete Vulkan Video decode implementation currently available on Linux.

### GStreamer vkvideo Plugin

The GStreamer `vkvideo` plugin in `gst-plugins-bad` provides `vkh264dec`, `vkh265dec`, and `vkav1dec` elements backed by Vulkan Video. These elements are in active development in the GStreamer main branch but were not in a stable release as of GStreamer 1.24 (March 2024); they are targeted for GStreamer 1.26. The encode extensions (`VK_KHR_video_encode_h264`, `VK_KHR_video_encode_h265`, `VK_KHR_video_encode_av1`) follow the same architecture and are the primary blocker for a complete GStreamer Vulkan Video pipeline replacement.

The transition from VA-API GStreamer elements to Vulkan Video elements is gated on driver completeness. RADV's VA-API path via `radeonsi_drv_video.so` is more broadly tested today; the `vkvideo` elements are expected to become the default on Mesa/RADV once Vulkan Video encode reaches parity.

---

## Integrations

**Chapter 1 (DRM Architecture)**: VA-API drivers access GPU memory through DRM render nodes (`/dev/dri/renderD128`). `vaExportSurfaceHandle` calls `DRM_IOCTL_PRIME_HANDLE_TO_FD` under the hood to convert the GEM handle into a DMA-BUF fd. The render node security model (Chapter 1) controls which processes can call VA-API.

**Chapter 4 (GPU Memory Management)**: The DMA-BUF mechanism is the thread running through this entire chapter — every API boundary discussion (VA-API → EGL, V4L2 → Vulkan, PipeWire → encoder) crosses the same DMA-BUF import/export protocol. DRM format modifiers (Chapter 4) must match at every import boundary; a modifier mismatch causes corruption or a detiling copy.

**Chapter 5 (x86 GPU Drivers)**: The `amdgpu` driver exposes the VCN hardware through `AMDGPU_HW_IP_VCN_DEC`/`_ENC` queue types; `i915` exposes MFX (Quick Sync) and HCP hardware. These kernel drivers are what Mesa's VA-API state tracker and RADV's Vulkan Video implementation submit hardware commands to.

**Chapter 6 (ARM GPU Drivers)**: V4L2 stateless codecs are the primary decode path for Rockchip, Allwinner, and Raspberry Pi SoCs. The RKVDEC driver (`drivers/media/platform/rockchip/rkvdec/`) and Cedrus driver use the same GEM/DMA-BUF infrastructure as the ARM GPU drivers. Section 9's Media Controller ISP coverage extends the Chapter 6 camera subsystem discussion.

**Chapter 13 (Gallium3D)**: Mesa's VA-API front end uses the Gallium `pipe_video_codec` interface defined in `src/gallium/include/pipe/p_video_codec.h`. The `radeonsi` and `iris` Gallium drivers implement this interface using the same pipe driver backend as their OpenGL and OpenCL paths.

**Chapter 21 (wlroots)**: wlroots implements `zwlr_screencopy_manager_v1`, one end of the PipeWire screen capture pipeline. The screencopy protocol copy from the wlroots compositor framebuffer is the source DMA-BUF in the xdg-desktop-portal/PipeWire screen capture chain.

**Chapter 22 (Production Compositors)**: gamescope uses VA-API surfaces for video overlay planes. The compositor implements the screencopy protocol (`zwlr_screencopy_manager_v1` or `ext_image_copy_capture_v1`) that feeds the PipeWire screencopy SPA plugin.

**Chapter 24 (Vulkan/EGL)**: Section 4 of Chapter 24 describes importing DMA-BUF into EGL (`eglCreateImageKHR` with `EGL_LINUX_DMA_BUF_EXT`). Every DMA-BUF import example in this chapter uses that same mechanism. The `VK_EXT_external_memory_fd` Vulkan extension is the Vulkan-side equivalent.

**Chapter 25 (GPU Compute)**: The V4L2 camera → DMA-BUF → Vulkan compute pattern in Section 5 and the libcamera → DMA-BUF → Vulkan compute pattern in Section 9 both apply the same buffer-sharing infrastructure described in Chapter 25.

**Chapter 27 (VR/AR)**: Monado uses V4L2 cameras for inside-out tracking. The DMA-BUF path from camera sensor to tracking compute shader uses the same infrastructure as Sections 5 and 9. libcamera's pipeline handler architecture is directly applicable to Monado's camera integration.

---

## References

1. libva API documentation — [https://intel.github.io/libva/](https://intel.github.io/libva/)
2. libva `va_drmcommon.h` (VADRMPRIMESurfaceDescriptor) — [https://github.com/intel/libva/blob/master/va/va_drmcommon.h](https://github.com/intel/libva/blob/master/va/va_drmcommon.h)
3. libva `va.h` (vaExportSurfaceHandle) — [https://github.com/intel/libva/blob/master/va/va.h](https://github.com/intel/libva/blob/master/va/va.h)
4. Mesa VA-API state tracker source — [https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/va](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/va)
5. Mesa Gallium pipe_video_codec interface — [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/gallium/include/pipe/p_video_codec.h](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/gallium/include/pipe/p_video_codec.h)
6. intel-media-driver (iHD) — [https://github.com/intel/media-driver](https://github.com/intel/media-driver)
7. nvidia-vaapi-driver — [https://github.com/elFarto/nvidia-vaapi-driver](https://github.com/elFarto/nvidia-vaapi-driver)
8. V4L2 Request API documentation — [https://docs.kernel.org/userspace-api/media/mediactl/request-api.html](https://docs.kernel.org/userspace-api/media/mediactl/request-api.html)
9. V4L2 stateless decoder interface — [https://www.kernel.org/doc/html/v6.9/userspace-api/media/v4l/dev-stateless-decoder.html](https://www.kernel.org/doc/html/v6.9/userspace-api/media/v4l/dev-stateless-decoder.html)
10. V4L2 stateless codec control reference — [https://docs.kernel.org/userspace-api/media/v4l/ext-ctrls-codec-stateless.html](https://docs.kernel.org/userspace-api/media/v4l/ext-ctrls-codec-stateless.html)
11. MEDIA_IOC_REQUEST_ALLOC documentation — [https://docs.kernel.org/userspace-api/media/mediactl/media-ioc-request-alloc.html](https://docs.kernel.org/userspace-api/media/mediactl/media-ioc-request-alloc.html)
12. Rockchip RKVDEC kernel driver — [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/media/platform/rockchip/rkvdec](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/media/platform/rockchip/rkvdec)
13. GStreamer v4l2codecs plugin — [https://gitlab.freedesktop.org/gstreamer/gstreamer/-/tree/main/subprojects/gst-plugins-bad/sys/v4l2codecs](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/tree/main/subprojects/gst-plugins-bad/sys/v4l2codecs)
14. GStreamer vaapidecodebin documentation — [https://gstreamer.freedesktop.org/documentation/vaapi/vaapidecodebin.html](https://gstreamer.freedesktop.org/documentation/vaapi/vaapidecodebin.html)
15. PipeWire DMA-BUF sharing documentation — [https://docs.pipewire.org/page_dma_buf.html](https://docs.pipewire.org/page_dma_buf.html)
16. PipeWire pw_stream API — [https://docs.pipewire.org/group__pw__stream.html](https://docs.pipewire.org/group__pw__stream.html)
17. PipeWire video capture tutorial — [https://docs.pipewire.org/page_tutorial5.html](https://docs.pipewire.org/page_tutorial5.html)
18. xdg-desktop-portal ScreenCast interface — [https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html)
19. wireplumber documentation — [https://pipewire.pages.freedesktop.org/wireplumber/](https://pipewire.pages.freedesktop.org/wireplumber/)
20. Vulkan Video extensions overview (Khronos) — [https://www.khronos.org/blog/an-introduction-to-vulkan-video](https://www.khronos.org/blog/an-introduction-to-vulkan-video)
21. Khronos releases AV1 decode and H.264/H.265 encode in Vulkan Video — [https://www.khronos.org/blog/khronos-releases-vulkan-video-av1-decode-extension-vulkan-sdk-now-supports-h.264-h.265-encode](https://www.khronos.org/blog/khronos-releases-vulkan-video-av1-decode-extension-vulkan-sdk-now-supports-h.264-h.265-encode)
22. VK_KHR_video_decode_h264 specification — [https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_video_decode_h264.html](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_video_decode_h264.html)
23. RADV Vulkan Video AV1 decode (Phoronix) — [https://www.phoronix.com/news/RADV-VK_KHR_video_decode_av1](https://www.phoronix.com/news/RADV-VK_KHR_video_decode_av1)
24. RADV Vulkan Video implementation — [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_video.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_video.c)
25. GStreamer vkvideo plugin — [https://gitlab.freedesktop.org/gstreamer/gstreamer/-/tree/main/subprojects/gst-plugins-bad/ext/vulkan/vkvideo](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/tree/main/subprojects/gst-plugins-bad/ext/vulkan/vkvideo)
26. libcamera documentation — [https://libcamera.org/api-html/](https://libcamera.org/api-html/)
27. libcamera pipeline handlers source — [https://git.libcamera.org/libcamera/libcamera.git/tree/src/libcamera/pipeline](https://git.libcamera.org/libcamera/libcamera.git/tree/src/libcamera/pipeline)
28. Media Controller API documentation — [https://www.kernel.org/doc/html/latest/userspace-api/media/mediactl/media-controller.html](https://www.kernel.org/doc/html/latest/userspace-api/media/mediactl/media-controller.html)
29. V4L2 subdevice API documentation — [https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-subdev.html](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-subdev.html)
30. Collabora: Zero-copy video on Linux — [https://www.collabora.com/news-and-blog/blog/2020/03/23/zero-copy-video-on-linux/](https://www.collabora.com/news-and-blog/blog/2020/03/23/zero-copy-video-on-linux/)
31. LWN: The V4L2 stateless video codec API — [https://lwn.net/Articles/786714/](https://lwn.net/Articles/786714/)
32. airlied blog: RADV Vulkan AV1 video decode status — [https://airlied.blogspot.com/2024/02/radv-vulkan-av1-video-decode-status.html](https://airlied.blogspot.com/2024/02/radv-vulkan-av1-video-decode-status.html)
