# Chapter 145: XWayland: Architecture and the X11-to-Wayland Bridge

This chapter targets **systems and driver developers** who need to understand how XWayland bridges the X11 and Wayland worlds at the protocol and implementation level, and **application developers** debugging X11 application compatibility, scaling, clipboard, or performance issues on a Wayland compositor. It traces the complete data path from an X11 client issuing a drawing call to the frame arriving at the Wayland compositor, and examines the subtleties that make that path work—and the edge cases where it fails.

---

## Table of Contents

- [XWayland's Role and Overall Architecture](#xwaylands-role-and-overall-architecture)
  - [1.1 What is XWayland?](#11-what-is-xwayland)
  - [1.2 What is X11?](#12-what-is-x11)
  - [1.3 What is Wayland?](#13-what-is-wayland)
  - [1.4 What is DRI3?](#14-what-is-dri3)
- [Process Model: Launching XWayland](#process-model-launching-xwayland)
  - [Compositor-Managed Launch](#compositor-managed-launch)
  - [Rootless vs. Rootful Mode](#rootless-vs-rootful-mode)
- [Wayland Client Side: Mapping X11 Windows to Wayland Surfaces](#wayland-client-side-mapping-x11-windows-to-wayland-surfaces)
  - [xwayland-shell-v1 and the WL_SURFACE_ID Race](#xwayland-shell-v1-and-the-wl_surface_id-race)
  - [Override-Redirect Windows and Popup Mapping](#override-redirect-windows-and-popup-mapping)
- [DRI3, GLAMOR, and Buffer Sharing](#dri3-glamor-and-buffer-sharing)
  - [GBM Buffer Allocation Pipeline](#gbm-buffer-allocation-pipeline)
  - [PRESENT Extension: Flip vs. Blit](#present-extension-flip-vs-blit)
  - [Software Fallback: The SHM Path](#software-fallback-the-shm-path)
- [GLX on XWayland](#glx-on-xwayland)
- [Explicit Synchronisation: linux-drm-syncobj-v1](#explicit-synchronisation-linux-drm-syncobj-v1)
- [Input Handling: Translating Wayland Events to X11](#input-handling-translating-wayland-events-to-x11)
  - [Keyboard: XKB Map Synchronisation and Keycode Offset](#keyboard-xkb-map-synchronisation-and-keycode-offset)
  - [Pointer: Absolute, Relative, and Grab Emulation](#pointer-absolute-relative-and-grab-emulation)
  - [Touch and Tablet Input](#touch-and-tablet-input)
- [Clipboard and Selections](#clipboard-and-selections)
  - [X11 Selections and Wayland Data Devices](#x11-selections-and-wayland-data-devices)
  - [INCR Protocol for Large Payloads](#incr-protocol-for-large-payloads)
  - [Drag-and-Drop: XDND to wl_data_offer](#drag-and-drop-xdnd-to-wl_data_offer)
- [Rootless Mode Internals: Window Properties and Compositor Integration](#rootless-mode-internals-window-properties-and-compositor-integration)
- [Common XWayland Issues and Workarounds](#common-xwayland-issues-and-workarounds)
  - [Screen Tearing and the DRI3 vs. SHM Path](#screen-tearing-and-the-dri3-vs-shm-path)
  - [HiDPI and Fractional Scaling](#hidpi-and-fractional-scaling)
  - [Cursor Scaling](#cursor-scaling)
  - [Anti-Cheat and EAC/BattleEye](#anti-cheat-and-eacbattleye)
- [XWayland vs. Rootful Xwayland](#xwayland-vs-rootful-xwayland)
- [Debugging XWayland](#debugging-xwayland)
- [xwayland-satellite: Decoupling XWayland from the Compositor](#xwayland-satellite-decoupling-xwayland-from-the-compositor)
- [Strategic Outlook: Will XWayland Ever Be Retired?](#strategic-outlook-will-xwayland-ever-be-retired)
- [Roadmap](#roadmap)
- [Integrations](#integrations)

---

## XWayland's Role and Overall Architecture

When the Linux desktop transitioned from Xorg to Wayland compositors, an enormous body of X11 software—games, productivity tools, SDKs, Java toolkits, Wine—remained written to the X11 protocol. XWayland is the bridge that makes this software work unmodified on a Wayland compositor.

XWayland's architecture is unusual: it is simultaneously an **X11 server** and a **Wayland client**. It accepts X11 protocol connections from legacy X11 applications on a conventional Unix domain socket (e.g., `/tmp/.X11-unix/X1`), implementing the full X11 drawing protocol. At the same time, it connects to the running Wayland compositor as a Wayland client, presenting X11 window content as `wl_surface` objects backed by DMA-BUF shared GPU memory. The Wayland compositor handles compositing, display output, and user input — XWayland simply translates between the two worlds.

The XWayland DDX (Device Dependent X) lives in `hw/xwayland/` in the xorg-server source tree. Starting with the xwayland 22.1 release in 2022, it was split into a [standalone repository](https://gitlab.freedesktop.org/xorg/xwayland) and package, separating its release cycle from the Xorg server. The core source files are:

| Source File | Role |
|---|---|
| `hw/xwayland/xwayland.c` | Entry points, `xwl_screen` and `xwl_window` core structs, initialisation |
| `hw/xwayland/xwayland-window.c` | `xwl_realize_window()`, `xwl_unrealize_window()`, `xwl_window_post_damage()` |
| `hw/xwayland/xwayland-output.c` | Wayland output handling, RandR emulation, xdg-output integration |
| `hw/xwayland/xwayland-input.c` | Full seat/pointer/keyboard/touch/tablet pipeline |
| `hw/xwayland/xwayland-glamor.c` | GLAMOR `egl_backend` interface; `post_damage` hook |
| `hw/xwayland/xwayland-glamor-gbm.c` | GBM buffer allocation, DRI3 open, DMA-BUF export |
| `hw/xwayland/xwayland-present.c` | PRESENT extension; flip vs. blit path; vsync submission |
| `hw/xwayland/xwayland-shm.c` | Software (SHM) fallback rendering path |

[Source: xorg/xwayland, GitLab](https://gitlab.freedesktop.org/xorg/xserver/-/tree/master/hw/xwayland)

A critical architectural subtlety: in rootless mode, the Wayland compositor is simultaneously the **Wayland server** to XWayland (which presents surfaces as a Wayland client) and an **X11 client** of XWayland (the XWM component of the compositor uses the X11 protocol to manage windows). This bidirectional dependency requires that all X11 communications from the compositor's XWM be **asynchronous** to prevent deadlock. If the compositor makes a synchronous X11 call while XWayland is blocked awaiting a Wayland reply from the compositor, the system deadlocks. This design constraint shapes the entire implementation.

The compositor's X11 window manager (XWM) communicates with XWayland over a dedicated `xcb_connection_t`. It receives X11 events such as `MapRequest`, `ConfigureRequest`, and `ClientMessage`; reads window properties; and sends X11 protocol requests to position, resize, or configure windows. Simultaneously, the same compositor process is receiving Wayland events from XWayland (as a Wayland client) via the Wayland socket. Both channels must be serviced without blocking either. Compositors implement this with a two-event-loop architecture or by integrating both fd poll descriptors into a single `wl_event_loop`. [Source: Wayland Freedesktop XWayland documentation](https://wayland.freedesktop.org/docs/book/Xwayland.html)

### 1.1 What is XWayland?

XWayland is a compatibility server that allows unmodified X11 applications to run on Wayland compositors. It occupies a dual role: it acts as an X11 server for legacy applications, accepting X11 protocol connections over a Unix domain socket at a display address such as `:1`; simultaneously, it acts as a Wayland client, connecting to the running Wayland compositor and presenting X11 window content as `wl_surface` objects backed by shared GPU memory.

The XWayland binary is the X11 server code base compiled as a standalone executable rather than being embedded in Xorg. It implements the Device Dependent X (DDX) layer found in `hw/xwayland/` of the xorg-server source tree. Since the xwayland 22.1 release in 2022, this code resides in its own [upstream repository](https://gitlab.freedesktop.org/xorg/xwayland), separating its release cycle from the Xorg server and allowing features specific to the Wayland compatibility use case to ship independently.

XWayland is not a background daemon. Each Wayland compositor that supports X11 clients launches its own XWayland process on demand, and may terminate it when all X11 clients exit to reclaim memory and GPU resources. There is always exactly one XWayland instance per Wayland compositor session rather than a global X11 server shared across compositors.

### 1.2 What is X11?

The X Window System, version 11 (X11), is a network-transparent display protocol that has been the dominant windowing infrastructure on Unix and Linux systems since 1987. The protocol defines how client applications request drawing operations — rectangles, images, text, bitmaps — and how the server manages windows, input focus, and display output. Client-server separation is fundamental: X11 was designed so a client on one machine can display on a server running on another, over a network or a local Unix socket.

On Linux, the traditional X11 server implementation is Xorg, which communicates with the kernel through the Direct Rendering Manager (DRM) subsystem and manages GPU rendering, display outputs, and input devices. X11 defines hundreds of protocol extensions accumulated over decades: MIT-SHM, DRI3, PRESENT, XRandR, XKB, SHAPE, and many others. Client-side libraries such as `libX11`, `libxcb`, and the older Xlib provide bindings to the wire protocol.

Despite Wayland being its intended successor, X11 remains relevant because a large body of software — games, Java-based toolkits, Wine, legacy productivity applications — was written against the X11 API. XWayland implements full X11 server semantics for these applications while routing their output through the Wayland compositor, making the migration transparent to the application.

### 1.3 What is Wayland?

Wayland is a display server protocol designed as a modern replacement for X11 on Linux. Rather than a monolithic server, Wayland defines a lightweight protocol between clients and a compositor. The compositor manages window layout, display output, and GPU compositing; clients communicate with it directly via Unix domain sockets using the Wayland wire protocol. There is no separate display server process — the compositor is the display server.

The Wayland protocol is defined as a set of XML interface files compiled by the `wayland-scanner` tool into C bindings. Core interfaces include `wl_surface` (a composited buffer region), `wl_buffer` (the underlying pixel data), `xdg_toplevel` and `xdg_popup` (window management roles from the xdg-shell protocol), and `wl_seat` (aggregated input devices). Extensions such as `zwp_linux_dmabuf_v1`, `xwayland-shell-v1`, and `linux-drm-syncobj-v1` are negotiated at runtime through the `wl_registry` global binding mechanism. [Source: Wayland Protocol Specification](https://wayland.freedesktop.org/docs/html/)

Compositors in common use include Mutter (GNOME Shell), KWin (KDE Plasma), sway, Hyprland, and weston (the reference implementation). Each compositor implements its own Wayland server and manages its own XWayland subprocess when X11 compatibility is required. In this architecture, XWayland is a Wayland client of the compositor while simultaneously being an X11 server for the legacy applications it hosts.

### 1.4 What is DRI3?

DRI3 (Direct Rendering Infrastructure, version 3) is an X11 protocol extension that enables GPU-accelerated buffer sharing between X11 clients, the X11 server, and — in the XWayland context — the Wayland compositor. The extension is defined in the X.org protocol specification and implemented in `libxcb-dri3`. Its key mechanism is the ability to pass DMA-BUF file descriptors across process boundaries: a DMA-BUF is a kernel-managed shared buffer that GPU drivers can render into and that multiple processes can map or import without copying pixel data between address spaces.

In the XWayland rendering pipeline, DRI3 works as follows: an X11 client or the GLAMOR OpenGL acceleration layer inside XWayland allocates a GBM (Generic Buffer Manager) buffer object, exports its DMA-BUF file descriptor, and imports it as a pixmap via the `DRI3PixmapFromBuffers` request. XWayland then exports that same DMA-BUF to the Wayland compositor via `zwp_linux_dmabuf_v1`, allowing the compositor to import it as a `wl_buffer` and scan it out to the display directly. This zero-copy path is the preferred rendering route; without DRI3, XWayland falls back to the SHM (shared memory) path, which requires a full CPU copy of each frame.

DRI3 superseded DRI2, which relied on indirect rendering and per-request buffer negotiation with no file-descriptor exchange. The DRI3 model, combined with the `zwp_linux_dmabuf_v1` Wayland protocol, forms the backbone of the modern GPU buffer sharing architecture used throughout the Linux graphics stack.

---

## Process Model: Launching XWayland

### Compositor-Managed Launch

XWayland is not a background service started at login. Each Wayland compositor that supports X11 clients launches its own XWayland process, taking responsibility for providing the listening sockets, authentication, and lifecycle management.

Mutter (GNOME Shell's compositor) provides the most complete reference implementation. Its launcher lives in [`src/wayland/meta-xwayland.c`](https://github.com/GNOME/mutter/blob/main/src/wayland/meta-xwayland.c). The launch sequence is:

1. **Display number selection.** `choose_xdisplay()` tries display numbers starting from `:1`, attempting to create a lock file at `/tmp/.X<N>-lock`. Unlike a full Xorg server, the lock file contains the compositor's PID (not XWayland's). Up to 50 display numbers are attempted before failing.

2. **Socket binding.** Two sockets are bound: an abstract Unix socket (`\0/tmp/.X11-unix/X<N>`) and a filesystem socket (`/tmp/.X11-unix/X<N>`). Both file descriptors are passed to XWayland via `-listenfd`.

3. **Authentication.** Mutter generates a 16-byte `MIT-MAGIC-COOKIE-1` via `getrandom()` and writes an Xauthority file in `/run/user/UID/`. XWayland receives this with `-auth`.

4. **Readiness detection.** A `socketpair()` is created and the write end passed as `-displayfd`. When XWayland has finished initialising, it writes the display number string to this pipe (e.g., `"1\n"`). Mutter's async I/O detects this via `on_displayfd_ready()` and proceeds with X11 display initialisation.

The resulting exec arguments:

```bash
# Mutter XWayland launch (from meta-xwayland.c)
/usr/bin/Xwayland :1 \
  -rootless \
  -noreset \
  -accessx \
  -auth /run/user/1000/mutter-Xauthority-XXXXXX \
  -listenfd 7 \
  -listenfd 8 \
  -displayfd 9 \
  -enable-ei-portal
```

[Source: meta-xwayland.c, GNOME/mutter](https://github.com/GNOME/mutter/blob/main/src/wayland/meta-xwayland.c)

**On-demand startup.** Mutter supports a `META_X11_DISPLAY_POLICY_ON_DEMAND` mode where XWayland is not started until an X11 client actually connects. The compositor keeps the listening socket bound but defers spawning the XWayland process until the first connection arrives. Once all X11 clients exit, XWayland can be killed to free memory and GPU resources. This is the default in GNOME since approximately 3.36 (commit `141373f0`).

**wlroots-based compositors** (sway, Hyprland, labwc) use the `wlr_xwayland` API, which handles the same launch sequence internally. The `wlr_xwayland` struct exposes a `ready` signal fired when XWayland is initialised and an `xwm` pointer to the embedded X11 window manager. [Source: wlroots xwayland.h](https://wlroots.pages.freedesktop.org/wlroots/wlr/xwayland/xwayland.h.html)

### Rootless vs. Rootful Mode

**Rootless mode** (the default, invoked with `-rootless`) is the standard operating mode for desktop compositors. Each top-level X11 window becomes its own `wl_surface` managed independently by the Wayland compositor. The X11 root window has no visual representation — it exists only as a protocol artefact.

**Rootful mode** (no `-rootless` flag) presents the entire X11 desktop as a single `wl_surface`. The X11 root window occupies a fixed rectangle on the Wayland compositor. Use cases include:

- Remote desktop via `xrdp`: modern xrdp 0.9.x spawns rootful XWayland and connects the RDP client to the X11 root window surface.
- VNC sessions under Wayland.
- Fixed-resolution compatibility environments: `Xwayland :1 -geometry 1920x1080`.

The performance trade-off is significant: rootful mode loses the zero-copy DMA-BUF flip path because all X11 content is aggregated into a single root surface, adding an extra compositing step. Rootless mode with DRI3 active is substantially faster for GPU-accelerated applications.

---

## Wayland Client Side: Mapping X11 Windows to Wayland Surfaces

In rootless mode, XWayland must map each X11 top-level window to a Wayland surface presented to the compositor. The core implementation is in `xwl_realize_window()` (`hw/xwayland/xwayland-window.c`), called when an X11 window transitions from `Withdrawn` to `Normal` state (i.e., receives a `MapRequest`). This function:

1. Creates a `wl_surface` via the Wayland compositor's `wl_compositor` global.
2. Assigns either an `xdg_toplevel` (for normal windows) or `xdg_popup` (for transient/popup windows).
3. Allocates `xwl_window` state tracking both the Wayland surface and the X11 window.

The compositor's XWM component (running as an X11 client of XWayland) independently receives `MapRequest` events and reads X11 window properties to determine window type, title, class, and parent relationships. This metadata guides the compositor's decision about which Wayland shell role to assign.

### xwayland-shell-v1 and the WL_SURFACE_ID Race

The original mechanism for associating an X11 window with a Wayland surface used the `WL_SURFACE_ID` X11 ClientMessage atom. XWayland would create a `wl_surface`, then send an X11 ClientMessage to the window containing the 32-bit Wayland object ID. However, Wayland object IDs are ephemeral and process-local — the X11 ClientMessage and the Wayland `wl_surface_create` event travel on separate channels and can arrive in different orders at the compositor's XWM. Compositors worked around this with an "unpaired window list" (e.g., `unpaired_window_list` in Weston's `window-manager.c`), holding unresolved windows until both sides arrived.

The `xwayland-shell-v1` protocol, authored in 2022 by Joshua Ashton (Collabora/Valve), solves this race definitively using a monotonic 64-bit serial. The association protocol is:

```
XWayland side (X11 channel):
  1. XWayland generates monotonic 64-bit serial S
  2. Sends X11 ClientMessage to window W with atom WL_SURFACE_SERIAL,
     value = (S_lo, S_hi) as two 32-bit integers

XWayland side (Wayland channel):
  3. Creates wl_surface
  4. Calls: xwayland_shell_v1.get_xwayland_surface(xwl_surface, wl_surface)
  5. Calls: xwl_surface.set_serial(S_lo, S_hi)
  6. Calls: wl_surface.commit()

Compositor XWM side:
  7. Receives WL_SURFACE_SERIAL ClientMessage: stores S → W mapping
  8. Receives xwl_surface.set_serial from Wayland channel
  9. Matches S from both channels → associates wl_surface with X11 window W
```

Because Wayland message ordering within a connection is guaranteed, and the X11 serial is monotonically increasing, the compositor can safely match associations without races. An XWayland server using `xwayland-shell-v1` must **not** set the `WL_SURFACE_ID` atom — the two mechanisms are mutually exclusive. [Source: xwayland-shell-v1 protocol](https://wayland.app/protocols/xwayland-shell-v1)

### Override-Redirect Windows and Popup Mapping

X11 override-redirect windows (`override_redirect = True`) bypass the window manager entirely in the Xorg model — the WM is not notified and cannot manage them. They are used for menus, tooltips, combo-box dropdowns, and similar transient UI elements.

In XWayland rootless mode, override-redirect windows cannot simply be ignored by the compositor's XWM. Instead, XWayland creates them as `xdg_popup` surfaces, which in Wayland have an explicit parent surface and grab semantics. The `bool override_redirect` field in `wlr_xwayland_surface` ([wlroots xwayland.h](https://wlroots.pages.freedesktop.org/wlroots/wlr/xwayland/xwayland.h.html)) flags these windows. The compositor must assign them as popups relative to their X11 parent window to maintain correct z-ordering.

This is a significant semantic mismatch: Wayland popups have explicit grab rules and dismissal semantics (the compositor controls popup stacks), while X11 override-redirect windows have none. Compositors approximate X11 popup behaviour by creating transient `xdg_popup` surfaces and relying on the application's own input handling for dismissal.

---

## DRI3, GLAMOR, and Buffer Sharing

XWayland uses DRI3 as the exclusive GPU buffer sharing mechanism between X11 clients and the Wayland compositor. There is no DRI2 support. The rendering acceleration layer is GLAMOR — an OpenGL-based 2D rendering library that replaces the traditional DDX rendering paths with OpenGL ES shaders.

**GLAMOR's `egl_backend` abstraction** (`hw/xwayland/xwayland-glamor.h`) provides a pluggable interface. The GBM backend (`hw/xwayland/xwayland-glamor-gbm.c`) is the standard backend on modern Linux. A historical EGLStreams backend for older NVIDIA proprietary drivers was removed in XWayland 24.1, as NVIDIA's driver has supported GBM since version 530.

The per-pixmap GPU state is tracked in `xwl_pixmap`:

```c
/* hw/xwayland/xwayland-glamor-gbm.c — per-pixmap GBM buffer state */
struct xwl_pixmap {
    struct wl_buffer *buffer;   /* Wayland buffer sent to compositor */
    EGLImage image;             /* imported EGLImage for GPU access */
    unsigned int texture;       /* GL texture name for GLAMOR rendering */
    struct gbm_bo *bo;          /* underlying GBM buffer object */
    Bool implicit_modifier;     /* whether modifier was implicitly chosen */
};
```

The per-screen GBM device state is in `xwl_gbm_private`:

```c
/* hw/xwayland/xwayland-glamor-gbm.c — per-screen GBM device state */
struct xwl_gbm_private {
    drmDevice *device;
    char *device_name;
    struct gbm_device *gbm;
    struct wl_drm *drm;         /* legacy wl_drm for authentication */
    int drm_fd;
    int fd_render_node;         /* DRM render node (skips wl_drm auth) */
    Bool drm_authenticated;
    uint32_t capabilities;
    int dmabuf_capable;         /* zwp_linux_dmabuf_v1 available */
};
```

[Source: xorg/xserver hw/xwayland](https://gitlab.freedesktop.org/xorg/xserver/-/tree/master/hw/xwayland)

### GBM Buffer Allocation Pipeline

When an X11 client renders to a pixmap (directly, via GLX, or via 2D drawing operations that GLAMOR accelerates), the complete pipeline to deliver that content to the Wayland compositor is:

```c
/* Step 1: Allocate GBM buffer with format modifiers
 * hw/xwayland/xwayland-glamor-gbm.c: xwl_glamor_gbm_create_pixmap() */
bo = gbm_bo_create_with_modifiers(gbm_device,
    width, height, GBM_FORMAT_ARGB8888,
    modifiers, num_modifiers);
/* Falls back to gbm_bo_create() if modifiers are unsupported */

/* Step 2: Import as EGL image from DMA-BUF planes
 * Using EGL_EXT_image_dma_buf_import_modifiers */
EGLint attribs[] = {
    EGL_WIDTH, width,
    EGL_HEIGHT, height,
    EGL_LINUX_DRM_FOURCC_EXT, GBM_FORMAT_ARGB8888,
    EGL_DMA_BUF_PLANE0_FD_EXT, gbm_bo_get_fd_for_plane(bo, 0),
    EGL_DMA_BUF_PLANE0_OFFSET_EXT, gbm_bo_get_offset(bo, 0),
    EGL_DMA_BUF_PLANE0_PITCH_EXT, gbm_bo_get_stride_for_plane(bo, 0),
    EGL_DMA_BUF_PLANE0_MODIFIER_LO_EXT, modifier & 0xffffffff,
    EGL_DMA_BUF_PLANE0_MODIFIER_HI_EXT, modifier >> 32,
    EGL_NONE
};
EGLImage image = eglCreateImageKHR(egl_display, EGL_NO_CONTEXT,
    EGL_LINUX_DMA_BUF_EXT, NULL, attribs);

/* Step 3: Bind to GL texture for GLAMOR 2D rendering */
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, image);

/* Step 4: Export DMA-BUF FDs and create wl_buffer
 * hw/xwayland/xwayland-glamor-gbm.c: xwl_glamor_gbm_get_wl_buffer_for_pixmap() */
int fd = gbm_bo_get_fd_for_plane(bo, 0);
struct zwp_linux_buffer_params_v1 *params =
    zwp_linux_dmabuf_v1_create_params(dmabuf);
zwp_linux_buffer_params_v1_add(params, fd, 0 /* plane_idx */,
    offset, stride,
    modifier >> 32, modifier & 0xffffffff);
struct wl_buffer *wl_buf =
    zwp_linux_buffer_params_v1_create_immed(params,
        width, height, DRM_FORMAT_ARGB8888, 0);

/* Step 5: Attach to Wayland surface and commit
 * hw/xwayland/xwayland-window.c: xwl_window_post_damage() */
wl_surface_attach(surface, wl_buf, 0, 0);
wl_surface_damage_buffer(surface, 0, 0, width, height);
wl_surface_commit(surface);
```

The `xwl_dri3_open_client()` function handles DRI3's `OpenClient` request. If a render node (`fd_render_node`) is available, it returns the render node FD directly, bypassing `wl_drm_authenticate()`. Render nodes (e.g., `/dev/dri/renderD128`) do not require DRM master authentication, so this is the preferred path on modern systems. [Source: DRI3 protocol spec](https://cgit.freedesktop.org/xorg/proto/dri3proto/plain/dri3proto.txt)

**Critical dependency**: DRI3 must be enabled when GLAMOR is active. If DRI3 is disabled (e.g., by a misconfigured environment), `glamor_fds_from_pixmap()` returns early and pixmaps appear blank. Setting `XWAYLAND_NO_GLAMOR=1` disables both GLAMOR and DRI3, forcing the SHM software path — this eliminates GPU acceleration and is a primary source of tearing and performance issues.

### PRESENT Extension: Flip vs. Blit

XWayland implements the X11 PRESENT extension (`hw/xwayland/xwayland-present.c`), which provides `XPresentPixmap` for synchronised buffer swaps tied to vertical blanking. The implementation has two paths:

- **Flip path**: zero-copy DMA-BUF handoff. The pixmap's GBM buffer object is attached directly to the Wayland surface and submitted to the compositor. No pixel copying occurs.
- **Blit path**: the pixmap contents are copied to a compositor-owned buffer. This is slower but required when the buffer might still be in use by the X11 client.

The flip path in rootless mode has historically caused compatibility issues with Wine games (freedesktop GitLab bug #622: "Activate Present flips in rootless mode with Glamor breaks several wine games"). The constraint is that the buffer must not be simultaneously in use by both the X11 client and the compositor — a requirement that native X11 clients are less likely to violate than Wine's translation layer. Recent XWayland versions have refined the conditions under which flip is preferred. [Source: xwayland-present.c](https://gitlab.freedesktop.org/xorg/xserver/-/blob/master/hw/xwayland/xwayland-present.c)

### Software Fallback: The SHM Path

When GLAMOR/DRI3 is unavailable or disabled, `hw/xwayland/xwayland-shm.c` provides a software rendering path using `wl_shm`. Pixmaps are allocated in shared memory, and `wl_shm_pool` buffers are submitted to the compositor. This path is entirely CPU-rendered, causes tearing on most compositors (which do not synchronise `wl_shm` buffer submission with scanout), and should be avoided for any performance-sensitive use.

---

## GLX on XWayland

GLX applications (OpenGL via the GLX extension) on XWayland use Mesa's GLX DRI backend, which goes through the DRI3 path. When an application calls `glXCreateWindow()`:

1. Mesa's DRI3 loader calls the DRI3 `OpenClient` request on the X connection, receiving a render node FD from `xwl_dri3_open_client()`.
2. Mesa creates a GBM device from the render node and allocates GBM buffer objects for the back buffer.
3. GLAMOR imports these buffers as `xwl_pixmap` instances.
4. When the application calls `glXSwapBuffers()`, Mesa calls `XPresentPixmap()` with the rendered back buffer.
5. XWayland's PRESENT handler calls `xwl_glamor_gbm_get_wl_buffer_for_pixmap()` to obtain a `wl_buffer` backed by the GBM BO's DMA-BUF FD.
6. This `wl_buffer` is submitted via `wl_surface_attach()` / `wl_surface_commit()` to the compositor.

The compositor thus receives a native DMA-BUF buffer that was rendered by Mesa directly — no pixel copies are involved in the hot path. The X11 connection carries only control flow; pixel data flows via shared GPU memory from Mesa to the compositor.

The GLX visual selection and EGL context creation both occur on the same DRM device as XWayland's GLAMOR context, ensuring compatibility. Applications that attempt to use a different GPU (multi-GPU systems) may encounter mismatches; in such cases the GLAMOR context and the application's GLX context must use the same DRM device for DMA-BUF import to succeed.

**Vulkan on XWayland.** Vulkan applications can also run on XWayland. Mesa's WSI (Window System Integration) for Vulkan on X11 uses DRI3/DMA-BUF: `vkCreateXlibSurfaceKHR()` creates an X11 window surface, and present operations use `XPresentPixmap()` in the same way as GLX. The VK_KHR_xcb_surface and VK_KHR_xlib_surface extensions both funnel into the same DRI3-based present path. For optimal performance, Vulkan applications should prefer `VK_KHR_wayland_surface` by checking `$WAYLAND_DISPLAY` and using native Wayland WSI when available, which avoids the XWayland translation layer entirely.

---

## Explicit Synchronisation: linux-drm-syncobj-v1

Before explicit synchronisation, XWayland relied on implicit sync: the kernel's DMA-BUF fence infrastructure ensured that the compositor would not read a buffer while the GPU was still writing it. This worked on most drivers, but NVIDIA's proprietary driver historically did not honour implicit sync in the same way, causing flickering black windows in applications like Chromium, Steam, and Spotify.

XWayland 24.1 (released 2024) added support for `linux-drm-syncobj-v1`, the Wayland explicit synchronisation protocol. With this protocol, XWayland explicitly communicates GPU timeline synchronisation points alongside each buffer submission. [Source: linux-drm-syncobj-v1 protocol](https://wayland.app/protocols/linux-drm-syncobj-v1)

Requirements: XWayland ≥ 24.1, a compositor implementing `linux-drm-syncobj-v1` (KWin 6.1+), and NVIDIA driver ≥ 555.58.

The mechanism:

```c
/* XWayland creates a DRM syncobj timeline
 * hw/xwayland/xwayland-present.c */

/* Import DRM syncobj FD as Wayland timeline object */
struct wp_linux_drm_syncobj_timeline_v1 *timeline =
    wp_linux_drm_syncobj_manager_v1_import_timeline(mgr, syncobj_fd);

/* Per-frame explicit sync sequence: */
uint64_t acquire_point = frame_counter;      /* GPU must reach before compositor reads */
uint64_t release_point = frame_counter + 1; /* compositor signals when buffer is free */

/* Set acquire point: compositor waits on this before accessing buffer */
wp_linux_drm_syncobj_surface_v1_set_acquire_point(
    syncobj_surface, timeline,
    acquire_point >> 32, acquire_point & 0xffffffff);

/* Set release point: compositor signals this when buffer is released */
wp_linux_drm_syncobj_surface_v1_set_release_point(
    syncobj_surface, timeline,
    release_point >> 32, release_point & 0xffffffff);

/* Both points must be set before commit; buffer attachment follows */
wl_surface_attach(surface, buffer, 0, 0);
wl_surface_commit(surface);
/* frame_counter advances monotonically per surface per commit */
```

With explicit sync, the compositor can safely pipeline multiple frames without stalls: it submits the buffer to the display hardware with a DRM syncobj wait, and signals the release timeline point via DRM when the hardware is done. XWayland can then immediately reuse the buffer for the next frame.

The same release of XWayland 24.1 also added OpenGL ES 3.0 shader support in GLAMOR and removed the EGLStreams backend entirely. [Source: XWayland 24.1 release notes, Phoronix](https://www.phoronix.com/news/XWayland-24.1-Released)

---

## Input Handling: Translating Wayland Events to X11

XWayland's input subsystem (`hw/xwayland/xwayland-input.c`) receives Wayland `wl_seat` events and translates them into X11 input events dispatched to X11 client windows. The system maintains `xwl_seat` structs tracking one Wayland seat and all associated X11 device representations.

### Keyboard: XKB Map Synchronisation and Keycode Offset

Wayland keyboards carry an XKB keymap as a text string via `wl_keyboard.keymap`. XWayland uses this keymap to initialise its own XKB state via `xkb_keymap_new_from_string()`, ensuring that the XKB keymap is identical between the compositor and XWayland. This means key symbols, dead keys, and compose sequences all behave consistently.

A critical constant offset: Wayland uses Linux evdev keycodes (starting at 1), while XKB uses keycodes starting at 8. The conversion is:

```c
/* hw/xwayland/xwayland-input.c */
/* xwl_keyboard_proc() initialisation */
xkb_keycode_t xkb_keycode = wayland_key_code + 8;
```

This +8 offset must be applied to all key events. Modifier state is tracked via `xkb_state_update_mask()` called on each `wl_keyboard.modifiers` event, synchronising shift, ctrl, alt, super, and lock states between the compositor and XWayland's X11 client view.

```c
/* hw/xwayland/xwayland-input.c: keyboard_handle_modifiers */
static void keyboard_handle_modifiers(void *data,
    struct wl_keyboard *keyboard,
    uint32_t serial, uint32_t mods_depressed,
    uint32_t mods_latched, uint32_t mods_locked,
    uint32_t group)
{
    struct xwl_seat *xwl_seat = data;
    xkb_state_update_mask(xwl_seat->xkb_state,
        mods_depressed, mods_latched, mods_locked, 0, 0, group);
    /* Dispatch updated modifier state to X11 clients */
}
```

### Pointer: Absolute, Relative, and Grab Emulation

XWayland registers three virtual X11 pointer devices: an absolute pointer, a relative pointer, and a pointer gestures device. Wayland `wl_pointer` events are translated:

```c
/* hw/xwayland/xwayland-input.c: pointer protocol listeners */
static const struct wl_pointer_listener xwl_pointer_listener = {
    .enter  = pointer_handle_enter,   /* focus tracking */
    .leave  = pointer_handle_leave,
    .motion = pointer_handle_motion,  /* absolute coordinates */
    .button = pointer_handle_button,  /* button press/release */
    .axis   = pointer_handle_axis,    /* scroll */
    .frame  = pointer_handle_frame,   /* event group boundary */
    /* ... */
};
```

**Relative motion for games.** Applications using SDL2 relative mouse mode (games) require unaccelerated raw pointer deltas. XWayland registers a `zwp_relative_pointer_v1` listener to receive `(dx_unaccel, dy_unaccel)` values, which it injects as XI raw events. The function `relative_pointer_handle_relative_motion()` handles this translation. [Source: zwp_relative_pointer_v1 protocol](https://wayland.app/protocols/relative-pointer-unstable-v1)

**Pointer grab emulation.** X11 allows clients to grab the pointer (exclusive input capture). Wayland has no equivalent mechanism for arbitrary clients. XWayland approximates pointer grabs via:

- `maybe_fake_grab_devices()`: when an X11 client requests a grab, XWayland sets internal state to route all pointer events to that client regardless of Wayland surface focus.
- `xwl_seat_maybe_lock_on_hidden_cursor()`: when an X11 client hides the cursor via `XFixesHideCursor()`, XWayland requests a `zwp_pointer_constraints_v1` pointer lock on the Wayland compositor. This prevents the compositor from moving the host cursor while the X11 application owns it (as expected by fullscreen games).

The `zwp_xwayland_keyboard_grab_unstable_v1` protocol (merged in xorg-server 1.20 / 2018) is a Wayland extension exclusively for XWayland. It requests that all keyboard events be forwarded to a surface even without focus — used by screen lockers and password entry dialogs. **The compositor is not required to honour this request**, and there is no error notification if it does not. The `_XWAYLAND_MAY_GRAB_KEYBOARD` X11 window property is a hint to the compositor that the application may request a keyboard grab. [Source: xwayland-keyboard-grab protocol](https://wayland.app/protocols/xwayland-keyboard-grab-unstable-v1)

### Touch and Tablet Input

XWayland creates a touch device supporting up to 20 simultaneous contact points, mapping Wayland `wl_touch` events to X11 XI2 multitouch events via `xwl_touch_proc()`. Tablet devices (drawing tablets, stylus pens) are exposed as XI2 tablet tools with 6 axes (x, y, pressure, tilt-x, tilt-y, wheel) and 9 buttons via `xwl_tablet_proc()`. These are translated from the `zwp_tablet_manager_v2` / `zwp_tablet_tool_v2` Wayland protocols. [Source: hw/xwayland/xwayland-input.c](https://gitlab.freedesktop.org/xorg/xserver/-/blob/master/hw/xwayland/xwayland-input.c)

---

## Clipboard and Selections

### X11 Selections and Wayland Data Devices

X11 has three selection types:
- `PRIMARY` — set on text highlight; pasted with middle-click
- `CLIPBOARD` — explicit cut/copy/paste
- `SECONDARY` — rarely used

Wayland exposes two corresponding mechanisms:
- `wl_data_device` / `wl_data_source` for `CLIPBOARD`
- `zwp_primary_selection_unstable_v1` for `PRIMARY`

XWayland maintains a bidirectional bridge between these, implemented in selection-handling code spread across compositor-specific implementations. Weston's version lives in [`xwayland/selection.c`](https://github.com/wayland-mirror/weston/blob/master/xwayland/selection.c). The bridge must handle:

1. **X11 → Wayland**: when an X11 client asserts selection ownership (e.g., user copies text in a terminal), XWayland announces itself as the Wayland clipboard owner via `wl_data_source`. When a Wayland client requests the clipboard, XWayland forwards the request to the X11 owner and pipes the response back.

2. **Wayland → X11**: when a Wayland client owns the clipboard and an X11 client requests it, XWayland requests the data from the Wayland data device and injects it into the X11 selection transfer.

The key function in Weston's implementation: `weston_wm_handle_selection_event()` dispatches XFixes `SelectionNotify`, `SelectionRequest`, `SelectionNotify`, and `PropertyNotify` events through the bridge.

The challenge is format conversion: X11 selections use atoms (e.g., `UTF8_STRING`, `STRING`, `text/html`) while Wayland uses MIME type strings (`text/plain;charset=utf-8`, `text/html`). XWayland maintains atom-to-MIME-type mapping tables for this translation. [Source: Clipboard sync internals, Martin Gräßlin 2016](https://blog.martin-graesslin.com/blog/2016/07/synchronizing-the-x11-and-wayland-clipboard/)

### INCR Protocol for Large Payloads

X11 has a per-request payload limit of approximately 256 KB. For clipboard content exceeding this — common with image data or large text — X11 uses the INCR (incremental) transfer protocol:

1. The selection owner sets the target property to the `INCR` atom with the total size as the value.
2. The requestor acknowledges by deleting the property.
3. The owner delivers the next chunk by setting the property to the chunk data.
4. The requestor reads and deletes again, triggering the next chunk.
5. A zero-length property signals transfer completion.

XWayland's selection bridge must coordinate INCR with the Wayland data pipe: on the X11 side it monitors property deletion events; on the Wayland side it reads from a pipe FD connected to the Wayland data source. This requires non-blocking I/O integrated with the X11 event loop. Weston's implementation at lines 316–393 of `selection.c` handles this coordination. [Source: Xwayland selection internals](https://xeechou.net/posts/xwayland-selection/)

### Drag-and-Drop: XDND to wl_data_offer

X11 uses the XDND protocol (version 5) for drag-and-drop. Weston's `xwayland/dnd.c` implements the bridge:

1. A dedicated XDND proxy window is created by `weston_wm_dnd_init()`, advertising XDND version 4 in its `XdndAware` property.
2. When an X11 drag enters this proxy, `handle_enter()` reads `XdndTypeList` and maps X11 type atoms (e.g., `UTF8_STRING`) to MIME strings (`text/plain;charset=utf-8`).
3. XWayland creates a `wl_data_source` on the Wayland side, advertising the mapped MIME types.
4. Wayland `wl_data_offer` events are translated back to XDND `XdndStatus` responses.
5. On drop, XWayland triggers `XdndFinished` to the X11 source after the Wayland drop completes.

[Source: Weston xwayland/dnd.c](https://github.com/wayland-mirror/weston/blob/master/xwayland/dnd.c)

---

## Rootless Mode Internals: Window Properties and Compositor Integration

When an X11 window maps in rootless mode, the compositor's XWM component reads X11 window properties to populate the `wlr_xwayland_surface` struct (wlroots) or its equivalent. Key properties and their roles:

| X11 Property | Wayland/Compositor Use |
|---|---|
| `WM_CLASS` (`class`, `instance`) | Compositor window grouping, taskbar icons, window rules |
| `_NET_WM_PID` | Security checks, app identification, systemd scope assignment |
| `_NET_WM_NAME` | Window title displayed in compositor decorations |
| `_NET_WM_WINDOW_TYPE` | Determines shell role: `_NORMAL` → toplevel, `_POPUP_MENU` → popup |
| `_NET_WM_STATE` | `_MAXIMIZED_VERT/HORZ`, `_FULLSCREEN`, `_ABOVE`, `_BELOW` |
| `_MOTIF_WM_HINTS` | Server-side decoration control (CSD vs. SSD) |
| `WM_PROTOCOLS` / `WM_DELETE_WINDOW` | Close button behaviour |
| `_NET_WM_SYNC_REQUEST` | Resize synchronisation (see below) |

[Source: wlroots xwayland.h](https://wlroots.pages.freedesktop.org/wlroots/wlr/xwayland/xwayland.h.html)

**Resize synchronisation and `_XWAYLAND_ALLOW_COMMITS`.** Interactive window resize exposes a subtle race:

1. The WM sends `_NET_WM_SYNC_REQUEST` with a counter serial.
2. The WM updates the window's position in X11 (moves it to its new location on screen).
3. The X11 application repaints and increments the XSync counter.
4. XWayland asynchronously attaches the new `wl_buffer` to the `wl_surface`.
5. Problem: the compositor may receive the new buffer before or after the position update, causing a frame where the old content appears at the new position or vice versa.

KWin (KDE Plasma 6) solves this by setting the `_XWAYLAND_ALLOW_COMMITS` X11 property to `0` on the window during resize, blocking XWayland from issuing `wl_surface.commit()`. Once the XSync counter acknowledges the repaint, KWin sets `_XWAYLAND_ALLOW_COMMITS` back to `1`, allowing the buffer commit to proceed. This ensures position and content changes are delivered atomically to the Wayland compositor. [Source: KWin XWayland resize blog](https://blog.vladzahorodnii.com/2024/10/28/improving-xwayland-window-resizing/)

---

## Common XWayland Issues and Workarounds

### Screen Tearing and the DRI3 vs. SHM Path

Screen tearing in XWayland applications almost always indicates that the DRI3/GLAMOR path is inactive and the SHM software path is being used. The compositor receives `wl_shm` buffers and has no reliable mechanism to synchronise their display with vertical blanking.

Diagnosis:

```bash
# Check if DRI3 is active for GLX
glxinfo | grep -E 'direct|DRI3'
# "direct rendering: Yes" and DRI3 extension present confirms DRI3 path

# Check GLAMOR is active
LIBGL_DEBUG=verbose glxinfo 2>&1 | grep -i glamor

# Confirm zwp_linux_dmabuf_v1 is available to XWayland
wayland-info 2>/dev/null | grep dmabuf

# Force software path for testing (will cause tearing)
XWAYLAND_NO_GLAMOR=1 application_name
```

If DRI3 is absent, check that the DRM render node is accessible (`/dev/dri/renderD128` readable by the user) and that `libGL`/`libEGL` is linked against Mesa rather than a non-accelerated stub.

### HiDPI and Fractional Scaling

X11's DPI model predates fractional scaling. XWayland handles HiDPI through several mechanisms:

- **Integer scaling**: `Xwayland -dpi 192` sets 2× integer scale. X11 apps see a 192 DPI display and scale UI elements at 2×. GTK3 and Qt respect `GDK_SCALE=2` and `QT_SCALE_FACTOR=2` environment variables.

- **`_XWAYLAND_GLOBAL_OUTPUT_SCALE`**: an X11 root window property XWayland sets based on the Wayland output scale factor. Applications that query this (via the XWayland-specific API) can adapt their rendering.

- **Fractional scale (GNOME 46+)**: Mutter MR in early 2024 added `wp_fractional_scale_v1` support for XWayland windows. XWayland negotiates a fractional scale (e.g., 1.5×) with the compositor. The X11 client renders at a larger logical resolution and XWayland uses `wp_viewporter` to downsample to the physical pixels. X11 apps that do not support fractional DPI will appear blurry at non-integer scales — this is expected and inescapable without native Wayland support.

Applications should be encouraged to use `MOZ_ENABLE_WAYLAND=1` (Firefox) or equivalent native Wayland modes, which bypass XWayland entirely and get pixel-perfect fractional scaling via `wp_fractional_scale_v1`. [Source: XWayland man page](https://man.archlinux.org/man/extra/xorg-xwayland/Xwayland.1.en)

### Cursor Scaling

X11 cursor themes are loaded by `libxcursor` based on `XCURSOR_SIZE` and `XCURSOR_THEME` environment variables. XWayland does not automatically scale cursors to match HiDPI output. Setting `XCURSOR_SIZE=48` (for 2× on a 24px cursor theme) is the manual workaround. Some compositors set `XCURSOR_SIZE` in the environment when launching XWayland.

### Anti-Cheat and EAC/BattleEye

Easy Anti-Cheat (EAC) and BattleEye perform kernel-level process inspection. On XWayland, the game process is a standard Linux process — there is no special sandboxing from XWayland itself. However, EAC's Linux port (which Valve worked with EAC to enable for Proton/Steam Play) does support XWayland since approximately 2022. Games that do not have a native Linux EAC build still cannot run on XWayland (or any Linux runtime) due to the missing Windows kernel driver component, not due to any XWayland-specific limitation.

BattleEye's Linux support followed a similar trajectory. Applications should check whether the specific game's EAC/BattleEye integration has a native Linux build — XWayland itself is not the blocking factor.

---

## XWayland vs. Rootful Xwayland

| Characteristic | Rootless Mode | Rootful Mode |
|---|---|---|
| Window presentation | Each X11 window → separate `wl_surface` | Entire X11 desktop → single `wl_surface` |
| Compositor integration | Full (window decorations, tiling, virtual desktops) | Minimal (single surface, no per-window integration) |
| GPU path | DRI3 DMA-BUF zero-copy | Extra compositing step; wl_shm path common |
| Performance | High (flip path) | Lower (extra blit) |
| Use case | Desktop applications, games | Remote desktop (xrdp, VNC), compatibility testing |
| CSD/SSD | Compositor decides per-window | Compositor sees only the root surface |

**xrdp integration.** Modern `xrdp` 0.9.x can use rootful XWayland as its X11 server when running under a Wayland host. The RDP client connects to xrdp, which drives a rootful XWayland session. The X11 root window surface is captured and transmitted as the RDP desktop image. This architecture avoids the heavyweight Xorg server stack while maintaining X11 compatibility. [Source: Xwayland man page](https://man.archlinux.org/man/extra/xorg-xwayland/Xwayland.1.en)

**WSLg (Windows Subsystem for Linux GUI).** Microsoft's WSLg stack uses a rootful XWayland approach combined with RDP: Weston acts as the XWayland host and RDP server, and FreeRDP running as an RDP client on the Windows side renders the compositor output in a `mstsc.exe` window. The `DISPLAY` and `WAYLAND_DISPLAY` environment variables are automatically set in the WSL2 container so that both X11 and native Wayland applications work transparently. This is a direct application of the rootful XWayland architecture described above, extended with RDP transport. The XWayland instance in WSLg is not special — it is the standard `Xwayland` binary launched by Weston, making the same DRI3/DMA-BUF path available for GPU-accelerated X11 applications when the appropriate GPU driver (Hyper-V virtual GPU) supports it.

**libdecor in rootful mode.** XWayland 23.1 added libdecor integration for rootful mode, providing compositor-style window decorations even in the single-surface rootful presentation.

---

## Debugging XWayland

A systematic approach to diagnosing XWayland issues:

```bash
# 1. Protocol-level Wayland event tracing
WAYLAND_DEBUG=1 some_x11_app 2>&1 | grep -E 'xwayland|wl_surface|xdg'

# 2. DRI3/OpenGL path verbosity
LIBGL_DEBUG=verbose glxinfo 2>&1 | head -60
# Look for: "DRI3 support", "GLAMOR", render node path

# 3. Inspect X11 window properties
xwininfo -root -tree | grep -i "application name"
# Get window ID, then:
xprop -id 0x<window_id>
# Look for: WM_CLASS, _NET_WM_PID, _NET_WM_NAME, WM_STATE,
#           _NET_WM_WINDOW_TYPE, _MOTIF_WM_HINTS

# 4. Check environment variables
echo "X11 display:     $DISPLAY"        # e.g., :1
echo "Wayland display: $WAYLAND_DISPLAY" # e.g., wayland-0
echo "XAUTHORITY:      $XAUTHORITY"

# 5. Compositor-supported Wayland protocols
wayland-info 2>/dev/null | grep -E 'xwayland|syncobj|dmabuf|primary|fractional'

# 6. Verify X11 socket and lock file
ls /tmp/.X*
cat /tmp/.X1-lock   # should contain compositor PID

# 7. Verify DRM devices
ls /dev/dri/         # card0 (master), renderD128 (render node)
stat -c "%a %G" /dev/dri/renderD128  # should be 660, group render

# 8. Test explicit sync availability (XWayland 24.1+)
WAYLAND_DEBUG=1 glxgears 2>&1 | grep syncobj

# 9. Disable GLAMOR for isolation testing
XWAYLAND_NO_GLAMOR=1 application_name

# 10. XKB keymap verification
setxkbmap -print   # shows active XKB keymap in XWayland session
xkbcomp :1 /tmp/xkb-dump.xkb  # dump compiled keymap
```

For compositor-specific debugging:

- **KWin**: `QT_LOGGING_RULES="kwin.xwayland.*=true" kwin_wayland --xwayland`
- **Mutter**: `MUTTER_DEBUG_XWAYLAND=1 gnome-shell --wayland`
- **Weston**: `WESTON_DEBUG=xwm weston`

**Checking XWayland version and features:**

```bash
# Version of the installed XWayland binary
Xwayland -version

# Confirm which protocols XWayland negotiated with the compositor
WAYLAND_DEBUG=1 Xwayland :9 -rootless -displayfd 1 1>/dev/null 2>&1 | \
  grep -E 'linux_dmabuf|syncobj|xwayland_shell|fractional'

# Check X11 atoms registered by XWayland
DISPLAY=:1 xlsatoms | grep -i xwayland

# Inspect DRI3 extension availability inside XWayland session
DISPLAY=:1 xdpyinfo | grep -A2 DRI3

# Verify GLAMOR is in use (look for EGLImage-backed pixmaps)
DISPLAY=:1 glxinfo -display :1 | grep -E 'direct|GLX_OML|GL_EXT_buffer'
```

The distinction between `$DISPLAY` and `$WAYLAND_DISPLAY` is fundamental: native Wayland applications use `$WAYLAND_DISPLAY` and never touch `$DISPLAY`; X11 applications use `$DISPLAY` and are unaware of `$WAYLAND_DISPLAY`. Applications that support both (e.g., Firefox with `MOZ_ENABLE_WAYLAND=1`) check `$WAYLAND_DISPLAY` first. If both are set, an application using `$DISPLAY` is running under XWayland; one using `$WAYLAND_DISPLAY` is native.

---

## xwayland-satellite: Decoupling XWayland from the Compositor

`xwayland-satellite` (by Supreeeme, written in Rust) is a standalone process that implements rootless XWayland support on top of any compositor that exposes `xdg_wm_base` and `wp_viewporter`. It acts as an intermediary:

1. Connects to the host Wayland compositor as a regular Wayland client.
2. Spawns XWayland with `-rootless`.
3. Acts as the X11 window manager for XWayland, presenting X11 windows as `xdg_toplevel` / `xdg_popup` surfaces to the host compositor.
4. Uses `wp_viewporter` for content scaling.
5. Optionally uses `zwp_linux_dmabuf_v1`, `wp_fractional_scale_v1`, and `zwp_pointer_constraints_v1` when available.

Requirements: XWayland ≥ 23.1, host compositor with `xdg_wm_base` v5. [Source: xwayland-satellite GitHub](https://github.com/Supreeeme/xwayland-satellite)

This architecture enables tiling compositors like niri and early versions of Hyprland to support XWayland without implementing the full X11 window manager protocol themselves. The compositor sees only regular Wayland `xdg_toplevel` surfaces — it does not need `xwayland-shell-v1` or any XWayland-specific protocol knowledge.

The trade-off is reduced integration: compositor-specific XWayland features (KWin's `_XWAYLAND_ALLOW_COMMITS` resize sync, GNOME Shell's fractional scale integration) are not available through xwayland-satellite. It is a compatibility layer, not a full-fidelity replacement for embedded XWayland support.

---

## Strategic Outlook: Will XWayland Ever Be Retired?

The framing question — "when will XWayland be retired?" — contains a definitional ambiguity that shapes the entire answer. Two distinct retirements are often conflated: retirement of the **X.org server** (the compositor-as-Xserver model where a display server forks `/usr/bin/X`), and retirement of **XWayland** (the Wayland-client X11 bridge). The first is already underway; the second is not scheduled and is unlikely on any foreseeable horizon.

GNOME's long-term goal is to ship without the standalone X.org server — meaning GNOME Shell will no longer launch an Xorg session at all. Fedora 40 removed Xorg from the default GNOME session in 2024, and Red Hat has committed to dropping the standalone Xorg server from RHEL 10 entirely. [Source: Red Hat RHEL 10 plans for Wayland and Xorg server](https://www.redhat.com/en/blog/rhel-10-plans-wayland-and-xorg-server) KDE Plasma has similarly committed to becoming Wayland-native, with the KWin team explicitly positioning Wayland as the only forward path. [Source: KDE Blogs — Going all-in on a Wayland future](https://blogs.kde.org/2025/11/26/going-all-in-on-a-wayland-future/) Neither of these commitments touches XWayland, which will continue to ship as the X11 compatibility bridge in both desktop environments indefinitely.

**The practical sunset blockers.** Four application categories currently make XWayland load-bearing infrastructure with no near-term replacement path:

- *Anti-cheat and kernel-level game security.* EAC and BattleEye anti-cheat systems rely on `ptrace`, `/proc/[pid]/mem` introspection, and kernel modules that are X11-session-aware; several titles explicitly whitelist Linux only when an X11 display is present. Without platform-level certification that Wayland is equally auditable, anti-cheat vendors will not drop the X11 requirement. Native Wayland support in this category requires engine-level changes, not just toolkit ports.
- *Accessibility tooling.* AT-SPI2 (the Linux accessibility bus) bridges Wayland via a compositor-side accessibility socket, but screen readers (Orca), screen magnifiers, and switch-access tools have deep dependencies on X11 `XAccessibility`, `XTest`, and `XRecord` extensions for programmatic input synthesis and on-screen DOM traversal. The `atspi-wayland` protocol work is ongoing but not yet feature-complete for full assistive-technology workflows. (See Ch175 — AT-SPI2 and Compositor Accessibility.)
- *Screen sharing and remote desktop.* Pipewire-based screen capture via `xdg-desktop-portal` is now standard for native Wayland apps, but many professional conferencing and recording tools (OBS, corporate virtual meeting software) still emit or consume `DISPLAY`-based capture paths. The transition requires both application changes and Wayland portal API stabilisation for capture-source selection.
- *Input method editors for CJK text.* XIM (X Input Method) is still in active use by legacy CJK applications. The XIM-to-`zwp_text_input_v3` bridge in XWayland has known ordering bugs under rapid composition events. Until applications migrate to Fcitx5's or IBus's native Wayland input interfaces, XWayland remains the only reliable input path.

**The xwayland-satellite path.** Rather than leading to XWayland's removal, the `xwayland-satellite` project points toward a different trajectory: making XWayland lower-maintenance and lighter-weight indefinitely. [Source: xwayland-satellite GitHub](https://github.com/Supreeeme/xwayland-satellite) By decoupling the X11 window manager from the compositor process, xwayland-satellite reduces the implementation burden on compositor authors to zero — they never need to handle `xcb_connection_t`, X11 atoms, or `_NET_WM_*` properties. If this model becomes standard, XWayland evolves from a demanding compositor integration obligation into a background daemon, sustained by a small focused team, needed only when an X11 application is actually running. This is a sustainable steady state, not a path to removal.

**The protocol gap XWayland cannot bridge.** Regardless of maintenance trajectory, certain capabilities introduced by modern Wayland protocols are structurally inexpressible to X11 applications. The `wp_tearing_control_v1` protocol allows a Wayland application to opt into tearing for low-latency rendering — an X11 application cannot discover or signal this intent through any X11 extension. The `linux-drm-syncobj-v1` explicit sync timeline, while implemented in XWayland 24.1, exposes timeline fence semantics that have no X11 protocol equivalent; XWayland approximates synchronisation rather than exposing it. HDR surface metadata from `xx-color-management-v1` can be received by XWayland's root window ICC bridge, but X11 applications cannot consume per-frame HDR tone-mapping hints. `wp_fractional_scale_v1` can be read by XWayland to set a synthetic RandR DPI, but the X11 application must then honour the DPI itself — fractional rendering quality remains compositor-native-only. These gaps will widen as the Wayland protocol ecosystem develops: each new capability must be bridged imperfectly or not at all.

**Realistic timeline.** On a 5–10 year horizon, XWayland is best understood as **permanent infrastructure**, not a transitional technology with a scheduled sunset. Major distributions will complete the X.org server removal over 2–4 years, leaving XWayland as the sole maintained X11 implementation. The application long tail — the set of X11 applications that will never receive a native Wayland port — will shrink but will not reach zero: there are proprietary ISV applications, industrial control software, and niche scientific tools for which the developer is either unable or unwilling to port. For this population, XWayland is the final maintenance environment.

What "X11 retirement" actually means in practice is therefore narrow and already in progress: compositor developers stop forking an X.org server process; display managers stop offering "GNOME on Xorg" sessions; distributions stop shipping `xf86-video-*` DDX drivers. XWayland itself remains, evolves, and carries forward the X11 application ecosystem for as long as those applications exist. The question is not whether XWayland will be retired — it will not — but whether it will be maintained actively or enter a slower-moving legacy phase. The RHEL 10 commitment and xwayland-satellite's architecture both suggest it will remain a well-maintained component for the foreseeable future.

---

## Roadmap

### Near-term (6–12 months)

- **Wayland color management integration for X11 apps.** The `xx-color-management-v4` protocol merged into wayland-protocols 1.41 in early 2025, enabling Wayland-native compositors to advertise ICC profiles and HDR metadata. XWayland's next development cycle is expected to bridge X11 `ICC_PROFILE` root window property and `EDID` atom data into the new `xx-color-management-v1` Wayland interface, allowing X11 applications to receive at least sRGB-tagged surfaces correctly. Currently, X11 apps via XWayland are clamped to sRGB even on HDR-capable displays. [Source: Wayland Colour Management and HDR Protocol merged, GamingOnLinux](https://www.gamingonlinux.com/2025/02/wayland-colour-management-and-hdr-protocol-finally-merged/)
- **Broader explicit sync adoption across compositors.** Explicit sync via `linux-drm-syncobj-v1` landed in XWayland 24.1 and wlroots 0.19.0/Sway 1.11. The near-term work is completing the same support in smaller compositors (labwc, Hyprland stable branch) and ensuring the NVIDIA proprietary driver's `syncobj` timeline path is exercised correctly end-to-end. [Source: XWayland 24.1 Released With Explicit Sync, Phoronix](https://www.phoronix.com/news/XWayland-24.1-Released)
- **Improved on-demand XWayland lifecycle management.** GNOME's on-demand XWayland launch (default since 3.36) is being refined to handle the edge case where the XWayland process exits while an X11 socket connection is being established, causing a race. Patches addressing re-spawn logic and socket handoff ordering are expected upstream. Note: needs verification against current freedesktop.org GitLab issue tracker.
- **xwayland-satellite fractional scaling fixes.** The `xwayland-satellite` project has open issues around fractional scaling being rounded up at non-integer scale factors (e.g., 1.5× rendering at 2×), causing oversized windows. Fixes depend on `wp_fractional_scale_v1` interaction with `wp_viewporter` being correctly sequenced. [Source: xwayland-satellite fractional scaling issue #168, GitHub](https://github.com/Supreeeme/xwayland-satellite/issues/168)
- **KDE Plasma 6.x Wayland-exclusive transition.** KDE's roadmap to become Wayland-only means Plasma is increasingly relying on XWayland as the sole X11 compatibility path. Near-term KWin XWayland work focuses on removing remaining fallbacks that require a native X11 session. [Source: KDE Blogs — Going all-in on a Wayland future](https://blogs.kde.org/2025/11/26/going-all-in-on-a-wayland-future/)

### Medium-term (1–3 years)

- **HDR passthrough for X11 applications via XWayland.** Once `xx-color-management-v1` is stable and compositor support is widespread, the longer-term goal is allowing X11 applications that set `_NET_WM_HDR_CAPABLE` or equivalent atoms to receive an HDR surface from the compositor. This requires XWayland to translate X11 color hints to Wayland color management protocol objects. The architecture for this is under discussion in freedesktop mailing lists. [Source: HDR ArchWiki, compositor HDR support status](https://wiki.archlinux.org/title/HDR_monitor_support)
- **Input method (IM) protocol modernisation.** XWayland currently bridges X11 input methods (XIM) to Wayland text-input protocols (`zwp_text_input_v3`). The mismatch between XIM's synchronous pull model and Wayland's event-push model causes subtle ordering bugs with CJK input methods. A cleaner IM bridge protocol is being proposed within the Wayland protocols working group. Note: needs verification against current wayland-protocols MR tracker.
- **Stabilising `xwayland-shell-v1`.** The `xwayland-shell-v1` Wayland protocol (which solved the surface-ID race condition) is still tagged as unstable. Promoting it to stable requires inter-compositor agreement on the final protocol shape. This is expected to happen as Wayland-exclusive compositors proliferate and the protocol sees wider deployment. Note: needs verification against current wayland-protocols version.
- **Security: privilege separation for XWayland.** XWayland runs as the same Unix user as the Wayland compositor client it serves, with full access to the X11 socket. Proposals for sandboxing XWayland (running it in a separate namespace with restricted capabilities) have been discussed as a medium-term hardening goal, particularly relevant for distributions enabling XDG Portals by default. Note: needs verification against current security roadmap discussions.
- **RHEL 10 and enterprise distribution migration.** Red Hat has announced plans to drop the standalone Xorg server from RHEL 10, relying entirely on XWayland for X11 compatibility. This creates an enterprise support obligation that will drive additional XWayland stability and regression testing investment over the 1–3 year horizon. [Source: Red Hat RHEL 10 plans for Wayland and Xorg server](https://www.redhat.com/en/blog/rhel-10-plans-wayland-and-xorg-server)

### Long-term

- **Gradual X11 application sunset.** As major toolkits (GTK, Qt, Electron, JavaFX) complete their Wayland-native backends, the fraction of applications requiring XWayland will shrink. The long-term architectural direction is XWayland transitioning from a daily-driver path to a legacy compatibility layer serving only games (via Wine/Proton), proprietary ISV software, and specialised industrial applications. The X11 protocol itself will not gain new features.
- **Possible GLAMOR replacement with a leaner EGL-only path.** GLAMOR was designed as a general-purpose X11 acceleration layer using OpenGL. In the context of XWayland, where GLAMOR is used only for DMA-BUF allocation and EGLImage-backed pixmaps, a leaner replacement that omits the OpenGL drawing paths has been speculatively discussed to reduce XWayland's dependency surface. Note: speculative, no concrete proposal exists as of mid-2026.
- **xwayland-satellite as the default model.** If Wayland compositor authors converge on `xdg_wm_base` v5 as a sufficient substrate for X11 window management, `xwayland-satellite`'s model — decoupling XWM from the compositor process — could become the standard approach, eliminating the need for each compositor to embed a full XWM. This would simplify the compositor development surface at the cost of reduced integration depth.
- **XWayland as the reference for X11 security auditing.** As Xorg is removed from major distributions, XWayland becomes the only maintained X11 server implementation. Long-term, security researchers and distributions may invest in formal auditing of XWayland's X11 protocol handling, since a vulnerability in XWayland's X11 parser now affects all Linux desktop users running X11 applications under Wayland.

---

## Integrations

**Ch4 — GBM and the Wayland Compositor Buffer Pipeline.** The Wayland compositor receives `wl_buffer` objects backed by DMA-BUF FDs from XWayland's `zwp_linux_dmabuf_v1` path. The compositor imports these via `gbm_bo_import(GBM_BO_IMPORT_FD_MODIFIER)` or `eglCreateImageKHR(EGL_LINUX_DMA_BUF_EXT)` for use in its scene graph. With implicit sync, `wl_buffer.release` signals XWayland when the buffer can be reused. With explicit sync (`linux-drm-syncobj-v1`, Ch75), the release signal becomes a DRM syncobj timeline point rather than a Wayland event.

**Ch20 — Wayland Protocol Fundamentals and linux-dmabuf.** XWayland uses `zwp_linux_dmabuf_v1` to submit DMA-BUF backed `wl_buffer` objects. The `zwp_linux_buffer_params_v1` interface assembles multi-plane buffers from per-plane `(fd, offset, stride, modifier)` tuples. Format/modifier negotiation via `zwp_linux_dmabuf_v1_feedback` ensures XWayland allocates GBM buffers in formats the compositor can scanout directly. XWayland's `xwl_glamor_gbm_get_wl_buffer_for_pixmap()` is the function that bridges GLAMOR pixmaps to Wayland buffer objects.

**Ch22 — Wayland Input Handling.** XWayland consumes `wl_seat` capabilities (wl_pointer, wl_keyboard, wl_touch), translating Wayland event sequences to X11 input events. The `zwp_relative_pointer_v1` protocol provides unaccelerated relative motion deltas for games. `zwp_pointer_constraints_v1` enables the pointer lock used when X11 hides the cursor. The keyboard grab protocol (`zwp_xwayland_keyboard_grab_unstable_v1`) is a Wayland extension exclusive to XWayland.

**Ch75 — Explicit GPU Synchronisation.** `linux-drm-syncobj-v1` is directly implemented by XWayland 24.1+ as described in the Explicit Synchronisation section above. The `wp_linux_drm_syncobj_surface_v1` per-surface interface coordinates GPU pipeline timing between XWayland and the compositor, solving the NVIDIA implicit sync problem that caused flickering in many applications.

**Ch95 — X11/Xorg Legacy.** XWayland is the mechanism by which the X11/Xorg application ecosystem is preserved under Wayland compositors. Full Xorg requires KMS/DRM modesetting and `xf86-input-*` input drivers; XWayland bypasses all of this, using Wayland for display and input instead. The historical arc: Xorg (1984 origins) → XWayland prototype by Kristian Høgsberg (2013) → production rootless XWayland in Weston and Mutter → standalone XWayland package (2022) → on-demand XWayland in compositors → Fedora 40 removing Xorg from the default GNOME session (2024). Understanding XWayland's architecture makes clear why X11 sunset is achievable: all meaningful X11 functionality is now brokered through the Wayland protocol stack, with XWayland acting as a translation layer rather than a native display path.

**Ch73 — Asahi and Apple Silicon.** The AGX GPU driver on Apple Silicon must support the DMA-BUF / DRI3 path for XWayland to provide hardware-accelerated rendering. The `zwp_linux_dmabuf_v1` format/modifier feedback mechanism is the critical link: XWayland negotiates supported format/modifier combinations with the Asahi compositor to ensure GBM buffer allocation produces buffers that can be displayed without a copy.

**Ch138 — Wayland Fractional Scaling and HiDPI.** XWayland's fractional scale support (GNOME 46+, via `wp_fractional_scale_v1`) builds on the infrastructure described in Ch138. X11 applications that do not implement fractional DPI will blur at non-integer scales — the fundamental limitation that makes native Wayland porting desirable. The `wp_viewporter` protocol used by xwayland-satellite for scaling is the same protocol used in Ch138's fractional scale compositor implementation.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
