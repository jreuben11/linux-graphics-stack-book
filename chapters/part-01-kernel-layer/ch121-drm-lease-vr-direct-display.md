# Chapter 121: DRM Lease and VR Direct Display

> **Part**: Part I — The Kernel Layer
> **Audience**: VR compositor developers, OpenXR runtime implementers, Wayland compositor authors, and display engineers working with head-mounted displays (HMDs)
> **Status**: First draft — 2026-06-19

## Table of Contents

- [Overview](#overview)
- [1. The VR Latency Problem](#1-the-vr-latency-problem)
- [2. DRM Lease API](#2-drm-lease-api)
- [3. Connector Discovery for VR HMDs](#3-connector-discovery-for-vr-hmds)
- [4. Monado OpenXR Runtime](#4-monado-openxr-runtime)
- [5. wp_drm_lease_device_v1 Wayland Protocol](#5-wp_drm_lease_device_v1-wayland-protocol)
- [6. SteamVR on Linux](#6-steamvr-on-linux)
- [6b. ALVR: Wireless PC VR Streaming via DRM Lease](#6b-alvr-wireless-pc-vr-streaming-via-drm-lease)
- [7. Direct-to-Display Vulkan Swapchain](#7-direct-to-display-vulkan-swapchain)
- [8. Timing and Synchronisation for VR](#8-timing-and-synchronisation-for-vr)
- [9. Practical: Setting Up VR on Linux](#9-practical-setting-up-vr-on-linux)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

Virtual reality headsets impose display requirements that are fundamentally incompatible with how desktop compositors work. A desktop compositor — whether Mutter, KWin, or wlroots-based — owns the kernel's KMS (Kernel Mode Setting) master context and intermediates every frame that reaches the screen. For an HMD displaying stereo imagery at 90 or 120 Hz, that intermediation adds a frame of latency: the VR runtime renders a frame, hands it to the compositor, and the compositor queues it for the next VBLANK. That single extra frame — roughly 11 ms at 90 Hz — is enough to induce motion sickness for many users.

**DRM lease** is the kernel mechanism that solves this. Introduced in Linux 4.15 (kernel commit 62884cd386b8, merged in January 2018), DRM lease allows the KMS master — typically the desktop compositor — to delegate exclusive ownership of specific display objects (CRTCs, connectors, and planes) to another process via a new file descriptor. The lessee obtains full KMS modesetting authority over those objects, drives them directly without involving the desktop compositor in the rendering critical path, and returns them when the VR session ends.

This chapter covers the DRM lease kernel API, how VR runtimes discover and identify HMD connectors, the Monado OpenXR runtime's use of the lease mechanism, the `wp_drm_lease_device_v1` Wayland protocol that allows unprivileged clients to request a lease, the direct-to-display Vulkan swapchain path via `VK_EXT_acquire_drm_display`, and the frame timing machinery that VR runtimes use to hit their display's VBLANK with microsecond precision.

---

## 1. The VR Latency Problem

### Motion-to-Photon Latency

The defining comfort metric for VR is **motion-to-photon latency**: the time elapsed between a user moving their head and the updated image appearing on the HMD's displays. A latency above approximately 20 ms causes most users to experience disorientation or nausea; values below 15 ms are considered comfortable for sustained use. Achieving this budget requires carefully accounting for every stage in the pipeline:

| Stage | Typical budget |
|---|---|
| IMU sampling and pose prediction | ~1 ms |
| Application CPU work (scene graph, draw calls) | ~3 ms |
| GPU rendering (geometry + shading) | ~7 ms |
| Display compositor warp pass | ~2 ms |
| KMS atomic commit → VBLANK | ~1 ms |
| Display panel propagation delay | ~2–4 ms |
| **Total** | **~16–18 ms** |

This budget leaves no room for an additional compositor frame. If the desktop compositor buffers the VR output — completing its own compositing pass before submitting the result to KMS — an entire additional refresh period (11 ms at 90 Hz, 8 ms at 120 Hz) is injected into the pipeline. Direct display access is not optional; it is required.

### Asynchronous TimeWarp

Even with direct display, the GPU render may occasionally run long and miss the target VBLANK. **Asynchronous TimeWarp** (ATW) is the safety net: a lightweight GPU pass that runs immediately before the VBLANK deadline and reprojects the previous frame's output using the most recent head pose. Because the head pose update requires only a matrix multiply on an already-rendered texture, ATW can complete in under 2 ms on modern GPUs. Monado implements ATW as a separate Vulkan compute submission on a high-priority queue, submitted after the main render queue drains or when a deadline timer fires. [Source](https://monado.freedesktop.org/direct-mode.html)

### Why DRM Lease Rather Than VFIO

An alternative — assigning the GPU entirely to a VM via VFIO passthrough — exists for some VFIO-capable systems but introduces its own problems: the GPU driver stack does not run inside the host kernel, making sharing between the VR workload and the desktop non-trivial; VFIO passthrough has non-negligible latency from IOMMU mapping and VM context switches; and it requires hardware that supports multi-GPU or SR-IOV configurations. DRM lease is strictly preferable when the VR headset and the desktop monitor are both attached to the same GPU: the kernel driver remains fully active, existing buffer sharing via DMA-BUF works normally, and the overhead is a single ioctl rather than a VM boundary crossing.

---

## 2. DRM Lease API

### Overview and History

DRM lease was designed and implemented by Keith Packard and merged into the Linux 4.15 kernel in January 2018 as part of the DRM core. The mechanism creates a hierarchy of **DRM masters** (the kernel-side structure tracking which process holds KMS authority). Before DRM lease, exactly one master existed per DRM device; with DRM lease, a master (the **lessor**) can create child masters (**lessees**) that hold exclusive access to a subset of KMS objects. The lessor retains access to all non-leased objects. [Source](https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html)

### Kernel Data Structures

The key kernel-side structure is `struct drm_master` in `include/drm/drm_auth.h`. Lease-relevant fields added in 4.15:

```c
/* include/drm/drm_auth.h */
struct drm_master {
    /* ... existing fields ... */
    struct drm_master   *lessor;       /* NULL for owner; non-NULL for lessee */
    int                  lessee_id;    /* unique ID assigned to this lessee */
    struct list_head     lessee_list;  /* entry in lessor's lessees list */
    struct list_head     lessees;      /* list of drm_masters leasing from us */
    struct idr           leases;       /* IDR of KMS object IDs leased out */
    struct idr           lessee_idr;   /* IDR of all lessees under the owner */
};
```

The implementation lives in `drivers/gpu/drm/drm_lease.c`. The predicate `drm_lease_held(file_priv, id)` is called from every KMS ioctl handler that touches a KMS object: it returns `true` if the calling `drm_file` has authority over the specified object ID, either as the owner or as a lessee whose lease covers that ID. [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_lease.c)

### The Four Lease IOCTLs

All four lease management ioctls were added simultaneously in kernel 4.15 and are defined in `include/uapi/drm/drm_mode.h`:

#### DRM_IOCTL_MODE_CREATE_LEASE

Creates a new lease. The caller must be the current DRM master.

```c
/* include/uapi/drm/drm_mode.h */
struct drm_mode_create_lease {
    __u64 object_ids;    /* pointer to array of KMS object IDs to lease */
    __u32 object_count;  /* number of IDs in the array */
    __u32 flags;         /* O_CLOEXEC, O_NONBLOCK accepted */
    __u32 lessee_id;     /* output: unique lessee ID assigned by kernel */
    __u32 fd;            /* output: new DRM fd for the lessee */
};
```

The kernel validates that each ID in `object_ids` refers to a CRTC, connector, or plane; that none are already leased; and that the requested objects form a valid display pipeline (connector + CRTC + at least a primary plane). On success it installs the new fd into the calling process's file table and returns the lessee ID and fd number. The lessee fd is a fully functional DRM file descriptor — the lessee can call `DRM_IOCTL_MODE_ATOMIC` on it to modesetting the leased objects exactly as if it were the primary master. [Source](https://lore.kernel.org/lkml/20171005061310.29919-7-keithp@keithp.com/)

#### DRM_IOCTL_MODE_LIST_LESSEES

Returns the lessee IDs of all currently active lessees for the calling master:

```c
struct drm_mode_list_lessees {
    __u32 count_lessees; /* input/output: capacity / actual count */
    __u32 pad;
    __u64 lessees_ptr;   /* pointer to __u64 array to fill with lessee IDs */
};
```

#### DRM_IOCTL_MODE_GET_LEASE

Returns the KMS object IDs covered by a specific lease. The lessee calls this to query its own lease; the lessor calls it with a lessee_id to inspect a child lease:

```c
struct drm_mode_get_lease {
    __u32 count_objects; /* input/output: capacity / actual object count */
    __u32 pad;
    __u64 objects_ptr;   /* pointer to __u32 array to fill with object IDs */
};
```

#### DRM_IOCTL_MODE_REVOKE_LEASE

Terminates a lease. The lessor calls this with the lessee ID to revoke:

```c
struct drm_mode_revoke_lease {
    __u32 lessee_id;     /* ID of the lease to revoke */
};
```

After revocation, any subsequent KMS ioctl on the lessee fd that touches a formerly-leased object returns `EACCES`. The lessee fd itself remains open (the file descriptor is not closed), but it has no remaining authority over any display object. The lessee's process detects this via the `wp_drm_lease_v1.finished` event if the Wayland protocol path was used, or via polling on the lessee fd using a `POLLHUP`-style mechanism.

### libdrm Wrappers

libdrm (version ≥ 2.4.89) wraps all four ioctls:

```c
/* libdrm/xf86drm.h */
int drmModeCreateLease(int fd,
                       const uint32_t *objects,
                       int num_objects,
                       int flags,
                       uint32_t *lessee_id);

int drmModeListLessees(int fd,
                       uint32_t *count,
                       uint32_t **lessee_ids);

int drmModeGetLease(int fd,
                    drmModeObjectListPtr *objects);

int drmModeRevokeLease(int fd, uint32_t lessee_id);
```

[Source](https://gitlab.freedesktop.org/mesa/drm/-/blob/main/xf86drm.h)

### Lease Lifetime and Cleanup

A lease is automatically cleaned up when the lessee fd is closed: the kernel calls `drm_lease_destroy()`, which removes all leased objects from the lessee's IDR and moves them back to the lessor's available pool. This means a VR session that crashes without explicitly closing its DRM fd still releases the HMD connector — the kernel handles cleanup via the file descriptor reference count.

---

## 3. Connector Discovery for VR HMDs

### HMDs as Regular Monitors

From the kernel's perspective, a VR headset connected via DisplayPort, HDMI, or USB-C DisplayPort Alternate Mode is indistinguishable from a monitor. The HMD appears as a regular DRM connector, and `DRM_IOCTL_MODE_GETRESOURCES` enumerates it alongside the user's desktop displays. The challenge is identification: the VR runtime must identify which connector belongs to the HMD and which belong to the user's monitor setup, without stealing the wrong display.

### The non-desktop EDID Quirk

The kernel's EDID parsing code in `drivers/gpu/drm/drm_edid.c` maintains a quirk table (`edid_quirk_list`) that flags specific vendor/product ID combinations with `EDID_QUIRK_NON_DESKTOP`. When this flag is set, the connector is marked with the `non-desktop` DRM property set to 1. Compositors and VR runtimes query this property to distinguish HMDs from regular displays.

Known HMDs and XR devices flagged in the kernel:

| Vendor ID | Description |
|---|---|
| `VLV` | Valve Index (product IDs 0x91a8–0x91bf) |
| `HVR` | HTC Vive and Vive Pro (0xaa01, 0xaa02) |
| `OVR` | Oculus HMDs (0x0001, 0x0003, 0x0004, 0x0012) |
| `SNY` | Sony PlayStation VR (0x0704) |
| `ACR` | Acer Windows Mixed Reality (0x7fce) |
| `LEN` | Lenovo Windows Mixed Reality (0x0408) |
| `SEC` | Samsung Windows Mixed Reality (0x144a) |
| `SEN` | Sensics OSVR (0x1019) |

[Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_edid.c)

The `non-desktop` property is a standard DRM connector property that compositors check before including a connector in their output layout. A compositor that respects `non-desktop` will advertise the HMD connector to clients via `wp_drm_lease_device_v1` (see Section 5) rather than scanning it out as a desktop output. When a compositor does not recognise the `non-desktop` property or runs an older version that predates lease support, the HMD appears on the desktop as an additional monitor — the user sees the headset's displays mirrored — and VR runtimes must use a different acquisition path.

### Connector Enumeration in Userspace

A VR runtime that acquires the DRM fd directly (not via Wayland) enumerates connectors with `DRM_IOCTL_MODE_GETRESOURCES` and then for each connector calls `DRM_IOCTL_MODE_GETCONNECTOR` and `DRM_IOCTL_MODE_OBJ_GETPROPERTIES` to read the `non-desktop` property. The libdrm equivalent:

```c
drmModeRes *res = drmModeGetResources(drm_fd);
for (int i = 0; i < res->count_connectors; i++) {
    drmModeConnector *conn = drmModeGetConnector(drm_fd,
                                                  res->connectors[i]);
    drmModeObjectProperties *props =
        drmModeObjectGetProperties(drm_fd,
                                   res->connectors[i],
                                   DRM_MODE_OBJECT_CONNECTOR);
    for (uint32_t j = 0; j < props->count_props; j++) {
        drmModePropertyRes *prop =
            drmModeGetProperty(drm_fd, props->props[j]);
        if (strcmp(prop->name, "non-desktop") == 0) {
            uint64_t val = props->prop_values[j];
            if (val == 1) {
                /* this is an HMD connector */
            }
        }
        drmModeFreeProperty(prop);
    }
    drmModeFreeObjectProperties(props);
    drmModeFreeConnector(conn);
}
drmModeFreeResources(res);
```

### VK_EXT_acquire_drm_display

`VK_EXT_acquire_drm_display` (registered extension number 286, revision 1) is the Vulkan bridge from a DRM connector to a `VkDisplayKHR`. It was authored by Simon Zeni and merged into the Vulkan specification in 2021. It provides two entry points:

```c
/* Vulkan 1.3 extension: VK_EXT_acquire_drm_display */

/* Convert a DRM connector ID to a VkDisplayKHR */
VkResult vkGetDrmDisplayEXT(
    VkPhysicalDevice    physicalDevice,
    int32_t             drmFd,
    uint32_t            connectorId,
    VkDisplayKHR       *display);

/* Acquire exclusive Vulkan control of the display */
VkResult vkAcquireDrmDisplayEXT(
    VkPhysicalDevice    physicalDevice,
    int32_t             drmFd,
    VkDisplayKHR        display);
```

The `drmFd` passed to both functions must be the **lessee fd** — the fd returned by `DRM_IOCTL_MODE_CREATE_LEASE` that has authority over the target connector. The Vulkan driver uses the fd to verify the calling process is the DRM master for that connector; if it is not, `vkAcquireDrmDisplayEXT` returns `VK_ERROR_INITIALIZATION_FAILED`. After acquisition, the application creates a `VkSurfaceKHR` via `vkCreateDisplayPlaneSurfaceKHR` and presents to the HMD using a normal Vulkan swapchain (see Section 7).

RADV (AMD) and ANV (Intel) both support `VK_EXT_acquire_drm_display`, added by Keith Packard's series of Mesa patches that wired the full direct display stack into Mesa's WSI layer. [Source](https://www.phoronix.com/news/KeithP-Vulkan-Direct-Display)

---

## 4. Monado OpenXR Runtime

### Architecture

**Monado** is the open-source OpenXR 1.0 runtime developed by Collabora and the broader FreeDesktop community. [Source](https://monado.freedesktop.org/) It is structured as two cooperating components:

- `monado-service`: a privileged system daemon that holds the DRM master lease and owns the Vulkan swapchain presented to the HMD
- Client library (`libopenxr_monado.so`): the OpenXR API implementation that games and applications link against; it communicates with `monado-service` via a Unix domain socket using a custom IPC protocol

The compositor half of `monado-service` lives in `src/xrt/compositor/` and is responsible for distortion correction, ATW, and submitting frames to the display. [Source](https://monado.pages.freedesktop.org/monado/understanding-targets.html)

### Display Backend Selection

Monado supports three display acquisition paths, selected at startup based on available environment and compositor support:

1. **Wayland DRM lease** (`comp_window_direct_wayland.c`): requests the lease via the `wp_drm_lease_device_v1` Wayland protocol. This is the preferred path when running under a modern Wayland compositor that supports the protocol (wlroots, KWin/Plasma 6, Mutter/GNOME 47+).

2. **X11 XRandR direct mode** (`comp_window_direct.c`): acquires the HMD display via `VK_EXT_acquire_xlib_display`, which uses the XRandR lease API on top of the DRM lease. Used under X11 or XWayland.

3. **KMS/DRM direct** (`XRT_COMPOSITOR_FORCE_VK_DISPLAY=1`): bypasses all window system layers and drives the HMD via `VK_KHR_display` with the calling process as DRM master. Requires running as root or with the calling user in the `video` group. Introduced in Monado v21.0.0.

### The Wayland DRM Lease Path in Detail

When running under a Wayland compositor with `wp_drm_lease_device_v1` support, Monado's `comp_window_direct_wayland.c` follows this sequence:

1. Bind to the `wp_drm_lease_device_v1` global advertised by the compositor.
2. Receive the compositor's `drm_fd` event: a non-master DRM fd for enumerating connectors without taking master ownership.
3. Receive `connector` events enumerating the HMD connectors the compositor has identified.
4. Call `create_lease_request()` → `request_connector()` for the target HMD connector → `submit()`.
5. Receive the `wp_drm_lease_v1.lease_fd` event containing the lessee fd with KMS master authority.
6. Call `vkGetDrmDisplayEXT(physicalDevice, lease_fd, connector_id, &display)` to get the `VkDisplayKHR`.
7. Call `vkAcquireDrmDisplayEXT(physicalDevice, lease_fd, display)` to take Vulkan ownership.
8. Create `VkSurfaceKHR` via `vkCreateDisplayPlaneSurfaceKHR` and create the swapchain.
9. Begin the compositor loop: render left+right eye views, apply lens distortion and chromatic aberration correction, time-warp the output to the latest pose, submit to the swapchain, call `vkQueuePresentKHR`.

```c
/* Simplified Monado DRM lease acquisition sequence */
/* src/xrt/compositor/main/comp_window_direct_wayland.c */

static void
lease_fd_handler(void *data,
                 struct wp_drm_lease_v1 *lease,
                 int fd)
{
    struct direct_wayland_lease *dwl = data;
    dwl->drm_fd = fd;  /* lessee fd, now DRM master for HMD connector */
}

static const struct wp_drm_lease_v1_listener lease_listener = {
    .lease_fd = lease_fd_handler,
    .finished = lease_finished_handler,
};
```

[Source](https://monado.pages.freedesktop.org/monado/comp__window__direct__wayland_8c.html)

### Compositor Loop and Timing

Monado's compositor loop runs on a dedicated high-priority thread. At the start of each frame it calls `xrt_frame_times_predict()` to compute the target VBLANK timestamp, renders both eye views, runs ATW if required, and calls `vkQueuePresentKHR` with a fence that will be signalled when the GPU work completes. The present is constrained to arrive before the target VBLANK by the `VK_EXT_present_timing` extension (see Section 8). The `u_timing_frame` predictor module learns the GPU render time over successive frames and adjusts the render start time to ensure the GPU finishes before the VBLANK with a configurable safety margin (default: 2 ms). [Source](https://monado.freedesktop.org/getting-started.html)

---

## 5. wp_drm_lease_device_v1 Wayland Protocol

### Motivation

DRM lease requires the calling process to be the current DRM master, or to receive a lessee fd from the DRM master. Under Wayland, the compositor is the DRM master. A VR runtime running as an unprivileged Wayland client cannot call `DRM_IOCTL_MODE_CREATE_LEASE` directly — it must ask the compositor to create the lease on its behalf. The `wp_drm_lease_device_v1` protocol, defined in `wayland-protocols/staging/drm-lease/drm-lease-v1.xml`, is the mechanism for this. [Source](https://gitlab.freedesktop.org/wayland/wayland-protocols)

### Protocol Interfaces

The protocol defines four interfaces:

**`wp_drm_lease_device_v1`** — advertised as a Wayland global by compositors that support the protocol, one per DRM node. Sends:
- `drm_fd`: a non-master fd for the DRM device (for connector enumeration only)
- `connector`: announces each HMD connector available for lease
- `done`: signals all current connectors have been announced
- `released`: response to the client's `release` request

**`wp_drm_lease_connector_v1`** — represents one leasable connector. Carries:
- `name`: human-readable name (e.g., `"DP-1"`)
- `description`: e.g., `"Valve Index"`)
- `connector_id`: the DRM connector ID for use with `DRM_IOCTL_MODE_GETCONNECTOR`
- `withdrawn`: signals the connector is no longer leasable (unplugged, or taken by another client)

**`wp_drm_lease_request_v1`** — assembles the list of connectors to lease. Requests:
- `request_connector(connector)`: add a connector to the request
- `submit()`: submit the assembled request (creates the underlying `DRM_IOCTL_MODE_CREATE_LEASE` call inside the compositor)

**`wp_drm_lease_v1`** — represents the live lease. Sends:
- `lease_fd`: the lessee fd (DRM master for leased objects)
- `finished`: signals the lease has been revoked (compositor shutdown, HMD unplugged, or explicit revocation)

### Protocol Flow

```
Compositor                          Client (VR Runtime)
    |                                      |
    |--- wp_drm_lease_device_v1 global --->|  (compositor advertises)
    |<-- bind(wp_drm_lease_device_v1) -----|
    |--- drm_fd(non_master_fd) ----------->|  (enumerate connectors safely)
    |--- connector(hmd_conn) ------------->|  (HMD connector available)
    |--- done() -------------------------->|
    |                                      |
    |<-- create_lease_request() -----------|
    |<-- request_connector(hmd_conn) ------|
    |<-- submit() --------------------------|  (triggers CREATE_LEASE ioctl)
    |                                      |
    |--- wp_drm_lease_v1.lease_fd(fd) ---->|  (lessee fd — DRM master for HMD)
    |                                      |
    |                          [VR session running]
    |                                      |
    |--- wp_drm_lease_v1.finished() ------>|  (HMD unplugged / session end)
```

### Compositor Implementations

As of 2026, `wp_drm_lease_device_v1` is implemented by a wide range of Wayland compositors: wlroots-based compositors (Sway, Cage, Labwc, river, Hyprland, Wayfire), KWin (Plasma 6+), Mutter (GNOME 47+, merged June 2024 via MR !3739 authored by José Expósito based on earlier work by Jonas Ådahl), COSMIC, Gamescope, Weston, niri, and others. [Source](https://wayland.app/protocols/drm-lease-v1) [Source](https://gamingonlinux.com/2024/06/drm-lease-protocol-support-finally-merged-for-gnome-wayland-great-for-vr-fans/)

The wlroots implementation (`types/wlr_drm_lease_v1.c`) exposes `wlr_drm_lease_manager_v1` for compositor authors: it handles the global advertisement, connector enumeration, and lease creation automatically, notifying the compositor via `wl_signal` events when a client requests a lease. The compositor calls `wlr_drm_lease_request_v1_grant()` to approve and receive the lessee fd. [Source](https://github.com/swaywm/wlroots/blob/master/types/wlr_drm_lease_v1.c)

---

## 6. SteamVR on Linux

### SteamVR Architecture

SteamVR on Linux is Valve's proprietary OpenXR runtime and VR compositor, distributed via Steam. It consists of several components running as system services:

- `vr-server`: the main SteamVR compositor process; drives the HMD display and handles OpenXR session management
- `lighthouse_watchman` (or `steamvr_lh`): the LightHouse tracking service for Valve Index and HTC Vive base stations
- `vrwebhelper`: browser-based SteamVR dashboard compositor

### Display Acquisition History on Linux

SteamVR's Linux display acquisition path has evolved significantly:

1. **Early approach (pre-2021)**: `VK_EXT_direct_mode_display` + `VK_EXT_acquire_xlib_display` on X11. Required XRandR 1.6 support in the X server and was fragile across driver versions.

2. **Wayland DRM lease (2022+)**: SteamVR adopted `wp_drm_lease_device_v1` for Wayland sessions. The transition was gradual; early Wayland support required the user to be in the `video` group for fallback direct DRM access.

3. **Current (2024+)**: On supported compositors (Sway, KWin/Plasma 6, Mutter/GNOME 47+), SteamVR uses `wp_drm_lease_device_v1` exclusively and does not require elevated privileges. On unsupported compositors it falls back to direct DRM access. [Source](https://steamcommunity.com/sharedfiles/filedetails/?id=2984005943)

### NVIDIA Specifics

NVIDIA's proprietary driver complicates the DRM lease path. The NVIDIA driver does not implement the standard `drm_lease` kernel interface in the same way as AMD and Intel drivers. SteamVR on NVIDIA historically required the proprietary NVIDIA "direct mode" mechanism (`VK_NV_acquire_winit_display` or `VK_EXT_acquire_drm_display` via the proprietary path) rather than the standard DRM lease ioctl. NVIDIA's Vulkan driver supports `VK_EXT_present_timing` (see Section 8), and open-source `nvidia-open` driver support for the standard DRM lease path is an ongoing area of work. [Source](https://forums.developer.nvidia.com/t/nvidia-proprietary-non-open-modules-completely-unable-to-acquire-a-drm-lease-on-any-display-server-all-known-nvidia-drivers-any-hardware/341244)

### Valve's Ecosystem Contributions

Valve has contributed significantly to the open-source VR stack on Linux:
- Co-authored the `wp_drm_lease_device_v1` protocol specification
- Added HMD EDID quirks for Valve Index to the upstream Linux kernel
- Maintains SteamVR-for-Linux bug tracking publicly at `https://github.com/ValveSoftware/SteamVR-for-Linux`

The **Monado migration path** is increasingly viable: users wanting a fully open-source VR stack can install Monado as their system OpenXR runtime (`/etc/openxr/active_runtime.json`), which then handles DRM lease acquisition and HMD display management, while individual games use their native OpenXR API calls transparently through Monado's client library.

### Known Issues

A recurring issue (as of 2024–2025) affects GNOME 47 + Mutter 47.x combinations: SteamVR reports "Error 498: Failed to lease display" because GNOME 47 introduced DRM lease support but with early implementation bugs around connector advertisement timing. The upstream fix was backported to Mutter 47.2+. Users on older Mutter versions can work around the issue by running the X11 session or switching to a wlroots-based compositor. [Source](https://github.com/ValveSoftware/SteamVR-for-Linux/issues/753)

---

## 6b. ALVR: Wireless PC VR Streaming via DRM Lease

ALVR (Air Light VR) is an open-source streaming VR runtime ([github.com/alvr-org/ALVR](https://github.com/alvr-org/ALVR)) that turns a standalone Wi-Fi headset — Meta Quest 2/3/Pro, Pico 4, or any OpenXR-capable standalone device — into a wireless PC VR display. On Linux, DRM lease is the mechanism that makes this possible: ALVR's SteamVR driver requests a lease for a virtual display output so that SteamVR sees a standard KMS-driven "monitor" with VR-appropriate VBLANK timing, without ALVR needing to own the entire DRM master.

### Architecture

ALVR splits into two components:

```
PC (Linux)                              Headset (Android/AOSP)
┌────────────────────────────┐          ┌────────────────────────────┐
│ SteamVR / OpenXR game      │          │  ALVR Client (sideloaded)  │
│         │ OpenXR API       │          │         │                  │
│  ALVR SteamVR Driver       │  Wi-Fi   │  Video decoder (MediaCodec)│
│  (OpenVR plugin .so)       │◄────────►│  ATW reprojection          │
│         │                  │  H.264/  │  Head pose → PC (UDP)      │
│  alvr_server (Rust)        │  HEVC/   │                            │
│  ├─ Encode: NVENC/AMF/VAAPI│  AV1     └────────────────────────────┘
│  ├─ DRM virtual display    │
│  ├─ Pose prediction        │
│  └─ Audio capture (PipeWire│
└────────────────────────────┘
```

`alvr_server` is a Rust daemon that:
1. Registers a **virtual display** with the system so SteamVR sees a connected HMD
2. Receives **head pose packets** from the headset over UDP (tracked at 120 Hz)
3. Captures the rendered stereo frame and **encodes** it via NVENC (NVIDIA), AMF (AMD), or VA-API (Intel/AMD open-source)
4. **Streams** the encoded frames to the headset over Wi-Fi (ideally Wi-Fi 6 at 5 GHz, dedicated AP)
5. **Receives** the headset's decoded display confirmations for frame pacing

### DRM Lease Integration on Linux

The virtual display that SteamVR drives is not a real connector. ALVR uses one of two approaches depending on the system configuration:

**Approach 1 — `wp_drm_lease_device_v1` with a dummy connector.** On Wayland compositors that support DRM lease, ALVR requests a lease for a dedicated dummy connector (created by the `drm_vkms` or `evdi` virtual KMS driver). SteamVR then drives this connector as if it were a real HMD, and ALVR intercepts the framebuffer presented to that connector.

**Approach 2 — `evdi` (Extensible Virtual Display Interface).** ALVR uses the [`evdi`](https://github.com/DisplayLink/evdi) kernel module (developed by DisplayLink, also used by the EVDI chapter of this book) to create a virtual `/dev/dri/cardN` device that exports framebuffers to userspace via `EVDI_GRAB_PIXELS`. SteamVR modesetting calls on this device are intercepted by the ALVR server:

```c
/* evdi userspace API — alvr_server captures frames from the virtual display */
evdi_handle display = evdi_open(evdi_find_available());
evdi_connect(display, &edid_data, edid_size, pixel_area);

struct evdi_buffer buf = {
    .id     = 0,
    .buffer = malloc(width * height * 4),
    .width  = width, .height = height, .stride = width * 4,
    .rects  = damage_rects, .rect_count = MAX_RECTS,
};
evdi_register_buffer(display, buf);

/* Main capture loop */
while (running) {
    evdi_handle_events(display, &event_ctx); /* fires update_ready_handler */
    evdi_grab_pixels(display, damage_rects, &rect_count);
    encode_and_stream(buf.buffer, width, height);
}
```

### Video Encoding and Latency Pipeline

ALVR's latency budget from render-complete to photons at the headset's panel:

| Stage | Typical latency | Notes |
|---|---|---|
| Frame capture from virtual display | 0.5–1 ms | `evdi_grab_pixels` or DRM lease framebuffer readback |
| Hardware encode (NVENC/AMF) | 2–5 ms | P-frame with low-latency preset; NVENC `NV_ENC_PRESET_P1` |
| Network transmission (Wi-Fi 6) | 2–6 ms | 1080p×2 at 150 Mbps; jitter is the main variable |
| Hardware decode (MediaCodec) | 2–4 ms | Android MediaCodec H.264/HEVC/AV1 on headset SoC |
| ATW reprojection on headset | 1–2 ms | Applied using latest pose, not the render-time pose |
| **Total motion-to-photon** | **~20–40 ms** | Native wired VR targets 11–15 ms; wireless adds ~10–20 ms |

The **pose prediction** pipeline is critical: by the time a frame reaches the headset display, 20–40 ms have elapsed since the GPU started rendering it. ALVR predicts the head pose at display time using a Kalman filter over the IMU data stream. SteamVR's ATW then applies a final correction using the most recent pose received just before display.

### Codec Comparison on Linux

| Codec | Encoder | Quality at 150 Mbps | Linux driver |
|---|---|---|---|
| H.264 | NVENC / AMF / VA-API | Good; universally supported on headsets | All major GPUs |
| HEVC (H.265) | NVENC / AMF / VA-API | Better quality per bit; Quest 2 requires sideload unlock | Most GPUs, see ch50 |
| AV1 | NVENC (RTX 4000+) / AMF (RX 7000+) | Best quality; lowest bitrate for same visual quality | RTX 4000+ / RX 7000+ |

AV1 encoding is the recommended choice on supported hardware: at 150 Mbps it delivers visibly better image quality than HEVC, and with hardware AV1 decoders on Quest 3 and Pico 4, the decode latency is equivalent to HEVC.

### Audio Routing

ALVR captures audio from the Linux system using PipeWire (see ch37). `alvr_server` creates a PipeWire sink node; ALVR registers it as the default audio output, so game audio flows into ALVR's encoder and is streamed alongside video as an AAC or Opus audio track to the headset's speakers or 3.5 mm jack.

```bash
# ALVR's PipeWire sink appears as a normal audio device
pw-cli list-objects | grep -A3 "alvr"
# id 45, type PipeWire:Interface:Node
#   node.name = "alvr_audio_sink"
#   media.class = "Audio/Sink"
```

### Linux-Specific Setup Notes

```bash
# Install ALVR server AppImage (from GitHub releases)
chmod +x ALVR_Launcher.AppImage && ./ALVR_Launcher.AppImage

# Ensure the user is in the video/render groups for DRM access
sudo usermod -aG video,render $USER

# For evdi-based virtual display:
sudo modprobe evdi initial_device_count=1
# ALVR will claim /dev/dri/card1 (or whichever evdi creates)

# Check encoding capability
vainfo 2>/dev/null | grep -i "h264\|hevc\|av1"      # VA-API (Intel/AMD open)
nvidia-smi | grep -i enc                              # NVENC availability
```

**Firewall note:** ALVR uses UDP port 9944 (control) and TCP port 9944 (streaming). On systems with `nftables`/`firewalld`, these must be opened on the Wi-Fi interface for the headset's IP.

### Relationship to Wired VR (SteamVR + Monado)

| | ALVR (wireless) | SteamVR direct (wired) | Monado (wired) |
|---|---|---|---|
| **Latency** | 20–40 ms total | 11–15 ms | 11–15 ms |
| **HMD connection** | Wi-Fi | USB-C / DisplayPort | USB-C / DisplayPort |
| **DRM lease usage** | Virtual connector via evdi | Real HMD connector | Real HMD connector |
| **Open source** | Yes (ALVR, Apache 2.0) | No (SteamVR proprietary) | Yes (Monado, BSL-1.0) |
| **OpenXR runtime** | SteamVR (via ALVR plugin) | SteamVR | Monado |
| **Supported HMDs** | Quest 2/3/Pro, Pico 4 | Index, Vive, Pimax | Index, Vive, WMR, Quest |

ALVR is the dominant choice for users who own a standalone headset and want PC VR on Linux without a USB tether. For wired setups, Monado (§4) is the preferred open-source path. SteamVR remains necessary for titles that use SteamVR-specific extensions not yet covered by Monado's OpenXR implementation.

---

## 7. Direct-to-Display Vulkan Swapchain

### VK_KHR_display: Vulkan's Window-System-Free Path

`VK_KHR_display` is the Vulkan extension that allows an application to enumerate physical displays and create swapchains without any window system involvement. It is the foundation of all VR direct display paths. Key entry points:

```c
/* Enumerate displays on a physical device */
VkResult vkGetPhysicalDeviceDisplayPropertiesKHR(
    VkPhysicalDevice              physicalDevice,
    uint32_t                     *pPropertyCount,
    VkDisplayPropertiesKHR       *pProperties);   /* name, planeReorderPossible, etc. */

/* Enumerate display modes (resolution + refresh rate combinations) */
VkResult vkGetDisplayModePropertiesKHR(
    VkPhysicalDevice              physicalDevice,
    VkDisplayKHR                  display,
    uint32_t                     *pPropertyCount,
    VkDisplayModePropertiesKHR   *pProperties);   /* displayMode, parameters */

/* Query which planes support a given display */
VkResult vkGetDisplayPlaneCapabilitiesKHR(
    VkPhysicalDevice              physicalDevice,
    VkDisplayModeKHR              mode,
    uint32_t                      planeIndex,
    VkDisplayPlaneCapabilitiesKHR *pCapabilities);

/* Create a surface for a specific display plane */
VkResult vkCreateDisplayPlaneSurfaceKHR(
    VkInstance                          instance,
    const VkDisplaySurfaceCreateInfoKHR *pCreateInfo,
    const VkAllocationCallbacks         *pAllocator,
    VkSurfaceKHR                        *pSurface);
```

The `VkDisplaySurfaceCreateInfoKHR` carries the display mode handle (resolution + refresh rate), the plane index, plane stacking order, transform, alpha mode, and image extent. [Source](https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_acquire_drm_display.html)

### Complete Direct-to-Display Path

The full sequence for a VR runtime to set up a direct Vulkan swapchain on an HMD, combining DRM lease with Vulkan display extensions:

```c
/* Step 1: Obtain lessee fd (via wp_drm_lease_device_v1 or drmModeCreateLease) */
int lessee_fd = /* ... from DRM_IOCTL_MODE_CREATE_LEASE or Wayland protocol ... */;

/* Step 2: Get VkDisplayKHR from DRM connector ID */
VkDisplayKHR hmd_display;
vkGetDrmDisplayEXT(physical_device, lessee_fd, hmd_connector_id, &hmd_display);

/* Step 3: Acquire exclusive Vulkan control */
vkAcquireDrmDisplayEXT(physical_device, lessee_fd, hmd_display);

/* Step 4: Choose display mode (resolution + refresh rate) */
uint32_t mode_count = 0;
vkGetDisplayModePropertiesKHR(physical_device, hmd_display, &mode_count, NULL);
VkDisplayModePropertiesKHR *modes =
    malloc(mode_count * sizeof(VkDisplayModePropertiesKHR));
vkGetDisplayModePropertiesKHR(physical_device, hmd_display, &mode_count, modes);
/* Select modes[i] with desired resolution and refresh rate */
VkDisplayModeKHR selected_mode = modes[target_mode].displayMode;

/* Step 5: Create surface */
VkDisplaySurfaceCreateInfoKHR surface_info = {
    .sType           = VK_STRUCTURE_TYPE_DISPLAY_SURFACE_CREATE_INFO_KHR,
    .displayMode     = selected_mode,
    .planeIndex      = 0,
    .planeStackIndex = 0,
    .transform       = VK_SURFACE_TRANSFORM_IDENTITY_BIT_KHR,
    .globalAlpha     = 1.0f,
    .alphaMode       = VK_DISPLAY_PLANE_ALPHA_OPAQUE_BIT_KHR,
    .imageExtent     = { hmd_width, hmd_height },
};
VkSurfaceKHR surface;
vkCreateDisplayPlaneSurfaceKHR(instance, &surface_info, NULL, &surface);

/* Step 6: Create swapchain on the HMD surface (standard VK_KHR_swapchain) */
VkSwapchainCreateInfoKHR swapchain_info = {
    .sType           = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR,
    .surface         = surface,
    .minImageCount   = 3,
    .imageFormat     = VK_FORMAT_B8G8R8A8_SRGB,
    .imageColorSpace = VK_COLOR_SPACE_SRGB_NONLINEAR_KHR,
    .imageExtent     = { hmd_width, hmd_height },
    .imageArrayLayers= 1,
    .imageUsage      = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT,
    .presentMode     = VK_PRESENT_MODE_FIFO_KHR,
    /* ... */
};
VkSwapchainKHR swapchain;
vkCreateSwapchainKHR(device, &swapchain_info, NULL, &swapchain);
```

### Mesa WSI Implementation

Mesa's WSI (Window System Integration) layer implements `VK_KHR_display` in `src/vulkan/wsi/wsi_common_display.c`. This file handles display enumeration, mode setting, and swapchain creation for the DRM/KMS direct path. The implementation uses `drmModeGetResources`, `drmModeGetConnector`, and the DRM atomic API internally to drive the display, translating Vulkan swapchain operations into KMS atomic commits.

The `VK_EXT_acquire_drm_display` implementation is in the same WSI layer and adds the lessee fd validation logic: when `vkAcquireDrmDisplayEXT` is called, the WSI layer calls `drmSetMaster` on the lessee fd (which will succeed only if the lessee fd is already the KMS master) and then proceeds with normal `VK_KHR_display` enumeration restricted to the leased connector. [Source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common_display.c)

---

## 8. Timing and Synchronisation for VR

### The Frame Timing Challenge

VR requires that each frame is presented at a specific VBLANK interval. Presenting a frame one VBLANK early wastes a frame of prediction; presenting a frame one VBLANK late causes the previous (potentially stale) frame to be displayed for two refresh periods, doubling the motion-to-photon latency for that frame and producing a visible stutter. The goal is to submit the KMS page flip (or Vulkan present) at the latest possible moment that still guarantees delivery at the target VBLANK.

### DRM VBLANK Events

The DRM subsystem delivers VBLANK notification via file descriptor events. After calling `DRM_IOCTL_MODE_ATOMIC` with `DRM_MODE_PAGE_FLIP_EVENT` set in the flags field, the kernel queues a `drm_event_vblank` on the DRM fd when the page flip completes:

```c
/* include/uapi/drm/drm.h */
struct drm_event_vblank {
    struct drm_event    base;         /* .type = DRM_EVENT_FLIP_COMPLETE */
    __u64               user_data;    /* value from drmModePageFlip flags */
    __u32               tv_sec;       /* timestamp: seconds */
    __u32               tv_usec;      /* timestamp: microseconds */
    __u32               sequence;     /* VBLANK counter value */
    __u32               crtc_id;      /* which CRTC */
};
```

The `tv_sec` / `tv_usec` timestamp is sourced from `CLOCK_MONOTONIC` at the moment the VBLANK interrupt fires in the kernel. This timestamp is the authoritative ground truth for when the previous frame was displayed, and it is the input to the VR runtime's frame predictor. [Source](https://egeeks.github.io/kernal/drm/ch02s03.html)

### VK_EXT_present_timing

`VK_EXT_present_timing` is a Vulkan extension that allows applications to query accurate present timestamps from the Vulkan presentation engine and to schedule a present to occur no earlier than a specified time. It was merged into the Vulkan specification in November 2025 as part of Vulkan 1.4.335, after approximately five years of development with contributions from Google, NVIDIA, AMD, Collabora, Samsung, Unity, and Red Hat. [Source](https://www.khronos.org/blog/vk-ext-present-timing-the-journey-to-state-of-the-art-frame-pacing-in-vulkan)

The extension supersedes the earlier `VK_GOOGLE_display_timing` extension (which provided `vkGetRefreshCycleDurationGOOGLE` / `vkGetPastPresentationTimingGOOGLE`) with a more complete and portable API. Key entry points defined by `VK_EXT_present_timing`:

- `vkGetSwapchainTimingPropertiesEXT`: query the swapchain's display timing capabilities and refresh cycle properties
- `vkGetSwapchainTimeDomainPropertiesEXT`: query available time domains for correlating CPU and GPU timestamps
- `vkSetSwapchainPresentTimingQueueSizeEXT`: configure the depth of the timing history ring buffer
- `vkGetPastPresentationTimingEXT`: retrieve recorded timestamps for recently completed presents (`VkPastPresentationTimingEXT`)
- `VkPresentTimingsInfoEXT` in the `vkQueuePresentKHR` `pNext` chain: schedule a present to occur no earlier than a target time, with per-swapchain-image `VkPresentTimingInfoEXT` entries

The `VkPresentStageTimeEXT` structure reports timestamps at multiple pipeline stages (`VK_PRESENT_STAGE_QUEUE_OPERATIONS_COMPLETE_BIT_EXT`, `VK_PRESENT_STAGE_IMAGE_LATCHED_BIT_EXT`, `VK_PRESENT_STAGE_IMAGE_PRESENTED_BIT_EXT`), giving runtimes precise visibility into where latency accumulates. [Source](https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_present_timing.html)

Driver support as of 2026: NVIDIA's proprietary Linux Vulkan driver has supported `VK_EXT_present_timing` since its ratification. Mesa 26.1 added support for X11 and Wayland paths in RADV and ANV. [Source](https://www.linux.org/threads/phoronix-vulkan-vk_ext_present_timing-merged-to-mesa-26-1-for-x11-wayland.61731/)

For the KMS direct display path relevant to VR, `VK_EXT_present_timing` maps the "present no earlier than" constraint to a targeted KMS atomic commit: the Mesa WSI layer computes the required page flip time and uses `drmWaitVBlank` with sequence targeting to defer the commit until the correct VBLANK interval.

### Monado's Frame Timing Architecture

Monado's `u_timing_frame` module (in `src/xrt/auxiliary/timing/`) implements a predictive frame pacing algorithm. The timing module provides three core operations to the compositor: predicting the target VBLANK timestamp for the next frame (so the compositor knows what pose timestamp to sample and when to start rendering), recording the actual GPU completion time when the render finishes, and receiving the actual display timestamp from `VK_EXT_present_timing` or `DRM_EVENT_FLIP_COMPLETE` after the frame is shown.

The algorithm works as follows:

1. At startup, it measures GPU render time over several frames to establish a baseline.
2. Before each frame, the timing module computes: the target display time (the VBLANK the frame should hit), the predicted pose time (slightly before the VBLANK to account for display propagation delay), and the render start time (working backward from the VBLANK by predicted GPU render time plus a configurable safety margin, defaulting to 2 ms).
3. After each frame completes, the timing module receives the actual GPU completion time and the actual present timestamp, and computes the deviation from prediction.
4. The predictor adjusts its GPU time estimate using an exponentially-weighted moving average, ensuring subsequent frames start at the correct time.

This feedback loop allows Monado to keep frame delivery consistently within 1–2 ms of the target VBLANK even under variable GPU load. [Source](https://monado.freedesktop.org/getting-started.html)

### GPU Queue Priority

Modern VR runtimes create their Vulkan queues with `VK_QUEUE_GLOBAL_PRIORITY_HIGH_EXT` (from `VK_EXT_global_priority`). On AMD (via AMDGPU's GPU scheduler) and Intel (via i915's GuC submission), high-priority queues preempt lower-priority workloads at submission boundaries. This prevents desktop GPU work — Chrome's GPU process, video decode, etc. — from blocking the VR compositor's critical-path render pass. Monado explicitly requests high-priority queues for its ATW pass:

```c
/* Monado ATW queue creation — high priority for time-critical reprojection */
VkDeviceQueueGlobalPriorityCreateInfoEXT priority_info = {
    .sType          = VK_STRUCTURE_TYPE_DEVICE_QUEUE_GLOBAL_PRIORITY_CREATE_INFO_EXT,
    .globalPriority = VK_QUEUE_GLOBAL_PRIORITY_HIGH_EXT,
};
```

---

## 9. Practical: Setting Up VR on Linux

### Hardware Compatibility

| Headset | Connection | Linux Status |
|---|---|---|
| Valve Index | DisplayPort + USB 3.0 | Full support (SteamVR + Monado) |
| HTC Vive / Vive Pro | HDMI + USB 2.0 | Full support via SteamVR |
| Meta Quest 2/3 (wired) | USB-C (adb + ALVR) | Requires ALVR streaming client |
| Meta Quest 2/3 (wireless) | Wi-Fi 6 + ALVR | Works via ALVR server on host |
| PlayStation VR2 | USB-C (limited) | Basic display; no tracking support |
| Varjo headsets | DisplayPort | Supported via Monado |

### Prerequisites

```bash
# Add user to video group for DRM access without root
sudo usermod -a -G video $USER
# (Log out and back in for group membership to take effect)

# Verify DRM lease capability
ls -la /dev/dri/card*
# Confirm kernel ≥ 4.15 for DRM lease support
uname -r

# Install Monado (Ubuntu/Debian)
sudo apt install monado monado-utils

# Set Monado as the active OpenXR runtime
mkdir -p ~/.config/openxr/1
cat > ~/.config/openxr/1/active_runtime.json << 'EOF'
{
    "file_format_version": "1.0.0",
    "runtime": {
        "library_path": "/usr/lib/x86_64-linux-gnu/libopenxr_monado.so"
    }
}
EOF
```

### Identifying HMD Connectors

```bash
# List all DRM connectors and check for non-desktop property
for conn in /sys/class/drm/card*-*/; do
    name=$(basename $conn)
    nd=$(cat "$conn/non_desktop" 2>/dev/null)
    if [ "$nd" = "1" ]; then
        echo "HMD connector: $name"
    fi
done

# Alternatively, use drm_info:
drm_info | grep -A5 "non-desktop"

# Check EDID vendor for Valve Index identification:
edid-decode /sys/class/drm/card0-DP-2/edid 2>/dev/null | grep "Manufacturer"
# Should show: Manufacturer: VLV
```

### Running VR Under Different Compositor Setups

**Under wlroots/Sway (recommended)**:
```bash
# wp_drm_lease_device_v1 is supported natively
# Launch Monado service — it will request the lease from Sway automatically
monado-service &
# Launch an OpenXR application
/path/to/openxr_app
```

**Under KDE Plasma 6 (KWin)**:
```bash
# Ensure Plasma 6.0+ is installed (KWin 6.x includes drm-lease-v1 support)
# No special configuration needed; Monado or SteamVR will auto-detect
monado-service &
```

**Under GNOME 47+ (Mutter)**:
```bash
# GNOME 47+ is required; GNOME 47.2+ for stable DRM lease support
# Verify Mutter version:
mutter --version

# If using SteamVR: launch from Steam and select Valve Index / HTC Vive
# If using Monado:
XRT_COMPOSITOR_LOG=debug monado-service 2>&1 | tee /tmp/monado.log
```

**KMS direct mode (no compositor)**:
```bash
# Required when running without a desktop compositor, or on unpatched systems
# Must be in 'video' group or run as root
XRT_COMPOSITOR_FORCE_VK_DISPLAY=1 monado-service
```

### Debugging

```bash
# Monado verbose logging
XRT_COMPOSITOR_LOG=debug XRT_PRINT_OPTIONS=1 monado-service

# Inspect DRM lease fd info (find the monado-service PID first)
PID=$(pgrep monado-service)
cat /proc/$PID/fdinfo/* 2>/dev/null | grep -A5 "drm-"

# Check which connectors are leased:
# (requires root or being the DRM master)
drm_info  # shows lease status per connector

# SteamVR log location
cat ~/.local/share/Steam/logs/vrserver.txt | grep -i "lease\|drm\|direct"

# Monitor VBLANK events on the lessee fd (for debugging timing):
# Use evtest or a custom tool reading DRM_EVENT_FLIP_COMPLETE from the lease fd

# Check lighthouse tracking
systemctl --user status lighthouse-watchman

# ALVR for Meta Quest wireless streaming setup
# Install ALVR from Flathub:
flatpak install flathub alvr
# Set ALVR as OpenXR runtime in ALVR dashboard, then launch:
alvr_dashboard
```

### USB-C DisplayPort and MST

Some HMDs (notably newer Varjo headsets and some Pimax models) connect via USB-C DisplayPort Alternate Mode. On Linux, USB-C DP Alt Mode output is handled by the `ucsi_acpi` or `ucsi_ccg` driver for Type-C port controllers, which ultimately presents as a standard DisplayPort connector to DRM. If the USB-C dock or hub involves a DisplayPort MST (Multi-Stream Transport) hub, the `drm_dp_mst_topology_mgr` in the DRM core manages virtual connector allocation — each downstream MST port appears as a separate DRM connector, and the VR runtime identifies the HMD among them by the usual `non-desktop` property.

```bash
# Check for MST topology
dmesg | grep -i "mst\|multi-stream"
# List MST connectors
drm_info | grep -i "mst"
```

---

## Roadmap

### Near-term (6–12 months)

- **`VK_GOOGLE_display_timing` direct display support (Mesa 26.2)**: Mesa 26.2 is landing `VK_GOOGLE_display_timing` support for `VK_KHR_display` (the direct-to-display path used by VR runtimes), built on top of the `VK_EXT_present_timing` foundation merged in Mesa 26.1. This allows VR runtimes to query per-frame display timestamps and tune their composition timing loops without relying on estimated VBLANK intervals. [Source](https://www.phoronix.com/news/VK-Google-Timing-KHR-Display)
- **NVIDIA proprietary driver DRM lease latency fix**: A significant regression where NVIDIA's proprietary userland injects an extra compositing pass into the DRM lease path — making VR effectively unusable on NVIDIA hardware — remains unresolved as of driver series 570. Community pressure and NVIDIA developer forum threads are ongoing; a targeted driver fix is anticipated in a 2026 release cycle, though no public commitment exists. [Source](https://forums.developer.nvidia.com/t/substantial-drm-lease-presentation-latency-resulting-in-unusable-vr-hmd-experience/332386)
- **Monado `xrt_future` and `XR_EXT_future` extension**: Monado 25.1.0 introduced the `xrt_future` internal system and initial support for the `XR_EXT_future` OpenXR extension, enabling asynchronous operation completion that improves compositor scheduling under the DRM lease path. Further integration with the direct-display compositor backend is expected through 2026. [Source](https://www.collabora.com/news-and-blog/news-and-events/monado-25-1-0-enabling-tomorrows-openxr-experiences.html)
- **Broader `VK_EXT_present_timing` VR adoption**: Now that `VK_EXT_present_timing` is supported across RADV, ANV, NVK, Turnip, and PanVK in Mesa 26.1, VR runtimes including Monado are expected to update their Vulkan composition backends to use presentation deadline scheduling rather than the older `VK_GOOGLE_display_timing` heuristics. [Source](https://www.phoronix.com/forums/forum/linux-graphics-x-org-drivers/vulkan/1608746-vulkan-vk_ext_present_timing-merged-to-mesa-26-1-for-x11-wayland)

### Medium-term (1–3 years)

- **Monado as common XR foundation for vendor runtimes**: As of early 2026, Monado has been adopted as the upstream foundation by multiple commercial XR platforms including Google AndroidXR, NVIDIA CloudXR, Pico, and Qualcomm Snapdragon Spaces. This growing vendor adoption is expected to drive upstreaming of proprietary DRM lease improvements — particularly around multi-GPU heterogeneous display topologies and wireless streaming compositor paths — back into the open-source codebase. [Source](https://www.gamingonlinux.com/2026/03/monado-becomes-the-open-source-foundation-for-various-openxr-vr-vendors/)
- **`wp_drm_lease_device_v1` promotion from staging**: The DRM lease Wayland protocol has been in `staging` status since its merge into wayland-protocols in 2021. As compositor adoption matures — KWin/Plasma and Mutter/GNOME both now implement it — a proposal to graduate the protocol to `stable` within wayland-protocols is a natural next step. Note: needs verification of specific timeline.
- **OpenXR hand tracking and eye tracking integration with DRM lease path**: The `XR_EXT_hand_tracking_data_source` extension (landed in Monado 25.1.0) and forthcoming eye-tracking extensions require low-latency sensor data to reach the compositor's ATW (Asynchronous TimeWarp) pass. Tighter integration between these input paths and the DRM lease compositor — including priority-boosted Vulkan queue submission for sensor-driven reprojection — is an active design area in the Monado compositor subsystem. Note: needs verification.
- **VRR (Variable Refresh Rate) on leased CRTCs**: Some VR HMDs support VRR modes that reduce judder when frames run long. Extending the kernel's `VRR_ENABLED` CRTC property path to cooperate correctly with the DRM lease hierarchy (so a lessee can negotiate VRR without requiring lessor involvement on each mode change) is a pending design discussion in the DRM mailing list community. Note: needs verification of specific patchset status.

### Long-term

- **SR-IOV and multi-tenant GPU VR**: As AMD and Intel extend SR-IOV (Single Root I/O Virtualisation) support in their open-source GPU drivers, a hybrid model may emerge where VR workloads run in a lightweight VM partition sharing the GPU with the host desktop via SR-IOV, eliminating the need for DRM lease entirely in favour of hardware-enforced resource partitioning. DRM lease would remain relevant for non-SR-IOV hardware indefinitely. Note: needs verification.
- **Wireless VR compositor path standardisation**: The ALVR and WiVRn projects provide wireless streaming for standalone headsets (Meta Quest, Pico) over Wi-Fi 6/6E. A Wayland protocol extension for signalling streaming compositor intent — and coordinating with the DRM lease layer to select appropriate encoder clock domains — has been discussed in the vronlinux community but has no formal proposal yet. Note: needs verification.
- **Kernel-side lease revocation improvements**: The current DRM lease revocation path (`DRM_IOCTL_MODE_REVOKE_LEASE`) is synchronous and does not provide the lessee with a grace period to complete in-flight page flips. A more cooperative revocation protocol — potentially using a new kernel uapi notification event on the lease fd — has been noted as a desirable improvement for clean HMD hot-unplug handling, but no patchset has been posted. Note: needs verification.

---

## Integrations

**Chapter 2 (KMS: The Display Pipeline)**: The DRM lease mechanism operates entirely within the KMS object model. The CRTC, connector, and plane IDs passed to `DRM_IOCTL_MODE_CREATE_LEASE` are the same KMS object IDs that Chapter 2 describes in Section 1. The lessee's atomic commits (`DRM_IOCTL_MODE_ATOMIC`) and page flip events (`DRM_EVENT_FLIP_COMPLETE`) use the same kernel paths described in Chapter 2, Sections 3 and 6. Understanding the KMS object lifecycle and the atomic commit transaction model is prerequisite knowledge for working with DRM lease.

**Chapter 20 (Wayland Protocol Architecture)**: The `wp_drm_lease_device_v1` protocol is a Wayland extension protocol. Chapter 20 covers the Wayland protocol wire format, global advertisement, and object lifecycle rules that underpin the lease protocol's `wp_drm_lease_device_v1` global, `create_lease_request` sequence, and `finished` event semantics.

**Chapter 21 (wlroots)**: wlroots implements `wp_drm_lease_device_v1` via `wlr_drm_lease_v1.c` and `wlr_drm_lease_v1.h`. Chapter 21 covers wlroots compositor authoring, including how to wire up `wlr_drm_lease_manager_v1` in a compositor and handle lease grant/deny decisions. The `wlr_output` abstraction and its relationship to DRM CRTCs is the correct context for understanding which objects to exclude from the compositor's own output management when a connector is leased.

**Chapter 24 (Vulkan for Application Developers)**: `VK_EXT_acquire_drm_display`, `VK_KHR_display`, `vkCreateDisplayPlaneSurfaceKHR`, and `VK_EXT_present_timing` are Vulkan extensions used directly by VR runtimes. Chapter 24 covers the Vulkan instance extension model, physical device feature queries, and swapchain creation — the necessary context for understanding how the VR-specific extensions plug into the standard Vulkan API.

**Chapter 25 (OpenXR and Monado)**: Chapter 25 covers Monado's full architecture: the XRT (XR Runtime) abstraction layer, the IPC protocol between `monado-service` and client applications, the distortion mesh computation, and the OpenXR session lifecycle. The DRM lease path described in this chapter is one of several display backends that Monado's compositor can use; Chapter 25 provides the broader context of how Monado selects and manages those backends.

**Chapter 75 (Explicit GPU Synchronisation)**: VR runtimes make heavy use of `VK_EXT_external_fence_fd` and `VK_KHR_timeline_semaphore` to synchronise between the render queue, the ATW queue, and the KMS page flip. The `IN_FENCE_FD` KMS property (which accepts a `sync_file` fd) is used by Monado's compositor to wait for the GPU render to complete before the kernel initiates the page flip. Chapter 75 covers the full `sync_file` / `dma-fence` / DRM fence chain in depth.

**Chapter 112 (Variable Refresh Rate)**: Some VR headsets support Variable Refresh Rate (VRR) modes, where the display refresh interval adapts to the render time. For ATW reprojection, VRR is particularly valuable: if a frame runs slightly long, VRR allows the display to extend the current frame's hold time rather than double-displaying the previous frame. The VRR KMS infrastructure — the `VRR_ENABLED` CRTC property and the `drm_crtc_info` VRR range — covered in Chapter 112 applies directly to leased CRTCs.

---

## References

1. [DRM Display Resource Leasing](https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html) — Official kernel documentation on the DRM lease API and the `drm_master` lease hierarchy
2. [`drivers/gpu/drm/drm_lease.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_lease.c) — Kernel implementation: `drm_mode_create_lease_ioctl`, `drm_lease_held`, revocation and cleanup
3. [`include/uapi/drm/drm_mode.h`](https://elixir.bootlin.com/linux/latest/source/include/uapi/drm/drm_mode.h) — `struct drm_mode_create_lease`, `struct drm_mode_list_lessees`, `struct drm_mode_get_lease`, `struct drm_mode_revoke_lease` — the four lease ioctl structs
4. [Keith Packard — DRM Lease original patch series v4](https://lore.kernel.org/lkml/20171005061310.29919-7-keithp@keithp.com/) — The lkml submission introducing all four lease ioctls, with design rationale
5. [wp_drm_lease_device_v1 Protocol Specification](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/staging/drm-lease/drm-lease-v1.xml) — Wayland protocol XML; defines all four interfaces and their event/request sequences
6. [Wayland Explorer: drm-lease-v1](https://wayland.app/protocols/drm-lease-v1) — Protocol reference with compositor implementation matrix
7. [VK_EXT_acquire_drm_display — Khronos Registry](https://khronos.org/registry/vulkan/specs/1.3-extensions/man/html/VK_EXT_acquire_drm_display.html) — `vkGetDrmDisplayEXT` and `vkAcquireDrmDisplayEXT` specification
8. [VK_KHR_display — Khronos Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_display.html) — Direct-to-display Vulkan swapchain: `vkGetPhysicalDeviceDisplayPropertiesKHR`, `vkCreateDisplayPlaneSurfaceKHR`
9. [VK_EXT_present_timing — Khronos Blog](https://www.khronos.org/blog/vk-ext-present-timing-the-journey-to-state-of-the-art-frame-pacing-in-vulkan) — Extension rationale, history, and API overview; merged Vulkan 1.4.335 (November 2025)
10. [Mesa WSI display implementation](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common_display.c) — `wsi_common_display.c`: RADV/ANV implementation of `VK_KHR_display` and `VK_EXT_acquire_drm_display`
11. [Monado Direct Mode Documentation](https://monado.freedesktop.org/direct-mode.html) — Monado's three display acquisition paths: Wayland DRM lease, X11 direct, KMS direct
12. [Monado comp_window_direct_wayland.c](https://monado.pages.freedesktop.org/monado/comp__window__direct__wayland_8c.html) — Doxygen reference for Monado's Wayland DRM lease compositor window code
13. [Monado Getting Started](https://monado.freedesktop.org/getting-started.html) — Environment variables, runtime configuration, and `XRT_COMPOSITOR_LOG` debugging
14. [GNOME Merges Wayland DRM Lease](https://gamingonlinux.com/2024/06/drm-lease-protocol-support-finally-merged-for-gnome-wayland-great-for-vr-fans/) — Coverage of Mutter MR !3739 (GNOME 47, merged June 2024)
15. [SteamVR Linux Issues — DRM Lease](https://github.com/ValveSoftware/SteamVR-for-Linux/issues/753) — GNOME 47 + Mutter 47 DRM lease failure bug tracker; contains workarounds
16. [VR on Linux Community Guide](https://steamcommunity.com/sharedfiles/filedetails/?id=2984005943) — Community hardware and software compatibility matrix
17. [Keith Packard: Vulkan Direct Display Extensions into RADV/ANV](https://www.phoronix.com/news/KeithP-Vulkan-Direct-Display) — Phoronix coverage of Keith Packard's Mesa patches adding VK_EXT_acquire_drm_display and VK_KHR_display to RADV and ANV
18. [wlroots wlr_drm_lease_v1.c](https://github.com/swaywm/wlroots/blob/master/types/wlr_drm_lease_v1.c) — Reference implementation of `wp_drm_lease_device_v1` on the compositor side
19. [`drivers/gpu/drm/drm_edid.c`](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_edid.c) — EDID quirk table with `EDID_QUIRK_NON_DESKTOP` entries for Valve Index ("VLV"), Sony PSVR, Oculus, Varjo, and other HMDs
20. [VBlank Event Handling — DRM HOWTO](https://egeeks.github.io/kernal/drm/ch02s03.html) — `DRM_EVENT_FLIP_COMPLETE`, `drm_event_vblank` structure, and VBLANK event delivery mechanics
21. [NVIDIA DRM Lease Issues — Developer Forums](https://forums.developer.nvidia.com/t/nvidia-proprietary-non-open-modules-completely-unable-to-acquire-a-drm-lease-on-any-display-server-all-known-nvidia-drivers-any-hardware/341244) — Known limitations of NVIDIA proprietary driver with standard DRM lease ioctls
22. [Mesa 26.1 VK_EXT_present_timing Support](https://www.linux.org/threads/phoronix-vulkan-vk_ext_present_timing-merged-to-mesa-26-1-for-x11-wayland.61731/) — Mesa 26.1 adds VK_EXT_present_timing for X11 and Wayland in RADV and ANV
23. [libdrm DRM lease wrappers](https://gitlab.freedesktop.org/mesa/drm/-/blob/main/xf86drm.h) — `drmModeCreateLease`, `drmModeListLessees`, `drmModeGetLease`, `drmModeRevokeLease` declarations in libdrm ≥ 2.4.89

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
