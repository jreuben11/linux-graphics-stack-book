# Chapter 22: Production Compositors

> **Part**: Part VI — The Display Stack
> **Audience**: Both — systems developers gain architectural insight into production compositor internals; application developers understand the feature capabilities and extension support that determines what their applications can rely on
> **Status**: First draft — 2026-06-06

## Table of Contents

- [Overview](#overview)
- [1. Compositor Landscape Overview](#1-compositor-landscape-overview)
- [2. Mutter: GNOME Shell's Compositor](#2-mutter-gnome-shells-compositor)
- [3. KWin: KDE Plasma's Compositor](#3-kwin-kde-plasmas-compositor)
- [4. Sway: Tiling with wlroots](#4-sway-tiling-with-wlroots)
- [5. Hyprland: Animations and Extensibility](#5-hyprland-animations-and-extensibility)
- [6. gamescope: Valve's Micro-Compositor](#6-gamescope-valves-micro-compositor)
- [7. Protocol Extension Landscape and Compositor Compatibility](#7-protocol-extension-landscape-and-compositor-compatibility)
- [8. Common Compositor Debugging Techniques](#8-common-compositor-debugging-techniques)
- [9. Measuring Compositor Latency and Frame Pacing](#9-measuring-compositor-latency-and-frame-pacing)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

Five production Wayland compositors collectively define the Linux desktop experience in 2026: Mutter powers GNOME Shell, KWin powers KDE Plasma, Sway provides i3-compatible tiling on Wayland, Hyprland delivers animated dynamic tiling with an extensible plugin system, and gamescope is Valve's game-focused micro-compositor that serves as the session compositor on Steam Deck. Each has made architectural trade-offs that reflect its design goals, and those trade-offs propagate into every corner of the graphics stack — from which KMS features get exposed to applications, to which Wayland protocol extensions developers can safely target.

This chapter examines each compositor's architecture from the perspective of graphics developers and systems programmers: how they drive KMS, how they structure their rendering pipeline, how they handle multi-GPU configurations, and which protocol extensions they support. The material builds directly on Chapter 20 (Wayland Fundamentals) and Chapter 21 (wlroots), and connects forward to Chapter 23 (XWayland), Chapter 26 (hardware video), and Chapter 28 (Windows compatibility via Proton). The goal is not to document user-facing configuration but to explain the internal mechanisms that matter when you are writing clients, drivers, or compositor extensions.

After reading this chapter you will understand why Mutter offloads all KMS ioctls to a dedicated thread, why KWin was able to implement HDR and explicit synchronisation ahead of every other desktop compositor, what makes gamescope uniquely suited to gaming workloads, and how to approach the problem of writing portable Wayland applications when protocol support varies across compositors.

---

## 1. Compositor Landscape Overview

The Linux compositor ecosystem can be characterised along several axes. The first is backend provenance: Sway and Hyprland are built on wlroots, the shared compositor toolkit described in Chapter 21, while Mutter, KWin, and gamescope each maintain a custom DRM/KMS backend. This distinction matters because wlroots-based compositors inherit the same rendering abstractions and protocol implementations from a shared library, while custom-backend compositors are free to optimise — or specialise — their entire display path. gamescope takes this the furthest, implementing a Vulkan-first compositor where wlroots would be entirely the wrong abstraction.

The second axis is purpose. Mutter and KWin are full desktop compositors deeply integrated with their respective desktop environments; they manage the session, handle input, run window management logic, and provide a rich effects pipeline. Sway and Hyprland are standalone compositors with no desktop shell dependency — they implement just enough infrastructure for a productive tiling workflow. gamescope is a micro-compositor: it hosts a single application (or a small cluster of applications forming a game plus overlays) and presents the result to a display with the lowest possible latency.

The third axis is layout philosophy. KWin and Mutter both support floating and tiling window arrangements (the latter via GNOME's overview mode and KDE's Window Tiling feature). Sway implements strict tree-based tiling in the style of i3. Hyprland implements dynamic tiling with first-class animation support. gamescope does not implement window management in the traditional sense at all: it gives the game the full display.

Looking at protocol extension support, KWin leads the field: it was the first desktop compositor to ship `wp_linux_drm_syncobj_v1` (explicit sync, Plasma 6.1), and it has the most complete implementation of HDR output management via `HDR_OUTPUT_METADATA` KMS properties (Plasma 6.0). Mutter added `wp_color_management_v1` server support in GNOME 48 (2025). gamescope implements its own HDR pipeline independently of the standardised protocols, bypassing the compositor protocol layer entirely when running in direct KMS mode. Sway and Hyprland, being wlroots-based, inherit new protocol implementations as wlroots adds them, which means they tend to lag the custom-backend compositors on cutting-edge features but gain those features without compositor-specific engineering effort.

The following sections examine each compositor in detail, beginning with the two most architecturally complex: Mutter and KWin.

---

## 2. Mutter: GNOME Shell's Compositor

### Architecture

Mutter is a library, not an application. GNOME Shell loads Mutter as a library and exposes its functionality through a GObject-based API to GNOME Shell's JavaScript (GJS) scripting environment. This separation of concerns is important: window management policy lives in GNOME Shell's JavaScript code, while the mechanics of compositing, KMS interaction, and Wayland protocol handling live in Mutter's C implementation. When users see GNOME's animations — the workspace slide, the application spread, the magic lamp minimize — those animations are implemented as JavaScript calling into Mutter's Clutter-based scene graph.

The central architectural divide in Mutter is the choice of backend. `MetaBackendNative` is the production backend for Wayland sessions: it directly owns the DRM file descriptor, drives KMS, allocates GBM buffers, and manages EGL surfaces. `MetaBackendX11` is used when Mutter runs as a compositing manager inside an existing X11 session, relying on X11's window system infrastructure rather than raw KMS. On a modern GNOME desktop, `MetaBackendNative` is what runs.

### The KMS Object Model

Mutter wraps the KMS resource hierarchy described in Chapter 2 with a set of GObject classes. A `MetaGpu` represents a GPU (a `/dev/dri/cardN` device). Each GPU contains `MetaCrtc` objects (hardware display controllers) and `MetaOutput` objects (physical connectors). `MetaMonitor` represents the logical display as presented to the user — a monitor can be formed from multiple CRTCs in the case of hardware cloning, or from a single CRTC for the common case. `MetaMonitorManager` is the singleton that tracks all monitors, processes hotplug events from DRM, and exposes the display layout to GNOME Shell.

### The KMS Thread

The most sophisticated piece of engineering in Mutter is its KMS thread design. The problem it solves is straightforward to state but subtle to implement: `ioctl(DRM_IOCTL_ATOMIC_COMMIT)` can block for up to one full VBlank interval (16.7 ms at 60 Hz) when the kernel is waiting for the previous page flip to complete before accepting the next one. If this ioctl runs on the Clutter main thread — where all GJS execution, input event dispatch, and scene graph traversal also happen — then every frame causes the entire shell UI to stall for up to one VBlank. At 60 Hz this is acceptable in most cases, but under real-time scheduling constraints or on high-refresh-rate displays, it becomes a source of jitter and dropped frames.

Mutter's solution is the `MetaKms` subsystem: all KMS ioctls run on a dedicated kernel mode-setting thread. The public-facing `MetaKms` API is safe to call from the main thread; it enqueues work items that the KMS thread processes asynchronously. A `MetaKmsUpdate` object represents a transactional batch of KMS operations — plane assignments, mode sets, property changes — that will be processed atomically when posted via `meta_kms_device_post_pending_update`. The KMS thread picks up these updates, submits the atomic commit, and notifies the main thread via a callback when the page flip event arrives.

```c
/* Source: mutter/src/backends/native/meta-kms-update.h and meta-kms-device.h
 * (GNOME/mutter main branch, June 2026)
 * MetaKmsUpdate represents a single atomic commit transaction.
 * Callers build up the update on the main thread, then post it;
 * the KMS thread owns the actual DRM_IOCTL_ATOMIC_COMMIT call.
 */

/* Create a new update transaction for a specific device */
MetaKmsUpdate *update = meta_kms_update_new(kms_device);

/* Assign a framebuffer to a plane; last arg is MetaKmsAssignPlaneFlag */
meta_kms_update_assign_plane(update, crtc, plane, buffer,
                              src_rect, dst_rect,
                              META_KMS_ASSIGN_PLANE_FLAG_NONE);

/* Set the CRTC gamma via a MetaGammaLut struct */
meta_kms_update_set_crtc_gamma(update, crtc, gamma);

/* Post asynchronously; the KMS thread takes ownership and calls
 * DRM_IOCTL_ATOMIC_COMMIT without blocking the Clutter loop.
 * Use meta_kms_device_process_update_sync() for a blocking variant. */
meta_kms_device_post_update(kms_device, update,
                             META_KMS_UPDATE_FLAG_NONE);
```

In 2024, Mutter switched the KMS thread from real-time (`SCHED_RR`) priority to high-priority scheduling to avoid situations where the real-time thread could hold the CPU indefinitely and cause GNOME Shell's process to be killed by the watchdog. The underlying design — a dedicated thread owning all KMS ioctls — remains unchanged.

### The Clutter Scene Graph and Repaint Loop

Mutter uses the Clutter toolkit as its scene graph. Every visible element is a `ClutterActor`; the root is a `ClutterStage` mapped to one EGL surface per CRTC. Within the scene graph, `MetaSurfaceActor` wraps a `MetaWaylandSurface` (for Wayland clients) or an XWayland window (as `MetaSurfaceActorX11`). Each window is represented by a `MetaWindowActor` that contains a `MetaSurfaceActor` plus space for decorations, shadows, and effect rendering.

The repaint loop is damage-driven. When a client commits a new buffer to a `wl_surface`, Mutter marks the corresponding `MetaSurfaceActor` as damaged and calls `clutter_stage_schedule_update`. Clutter's frame clock — a `ClutterFrameClock` tracking VBlank intervals using `CLOCK_MONOTONIC` timestamps from DRM page flip events — schedules a repaint callback timed to arrive just before the predicted VBlank. The repaint traverses the scene graph, composites dirty actors into the EGL framebuffer, and then posts a `MetaKmsUpdate` with the result to the KMS thread for atomic commit.

### Effects and the MetaPlugin Interface

GNOME Shell effects are implemented through `MetaPlugin`, a GObject interface that receives lifecycle notifications for windows. The `map`, `minimize`, `destroy`, and `switch_workspace` virtual methods are called by Mutter at the appropriate moments; GNOME Shell's default plugin implements them using Clutter timeline transitions. GPU shader effects are applied via `ClutterEffect` and `ClutterOffscreenEffect`, which redirect a subtree of the scene graph to an offscreen framebuffer, apply a fragment shader, and composite the result back.

### Colour Management

Mutter's `MetaColorManager` reads ICC profiles from `colord`, maps them to KMS colour transformation matrix (`CTM`) and gamma (`GAMMA_LUT`) properties, and programs them through the `MetaKmsUpdate` path. With GNOME 48, Mutter merged server-side support for `wp_color_management_v1` — but note that as of wayland-protocols 1.47 (December 2025), this protocol remains in the **staging** category (not yet promoted to stable), at version 2. Applications using `wp_color_management_v1` should check for protocol availability at runtime and handle its absence gracefully, as discussed in Section 7.

### XWayland in Mutter

`MetaXWaylandManager` supervises the XWayland process. Mutter starts XWayland lazily on first demand from an X11 application and emits a "ready" signal when XWayland's listening socket is available. Override-redirect windows (tooltips, popup menus, xdg-popup equivalents in the X11 world) require special handling because they bypass the normal window manager stack; Mutter tracks them via `META_WINDOW_OVERRIDE_REDIRECT` and composites them above all other windows. See Chapter 23 for the full XWayland architecture.

### EGLStreams Removed in GNOME 51

GNOME 51 (2025) removed `EGLDevice` / `EGLStreams` support from Mutter. EGLStreams was an NVIDIA-proprietary buffer-sharing mechanism dating from the GNOME 3.x era that allowed NVIDIA's proprietary driver to hand buffers to Mutter without going through the GBM / `linux-dmabuf` path used by every other GPU driver. The mechanism required Mutter to maintain a separate code path — a `MetaRendererNativeGpuData` mode for EGLDevice alongside the standard GBM mode — increasing complexity without benefiting users on modern hardware.

The removal is safe because all current NVIDIA configurations on Wayland use GBM and `linux-dmabuf`. NVIDIA's proprietary driver added GBM support in driver version 495; nvidia-open, which ships with all Turing+ GPUs in distributions that adopt it, has used GBM from its first release. Explicit sync via `wp_linux_drm_syncobj_v1` (Chapter 3) — the mechanism that closed the NVIDIA tearing and frame-race issue — operates on top of `linux-dmabuf`, not EGLStreams. The EGLStreams removal does not affect explicit sync. See Chapter 3 for the distinction between these two mechanisms.

---

## 3. KWin: KDE Plasma's Compositor

### Architecture Overview

KWin is simultaneously an X11 window manager and a Wayland compositor. In an X11 session it uses the X Composite extension to obtain off-screen redirects of all windows and composites them using OpenGL or QPainter. In a Wayland session it is the DRM/KMS owner and composites Wayland surfaces directly. The `KWin::Compositor` class hierarchy handles both modes; the Wayland path is routed through `KWin::WaylandCompositor`, which owns the `DrmBackend`.

### The Rendering Scene

KWin's rendering is abstracted by a scene system. `SceneOpenGL` is the production renderer: it uses KWin's own OpenGL/GLES2 wrapper layer (`GLPlatform`, `GLShader`, `GLFramebuffer`) rather than calling Mesa directly, which gives KWin control over shader compilation and caching. `ScenePainter` is a software fallback using QPainter, used in cases where GPU rendering is unavailable. A Vulkan backend has been under development but was not in production use as of early 2026. Each output gets a `SceneDelegate` that owns the per-output rendering pass.

### The Effect System

KWin's effect system is the most powerful in any Linux desktop compositor. Every effect is a shared library implementing the `Effect` base class. At render time, KWin calls each effect's `prePaintScreen`, `paintScreen`, `prePaintWindow`, and `paintWindow` hooks, allowing effects to intercept and modify the rendering pipeline at multiple levels.

```cpp
/* Source: kwin/src/effect/effect.h and a typical effect plugin
 * An effect overrides paintWindow to modify how each window is drawn.
 * PaintWindowData carries opacity, brightness, saturation, and the
 * damage region; effects can alter these before calling the next stage.
 */
class MyEffect : public KWin::Effect {
    Q_OBJECT
public:
    void paintWindow(KWin::PaintWindowData &data) override {
        if (isAnimating(data.window())) {
            data.setOpacity(data.opacity() * m_animProgress);
        }
        effects->drawWindow(data);  /* call down the chain */
    }
private:
    float m_animProgress = 1.0f;
};
```

Built-in effects include the Kawase blur (a dual-pass blur using progressive downscaling that achieves large blur radii at low cost), the "Magic Lamp" minimize animation, "Wobbly Windows" (a spring-mesh simulation applied to window vertices at render time), and the Overview effect that implements the GNOME-style expose view. The blur effect uses a compositor-specific Wayland extension — `org_kde_kwin_blur` — to allow clients to request blurred backgrounds beneath transparent windows.

### DRM/KMS Backend

KWin's DRM backend is implemented in `kwin/src/backends/drm/`. The object hierarchy mirrors the kernel's: `DrmBackend` owns one or more `DrmGpu` objects, each of which contains `DrmOutput`, `DrmCrtc`, `DrmPlane`, and `DrmConnector` objects. All modesetting uses the atomic API; `DrmAtomicCommit` wraps the `drmModeAtomicCommit` call with TEST_ONLY support for validating configurations before committing them.

Multi-GPU operation places the rendering GPU (typically the discrete GPU) as the primary and uses secondary GPUs for outputs. The `DrmGpu::renderingEnabled()` flag controls whether a GPU participates in rendering or only in scanout, enabling configurations like an NVIDIA discrete GPU driving an external display via KMS while an integrated Intel GPU handles the internal laptop panel.

### Direct Scanout

When a Wayland client presents a buffer that is already at the correct display resolution, in a format and with a modifier that the target KMS plane supports natively, KWin can bypass its compositing pass entirely and place the client buffer directly on a hardware plane. The check in `DrmOutput::tryDirectScanout` tests the buffer's format and DRM format modifier against the plane's supported list, then submits a TEST_ONLY atomic commit to verify that the kernel's driver will accept the assignment. If the test passes, KWin submits the real commit with the client buffer occupying the primary plane — no GPU compositing occurs. For full-screen games on native resolution this can eliminate a full GPU render pass per frame and reduce latency significantly.

### HDR Support (Plasma 6.0+)

KWin's HDR implementation, the most complete on any Linux desktop compositor as of 2026, was introduced in Plasma 6.0 (released February 2024). The central abstraction is `ColorDescription`, which models the colour volume, peak luminance, and transfer function of a surface or output. KWin's compositing shader performs tone mapping from the source colour space to the output colour space for every window — an HDR game can be composited alongside SDR desktop windows because KWin operates in a linearised scene-referred colour space throughout.

On the output side, KWin maps `ColorDescription` to the `HDR_OUTPUT_METADATA` and `COLORSPACE` KMS properties via `DrmOutput::setColorDescription`. It discovers HDR-capable displays by reading EDID HDRDB (HDR Static Metadata Descriptor Block) information through the `DrmOutput::information()` path and checking for `HDR_OUTPUT_METADATA` plane support in the atomic property namespace.

Client-side HDR metadata arrives via `wp_color_management_v1` (staging, version 2 as of 2025) and is attached to the surface's `ColorDescription`. KWin then incorporates that metadata into its tone-mapping pass.

### Explicit Synchronisation (Plasma 6.1+)

The `wp_linux_drm_syncobj_v1` protocol, which allows clients and compositors to express acquire/release synchronisation using DRM timeline semaphores, was first implemented in KWin for Plasma 6.1 (June 2024). This was a particularly important feature for NVIDIA proprietary driver users: before explicit sync, the implicit synchronisation model in Wayland caused severe rendering artefacts because NVIDIA's kernel driver does not participate in the implicit fence infrastructure that Mesa-based drivers provide. With explicit sync, clients attach DRM syncobj timeline points to each buffer commit; KWin waits on the acquire point before reading the buffer and signals the release point after the compositing pass completes. This eliminated an entire class of visual corruption that had plagued NVIDIA Wayland users.

```cpp
/* Source: kwin/src/backends/drm/drm_output.cpp (illustrative)
 * When explicit sync is active, the DRM atomic commit carries timeline
 * fence acquire/release points rather than relying on implicit fencing.
 * The acquire point must be signalled by the client before KWin reads
 * the buffer; the release point is signalled after scanout completes.
 */
DrmAtomicCommit commit(m_pipeline->gpu());
if (m_syncObj) {
    commit.addProperty(m_primaryPlane, DrmPlane::PropertyIndex::InFence,
                       m_syncObj->acquireTimelinePoint());
    commit.addProperty(m_primaryPlane, DrmPlane::PropertyIndex::OutFence,
                       m_syncObj->releaseTimelinePoint());
}
commit.addProperty(m_primaryPlane, DrmPlane::PropertyIndex::FbId,
                   m_currentBuffer->framebuffer()->id());
commit.commit();
```

### VRR and Scripting

KWin implements per-output VRR (Variable Refresh Rate / adaptive sync) through `DrmOutput::setVrrPolicy`. Applications can request VRR via the `wp_fifo_v1` protocol or via KWin's own game detection heuristics. The scripting API allows users to write JavaScript rules that are evaluated for every window (`KWin.readConfig`, `registerShortcut`, the `org.kde.kwin.qml` QML context), enabling custom window management policies without writing a KWin plugin.

---

## 4. Sway: Tiling with wlroots

### Relationship to i3 and Architecture

Sway is the spiritual successor to i3 on Wayland. It implements the same configuration file syntax, the same tree-based tiling algorithm, and the same IPC protocol as i3, allowing users to migrate from an i3 X11 setup with minimal change to their workflows. Underneath, Sway is built on wlroots — it delegates DRM/KMS management, input handling, EGL/GLES2 rendering, and core Wayland protocol implementations to the wlroots library described in Chapter 21.

The top-level initialisation in `sway/server.c` creates a `wlr_backend` (which wlroots auto-selects as DRM, libinput, and Wayland/X11 as appropriate), a `wlr_renderer` (GLES2 for hardware, Pixman for software), and a `wlr_compositor`. From there, Sway registers listeners for wlroots events — `wlr_xdg_shell::new_surface`, `wlr_output::frame`, `wlr_seat::pointer_grab_begin` — and implements its window management logic on top of them.

### The Container Tree

Sway's layout is a rooted tree. At the top level, a workspace belongs to an output; within a workspace, containers can be arranged in horizontal or vertical split mode or in tabbed/stacked mode. Leaf nodes are `sway_container` objects that wrap a `wlr_xdg_surface` (for Wayland clients) or a `wlr_xwayland_surface`. Floating windows exist in a separate list attached to the workspace and are rendered above tiled containers.

Window criteria — the `[app_id="firefox"]` matchers familiar from i3 — are evaluated against the `app_id` and `title` properties of `wlr_xdg_toplevel` (for Wayland) or the WM_CLASS and WM_NAME X11 properties (for XWayland). Sway evaluates criteria at window creation and when window properties change, allowing `assign` rules to direct specific applications to pre-determined workspaces or layouts.

### IPC Protocol

Sway's IPC protocol is a superset of i3's, carried over a Unix domain socket. Commands are JSON-encoded messages with a fixed 14-byte header (magic string, payload length, message type). The `swaymsg -t subscribe '["window", "workspace"]'` command subscribes to events; Sway emits a JSON object for each event containing the event type, the affected container's properties (app_id, title, focused, PID, workspace), and the triggering action.

```bash
# Source: sway IPC protocol, sway-ipc(7) manpage
# Subscribe to window focus events and print each event as it arrives.
swaymsg -t subscribe '["window"]' | while IFS= read -r event; do
    echo "$event" | jq '.change, .container.app_id, .container.focused'
done
```

Applications that need to communicate with Sway — status bars, window switchers, notification daemons — use this IPC interface. The `swaybar` protocol extends IPC to allow a status bar program to stream JSON "blocks" to Sway for rendering in the bar, identical to i3's JSON bar protocol.

### Layer Shell and Security

Sway uses `zwlr_layer_shell_v1` for its own system UI: `swaybar` is a layer shell surface on the bottom layer, `swaylock` uses `ext_session_lock_v1` to present a secure lock screen that the compositor prevents other surfaces from covering or screenshotting. When `ext_session_lock_v1` is active, Sway does not allow `zwlr_screencopy_manager_v1` captures, preventing applications from screenshotting through the lock screen — a security property that depends on Sway's cooperation and is not enforced at the Wayland protocol level alone.

Sway's rendering path does not include a custom effects pipeline. Window transitions are instantaneous; there are no animations. The wlroots scene graph handles damage tracking and GLES2 compositing. This simplicity is a feature for users who prioritise latency and predictability over visual effects.

---

## 5. Hyprland: Animations and Extensibility

### Design Philosophy

Hyprland occupies a position between Sway's minimalism and KWin's full-featured complexity. It is wlroots-based like Sway, but has invested heavily in a custom rendering layer, a first-class animation system, and an extensible plugin API — features that Sway deliberately excludes. The result is a compositor that offers animated workspace transitions, rounded window corners, per-window blur, and drop shadows while still operating within the wlroots abstraction layer.

This rapid evolution comes with a trade-off: Hyprland's internal APIs change frequently between versions. The discussion here focuses on stable, documented concepts rather than internal function signatures that may change in a minor release.

### The Animation System

`CHyprAnimationManager` manages all time-varying properties in Hyprland. Every animatable property — window position, workspace render offset, window alpha, blur radius — is wrapped in an `CAnimatedVariable` that holds a current value, a target value, a duration, and a bezier-curve easing function. On each compositor frame, `CHyprAnimationManager` advances all active animations by the elapsed frame time and marks affected windows as needing repaint.

The bezier curve system is user-configurable via the `hypr-animations.conf` DSL: users define named curves by their two control points (`bezier = mySmooth, 0.05, 0.9, 0.1, 1.05`) and assign them to specific animation categories (`animation = windows, 1, 7, mySmooth`). This gives fine-grained control over the feel of each animation type without requiring a plugin.

### Custom Rendering Pipeline

Hyprland wraps the wlroots GLES2 renderer through a `CRenderer` class that adds additional rendering passes. Rounded window corners are implemented by rendering each window to an offscreen buffer, then sampling that buffer through a GLSL fragment shader that discards fragments outside a rounded rectangle mask. Window blur is a separate multi-pass operation: Hyprland captures the area behind a transparent window, applies iterative downscaling and upscaling blur passes, and composites the result as a background before drawing the window itself.

Window animation during close uses `renderSnapshot` — a mechanism that captures a window's last rendered state into a texture and continues animating that texture even after the underlying Wayland surface has been destroyed. This is architecturally similar to how other compositors handle close animations but implemented independently in Hyprland's rendering layer.

### The Plugin API

Hyprland's plugin API allows third-party code to hook into the compositor at well-defined points. Plugins are shared libraries loaded via `hyprpm`, Hyprland's plugin manager. The `HyprlandAPI` header declares the stable interface: plugins register for lifecycle hooks (`HyprlandAPI::registerHook`), keybindings, configuration variables, and IPC commands.

```cpp
/* Source: Hyprland/src/plugins/PluginAPI.hpp and a typical plugin
 * The APICALL macro handles cross-library symbol visibility.
 * Plugins register hooks at load time and Hyprland calls them at the
 * appropriate points in the render/event pipeline.
 */
APICALL EXPORT std::string PLUGIN_API_VERSION() {
    return HYPRLAND_API_VERSION;
}

APICALL EXPORT PLUGIN_DESCRIPTION_INFO PLUGIN_INIT(HANDLE handle) {
    HyprlandAPI::registerHook(handle, "preRender",
        [](void *self, SCallbackInfo &info, std::any data) {
            auto *wksp = std::any_cast<CWorkspace *>(data);
            /* manipulate render state before drawing this workspace */
        });
    return {"My Plugin", "Example hook plugin", "1.0.0"};
}
```

The `hyprspace` overview plugin is a widely-used example: it implements a GNOME-like expose mode by scaling and repositioning all windows into a grid, using Hyprland's animation infrastructure for the transition.

### Hyprland-Specific Protocols

Hyprland defines several compositor-specific Wayland protocols: `hyprland_global_shortcuts_manager_v1` allows applications to register global keybindings; `hyprland_toplevel_export_manager_v1` enables per-window screenshots without going through the full `zwlr_screencopy_manager_v1` path; `hyprland_focus_grab_v1` provides a session-modal focus grab mechanism. These protocols are only available on Hyprland; applications using them must detect availability via `wl_registry` and implement fallback behaviour for other compositors.

Hyprland's IPC protocol is its own JSON socket format, not i3-compatible. The `hyprctl` command-line tool covers most administrative use cases; the socket protocol supports event subscription for window, monitor, and keyboard events.

---

## 6. gamescope: Valve's Micro-Compositor

### Design Goals and Architecture

gamescope is a purpose-built compositor for games. Its job is to accept one or more Wayland surfaces from a game (or a game and its overlays), optionally apply upscaling and colour processing, and present the result to a display with the lowest achievable latency. Every design decision follows from this mandate: gamescope does not implement a general-purpose window manager, does not have an effects pipeline for window animations, and does not attempt to be a drop-in desktop compositor.

gamescope operates in two modes. In nested mode it runs as a Wayland client inside the desktop compositor, presenting the composited game output as a single surface. This is the mode most desktop users encounter. In direct KMS mode — the default on Steam Deck — gamescope takes exclusive ownership of the DRM file descriptor, bypasses the desktop compositor entirely, and drives the display hardware directly. Direct KMS mode eliminates the latency contribution of the outer compositor and allows gamescope to control VBlank timing precisely.

The backend abstraction in `gamescope/src/Backends/` supports multiple display backends: DRM (direct KMS), SDL, Wayland (nested), and Headless (for testing). `gamescope/src/steamcompmgr.cpp` contains the main compositor loop — at over 5,000 lines it is intentionally dense, encoding Valve's accumulated knowledge of game-specific edge cases, XWayland window management quirks, and HDR metadata handling. The Vulkan renderer lives in `gamescope/src/rendervulkan.cpp`.

### The Vulkan Compositing Pipeline

gamescope's compositing is Vulkan-first. Rather than using OpenGL/GLES2, all compositing operations — layer blending, upscaling, colour space conversion — are implemented as Vulkan compute or graphics shaders. This choice was motivated by the need for precise control over GPU resources and by Vulkan's explicit synchronisation model, which dovetails naturally with gamescope's DMA-BUF import workflow.

The compositing pipeline for a typical game frame proceeds as follows. The game renders a frame via either `VK_KHR_wayland_surface` or `wl_egl_surface` and commits it to a Wayland surface. gamescope imports the committed buffer as a DMA-BUF, using `VK_EXT_image_drm_format_modifier` to import it into a Vulkan image with the exact DRM format modifier that the GPU allocated. The buffer is then passed through the configured colour transform (SDR-to-HDR tone mapping if the display is HDR-capable, or identity if already correct). If upscaling is requested, the image passes through the upscaling shader. Finally, overlays — MangoHud, the Steam overlay — are composited on top, and the result is presented to KMS via an atomic commit.

### FSR Upscaling

AMD's FidelityFX Super Resolution (FSR) algorithm is integrated directly into gamescope as a Vulkan compute shader. The `fsr_upscale.comp` compute shader implements the FSR 1.0 (EASU + RCAS) algorithm: a spatial edge-adaptive upsampling pass followed by a contrast-adaptive sharpening pass. The game renders at a lower-than-native resolution (for example, 720p on Steam Deck's 800p panel), and gamescope upscales to display resolution using FSR.

```glsl
/* Source: gamescope/src/shaders/fsr_upscale.comp
 * FSR1 EASU (Edge Adaptive Spatial Upsampling) compute shader.
 * Push constants carry the upscale ratio and sharpness parameters.
 * Input: game framebuffer at native render resolution (e.g. 720p)
 * Output: upscaled framebuffer at display resolution (e.g. 800p)
 */
layout(push_constant) uniform PushConstants {
    uvec4 const0;   /* EASU input/output dimensions */
    uvec4 const1;
    uvec4 const2;
    uvec4 const3;
    uvec4 sample;   /* RCAS sharpness */
} pc;

layout(local_size_x = 64) in;
void main() {
    uvec2 gxy = gl_GlobalInvocationID.xy;
    /* FsrEasuF reads from input image, writes to output image */
    AF3 color;
    FsrEasuF(color, gxy, pc.const0, pc.const1, pc.const2, pc.const3);
    imageStore(outputImage, ivec2(gxy), vec4(color, 1.0));
}
```

The `GAMESCOPE_FORCE_WINDOWS_FULLSCREEN` and `--fsr-upscaling` flags control upscaling activation. gamescope also supports FSR 2 (temporal upscaling, which requires motion vector input from the game), NIS (NVIDIA Image Scaling, applicable on non-AMD hardware), and integer scaling for pixel-art games.

### HDR Pipeline

gamescope's HDR support predates any standardised Wayland colour management protocol. It detects HDR-capable displays by reading the `HDR_OUTPUT_METADATA` KMS property and EDID HDRDB data from the connected display. The `GAMESCOPE_HDR_ENABLED` environment variable enables HDR mode; `GAMESCOPE_HDR_PEAK_NITS` overrides the peak luminance target.

For SDR games on an HDR display, gamescope applies a tone mapping curve in a compute shader that maps the SDR colour volume (0–100 nit reference white) into the HDR colour volume of the connected display, providing a subjectively brighter and more saturated image than simple SDR passthrough. For games that natively render in HDR (games using `VK_EXT_hdr_metadata` via DXVK or native Vulkan), gamescope passes the HDR metadata through to KMS, instructing the display hardware to operate in PQ (Perceptual Quantizer) or HLG mode.

Setting the HDR output state requires programming two KMS properties on the CRTC or connector: `HDR_OUTPUT_METADATA` (a blob containing the SMPTE ST 2086 static metadata structure) and `COLORSPACE` (selecting the BT.2020 primaries / ST 2084 PQ transfer function combination). gamescope does this through its custom DRM backend rather than via a higher-level abstraction, as described in Chapter 3.

### Direct Scanout

When the game's buffer meets all conditions for hardware plane scanout — it is at display resolution, in a format and modifier that the KMS primary plane supports, without any overlays that would require compositing — gamescope bypasses the Vulkan compositing pass entirely. The game's buffer is placed directly on the KMS plane via the `liftoff_layer` assignment (see below), achieving zero-copy presentation. The `drm_plane::can_do_direct_scanout()` check in gamescope's DRM backend encapsulates all the conditions: format compatibility, modifier support, no scaling required, no colour space mismatch.

This optimisation is critical for latency: the Vulkan compositing pass takes measurable GPU time, and on Steam Deck's mobile GPU that time is a significant fraction of the frame budget at 60 Hz. Direct scanout can eliminate 2–4 ms of GPU time per frame in the common case.

### libliftoff Integration

gamescope uses libliftoff — the lightweight KMS plane allocation library from Simon Ser — to manage hardware plane assignment for multi-layer scenes. libliftoff's API is straightforward: you create a `liftoff_device` from a DRM file descriptor, register all available planes with `liftoff_device_register_all_planes`, create a `liftoff_output` for each CRTC, and create `liftoff_layer` objects for each compositor layer (game window, overlay, cursor).

For each frame, you call `liftoff_layer_set_property` to set KMS properties on each layer — `FB_ID`, `CRTC_X`, `CRTC_Y`, `CRTC_W`, `CRTC_H`, `SRC_X`, `SRC_Y`, `SRC_W`, `SRC_H`, `ZPOS` — and then call `liftoff_output_apply` with a `drmModeAtomicReq`. libliftoff attempts to assign each layer to a hardware plane using a TEST_ONLY atomic commit; layers that cannot be assigned to hardware planes fall back to composition by the Vulkan renderer.

```c
/* Source: gamescope/src/Backends/ (illustrative, based on libliftoff API) */
struct liftoff_layer *game_layer = liftoff_layer_create(output);
struct liftoff_layer *overlay_layer = liftoff_layer_create(output);

/* Set properties for the game frame at display resolution */
liftoff_layer_set_property(game_layer, "FB_ID", game_fb_id);
liftoff_layer_set_property(game_layer, "CRTC_W", display_width);
liftoff_layer_set_property(game_layer, "CRTC_H", display_height);
liftoff_layer_set_property(game_layer, "ZPOS", 0);

/* Set properties for the overlay (MangoHud, Steam overlay) */
liftoff_layer_set_property(overlay_layer, "FB_ID", overlay_fb_id);
liftoff_layer_set_property(overlay_layer, "ZPOS", 1);

/* libliftoff performs TEST_ONLY commits to find the optimal plane
 * assignment; layers that cannot be hardware-composited fall back.
 * liftoff_output_apply signature: (output, req, flags) — 3 args */
drmModeAtomicReq *req = drmModeAtomicAlloc();
if (liftoff_output_apply(output, req, 0) < 0) {
    /* fall back to Vulkan compositing for unassigned layers */
}
drmModeAtomicCommit(drm_fd, req, DRM_MODE_ATOMIC_NONBLOCK |
                    DRM_MODE_PAGE_FLIP_EVENT, NULL);
```

The value of this approach is that gamescope can handle configurations ranging from a single full-screen game (likely to achieve full hardware direct scanout) to a game with multiple overlays (requiring plane allocation for cursor, overlay, and game layers) without hard-coding plane assignment logic.

### Steam Deck Session

On Steam Deck, gamescope runs as the session compositor via `gamescope-session`. It owns the display from session start, hosts XWayland for games using Proton/DXVK, and manages the Steam UI as a layer shell-equivalent surface. The `STEAM_DISPLAY_REFRESH_LIMITS` interface allows Steam to communicate per-game frame-rate limits to gamescope, enabling Steam Deck's "FPS limiter" feature. Performance counters — GPU busy percentage via `/sys/class/drm/card0/device/gpu_busy_percent` on AMD hardware, frame timing via `wp_presentation` from the game client — feed MangoHud's telemetry overlay.

---

## 7. Protocol Extension Landscape and Compositor Compatibility

### The Extension Stability Ladder

Wayland protocol extensions are distributed across several categories that carry different stability guarantees. The `stable` directory in wayland-protocols contains protocols with frozen APIs: `xdg-shell`, `presentation-time`, `viewporter`, `linux-dmabuf-v1` (v1 stable). Applications can depend on these unconditionally on any reasonably modern compositor. The `staging` directory contains protocols that are implemented and expected to be stable but are still in a testing phase — API may change with a version bump but breakage is rare. `wp_color_management_v1` and `wp_linux_drm_syncobj_v1` are staging as of 2025. The `unstable` directory contains deprecated protocols superseded by staging/stable equivalents; `xdg-output-unstable-v1` is superseded by `xdg-output` in staging.

Beyond the shared repository, each compositor maintains its own extensions. KDE protocols (`org_kde_kwin_blur`, `org_kde_kwin_server_decoration`) are stable within the KDE ecosystem but not available elsewhere. wlroots protocols (`zwlr_layer_shell_v1`, `zwlr_screencopy_manager_v1`) are implemented by Sway, Hyprland, and any other wlroots-based compositor, but not by Mutter or KWin (both of which have their own equivalents or alternatives). Hyprland protocols are Hyprland-only.

### Extension Support Matrix

The following table summarises protocol availability across the five compositors as of 2025–2026. "Yes" means the compositor implements the server side; "No" means it does not; "Partial" means implementation is in progress or only partially complete.

| Protocol | Mutter (GNOME 48) | KWin (Plasma 6.1) | Sway | Hyprland | gamescope |
|---|---|---|---|---|---|
| `xdg-shell` | Yes | Yes | Yes | Yes | Yes |
| `wp_presentation` | Yes | Yes | Yes | Yes | Yes |
| `linux-dmabuf-v1` | Yes | Yes | Yes | Yes | Yes |
| `wp_linux_drm_syncobj_v1` | Partial | Yes | Yes (wlroots) | Yes (wlroots) | No |
| `wp_color_management_v1` | Yes (GNOME 48) | Yes | Partial | Partial | No |
| `wp_fifo_v1` | Partial | Yes | Yes (wlroots) | Yes (wlroots) | No |
| `zwlr_layer_shell_v1` | No | No | Yes | Yes | No |
| `zwlr_screencopy_manager_v1` | No | No | Yes | Yes | Partial |
| `ext_session_lock_v1` | No | No | Yes | Yes | No |
| `xdg_decoration` | Yes | Yes | Yes | Yes | No |

### Portability Strategies

For application developers targeting multiple compositors, the safest baseline is the `stable` category. `wp_viewporter` (scaling and cropping surfaces without client-side rendering), `wp_single_pixel_buffer_v1` (creating solid-colour surfaces without allocating GPU memory), and `wp_presentation` (frame timing feedback) are stable and widely implemented. For capabilities that exist only in `staging` or compositor-specific protocols, use the `wl_registry_add_listener` pattern to detect availability at runtime:

```c
/* Source: Wayland client application, wl_registry listener pattern */
static void registry_global(void *data, struct wl_registry *registry,
                             uint32_t name, const char *interface,
                             uint32_t version) {
    struct app_state *app = data;
    if (strcmp(interface, wp_presentation_interface.name) == 0) {
        app->presentation = wl_registry_bind(registry, name,
                                             &wp_presentation_interface,
                                             MIN(version, 1));
    } else if (strcmp(interface, wp_color_manager_v1_interface.name) == 0) {
        app->color_manager = wl_registry_bind(registry, name,
                                              &wp_color_manager_v1_interface,
                                              MIN(version, 2));
    }
    /* absence of wp_color_manager_v1 is handled gracefully downstream */
}

/* After registering the listener, do a synchronous roundtrip to
 * ensure all globals are enumerated before proceeding. */
wl_display_roundtrip(display);
```

### The xdg-desktop-portal Layer

For high-level application needs — screen capture, file chooser, clipboard, inhibiting screen blanking — the xdg-desktop-portal D-Bus service provides a compositor-agnostic interface. The portal backend (the component that translates portal D-Bus calls into compositor actions) is provided by each desktop environment: `xdg-desktop-portal-gnome` for GNOME, `xdg-desktop-portal-kde` for KDE, `xdg-desktop-portal-wlr` for wlroots-based compositors. Applications using portals for screen capture do not need to know which compositor is running; the portal handles the `zwlr_screencopy_manager_v1` or `org_kde_kwin_screencasting` details internally. This is the recommended approach for any application that does not require direct compositor feature access.

---

## 8. Common Compositor Debugging Techniques

### Protocol-Level Debugging

The `WAYLAND_DEBUG=1` environment variable is the first tool in any Wayland debugging session. When set, the wayland-client library logs every protocol message sent and received, tagged with the object ID, interface name, and opcode. This is indispensable for diagnosing common issues: a client that is not acknowledging `xdg_surface.configure` events, a buffer that is being committed before the compositor has sent a `wl_buffer.release`, or a protocol sequence that is valid in one compositor but triggers an error in another. Setting `WAYLAND_DEBUG=server` on the compositor side shows the server's view of the protocol traffic.

Each compositor also exposes its own debug variables. Mutter responds to `MUTTER_DEBUG_REDRAWS=1` (marks damaged regions on screen), `MUTTER_DEBUG_PAINT=1` (logs the repaint loop), and `MUTTER_DEBUG_KMS_THREADING=1` (logs KMS thread activity). KWin's `KWIN_OPENGL_INTERFACE` selects the GL backend (`egl` or `glx`); `KWIN_DRM_USE_MODIFIERS=0` disables DRM format modifiers for debugging modifier compatibility issues. For Sway, `SWAY_DEBUG` can be set along with Sway's `--verbose` flag for detailed wlroots-level logging.

### DRM and Kernel Debugging

For issues at the KMS layer — atomic commit failures, mode setting errors, VRR not activating — the kernel's `drm.debug` module parameter is essential. Setting `drm.debug=0xff` (all bits set) in the kernel command line or via `echo 0xff > /sys/module/drm/parameters/debug` enables verbose DRM logging in `dmesg`. Atomic commit failures appear as messages of the form `[drm:drm_atomic_helper_commit_modeset_disables] FAILED to commit modeset` with the specific property or plane that caused the rejection. `LIBDRM_DEBUG=1` enables libdrm's own logging of ioctl arguments and return codes, which is useful for verifying that the compositor is submitting the expected atomic state.

The kernel's `drm` ftrace event group provides structured, low-overhead event tracing at the KMS level:

```bash
# Source: Linux kernel ftrace infrastructure
# Capture page flip events and CRTC state changes for 5 seconds
trace-cmd record -e 'drm:drm_vblank_event' \
                 -e 'drm:drm_crtc_state' \
                 -p nop -- sleep 5
trace-cmd report | grep drm_vblank_event | head -20
```

The `drm:drm_vblank_event` event records the `CLOCK_MONOTONIC` timestamp at which each VBlank interrupt was delivered to the kernel, the CRTC index, and the frame sequence number. Comparing these timestamps with compositor-level frame timestamps isolates where scheduling latency originates.

### Frame Timing Tools

`wayland-info` enumerates all globals advertised by a running compositor — useful for quickly checking which protocol extensions are available. The `wp_presentation` protocol provides per-frame timing feedback; the minimal frame timing logger pattern described in Section 9 can be used to collect production-quality latency data from any `wp_presentation`-capable compositor.

For GPU hang situations — which manifest as DRM error messages and a frozen display — the compositor typically attempts a GPU reset followed by a re-initialisation of its rendering context. On modern kernels, AMD and Intel GPUs support context-level reset without requiring a full GPU hang (TDR-style recovery); NVIDIA's open-source kernel module also supports this. If the compositor fails to recover, a session restart is required. The `journalctl -b -p err` command collects the relevant kernel and compositor error messages from the current boot.

---

## 9. Measuring Compositor Latency and Frame Pacing

### Measurement Concepts

Before reaching for tools, it is important to distinguish between two related but distinct metrics. Motion-to-photon latency is the end-to-end time from an input event — a mouse click, gamepad button press, touchscreen tap — to the corresponding change appearing on the display. This is the metric that determines perceived responsiveness; for gaming and VR applications, latencies above 20 ms are perceptible. Frame pacing, by contrast, is about consistency: delivering frames at regular intervals (every 16.67 ms at 60 Hz) rather than in irregular bursts. Uneven pacing is perceptually worse than a slightly lower mean frame rate, because the human visual system is acutely sensitive to motion stutters even when the average frame rate is acceptable.

The compositor contributes to both metrics. Its compositing pass adds GPU time between the client's `wl_surface.commit` and the KMS page flip. Its scheduling policy determines how much of the frame budget remains available to the application after the compositor's work is done. A compositor that schedules its repaint too early in the VBlank interval reduces application render time; one that schedules too late risks missing the VBlank and adding a full frame of latency.

The latency budget decomposition for a Wayland game looks like: application render time + compositor compositing time + display propagation delay (panel response, backlight modulation). The compositor contributes the middle term and can inflate the first term through scheduling decisions.

### The wp_presentation Protocol

The `wp_presentation` Wayland protocol provides the foundation for software-based frame timing measurement. A client binds the `wp_presentation` global from the registry and calls `wp_presentation.feedback(surface)` at each commit to request a feedback object for that frame. The compositor responds with one of three events on the feedback object: `presented` (the frame was actually shown on the display), `discarded` (the frame was superseded before display, for example because a newer frame arrived before the previous VBlank), or `invalid` (timing data is unavailable, for example when software rendering is active).

The `presented` event carries four fields: `tv_sec` and `tv_nsec` are the `CLOCK_MONOTONIC` timestamp of the actual hardware page flip; `refresh` is the display period in nanoseconds (one billion divided by the display's refresh rate in Hz); `seq` is a monotonically increasing hardware frame counter; and `flags` is a bitmask including `VSYNC` (frame aligned to VBlank), `HW_CLOCK` (timestamp from hardware), `HW_COMPLETION` (hardware completed the flip), and `ZERO_COPY` (the client buffer was scanned out directly without a compositing pass). The `ZERO_COPY` flag is particularly valuable for gaming diagnostics: its presence confirms that direct scanout is active and the compositor compositing pass was bypassed.

A minimal frame timing logger can be built around `wp_presentation`:

```c
/* Source: example wp_presentation client (application developer pattern)
 * This pattern records commit timestamps and correlates them with
 * presented events to compute per-frame compositor latency.
 */
struct frame_record {
    struct timespec commit_time;
    struct wp_presentation_feedback *feedback;
};

static const struct wp_presentation_feedback_listener feedback_listener = {
    .sync_output = noop,
    .presented = handle_presented,
    .discarded = handle_discarded,
};

static void handle_presented(void *data,
                              struct wp_presentation_feedback *feedback,
                              uint32_t tv_sec_hi, uint32_t tv_sec_lo,
                              uint32_t tv_nsec, uint32_t refresh,
                              uint32_t seq_hi, uint32_t seq_lo,
                              uint32_t flags) {
    struct frame_record *rec = data;
    struct timespec flip = {
        .tv_sec  = ((uint64_t)tv_sec_hi << 32) | tv_sec_lo,
        .tv_nsec = tv_nsec,
    };
    /* Compute latency: flip time minus commit time */
    double latency_ms = (flip.tv_sec - rec->commit_time.tv_sec) * 1000.0
                      + (flip.tv_nsec - rec->commit_time.tv_nsec) / 1e6;
    fprintf(stdout, "latency=%.2fms refresh=%uns flags=0x%x\n",
            latency_ms, refresh, flags);
    wp_presentation_feedback_destroy(feedback);
    free(rec);
}

/* At commit time: */
struct frame_record *rec = calloc(1, sizeof(*rec));
clock_gettime(CLOCK_MONOTONIC, &rec->commit_time);
rec->feedback = wp_presentation_feedback(app->presentation, surface);
wp_presentation_feedback_add_listener(rec->feedback,
                                       &feedback_listener, rec);
wl_surface_commit(surface);
```

Accumulating the `latency_ms` values over many frames and computing P50, P95, and P99 percentiles reveals the compositor's frame scheduling behaviour. On a healthy 60 Hz system, P50 should be around 8–12 ms (half a frame); spikes above 33 ms (two frames) indicate dropped frames.

An important caveat: the `HW_CLOCK` and `HW_COMPLETION` flags are only set when the compositor has genuine hardware page flip timestamps from the DRM page flip event. On virtual machines, with software rendering active, or on compositors that synthesise timestamps rather than reading them from the kernel's VBlank interrupt handler, the timestamps will be less precise. The measurement methodology is most reliable on bare-metal hardware with a compositing-capable GPU.

### Ground-Truth Measurement Methods

Software timestamps from `wp_presentation` capture compositor-reported latency, which excludes display propagation delay and may include compositor scheduling artefacts. For end-to-end motion-to-photon measurement, hardware-based methods provide ground truth.

Display capture cards — the Elgato 4K60 Pro, AVerMedia Live Gamer 4K, and similar devices — capture the actual display output as a video stream with hardware timestamps. By comparing the application's frame submission timestamps (from `wp_presentation` or Vulkan timestamp queries) against the capture-card timestamp when the frame becomes visible, the full end-to-end latency is measured without relying on compositor-reported times. This method requires a capture card with a known, low-latency capture pipeline and careful timestamp alignment between the capture card's clock and the host's `CLOCK_MONOTONIC`.

The high-speed camera method is a cost-effective alternative for single-point measurements. Recording the display and input device simultaneously at 240 fps or higher, then counting frames between the input event (a key press indicator or mouse button LED) and the visible change on the display, gives end-to-end latency with no software instrumentation. A standard 240 fps camera provides 4.2 ms per-frame resolution, sufficient for distinguishing 8 ms vs. 16 ms differences but not for sub-frame precision.

Valve's Steam Deck frame pacing research, published at XDC 2022, provides a detailed case study. The work identified that VBlank interrupt timing jitter of ±50 µs at the kernel level compounded across gamescope's scheduling logic to produce visible pacing artefacts at 40 Hz. The fix adjusted gamescope's present scheduling to target the midpoint of the VBlank window rather than its leading edge, reducing sensitivity to interrupt jitter. This research validated the importance of comparing kernel-level VBlank timestamps (from ftrace) against compositor-level scheduling timestamps when diagnosing pacing problems.

### Compositor Scheduling Models

Each compositor implements a different scheduling model for triggering repaints relative to VBlank.

KWin's approach uses `RenderLoop::estimatedVBlankTime()`, which predicts the next VBlank by extrapolating from the most recent hardware page flip timestamp and the display's nominal refresh period. KWin schedules repaints to complete with a configurable safety margin before the predicted VBlank, trading some application render budget for reduced risk of missing the flip. The safety margin is adaptive: if recent frames have consistently completed early, the margin shrinks; if frames are arriving close to the deadline, it grows.

Mutter's `ClutterFrameClock` tracks VBlank intervals using `CLOCK_MONOTONIC` timestamps from DRM page flip events and dispatches repaint callbacks at `vblank_time - render_time_estimate`, where `render_time_estimate` is a rolling average of recent frame render times. The estimate updates automatically as rendering load changes.

gamescope's scheduler is the simplest: a dedicated async repaint loop keyed to `POLLPRI` on the DRM file descriptor. VBlank notification triggers immediate frame composition and atomic commit submission, with minimal buffering by design. This approach minimises latency at the cost of offering less scheduling flexibility — gamescope cannot easily adapt to a varying application render time the way KWin does.

### Tools

**MangoHud** is the most accessible frame timing tool for Vulkan and OpenGL games. Setting `MANGOHUD=1` enables the overlay, which renders per-frame timing graphs showing frame time, GPU time, CPU time, and VRAM usage. The logging mode — `MANGOHUD_CONFIG=log_duration=10,output_file=/tmp/mango.log` — captures per-frame timing data to a CSV file from which P95 and P99 latency percentiles can be computed. MangoHud hooks into Vulkan and OpenGL present calls via a Vulkan layer and LD_PRELOAD mechanism, so it measures the timing of the present call from the application's perspective, below the compositor boundary. This makes it useful for measuring application-side frame production time independently of compositor scheduling.

**ftrace KMS events** provide ground-truth timestamps from the kernel's VBlank interrupt handler, bypassing all compositor and userspace timing. The `drm:drm_vblank_event` trace event records the exact `CLOCK_MONOTONIC` timestamp at which each VBlank interrupt was processed, independent of when the compositor woke up to handle it. Comparing these kernel timestamps with compositor-level repaint timestamps quantifies compositor wake-up latency — the time between the VBlank interrupt and the compositor's response to it.

```bash
# Source: Linux kernel trace-cmd usage
# Record VBlank events and CRTC state transitions for a 10-second window
trace-cmd record -e 'drm:drm_vblank_event' \
                 -e 'drm:drm_crtc_state'   \
                 -- sleep 10
# Analyse: extract VBlank timestamps and compute inter-frame intervals
trace-cmd report trace.dat | awk '
  /drm_vblank_event/ { 
    ts = $1+0; 
    if (prev > 0) printf "interval=%.3fms\n", (ts-prev)*1000; 
    prev=ts 
  }'
```

The `trace-cmd` output, combined with `wp_presentation` feedback data, can produce a full picture of compositor latency: kernel VBlank timestamp, compositor wake-up timestamp, application commit timestamp, and final presentation timestamp are all correlated.

**IGT GPU Tools** contains KMS benchmark tests that bypass the compositor entirely and measure raw atomic commit latency. The `kms_flip` and `kms_atomic_interruptible` tests schedule page flips and record the time between the commit ioctl and the kernel's flip completion callback. These measurements establish the hardware lower bound: the minimum latency any compositor can achieve on this hardware, given zero compositing overhead. Comparing IGT results with MangoHud or `wp_presentation` measurements quantifies how much overhead a specific compositor adds above that minimum.

**vkBasalt** is a Vulkan post-processing layer (`VK_LAYER_vkbasalt_post_processing`) that applies effects (CAS sharpening, SMAA, etc.) between the application's Vulkan rendering and the present call. It records present timestamps via `VK_EXT_display_control` where available, providing per-frame timing data from the Vulkan present path independently of the compositor's `wp_presentation` mechanism. When both `wp_presentation` timestamps and vkBasalt timestamps are available for the same frame sequence, the difference isolates the time between the Vulkan `vkQueuePresentKHR` call and the compositor's actual page flip — the compositor-internal scheduling delay.

---

## Integrations

**Chapter 2 (KMS and Atomic Modesetting)**: The atomic commit API described in Chapter 2 is the substrate on which every compositor in this chapter builds. Mutter's `MetaKmsUpdate` is an abstraction over `DRM_IOCTL_ATOMIC_COMMIT`; KWin's `DrmAtomicCommit` wraps the same ioctl; gamescope drives it directly through its custom DRM backend. Direct scanout — the hardware plane promotion optimisation that eliminates the GPU compositing pass — is a specific use of KMS overlay planes discussed in Chapter 2; KWin's `DrmOutput::tryDirectScanout` and gamescope's `drm_plane::can_do_direct_scanout()` implement the same underlying technique.

**Chapter 3 (Advanced Display Features)**: KWin's HDR implementation programs `HDR_OUTPUT_METADATA` and `COLORSPACE` KMS properties using the interfaces described in Chapter 3. gamescope's HDR pipeline reads EDID-reported luminance limits via KMS property introspection. The explicit sync implementation in KWin (`wp_linux_drm_syncobj_v1`, Plasma 6.1) uses DRM timeline semaphores described in Chapter 3's synchronisation section. VRR activation via `DrmOutput::setVrrPolicy` in KWin uses the `VRR_ENABLED` KMS connector property.

**Chapter 4 (GPU Memory Management and DMA-BUF)**: gamescope imports game frame buffers as DMA-BUF objects using `VK_EXT_image_drm_format_modifier`, the Vulkan extension that allows Vulkan to import a DMA-BUF with an explicit DRM format modifier. KWin's direct scanout path checks the `DrmPlane`'s supported format/modifier list against the client buffer's format modifier before attempting hardware promotion. Mutter's `MetaRendererNative` allocates GBM buffers for the EGL framebuffer using `gbm_bo_create_with_modifiers`.

**Chapter 15 (ACO: The AMD Compiler Backend)**: gamescope's FSR compute shaders (`fsr_upscale.comp`, colour transform shaders) compile through Mesa's NIR/ACO pipeline on AMD hardware. Shader compilation latency affects game startup time when gamescope first dispatches these shaders. The Vulkan pipeline cache mechanisms described in Chapter 15 apply here: gamescope should (and does) use persistent pipeline caches to amortise shader compilation across sessions.

**Chapter 18 (Vulkan Drivers)**: gamescope uses Mesa RADV (AMD) and ANV (Intel) for its Vulkan compositing pass. `VK_EXT_image_drm_format_modifier` is a required Vulkan extension for gamescope's DMA-BUF import workflow; this extension is implemented in both RADV and ANV. The `VK_KHR_timeline_semaphore` extension is used for synchronisation between the Vulkan compositing pass and KMS scanout in the non-direct-scanout case.

**Chapter 20 (Wayland Fundamentals)**: All five compositors implement the server side of the protocols introduced in Chapter 20. The `wp_presentation` protocol's `presented` event structure — `tv_sec`, `tv_nsec`, `refresh`, `seq`, `flags` — is the core data source for the latency measurement methodology in Section 9. The `wp_linux_drm_syncobj_v1` protocol described in Chapter 20's synchronisation section was first shipped in a desktop compositor by KWin for Plasma 6.1. The extension support matrix in Section 7 completes the picture of which protocols are safe for application developers to target.

**Chapter 21 (wlroots)**: Sway and Hyprland both build on wlroots; gamescope, Mutter, and KWin do not. This distinction means that wlroots protocol additions (new staging protocol implementations, renderer improvements) flow automatically to Sway and Hyprland on a version update, while Mutter and KWin must implement each protocol independently. The trade-off is that Mutter and KWin can optimise their entire display path — Mutter's KMS thread, KWin's direct scanout logic — in ways that are not possible within wlroots' more general abstraction.

**Chapter 23 (XWayland and Legacy Application Compatibility)**: All five compositors run XWayland. gamescope is the deployment target for Proton on Steam Deck; it hosts XWayland for DXVK/VKD3D-Proton games. Mutter's `MetaXWaylandManager` and KWin's XWayland integration are the primary paths for X11 applications on modern GNOME and Plasma desktops. The override-redirect handling discussed in the Mutter section is a recurring source of compatibility issues between X11 application expectations and Wayland compositor behaviour.

**Chapter 26 (Hardware Video Acceleration)**: The screencopy protocol implementations in each compositor — `zwlr_screencopy_manager_v1` in Sway and Hyprland, compositor-specific interfaces in KWin and Mutter — are the source of the PipeWire video capture streams used by screen recorders and video conferencing applications. gamescope's DMA-BUF export path provides compositor frames to recording software on Steam Deck.

**Chapter 28 (Windows Compatibility via Proton)**: gamescope is the production deployment environment for Proton on Steam Deck. DXVK (Direct3D 9/10/11 on Vulkan) and VKD3D-Proton (Direct3D 12 on Vulkan) games run as gamescope Wayland clients. gamescope's FSR upscaling activates for games that do not natively render at the display's native resolution, which is the common case for older titles that have fixed render resolution settings.

**Chapter 29 (Upscaling Algorithms)**: gamescope FSR 1 and FSR 2 integration is covered in depth in Section 6 of this chapter. The FSR compute shader pipeline — EASU upsampling pass followed by RCAS sharpening — is the implementation of the algorithm described in Chapter 29. vkBasalt operates below the compositor boundary as a Vulkan layer; its CAS sharpening pass and the frame timing instrumentation described in Section 9 are complementary tools to the upscaling pipeline.

**Chapter 30 (GPU Performance Profiling)**: The latency measurement methodology in Section 9 — MangoHud CSV logging, ftrace `drm:drm_vblank_event` capture, `wp_presentation` frame timing loggers, and IGT KMS benchmarks — feeds into the broader GPU performance profiling workflow in Chapter 30. The pattern of comparing IGT lower-bound measurements with application-level measurements to quantify compositor overhead is directly applicable to any GPU performance investigation that involves a compositor in the display path.

---

## References

1. [Mutter source — GNOME/mutter](https://gitlab.gnome.org/GNOME/mutter) — primary source for MetaKms, MetaBackendNative, Clutter scene graph, and wp_color_management_v1 server implementation; `src/backends/native/` and `src/compositor/`
2. [KWin source — KDE/kwin](https://invent.kde.org/plasma/kwin) — primary source for DrmBackend, SceneOpenGL, effect system, HDR pipeline, and explicit sync; `src/backends/drm/` and `src/scene/`
3. [Sway source — swaywm/sway](https://github.com/swaywm/sway) — primary source for IPC protocol, container tree, and wlroots integration
4. [Hyprland source — hyprwm/Hyprland](https://github.com/hyprwm/Hyprland) — primary source for CHyprAnimationManager, CRenderer, plugin API, and Hyprland-specific protocols
5. [gamescope source — ValveSoftware/gamescope](https://github.com/ValveSoftware/gamescope) — primary source; `src/steamcompmgr.cpp` for the compositor loop, `src/rendervulkan.cpp` for Vulkan compositing, `src/Backends/` for DRM/Wayland/SDL backends
6. [libliftoff — emersion/libliftoff](https://gitlab.freedesktop.org/emersion/libliftoff) — KMS plane allocation library used by gamescope; `include/libliftoff.h` for the full API surface (the GitHub mirror is archived; the canonical repository is on freedesktop.org GitLab)
7. [XDC 2023 — KWin HDR (Xaver Hugl)](https://indico.freedesktop.org/event/4/contributions/107/) — detailed walkthrough of KWin's HDR implementation and tone-mapping shader design
8. [XDC 2022 — gamescope and direct KMS (Joshua Ashton / Pierre-Loup Griffais)](https://indico.freedesktop.org/event/2/contributions/62/) — gamescope architecture, direct KMS mode, and VBlank jitter analysis
9. [LWN: "The KWin compositor gets HDR support" (2023)](https://lwn.net/Articles/939063/) — accessible summary of Plasma 6.0 HDR landing
10. [LWN: "Gamescope: an exploration" (2022)](https://lwn.net/Articles/908613/) — architectural overview of gamescope's design goals and compositor loop
11. [Valve Steam Deck HDR blog post](https://store.steampowered.com/news/app/1675200/view/3698022584126307632) — Valve's description of Steam Deck HDR implementation and tuning
12. [KWin Effect API — effecthandler.h](https://invent.kde.org/plasma/kwin/-/blob/master/src/effect/effecthandler.h) — canonical reference for the KWin effect plugin API
13. [Hyprland plugin documentation](https://wiki.hyprland.org/Plugins/Development/) — stable plugin API reference; HyprlandAPI header and lifecycle macros
14. [FOSDEM 2024 — Colour management on Linux (Sebastian Wick)](https://fosdem.org/2024/schedule/event/fosdem-2024-2588-color-management-on-linux/) — history and current status of wp_color_management_v1 standardisation
15. [LWN: "Explicit synchronisation in Wayland compositors" (2024)](https://lwn.net/Articles/959765/) — technical background on wp_linux_drm_syncobj_v1 and KWin's implementation
16. [wp_presentation Wayland protocol specification](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/stable/presentation-time/presentation-time.xml) — normative specification for the presentation feedback protocol
17. [MangoHud documentation and source](https://github.com/flightlessmango/MangoHud) — logging mode, config reference, and Vulkan layer implementation
18. [Valve Steam Deck frame pacing post-mortem (Pierre-Loup Griffais, XDC 2022)](https://indico.freedesktop.org/event/2/) — VBlank jitter analysis and gamescope scheduling fix methodology
19. [IGT GPU Tools — kms_flip and kms_atomic tests](https://gitlab.freedesktop.org/drm/igt-gpu-tools) — atomic commit latency benchmarks that establish the compositor-bypassed performance baseline
20. [Linux kernel ftrace documentation — DRM events](https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html) — DRM uAPI reference including KMS property namespaces; see also `Documentation/trace/events.rst` for ftrace event infrastructure
21. [vkBasalt source and configuration](https://github.com/DadSchoorse/vkBasalt) — Vulkan post-processing layer with frame timing instrumentation via VK_EXT_display_control
22. [wp_color_management_v1 protocol — Wayland Explorer](https://wayland.app/protocols/color-management-v1) — current staging status (version 2) and interface documentation
23. [Mutter KMS abstractions documentation](https://github.com/GNOME/mutter/blob/main/doc/mutter-kms-abstractions.md) — official Mutter documentation on MetaKms, MetaKmsDevice, MetaKmsUpdate design
24. [Phoronix: GNOME Mutter Switches To High Priority KMS Thread](https://www.phoronix.com/news/GNOME-High-Priority-KMS-Thread) — coverage of the 2024 KMS thread priority change from SCHED_RR to high priority
25. [Phoronix: KDE's KWin Merges Wayland Explicit Sync Support](https://www.phoronix.com/news/KDE-KWin-Lands-Explicit-Sync) — coverage of wp_linux_drm_syncobj_v1 landing in KWin for Plasma 6.1

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
