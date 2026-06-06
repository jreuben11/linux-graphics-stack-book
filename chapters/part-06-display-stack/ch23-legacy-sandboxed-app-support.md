# Chapter 23: Legacy and Sandboxed App Support

**Audiences**: Systems and driver developers needing to understand XWayland's architecture, clipboard bridging, and explicit synchronization; application developers needing to understand how X11 and sandboxed applications interact with the Wayland compositor; and operators responsible for deploying Flatpak applications who need to reason about GPU access security trade-offs.

---

## Table of Contents

1. [The Compatibility Problem](#1-the-compatibility-problem)
2. [XWayland Architecture](#2-xwayland-architecture)
3. [Glamor: GPU-Accelerated X11 Rendering](#3-glamor-gpu-accelerated-x11-rendering)
4. [Clipboard, Selection, and Drag-and-Drop Bridging](#4-clipboard-selection-and-drag-and-drop-bridging)
5. [DPI Scaling and HiDPI Under XWayland](#5-dpi-scaling-and-hidpi-under-xwayland)
6. [Explicit Sync for XWayland](#6-explicit-sync-for-xwayland)
7. [xdg-desktop-portal: Architecture](#7-xdg-desktop-portal-architecture)
8. [Portal: Screen Cast and Remote Desktop](#8-portal-screen-cast-and-remote-desktop)
9. [Portal: GPU Surface Sharing with Sandboxed Apps](#9-portal-gpu-surface-sharing-with-sandboxed-apps)
10. [Running X11 Applications on Wayland: Practical Guide](#10-running-x11-applications-on-wayland-practical-guide)
11. [Flatpak GPU Access: Security Model and Current Limitations](#11-flatpak-gpu-access-security-model-and-current-limitations)
12. [Integrations](#12-integrations)
13. [References](#13-references)

---

## 1. The Compatibility Problem

The Wayland protocol has been the default session type on major Linux distributions since GNOME 3.22 (2016) and KDE Plasma 5.24 (2022), yet two large classes of applications cannot connect directly to a Wayland compositor. The first class comprises legacy X11 applications: productivity suites, scientific software such as MATLAB and Mathematica, games running under Wine or Proton, older toolkit versions (GTK2, Qt4, and many custom toolkits), and specialised tools that depend on X11 extensions with no Wayland equivalent — `XTest` for synthetic input, `RECORD` for input monitoring, and `XGrabKey` for global hotkey registration. These applications were written to the X11 protocol and cannot be recompiled without major engineering effort. The second class comprises sandboxed applications distributed via Flatpak, Snap, or OCI container formats, which have explicit restrictions on their access to host system resources: the Flatpak sandbox by default denies access to `/dev/dri`, the Wayland socket, PulseAudio, and the file system outside a limited set of allowed directories.

The solution to the first problem is XWayland: a full X11 server that runs as a Wayland client, presenting X11 window content to the compositor as standard `wl_surface` objects while accepting connections from X11 applications on a Unix domain socket. From the perspective of an X11 application, XWayland is indistinguishable from a native X11 server. From the perspective of the Wayland compositor, XWayland is a privileged Wayland client that manages a set of surfaces.

The solution to the second problem is `xdg-desktop-portal`: a D-Bus service that runs outside the sandbox and provides mediated access to compositor services — screen capture, file selection, camera access, and GPU context creation (the last of which remains under development as of 2025). Sandboxed applications call portal methods over D-Bus, which the portal backend translates into compositor-specific operations without giving the application direct access to kernel devices.

These two mechanisms interact: a Flatpak-sandboxed application may itself be an X11 application running through XWayland. In that case, the X11 Unix socket is proxied into the sandbox via the Flatpak `--socket=x11` permission, and the XWayland server itself runs outside the sandbox. The portal and XWayland layers stack orthogonally. A sandboxed X11 application that relies on basic 2D drawing goes through the proxy socket to XWayland, where Glamor accelerates rendering entirely within the XWayland process (which has full GPU access). A sandboxed X11 application that uses OpenGL or Vulkan additionally requires the `--device=dri` permission so that Mesa's DRI3 client code inside the sandbox can open the DRM render node directly.

---

## 2. XWayland Architecture

XWayland simultaneously occupies two roles: it is an X11 server, holding the full DIX/DDX split state machine of the X Window System and accepting connections on a Unix domain socket; and it is a Wayland client, maintaining a `wl_display` connection to the compositor and creating `wl_surface` objects on behalf of each X11 top-level window. The code implementing this duality lives in `xserver/hw/xwayland/` in the X.Org Server repository [Source](https://gitlab.freedesktop.org/xorg/xserver).

### Startup Sequence

The XWayland startup sequence involves a careful handshake between the compositor and the server. The compositor (or systemd socket activation for Wayland socket-pair activation) creates a listening Unix socket for X11 clients, typically `/tmp/.X11-unix/X0`, and passes the file descriptor to XWayland via the `-displayfd` argument. XWayland starts with `-rootless` to enable the window-per-surface model, and with `-wm` to specify the Wayland socket to connect to. The key startup sequence is:

1. XWayland connects to the compositor as a Wayland client and binds the `xwayland_shell_v1` or `xdg_wm_base` global depending on compositor version.
2. XWayland registers the `_XWAYLAND_COMPOSITOR_CALLBACK` X11 property atom to signal readiness to the compositor's XWM (X Window Manager) component.
3. The compositor spawns an internal XWM client (in wlroots, this is `wlr_xwayland`; in Mutter it is the built-in XWM) that connects to XWayland's X11 socket and monitors for `MapRequest`, `ConfigureRequest`, and `ClientMessage` events.
4. XWayland writes the display number to `displayfd` to signal readiness; the compositor notifies X11 clients of the available display.

From `xserver/hw/xwayland/xwayland.c`, the core initialisation sequence calls `xwl_screen_init()`, which sets up the EGL context via Glamor and binds the Wayland globals needed for buffer presentation:

```c
/* xserver/hw/xwayland/xwayland.c */
static Bool
xwl_screen_init(ScreenPtr pScreen, int argc, char **argv)
{
    struct xwl_screen *xwl_screen;

    xwl_screen = xwl_screen_get(pScreen);
    /* ... */
    xwl_screen->wl_display = wl_display_connect(NULL);
    xwl_screen->registry = wl_display_get_registry(xwl_screen->wl_display);
    wl_registry_add_listener(xwl_screen->registry,
                             &registry_listener, xwl_screen);
    wl_display_roundtrip(xwl_screen->wl_display);
    /* bind wl_compositor, wl_shm, zwp_linux_dmabuf, wp_linux_drm_syncobj_manager, etc. */
}
```

### Rootless vs. Full-Screen Mode

In **rootless mode** (the default), each X11 top-level window becomes its own `wl_surface`, and the compositor's window manager treats these surfaces as distinct toplevels, applying decorations, tiling rules, and focus policy just as it would for native Wayland clients. Override-redirect windows — menus, tooltips, popup decorations — are assigned `wl_surface` objects without an `xdg_toplevel` role; compositors handle these as unmanaged surfaces at the requested positions.

In **full-screen mode** (rootful or non-rootless mode), XWayland runs inside a single Wayland surface. The X11 root window maps to this surface; all X11 children are composited together inside it. This is the mode used by gamescope for Proton/Steam games, where the compositor wants exclusive control of a full-screen window with the entire X11 session inside it. Full-screen mode is architecturally simpler (single surface, no need to map individual windows) but loses per-window compositor management.

### The X11–Wayland Window Mapping

The `xwl_window` structure (in `xserver/hw/xwayland/xwayland-window.c`) tracks the association between an X11 `Window` ID and its corresponding Wayland `wl_surface`. When an X11 client calls `XMapWindow()`, the DIX layer calls the `RealizeWindow` hook, which triggers `xwl_realize_window()`. This function allocates the `xwl_window`, creates a `wl_surface` via `wl_compositor_create_surface()`, and sends the surface's object ID back to the X11 window as a `WL_SURFACE_ID` ClientMessage. The compositor's XWM component receives this ClientMessage and associates the surface with the compositor's window record.

```c
/* xserver/hw/xwayland/xwayland-window.c */
static Bool
xwl_realize_window(WindowPtr window)
{
    struct xwl_window *xwl_window;
    struct xwl_screen *xwl_screen = xwl_screen_get(window->drawable.pScreen);

    xwl_window = calloc(1, sizeof *xwl_window);
    xwl_window->window = window;
    xwl_window->surface = wl_compositor_create_surface(xwl_screen->compositor);

    /* Notify XWM about the surface<->window association */
    xwl_window_send_wl_surface_id(xwl_window, window);

    if (window->redirectDraw == RedirectDrawManual)
        xwl_window_init_shm_pixmap(xwl_window);

    return TRUE;
}
```

The `_NET_WM_PID`, `_NET_WM_NAME`, and `WM_CLASS` atoms from the X11 window are relayed to the compositor via `xdg_toplevel_set_title()` and `xdg_toplevel_set_app_id()`, enabling the compositor to apply window management policy to X11 applications just as it does for native Wayland clients.

### Input Focus Bridging

Input focus in Wayland is controlled by the compositor through the `wl_keyboard.enter`/`wl_keyboard.leave` events delivered to the active surface. XWayland receives these events as a Wayland client and must translate them into the X11 focus model: calling `SetInputFocus()` to direct keyboard input to the correct X11 client window. The focus bridging code in `xserver/hw/xwayland/xwayland-input.c` manages this translation, including handling the case where an X11 client attempts `XGrabKeyboard()` — a grab that must be mediated through XWayland's Wayland seat rather than being allowed to propagate as a true keyboard grab.

---

## 3. Glamor: GPU-Accelerated X11 Rendering

Glamor is the OpenGL ES 2.0 / OpenGL 3.1 implementation of the X11 rendering model that runs inside the DDX (Device-Dependent X) layer of the X server. Rather than implementing 2D drawing operations in software or requiring driver-specific acceleration hooks, Glamor translates X11 Pixmap operations into GLES2 draw calls, effectively using the GPU's shader pipeline as a general-purpose 2D accelerator. XWayland uses Glamor as its exclusive DDX; all X11 drawing ultimately goes through Glamor. [Source](https://www.x.org/wiki/Development/Documentation/Glamor/)

### EGL Context Initialisation

Glamor initialises an EGL context using `EGL_KHR_surfaceless_context` (no display surface is needed since rendering goes into GEM-backed Pixmaps). The key initialisation path is in `xserver/glamor/glamor_egl.c`:

```c
/* xserver/glamor/glamor_egl.c */
Bool
glamor_egl_init(ScreenPtr screen, int fd)
{
    struct glamor_egl_screen_private *glamor_egl;

    glamor_egl = glamor_egl_get_screen_private(screen);
    glamor_egl->display = glamor_egl_get_display(EGL_PLATFORM_GBM_MESA, fd);
    /* ... */
    glamor_egl->context = eglCreateContext(glamor_egl->display,
                                           NULL, /* no config needed */
                                           EGL_NO_CONTEXT,
                                           context_attribs);
    eglMakeCurrent(glamor_egl->display, EGL_NO_SURFACE,
                   EGL_NO_SURFACE, glamor_egl->context);

    glamor_init(screen, GLAMOR_USE_EGL_SCREEN);
    return TRUE;
}
```

The `xwl_glamor_egl_make_current()` wrapper in `xserver/hw/xwayland/xwayland-glamor.c` optimises context switching by tracking the last active context and skipping redundant `eglMakeCurrent()` calls.

### Pixmap Allocation and GEM Backing

When an X11 client creates a window or Pixmap, Glamor must allocate a GPU-resident buffer for it. This path goes through `glamor_create_pixmap()`, which calls into the EGL backend to allocate a GEM buffer object via `gbm_bo_create()`, then wraps it as an `EGLImage` using `EGL_KHR_image_base`. The resulting `EGLImage` is imported into OpenGL as a texture via `GL_OES_EGL_image`, giving the GLES2 renderer read/write access to the backing GEM buffer.

For the DMA-BUF export path — needed when XWayland must present a Pixmap as a Wayland buffer — the GEM buffer's DMA-BUF file descriptor is obtained via `gbm_bo_get_fd()` and then wrapped in a `wl_buffer` using `zwp_linux_dmabuf_v1_create_params`. The format and modifier negotiation uses the `zwp_linux_dmabuf_v1` feedback mechanism (protocol version 4) to ensure the compositor receives DMA-BUF buffers in a format it can scanout or composite without additional copies. The per-window feedback path in `xwl_dmabuf_setup_feedback_for_window()` handles this negotiation on a per-surface basis.

### The X11 Rendering Pipeline Through Glamor

Consider an X11 client calling `XFillRectangle()`. The call travels from the client over the X11 protocol to XWayland's DIX layer, which validates the GC (Graphics Context) parameters and calls the DDX `PolyFillRect()` hook. Glamor's hook, `glamor_poly_fill_rect()` in `xserver/glamor/glamor_fill.c`, translates the rectangle list into VBO geometry (four vertices per rectangle), selects the appropriate fragment shader (solid fill, tile, stipple), sets up the framebuffer object with the target Pixmap's texture as the attachment, and issues a `glDrawArrays()` call. The GPU executes the draw call and the result lands in the GEM-backed texture backing the Pixmap.

For XRender COMPOSITE operations — the primary rendering path used by Cairo, Pango, and GTK's drawing code — Glamor's `glamor_composite_rects()` in `xserver/glamor/glamor_render.c` constructs a textured quad, selects a fragment shader encoding the XRender compositing operator (Over, Add, Xor, etc.), and issues a `glDrawArrays()` call with the source and mask textures bound. The XRender operators map cleanly to GLES2 blending and arithmetic operations.

### DRI3 and the Present Extension

When an X11 client uses hardware-accelerated OpenGL via GLX (`glXSwapBuffers()`) or EGL (`eglSwapBuffers()`), it follows a different path that bypasses Glamor entirely. Mesa's GLX driver (`xserver/glamor/` and `src/glx/` in the Mesa repository) allocates GPU buffers via DRI3 (`DRI3Open`, `DRI3PixmapFromBuffers`), renders directly into them, and then uses the X11 Present extension to deliver the completed buffer to XWayland's window. XWayland receives this Present request and converts the GEM-backed Pixmap into a DMA-BUF for submission to the compositor as a `wl_buffer`. This path is zero-copy from Mesa's framebuffer to the compositor: no pixel data is ever read back to CPU.

### Performance Characteristics

Glamor is highly efficient for large compositing operations (full-window composites, large texture blits) because the GLES2 draw call overhead is amortised over many pixels. It is poor for large numbers of small drawing operations — many `XFillRectangle()` calls covering tiny rectangles — because each call incurs GL command submission overhead. This asymmetry explains a common pattern: GTK3/GTK4 applications that use Cairo for 2D rendering perform well under XWayland (Cairo uses large XRender COMPOSITE operations that map to single GLES2 draws), while older applications using small XFillRectangle loops for custom drawing suffer visible slowdown. The `XWAYLAND_NO_GLAMOR=1` environment variable disables Glamor entirely and falls back to software rendering via XShm, which is slower for all operations but eliminates the GPU overhead for pathological small-draw workloads.

---

## 4. Clipboard, Selection, and Drag-and-Drop Bridging

### The Two Clipboard Models

The X11 clipboard model is based on selections: named atoms (`CLIPBOARD`, `PRIMARY`, `SECONDARY`) with an owner (`XSetSelectionOwner`). When a receiver wants the selection contents, it calls `XConvertSelection()`, and the selection owner responds asynchronously via `SelectionNotify` events, delivering data in a negotiated format (MIME type). For large payloads, the INCR protocol allows incremental delivery in chunks.

The Wayland clipboard model operates through the `wl_data_device_manager` global. A Wayland client offering data creates a `wl_data_source`, announces MIME types, and calls `wl_data_device_set_selection()`. A receiving client gets a `wl_data_offer` when focus enters its surface, and reads data through a file descriptor pipe.

### The XWayland Bridge

XWayland must act as a bidirectional bridge between these two models. The clipboard bridge lives in `xserver/hw/xwayland/xwayland-clipboard.c` and manages a `xwl_clipboard_manager` that monitors both X11 selection ownership changes and Wayland clipboard events.

When an X11 client takes ownership of the CLIPBOARD selection (via `XSetSelectionOwner()`), XWayland's `xwl_clipboard_x11_owner_notify()` handler fires. XWayland creates a `wl_data_source` offering the selection's available target formats (obtained via the `TARGETS` atom conversion) and calls `wl_data_device_set_selection()` to notify the compositor. When a Wayland client then requests a paste, the compositor signals `wl_data_source.send` with a MIME type and a pipe file descriptor. XWayland receives this event and satisfies it by calling `XConvertSelection()` on the X11 side, writing the resulting data into the pipe.

The reverse direction: when a Wayland client places data on the clipboard, the compositor sends `wl_data_device.selection` to XWayland's `wl_data_device`. XWayland creates an internal X11 selection owner proxy, calls `XSetSelectionOwner()` to claim CLIPBOARD on behalf of the Wayland client, and when any X11 client subsequently requests a `SelectionNotify`, XWayland reads from the `wl_data_offer` pipe and delivers the data to the requesting X11 client.

### PRIMARY Selection

The PRIMARY selection (middle-click paste in X11) is bridged via the `zwp_primary_selection_device_manager_v1` Wayland protocol on the Wayland side. XWayland monitors X11 PRIMARY selection ownership changes and mirrors them to `zwp_primary_selection_source_v1` objects, and vice versa. Applications that depend on X11 PRIMARY selection behaviour will find it working correctly in modern XWayland versions, though with the caveat that not all compositors implement `zwp_primary_selection_device_manager_v1`.

### Drag-and-Drop

Cross-application drag-and-drop between X11 and Wayland applications requires XWayland to bridge the X11 XDND (X Drag-and-Drop) protocol and the Wayland `wl_data_device` drag protocol. This is significantly more complex than clipboard because drag-and-drop involves pointer motion, enter/leave events, and negotiated data delivery — all of which have different semantics in X11 and Wayland. XDND uses `XdndStatus`, `XdndDrop`, and `XdndFinished` ClientMessages with window IDs; Wayland uses `wl_data_device.enter`/`motion`/`drop`/`leave` events associated with surfaces. XWayland's drag bridge translates between these, but MIME type conversion (particularly Wayland URIs vs. X11 `text/uri-list`) can be lossy.

Known limitations of the clipboard bridge include: the INCR protocol for large X11 clipboard transfers is imperfectly supported, leading to truncated pastes for very large selections; some MIME type conversions lose metadata; and rapid paste operations can hit race conditions where selection ownership changes outpace bridge synchronisation.

---

## 5. DPI Scaling and HiDPI Under XWayland

### The Core Incompatibility

X11 uses physical pixels throughout: window sizes, font metrics, and drawing coordinates are all in physical pixels. Wayland uses logical coordinates with a scale factor communicated to clients via `wl_output.scale` (integer scaling) or the `wp_fractional_scale_v1` protocol (fractional scaling). An X11 application running under XWayland therefore has no inherent mechanism to query the logical scale factor of the compositor's output — it sees XWayland's virtual display, whose resolution and DPI XWayland chooses at startup.

### Scaling Approaches

**`Xwayland -dpi <value>`**: Overrides the DPI reported to X11 clients, affecting font sizing via Xft and the `DisplayWidth`/`DisplayHeight` metrics. Setting `-dpi 192` on a 4K display tells X11 applications to size fonts and UI elements for a 192 DPI display. This is the simplest approach but does not affect window or widget layout — the application still renders at physical pixels, and the compositor scales the resulting surface.

**Toolkit scaling (`GDK_SCALE=2`, `QT_SCALE_FACTOR=2`)**: These environment variables instruct GTK and Qt to render at 2x logical size. The application itself scales its widgets; the result is a surface twice as large as needed, then displayed at 1:1. This produces sharp rendering on HiDPI displays but is toolkit-specific.

**XWayland's `-scale` flag**: A relatively recent addition that makes XWayland render X11 content at the specified scale factor and then present a logically-scaled Wayland surface. At `-scale 2`, XWayland tells X11 applications the display is 2x larger than it is logically and then presents the surface at half the size to the compositor. This achieves integer HiDPI scaling for X11 applications without toolkit changes.

**`wp_fractional_scale_v1`**: XWayland, as a Wayland client, can receive the fractional scale factor from the compositor via this protocol. When the compositor outputs a 1.5x scale factor, XWayland can adapt its virtual display metrics accordingly, though fractional scaling for X11 applications remains imperfect because X11's pixel-exact coordinate system does not accommodate non-integer scales gracefully.

### Per-Output Scaling and the Mixed DPI Problem

When a laptop has one HiDPI internal display (2x) and one external standard-DPI display (1x), the compositor presents two outputs with different scale factors. XWayland monitors Wayland output events (`wl_output.scale`, `wl_output.mode`) and updates its RandR output properties accordingly in `xserver/hw/xwayland/xwayland-output.c`. Applications querying `XRandR` will see scale changes when windows move between outputs, but many X11 applications cache the initial DPI and do not re-query it — so DPI changes on window migration are often ignored.

The `Xft.dpi` resource (set via `~/.Xresources` and loaded by `xrdb`) is the universally respected mechanism for informing X11 toolkits about display DPI. Applications that honour `Xft.dpi` will scale correctly regardless of which scaling approach is used. The recommended approach for a HiDPI setup is to set `Xft.dpi: 192` in `~/.Xresources` and use `xrdb -merge ~/.Xresources` at session start.

---

## 6. Explicit Sync for XWayland

### The Implicit Sync Problem

The fundamental issue with XWayland rendering under Wayland is synchronisation: a GPU buffer passed from XWayland to the compositor as a `wl_buffer` may not yet have its rendering completed. When Mesa or the NVIDIA driver renders into a buffer, the rendering commands are enqueued on a GPU command stream; the CPU-side call returns before the GPU finishes. Under implicit synchronisation — the mechanism used by Linux's DMA-BUF with `DMA_BUF_IOCTL_SYNC` and the in-kernel fence tracking — the kernel tracks completion fences and ensures that a buffer consumer (the compositor's KMS scanout) waits for the producer (Mesa's command stream) to complete before using the buffer.

The NVIDIA proprietary driver historically did not participate in the kernel's implicit fence framework. When XWayland submitted a `wl_buffer` backed by a GEM buffer that NVIDIA's driver had rendered into, the kernel had no mechanism to wait for NVIDIA's GPU command stream to complete. The result was tearing and corruption: the compositor would composite the buffer before NVIDIA had finished writing to it. Earlier workarounds required XWayland to perform a CPU-side flush — calling `glFinish()` before submitting the buffer — which serialised rendering and introduced significant latency. [Source](https://lwn.net/Articles/937565/)

### The `xwayland-explicit-synchronization` Protocol

The `xwayland_explicit_synchronization_v1` protocol (drafted 2022) provided an early partial solution: it allowed the compositor to supply a release fence to XWayland that XWayland would pass to the X11 client's EGL driver as the "back buffer available" signal. This ensured the client would not start rendering into a buffer the compositor was still reading. It solved the consumer-side issue but did not fully solve the producer-side issue for NVIDIA.

### `wp_linux_drm_syncobj_v1`: The Complete Solution

The `linux-drm-syncobj-v1` protocol (merged into wayland-protocols 1.34 in March 2024 [Source](https://www.gamingonlinux.com/2024/03/explicit-sync-wayland-protocol-merged-wayland-protocols-1-34-released/)) provides a complete explicit synchronisation mechanism based on DRM synchronisation object timelines. A syncobj timeline is a monotonically increasing counter; clients signal a timeline point when GPU work is complete and wait on a timeline point before beginning new work.

XWayland, as a Wayland client, binds `wp_linux_drm_syncobj_manager_v1` from the compositor's global registry. For each `wl_surface` it manages, it creates a `wp_linux_drm_syncobj_surface_v1` extension object and uses it to communicate synchronisation points per commit:

```c
/* xserver/hw/xwayland/xwayland-surface.c */
static void
xwl_present_set_sync_points(struct xwl_window *xwl_window,
                             struct xwl_present_window *present_window)
{
    struct wp_linux_drm_syncobj_surface_v1 *sync_surface =
        xwl_window->drm_syncobj_surface;

    /* acquire point: compositor must wait before compositing */
    wp_linux_drm_syncobj_surface_v1_set_acquire_point(
        sync_surface,
        present_window->acquire_timeline,
        (uint32_t)(present_window->acquire_point >> 32),
        (uint32_t)(present_window->acquire_point & 0xFFFFFFFF));

    /* release point: compositor signals when done with buffer */
    wp_linux_drm_syncobj_surface_v1_set_release_point(
        sync_surface,
        present_window->release_timeline,
        (uint32_t)(present_window->release_point >> 32),
        (uint32_t)(present_window->release_point & 0xFFFFFFFF));
}
```

The acquire timeline point is signalled by Mesa's GLX driver (or NVIDIA's EGL driver) when the GPU finishes rendering. The release timeline point is signalled by the compositor when it is done reading the buffer, allowing the client to reuse it. This eliminates both the tearing problem (compositor waits for acquire) and the frame pacing problem (client waits for release before re-rendering).

### Version Requirements

Full `wp_linux_drm_syncobj_v1` support for XWayland requires:
- **xserver 24.1** (2024) or later for XWayland's producer side
- **wayland-protocols 1.34** (March 2024) for the protocol definition
- **KWin 6.1** (2024) for the compositor consumer side [Source](https://invent.kde.org/plasma/kwin/-/merge_requests/4693)
- **Mutter 46** (GNOME 46, 2024) for the GNOME compositor consumer side
- **Sway 1.9** (August 2024) for the Sway/wlroots compositor consumer side [Source](https://www.phoronix.com/news/Sway-Explicit-Sync-Merged)
- **NVIDIA driver 555** (May 2024) for the producer side on NVIDIA hardware
- **Mesa 24.x** for the open-source driver producer side

Compositors and drivers that do not yet include complete syncobj support fall back to the older implicit-sync or `glFinish()` paths. Systems running NVIDIA driver 555+ with KWin 6.1+ or Mutter 46+ will have fully correct explicit sync for both XWayland and native Wayland clients.

---

## 7. xdg-desktop-portal: Architecture

`xdg-desktop-portal` is a D-Bus service that provides a cross-compositor, cross-session API for sandboxed applications to access system resources that would otherwise require direct file system or device access. The project lives at [github.com/flatpak/xdg-desktop-portal](https://github.com/flatpak/xdg-desktop-portal) and its documentation at [flatpak.github.io/xdg-desktop-portal](https://flatpak.github.io/xdg-desktop-portal/docs/).

### The Two-Process Model

The portal is split into a frontend process and one or more backend processes:

**Frontend (`xdg-desktop-portal`)**: Exposes the `org.freedesktop.portal.*` well-known D-Bus names. Handles app-ID verification, enforces per-app permission storage (in `~/.local/share/xdg-desktop-portal/permissions.db`), and routes requests to the appropriate backend. The frontend is desktop-agnostic.

**Backend processes**: Each desktop environment ships its own backend: `xdg-desktop-portal-gnome` for GNOME/Mutter, `xdg-desktop-portal-kde` for KDE Plasma/KWin, `xdg-desktop-portal-wlr` for wlroots-based compositors (Sway, Hyprland via its own backend `xdg-desktop-portal-hyprland`). Backends implement the `org.freedesktop.impl.portal.*` D-Bus names — the internal backend interface. They perform the actual compositor-specific operations: showing a window selection dialog, capturing a screen, injecting input.

### D-Bus Activation

Portals are activated on demand via systemd socket activation. When a sandboxed application calls a portal method, systemd activates the portal service, which starts the frontend process, which in turn activates the appropriate backend. The `org.freedesktop.portal.Desktop` and the individual portal names (`org.freedesktop.portal.ScreenCast`, etc.) are all well-known names that a running process holds.

### The Request/Response Pattern

Portal operations that require user interaction follow an asynchronous Request/Response pattern. Each portal method call returns a D-Bus object path representing a `Request` object. The calling application registers a signal handler on `org.freedesktop.portal.Request.Response` at that path before the call returns. When the portal operation completes (user closes the dialog, grants or denies permission), the portal signals `Response(uint32 response, dict results)` where `response` is 0 (success), 1 (user cancelled), or 2 (error). This pattern is necessary because portal operations often involve modal dialogs that may take seconds or minutes to complete.

```c
/* Example: registering for the Response signal before calling CreateSession */
DBusConnection *conn = dbus_bus_get(DBUS_BUS_SESSION, NULL);
/* Register signal match before the method call to avoid races */
dbus_bus_add_match(conn,
    "type='signal',interface='org.freedesktop.portal.Request',"
    "member='Response'", NULL);

/* Call CreateSession */
DBusMessage *msg = dbus_message_new_method_call(
    "org.freedesktop.portal.Desktop",
    "/org/freedesktop/portal/desktop",
    "org.freedesktop.portal.ScreenCast",
    "CreateSession");
/* ... append options dict ... */
```

### App-ID Verification

The portal determines the requesting application's identity by querying the kernel credentials of the D-Bus connection's process. For Flatpak apps, the sandbox infrastructure exposes the app-ID (e.g., `com.obsproject.Studio`) through the `org.freedesktop.Flatpak` D-Bus service. For non-sandboxed processes, the app-ID is derived from the process executable name. This identity is used to look up and store per-app portal permissions.

Flatpak 1.16 introduced the ability to create a private Wayland socket using `wp_security_context_v1`, which allows the compositor to correctly identify connections from sandboxed applications at the Wayland protocol level, complementing the D-Bus-level identification that portals use.

### Backend Selection

The portal frontend selects a backend by examining the `interfaces` property of registered `PortalImplementation` objects. If multiple backends implement the same portal interface, the frontend uses a priority system based on the current desktop environment (detected via `XDG_CURRENT_DESKTOP`). On GNOME, `xdg-desktop-portal-gnome` handles all portals; on Sway, `xdg-desktop-portal-wlr` handles screen cast while a separate backend handles other portals.

---

## 8. Portal: Screen Cast and Remote Desktop

### `org.freedesktop.portal.ScreenCast`

The ScreenCast portal provides an application-level API for screen recording, virtual cameras, and remote desktop software to capture display output. The complete flow consists of four method calls [Source](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html):

**`CreateSession(options: a{sv}) → handle: o`**
Creates a screen cast session. The session handle is delivered in the `Response` signal under the key `session_handle`. The session is a persistent object that subsequent calls operate on.

**`SelectSources(session_handle: o, options: a{sv}) → handle: o`**
Configures what will be captured before starting. Key options:
- `types` (uint32, bitmask): `MONITOR=1`, `WINDOW=2`, `VIRTUAL=4` — what source types to offer the user
- `multiple` (bool): whether the user can select multiple sources
- `cursor_mode` (uint32): `Hidden=1`, `Embedded=2` (cursor baked into frame), `Metadata=4` (cursor position as PipeWire stream metadata)
- `persist_mode` (uint32): whether to remember the user's selection for future sessions
- `restore_token` (string): single-use token from a previous session to restore the source selection without showing a dialog

**`Start(session_handle: o, parent_window: s, options: a{sv}) → handle: o`**
Begins the capture. If no `restore_token` is in effect, the compositor backend shows a source selection dialog. The `Response` signal delivers `streams`: an array of `(node_id: u, properties: a{sv})` tuples. Each stream has a `pipewire-serial` (preferred over `node_id` for targeting across reconnections) and optionally `position`, `size`, and `source_type`.

**`OpenPipeWireRemote(session_handle: o, options: a{sv}) → fd: h`**
Returns a file descriptor to the PipeWire remote. The application passes this fd to `pw_context_connect_fd()` to establish a PipeWire connection, then creates a `pw_stream` targeting the node identified by `pipewire-serial`.

### The PipeWire DMA-BUF Zero-Copy Path

When both the portal backend and the PipeWire consumer support DMA-BUF buffer transport, screen capture frames travel from the compositor's framebuffer to the consumer's GPU texture without any CPU copy:

1. **Compositor capture**: The compositor backend (e.g., `xdg-desktop-portal-gnome` using Mutter's screencasting D-Bus API, or `xdg-desktop-portal-wlr` using `zwlr_screencopy_manager_v1`) captures a completed frame. In the DMA-BUF path, the compositor exports the frame as a DMA-BUF fd without reading back pixels to CPU.

2. **PipeWire buffer**: The portal backend advertises `SPA_DATA_DmaBuf` buffer type on the PipeWire stream. Consumers that support DMA-BUF import (OBS Studio, WebRTC in Chromium/Firefox, gstreamer with `dmabufmeta`) negotiate this type during stream parameter setup.

3. **Consumer import**: The consumer receives a PipeWire buffer with `spa_data.type = SPA_DATA_DmaBuf` and `spa_data.fd` containing the DMA-BUF file descriptor. To use it as a GPU texture, the consumer calls:

```c
/* Importing a PipeWire DMA-BUF frame into EGL */
EGLint attribs[] = {
    EGL_WIDTH,                  frame_width,
    EGL_HEIGHT,                 frame_height,
    EGL_LINUX_DRM_FOURCC_EXT,   DRM_FORMAT_XRGB8888,
    EGL_DMA_BUF_PLANE0_FD_EXT,      spa_data->fd,
    EGL_DMA_BUF_PLANE0_OFFSET_EXT,  0,
    EGL_DMA_BUF_PLANE0_PITCH_EXT,   spa_data->chunk->stride,
    EGL_DMA_BUF_PLANE0_MODIFIER_LO_EXT, modifier & 0xFFFFFFFF,
    EGL_DMA_BUF_PLANE0_MODIFIER_HI_EXT, modifier >> 32,
    EGL_NONE
};
EGLImageKHR img = eglCreateImageKHR(egl_display, EGL_NO_CONTEXT,
                                     EGL_LINUX_DMA_BUF_EXT,
                                     NULL, attribs);
glBindTexture(GL_TEXTURE_2D, texture_id);
glEGLImageTargetTexture2DOES(GL_TEXTURE_2D, img);
```

If the consumer does not support DMA-BUF import, PipeWire falls back to a memory-mapped `mmap()` buffer (`SPA_DATA_MemFd`), which requires one CPU copy per frame from the compositor's buffer to the shared memory region. OBS Studio versions prior to 27.2 used the memory-mapped path; OBS 28+ supports DMA-BUF import.

### Window-Level vs. Output-Level Capture

An important limitation of the wlroots-based portal backend (`xdg-desktop-portal-wlr`) is that `zwlr_screencopy_manager_v1` only supports output-level capture — it captures an entire monitor, not individual windows. The `AvailableSourceTypes` property on `xdg-desktop-portal-wlr` therefore does not include `WINDOW` (type 2).

The GNOME portal backend (`xdg-desktop-portal-gnome`) uses a Mutter-specific screen recording D-Bus interface (`org.gnome.Mutter.ScreenCast`) that supports individual window capture. This means that video conferencing software (Zoom, Teams via Wayland portals, Google Meet in Chromium) can offer "Share a window" as opposed to "Share a screen" only when running under GNOME/Mutter.

### `org.freedesktop.portal.RemoteDesktop`

The `RemoteDesktop` portal extends `ScreenCast` by adding synthetic input injection. Applications start a `RemoteDesktop` session alongside a `ScreenCast` session (passing the `remote_desktop_session` option to `ScreenCast.CreateSession`). After explicit user permission, the `RemoteDesktop` portal exposes methods:
- `NotifyPointerMotion(session, options, dx, dy)` — relative pointer motion
- `NotifyPointerButton(session, options, button, state)` — pointer button press/release
- `NotifyKeyboardKeycode(session, options, keycode, state)` — keyboard key press/release
- `NotifyKeyboardKeysym(session, options, keysym, state)` — keysym-based input

These map to Wayland virtual input protocols (`zwlr_virtual_pointer_v1`, `zwp_virtual_keyboard_v1`) in the backend implementation.

### Screen Sharing in Browsers

Chromium and Firefox implement their own PipeWire/portal ScreenCast integration rather than using the `libportal` convenience library. Chromium's implementation lives in `chrome/browser/media/webrtc/desktop_media_picker_views.cc` and `media/capture/video/linux/` (the PipeWire capturer). Firefox's implementation is in `widget/gtk/nsWaylandDisplay.cpp` and related files. Both browsers call the portal D-Bus methods directly, negotiate PipeWire stream parameters themselves, and import DMA-BUF frames into their GPU pipelines. This means browser screen sharing behaviour can differ from native application behaviour even when the underlying portal version is the same — the browser's PipeWire consumer implementation determines whether DMA-BUF zero-copy is available.

---

## 9. Portal: GPU Surface Sharing with Sandboxed Apps

### The Access Problem

A sandboxed Flatpak application, by default, has no access to `/dev/dri`. It cannot open a DRM render node, initialise EGL, or call Vulkan. The sandbox uses Linux user namespaces, seccomp filters, and bind mounts to enforce this restriction. The Wayland socket is accessible (via `--socket=wayland`), allowing the app to submit pre-rendered buffers as `wl_shm` shared memory — but software rendering via `llvmpipe` or `softpipe` is often too slow for GPU-intensive applications.

### `--device=dri`: The Coarse-Grained Solution

The `--device=dri` Flatpak permission grants the sandboxed process read/write access to all `/dev/dri/renderD*` and `/dev/dri/card*` device nodes. This enables Mesa's full GPU acceleration path, DRI3 buffer sharing, and Vulkan. The vast majority of graphically-accelerated Flatpak applications — games, Blender, Kdenlive, Steam — use `--device=dri`.

A Flatpak manifest fragment for an application requiring GPU access:

```yaml
# com.example.GraphicsApp.yaml
finish-args:
  - --socket=wayland          # Wayland socket access
  - --socket=x11              # X11/XWayland socket access (for legacy GLX)
  - --device=dri              # /dev/dri/renderD* and card* access
  - --share=ipc               # Shared memory IPC (needed for XShm)
  # Environment overrides:
  - --env=MESA_LOADER_DRIVER_OVERRIDE=radeonsi  # Force specific Mesa driver
  - --env=DRI_PRIME=1         # Force discrete GPU on multi-GPU systems
```

The `--device=dri` permission grants access to all DRI devices simultaneously. There is currently no Flatpak mechanism to grant access to only a single render node, to restrict access to specific GPU compute queues, or to limit buffer sizes. On a multi-GPU system, `--device=dri` exposes all GPUs to the sandboxed application.

### `XDG_RUNTIME_DIR` and Wayland Socket Access

Flatpak provides the Wayland compositor socket to sandboxed applications by bind-mounting the Wayland socket from the host's `XDG_RUNTIME_DIR` into the sandbox's runtime directory. The application uses the standard `WAYLAND_DISPLAY` environment variable to locate the socket. This mechanism works regardless of `--device=dri` — submitting `wl_shm` buffers or pre-allocated DMA-BUFs through the Wayland protocol does not require GPU device access.

### `wp_security_context_v1`: Wayland-Level Identity

The `wp_security_context_v1` protocol (defined in wayland-protocols `staging/security-context/`) allows the sandbox infrastructure to associate metadata with a Wayland client connection before it is fully established. When Flatpak 1.16+ sets up a sandboxed application's Wayland socket, it uses `wp_security_context_v1` to propagate the app's sandbox ID (e.g., `com.example.App`) and the sandbox engine identifier (`flatpak`) to the compositor.

The compositor can use this information to:
- Restrict which Wayland globals are visible to the sandboxed client (e.g., not advertising `zwlr_output_manager_v1` or `wl_drm` to sandboxed apps)
- Limit DMA-BUF format/modifier combinations
- Deny access to privileged protocols such as `zwlr_screencopy_manager_v1`

Critically, `wp_security_context_v1` operates entirely at the Wayland protocol layer. It does not interact with the DRM device node access granted by `--device=dri`. A sandboxed app granted `--device=dri` still has direct kernel-level GPU access through the device node, regardless of any security context restrictions at the Wayland layer. The two mechanisms address orthogonal attack surfaces.

### EGL in Flatpak: Mesa Bundling

Mesa is bundled in the Flatpak runtime (`org.freedesktop.Platform`). Sandboxed applications link against the runtime's Mesa libraries, not the host system's Mesa. This ensures ABI compatibility between the application and the Mesa build, at the cost of Mesa updates in the runtime not immediately propagating to all applications. The `MESA_LOADER_DRIVER_OVERRIDE` environment variable can be used in the Flatpak manifest to force Mesa to load a specific Gallium driver (e.g., `radeonsi`, `iris`, `zink`, `llvmpipe`), bypassing the normal driver probe by device ID. This is useful for CI environments where GPU access may not be available and software rendering is acceptable.

### The Unsolved Problem: Portal-Mediated GPU Context Creation

The long-term goal is to allow sandboxed applications to request GPU context creation via `xdg-desktop-portal` rather than opening `/dev/dri` directly. Under this model, the portal (running outside the sandbox, with full GPU access) would create an EGL or Vulkan context on the application's behalf and pass back a restricted file descriptor representing the context. The concept is analogous to how KMS display access is mediated: applications never open `/dev/dri/card0` directly for modesetting; they go through the compositor, which mediates display access.

As of 2025, no complete GPU portal implementation exists in `xdg-desktop-portal`. The proposal has been discussed at XDC (most recently at XDC 2024) and in freedesktop mailing lists [Source](https://gitlab.freedesktop.org/xdg/xdg-desktop-portal/-/issues/). The `--device=dri` permission remains the only practical mechanism for hardware-accelerated rendering in Flatpak. This means Flatpak's GPU sandboxing is meaningfully weaker than its filesystem sandboxing — granting `--device=dri` is equivalent to giving the application an unmediated channel to the GPU hardware.

---

## 10. Running X11 Applications on Wayland: Practical Guide

### Toolkit Backend Selection

Modern GTK3/GTK4 and Qt5/Qt6 applications auto-detect the Wayland backend at runtime. GTK checks `$GDK_BACKEND` and falls back to Wayland if the Wayland socket is available and `$DISPLAY` is absent. Qt checks `$QT_QPA_PLATFORM`. To force specific behaviour:

```bash
# Force Wayland backend for GTK application
GDK_BACKEND=wayland myapp

# Force X11/XWayland for GTK application
GDK_BACKEND=x11 myapp

# Force Wayland for Qt application
QT_QPA_PLATFORM=wayland myapp

# Force X11/XWayland for Qt application
QT_QPA_PLATFORM=xcb myapp
```

Applications that use X11-only features (XInput2 raw input, `XGrabKey` for global hotkeys, `XRecord` for input monitoring, or `XTest` for synthetic input) must run through XWayland — these extensions have no Wayland equivalents accessible to untrusted clients.

### Common XWayland Failure Modes

**Wrong DPI**: Fonts appear tiny or enormous. Fix: set `Xft.dpi: 192` (or appropriate value) in `~/.Xresources` and run `xrdb -merge ~/.Xresources`. Check with `xdpyinfo | grep resolution`.

**Missing global hotkeys**: `XGrabKey` does not work across Wayland application boundaries because the compositor controls keyboard focus. Fix: use compositor-specific hotkey configuration (`swaymsg bindsym` for Sway, KWin shortcuts for KDE, GNOME keyboard shortcuts). Applications that use `XGrabKey` must either be ported to use Wayland-native input inhibitor protocols (`zwp_keyboard_shortcuts_inhibit_manager_v1`) or accept that global hotkeys will not work.

**Clipboard truncation or failure**: Large INCR-protocol clipboard transfers occasionally fail across the XWayland bridge. Use a clipboard manager that explicitly handles the X11/Wayland bridge (e.g., `wl-clipboard` with `wl-paste` piped to `xclip`).

**Drag-and-drop failure across X11/Wayland boundary**: Applications on opposite sides of the XWayland boundary may fail to exchange drag-and-drop payloads. There is no complete fix; the most reliable approach is to ensure both apps are on the same side (either both native Wayland or both X11).

**`xdotool` position queries returning wrong values**: X11 window position queries reflect XWayland's virtual coordinate system, not the Wayland compositor's logical coordinate space. Applications that depend on absolute screen position for window management (screenshot tools, position-based automation) need Wayland-native equivalents (`wlr-randr`, `slurp`, `grim`).

### Detecting XWayland from Within an Application

An application can detect whether it is running as an X11 client under XWayland by checking for the `XWAYLAND` X11 extension:

```c
#include <xcb/xcb.h>
#include <xcb/xwayland.h>  /* if available */

/* Check for Wayland-side XWayland marker */
bool is_xwayland(xcb_connection_t *conn) {
    /* Check for the XWAYLAND extension */
    xcb_query_extension_cookie_t c =
        xcb_query_extension(conn, strlen("XWAYLAND"), "XWAYLAND");
    xcb_query_extension_reply_t *r = xcb_query_extension_reply(conn, c, NULL);
    bool result = r && r->present;
    free(r);
    /* Also check for WAYLAND_DISPLAY in environment */
    return result || (getenv("WAYLAND_DISPLAY") != NULL &&
                      getenv("DISPLAY") != NULL);
}
```

### Debugging Tools

- **`xlsclients`**: Lists X11 clients connected to XWayland. Confirms which applications are using XWayland vs. native Wayland.
- **`xprop -root`**: Dumps root window properties, including XWayland-specific atoms.
- **`xev`**: Monitors X11 input events. Useful for diagnosing keyboard/pointer event delivery under XWayland.
- **`xrandr`**: Shows XWayland's virtual display configuration. DPI and resolution visible here affect X11 applications.
- **`XWAYLAND_DEBUG=1`**: Environment variable that enables verbose XWayland debugging output. Logs surface creation, focus changes, and protocol interactions.
- **`XWAYLAND_NO_GLAMOR=1`**: Disables Glamor acceleration. Useful for isolating rendering bugs from Glamor vs. application logic.

---

## 11. Flatpak GPU Access: Security Model and Current Limitations

### `--device=dri`: Security Implications

Granting a sandboxed application the `--device=dri` permission establishes a covert channel between the sandboxed process and the GPU hardware that bypasses the filesystem and process isolation provided by the rest of the Flatpak sandbox. The security implications are multifold:

**GPU compute side-channels**: A sandboxed application with GPU access can execute arbitrary compute shaders. Timing measurements on shared GPU caches (the GPU's last-level cache, texture cache, L1/L2 caches) can leak information about the memory access patterns of other GPU workloads on the same system. Published research on GPU side-channel attacks (e.g., "Rendered Insecure" by Naghibijouybari et al.) demonstrates that co-resident GPU processes can infer memory access patterns, cryptographic key material, and screen contents from GPU cache timing. Desktop systems rarely have multiple mutually-distrusting processes sharing the GPU simultaneously (unlike cloud GPU virtualization), so this risk is primarily relevant in multi-user or highly security-sensitive contexts.

**VRAM residual data**: When a GPU buffer is freed and reallocated, the previous buffer contents may still be present in VRAM (the GPU driver's buffer management may not zero memory on deallocation by default). A sandboxed application could allocate large GPU buffers and scan their contents for residual data from previous GPU operations — rendered frames, texture data, or compute results from other applications. Modern Mesa drivers perform zeroing of newly allocated buffers, and the kernel's GPU VM isolation (each DRM client has its own GPUVM address space, meaning processes cannot read each other's GPU VA mappings through normal GPU operations) limits the scope of such attacks. However, VRAM zeroing policies vary by driver and are not universally enforced.

**DRM ioctl surface**: Opening a render node grants access to a wide range of DRM ioctls beyond simple rendering: `DRM_IOCTL_MODE_CREATEPROPBLOB`, `DRM_IOCTL_PRIME_FD_TO_HANDLE`, and other operations that manipulate kernel-level objects. A malicious sandboxed process could use these ioctls to escalate privileges or interfere with other DRM clients if kernel bugs exist in the ioctl handlers. The kernel's GPU VM isolation reduces but does not eliminate this surface.

### GPU VM Isolation as a Partial Mitigation

Linux kernel GPU VM isolation (introduced with per-process GPUVM support in the DRM subsystem, available in the Intel and AMD drivers via the Compute VM work) assigns each DRM file descriptor its own GPU virtual memory address space. Process A cannot read from Process B's GPUVM mappings through normal GPU rendering or compute operations. This is a necessary and meaningful mitigation — it prevents the most straightforward VRAM snooping attacks — but it does not prevent timing side-channels or attacks that exploit shared hardware resources (caches, memory controllers, compute units) rather than direct GPUVM access.

### `wp_security_context_v1`: Wayland-Layer Identity and Restrictions

The `wp_security_context_v1` protocol fills a different need: it enables the compositor to know, at the Wayland protocol level, that a connecting client is a Flatpak sandbox with a specific app-ID. Flatpak 1.16+ sets up the security context when the sandboxed application's Wayland connection is established:

```c
/* Flatpak sandbox setup: establishing a connection with security context */
/* (conceptual — actual implementation in Flatpak's wayland-socket.c) */
struct wp_security_context_manager_v1 *sec_ctx_manager = /* bound from registry */;
struct wp_security_context_v1 *ctx =
    wp_security_context_manager_v1_create_listener(sec_ctx_manager,
                                                    listen_fd,
                                                    close_fd);
wp_security_context_v1_set_sandbox_engine(ctx, "flatpak");
wp_security_context_v1_set_app_id(ctx, "com.example.App");
wp_security_context_v1_set_instance_id(ctx, instance_id);
wp_security_context_v1_commit(ctx);
wp_security_context_v1_destroy(ctx);
```

The compositor receives this metadata and can choose to restrict which Wayland globals are visible to the sandboxed client. For example, a compositor might not advertise `zwlr_screencopy_manager_v1` to sandboxed clients identified via `wp_security_context_v1`, preventing them from capturing the screen without going through the portal. As of 2024–2025, compositor enforcement of security-context-based restrictions is nascent: KWin and Mutter have partial implementation, but the security policy (which globals to restrict, for which sandbox engines) is not yet standardised.

Critically, `wp_security_context_v1` does not interact with the DRM layer at all. It governs what Wayland protocol messages the sandboxed client can exchange with the compositor; it has no effect on the device node access granted by `--device=dri`.

### The XWayland–Flatpak Interaction

When an X11 application runs inside a Flatpak sandbox, the layering is:

1. The compositor launches XWayland **outside** the sandbox (XWayland is a host system process).
2. Flatpak bind-mounts the X11 Unix socket into the sandbox via `--socket=x11`.
3. The sandboxed X11 application connects to XWayland's socket normally.
4. XWayland handles the X11 protocol and uses Glamor (GPU acceleration, outside the sandbox) for 2D rendering operations.
5. If the sandboxed application uses `glXMakeCurrent()` or `eglBindAPI(EGL_OPENGL_API)` with a GLX drawable, Mesa's DRI3 client code inside the sandbox attempts to open `/dev/dri/renderDN` directly. **This requires `--device=dri` in the sandbox.**
6. Without `--device=dri`, Mesa falls back to `XShm` (X11 shared memory, CPU rendering) or software rendering via `llvmpipe`.

The key insight: **basic 2D X11 applications do not require `--device=dri`** — their rendering goes through the XWayland socket to XWayland's Glamor, which has GPU access as a host process. **OpenGL/Vulkan X11 applications require `--device=dri`** for DRI3 access from within the sandbox.

### Practical Guidance for Flatpak Packagers

**Audit `--device=dri` usage**: A sandboxed application should request `--device=dri` only if it genuinely requires hardware-accelerated 3D rendering or Vulkan. Applications that use only 2D drawing (text editors, music players, basic utilities) do not need GPU access, even under XWayland.

```bash
# Audit installed Flatpaks for --device=dri
flatpak info --show-permissions $(flatpak list --app --columns=application) \
    2>/dev/null | grep -B5 "dri"

# Or iterate explicitly:
for app in $(flatpak list --app --columns=application); do
    perms=$(flatpak info --show-permissions "$app" 2>/dev/null)
    if echo "$perms" | grep -q "dri"; then
        echo "GPU access: $app"
    fi
done
```

**Document GPU access in manifests**: The Flatpak manifest format allows comments; use them to explain why `--device=dri` is required:

```yaml
finish-args:
  # Required for hardware-accelerated Vulkan rendering (scene3d engine).
  # Remove and test with LIBGL_ALWAYS_SOFTWARE=1 before adding this.
  - --device=dri
```

**Use software rendering for CI**: Add `--env=LIBGL_ALWAYS_SOFTWARE=1` or `--env=MESA_LOADER_DRIVER_OVERRIDE=llvmpipe` in CI Flatpak runs to avoid requiring GPU access in non-interactive build and test pipelines.

**Track the GPU portal**: When a stable GPU portal implementation lands in `xdg-desktop-portal`, migrate to it and remove `--device=dri`. Follow the upstream issue at [gitlab.freedesktop.org/xdg/xdg-desktop-portal/-/issues/](https://gitlab.freedesktop.org/xdg/xdg-desktop-portal/-/issues/).

**Snap vs. Flatpak**: The Snap package format uses a different permission model (`snapd interfaces`) with its own portal integration approach. Snap's `hardware-observe` and `opengl` interfaces control DRI device access, mediated by AppArmor profiles rather than filesystem namespace mounts. The portal integration via `xdg-desktop-portal` is broadly compatible across both Flatpak and Snap environments, as portal access requires only a D-Bus session connection.

---

## 12. Integrations

**Chapter 1 (DRM Architecture)**: DRI3 is the X11 protocol that allows X11 applications to allocate GPU buffers directly. XWayland uses DRI3 to accept client-allocated GPU buffers (from Mesa's GLX driver) and present them to the compositor. Render nodes (`/dev/dri/renderDN`) are the device nodes DRI3 clients open. The security model of render nodes — designed to be world-readable/writable for unprivileged GPU compute — is why Flatpak's `--device=dri` represents a significant sandbox escape for GPU capabilities.

**Chapter 2 (KMS)**: Gamescope's full-screen XWayland mode gives XWayland's backing surface direct KMS scanout path consideration; the compositor may promote XWayland's surface to the primary plane. Rootless XWayland surfaces participate in the compositor's normal KMS plane assignment via the Wayland buffer-to-KMS path.

**Chapter 3 (Advanced Display)**: The `wp_linux_drm_syncobj_v1` explicit synchronisation solution described in this chapter for XWayland is the same DRM syncobj timeline mechanism used by native Wayland clients. The protocol-level solution is unified; the complexity in XWayland's case is bridging the X11 Present extension's synchronisation model to the Wayland syncobj timeline.

**Chapter 4 (GPU Memory)**: Glamor allocates GEM-backed Pixmaps via Mesa EGL; DRI3 buffer passing from GLX clients uses PRIME handles. The GPU VM isolation mechanism discussed in Chapter 4 (each DRM client owning its own GPUVM address space) is the kernel-level mitigation limiting the blast radius of a compromised sandboxed GPU client.

**Chapter 19 (OpenGL Drivers)**: Glamor calls Mesa's GLES2/GL implementation (radeonsi, iris, zink) for all X11 rendering. GLX application rendering under XWayland goes through Mesa's `src/glx/` GLX driver layer, not the Wayland EGL path. The distinction between the DRI3/GLX path and the Wayland EGL path is important: applications running under XWayland that call `eglSwapBuffers()` via EGL with a GLX surface are using DRI3, not the `EGL_PLATFORM_WAYLAND_EXT` path described in Chapter 19.

**Chapter 20 (Wayland Fundamentals)**: XWayland is a Wayland client that uses `wl_surface`, `linux-dmabuf`, and `wp_linux_drm_syncobj_v1`. The `wp_security_context_v1` protocol covered in Chapter 20's protocol survey is the Wayland-layer mechanism for propagating Flatpak sandbox identity to the compositor.

**Chapter 21 (wlroots)**: wlroots' `wlr_xwayland` structure manages the XWayland process lifecycle — starting, restarting on crash, and exposing an interface for the compositor to move windows. The wlroots compositor assigns input focus to XWayland surfaces using the same seat focus mechanisms as native Wayland clients.

**Chapter 22 (Production Compositors)**: Gamescope's non-rootless XWayland is the standard deployment model for Proton games on Steam Deck. Mutter and KWin's XWayland explicit sync support determines whether NVIDIA users see tearing with X11 applications. The `xdg-desktop-portal-gnome` and `xdg-desktop-portal-kde` backends depend on compositor-private D-Bus APIs (Mutter ScreenCast, KWin's screencasting interface) for portal screen capture.

**Chapter 24 (Vulkan/EGL for Apps)**: Applications using EGL under XWayland go through the DRI3 path, not the Wayland EGL path. They allocate buffers via Mesa EGL and render into them, then use the X11 Present extension to deliver them to XWayland. This is why some EGL features (notably buffer age tracking and partial presentation) behave differently under XWayland vs. native Wayland.

**Chapter 26 (Hardware Video)**: The ScreenCast portal is how video conferencing software (WebRTC-based meeting clients, Zoom, OBS) captures the screen on Linux under Wayland. The PipeWire DMA-BUF path from the portal's capture to the video encoder is the zero-copy route described in Chapter 26; the portal effectively functions as the buffer producer in a PipeWire DMA-BUF pipeline.

---

## 13. References

1. **XWayland source code** — `xserver/hw/xwayland/`:
   https://gitlab.freedesktop.org/xorg/xserver

2. **Glamor source code** — `xserver/glamor/`:
   https://gitlab.freedesktop.org/xorg/xserver/-/tree/master/glamor

3. **xdg-desktop-portal**:
   https://github.com/flatpak/xdg-desktop-portal

4. **xdg-desktop-portal documentation**:
   https://flatpak.github.io/xdg-desktop-portal/docs/

5. **org.freedesktop.portal.ScreenCast D-Bus specification**:
   https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.ScreenCast.html

6. **xdg-desktop-portal-wlr**:
   https://github.com/emersion/xdg-desktop-portal-wlr

7. **xdg-desktop-portal-wlr PipeWire screencast implementation** (`src/screencast/pipewire_screencast.c`):
   https://github.com/emersion/xdg-desktop-portal-wlr/blob/master/src/screencast/pipewire_screencast.c

8. **xdg-desktop-portal-gnome**:
   https://gitlab.gnome.org/GNOME/xdg-desktop-portal-gnome

9. **linux-drm-syncobj-v1 Wayland protocol (wayland-protocols 1.34)**:
   https://wayland.app/protocols/linux-drm-syncobj-v1

10. **Explicit Sync Wayland Protocol Merged — wayland-protocols 1.34** (GamingOnLinux, March 2024):
    https://www.gamingonlinux.com/2024/03/explicit-sync-wayland-protocol-merged-wayland-protocols-1-34-released/

11. **KWin linux-drm-syncobj-v1 implementation MR**:
    https://invent.kde.org/plasma/kwin/-/merge_requests/4693

12. **Sway Explicit Sync Merged** (Phoronix, August 2024):
    https://www.phoronix.com/news/Sway-Explicit-Sync-Merged

13. **"Explicit Sync in Wayland" — Xaver Hugl blog post** (April 2024):
    https://zamundaaa.github.io/wayland/2024/04/05/explicit-sync.html

14. **"Explicit GPU Synchronization Merged For XWayland"** (Phoronix):
    https://www.phoronix.com/news/Explicit-GPU-Sync-XWayland-Go

15. **LWN: XWayland explicit synchronisation coverage** (2023):
    https://lwn.net/Articles/937565/

16. **wp_security_context_v1 protocol XML** (wayland-protocols staging):
    https://gitlab.freedesktop.org/wayland/wayland-protocols/-/tree/main/staging/security-context

17. **Flatpak 1.16 release notes** (Wayland security context, USB upgrades):
    https://www.colocrossing.com/blog/flatpak-1-16-official-release/

18. **Flatpak sandbox permissions reference** (`--device=dri` documentation):
    https://docs.flatpak.org/en/latest/sandbox-permissions.html

19. **XWayland xwayland-glamor.c source (GitHub mirror)**:
    https://github.com/mirror/xserver/blob/master/hw/xwayland/xwayland-glamor.c

20. **Glamor design document** (X.Org wiki):
    https://www.x.org/wiki/Development/Documentation/Glamor/

21. **XDG Desktop Portal — Arch Wiki**:
    https://wiki.archlinux.org/title/XDG_Desktop_Portal

22. **XWayland manual page** (Arch Linux):
    https://man.archlinux.org/man/extra/xorg-xwayland/Xwayland.1.en

23. **"Running X apps on Wayland"** (Arch Wiki):
    https://wiki.archlinux.org/title/wayland#XWayland

24. **DRM render node security design** (kernel documentation):
    https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html#render-nodes

25. **GPU portal proposal tracking issue** (xdg-desktop-portal GitLab):
    https://gitlab.freedesktop.org/xdg/xdg-desktop-portal/-/issues/

26. **PipeWire documentation** (screen cast consumer reference):
    https://docs.pipewire.org/

27. **"When Flatpak's Sandbox Cracks"** (Linux Journal — security analysis):
    https://www.linuxjournal.com/content/when-flatpaks-sandbox-cracks-real-life-security-issues-beyond-ideal

28. **xdg-desktop-portal-hyprland** (Hyprland wiki):
    https://wiki.hypr.land/Hypr-Ecosystem/xdg-desktop-portal-hyprland/

29. **XWayland Now Allows Better Choosing Between OpenGL and OpenGL ES** (Phoronix — `-glamor` flag):
    https://www.phoronix.com/news/XWayland-GLAMOR-GL-GLES

30. **"XWayland: the bridge between old and new"** (LWN, 2016):
    https://lwn.net/Articles/683126/

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
