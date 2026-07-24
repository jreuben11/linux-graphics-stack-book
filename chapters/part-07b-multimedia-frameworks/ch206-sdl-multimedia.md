# Chapter 206: SDL3 — Cross-Platform Multimedia Integration on Linux

> **Part**: Part VII-B — Multimedia Frameworks and Desktop Integration
> **Audience**: Graphics application developers, game developers, and tooling engineers who need a single library to bridge audio, input, windowing, and GPU access on Linux; systems developers who want to understand how a high-level multimedia layer maps onto the Linux graphics and audio stack.
> **Status**: First draft — 2026-07-18

SDL (Simple DirectMedia Layer) is the de facto cross-platform multimedia integration library for C/C++ applications and games. Where Qt (Ch39) and GTK (Ch39) are desktop widget toolkits, SDL is a lower-level integration layer: it exposes a unified API for windowing, input, audio, and now GPU access without providing a widget hierarchy. Every serious Linux game or multimedia application either uses SDL directly or re-implements a substantial subset of what SDL provides.

SDL3.2.0 became the first stable, ABI-frozen release of the SDL 3 branch in January 2025. SDL2, the previous series, is now in maintenance mode — critical bug fixes only, no new features. This chapter covers SDL3 as the current target, noting SDL2 differences where relevant for the large installed base of SDL2 code still in production.

---

## Table of Contents

