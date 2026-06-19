# Chapter 99: Automotive and Embedded Linux Graphics

> **Part**: Part XXI — Platform, Legacy, and History
> **Audience**: Embedded Linux engineers, automotive software developers, IoT/kiosk developers, SoC bring-up engineers
> **Status**: First draft — 2026-06-19

## Table of Contents

- [Overview](#overview)
- [1. The Automotive and Embedded Graphics Landscape](#1-the-automotive-and-embedded-graphics-landscape)
- [2. Automotive Grade Linux (AGL)](#2-automotive-grade-linux-agl)
- [3. GENIVI/COVESA and the Wayland Transition](#3-genivicovesa-and-the-wayland-transition)
- [4. Weston in Embedded Mode](#4-weston-in-embedded-mode)
- [5. cage: Single-Application Kiosk Compositor](#5-cage-single-application-kiosk-compositor)
- [6. Qt for Automotive](#6-qt-for-automotive)
- [7. Yocto and Buildroot Integration](#7-yocto-and-buildroot-integration)
- [8. GPU Driver Bring-Up on New SoCs](#8-gpu-driver-bring-up-on-new-socs)
- [9. Display Panel Bring-Up](#9-display-panel-bring-up)
- [10. Remote Rendering in Automotive Cabin Networks](#10-remote-rendering-in-automotive-cabin-networks)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

This chapter targets four overlapping audiences: embedded Linux engineers who port and maintain BSPs (Board Support Packages) for automotive-grade SoCs; automotive software developers building IVI (In-Vehicle Infotainment), digital instrument clusters, and ADAS HMI; IoT/kiosk developers deploying locked-down single-application displays in retail, industrial, and signage contexts; and SoC bring-up engineers who must get a GPU and display pipeline running on new silicon with nothing more than a device-tree binding and a datasheet.

The Linux graphics stack on embedded and automotive platforms is simultaneously simpler and more constrained than on the desktop. There is often a single display, a fixed resolution, no window manager in the traditional sense, and a hard real-time rendering budget dictated by display refresh. At the same time, the regulatory context is uniquely demanding: ISO 26262 functional safety governs HMI in safety-critical path (ASIL A through ASIL D), while ASPICE (Automotive SPICE) frames the software development process. GPU drivers must either achieve or be partitioned away from those safety integrity levels.

This chapter traces the complete embedded graphics path: from the SoC GPU and its Linux DRM driver through the Wayland compositor (AGL's agl-compositor, standalone Weston, or the cage kiosk compositor) to the application toolkit (Qt Automotive Suite), the build system integration (Yocto, Buildroot), and the networking layer that ships rendered frames to secondary displays across automotive cabin Ethernet.

---

## 1. The Automotive and Embedded Graphics Landscape

### Use Cases

Automotive and embedded Linux graphics covers a diverse set of display contexts:

- **IVI (In-Vehicle Infotainment)**: The head unit — navigation, media, phone mirroring (Android Auto, CarPlay), app ecosystem. Requires Wayland compositor with multi-application support, typically running at 1920×720 or higher on a wide-format centre display.
- **Digital Instrument Cluster**: Speedometer, fuel gauge, ADAS status readout behind the steering wheel. Real-time rendering at 60 Hz (sometimes 120 Hz on high-end platforms), with strict latency requirements for warning indicators. ISO 26262 ASIL-B or higher applies to warning lamp rendering.
- **ADAS HMI**: Visualisation of sensor fusion results — detected objects, lane overlays, parking-assist camera grids — on a separate display or in the instrument cluster.
- **HUD (Head-Up Display)**: Rendered to a projector that reflects off the windscreen. Extremely latency-sensitive; typically driven by a dedicated safety-partitioned SoC.
- **Industrial HMI and Kiosk**: Retail point-of-sale, factory floor panels, ATMs, digital signage. Single-application display, must never expose a desktop shell, and must restart automatically on application crash.

### The Shift from Android Automotive to Linux IVI

Android Automotive OS (AAOS) captured much of the automotive HMI market in the 2019–2023 timeframe, but its GPL licensing constraints, proprietary Google Mobile Services dependency, and Android's inherent app-sandbox overhead have pushed several OEMs (particularly Japanese and European) toward Linux-native stacks. The Linux Foundation's Automotive Grade Linux project and the COVESA (formerly GENIVI) ecosystem provide the alternative. As of 2026, Toyota, Subaru, and Suzuki ship Linux IVI in production, and AGL's SoDeV platform [announced in 2025](https://www.linuxfoundation.org/press/agl_sodev) positions AGL as the software-defined vehicle (SDV) foundation across infotainment, cluster, and telematics ECUs.

### Regulatory Context

**ISO 26262:2018** defines functional safety for electrical and electronic systems in road vehicles. It defines Automotive Safety Integrity Levels (ASIL A–D, with QM for non-safety) based on severity, exposure, and controllability of potential hazards. A rendering engine that displays a safety-critical warning (e.g. "BRAKE FAILURE") must be qualified at the appropriate ASIL level or be partitioned into a separate process/CPU/display controller that is independently certified.

**ASPICE (Automotive SPICE)** is a process capability model that governs software development process maturity, analogous to CMMI for automotive. It does not directly constrain runtime behaviour but shapes the traceability and testing rigour required for the software development lifecycle.

In practice, the common architectural response is **mixed-criticality partitioning**: the safety-critical rendering path (instrument cluster warnings, speed readout) runs in an ASIL-B or ASIL-D qualified process — either Qt Safe Renderer or a proprietary safety-renderer — while the rich infotainment HMI (navigation, media) runs in a QM-rated Linux process. Communication between the two is constrained to formal interfaces (shared memory with explicit ownership transfer, or SOME/IP services).

### SoC Landscape

The automotive SoC market as of 2026 is dominated by a handful of platforms, each with a distinct GPU driver story:

| SoC | GPU | Linux DRM Driver | Target |
|-----|-----|-----------------|--------|
| NXP i.MX8M | Vivante GC7000L | `etnaviv` (mainline) | IVI, industrial |
| NXP i.MX8QXP | Vivante GC7000 | `etnaviv` (mainline) | IVI |
| Renesas R-Car H3/M3 (Gen3) | Imagination PowerVR GX6650 | Proprietary DDK + `rcar-du` display | IVI, cluster |
| Renesas RZ/G3L | ARM Mali-G31 | `panfrost` (mainline) | Cluster |
| Qualcomm SA8155P | Adreno 640 | `msm` (mainline) | IVI (Snapdragon Ride) |
| Qualcomm SA8295P | Adreno 695 | `msm` (mainline) | IVI + ADAS |
| Texas Instruments TDA4VM / J721E | PowerVR Rogue GE8430 | Proprietary + `tidss` display | ADAS |

**NXP i.MX8** uses Vivante GPU IP, covered by the fully open `etnaviv` kernel driver and Mesa Gallium3D backend. The `imx-drm` display driver handles the LCDIF/DCSS scanout engine. This makes i.MX8 the most Linux-upstream-friendly automotive GPU platform [Source](https://pengutronix.de/en/blog/2021-02-26-showcase-i-mx8mp.html).

**Renesas R-Car** Gen3 (H3, M3) pairs the open `rcar-du` DRM display driver with a proprietary PowerVR userspace DDK; as of 2026 no open-source 3D driver exists for the GX6650. The `rcar-du` driver handles KMS modesetting via atomic commits. Display bring-up is fully mainline; 3D acceleration requires the Imagination Binary DDK [Source](https://cgit.freedesktop.org/~airlied/linux/commit/?id=4bf8e1962f91eed5dbee168d2348983dda0a518f).

**Qualcomm SA8295P** ships the Adreno 695 GPU, which uses the mainline `msm` DRM driver (`drivers/gpu/drm/msm/`). Qualcomm has steadily upstreamed Adreno support; Adreno 800-series patches arrived in the kernel in late 2025 [Source](https://www.phoronix.com/news/Linux-Kernel-Adreno-800-Patches). Mesa's `turnip` Vulkan driver and `freedreno` OpenGL driver cover Adreno 6xx.

**TI TDA4VM** carries a PowerVR Rogue 8XE GE8430 GPU providing ~100 GFLOPS for cluster and mirror-replacement rendering. The display subsystem (`tidss`) is mainline; GPU acceleration relies on TI's SDK with the PowerVR DDK. The TDA4VM targets ASIL-B/C applications and pairs its Cortex-A72 Linux environment with Cortex-R5F real-time cores for safety processing.

---

## 2. Automotive Grade Linux (AGL)

### AGL UCB Architecture

Automotive Grade Linux (AGL) is a Linux Foundation collaborative project producing the **Unified Code Base (UCB)**, a Yocto-based embedded Linux distribution for automotive platforms [Source](https://www.linuxfoundation.org/press/press-release/agl_ucb9). The most recent UCB release as of mid-2026 — "Ultimate Unagi" — forms the software basis of the **SoDeV** (Software Defined Vehicle) reference platform announced at the end of 2025.

AGL UCB is a Yocto Project-based distro composed of layered metadata:

```
meta-agl           — core AGL distro layer (DISTRO = "agl")
meta-agl-demo      — demo applications and reference HMI
meta-agl-distro    — machine-specific BSP integration shims
meta-openembedded  — broad package collection (Python, Qt, etc.)
poky               — Yocto reference distro (bitbake, devtool, etc.)
```

To build an AGL image for the Raspberry Pi 4 (used for developer testing):

```bash
source meta-agl/scripts/aglsetup.sh -m raspberrypi4 -b build agl-demo-platform
bitbake agl-demo-platform
```

### AFB: Application Framework Binder

AGL's microservice architecture is glued together by the **Application Framework Binder (AFB)** — a JSON-RPC over WebSocket service mesh that allows in-vehicle services (media, navigation, HVAC control) to expose typed APIs consumed by the HMI layer. Each service is an `afb-binding` shared object loaded by the `afb-daemon` process:

```c
// minimal afb-binding stub
#include <afb/afb-binding.h>

static void my_verb(afb_req_t req) {
    afb_req_reply(req, json_object_new_string("pong"), NULL, NULL);
}

static const afb_verb_t verbs[] = {
    { .verb = "ping", .callback = my_verb },
    { 0 }
};

const afb_binding_t afbBindingV4 = {
    .api   = "myservice",
    .verbs = verbs,
};
```

The HMI layer (a Qt Quick or HTML5 application) calls `myservice/ping` over the local WebSocket endpoint. This decoupling means the HMI can be restarted independently of the service backend.

### agl-compositor

The AGL Wayland compositor — **agl-compositor** — is built on **libweston** and **libweston-desktop**, not on wlroots. Collabora, the primary author, explicitly chose libweston to leverage Weston's existing DRM/KMS backend, EGL rendering path, and input handling, while adding AGL-specific private Wayland protocol extensions on top [Source](https://www.collabora.com/news-and-blog/news-and-events/a-libweston-based-compositor-for-automotive-grade-linux.html).

agl-compositor implements two private protocol extensions:

- `agl_shell` — assigns surface roles: `set_panel` (persistent taskbar), `set_background`, `activate_app`. Applications call `agl_shell_activate_app(shell, app_id, output)` to bring a specific application to foreground on a given output.
- `agl_shell_desktop` — manages split-screen / floating layouts for the infotainment zone.

A **policy engine** plugin (loaded as a shared object) implements role-based arbitration: which apps may display on which outputs, and under what conditions. This is important in automotive contexts where the navigation app must be able to preempt the media player on the main display during turn-by-turn announcements.

```
agl-compositor
├── compositor.c        — libweston compositor lifecycle
├── ivi-compositor.h    — agl_shell/agl_shell_desktop state
├── policy.c            — default allow-all policy
├── policy-rba.c        — COVESA Role-Based Arbitration plugin
└── protocols/
    ├── agl-shell.xml
    └── agl-shell-desktop.xml
```

### Waltham: Remote Output for Rear-Seat Displays

AGL uses **Waltham** — a network serialisation layer for Wayland protocol messages over TCP — to forward surface content from the head unit SoC to secondary displays (rear-seat entertainment screens, HUD projectors). agl-compositor loads `remoting-plugin` from libweston, which captures DMA-BUF frames from the GPU and dispatches them via Waltham. Input events flow back over the same TCP connection.

### AGL in Production

- **Toyota** ships AGL UCB in the Prius, Camry, and other models for the head unit.
- **Subaru** adopted AGL for its SUBARU STARLINK multimedia system.
- **Suzuki** deployed AGL on Maruti Suzuki models for the Indian market.

---

## 3. GENIVI/COVESA and the Wayland Transition

### GENIVI History and IVI Layer Manager

**GENIVI Alliance** was founded in 2009 by BMW, GM, Intel, Magneti Marelli, PSA, Visteon, and Wind River to create a common automotive IVI software platform. Its key graphics contribution was the **IVI Layer Manager (ILM)** — a surface compositing service running above X11 or (later) Wayland that gave automotive application managers explicit control over layer z-order, opacity, position, and scaling. GENIVI renamed itself **COVESA** (Connected Vehicle Systems Alliance) in 2021; the wayland-ivi-extension repository migrated to [COVESA/wayland-ivi-extension](https://github.com/COVESA/wayland-ivi-extension).

The ILM API defined surface management in terms of opaque integer IDs:

```c
#include <ilm/ilm_control.h>

// Create a 1920×720 display layer with ID 1000
t_ilm_layer layer;
ilm_layerCreateWithDimension(&layer, 1920, 720);

// Set the destination rectangle of a surface on screen
ilm_surfaceSetDestinationRectangle(
    surface_id,   /* t_ilm_surface */
    x, y,         /* pixel offset */
    width, height /* pixel extent */
);
ilm_commitChanges();
```

The `LayerManagerService` daemon brokered these requests from multiple client processes and issued the final composite commands to the display hardware. On early GENIVI platforms this sat above X11; on later platforms it sat above a Weston instance configured with the `ivi-shell` plugin.

### The ivi-shell Bridge

Weston shipped an `ivi-shell` plugin implementing the `ivi-application` and `ivi-layout` Wayland protocols. Applications called `ivi_application_surface_create(ivi_application, ivi_id, wl_surface)` to register their Wayland surface with the IVI controller. The ILM API on the controller side then mapped IVI IDs to surfaces:

```c
// From the LayerManagerControl example (COVESA SDK)
ilmErrorTypes ret = ilm_init();
ilm_surfaceSetDestinationRectangle(42 /* ivi_id */, 0, 0, 1920, 720);
ilm_layerAddSurface(1000 /* layer_id */, 42);
ilm_displaySetRenderOrder(0 /* display_id */, &layer, 1);
ilm_commitChanges();
```

### Why Wayland Won in Automotive

The industry's migration from ILM to native Wayland compositors (agl-compositor, proprietary OEM compositors) was driven by several converging factors:

1. **Security model**: Wayland clients cannot read each other's buffers or inject input events — a hard requirement for automotive systems where the navigation app and the payment-terminal app share a display.
2. **DRM/KMS integration**: Wayland compositors talk directly to the KMS atomic API for hardware plane assignment, VSync synchronisation, and HDCP. ILM abstracted this poorly.
3. **Upstream ecosystem**: Upstream Mesa, wlroots, and GTK/Qt all target Wayland natively. Maintaining X11 BSP patches became unsustainable for embedded teams.
4. **GPU-direct composition**: DMA-BUF sharing between application and compositor eliminates CPU copies for zero-copy surface compositing — critical for automotive bandwidth budgets.

The `wayland-ivi-extension` `agl_shell` approach is now the accepted bridge: it preserves the ILM-style explicit surface lifecycle control (which automotive application frameworks require) while expressing it as native Wayland protocol extensions rather than a parallel compositor API.

---

## 4. Weston in Embedded Mode

### DRM Backend Configuration

Weston, the reference Wayland compositor, is widely deployed on embedded Linux systems where the full weight of a desktop compositor (GNOME Shell, KDE Plasma) is unnecessary. Configuration is via `weston.ini`:

```ini
[core]
backend=drm-backend.so
idle-time=0
require-input=false

[output]
name=HDMI-A-1
mode=1920x1080
transform=normal
scale=1

[output]
name=DSI-1
mode=1280x800
transform=rotate-90
```

`idle-time=0` disables the idle timeout — essential for kiosk and always-on industrial displays. `require-input=false` allows Weston to start with no input devices, which is common on headless industrial panels driven by remote touch overlays.

### EGL/GLES2 Rendering Pipeline

Weston's DRM backend allocates GBM (Generic Buffer Manager) surfaces via `gbm_surface_create()`, acquires EGL window surfaces via `eglCreateWindowSurface()`, and renders compositor outputs using GLES2 shaders. The primary plane carries the compositor's texture-mapped output; overlay planes (hardware planes exposed by the KMS CRTC) are assigned to video surfaces via the `drm-backend`'s direct-scanout path.

```
Application (Wayland client)
  → wl_surface attach (DMA-BUF)
  → Weston DRM backend
      → Attempt KMS direct scanout (overlay plane)
      → Fallback: GLES2 composite to primary GBM surface
  → KMS atomic commit
  → Display output
```

### IVI Shell Module

Weston ships `ivi-shell.so`, loaded via `weston.ini`:

```ini
[core]
shell=ivi-shell.so

[ivi-shell]
ivi-input-module=ivi-input-controller.so
```

With `ivi-shell` active, Weston implements the `ivi_application` and `ivi_layout` protocols from `wayland-ivi-extension`, enabling the ILM-compatible surface management described in §3. This is the deployment model for COVESA-based IVI systems that have not yet migrated to a native compositor like agl-compositor.

### Headless Backend for CI

Embedded BSP developers routinely need to run Weston-based CI pipelines without display hardware. Weston's headless backend renders to an off-screen buffer:

```bash
weston --backend=headless-backend.so \
       --width=1920 --height=1080 \
       --no-config
```

Combined with `weston-screenshooter` and a Wayland client test harness (wlcs, or a custom Rust test suite), this enables screenshot-based regression testing of UI rendering on the same Yocto image that ships to hardware.

### Panel Plugin and Taskbar

For development/debug builds, Weston's `weston-terminal` provides a shell terminal on the embedded target, and the `panel` plugin provides a minimal taskbar. Production builds strip these; the shell is set to `ivi-shell.so` or `kiosk-shell.so` (a minimalist shell in recent Weston versions that maximises a single client, analogous to `cage`).

---

## 5. cage: Single-Application Kiosk Compositor

### Architecture

**cage** ([https://github.com/cage-kiosk/cage](https://github.com/cage-kiosk/cage)) is a minimal Wayland compositor specifically designed for kiosk deployments. It is built on **wlroots** (version 0.20 as of its 0.3.0 release in April 2026) [Source](https://github.com/cage-kiosk/cage), inheriting wlroots' DRM/KMS backend, EGL/GLES2 output rendering, libinput input handling, and xdg-shell implementation.

cage's design principle is extreme simplicity: it launches a single application, maps that application's `xdg_toplevel` surface fullscreen to the output, and prevents any window management interaction. Dialog surfaces (popup windows, menus) are centred on screen but cannot be resized or moved. There is no taskbar, no workspace switcher, and no way for the user to escape to a desktop.

```
cage
├── main.c       — argument parsing, WL display init
├── output.c     — wlr_output configuration, DRM/KMS backend
├── seat.c       — wlr_seat, libinput passthrough
├── server.c     — wlr_compositor, xdg-shell handler
└── view.c       — single wlr_xdg_toplevel, fullscreen constraint
```

### Basic Usage

```bash
# Launch cage with the kiosk application
cage /usr/bin/my_infotainment_app

# With X11 support for legacy applications (if wlroots built with XWayland)
cage -s -- /usr/bin/my_legacy_x11_app
```

The `-s` flag enables XWayland mode. When launched at a Linux VT (virtual terminal), cage drops into the DRM/KMS backend directly, taking KMS ownership of the display without an existing Wayland compositor.

### On-Screen Keyboard Integration

For touchscreen kiosk deployments without a physical keyboard, cage integrates with virtual keyboard implementations via the `zwp_virtual_keyboard_v1` and `zwp_input_method_v2` Wayland protocols (both from `wayland-protocols`). Common OSK choices for embedded Linux:

- **squeekboard** — GNOME Mobile on-screen keyboard, packaged in Yocto via `meta-squeekboard`
- **wvkbd** — lightweight wlroots-native OSK, minimal dependencies

Since cage uses wlroots' seat implementation, any OSK that speaks standard input-method protocols works without modification.

### Production Deployment with systemd

A cage-based kiosk service with automatic restart:

```ini
# /etc/systemd/system/kiosk.service
[Unit]
Description=Kiosk Application
After=systemd-udevd.service
Requires=systemd-udevd.service

[Service]
User=kiosk
Environment=XDG_RUNTIME_DIR=/run/user/1000
ExecStart=/usr/bin/cage /usr/bin/my_kiosk_app
Restart=always
RestartSec=2

[Install]
WantedBy=graphical.target
```

`Restart=always` ensures that if the kiosk application crashes (OOM, assertion failure, GPU reset), systemd relaunches cage and the application automatically. `RestartSec=2` provides a brief cooling-off period to avoid hard restart loops on persistent crashes.

### Use Cases

- **Digital menu boards**: Restaurant chains deploy cage-wrapped Chromium (`cage -- chromium-browser --kiosk https://menu.example.com`) on i.MX8 or Raspberry Pi CM4 hardware.
- **ATMs and payment terminals**: cage enforces that the payment application is the only visible surface, preventing UI redressing attacks.
- **Industrial HMI terminals**: Manufacturing floor displays running SCADA visualisation tools.
- **Automotive infotainment** (simple configurations): Where the IVI app is a single Wayland client rather than a multi-process compositor-managed stack.

---

## 6. Qt for Automotive

### Qt Interface Framework (formerly Qt IVI)

Qt's automotive portfolio was reorganised in Qt 6: **Qt IVI** (In-Vehicle Infotainment) was renamed **Qt Interface Framework**. It provides a code-generation tool (`ivigenerator`) that takes IDL-format service descriptions and generates Qt/QML bindings with backend plugin stubs. This abstracts the gap between the HMI (QML) and the underlying middleware (AFB, SOME/IP services, D-Bus):

```
ivigenerator climate.qface → 
  ClimateControl.h/.cpp   (feature API, QML-exposed)
  ClimateControlBackend.h (backend plugin interface)
  
Application code:
  ClimateControl { id: climate }
  Text { text: "Temp: " + climate.targetTemperature }
```

### QtApplicationManager

**QtApplicationManager** provides a multi-process application management layer for Qt-based IVI systems. In multi-process mode, each application runs as a separate Wayland client whose window is a `ApplicationManagerWindow` (an xdg-toplevel surface managed by the application manager's built-in Wayland compositor). The application manager enforces resource quotas, lifecycle management, and inter-application communication via D-Bus or QtInterfaceFramework.

```qml
// Launcher QML: request navigation app
ApplicationManager.startApplication("com.example.navigation")

// Inside navigation app:
ApplicationManagerWindow {
    id: root
    width: 1920; height: 720
    // ... QML UI
}
```

In single-process mode (for memory-constrained SoCs), all applications share one Qt event loop and one Wayland connection — the compositor becomes a QML item, and inter-app isolation is only by convention.

### QSGRenderNode for Cluster GPU Rendering

Digital instrument clusters render highly dynamic content (speedometer needle sweeping at 60 Hz, tachometer, fuel gauge animations). Qt Quick's scene graph runs on the GPU via either OpenGL ES or Vulkan (via Qt RHI). For custom GPU effects — per-pixel speedometer rendering, particle effects for turn indicators — `QSGRenderNode` allows injection of raw GPU commands into the scene graph render pass:

```cpp
class NeedleNode : public QSGRenderNode {
    void render(const RenderState *state) override {
        // Issue OpenGL ES or Vulkan commands directly
        // Receives the current projection matrix from state->projectionMatrix()
        renderNeedleGeometry(state->projectionMatrix());
    }
    RenderingFlags flags() const override {
        return BoundedRectRendering | DepthAwareRendering;
    }
};
```

### Qt Safe Renderer and Functional Safety

**Qt Safe Renderer** is Qt's ASIL-certified rendering component, qualified up to **ASIL D** under ISO 26262 [Source](https://www.qt.io/product/functional-safety-and-qt). It runs as an independent process with its own direct rendering path — bypassing the QML engine, V8 JavaScript engine, and standard Qt Quick scene graph — allowing safety-critical visual elements (speed digit, warning lamps, gear indicator) to continue rendering correctly even if the main infotainment Qt process crashes or hangs.

Architecture:

```
[Main Qt Quick process]          [Qt Safe Renderer process]
  QML engine                       Safe layout engine (MISRA C++)
  Navigation HMI                   Warning lamp bitmaps
  Media player UI                  Speed digit (7-segment glyphs)
        |                                   |
        | Wayland (agl-compositor)          | Direct KMS/DRM plane
        |                                   |   (hardware overlay)
        +------- Display output +-----------+
```

The Safe Renderer process holds a dedicated KMS overlay plane (or drives a separate display controller entirely on dual-display clusters) and updates it via direct DRM IOCTL calls, immune to the QM-rated main process state. The safe rendering data (glyph bitmaps, layout coordinates) is authored in Qt Safe Designer and compiled into a binary format consumed by the runtime.

The runtime library follows **MISRA C++ 2008** and is developed under a full ISO 26262 development process with independent tool qualification, failure mode analysis (FMEA), and safety case documentation.

### Qt Automotive Suite

Qt Automotive Suite (commercial licence) bundles QtApplicationManager, Qt Interface Framework, Qt Safe Renderer, a Neptune 3 reference UI, and pre-integrated BSP layers for the Renesas R-Car and NXP i.MX8 platforms. The suite is the fastest path to a production-quality AGL/Linux IVI system for teams without dedicated platform engineering resources [Source](https://www.qt.io/qt-automotive-suite/).

---

## 7. Yocto and Buildroot Integration

### Yocto Layer Structure for Automotive Graphics

A typical automotive Yocto project adds the following layers on top of `poky`:

```
poky/
meta-openembedded/
  meta-oe/                  — wayland, weston, libdrm, mesa
  meta-python/              — Python3, used by devtools
meta-qt6/                   — Qt6 and Qt Interface Framework
meta-agl/                   — AGL UCB distro
meta-freescale/             — NXP i.MX8 BSP (if using NXP)
meta-renesas/               — Renesas R-Car BSP (if using Renesas)
meta-mylayer/               — project-specific customisation
```

### IMAGE_INSTALL and DISTRO_FEATURES

Core graphics packages for a Wayland-based embedded image:

```bitbake
# In local.conf or distro config
DISTRO_FEATURES:append = " wayland opengl"

# In the image recipe
IMAGE_INSTALL:append = " \
    wayland \
    weston \
    wayland-protocols \
    libdrm \
    mesa \
    mesa-megadriver \
    libgbm \
    libinput \
    xkeyboard-config \
"
```

For GPU firmware packaging (required before the GPU driver can operate):

```bitbake
# NXP i.MX8 — Vivante GPU firmware is included in the BSP layer
# meta-freescale provides: firmware-imx (includes GPU microcode)
IMAGE_INSTALL:append = " firmware-imx"

# ARM Mali (e.g. RZ/G3L)
IMAGE_INSTALL:append = " kernel-module-mali mali-userspace"
# meta-renesas or meta-arm provides the Mali firmware package
```

### Cross-Compilation with devtool

Iterating on Mesa's etnaviv driver without a full bitbake rebuild:

```bash
# Set up an SDK sysroot for the i.MX8 target machine
bitbake -c populate_sdk core-image-minimal
. build/tmp/deploy/sdk/poky-glibc-x86_64-core-image-minimal-cortexa53-imx8mqevk-toolchain-*.sh

# Use devtool to modify and rebuild Mesa against the target sysroot
devtool modify mesa
# ... edit drivers/etnaviv/ or src/gallium/drivers/etnaviv/ ...
devtool build mesa
devtool deploy-target mesa root@192.168.1.50:/
```

`devtool` creates a local workspace checkout, cross-compiles against the embedded sysroot, and deploys the resulting binaries over SSH — enabling rapid GPU driver iteration without reflashing the board.

### MACHINE Configuration for NXP i.MX8

The `imx8mqevk.conf` machine configuration (from `meta-freescale`) sets:

```bitbake
# meta-freescale/conf/machine/imx8mqevk.conf
DEFAULTTUNE = "cortexa53-crypto"
require conf/machine/include/imx8mq.inc

MACHINE_FEATURES += "wayland"
MACHINE_EXTRA_RRECOMMENDS += "kernel-modules"

PREFERRED_PROVIDER_virtual/kernel = "linux-imx"
PREFERRED_PROVIDER_virtual/libgles2 = "mesa"
PREFERRED_PROVIDER_virtual/egl = "mesa"

UBOOT_MACHINE = "imx8mq_evk_defconfig"
```

The `virtual/libgles2` and `virtual/egl` provider assignments select the open-source Mesa/etnaviv path over NXP's proprietary GPU binaries.

### Buildroot for Minimal Footprint

Buildroot is preferred over Yocto on single-board systems requiring sub-100 MB root filesystems or extremely fast build times. Key Kconfig options for automotive/kiosk graphics:

```kconfig
BR2_PACKAGE_WESTON=y
BR2_PACKAGE_WESTON_BACKEND_DRM=y
BR2_PACKAGE_MESA3D=y
BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_ETNAVIV=y   # NXP i.MX8 / Vivante
BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_PANFROST=y  # ARM Mali (Panfrost open driver)
BR2_PACKAGE_MESA3D_OPENGL_EGL=y
BR2_PACKAGE_MESA3D_OPENGL_ES=y
BR2_PACKAGE_LIBDRM=y
BR2_PACKAGE_CAGE=y                            # kiosk compositor
```

Buildroot's single-config approach is simpler than Yocto for teams who need a working kiosk image fast but sacrifices Yocto's incremental build caching and extensible layer model.

---

## 8. GPU Driver Bring-Up on New SoCs

### Device Tree Binding

Every SoC GPU in Linux is described in the device tree. The GPU node declares its hardware resources: MMIO register ranges, interrupts, clocks, power domains, and IOMMU attachment. A representative Vivante (etnaviv) GPU node for an i.MX8-class SoC:

```dts
gpu: gpu@38000000 {
    compatible = "vivante,gc";        /* matches etnaviv of_device_id */
    reg = <0x0 0x38000000 0x0 0x40000>;   /* GPU MMIO window (256 KB) */
    interrupts = <GIC_SPI 3 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX8MQ_CLK_GPU_CORE_ROOT>,
             <&clks IMX8MQ_CLK_GPU_SHADER_ROOT>,
             <&clks IMX8MQ_CLK_GPU_AXI>;
    clock-names = "core", "shader", "bus";
    assigned-clocks = <&clks IMX8MQ_CLK_GPU_CORE_SRC>,
                      <&clks IMX8MQ_CLK_GPU_SHADER_SRC>;
    assigned-clock-rates = <800000000>, <800000000>;
    power-domains = <&pgc_gpu>;
    iommus = <&smmu 0x7 0x0>;   /* SMMU stream ID */
    operating-points-v2 = <&gpu_opp_table>;
    status = "okay";
};
```

The `compatible` string `"vivante,gc"` is the only compatible used in the mainline i.MX8MQ device tree (`arch/arm64/boot/dts/freescale/imx8mq.dtsi`) — this is the single generic binding that the etnaviv driver matches against all Vivante GCxxx series GPU cores [Source](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/freescale/imx8mq.dtsi).

### DRM Driver Registration

A minimal SoC GPU DRM driver lifecycle (simplified for readability):

```c
/* drivers/gpu/drm/mygpu/mygpu_drv.c */

static const struct of_device_id mygpu_of_match[] = {
    { .compatible = "vendor,mygpu-v2" },
    { }
};
MODULE_DEVICE_TABLE(of, mygpu_of_match);

static int mygpu_probe(struct platform_device *pdev)
{
    struct mygpu_device *mdev;
    struct drm_device *drm;

    /* Modern idiom: embed drm_device in driver-private struct */
    mdev = devm_drm_dev_alloc(&pdev->dev, &mygpu_driver,
                               struct mygpu_device, drm);
    if (IS_ERR(mdev))
        return PTR_ERR(mdev);
    drm = &mdev->drm;

    /* Map MMIO registers */
    mdev->regs = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(mdev->regs))
        return PTR_ERR(mdev->regs);

    /* Initialise GEM, KMS, command submission */
    ret = mygpu_gem_init(mdev);
    ret = mygpu_kms_init(mdev);   /* drm_mode_config_init, CRTC/planes */

    /* Advertise to userspace */
    ret = drm_dev_register(drm, 0);
    return ret;
}

static struct platform_driver mygpu_platform_driver = {
    .driver = {
        .name           = "mygpu",
        .of_match_table = mygpu_of_match,
        .pm             = &mygpu_pm_ops,
    },
    .probe  = mygpu_probe,
    .remove = mygpu_remove,
};
module_platform_driver(mygpu_platform_driver);
```

The `module_platform_driver()` macro expands to the `module_init`/`module_exit` pair that calls `platform_driver_register()` / `platform_driver_unregister()`.

### IOMMU Integration

Most automotive SoCs include a System MMU (SMMU or IOMMU) that provides per-device address space isolation for DMA — critical for ASIL-rated systems where a misbehaving GPU DMA transaction must not corrupt the safety-partition memory. The kernel DRM core integrates with the IOMMU subsystem via the DMA API:

```c
/* GPU command buffer DMA mapping with IOMMU */
struct sg_table sgt;
sg_alloc_table_from_pages(&sgt, pages, npages, 0, size, GFP_KERNEL);
dma_map_sg(&gpu->dev, sgt.sgl, sgt.nents, DMA_TO_DEVICE);
/* GPU sees IOVA (I/O Virtual Address), not physical addresses */
```

The SMMU translates IOVA to PA, enforcing that the GPU can only access memory regions explicitly mapped for it.

### Early Framebuffer and Bring-Up Debug

Before the full DRM driver is operational, the bootloader (U-Boot) or EFI stub configures a `simple-framebuffer` device tree node that the kernel's `simplefb` driver uses for a basic pixel-writable display:

```dts
framebuffer0: framebuffer@c0000000 {
    compatible = "simple-framebuffer";
    reg = <0x0 0xc0000000 0x0 0x00800000>;  /* 8 MB for 1920×1080×32bpp */
    width = <1920>;
    height = <1080>;
    stride = <7680>;   /* 1920 * 4 bytes per pixel */
    format = "a8r8g8b8";
    status = "okay";
};
```

`simplefb` provides `/dev/fb0` to the kernel console and to fbdev applications, allowing early display output (kernel log, splash screen) before `etnaviv` or `panfrost` initialises. Once the DRM driver loads, it evicts the simplefb driver by calling `drm_aperture_remove_framebuffers()` (the current mainline API in `drivers/gpu/drm/drm_aperture.c`; older kernels used `aperture_remove_conflicting_devices()`) [Source](https://docs.kernel.org/6.16/driver-api/aperture.html). This removes all framebuffer drivers occupying the same MMIO aperture, freeing the display hardware for the DRM driver's KMS pipeline.

### Debug Tools

```bash
# Enumerate DRM resources on the target
drm_info          # lists CRTCs, planes, connectors, properties

# Kernel DRM debugfs (requires CONFIG_DRM_DEBUG=y)
ls /sys/kernel/debug/dri/0/
cat /sys/kernel/debug/dri/0/state       # current atomic KMS state
cat /sys/kernel/debug/dri/0/etnaviv/gpu0  # etnaviv-specific state

# GPU clock and power state
cat /sys/bus/platform/drivers/etnaviv/*/devfreq/*/cur_freq

# Check GPU firmware load
dmesg | grep -i "etnaviv\|vivante\|panfrost\|mali"
```

---

## 9. Display Panel Bring-Up

### MIPI DSI Panel Driver

MIPI DSI is the dominant panel interface for automotive centre-stack displays (10"–15" format), instrument clusters, and smartphone-sized displays. A Linux DRM panel driver implements `drm_panel_funcs` to manage the panel power sequence:

```c
/* drivers/gpu/drm/panel/panel-my-dsi.c */

static int my_panel_prepare(struct drm_panel *panel)
{
    struct my_panel *p = to_my_panel(panel);

    /* Power on sequence: VDDIO → VDD → reset deassertion */
    regulator_enable(p->supply_vddio);
    usleep_range(2000, 3000);
    regulator_enable(p->supply_vdd);
    usleep_range(5000, 6000);

    gpiod_set_value_cansleep(p->reset_gpio, 1);
    usleep_range(1000, 2000);
    gpiod_set_value_cansleep(p->reset_gpio, 0);
    msleep(120);   /* panel datasheet: 120ms post-reset delay */

    return 0;
}

static int my_panel_enable(struct drm_panel *panel)
{
    struct my_panel *p = to_my_panel(panel);

    /* Send DCS initialisation sequence over MIPI DSI */
    mipi_dsi_dcs_exit_sleep_mode(p->dsi);  /* DCS 0x11 */
    msleep(5);
    mipi_dsi_dcs_set_display_on(p->dsi);   /* DCS 0x29 */

    /* Custom vendor commands (panel-specific register initialisation) */
    mipi_dsi_dcs_write_seq(p->dsi, 0xB0, 0x00);  /* manufacturer CMD enable */
    mipi_dsi_dcs_write_seq(p->dsi, 0xB3, 0x04, 0x08, 0x00, 0x22, 0x00);

    backlight_enable(p->backlight);
    return 0;
}

static int my_panel_disable(struct drm_panel *panel)
{
    struct my_panel *p = to_my_panel(panel);
    backlight_disable(p->backlight);
    mipi_dsi_dcs_set_display_off(p->dsi);  /* DCS 0x28 */
    msleep(20);
    mipi_dsi_dcs_enter_sleep_mode(p->dsi); /* DCS 0x10 */
    msleep(80);
    return 0;
}

static const struct drm_panel_funcs my_panel_funcs = {
    .prepare   = my_panel_prepare,
    .enable    = my_panel_enable,
    .disable   = my_panel_disable,
    .unprepare = my_panel_unprepare,
    .get_modes = my_panel_get_modes,
};
```

The `prepare()` → `enable()` ordering is contractual: the MIPI DSI controller sends DCS commands only after `prepare()` has completed (regulators on, reset released, panel in LP-11 idle state); `enable()` is called once the video stream from the host is running.

For standard panels whose initialisation sequence follows public datasheets, `drivers/gpu/drm/panel/panel-simple.c` provides a descriptor-driven framework where only the display timings and DCS sequence need to be declared:

```c
static const struct panel_desc my_standard_panel = {
    .modes = {
        .clock      = 148500,    /* kHz */
        .hdisplay   = 1920, .hsync_start = 2008, .hsync_end = 2052, .htotal = 2200,
        .vdisplay   = 1080, .vsync_start = 1084, .vsync_end = 1089, .vtotal = 1125,
        .flags      = DRM_MODE_FLAG_NHSYNC | DRM_MODE_FLAG_NVSYNC,
    },
    .bpc        = 8,
    .size       = { .width = 344, .height = 194 },  /* mm */
    .connector_type = DRM_MODE_CONNECTOR_DSI,
};
```

### LVDS Panels via Bridge Chip

LVDS (Low-Voltage Differential Signalling) remains common in automotive cluster displays (8"–12" diagonal, up to 1280×800). SoCs with only a MIPI DSI output use a bridge chip such as the **Toshiba TC358764** to convert DSI to LVDS. The kernel DRM bridge driver `drivers/gpu/drm/bridge/tc358764.c` implements `drm_bridge_funcs`:

```c
/* Simplified from tc358764.c */
static void tc358764_pre_enable(struct drm_bridge *bridge)
{
    struct tc358764 *ctx = bridge_to_tc358764(bridge);

    regulator_bulk_enable(ARRAY_SIZE(ctx->supplies), ctx->supplies);
    usleep_range(10000, 15000);

    /* GPIO reset pulse */
    gpiod_set_value(ctx->gpio_reset, 1);
    usleep_range(1000, 2000);
    gpiod_set_value(ctx->gpio_reset, 0);

    /* Configure PPI timing, DSI lanes, LVDS output */
    tc358764_write(ctx, PPI_LINEINITCNT, 0x00000E00);
    tc358764_write(ctx, PPI_LPTXTIMECNT, 0x00000004);
    tc358764_write(ctx, PPI_D0S_CLRSIPOCOUNT, 0x00000003);
    /* ... ~25 register writes total ... */
    tc358764_write(ctx, LVMX0003, 0x03020100);  /* LVDS bit mux */
    tc358764_write(ctx, LVCFG, 0x00000001);     /* Enable LVDS output */
}
```

The bridge chain in the device tree connects: DSI controller → TC358764 bridge → LVDS panel:

```dts
/* DSI controller node */
dsi@30a00000 {
    ports {
        port@1 {
            dsi_out: endpoint {
                remote-endpoint = <&bridge_in>;
            };
        };
    };
};

/* TC358764 bridge */
bridge@0e {
    compatible = "toshiba,tc358764";
    reg = <0x0e>;
    vddc-supply    = <&reg_1v2>;
    vddio-supply   = <&reg_1v8>;
    vddlvds-supply = <&reg_3v3>;
    reset-gpios    = <&gpio3 15 GPIO_ACTIVE_LOW>;
    ports {
        port@0 { bridge_in:  endpoint { remote-endpoint = <&dsi_out>; }; };
        port@1 { bridge_out: endpoint { remote-endpoint = <&panel_in>; }; };
    };
};
```

### eDP for Internal Displays

Laptop-style embedded displays and some automotive rear-seat units use eDP (embedded DisplayPort). The kernel's `drm_dp_aux` AUX channel exposes an I²C adapter (`aux->ddc`) over which EDID is fetched using the same `drm_get_edid()` call used for HDMI/VGA:

```c
/* Read EDID from an eDP panel via the DP AUX DDC adapter */
edid = drm_get_edid(connector, &aux->ddc);
if (edid) {
    drm_connector_update_edid_property(connector, edid);
    drm_add_edid_modes(connector, edid);
}
```

The `drm_dp_aux` structure registers its embedded I²C adapter (`aux->ddc`) with the kernel I²C bus; `drm_get_edid()` issues the I²C EDID segment reads over it. DPCD (DisplayPort Configuration Data) reads — link training status, sink capability — are performed separately via `drm_dp_dpcd_read()`.

For displays without EDID (common in embedded fixed panels), the driver provides a hardcoded mode via `drm_add_modes_noedid()` or a device-tree `display-timings` node.

### EDID via DDC I²C

HDMI-attached displays on automotive media boxes and industrial panels carry EDID over DDC (Display Data Channel) I²C. The kernel reads EDID with:

```c
edid = drm_get_edid(connector, ddc_i2c_adapter);
if (edid) {
    drm_connector_update_edid_property(connector, edid);
    drm_add_edid_modes(connector, edid);
}
```

On industrial panels with non-standard or missing EDID, `drm_edid_override_connector_update()` allows a firmware/device-tree-supplied EDID binary to be injected, bypassing the DDC read. The function exists in upstream kernels (a bug in its return value was fixed in 6.7-rc4); the precise kernel version in which the current API stabilised is not definitively documented — use a "needs verification" callout if targeting a specific kernel in production: **Note: verify `drm_edid_override_connector_update()` availability against your target kernel version.**

---

## 10. Remote Rendering in Automotive Cabin Networks

### Use Case: Centralised Compute with Remote Displays

Modern software-defined vehicles consolidate compute onto one or two high-performance SoCs (e.g. SA8295P, NVIDIA Orin) while keeping display hardware distributed: instrument cluster bezel, centre stack, rear-seat headrests, and HUD all have their own display controllers. Rendering for some of these secondary displays — particularly the instrument cluster — may occur on the central compute SoC rather than on a dedicated display SoC. The rendered frames must then be transmitted over the in-vehicle Ethernet backbone.

**Important architectural note**: SOME/IP (Service-Oriented Middleware over IP) is a service/signal RPC transport — it carries typed service invocations and signal values (speed=83, gear=3, warning=BRAKE), not compressed video frames. A common architectural split is:

- SOME/IP carries **state signals** from the vehicle middleware to the cluster SoC, which **renders locally**.
- Alternatively, a central compute SoC renders the cluster HMI and streams **compressed video** via RTP/UDP H.264 to the cluster display SoC.

Both patterns appear in production; the choice depends on ASIL partitioning requirements (local rendering is easier to certify), display SoC capability, and latency budget.

### GStreamer Pipeline for H.264 Frame Streaming

A GStreamer transmit pipeline on the central compute SoC, capturing a GPU-rendered frame via V4L2 and encoding it:

```bash
gst-launch-1.0 \
  v4l2src device=/dev/video0 \
  ! video/x-raw,format=NV12,width=1280,height=720,framerate=60/1 \
  ! v4l2h264enc extra-controls="controls,h264_i_frame_period=30" \
  ! h264parse \
  ! rtph264pay pt=96 \
  ! udpsink host=192.168.10.2 port=5004
```

On the cluster SoC receiver:

```bash
gst-launch-1.0 \
  udpsrc port=5004 caps="application/x-rtp,payload=96" \
  ! rtph264depay \
  ! avdec_h264 \
  ! videoconvert \
  ! waylandsink
```

For hardware-accelerated decode on an i.MX8 cluster SoC:

```bash
  ! v4l2h264dec \
  ! waylandsink
```

### Latency Budget

A representative latency budget for non-safety cluster graphics via H.264 streaming:

| Stage | Typical Latency |
|-------|----------------|
| GPU render to V4L2 capture | ~2 ms |
| H.264 encode (hardware) | ~4–6 ms |
| Gigabit Ethernet frame transmission | ~0.1 ms |
| H.264 decode (hardware) | ~4–6 ms |
| Display VSYNC alignment | ~0–8 ms |
| **Total** | **~12–22 ms** |

A 20 ms end-to-end latency is acceptable for secondary information displays (media, navigation, HVAC thumbnails on the cluster) but is **not acceptable for primary safety displays** (speed, warnings). Safety-critical displays must either render locally on the cluster SoC or use a dedicated hardware path with deterministic latency.

### Waltham for Wayland Surface Forwarding

As noted in §2, AGL uses **Waltham** for forwarding Wayland protocol messages (surface buffer commits, input events) over TCP to secondary display SoCs. Waltham is more efficient than H.264 streaming for vector/UI content: it forwards DMA-BUF handles (where the two SoCs share a PCIe or NVMe interconnect) or compressed buffer snapshots. The remote SoC's `waltham-receiver` compositor reconstructs the Wayland surface and presents it via its local KMS pipeline.

**Note**: Waltham adoption in production is limited and its real-world latency and bandwidth characteristics over automotive Ethernet (100Base-T1, 1000Base-T1) are not publicly benchmarked as of 2026; treat the above as architectural description rather than tested specification.

### Adaptive AUTOSAR Graphics Services

Adaptive AUTOSAR (AP) defines a standardised software architecture for SDV high-compute ECUs. AP defines an `ara::com` communication framework (over SOME/IP) but does not standardise a graphics API at the AP level. Vendor implementations (ETAS, Vector, Elektrobit) typically integrate AGL UCB or a custom Linux-based HMI container as an AP adaptive application, exposing system state via SOME/IP services and receiving rendered surface updates via Waltham or direct DMA-BUF sharing.

---

## Integrations

- **Chapter 1 (DRM Architecture)**: Platform GPU drivers — `etnaviv` for Vivante, `msm` for Adreno, `panfrost` for Mali — all register with the DRM core via `platform_driver_register()`, implement `struct drm_driver`, and follow the `drm_dev_alloc` / `drm_dev_register` lifecycle described in Ch1.

- **Chapter 2 (KMS: The Display Pipeline)**: All display bring-up in this chapter — DSI panel drivers, LVDS bridge chips, atomic modeset for automotive outputs — uses the KMS atomic commit API (`DRM_IOCTL_MODE_ATOMIC`). Plane assignment for Qt Safe Renderer's direct overlay maps directly onto KMS primary and overlay plane semantics.

- **Chapter 21 (Building Compositors with wlroots)**: cage is a wlroots 0.20-based compositor; the wlroots DRM backend, seat abstraction, and xdg-shell implementation are the backbone of the kiosk deployment model. agl-compositor is *not* wlroots-based; it uses libweston.

- **Chapter 39 (Qt and GTK GPU Rendering)**: Qt Interface Framework, QtApplicationManager, Qt Safe Renderer, and `QSGRenderNode` are all Qt-internal components whose GPU rendering ultimately goes through Qt RHI (OpenGL ES / Vulkan) on the target GPU.

- **Chapter 58 (GStreamer: Pipeline-Based Multimedia)**: The H.264 remote rendering pipeline in §10 is a GStreamer pipeline using `v4l2src`, `v4l2h264enc`, `rtph264pay`, and `udpsink` on the transmit side; `rtph264depay`, `v4l2h264dec`, and `waylandsink` on the receive side.

- **Chapter 90 (Open ARM GPU Drivers — Lima, Panfrost, and Panthor)**: Renesas RZ/G3L and other ARM Mali-bearing embedded SoCs use Panfrost (for Mali Midgard/Bifrost) or Panthor (for Mali CSF/Gen10+). This chapter's §8 bring-up methodology applies directly to those drivers.

- **Chapter 100 (etnaviv: The Vivante GPU Open Driver)**: NXP i.MX6 and i.MX8 use the etnaviv kernel DRM driver and Mesa Gallium3D backend covered in depth in Ch100. This chapter's Yocto integration and device tree examples are the BSP layer above that driver.

- **Chapter 92 (The Raspberry Pi GPU Stack)**: The Raspberry Pi CM4 and CM5 are widely used as automotive/kiosk compute modules. The V3D kernel driver, Weston bring-up, and Buildroot/Yocto integration patterns described there parallel this chapter's i.MX8 bring-up narrative.

---

## References

1. Automotive Grade Linux — SoDeV Platform announcement: [https://www.linuxfoundation.org/press/agl_sodev](https://www.linuxfoundation.org/press/agl_sodev)
2. AGL UCB 9.0 release: [https://www.linuxfoundation.org/press/press-release/agl_ucb9](https://www.linuxfoundation.org/press/press-release/agl_ucb9)
3. Collabora — "A libweston-based compositor for Automotive Grade Linux": [https://www.collabora.com/news-and-blog/news-and-events/a-libweston-based-compositor-for-automotive-grade-linux.html](https://www.collabora.com/news-and-blog/news-and-events/a-libweston-based-compositor-for-automotive-grade-linux.html)
4. COVESA wayland-ivi-extension repository: [https://github.com/COVESA/wayland-ivi-extension](https://github.com/COVESA/wayland-ivi-extension)
5. cage kiosk compositor GitHub: [https://github.com/cage-kiosk/cage](https://github.com/cage-kiosk/cage)
6. Qt Safe Renderer — Functional Safety and Qt: [https://www.qt.io/product/functional-safety-and-qt](https://www.qt.io/product/functional-safety-and-qt)
7. Qt Automotive Suite: [https://www.qt.io/qt-automotive-suite/](https://www.qt.io/qt-automotive-suite/)
8. Qt Interface Framework documentation: [https://doc.qt.io/QtInterfaceFramework/](https://doc.qt.io/QtInterfaceFramework/)
9. Yocto Project — Using Wayland and Weston: [https://docs.yoctoproject.org/dev-manual/wayland.html](https://docs.yoctoproject.org/dev-manual/wayland.html)
10. DRM Internals — Linux kernel documentation: [https://docs.kernel.org/gpu/drm-internals.html](https://docs.kernel.org/gpu/drm-internals.html)
11. Linux GPU Driver Developer's Guide: [https://dri.freedesktop.org/docs/drm/gpu.html](https://dri.freedesktop.org/docs/drm/gpu.html)
12. TC358764 DSI/LVDS bridge kernel driver: [https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/bridge/tc358764.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/bridge/tc358764.c)
13. TC358764 device tree binding documentation: [https://www.kernel.org/doc/Documentation/devicetree/bindings/display/bridge/toshiba,tc358764.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/display/bridge/toshiba,tc358764.txt)
14. R-Car DU DRM driver commit: [https://cgit.freedesktop.org/~airlied/linux/commit/?id=4bf8e1962f91eed5dbee168d2348983dda0a518f](https://cgit.freedesktop.org/~airlied/linux/commit/?id=4bf8e1962f91eed5dbee168d2348983dda0a518f)
15. Pengutronix — i.MX8MP graphics showcase: [https://pengutronix.de/en/blog/2021-02-26-showcase-i-mx8mp.html](https://pengutronix.de/en/blog/2021-02-26-showcase-i-mx8mp.html)
16. Qualcomm — Adreno 800 series Linux driver patches: [https://www.phoronix.com/news/Linux-Kernel-Adreno-800-Patches](https://www.phoronix.com/news/Linux-Kernel-Adreno-800-Patches)
17. AGL Instrument Cluster Expert Group on GitHub: [https://github.com/agl-ic-eg](https://github.com/agl-ic-eg)
18. Weston compositor documentation: [https://wayland.freedesktop.org/weston.html](https://wayland.freedesktop.org/weston.html)
