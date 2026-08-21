# Chapter 202: Vulkan WSI Deep Dive

> **Part**: Part VII-A — GPU APIs
> **Audience**: Primarily graphics application developers integrating Vulkan presentation correctly on Linux, and systems/driver developers working on compositor or display-server presentation paths.
> **Status**: First draft — 2026-08-21

Vulkan defines rendering with no notion of a screen. Everything that gets a rendered image in front of a user — surface creation, format negotiation, buffer handoff to a compositor or display server, frame pacing, tear control — lives in a family of optional, platform-specific extensions collectively called the Window System Integration (WSI) layer. On Linux this layer has three genuinely different backends (Wayland, X11/XCB, and direct-to-KMS display) plus a headless no-op backend, and Mesa implements all of them in a single shared component, `src/vulkan/wsi/`, that every Mesa Vulkan driver (RADV, ANV, NVK, Turnip, PanVK, v3dv, Honeykrisp) links against. This chapter works through that shared layer end to end: the platform-agnostic surface and swapchain API, the Wayland and X11 presentation paths and what actually crosses the wire to the compositor or X server, the present-mode and frame-pacing extensions that have landed in the last two Vulkan cycles, direct display and headless presentation, and HDR swapchains. Chapter 24 and Chapter 150 cover the same territory from the EGL side and from the application-developer's initialization sequence; this chapter assumes that context and goes deeper into the WSI-specific mechanics and the Mesa internals that implement them.

---

## Table of Contents

