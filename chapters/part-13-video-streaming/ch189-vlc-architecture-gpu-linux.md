# Chapter 189: VLC Media Player — Architecture, GPU Acceleration, and the Linux Graphics Stack

> **Part**: Part XIII — Video Streaming on Linux
> **Audiences**: Multimedia application developers; systems developers understanding GPU-accelerated media playback; VLC contributors working on Linux rendering paths; power users customising VLC GPU/display behaviour
> **Status**: First draft — 2026-06-20

---

## Table of Contents

1. [VLC Architecture Overview](#vlc-architecture-overview)
2. [Demux and Input Layer](#demux-and-input-layer)
3. [Hardware Video Decode — VA-API Path](#hardware-video-decode--va-api-path)
4. [Embedded Hardware Decode — V4L2 and MMAL Paths](#v4l2-hardware-decode-path)
5. [Video Output — Wayland Zero-Copy Path](#video-output--wayland-zero-copy-path)
6. [Video Output — Vulkan Renderer (VLC 4.0 master)](#video-output--vulkan-renderer-vlc-40-master)
7. [OpenGL/EGL Renderer](#openglegl-renderer)
8. [Audio Output on Linux](#audio-output-on-linux)
9. [Transcoding and Streaming with --sout](#transcoding-and-streaming-with---sout)
10. [VLC 4.0 and Future Direction](#vlc-40-and-future-direction)
11. [Integrations](#integrations)
12. [References](#references)

---

## VLC Architecture Overview

VLC is a plugin-based multimedia player and framework developed by VideoLAN. Every major functional component is implemented as a dynamically loaded plugin, registered in a central **module bank** at startup:

- **input access**
- **demuxer**
- **packetiser**
- **decoder**
- **audio output**
- **video output**
- **user interface**

This design makes VLC highly portable: the same core ships on Linux, Windows, macOS, Android, and iOS by swapping plugins without changing the core pipeline engine.

[Source: VideoLAN VLC source root](https://github.com/videolan/vlc)

VLC is positioned differently from GStreamer (Chapter 58) and FFmpeg (Chapter 57): it is first a consumer product with a full GUI, and second a programmable framework. Where GStreamer demands the application wire up pipeline elements explicitly, VLC chooses the right plugin automatically based on the URL and capability scores. Where FFmpeg is a toolbox of codecs and filters with no concept of a user-facing player, VLC wraps all of FFmpeg's functionality behind a consistent plugin API while adding its own native parsers, output modules, and network protocols.

### Core Objects

Three C types anchor the VLC object hierarchy:

- **`vlc_object_t`** — the base type. Every module instance, every player, every playlist item inherits from `vlc_object_t`. The object tree enables parent-child relationships used for configuration variable inheritance and resource cleanup.
- **`input_thread_t`** — represents an active playback pipeline. Created when a media starts playing; it owns the access, demux, decoder, and output threads.
- **`vout_thread_t`** / **`aout_instance_t`** — the video output and audio output threads, each running a dedicated event loop.

The module bank (`src/modules/`) is populated at startup by scanning plugin directories (`lib/vlc/plugins/` on Linux). Each plugin `.so` is `dlopen()`'d, its `vlc_entry` symbol resolved, and registered into an in-memory module list. Subsequent calls to `vlc_module_load()` search this list by capability string and score.

### Plugin Registration

Each plugin exports an entry-point function built from the `vlc_module_begin()` / `vlc_module_end()` macros defined in `include/vlc_plugin.h`. The macros expand to a standard C symbol `vlc_entry()` that calls `vlc_plugin_set()` and `vlc_module_set()` to register capability, description, callback functions, and a numeric **score**. When the core needs a component of a given capability (e.g. `"video decoder"` or `"audio output"`), it queries the module bank and selects the plugin with the highest score that succeeds its `open()` callback:

```c
/* Minimal plugin descriptor — simplified from include/vlc_plugin.h */
#define vlc_module_begin() \
    EXTERN_SYMBOL DLL_SYMBOL \
    int CDECL_SYMBOL VLC_SYMBOL(vlc_entry)(vlc_set_cb vlc_set, void *opaque) \
    { \
        module_t *module; \
        struct vlc_param *config = NULL; \
        if (vlc_plugin_set(VLC_MODULE_CREATE, &module)) \
            goto error; \
        if (vlc_module_set(VLC_MODULE_NAME, (MODULE_STRING))) \
            goto error;

#define vlc_module_end() \
        (void) config; \
        return 0; \
    error: \
        return -1; \
    }

#define vlc_module_set(...) vlc_set(opaque, module, __VA_ARGS__)
```

[Source: `include/vlc_plugin.h`](https://github.com/videolan/vlc/blob/master/include/vlc_plugin.h)

A concrete decoder plugin starts its descriptor block like:

```c
vlc_module_begin()
    set_description("VA-API H.264 hardware decoder")
    set_capability("video decoder", 800)   /* high score → selected over SW */
    set_callbacks(Open, Close)
    add_shortcut("vaapi_h264")
vlc_module_end()
```

### Pipeline Data Flow

The playback pipeline flows:

```
Input URL
  └─ Access module (file://, http://, dvd://, rtp://)
       └─ Stream/demux module (mkv, mp4, ts, ogg, avformat…)
            ├─ Packetiser (h264, aac, …)
            │    └─ Decoder (VA-API, FFmpeg, V4L2, …)
            │         └─ Video output (Wayland/DMABuf, OpenGL/EGL, Vulkan)
            └─ Audio output (PipeWire, PulseAudio, ALSA)
```

Elementary streams flow as `block_t` linked lists. The decoder thread pulls encoded blocks, hands decoded `picture_t` frames to the video output chain, and keeps audio/video in sync through `es_out_t`.

### libVLC Public Embedding API

Applications embed VLC through **libVLC**, a stable C library whose implementation lives in `lib/` ([source tree](https://github.com/videolan/vlc/tree/master/lib)). The three main types are:

- **`libvlc_instance_t`** — the VLC engine instance (created once per process)
- **`libvlc_media_t`** — a media descriptor (URL + options)
- **`libvlc_media_player_t`** — the active playback engine attached to a media

```c
/* Minimal libvlc embedding example */
#include <vlc/vlc.h>

int main(void) {
    const char *vlc_argv[] = {
        "--no-osd",
        "--avcodec-hw=vaapi",   /* prefer VA-API hardware decode */
    };
    libvlc_instance_t    *inst   = libvlc_new(2, vlc_argv);
    libvlc_media_t       *media  = libvlc_media_new_path("/home/user/video.mkv");
    libvlc_media_player_t *player = libvlc_media_player_new(inst);

    libvlc_media_player_set_media(player, media);
    libvlc_media_release(media);          /* player holds its own ref */

    libvlc_media_player_play(player);
    /* … event loop / sleep … */

    libvlc_media_player_release(player);
    libvlc_release(inst);
    return 0;
}
```

[Source: `lib/media_player.c`](https://github.com/videolan/vlc/blob/master/lib/media_player.c)

### Build System

VLC 3.x uses **autotools**; the master branch (targeting 4.0) has migrated to **Meson** alongside the legacy `configure.ac`. Cross-platform packaging is handled by the `contrib/` tree, which fetches and patches third-party libraries (FFmpeg, libplacebo, libva, etc.) into a sysroot for each target.

On Linux distributions, VLC is typically split into:
- `vlc` — the Qt GUI front-end binary
- `libvlc` — the shared library exposing the embedding API (soname varies by distribution; `libvlc.so.5` on Debian/Ubuntu as of 3.0.x)
- `libvlccore` — the core engine handling the object system, module bank, and threading
- `vlc-plugin-*` packages — individual codec, access, and output plugin packages (exact package names vary by distribution; common examples include `vlc-plugin-base`, `vlc-plugin-video-output`, and GPU-specific packages)

This packaging mirrors the plugin boundary at the package level: distributors can ship the player without optional GPU plugins and users can install them independently.

---

## Demux and Input Layer

### Access Modules

Access modules present a uniform byte-stream or block-stream interface regardless of the underlying transport. Common Linux access plugins include:

| Module | Source | Notes |
|--------|--------|-------|
| `file` | local filesystem | splice() optimisation for large files |
| `http` | libcurl or built-in | HLS playlist detection |
| `rtp` / `rtsp` | UDP/TCP | jitter buffer, RTCP SR for sync |
| `dvd` | libdvdnav + libdvdcss | DVD navigation |
| `bluray` | libbluray | BD-J menus |
| `v4l2` | `modules/access/v4l2/` | webcam / TV capture |

### Clock Hierarchy and ES Management

Time in VLC is represented as `vlc_tick_t` — a signed 64-bit integer counting nanoseconds, similar to `pts`/`dts` in GStreamer. Key constants:

- `VLC_TICK_0` — the zero reference point
- `VLC_TICK_INVALID` — sentinel indicating no valid timestamp
- `vlc_tick_now()` — monotonic wall clock in `vlc_tick_t` units

The `es_out_t` interface sits between the demux and the decoder. The demux registers each Elementary Stream with `es_out_Add()` (receives an `es_out_id_t` token), sends data with `es_out_Send(id, block)`, and advances the master clock with `es_out_SetPCR(es_out, pcr_value)`. The `es_out` implementation dispatches blocks to decoder threads based on `es_out_id_t`, handles decoder selection, and manages the synchronisation state machine.

For adaptive streaming sources, `es_out` supports stream switching: calling `es_out_Control(es_out, ES_OUT_SET_ES, new_id)` at a segment boundary atomically swaps the active representation mid-stream.

### Demux Modules

Above the access layer, demux modules parse container formats and expose **Elementary Streams** (`es_out_t`). Key modules:

- **`avformat`** — wraps `libavformat` from FFmpeg for virtually all containers not handled natively. Lives in `modules/demux/avformat/`.
- **`mkv`** — native Matroska/WebM parser; supports ordered chapters and edition switching.
- **`mp4`** — ISO BMFF/MP4 demuxer with fragment (CMAF/fMP4) support.
- **`ts`** — MPEG-2 Transport Stream; extracts PCR for master clock, handles multi-program streams.
- **`ogg`** — Ogg container for Theora/Vorbis/Opus/FLAC.

[Source: `modules/demux/`](https://github.com/videolan/vlc/tree/master/modules/demux)

### ES Management and AV Sync

Every elementary stream is represented by an `es_out_id_t` registered with `es_out_Add()`. The demux calls `es_out_Send()` to push encoded `block_t` data downstream. Timestamps on each block carry `PTS` and `DTS` in `VLC_TICK_*` units (nanoseconds). For MPEG-2 TS, the PCR (Program Clock Reference) extracted by the `ts` demux establishes the **master clock**; `es_out_SetPCR()` advances it. Audio is typically the secondary reference — the video output waits for the audio clock to compute how long to hold each decoded frame.

The `block_t` structure carries:

```c
/* src/misc/block.c — simplified */
struct block_t {
    struct block_t  *p_next;        /* linked list for block chains */
    uint8_t         *p_buffer;      /* pointer into p_start */
    size_t           i_buffer;      /* usable bytes */
    vlc_tick_t       i_pts;         /* Presentation TimeStamp */
    vlc_tick_t       i_dts;         /* Decoding TimeStamp */
    vlc_tick_t       i_length;      /* duration (for audio) */
    unsigned         i_flags;       /* BLOCK_FLAG_DISCONTINUITY, _PREROLL, etc. */
};
```

[Source: `include/vlc_block.h`](https://github.com/videolan/vlc/blob/master/include/vlc_block.h)

### Adaptive Bitrate Streaming

HLS and DASH adaptive streaming is implemented by the `adaptive` demux plugin (`modules/demux/adaptive/`). It uses libcurl for segment download, maintains a bandwidth estimator, and switches representations based on download throughput. The representation index is an XML/M3U8 playlist fetched by the `http` access module; segment data flows through the same access infrastructure as regular files.

The adaptive demux exposes multiple representations as separate ES tracks: at segment boundaries it calls `es_out_Control(ES_OUT_SET_ES, new_es_id)` to switch to the higher- or lower-bitrate representation. The bandwidth estimator measures segment download speed over a sliding window and applies hysteresis to avoid oscillation near a bitrate boundary.

[Source: `modules/demux/adaptive/`](https://github.com/videolan/vlc/tree/master/modules/demux/adaptive)

### Packetisers

Between the demux and decoder, **packetiser** modules (capability `"packetizer"`) reassemble framing for formats like H.264 Annex B and RTP-packetised H.264. For example, `modules/packetizer/h264.c` accumulates NAL units from a `block_t` chain and emits access-unit-aligned blocks with correct DTS/PTS set. Packetisers are typically transparent to application code but essential for hardware decoders that require whole access units, not raw byte-stream fragments.

---

## Hardware Video Decode — VA-API Path

### Module Location

VLC's VA-API support is split across two directories:

1. **`modules/hw/vaapi/`** — the core VA-API helper library used across VLC's GPU subsystem. [Source](https://github.com/videolan/vlc/tree/master/modules/hw/vaapi). Files:
   - `vlc_vaapi.c` / `vlc_vaapi.h` — the shared `vlc_va_t` interface, display handle management, surface pool helpers
   - `decoder_device.c` — `vlc_decoder_device` implementation wrapping a `VADisplay`
   - `chroma.c` — chroma/pixel format conversion between VLC and VA-API `VAImageFormat`
   - `filters.c` / `filters.h` — VA-API VPP (video post-processing) filter wrappers for de-interlace and scaling

2. **`modules/codec/avcodec/vaapi.c`** — the VA-API hwaccel bridge inside VLC's FFmpeg (`avcodec`) codec plugin. This is where hardware decode actually happens: FFmpeg's `AVCodecContext` is configured with a `VADisplay`-backed `AVHWDeviceContext`, and decoded `AVFrame`s carry `VASurface` handles. [Source](https://github.com/videolan/vlc/tree/master/modules/codec/avcodec)

This architecture means VLC hardware decodes through FFmpeg's `avcodec` wrapper with a VA-API `AVHWContext`, rather than through a standalone VA-API decoder as some other players implement.

### VA-API Decode Flow

The high-level decode sequence maps directly onto the libva API:

```c
/* Pseudocode — actual VLC VA-API flow */

/* 1. Query driver capabilities */
vaQueryConfigProfiles(dpy, profiles, &nprofiles);
vaQueryConfigEntrypoints(dpy, VAProfileH264High, entrypoints, &n);

/* 2. Create a config and decode context */
vaCreateConfig(dpy, VAProfileH264High, VAEntrypointVLD, attrs, n, &cfg);
vaCreateContext(dpy, cfg, width, height, VA_PROGRESSIVE, surfaces,
                POOL_SIZE, &ctx);

/* 3. Per-frame decode */
vaBeginPicture(dpy, ctx, render_target);
vaRenderPicture(dpy, ctx, &pic_param_buf, 1);   /* VAPictureParameterBufferH264 */
vaRenderPicture(dpy, ctx, &slice_buf, 1);        /* VASliceDataBufferType */
vaEndPicture(dpy, ctx);
vaSyncSurface(dpy, render_target);               /* or use fences */
```

[Source: libva API reference](https://intel.github.io/libva/)

### Surface Pool and Recycling

VLC pre-allocates a pool of `VASurface` handles at context creation. Each decoded picture holds a reference to its surface; the pool reclaims the surface when the picture reference count drops to zero (after display). Pool exhaustion causes the decoder to stall, so pool size must accommodate pipeline depth (typically 4–8 frames for B-frame reordering plus display latency).

### Zero-Copy DMA-BUF Export

The performance-critical path avoids any CPU copy between the VA-API decoder output and the display. VLC uses the `VADRMPRIMESurfaceDescriptor` export path:

```c
VADRMPRIMESurfaceDescriptor desc;
vaExportSurfaceHandle(dpy, surface,
                      VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2,
                      VA_EXPORT_SURFACE_READ_ONLY |
                      VA_EXPORT_SURFACE_SEPARATE_LAYERS,
                      &desc);
/* desc.objects[i].fd  — DMA-BUF file descriptor per memory object */
/* desc.layers[i]      — DRM fourcc + modifier + plane offsets/strides */
```

The `VA_EXPORT_SURFACE_DRMPRIME_2` flag (via `VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2`) is the modern export mechanism that carries DRM format modifiers, enabling import into Wayland's `zwp_linux_dmabuf_v1` or EGL's `EGL_LINUX_DMA_BUF_EXT` without tiling/layout mismatch.

[Source: libva `va_drm.h`](https://github.com/intel/libva/blob/master/va/va_drm.h)

### Platform Support

| Driver stack | Supported codecs | Notes |
|---|---|---|
| `i965` (Intel Gen 6–9) | H.264, MPEG-2, VC-1 | Legacy; `iHD` preferred on Gen 9+ |
| `iHD` (Intel Gen 9–14+) | H.264, H.265, VP9, AV1 | `libva-intel-driver` → `intel-media-driver` |
| `radeonsi` (AMD Mesa) | H.264, H.265, VP9, AV1 | Via `libva-mesa-driver` |
| NVIDIA VA-API | H.264 (limited) | Upstream support via `nvidia-vaapi-driver`; quality varies |

### Decoder Selection

VLC's module bank selects the VA-API decoder automatically based on score (800 for vaapi vs ~600 for FFmpeg software). The CLI option `--codec=vaapi_h264` forces selection. `--avcodec-hw=vaapi` enables VA-API inside the `avcodec` wrapper module (which uses `FFmpeg`'s `AVHWContext` infrastructure as an alternative path).

---

## Embedded Hardware Decode — V4L2 and MMAL Paths

### Module Location

VLC does not have a generic `modules/codec/v4l2/` decode plugin; instead, it approaches embedded hardware acceleration differently depending on the platform:

- **Raspberry Pi** (VideoCore IV/VI): `modules/hw/mmal/` — uses the Broadcom **MMAL** (Multi-Media Abstraction Layer) API, not V4L2 M2M, for hardware decode on Raspberry Pi OS. The `codec.c` file implements the MMAL component lifecycle; `vout.c` is a zero-copy display path using MMAL renderer directly to the GPU. [Source: `modules/hw/mmal/`](https://github.com/videolan/vlc/tree/master/modules/hw/mmal)

- **NVIDIA** (desktop/Tegra): `modules/hw/nvdec/` — uses NVIDIA's CUVID/NVDEC API for hardware decode on NVIDIA GPUs and Tegra SoCs.

- **Other embedded platforms**: VLC relies on FFmpeg's V4L2 hwaccel support within `modules/codec/avcodec/` when a V4L2 M2M codec is detected by FFmpeg's `AVHWDeviceType`. This is not a native VLC V4L2 plugin.

The **V4L2 M2M** (mem-to-mem) API does underpin embedded hardware decoders at the kernel level, and understanding it is essential for troubleshooting VLC on those platforms. It targets embedded SoCs where the video accelerator exposes the same device node for an OUTPUT queue (compressed bitstream input) and a CAPTURE queue (decoded frames output).

### V4L2 M2M Decode Flow

```c
/* V4L2 M2M decode — simplified */

/* OUTPUT queue: send compressed NALUs */
struct v4l2_buffer obuf = {
    .type   = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
    .memory = V4L2_MEMORY_MMAP,
    .index  = buf_idx,
};
ioctl(fd, VIDIOC_QBUF, &obuf);   /* enqueue encoded data */
ioctl(fd, VIDIOC_STREAMON, &OUTPUT_type);

/* CAPTURE queue: receive decoded frames */
struct v4l2_buffer cbuf = {
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
    .memory = V4L2_MEMORY_DMABUF,  /* zero-copy: kernel exports DMA-BUF */
};
ioctl(fd, VIDIOC_DQBUF, &cbuf);   /* dequeue decoded frame */
/* cbuf.m.planes[0].m.fd — DMA-BUF fd for the Y plane */
```

[Source: V4L2 M2M kernel documentation](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-mem2mem.html)

### V4L2 M2M Architecture (Kernel-Level)

V4L2 M2M is the kernel interface that embedded SoC video decoders expose. Understanding it is necessary for troubleshooting VLC playback on ARM boards, even if VLC's userspace path goes through FFmpeg's hwaccel layer rather than directly:

The **V4L2 stateless API** (used by modern drivers like `rkvdec`, `hantro`) requires the application to provide full codec slice parameters each frame (in a structured control set like `V4L2_CID_STATELESS_H264_SPS`, `_PPS`, `_SLICE_PARAMS`); the kernel driver handles only DMA scheduling and hardware bitfield injection. The **stateful API** (`bcm2835-codec` for Raspberry Pi, Qualcomm Venus) hides this complexity — the driver manages codec state internally, requiring only raw bitstream input.

[Source: V4L2 stateless codec API](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-stateless-decoder.html)

### V4L2 Platform Coverage

| Platform | Kernel driver | VLC path | API type |
|---|---|---|---|
| Raspberry Pi 4/5 (VideoCore VI) | `bcm2835-codec` | `modules/hw/mmal/` (MMAL, not V4L2) | MMAL |
| Rockchip RK3399/RK3588 | `rkvdec` | FFmpeg V4L2 hwaccel → `avcodec` | V4L2 stateless |
| Amlogic S905/S922 | `meson-vdec` | FFmpeg V4L2 hwaccel → `avcodec` | V4L2 stateful/stateless |
| Qualcomm Snapdragon (Venus) | `venus` | FFmpeg V4L2 hwaccel → `avcodec` | V4L2 stateful |
| AllWinner (CEDRUS) | `cedrus` | FFmpeg V4L2 hwaccel → `avcodec` | V4L2 stateless |

### V4L2 Capture Access (Webcams, TV Tuners)

VLC uses V4L2 extensively for camera and tuner *capture* (input, not decode) via `modules/access/v4l2/`. [Source](https://github.com/videolan/vlc/tree/master/modules/access/v4l2). Key files:

- `v4l2.c` — module entry, device enumeration
- `video.c` — pixel format negotiation, `VIDIOC_S_FMT`, `VIDIOC_QBUF`/`VIDIOC_DQBUF` capture loop
- `access.c` — stream access wrapper for non-video (radio) devices
- `controls.c` — V4L2 controls (brightness, contrast, gain) exposed as VLC variables

The access module enumerates available pixel formats with `VIDIOC_ENUM_FMT`, selects the best match for VLC's internal chroma list, and captures frames. Memory mode is either `V4L2_MEMORY_MMAP` (kernel-mapped buffers) or `V4L2_MEMORY_USERPTR` (application-allocated). DMA-BUF export (`V4L2_MEMORY_DMABUF`) can enable zero-copy delivery to a Wayland compositor or OpenGL texture when the webcam driver supports it.

Webcam URL: `vlc v4l2:///dev/video0 :v4l2-width=1920 :v4l2-height=1080`

---

## Video Output — Wayland Zero-Copy Path

### Module Structure

The Wayland video output lives in `modules/video_output/wayland/`. [Source](https://github.com/videolan/vlc/tree/master/modules/video_output/wayland). Key files:

- `output.c` / `output.h` — core Wayland output: `wl_display`, `wl_surface`, `wl_compositor`
- `xdg-shell.c` — `xdg_surface` / `xdg_toplevel` for desktop window management
- `shm.c` — shared-memory fallback path (CPU copy via `wl_shm`): used when DMA-BUF import is unavailable
- `vulkan.c` — Vulkan WSI on Wayland (see Section 6)
- `input.c` / `input.h` — pointer and keyboard input handling
- `registry.c` / `registry.h` — Wayland protocol registry and interface binding

### Window Setup and Protocol Binding

VLC's Wayland output binds a minimal set of Wayland globals at startup:

| Global | Purpose |
|--------|---------|
| `wl_compositor` | Create `wl_surface` objects |
| `xdg_wm_base` | Stable XDG shell: desktop windows |
| `wp_viewporter` | GPU-side scaling of the video surface |
| `zwp_linux_dmabuf_v1` | Import DMA-BUF file descriptors as `wl_buffer` |
| `wl_seat` | Keyboard and pointer input |
| `linux_drm_syncobj_manager_v1` | Explicit GPU fences (VLC 4.0 master, if available) |

On Wayland, VLC creates an `xdg_toplevel` (via the XDG Shell protocol) to receive a desktop window, and uses `wp_viewporter` for GPU-side scaling — the compositor scales the video surface to the output size without any CPU involvement.

```c
/* Pseudocode — Wayland output init (output.c) */
wl_surface      *surface   = wl_compositor_create_surface(compositor);
xdg_surface     *xdg_surf  = xdg_wm_base_get_xdg_surface(wm_base, surface);
xdg_toplevel    *toplevel  = xdg_surface_get_toplevel(xdg_surf);
xdg_toplevel_set_title(toplevel, "VLC");

wp_viewport     *viewport  = wp_viewporter_get_viewport(viewporter, surface);
wp_viewport_set_destination(viewport, output_width, output_height);
```

### DMA-BUF Zero-Copy via zwp_linux_dmabuf_v1

When VA-API or V4L2 decoded frames are available as DMA-BUF file descriptors, VLC imports them directly as `wl_buffer` objects using the `zwp_linux_dmabuf_v1` Wayland protocol extension. The decoded frame never touches the CPU:

```c
/* Zero-copy: VA-API DMA-BUF → wl_buffer */

/* 1. Export VASurface as DMA-BUF (see Section 3) */
vaExportSurfaceHandle(va_dpy, surface,
                      VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2,
                      VA_EXPORT_SURFACE_READ_ONLY, &prime_desc);

/* 2. Create dmabuf params and import each plane */
zwp_linux_buffer_params_v1 *params =
    zwp_linux_dmabuf_v1_create_params(dmabuf_proto);

for (int i = 0; i < prime_desc.num_layers; i++) {
    zwp_linux_buffer_params_v1_add(params,
        prime_desc.objects[prime_desc.layers[i].object_index[0]].fd,
        i,                                          /* plane index */
        prime_desc.layers[i].offset[0],
        prime_desc.layers[i].pitch[0],
        prime_desc.objects[0].drm_format_modifier >> 32,
        prime_desc.objects[0].drm_format_modifier & 0xffffffff);
}

/* 3. Commit wl_buffer to wl_surface */
struct wl_buffer *wl_buf =
    zwp_linux_buffer_params_v1_create_immed(params, width, height,
                                            DRM_FORMAT_NV12, 0);
wl_surface_attach(surface, wl_buf, 0, 0);
wl_surface_commit(surface);
```

[Source: `zwp_linux_dmabuf_v1` protocol specification](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml)

The buffer lifecycle is: `VASurface` → `VADRMPRIMESurfaceDescriptor` → `zwp_linux_buffer_params_v1` → `wl_buffer` → `wl_surface_attach()`. The compositor (e.g. Mutter, KWin, sway) imports the DMA-BUF into its own GPU context for final composition with other surfaces.

### Explicit Sync (VLC master / 4.0)

Modern compositors advertise `linux-drm-syncobj-v1` for **explicit synchronisation** — passing GPU timeline fence objects alongside buffer submissions so the compositor waits on the decoder's completion fence before scanning out. In the VLC 4.0 master branch, the Wayland output can attach a `drm_syncobj` release fence to `wl_surface` commits, eliminating `vaSyncSurface()` busy-waits on the CPU and reducing display latency. Note: this feature is under active development and may not be in a stable release as of mid-2026.

[Source: `linux-drm-syncobj-v1` protocol](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/linux-drm-syncobj/linux-drm-syncobj-v1.xml)

---

## Video Output — Vulkan Renderer (VLC 4.0 master)

### Module Location

The Vulkan video output module lives in `modules/video_output/vulkan/` and `modules/video_output/wayland/vulkan.c`. [Source](https://github.com/videolan/vlc/tree/master/modules/video_output/vulkan). Key files:

- `platform.c` / `platform.h` — abstract Vulkan surface platform interface
- `platform_xcb.c` — X11/XCB: `VK_KHR_xcb_surface`
- `instance.h` — Vulkan instance and device selection helpers

The Wayland integration (`wayland/vulkan.c`) uses `VK_KHR_wayland_surface` to create the swapchain surface directly from the `wl_display` and `wl_surface`.

> **Note**: As of June 2026, VLC 4.0 is unreleased; the Vulkan renderer exists on the master branch and nightly builds but is not yet the stable default. VideoLAN's SoC 2026 projects include porting video filters (tonemapping, subtitle rendering, deinterlace) from OpenGL to Vulkan, indicating the Vulkan path is still maturing. [Source: SoC 2026 wiki](https://wiki.videolan.org/SoC_2026/)

### VA-API → Vulkan Image Import

For true zero-copy from the hardware decoder to Vulkan rendering, VLC imports a decoded `VASurface` as a `VkImage` using the `VK_EXT_external_memory_dma_buf` + `VK_EXT_image_drm_format_modifier` extension pair:

```c
/* Import DMA-BUF fd into Vulkan — pseudocode */

/* Plane layout from VA-API export */
VkSubresourceLayout plane_layouts[2] = {
    { .offset = desc.layers[0].offset[0], .rowPitch = desc.layers[0].pitch[0] },
    { .offset = desc.layers[0].offset[1], .rowPitch = desc.layers[0].pitch[1] },
};

VkImageDrmFormatModifierExplicitCreateInfoEXT mod_info = {
    .sType              = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_EXPLICIT_CREATE_INFO_EXT,
    .drmFormatModifier  = desc.objects[0].drm_format_modifier,
    .drmFormatModifierPlaneCount = 2,
    .pPlaneLayouts      = plane_layouts,
};

VkExternalMemoryImageCreateInfo ext_mem_info = {
    .sType       = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO,
    .pNext       = &mod_info,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};

VkImageCreateInfo img_info = {
    .sType  = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext  = &ext_mem_info,
    .format = VK_FORMAT_G8_B8R8_2PLANE_420_UNORM,  /* NV12 */
    .tiling = VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT,
    /* … width, height, usage flags … */
};

vkCreateImage(device, &img_info, NULL, &vk_image);

/* Import the DMA-BUF memory */
VkImportMemoryFdInfoKHR import_info = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = prime_desc.objects[0].fd,
};
vkAllocateMemory(device, &alloc_info_with_import, NULL, &image_memory);
vkBindImageMemory(device, vk_image, image_memory, 0);
```

[Source: `VK_EXT_image_drm_format_modifier` specification](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html)

### libplacebo Integration

VLC's Vulkan renderer delegates high-quality image processing — HDR tone mapping, debanding, film grain synthesis, upscaling — to **libplacebo**. The canonical libplacebo repository is at `code.videolan.org/videolan/libplacebo`; the GitHub mirror at `github.com/haasn/libplacebo` is maintained by the same author. libplacebo originated from mpv's shader system and was adopted by VLC for the Vulkan renderer:

```c
/* libplacebo integration — pseudocode */
pl_vulkan     plvk  = pl_vulkan_create(pl_log, &vulkan_params);
pl_renderer   rend  = pl_renderer_create(pl_log, plvk->gpu);
pl_tex        src   = pl_tex_create(plvk->gpu, &(struct pl_tex_params){
    .w         = frame_width,
    .h         = frame_height,
    .format    = pl_find_fourcc(plvk->gpu, MKTAG('N','V','1','2')),
    .sampleable = true,
});

/* For HDR content: tone-map PQ → SDR via Vulkan compute */
struct pl_render_params render_params = pl_render_default_params;
render_params.color_map_params = &pl_color_map_default_params;  /* BT.2020 → BT.709 */
render_params.upscaler         = &pl_filter_mitchell;

pl_render_image(rend, &frame, &target, &render_params);
```

[Source: libplacebo documentation](https://code.videolan.org/videolan/libplacebo)

libplacebo capabilities used in VLC:

- **HDR tone mapping**: PQ (ST.2084) → SDR BT.709 or PQ → HLG via scene-based histogram tone mapping (Vulkan compute shaders)
- **Debanding**: removes banding artefacts from 8-bit sources
- **Upscaling**: Lanczos, Mitchell-Netravali, or AMD FidelityFX Super Resolution (FSR) integer scaling
- **Film grain synthesis**: AV1 film grain via the AV1 spec's grain table (GPU-side)

### Subtitle and OSD Rendering

FreeType-rasterised subtitle bitmaps are uploaded as `VkImage` textures and composited via a fragment shader over the video image. The subtitle compositing occurs in the same Vulkan render pass as YUV→RGB conversion, avoiding a second render pass.

---

## OpenGL/EGL Renderer

### Module Location

The OpenGL video output lives in `modules/video_output/opengl/`. [Source](https://github.com/videolan/vlc/tree/master/modules/video_output/opengl). This is the current stable hardware-accelerated renderer on Linux (as of VLC 3.0.x).

Key files:
- `display.c` — main display module; selects EGL or GLX context
- `renderer.c` — YUV→RGB shader compilation and draw call dispatch
- `sampler.c` — texture sampler abstraction: handles NV12, I420, P010, RGBA, etc.
- `interop_vaapi.c` — VA-API EGLImage interop (zero-copy import of VASurface)
- `interop_sw.c` — software fallback: uploads `picture_t` data via `glTexImage2D`
- `sub_renderer.c` — subtitle/OSD rendering via GL texture atlas
- `egl.c` / `egl_display.c` — EGL context creation for Wayland (`wl_egl_window`) and KMS/DRM (`EGL_PLATFORM_GBM_KHR`)
- `egl_display_gbm.c` — GBM-based EGL display for KMS direct rendering (no compositor)

### EGL Context Setup

On Wayland, VLC creates an EGL context via `wl_egl_window` and `eglCreateWindowSurface()`:

```c
/* EGL on Wayland */
EGLDisplay egl_dpy = eglGetPlatformDisplay(EGL_PLATFORM_WAYLAND_KHR,
                                           wl_display, NULL);
eglInitialize(egl_dpy, &major, &minor);

EGLConfig cfg;
eglChooseConfig(egl_dpy, attribs, &cfg, 1, &n);

struct wl_egl_window *wl_win = wl_egl_window_create(wl_surface, w, h);
EGLSurface egl_surf = eglCreateWindowSurface(egl_dpy, cfg,
                                             (EGLNativeWindowType)wl_win, NULL);
EGLContext ctx = eglCreateContext(egl_dpy, cfg, EGL_NO_CONTEXT, ctx_attribs);
eglMakeCurrent(egl_dpy, egl_surf, egl_surf, ctx);
```

### GLSL YUV→RGB Conversion

`renderer.c` compiles a GLSL fragment shader that performs colour space conversion. For NV12 (Y + interleaved UV):

```glsl
/* GLSL fragment shader — YCbCr BT.709 → RGB (simplified) */
uniform sampler2D tex_y;    /* luminance plane */
uniform sampler2D tex_uv;   /* interleaved chroma plane */
uniform mat4 yuv_matrix;    /* BT.601 or BT.709 or BT.2020 conversion */

in vec2 TexCoord;
out vec4 FragColor;

void main() {
    float y  = texture(tex_y,  TexCoord).r;
    vec2  uv = texture(tex_uv, TexCoord).rg - vec2(0.5);
    vec3  yuv = vec3(y - 0.0625, uv.x, uv.y);
    FragColor = vec4(clamp(mat3(yuv_matrix) * yuv, 0.0, 1.0), 1.0);
}
```

The matrix is selected at runtime based on the frame's `video_color_space_t` (BT.601 / BT.709 / BT.2020).

### VA-API EGLImage Zero-Copy Interop

`interop_vaapi.c` imports a decoded `VASurface` into OpenGL without CPU readback, using the `EGL_EXT_image_dma_buf_import` extension:

```c
/* interop_vaapi.c — zero-copy VA-API → EGLImage → GL texture */

/* Export VASurface as DMA-BUF */
vaExportSurfaceHandle(va_dpy, va_surface,
                      VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2,
                      VA_EXPORT_SURFACE_READ_ONLY, &prime_desc);

/* Create EGLImage from DMA-BUF */
EGLAttrib img_attrs[] = {
    EGL_LINUX_DRM_FOURCC_EXT,         DRM_FORMAT_NV12,
    EGL_WIDTH,                         frame_width,
    EGL_HEIGHT,                        frame_height,
    EGL_DMA_BUF_PLANE0_FD_EXT,        prime_desc.objects[0].fd,
    EGL_DMA_BUF_PLANE0_OFFSET_EXT,    prime_desc.layers[0].offset[0],
    EGL_DMA_BUF_PLANE0_PITCH_EXT,     prime_desc.layers[0].pitch[0],
    EGL_DMA_BUF_PLANE1_FD_EXT,        prime_desc.objects[0].fd,
    EGL_DMA_BUF_PLANE1_OFFSET_EXT,    prime_desc.layers[0].offset[1],
    EGL_DMA_BUF_PLANE1_PITCH_EXT,     prime_desc.layers[0].pitch[1],
    EGL_NONE,
};
EGLImage egl_img = eglCreateImage(egl_dpy, EGL_NO_CONTEXT,
                                   EGL_LINUX_DMA_BUF_EXT, NULL, img_attrs);

/* Bind as GL texture */
glBindTexture(GL_TEXTURE_2D, tex_y);
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, egl_img);
```

[Source: `EGL_EXT_image_dma_buf_import` extension specification](https://registry.khronos.org/EGL/extensions/EXT/EGL_EXT_image_dma_buf_import.txt)

This path enables complete GPU-to-GPU video decode and display without touching system RAM for the decoded pixel data.

### GBM/KMS Direct Rendering

The `egl_display_gbm.c` backend enables VLC to render directly to a KMS `CRTC` without any compositor, using GBM (`libgbm`) as the EGL platform. This is useful for kiosk or embedded scenarios:

```c
/* KMS/GBM direct rendering setup */
int drm_fd   = open("/dev/dri/card0", O_RDWR);
struct gbm_device  *gbm  = gbm_create_device(drm_fd);
EGLDisplay egl_dpy = eglGetPlatformDisplay(EGL_PLATFORM_GBM_KHR, gbm, NULL);
eglInitialize(egl_dpy, NULL, NULL);

struct gbm_surface *gbm_surf = gbm_surface_create(gbm, w, h,
                                                   GBM_FORMAT_XRGB8888,
                                                   GBM_BO_USE_SCANOUT |
                                                   GBM_BO_USE_RENDERING);
EGLSurface egl_surf = eglCreateWindowSurface(egl_dpy, cfg,
                                              (EGLNativeWindowType)gbm_surf, NULL);
```

After `eglSwapBuffers()`, VLC locks the front buffer with `gbm_surface_lock_front_buffer()`, extracts its DRM framebuffer handle via `drmModeAddFB2()`, and calls `drmModePageFlip()` to present it. This path is used in Raspberry Pi OS kiosk builds and digital signage installations running without a Wayland compositor.

### OpenGL Interop Module Architecture

The OpenGL vout uses a two-level interop abstraction that separates **texture upload** from **rendering**:

1. **Interop module** (`vlc_gl_interop_t`): owns the mapping from `picture_t` → GL textures. For VA-API it creates `EGLImage` objects; for software frames it uploads via `glTexSubImage2D`. Multiple interop modules can coexist in the module bank; the renderer selects the highest-scoring one that accepts the decoder's output chroma.

2. **Sampler module** (`vlc_gl_sampler_t`): generates GLSL declarations for the interop textures and injects uniform binding calls. The sampler abstracts the difference between a single RGBA texture, a biplanar NV12 texture pair, and a triplanar I420 texture triple.

3. **Renderer** (`renderer.c`): compiles the final GLSL program by linking sampler output with geometry (a simple quad), handles viewport transforms, and issues the draw call.

This layering means that adding support for a new pixel format (e.g. 10-bit P010 for HEVC HDR content) requires only a new sampler GLSL snippet and an updated interop upload path — the renderer geometry code is reused unchanged.

---

## Audio Output on Linux

### ALSA

The ALSA audio output module is `modules/audio_output/alsa.c`. It opens the PCM device with `snd_pcm_open()`, sets hardware parameters (sample rate, channel count, format, period size), and writes decoded PCM in a loop using `snd_pcm_writei()` or `snd_pcm_writen()`. The module monitors underruns with `snd_pcm_status()` and calls `snd_pcm_recover()` on `EPIPE`.

ALSA is used for **IEC 958 / SPDIF passthrough** (AC-3, DTS, EAC-3) — the module opens `hw:0,1` with `snd_pcm_format_t = SND_PCM_FORMAT_S16_LE` and sets the MPEG status byte to indicate non-PCM content. For HDMI audio, ALSA's HDA codec driver exposes IEC 958 via the `SNDRV_PCM_IOCTL_SYNC_PTR` path.

A typical ALSA write cycle in VLC:

```c
/* modules/audio_output/alsa.c — simplified write loop */
snd_pcm_t *pcm;
snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, 0);

snd_pcm_hw_params_t *hw_params;
snd_pcm_hw_params_any(pcm, hw_params);
snd_pcm_hw_params_set_format(pcm, hw_params, SND_PCM_FORMAT_S16_LE);
snd_pcm_hw_params_set_rate_near(pcm, hw_params, &rate, 0);
snd_pcm_hw_params_set_channels(pcm, hw_params, 2);
snd_pcm_hw_params(pcm, hw_params);

/* Per-period write */
snd_pcm_sframes_t written = snd_pcm_writei(pcm, pcm_buf, frames);
if (written == -EPIPE)
    snd_pcm_recover(pcm, written, 0);
```

[Source: `modules/audio_output/alsa.c`](https://github.com/videolan/vlc/blob/master/modules/audio_output/alsa.c)

### PipeWire (Preferred on Modern Wayland Desktops)

`modules/audio_output/pipewire.c` integrates with the PipeWire session manager using the simple stream API:

```c
/* PipeWire audio output — simplified */
struct pw_main_loop *loop = pw_main_loop_new(NULL);
struct pw_context   *ctx  = pw_context_new(pw_main_loop_get_loop(loop),
                                           NULL, 0);
struct pw_core      *core = pw_context_connect(ctx, NULL, 0);

/* SPA audio format negotiation */
struct spa_audio_info_raw audio_info = {
    .format   = SPA_AUDIO_FORMAT_S16_LE,
    .rate     = 48000,
    .channels = 2,
};

struct pw_stream *stream = pw_stream_new_simple(
    pw_main_loop_get_loop(loop), "VLC audio",
    pw_properties_new(PW_KEY_MEDIA_TYPE,     "Audio",
                      PW_KEY_MEDIA_CATEGORY, "Playback",
                      PW_KEY_MEDIA_ROLE,     "Movie", NULL),
    &stream_events, vlc_module);

struct spa_pod *params[1];
params[0] = spa_format_audio_raw_build(&b, SPA_PARAM_EnumFormat, &audio_info);

pw_stream_connect(stream, PW_DIRECTION_OUTPUT, PW_ID_ANY,
                  PW_STREAM_FLAG_AUTOCONNECT | PW_STREAM_FLAG_MAP_BUFFERS,
                  params, 1);
```

[Source: `modules/audio_output/pipewire.c`](https://github.com/videolan/vlc/blob/master/modules/audio_output/pipewire.c)

PipeWire is preferred on modern GNOME (≥42) and KDE (≥5.25) Wayland desktops where PulseAudio compatibility is provided via `pipewire-pulse`. VLC also uses PipeWire for screen capture via the `xdg-desktop-portal` screencast portal (relevant for `vlc screen://`).

### PulseAudio

`modules/audio_output/pulse.c` uses the async PulseAudio API (`pa_stream_new()`, `pa_stream_connect_playback()`, `pa_stream_write()`) and registers for server-side volume/mute callbacks. PulseAudio remains the fallback on systems without PipeWire.

### AV Synchronisation

The audio clock drives video presentation timing during file playback. The audio output module reports its running clock position (accounting for driver/hardware latency) so the video output module can compute when to display each frame. For MPEG-2 TS streams, the PCR from the demux is the master; the audio output slews its internal clock to match. Audio latency is queried from `snd_pcm_status_get_delay()` (ALSA) or from the PipeWire node's reported latency.

The synchronisation state machine handles:
- **Normal play**: audio clock is master; video output sleeps until `pts - pipeline_latency`
- **Clock discontinuities**: after seeking, all decoders flush, PCR is re-locked, and audio prerolls before video displays
- **Speed control**: VLC's clock abstraction stretches/shrinks presentation timestamps uniformly across audio (which uses a time-stretching filter) and video (which drops or duplicates frames)

### HDMI/SPDIF Passthrough

AC-3, DTS, DTS-HD, EAC-3, and TrueHD can be passed through bitstream-intact to AV receivers without decoding. Configuration:

```bash
# Force SPDIF/IEC 958 passthrough via ALSA
vlc --audio-passthrough --sout-audio-passthrough video.mkv

# Or via PipeWire RAW passthrough (Note: needs verification for TrueHD)
vlc --aout=pipewire --audio-passthrough video.mkv
```

The `alsa.c` module detects `AUDIO_FORMAT_SPDIFB` (big-endian IEC 958) and opens the ALSA device in IEC 958 mode rather than PCM. The SPDIF framing (IEC 60958 preamble, channel status bits) is inserted by VLC's `spdif.c` packetiser before the ALSA write call.

For HDMI multi-channel audio (7.1 PCM, DTS-HD MA, TrueHD), VLC queries the ALSA HDMI sink's `eld` (`EDID-Like Data`) via `snd_hctl_find_elem()` to determine which formats the connected display/receiver advertises as supported. The ALSA `hdmi` plugin then negotiates the correct IEC 60958 channel status. See Chapter 140 for the full HDMI audio endpoint architecture.

---

## Transcoding and Streaming with --sout

### Stream Output Chain Syntax

VLC's `--sout` (stream output) option takes a **chain expression** — a sequence of modules joined by `:` that the encoded/decoded stream is fed through. Each module name is followed by options in `{}`. The chain is parsed left-to-right.

```
#module1{opt=val,...}:module2{opt=val,...}:sink{...}
```

The `#` prefix indicates a chain. Pipes between modules pass `block_t` encoded data; a `transcode` module sits between the demux and the mux/sink.

### Transcode and Network Stream Example

```bash
# Transcode MKV H.265 → H.264/AAC, stream via RTP/TS to remote host
vlc input.mkv \
  --sout "#transcode{vcodec=h264,venc=x264{preset=fast,crf=22}, \
                     acodec=mp4a,ab=128,channels=2,samplerate=44100}: \
          rtp{dst=192.168.1.10,port=5004,mux=ts}"
```

### VA-API Hardware Encode in Transcode

When available, VLC can use VA-API H.264 encode via the `vaapi` encoder plugin:

```bash
# Hardware H.264 encode via VA-API (Intel/AMD)
vlc input.mkv \
  --sout "#transcode{vcodec=h264,venc=vaapi{quality=25}, \
                     acodec=mp4a,ab=128}: \
          file{dst=output_hw.mp4,mux=mp4}"
```

VA-API encode support in VLC's `transcode` module covers H.264 and HEVC on Intel/AMD; quality and options depend on driver capabilities. The underlying `modules/stream_out/transcode/` pipeline calls the encoder plugin's `encode()` callback per decoded `picture_t`. 

When hardware decode and hardware encode are both active, VLC can route decoded `picture_t` objects directly from the VA-API decoder surface pool to the VA-API encoder — avoiding a GPU→CPU→GPU round-trip. This zero-copy transcode path requires both the decoder and encoder to share the same VA-API `VADisplay` handle so surfaces can be passed without export. Note: the exact internal opaque pixel format identifier for VA-API decoded frames varies by VLC version — consult `modules/hw/vaapi/vlc_vaapi.h` for the current definition.

[Source: `modules/stream_out/transcode/`](https://github.com/videolan/vlc/tree/master/modules/stream_out/transcode)

[Source: `modules/stream_out/transcode/`](https://github.com/videolan/vlc/tree/master/modules/stream_out/transcode)

### HTTP Streaming Server

VLC can serve live or on-demand content over HTTP:

```bash
# Serve re-encoded stream over HTTP (e.g. for smart TVs on LAN)
vlc input.mkv \
  --sout "#transcode{vcodec=h264,venc=x264,acodec=mp4a}: \
          http{mux=ts,dst=:8080/stream}"
```

A browser or media player can then consume `http://server-ip:8080/stream` as an MPEG-TS HTTP stream.

### Mosaic Filter

The `mosaic` stream output filter enables multi-stream mixing — N input videos are composited into a single output with configurable layout. Each input is a separate VLC instance connected via a bridge stream output:

```bash
# Mosaic bridge sender (source stream A)
vlc source_a.mp4 \
  --sout "#transcode{vcodec=mp4v}:mosaic-bridge{id=1,width=320,height=240}"

# Mosaic compositor
vlc --mosaic-rows=1 --mosaic-cols=2 --mosaic-position=0 \
  --sout "#mosaic:display"
```

---

## VLC 4.0 and Future Direction

### Release Status (as of June 2026)

VLC 4.0 is under active development on the master branch. Nightly builds are available at [nightlies.videolan.org](https://nightlies.videolan.org/). The latest stable release is 3.0.23 (January 2026). VLC 4.0 introduces breaking API changes in libVLC and several architectural overhauls; no stable release date has been announced at time of writing.

[Source: VLC nightly builds](https://nightlies.videolan.org/)

### Qt6 UI and Interface Overhaul

VLC 4.0 migrates the desktop interface from Qt5 to Qt6 (a first update landed in nightly builds in late 2025 per [OMG Ubuntu reporting](https://www.omgubuntu.co.uk/2025/09/vlc-player-update-2025)). Qt6 brings native Wayland QPA support — VLC runs as a native Wayland client rather than falling back to XWayland, enabling proper HiDPI scaling, Wayland input method integration, and `xdg_toplevel` window decorations via Qt's Wayland decoration plugin.

An experimental **libadwaita**-based UI is in development for better GNOME integration (using GTK4 and the libadwaita adaptive widget set). A separate **macOS**-style SwiftUI front-end also exists for iOS/iPadOS.

The **media library** (`medialibrary`, a separate C++ library developed by VideoLAN) provides local content indexing: it scans configured directories, extracts metadata from audio/video files, and stores results in an SQLite database. Artwork is fetched from MusicBrainz (audio) and TheMovieDB (video). In the VLC 4.0 Qt6 UI, the home screen browser replaces the flat playlist with a media library grid view.

### Vulkan as Default Renderer (in progress)

The VLC master branch is migrating to Vulkan (via libplacebo) as the primary GPU renderer on platforms that support it, replacing the OpenGL/EGL renderer as the high-quality default. As VideoLAN's SoC 2026 projects show, core features (subtitle rendering, tonemapping, deinterlace) are still being ported from OpenGL to Vulkan shaders. The full zero-copy pipeline on completion would be:

```
VA-API decode → VASurface
    ↓ vaExportSurfaceHandle (DRM PRIME 2)
DMA-BUF fd + DRM format modifier
    ↓ vkCreateImage (VK_EXT_image_drm_format_modifier)
VkImage (NV12, tiled)
    ↓ pl_renderer_render() [libplacebo]
       · HDR tone mapping (PQ→SDR via compute shader)
       · Debanding, film grain, scaling
VkSwapchainKHR present (via VK_KHR_wayland_surface)
    ↓
wl_surface → compositor → display
```

[Source: VideoLAN SoC 2026](https://wiki.videolan.org/SoC_2026/)

### libplacebo HDR Tone Mapping

libplacebo (canonical home: [code.videolan.org/videolan/libplacebo](https://code.videolan.org/videolan/libplacebo)) provides:

- **BT.2020 PQ → BT.709 SDR** tone mapping for HDR10 content on SDR displays
- **Dolby Vision** processing (Profile 5 IPT-PQ colour space, a SoC 2026 project)
- **HLG → SDR** conversion for broadcast HDR
- Scene-based histogram tone mapping that adapts to average picture level per scene

The Vulkan compute backend runs these transformations entirely on the GPU, making them compatible with the zero-copy pipeline described above.

### Debugging VLC GPU Pipelines

For contributors and advanced users, VLC's internal subsystems expose debugging output through environment variables and command-line options:

```bash
# Verbose plugin selection (shows scored module list)
vlc -vvv --verbose=3 video.mkv 2>&1 | grep -i "module\|vout\|vaapi"

# List loaded modules and their capabilities
vlc --list

# Force specific video output and decoder
vlc --vout=gl --codec=vaapi_h264 video.mkv

# Disable hardware decode entirely (software fallback)
vlc --avcodec-hw=none video.mkv

# Check which VA-API driver is loaded
LIBVA_MESSAGING_LEVEL=1 vlc video.mkv 2>&1 | head -20

# PipeWire audio with verbose SPA logging
PIPEWIRE_DEBUG=3 vlc --aout=pipewire video.mkv
```

The `--codec` flag accepts any module shortcut string (e.g. `vaapi_h264`, `vaapi_hevc`, `dav1d` for AV1 software decode). `--vout` accepts `gl` (OpenGL/EGL), `wayland` (SHM fallback), `vulkan`, `xcb_glx`, `drm` (KMS direct).

### VLC as a Platform

**libVLC** is the embeddable interface for third-party applications:

- **vlcj** — Java bindings for desktop embedding ([github.com/caprica/vlcj](https://github.com/caprica/vlcj)); wraps the full libVLC event model and media discovery API
- **VLC-Android** / **VLC-iOS** — use libVLC directly with native Vulkan/Metal renderers; the Android build uses an OpenGL ES 3.0 renderer with `SurfaceTexture` for display
- **VLC.js** — experimental browser-hosted VLC via WebAssembly (Emscripten build); software decode only, useful for embedded players in web kiosks
- **python-vlc** — ctypes-based Python bindings generated from the libVLC header; widely used for scriptable media players and test harnesses

The libVLC API (`libvlc_instance_t`, `libvlc_media_player_t`, `libvlc_event_t`) is stable across minor versions within a major; VLC 4.0 is a breaking change for some callbacks. Applications that rely on `libvlc_media_player_set_xwindow()` (X11 embed) or `libvlc_media_player_set_hwnd()` (Win32 embed) need review as 4.0 moves toward native Wayland windowing.

### Comparison with GStreamer and FFmpeg

VLC, GStreamer, and FFmpeg serve overlapping but distinct roles in the Linux multimedia ecosystem:

| Aspect | VLC | GStreamer | FFmpeg |
|--------|-----|-----------|--------|
| Primary use case | End-user player + embedding | Application pipeline framework | Codec/container library + CLI tool |
| Plugin model | Score-based auto-selection | Explicit graph wiring | Static codec registry |
| VA-API path | `modules/codec/vaapi/` | `va` plugin (`gst-plugins-bad`) | `AVHWContext` / `vaapi` hwaccel |
| Wayland zero-copy | `zwp_linux_dmabuf_v1` in vout | `GstDmaBufAllocator` + `waylandsink` | N/A (no display layer) |
| Vulkan renderer | In development (4.0 master) | `vulkansink`, `vulkandecodebin` | `vulkan` hwaccel (encode/decode) |
| Audio | PipeWire → PulseAudio → ALSA | `pipewiresink` → `pulsesink` → `alsasink` | `libavdevice` (no session management) |
| Transcoding | `--sout` chain | `gst-launch` pipeline | `ffmpeg -i ... -c:v ...` |

This chapter's VA-API and Wayland sections are directly applicable to GStreamer's `va` and `waylandsink` elements (Chapter 58) — the underlying kernel interfaces (`vaExportSurfaceHandle`, `zwp_linux_dmabuf_v1`) are identical; only the framework wrappers differ.

---

## Integrations

This chapter connects directly to the following chapters:

- **Chapter 26 — VA-API Hardware Decode**: VLC's primary GPU decode path. Section 3 of this chapter shows the exact VA-API surface pool and DMA-BUF export calls VLC uses; Chapter 26 covers the full libva API, driver architecture (i965, iHD, radeonsi), and entrypoint discovery.

- **Chapter 38 — PipeWire**: VLC's preferred audio output on modern Wayland desktops (Section 8). Chapter 38 covers the PipeWire session graph, `pw_stream` API, and SPA node negotiation in depth. VLC also uses PipeWire for screen capture via the XDG portal.

- **Chapter 57 — FFmpeg**: VLC's `avcodec` and `avformat` modules wrap FFmpeg for software decode and many container formats. Section 2 describes the `avformat` demux; Chapter 57 covers FFmpeg's internal API, codec threading, and hardware acceleration contexts that VLC can leverage via `--avcodec-hw=vaapi`.

- **Chapter 58 — GStreamer**: Architectural comparison of VLC's plugin pipeline versus GStreamer's pad-and-caps graph. Both use VA-API for decode, PipeWire for audio, and `zwp_linux_dmabuf_v1` for Wayland zero-copy. GStreamer's `vapostproc` element is analogous to VLC's `interop_vaapi.c` in libplacebo integration purpose.

- **Chapter 20 — Wayland**: Section 5 of this chapter uses `zwp_linux_dmabuf_v1`, `xdg_toplevel`, `wp_viewporter`, and `linux-drm-syncobj-v1` — all Wayland protocols detailed in Chapter 20.

- **Chapter 24 — Vulkan**: Section 6 uses `VK_EXT_external_memory_dma_buf`, `VK_EXT_image_drm_format_modifier`, `VK_KHR_wayland_surface`, and `VkSwapchainKHR` — covered in depth in Chapter 24.

- **Chapter 142 — V4L2 Media Subsystem**: VLC's V4L2 camera access module and the V4L2 M2M API that underlies embedded decode (via FFmpeg's V4L2 hwaccel on SoCs like Rockchip and AllWinner) are described in Chapter 142. The `VIDIOC_QBUF`/`DQBUF` cycle, DMA-BUF export, and stateless codec control sets are covered there. VLC's Raspberry Pi path uses MMAL (`modules/hw/mmal/`) rather than V4L2 directly.

- **Chapter 158 — HDR Display**: libplacebo's PQ→SDR tone mapping (Sections 6, 10) and the display stack requirements for HDR10 and HLG output are covered in Chapter 158.

- **Chapter 186 — NV12/P010 Pixel Formats**: The NV12 (8-bit) and P010 (10-bit) formats that flow between VA-API, EGL, Vulkan, and Wayland in this chapter are dissected in Chapter 186, including DRM format modifier semantics.

- **Chapter 140 — HDMI Audio Passthrough via ALSA/PipeWire**: Section 8 of this chapter introduces IEC 958 passthrough; Chapter 140 covers HDMI audio endpoint enumeration, ELD (EDID-Like Data) parsing, and compressed audio format negotiation in detail.

---

## References

- VideoLAN VLC source: [github.com/videolan/vlc](https://github.com/videolan/vlc)
- libVLC API source (`lib/`): [github.com/videolan/vlc/tree/master/lib](https://github.com/videolan/vlc/tree/master/lib)
- VLC plugin system (`include/vlc_plugin.h`): [github.com/videolan/vlc/blob/master/include/vlc_plugin.h](https://github.com/videolan/vlc/blob/master/include/vlc_plugin.h)
- VA-API helper modules (`modules/hw/vaapi/`): [github.com/videolan/vlc/tree/master/modules/hw/vaapi](https://github.com/videolan/vlc/tree/master/modules/hw/vaapi)
- VA-API FFmpeg bridge (`modules/codec/avcodec/vaapi.c`): [github.com/videolan/vlc/blob/master/modules/codec/avcodec/vaapi.c](https://github.com/videolan/vlc/blob/master/modules/codec/avcodec/vaapi.c)
- MMAL hardware decode (Raspberry Pi): [github.com/videolan/vlc/tree/master/modules/hw/mmal](https://github.com/videolan/vlc/tree/master/modules/hw/mmal)
- V4L2 capture access: [github.com/videolan/vlc/tree/master/modules/access/v4l2](https://github.com/videolan/vlc/tree/master/modules/access/v4l2)
- Wayland video output: [github.com/videolan/vlc/tree/master/modules/video_output/wayland](https://github.com/videolan/vlc/tree/master/modules/video_output/wayland)
- Vulkan video output: [github.com/videolan/vlc/tree/master/modules/video_output/vulkan](https://github.com/videolan/vlc/tree/master/modules/video_output/vulkan)
- OpenGL video output: [github.com/videolan/vlc/tree/master/modules/video_output/opengl](https://github.com/videolan/vlc/tree/master/modules/video_output/opengl)
- libplacebo (canonical): [code.videolan.org/videolan/libplacebo](https://code.videolan.org/videolan/libplacebo)
- libplacebo GitHub mirror: [github.com/haasn/libplacebo](https://github.com/haasn/libplacebo)
- libva (VA-API): [github.com/intel/libva](https://github.com/intel/libva)
- `zwp_linux_dmabuf_v1` protocol: [gitlab.freedesktop.org/wayland/wayland-protocols](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml)
- `linux-drm-syncobj-v1` protocol: [gitlab.freedesktop.org/wayland/wayland-protocols (staging)](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/linux-drm-syncobj/linux-drm-syncobj-v1.xml)
- `VK_EXT_image_drm_format_modifier`: [registry.khronos.org/vulkan/specs](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html)
- V4L2 M2M API: [kernel.org/doc/html/latest/userspace-api/media/v4l/dev-mem2mem.html](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/dev-mem2mem.html)
- VideoLAN SoC 2026: [wiki.videolan.org/SoC_2026](https://wiki.videolan.org/SoC_2026/)
- VLC nightly builds: [nightlies.videolan.org](https://nightlies.videolan.org/)

## Roadmap

### Near-term (6–12 months)
- VLC 4.0 stable release is approaching; the Qt6 UI, Vulkan renderer, and libplacebo HDR pipeline are targeted for the first stable 4.0 tag, with subtitle rendering in Vulkan shaders completing the feature set required for a production release.
- The `linux-drm-syncobj-v1` explicit synchronisation path in the Wayland output is being finalised; once compositor support (Mutter 48+, KWin 6.4+) is widespread, it will land as the default, eliminating `vaSyncSurface()` CPU stalls.
- libplacebo Dolby Vision Profile 5 (IPT-PQ) processing is an active VideoLAN SoC 2026 project; when complete, VLC will natively tone-map Dolby Vision streams on the Vulkan path without CPU preprocessing.
- NVIDIA VA-API decode quality is improving via `nvidia-vaapi-driver` upstream contributions; AV1 hardware decode on RTX 40-series via the VA-API bridge is expected to stabilise in the next VLC stable update cycle.

### Medium-term (1–3 years)
- The V4L2 stateless codec API adoption is broadening across ARM SoC vendors (MediaTek MT8195, Qualcomm SA8775P); VLC's FFmpeg `avcodec` V4L2 hwaccel path will gain zero-copy DMA-BUF export to Wayland on these platforms, matching the existing VA-API pipeline quality.
- VLC's `--sout` transcode chain is expected to gain a zero-copy VA-API encode path where both decoder and encoder share a `VADisplay`, eliminating the GPU→CPU→GPU round-trip for HEVC and AV1 encode on Intel Arc and AMD RDNA3 hardware.
- PipeWire integration will deepen beyond playback: VLC's screen capture (`vlc screen://`) is expected to fully adopt the XDG desktop portal screencast API with camera and microphone isolation, making it compliant with Flatpak sandbox constraints.
- The libadwaita/GTK4 UI prototype may mature into a shipping alternative to the Qt6 interface for GNOME-centric distributions, providing adaptive layout for tablet and portable use cases.

### Long-term
- As Vulkan Video (VK_KHR_video_decode_queue) matures and driver support broadens, VLC may add a native Vulkan decode path that keeps the entire pipeline — decode, tone mapping, scaling, composition — within a single Vulkan device, removing the VA-API inter-API hand-off entirely.
- WebAssembly / WASI threading improvements may make VLC.js a viable hardware-decode-capable player in browsers, with Vulkan via WebGPU as the renderer backend, erasing the current software-only limitation for browser-embedded VLC.
- Long-term convergence of VLC's Android, iOS, and desktop codebases around a shared libVLC core with platform-specific Vulkan/Metal/D3D12 renderers — a direction signalled by the modular renderer abstraction introduced in VLC 4.0 — could make VLC a true cross-platform Vulkan media framework.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
