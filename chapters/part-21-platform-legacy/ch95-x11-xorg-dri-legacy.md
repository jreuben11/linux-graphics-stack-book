# Chapter 95: X11/Xorg Architecture and the DRI Legacy Stack

> **Part**: Part XXI — Platform, Legacy, and History
> **Audiences**: Systems developers maintaining X11 applications; display-server engineers; anyone who wants to understand why Wayland was necessary and how XWayland bridges the two worlds
> **Status**: First draft — 2026-06-19

---

## Table of Contents

- [1. Introduction — Why X11 Matters in 2026](#1-introduction--why-x11-matters-in-2026)
  - [1.1 What is X11?](#11-what-is-x11)
  - [1.2 What is Xorg?](#12-what-is-xorg)
  - [1.3 What is DRI?](#13-what-is-dri)
  - [1.4 What is XWayland?](#14-what-is-xwayland)
- [2. X11 Protocol Fundamentals](#2-x11-protocol-fundamentals)
- [3. XServer Extension Model](#3-xserver-extension-model)
- [4. GLX: OpenGL Meets X11](#4-glx-opengl-meets-x11)
- [5. AIGLX — Accelerated Indirect GLX](#5-aiglx--accelerated-indirect-glx)
- [6. DRI1, DRI2, DRI3 Evolution](#6-dri1-dri2-dri3-evolution)
- [7. The Composite Extension](#7-the-composite-extension)
- [8. XRANDR and Multi-Monitor](#8-xrandr-and-multi-monitor)
- [9. XWayland: The Bridge](#9-xwayland-the-bridge)
- [10. Security: X11 vs Wayland](#10-security-x11-vs-wayland)
- [11. Debugging X11 Applications](#11-debugging-x11-applications)
- [12. Integrations](#12-integrations)

---

## 1. Introduction — Why X11 Matters in 2026

X11 is not dead. Enterprise Linux deployments — Red Hat Enterprise Linux, SUSE Linux Enterprise, and their derivatives — still ship X11-based sessions as a supported option, in part because critical business software (proprietary CAD tools, legacy Java Swing applications, decades-old Motif programs) depends on X11 semantics that have no exact Wayland equivalent. Chromebooks run Linux apps under an XWayland-backed Crostini container. Remote desktop protocols (VNC, NX, X2Go) still speak X over network sockets. Even on a modern GNOME 46 or KDE Plasma 6 system running a Wayland compositor, *every X11 application* — and there remain millions of them — runs inside XWayland, an X11 server that is itself a Wayland client. Understanding X11 is therefore not archaeological interest: it is prerequisite knowledge for understanding the compatibility layer on which the Linux desktop depends today.

Beyond pragmatics, X11's architectural choices — and their failure modes — are the direct explanation for Wayland's design. Every Wayland protocol that seems over-engineered (the strict per-surface isolation, the absence of global input capture, the mandatory compositor-mediated clipboard) is a considered response to a concrete X11 problem. You cannot reason about Wayland's design philosophy without understanding what X11 got wrong and why those errors were structurally inevitable given X11's client-server model.

**X11 lineage.** The X Window System originated at MIT in the early 1980s. X10, released in 1984, established the client-server model: a single display server process mediating access to screen, keyboard, and pointer on behalf of multiple client programs, potentially running on remote machines. X11, the protocol revision that has not changed its major version number since, was specified in 1987; RFC 1013 ["X Window System Protocol, version 11: Alpha update April 1987"](https://www.rfc-editor.org/rfc/rfc1013.html) records the initial specification. The 1987 design choices — a single privileged server, a shared-namespace event model, X atoms as a global interning table — were sensible for a network-transparent window system serving university workstations. They became architectural liabilities when the display server moved from a Sun-3 on a departmental LAN to the GPU-accelerated composited desktop of the 2010s and beyond.

### 1.1 What is X11?

X11 (the X Window System, protocol version 11) is a client-server protocol for building graphical user interfaces on Unix-like operating systems. The server — called the display server — owns exclusive access to the physical screen, keyboard, and pointer. Clients connect to the server over a transport (Unix-domain socket locally, TCP remotely) and issue requests to create windows, draw graphics, and receive input events. The server is authoritative: clients never write directly to the framebuffer or read raw input; everything passes through the server.

X11's defining characteristic is network transparency: a client running on a remote host can render into windows on a local display, because the protocol operates purely over a byte stream. This architecture was specified in 1987 and has not changed its major version number since; it is governed by the X.Org Foundation and documented at [x.org](https://www.x.org/releases/X11R7.6/doc/xproto/x11protocol.html). Network transparency came at a cost — the single shared server process became a bottleneck for GPU-accelerated rendering, motivating the DRI architecture described in Section 6. The global, permissive event model (any client may observe events on any visible window) also created fundamental security problems addressed in Section 10. Understanding both strengths and failure modes of X11 is necessary groundwork for understanding why Wayland replaced it.

### 1.2 What is Xorg?

Xorg is the dominant open-source implementation of the X11 display server. It is the reference server that ships with virtually every Linux distribution, maintained at [gitlab.freedesktop.org/xorg/xserver](https://gitlab.freedesktop.org/xorg/xserver). Xorg runs as a privileged process (historically as root, later via `systemd-logind` device takeover) and loads a set of kernel and userspace driver components: a kernel DRM driver for modesetting and buffer management, a DDX (Device-Dependent X) driver for display output configuration, and extension modules for OpenGL (GLX), compositing (Composite), and display configuration (RANDR). The modesetting DDX driver, which relies entirely on the kernel DRM/KMS interfaces, is now the standard path; older vendor-specific DDX drivers (the `xf86-video-*` family) are largely deprecated.

Xorg's extension architecture means that the core server binary changes rarely — capabilities such as OpenGL support, multi-touch input, and display reconfiguration were added without protocol version bumps. The drawback is that extensions loaded into the server process share its address space and privilege level, meaning a misbehaving extension can crash or compromise the entire display. This chapter examines the most significant extensions — GLX, Composite, RANDR, and the DRI family — in Sections 3 through 8.

### 1.3 What is DRI?

DRI (Direct Rendering Infrastructure) is the protocol and kernel subsystem that allows Mesa (the OpenGL and Vulkan implementation) to submit GPU commands directly to the kernel DRM driver, bypassing the X server process entirely for rendering work. Without DRI, all GPU rendering had to pass through the X server, which serialised it like any other X request and imposed a severe latency penalty. With DRI, a client obtains an authenticated channel to the GPU kernel driver, allocates GPU buffers, and issues rendering commands without X server involvement; only the final buffer-swap is coordinated with the display server.

DRI evolved through three generations — DRI1 (1998), DRI2 (2008), and DRI3 (2013) — each addressing critical deficiencies of the previous design around buffer sharing, multi-GPU support, and security. DRI is implemented jointly in the kernel (`drivers/gpu/drm/` in the Linux source tree), in Mesa (`src/loader/`, `src/glx/`), and in the X server extension (`hw/xfree86/dri3/`). Section 6 traces the full evolution. The same kernel DRM subsystem that powered DRI is the foundation on which Wayland compositors operate natively, making DRI history directly relevant to understanding the modern Linux graphics stack.

### 1.4 What is XWayland?

XWayland is an X11 display server that runs as a Wayland client. On a system running a Wayland compositor (GNOME's Mutter, KDE's KWin, sway, wlroots-based compositors), XWayland starts on demand to handle X11 applications. Those applications connect to XWayland exactly as they would to a conventional Xorg server; XWayland translates their X11 protocol operations into Wayland protocol on the compositor side. From the Wayland compositor's perspective, XWayland is simply another Wayland client that happens to produce surfaces on behalf of many X11 programs.

XWayland is maintained in the same repository as Xorg ([gitlab.freedesktop.org/xorg/xserver](https://gitlab.freedesktop.org/xorg/xserver), `hw/xwayland/`) and shares the core Xorg DIX (Device-Independent X) layer. Its significance is that it means no Linux desktop ships without X11 support: even a compositor that never started Xorg still runs X11 applications through XWayland. The translation layer is not lossless — certain X11 semantics (global hotkeys via `XGrabKey`, precise cursor positioning across windows, clipboard synchronisation timing) require explicit workarounds — but for the vast majority of X11 applications it is transparent. Section 9 covers the XWayland architecture and its limitations in detail.

---

## 2. X11 Protocol Fundamentals

### The Client-Server Model

An X11 display server is a single process (typically `Xorg`) that owns exclusive access to the display hardware and all input devices. Clients connect to it over a transport. On a local machine the transport is a Unix-domain socket at `/tmp/.X11-unix/X0` (for display `:0`) or a TCP socket on port `6000 + display_number`. The `DISPLAY` environment variable tells a client which display to connect to: `DISPLAY=:0` means display 0 on the local machine, `DISPLAY=host:0` means TCP to port 6000 on `host`, and `DISPLAY=:0.1` means screen 1 of display 0.

The X11 wire protocol carries four message types:

- **Requests** — sent by the client to the server; each begins with a 1-byte major opcode, a 1-byte sub-opcode or length modifier, and a 2-byte total length in 32-bit words. Major opcodes 1–127 are core protocol; 128–255 are reserved for extensions.
- **Replies** — server responses to requests that generate them (not all requests generate replies). Each reply carries a 32-bit sequence number and a 4-byte length followed by the reply body.
- **Events** — asynchronous notifications from the server (key press, pointer motion, window exposure, etc.); each is 32 bytes, numbered 2–34 in the core protocol, with extensions adding higher event codes.
- **Errors** — reported asynchronously or synchronously depending on request type; 32 bytes with an error code, major opcode, and sequence number.

The X protocol uses native-byte-order encoding negotiated at connection setup. The connection handshake begins with the client sending a byte indicating byte order (`0x42` for big-endian, `0x6C` for little-endian), followed by protocol version numbers (major 11, minor 0), auth-protocol name and data. The server responds with success or failure and an information block containing the resource base and mask, the number of screens, and per-screen depth/visual information. The X11 protocol specification is maintained at [x.org](https://www.x.org/releases/X11R7.6/doc/xproto/x11protocol.html).

### Window Hierarchy

X11 windows form a strict tree. The root window is the full-screen rectangle belonging to each screen; everything else is a descendant. A top-level window (what a user thinks of as a "window") is a direct child of the root. Dialog boxes and tooltips are children of top-level windows. The hierarchy is significant: input events are delivered to the innermost window under the pointer, then propagated up the tree unless a client selects events on an ancestor with `XGrabButton` or similar, and *any* client can select events on any window it can find — a key source of X11's security problems (Section 10).

### Atoms, Properties, and Selections

X atoms are 32-bit integers that serve as interned strings: `XInternAtom(dpy, "_NET_WM_NAME", False)` returns a server-unique integer for that string. Properties are typed key-value pairs attached to windows: `XGetWindowProperty` fetches the value of an atom on a window. Atoms and properties underpin every high-level X11 mechanism: window manager hints (the `WM_CLASS`, `WM_PROTOCOLS`, `_NET_WM_STATE` properties), clipboard management (`CLIPBOARD`, `PRIMARY`, `SECONDARY` selections), drag-and-drop (XDND protocol), and systray embedding.

The X selection mechanism is the X11 clipboard. Three selections co-exist: `PRIMARY` (the middle-click-to-paste X tradition, set on any text highlight), `CLIPBOARD` (the explicit Ctrl+C clipboard), and `SECONDARY` (rarely used). A client owns a selection by calling `XSetSelectionOwner`; when another client wants the content it sends a `SelectionRequest` event, the owner responds by converting the data into the requested format (atom `UTF8_STRING`, `STRING`, `image/png`, etc.) and placing it in a specified property on the requesting window, then sends a `SelectionNotify` event. This protocol is sound but requires the owning client to remain alive to serve requests — clipboard managers (like `clipit` or `xclipboard`) work around this by eagerly fetching and caching selection content.

ICCCM (Inter-Client Communication Conventions Manual) standardises window manager interactions: `WM_DELETE_WINDOW` protocol, `WM_NORMAL_HINTS` for size constraints, `WM_ICON_NAME`. EWMH (Extended Window Manager Hints, `_NET_*` atoms) extends ICCCM with taskbar visibility, virtual desktops, struts for panels, and the `_NET_WM_PID` property.

**Inspection tools:**

```bash
# Show server information including extensions
xdpyinfo -display :0

# List all atoms known to the server
xlsatoms -display :0

# Inspect window properties
xprop -root
xprop -id 0x1234567

# Walk the window tree
xwininfo -root -tree
```

---

## 3. XServer Extension Model

The X11 core protocol (opcodes 1–127, events 2–34, errors 0–17) is intentionally minimal. All significant functionality — OpenGL rendering, compositing, display configuration, input device management, screen rendering — lives in named extensions loaded by the X server at startup. The extension mechanism is the reason X11 has remained relevant for nearly four decades: hardware capabilities can be exposed without breaking existing clients.

### Extension Loading and Negotiation

The Xorg server loads extensions from shared libraries in `/usr/lib/xorg/modules/extensions/` (path varies by distribution). The core extension infrastructure is in `dix/extension.c` in the Xorg source. An extension registers itself by calling `AddExtension(name, num_events, num_errors, mainProc, swappedMainProc, closeDownProc, setupProc)`. The server assigns the extension a base opcode (128–255), a base event code, and a base error code.

A client negotiates an extension via `XQueryExtension(dpy, "GLX", &major_opcode, &first_event, &first_error)`. If the extension is present, the client uses the returned `major_opcode` for all subsequent requests to that extension, with a sub-opcode in the second byte distinguishing individual requests.

### Key Extensions

| Extension | Purpose | Introduced |
|-----------|---------|-----------|
| **GLX** | OpenGL binding to X11 windows | X11R6 (1994) |
| **Composite** | Off-screen backing store for windows | X.Org 6.8.0 (2004) |
| **Damage** | Efficient dirty-region tracking | X.Org 6.8.0 (2004) |
| **Fixes** | Miscellaneous fixes (cursor locking, region ops) | X.Org 6.8.0 (2004) |
| **RENDER** | Server-side anti-aliased rendering, alpha compositing | X11R6.6 (2001) |
| **RANDR** | Dynamic display reconfiguration | XFree86 4.3 / RANDR 1.0 (2001) |
| **XInput2 (XI2)** | Multi-touch, device hierarchy events | X.Org 1.7 (October 2009) |
| **Present** | Synchronised frame delivery with VBLANK | X.Org 1.15 (2013) |
| **DRI3** | Buffer passing by file descriptor | X.Org 1.15 (2013) |

The `glxinfo` and `xdpyinfo` tools report which extensions are loaded:

```bash
glxinfo -B              # GLX version, renderer, direct rendering status
xdpyinfo | grep -i ext  # full extension list
```

---

## 4. GLX: OpenGL Meets X11

GLX is the X11 extension that binds OpenGL rendering to X11 windows and pixmaps. Without GLX, OpenGL contexts have no connection to the X display hierarchy; with GLX, an application can create an OpenGL context tied to a specific X screen, make it current for a specific X drawable (window or pixmap), and blit the finished frame into the window via `glXSwapBuffers`. Mesa implements GLX in `src/glx/` in the Mesa repository [Mesa GLX source](https://github.com/Mesa3D/mesa/tree/main/src/glx); the client-side code in `libGL.so.1` (or the GLVND-split `libGLX.so.0`) speaks the GLX wire protocol to the server-side GLX extension, which in turn calls into the DRI driver.

### Visuals and FBConfigs

The original GLX 1.0/1.1 API used **XVisualInfo** to describe the pixel format: colour depth, RGB channel sizes, alpha, stencil, depth buffer presence. An application called:

```c
/* GLX 1.0/1.1 visual selection — src/glx/glxcmds.c */
int attribs[] = {
    GLX_RGBA,
    GLX_DOUBLEBUFFER,
    GLX_DEPTH_SIZE, 24,
    None
};
XVisualInfo *vi = glXChooseVisual(dpy, screen, attribs);
```

GLX 1.3 (1998) introduced **GLXFBConfig**, a richer format descriptor that is drawable-type-aware. An FBConfig can describe rendering to a window, a pixmap, or an off-screen pbuffer independently, and adds attributes like `GLX_SAMPLE_BUFFERS` for multisampling:

```c
/* GLX 1.3 FBConfig selection */
int fb_attribs[] = {
    GLX_RENDER_TYPE,    GLX_RGBA_BIT,
    GLX_DRAWABLE_TYPE,  GLX_WINDOW_BIT,
    GLX_DOUBLEBUFFER,   True,
    GLX_DEPTH_SIZE,     24,
    GLX_STENCIL_SIZE,   8,
    None
};
int n_configs;
GLXFBConfig *configs = glXChooseFBConfig(dpy, screen, fb_attribs, &n_configs);
```

GLX 1.3 also introduced distinct drawable types:
- `GLXWindow` — an OpenGL-renderable wrapper around an `XWindow`
- `GLXPixmap` — an OpenGL-renderable wrapper around an `XPixmap`
- `GLXPbuffer` — an off-screen rendering surface without an underlying X drawable

### Context Creation and Rendering

```c
/* Classic GLX context creation */
GLXContext ctx = glXCreateContext(dpy, vi, NULL /* share list */, True /* direct */);
glXMakeCurrent(dpy, win, ctx);
/* ... render with OpenGL calls ... */
glXSwapBuffers(dpy, win);     /* swap front/back buffers */
glXDestroyContext(dpy, ctx);
```

The `direct` flag in `glXCreateContext` requests that the driver bypass the X server for rendering and write directly to GPU memory — indirect rendering routes all GL calls through the X server's GLX request handler, which serialises them like any other X request. On modern systems, indirect rendering is deprecated and `LIBGL_ALWAYS_INDIRECT=1` is needed to force it.

**GLX_ARB_create_context** (2009) added the ability to request a specific OpenGL version and profile — essential for OpenGL 3.2+ core contexts that remove deprecated fixed-function state:

```c
int ctx_attribs[] = {
    GLX_CONTEXT_MAJOR_VERSION_ARB, 4,
    GLX_CONTEXT_MINOR_VERSION_ARB, 6,
    GLX_CONTEXT_PROFILE_MASK_ARB, GLX_CONTEXT_CORE_PROFILE_BIT_ARB,
    None
};
GLXContext ctx = glXCreateContextAttribsARB(dpy, fbc, NULL, True, ctx_attribs);
```

**GLX_EXT_swap_control** provides vsync control: `glXSwapIntervalEXT(dpy, drawable, 1)` enables sync-to-vblank; `0` allows tearing. The Mesa implementation lives in `src/glx/glx_pbuffer.c` and delegates to the DRI2/DRI3 swap interval path.

### The GLX Dispatch ABI

Mesa's GLX client code maintains a dispatch table — an array of function pointers that the linker resolves at startup. When GLVND (OpenGL Vendor-Neutral Dispatch) is in use (which is the default since Mesa 17.3), `libGLX.so.0` dispatches to the correct ICD (Installable Client Driver) based on which X screen the context is on. The Mesa GLVND ICD is registered via `/usr/share/glvnd/egl_vendor.d/` and `/usr/share/glvnd/opengl_vendor.d/` JSON files. [Source: Mesa GLVND documentation](https://docs.mesa3d.org/systems.html).

---

## 5. AIGLX — Accelerated Indirect GLX

In 2004, the X.Org Composite extension made compositing window managers technically possible: each window could be redirected to its own off-screen pixmap and a compositor could blend them into the final display. But the compositing manager itself needed OpenGL to perform GPU-accelerated blending (to do drop shadows, wobble effects, transparency). The problem was that the compositing manager ran *as an X client*, not as the X server, and the only way for an X client to use OpenGL on X11 was through GLX — which, without DRI, meant painfully slow indirect rendering.

**AIGLX** (Accelerated Indirect GLX) was a joint project by Red Hat and the X.Org Foundation, shipped in Xorg 7.1 and Fedora Core 5 (2006) [Fedora AIGLX wiki](https://fedoraproject.org/wiki/RenderingProject/aiglx). It loaded DRI hardware drivers into the X server process itself and enabled those drivers to service *indirect* GLX rendering requests with actual hardware acceleration — hence "accelerated indirect." AIGLX required that the hardware's DRI driver was loaded and functional, and that the GL extension `GLX_EXT_texture_from_pixmap` was supported.

### Texture From Pixmap

`GLX_EXT_texture_from_pixmap` ([Khronos registry](https://registry.khronos.org/OpenGL/extensions/EXT/GLX_EXT_texture_from_pixmap.txt)) allows an OpenGL application to bind an X Pixmap directly as a texture, without a CPU copy. The compositing manager (Compiz, Mutter-X11, KWin-X11) uses this to:

1. Call `XCompositeNameWindowPixmap` to obtain a pixmap handle pointing to a window's off-screen backing store.
2. Call `glXBindTexImageEXT(dpy, glx_pixmap, GLX_FRONT_LEFT_EXT, NULL)` to bind that pixmap as an OpenGL texture.
3. Render a textured quad covering the window's position in the compositor's framebuffer.
4. Call `glXReleaseTexImageEXT` when done.

This is the flow Compiz (released February 2006 by Novell) used to implement its famous cube desktop, wobbly windows, and window-switch effects. NVIDIA added `GLX_EXT_texture_from_pixmap` support in driver release 1.0-9625.

### AIGLX vs XGL

Concurrently, Novell/SUSE shipped **XGL** — a completely separate X server implemented as an OpenGL client, meaning the entire X protocol was composited by an OpenGL layer. XGL worked immediately with binary proprietary drivers but required replacing the entire X server. AIGLX was the winning approach because it required minimal server changes and worked within the existing driver model. By Xorg 7.2 (2007), AIGLX was standard and XGL was abandoned.

---

## 6. DRI1, DRI2, DRI3 Evolution

The Direct Rendering Infrastructure (DRI) is the protocol stack that lets Mesa bypass the X server and submit GPU commands directly to the kernel driver. Each generation fixed critical problems of the previous one.

### DRI1 (1998) — Shared Kernel Buffer and the DRM Lock

DRI1 was designed at Precision Insight in 1998. The architecture: the DRM kernel module allocated a shared memory region visible to both the X server and all DRI clients (via `drmMap`). Each client that wanted to render obtained the DRM authentication token from the X server, then called `drmGetMagic` / `drmAuthMagic` to prove it was authorised. To prevent races, the protocol required each client to acquire the **DRM lock** before issuing GPU commands — a single kernel mutex serialising all GPU access across the X server and every DRI client. The X server also held the lock while doing 2D drawing.

Problems with DRI1:
- The DRM lock was a contention bottleneck; if a client held the lock during a long rendering job, all other GPU users stalled.
- The X server had to participate in every context switch.
- Compositing managers were impossible: DRI1 gave the application direct scanout access with no mechanism for a compositor to intercept rendered content.

### DRI2 (2008) — Per-Drawable Buffer Management

DRI2, designed by Jesse Barnes and Keith Packard, appeared in Mesa 7.3 and Xorg 1.6 (2008-2009). It eliminated the DRM lock and introduced per-drawable buffer management. The key protocol requests were:

- `DRI2GetBuffers(drawable, buffer_types)` — returns the names of DRM GEM buffers (front, back, depth, stencil) allocated for a drawable. Names are global GEM handles, not file descriptors.
- `DRI2SwapBuffers(drawable, target_msc, divisor, remainder)` — instructs the server to swap the back buffer to front, optionally at a specific vblank count.
- `DRI2WaitMSC` / `DRI2WaitSBC` — wait for a media stream counter or swap buffer count.

DRI2 solved the lock problem: applications could render into their back buffer without coordination. But it introduced a new race: the server and client shared GEM buffer *by name* (a global integer), meaning the server could theoretically read a buffer while the client was still writing to it. More critically, under a compositing manager, there was a **tearing window**: the compositor's `glXBindTexImageEXT` call could race with the application's `DRI2SwapBuffers`. Compiz flicker and rendering corruption under compositing managers were persistent DRI2 complaints [CompositeSwap analysis, freedesktop.org](https://dri.freedesktop.org/wiki/CompositeSwap/). DRI2 also required the X server to allocate GEM buffers on the client's behalf — a design that broke for PRIME multi-GPU configurations.

### DRI3 (2013) — Buffer Passing by File Descriptor

DRI3, designed by Keith Packard, shipped in Xorg 1.15 and Mesa 10.1 (late 2013). The protocol specification lives at [cgit.freedesktop.org/xorg/proto/dri3proto](https://cgit.freedesktop.org/xorg/proto/dri3proto/tree/dri3proto.txt). The fundamental change: instead of passing GEM buffer names (global integers), DRI3 passes **file descriptors** via Unix `SCM_RIGHTS` over the X socket. A file descriptor is a capability object — the recipient can use it without knowing the global GEM handle, and the kernel enforces access rights.

DRI3 core requests:

| Request | Opcode | Description |
|---------|--------|-------------|
| `DRI3QueryVersion` | 0 | Negotiate protocol version |
| `DRI3Open` | 1 | Client opens a DRM render node; server returns an fd |
| `DRI3PixmapFromBuffer` | 2 | Import a GPU buffer (fd) as an X Pixmap |
| `DRI3BufferFromPixmap` | 3 | Export an X Pixmap as a GPU buffer fd |
| `DRI3FenceFromFD` | 4 | Import a sync fence fd as an X SyncFence |
| `DRI3FDFromFence` | 5 | Export an X SyncFence as a sync fence fd |

DRI3 v1.2 (2017) added multi-plane buffer support for YUV video formats: `DRI3PixmapFromBuffers` (plural) accepting an array of fds and strides for each plane.

The DRI3 flow for a typical application:
1. Mesa calls `DRI3Open` → receives the DRM render node fd (`/dev/dri/renderD128`).
2. Mesa allocates a GBM buffer object (BO) on that render node.
3. Mesa exports the BO as a DMA-BUF fd via `gbm_bo_get_fd`.
4. Mesa calls `DRI3PixmapFromBuffer` with that fd → receives an `XPixmap` handle.
5. Mesa renders into the BO.
6. Mesa hands the Pixmap to the Present extension for synchronised display.

There is no shared state, no global name, and no possibility of the server accessing the buffer while the client is still writing — proper producer/consumer semantics.

### The Present Extension (2013)

The **Present** extension, also by Keith Packard and shipped alongside DRI3, solved the *display timing* half of the DRI3 story. `DRI2SwapBuffers` had no reliable way to target a specific vblank; Present provides:

- `xcb_present_pixmap(conn, window, pixmap, serial, valid, update, x_off, y_off, crtc, wait_fence, idle_fence, options, target_msc, divisor, remainder, notifies, n_notifies)` — schedule `pixmap` to appear on screen at vblank `target_msc`. (The Present extension has no Xlib binding; applications use the XCB API via `libxcb-present`.)
- `PresentCompleteNotify` event — delivered when the pixmap actually appeared, with the actual MSC (Media Stream Counter) and UST (UNIX System Time) timestamp.
- `PresentIdleNotify` event — delivered when the compositor is done with the pixmap and the client may reuse it.

The **MSC** is a monotonically increasing counter incremented once per vblank, per CRTC. By specifying `target_msc`, `divisor`, and `remainder`, an application can schedule presentation at precise vblank boundaries (e.g., "present at the next even vblank for 30 FPS from a 60 Hz source"). This is the mechanism XWayland uses to feed frames to the Wayland compositor with accurate timing (Section 9).

---

## 7. The Composite Extension

The X Composite extension (version 0.4, ratified with X.Org 6.8.0 in 2004) enables compositing window managers by providing off-screen backing stores for windows [Composite protocol spec](https://www.x.org/releases/X11R7.5/doc/compositeproto/compositeproto.txt). Without it, all window content is drawn directly to the screen's framebuffer — a compositor has no way to intercept it.

### Redirection

The central operation is **redirection**: telling the server to draw a window (and all its descendants) into an off-screen pixmap instead of the screen framebuffer.

```c
/* Redirect the window to off-screen storage (automatic = server manages) */
XCompositeRedirectWindow(dpy, window, CompositeRedirectAutomatic);

/* Redirect all children of the root window */
XCompositeRedirectSubwindows(dpy, root, CompositeRedirectAutomatic);
```

`CompositeRedirectAutomatic` means the server manages the pixmap contents. `CompositeRedirectManual` means the compositor takes responsibility and must explicitly call `XCompositeNameWindowPixmap` to access the contents.

### NameWindowPixmap and Compositing

```c
/* Obtain a pixmap handle pointing to the window's off-screen storage */
Pixmap pix = XCompositeNameWindowPixmap(dpy, window);
```

The compositor then binds this pixmap as an OpenGL texture via `GLX_EXT_texture_from_pixmap` (AIGLX, Section 5), renders a textured quad, and presents the composited output.

### Damage Extension

Without the Damage extension, the compositor would have to repaint the entire screen every frame — prohibitively expensive. The Damage extension ([damage protocol spec](https://www.x.org/releases/X11R7.5/doc/damageproto/damageproto.txt)) tracks which regions of a drawable have been modified since the last check:

```c
Damage dmg = XDamageCreate(dpy, drawable, XDamageReportDeltaRectangles);
/* ... later, in the event loop: */
XDamageSubtract(dpy, dmg, None, repaint_region); /* Fetch and clear damage */
```

`XDamageSubtract` returns the accumulated dirty region and resets it atomically, giving the compositor a list of rectangles to repaint.

### Fixes Extension

The X Fixes extension provides region operations (union, intersection, subtract) on pixmap regions — used by compositors to clip damage rectangles — and a cursor image API. It also adds the `XFixesSetWindowShapeRegion` used for shaped (non-rectangular) windows and the `BarrierFence` mechanism for pointer barriers.

### Compositing Performance and Unredirected Windows

A composited X11 desktop has a fundamental performance cost: every window's content goes into an off-screen pixmap, then gets read as a texture, then composited into the final framebuffer. For fullscreen games, this double-copy (game → off-screen, off-screen → compositor texture → framebuffer) adds latency and wastes GPU bandwidth. Compositors implement **unredirection** for fullscreen clients: if a window covers the entire CRTC, the compositor calls `XCompositeUnredirectWindow` and reverts to direct scanout, matching the performance of an uncomposited desktop.

Mutter (GNOME's compositor) implements this via `MetaWindowActor` unredirection; KWin has an equivalent fullscreen unredirection path.

---

## 8. XRANDR and Multi-Monitor

The X Resize, Rotate, and Reflect extension (RANDR) was introduced in X11R6.7 (2002) as a simple rotate/resize mechanism; RANDR 1.2 (2006, Xorg 7.1) was a complete redesign adding full multi-monitor support with hotplug.

### RANDR Object Model

RANDR 1.2 mirrors the DRM/KMS object model but at the X protocol level:

- **Output** (`RROutput`) — a physical connector (HDMI-1, DP-2, eDP-1); has properties for EDID, backlight, scaling mode.
- **CRTC** (`RRCrtc`) — a display controller; each CRTC drives one or more outputs at a given mode.
- **Mode** (`RRMode`) — a timing mode (resolution + refresh rate).
- **Screen** — the virtual framebuffer spanning all CRTCs.

Under the hood, the `xf86-video-modesetting` DDX driver translates RANDR requests directly to DRM/KMS ioctls:

- `RRSetCrtcConfig` → `drmModeSetCrtc` or `drmModeAtomicCommit`
- `RRCreateMode` → `drmModeAddMode`
- RANDR output = DRM connector; RANDR CRTC = DRM CRTC

The **xf86-video-modesetting** driver is the generic KMS-based Xorg DDX (Device Dependent X), shipped in the `xorg-server` package since Xorg 1.17 (2014). It requires only that the GPU have a working DRM/KMS driver — it performs all display configuration via `libdrm` and leaves rendering to Mesa over DRI3. Specific hardware DDX drivers like `xf86-video-intel` (for older Intel chips) and `xf86-video-amdgpu` (legacy AMD) still exist but are increasingly replaced by modesetting.

```bash
# Query connected outputs, current modes, and CRTC configuration
xrandr --query

# Set HDMI-1 to 1920×1080@60Hz and position it right of eDP-1
xrandr --output HDMI-1 --mode 1920x1080 --rate 60 --right-of eDP-1

# Mark eDP-1 as the primary output
xrandr --output eDP-1 --primary
```

### Hotplug and `RRNotify`

When a monitor is connected or disconnected, the DRM kernel driver raises a `hotplug` event (from the display engine's HPD — Hot Plug Detect — interrupt). The `xf86-video-modesetting` DDX receives this via a `udev` event, re-enumerates connected outputs, and sends an `RRNotify` X event to clients that have subscribed with `XRRSelectInput(dpy, root, RROutputChangeNotifyMask)`. Desktop environments use this to reconfigure monitor layouts automatically.

### DPI and Scaling

RANDR 1.5 (2015) added per-CRTC transform and scale factors. However, HiDPI scaling in X11 is fundamentally broken by design: the X protocol exposes a single global DPI to clients, and many older toolkits baked pixel assumptions into layout code. GNOME under X11 uses integer scaling (2×) via RANDR's `RRSetPanning` with a fractional framebuffer upscale. This is one of the key reasons HiDPI support on X11 remains inferior to Wayland, where every surface has its own scale factor.

---

## 9. XWayland: The Bridge

XWayland is an X11 server that runs *as a Wayland client* — it speaks X11 to X applications and Wayland to the compositor. It is the mechanism by which X11 applications work on modern Wayland desktops without requiring any X11 changes in the compositor itself. XWayland was initially contributed by Kristian Høgsberg in 2013 and is maintained in the [xwayland repository](https://gitlab.freedesktop.org/xorg/xserver).

### Rootless vs. Rootful Modes

XWayland has two operating modes:

- **Rootful mode**: XWayland runs in a single top-level Wayland window that contains the entire X11 desktop. This is similar to running `Xephyr`. Useful for development and isolation but not for production desktop use.
- **Rootless mode** (the production mode): Each X11 window is mapped to a separate `wl_surface` in the Wayland compositor. The Wayland compositor's embedded X Window Manager (XWM) manages the mapping. An X11 app gets a first-class Wayland surface — it appears as a normal window in the compositor's window list, can be resized, composited, and receives input as if it were a native Wayland client.

The XWM communicates with XWayland over the same X socket. The association between an X11 window and its `wl_surface` has evolved across XWayland versions. The legacy mechanism sent a `ClientMessage` of type atom `WL_SURFACE_ID` carrying the Wayland object ID — but this had a race condition between the X and Wayland event timelines. Modern XWayland (2023+) uses the **`xwayland-shell-v1`** Wayland protocol ([Wayland Explorer](https://wayland.app/protocols/xwayland-shell-v1)): XWayland calls `xwayland_surface_v1.set_serial()` on the `wl_surface` and sends a `WL_SURFACE_SERIAL` ClientMessage on the X window, both carrying the same monotonic serial number. The compositor matches on the serial, eliminating the race. Compositors that have bound `xwayland_shell_v1` must not expect `WL_SURFACE_ID` messages.

### The DRI3 Buffer Flow

The rendering path for a hardware-accelerated X11 app under XWayland is:

1. **Mesa** opens the DRM render node via `DRI3Open` (getting back an fd from XWayland).
2. **Mesa** allocates a GBM buffer object and renders into it.
3. **Mesa** exports the BO as a DMA-BUF fd and calls `DRI3PixmapFromBuffer` → XWayland creates an `XPixmap` backed by that DMA-BUF.
4. **Mesa** calls `XPresentPixmap` (Present extension) to schedule the pixmap for display at a specific MSC.
5. **XWayland** receives the Present request, imports the pixmap's DMA-BUF as a `wl_buffer` via the `linux-dmabuf-unstable-v1` Wayland protocol, and submits it to the compositor via `wl_surface.commit`.
6. The **Wayland compositor** receives the `wl_buffer` and composites it with other surfaces at the next frame.

Critically, no pixel data is copied. The same GPU buffer travels from Mesa through the X protocol (as an fd), through XWayland (as a `wl_buffer`), to the compositor for display.

### Explicit Synchronisation

Early XWayland used implicit synchronisation (DMA-BUF implicit fences), which worked for open-source Mesa drivers that integrate with the kernel fence framework. For NVIDIA, where implicit sync is not available, this caused flickering, black windows, and rendering corruption.

XWayland 24.1 (2024) and Mesa 24.1 (2024) added support for the `linux-drm-syncobj-v1` Wayland protocol for explicit synchronisation. The DRI3 and Present extensions grew new requests to pass DRM sync objects as fds. The flow:

1. Mesa signals a DRM timeline sync object when rendering is complete.
2. Mesa passes the sync object fd to XWayland via the extended DRI3 fence protocol.
3. XWayland passes the acquire/release sync objects to the compositor via `wp_linux_drm_syncobj_surface_v1`.
4. The compositor waits on the acquire fence before scanning out, and signals the release fence when done.

This path is supported in GNOME Mutter (as of GNOME 46), KDE KWin (Plasma 6.1), and NVIDIA's `EGL-Wayland` library [Explicit sync merged into XWayland, GamingOnLinux 2024](https://www.gamingonlinux.com/2024/04/explicit-gpu-synchronization-for-xwayland-now-merged/).

### Clipboard Bridging

XWayland maintains a bridge between X11 selections (`CLIPBOARD`, `PRIMARY`) and the Wayland `wl_data_device_manager` / `zwp_primary_selection_device_manager_v1` protocols. When an X11 app copies text (takes ownership of `CLIPBOARD`), XWayland offers that content on the Wayland data device so Wayland-native apps can paste it. The reverse bridge (Wayland → X11) works similarly. This bridging is implemented in XWayland's `xwayland/selection.c`.

### HiDPI and `_XWAYLAND_MAY_GRAB_KEYBOARD`

On HiDPI displays, XWayland scales X11 content by an integer factor (set via `--scale` or negotiated from the compositor). The X protocol sees a virtual screen at 1× size; XWayland upscales all surfaces before presenting to the compositor. Fractional scaling (e.g., 1.5×) requires the `wp_viewporter` Wayland protocol and results in blurry rendering for X11 apps — a fundamental limitation since X11 clients lay out in logical pixel units.

The `_XWAYLAND_MAY_GRAB_KEYBOARD` X atom is set by compositors to indicate that XWayland-initiated keyboard grabs (e.g., from popup menus or fullscreen games) are permitted. Without this atom, XWayland ignores `XGrabKeyboard` calls to prevent X11 clients from hijacking keyboard input across the Wayland session boundary.

### XWayland Performance vs. Native Wayland

XWayland introduces several overhead sources:
- Extra latency: the Present → Wayland commit path adds one protocol round trip vs. a native Wayland client's direct `wl_surface.commit`.
- The XWM round trip for window geometry changes (X11 configure notify → Wayland role request).
- Inability to use Wayland-native protocols directly (e.g., `xdg-decoration`, `wp_color_management_v1`).

In practice, for most applications the difference is imperceptible. For latency-critical games, native Wayland (via SDL2's Wayland backend, Proton's WSILayer, or gamescope) is preferred.

---

## 10. Security: X11 vs Wayland

X11's security model is, bluntly, the absence of a security model. The following attacks are trivially implementable by any unprivileged process that can connect to the X display — which in a typical desktop session means any program the user runs:

### X11 Security Vulnerabilities

**Global keylogging.** Any X client can call `XGrabKeyboard(dpy, root, False, GrabModeAsync, GrabModeAsync, CurrentTime)` or, more subtly, register a passive key grab with `XGrabKey` on the root window. Alternatively, it can call `XSelectInput(dpy, root, KeyPressMask | KeyReleaseMask)` — the root window receives all key events before any client. The `xev` tool does exactly this; any process running as the same user can implement a keystroke logger in under 20 lines of C.

**Screen capture.** `XGetImage(dpy, root, 0, 0, width, height, AllPlanes, ZPixmap)` reads the entire screen framebuffer into a client buffer. No permission is required beyond connection authorisation. This enables both screenshot malware and password-dialog scraping.

**Input injection.** `XSendEvent(dpy, target_window, False, KeyPressMask, &event)` injects a synthetic key event into any window without the user's knowledge. Tools like `xdotool` use this legitimately for automation, but it is equally useful for attacks.

**Window reparenting.** Any client can call `XReparentWindow` to move any window to a new parent — the basis of window embedding in legacy toolkits but also a mechanism for visual spoofing.

### Xauthority: A Weak Mitigation

X11's only authentication mechanism is the `Xauthority` cookie: a shared 128-bit random value stored in `~/.Xauthority`. A client must present this cookie at connection setup. This prevents *unauthenticated* processes from connecting but does nothing to isolate clients that have already authenticated — which is every process running as the same user.

The X11 SECURITY extension (early 1990s) added trusted/untrusted client categories, but it was never widely deployed and is not enforced by default in any major distribution [X11 SECURITY extension history, OSnews](https://www.osnews.com/story/142962/the-x11-security-extension-from-the-1990s/).

### Wayland's Isolation Model

Wayland's security model stems directly from X11's failures. The Wayland compositor is the *only* entity that sees raw input and screen content:

- **Input isolation**: the compositor routes pointer and keyboard events only to the surface currently under the cursor or holding keyboard focus. No other client receives those events. Background clients cannot capture keystrokes.
- **No global screen capture**: a Wayland client cannot read pixels from any surface but its own. Screen capture requires the `xdg-desktop-portal` / `org.freedesktop.portal.ScreenCast` portal, which presents a user-visible consent dialog. The compositor implements the actual capture via `ext-image-copy-capture-v1` or `wlr-screencopy-v1`.
- **No synthetic input without portal**: global input injection requires `org.freedesktop.portal.RemoteDesktop`, also user-mediated.
- **Per-surface isolation**: each application's surface is a separate Wayland object; the compositor sees all surfaces but clients cannot reference each other's surfaces.

This means that under Wayland, the keystroke-logger example above is impossible at the application level — it would require compromising the compositor process itself, which runs with no greater privileges than the user session.

The trade-off: Wayland's isolation breaks some X11 workflows that relied on cross-client access — global hotkeys without portal infrastructure, screenshot tools that read arbitrary windows, automation frameworks built on `XSendEvent`. These are being replaced by portal-mediated alternatives, but the transition is ongoing.

---

## 11. Debugging X11 Applications

### Environment Variables

```bash
# Print verbose GLX / DRI driver loading messages
LIBGL_DEBUG=verbose glxgears

# Print verbose GLX dispatch messages (GLVND)
LIBGLX_DEBUG=verbose glxgears

# Force software (llvmpipe) rendering, bypassing DRI
LIBGL_ALWAYS_SOFTWARE=1 glxgears

# Force indirect GLX rendering (all GL calls go through X server)
LIBGL_ALWAYS_INDIRECT=1 glxgears

# Override the X display
DISPLAY=:1 myapp

# Print Mesa driver environment variables
MESA_DEBUG=1 glxgears
```

### Nested X Servers

**Xephyr** is an Xorg backend that renders its clients into a Wayland or X11 window — useful for testing window managers and X11 applications in isolation:

```bash
Xephyr :5 -screen 1280x720 &
DISPLAY=:5 openbox &
DISPLAY=:5 xterm
```

**Xvfb** (virtual framebuffer) provides a headless X11 server for CI:

```bash
Xvfb :99 -screen 0 1280x720x24 &
DISPLAY=:99 glxinfo
```

### X11 Inspection Tools

```bash
# Display server information, extension list, protocol version
xdpyinfo -display :0

# Window properties and class/name
xprop -display :0 -root
xprop -display :0 -id $(xdotool getactivewindow)

# Window geometry and tree
xwininfo -display :0 -root -tree

# All atoms on the display
xlsatoms -display :0

# GLX renderer, version, and extension information
glxinfo -B -display :0
glxinfo -display :0 | grep -E "render|direct|vendor|version"
```

### Protocol Tracing

**Xtrace** (`x11trace`) intercepts X protocol traffic between a client and the server, printing decoded requests and events — the X equivalent of `strace` for socket traffic:

```bash
# Launch app through Xtrace, logging to file
xtrace --display :0 --new-display :10 -- myapp &
# The app connects to :10; Xtrace proxies to :0 and logs everything
```

For DRI3 and Present traffic specifically, look for `DRI3Open`, `DRI3PixmapFromBuffer`, `PresentPixmap`, and `PresentCompleteNotify` messages in the trace.

**strace** on the X socket directly (though verbose):

```bash
strace -e trace=sendmsg,recvmsg -p $(pgrep myapp) 2>&1 | head -100
```

### Mesa DRI Debugging

```bash
# Show which DRI driver is loaded and the render node used
LIBGL_DEBUG=verbose glxinfo 2>&1 | grep -i "dri\|render\|device"

# Mesa shader compile debug (dumps NIR to stderr)
MESA_SHADER_DUMP_PATH=/tmp/shaders glxgears

# AMD GPU: print command stream
RADEON_DUMP_CS=1 glxgears

# Full GL error checking via Mesa's built-in debug context
MESA_DEBUG=gl_errors glxgears
```

For issues specifically with XWayland, check Wayland compositor logs and look for DRI3 errors:

```bash
# Run XWayland manually for testing (separate from compositor)
Xwayland :10 -rootless 2>&1 | grep -i "dri3\|present\|sync"

# NVIDIA explicit sync status
LIBGL_DEBUG=verbose glxinfo 2>&1 | grep -i "syncobj\|explicit"
```

---

## 12. Integrations

- **Chapter 1 (DRM Architecture)**: DRI3 uses DRM render nodes (`/dev/dri/renderD128`) opened via `DRI3Open`. The `xf86-video-modesetting` DDX wraps KMS/DRM ioctls for all RANDR operations on Xorg. The Present extension's MSC maps directly to the DRM vblank event counter.

- **Chapter 4 (GPU Memory Management)**: DRI3 buffer passing uses DMA-BUF file descriptors; the GPU buffers that Mesa exports to XWayland via `DRI3PixmapFromBuffer` are GBM buffer objects allocated on the DRM render node. GBM's `gbm_bo_get_fd` produces the fd that DRI3 transports over the X socket.

- **Chapter 12 (Mesa Loader and Driver Dispatch)**: The GLX driver loading path inside Mesa (`src/glx/`) is how X11 applications obtain OpenGL contexts. GLVND's ICD dispatch (`libGLX.so.0`) selects the Mesa ICD for the X screen. The same Mesa loader that Chapter 12 describes for EGL is reused for GLX context creation.

- **Chapter 20 (Wayland Protocol Fundamentals)**: XWayland is a Wayland client that uses `linux-dmabuf-unstable-v1` to hand DRI3 DMA-BUF frames to the compositor. It uses `wl_data_device_manager` for clipboard bridging, `wp_viewporter` for HiDPI scaling, and the `xwayland-shell-v1` protocol for race-free X11-window-to-Wayland-surface association. The `linux-drm-syncobj-v1` protocol (Chapter 75) is used for explicit sync between XWayland and the compositor.

- **Chapter 23 (Legacy and Sandboxed App Support)**: Chapter 23 covers XWayland from the application-compatibility perspective — how XDG portals bridge X11 clipboard and screen-capture to Wayland sandboxed apps. This chapter covers the *X11 protocol and DRI stack* that XWayland implements internally, providing the complementary kernel-level view.

- **Chapter 30 (Debugging and Profiling)**: The debugging tools listed in Section 11 (`LIBGL_DEBUG`, `LIBGLX_DEBUG`, `xdpyinfo`, `xlsatoms`, `Xtrace`) are the X11-specific counterparts to the general Mesa/Vulkan debugging coverage in Chapter 30. The `MESA_DEBUG` and NIR dump mechanisms described there apply equally to GLX-based OpenGL contexts.

- **Chapter 75 (Explicit GPU Synchronisation)**: The `linux-drm-syncobj-v1` Wayland protocol used by XWayland is the Wayland surface of the DRM sync object infrastructure covered in Chapter 75. XWayland's explicit sync handshake — exporting DRM timeline sync objects as fds, passing them via the DRI3 fence protocol, then forwarding them via `wp_linux_drm_syncobj_surface_v1` — exercises the full explicit sync stack from Mesa fence to compositor.

---

## References

- [X Window System Protocol, version 11 (RFC 1013)](https://www.rfc-editor.org/rfc/rfc1013.html) — original 1987 specification alpha
- [X11 Protocol Reference (X.Org, X11R7.6)](https://www.x.org/releases/X11R7.6/doc/xproto/x11protocol.html)
- [GLX_EXT_texture_from_pixmap specification (Khronos)](https://registry.khronos.org/OpenGL/extensions/EXT/GLX_EXT_texture_from_pixmap.txt)
- [DRI3 protocol specification (cgit.freedesktop.org)](https://cgit.freedesktop.org/xorg/proto/dri3proto/tree/dri3proto.txt)
- [DRI2 Extension Version 2.0 (x.org)](https://www.x.org/releases/X11R7.6/doc/dri2proto/dri2proto.txt)
- [Composite Extension Version 0.4 (x.org)](https://www.x.org/releases/X11R7.5/doc/compositeproto/compositeproto.txt)
- [Fedora AIGLX Project wiki](https://fedoraproject.org/wiki/RenderingProject/aiglx)
- [CompositeSwap — DRI2 compositing race analysis (freedesktop.org)](https://dri.freedesktop.org/wiki/CompositeSwap/)
- [XWayland architecture overview (wayland.freedesktop.org)](https://wayland.freedesktop.org/docs/book/Xwayland.html)
- [Explicit GPU synchronisation for XWayland merged (GamingOnLinux, April 2024)](https://www.gamingonlinux.com/2024/04/explicit-gpu-synchronization-for-xwayland-now-merged/)
- [X11 security isolation analysis (Tim Starling's blog, 2016)](https://tstarling.com/blog/2016/06/x11-security-isolation/)
- [The X11 SECURITY extension history (OSnews, 2025)](https://www.osnews.com/story/142962/the-x11-security-extension-from-the-1990s/)
- [Mesa source tree — src/glx/](https://github.com/Mesa3D/mesa/tree/main/src/glx)
- [xf86-video-modesetting (cgit.freedesktop.org)](https://cgit.freedesktop.org/xorg/driver/xf86-video-modesetting/)

## Roadmap

### Near-term (6–12 months)
- **XWayland fractional scaling improvements**: The `wp_fractional_scale_v1` Wayland protocol is being integrated more deeply into XWayland to reduce blurriness when running X11 apps at non-integer HiDPI scales (e.g., 1.5×, 1.25×); work is tracked in the XWayland merge request queue at freedesktop.org.
- **Explicit sync stabilisation**: The `linux-drm-syncobj-v1` Wayland protocol (shipped experimentally in XWayland 24.1 and Mesa 24.1) is being hardened across GNOME Mutter, KDE KWin, and the NVIDIA driver stack; remaining edge cases with multi-GPU (PRIME) and GLX compositing are being addressed in Mesa's `src/glx/` explicit-sync paths.
- **`xwayland-shell-v1` adoption**: Compositors that still rely on the legacy `WL_SURFACE_ID` ClientMessage for X11-window-to-Wayland-surface association are migrating to the race-free `xwayland-shell-v1` protocol; GNOME and KDE completed adoption in 2024, with smaller compositors (labwc, Hyprland) following.
- **DRI3 v1.4 multi-plane buffer extensions**: Proposed DRI3 protocol additions to handle multi-plane modifier-negotiation for compressed framebuffer formats (AFBC, UBWC) are being discussed in the xorg-devel mailing list, driven by mobile/embedded use cases where X11 still runs on ARM SoCs.

### Medium-term (1–3 years)
- **Xorg server maintenance-only mode**: The X.Org Foundation has signalled that Xorg will enter maintenance-only status — no new feature development, only security and bug fixes — as the ecosystem completes migration to Wayland. The modesetting DDX and XWayland are expected to remain actively developed as the primary compatibility surface.
- **XWayland input method (IME) protocol**: A standardised Wayland protocol for input method editors (`zwp_input_method_v2` successor) is being designed that XWayland can bridge to X11 applications using XIM, eliminating the current breakage of CJK input methods under XWayland on many compositors.
- **Indirect GLX removal from Mesa**: Mesa's indirect GLX rendering path (`libGL.so.1` server-side GLX request handling) is slated for removal after a deprecation period; the `LIBGL_ALWAYS_INDIRECT` fallback path will be dropped, and applications that require indirect rendering will need to use software rendering (llvmpipe/softpipe) explicitly.
- **XDG portal expansion for remaining X11 gaps**: The `org.freedesktop.portal.GlobalShortcuts` and `org.freedesktop.portal.InputCapture` portals are being finalised to replace the last widely-used X11-only workflows (global hotkeys, gaming input capture) that currently require XWayland keyboard-grab workarounds.

### Long-term
- **X11 protocol freeze and archival**: As XWayland matures and the remaining X11 application base either ports to Wayland or reaches end-of-life, X11 as an active protocol is expected to be frozen in place rather than extended — new display features (HDR, variable refresh rate, colour management) will only be available through native Wayland protocols.
- **GLX deprecation in Mesa**: Once the major Linux desktop toolkits (GTK4, Qt 6) complete their EGL-only paths and XWayland itself switches to EGL/DRI3 for all rendering, the GLX implementation in Mesa (`src/glx/`) is expected to be deprecated and eventually removed, with EGL-on-Wayland becoming the sole OpenGL surface path.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
