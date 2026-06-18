# Chapter 38: PipeWire and the Video Session Layer

> **Part**: Part VII — Application APIs & Middleware
> **Audience**: Graphics application developers who need to route GPU-produced buffers through the session-layer multimedia infrastructure; systems developers interested in the DMA-BUF / PipeWire integration that connects kernel video capture, GPU compute, and Wayland screen sharing
> **Status**: First draft — 2026-06-18

## Table of Contents

- [Overview](#overview)
- [1. PipeWire Architecture: The Graph Model and SPA](#1-pipewire-architecture-the-graph-model-and-spa)
- [2. Session Management: WirePlumber and Policy](#2-session-management-wireplumber-and-policy)
- [3. Video Capture Pipeline: V4L2 and libcamera Sources](#3-video-capture-pipeline-v4l2-and-libcamera-sources)
- [4. Screen Capture and Portal Integration](#4-screen-capture-and-portal-integration)
- [5. Remote Desktop and Streaming](#5-remote-desktop-and-streaming)
- [6. Buffer Formats and GPU Interop](#6-buffer-formats-and-gpu-interop)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

PipeWire is the unified session-layer multimedia daemon that now underpins virtually every modern Linux desktop. Introduced by Wim Taymans at Red Hat and first shipping as default in Fedora 34 (2021), it replaced a historically fractured landscape: PulseAudio handled audio routing, JACK served professional low-latency audio, ALSA was accessed directly by many applications, and video was scattered across V4L2 ioctl loops, v4l2loopback kernel modules, and ad-hoc GStreamer pipelines stitched together by compositors. PipeWire unifies all of these under a single graph-based multimedia engine, providing zero-copy DMA-BUF buffer passing, real-time scheduling, and a robust security model built on portal-controlled permissions.

For the graphics stack, PipeWire is not merely an audio daemon that happens to handle video: it is the primary conduit through which three high-value data flows operate. First, Wayland screen capture — the mechanism by which OBS Studio, video-conferencing applications, and WebRTC in browsers receive compositor output — travels through a PipeWire stream negotiated via the `org.freedesktop.portal.ScreenCast` D-Bus interface. Second, hardware video capture from webcams (through V4L2 or libcamera) reaches applications as DMA-BUF-backed PipeWire streams, enabling zero-copy paths all the way from the camera sensor's ISP to a GPU encode engine. Third, rendered GPU frames from Vulkan or EGL applications can be injected back into PipeWire as `SPA_DATA_DmaBuf` buffers and consumed by encoders, recorders, or remote-desktop servers.

This chapter is structured for three audiences. **Graphics application developers** will find in sections 4 and 6 a precise description of the portal ScreenCast flow and the DMA-BUF modifier negotiation that determines whether screen capture is zero-copy or falls back to a CPU blit. **Systems and driver developers** will find in sections 1–3 a thorough treatment of the SPA plugin ABI, the graph scheduler, the V4L2 / libcamera source nodes, and the buffer-lifecycle contracts that cross kernel/userspace boundaries. **Browser and compositor engineers** will find in sections 4–5 the exact D-Bus interfaces, portal backend choices per compositor, and the WebRTC plumbing that makes in-browser screen sharing work.

---

## 1. PipeWire Architecture: The Graph Model and SPA

### 1.1 The Node/Link/Port Graph

PipeWire models all multimedia data flow as a directed graph of *nodes* connected by *links*. Each node (`pw_node`) represents an entity that either consumes, produces, or transforms buffers. Nodes expose *ports* — typed endpoints that carry data in one direction: output ports emit buffers into the graph; input ports consume them. A *link* (`pw_link`) connects one output port to one input port, provided they have compatible formats. [Source](https://docs.pipewire.org/page_overview.html)

Sources — such as a V4L2 camera node or a screen-capture node — have only output ports. Sinks — such as an ALSA PCM device or a recording sink — have only input ports. Processing nodes (resamplers, format converters, encoders) have both. Every object — node, link, port, device, session — is identified by an integer object ID visible through the `pw_registry`, and clients interact with remote objects through local *proxy* handles.

```
+--------------------+        pw_link         +--------------------+
|  pw_node (camera)  |   output ──────────►   |  pw_node (encoder) |
|                    |                         |                    |
|   spa_node impl    |   input  ◄──────────   |   spa_node impl    |
+--------------------+        pw_link         +--------------------+
```

```mermaid
graph LR
    CAM["pw_node\n(camera source)"] -- "pw_link\n(output port)" --> ENC["pw_node\n(encoder)"]
    ENC -- "pw_link\n(output port)" --> SINK["pw_node\n(recording sink)"]
```

Applications almost never interact with `pw_node` directly. Instead, they use the higher-level `pw_stream` API, which creates a node with a single port inside the PipeWire daemon, takes care of format negotiation, buffer allocation, and wires up the necessary event callbacks. The `pw_stream` wraps a `pw_client_node` proxy plus an adapter node that can perform lightweight format conversion on the data path.

The IPC substrate uses UNIX domain sockets at `$XDG_RUNTIME_DIR/pipewire-0` (configurable via `core.name` in `pipewire.conf`). Buffer data is shared via `memfd` or DMA-BUF file descriptors — the control channel carries only metadata and signals. This design means data never passes through the socket: file descriptors are exchanged once during negotiation, after which the data exchange is purely in shared memory or GPU memory. [Source](https://docs.pipewire.org/page_overview.html)

### 1.2 SPA: The Simple Plugin API

Below `libpipewire` sits SPA (Simple Plugin API) — the plugin ABI that every codec, device backend, format negotiator, and buffer allocator implements. SPA was designed around three constraints: no heap allocation inside plugin code (callers supply pre-sized memory for every object), a purely event-driven result model (no blocking calls), and stable binary compatibility across versions. [Source](https://docs.pipewire.org/page_spa_plugins.html)

An SPA plugin is a shared library that exports a single public symbol, `spa_handle_factory_enum`, returning an array of `spa_handle_factory` structures. Each factory enumerates the *interfaces* it can instantiate — a given `.so` may provide a video converter, a buffer allocator, and a format enumerator. Plugins are typically located under `/usr/lib/spa-0.2/` and found via a `SPA_PLUGIN_DIR` search path.

The central interface for a processing unit is `spa_node`:

```c
/* spa/include/spa/node/node.h (simplified) */
struct spa_node_methods {
    uint32_t version;
    int (*add_listener)(void *object, struct spa_hook *listener,
                        const struct spa_node_events *events, void *data);
    int (*set_callbacks)(void *object, const struct spa_node_callbacks *callbacks,
                         void *data);
    int (*enum_params)(void *object, int seq, uint32_t id, uint32_t start,
                       uint32_t num, const struct spa_pod *filter);
    int (*set_param)(void *object, uint32_t id, uint32_t flags,
                     const struct spa_pod *param);
    int (*port_enum_params)(void *object, int seq, enum spa_direction direction,
                            uint32_t port_id, uint32_t id, uint32_t start,
                            uint32_t num, const struct spa_pod *filter);
    int (*port_set_param)(void *object, enum spa_direction direction,
                          uint32_t port_id, uint32_t id, uint32_t flags,
                          const struct spa_pod *param);
    int (*port_use_buffers)(void *object, enum spa_direction direction,
                            uint32_t port_id, uint32_t flags,
                            struct spa_buffer **buffers, uint32_t n_buffers);
    int (*process)(void *object);
};
```
[Source: `spa/include/spa/node/node.h`, PipeWire GitLab](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/spa/include/spa/node/node.h)

Parameters — formats, buffer requirements, port capabilities — are exchanged as *SPA pods* (`spa_pod`). A pod is a self-describing binary value: a 32-bit size followed by a 32-bit type tag, then the payload. Pods compose: an `SPA_TYPE_OBJECT_Format` pod contains property pods keyed by SPA parameter constants such as `SPA_FORMAT_VIDEO_format`, `SPA_FORMAT_VIDEO_size`, and `SPA_FORMAT_VIDEO_modifier`. The `spa_pod_builder` API constructs pods on stack-allocated buffers; `spa_pod_parser` decodes them.

Video format objects use `spa_video_info_raw`, which is the concrete structure populated by `spa_format_video_raw_parse()` after format negotiation:

```c
/* spa/include/spa/param/video/raw.h */
struct spa_video_info_raw {
    enum spa_video_format  format;        /* e.g. SPA_VIDEO_FORMAT_NV12   */
    uint32_t               flags;
    uint64_t               modifier;      /* DRM modifier; only used with DMA-BUF */
    struct spa_rectangle   size;          /* width × height               */
    struct spa_fraction    framerate;     /* 0/1 = variable               */
    struct spa_fraction    max_framerate;
    uint32_t               views;
    enum spa_video_interlace_mode  interlace_mode;
    struct spa_fraction    pixel_aspect_ratio;
    /* … colorimetry fields … */
    enum spa_video_color_range      color_range;
    enum spa_video_color_matrix     color_matrix;
    enum spa_video_transfer_function transfer_function;
    enum spa_video_color_primaries  color_primaries;
};
```
[Source: PipeWire docs `spa_video_info_raw` Struct Reference](https://docs.pipewire.org/structspa__video__info__raw.html)

Note that `spa_video_info_raw` is the single structure used for both ordinary video and DMA-BUF video. The `modifier` field is populated only when DMA-BUF negotiation succeeds. There is no separate `spa_video_info_dma_buf` type; DMA-BUF-ness is indicated by setting `SPA_PARAM_BUFFERS_dataType` to `1 << SPA_DATA_DmaBuf` during buffer parameter negotiation.

### 1.3 The Daemon, Client Libraries, and Socket

The `pipewire` daemon process owns the native protocol socket and executes the real-time data thread. It does not implement policy (which nodes to link, which formats to negotiate) — that is left to the session manager. The daemon's responsibilities are: maintaining the object graph in shared memory, driving the data-processing cycle, and forwarding IPC messages between clients.

Applications link against `libpipewire-0.3.so` and communicate with the daemon through its native protocol. The library exposes a main loop (`pw_main_loop` or `pw_thread_loop`) that integrates with `epoll` via eventfd. A `pw_context` manages SPA plugin loading; a `pw_core` holds the connection to the daemon and the proxy to the `pw_registry`.

For compatibility, separate daemons implement legacy protocols on top of PipeWire's graph:
- `pipewire-pulse` — a PulseAudio wire-protocol server that creates virtual sink/source nodes.
- `pw-jack` — an `LD_LIBRARY_PATH`-based shim that redirects JACK clients through PipeWire.
- The ALSA PCM plugin in `~/.config/alsa/asound.conf` routes `libasound` playback into PipeWire streams.

```mermaid
graph TD
    APP["Application"] -- "libpipewire-0.3.so\n(pw_context / pw_core)" --> SOCK["UNIX socket\n$XDG_RUNTIME_DIR/pipewire-0"]
    PULSE["PulseAudio client"] -- "pipewire-pulse\nwire protocol" --> SOCK
    JACK["JACK client"] -- "pw-jack\nLD_LIBRARY_PATH shim" --> SOCK
    ALSA["libasound client"] -- "ALSA PCM plugin\nasound.conf" --> SOCK
    SOCK --> DAEMON["pipewire daemon\n(object graph + data thread)"]
    DAEMON -- "pw_registry" --> REG["pw_registry\n(object IDs)"]
    DAEMON -- "session manager IPC" --> WP["WirePlumber\n(policy)"]
```

### 1.4 Scheduling: The Driver Node and the Quantum

The PipeWire data thread operates on a pull-based *graph cycle* model. One node in the graph is elected as the *driver*: it owns a hardware timer (for ALSA, the DMA period interrupt; for video, a v4l2 frame-ready event) and emits a ready signal that propagates through the dependency graph. Other nodes are *followers*: they process only after all upstream dependencies have completed, signalled by decrementing an atomic counter and writing to an `eventfd`. [Source](https://docs.pipewire.org/page_scheduling.html)

```mermaid
graph TD
    HW["Hardware timer\n(DMA period / v4l2 frame-ready)"] -- "ready signal" --> DRIVER["Driver node\n(SCHED_FIFO data thread)"]
    DRIVER -- "decrements atomic counter" --> F1["Follower node A"]
    DRIVER -- "decrements atomic counter" --> F2["Follower node B"]
    F1 -- "eventfd signal\nwhen upstream complete" --> F3["Follower node C"]
    F2 -- "eventfd signal\nwhen upstream complete" --> F3
```

The *quantum* is the number of samples (for audio) or frames (for video) processed per graph cycle. For audio it defaults to `default.clock.quantum = 1024` samples at `default.clock.rate = 48000 Hz`, giving a 21.3 ms period. Clients can request smaller quanta for lower latency (bounded by `default.clock.min-quantum = 32`) or larger quanta for efficiency (capped at `default.clock.quantum-limit = 8192`). The graph uses the smallest quantum requested by any running client, rounded down to a power of two. [Source](https://docs.pipewire.org/page_man_pipewire_conf_5.html)

The data thread runs with `SCHED_FIFO` real-time priority to prevent preemption by other processes. This is granted either via the `module-rt` module (which requires `RLIMIT_RTPRIO` to be set sufficiently for the calling process — typically via `/etc/security/limits.d/`) or via the RTKit D-Bus service (which implements privilege escalation for ordinary users without requiring raised resource limits). The kernel's `RLIMIT_RTTIME` limit caps the maximum wall-clock time a thread may spend in real-time scheduling without yielding, providing a safety valve against runaway real-time threads consuming all CPU. [Source: PipeWire RT module docs](https://docs.pipewire.org/page_module_rt.html)

### 1.5 Format Negotiation via SPA Pods

Format negotiation between two nodes proceeds in two phases. During the *enumeration* phase, each port calls `spa_node_port_enum_params(SPA_PARAM_EnumFormat)` to list all format capabilities as an array of SPA pod objects — each encoding constraints on media type, format, size, and framerate using `SPA_CHOICE_Enum` or `SPA_CHOICE_Range` pods. The session manager or the adapter node intersects the capability sets to find a mutually acceptable format.

During the *fixation* phase, the negotiated format is committed with `spa_node_port_set_param(SPA_PARAM_Format)`, followed by buffer negotiation (`SPA_PARAM_Buffers`) that determines memory type (`SPA_DATA_MemFd`, `SPA_DATA_DmaBuf`), buffer count, stride, and alignment. This two-phase model is the foundation of the zero-copy DMA-BUF path explored in section 6.

---

## 2. Session Management: WirePlumber and Policy

### 2.1 The Role of the Session Manager

The PipeWire daemon deliberately has no built-in policy. It creates and manages graph objects but never decides which microphone to route to which application or whether a Flatpak is allowed to see the camera. These decisions are the exclusive domain of the *session manager* — a separate daemon that connects to PipeWire as a privileged client and uses the full object management API. [Source](https://docs.pipewire.org/page_session_manager.html)

Two session managers have existed: `pipewire-media-session` (the early reference implementation, now used mainly for debugging and embedded systems) and **WirePlumber** (the current standard on all major Linux desktops since 2021). This section covers WirePlumber exclusively.

### 2.2 WirePlumber Architecture

WirePlumber is a GObject-based daemon with a Lua 5.4/5.5 scripting engine embedded via the `wplua` library. The daemon itself is a thin plugin host: at startup it loads a set of C plugins (for the event loop, the PipeWire connection, the Lua engine) and then executes Lua scripts that implement all policy logic. [Source](https://pipewire.pages.freedesktop.org/wireplumber/)

The key architectural abstraction is the `WpObjectManager`, which watches the PipeWire object registry for objects matching a given `WpObjectInterest` (a type + property filter). When a matching object appears, the manager fires a Lua callback. Scripts use this to react to devices coming online, streams being created, or links changing state.

WirePlumber 0.5.x reorganised the main linking script from a monolithic `policy-node.lua` into an *event dispatcher* model with hooks. When a new stream node appears, the dispatcher raises a `select-target` event. A chain of prioritised hook scripts handles it: [Source](https://www.collabora.com/news-and-blog/blog/2023/10/30/wireplumber-exploring-lua-scripts-with-event-dispatcher/)

```lua
-- wireplumber/scripts/linking/find-best-target.lua (simplified illustration)
SimpleEventHook {
    name = "linking/find-best-target",
    after = "linking/find-default-target",
    before = "linking/prepare-link",
    execute = function(event)
        local si_props  = event:get_subject():get_properties()
        local media_cls = si_props["media.class"]
        -- find highest-priority node with matching media class
        local target = find_best_node(media_cls)
        if target then
            event:set_data("target", target)
        end
    end
}
```

Scripts are loaded from `/usr/share/wireplumber/scripts/` and can be overridden per-user from `~/.config/wireplumber/`. Administrators or embedded-system vendors supply entirely custom scripts without patching WirePlumber itself. [Source](https://gkiagia.gr/2025-04-22-wireplumber-configuration-on-embedded/)

### 2.3 Device and Node Lifecycle

WirePlumber creates `pw_node` instances for hardware devices by loading the appropriate SPA *monitor* plugin: `api.alsa.enum.udev` enumerates ALSA cards, `api.v4l2.enum.udev` enumerates V4L2 video devices, and `api.libcamera.enum.devices` enumerates libcamera-managed cameras. When `udev` reports a new `/dev/video0` device, the V4L2 monitor instantiates a `pw_node` wrapping the `spa-v4l2` plugin; when the device disappears, WirePlumber unlinks and destroys the node.

The *linking policy* maps media classes to each other. Stream nodes carry a `media.class` property such as `Stream/Output/Audio` (playback), `Stream/Input/Audio` (capture), or `Stream/Input/Video` (video capture). Device nodes carry `Audio/Sink`, `Audio/Source`, or `Video/Source`. The linker automatically routes `Stream/Input/Video` nodes to `Video/Source` nodes unless overridden by `node.target` or metadata. [Source](https://pipewire.pages.freedesktop.org/wireplumber/policies/linking.html)

```mermaid
graph TD
    UDEV["udev\n(kernel device event)"] -- "new /dev/videoN" --> V4L2MON["api.v4l2.enum.udev\n(SPA monitor plugin)"]
    UDEV -- "new camera device" --> LCMON["api.libcamera.enum.devices\n(SPA monitor plugin)"]
    UDEV -- "new ALSA card" --> ALSAMON["api.alsa.enum.udev\n(SPA monitor plugin)"]
    V4L2MON -- "instantiates" --> V4L2NODE["pw_node\n(spa-v4l2, Video/Source)"]
    LCMON -- "instantiates" --> LCNODE["pw_node\n(libcamera, Video/Source)"]
    ALSAMON -- "instantiates" --> ALSANODE["pw_node\n(Audio/Sink or Audio/Source)"]
    V4L2NODE -- "pw_registry object" --> WP["WirePlumber linker\n(select-target event)"]
    LCNODE -- "pw_registry object" --> WP
    ALSANODE -- "pw_registry object" --> WP
    WP -- "pw_link" --> STREAM["Stream node\n(Stream/Input/Video or Stream/Input/Audio)"]
```

Key linking properties on stream nodes:
- `target.object`: name or serial of the desired target node; accepts both device and stream names, enabling stream-to-stream routing.
- `node.dont-reconnect`: when `false` (default), the linker reconnects the stream if the target disappears.
- `node.dont-move`: when `true`, metadata changes cannot relink the stream at runtime.

### 2.4 Default Sink/Source and Metadata

PipeWire supports a `pw_metadata` object (keyed `default`) that stores key-value pairs persisted across sessions. WirePlumber writes `default.audio.sink`, `default.audio.source`, and `default.video.source` metadata keys to record which device the user has designated as the session default. Tools like `pavucontrol` and `pactl` read and write these keys; WirePlumber's `find-default-target.lua` hook consults them during link target selection.

### 2.5 Security Model: Permissions and Portals

PipeWire implements a permissions system analogous to UNIX file permissions. Every object in the graph has a permission bitmask per client: `PW_PERM_R` (read — see the object exists), `PW_PERM_W` (write — modify properties), `PW_PERM_X` (execute — call methods / connect to the object), and `PW_PERM_M` (metadata — read/write metadata for the object). [Source: PipeWire Session Manager docs](https://docs.pipewire.org/page_session_manager.html)

By default, WirePlumber grants ordinary clients a restrictive permission set. The `portal-permissionstore` plugin bridges PipeWire to the XDG Desktop Portal permission store: when a Flatpak application connects through the camera portal, WirePlumber checks the portal permission store entry for that application and grants `PW_PERM_R | PW_PERM_X` on camera nodes only if the user previously approved camera access. Ordinary sideloaded apps that connect directly to the PipeWire socket receive a *flat* permission set — full access to all graph objects — which is suitable for trusted system services but not for sandboxed applications.

```
graph TD
    A[Flatpak App] -->|D-Bus request| B[xdg-desktop-portal]
    B -->|permission check| C[portal permission store]
    C -->|approved| D[WirePlumber]
    D -->|grants PW_PERM_R+X on camera node| E[PipeWire daemon]
    A -->|PipeWire socket| E
    E -->|camera stream| A
```

The portal flow (section 4) extends this by providing a *scoped* PipeWire file descriptor that exposes only the specific screen-cast nodes authorised for the requesting application.

---

## 3. Video Capture Pipeline: V4L2 and libcamera Sources

### 3.1 The `spa-v4l2` Plugin

The `spa-v4l2` plugin (`spa/plugins/v4l2/`) wraps a V4L2 character device (`/dev/videoN`) as a PipeWire source node. WirePlumber loads it when the `api.v4l2.enum.udev` monitor discovers a V4L2 device, creating a `pw_node` with a single output port carrying video data. [Source](https://github.com/PipeWire/pipewire/blob/master/spa/plugins/v4l2/v4l2-utils.c)

The plugin calls `VIDIOC_ENUM_FMT` to discover all pixel formats the device supports, maps V4L2 fourcc codes to SPA format identifiers (e.g. `V4L2_PIX_FMT_YUYV` → `SPA_VIDEO_FORMAT_YUY2`, `V4L2_PIX_FMT_NV12` → `SPA_VIDEO_FORMAT_NV12`, `V4L2_PIX_FMT_MJPEG` → `SPA_VIDEO_FORMAT_ENCODED`), and exposes the complete capability set as `SPA_PARAM_EnumFormat` pods.

Buffer memory type negotiation distinguishes two paths:

- **`V4L2_MEMORY_MMAP`** (default for most UVC webcams): the kernel driver allocates buffers in a memory region mmapped into the PipeWire process. The SPA plugin wraps the mmapped pointer in an `spa_buffer` with `SPA_DATA_MemPtr` data type. Consumers cannot treat these as DMA-BUF handles.

- **`V4L2_MEMORY_DMABUF`**: the plugin exports each V4L2 buffer as a DMA-BUF file descriptor using `VIDIOC_EXPBUF`, then passes those fds as `SPA_DATA_DmaBuf` entries in the `spa_buffer`. This path is available on drivers that support it (most modern UVC, many platform camera drivers) and enables true zero-copy transfer to GPU encoders or display.

The plugin's format-negotiation code for the V4L2 DMABUF path is in `spa/plugins/v4l2/v4l2-utils.c`. When a consumer negotiates `SPA_DATA_DmaBuf`, the plugin sets `port->memtype = V4L2_MEMORY_DMABUF` in its stream setup and switches from `VIDIOC_DQBUF`-with-mmap to `VIDIOC_DQBUF`-with-exported fds.

A special case documented in the PipeWire DMA-BUF guide applies to V4L2 camera sources: unlike GPU-rendered buffers, V4L2 camera frames do not carry DRM tiling modifiers, because the kernel V4L2 API predates the DRM modifier concept. Both the plugin (producer) and the consumer advertise `SPA_DATA_DmaBuf` support via `SPA_PARAM_BUFFERS_dataType`, but neither announces modifiers. The consumer receives a linear DMA-BUF that is safe to `mmap()` and import into Vulkan with `VK_IMAGE_TILING_LINEAR`. [Source](https://eh5.pages.freedesktop.org/pipewire/page_dma_buf.html)

### 3.2 libcamera Integration

Modern embedded cameras (CSI-2 / MIPI) and complex USB cameras require per-lens configuration, auto-exposure pipelines, and ISP tuning that V4L2's ioctl interface cannot adequately express. libcamera provides a portable abstraction for these pipelines and is the primary interface for Raspberry Pi cameras, many ChromeOS sensors, and increasingly laptop webcams. [Source](https://blogs.gnome.org/uraeus/2021/10/01/pipewire-and-fixing-the-linux-video-capture-stack/)

PipeWire includes a `libcamera` SPA plugin (`spa/plugins/libcamera/`) that wraps a `libcamera::Camera` object as a PipeWire source node. The integration was merged into PipeWire mainline and is loaded by WirePlumber's `api.libcamera.enum.devices` monitor when libcamera is present. [Source](https://test.www.collabora.com/news-and-blog/blog/2020/09/11/integrating-libcamera-into-pipewire/)

The buffer lifecycle for libcamera frames is more complex than V4L2:

1. **Allocation**: libcamera allocates `FrameBuffer` objects through `libcamera::FrameBufferAllocator`. Each `FrameBuffer` consists of one or more `FrameBuffer::Plane` entries, each holding a DMA-BUF fd and an offset.
2. **Import into SPA**: The plugin wraps each `FrameBuffer` as an `spa_buffer`: `spa_data[0].type = SPA_DATA_DmaBuf`, `spa_data[0].fd = plane.fd`, `spa_data[0].mapoffset = plane.offset`, `spa_data[0].maxsize = plane.length`.
3. **Queueing**: The `libcamera::Request` is queued into the camera; on completion, the plugin's `requestComplete` callback fires. The plugin marks the corresponding `spa_buffer` as ready and signals the PipeWire data thread via eventfd.
4. **Reuse**: After downstream consumers finish with the buffer, PipeWire returns it via `port_reuse_buffer()`, and the plugin re-queues the `libcamera::Request`.

```mermaid
graph TD
    ALLOC["libcamera::FrameBufferAllocator\nallocates FrameBuffer\n(DMA-BUF fd + offset per plane)"] -- "wrap as spa_buffer\nSPA_DATA_DmaBuf" --> IMPORT["SPA plugin imports\nspa_data fd / mapoffset / maxsize"]
    IMPORT -- "libcamera::Request queued" --> QUEUE["Camera hardware\ncaptures frame"]
    QUEUE -- "requestComplete callback\nmarks spa_buffer ready\nsignals eventfd" --> READY["PipeWire data thread\ndelivers buffer to consumer"]
    READY -- "port_reuse_buffer()\nafter consumer finishes" --> QUEUE
```

libcamera-based devices do not expose raw V4L2 format codes; the SPA plugin translates `libcamera::PixelFormat` to SPA format identifiers. Multi-planar formats (NV12, YUV420) produce multi-element `spa_buffer` arrays.

### 3.3 Format Negotiation: YUYV, NV12, MJPEG

A typical UVC webcam advertises three main format families:

| V4L2 fourcc | SPA format constant | Notes |
|---|---|---|
| `YUYV` | `SPA_VIDEO_FORMAT_YUY2` | Packed 4:2:2, universal but bandwidth-heavy |
| `NV12` | `SPA_VIDEO_FORMAT_NV12` | Semi-planar 4:2:0; preferred by VA-API encoders |
| `MJPEG` | `SPA_VIDEO_FORMAT_ENCODED` | In-camera JPEG; low USB bandwidth, requires decode |
| `H264` | `SPA_VIDEO_FORMAT_ENCODED` | UVC H.264 capable cameras |

When both the camera (producer) and the consumer (e.g. a video-conferencing app) have negotiated a format, the adapter node inside PipeWire can perform lightweight conversion — for example, from `YUY2` to `BGRA` for applications that only accept packed RGBA. The `spa-videoconvert` plugin handles this, and the session manager selects it automatically when producer and consumer formats do not match directly.

---

## 4. Screen Capture and Portal Integration

### 4.1 The `org.freedesktop.portal.ScreenCast` Interface

The XDG Desktop Portal defines `org.freedesktop.portal.ScreenCast` as the D-Bus interface through which sandboxed applications (Flatpaks, Snaps, or any browser) request Wayland screen capture. The portal enforces user consent: no application can capture screen content without the compositor presenting a picker dialog. [Source](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html)

The interface (version 6 as of 2025) exposes the following methods:

- **`CreateSession`** — creates a session object; returns a session handle.
- **`SelectSources`** — configures which content to capture: `types` bitmask (MONITOR=1, WINDOW=2, VIRTUAL=4), `cursor_mode` (Hidden/Embedded/Metadata), `persist_mode` for token-based session restoration, and `multiple` to allow multi-monitor capture.
- **`Start`** — triggers the user picker dialog; on approval returns an array of stream descriptors, each carrying `pipewire-serial` (the stable 64-bit object serial of the PipeWire stream node), `source_type`, and spatial metadata.
- **`OpenPipeWireRemote`** — returns an `h` (UNIX fd) pointing to a PipeWire socket. The file descriptor connects to a scoped PipeWire context that exposes *only* the stream nodes authorised for this session. Applications use this fd with `pw_context_connect_fd()` to create a `pw_core` and then connect a `pw_stream` using the `pipewire-serial` as the `PW_KEY_TARGET_OBJECT`.

```
graph LR
    App -->|D-Bus CreateSession| Portal[xdg-desktop-portal]
    Portal -->|compositor portal backend| Compositor
    Compositor -->|user picks source| User
    User -->|approves| Compositor
    Compositor -->|creates pw_node| PW[PipeWire daemon]
    Portal -->|OpenPipeWireRemote fd| App
    App -->|pw_context_connect_fd| PW
    App -->|pw_stream_connect| PW
    PW -->|DMA-BUF frames| App
```

### 4.2 Compositor Portal Backends

The `xdg-desktop-portal` process forwards `ScreenCast` requests to a *backend* that implements `org.freedesktop.impl.portal.ScreenCast` — a private D-Bus interface answered by compositor-side code:

- **`xdg-desktop-portal-gnome`** (`xdpp-gnome`): integrated with GNOME Shell (mutter). Mutter implements the backend using a KMS CRTC readback or, for window capture, the `wlr-screencopy-unstable-v1`-compatible internal API. The captured frames are placed into DMA-BUF buffers and passed to a PipeWire source node. Full DMA-BUF modifier negotiation was implemented here to support NVIDIA's proprietary tiling formats. [Source](https://botmonster.com/self-hosting/wayland-screen-sharing-fix-video-calls-linux/)

- **`xdg-desktop-portal-kde`** (KDE Plasma / KWin): implemented in KWin directly. KWin uses its own KMS/DRM infrastructure to read back screen content into a PipeWire node. Supports monitor and window capture.

- **`xdg-desktop-portal-wlr`** (`xdpw`): for all wlroots-based compositors (Sway, Hyprland, river, niri). Implements capture using the `wlr-screencopy-unstable-v1` Wayland protocol, which allows a privileged client to receive compositor-rendered frames. The portal allocates GBM buffers using `gbm_bo_create_with_modifiers2()`, performs a test allocation to validate modifier compatibility, and offers the resulting DMA-BUF buffers to PipeWire. If modifier negotiation fails, it falls back to shared-memory (`SPA_DATA_MemFd`). [Source: `xdg-desktop-portal-wlr/src/screencast/pipewire_screencast.c`](https://github.com/emersion/xdg-desktop-portal-wlr/blob/master/src/screencast/pipewire_screencast.c)

```mermaid
graph TD
    PORTAL["xdg-desktop-portal\norg.freedesktop.portal.ScreenCast"] -- "org.freedesktop.impl.portal.ScreenCast\n(D-Bus backend)" --> GNOME["xdg-desktop-portal-gnome\n(mutter / GNOME Shell)"]
    PORTAL -- "org.freedesktop.impl.portal.ScreenCast\n(D-Bus backend)" --> KDE["xdg-desktop-portal-kde\n(KWin / KDE Plasma)"]
    PORTAL -- "org.freedesktop.impl.portal.ScreenCast\n(D-Bus backend)" --> WLR["xdg-desktop-portal-wlr\n(wlroots compositors)"]
    GNOME -- "KMS CRTC readback\nDMA-BUF modifier negotiation" --> PW["PipeWire source node\n(pw_node, Video/Source)"]
    KDE -- "KMS/DRM readback" --> PW
    WLR -- "wlr-screencopy-unstable-v1\ngbm_bo_create_with_modifiers2" --> PW
```

The broader Wayland ecosystem is moving toward the standardised `ext-image-capture-source-v1` and `ext-image-copy-capture-v1` protocols, which will replace the wlroots-specific screencopy protocol across compositors. Support landed in wlroots and Sway in 2024–2025; hyprland-specific portal (`xdg-desktop-portal-hyprland`) already implements this newer path.

### 4.3 The Screen-Cast `pw_stream` on the Consumer Side

The screen-cast `pw_stream` returned by the portal carries DMA-BUF frames from the compositor. Setting up a consumer involves:

```c
/* Illustrative — simplified from xdg-desktop-portal consumer pattern */

/* 1. Connect to the scoped PipeWire remote */
struct pw_core *core = pw_context_connect_fd(ctx, portal_pipewire_fd,
                                             NULL, 0);

/* 2. Create a capture stream */
struct pw_stream *stream = pw_stream_new(core, "screen-capture",
    pw_properties_new(PW_KEY_MEDIA_TYPE,     "Video",
                      PW_KEY_MEDIA_CATEGORY, "Capture",
                      PW_KEY_TARGET_OBJECT,  pipewire_serial_str,
                      NULL));

/* 3. Announce DMA-BUF capability: offer both DMA-BUF and MemFd */
/* (format pod construction omitted for brevity; see section 6) */
pw_stream_connect(stream, PW_DIRECTION_INPUT,
                  PW_ID_ANY,
                  PW_STREAM_FLAG_AUTOCONNECT |
                  PW_STREAM_FLAG_MAP_BUFFERS,
                  params, n_params);

/* 4. In on_process callback: */
static void on_process(void *data) {
    struct pw_buffer *b = pw_stream_dequeue_buffer(stream);
    struct spa_buffer *buf = b->buffer;
    if (buf->datas[0].type == SPA_DATA_DmaBuf) {
        int dma_fd = buf->datas[0].fd;
        /* import into EGL/Vulkan for zero-copy rendering */
    }
    pw_stream_queue_buffer(stream, b);
}
```

The `SPA_META_Header` metadata block on each buffer carries a `pts` field (presentation timestamp in nanoseconds relative to `CLOCK_MONOTONIC`) and a `seq` sequence number. `SPA_META_VideoDamage` carries up to 16 changed rectangles, allowing consumers to perform partial updates rather than full-frame processing. `SPA_META_VideoTransform` signals 90/180/270 degree rotations (relevant for phone-orientation screen sharing).

### 4.4 OBS Studio and WebRTC Consumers

**OBS Studio** implements PipeWire screen capture via its `pipewire` plugin (`plugins/linux-pipewire/`). It calls `org.freedesktop.portal.ScreenCast` internally (through `libportal` or direct D-Bus), handles the `pw_stream` setup, and then feeds frames into OBS's internal buffer pipeline for encoding and streaming.

**Chromium / Chrome** has supported PipeWire-based screen capture since Chrome 110 (2023), where it was enabled by default on Wayland. When running under Wayland and a WebRTC `getDisplayMedia()` call is made, Chrome invokes the portal D-Bus interface, receives the PipeWire fd, and reads frames through its `webrtc_desktop_capture` `PipeWireCapturer` implementation. Mozilla Firefox uses a similar path via its own portal-aware screen-capture code. [Source: Screen Sharing on Wayland research](https://botmonster.com/self-hosting/wayland-screen-sharing-fix-video-calls-linux/)

---

## 5. Remote Desktop and Streaming

### 5.1 `org.freedesktop.portal.RemoteDesktop`

The `org.freedesktop.portal.RemoteDesktop` D-Bus interface extends the ScreenCast flow with input injection capabilities. A remote desktop session is initiated alongside (or independently from) a ScreenCast session: [Source](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html)

- **`CreateSession`** — creates a remote desktop session handle.
- **`SelectDevices`** — specifies which input device categories to control via bitmask: `KEYBOARD=1`, `POINTER=2`, `TOUCHSCREEN=4`.
- **`Start`** — activates the session; in practice, user consent is shown only for the screen-content portion, not for input injection when both are requested together in the same session.
- **Input injection methods**: `NotifyPointerMotion` (relative delta), `NotifyPointerMotionAbsolute` (position in stream logical coordinates), `NotifyPointerButton` (Linux evdev button codes), `NotifyPointerAxis` (scroll events), `NotifyKeyboardKeycode`, `NotifyKeyboardKeysym`, and touch events.
- **`ConnectToEIS`** — returns an fd for a `libei` (Event Input System) connection. Once an EIS connection is established, input must be routed exclusively through EIS rather than through the individual `Notify*` D-Bus methods. The `mapping_id` field in ScreenCast stream properties correlates libei device regions to specific streams for absolute pointer positioning.

ScreenCast and RemoteDesktop sessions are designed to be combined: a client calls `org.freedesktop.portal.ScreenCast.SelectSources` and `OpenPipeWireRemote` on the same session object to receive both screen content and input access.

### 5.2 GNOME Remote Desktop and FreeRDP

GNOME's `gnome-remote-desktop` daemon implements an RDP server for sharing the GNOME desktop with Windows / macOS RDP clients. Internally, it uses PipeWire as the video source: mutter (the GNOME compositor) creates a screen-cast PipeWire node, and gnome-remote-desktop reads that stream, encodes it, and forwards the encoded data to FreeRDP for RDP client delivery.

A zero-copy rendering path was implemented (merged via GitLab MR !294) that threads DMA-BUF frames from the PipeWire source through a Vulkan import step and into VA-API hardware AVC (H.264) encoding, without any CPU blit: [Source](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/294)

1. PipeWire `on_process` callback dequeues a `pw_buffer` containing a compositor DMA-BUF.
2. The buffer's fd is imported into a Vulkan `VkImage` via `VkExternalMemoryImageCreateInfo` / `VkImportMemoryFdInfoKHR`.
3. Colour conversion and damage region compositing run in Vulkan compute shaders.
4. The Vulkan-rendered output is exported as a VA-API `VASurface` via the `VK_EXT_external_memory_dma_buf` / `VK_EXT_image_drm_format_modifier` chain.
5. `vaBeginPicture` / `vaRenderPicture` / `vaEndPicture` encode the surface to H.264 using the GPU's dedicated media engine.
6. The compressed bitstream is passed to FreeRDP for packetisation and transmission to the RDP client.

```mermaid
graph TD
    PW2["PipeWire on_process\ndequeues pw_buffer\n(compositor DMA-BUF)"] -- "DMA-BUF fd" --> VK["Vulkan VkImage\nVkExternalMemoryImageCreateInfo\nVkImportMemoryFdInfoKHR"]
    VK -- "Vulkan compute shaders\ncolour conversion + damage compositing" --> VKOUT["Vulkan-rendered output"]
    VKOUT -- "VK_EXT_external_memory_dma_buf\nVK_EXT_image_drm_format_modifier" --> VASURF["VA-API VASurface"]
    VASURF -- "vaBeginPicture\nvaRenderPicture\nvaEndPicture" --> H264["H.264 compressed bitstream\n(GPU media engine)"]
    H264 -- "packetisation" --> FREERDP["FreeRDP\nRDP client delivery"]
```

This pipeline makes large-display remote desktop (4K, 8K) practical: the CPU touches only the compressed bitstream, never the raw frame data.

### 5.3 Hardware Encode via VA-API and V4L2 Stateless

PipeWire's capture node can be connected to hardware encode without any gnome-remote-desktop involvement. The GStreamer `pipewiresrc` element provides a standard `GstElement` source backed by a PipeWire stream, enabling arbitrary GStreamer pipelines to consume PipeWire nodes:

```bash
# Capture PipeWire node ID 43 and encode to H.264 using VA-API
gst-launch-1.0 -e \
  pipewiresrc path=43 do-timestamp=true ! \
  vaapipostproc ! queue ! \
  vaapih264enc ! h264parse ! \
  matroskamux ! filesink location=/tmp/screencast.mkv
```

The `path` parameter is the PipeWire node object serial (returned by the portal's `pipewire-serial` field). `vaapipostproc` handles colour-space conversion from whatever format the PipeWire node emits to the NV12 or BGRA that `vaapih264enc` expects. When the PipeWire node provides a DMA-BUF, VA-API can import it via `vaCreateBuffer` with `VAExternalMemoryDRMPrimeFD`, avoiding any CPU copy. [Source: GStreamer discourse, H264 PipeWire issues](https://discourse.gstreamer.org/t/h264-pipewire-issues/4764)

For V4L2 stateless encode (common on ARM SoCs and Raspberry Pi), the equivalent path imports the DMA-BUF fd into a V4L2 `CAPTURE` queue buffer using `V4L2_MEMORY_DMABUF`, feeding it directly to the codec's DMA engine.

```
graph TD
    PW[PipeWire screen-cast node] -->|DMA-BUF fd| GST[pipewiresrc GstElement]
    GST --> PP[vaapipostproc NV12 conversion]
    PP --> ENC[vaapih264enc VA-API encode]
    ENC --> H264[H.264 bitstream]
    H264 --> MKV[matroskamux / filesink]
```

### 5.4 GStreamer `pipewiresrc` as a Generic Consumer

PipeWire ships two GStreamer elements as part of `gst-plugin-pipewire`:

- **`pipewiresrc`**: implements `GstPushSrc`, creates a PipeWire `pw_stream` in input direction, and pushes `GstBuffer` objects wrapping DMA-BUF fds (via `gst_dmabuf_allocator_alloc`) or falls back to `GstMapInfo`-accessible memory.
- **`pipewiresink`**: implements `GstBaseSink`, creates a PipeWire `pw_stream` in output direction, pushing application-produced buffers into the PipeWire graph.

Format translation between GStreamer caps (`video/x-raw,format=NV12,width=1920,...`) and SPA pods is handled inside `gst-plugin-pipewire` by the `gst_caps_to_format` family of helper functions in `gst/pipewire/gstpipewiresrc.c`. DMA-BUF buffers are identified by the GStreamer feature `memory:DMABuf` on the caps.

---

## 6. Buffer Formats and GPU Interop

### 6.1 Exporting a Rendered Frame into PipeWire

A Vulkan or EGL application that produces video frames on the GPU can inject them into PipeWire as a producer (output) `pw_stream` with `SPA_DATA_DmaBuf` buffers. This enables a rendered scene to flow through the PipeWire graph to recorders, encoders, or virtual camera outputs without any GPU→CPU readback. [Source](https://docs.pipewire.org/page_dma_buf.html)

The output stream is created with `PW_DIRECTION_OUTPUT` and `media.class = "Video/Source"`. During format negotiation, the application must offer its DMA-BUF capabilities via `SPA_PARAM_EnumFormat` pods. According to the PipeWire DMA-BUF documentation, a producer must offer *two* format parameter objects per pixel format: one that includes `SPA_FORMAT_VIDEO_modifier` (for modifier-aware consumers) and one without (as a fallback): [Source: PipeWire DMA-BUF Sharing guide](https://eh5.pages.freedesktop.org/pipewire/page_dma_buf.html)

```c
/* Simplified modifier-aware format announcement (illustrative).
 * Real upstream examples: spa/examples/video-src-fixate.c */
static void build_format_with_modifiers(struct spa_pod_builder *b,
                                        uint32_t drm_format,
                                        uint64_t *modifiers, int n_mods)
{
    struct spa_pod_frame f[2];
    spa_pod_builder_push_object(b, &f[0], SPA_TYPE_OBJECT_Format,
                                SPA_PARAM_EnumFormat);
    spa_pod_builder_add(b,
        SPA_FORMAT_mediaType,    SPA_POD_Id(SPA_MEDIA_TYPE_video),
        SPA_FORMAT_mediaSubtype, SPA_POD_Id(SPA_MEDIA_SUBTYPE_raw),
        SPA_FORMAT_VIDEO_format, SPA_POD_Id(drm_format),
        0);
    /* Modifier choice: SPA_POD_PROP_FLAG_DONT_FIXATE lets the consumer
     * narrow the choice further before we commit. */
    spa_pod_builder_prop(b, SPA_FORMAT_VIDEO_modifier,
        SPA_POD_PROP_FLAG_MANDATORY | SPA_POD_PROP_FLAG_DONT_FIXATE);
    spa_pod_builder_push_choice(b, &f[1], SPA_CHOICE_Enum, 0);
    /* First entry is the preferred value */
    spa_pod_builder_long(b, modifiers[0]);
    for (int i = 0; i < n_mods; i++)
        spa_pod_builder_long(b, modifiers[i]);
    /* DRM_FORMAT_MOD_INVALID signals modifier-less DMA-BUF support */
    spa_pod_builder_long(b, DRM_FORMAT_MOD_INVALID);
    spa_pod_builder_pop(b, &f[1]);
    spa_pod_builder_pop(b, &f[0]);
}
```

### 6.2 The `spa_video_info_raw` Modifier Field

Once the consumer has selected a format, `param_changed` fires on the producer with `SPA_PARAM_Format`. The producer parses the negotiated pod:

```c
static void on_param_changed(void *data, uint32_t id,
                              const struct spa_pod *param)
{
    if (id != SPA_PARAM_Format || param == NULL) return;

    struct spa_video_info_raw info;
    spa_format_video_raw_parse(param, &info);
    /* info.format   = negotiated pixel format (e.g. SPA_VIDEO_FORMAT_NV12) */
    /* info.modifier = negotiated DRM modifier, or 0 if not negotiated       */

    /* Check whether fixation is still needed */
    const struct spa_pod_prop *mod_prop =
        spa_pod_find_prop(param, NULL, SPA_FORMAT_VIDEO_modifier);
    if (mod_prop &&
        (mod_prop->flags & SPA_POD_PROP_FLAG_DONT_FIXATE)) {
        /* Consumer wants us to fixate: select one modifier,
         * test-allocate, then call pw_stream_update_params() again
         * with a single-modifier EnumFormat pod (without DONT_FIXATE). */
        fixate_modifier(data, &info);
    } else {
        /* Modifier is fixed; allocate DMA-BUF with info.modifier */
        allocate_dma_buf(data, &info);
    }
}
```

The `modifier` field in `spa_video_info_raw` is the primary carrier of DRM format modifier information. It is valid only when the negotiated buffer type is `SPA_DATA_DmaBuf`; for shared-memory buffers the field is zero.

### 6.3 DRM Format and Modifier Negotiation

DRM format modifiers encode tiling and compression layout information as a 64-bit value. Examples relevant to screen capture:

| Modifier constant | Meaning |
|---|---|
| `DRM_FORMAT_MOD_LINEAR` | Pitch-linear layout; GPU-portable, safe to mmap |
| `I915_FORMAT_MOD_X_TILED` | Intel X-tiled layout; efficient for render targets |
| `I915_FORMAT_MOD_Y_TILED_CCS` | Intel Y-tiled + colour-compression (Xe1/Gen9+) |
| `AMD_FMT_MOD_*` | AMDGPU display-compression variants (DCN) |
| `DRM_FORMAT_MOD_INVALID` | Signal that modifier-less (linear) DMA-BUF is accepted |

A producer that renders into a GPU framebuffer typically creates the allocation with `gbm_bo_create_with_modifiers2()` or via Vulkan's `VkImageDrmFormatModifierExplicitCreateInfoEXT`. It then exports the DMA-BUF fd with `gbm_bo_get_fd()` or `vkGetMemoryFdKHR()` and announces the modifier obtained from `gbm_bo_get_modifier()` or `VkImageDrmFormatModifierPropertiesEXT`.

A consumer (such as the VA-API encoder or a compositor) imports the DMA-BUF using the modifier as a hint for its memory-access engine. If the consumer does not support the modifier, the negotiation phase eliminates it from consideration and falls back to `DRM_FORMAT_MOD_LINEAR` or, ultimately, to `SPA_DATA_MemFd`.

### 6.4 Zero-Copy Round-Trip vs CPU Fallback

The complete zero-copy path from GPU render to encode/stream is:

```
graph LR
    GPU[Vulkan render pass] -->|VkImage DMA-BUF export| DMABUF[DMA-BUF fd + modifier]
    DMABUF -->|SPA_DATA_DmaBuf| PW[PipeWire pw_stream]
    PW -->|portal ScreenCast stream| Portal[xdg-desktop-portal]
    Portal -->|DMA-BUF fd| OBS[OBS Studio / WebRTC]
    OBS -->|vaCreateBuffer VA_SURFACE| VAAPI[VA-API encode engine]
    VAAPI -->|H.264/HEVC bitstream| Net[network / file]
```

The zero-copy path degrades gracefully when modifier negotiation fails at any step:

1. **Producer cannot export the consumer's required modifier** → producer re-negotiates with `DRM_FORMAT_MOD_LINEAR` or drops to shared memory.
2. **Consumer cannot import the DMA-BUF at all** → SPA negotiation selects `SPA_DATA_MemFd`; PipeWire performs a CPU copy from the DMA-BUF (via mmap) into the shared memory region.
3. **Modifier-less V4L2 DMA-BUF is received by a GPU consumer** → the consumer imports the fd with `VK_IMAGE_TILING_LINEAR`; it can read the data but cannot apply hardware decompression.

The `on_add_buffer` callback in the consumer indicates which memory type was actually negotiated:

```c
static void on_add_buffer(void *data, struct pw_buffer *buf) {
    struct spa_buffer *sb = buf->buffer;
    if (sb->datas[0].type == SPA_DATA_DmaBuf) {
        /* Zero-copy path: import dma_fd into Vulkan/EGL/VA-API */
        int dma_fd = sb->datas[0].fd;
        import_gpu_buffer(dma_fd, info.modifier, info.size);
    } else {
        /* Fallback: sb->datas[0].data is a CPU-accessible pointer */
        /* (mapped by PipeWire when PW_STREAM_FLAG_MAP_BUFFERS is set) */
    }
}
```

### 6.5 Practical GPU-to-PipeWire Injection

The `pw-capture` project (archived 2025) demonstrated the Vulkan-layer approach: a `VkLayer` intercepts `vkQueuePresentKHR`, exports the swapchain image as a DMA-BUF via `VkExternalMemoryImageCreateInfo` with `VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT`, and pushes the fd into a `pw_stream` with `SPA_DATA_DmaBuf`. The consumer sees it as a standard PipeWire `Video/Source` node. [Source](https://github.com/EHfive/pw-capture)

From PipeWire 0.3.80 onwards, the built-in Vulkan SPA plugin (`spa/plugins/vulkan/`) also supports DMA-BUF, enabling PipeWire itself to run Vulkan compute shaders on frames passing through the graph — for example, deinterlacing or colour-correction filters applied in the data thread without any CPU involvement. [Source](https://www.phoronix.com/news/PipeWire-0.3.80)

---

## Integrations

- **Chapter 4 — DMA-BUF Kernel Subsystem**: The `SPA_DATA_DmaBuf` type, modifier propagation, and the `VIDIOC_EXPBUF` path in the V4L2 SPA plugin all rely on the kernel `dma_buf` framework covered there. The modifier constants (`DRM_FORMAT_MOD_LINEAR`, `I915_FORMAT_MOD_Y_TILED_CCS`, etc.) are defined in `include/uapi/drm/drm_fourcc.h`.

- **Chapter 20 — Wayland Protocols: Screen Capture**: The `wlr-screencopy-unstable-v1` and `ext-image-copy-capture-v1` protocols are the compositor-side mechanisms that `xdg-desktop-portal-wlr` and `xdg-desktop-portal-hyprland` use to pull frames from the compositor into PipeWire. The protocol semantics (buffer format negotiation, damage tracking) are covered in depth in Chapter 20.

- **Chapter 22 — Production Compositors**: Mutter (GNOME), KWin (KDE), and wlroots implement the `org.freedesktop.impl.portal.ScreenCast` D-Bus backend. Their KMS/DRM readback paths and GBM buffer allocation strategies are covered in Chapter 22.

- **Chapter 26 — VA-API and Hardware Video Decode/Encode**: Section 5.2 of this chapter describes the zero-copy chain from PipeWire DMA-BUF to VA-API encode. Chapter 26 covers the VA-API surface lifecycle, `vaCreateBuffer(VAExternalMemoryDRMPrimeFD)`, and the relationship between VASurfaces and DRM objects.

- **Chapter 26 — V4L2**: The V4L2 kernel API, `VIDIOC_DQBUF`, `VIDIOC_EXPBUF`, `V4L2_MEMORY_DMABUF`, and the stateless codec interface used in section 5.3 are covered in detail in the V4L2 section of Chapter 26.

- **Chapter 39 — Qt and GTK Screen Capture APIs**: Qt's `QMediaCaptureSession` and GTK's `gtk_media_stream` both use portal APIs under the hood on Wayland. The portal flow described in section 4 is the shared substrate; Chapter 39 covers the toolkit-level wrapper abstractions.

- **Appendix K — Remote Display and Streaming**: The GNOME Remote Desktop / FreeRDP integration described in section 5.2, and the broader landscape of VNC, RDP, and WebRTC streaming on Linux, is catalogued in Appendix K.

---

## References

1. **PipeWire project documentation** — [https://docs.pipewire.org/](https://docs.pipewire.org/)
2. **PipeWire source repository** (GitLab mirror) — [https://github.com/PipeWire/pipewire](https://github.com/PipeWire/pipewire)
3. **PipeWire DMA-BUF Sharing guide** — [https://eh5.pages.freedesktop.org/pipewire/page_dma_buf.html](https://eh5.pages.freedesktop.org/pipewire/page_dma_buf.html)
4. **`spa_video_info_raw` Struct Reference** — [https://docs.pipewire.org/structspa__video__info__raw.html](https://docs.pipewire.org/structspa__video__info__raw.html)
5. **PipeWire `pw_stream` API** — [https://docs.pipewire.org/page_streams.html](https://docs.pipewire.org/page_streams.html)
6. **PipeWire SPA Plugins overview** — [https://docs.pipewire.org/page_spa_plugins.html](https://docs.pipewire.org/page_spa_plugins.html)
7. **PipeWire Graph Scheduling** — [https://docs.pipewire.org/page_scheduling.html](https://docs.pipewire.org/page_scheduling.html)
8. **PipeWire `pipewire.conf` man page** — [https://docs.pipewire.org/page_man_pipewire_conf_5.html](https://docs.pipewire.org/page_man_pipewire_conf_5.html)
9. **PipeWire RT module** — [https://docs.pipewire.org/page_module_rt.html](https://docs.pipewire.org/page_module_rt.html)
10. **PipeWire Session Manager overview** — [https://docs.pipewire.org/page_session_manager.html](https://docs.pipewire.org/page_session_manager.html)
11. **PipeWire Video Capture Tutorial (Part 5)** — [https://docs.pipewire.org/page_tutorial5.html](https://docs.pipewire.org/page_tutorial5.html)
12. **`spa_node` interface reference** — [https://docs.pipewire.org/devel/group__spa__node.html](https://docs.pipewire.org/devel/group__spa__node.html)
13. **WirePlumber documentation** — [https://pipewire.pages.freedesktop.org/wireplumber/](https://pipewire.pages.freedesktop.org/wireplumber/)
14. **WirePlumber linking policy** — [https://pipewire.pages.freedesktop.org/wireplumber/policies/linking.html](https://pipewire.pages.freedesktop.org/wireplumber/policies/linking.html)
15. **WirePlumber Event Dispatcher (Collabora blog)** — [https://www.collabora.com/news-and-blog/blog/2023/10/30/wireplumber-exploring-lua-scripts-with-event-dispatcher/](https://www.collabora.com/news-and-blog/blog/2023/10/30/wireplumber-exploring-lua-scripts-with-event-dispatcher/)
16. **WirePlumber configuration for embedded (Kiagiadakis, 2025)** — [https://gkiagia.gr/2025-04-22-wireplumber-configuration-on-embedded/](https://gkiagia.gr/2025-04-22-wireplumber-configuration-on-embedded/)
17. **`org.freedesktop.portal.ScreenCast` D-Bus reference** — [https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html)
18. **`org.freedesktop.portal.RemoteDesktop` D-Bus reference** — [https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop.html)
19. **`xdg-desktop-portal-wlr` screencasting source** — [`src/screencast/pipewire_screencast.c`](https://github.com/emersion/xdg-desktop-portal-wlr/blob/master/src/screencast/pipewire_screencast.c)
20. **GNOME Remote Desktop zero-copy rendering MR** — [https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/294](https://gitlab.gnome.org/GNOME/gnome-remote-desktop/-/merge_requests/294)
21. **PipeWire and fixing the Linux Video Capture Stack (Schaller/GNOME blog)** — [https://blogs.gnome.org/uraeus/2021/10/01/pipewire-and-fixing-the-linux-video-capture-stack/](https://blogs.gnome.org/uraeus/2021/10/01/pipewire-and-fixing-the-linux-video-capture-stack/)
22. **Integrating libcamera into PipeWire (Collabora, 2020)** — [https://test.www.collabora.com/news-and-blog/blog/2020/09/11/integrating-libcamera-into-pipewire/](https://test.www.collabora.com/news-and-blog/blog/2020/09/11/integrating-libcamera-into-pipewire/)
23. **PipeWire 0.3.80 Vulkan DMA-BUF support (Phoronix)** — [https://www.phoronix.com/news/PipeWire-0.3.80](https://www.phoronix.com/news/PipeWire-0.3.80)
24. **`pw-capture` Vulkan/OpenGL PipeWire capture** (archived) — [https://github.com/EHfive/pw-capture](https://github.com/EHfive/pw-capture)
25. **An Introduction to PipeWire (Bootlin)** — [https://bootlin.com/blog/an-introduction-to-pipewire/](https://bootlin.com/blog/an-introduction-to-pipewire/)
26. **GStreamer + PipeWire todo list (Raghavan, 2024)** — [https://arunraghavan.net/2024/12/gstreamer-pipewire-a-todo-list/](https://arunraghavan.net/2024/12/gstreamer-pipewire-a-todo-list/)
27. **V4L2 SPA plugin source** — [`spa/plugins/v4l2/v4l2-utils.c`](https://github.com/PipeWire/pipewire/blob/master/spa/plugins/v4l2/v4l2-utils.c)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