- [1. WSI Abstraction Overview](#1-wsi-abstraction-overview)
  - [1.1 `VK_KHR_surface` as the Platform-Agnostic Handle](#11-vk_khr_surface-as-the-platform-agnostic-handle)
  - [1.2 Capability, Format, and Present-Mode Queries](#12-capability-format-and-present-mode-queries)
- [2. The Wayland WSI Path](#2-the-wayland-wsi-path)
  - [2.1 `VK_KHR_wayland_surface` and Surface Creation](#21-vk_khr_wayland_surface-and-surface-creation)
  - [2.2 Mesa's `wsi_common_wayland.c`](#22-mesas-wsi_common_waylandc)
  - [2.3 Modifier Negotiation via `zwp_linux_dmabuf_v1` Feedback](#23-modifier-negotiation-via-zwp_linux_dmabuf_v1-feedback)
  - [2.4 From `vkQueuePresentKHR` to `drmModeAtomicCommit`](#24-from-vkqueuepresentkhr-to-drmmodeatomiccommit)
  - [2.5 Explicit Sync via `wp_linux_drm_syncobj_v1`](#25-explicit-sync-via-wp_linux_drm_syncobj_v1)
- [3. The X11/XCB WSI Path](#3-the-x11xcb-wsi-path)
  - [3.1 `VK_KHR_xcb_surface` and `VK_KHR_xlib_surface`](#31-vk_khr_xcb_surface-and-vk_khr_xlib_surface)
  - [3.2 DRI3 and the Present Extension](#32-dri3-and-the-present-extension)
  - [3.3 Redirected vs. Direct Present, and Glamor](#33-redirected-vs-direct-present-and-glamor)
  - [3.4 HiDPI Under XWayland](#34-hidpi-under-xwayland)
- [4. Present Modes in Depth](#4-present-modes-in-depth)
  - [4.1 FIFO and FIFO_RELAXED](#41-fifo-and-fifo_relaxed)
  - [4.2 MAILBOX and IMMEDIATE](#42-mailbox-and-immediate)
  - [4.3 Per-Driver and Per-Platform Support](#43-per-driver-and-per-platform-support)
- [5. Swapchain Creation](#5-swapchain-creation)
  - [5.1 `VkSwapchainCreateInfoKHR` Field by Field](#51-vkswapchaincreateinfokhr-field-by-field)
  - [5.2 `VK_SWAPCHAIN_CREATE_MUTABLE_FORMAT_BIT_KHR`](#52-vk_swapchain_create_mutable_format_bit_khr)
  - [5.3 Old Swapchain Recycling](#53-old-swapchain-recycling)
- [6. Image Acquisition and Presentation](#6-image-acquisition-and-presentation)
  - [6.1 `vkAcquireNextImage2KHR`](#61-vkacquirenextimage2khr)
  - [6.2 `vkQueuePresentKHR` and `VkPresentInfoKHR`](#62-vkqueuepresentkhr-and-vkpresentinfokhr)
  - [6.3 `VK_KHR_present_id` and `VK_KHR_present_wait`](#63-vk_khr_present_id-and-vk_khr_present_wait)
- [7. `VK_EXT_swapchain_maintenance1`](#7-vk_ext_swapchain_maintenance1)
- [8. `VK_EXT_present_timing`](#8-vk_ext_present_timing)
- [9. Direct Display](#9-direct-display)
  - [9.1 `VK_KHR_display`](#91-vk_khr_display)
  - [9.2 `VK_EXT_acquire_drm_display`](#92-vk_ext_acquire_drm_display)
- [10. Headless WSI](#10-headless-wsi)
- [11. HDR Swapchains](#11-hdr-swapchains)
- [12. Mesa WSI Common Layer Architecture Summary](#12-mesa-wsi-common-layer-architecture-summary)
- [Integrations](#integrations)
- [References](#references)

---

## 1. WSI Abstraction Overview

### 1.1 `VK_KHR_surface` as the Platform-Agnostic Handle

`VK_KHR_surface` is the instance-level extension that introduces `VkSurfaceKHR`, an opaque handle representing "an abstract type of surface to present rendered images to." It carries no platform-specific data itself — every field that references a native window, display connection, or `wl_surface` lives in the platform-specific creation extension that produces the handle (`VK_KHR_wayland_surface`, `VK_KHR_xcb_surface`, `VK_KHR_xlib_surface`, `VK_KHR_display`, `VK_EXT_headless_surface`, and so on) [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_surface.html). Every subsequent swapchain operation — capability queries, format enumeration, present-mode enumeration, and swapchain creation itself via `VK_KHR_swapchain` — takes this handle, not the platform object, so application code that separates platform setup from rendering can treat `VkSurfaceKHR` uniformly once it exists.

A `VkSurfaceKHR` is also tied to a specific `VkInstance` and is queried per-`VkPhysicalDevice`; a given physical device and queue family may or may not support presentation to a given surface, which is why every correct Vulkan application calls `vkGetPhysicalDeviceSurfaceSupportKHR` (or the platform-specific `vkGetPhysicalDeviceWaylandPresentationSupportKHR` / `vkGetPhysicalDeviceXcbPresentationSupportKHR` predicate, which can be called before a surface even exists) as part of queue family selection.

### 1.2 Capability, Format, and Present-Mode Queries

Three queries must run before `vkCreateSwapchainKHR` is called, and their results constrain every field of `VkSwapchainCreateInfoKHR`:

- **`vkGetPhysicalDeviceSurfaceCapabilitiesKHR`** returns a `VkSurfaceCapabilitiesKHR` with `minImageCount`/`maxImageCount`, `currentExtent`/`minImageExtent`/`maxImageExtent`, `supportedTransforms` and `currentTransform`, `supportedCompositeAlpha`, and `supportedUsageFlags`. On Wayland, `currentExtent` is commonly reported as `(0xFFFFFFFF, 0xFFFFFFFF)` — meaning "the surface size is determined by the swapchain extent the application requests" — because Wayland has no independent concept of window geometry until the client commits a buffer.
- **`vkGetPhysicalDeviceSurfaceFormatsKHR`** enumerates `VkSurfaceFormatKHR` pairs of `VkFormat` and `VkColorSpaceKHR`. The color-space list is where HDR and wide-gamut swapchains announce themselves (§11).
- **`vkGetPhysicalDeviceSurfacePresentModesKHR`** enumerates the `VkPresentModeKHR` values the platform/driver combination supports for that surface (§4).

[Source: VK_KHR_surface man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_surface.html)

---

## 2. The Wayland WSI Path

### 2.1 `VK_KHR_wayland_surface` and Surface Creation

`VK_KHR_wayland_surface` supplies `vkCreateWaylandSurfaceKHR`, which takes a `VkWaylandSurfaceCreateInfoKHR`:

```c
typedef struct VkWaylandSurfaceCreateInfoKHR {
    VkStructureType                sType;
    const void*                    pNext;
    VkWaylandSurfaceCreateFlagsKHR flags;
    struct wl_display*             display;
    struct wl_surface*             surface;
} VkWaylandSurfaceCreateInfoKHR;
```

The application owns `wl_display` and `wl_surface` — Vulkan does not create either; it wraps a client-side connection and role-less surface that the application already established via `libwayland-client` (typically through a toolkit). The extension's own specification text mandates that Wayland surfaces report `VK_PRESENT_MODE_MAILBOX_KHR` as a supported mode, because "Wayland is an inherently mailbox window system" [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_wayland_surface.html).

### 2.2 Mesa's `wsi_common_wayland.c`

Mesa implements the Wayland backend once, shared by every Mesa Vulkan driver, in `src/vulkan/wsi/wsi_common_wayland.c`. It is the largest of the per-platform WSI files and does most of the Wayland-specific work described in this section: binding to `wl_registry` globals (`wl_compositor`, `zwp_linux_dmabuf_v1`, `wp_presentation`, `wp_linux_drm_syncobj_manager_v1`, `wp_tearing_control_manager_v1`), driving the dmabuf-feedback modifier negotiation, and creating the per-image `wl_buffer` objects that get attached and committed to the surface on present [Source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common_wayland.c). The lower-level DMA-BUF image allocation, explicit/implicit sync setup, and modifier filtering that this file and the X11 backend both call into live in a separate shared file, `src/vulkan/wsi/wsi_common_drm.c` — its exported entry points (`wsi_drm_configure_image`, `wsi_create_image_explicit_sync_drm`, `wsi_drm_init_swapchain_implicit_sync`) are platform-agnostic DRM/DMA-BUF plumbing, not a VK_KHR_display implementation (see §9 for where the display extensions actually live).

### 2.3 Modifier Negotiation via `zwp_linux_dmabuf_v1` Feedback

A Wayland swapchain image is a GPU buffer with a specific memory layout — described by a DRM format modifier — and the compositor's display/composition hardware may only accept a subset of the modifiers the GPU driver can produce. `zwp_linux_dmabuf_v1` version 4 added the *dmabuf-feedback* mechanism specifically to solve this negotiation: instead of a flat, connection-wide modifier list, a client can request `wl_surface`-scoped feedback via `get_surface_feedback`, and the compositor replies with `tranche` events, each carrying a `format_table` (fourcc + modifier pairs) and a `tranche_target_device` (the DRM device the compositor prefers those formats for — its scanout device, or its renderer if it will composite the buffer). Mesa's Wayland WSI backend consumes this feedback to restrict the modifier list it passes into `gbm_bo_create_with_modifiers2` (§12) when allocating swapchain images, preferring modifiers the compositor can scan out directly over ones that force a copy [Source: linux-dmabuf-v1 protocol](https://wayland.app/protocols/linux-dmabuf-v1). Where no feedback is available (an older compositor, or the client not requesting it), Mesa falls back to the connection-wide `zwp_linux_dmabuf_v1.modifier` event list advertised at binding time.

### 2.4 From `vkQueuePresentKHR` to `drmModeAtomicCommit`

The Wayland presentation path has three distinct hops, and understanding where each one happens is the difference between debugging in the application, in Mesa, or in the compositor:

1. **Application → compositor (Wayland protocol).** `vkQueuePresentKHR` on a Wayland swapchain image causes Mesa to call `wl_surface_attach` with the `wl_buffer` backing that image (created from the DMA-BUF at swapchain-creation time), `wl_surface_damage_buffer` for the full extent, and `wl_surface_commit`. This is ordinary client-side Wayland protocol traffic — no kernel ioctl fires yet.
2. **Compositor import.** The compositor (a Wayland client library on the server side of the protocol, itself a DRM/KMS client) receives the buffer commit, imports the DMA-BUF file descriptor as a GEM object via `DRM_IOCTL_PRIME_FD_TO_HANDLE`, and either composites it (if the surface is not fullscreen/does not qualify for direct scanout) or treats it as a scanout candidate.
3. **Compositor → kernel (KMS).** If the compositor's scene graph resolves to a scanout-eligible configuration — most commonly a single fullscreen, unredirected surface whose format/modifier the display plane accepts — the compositor issues `drmModeAtomicCommit` with that GEM buffer's framebuffer object bound directly to a hardware plane, bypassing GPU composition entirely for that frame. Otherwise the compositor's own renderer (its GPU context, potentially reusing the same Mesa WSI machinery) composites the buffer into a scene and *that* composited frame is what reaches `drmModeAtomicCommit`.

Direct scanout is a compositor policy decision made per-frame, not something the Vulkan application or the WSI layer can request explicitly; applications can only make it more likely by presenting a buffer whose format, modifier, and geometry match what the compositor's scanout hardware and current mode accept.

### 2.5 Explicit Sync via `wp_linux_drm_syncobj_v1`

`wp_linux_drm_syncobj_v1` is a Wayland protocol extension — stable as of `linux-drm-syncobj-v1` — that lets a client attach explicit DRM sync-object timeline points to a surface's buffer acquire and release, superseding the older implicit-fencing model where the kernel's DMA-BUF reservation object carried fences attached out-of-band [Source](https://wayland.app/protocols/linux-drm-syncobj-v1). A client requests a `wp_linux_drm_syncobj_surface_v1` from the global `wp_linux_drm_syncobj_manager_v1`, then on each `wl_surface_commit` calls `set_acquire_point` and `set_release_point` with a `wp_linux_drm_syncobj_timeline_v1` (wrapping a kernel `drm_syncobj`) and a 64-bit timeline point each. The compositor will not read the committed buffer's contents until the GPU signals the acquire point, and will not signal the release point back until it has finished reading — replacing what used to be an implicit fence race with an explicit, driver-independent handshake.

Mesa's Vulkan WSI Wayland backend binds this protocol when present and maps it onto Vulkan's own explicit-sync primitives: a swapchain image's acquire semaphore (a Vulkan timeline semaphore backed by a `drm_syncobj` via `VK_KHR_external_semaphore_fd`) becomes the release point the compositor waits on, and the present-complete signal becomes the acquire point the application's next frame waits on. This is the same `drm_syncobj` timeline mechanism used by KMS's own explicit-sync uAPI (`IN_FENCE_FD`/`OUT_FENCE_PTR` properties), which Chapter 75 covers end to end from the kernel side; this section is specifically the Wayland-protocol leg of that same timeline crossing the client/compositor boundary.

---

## 3. The X11/XCB WSI Path

### 3.1 `VK_KHR_xcb_surface` and `VK_KHR_xlib_surface`

`VK_KHR_xcb_surface` provides `vkCreateXcbSurfaceKHR` taking a `VkXcbSurfaceCreateInfoKHR` (`xcb_connection_t*` plus `xcb_window_t`); `VK_KHR_xlib_surface` provides the Xlib-typed equivalent (`Display*` plus `Window`). Both are thin wrappers — Mesa's XCB and Xlib code paths converge on the same underlying implementation in `src/vulkan/wsi/wsi_common_x11.c`, since Xlib itself is implemented in terms of XCB on Linux. Toolkits that already hold an XCB connection (most GTK and Qt applications under X11/XWayland) use `VK_KHR_xcb_surface` directly; Xlib-only codebases use `VK_KHR_xlib_surface`.

### 3.2 DRI3 and the Present Extension

The mechanism underneath both surface types is DRI3 combined with the X Present extension. DRI3 provides the buffer-passing primitives — `xcb_dri3_pixmap_from_buffers` (or the older single-plane `xcb_dri3_pixmap_from_buffer`) wraps a client-side DMA-BUF file descriptor as a server-side X `Pixmap`, transferred over the X protocol connection via ordinary POSIX FD-passing (`SCM_RIGHTS` over the local socket) rather than a data copy [Source: DRI3 protocol](https://cgit.freedesktop.org/xorg/proto/dri3proto/plain/dri3proto.txt). Present then does the actual display-side work: `xcb_present_pixmap` schedules that pixmap to become the window's visible contents, optionally synchronized to a target MSC (media stream counter, the X server's vblank-driven frame counter) and carrying `xcb_sync_fence_t` objects for completion signaling — the X11-protocol analogue of the Wayland `frame` callback and explicit-sync fences described in §2.5. Present's design goal, stated in its original protocol proposal, was exactly this: "a complete direct rendering solution" that gets a client-rendered buffer onto the screen at a specific vblank without a compositing round-trip when direct presentation is possible [Source](https://keithp.com/blogs/dri3_extension/).

### 3.3 Redirected vs. Direct Present, and Glamor

Whether a `xcb_present_pixmap` call actually reaches the display plane depends on whether the target window is redirected by a compositing manager. An unredirected, fullscreen window (the classic case: a game or video player with no compositor running, or one running under a compositor that unredirects fullscreen clients) gets **direct present**: the X server's DDX driver can flip the pixmap onto the CRTC directly, equivalent in spirit to the Wayland direct-scanout path in §2.4. A redirected window — the normal case under any running compositing window manager — gets its Present updates composited into the redirected off-screen pixmap first, which the compositing manager's own render loop then presents; this adds at least one composition pass and typically one frame of latency versus direct present.

Mesa's X11 WSI backend runs on top of **Glamor**, the GL-based 2D acceleration layer that most current X server DDX drivers (`xf86-video-amdgpu`, `xf86-video-ati`, the modesetting driver) use for their own hardware-accelerated 2D operations and DRI3 pixmap backing. This matters mainly as an architectural note: the X server's own rendering (Present's copy path when direct present is unavailable, and core X drawing) shares GPU context/driver infrastructure with the Vulkan client through Glamor, rather than routing through a separate software path.

### 3.4 HiDPI Under XWayland

XWayland — an embedded X server translating X11 protocol into Wayland client calls — presents Vulkan XCB/Xlib surfaces to a Wayland compositor exactly as described in §2.4, one layer removed: the Vulkan application talks DRI3/Present to XWayland, and XWayland itself is the Wayland client that attaches/commits `wl_buffer`s to the compositor. HiDPI handling under this arrangement is a known rough edge: X11 has no native fractional-scale-per-output geometry model, so XWayland historically rounds an application's window to whichever integer scale factor its output falls under (or applies a global scale), which routinely produces blurry or incorrectly-sized Vulkan swapchain images for XWayland clients on outputs a compositor has scaled fractionally. Applications that need pixel-perfect HiDPI presentation on Wayland outputs should use the native `VK_KHR_wayland_surface` path rather than XWayland where the toolkit supports it; this is one of the concrete reasons the WSI layer offers both paths side by side rather than treating X11-via-XWayland as a universal compatibility shim.

---

## 4. Present Modes in Depth

### 4.1 FIFO and FIFO_RELAXED

`VK_PRESENT_MODE_FIFO_KHR` queues presentation requests in a FIFO and the display consumes one per vertical blanking period; if the queue is full, `vkQueuePresentKHR` blocks. This is the only present mode every Vulkan implementation is required to support, and it is the direct analogue of traditional double/triple-buffered vsync: no tearing, but a full frame of latency can accumulate if the application cannot keep pace with the display's refresh rate. `VK_PRESENT_MODE_FIFO_RELAXED_KHR` behaves identically except when the application's frame arrives *after* the target vblank has already passed — in that one case the presentation engine does not wait for the next vblank and presents immediately, trading a single visible tear for avoiding an extra frame of added latency when the application is already running behind.

### 4.2 MAILBOX and IMMEDIATE

`VK_PRESENT_MODE_MAILBOX_KHR` also waits for vblank to actually present, but the queue depth is effectively one: a newly submitted image *replaces* whatever image was previously waiting, rather than queuing behind it, so `vkQueuePresentKHR` never blocks on a full queue and the display always shows the most recently completed frame at the next vblank. This is the standard low-latency triple-buffering mode — no tearing, minimal added latency, at the cost of GPU work done on discarded frames when the application renders faster than the display refreshes. `VK_PRESENT_MODE_IMMEDIATE_KHR` presents each image the moment it is submitted with no vblank synchronization at all, which can tear but minimizes latency and lets the application's frame rate run uncapped.

### 4.3 Per-Driver and Per-Platform Support

FIFO is universal by specification requirement. `VK_PRESENT_MODE_MAILBOX_KHR` is a *mandatory* mode specifically for Wayland surfaces — the `VK_KHR_wayland_surface` specification requires `vkGetPhysicalDeviceSurfacePresentModesKHR` to report it, on the reasoning that Wayland's own buffer-submission model is inherently mailbox-shaped [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_wayland_surface.html); Mesa's Wayland WSI backend implements this by holding at most one pending image and replacing it on each new present. `VK_PRESENT_MODE_IMMEDIATE_KHR` has no such mandate on Wayland precisely because tearing requires cooperation from the compositor: Mesa's Wayland backend only advertises `IMMEDIATE` when the compositor implements `wp_tearing_control_manager_v1`, the tearing-control protocol extension that lets a client mark specific buffer commits as tear-allowed so the compositor can route them straight to a scanout plane without waiting on the next full vblank interval [Source](https://www.phoronix.com/news/Mesa-VLK-WSI-Wayland-IMMEDIATE). Compositors that do not implement tearing-control simply never expose `IMMEDIATE` to Wayland clients, and Mesa's WSI layer reports the mode as unsupported rather than approximating it. Under X11/XCB, DRI3+Present has always supported an explicit no-sync present option (`XCB_PRESENT_OPTION_ASYNC` in `xcb_present_pixmap`), so `IMMEDIATE` availability there depends only on the X server and DDX driver rather than a separate protocol negotiation. Direct-display swapchains (§9) are unconstrained by any compositor and generally expose the fullest present-mode set, including `IMMEDIATE`, since the driver owns the CRTC outright.

Present-mode availability on all three backends is otherwise a driver/Mesa-version question rather than a hardware one: RADV, ANV, and NVK all route through the same shared `wsi_common_wayland.c`/`wsi_common_x11.c` code, so a mode's availability tracks the WSI-layer feature (e.g., tearing-control wiring, `VK_GOOGLE_display_timing` opt-in registration) landing in a given Mesa release rather than diverging per hardware vendor the way, say, ray-tracing feature levels do.

---

## 5. Swapchain Creation

### 5.1 `VkSwapchainCreateInfoKHR` Field by Field

```c
typedef struct VkSwapchainCreateInfoKHR {
    VkStructureType                  sType;
    const void*                      pNext;
    VkSwapchainCreateFlagsKHR        flags;
    VkSurfaceKHR                     surface;
    uint32_t                         minImageCount;
    VkFormat                         imageFormat;
    VkColorSpaceKHR                  imageColorSpace;
    VkExtent2D                       imageExtent;
    uint32_t                         imageArrayLayers;
    VkImageUsageFlags                imageUsage;
    VkSharingMode                    imageSharingMode;
    uint32_t                         queueFamilyIndexCount;
    const uint32_t*                  pQueueFamilyIndices;
    VkSurfaceTransformFlagBitsKHR    preTransform;
    VkCompositeAlphaFlagBitsKHR      compositeAlpha;
    VkPresentModeKHR                 presentMode;
    VkBool32                         clipped;
    VkSwapchainKHR                   oldSwapchain;
} VkSwapchainCreateInfoKHR;
```

[Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkSwapchainCreateInfoKHR.html)

`minImageCount` must fall within the `[minImageCount, maxImageCount]` range from the capabilities query (`maxImageCount == 0` means unbounded); `imageFormat`/`imageColorSpace` must be a pair returned by the formats query; `imageExtent` must respect `minImageExtent`/`maxImageExtent`, and on Wayland is typically the application's own choice since `currentExtent` is unconstrained there (§1.2). `compositeAlpha` selects how the presentation engine treats the swapchain's own alpha channel against whatever is behind it: `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR` (the alpha channel is ignored, the standard case for opaque application windows) versus `VK_COMPOSITE_ALPHA_PRE_MULTIPLIED_BIT_KHR` / `VK_COMPOSITE_ALPHA_POST_MULTIPLIED_BIT_KHR` (the surface is treated as semi-transparent, letting the compositor blend it against the desktop — relevant for translucent overlay windows, and only meaningful when `supportedCompositeAlpha` from the capabilities query actually includes it, which Wayland compositors expose per their support for `wl_surface` alpha blending).

### 5.2 `VK_SWAPCHAIN_CREATE_MUTABLE_FORMAT_BIT_KHR`

Set in `flags` together with a `VkImageFormatListCreateInfoKHR` chained to `pNext`, `VK_SWAPCHAIN_CREATE_MUTABLE_FORMAT_BIT_KHR` allows swapchain images to be reinterpreted through an image view of a different but format-list-compatible `VkFormat` than the one the swapchain was created with — most commonly, creating an sRGB swapchain but rendering through a `VK_FORMAT_*_UNORM` view, or vice versa, so a shader can write linear values while the presentation engine still reads/composites the sRGB-encoded result [Source: VK_KHR_swapchain_mutable_format](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_swapchain_mutable_format.html). Without this flag, a swapchain image's format is fixed for the image's lifetime and only `VK_FORMAT_*_SRGB`/`VK_FORMAT_*_UNORM` pairs of the identical bit layout may even be requested through the format-list mechanism.

### 5.3 Old Swapchain Recycling

`oldSwapchain` exists so that resizing or reconfiguring presentation (typically on `VK_ERROR_OUT_OF_DATE_KHR` from `vkQueuePresentKHR`, or a Wayland `xdg_surface.configure`/X11 `ConfigureNotify` resize) does not require tearing down the surface and losing in-flight images. The application passes the existing `VkSwapchainKHR` as `oldSwapchain` when creating the replacement; the implementation is permitted (though not required) to reuse the old swapchain's already-allocated resources for the new one, and the old handle becomes retired — its images may still be legitimately in flight for presentation, but no new images may be acquired from it. The application must still explicitly destroy the retired `oldSwapchain` once it has finished with any images acquired from it before the replacement was created; recycling avoids reallocation cost, it does not avoid the destroy call.

---

## 6. Image Acquisition and Presentation

### 6.1 `vkAcquireNextImage2KHR`

```c
VkResult vkAcquireNextImage2KHR(
    VkDevice                        device,
    const VkAcquireNextImageInfoKHR* pAcquireInfo,
    uint32_t*                       pImageIndex);
```

`VkAcquireNextImageInfoKHR` extends the original `vkAcquireNextImageKHR` with a `deviceMask` field, needed because device groups (`VK_KHR_device_group`) require the caller to specify which physical devices in the group the acquired image should be made available on — a single-GPU application has no reason to prefer `vkAcquireNextImage2KHR` over the original entry point beyond forward-compatibility, but multi-GPU (SLI/Crossfire-style, or more commonly today, discrete + integrated hybrid) configurations must use it [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/vkAcquireNextImage2KHR.html). Acquisition takes an optional `VkSemaphore` and/or `VkFence` to signal when the acquired image is actually available for the application to begin writing to — critical on Wayland/X11, where an image just presented may still be held by the compositor/X server for some time after `vkQueuePresentKHR` returns.

### 6.2 `vkQueuePresentKHR` and `VkPresentInfoKHR`

`VkPresentInfoKHR` carries the wait semaphores (typically the semaphore signaled by the queue submission that finished rendering into the image), the swapchain(s) and image index(es) being presented — presentation supports submitting to multiple swapchains in one call for multi-window applications — and an optional `pResults` array to get a per-swapchain `VkResult` back rather than only the aggregate result. `VkSwapchainPresentFenceInfoEXT`, added by `VK_EXT_swapchain_maintenance1` (§7), chains onto `VkPresentInfoKHR` to attach a `VkFence` per swapchain that the implementation signals once that swapchain's presentation resources may safely be reused or destroyed — a capability the base `VK_KHR_swapchain` extension conspicuously lacks, forcing applications without it to either over-allocate images or rely on the coarser `vkDeviceWaitIdle`.

### 6.3 `VK_KHR_present_id` and `VK_KHR_present_wait`

`VK_KHR_present_id` (dating to 2019, alongside `VK_KHR_present_wait`) adds a `VkPresentIdKHR` structure chained to `VkPresentInfoKHR` carrying a monotonically increasing, application-assigned 64-bit `presentIds[]` value per swapchain in the present call. `VK_KHR_present_wait` then adds `vkWaitForPresentKHR(device, swapchain, presentId, timeout)`, which blocks until the implementation confirms that the present operation carrying that ID (or a later one) has actually completed on the display — as opposed to merely having been submitted to the presentation engine's queue [Source](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_present_wait.html). This is the mechanism that lets an application implement its own frame-pacing loop against *actual* display completion rather than against `vkQueuePresentKHR` return, which on a mailbox/FIFO queue only indicates the request was accepted, not that it reached the screen. `VK_KHR_present_wait2`/`VK_KHR_present_id2` are newer, spec-cleanup revisions of the same mechanism and are not treated as a separate mechanism here.

---

## 7. `VK_EXT_swapchain_maintenance1`

`VK_EXT_swapchain_maintenance1` (later folded verbatim into `VK_KHR_swapchain_maintenance1` — "all functionality in this extension is included in `VK_KHR_swapchain_maintenance1`, with the suffix changed to KHR" [Source](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_swapchain_maintenance1.adoc)) bundles a set of small, previously-missing WSI capabilities:

- **`VkSwapchainPresentModeInfoEXT`**, chained onto `VkPresentInfoKHR`, lets a single present call switch the effective present mode for that frame from a set the application declared compatible at swapchain-creation time — via `VkSwapchainPresentModesCreateInfoEXT` — without recreating the swapchain. An application can, for instance, present with `FIFO` while idle and switch to `MAILBOX` under load, per frame, with no swapchain teardown.
- **`VkSwapchainPresentScalingCreateInfoEXT`** lets an application declare how the presentation engine should handle a swapchain image whose extent no longer matches the surface's current size — stretch, letterbox/pillarbox, or clip — which lets implementations avoid handing back `VK_ERROR_OUT_OF_DATE_KHR` (and thus a mandatory swapchain recreation) purely because the window was resized by one pixel.
- **`VkSwapchainPresentFenceInfoEXT`** (§6.2) plus **`VkReleaseSwapchainImagesInfoEXT`** (via `vkReleaseSwapchainImagesEXT`) together give an application deferred, explicit control over swapchain resource lifetime: the fence tells the application when a presented image's resources are safe to reuse or the swapchain safe to destroy, and `vkReleaseSwapchainImagesEXT` lets an application give back an *acquired-but-never-presented* image (one it decided mid-frame not to use) without going through a full present cycle.

Together these close a long-standing WSI gap: prior to this extension, safely destroying or resizing a swapchain generally required an application-side `vkDeviceWaitIdle`, which stalls the entire device rather than just the resources involved in that one swapchain. `VK_EXT_swapchain_maintenance1` was developed with NVIDIA, Google, Samsung, Valve, Arm, Collabora, and Huawei listed as contributors and was promulgated alongside `VK_EXT_surface_maintenance1` [Source](https://github.com/KhronosGroup/Vulkan-Docs/issues/2006). Mesa 26.1.0 reports `VK_{KHR,EXT}_{surface,swapchain}_maintenance1` support on PanVK, joining earlier support already present on RADV/ANV/NVK/Turnip from prior Mesa releases [Source](https://docs.mesa3d.org/relnotes/26.1.0.html).

---

## 8. `VK_EXT_present_timing`

`VK_EXT_present_timing` is the successor to two earlier, less precise attempts at Vulkan frame-pacing telemetry — the vendor extension `VK_GOOGLE_display_timing` and `VK_KHR_present_wait` (§6.3) — and represents, per the Khronos announcement, "the culmination of" a multi-year effort originally opened by an NVIDIA engineer in 2020 and eventually landing in the Vulkan registry with contributions credited to Google, NVIDIA, AMD, Collabora, Samsung, Unity, and Red Hat [Source](https://www.khronos.org/blog/vk-ext-present-timing-the-journey-to-state-of-the-art-frame-pacing-in-vulkan). It combines two capabilities the older extensions offered only separately: feedback about *past* presentations (queried timestamps at multiple pipeline stages — when a frame was dequeued for presentation, when its first pixel was sent to the display, when it actually became visible) and forward scheduling — an application can request that a specific present happen no earlier than a specific target time, using `VkPresentTimingInfoEXT` chained onto `VkPresentInfoKHR` for the request side and querying completed-present statistics back through the swapchain [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_present_timing.html).

**Mesa support status, verified against current release notes rather than assumed from the outline:** the outline for this chapter stated the extension "merged Mesa 26.1 for RADV/ANV/NVK/PanVK/Turnip." That claim checks out against the primary source — the Mesa 26.1.0 release notes (2026-05-06) list "VK_EXT_present_timing on RADV, NVK, Turnip, ANV, Honeykrisp, panvk" as a new feature of that release [Source](https://docs.mesa3d.org/relnotes/26.1.0.html), one additional driver (Honeykrisp, the Asahi Linux Apple-silicon Vulkan driver) beyond what the outline named. That initial 26.1 landing, however, covered the **Wayland** WSI backend only — Mesa's Wayland present-timing support had reportedly been complete since the prior year, ahead of the driver-level rollout — while **X11/XWayland** support followed separately, authored by a Valve engineer and merged after roughly four months of review into Mesa 26.2.0 (released 2026-08-05), whose release notes list "`VK_EXT_present_timing` now also on wsi/x11" and "`VK_EXT_present_timing` on hasvk" as new items [Source: Mesa 26.2.0 release notes](https://docs.mesa3d.org/relnotes/26.2.0.html) [Source: Phoronix](https://www.phoronix.com/news/Mesa-26.2-X11-Present-Timing). Mesa 26.1.3 (2026-06-18) shipped compliance fixes in the interim, including "deal with vblank-less systems for `VK_EXT_present_timing`" and "allow `VK_EXT_present_timing` present without `presentStageQueries`" [Source](https://docs.mesa3d.org/relnotes/26.1.3.html). As of Mesa 26.2.x, present timing is therefore implemented on both Wayland and X11/XWayland backends across RADV, ANV, hasvk, NVK, Turnip, Honeykrisp, and PanVK — a materially broader and more recent state than the single-version, single-backend framing in the original outline.

**Note: needs verification.** No file named `wsi_common_present_timing.c` exists in Mesa's WSI source tree as of the version checked (`src/vulkan/wsi/` currently contains `wsi_common.c`, `wsi_common_display.c`, `wsi_common_drm.c`, `wsi_common_headless.c`, `wsi_common_wayland.c`, `wsi_common_x11.c`, plus Windows/Metal backends) [Source: Mesa WSI directory listing](https://github.com/chaotic-cx/mesa-mirror/tree/main/src/vulkan/wsi). The present-timing implementation appears to be integrated into the shared `wsi_common.c` core plus the per-backend Wayland and X11 files rather than isolated in a dedicated file; this chapter does not cite a `wsi_common_present_timing.c` path because it could not confirm one exists, and readers should locate the present-timing code by searching Mesa's tree for `present_timing` rather than assuming a fixed filename.

---

## 9. Direct Display

### 9.1 `VK_KHR_display`

`VK_KHR_display` is the instance extension that lets a Vulkan application present directly to a physical display with no window system in between at all — the kiosk/signage/embedded case, and historically the initial basis for VR headset direct-mode presentation before compositor-mediated VR paths matured. Its core enumeration chain is `vkGetPhysicalDeviceDisplayPropertiesKHR` (enumerate `VkDisplayKHR` handles — physical outputs), `vkGetPhysicalDeviceDisplayPlanePropertiesKHR` (enumerate display planes and which displays each can drive), `vkGetDisplayModePropertiesKHR` (enumerate the mode list — resolution/refresh pairs — for a given display), and finally `vkCreateDisplayPlaneSurfaceKHR`, which produces an ordinary `VkSurfaceKHR` from a chosen display, plane, and mode, consumable by `VK_KHR_swapchain` exactly like a windowed surface [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display.html).

### 9.2 `VK_EXT_acquire_drm_display`

`VK_KHR_display`'s enumeration is display-server-agnostic by design, which leaves a gap on Linux: a `VkDisplayKHR` handle needs to correspond to an actual DRM connector, and something has to hand the driver authority over that connector's CRTC (the same authority a display server normally holds). `VK_EXT_acquire_drm_display` closes that gap with `vkAcquireDrmDisplayEXT(physicalDevice, drmFd, display)`, which associates a `VkDisplayKHR` with a connector on an already-open DRM file descriptor the application supplies — commonly obtained via `DRM_IOCTL_MODE_CREATE_LEASE` against a running compositor's DRM master, so a Vulkan application can take exclusive control of one output while the compositor retains the rest, rather than requiring the application to seize DRM master over the whole device [Source: VK_EXT_acquire_drm_display](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_acquire_drm_display.html). A companion `vkGetDrmDisplayEXT` maps a DRM connector ID back to the `VkDisplayKHR` enumerated by `VK_KHR_display`, closing the loop between the two ID spaces.

Both extensions are implemented in Mesa's `src/vulkan/wsi/wsi_common_display.c` — not `wsi_common_drm.c`, which (§2.2, §12) is the shared DMA-BUF/sync-object plumbing used by the Wayland and X11 backends rather than the display-extension implementation. `wsi_common_display.c` implements the full `vkGetPhysicalDeviceDisplayPropertiesKHR`/`vkCreateDisplayPlaneSurfaceKHR`/DRM-lease call chain directly against libdrm, managing connector enumeration, mode selection, and CRTC allocation without going through GBM or a Wayland/X11 buffer-sharing path at all, since there is no compositor on the other end to negotiate modifiers or buffer ownership with [Source: Mesa wsi_common_display.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common_display.c).

---

## 10. Headless WSI

`VK_EXT_headless_surface` exists for the case where an application wants to use the full `VK_KHR_swapchain` API — including image acquisition, present-mode semantics, and swapchain lifetime management — without any real presentation target: CI rendering pipelines, server-side rendering, and offscreen render farms that would otherwise have to special-case "no surface" throughout their frame loop. `vkCreateHeadlessSurfaceEXT` takes a nearly empty `VkHeadlessSurfaceCreateInfoEXT` (`sType`, `pNext`, and a reserved, must-be-zero `flags` field — no platform handle of any kind), and the specification is explicit that presenting to the resulting swapchain "is by default a no-op, resulting in no externally-visible result," while leaving the door open for future extensions to layer real behavior — such as writing frames to a file — on top of the same headless surface type [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_headless_surface.html).

This is the direct Vulkan-WSI analogue of `EGL_EXT_platform_device` combined with GBM headless allocation on the EGL side (Chapter 150 §3, Chapter 24 §8): both give an application a fully functional presentation-shaped API surface with the platform-integration half stripped out, which matters for code that is otherwise structured around the acquire/render/present loop and would rather satisfy that loop's API contract trivially than restructure around a separate offscreen-rendering code path. Mesa implements the backend in `src/vulkan/wsi/wsi_common_headless.c`, allocating ordinary GPU images with no DMA-BUF export, compositor negotiation, or DRM interaction at all.

---

## 11. HDR Swapchains

An HDR-capable swapchain is, mechanically, just a swapchain created with one of the wide-gamut/HDR entries from the `VkColorSpaceKHR` enumeration returned by `vkGetPhysicalDeviceSurfaceFormatsKHR` — most relevantly `VK_COLOR_SPACE_HDR10_ST2084_EXT` (BT.2020 primaries, SMPTE ST 2084 PQ transfer function, the mode consumed by most HDR10 displays and matching what the KMS HDR metadata/PQ EOTF path expects — Chapter 74 covers the display-side metadata handshake in depth), `VK_COLOR_SPACE_EXTENDED_SRGB_LINEAR_EXT` (scRGB: linear light values referenced to the sRGB primaries but permitted to exceed `[0,1]`, the format most compositors expect for HDR-capable linear blending), and `VK_COLOR_SPACE_BT2020_LINEAR_EXT` (linear light in BT.2020 primaries, without a PQ or scRGB-style extended range convention) — all defined by `VK_EXT_swapchain_colorspace` [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_swapchain_colorspace.html). Whether a given surface actually reports any of these depends entirely on the platform and current output: a Wayland compositor only advertises an HDR-capable color space for a surface once it has both an HDR-capable output attached and its own HDR pipeline enabled, and an application selecting one of these color spaces without compositor cooperation gets a swapchain that is technically valid but rendered through whatever tone-mapping or clamping the compositor applies to non-negotiated HDR content.

On Wayland that negotiation is precisely the job of `wp_color_management_v1`, the color-management protocol extension that reached the `wayland-protocols` staging directory in February 2025 (`wayland-protocols` 1.41) [Source](https://www.osnews.com/story/141803/12-years-of-incubating-wayland-color-management/). A client creates a `wp_color_management_surface_v1` for its `wl_surface` and uses it to declare that surface's actual color space and HDR characteristics — image description, primaries, transfer characteristic, and mastering/content light-level metadata — so the compositor can perform correct tone-mapping and blending against the rest of the desktop rather than treating the buffer as opaque sRGB content. A Vulkan application driving an HDR swapchain on Wayland is expected to pair its `VkColorSpaceKHR` selection with the matching `wp_color_management_v1` surface declaration; Vulkan's own WSI layer has no protocol-level visibility into `wp_color_management_v1` itself (it is EGL/GL and application-toolkit code that has historically owned this coordination), which is tracked as an open coordination question in Khronos's own Vulkan-Docs issue tracker for Wayland color management [Source](https://github.com/KhronosGroup/Vulkan-Docs/issues/2307). Chapter 74 covers the compositor and kernel-side HDR/WCG metadata pipeline this swapchain selection ultimately feeds into.

---

## 12. Mesa WSI Common Layer Architecture Summary

Every Mesa Vulkan driver links a single shared WSI implementation rather than each maintaining its own. The layout, current as of the Mesa tree checked for this chapter [Source: directory listing](https://github.com/chaotic-cx/mesa-mirror/tree/main/src/vulkan/wsi):

| File | Responsibility |
|---|---|
| `wsi_common.c` / `wsi_common.h` | Platform-agnostic core: `VkSwapchainKHR` object lifetime, `vkQueuePresentKHR`/`vkAcquireNextImage2KHR` dispatch, present-mode and capability plumbing shared by every backend |
| `wsi_common_wayland.c` | `VK_KHR_wayland_surface` backend (§2) |
| `wsi_common_x11.c` | `VK_KHR_xcb_surface`/`VK_KHR_xlib_surface` backend, DRI3/Present (§3) |
| `wsi_common_display.c` | `VK_KHR_display` and `VK_EXT_acquire_drm_display` (§9) |
| `wsi_common_drm.c` | Shared DMA-BUF image allocation, modifier filtering, and explicit/implicit sync setup used by the Wayland and X11 backends (§2.2) |
| `wsi_common_headless.c` | `VK_EXT_headless_surface` (§10) |
| `wsi_common_win32.cpp`, `wsi_common_metal.c`/`.m` | Non-Linux backends, out of scope here |

The two backends that actually allocate GPU-shareable images on Linux — Wayland and X11 — both go through GBM for the underlying buffer object, calling `gbm_bo_create_with_modifiers2(gbm_device, width, height, format, modifiers, count, flags)` rather than the older `gbm_bo_create_with_modifiers`, specifically because the newer entry point also accepts GBM usage flags (`GBM_BO_USE_SCANOUT`, `GBM_BO_USE_RENDERING`, and so on) alongside the modifier list — a gap in the original function that forced allocation-time assumptions the newer entry point removed [Source](https://cgit.freedesktop.org/mesa/mesa/commit/?id=268e12c605341eedfda22bdbbf623aa123a290e8). The modifier list passed to that call is exactly the negotiated set from §2.3 on Wayland (dmabuf-feedback-restricted where available) or a driver-queried DRI3-advertised list on X11; `wsi_common_drm.c` (not to be confused with the display-extension file, per §9.2) is where that negotiated, backend-agnostic allocation and the subsequent explicit-sync wiring described in §2.5 actually live, called into by both `wsi_common_wayland.c` and `wsi_common_x11.c` rather than duplicated in each.

---

## Integrations

- **[Chapter 24 — Vulkan and EGL for Application Developers](ch24-vulkan-egl-application-developers.md)** covers WSI initialization from the application-developer entry point — instance/device setup, the same Wayland swapchain and present-mode material at introductory depth, and the explicit-sync story from fences and semaphores down to `drm_syncobj`, which this chapter assumes as background for §2.5.
- **[Chapter 150 — EGL Architecture and DMA-BUF Integration](ch150-egl-architecture-dmabuf.md)** is the EGL-side counterpart to this chapter's presentation path: `EGLImage`, `EGL_EXT_image_dma_buf_import`, and `zwp_linux_dmabuf_v1` from the GL/EGL perspective rather than Vulkan WSI's, sharing the same underlying kernel DMA-BUF and modifier-negotiation mechanisms described here in §2.3.
- **[Chapter 20 — Wayland Protocol Fundamentals](../part-06a-wayland-compositor/ch20-wayland-protocol-fundamentals.md)** covers `wl_surface`, `wl_buffer`, and the base `zwp_linux_dmabuf_v1` protocol that §2 builds on.
- **[Chapter 74 — HDR and Wide Color Gamut](../part-06b-display-services/ch74-hdr-wide-color-gamut.md)** covers the compositor- and kernel-side HDR metadata pipeline (KMS HDR properties, PQ/HLG handling) that an HDR swapchain (§11) ultimately feeds.
- **[Chapter 75 — Explicit GPU Synchronisation](../part-06b-display-services/ch75-explicit-gpu-sync.md)** covers `drm_syncobj` and the kernel-level explicit-sync uAPI in depth; §2.5 of this chapter is specifically the Wayland-protocol (`wp_linux_drm_syncobj_v1`) leg of that same synchronization chain.
- **[Chapter 112 — VRR, FreeSync, and Frame Pacing](../part-06b-display-services/ch112-vrr-freesync-frame-pacing.md)** covers frame-pacing strategy and variable refresh rate at the compositor/kernel level; §8's `VK_EXT_present_timing` and §4's FIFO-family present modes are the Vulkan-application-side half of the same frame-pacing problem.

---

## References

- [VK_KHR_surface man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_surface.html)
- [VK_KHR_wayland_surface man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_wayland_surface.html)
- [Vulkan WSI chapter — Window System Integration](https://docs.vulkan.org/spec/latest/chapters/VK_KHR_surface/wsi.html)
- [Mesa `wsi_common_wayland.c` source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common_wayland.c)
- [Mesa `wsi_common_display.c` source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common_display.c)
- [Mesa WSI directory listing (mirror)](https://github.com/chaotic-cx/mesa-mirror/tree/main/src/vulkan/wsi)
- [linux-dmabuf-v1 protocol (dmabuf-feedback)](https://wayland.app/protocols/linux-dmabuf-v1)
- [linux-drm-syncobj-v1 protocol](https://wayland.app/protocols/linux-drm-syncobj-v1)
- [DRI3 protocol specification](https://cgit.freedesktop.org/xorg/proto/dri3proto/plain/dri3proto.txt)
- [DRI3 extension announcement — keithp.com](https://keithp.com/blogs/dri3_extension/)
- [Mesa's Vulkan WSI/Wayland code adds IMMEDIATE present mode — Phoronix](https://www.phoronix.com/news/Mesa-VLK-WSI-Wayland-IMMEDIATE)
- [VkSwapchainCreateInfoKHR man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VkSwapchainCreateInfoKHR.html)
- [VK_KHR_swapchain_mutable_format man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_swapchain_mutable_format.html)
- [vkAcquireNextImage2KHR man page](https://registry.khronos.org/vulkan/specs/latest/man/html/vkAcquireNextImage2KHR.html)
- [VK_KHR_present_wait documentation](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_present_wait.html)
- [VK_EXT_swapchain_maintenance1 proposal](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/proposals/VK_EXT_swapchain_maintenance1.adoc)
- [VK_EXT_surface_maintenance1 / VK_EXT_swapchain_maintenance1 release task list](https://github.com/KhronosGroup/Vulkan-Docs/issues/2006)
- [VK_EXT_present_timing: the Journey to State-of-the-Art Frame Pacing in Vulkan — Khronos Blog](https://www.khronos.org/blog/vk-ext-present-timing-the-journey-to-state-of-the-art-frame-pacing-in-vulkan)
- [VK_EXT_present_timing feature proposal](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_present_timing.html)
- [Mesa 26.1.0 Release Notes](https://docs.mesa3d.org/relnotes/26.1.0.html)
- [Mesa 26.1.3 Release Notes](https://docs.mesa3d.org/relnotes/26.1.3.html)
- [Mesa 26.2.0 Release Notes](https://docs.mesa3d.org/relnotes/26.2.0.html)
- [Mesa's Vulkan Present Timing Now on X11/XWayland — Phoronix](https://www.phoronix.com/news/Mesa-26.2-X11-Present-Timing)
- [VK_KHR_display man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display.html)
- [VK_EXT_acquire_drm_display man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_acquire_drm_display.html)
- [VK_EXT_headless_surface man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_headless_surface.html)
- [VK_EXT_swapchain_colorspace man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_swapchain_colorspace.html)
- [12 years of incubating Wayland color management — Collabora](https://www.collabora.com/news-and-blog/news-and-events/12-years-of-incubating-wayland-color-management.html)
- [Color management on the Vulkan Wayland platform — Vulkan-Docs issue #2307](https://github.com/KhronosGroup/Vulkan-Docs/issues/2307)
- [gbm: add gbm_{bo,surface}_create_with_modifiers2 — Mesa commit](https://cgit.freedesktop.org/mesa/mesa/commit/?id=268e12c605341eedfda22bdbbf623aa123a290e8)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
