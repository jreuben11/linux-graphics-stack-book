# Chapter 153: OBS Studio GPU Pipeline on Linux

**Target audiences**: Content creators and developers who need to understand how OBS Studio uses Linux GPU infrastructure for capture, composition, and encoding; systems engineers integrating OBS into streaming pipelines; and developers extending OBS with custom sources or encoders.

---

## Table of Contents

1. [Introduction](#introduction)
2. [OBS Architecture Overview](#obs-architecture-overview)
3. [GPU Capture: PipeWire and DMA-BUF Screen Capture](#gpu-capture-pipewire-and-dma-buf-screen-capture)
4. [OBS Scene Graph and GPU Composition](#obs-scene-graph-and-gpu-composition)
5. [Hardware Video Encoding: VA-API, NVENC, and AMF](#hardware-video-encoding-va-api-nvenc-and-amf)
6. [Multi-GPU Routing in OBS](#multi-gpu-routing-in-obs)
7. [Virtual Camera and Webcam Sources](#virtual-camera-and-webcam-sources)
8. [OBS Plugins and the libobs API](#obs-plugins-and-the-libobs-api)
9. [Performance Tuning and Diagnostics](#performance-tuning-and-diagnostics)
10. [Integrations](#integrations)

---

## Introduction

OBS Studio is the dominant open-source tool for video recording and live streaming. On Linux, OBS sits at the intersection of nearly every GPU subsystem covered in this book: it captures screen content via PipeWire and DMA-BUF, composes scenes with OpenGL or Vulkan, and encodes the output with VA-API (Intel/AMD), NVENC (NVIDIA), or AMF (AMD). When things go wrong — dropped frames, encoding lag, capture failure — the fault usually lies somewhere in this chain.

This chapter traces the complete Linux OBS GPU pipeline from screen capture to encoded output, explaining the kernel and Mesa infrastructure that each step relies on. It is also a case-study in how a complex application ties together the xdg-desktop-portal, PipeWire, DMA-BUF, EGLImage, and hardware video encoding in a single coherent pipeline.

[OBS Studio source](https://github.com/obsproject/obs-studio) | [obs-vkcapture](https://github.com/nowrep/obs-vkcapture)

---

## OBS Architecture Overview

### libobs: The Core Framework

OBS is built on **libobs**, a modular C framework that abstracts:

- **Sources** — video inputs (screen capture, webcam, game capture, browser)
- **Filters** — per-source GPU effects (colour correction, chroma key, denoising)
- **Transitions** — between scenes
- **Outputs** — encoded streams (RTMP, SRT, HLS) and recordings (MP4, MKV)
- **Encoders** — video/audio compression (software x264, hardware VA-API, NVENC)
- **Services** — streaming destinations (Twitch, YouTube, custom RTMP)

```
Screen         Game        Webcam
Capture       Capture      (V4L2)
   │             │            │
   ▼             ▼            ▼
[PipeWire]  [EGL/VK layer]  [V4L2 capture]
   │             │            │
   └─────────────┴────────────┘
                 │
         [libobs GPU Scene Graph]
           OpenGL / Vulkan
                 │
         [Video Encoder]
        VA-API / NVENC / AMF
                 │
         [Output Muxer]
          RTMP / SRT / MP4
```

### Rendering Backends

OBS supports two rendering backends on Linux:
- **OpenGL** (default, via `obs-opengl` plugin): uses EGL on Wayland, GLX on X11
- **Vulkan** (experimental via `obs-vulkan`): uses Vulkan swapchain directly

```bash
# Select backend:
obs --gpu-priority  # hint for multi-GPU
OBS_USE_EGL=1 obs  # force EGL (Wayland-native path)
```

---

## GPU Capture: PipeWire and DMA-BUF Screen Capture

### The Portal Path

On Wayland, OBS captures the screen via the xdg-desktop-portal's ScreenCast interface:

```
OBS → D-Bus → org.freedesktop.portal.ScreenCast
  → user ACL dialog (GNOME/KDE native)
  → portal opens PipeWire stream
  → OBS receives PipeWire DMA-BUF stream
```

```bash
# Debug portal interactions:
G_MESSAGES_DEBUG=all obs 2>&1 | grep portal
# Or force portal:
OBS_PORTAL_USE_MEMORY=1 obs  # memory-backed (no DMA-BUF, for compatibility)
```

### PipeWire DMA-BUF Capture

The `linux-capture` OBS plugin (`plugins/linux-capture/`) receives PipeWire frames as DMA-BUF file descriptors:

```c
/* Conceptual: obs-studio/plugins/linux-capture/pipewire.c */
static void on_process(void *userdata)
{
    struct pw_buffer *b = pw_stream_dequeue_buffer(stream);
    struct spa_data *d = &b->buffer->datas[0];

    if (d->type == SPA_DATA_DmaBuf) {
        /* Import DMA-BUF into OBS texture: */
        gs_texture_t *tex = gs_texture_create_from_dmabuf(
            d->fd,
            d->chunk->size,     /* stride × height */
            frame_width, frame_height,
            DRM_FORMAT_ARGB8888);
        /* tex is now a GPU texture backed by the screen DMA-BUF */
        obs_source_output_video(source, &frame);
    }
    pw_stream_queue_buffer(stream, b);
}
```

### gs_texture_create_from_dmabuf

`gs_texture_create_from_dmabuf` (in `libobs-opengl/`) performs:

```c
/* OpenGL path: */
EGLImage image = eglCreateImageKHR(display, EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT, NULL, attribs); /* attribs include FD, format */

GLuint tex;
glGenTextures(1, &tex);
glBindTexture(GL_TEXTURE_2D, tex);
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, image);
/* tex is now the screen content as a GPU texture — zero copy */
```

This is the zero-copy path: the screen content DMA-BUF goes directly to a GPU texture without any CPU involvement.

### obs-vkcapture: Vulkan Game Capture

For capturing Vulkan games without going through the desktop compositor, [obs-vkcapture](https://github.com/nowrep/obs-vkcapture) injects a Vulkan layer into the game:

```bash
# Install and use:
# pacman -S obs-vkcapture
obs-gamecapture gamename  # wraps game with VK_LAYER_OBS_vkcapture
```

The layer intercepts `vkQueuePresentKHR`, captures the swapchain image as a DMA-BUF, and sends it to OBS via a shared memory / DMA-BUF socket. This bypasses the compositor entirely and works even in full-screen exclusive mode.

```c
/* obs-vkcapture layer (simplified): */
VkResult vkQueuePresentKHR(VkQueue queue,
    const VkPresentInfoKHR *pPresentInfo)
{
    /* Copy swapchain image to capture buffer */
    capture_frame_to_dmabuf(queue, pPresentInfo->pSwapchains[0]);
    /* Then call the real present */
    return real_vkQueuePresentKHR(queue, pPresentInfo);
}
```

### X11 Capture (Legacy)

On X11, OBS uses the `xshm` source (X11 shared memory via MIT-SHM) or the `xcomposite` source (compositor redirected window). Both are CPU-intensive copy operations. The PipeWire/DMA-BUF path is strongly preferred on Wayland for performance.

---

## OBS Scene Graph and GPU Composition

### Scene and Source Rendering

OBS's rendering model:

```
Scene
 ├─ Source A (screen capture) ──→ GPU Texture
 ├─ Source B (webcam)         ──→ GPU Texture
 ├─ Source C (browser)        ──→ GPU Texture
 └─ Filter (colour correct)   ──→ GPU pass
          │
     Render to FBO (output resolution, e.g. 1920×1080)
          │
     Present to preview + encoder input
```

Each source is a GPU texture; the scene graph composites them using OpenGL (with `glBlendFunc`, texture sampling, transform matrices).

### Effect Shaders

OBS uses its own HLSL-like shader language (`.effect` files) compiled to GLSL/HLSL at runtime:

```hlsl
/* Example: obs-studio/data/effects/default.effect */
uniform texture2d image;
uniform float4x4 ViewProj;

struct VertData {
    float4 pos : POSITION;
    float2 uv  : TEXCOORD0;
};

VertData VSDefault(VertData v_in) {
    VertData vert_out;
    vert_out.pos = mul(float4(v_in.pos.xyz, 1.0), ViewProj);
    vert_out.uv  = v_in.uv;
    return vert_out;
}

float4 PSDefault(VertData v_in) : TARGET {
    return image.Sample(def_sampler, v_in.uv);
}
```

These are compiled via `gs_effect_create()` → Mesa GLSL compiler → GPU shader.

### Chroma Key Filter

The chroma key filter is a GPU effect shader:

```glsl
/* Simplified chroma key (green screen removal): */
uniform vec4 key_color;      /* e.g. vec4(0, 1, 0, 1) */
uniform float similarity;
uniform float smoothness;

vec4 ps_chroma_key(float2 uv) {
    vec4 color = texture(image, uv);
    float chromaDist = distance(color.rgb, key_color.rgb);
    float mask = smoothstep(similarity,
                            similarity + smoothness,
                            chromaDist);
    return vec4(color.rgb, color.a * mask);
}
```

Runs entirely on GPU per-frame with no CPU involvement.

### Frame Timing and Tick Rate

OBS uses a fixed tick rate matching the output frame rate. The rendering thread calls `obs_render_main_texture()` at, e.g., 1/60 s intervals. GPU composition must complete before the encoder's input deadline:

```c
/* obs-studio/libobs/obs-video.c */
static void render_video(struct obs_core_video_mix *video, ...)
{
    gs_begin_scene();
    obs_render_main_texture();  /* GPU scene composition */
    gs_end_scene();
    /* Signal encoder thread that frame is ready */
}
```

---

## Hardware Video Encoding: VA-API, NVENC, and AMF

### VA-API Encoding (Intel and AMD)

OBS uses VA-API for H.264, H.265, and AV1 encoding on Intel and AMD:

```bash
# Enable VA-API encoder in OBS:
# Settings → Output → Encoder → "VA-API H.264" or "VA-API HEVC"

# Debug VA-API:
LIBVA_MESSAGING_LEVEL=1 obs 2>&1 | grep -i vaapi
# List VA-API devices:
vainfo --display drm --device /dev/dri/renderD128
```

The OBS VA-API plugin (`plugins/obs-ffmpeg/vaapi.c`) uses FFmpeg's AVCodecContext with VAAPI hardware acceleration:

```c
av_hwdevice_ctx_create(&hw_device_ctx,
    AV_HWDEVICE_TYPE_VAAPI, "/dev/dri/renderD128", NULL, 0);

avcodec_find_encoder_by_name("h264_vaapi");
/* Configure: rc_mode, bitrate, qp, profile */
```

The critical path for zero-copy GPU→encoder:

```
OBS OpenGL FBO → EGL export as DMA-BUF → VA-API surface import → encoded
```

OBS achieves this via `obs_encoder_get_extra_data` and FFmpeg's DRM/VAAPI interop:

```c
/* Transfer GPU frame to VA-API surface via DRM Prime: */
av_hwframe_transfer_data(vaapi_frame, drm_frame, 0);
/* drm_frame carries the DMA-BUF FD from OBS's OpenGL render */
```

### NVENC Encoding (NVIDIA)

On NVIDIA (proprietary driver), OBS uses NVENC directly:

```bash
# OBS Settings → Output → Encoder → "NVENC H.264" or "NVENC HEVC"
# Requires: nvidia driver ≥ 520; nvidia-open ≥ 525 also works

# Debug:
NV_DRIVER_LOG_LEVEL=5 obs 2>&1 | grep nvenc
```

The NVENC path:
1. OBS renders to OpenGL FBO
2. Registers OpenGL texture with NVENC via `NvEncRegisterResource`/`NvEncMapInputResource`
3. NVENC GPU copies texture to encoder input surface
4. NVENC encodes → bitstream

There is no DMA-BUF path for NVENC; NVIDIA uses its own CUDA-based interop (`cuGraphicsGLRegisterImage`). A DMA-BUF zero-copy path exists only with `EGL_NV_stream_consumer_eglimage` and is not yet in mainstream OBS.

### AMF Encoding (AMD)

AMD's AMF (Advanced Media Framework) is available on AMD GPUs on Linux (since AMF 1.4.26, 2022):

```bash
# Requires: amdgpu driver + Vulkan support
# OBS: Settings → Output → Encoder → "AMD HW H.264 (AMF)"

# Check AMF support:
ls /dev/dri/card* && vulkaninfo | grep AMD
```

AMF on Linux uses Vulkan interop: OBS renders to a Vulkan image, AMF imports it via `AMFSurface::InteropInitFrom(VkImage)`, and encodes. This is a zero-copy Vulkan path.

### Comparing Encoder Paths

| Encoder | GPU Support | Zero-Copy | DMA-BUF |
|---|---|---|---|
| VA-API H.264/HEVC/AV1 | Intel, AMD | Yes (via DRM Prime) | Yes |
| NVENC | NVIDIA | Partial (CUDA GL interop) | No |
| AMF | AMD | Yes (Vulkan interop) | Indirect |
| x264/x265 (SW) | Any | No (CPU copy) | No |

For AMD GPUs, VA-API and AMF both work; AMF often has lower latency (dedicated encoder queue on RDNA).

---

## Multi-GPU Routing in OBS

### The Problem

A common Linux configuration: Intel iGPU drives the display (Wayland compositor runs here), NVIDIA dGPU is for performance (games, rendering). OBS must:
1. Capture from the iGPU (compositor's DMA-BUF)
2. Potentially composite on dGPU
3. Encode on whichever has the best encoder

### DRI_PRIME for OBS

```bash
# Force OBS to use a specific GPU:
DRI_PRIME=1 obs          # use the secondary GPU (e.g. NVIDIA via PRIME)
MESA_VK_DEVICE_SELECT=vendor:0x1002 obs  # force AMD Vulkan device

# Or for NVIDIA proprietary:
__NV_PRIME_RENDER_OFFLOAD=1 __VK_LAYER_NV_optimus=NVIDIA_only \
    __GLX_VENDOR_LIBRARY_NAME=nvidia obs
```

### Cross-GPU DMA-BUF

When OBS captures a DMA-BUF from the iGPU compositor and imports it into the dGPU for composition:
- If GPUs share a DMA-BUF importer (both support `EGL_EXT_image_dma_buf_import`)
- Mesa handles the cross-device import; a PCIe transfer occurs transparently
- Performance cost: ~1 ms for 1080p frame transfer via PCIe

The alternative is CPU-side copy (`glReadPixels` → system RAM → upload to second GPU) which costs 5–15 ms and is generally avoided.

### obs-vkcapture Multi-GPU

`obs-vkcapture` game capture sends the game's Vulkan swapchain image via DMA-BUF to OBS running on a different GPU:

```
NVIDIA game → DMA-BUF (NVIDIA→CPU→AMD or direct P2P) → OBS (AMD)
```

RDNA3 and NVIDIA Ampere+ support peer-to-peer DMA without CPU involvement on systems with PCIe P2P.

---

## Virtual Camera and Webcam Sources

### v4l2loopback Virtual Camera

OBS's virtual camera output uses `v4l2loopback` to create a virtual V4L2 device:

```bash
# Install v4l2loopback:
# modprobe v4l2loopback devices=1 video_nr=10 card_label="OBS Virtual Camera"

# Start OBS virtual camera:
# In OBS: Tools → Start Virtual Camera

# Use in another app:
mpv /dev/video10
ffmpeg -f v4l2 -i /dev/video10 output.mp4
```

OBS writes CPU-side YUV frames to the v4l2loopback device. The path is:
```
OBS OpenGL FBO → glReadPixels (CPU) → YUV convert → write to /dev/video10
```

This involves a GPU→CPU→kernel copy. A DMA-BUF path for virtual cameras is under development in v4l2loopback (camera DMA-BUF passthrough).

### V4L2 Webcam Source

Physical webcams (USB, integrated) appear as V4L2 devices. OBS reads them via `VIDIOC_DQBUF`:

```c
/* plugins/linux-v4l2/v4l2-input.c */
struct v4l2_buffer buf = {0};
buf.type   = V4L2_BUF_TYPE_VIDEO_CAPTURE;
buf.memory = V4L2_MEMORY_MMAP;
ioctl(fd, VIDIOC_DQBUF, &buf);
/* buf.m.offset → mmap'd frame data → upload to GPU texture */
```

Modern webcams with MJPEG or H.264 output can be hardware-decoded via VA-API before upload.

### Browser Source via CEF

OBS's browser source embeds Chromium (via CEF — Chromium Embedded Framework). CEF renders web pages with GPU acceleration (Dawn/WebGPU or ANGLE/WebGL). On Linux:
- CEF renders to an offscreen framebuffer
- Shares it with OBS via shared memory or DMA-BUF
- OBS composites it as a GPU texture

```bash
# Debug CEF in OBS:
obs --enable-gpu-debug
# CEF log at: ~/.config/obs-studio/logs/
```

---

## OBS Plugins and the libobs API

### Writing a Source Plugin

```c
/* Minimal OBS source plugin for a custom GPU texture: */
#include <obs-module.h>

static obs_properties_t *my_source_properties(void *data)
{
    obs_properties_t *props = obs_properties_create();
    obs_properties_add_int(props, "width", "Width", 320, 7680, 1);
    obs_properties_add_int(props, "height", "Height", 240, 4320, 1);
    return props;
}

static void my_source_render(void *data, gs_effect_t *effect)
{
    struct my_source *s = data;
    /* s->texture was created from a DMA-BUF or rendered offscreen */
    gs_effect_set_texture(gs_effect_get_param_by_name(effect, "image"),
                          s->texture);
    gs_draw_sprite(s->texture, 0, s->width, s->height);
}

static struct obs_source_info my_source_info = {
    .id            = "my_custom_source",
    .type          = OBS_SOURCE_TYPE_INPUT,
    .output_flags  = OBS_SOURCE_VIDEO,
    .get_name      = my_source_get_name,
    .create        = my_source_create,
    .destroy       = my_source_destroy,
    .video_render  = my_source_render,
    .get_properties = my_source_properties,
};

OBS_DECLARE_MODULE()
bool obs_module_load(void) {
    obs_register_source(&my_source_info);
    return true;
}
```

### Writing a VA-API Encoder Plugin

Custom hardware encoder plugins use `obs_encoder_info`:

```c
static bool vaapi_encode(void *data,
    struct encoder_frame *frame, struct encoder_packet *packet,
    bool *received_packet)
{
    /* frame->data[0] = GPU texture handle or DMA-BUF FD */
    /* Submit to VA-API, get back bitstream in packet->data */
    return true;
}

static struct obs_encoder_info vaapi_encoder = {
    .id          = "my_vaapi_h264",
    .type        = OBS_ENCODER_VIDEO,
    .codec       = "h264",
    .encode      = vaapi_encode,
    .create      = vaapi_create,
    .destroy     = vaapi_destroy,
};
```

### libobs GPU Context

OBS manages a single shared OpenGL context. All GPU operations (source render, effect apply, encoder input copy) run in the video render thread. Plugins must not create their own GL contexts:

```c
/* Enter the OBS graphics context from a plugin thread: */
obs_enter_graphics();
gs_texture_t *tex = gs_texture_create(w, h, GS_RGBA, 1, NULL, GS_DYNAMIC);
obs_leave_graphics();
```

---

## Performance Tuning and Diagnostics

### Frame Timing Analysis

```bash
# OBS Stats dock: View → Stats
# Key metrics:
# - Render lag: time to compose the scene (should be < frame_time/2)
# - Frame time: total time per output frame
# - Skipped frames due to encoding lag: encoder is the bottleneck
# - Dropped frames due to network: output/stream is the bottleneck

# Enable verbose logging:
obs --verbose 2>&1 | grep -i "frame\|encode\|drop"
```

### VA-API Performance

```bash
# Monitor VA-API encoder:
vainfo --display drm --device /dev/dri/renderD128

# Check GPU utilisation during OBS:
intel_gpu_top  # Intel
radeontop      # AMD
nvidia-smi dmon -s u  # NVIDIA

# Confirm hardware encoder is active (not software fallback):
obs 2>&1 | grep -i "vaapi\|nvenc\|amf\|encoder"
```

### Common Performance Issues

**Problem: High CPU usage during screen capture**
- Cause: Using `xshm` on X11 or memory-backed PipeWire (not DMA-BUF)
- Fix: Use Wayland + portal for DMA-BUF capture; set `OBS_PORTAL_USE_MEMORY=0`

**Problem: Encoding lag on AMD with VA-API**
- Cause: Mesa VA-API driver overhead; or using wrong render device
- Fix: `LIBVA_DRIVER_NAME=radeonsi vainfo`; try AMF if RDNA2+; check `DRI_PRIME`

**Problem: Black screen on Wayland capture**
- Cause: Portal permission not granted, or compositor doesn't support ScreenCast
- Fix: `xdg-desktop-portal` v1.16+; check `systemctl --user status xdg-desktop-portal`

**Problem: obs-vkcapture not capturing game**
- Cause: Vulkan layer not loaded, or game uses Proton's VK layer stack
- Fix: `VK_LAYER_PATH=/usr/share/vulkan/implicit_layer.d obs-gamecapture game`; check `ENABLE_VKBASALT=0`

### Checking the Complete Pipeline

```bash
# Full diagnostic:
echo "=== DRM devices ===" && ls /dev/dri/
echo "=== VA-API ===" && vainfo 2>&1 | head -20
echo "=== PipeWire ===" && pw-cli list-objects | grep -c "PipeWire"
echo "=== Portals ===" && systemctl --user status xdg-desktop-portal
echo "=== GPU ===" && vulkaninfo --summary 2>/dev/null | head -20
```

---

## Integrations

- **Ch38 (PipeWire)** — PipeWire is the screen capture transport; OBS receives frames as a PipeWire stream with DMA-BUF buffer type
- **Ch123 (Screen Capture)** — `org.freedesktop.portal.ScreenCast` is the portal used by OBS; `zwp_linux_dmabuf_v1` is the Wayland protocol for zero-copy frame delivery
- **Ch26 (Hardware Video)** — VA-API encoding in OBS uses the same VAAPI driver and `libva` as hardware video decode; AMF uses Mesa Vulkan
- **Ch142 (EGL and DMA-BUF)** — `gs_texture_create_from_dmabuf` uses `eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)` to import PipeWire DMA-BUF frames as GPU textures
- **Ch49 (Multi-GPU PRIME)** — `DRI_PRIME=1` routing for OBS to use the performance GPU for encoding while the iGPU handles display
- **Ch55 (GPU Containers)** — OBS in containers requires `/dev/dri/renderD128` passthrough and portal socket access
- **Ch80 (GPU Security)** — The portal ScreenCast ACL is the security boundary for screen capture; without it any application could capture the screen
- **Ch135 (Vulkan Ray Tracing)** — obs-vkcapture Vulkan layer intercepts the game's swapchain at `vkQueuePresentKHR`, the same hook point used by RenderDoc (Ch125)
