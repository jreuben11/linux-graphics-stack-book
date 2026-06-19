# Chapter 118: Screen Capture and Remote Desktop on Linux

**Target audiences**: Application developers who need to capture or share display output; Wayland compositor authors implementing capture protocols; OBS and streaming engineers; remote desktop and WebRTC infrastructure engineers.

---

## Table of Contents

1. [Introduction: The Isolation Problem](#1-introduction-the-isolation-problem)
2. [X11 Screen Capture (Legacy)](#2-x11-screen-capture-legacy)
3. [KMS Writeback Connectors](#3-kms-writeback-connectors)
4. [wlr-screencopy Protocol](#4-wlr-screencopy-protocol)
5. [ext-image-copy-capture Protocol](#5-ext-image-copy-capture-protocol)
6. [PipeWire Screencast](#6-pipewire-screencast)
7. [xdg-desktop-portal ScreenCast](#7-xdg-desktop-portal-screencast)
8. [OBS Studio on Wayland](#8-obs-studio-on-wayland)
9. [WebRTC Screen Sharing](#9-webrtc-screen-sharing)
10. [Remote Desktop: RDP, VNC, and Game Streaming](#10-remote-desktop-rdp-vnc-and-game-streaming)
11. [Integrations](#11-integrations)

---

## 1. Introduction: The Isolation Problem

Screen capture is a deceptively deep problem on Linux. For nearly three decades under X11 the answer was trivial: any client could call `XGetImage` on any window or the root window and receive pixel data immediately, with no permission check and no user consent. This worked because X11's security model trusts all local clients equally — a design decision that made sense in the era of shared terminal servers but is entirely unsuitable for modern desktops where untrusted applications run alongside password managers, banking software, and private communications.

Wayland made a different choice. Every Wayland compositor enforces strict client isolation: a client can only access its own surfaces. There is no analogue to `XGetImage` in the core Wayland protocol. If you want to capture the screen you must go through the compositor, and compositors are free — and encouraged — to prompt the user for permission before granting access.

This architectural change forced a complete reimagining of how screen capture works. The modern stack has several layers:

- **Kernel layer**: KMS writeback connectors allow the display controller hardware to write composed frames to a GEM buffer object, bypassing the 3D engine entirely.
- **Compositor protocols**: `wlr-screencopy-v1` (wlroots family) and the now-standard `ext-image-copy-capture-v1` (merged into wayland-protocols staging) define how clients request frame data from the compositor.
- **PipeWire**: acts as the universal video bus, brokering screen capture streams between the compositor (producer) and applications (consumers) using negotiated formats including zero-copy DMA-BUF.
- **xdg-desktop-portal**: provides a D-Bus permission gate so that sandboxed applications (Flatpak, Snap, browsers) can request a screencasting session subject to user consent.
- **Applications**: OBS Studio, WebRTC browsers, and remote desktop servers sit at the top, consuming PipeWire streams or sending them over the network.

This chapter traces the complete path from kernel write-back all the way to network-transmitted H.264, covering each layer's API, implementation status, and performance characteristics.

---

## 2. X11 Screen Capture (Legacy)

Despite being a legacy path, X11 screen capture is still relevant because XWayland provides an X11 translation layer for legacy applications, and many existing tools have not yet been ported.

### XGetImage and XShmGetImage

The fundamental X11 capture call is `XGetImage`:

```c
XImage *XGetImage(Display *display, Drawable drawable,
                  int x, int y,
                  unsigned int width, unsigned int height,
                  unsigned long plane_mask,
                  int format);   /* XYPixmap or ZPixmap */
```

This requests that the X server copy pixel data from any `Drawable` — including the root window (`DefaultRootWindow(display)`) — into client memory. The call crosses the Unix socket, copies data from the X server's framebuffer, and delivers it to the client. At 4K resolution (3840×2160 × 4 bytes) a single frame transfer is roughly 31 MB, making this unsuitable for video capture at 60 fps.

`XShmGetImage` from the MIT-SHM extension eliminates the socket copy by sharing a POSIX shared memory segment between the X server and client. The server writes directly into the shared segment and the client reads from it in place. This was the dominant approach for X11 screen recorders, video capture tools, and remote desktop servers for two decades.

```c
/* Allocate shared image */
XShmSegmentInfo shminfo;
XImage *img = XShmCreateImage(dpy, DefaultVisual(dpy, screen),
                               DefaultDepth(dpy, screen), ZPixmap,
                               NULL, &shminfo, width, height);
shminfo.shmid = shmget(IPC_PRIVATE,
                        img->bytes_per_line * img->height,
                        IPC_CREAT | 0777);
shminfo.shmaddr = img->data = shmat(shminfo.shmid, NULL, 0);
shminfo.readOnly = False;
XShmAttach(dpy, &shminfo);

/* Capture */
XShmGetImage(dpy, DefaultRootWindow(dpy), img, 0, 0, AllPlanes);
/* img->data now contains BGRA pixels */
```

### XComposite and Damage-Tracked Capture

When a compositing window manager is running, windows are redirected off-screen. The `XComposite` extension's `XCompositeNameWindowPixmap` function returns a reference to a window's backing pixmap, enabling tools like screenshot utilities to capture individual windows as they appear on-screen rather than clipping the root window. This is the mechanism that tools like GNOME's screenshot tool used on X11.

The XComposite extension's off-screen redirection also introduces a subtle race condition for continuous capture. A capture tool that calls `XGetImage` on the root window captures the compositor's output buffer — the fully composited scene including all window stacking, opacity, and effects. This is correct for "what the user sees" but if the compositor is mid-composite, the root window may contain a partially updated frame. `XShmGetImage` has the same race because it is not synchronized to the compositor's render cycle.

More sophisticated tools avoided this race by using `XCompositeRedirectSubwindows(display, root, CompositeRedirectAutomatic)` together with damage notifications via the `XDamage` extension. When `XDamageNotify` arrives, the tool knows the compositor has updated the root pixmap and can safely capture. This `XDamage` + `XShmGetImage` pattern was used by VNC servers (x11vnc), remote desktop clients, and OBS's `xshm` source.

### CLI Capture Tools

`xwd` (X Window Dump) and ImageMagick's `import` command use `XGetImage`. `scrot` uses `XShmGetImage` for speed. On X11, `ffmpeg -f x11grab` uses XShm for screen recording:

```bash
ffmpeg -f x11grab -r 30 -s 1920x1080 -i :0.0+0,0 -c:v libx264 output.mp4
```

OBS Studio's `xshm` source type wraps the same `XShmGetImage` interface, polling for new frames at the configured capture framerate.

### The Security Problem

Any X11 client — whether it arrived legitimately or was injected as malware — can capture any window's pixels, log keystrokes via `XGrabKeyboard` or `XQueryKeymap`, and synthesize arbitrary input events via `XSendEvent`. There is no prompt, no permission, no audit trail. This is the primary security motivation for Wayland's isolation model and the elaborate portal machinery described in later sections.

The attack surface is not theoretical. A malicious X11 application installed in a user's session can capture passwords typed into other applications, take screenshots of banking sessions, and inject keystrokes. Because X11 grants equal trust to all local connections, there is no runtime distinction between the system's login screen and a rogue application. [Source](https://wayland.freedesktop.org/faq.html)

### The XWayland Bridge

Under Wayland with XWayland, X11 screen capture tools see a limited subset of the full desktop. XWayland provides each X11 application its own X11 screen backed by a Wayland sub-surface; `XGetImage` on the XWayland root captures only the X11 composited image, not the Wayland desktop behind it. Legacy capture tools like `scrot` running inside XWayland see only the XWayland virtual screen — a contained subsection of the display. For full-desktop capture under Wayland, the portal path (Section 7) is mandatory.

---

## 3. KMS Writeback Connectors

Before touching compositor-level protocols, it is worth understanding the hardware path that makes efficient screen capture possible: the KMS writeback connector.

### Concept

A writeback connector is a DRM connector of type `DRM_MODE_CONNECTOR_WRITEBACK` that, instead of driving a physical panel, writes the composed display output to a GEM buffer object. The display controller's hardware blender — the same engine that layers cursor planes, overlay planes, and primary planes together for scan-out — also writes its output to a memory buffer accessible to the CPU and GPU. This enables hardware-accelerated capture of the fully composed framebuffer without any shader passes, CPU copies, or compositor involvement beyond configuring the connector. [Source](https://www.kernel.org/doc/html/latest/gpu/drm-kms-helpers.html)

### Kernel Structures

The core data types are defined in `include/drm/drm_writeback.h` in the kernel tree:

```c
struct drm_writeback_connector {
    struct drm_connector base;
    struct drm_encoder encoder;
    struct drm_property_blob *pixel_formats_blob_ptr;
    spinlock_t job_lock;
    struct list_head job_queue;
    unsigned int fence_context;
    spinlock_t fence_lock;
    unsigned long fence_seqno;
    char timeline_name[32];
};

struct drm_writeback_job {
    struct drm_writeback_connector *connector;
    bool prepared;
    struct work_struct cleanup_work;
    struct list_head list_entry;
    struct drm_framebuffer *fb;    /* output framebuffer */
    struct dma_fence *out_fence;   /* signals when write is done */
    void *priv;
};
```

[Source](https://raw.githubusercontent.com/torvalds/linux/master/include/drm/drm_writeback.h)

Drivers initialise a writeback connector with:

```c
int drm_writeback_connector_init(struct drm_device *dev,
    struct drm_writeback_connector *wb_connector,
    const struct drm_connector_funcs *con_funcs,
    const struct drm_encoder_helper_funcs *enc_helper_funcs,
    const u32 *formats, int n_formats,
    u32 possible_crtcs);
```

The `formats` array lists the pixel formats the writeback engine supports (e.g., `DRM_FORMAT_XRGB8888`, `DRM_FORMAT_NV12`). These are exposed to userspace as the `WRITEBACK_PIXEL_FORMATS` blob property. A managed allocator variant `drmm_writeback_connector_init` uses devres to handle lifetime automatically.

### Atomic Commit Integration

Writeback is driven through the standard atomic modesetting path. Userspace sets the `WRITEBACK_FB_ID` connector property to the GEM-backed framebuffer it wants the hardware to write into, and the `WRITEBACK_OUT_FENCE_PTR` property to an address where the kernel will store a sync file fd. The atomic commit then includes the writeback connector in the plane/CRTC reconfiguration. The helper `drm_atomic_helper_commit_writebacks()` iterates connectors in the new atomic state and calls the driver's encoder `atomic_commit` hook for each pending writeback job. Drivers signal completion via `drm_writeback_signal_completion()`, which resolves the out-fence. [Source](https://www.kernel.org/doc/html/latest/gpu/drm-kms-helpers.html)

### Hardware Support

Writeback connector support in mainline Linux is limited to hardware that physically implements the feature in its display controller. The authoritative list comes from drivers that call `drm_writeback_connector_init` or `drmm_writeback_connector_init` in the kernel tree (as of Linux 6.10):

- **ARM Mali Display Processor** (`drivers/gpu/drm/arm/malidp_mw.c`, DP500/DP550/DP650): one of the first writeback implementations, merged for Linux 4.19.
- **Arm Komeda display processor** (`drivers/gpu/drm/arm/display/komeda/komeda_wb_connector.c`): the successor to Mali-DP, used on the D71 and D32 display processors. The Komeda driver's documentation explicitly describes the writeback layer (`wb5`) that writes composition output to memory. [Source](https://docs.kernel.org/gpu/komeda-kms.html)
- **Broadcom VideoCore IV** (`drivers/gpu/drm/vc4/vc4_txp.c`): the TXP (Transposer) block on Raspberry Pi 4 and CM4 implements writeback; it is exposed as a writeback connector in the `vc4` DRM driver.
- **AMD AMDGPU Display Manager** (`drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_wb.c`): AMD added writeback connector support to amdgpu-dm for supported APU/GPU display engines.
- **Renesas R-Car** (`drivers/gpu/drm/renesas/rcar-du/rcar_du_writeback.c`): the R-Car display unit exposes a writeback path on select SoCs.
- **Qualcomm Snapdragon DPU** (`drivers/gpu/drm/msm/disp/dpu1/dpu_writeback.c`): the Display Processing Unit 1 in modern Snapdragon SoCs supports writeback.
- **VKMS** (`drivers/gpu/drm/vkms/vkms_writeback.c`): the virtual KMS driver implements writeback for testing and CI use — it writes to memory rather than a display.

Desktop-class Intel GPUs do not expose KMS writeback connectors. On Intel platforms (and any platform not listed above), compositors must use CPU-side readbacks or GPU render passes for capture.

### Use Case: Compositor Capture

A compositor that knows the hardware supports writeback can attach a writeback connector to its KMS CRTC and receive composed frames in a GEM buffer as part of its normal atomic commit — zero CPU cycles for the pixel transfer, and completed within the display controller's output timing. The compositor then imports the GEM buffer as a DMA-BUF and passes it directly into a PipeWire source node. This is the lowest-latency, most power-efficient capture path available on supported hardware.

---

## 4. wlr-screencopy Protocol

Before an upstream standard existed, the wlroots project defined its own screen capture protocol: `zwlr_screencopy_manager_v1`. This became the de-facto standard for the wlroots family of compositors (Sway, Hyprland, labwc, Wayfire, River) and spawned a rich ecosystem of capture tools. [Source](https://gitlab.freedesktop.org/wlroots/wlr-protocols)

### Protocol Objects

**`zwlr_screencopy_manager_v1`** (interface version 3) is a factory. Its two key requests are:

```text
capture_output(frame, overlay_cursor, output)
capture_output_region(frame, overlay_cursor, output, x, y, width, height)
```

Both produce a **`zwlr_screencopy_frame_v1`** object representing one frame to be captured from a `wl_output`.

**`zwlr_screencopy_frame_v1`** drives the copy lifecycle:

- The compositor sends a `buffer` event that specifies the required wl_shm format, width, height, and stride that the client must use.
- From version 3, the compositor also sends `linux_dmabuf` events listing DMA-BUF formats and modifiers, followed by `buffer_done` to indicate all format options have been enumerated.
- The client allocates a conforming `wl_buffer` (either a wl_shm buffer or a dmabuf-backed buffer).
- The client calls `copy(buffer)` to begin the transfer. The compositor captures the next rendered frame into that buffer.
- The compositor sends `flags` (indicating `y_invert` if needed), optionally `damage` (the changed rectangle), and finally `ready(tv_sec_hi, tv_sec_lo, tv_nsec)` with the frame timestamp.
- If capture fails, the compositor sends `failed`.

Version 2 added `copy_with_damage`: the copy only proceeds once the compositor detects that the output has changed (damage), enabling power-efficient polling. Version 3 added DMA-BUF buffer negotiation so that capture can proceed without CPU-visible copies on hardware that supports it.

### Tools in the Ecosystem

**grim** ([https://sr.ht/~emersion/grim/](https://sr.ht/~emersion/grim/)) is the reference screenshot tool. It opens `zwlr_screencopy_manager_v1`, creates a frame, allocates a wl_shm buffer, copies the frame, and writes a PNG. It composes with `slurp` for interactive region selection:

```bash
grim -g "$(slurp)" ~/screenshot.png
```

**wf-recorder** ([https://github.com/ammen99/wf-recorder](https://github.com/ammen99/wf-recorder)) uses wlr-screencopy for frame acquisition and FFmpeg for encoding. It only requests new frames when damage arrives (`copy_with_damage`), producing a variable-framerate output that FFmpeg timestamps accurately.

**wl-screenrec** ([https://github.com/russelltg/wl-screenrec](https://github.com/russelltg/wl-screenrec)) is a Rust-native alternative that takes the DMA-BUF path: it allocates GBM-backed buffers, presents them as wl_buffer objects via the linux-dmabuf protocol, and then passes the captured DMA-BUF directly to the VA-API encoder — achieving zero CPU copies end to end.

### Deprecation Status

`wlr-screencopy` is formally deprecated in favour of `ext-image-copy-capture-v1` (Section 5), but continues to be maintained in `wlr-protocols` for backward compatibility. It remains the only capture protocol supported by `xdg-desktop-portal-wlr`, so it underpins PipeWire screencasting on wlroots compositors in production today.

---

## 5. ext-image-copy-capture Protocol

After nearly three years of development and iteration, the Wayland community merged two complementary protocols into `wayland-protocols` (staging status): `ext-image-capture-source-v1` and `ext-image-copy-capture-v1`. These supersede `wlr-screencopy` with a cleaner session-based design, proper cursor capture, and broad compositor adoption. [Source](https://wayland.app/protocols/ext-image-copy-capture-v1)

### Separation of Source and Copy

The design separates *what to capture* from *how to capture it*:

**`ext-image-capture-source-v1`** is an opaque handle representing a capturable image source — a `wl_output`, a toplevel window handle, or any other resource a compositor wants to expose. It is created by source-specific managers:

- `ext_output_image_capture_source_manager_v1::create_source(output)` — captures an output.
- `ext_foreign_toplevel_image_capture_source_manager_v1::create_source(toplevel_handle)` — captures an application window, referenced via `ext_foreign_toplevel_handle_v1`.

[Source](https://wayland.app/protocols/ext-image-capture-source-v1)

The source object is then passed to the copy machinery.

### Copy Manager and Sessions

**`ext_image_copy_capture_manager_v1`** has two factory requests:

```text
create_session(session, source, options)
create_pointer_cursor_session(cursor_session, source, pointer)
```

The `options` bitfield currently defines `paint_cursors` (value 1), which asks the compositor to paint the software cursor into captured frames.

**`ext_image_copy_capture_session_v1`** represents an ongoing capture relationship. The compositor advertises buffer constraints through events before any frames are captured:

- `buffer_size(width, height)` — pixel dimensions required.
- `shm_format(drm_format)` — a supported shared-memory format (repeating).
- `dmabuf_device(dev)` — the DRM device node for DMA-BUF allocation.
- `dmabuf_format(format, modifier)` — a supported DMA-BUF format/modifier pair (repeating).
- `done` — all constraints have been sent; client may now allocate buffers.
- `stopped` — the source is gone; destroy the session.

This constraint-advertisement model is a key improvement over wlr-screencopy: the client knows the exact buffer requirements before it allocates memory, avoiding wasted allocations if the compositor changes resolution or format.

### Frame Lifecycle

The client calls `create_frame()` on the session to get a **`ext_image_copy_capture_frame_v1`**. The protocol enforces one live frame at a time (`duplicate_frame` error if violated). The frame lifecycle is:

```text
attach_buffer(buffer)     -- attach a wl_buffer meeting constraints
damage_buffer(x, y, w, h) -- optionally declare which region needs capture
capture()                 -- begin the capture
```

The compositor responds with:

- `transform(wl_output_transform)` — buffer transformation in effect.
- `damage(x, y, w, h)` — rectangles that changed (full frame for first capture).
- `presentation_time(tv_sec_hi, tv_sec_lo, tv_nsec)` — when the frame was presented.
- `ready` — frame data is available in the buffer.
- `failed(reason)` — capture failed: `unknown` (0), `buffer_constraints` (1, buffer was wrong), or `stopped` (2, session ended).

### Cursor Capture Session

`create_pointer_cursor_session` creates an `ext_image_copy_capture_cursor_session_v1` which manages cursor appearance separately. Its events track:

- `enter` / `leave` — cursor entering or leaving the captured area.
- `position(x, y)` — cursor position in buffer coordinates.
- `hotspot(x, y)` — offset from cursor image origin to pointer tip.

This allows remote desktop clients to render the cursor independently with sub-pixel accuracy, or to include it in the video stream, at the client's choice.

### Compositor Support

As of 2025, `ext-image-copy-capture-v1` is implemented by a broad set of compositors: Sway, Hyprland, labwc, KWin (KDE), Mutter (GNOME), COSMIC, Wayfire, Niri, Jay, river, Phoc, Weston, Cage, GameScope, Louvre, Mir, Muffin, and Treeland. This breadth reflects the community's convergence on this protocol as the successor standard. [Source](https://wayland.app/protocols/ext-image-copy-capture-v1)

---

## 6. PipeWire Screencast

PipeWire is the universal multimedia bus on modern Linux systems. For screen capture, it provides the transport layer between the compositor (which produces frames via the protocols above) and applications that consume those frames (OBS, browser tabs, recording tools). [Source](https://pipewire.pages.freedesktop.org/pipewire/)

### Architecture

The compositor or its portal backend creates a **source node** in PipeWire — a `pw_node` that produces video buffers. Applications create a **sink node** or connect an input `pw_stream` to that source node. WirePlumber, the session manager, links producer and consumer nodes when the policy permits it.

The video pipeline is implemented as:

```text
compositor ──[ext-image-copy or wlr-screencopy]──► portal backend
portal backend ──────────────────────────────────► PipeWire source node
PipeWire ────────────────────────────────────────► consumer pw_stream
```

There is no PipeWire module that independently pulls frames from the compositor. The compositor (or the portal backend acting on its behalf) pushes frames into PipeWire.

### pw_stream API for Video Consumers

An application consuming a screencast stream calls:

```c
struct pw_stream *stream;
stream = pw_stream_new(core, "screen-consumer",
                       pw_properties_new(
                           PW_KEY_MEDIA_TYPE, "Video",
                           PW_KEY_MEDIA_CATEGORY, "Capture",
                           NULL));

static const struct pw_stream_events stream_events = {
    PW_VERSION_STREAM_EVENTS,
    .process = on_process,   /* called when a buffer is ready */
};
pw_stream_add_listener(stream, &hook, &stream_events, NULL);

/* Format parameters describing what we can accept */
uint8_t buffer[1024];
struct spa_pod_builder b = SPA_POD_BUILDER_INIT(buffer, sizeof(buffer));
const struct spa_pod *params[1];
params[0] = spa_format_video_raw_build(&b, SPA_PARAM_EnumFormat,
    &SPA_VIDEO_INFO_RAW_INIT(
        .format = SPA_VIDEO_FORMAT_BGRA,
        .size = { 1920, 1080 },
        .framerate = { 0, 1 }));

pw_stream_connect(stream,
                  PW_DIRECTION_INPUT,
                  target_node_id,   /* PipeWire node id from portal */
                  PW_STREAM_FLAG_AUTOCONNECT |
                  PW_STREAM_FLAG_MAP_BUFFERS,
                  params, 1);
```

In the `process` callback:

```c
static void on_process(void *userdata) {
    struct pw_stream *stream = userdata;
    struct pw_buffer *buf = pw_stream_dequeue_buffer(stream);
    if (!buf) return;

    struct spa_buffer *sbuf = buf->buffer;
    /* sbuf->datas[0].type is SPA_DATA_MemPtr or SPA_DATA_DmaBuf */
    /* sbuf->datas[0].data for memory-mapped, or .fd for DMA-BUF */

    /* ... consume frame ... */

    pw_stream_queue_buffer(stream, buf);   /* release back to producer */
}
```

[Source](https://pipewire.pages.freedesktop.org/pipewire/group__pw__stream.html)

### DMA-BUF Zero-Copy Path

When both the compositor and the consumer support DMA-BUF, PipeWire negotiates `SPA_DATA_DmaBuf` buffers. The compositor allocates GEM buffers backed by device memory; PipeWire exports them as DMA-BUF file descriptors; the consumer (e.g., OBS or a WebRTC encoder) imports the fd as a GPU texture (via `EGL_EXT_image_dma_buf_import` or Vulkan's external memory) and reads the pixels directly on the GPU without any CPU copy.

DMA-BUF format/modifier negotiation is a two-party protocol between producer and consumer, mediated by PipeWire. The producer advertises supported `(format, modifier)` pairs; the consumer filters to those it can import; PipeWire allocates conforming buffers using the agreed modifier. Because the optimal modifier for a given `(format, use-flags)` combination can only be determined by the allocator, PipeWire delegates buffer allocation to the party that is responsible — typically the compositor. [Source](https://pipewire.pages.freedesktop.org/pipewire/page_dma_buf.html)

### Video Format Support

Common negotiated video formats for screen capture:

| `spa_video_format` constant | Description |
|---|---|
| `SPA_VIDEO_FORMAT_BGRA` | 32-bit BGRA, alpha channel unused; most common for shared-memory path |
| `SPA_VIDEO_FORMAT_BGRX` | Same without alpha |
| `SPA_VIDEO_FORMAT_RGBx` / `RGBA` | Less common, some compositors |
| `SPA_VIDEO_FORMAT_NV12` | YUV 4:2:0 planar; preferred for hardware encoders |
| `SPA_VIDEO_FORMAT_xRGB` / `XBGR` | 32-bit variants |

### Building a SPA Video Format Pod

The format negotiation between producer and consumer is carried in SPA POD (Plain Old Data) objects. A consumer building a format parameter for screen capture declares what formats it can accept:

```c
#include <spa/param/video/format-utils.h>
#include <spa/pod/builder.h>

/* Build an EnumFormat pod that lists acceptable video formats */
static const struct spa_pod *build_video_format(struct spa_pod_builder *b)
{
    struct spa_pod_frame frame;
    spa_pod_builder_push_object(b, &frame,
                                SPA_TYPE_OBJECT_Format,
                                SPA_PARAM_EnumFormat);
    spa_pod_builder_add(b,
        SPA_FORMAT_mediaType,    SPA_POD_Id(SPA_MEDIA_TYPE_video),
        SPA_FORMAT_mediaSubtype, SPA_POD_Id(SPA_MEDIA_SUBTYPE_raw),
        SPA_FORMAT_VIDEO_format, SPA_POD_CHOICE_ENUM_Id(3,
                                     SPA_VIDEO_FORMAT_BGRA,
                                     SPA_VIDEO_FORMAT_BGRx,
                                     SPA_VIDEO_FORMAT_NV12),
        SPA_FORMAT_VIDEO_size, SPA_POD_CHOICE_RANGE_Rectangle(
                                     &SPA_RECTANGLE(320, 240),
                                     &SPA_RECTANGLE(1, 1),
                                     &SPA_RECTANGLE(8192, 4320)),
        SPA_FORMAT_VIDEO_framerate, SPA_POD_CHOICE_RANGE_Fraction(
                                     &SPA_FRACTION(0, 1),
                                     &SPA_FRACTION(0, 1),
                                     &SPA_FRACTION(240, 1)),
        0);
    return spa_pod_builder_pop(b, &frame);
}
```

The producer (compositor side) responds with a single fixed format that satisfies both sides. PipeWire stores the negotiated format as the stream's active format, which both sides then use to allocate buffers.

### WirePlumber Session Management

WirePlumber monitors the PipeWire graph for source nodes tagged as screencasts and applies policy rules before linking them to consumers. By default, screencasting requires an active portal session to be established — WirePlumber checks that the consumer node was authorized by a portal call before completing the link. This is how the portal permission dialog translates into PipeWire graph topology.

When the portal backend creates the PipeWire source node, it sets node properties including `media.role = "Screen"` and `portal.is-default-permitted = false`. WirePlumber's default policy module holds the link in a pending state until a matching portal permission record exists for the requesting application's PID. Only then does WirePlumber call `pw_link_create()` to connect producer and consumer, completing the graph. [Source](https://pipewire.pages.freedesktop.org/wireplumber/design/understanding_session_management.html)

---

## 7. xdg-desktop-portal ScreenCast

The xdg-desktop-portal project defines a set of D-Bus interfaces that sandboxed applications can call to request privileged operations — screen capture, file access, printing — subject to user confirmation. For screen capture, the key interface is `org.freedesktop.portal.ScreenCast`. [Source](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html)

### ScreenCast Portal Interface

The portal exposes two relevant properties:

- `AvailableSourceTypes` — bitmask: `1` = MONITOR, `2` = WINDOW, `4` = VIRTUAL (virtual monitor, since v4).
- `AvailableCursorModes` — bitmask (since v2): `1` = Hidden, `2` = Embedded (cursor baked into stream), `4` = Metadata (cursor position delivered separately via PipeWire metadata).

The session lifecycle:

**Step 1: `CreateSession`**

```text
bus.call("org.freedesktop.portal.ScreenCast",
         "/org/freedesktop/portal/desktop",
         "org.freedesktop.portal.ScreenCast",
         "CreateSession",
         options: { "handle_token": "token1",
                    "session_handle_token": "session1" })
```

Returns a session handle (`/org/freedesktop/portal/desktop/session/...`).

**Step 2: `SelectSources`**

```text
SelectSources(session_handle, options: {
    "types":        dbus.UInt32(1 | 2),  /* MONITOR | WINDOW */
    "multiple":     dbus.Boolean(False),
    "cursor_mode":  dbus.UInt32(2),      /* Embedded */
    "persist_mode": dbus.UInt32(2),      /* persist until revoked */
    "restore_token": previous_token      /* remember user choice */
})
```

**Step 3: `Start`**

`Start(session_handle, parent_window, options)` triggers the compositor's source-selection UI. Upon user confirmation, it returns an array of stream descriptors. Each descriptor includes:

- `id` — PipeWire node ID (the source node to connect to).
- `position` (width, height of the source in logical pixels).
- `source_type` — which type was selected.
- `pipewire-serial` (since v6) — the PipeWire global serial for identifying the node.

**Step 4: `OpenPipeWireRemote`**

```text
fd = OpenPipeWireRemote(session_handle, options: {})
```

Returns a file descriptor for a PipeWire remote. The application calls `pw_context_connect_fd(context, fd, NULL, 0)` to connect to the PipeWire session, then connects a `pw_stream` to the node ID from step 3. All subsequent frame delivery happens over PipeWire — the D-Bus portal is no longer involved.

### persist_mode and Remembered Permissions

Since portal v4, `SelectSources` accepts `persist_mode`:

- `0`: No persistence — the user must approve every session.
- `1`: Persist while the application is running.
- `2`: Persist until revoked by the user (stored by the portal backend).

When `persist_mode` ≥ 1, `Start` returns a `restore_token` string. Passing that token back in the next `SelectSources` call skips the user dialog if the permission is still valid.

### RemoteDesktop Portal

`org.freedesktop.portal.RemoteDesktop` works alongside `ScreenCast` to enable full remote control. It adds input injection on top of the video stream. [Source](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html)

`SelectDevices(session, options: { "types": KEYBOARD | POINTER | TOUCHSCREEN })` requests which device types to control. After `Start`, the caller can inject events:

```text
NotifyKeyboardKeycode(session, options, keycode, key_state)
NotifyKeyboardKeysym(session, options, keysym, key_state)
NotifyPointerMotion(session, options, dx, dy)
NotifyPointerMotionAbsolute(session, options, stream, x, y)
NotifyPointerButton(session, options, button, button_state)
NotifyTouchDown(session, options, stream, slot, x, y)
```

Since portal v2, `ConnectToEIS(session, options)` returns a file descriptor for `libei` (Linux Event Interface), a more efficient binary protocol for injecting input events rather than individual D-Bus calls.

### Portal Backend Implementations

The portal interface is implemented by desktop-specific backends:

- **`xdg-desktop-portal-gnome`**: uses Mutter's internal screencasting API (a private D-Bus interface to GNOME Shell). Mutter captures frames using its own render pass and pushes them into PipeWire.
- **`xdg-desktop-portal-kde`**: hooks into KWin's screencasting infrastructure, which similarly uses an internal compositor API.
- **`xdg-desktop-portal-wlr`** ([https://github.com/emersion/xdg-desktop-portal-wlr](https://github.com/emersion/xdg-desktop-portal-wlr)): uses `zwlr_screencopy_v1` to pull frames from wlroots compositors. It is the portal backend for Sway, Hyprland, labwc, and related compositors. (Migration to `ext-image-copy-capture` is in progress.)

---

## 8. OBS Studio on Wayland

OBS Studio is the dominant open-source streaming and recording application. Its Wayland integration evolved substantially since OBS 27 (2021), when the `obs-xdg-portal` plugin was merged into the main codebase. [Source](https://feaneron.com/2021/03/30/obs-studio-on-wayland/)

### Source Types

OBS on Wayland offers the following relevant capture sources:

- **`pipewire-capture`**: the primary Wayland capture source. Internally calls the `org.freedesktop.portal.ScreenCast` D-Bus interface, obtains a PipeWire node ID, and connects an OBS input to that node.
- **`xshm`**: X11 shared-memory capture for applications running under XWayland. Still available as a fallback.
- **`v4l2`**: captures from V4L2 devices — webcams, capture cards, and V4L2 loopback virtual cameras.

### Portal to Texture Pipeline

When the user adds a Screen Capture source, OBS:

1. Calls `org.freedesktop.portal.ScreenCast::CreateSession`, then `SelectSources`, then `Start`.
2. Calls `OpenPipeWireRemote` to get a PipeWire fd.
3. Creates a `pw_stream` connected to the returned node ID. Sets up a `process` callback.
4. On each `process` callback, dequeues the `pw_buffer`.
5. If `sbuf->datas[0].type == SPA_DATA_DmaBuf`: imports the fd as an EGL external image using the `EGL_EXT_image_dma_buf_import` or `EGL_EXT_image_dma_buf_import_modifiers` extension:

```c
EGLImageKHR img = eglCreateImageKHR(
    egl_display, EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT, NULL,
    attribs  /* DMA-BUF fd, format, width, height, modifier */);
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, img);
/* Texture is now backed by compositor GPU memory — zero CPU copy */
```

The OBS rendering pipeline then uses this texture as a source for its scene graph. The `gs_texture_create_from_dmabuf()` function in OBS's graphics subsystem wraps this sequence.

6. If DMA-BUF is unavailable (software path): the buffer `data` pointer is a CPU-mapped pixel array. OBS uploads it to a texture via `glTexImage2D`.

### Output Encoding

OBS encodes the captured stream via selectable encoders:

- **VA-API** (`obs-vaapi`): sends frames to the hardware video encoder via `libva`, producing H.264 or HEVC bitstreams with very low CPU overhead.
- **NVENC** (via FFmpeg): uses NVIDIA's NVENC fixed-function encoder over FFmpeg's `AVCodecContext`.
- **Software x264 / x265**: CPU encoding; high quality but high CPU load.
- **AV1 via VA-API**: available on Intel Arc and AMD RDNA3+ hardware.

When the DMA-BUF path is active, the GPU memory containing the captured frame never touches system RAM: the compositor writes to GPU memory, PipeWire passes the DMA-BUF fd, OBS imports it as an EGL texture, and VA-API reads it from GPU memory for encoding. [Source](https://github.com/obsproject/rfcs/pull/14)

### Virtual Camera

OBS can output its composed scene as a virtual V4L2 camera using the `v4l2loopback` kernel module. After loading the module:

```bash
sudo modprobe v4l2loopback devices=1 video_nr=10 card_label="OBS Virtual Camera"
```

OBS writes frames into the loopback device at `/dev/video10`. Any V4L2 consumer — video conferencing apps, other OBS instances, `ffplay` — sees it as a standard camera.

---

## 9. WebRTC Screen Sharing

WebRTC's `getDisplayMedia()` is the standard JavaScript API for browser-based screen sharing. On Linux its implementation routes through xdg-desktop-portal. [Source](https://www.w3.org/TR/screen-capture/)

### Chromium Implementation

When a web page calls `navigator.mediaDevices.getDisplayMedia()` in Chromium, the browser invokes a PipeWire-based desktop capturer class (located under `content/browser/media/capture/` in the Chromium source tree). This implementation: [Source](https://source.chromium.org/chromium/chromium/src/+/main:content/browser/media/capture/)

1. Opens the `org.freedesktop.portal.ScreenCast` D-Bus interface using a helper in the browser process.
2. Performs the `CreateSession` → `SelectSources` → `Start` sequence, presenting the compositor's source-selection UI to the user.
3. Calls `OpenPipeWireRemote` to receive a PipeWire fd.
4. Connects a `pw_stream` in the GPU process to the node ID returned by `Start`.
5. On each incoming frame, imports the buffer — preferring `SPA_DATA_DmaBuf` to avoid GPU→CPU copies.

The captured texture enters Chromium's compositing pipeline as a `SharedImage`, flows through the Viz display compositor, and is delivered to the WebRTC encoder (typically `libvpx` for VP8/VP9, or OpenH264 / hardware VA-API for H.264).

The feature is gated by the `--enable-features=WebRTCPipeWireCapturer` flag in older Chromium builds; in current versions (2024+) it is enabled by default when `XDG_SESSION_TYPE=wayland` is detected.

### Firefox Implementation

Firefox uses a similar path. On Wayland it calls the `org.freedesktop.portal.ScreenCast` portal via its GTK/GDK integration layer (specifically through `GdkWaylandDisplay`). The resulting PipeWire stream is decoded into a `webrtc::VideoFrame` and fed into Firefox's libwebrtc fork.

### Cursor Metadata

When `cursor_mode` is set to `4` (Metadata), PipeWire delivers cursor position as side-channel information in the buffer's metadata map rather than compositing it into the pixel data. Chromium's `PipeWireDesktopCapturer` reads this metadata and synthesizes a separate `MouseCursor` region that WebRTC can encode or transmit separately, enabling remote viewers to render a native-resolution cursor instead of a cursor baked at the stream resolution.

### Latency Budget

The complete latency path for a WebRTC screen share session:

| Stage | Typical Latency |
|---|---|
| Compositor frame rendering | 2–8 ms |
| ext-image-copy-capture or wlr-screencopy | 1–3 ms |
| PipeWire buffer handoff | < 0.5 ms |
| Browser GPU import (DMA-BUF) | < 0.5 ms |
| WebRTC video encoder (VP9/H.264) | 5–15 ms |
| RTP packetization and transmission | 1–50 ms (local LAN to internet) |
| Receiver decode and playout buffer | 10–30 ms |
| **Total (LAN)** | **~25–60 ms** |

The dominant variable is network latency. The DMA-BUF path eliminates a full GPU→CPU→GPU cycle that would otherwise add 10–30 ms on high-resolution displays.

---

## 10. Remote Desktop: RDP, VNC, and Game Streaming

Remote desktop is screen capture plus input injection plus network transport plus decoder on the client side. The following covers the major Linux implementations.

### GNOME Remote Desktop

`gnome-remote-desktop` (GRD) is GNOME's built-in RDP and VNC server, enabled by default in GNOME 46 and later. [Source](https://gitlab.gnome.org/GNOME/gnome-remote-desktop)

**Architecture**: GRD runs as a user daemon (`gnome-remote-desktop-daemon`). It creates a `org.freedesktop.portal.ScreenCast` session to capture the active GNOME Shell session, and a `org.freedesktop.portal.RemoteDesktop` session to inject input events from the remote client.

**Encoding**: For RDP connections, GRD encodes the desktop stream using H.264 (AVC) via the RDP Graphics Pipeline Extension (EGFX). Hardware encoding is supported via VA-API using the `libva` interface, or via a Vulkan + VA-API path for zero-copy rendering. The server advertises both `H264 (AVC444)` and `H264 (AVC420)` capabilities during RDP capability exchange. [Source](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/294)

**Headless mode**: Since GNOME 46, GRD supports headless remote login sessions that start independently of an attached display, driven by GDM. This enables cloud desktop and VDI scenarios without requiring a physical monitor.

**Client compatibility**: Any RDP client works — Windows built-in Remote Desktop Connection, FreeRDP, Remmina, and GNOME Connections. The default port is 3389.

```bash
# Enable GNOME Remote Desktop (Wayland session)
grdctl rdp enable
grdctl rdp set-credentials <username> <password>
grdctl status
```

### xrdp (X11-Based)

`xrdp` ([https://github.com/neutrinolabs/xrdp](https://github.com/neutrinolabs/xrdp)) is the traditional Linux RDP server targeting X11 sessions. It runs alongside a display manager and spawns X11 sessions for each authenticated user.

**Architecture**: `xrdp-sesman` authenticates via PAM, then launches an X11 session through a configurable backend:
- **Xvfb** (X Virtual Framebuffer): a headless X server that renders into shared memory.
- **xorgxrdp**: a specialized Xorg video driver (`xrdp-glamor`) that captures the X11 framebuffer directly.
- **Xvnc**: bridges xrdp to a VNC session.

`xrdp` uses the `xorgxrdp` module to capture BGRA pixel data from the Xorg framebuffer, compresses it using RemoteFX or raw RDP bitmaps, and sends it over the Microsoft RDP protocol. Hardware acceleration via Glamor (OpenGL 2D acceleration for Xorg) can be active in the X session, but the final capture is still a CPU readback.

The latest stable release is xrdp 0.10.4.1 (July 2025). Xrdp does not support Wayland sessions natively; for Wayland, use GNOME Remote Desktop or a Wayland-native solution.

### KDE Plasma Remote Desktop

KDE's Plasma desktop exposes remote desktop via `xdg-desktop-portal-kde`. The `krfb` application provides VNC-based desktop sharing. For RDP, KWin implements `ext-image-copy-capture-v1` and exports the desktop stream via PipeWire; third-party RDP servers can consume it. [Source](https://bugs.kde.org/show_bug.cgi?id=513785)

### Sunshine: GPU-Accelerated Game Streaming

**Sunshine** ([https://github.com/LizardByte/Sunshine](https://github.com/LizardByte/Sunshine)) is an open-source self-hosted implementation of NVIDIA's discontinued GameStream protocol, compatible with the **Moonlight** client family. It targets ultra-low-latency game streaming rather than productivity remote desktop.

**Capture**: On Linux/Wayland, Sunshine captures via the KMS DRM path (direct framebuffer read using libdrm) or via the xdg-desktop-portal on compositors that support it. The capture is designed to run at the game's native framerate (up to 120 fps).

**Encoding**: Sunshine supports multiple hardware encoders:

| Encoder | Hardware | Codec |
|---|---|---|
| NVENC | NVIDIA (any modern GPU) | H.264, HEVC, AV1 |
| VA-API | AMD, Intel | H.264, HEVC, AV1 |
| Vulkan Video | Any Vulkan-capable GPU (v2026+) | H.264, HEVC |
| Software | CPU | H.264, HEVC |

Sunshine v2026.413 introduced Vulkan Video encode support (`VK_KHR_video_encode_h264`) as an alternative to VA-API. [Source](https://github.com/LizardByte/Sunshine/releases)

**Protocol**: The GameStream protocol uses RTSP for signaling and RTP over UDP for video/audio. Moonlight clients (Linux, Windows, macOS, Android, iOS, Steam Link) negotiate the stream parameters including resolution, codec, and bitrate.

**Latency breakdown for game streaming**:

| Stage | Typical Latency |
|---|---|
| GPU render completion | 5–16 ms (game frame time) |
| Frame capture (KMS path) | 1–2 ms |
| Hardware encode (NVENC/VA-API) | 2–5 ms |
| Network transmission (LAN) | 0.5–2 ms |
| Client decode (hardware) | 2–5 ms |
| Display scan-out | 0–16 ms |
| **Total (LAN)** | **~15–40 ms** |

LAN game streaming can approach the same perceived latency as local play when GPU and network conditions cooperate.

### Input Injection in Remote Desktop

All Wayland-native remote desktop solutions must inject input events through the compositor's input path. The options are:

1. **`org.freedesktop.portal.RemoteDesktop` D-Bus methods** (`NotifyKeyboardKeycode`, `NotifyPointerMotion`, etc.) — simple but adds D-Bus overhead per event.
2. **`ConnectToEIS`** — returns a `libei` fd. The caller speaks the EIS (Event Input Stream) binary protocol, which the compositor's `libeis` implementation delivers to the focused window. This is the preferred path for high-frequency mouse input.
3. **`zwlr_virtual_keyboard_v1`** and **`zwlr_virtual_pointer_v1`** (wlr-protocols): lower-level protocol extensions that compositors supporting wlroots protocols can implement. Used by wayvnc and other tools on wlroots compositors.

### WayVNC

**wayvnc** ([https://github.com/any1/wayvnc](https://github.com/any1/wayvnc)) is a VNC server for Wayland that uses `ext-image-copy-capture-v1` (or `wlr-screencopy` as fallback) for capture and `zwlr_virtual_keyboard_v1` / `zwlr_virtual_pointer_v1` for input. It demonstrates the minimal viable Wayland remote desktop stack without the complexity of RDP or the overhead of a portal.

wayvnc's encoding path supports tight, zlib, and raw VNC encodings. It does not use hardware-accelerated video encoding; the resulting bitrate is much higher than RDP/H.264 but the implementation is simpler and latency is lower for LAN connections without network constraints.

### Comparing Remote Desktop Approaches

| Solution | Protocol | Capture path | Encoding | Wayland-native |
|---|---|---|---|---|
| gnome-remote-desktop | RDP + VNC | Portal ScreenCast → PipeWire | H.264 via VA-API | Yes |
| xrdp | RDP | X11 framebuffer (xorgxrdp) | RemoteFX / raw bitmaps | No (X11 only) |
| wayvnc | VNC | wlr-screencopy / ext-image-copy | Tight / zlib | Yes |
| Sunshine | GameStream (RTSP/RTP) | KMS DRM / portal | H.264/HEVC/AV1 NVENC or VA-API | Yes (partial) |
| RustDesk | proprietary | Portal ScreenCast | VP9 / H.264 | Yes |

### Capture Latency and Encode Budget

The fundamental latency floor for any remote desktop system is determined by two factors: the capture interval (time from frame composition to buffer readiness) and the encode time (time to compress the captured frame into a transmittable bitstream).

**Capture interval**: With `ext-image-copy-capture-v1`, the compositor signals `ready` as soon as the framebuffer write completes — typically within 1–3 ms of the VBLANK that presented the frame. With `wlr-screencopy copy_with_damage`, capture only triggers when the compositor detects damage, which can save power but doesn't change the latency for actively changing content.

**Encode time**: H.264 hardware encoding (VA-API or NVENC) takes 2–5 ms end-to-end including DMA-BUF import. Software H.264 (x264) takes 10–40 ms depending on preset. AV1 hardware encoding (on Intel Arc or RDNA3) is comparable to H.264 hardware in latency but produces smaller bitstreams at equal quality, becoming preferred for bandwidth-constrained remote desktop.

**Network transport**: RDP uses TCP, which adds variable latency under packet loss. Game streaming protocols (Sunshine/Moonlight) use RTP over UDP with custom FEC (Forward Error Correction) and retransmission, enabling sub-100 ms round-trip latency on LAN and sub-150 ms on good internet connections.

---

## 11. Integrations

**Ch2 (KMS: The Display Pipeline)**: The KMS writeback connector (Section 3) is a first-class DRM object — a `drm_connector` with `DRM_MODE_CONNECTOR_WRITEBACK` type. It participates in atomic commits alongside CRTCs and planes, and its framebuffer output is a GEM buffer object in the DRM memory management system. Understanding KMS atomic modesetting is prerequisite to implementing writeback capture.

**Ch20 (Wayland Protocol Fundamentals)**: The `ext-image-copy-capture-v1` and `ext-image-capture-source-v1` protocols (Section 5) are Wayland protocol extensions built on core Wayland object lifecycle rules. `wl_buffer`, `wl_output`, and `ext_foreign_toplevel_handle_v1` are all core or staging Wayland objects. The linux-dmabuf protocol (Ch20 §10) underlies DMA-BUF buffer sharing in wlr-screencopy and ext-image-copy-capture.

**Ch21 (Building Compositors with wlroots)**: The `wlr-screencopy-v1` protocol (Section 4) originates in the wlroots project and remains a wlroots-family protocol. Compositors built on wlroots (Sway, Hyprland, labwc) implement it via wlroots's `wlr_screencopy_manager_v1` API. `xdg-desktop-portal-wlr` (Section 7) similarly depends on wlroots screencopy.

**Ch22 (Production Compositors)**: GNOME Shell (Mutter), KWin, and Sway each implement screen capture differently. Mutter uses its private screencasting D-Bus API; KWin implements `ext-image-copy-capture-v1` directly; Sway delegates to wlroots. The remote desktop implementations in Section 10 (gnome-remote-desktop, KDE's portal) sit on top of these compositor capture paths.

**Ch23 (Legacy and Sandboxed App Support)**: The xdg-desktop-portal (Section 7) is the security boundary that enables sandboxed Flatpak and Snap applications to request screen capture with user consent. The `org.freedesktop.portal.ScreenCast` and `org.freedesktop.portal.RemoteDesktop` interfaces are Flatpak's canonical screen-access APIs. The portal's permission store and `persist_mode` are the Flatpak-layer analogue to browser permission grants.

**Ch38 (PipeWire and the Video Session Layer)**: PipeWire (Section 6) is the transport backbone for all modern Linux screen capture. `pw_stream`, `spa_pod` format negotiation, DMA-BUF buffer types, and WirePlumber policy linking are all detailed in Ch38. This chapter's screencast usage is a specific application of the general PipeWire video streaming infrastructure.

**Ch46 (The Evolving Wayland Protocol Ecosystem)**: `ext-image-copy-capture-v1` is one of the protocols tracked in Ch46's survey of staging Wayland protocols. The relationship between `ext-image-capture-source-v1` and `ext_foreign_toplevel_handle_v1` (the window handle) reflects the protocol composition patterns described there.

**Ch52 (Firefox and WebRender)**: Firefox's `getDisplayMedia()` implementation (Section 9) routes through the xdg-desktop-portal on Wayland, shares the same PipeWire screencast infrastructure, and feeds captured frames into Firefox's libwebrtc fork. The interaction between Firefox's compositor (WebRender) and the PipeWire consumer thread is discussed in Ch52's coverage of Firefox's multimedia pipeline.

**Ch57 (FFmpeg Architecture and Programming)**: Remote desktop encoding (Section 10) relies on FFmpeg for software and some hardware encode paths. `wf-recorder` uses FFmpeg for muxing and encoding. xrdp uses FFmpeg-backed codecs for RemoteFX. Sunshine's software fallback encoder is also FFmpeg-based. The VA-API encode path used by gnome-remote-desktop and OBS is accessible via FFmpeg's `vaapi_encode` filter chain.

**Ch75 (Explicit GPU Synchronisation)**: DMA-BUF fences are integral to the writeback connector path (Section 3): `drm_writeback_job::out_fence` is a `dma_fence` that signals when the hardware has completed writing to the framebuffer. The `WRITEBACK_OUT_FENCE_PTR` KMS property exports this as a sync file. Consumers must wait on this fence before reading the buffer, using the explicit sync mechanisms described in Ch75.

**Ch112 (VRR — FreeSync and Frame Pacing)**: When VRR is active (Ch112), the compositor's frame delivery timing is variable. Screen capture clients using `copy_with_damage` adapt naturally — they receive a new capture frame whenever the compositor renders and presents one, inheriting the variable timing. However, capture tools that assume a fixed framerate (e.g., a recorder expecting 60 fps) must handle variable inter-frame intervals. The `presentation_time` event in `ext-image-copy-capture-v1` and the `ready` timestamp in `wlr-screencopy` provide accurate per-frame timestamps that enable variable-framerate encoding, solving the VRR/capture interaction.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