- [1. SDL Architecture: Subsystems and Hints](#1-sdl-architecture-subsystems-and-hints)
  - [1.3 What is SDL (Simple DirectMedia Layer)?](#13-what-is-sdl-simple-directmedia-layer)
  - [1.4 What is an SDL Subsystem?](#14-what-is-an-sdl-subsystem)
  - [1.5 What is the SDL Hint System?](#15-what-is-the-sdl-hint-system)
- [2. SDL on the Linux Stack: What It Wraps](#2-sdl-on-the-linux-stack-what-it-wraps)
- [3. Video Backend: Windows, EGL, and Vulkan Surfaces](#3-video-backend-windows-egl-and-vulkan-surfaces)
- [4. HiDPI and Fractional Scaling on Wayland](#4-hidpi-and-fractional-scaling-on-wayland)
- [5. Audio on Linux: PipeWire, ALSA, and the SDL_AudioStream Model](#5-audio-on-linux-pipewire-alsa-and-the-sdl_audiostream-model)
- [6. Input: Keyboard, Mouse, and Text Input on Wayland](#6-input-keyboard-mouse-and-text-input-on-wayland)
- [7. Gamepads, Joysticks, and Haptic Feedback](#7-gamepads-joysticks-and-haptic-feedback)
- [8. The SDL_Renderer 2D API: Surfaces and Textures](#8-the-sdl_renderer-2d-api-surfaces-and-textures)
- [9. Satellite Libraries: SDL_image, SDL_mixer, SDL_ttf](#9-satellite-libraries-sdl_image-sdl_mixer-sdl_ttf)
- [10. SDL3 Camera, Dialogs, and New APIs](#10-sdl3-camera-dialogs-and-new-apis)
- [11. SDL2 → SDL3 Migration Guide](#11-sdl2--sdl3-migration-guide)
- [12. Building and Packaging SDL3 on Linux](#12-building-and-packaging-sdl3-on-linux)
- [Integrations](#integrations)
- [References](#references)

---

## 1. SDL Architecture: Subsystems and Hints

SDL initialises as a collection of independent subsystems. A game that needs only audio and gamepad input never pays the cost of a display connection; a headless asset processor can use SDL's image and audio libraries without creating a window.

### 1.1 SDL_Init and Subsystem Flags

```c
#include <SDL3/SDL.h>

bool SDL_Init(SDL_InitFlags flags);
bool SDL_InitSubSystem(SDL_InitFlags flags);   // reference-counted
void SDL_QuitSubSystem(SDL_InitFlags flags);
void SDL_Quit(void);
SDL_InitFlags SDL_WasInit(SDL_InitFlags flags); // query active subsystems
```

`SDL_InitFlags` is `Uint32`. Each flag can be initialised separately and is reference-counted — multiple callers can init/quit the same subsystem safely:

| Flag | Hex | Auto-initialises | Notes |
|---|---|---|---|
| `SDL_INIT_AUDIO` | `0x00000010` | `SDL_INIT_EVENTS` | Audio device and driver |
| `SDL_INIT_VIDEO` | `0x00000020` | `SDL_INIT_EVENTS` | **Must be called from the main thread** |
| `SDL_INIT_JOYSTICK` | `0x00000200` | `SDL_INIT_EVENTS` | Raw joystick access |
| `SDL_INIT_HAPTIC` | `0x00001000` | — | Force-feedback devices |
| `SDL_INIT_GAMEPAD` | `0x00002000` | `SDL_INIT_JOYSTICK` | Mapped gamepad API |
| `SDL_INIT_EVENTS` | `0x00004000` | — | Event queue |
| `SDL_INIT_SENSOR` | `0x00008000` | `SDL_INIT_EVENTS` | Accelerometer/gyroscope |
| `SDL_INIT_CAMERA` | `0x00010000` | `SDL_INIT_EVENTS` | V4L2 / PipeWire camera (SDL3 only) |

`SDL_INIT_TIMER` (SDL2) is no longer required in SDL3 — timer functions work without explicit initialisation. `SDL_INIT_EVERYTHING` is removed in SDL3; init only what you need.

### 1.2 The Hint System

SDL's hint system controls backend selection and runtime behaviour. Every hint maps to an environment variable of the same name; environment variables take the highest priority:

```c
// Hints must be set BEFORE SDL_Init() for backend selection
bool SDL_SetHint(const char *name, const char *value);
bool SDL_SetHintWithPriority(const char *name, const char *value, SDL_HintPriority priority);
const char *SDL_GetHint(const char *name);

// Priority levels (highest wins):
// SDL_HINT_OVERRIDE  — environment variables, or explicit SDL_HINT_OVERRIDE calls
// SDL_HINT_NORMAL    — SDL_SetHint() default
// SDL_HINT_DEFAULT   — SDL internal defaults
```

**SDL3 naming change**: SDL2 used `SDL_VIDEODRIVER` and `SDL_AUDIODRIVER`; SDL3 uses `SDL_VIDEO_DRIVER` and `SDL_AUDIO_DRIVER`. The corresponding hint macros are `SDL_HINT_VIDEO_DRIVER` and `SDL_HINT_AUDIO_DRIVER`.

Key hints for Linux:

```c
// Select video backend (comma-separated = try in order)
SDL_SetHint(SDL_HINT_VIDEO_DRIVER, "wayland,x11");

// Select audio backend
SDL_SetHint(SDL_HINT_AUDIO_DRIVER, "pipewire");

// Select renderer backend (set before SDL_CreateRenderer)
SDL_SetHint(SDL_HINT_RENDER_DRIVER, "vulkan");

// Joystick: disable HIDAPI raw HID, use evdev instead
SDL_SetHint(SDL_HINT_JOYSTICK_HIDAPI, "0");
```

### 1.3 What is SDL (Simple DirectMedia Layer)?

SDL is a portable C library that provides a unified programming interface for windowing, input, audio, 2D rendering, and — since SDL3 — direct GPU access. The library operates as an abstraction layer between application code and a heterogeneous set of platform APIs: on Linux, it wraps Wayland and X11 for display, EGL and GLX for OpenGL context creation, the Vulkan WSI extensions for GPU surfaces, PipeWire, PulseAudio, and ALSA for audio, evdev and HIDAPI for input devices, and V4L2 plus the xdg-desktop-portal for camera access. Rather than adding widget-level concepts such as buttons, text fields, or layout managers, SDL deliberately stops at the OS and driver boundary, giving application code direct control over rendering while hiding the platform-specific details needed to set up that rendering environment. SDL3 is structured as a shared library (`libSDL3.so.0`) with a stable, ABI-frozen API from the 3.2.0 release onward. The header namespace uses the `SDL_` prefix throughout, and applications link against it via CMake `find_package(SDL3 REQUIRED)` or `pkg-config --libs sdl3`. This chapter covers SDL3's Linux-specific backend choices and how each subsystem maps to the kernel and compositor interfaces described elsewhere in this book. [Source](https://wiki.libsdl.org/SDL3/FrontPage)

### 1.4 What is an SDL Subsystem?

SDL groups its functionality into named subsystems — audio, video, events, joystick, gamepad, haptic, sensor, and camera — each of which can be independently initialised and shut down. An application that never opens a window never initialises the video subsystem; a command-line encoder that only transcodes audio never touches the input subsystem. The subsystem model is reference-counted: if two independent components both initialise `SDL_INIT_AUDIO`, the audio backend is opened only once and is torn down only when both have called `SDL_QuitSubSystem(SDL_INIT_AUDIO)`. Initialisation order within `SDL_Init` is deterministic: the events subsystem is always brought up first because it is required by all others, and the video subsystem must be initialised from the process's main thread because both Wayland and X11 impose this constraint at the compositor-protocol level. Each subsystem maps directly to one or more Linux kernel interfaces or display-server protocols: the video subsystem connects to the Wayland compositor over a Unix domain socket (`/run/user/<uid>/wayland-0`) or to an X11 display; the audio subsystem opens a PipeWire or ALSA client; the gamepad subsystem monitors `/dev/input/event*` via inotify for hotplug events. SDL3 removes `SDL_INIT_EVERYTHING` and the formerly mandatory `SDL_INIT_TIMER` flag, reflecting the principle that applications should declare only the capabilities they actually use.

### 1.5 What is the SDL Hint System?

The SDL hint system is a key-value configuration mechanism that controls driver selection, backend behaviour, and compatibility options at runtime. Hints are string-valued settings identified by macro names such as `SDL_HINT_VIDEO_DRIVER` or `SDL_HINT_AUDIO_DRIVER`. They follow a strict priority order: environment variables of the same name override all programmatic settings; `SDL_SetHintWithPriority` with `SDL_HINT_OVERRIDE` matches environment-variable precedence; `SDL_SetHint` applies at normal priority; and SDL's compiled-in defaults fill in last. Because many hints affect subsystem initialisation — particularly which display server, audio server, or renderer backend to select — they must be set before the corresponding `SDL_Init` call to take effect. On Linux, the hint system is the standard way to force a specific rendering path (for example, `SDL_HINT_RENDER_DRIVER=vulkan`), to select between Wayland and X11 (`SDL_HINT_VIDEO_DRIVER=wayland,x11`), or to disable problematic input drivers for a specific controller model. Hints also bridge SDL's portable API to kernel-level tuning: `SDL_HINT_JOYSTICK_HIDAPI=0` instructs the gamepad subsystem to use the evdev path rather than raw HID, which matters when a device exposes both `/dev/input/eventN` and `/dev/hidrawN` interfaces. SDL3 renamed several SDL2 hint names — `SDL_VIDEODRIVER` became `SDL_VIDEO_DRIVER`, for example — so applications migrating from SDL2 must audit every hint string they set.

---

## 2. SDL on the Linux Stack: What It Wraps

SDL abstracts a substantial slice of the Linux multimedia stack. Understanding what sits beneath each SDL subsystem clarifies both the performance characteristics and the failure modes:

| SDL subsystem | Linux layer(s) below |
|---|---|
| Video / Window | Wayland (`xdg-shell`, `libdecor`) or X11/XWayland; EGL for OpenGL ES; direct KMS/DRM (`kmsdrm` driver) |
| OpenGL context | EGL on Wayland (mandatory); GLX on X11 (default), EGL as fallback |
| Vulkan surface | `VK_KHR_wayland_surface` or `VK_KHR_xlib_surface` / `VK_KHR_xcb_surface` |
| Audio | PipeWire → PulseAudio → ALSA → JACK (probe order on SDL3 modern Linux) |
| Keyboard / Mouse | Wayland `wl_keyboard` / `wl_pointer` via `wl_seat` |
| Text input / IME | `zwp_text_input_v3` (text-input-unstable-v3) |
| Gamepad | HIDAPI (`/dev/hidraw*`) for known VIDs; evdev (`/dev/input/event*`) otherwise; udev hotplug |
| Haptic | Linux kernel FF (force-feedback) via evdev; HIDAPI for DualSense / Xbox BLE |
| Sensors | HIDAPI (controller IMU); V4L2/IIO on embedded |
| Camera | V4L2; PipeWire camera portal (SDL3) |
| File dialogs | `xdg-desktop-portal` (Ch23) |
| Clipboard | `wl_data_device` (Wayland); X11 atoms |

SDL2 defaulted to X11 even on Wayland-capable systems. SDL3 explicitly favours Wayland: `SDL_HINT_VIDEO_DRIVER` defaults to `"wayland,x11"` on Linux, and the audio backend probe order prefers PipeWire over PulseAudio when PipeWire is running. PipeWire backend support was added to SDL2 in 2021 and carries forward into SDL3 as the preferred backend.

---

## 3. Video Backend: Windows, EGL, and Vulkan Surfaces

### 3.1 Creating Windows

```c
SDL_Window *SDL_CreateWindow(const char *title, int w, int h, SDL_WindowFlags flags);
void SDL_DestroyWindow(SDL_Window *window);

// Convenience: create window + renderer together
bool SDL_CreateWindowAndRenderer(const char *title, int w, int h,
                                 SDL_WindowFlags flags,
                                 SDL_Window **window, SDL_Renderer **renderer);
```

Relevant `SDL_WindowFlags` on Linux:

| Flag | Effect |
|---|---|
| `SDL_WINDOW_RESIZABLE` | Enable user resize |
| `SDL_WINDOW_FULLSCREEN` | Exclusive fullscreen (or borderless with no mode set) |
| `SDL_WINDOW_BORDERLESS` | No window chrome (requires compositor support) |
| `SDL_WINDOW_HIGH_PIXEL_DENSITY` | Request 1:1 physical pixel mapping (HiDPI) |
| `SDL_WINDOW_OPENGL` | Prepare for OpenGL context (not needed for SDL_Renderer) |
| `SDL_WINDOW_VULKAN` | Prepare for Vulkan surface creation |
| `SDL_WINDOW_HIDDEN` | Create without mapping to the screen |

### 3.2 Fullscreen Modes

```c
// Borderless fullscreen (compositor scales the window):
bool SDL_SetWindowFullscreen(SDL_Window *window, true);
// No mode set = takes current desktop resolution with no modeswitch

// Exclusive fullscreen (real resolution change):
const SDL_DisplayMode **modes = SDL_GetFullscreenDisplayModes(displayID, &count);
bool SDL_SetWindowFullscreenMode(SDL_Window *window, modes[0]);
bool SDL_SetWindowFullscreen(SDL_Window *window, true);

// Sync Wayland async state (call after fullscreen changes on Wayland):
bool SDL_SyncWindow(SDL_Window *window);
```

**Wayland limitation**: `SDL_SetWindowPosition()` is non-functional for normal toplevels on Wayland — the compositor controls window placement. Applications should not assume initial window position.

### 3.3 EGL and OpenGL Contexts on Wayland

SDL uses EGL (not GLX) for OpenGL context creation on Wayland. The code path is `src/video/SDL_egl.c` with EGL function loading at runtime. On X11, SDL uses GLX by default (`src/video/x11/SDL_x11opengl.c`, loading `libGL.so.1`), with EGL available as a fallback.

```c
// Set OpenGL attributes before window creation:
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 4);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 6);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, SDL_GL_CONTEXT_PROFILE_CORE);
SDL_GL_SetAttribute(SDL_GL_DOUBLEBUFFER, 1);

SDL_Window *window = SDL_CreateWindow("GL App", 1280, 720,
                                      SDL_WINDOW_OPENGL | SDL_WINDOW_RESIZABLE);
SDL_GLContext ctx = SDL_GL_CreateContext(window);
SDL_GL_MakeCurrent(window, ctx);

// Vsync: 0=off, 1=on, -1=adaptive
SDL_GL_SetSwapInterval(-1);

// Render loop:
SDL_GL_SwapWindow(window);

SDL_GL_DestroyContext(ctx);
```

When the Wayland backend is active, `SDL_GL_CreateContext` calls `eglCreateContext` against the `wl_display`-backed EGL display. Mesa's EGL drivers (ANV, RADV, etc.) handle the rest.

### 3.4 Vulkan Surface Creation

SDL makes Vulkan surface creation backend-agnostic — the same calls work on Wayland, X11, and all other platforms:

```c
// Retrieve required instance extensions for the active video backend
Uint32 count;
char const * const *extensions = SDL_Vulkan_GetInstanceExtensions(&count);
// On Wayland: {"VK_KHR_surface", "VK_KHR_wayland_surface"}
// On X11:     {"VK_KHR_surface", "VK_KHR_xlib_surface"}

// Include these in VkInstanceCreateInfo.ppEnabledExtensionNames

// Create surface (window must have SDL_WINDOW_VULKAN flag)
VkSurfaceKHR surface;
bool ok = SDL_Vulkan_CreateSurface(window, instance, NULL, &surface);

// Query presentation support for a queue family
bool supports_present = SDL_Vulkan_GetPresentationSupport(
    instance, physical_device, queue_family_index);

// Cleanup
SDL_Vulkan_DestroySurface(instance, surface, NULL);
```

The Wayland implementation calls `vkCreateWaylandSurfaceKHR` internally; the X11 implementation tries `vkCreateXlibSurfaceKHR` or `vkCreateXcbSurfaceKHR`. Unlike OpenGL, SDL does not manage Vulkan swap-chain presentation — that is the application's responsibility via the Vulkan API directly.

### 3.5 Wayland Protocol Surface

When running on Wayland, SDL3 creates an `xdg_toplevel` via `xdg_wm_base` (xdg-shell protocol). On compositors that do not support server-side decorations (`zxdg_decoration_manager_v1` — i.e., GNOME), SDL falls back to **libdecor** for client-side window chrome. Without libdecor installed on GNOME, windows have no title bar or close/minimise buttons. Install the `libdecor` package (including its GTK plugin for GNOME) to resolve.

The full set of Wayland protocols SDL3 implements includes: `viewporter`, `wp_fractional_scale_v1`, `xdg-activation-v1`, `xdg-foreign-unstable-v2`, `zwp_keyboard_shortcuts_inhibit_manager_v1`, `xdg-toplevel-icon-v1`, `frog-color-management-v1`, `color-management-v1`, and many others — see `src/video/wayland/SDL_waylandvideo.c` in the SDL source for the authoritative list.

---

## 4. HiDPI and Fractional Scaling on Wayland

Wayland separates logical coordinates (what the application receives for window sizes and mouse events) from physical coordinates (pixels on the display). SDL3 is fully aware of this split:

```c
// Combined scale factor: pixel density × content scale
// e.g., 2.0 on a 4K display with 200% UI scaling
float scale = SDL_GetWindowDisplayScale(window);

// Pixel density ratio: window pixels / logical window size
float density = SDL_GetWindowPixelDensity(window);

// Actual pixel dimensions of the window surface (not logical)
int px_w, px_h;
SDL_GetWindowSizeInPixels(window, &px_w, &px_h);

// Display content scale (same as UI scale factor, e.g. 1.5 for 150%)
SDL_DisplayID display = SDL_GetDisplayForWindow(window);
float content_scale = SDL_GetDisplayContentScale(display);
```

SDL3 implements `wp_fractional_scale_v1` + `wp_viewporter` to communicate fractional scale factors (e.g., 1.5×, 1.25×) to the compositor. Mouse event coordinates arrive in logical units as floats; convert to render coordinates before mapping to UI elements:

```c
case SDL_EVENT_MOUSE_MOTION: {
    float render_x, render_y;
    SDL_RenderCoordinatesFromWindow(renderer,
                                    event.motion.x, event.motion.y,
                                    &render_x, &render_y);
    // render_x, render_y are in renderer's logical coordinate space
}
```

`SDL_WINDOW_HIGH_PIXEL_DENSITY` requests the compositor allocate a buffer at the physical pixel resolution. Without it, the compositor may allocate at a lower resolution and scale up, which is faster but blurrier.

---

## 5. Audio on Linux: PipeWire, ALSA, and the SDL_AudioStream Model

### 5.1 Backend Selection

SDL3's audio backend probe order on modern Linux: **PipeWire → PulseAudio → ALSA → JACK → OSS**. SDL3 detects whether PipeWire is running (via D-Bus / systemd service detection) and prefers the native PipeWire driver over PulseAudio. Force a specific backend:

```bash
SDL_AUDIO_DRIVER=pipewire   ./app    # PipeWire native
SDL_AUDIO_DRIVER=pulseaudio ./app    # PulseAudio
SDL_AUDIO_DRIVER=alsa       ./app    # ALSA direct
SDL_AUDIO_DRIVER=jack       ./app    # JACK low-latency
```

The PipeWire backend (`libpipewire-0.3`) was added to SDL2 in 2021 and is the preferred backend in SDL3 on modern desktops (Fedora 36+, Ubuntu 22.04+, Arch).

### 5.2 Audio Formats

```c
typedef Uint32 SDL_AudioFormat;
// SDL_AUDIO_U8     0x0008   unsigned 8-bit
// SDL_AUDIO_S8     0x8008   signed 8-bit
// SDL_AUDIO_S16    0x8010   signed 16-bit, native byte order
// SDL_AUDIO_S32    0x8020   signed 32-bit int, native byte order
// SDL_AUDIO_F32    0x8120   32-bit float, native byte order  ← recommended
```

`SDL_AUDIO_U16` variants are removed in SDL3. Use `SDL_AUDIO_F32` for new code — it is the native format for most modern audio hardware and the most flexible for mixing.

### 5.3 The SDL_AudioStream Model (SDL3)

SDL3 replaces SDL2's callback-centric model with `SDL_AudioStream` — a push/pull queue with built-in format conversion, resampling, channel remapping, gain, and pitch ratio:

```c
// Open a device and create a bound stream — all in one call:
SDL_AudioSpec spec = {
    .format   = SDL_AUDIO_F32,
    .channels = 2,
    .freq     = 48000,
};
SDL_AudioStream *stream = SDL_OpenAudioDeviceStream(
    SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK,
    &spec,
    NULL,  // callback: NULL = manual push
    NULL   // userdata
);
// Device starts PAUSED. Resume explicitly:
SDL_ResumeAudioStreamDevice(stream);

// Push PCM audio for playback:
float samples[2048];  // interleaved stereo f32
// ... fill samples ...
SDL_PutAudioStreamData(stream, samples, sizeof(samples));

// Query how much data is buffered (bytes):
int queued = SDL_GetAudioStreamQueued(stream);

// Gain and pitch:
SDL_SetAudioStreamGain(stream, 0.75f);           // 75% volume
SDL_SetAudioStreamFrequencyRatio(stream, 1.05f); // 5% pitch up

// Runtime format change (SDL resamples internally):
SDL_AudioSpec new_src = {SDL_AUDIO_S16, 1, 22050};
SDL_SetAudioStreamFormat(stream, &new_src, NULL); // dst=NULL keeps device format

// Cleanup:
SDL_DestroyAudioStream(stream); // also closes the device
```

Multiple streams can be bound to one device; SDL mixes them automatically before output. A post-mix callback fires after mixing for effects or metering:

```c
void postmix(void *userdata, const SDL_AudioSpec *spec, float *buffer, int buflen) {
    // buffer is interleaved float samples already mixed
    // apply mastering limiter, VU meter, etc.
}
SDL_SetAudioPostmixCallback(device_id, postmix, NULL);
```

### 5.4 Device Enumeration

```c
int count;
SDL_AudioDeviceID *devs = SDL_GetAudioPlaybackDevices(&count);
for (int i = 0; i < count; i++) {
    const char *name = SDL_GetAudioDeviceName(devs[i]);
    SDL_AudioSpec spec;
    SDL_GetAudioDeviceFormat(devs[i], &spec, NULL);
    printf("%s — %d Hz, %d ch, format 0x%04x\n",
           name, spec.freq, spec.channels, spec.format);
}
SDL_free(devs);

// Special IDs (no enumeration needed for default devices):
// SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK
// SDL_AUDIO_DEVICE_DEFAULT_RECORDING
```

### 5.5 Recording

```c
SDL_AudioSpec rec_spec = {SDL_AUDIO_F32, 1, 44100};  // mono
SDL_AudioStream *rec = SDL_OpenAudioDeviceStream(
    SDL_AUDIO_DEVICE_DEFAULT_RECORDING, &rec_spec, NULL, NULL);
SDL_ResumeAudioStreamDevice(rec);

// Pull captured audio:
int available = SDL_GetAudioStreamAvailable(rec);
if (available > 0) {
    Uint8 buf[4096];
    int got = SDL_GetAudioStreamData(rec, buf, sizeof(buf));
    // process buf (got bytes of SDL_AUDIO_F32 mono 44100 Hz)
}
```

### 5.6 Format Conversion (One-Shot)

```c
// Do NOT use for streaming — use SDL_AudioStream instead:
SDL_AudioSpec src = {SDL_AUDIO_S16, 2, 44100};
SDL_AudioSpec dst = {SDL_AUDIO_F32, 2, 48000};
Uint8 *dst_data;
int dst_len;
bool ok = SDL_ConvertAudioSamples(&src, src_data, src_len,
                                  &dst, &dst_data, &dst_len);
// Free with SDL_free(dst_data)
```

---

## 6. Input: Keyboard, Mouse, and Text Input on Wayland

### 6.1 Event Loop

The SDL event loop is the entry point for all input on all platforms:

```c
SDL_Event e;
while (SDL_PollEvent(&e)) {
    switch (e.type) {
    case SDL_EVENT_QUIT:
        running = false;
        break;

    case SDL_EVENT_KEY_DOWN:
        // e.key.key       — SDL_Keycode (virtual key, layout-dependent)
        // e.key.scancode  — SDL_Scancode (physical key position)
        // e.key.mod       — SDL_Keymod bitmask (Shift, Ctrl, Alt, …)
        // e.key.down      — true (always for KEY_DOWN)
        // e.key.repeat    — true if this is an auto-repeat event
        if (e.key.key == SDLK_ESCAPE) running = false;
        break;

    case SDL_EVENT_MOUSE_MOTION:
        // e.motion.x, e.motion.y       — absolute position (float, logical coords)
        // e.motion.xrel, e.motion.yrel — relative delta (float)
        break;

    case SDL_EVENT_MOUSE_BUTTON_DOWN:
        // e.button.button — SDL_BUTTON_LEFT/MIDDLE/RIGHT/X1/X2
        // e.button.x, e.button.y (float)
        break;

    case SDL_EVENT_WINDOW_RESIZED:
        // e.window.data1 = new logical width
        // e.window.data2 = new logical height
        // Call SDL_GetWindowSizeInPixels() for pixel dimensions
        break;

    case SDL_EVENT_WINDOW_PIXEL_SIZE_CHANGED:
        // Fires when the physical pixel size changes (scale factor changed)
        break;
    }
}
```

**SDL3 API changes from SDL2**: All event constants renamed to `SDL_EVENT_*`. `SDL_WINDOWEVENT` with sub-type flattened to individual `SDL_EVENT_WINDOW_MOVED`, `SDL_EVENT_WINDOW_RESIZED`, etc. Mouse coordinates are now `float`. Timestamps are nanoseconds. `SDL_Keysym` struct removed; fields promoted directly into `SDL_KeyboardEvent`.

### 6.2 Keyboard State Polling

```c
// Returns bool* array indexed by SDL_Scancode (SDL2 returned Uint8*)
int num_keys;
const bool *keys = SDL_GetKeyboardState(&num_keys);
if (keys[SDL_SCANCODE_W]) { /* W held */ }
if (keys[SDL_SCANCODE_LSHIFT]) { /* Left shift held */ }
```

**SDL3 key name changes**: Single-letter keycodes promoted to uppercase — `SDLK_a` → `SDLK_A`. Modifier constants renamed from `KMOD_*` to `SDL_KMOD_*`.

### 6.3 Mouse: Relative Mode and Warping

For first-person games, enable relative mouse mode to receive unbounded delta motion:

```c
// SDL3: per-window (SDL2 had global SDL_SetRelativeMouseMode)
bool SDL_SetWindowRelativeMouseMode(SDL_Window *window, bool enabled);
// On Wayland: uses zwp_relative_pointer_manager_v1 + zwp_pointer_constraints_v1
// (locked pointer = cursor hidden and constrained; deltas arrive via relative pointer)

// Get relative delta without activating grab (instant poll):
float xrel, yrel;
SDL_GetRelativeMouseState(&xrel, &yrel);

// Warping
bool SDL_WarpMouseInWindow(SDL_Window *window, float x, float y);  // works on Wayland
bool SDL_WarpMouseGlobal(float x, float y);  // fails on Wayland — returns SDL_FALSE
```

**Wayland limitation**: Global mouse state is unavailable outside the focused window on Wayland. `SDL_GetGlobalMouseState()` returns (0,0) when the cursor is over another application.

### 6.4 Text Input and IME

On Wayland, SDL uses `zwp_text_input_v3` for IME (Input Method Editor) support. Text input must be explicitly started:

```c
bool SDL_StartTextInput(SDL_Window *window);
bool SDL_StopTextInput(SDL_Window *window);

// Position the compositor's IME candidate popup near the input field:
SDL_Rect cursor_rect = {x, y, 0, font_height};
SDL_SetTextInputArea(window, &cursor_rect, 0);
```

Events during IME composition:

```c
case SDL_EVENT_TEXT_EDITING:
    // e.edit.text — current preedit/composition string (UTF-8)
    // e.edit.start, e.edit.length — cursor and selection in preedit
    // Render preedit with underline; do not commit to document
    break;

case SDL_EVENT_TEXT_INPUT:
    // e.text.text — committed text (UTF-8), ready to insert
    break;
```

Text input is **not** automatically active — `SDL_StartTextInput()` must be called when an editable field is focused, and `SDL_StopTextInput()` when it loses focus. During active IME composition, some `SDL_EVENT_KEY_DOWN` events may be suppressed by the input method.

---

## 7. Gamepads, Joysticks, and Haptic Feedback

### 7.1 The Gamepad API (SDL3)

SDL3 completely renamed the SDL2 `GameController` API to `Gamepad`. The semantic change reflects positional rather than label-based button names:

```c
// Enumerate connected gamepads:
int count;
SDL_JoystickID *pads = SDL_GetGamepads(&count);  // free with SDL_free()

for (int i = 0; i < count; i++) {
    SDL_Gamepad *pad = SDL_OpenGamepad(pads[i]);

    // State polling:
    Sint16 left_x  = SDL_GetGamepadAxis(pad, SDL_GAMEPAD_AXIS_LEFTX);
    Sint16 left_y  = SDL_GetGamepadAxis(pad, SDL_GAMEPAD_AXIS_LEFTY);
    Sint16 l_trig  = SDL_GetGamepadAxis(pad, SDL_GAMEPAD_AXIS_LEFT_TRIGGER);
    bool south     = SDL_GetGamepadButton(pad, SDL_GAMEPAD_BUTTON_SOUTH); // A / Cross

    // Regional label (what to show in UI: A, Cross, etc.)
    SDL_GamepadButtonLabel label = SDL_GetGamepadButtonLabel(pad, SDL_GAMEPAD_BUTTON_SOUTH);

    SDL_CloseGamepad(pad);
}
SDL_free(pads);
```

Button naming change — positional in SDL3, label in SDL2:

| SDL2 (`SDL_CONTROLLER_BUTTON_*`) | SDL3 (`SDL_GAMEPAD_BUTTON_*`) |
|---|---|
| `A` | `SOUTH` |
| `B` | `EAST` |
| `X` | `WEST` |
| `Y` | `NORTH` |
| `BACK` | `BACK` |
| `GUIDE` | `GUIDE` |
| `START` | `START` |

Axis values range from −32768 to 32767. Trigger axes range from 0 to 32767.

### 7.2 Linux Gamepad Backend

On Linux, SDL uses a two-tier strategy for gamepad access:

1. **HIDAPI** (`/dev/hidraw*`): Used for known controllers — Xbox, PlayStation (DualShock 4, DualSense), Nintendo Switch Pro Controller, Steam Controller. HIDAPI gives access to features not exposed via the kernel's generic evdev interface: DualSense adaptive trigger state, gyroscope/accelerometer, LED colour, touchpad, battery level. Steam Controller Bluetooth support is enabled by default since SDL November 2024.

2. **evdev** (`/dev/input/event*`): Fallback for unrecognised controllers or when HIDAPI is disabled (`SDL_HINT_JOYSTICK_HIDAPI=0`). Udev handles hotplug notification.

The SDL gamepad database (`gamecontrollerdb.txt`, embedded at compile time and updatable at runtime via `SDL_AddGamepadMapping`) maps raw evdev/HID inputs to the standard SDL gamepad layout. Without a mapping, a device appears only as a raw `SDL_Joystick`, not as a `SDL_Gamepad`.

### 7.3 Hotplug Events

```c
case SDL_EVENT_GAMEPAD_ADDED:
    // e.gdevice.which — SDL_JoystickID of new gamepad
    pad = SDL_OpenGamepad(e.gdevice.which);
    break;

case SDL_EVENT_GAMEPAD_REMOVED:
    // e.gdevice.which — SDL_JoystickID of removed gamepad
    // SDL_CloseGamepad() is safe to call even if called from the removed handler
    SDL_CloseGamepad(pad);
    break;
```

### 7.4 Rumble and Haptic

Gamepad-level rumble is available via the Gamepad API directly:

```c
// Dual-motor rumble (low and high frequency):
SDL_RumbleGamepad(pad, 0xFFFF, 0x8000, 500);  // 500 ms

// Trigger rumble (DualSense, Xbox Elite):
SDL_RumbleGamepadTriggers(pad, 0x8000, 0x8000, 200);

// RGB LED (DualShock 4, DualSense, some Xbox):
SDL_SetGamepadLED(pad, 0, 180, 255);  // cyan

// DualSense adaptive trigger effects (raw HID, device-specific):
// Send vendor-specific effect data via:
SDL_SendGamepadEffect(pad, effect_data, sizeof(effect_data));
```

The standalone Haptic API supports complex force-feedback effects on devices that expose the Linux kernel's `FF_*` interface or HIDAPI haptic:

```c
SDL_Haptic *hap = SDL_OpenHapticFromJoystick(SDL_GetGamepadJoystick(pad));
SDL_InitHapticRumble(hap);
SDL_PlayHapticRumble(hap, 0.8f, 300);  // 80% strength, 300 ms
SDL_CloseHaptic(hap);
```

### 7.5 Controller Sensors (Gyroscope / Accelerometer)

DualShock 4, DualSense, and Nintendo Switch controllers expose IMU data via HIDAPI:

```c
// Check and enable gamepad sensor:
if (SDL_GamepadHasSensor(pad, SDL_SENSOR_GYRO)) {
    SDL_SetGamepadSensorEnabled(pad, SDL_SENSOR_GYRO, true);
}

// Poll (or use SDL_EVENT_GAMEPAD_SENSOR_UPDATE events):
float gyro[3];
SDL_GetGamepadSensorData(pad, SDL_SENSOR_GYRO, gyro, 3);
// gyro[0-2] = angular velocity in radians/second (x, y, z)

float accel[3];
SDL_GetGamepadSensorData(pad, SDL_SENSOR_ACCEL, accel, 3);
// accel[0-2] = acceleration in m/s² (includes gravity, SDL_STANDARD_GRAVITY=9.80665)
```

---

## 8. The SDL_Renderer 2D API: Surfaces and Textures

SDL provides a hardware-accelerated 2D rendering API via `SDL_Renderer`. On Linux the renderer backends are: `opengl` (Mesa OpenGL), `opengles2` (Mesa GLES2), `vulkan` (Mesa Vulkan), `gpu` (SDL3 GPU API), and `software` (no GPU required).

### 8.1 SDL_Surface: CPU Pixel Buffers

`SDL_Surface` holds pixels in system RAM, directly accessible to the CPU:

```c
SDL_Surface *surf = SDL_CreateSurface(256, 256, SDL_PIXELFORMAT_RGBA8888);
SDL_Surface *bmp  = SDL_LoadBMP("image.bmp");

// CPU drawing:
SDL_FillSurfaceRect(surf, NULL, SDL_MapRGB(SDL_GetPixelFormatDetails(surf->format),
                                            NULL, 255, 128, 0));

// Software blit:
SDL_BlitSurface(bmp, &src_rect, surf, &dst_rect);

SDL_DestroySurface(surf);
```

**SDL3 pixel format rename**: `SDL_PIXELFORMAT_RGB888` → `SDL_PIXELFORMAT_XRGB8888`; `SDL_PIXELFORMAT_BGR888` → `SDL_PIXELFORMAT_XBGR8888`. `SDL_PixelFormat` is now the enum; `SDL_PixelFormatDetails` is the struct.

### 8.2 SDL_Texture: GPU Textures

Textures live in GPU memory and are only accessible through the renderer. Uploading a surface to a texture:

```c
// Upload surface → GPU texture:
SDL_Texture *tex = SDL_CreateTextureFromSurface(renderer, surface);
// Surface can be destroyed immediately after

// Create a blank streaming texture (CPU update each frame):
SDL_Texture *stream_tex = SDL_CreateTexture(
    renderer, SDL_PIXELFORMAT_RGBA8888,
    SDL_TEXTUREACCESS_STREAMING, 640, 480);

// Update streaming texture:
void *pixels;
int pitch;
SDL_LockTexture(stream_tex, NULL, &pixels, &pitch);
// write to pixels...
SDL_UnlockTexture(stream_tex);

// Create render target texture:
SDL_Texture *rt = SDL_CreateTexture(
    renderer, SDL_PIXELFORMAT_RGBA8888,
    SDL_TEXTUREACCESS_TARGET, 1920, 1080);
SDL_SetRenderTarget(renderer, rt);
// draw to rt...
SDL_SetRenderTarget(renderer, NULL);  // back to window
```

### 8.3 SDL_Renderer 2D Drawing

SDL3's renderer uses a deferred command queue — draw calls are batched until `SDL_RenderPresent` or an explicit `SDL_FlushRenderer`. All coordinates and sizes are `float` in SDL3 (SDL2 used `int` for most rect types):

```c
SDL_Renderer *renderer = SDL_CreateRenderer(window, NULL);
// NULL = auto-select best available backend
// Or: "opengl", "vulkan", "opengles2", "software"

// Clear:
SDL_SetRenderDrawColor(renderer, 30, 30, 30, 255);
SDL_RenderClear(renderer);

// Draw texture (SDL2: SDL_RenderCopy → SDL3: SDL_RenderTexture):
SDL_FRect dst = {100.0f, 100.0f, 200.0f, 150.0f};
SDL_RenderTexture(renderer, tex, NULL, &dst);

// With rotation (SDL2: SDL_RenderCopyEx → SDL3: SDL_RenderTextureRotated):
SDL_FPoint centre = {100.0f, 75.0f};
SDL_RenderTextureRotated(renderer, tex, NULL, &dst, 45.0, &centre, SDL_FLIP_NONE);

// Geometry (triangles):
SDL_Vertex verts[3] = {
    {{400, 100}, {255,   0,   0, 255}, {0.0f, 0.0f}},
    {{300, 300}, {  0, 255,   0, 255}, {0.0f, 0.0f}},
    {{500, 300}, {  0,   0, 255, 255}, {0.0f, 0.0f}},
};
SDL_RenderGeometry(renderer, NULL, verts, 3, NULL, 0);

// Primitives:
SDL_SetRenderDrawColor(renderer, 255, 255, 0, 255);
SDL_FRect rect = {50, 50, 100, 60};
SDL_RenderRect(renderer, &rect);   // outline
SDL_RenderFillRect(renderer, &rect); // filled

// Logical resolution (scale to any physical size):
SDL_SetRenderLogicalPresentation(renderer, 1280, 720,
                                 SDL_LOGICAL_PRESENTATION_LETTERBOX);

// Present:
SDL_RenderPresent(renderer);  // implicit flush + swap
```

---

## 9. Satellite Libraries: SDL_image, SDL_mixer, SDL_ttf

### 9.1 SDL_image

SDL_image ([github.com/libsdl-org/SDL_image](https://github.com/libsdl-org/SDL_image)) adds format support beyond BMP to SDL's surface system. Supported formats include PNG, JPEG, WebP, GIF, QOI, TGA, BMP, PNM, XCF, XPM, LBM, PCX, and (conditionally at build time) AVIF, JPEG-XL, TIFF, SVG:

```c
#include <SDL3_image/SDL_image.h>

// Load any supported format to SDL_Surface:
SDL_Surface *surf = IMG_Load("texture.png");
SDL_Surface *surf = IMG_Load_IO(SDL_IOFromFile("data.webp", "rb"), true);

// Load directly to GPU texture (wraps IMG_Load + SDL_CreateTextureFromSurface):
SDL_Texture *tex = IMG_LoadTexture(renderer, "sprite.png");

// Load animated GIF (returns frame array):
IMG_Animation *anim = IMG_LoadAnimation("animation.gif");
// anim->frames[i]  = SDL_Surface* for frame i
// anim->delays[i]  = delay in milliseconds
// anim->count      = total frame count
IMG_FreeAnimation(anim);
```

**GPU texture upload**: `IMG_LoadTexture` uploads via `SDL_CreateTextureFromSurface` internally. For the SDL3 GPU API (`SDL_GPUDevice`), load to `SDL_Surface` first then use `SDL_UploadToGPUTexture()`.

### 9.2 SDL_mixer

SDL_mixer ([github.com/libsdl-org/SDL_mixer](https://github.com/libsdl-org/SDL_mixer)) provides channel-based audio mixing on top of SDL's audio system. It distinguishes between *music* (single streaming track: OGG, MP3, FLAC, MIDI, MOD formats) and *chunks* (short samples played on numbered mixing channels):

```c
#include <SDL3_mixer/SDL_mixer.h>

// Open mixer on default device:
SDL_AudioSpec spec = {SDL_AUDIO_F32, 2, 48000};
Mix_OpenAudio(SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK, &spec);

// Music (streaming playback):
Mix_Music *music = Mix_LoadMUS("background.ogg");
Mix_PlayMusic(music, -1);         // -1 = loop forever
Mix_VolumeMusic(64);              // 0–128 (MIX_MAX_VOLUME = 128)

// Chunks (triggered sound effects):
Mix_Chunk *sfx = Mix_LoadWAV("explosion.wav");
int ch = Mix_PlayChannel(-1, sfx, 0);  // auto-assign channel, play once
Mix_Volume(ch, 96);                    // set volume on that channel

// Channel management:
Mix_AllocateChannels(16);  // expand from default 8
Mix_HaltChannel(-1);       // halt all channels

// Custom DSP effect inserted on a channel:
void reverb_effect(int chan, void *stream, int len, void *udata) {
    // process len bytes of audio in stream
}
Mix_RegisterEffect(ch, reverb_effect, NULL, NULL);

Mix_FreeChunk(sfx);
Mix_FreeMusic(music);
Mix_CloseAudio();
```

SDL3_mixer uses `SDL_AudioStream` internally and is compatible with SDL3's audio model.

### 9.3 SDL_ttf

SDL_ttf ([github.com/libsdl-org/SDL_ttf](https://github.com/libsdl-org/SDL_ttf)) wraps FreeType (and optionally HarfBuzz for correct shaping of complex scripts) for font rendering:

```c
#include <SDL3_ttf/SDL_ttf.h>

TTF_Init();

// Load font (point size in SDL3 is float):
TTF_Font *font = TTF_OpenFont("Roboto-Regular.ttf", 24.0f);

// Rendering modes (return SDL_Surface*):
SDL_Color white = {255, 255, 255, 255};
SDL_Color bg    = {0,   0,   0,   255};

SDL_Surface *s1 = TTF_RenderText_Solid  (font, "Hello", 0, white);
SDL_Surface *s2 = TTF_RenderText_Shaded (font, "Hello", 0, white, bg);
SDL_Surface *s3 = TTF_RenderText_Blended(font, "Hello", 0, white);  // best for overlaid
SDL_Surface *s4 = TTF_RenderText_LCD    (font, "Hello", 0, white, bg); // subpixel

// Wrapped (word-wrap at wrapLength pixels):
SDL_Surface *wrapped = TTF_RenderText_Blended_Wrapped(font, long_text, 0, white, 600);

// SDF (for scalable/shader-based rendering):
TTF_SetFontSDF(font, true);
SDL_Surface *sdf = TTF_RenderText_Blended(font, "Hello", 0, white);
// Alpha channel contains signed distance values, not RGBA
```

**SDL3_ttf Text Engine API** — GPU-backed text rendering for applications using the SDL3 GPU API:

```c
// Renderer-backed text engine:
TTF_TextEngine *engine = TTF_CreateRendererTextEngine(renderer);
TTF_Text *text = TTF_CreateText(engine, font, "Hello, World", 0);
TTF_DrawRendererText(text, 100.0f, 200.0f);
TTF_DestroyText(text);
TTF_DestroyRendererTextEngine(engine);

// GPU-backed text engine (for SDL3 SDL_GPU API):
TTF_TextEngine *gpu_engine = TTF_CreateGPUTextEngine(gpu_device);
```

Rendering mode summary:

| Mode | Quality | Use case |
|---|---|---|
| `Solid` | 1-bit alpha, aliased | Frequently-updated debug text |
| `Shaded` | Anti-aliased, fixed background | UI text on known-colour backgrounds |
| `Blended` | Anti-aliased, true RGBA | Overlaid UI text, HUD |
| `LCD` | Subpixel RGB, fixed background | Crisp body text on LCD displays |
| `SDF` (font flag) | Resolution-independent | Large text, shader-scaled text |

---

## 10. SDL3 Camera, Dialogs, and New APIs

### 10.1 Camera API

SDL3 adds a camera API that abstracts V4L2 and PipeWire camera sources:

```c
// Enumerate cameras:
int count;
SDL_CameraID *cameras = SDL_GetCameras(&count);

SDL_CameraSpec desired = {SDL_PIXELFORMAT_YUYV, SDL_COLORSPACE_BT601_LIMITED, 640, 480, 30, 1};
SDL_Camera *cam = SDL_OpenCamera(cameras[0], &desired);
// spec = NULL → let SDL choose format

// Frame acquisition (non-blocking):
Uint64 ts_ns;
SDL_Surface *frame = SDL_AcquireCameraFrame(cam, &ts_ns);
if (frame) {
    // process frame->pixels (YUYV, or SDL_PIXELFORMAT_RGB24, etc.)
    SDL_ReleaseCameraFrame(cam, frame);
}

SDL_CloseCamera(cam);
SDL_free(cameras);
```

Permission approval fires `SDL_EVENT_CAMERA_DEVICE_APPROVED` or `SDL_EVENT_CAMERA_DEVICE_DENIED` asynchronously.

### 10.2 File Dialog API

SDL3 exposes native file dialogs via `xdg-desktop-portal` on Linux (Flatpak-safe):

```c
void file_chosen(void *userdata, const char * const *files, int filter_idx) {
    if (files && files[0]) {
        printf("Selected: %s\n", files[0]);
    }
}

SDL_DialogFileFilter filters[] = {
    {"PNG images", "png"},
    {"JPEG images", "jpg;jpeg"},
    {"All files",   "*"},
};

SDL_ShowOpenFileDialog(file_chosen, NULL, window, filters, 3, NULL, false);
```

### 10.3 Properties System

SDL3 replaces `SDL_QueryTexture`, `SDL_GetRendererInfo`, and similar info-querying functions with a unified properties system:

```c
SDL_PropertiesID props = SDL_GetRendererProperties(renderer);
const char *backend = SDL_GetStringProperty(props, SDL_PROP_RENDERER_NAME_STRING, "unknown");

SDL_PropertiesID tex_props = SDL_GetTextureProperties(texture);
float w = SDL_GetFloatProperty(tex_props, SDL_PROP_TEXTURE_WIDTH_NUMBER, 0);
float h = SDL_GetFloatProperty(tex_props, SDL_PROP_TEXTURE_HEIGHT_NUMBER, 0);
```

### 10.4 Main Callbacks (Event-Loop Inversion)

SDL3 offers an alternative entry point suitable for platforms that require control of the main loop (iOS, Emscripten). On Linux it is optional but useful for clean lifecycle management:

```c
#define SDL_MAIN_USE_CALLBACKS
#include <SDL3/SDL_main.h>

SDL_AppResult SDL_AppInit(void **appstate, int argc, char *argv[]) {
    // One-time initialisation; return SDL_APP_CONTINUE to proceed
    return SDL_APP_CONTINUE;
}

SDL_AppResult SDL_AppIterate(void *appstate) {
    // Called every frame; return SDL_APP_CONTINUE to continue
    return SDL_APP_CONTINUE;
}

SDL_AppResult SDL_AppEvent(void *appstate, SDL_Event *event) {
    if (event->type == SDL_EVENT_QUIT) return SDL_APP_SUCCESS;
    return SDL_APP_CONTINUE;
}

void SDL_AppQuit(void *appstate, SDL_AppResult result) {
    // Cleanup
}
```

---

## 11. SDL2 → SDL3 Migration Guide

SDL provides `sdl2-compat` ([github.com/libsdl-org/sdl2-compat](https://github.com/libsdl-org/sdl2-compat)), a shared library that implements the SDL2 ABI on top of SDL3. Existing SDL2 binaries can run against `sdl2-compat` without recompilation. Recompiling against SDL3 directly is strongly recommended for new code. Key changes:

| Area | SDL2 | SDL3 |
|---|---|---|
| Header | `#include <SDL.h>` | `#include <SDL3/SDL.h>` |
| Init return | Negative int = failure | `bool` false = failure |
| Error strings | `SDL_GetError()` always valid | Same |
| Event constants | `SDL_KEYDOWN`, `SDL_QUIT` | `SDL_EVENT_KEY_DOWN`, `SDL_EVENT_QUIT` |
| Window events | `SDL_WINDOWEVENT` + sub-type | Flat `SDL_EVENT_WINDOW_RESIZED`, … |
| Key event fields | `event.key.keysym.sym` | `event.key.key` (SDL_Keysym removed) |
| Mouse coords | `int` | `float` |
| Renderer create | `SDL_CreateRenderer(win, -1, flags)` | `SDL_CreateRenderer(win, name)` |
| Render copy | `SDL_RenderCopy()` | `SDL_RenderTexture()` |
| Render copy ex | `SDL_RenderCopyEx()` | `SDL_RenderTextureRotated()` |
| Rect types | `SDL_Rect` (int) | `SDL_FRect` (float) |
| Audio API | Callback, `SDL_OpenAudio()` | Stream, `SDL_OpenAudioDeviceStream()` |
| Env var: video | `SDL_VIDEODRIVER` | `SDL_VIDEO_DRIVER` |
| Env var: audio | `SDL_AUDIODRIVER` | `SDL_AUDIO_DRIVER` |
| Controller | `SDL_GameController`, `SDL_CONTROLLER_BUTTON_A` | `SDL_Gamepad`, `SDL_GAMEPAD_BUTTON_SOUTH` |
| `SDL_INIT_EVERYTHING` | Available | Removed |
| `SDL_INIT_TIMER` | Required for timers | No longer needed |
| Pixel format | `SDL_PIXELFORMAT_RGB888` | `SDL_PIXELFORMAT_XRGB8888` |
| Timestamps | Milliseconds | Nanoseconds (`SDL_GetTicksNS()`) |
| Text input | `SDL_StartTextInput()` (global) | `SDL_StartTextInput(window)` |
| Relative mouse | `SDL_SetRelativeMouseMode()` | `SDL_SetWindowRelativeMouseMode(window, …)` |
| Texture query | `SDL_QueryTexture()` | `SDL_GetTextureSize()` + properties |

The full migration guide is at `docs/README-migration.md` in the SDL source tree and at [wiki.libsdl.org/SDL3/MigrationGuide](https://wiki.libsdl.org/SDL3/MigrationGuide).

---

## 12. Building and Packaging SDL3 on Linux

### 12.1 Build from Source

```bash
git clone https://github.com/libsdl-org/SDL.git && cd SDL
cmake -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DSDL_STATIC=OFF \
    -DSDL_WAYLAND=ON \
    -DSDL_X11=ON \
    -DSDL_PIPEWIRE=ON \
    -DSDL_PULSEAUDIO=ON \
    -DSDL_ALSA=ON \
    -DSDL_JACK=ON \
    -DSDL_HIDAPI=ON \
    -DSDL_VULKAN=ON
cmake --build build -j$(nproc)
sudo cmake --install build
```

Wayland build requires: `libwayland-dev`, `wayland-protocols`, `libxkbcommon-dev`, `libdecor-0-dev`. PipeWire requires `libpipewire-0.3-dev`. HIDAPI requires `libhidapi-dev` or builds its own from source.

### 12.2 Linking Applications

```cmake
# CMake (recommended):
find_package(SDL3 REQUIRED CONFIG REQUIRED COMPONENTS SDL3)
target_link_libraries(myapp PRIVATE SDL3::SDL3)

# Also link satellite libraries if used:
find_package(SDL3_image REQUIRED CONFIG REQUIRED COMPONENTS SDL3_image)
target_link_libraries(myapp PRIVATE SDL3_image::SDL3_image)
```

```bash
# pkg-config (for Makefiles):
pkg-config --cflags --libs sdl3
# Output: -I/usr/include -lSDL3
```

### 12.3 Diagnosing Backend Issues

```bash
# Check which video/audio driver SDL selected:
SDL_VIDEO_DRIVER=wayland SDL_AUDIO_DRIVER=pipewire \
  SDL_RENDER_DRIVER=vulkan ./myapp

# Verbose subsystem output:
SDL_DEBUG_BACKEND=1 ./myapp   # SDL3

# Check Wayland socket:
echo $WAYLAND_DISPLAY   # should be wayland-0 or wayland-1

# Check PipeWire:
pactl info | grep "Server Name"  # "PulseAudio (on PipeWire ...)"

# Force XWayland for an SDL2 app via sdl2-compat:
SDL_VIDEODRIVER=x11 ./legacy-sdl2-app

# Query available renderers:
SDL_SetHint(SDL_HINT_RENDER_DRIVER, "software");  # guaranteed fallback
```

### 12.4 Headless / CI Testing

```bash
# Offscreen video driver (no display required):
SDL_VIDEO_DRIVER=offscreen SDL_AUDIO_DRIVER=dummy ./myapp

# KMS/DRM (direct display, no compositor — e.g., embedded, kiosk):
SDL_VIDEO_DRIVER=kmsdrm ./myapp
```

The `offscreen` driver creates a framebuffer in system memory but no window. The `kmsdrm` driver opens a DRM device directly via `drmOpen()` and allocates GBM buffers for scanout — useful for kiosk systems running without a compositor.

---

## Integrations

**Chapter 39 — Qt and GTK GPU Rendering** covers SDL's counterpart toolkit libraries. Qt's `QRhi` and GTK4's `GskRenderer` provide widget-centric GPU rendering; SDL provides a lower-level integration layer for applications that manage their own UI. All three converge on the same Mesa Vulkan/OpenGL drivers and the same Wayland surface model.

**Chapter 38 — PipeWire and the Video Session Layer** is the audio session layer that SDL3's PipeWire backend connects to. SDL submits audio streams as PipeWire graph nodes; the PipeWire session manager (wireplumber) routes them to the hardware sink. The `SDL_AUDIO_DRIVER=pipewire` backend uses `libpipewire-0.3` directly.

**Chapter 38b — ALSA** is the fallback audio layer when PipeWire is not available. SDL's ALSA backend uses `libasound` PCM directly, bypassing the PipeWire/PulseAudio session layer.

**Chapter 20 — Wayland Protocol Fundamentals** covers the xdg-shell, `zwp_relative_pointer_v1`, `zwp_pointer_constraints_v1`, and `zwp_text_input_v3` protocols that SDL3's Wayland backend implements. Understanding the Wayland protocol stack explains SDL Wayland limitations (no global mouse position, no window placement, no global shortcuts).

**Chapter 18 — Mesa Vulkan Drivers** is what SDL's `SDL_Vulkan_CreateSurface` delivers surfaces to. SDL provides the `VkSurfaceKHR`; Mesa's ANV/RADV/NVK drivers implement the swapchain on top.

**Chapter 24 — Vulkan and EGL for Application Developers** covers the Vulkan and EGL APIs that SDL exposes via `SDL_Vulkan_CreateSurface` and `SDL_GL_CreateContext`. SDL abstracts the platform-specific surface creation; the Vulkan programming model is covered there.

**Chapter 81 — SDL3 GPU API** covers SDL's modern high-level GPU API (`SDL_GPUDevice`, compute passes, graphics pipelines) which sits above the SDL_Renderer 2D layer and complements the multimedia integration described in this chapter.

**Chapter 111 — Flatpak Graphics** covers GPU access in the Flatpak sandbox. SDL3 applications in Flatpak access the GPU via `/dev/dri/renderD*` (granted via `--device=dri`) and use `xdg-desktop-portal` for file dialogs (SDL3's `SDL_ShowOpenFileDialog`) and camera access.

---

## References

- [SDL3 — libsdl.org](https://libsdl.org) — Official SDL homepage and release announcements
- [SDL3 Wiki — Front Page](https://wiki.libsdl.org/SDL3/FrontPage) — API reference, category index
- [SDL3 Migration Guide (README-migration.md)](https://github.com/libsdl-org/SDL/blob/main/docs/README-migration.md) — Comprehensive SDL2→SDL3 API change list
- [SDL3.2.0 Release Announcement — January 2025](https://discourse.libsdl.org/t/announcing-the-sdl-3-official-release/57149) — First stable SDL3 release
- [SDL3 README-wayland](https://wiki.libsdl.org/SDL3/README-wayland) — Wayland backend capabilities and limitations
- [SDL3 README-highdpi](https://wiki.libsdl.org/SDL3/README-highdpi) — HiDPI and display scale documentation
- [SDL3 CategoryAudio](https://wiki.libsdl.org/SDL3/CategoryAudio) — Audio API reference
- [SDL3 CategoryGamepad](https://wiki.libsdl.org/SDL3/CategoryGamepad) — Gamepad API reference
- [SDL3 CategoryHaptic](https://wiki.libsdl.org/SDL3/CategoryHaptic) — Haptic / force-feedback API
- [SDL3 CategorySensor](https://wiki.libsdl.org/SDL3/CategorySensor) — Accelerometer / gyroscope API
- [SDL3 CategoryVulkan](https://wiki.libsdl.org/SDL3/CategoryVulkan) — Vulkan surface creation
- [SDL3 CategoryRender](https://wiki.libsdl.org/SDL3/CategoryRender) — 2D renderer API
- [SDL3 NewFeatures](https://wiki.libsdl.org/SDL3/NewFeatures) — New SDL3 APIs not present in SDL2
- [GitHub — libsdl-org/SDL](https://github.com/libsdl-org/SDL) — SDL3 source repository
- [GitHub — libsdl-org/sdl2-compat](https://github.com/libsdl-org/sdl2-compat) — SDL2 ABI compatibility layer on SDL3
- [GitHub — libsdl-org/SDL_image](https://github.com/libsdl-org/SDL_image) — SDL3_image satellite library
- [GitHub — libsdl-org/SDL_mixer](https://github.com/libsdl-org/SDL_mixer) — SDL3_mixer satellite library
- [GitHub — libsdl-org/SDL_ttf](https://github.com/libsdl-org/SDL_ttf) — SDL3_ttf satellite library (FreeType + HarfBuzz)
- [SDL3 PipeWire preference — Phoronix](https://www.phoronix.com/news/SDL-3.0-Prefer-PipeWire) — SDL3 prefers PipeWire over PulseAudio
- [SDL Steam Controller HIDAPI — Phoronix](https://www.phoronix.com/news/SDL-Steam-Controller-Default) — HIDAPI Steam Controller enabled by default (November 2024)
