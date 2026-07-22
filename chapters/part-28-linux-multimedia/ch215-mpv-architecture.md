# Chapter 215: MPV Architecture: libmpv, libplacebo, and GPU Video Output

> **Part**: Part XXVIII — Linux Multimedia
> **Audience**: Graphics application developers and systems engineers who want to understand or extend mpv's GPU-accelerated video pipeline.
> **Status**: First draft — 2026-07-22

mpv is the dominant GPU-accelerated video player on the Linux desktop, distinguished from legacy players by its event-driven client API, zero-copy hardware decode path, and a shader-based rendering engine that has since been spun out as an independent library. Where VLC and MPlayer expose monolithic blobs of state, mpv exposes a composable architecture: a libmpv embedding API, a pluggable video-output (VO) layer, and a rendering backend (libplacebo) that can be reused independently. This chapter traces the full data path from demuxer to display, covering every major component an application developer or systems engineer needs to understand to embed, extend, or debug mpv on a modern Linux system.

---

## Table of Contents

- [1. MPV Project Overview](#1-mpv-project-overview)
- [2. libmpv API](#2-libmpv-api)
- [3. Demuxer Layer](#3-demuxer-layer)
- [4. Decoder Pipeline and Zero-Copy DMA-buf](#4-decoder-pipeline-and-zero-copy-dma-buf)
- [5. Video Output Architecture](#5-video-output-architecture)
- [6. libplacebo: Shader-Based Video Processing](#6-libplacebo-shader-based-video-processing)
- [7. Shader System and Custom User Shaders](#7-shader-system-and-custom-user-shaders)
- [8. HDR and Colour Management](#8-hdr-and-colour-management)
- [9. Audio Pipeline and A/V Sync](#9-audio-pipeline-and-av-sync)
- [10. Wayland Integration](#10-wayland-integration)
- [11. Performance Profiling](#11-performance-profiling)
- [12. Extension Points: Scripting and IPC](#12-extension-points-scripting-and-ipc)
- [Integrations](#integrations)

---

## 1. MPV Project Overview

mpv descends from MPlayer via a two-step fork. The first fork, mplayer2 (2011), cleaned up MPlayer's aging codebase, dropped the ancient GUI front-end, and began replacing the monolithic option system. The second fork that became mpv (2012) went further: it stripped out nearly all legacy drivers, rewrote the option and property system, added a Lua scripting layer, introduced the libmpv embedding API, and replaced the video-output architecture with a modern GPU-shader model. This lineage is documented in mpv's README and the `CHANGES` file in the repository root. [Source](https://github.com/mpv-player/mpv)

**Design principles that distinguish mpv from legacy players:**

- *Single event-driven core.* The player core runs an event loop. All external interaction — property changes, command dispatch, render callbacks — passes through that loop via thread-safe message queues. No global mutable state is accessible without locking.
- *Minimal built-in GUI.* mpv deliberately does not embed a widget toolkit. The default interface is the OSC (On-Screen Controller), implemented as a Lua script. Embedding applications supply their own UI via libmpv.
- *Clean API surface.* The public API (headers in `include/mpv/`) is ABI-stable and versioned. Internal headers are not public. As of the current master branch, `MPV_CLIENT_API_VERSION = MPV_MAKE_VERSION(2, 5)`. [Source](https://github.com/mpv-player/mpv/blob/master/include/mpv/client.h)
- *Hardware-first decode path.* Non-copy hardware decoding (VAAPI, V4L2 stateless, VDPAU, nvdec, DXVA2) is attempted before software fallback. The hardware surface is exported as a DMA-buf and imported into the GPU rendering backend without a CPU round-trip.
- *libplacebo for all GPU rendering.* Since mpv 0.35.0 the primary VO (`gpu-next`) delegates all shader work to libplacebo, which is independently maintained and adopted by VLC and FFmpeg.

**LGPL vs GPL build flags.** mpv's core is dual-licensed. With `--Dgpl=false` the build produces an LGPL-only binary (fewer features — no postprocessing, no ytdl hook by default). Most distribution packages build with GPL enabled. The libmpv public headers (`include/mpv/`) are ISC-licensed regardless of build flags. [Source](https://github.com/mpv-player/mpv/blob/master/Copyright)

---

## 2. libmpv API

### 2.1 Handle Lifecycle

The libmpv handle is the entry point for every embedding scenario. The lifecycle is strict: create, configure, initialise, operate, destroy.

```c
// include/mpv/client.h (ISC licence)
// https://github.com/mpv-player/mpv/blob/master/include/mpv/client.h

mpv_handle *mpv_create(void);
int         mpv_initialize(mpv_handle *ctx);
void        mpv_destroy(mpv_handle *ctx);         // detach; core continues if other handles exist
void        mpv_terminate_destroy(mpv_handle *ctx); // blocks until full shutdown
```

Options **must** be set between `mpv_create()` and `mpv_initialize()`. Attempting to set initialisation-time options after `mpv_initialize()` returns `MPV_ERROR_OPTION_NOT_FOUND` or is silently ignored depending on the option. [Source](https://github.com/mpv-player/mpv/blob/master/include/mpv/client.h)

```c
mpv_handle *ctx = mpv_create();

// Select the libmpv render-API video output (no window creation by mpv)
mpv_set_option_string(ctx, "vo", "libmpv");
mpv_set_option_string(ctx, "hwdec", "auto");

mpv_initialize(ctx);

// Load a file
const char *cmd[] = {"loadfile", "/path/to/video.mkv", NULL};
mpv_command(ctx, cmd);
```

### 2.2 Property and Command API

```c
// Typed property access
int mpv_set_option(mpv_handle *ctx, const char *name,
                   mpv_format format, void *data);
int mpv_set_option_string(mpv_handle *ctx, const char *name,
                          const char *data);
int mpv_get_property(mpv_handle *ctx, const char *name,
                     mpv_format format, void *data);
int mpv_set_property(mpv_handle *ctx, const char *name,
                     mpv_format format, void *data);

// Command dispatch
int mpv_command(mpv_handle *ctx, const char **args);   // NULL-terminated array
int mpv_command_string(mpv_handle *ctx, const char *args); // text form
```

`mpv_format` values include `MPV_FORMAT_STRING`, `MPV_FORMAT_FLAG` (bool as int), `MPV_FORMAT_INT64`, `MPV_FORMAT_DOUBLE`, and `MPV_FORMAT_NODE` (a discriminated-union tree). Properties set at runtime (after `mpv_initialize`) take effect immediately for dynamic options; some options require a soft-reset (`mp.command("playlist-play-index", "current")`) to take effect. [Source](https://github.com/mpv-player/mpv/blob/master/include/mpv/client.h)

### 2.3 Wakeup Callback Pattern

`mpv_wait_event()` blocks until an event is queued or the timeout expires. Embedding into a GUI event loop requires the wakeup callback pattern:

```c
// Called from mpv's internal thread — must NOT call mpv_wait_event()
static void wakeup_cb(void *userdata) {
    MyApp *app = userdata;
    app->post_event(WAKEUP_MPV);   // post to GUI event queue; platform-specific
}

// In initialization:
mpv_set_wakeup_callback(ctx, wakeup_cb, app);

// In the GUI event handler for WAKEUP_MPV:
while (true) {
    mpv_event *ev = mpv_wait_event(ctx, 0);  // zero timeout = non-blocking
    if (ev->event_id == MPV_EVENT_NONE) break;
    handle_mpv_event(ev);
}
```

Not draining the event queue via `mpv_wait_event()` does not stall playback, but the internal ring buffer fills and eventually returns `MPV_ERROR_EVENT_QUEUE_FULL`, silently dropping subsequent events. [Source](https://github.com/mpv-player/mpv/blob/master/include/mpv/client.h)

### 2.4 Render API

The render API (`include/mpv/render.h`) decouples mpv's video output from any particular windowing system. The embedding application provides GPU context via `mpv_render_param` key–value pairs; mpv renders into whatever surface the application has bound. [Source](https://github.com/mpv-player/mpv/blob/master/include/mpv/render.h)

```c
// include/mpv/render.h
typedef struct mpv_render_context mpv_render_context;

int  mpv_render_context_create(mpv_render_context **res,
                               mpv_handle *mpv,
                               mpv_render_param *params);
void mpv_render_context_set_update_callback(mpv_render_context *ctx,
                                            mpv_render_update_fn callback,
                                            void *callback_ctx);
int  mpv_render_context_render(mpv_render_context *ctx,
                               mpv_render_param *params);
void mpv_render_context_free(mpv_render_context *ctx);
```

Only one `mpv_render_context` per `mpv_handle` is permitted. Key `MPV_RENDER_PARAM_*` types:

| Parameter | Value type | Purpose |
|---|---|---|
| `MPV_RENDER_PARAM_API_TYPE` | `char *` | `"opengl"` or `"sw"` |
| `MPV_RENDER_PARAM_OPENGL_INIT_PARAMS` | `mpv_opengl_init_params *` | `get_proc_address` callback |
| `MPV_RENDER_PARAM_ICC_PROFILE` | `mpv_byte_array *` | Raw ICC profile blob |
| `MPV_RENDER_PARAM_WL_DISPLAY` | `struct wl_display *` | Wayland EGL surface binding |
| `MPV_RENDER_PARAM_ADVANCED_CONTROL` | `int *` (0/1) | Enable direct rendering + GPU screenshots |

`MPV_RENDER_PARAM_ADVANCED_CONTROL` enables `vd-lavc-dr` (direct rendering into GPU-mapped buffers) and GPU-side screenshot capture. When enabled, the application's render thread must not call non-render-safe libmpv functions to avoid deadlock. Prior to API version 1.105 (mpv 0.30.0), all `mpv_render_*` calls had to originate from the same thread that called `mpv_render_context_create()`; newer versions allow multi-threaded render calls. [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/interface-changes.rst)

**Complete embedding sequence:**

```c
// 1. Create and configure
mpv_handle *mpv = mpv_create();
mpv_set_option_string(mpv, "vo", "libmpv");
mpv_initialize(mpv);

// 2. Create render context
mpv_opengl_init_params gl_params = { .get_proc_address = my_get_proc };
int adv = 1;
mpv_render_param params[] = {
    { MPV_RENDER_PARAM_API_TYPE,         MPV_RENDER_API_TYPE_OPENGL },
    { MPV_RENDER_PARAM_OPENGL_INIT_PARAMS, &gl_params },
    { MPV_RENDER_PARAM_ADVANCED_CONTROL,  &adv },
    { 0 }
};
mpv_render_context *rctx;
mpv_render_context_create(&rctx, mpv, params);

// 3. Register update callback (called when a new frame is ready)
mpv_render_context_set_update_callback(rctx, render_update_cb, NULL);

// 4. On callback: render into current GL framebuffer
int fbo = 0, flip_y = 1;
mpv_opengl_fbo fbo_params = { .fbo = fbo, .w = width, .h = height };
mpv_render_param render_params[] = {
    { MPV_RENDER_PARAM_OPENGL_FBO, &fbo_params },
    { MPV_RENDER_PARAM_FLIP_Y,     &flip_y     },
    { 0 }
};
mpv_render_context_render(rctx, render_params);

// 5. After buffer swap:
mpv_render_context_report_swap(rctx);

// 6. Shutdown
mpv_render_context_free(rctx);
mpv_terminate_destroy(mpv);
```

`mpv_render_context_report_swap()` tells mpv when the buffer swap actually occurred; this timestamp feeds the A/V sync algorithm's display-time estimate. [Source](https://github.com/mpv-player/mpv/blob/master/include/mpv/render.h)

---

## 3. Demuxer Layer

### 3.1 Demuxer Descriptor Interface

Each demuxer in mpv is described by a `demuxer_desc_t` (defined in `demux/demux.h`), a struct of function pointers that the generic demux wrapper invokes from a dedicated background thread:

```c
// demux/demux.h (simplified)
// https://github.com/mpv-player/mpv/blob/master/demux/demux.h
struct demuxer_desc {
    const char *name;           // e.g. "mkv", "lavf"
    const char *desc;           // user-visible description
    int  (*open)(struct demuxer *, enum demux_check check);
    int  (*read_packet)(struct demuxer *, struct demux_packet **pkt);
    void (*drop_buffers)(struct demuxer *);
    void (*close)(struct demuxer *);
    int  (*seek)(struct demuxer *, double rel_seek_secs, int flags);
    bool (*load_timeline)(struct timeline *);  // ordered chapters
    void (*switched_tracks)(struct demuxer *);
};
```

mpv probes each registered demuxer in order; `enum demux_check` controls strictness (`DEMUX_CHECK_FORCE`, `DEMUX_CHECK_REQUEST`, `DEMUX_CHECK_NORMAL`, `DEMUX_CHECK_UNSAFE`). [Source](https://github.com/mpv-player/mpv/blob/master/demux/demux.h)

The primary native demuxer is `demux_mkv` (Matroska and WebM). For all other container formats, `demux_lavf` delegates to libavformat (FFmpeg). The native Matroska demuxer is preferred for `.mkv` because it provides richer support for ordered chapters, attachments, cues-based seeking, and codec-private data extraction without FFmpeg's abstraction overhead.

### 3.2 Packet Cache Ring Buffer

The generic demux wrapper (`demux/demux.c`) runs the demuxer in a thread and feeds a ring-buffer packet cache. Key cache parameters (controlled by `--cache`, `--cache-secs`, `--demuxer-max-bytes`):

```
enable_cache    bool    -- use ring-buffer (required for backward seeking)
max_bytes       size    -- maximum cached packet bytes (forward)
max_bytes_bw    size    -- backward cache bytes
min_secs        double  -- minimum seconds to pre-buffer
hyst_secs       double  -- hysteresis to avoid thrashing
```

Seek flags exposed to the demuxer:

| Flag | Meaning |
|---|---|
| `SEEK_FACTOR` | Position 0..1 (byte-offset seek) |
| `SEEK_FORWARD` | Forward-biased key-frame seek |
| `SEEK_CACHED` | Packet-cache-only; fails if timestamp not in cache |
| `SEEK_SATAN` | Backward demuxing (re-demux from earlier key-frame) |
| `SEEK_HR` | High-resolution seek (may return non-keyframe) |
| `SEEK_BLOCK` | Block until seek completes |

The `demux_reader_state` structure exposes cache fill state to the player core: `fw_bytes` (bytes cached ahead), `num_seek_ranges` (up to 10 cached seek ranges with timestamps), `underrun` flag, and `total_bytes`. This drives the UI cache indicator and pre-fetch decisions. [Source](https://github.com/mpv-player/mpv/blob/master/demux/demux.h)

### 3.3 Ordered Chapter Timeline

Matroska ordered chapters allow a single `.mkv` to reference segments in other files (common in anime distribution). `demux_mkv` exposes cross-segment links via `matroska_chapter[]` entries, each carrying a `matroska_segment_uid`. The `load_timeline()` callback instantiates multiple demuxer instances and weaves them into a single continuous timeline via the timeline demuxer in `demux/timeline.c`. From the player core's perspective, this is a single seekable stream. [Source](https://github.com/mpv-player/mpv/blob/master/demux/demux.h)

---

## 4. Decoder Pipeline and Zero-Copy DMA-buf

### 4.1 Hardware Decode Paths

mpv selects hardware decoders via `--hwdec`. Available values and their VO requirements as of mpv 0.41.0 [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/man/options.rst):

| hwdec value | Required VO | Notes |
|---|---|---|
| `vaapi` | `gpu`, `gpu-next`, `vaapi`, `dmabuf-wayland` | VAAPI zero-copy path |
| `drm` | `gpu`, `gpu-next` | DRM PRIME from V4L2 stateless |
| `vulkan` | `gpu-next` | Vulkan Video Decoding |
| `vdpau` | `gpu` with `--gpu-context=x11` | NVIDIA legacy |
| `nvdec` | `gpu`, `gpu-next` | NVIDIA CUDA decoder |
| `vaapi-copy` | any | Copies back to system RAM |
| `drm-copy` | any | Copies back to system RAM |

In mpv 0.41.0 the Vulkan hwdec path is preferred over other APIs when available, and non-copy hwdec is tried before `-copy` fallbacks. [Source](https://github.com/mpv-player/mpv/releases/tag/v0.41.0)

### 4.2 AVHWFramesContext and the Decode Surface

Hardware decoders produce `AVFrame` objects with `format == AV_PIX_FMT_VAAPI` (or `AV_PIX_FMT_DRM_PRIME` for V4L2 stateless). The surface handle is embedded in `AVFrame.data[3]` as an opaque pointer. The `AVHWFramesContext` attached to the frame carries the format and pool configuration; mpv probes this with 128×128 test allocations to select the optimal format before committing to a decode session. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/hwdec/hwdec_vaapi.c)

### 4.3 VAAPI Zero-Copy DMA-buf Export

The zero-copy path from VAAPI decode to GPU rendering is implemented in `video/out/hwdec/hwdec_vaapi.c`. The key call is `vaExportSurfaceHandle()`:

```c
// video/out/hwdec/hwdec_vaapi.c (simplified)
// https://github.com/mpv-player/mpv/blob/master/video/out/hwdec/hwdec_vaapi.c

VADRMPRIMESurfaceDescriptor desc = {0};
VAStatus status = vaExportSurfaceHandle(
    va_display,
    va_surface_id,
    VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2,
    VA_EXPORT_SURFACE_READ_ONLY | VA_EXPORT_SURFACE_COMPOSED_LAYERS,
    &desc);

// desc.layers[i].drm_format_modifier carries the DRM format modifier
// desc.objects[j].fd is the DMA-buf file descriptor
// desc.objects[j].size is the allocation size
```

After export, `vaSyncSurface()` ensures decode completion. The DMA-buf FDs in `desc.objects[]` are then imported into the GPU backend:

- **OpenGL path**: `eglCreateImageKHR()` with `EGL_LINUX_DMA_BUF_EXT`, then `glEGLImageTargetTexture2DOES()`. This is the `dmabuf_interop_gl_init` path.
- **Vulkan path**: `VkImportMemoryFdInfoKHR` via `VK_EXT_external_memory_dma_buf`, producing a `VkImage` with `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT`. This is the `dmabuf_interop_pl_init` path (libplacebo's Vulkan importer).
- **Wayland path**: for `vo_dmabuf_wayland`, the FDs are wrapped in a `wl_buffer` via `zwp_linux_dmabuf_v1`.

The `ra_hwdec_mapper_driver` interface abstracts these paths. The mapper's `map()` function receives the `mp_image` and fills `mapper->tex[4]` (up to four `ra_tex` objects for multi-plane formats). The `unmap()` function releases the GPU-side references and closes the DMA-buf FDs. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/gpu/hwdec.h)

### 4.4 V4L2 Stateless DRM PRIME Path

V4L2 stateless decoders (used for embedded SoC decoders and some Intel/AMD iGPU paths) produce `AVDRMFrameDescriptor` directly in `AVFrame.data[0]`. The `AVDRMFrameDescriptor` carries the DMA-buf FD and DRM format modifier without going through VAAPI. The mpv `hwdec_drmprime.c` mapper translates this directly to `ra_tex` objects, following the same OpenGL EGL or Vulkan import route as the VAAPI path. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/gpu/hwdec.h)

---

## 5. Video Output Architecture

### 5.1 VO Driver Interface

Video output drivers implement the `vo_driver` struct, defined in `video/out/vo.h`:

```c
// video/out/vo.h (key fields, simplified)
// https://github.com/mpv-player/mpv/blob/master/video/out/vo.h
struct vo_driver {
    const char *name;
    const char *description;
    int caps;                // VO_CAP_* bitfield
    int priv_size;           // size of driver's private struct
    int (*preinit)(struct vo *vo);
    int (*query_format)(struct vo *vo, int format);
    int (*reconfig)(struct vo *vo, struct mp_image_params *params);
    int (*reconfig2)(struct vo *vo, struct mp_image *img);
    int (*control)(struct vo *vo, uint32_t request, void *data);
    struct mp_image *(*get_image_ts)(struct vo *vo,   // optional, direct rendering
                                     int imgfmt, int w, int h, int stride_align,
                                     int flags);
    void (*draw_frame)(struct vo *vo, struct vo_frame *frame);
    void (*flip_page)(struct vo *vo);
    void (*get_vsync)(struct vo *vo, struct vo_vsync_info *info);
    void (*wait_events)(struct vo *vo, int64_t until_time_us);
    void (*wakeup)(struct vo *vo);
    void (*uninit)(struct vo *vo);
};
```

The `VO_CAP_*` bitfield declares driver capabilities:

| Capability | Meaning |
|---|---|
| `VO_CAP_ROTATE90` | Supports 90° rotation steps |
| `VO_CAP_FILM_GRAIN` | AV1/H.274 film grain synthesis |
| `VO_CAP_VFLIP` | Vertical flip without shader overhead |
| `VO_CAP_FRAMEDROP` | Frame-drop-aware (skips late frames internally) |
| `VO_CAP_NORETAIN` | Does not retain frames between `draw_frame` calls |
| `VO_CAP_FRAMEOWNER` | Takes ownership of the `mp_image` buffer |

The `VOCTRL` enum provides the control interface between the player core and VO drivers via `control()`. Key entries include `VOCTRL_PERFORMANCE_DATA` (returns `struct voctrl_performance_data` with per-pass GPU timing), `VOCTRL_GET_ICC_PROFILE` (requests ICC blob from display), and `VOCTRL_SCREENSHOT`. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/vo.h)

### 5.2 vo_gpu_next (Default since mpv 0.41.0)

`vo_gpu_next` delegates all rendering to libplacebo. It became the default VO in mpv 0.41.0 (December 2024), superseding `vo_gpu`. [Source](https://github.com/mpv-player/mpv/releases/tag/v0.41.0)

```c
// video/out/vo_gpu_next.c
// https://github.com/mpv-player/mpv/blob/master/video/out/vo_gpu_next.c
const struct vo_driver video_out_gpu_next = {
    .description = "Video output based on libplacebo",
    .name        = "gpu-next",
    .caps        = VO_CAP_ROTATE90 | VO_CAP_FILM_GRAIN | VO_CAP_VFLIP,
    // ...
};
```

The driver's private state holds the libplacebo `pl_renderer`, the `pl_queue` for frame sequencing, `ra_hwdec_ctx` and `ra_hwdec_mapper` for hardware surface import, `user_hook[]` arrays loaded from `--glsl-shaders`, and `user_lut[]` from `--lut`. Performance data is collected per frame as `struct frame_info { int count; struct pl_dispatch_info info[VO_PASS_PERF_MAX]; }`. The integration headers pulled in by `vo_gpu_next.c` are `libplacebo/utils/libav.h` (`pl_frame_from_avframe()`), `libplacebo/utils/frame_queue.h` (the `pl_queue` API), and `libplacebo/shaders/icc.h` (ICC profile application).

**Requirements**: libplacebo >= 6.338.2 and FFmpeg >= 6.1. [Source](https://github.com/mpv-player/mpv/wiki/GPU-Next-vs-GPU)

### 5.3 vo_dmabuf_wayland

`vo_dmabuf_wayland` bypasses GPU shaders entirely: it exports the decoded DMA-buf directly as a Wayland buffer and lets the compositor's hardware overlay or fixed-function scaler handle the frame.

```c
// video/out/vo_dmabuf_wayland.c
// https://github.com/mpv-player/mpv/blob/master/video/out/vo_dmabuf_wayland.c
const struct vo_driver video_out_dmabuf_wayland = {
    .description = "Wayland dmabuf video output",
    .name        = "dmabuf-wayland",
    .caps        = VO_CAP_ROTATE90 | VO_CAP_FRAMEOWNER | VO_CAP_VFLIP,
};
```

It uses the `linux-dmabuf-v1`, `viewporter`, and `single-pixel-buffer-v1` Wayland protocols. A pool of `WL_BUFFERS_WANTED = 15` `wl_buffer` objects is maintained to avoid flickering while buffers are in use by the compositor. Colour management for non-BT.601 colour spaces requires the `color-management-v1` Wayland protocol. The trade-off is quality: this VO performs no debanding, upscaling, or tone mapping — those are either absent or left entirely to the compositor's fixed-function hardware path. It is intended for VAAPI or DRM PRIME hardware decode scenarios where power consumption outweighs image quality requirements. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/vo_dmabuf_wayland.c)

### 5.4 vo_drm (Direct KMS)

`vo_drm` targets headless or embedded scenarios without a compositor:

```c
// video/out/vo_drm.c
// https://github.com/mpv-player/mpv/blob/master/video/out/vo_drm.c
const struct vo_driver video_out_drm = {
    .name        = "drm",
    .description = "Direct Rendering Manager (software scaling)",
};
```

It uses dumb framebuffers (`drmModeCreateDumbBuffer`) and software scaling; no hardware decode path is supported (use `-copy` hwdec variants to copy frames to system RAM). Options: `--drm-connector`, `--drm-device`, `--drm-mode` (`preferred|highest|NxH[@R]`), `--drm-draw-plane` (`primary|overlay|N`), `--drm-format` (`xrgb8888|xbgr8888|xrgb2101010|xbgr2101010|yuyv`). Requires `--profile=sw-fast` for acceptable software-decode performance. [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/man/vo.rst)

---

## 6. libplacebo: Shader-Based Video Processing

libplacebo originated as a standalone rewrite of mpv's core rendering algorithms and is described in its README as such. [Source](https://github.com/haasn/libplacebo/blob/master/README.md) It is now adopted by VLC (Vulkan output) and as an FFmpeg video filter (`libplacebo`), making it the most widely used GPU video processing library on Linux.

### 6.1 Backend Architecture

libplacebo provides four API tiers:

- **Tier 0**: math, colorspace, tone/gamut mapping, filter kernels, dithering, cache — no GPU dependency
- **Tier 1**: `pl_gpu` (unified GPU abstraction), `pl_swapchain`, `pl_vulkan`, `pl_opengl`, `pl_d3d11`
- **Tier 2**: shader modules — `shaders/colorspace.h`, `shaders/deinterlacing.h`, `shaders/dithering.h`, `shaders/film_grain.h`, `shaders/icc.h`, `shaders/lut.h`, `shaders/sampling.h`, `shaders/custom.h`
- **Tier 3**: `pl_renderer`, `pl_dispatch` — the highest-level API; this is what most players use directly

Backend support: Vulkan (core 1.2+), OpenGL (GLSL >= 130, GL >= 3.0 or GLES >= 3.0), Direct3D 11 (feature level >= 9_1). [Source](https://github.com/haasn/libplacebo/blob/master/README.md)

### 6.2 pl_renderer API

```c
// libplacebo/renderer.h
// https://github.com/haasn/libplacebo/blob/master/src/include/libplacebo/renderer.h

pl_renderer pl_renderer_create(pl_log log, pl_gpu gpu);
void        pl_renderer_destroy(pl_renderer *rr);

// Single-frame rendering
bool pl_render_image(pl_renderer rr,
                     const pl_frame *image,
                     const pl_frame *target,
                     const pl_render_params *params);

// Multi-frame temporal blending
bool pl_render_image_mix(pl_renderer rr,
                         const pl_frame_mix *mix,
                         const pl_frame *target,
                         const pl_render_params *params);

// Shader cache (save/restore compiled SPIR-V or GLSL)
size_t pl_renderer_save(pl_renderer, uint8_t *out);
void   pl_renderer_load(pl_renderer, const uint8_t *cache);
```

`pl_renderer` is conceptually stateless from the caller's perspective — all configuration is passed per-call via `pl_render_params`. State retained internally includes the compiled shader cache and peak detection histograms. [Source](https://libplacebo.org/renderer/)

### 6.3 Frame Queue and Temporal Interpolation

`pl_queue` abstracts frame sequencing for the player:

```c
pl_queue pl_queue_create(pl_gpu gpu);

// Push a decoded frame (called from decoder thread)
void pl_queue_push_block(pl_queue, uint64_t timeout,
                         const pl_source_frame *frame);

// Update mix for current vsync time (called from VO)
enum pl_queue_status pl_queue_update(pl_queue,
                                     pl_frame_mix *mix,
                                     pl_queue_params params);
```

`pl_queue_update()` synthesises a `pl_frame_mix` — a weighted set of input frames at the current vsync timestamp — automatically, eliminating the need for the player to perform PTS normalisation. `pl_render_image_mix()` then performs temporal blending (e.g. Mitchell-Netravali weighted blend) across the mix frames for motion-compensated frame interpolation. [Source](https://libplacebo.org/renderer/)

### 6.4 Key pl_render_params Fields

```c
struct pl_render_params {
    const struct pl_filter_config *upscaler;      // e.g. &pl_filter_ewa_lanczos
    const struct pl_filter_config *downscaler;
    const struct pl_filter_config *frame_mixer;   // temporal blend kernel
    float antiringing_strength;
    const struct pl_deband_params   *deband_params;
    const struct pl_sigmoid_params  *sigmoid_params;
    const struct pl_color_adjustment *color_adjustment;
    const struct pl_peak_detect_params *peak_detect_params;
    const struct pl_color_map_params   *color_map_params;
    const struct pl_dither_params      *dither_params;
    const struct pl_icc_color_space    *icc_dst;  // output ICC profile
    const struct pl_hook * const *hooks;
    int num_hooks;
    // ... LUT, deinterlace, film grain, blend, error diffusion ...
};
```

Built-in upscalers include EWA Lanczos (`pl_filter_ewa_lanczos`, the `scale=ewa_lanczos` mpv option), Mitchell-Netravali, bicubic, bilinear, and the neural RAVU (Rapid and Accurate Video Upscaling) variants. [Source](https://github.com/haasn/libplacebo/blob/master/src/include/libplacebo/renderer.h)

---

## 7. Shader System and Custom User Shaders

### 7.1 GLSL Generation in libplacebo

libplacebo does not compile and link GLSL directly. Instead, it builds shader fragments via `pl_shader_obj` and `pl_dispatch`. Each rendering step appends GLSL source into a `pl_shader` object; `pl_dispatch_finish()` concatenates the fragments, compiles the combined shader, and dispatches the GPU work. Shader programs are cached by a hash of the combined source; the saved cache (from `pl_renderer_save()`) avoids recompilation on restart.

### 7.2 pl_hook_stage: Hook Points in the Pipeline

Custom shaders (libplacebo's extension point, exposed via mpv `--glsl-shaders`) are parsed into `pl_hook` objects and inserted at specific stages in the rendering pipeline. The available hook stages, defined in `shaders/custom.h`, are: [Source](https://github.com/haasn/libplacebo/blob/master/src/include/libplacebo/shaders/custom.h)

| Stage | Phase |
|---|---|
| `PL_HOOK_RGB_INPUT` | Before RGB plane processing |
| `PL_HOOK_LUMA_INPUT` | Before luma plane processing |
| `PL_HOOK_CHROMA_INPUT` | Before chroma plane processing |
| `PL_HOOK_ALPHA_INPUT` | Before alpha plane processing |
| `PL_HOOK_XYZ_INPUT` | Before XYZ plane processing |
| `PL_HOOK_CHROMA_SCALED` | After chroma upsampling |
| `PL_HOOK_NATIVE` | After plane combination, native colorspace |
| `PL_HOOK_LINEAR` | After linearisation |
| `PL_HOOK_SIGMOID` | After sigmoid transform |
| `PL_HOOK_PRE_KERNEL` | Before main upscaling kernel |
| `PL_HOOK_POST_KERNEL` | After main upscaling kernel |
| `PL_HOOK_SCALED` | After upscaling, output resolution |
| `PL_HOOK_PRE_OUTPUT` | Before dithering/blending |
| `PL_HOOK_OUTPUT` | Final output stage |

Input and native stages are "resizable" — the hook can change the output resolution. Fixed-size stages (`PL_HOOK_SCALED` through `PL_HOOK_OUTPUT`) run at output resolution.

### 7.3 User Shader File Format (.hook)

mpv exposes libplacebo hooks to users as `.hook` files loaded via `--glsl-shaders`. The file format uses `//!` directive comments: [Source](https://libplacebo.org/custom-shaders/)

```glsl
// example: simple sharpening hook at LUMA stage
//!HOOK LUMA
//!BIND HOOKED
//!DESC Simple sharpening

vec4 hook() {
    vec2 pt = HOOKED_pt;
    vec4 c = HOOKED_tex(HOOKED_pos);
    vec4 t = HOOKED_tex(HOOKED_pos + vec2(0.0,  pt.y));
    vec4 b = HOOKED_tex(HOOKED_pos + vec2(0.0, -pt.y));
    vec4 l = HOOKED_tex(HOOKED_pos + vec2(-pt.x, 0.0));
    vec4 r = HOOKED_tex(HOOKED_pos + vec2( pt.x, 0.0));
    return c + 0.25 * (4.0 * c - t - b - l - r);
}
```

Directive syntax:

| Directive | Meaning |
|---|---|
| `//!HOOK <stage>` | Target hook stage (e.g. `LUMA`, `NATIVE`, `OUTPUT`) |
| `//!BIND HOOKED` | Bind the stage's implicit texture (or a named texture) |
| `//!SAVE <name>` | Redirect hook output to a named texture |
| `//!WHEN <RPN>` | Conditional execution (RPN boolean expression) |
| `//!COMPUTE w h` | Use a compute shader instead of fragment |
| `//!PARAM` | Expose a tunable parameter |

The `HOOKED_tex(vec2)`, `HOOKED_texOff(vec2)`, `HOOKED_size`, and `HOOKED_pt` names are generated automatically from the `//!BIND HOOKED` directive.

In `vo_gpu_next`, user shaders are loaded as `struct user_hook { char *path; const struct pl_hook *hook; }` and passed in `pl_render_params.hooks`. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/vo_gpu_next.c)

---

## 8. HDR and Colour Management

### 8.1 Colour Space Representation in libplacebo

libplacebo's colour space system covers the full range of primaries and transfer functions encountered in modern video content. Key constants from `colorspace.h` [Source](https://github.com/haasn/libplacebo/blob/master/src/include/libplacebo/colorspace.h):

**Primaries**: `PL_COLOR_PRIM_BT_709` (HD/sRGB), `PL_COLOR_PRIM_BT_2020` (UltraHD), `PL_COLOR_PRIM_DCI_P3`, `PL_COLOR_PRIM_DISPLAY_P3`, `PL_COLOR_PRIM_ACES_AP0/AP1`, `PL_COLOR_PRIM_BT_601_525/625`.

**Transfer functions**: `PL_COLOR_TRC_SRGB`, `PL_COLOR_TRC_LINEAR`, `PL_COLOR_TRC_PQ` (SMPTE ST2084 / HDR10), `PL_COLOR_TRC_HLG` (ARIB STD-B67), `PL_COLOR_TRC_V_LOG`, `PL_COLOR_TRC_S_LOG1`, `PL_COLOR_TRC_S_LOG2`.

`pl_frame_from_avframe()` converts FFmpeg's `AVColorSpace`, `AVColorTransferCharacteristic`, and `AVColorPrimaries` enums to the corresponding libplacebo constants, bridging the FFmpeg and libplacebo colour models. [Source](https://github.com/haasn/libplacebo/blob/master/README.md)

### 8.2 HDR Metadata Types

```c
// libplacebo/colorspace.h
enum pl_hdr_metadata_type {
    PL_HDR_METADATA_HDR10,      // SMPTE ST2086 static mastering display
    PL_HDR_METADATA_HDR10PLUS,  // SMPTE ST2094-40 Annex B, dynamic Bezier EETF
    PL_HDR_METADATA_CIE_Y,      // CIE Y luminance scalar
    PL_HDR_METADATA_DOLBY_VISION, // Dolby Vision RPU (profile 5 / 8.1)
};

struct pl_hdr_metadata {
    struct pl_raw_primaries prim;  // mastering display primaries + white point
    float min_luma;                // cd/m², floor ~1e-6 (one PQ step above zero)
    float max_luma;                // cd/m²
    float max_cll;                 // content light level
    float max_fall;                // frame-average light level
    // HDR10+: Bezier EETF coefficients (dynamic per-scene)
    // Dolby Vision: reshaped BL + EL metadata
};
```

### 8.3 HDR10+ and Dolby Vision

HDR10+ (SMPTE ST2094-40) provides per-scene dynamic metadata via Bezier-curve EETF (Electro-optical Transfer Function). libplacebo applies the Bezier EETF from Annex B, adjusting the OOTF based on the ratio of the targeted display peak luminance to the mastering display peak luminance. Without metadata, it falls back to a constant Bezier curve. Dolby Vision Profile 5 is handled by reading the RPU side data and applying the BL-only reshaping, then converting to HDR10/PQ or SDR as required by the output. [Source](https://github.com/haasn/libplacebo/blob/master/src/include/libplacebo/colorspace.h)

### 8.4 Real-Time Peak Detection

For content without HDR10 static metadata or when HDR10+ is not available, libplacebo implements real-time peak detection via a per-frame histogram computed by a GPU compute shader. The peak luminance estimate converges over several frames using a configurable smoothing window. This is controlled by `pl_peak_detect_params` in `pl_render_params`.

### 8.5 ICC Profile Handling

ICC profiles can be fed into vo_gpu_next via two paths:
1. `MPV_RENDER_PARAM_ICC_PROFILE` at render context creation time (static profile for the display).
2. `VOCTRL_GET_ICC_PROFILE` — the VO queries the display's ICC profile at startup and on display change events.

The `--icc-use-luma` option (added mpv 0.38.0, `--vo=gpu-next` only) reads the ICC profile's luminance value to calibrate the display's white point. libplacebo supports colorimetrically accurate ICC application with soft gamut mapping, ITU-R BT.1886 EOTF emulation, black-point compensation, and custom 3DLUTs (`.cube` files). [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/interface-changes.rst)

### 8.6 LUT Types

```c
enum pl_lut_type {
    PL_LUT_UNKNOWN,      // infer from metadata
    PL_LUT_NATIVE,       // applied after bit-depth fix, in gamma light
    PL_LUT_NORMALIZED,   // applied in linear light, 1.0 = nominal peak
    PL_LUT_CONVERSION,   // fully replaces colour conversion + tone mapping
};
```

`PL_LUT_CONVERSION` is the most powerful: when applied to the output target, it replaces the entire colour conversion pipeline. When `PL_LUT_CONVERSION` is set, `color_map_params` is ignored. [Source](https://github.com/haasn/libplacebo/blob/master/src/include/libplacebo/renderer.h)

---

## 9. Audio Pipeline and A/V Sync

### 9.1 Audio Output Drivers

mpv's audio output layer mirrors the VO architecture: `ao_driver` structs with function-pointer callbacks for `init`, `uninit`, `reset`, `start`, `set_pause`, `control`, and optionally `hotplug_init`/`uninit` and `list_devs`.

**ao_pipewire** (`audio/out/ao_pipewire.c`) is the preferred AO on modern Linux desktops: [Source](https://github.com/mpv-player/mpv/blob/master/audio/out/ao_pipewire.c)

```c
// audio/out/ao_pipewire.c (struct priv, simplified)
struct priv {
    struct pw_thread_loop *loop;
    struct pw_stream      *stream;
    struct pw_core        *core;
    struct spa_hook        stream_listener;
    struct spa_hook        core_listener;
    enum { INIT_STATE_NONE, INIT_STATE_SUCCESS, INIT_STATE_ERROR } init_state;
    bool  muted;
    float volume;
    int   buffer_msec;
    char *remote;   // PipeWire remote name, NULL for default
};
```

Requirements: PipeWire >= 1.0.4 (for `pw_stream_get_nsec` nsec-precision timestamp). Known issue: heavy stutter at non-44100 Hz sample rates with PipeWire 1.4.3. Instability with `--video-sync=display-resample` is a known limitation; use `--video-sync=audio` with PipeWire.

**ao_alsa** uses libasound PCM directly, bypassing the PipeWire session layer. **ao_pulse** uses the PulseAudio async API. **ao_jack** provides low-latency JACK output for professional audio workflows.

### 9.2 Audio Filter Chain and Resampling

The audio filter chain sits between the decoder and the AO. Filters handle sample-rate conversion, channel remapping, dithered format conversion, and pitch correction for time-scaled playback. mpv delegates sample-rate conversion to libswresample, the same resampling library used by FFmpeg, which supports high-quality multi-stage polyphase resampling with configurable dither. For pitch-corrected tempo changes (`--speed` values other than 1.0 combined with `--audio-pitch-correction=yes`, the default), mpv applies scaletempo processing to preserve pitch while adjusting duration. [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/man/options.rst)

### 9.3 A/V Sync Algorithm

mpv uses the audio hardware clock as the primary reference. For each video frame, the player computes the display PTS relative to the current audio presentation time and video output latency:

```
display_pts = video_pts - audio_pts_now - video_latency
```

The VO receives a target display time in `vo_frame.pts`. `--video-sync` controls the synchronisation strategy:

| Mode | Behaviour |
|---|---|
| `audio` (default) | Soft-retime video frames to match audio clock |
| `display-resample` | Resample audio to match display refresh rate |
| `display-vdrop` | Drop/skip video frames to match display cadence |
| `display-vskip` | Duplicate/skip frames (no audio resampling) |
| `display-desync` | Ignore audio; video paced to display only |

Frame dropping (`--framedrop=vo`, the default) compares the current wall time against the video PTS at `flip_page()` time. Frames older than half a vsync interval are discarded. Drops stop below an effective 10 FPS rate to avoid completely losing the picture. [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/man/options.rst)

In `display-resample` mode, the audio resampler adjusts playback speed within `--video-sync-max-video-change` percent of nominal to absorb display clock drift without audible artefacts.

---

## 10. Wayland Integration

### 10.1 vo_gpu_next on Wayland

`vo_gpu_next` creates its Wayland window through the `gpu-context` layer. On Wayland, the context is `wayland-egl` (or `wayland-vk` for Vulkan). The context creates a `wl_surface`, allocates an `EGLSurface` via `eglCreateWindowSurface()` with a `wl_egl_window`, and hands the resulting EGL context to libplacebo's OpenGL backend or Vulkan backend.

`MPV_RENDER_PARAM_WL_DISPLAY` (a `struct wl_display *`) is passed at `mpv_render_context_create()` time when embedding in a Wayland application. This allows libmpv to use `EGL_KHR_platform_wayland` to create an EGL display connected to the embedding application's Wayland connection. [Source](https://github.com/mpv-player/mpv/blob/master/include/mpv/render.h)

### 10.2 vo_dmabuf_wayland DMA-buf Attach Path

For zero-copy presentation, `vo_dmabuf_wayland` exports the decoded hardware surface as a DMA-buf and creates a `wl_buffer` via `zwp_linux_dmabuf_v1`:

1. Decoder produces `mp_image` with `AV_PIX_FMT_VAAPI`.
2. `vaExportSurfaceHandle()` returns DMA-buf FDs.
3. `zwp_linux_dmabuf_v1_create_params()` → `zwp_linux_buffer_params_v1_add()` for each plane → `zwp_linux_buffer_params_v1_create_immed()` produces a `wl_buffer`.
4. `wl_surface_attach(surface, wl_buffer, 0, 0)` + `wl_surface_commit()` presents the frame.

The pool of 15 `wl_buffer` objects allows in-flight frames to be released asynchronously via the `wl_buffer::release` event. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/vo_dmabuf_wayland.c)

### 10.3 Frame Scheduling via wp_presentation-time

Both Wayland VOs use the `wp_presentation-time` protocol (`wp_presentation` interface) to obtain precise frame presentation timestamps. After each `wl_surface_commit()`, the client subscribes to `wp_presentation_feedback`. The compositor responds with a `presented` event carrying the hardware CLOCK_MONOTONIC presentation timestamp, which mpv feeds back into the A/V sync algorithm to calibrate display latency. This was introduced in Wayland via wayland-protocols and is required for accurate `display-resample` synchronisation. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/wayland_common.c)

### 10.4 mpv 0.41.0 Wayland Improvements

mpv 0.41.0 (December 2024) added Wayland clipboard writing, `wp-color-representation-v1` for accurate colour space signalling on `vo_dmabuf_wayland`, tablet input support, and `color-management-v2` protocol integration for display-referred HDR on Wayland. [Source](https://github.com/mpv-player/mpv/releases/tag/v0.41.0)

---

## 11. Performance Profiling

### 11.1 GPU Debug Mode

`--gpu-debug` enables GPU-level debugging:
- **OpenGL**: activates `GL_DEBUG_OUTPUT_SYNCHRONOUS` via `KHR_debug`, calls `glGetError()` after each draw call, and enables glDebugMessageCallback.
- **Vulkan**: activates the Vulkan validation layers (`VK_LAYER_KHRONOS_validation`) and `VK_EXT_debug_utils` message callbacks.

This adds substantial CPU overhead and is not suitable for performance measurement, but is invaluable for isolating GPU errors from VO or libplacebo bugs.

### 11.2 Stats Overlay

The built-in stats overlay (invoked by pressing `i` or `I`, or loaded via `--load-stats-overlay=yes`) displays per-frame GPU pass timings sourced from `VOCTRL_PERFORMANCE_DATA`.

```c
// video/out/vo.h
struct mp_pass_perf {
    uint64_t last;              // last measured ns
    uint64_t avg;               // rolling average ns
    uint64_t peak;              // peak ns
    uint64_t samples[256];      // ring buffer of samples
    uint64_t count;
};

struct mp_frame_perf {
    int             num_passes;
    struct mp_pass_perf pass[VO_PASS_PERF_MAX];  // up to 64 passes
    const char     *desc[VO_PASS_PERF_MAX];      // human-readable pass name
};
```

Each libplacebo shader dispatch maps to one pass entry. The frame timing graph shows: decode time, upload time, rendering passes (luma upscale, chroma upscale, tone mapping, dithering), and flip/present time. Dropped frame count and `display-fps` are also shown. [Source](https://github.com/mpv-player/mpv/blob/master/video/out/vo.h)

### 11.3 Log Level and Trace Output

```bash
# Maximum verbosity (extremely verbose — pipe to file)
mpv --msg-level=all=trace video.mkv 2>mpv-trace.log

# VO-level trace only (frame timing, VO decisions)
mpv --msg-level=vo=trace video.mkv

# Display performance data every second
mpv --display-fps-override=60 --msg-level=vo=debug video.mkv
```

The `--log-file=<path>` option redirects log output to a file in addition to stderr. libplacebo logs are relayed through mpv's log system at the corresponding verbosity level.

### 11.4 Dropped-Frame Detection Heuristics

mpv tracks drops at two points:
- **Decoder drops**: counted when the decoder skips frames to catch up (reported as `decoder-frame-drop-count` property).
- **VO drops**: counted when `draw_frame()` is called with `vo_frame.skipped = true` because the frame is already past its display deadline.

The `--framedrop` option controls the policy. Effective FPS can be read from the `estimated-display-fps` property. When the VO drop rate exceeds a threshold, mpv logs `A/V desync` warnings. The stats overlay aggregates these into the "Dropped" line.

---

## 12. Extension Points: Scripting and IPC

### 12.1 Lua and JavaScript Scripting

Scripts placed in `~/.config/mpv/scripts/` or loaded via `--script=<path>` run in isolated threads with full access to the player API. Each script has its own Lua 5.2 (or MuJS JavaScript) interpreter. [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/man/lua.rst)

Core Lua API functions:

```lua
-- Property access
mp.get_property("playback-time")           -- returns string
mp.get_property_number("playback-time")    -- returns number
mp.set_property("speed", "1.5")

-- Commands
mp.command("seek 10 relative")
mp.commandv("seek", "10", "relative")      -- vararg form, avoids quoting issues
mp.command_native({name="seek", value=10, whence="relative"}) -- table form

-- Events and observers
mp.register_event("file-loaded", function(event) ... end)
mp.observe_property("pause", "bool", function(name, val)
    print("Pause state: " .. tostring(val))
end)

-- Timers
mp.add_timeout(5.0, function() mp.command("quit") end)
mp.add_periodic_timer(1.0, function() ... end)

-- OSD
mp.osd_message("Hello from script", 3.0)  -- 3-second display
```

### 12.2 mp.add_hook: Pipeline Interception

```lua
-- on_load hook: modify the URI or inject options before the file opens
mp.add_hook("on_load", 50, function()
    local url = mp.get_property("stream-open-filename")
    if url:match("^myscheme://") then
        mp.set_property("stream-open-filename", transform_url(url))
    end
end)

-- on_preloaded hook: runs after tracks are scanned but before playback
mp.add_hook("on_preloaded", 50, function()
    -- Force audio track selection
    mp.set_property("aid", "1")
end)
```

The `priority` argument (50 is neutral) determines ordering among multiple hooks at the same stage. The player pauses at the hook point until all registered functions invoke continuation (implicitly, when the function returns). For deferred async work, call `hook:defer()` to suspend the hook, perform work, then call `hook:cont()` to resume. [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/man/lua.rst)

Built-in hook types: `on_load`, `on_load_fail`, `on_unload`, `on_preloaded`, `on_before_start_file`, `on_after_end_file`.

### 12.3 IPC Socket (JSON Protocol)

The IPC socket, enabled via `--input-ipc-server=<path>`, accepts JSON commands on a Unix domain socket:

```bash
# Start mpv with IPC server
mpv --input-ipc-server=/tmp/mpvsocket video.mkv &

# Send a command via socat
echo '{"command": ["get_property", "playback-time"], "request_id": 1}' \
  | socat - /tmp/mpvsocket

# Response:
# {"error":"success","data":12.483999,"request_id":1}

# Set volume without waiting for reply
echo '{"command": ["set_property", "volume", 80]}' \
  | socat - /tmp/mpvsocket

# Plain text commands are also accepted (no request_id, no reply):
echo 'seek 30 relative' | socat - /tmp/mpvsocket
```

Event stream (one JSON object per line on the socket, unsolicited):
```json
{"event":"file-loaded"}
{"event":"property-change","id":1,"name":"pause","data":false}
{"event":"end-file","reason":"eof"}
```

Subscribe to property changes with `{"command": ["observe_property", <id>, "<name>"]}`. `request_id` must be a 64-bit integer in range `-2^63..2^63-1`. [Source](https://github.com/mpv-player/mpv/blob/master/DOCS/man/ipc.rst)

### 12.4 JavaScript API

The JavaScript scripting API is identical in structure to the Lua API; `mp.` functions map one-to-one. JavaScript scripts use the MuJS interpreter bundled in mpv. Scripts ending in `.js` are loaded as JavaScript; `.lua` as Lua. There is no interoperability between scripts' interpreter states, but they share the same mpv property and event namespaces.

---

## Integrations

**Chapter 26 — Hardware Video (VA-API, V4L2, GStreamer)** covers the kernel-to-userspace VA-API decode pipeline that feeds mpv's `hwdec=vaapi` path. The `vaExportSurfaceHandle()` call described in §4.3 above consumes VA surfaces produced by Mesa's VAAPI drivers running on amdgpu/i915 kernel drivers. That chapter's description of `AVHWDeviceContext` and `AVHWFramesContext` is the FFmpeg layer that mpv's decoder thread manages.

**Chapter 38 — PipeWire and the Video Session Layer** is the substrate that `ao_pipewire` connects to. mpv submits audio as a PipeWire graph node; WirePlumber routes the node to the hardware sink. The PipeWire `pw_stream` lifecycle described in this chapter maps directly onto the `struct priv` fields of `ao_pipewire.c`.

**Chapter 20 — Wayland Protocol Fundamentals** covers the `linux-dmabuf-v1`, `wp_presentation-time`, `viewporter`, `zwp_linux_dmabuf_v1`, and `wp_color_management` protocols that `vo_dmabuf_wayland` and `vo_gpu_next`'s Wayland context use. The `wp_presentation_feedback.presented` event described in §10.3 is specified in that chapter's `wp_presentation` section.

**Chapter 24 — Vulkan and EGL for Application Developers** covers EGL surface creation, `eglCreateWindowSurface()` with `wl_egl_window`, and the Vulkan WSI path that `vo_gpu_next`'s `wayland-egl` and `wayland-vk` contexts build on. The `EGL_KHR_platform_wayland` extension and `MPV_RENDER_PARAM_WL_DISPLAY` embedding pattern described in §10.1 are rooted in the EGL architecture covered there.

**Chapter 217 — DLNA, UPnP, and the Linux Home Theater Stack** describes mpv's role as a renderer client in DLNA/UPnP media server setups, via integration with rygel-mpv glue or libmrpcdaemon. That chapter explains the control point protocol that dispatches `loadfile` commands to the mpv IPC socket described in §12.3.
