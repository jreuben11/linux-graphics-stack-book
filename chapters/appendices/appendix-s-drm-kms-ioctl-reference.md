# Appendix S: DRM/KMS ioctl Quick Reference

> **Reference baseline**: Linux kernel 6.12 / libdrm 2.4.123 / June 2026
> **Primary sources**: `include/uapi/drm/drm.h` | `include/uapi/drm/drm_mode.h` | [kernel.org DRM doc](https://www.kernel.org/doc/html/latest/gpu/)

**Audience**: Systems and driver developers implementing compositors, display servers, DRM clients, or kernel drivers. See Chapter 1 (DRM Architecture), Chapter 3 (KMS), and Chapter 4 (GEM/DMA-BUF) for the architectural context behind each ioctl.

All ioctls are issued via `drmIoctl(fd, request, arg)` (libdrm wrapper around `ioctl(2)`). The `fd` is a DRM device node opened via `open("/dev/dri/cardN", O_RDWR)` (master) or `open("/dev/dri/renderDN", O_RDWR)` (render node, non-privileged).

---

## Table of Contents

1. [Device and Authentication](#1-device-and-authentication)
2. [KMS Object Discovery](#2-kms-object-discovery)
3. [KMS Atomic Commit](#3-kms-atomic-commit)
4. [Legacy KMS (non-atomic)](#4-legacy-kms-non-atomic)
5. [Framebuffer Management](#5-framebuffer-management)
6. [GEM Buffer Management](#6-gem-buffer-management)
7. [PRIME — DMA-BUF Import/Export](#7-prime--dma-buf-importexport)
8. [Dumb Buffers (software rendering)](#8-dumb-buffers-software-rendering)
9. [Syncobj — GPU Synchronisation](#9-syncobj--gpu-synchronisation)
10. [Vblank and Timestamp](#10-vblank-and-timestamp)
11. [KMS Object Properties Reference](#11-kms-object-properties-reference)

---

## 1. Device and Authentication

| ioctl / function                           | libdrm wrapper                  | Description                                      |
|--------------------------------------------|---------------------------------|--------------------------------------------------|
| `DRM_IOCTL_VERSION`                        | `drmGetVersion()`               | Query driver name, version, date, description    |
| `DRM_IOCTL_GET_CAP`                        | `drmGetCap(fd, cap, &val)`      | Query driver capability (`DRM_CAP_*`)            |
| `DRM_IOCTL_SET_CLIENT_CAP`                 | `drmSetClientCap(fd, cap, val)` | Enable client capability (e.g. atomic, universal planes) |
| `DRM_IOCTL_GET_UNIQUE`                     | `drmGetBusid()`                 | Query DRM bus ID                                 |
| `DRM_IOCTL_AUTH_MAGIC`                     | `drmAuthMagic()`                | Authenticate a secondary client (legacy)         |
| `DRM_IOCTL_SET_MASTER` / `DROP_MASTER`     | `drmSetMaster()` / `drmDropMaster()` | Acquire/release DRM master privilege        |

**Key `DRM_CAP_*` values:**

| Capability                           | Value  | Description                                     |
|--------------------------------------|--------|-------------------------------------------------|
| `DRM_CAP_DUMB_BUFFER`                | `0x1`  | Dumb buffer support (all drivers)               |
| `DRM_CAP_VBLANK_HIGH_CRTC`           | `0x2`  | High CRTC index for vblank events               |
| `DRM_CAP_PRIME`                      | `0x5`  | PRIME DMA-BUF import/export (`DRM_PRIME_CAP_*`) |
| `DRM_CAP_TIMESTAMP_MONOTONIC`        | `0x6`  | Vblank timestamps use `CLOCK_MONOTONIC`         |
| `DRM_CAP_ASYNC_PAGE_FLIP`            | `0x7`  | Async page flip (tears, but low latency)        |
| `DRM_CAP_CURSOR_WIDTH/HEIGHT`        | `0x8/9`| Maximum cursor plane dimensions                 |
| `DRM_CAP_ADDFB2_MODIFIERS`           | `0x10` | `drmModeAddFB2WithModifiers` supported          |
| `DRM_CAP_PAGE_FLIP_TARGET`           | `0x11` | Target vblank for page flip                     |
| `DRM_CAP_SYNCOBJ`                    | `0x13` | Syncobj support                                 |
| `DRM_CAP_SYNCOBJ_TIMELINE`           | `0x14` | Timeline syncobj support                        |

**`DRM_CLIENT_CAP_*` values (must be set before use):**

| Client capability                       | Enables                                          |
|-----------------------------------------|--------------------------------------------------|
| `DRM_CLIENT_CAP_STEREO_3D`             | Stereo 3D display modes                          |
| `DRM_CLIENT_CAP_UNIVERSAL_PLANES`      | All planes visible (not just overlay)            |
| `DRM_CLIENT_CAP_ATOMIC`                | Atomic modesetting ioctls                        |
| `DRM_CLIENT_CAP_ASPECT_RATIO`          | Picture aspect ratio flags in modes              |
| `DRM_CLIENT_CAP_WRITEBACK_CONNECTORS`  | Writeback connector visibility                   |

---

## 2. KMS Object Discovery

```c
// Get all resources (CRTCs, encoders, connectors, framebuffers)
drmModeRes *res = drmModeGetResources(fd);
// res->crtcs[], res->connectors[], res->encoders[], res->fbs[]

// Get connector detail
drmModeConnector *conn = drmModeGetConnector(fd, conn_id);
// conn->connection: DRM_MODE_CONNECTED / DISCONNECTED / UNKNOWNCONNECTION
// conn->modes[]: drmModeModeInfo array
// conn->props[], conn->prop_values[]: property IDs and current values

// Get CRTC state
drmModeCrtc *crtc = drmModeGetCrtc(fd, crtc_id);

// Get encoder (legacy: links CRTC to connector)
drmModeEncoder *enc = drmModeGetEncoder(fd, enc_id);
// enc->possible_crtcs: bitmask of valid CRTCs for this encoder

// Get all planes (requires DRM_CLIENT_CAP_UNIVERSAL_PLANES)
drmModePlaneRes *planes = drmModeGetPlaneResources(fd);
drmModePlane *plane = drmModeGetPlane(fd, plane_id);

// Get object properties
drmModeObjectProperties *props = drmModeObjectGetProperties(fd, obj_id, obj_type);
// obj_type: DRM_MODE_OBJECT_CRTC / CONNECTOR / PLANE / FB / ENCODER

// Get property definition (name, type, enum values)
drmModePropertyRes *prop = drmModeGetProperty(fd, prop_id);

// Free all resources
drmModeFreeResources(res);
drmModeFreeConnector(conn);
// etc.
```

**Object types for `DRM_MODE_OBJECT_*`:**

| Constant                   | Value    | Object                  |
|----------------------------|----------|-------------------------|
| `DRM_MODE_OBJECT_CRTC`     | `0xcccccccc` | CRTC                |
| `DRM_MODE_OBJECT_CONNECTOR`| `0xc0c0c0c0` | Connector           |
| `DRM_MODE_OBJECT_ENCODER`  | `0xe0e0e0e0` | Encoder             |
| `DRM_MODE_OBJECT_MODE`     | `0xdededede` | Display mode        |
| `DRM_MODE_OBJECT_PROPERTY` | `0xb0b0b0b0` | Property            |
| `DRM_MODE_OBJECT_FB`       | `0xfbfbfbfb` | Framebuffer         |
| `DRM_MODE_OBJECT_BLOB`     | `0xbbbbbbbb` | Blob (opaque data)  |
| `DRM_MODE_OBJECT_PLANE`    | `0xeeeeeeee` | Plane               |
| `DRM_MODE_OBJECT_ANY`      | `0`      | Any type (for lookup)   |

---

## 3. KMS Atomic Commit

Atomic modesetting allows updating multiple KMS objects in a single, all-or-nothing transaction. Requires `DRM_CLIENT_CAP_ATOMIC`.

```c
// Build a property request
drmModeAtomicReq *req = drmModeAtomicAlloc();

// Add property changes: drmModeAtomicAddProperty(req, obj_id, prop_id, value)
drmModeAtomicAddProperty(req, crtc_id,      active_prop_id,       1);
drmModeAtomicAddProperty(req, crtc_id,      mode_id_prop_id,      mode_blob_id);
drmModeAtomicAddProperty(req, conn_id,      crtc_id_prop_id,      crtc_id);
drmModeAtomicAddProperty(req, plane_id,     crtc_id_prop_id,      crtc_id);
drmModeAtomicAddProperty(req, plane_id,     fb_id_prop_id,        fb_id);
drmModeAtomicAddProperty(req, plane_id,     src_x_prop_id,        0);
drmModeAtomicAddProperty(req, plane_id,     src_y_prop_id,        0);
drmModeAtomicAddProperty(req, plane_id,     src_w_prop_id,        width  << 16);  // 16.16 fixed
drmModeAtomicAddProperty(req, plane_id,     src_h_prop_id,        height << 16);
drmModeAtomicAddProperty(req, plane_id,     crtc_x_prop_id,       0);
drmModeAtomicAddProperty(req, plane_id,     crtc_y_prop_id,       0);
drmModeAtomicAddProperty(req, plane_id,     crtc_w_prop_id,       width);
drmModeAtomicAddProperty(req, plane_id,     crtc_h_prop_id,       height);

// Commit
uint32_t flags = DRM_MODE_ATOMIC_NONBLOCK | DRM_MODE_PAGE_FLIP_EVENT;
// flags options:
//   DRM_MODE_ATOMIC_TEST_ONLY   — validate without applying (dry run)
//   DRM_MODE_ATOMIC_NONBLOCK    — return immediately, get event via drmHandleEvent
//   DRM_MODE_ATOMIC_ALLOW_MODESET — permit full modesetting (first frame, VT switch)
//   DRM_MODE_PAGE_FLIP_EVENT    — generate DRM_EVENT_FLIP_COMPLETE event

int ret = drmModeAtomicCommit(fd, req, flags, user_data);
drmModeAtomicFree(req);
```

**Mode blob creation:**
```c
uint32_t mode_blob_id;
drmModeCreatePropertyBlob(fd, &mode_info, sizeof(drmModeModeInfo), &mode_blob_id);
// ... use in atomic commit ...
drmModeDestroyPropertyBlob(fd, mode_blob_id);
```

**Waiting for flip completion:**
```c
fd_set fds;
FD_SET(drm_fd, &fds);
select(drm_fd + 1, &fds, NULL, NULL, NULL);

drmEventContext evctx = {
    .version = DRM_EVENT_CONTEXT_VERSION,
    .page_flip_handler2 = on_page_flip,  // (fd, seq, sec, usec, crtc_id, user_data)
};
drmHandleEvent(drm_fd, &evctx);
```

---

## 4. Legacy KMS (non-atomic)

Use atomic instead where possible. Legacy ioctls remain for simple single-CRTC cases or older kernels.

| libdrm function                                    | Description                                           |
|----------------------------------------------------|-------------------------------------------------------|
| `drmModeSetCrtc(fd, crtc_id, fb_id, x, y, conn_ids, n_conns, mode)` | Set display mode + framebuffer |
| `drmModePageFlip(fd, crtc_id, fb_id, flags, user_data)` | Async page flip (generates event)           |
| `drmModeSetCursor(fd, crtc_id, bo_handle, w, h)`  | Set cursor BO                                         |
| `drmModeMoveCursor(fd, crtc_id, x, y)`            | Move cursor plane                                     |
| `drmModeSetPlane(fd, plane_id, crtc_id, fb_id, flags, crtc_x,y,w,h, src_x,y,w,h)` | Legacy plane set |

---

## 5. Framebuffer Management

```c
// Add framebuffer (format + modifier aware)
struct drm_mode_fb_cmd2 cmd = {
    .width  = w,
    .height = h,
    .pixel_format = DRM_FORMAT_XRGB8888,  // from drm_fourcc.h
    .flags  = DRM_MODE_FB_MODIFIERS,       // set when modifier != DRM_FORMAT_MOD_LINEAR
    .handles[0] = gem_handle,
    .pitches[0] = stride,
    .offsets[0] = 0,
    .modifier[0] = modifier,              // DRM_FORMAT_MOD_LINEAR or tiling modifier
};
drmModeAddFB2(fd, w, h, DRM_FORMAT_XRGB8888, cmd.handles, cmd.pitches,
              cmd.offsets, &fb_id, 0);
// With modifiers:
drmModeAddFB2WithModifiers(fd, w, h, fmt, handles, pitches, offsets,
                           modifiers, &fb_id, DRM_MODE_FB_MODIFIERS);

// Remove framebuffer
drmModeRmFB(fd, fb_id);

// Get framebuffer info
drmModeFB2 *fb = drmModeGetFB2(fd, fb_id);   // returns handles, modifiers
drmModeFreeFB2(fb);
```

---

## 6. GEM Buffer Management

GEM (Graphics Execution Manager) handles are per-file-descriptor references to kernel-managed buffer objects.

```c
// Create GEM BO (driver-specific; shown for generic dumb buffer path below)
// Close GEM handle (decrement refcount)
struct drm_gem_close close_args = { .handle = gem_handle };
drmIoctl(fd, DRM_IOCTL_GEM_CLOSE, &close_args);

// Map GEM BO for CPU access (driver-specific; generic via dumb buffer mmap offset)
struct drm_mode_map_dumb mmap_args = { .handle = gem_handle };
drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &mmap_args);
void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, mmap_args.offset);

// flink: convert GEM handle to a global name (legacy sharing; prefer PRIME)
struct drm_gem_flink flink = { .handle = gem_handle };
drmIoctl(fd, DRM_IOCTL_GEM_FLINK, &flink);
uint32_t global_name = flink.name;

// Open a flink name from another process
struct drm_gem_open open_args = { .name = global_name };
drmIoctl(fd, DRM_IOCTL_GEM_OPEN, &open_args);
uint32_t local_handle = open_args.handle;
```

---

## 7. PRIME — DMA-BUF Import/Export

PRIME replaces flink for cross-process and cross-device buffer sharing using DMA-BUF file descriptors.

```c
// Export GEM handle to DMA-BUF fd
int dmabuf_fd;
drmPrimeHandleToFD(fd, gem_handle, DRM_CLOEXEC | DRM_RDWR, &dmabuf_fd);
// Send dmabuf_fd to another process via SCM_RIGHTS (Unix socket)

// Import DMA-BUF fd into a GEM handle (on the same or a different device)
uint32_t imported_handle;
drmPrimeFDToHandle(fd, dmabuf_fd, &imported_handle);

// Underlying ioctls (libdrm wrappers above are preferred):
// DRM_IOCTL_PRIME_HANDLE_TO_FD  →  struct drm_prime_handle { handle, flags, fd }
// DRM_IOCTL_PRIME_FD_TO_HANDLE  →  struct drm_prime_handle { fd, flags, handle }
```

**Cross-device zero-copy:**
```
GPU A (render)          GPU B (display) or CPU
  GEM handle ─PRIME─► DMA-BUF fd ─PRIME─► GEM handle
                      (same physical pages, no copy)
```

---

## 8. Dumb Buffers (software rendering)

Dumb buffers are a driver-agnostic path for linear CPU-mapped pixel buffers, guaranteed by all KMS drivers. Not GPU-accelerated.

```c
// Create
struct drm_mode_create_dumb create = { .width=w, .height=h, .bpp=32 };
drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);
// create.handle = GEM handle; create.pitch = stride; create.size = total bytes

// Map for CPU access
struct drm_mode_map_dumb map = { .handle = create.handle };
drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);
uint8_t *pixels = mmap(NULL, create.size, PROT_READ|PROT_WRITE,
                        MAP_SHARED, fd, map.offset);

// Add as framebuffer
uint32_t fb_id;
drmModeAddFB(fd, w, h, 24, 32, create.pitch, create.handle, &fb_id);

// Destroy
struct drm_mode_destroy_dumb destroy = { .handle = create.handle };
drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
```

---

## 9. Syncobj — GPU Synchronisation

Syncobjs are kernel-managed synchronisation objects that wrap GPU timeline points. They replace the implicit fence model.

```c
// Create syncobj
uint32_t syncobj;
drmSyncobjCreate(fd, 0, &syncobj);
// flag: DRM_SYNCOBJ_CREATE_SIGNALED — start in signalled state

// Destroy
drmSyncobjDestroy(fd, syncobj);

// Export to sync_file fd (for passing to Wayland linux-drm-syncobj-v1)
int sync_fd;
drmSyncobjExportSyncFile(fd, syncobj, &sync_fd);

// Import from sync_file fd
drmSyncobjImportSyncFile(fd, syncobj, sync_fd);

// Export syncobj as fd (for cross-process sharing)
int obj_fd;
drmSyncobjHandleToFD(fd, syncobj, &obj_fd);
drmSyncobjFDToHandle(fd, obj_fd, &syncobj);

// Wait for one or more syncobjs to signal
drmSyncobjWait(fd, &syncobj, 1, timeout_ns, flags, &first_signalled);
// flags: DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL / WAIT_FOR_SUBMIT

// Reset syncobj to unsignalled
drmSyncobjReset(fd, &syncobj, 1);

// Signal syncobj from CPU (for testing)
drmSyncobjSignal(fd, &syncobj, 1);

// Timeline syncobjs (DRM_CAP_SYNCOBJ_TIMELINE)
drmSyncobjTimelineWait(fd, &syncobj, &point, 1, timeout_ns, flags, NULL);
drmSyncobjTimelineSignal(fd, &syncobj, &point, 1);
drmSyncobjTransfer(fd, dst, dst_point, src, src_point, 0);
```

---

## 10. Vblank and Timestamp

```c
// Request vblank notification
drmVBlank vbl = {
    .request.type = DRM_VBLANK_RELATIVE | DRM_VBLANK_EVENT
                    | (crtc_index << DRM_VBLANK_HIGH_CRTC_SHIFT),
    .request.sequence = 1,    // relative: next vblank; absolute: specific frame
    .request.signal   = (unsigned long)user_data,
};
drmWaitVBlank(fd, &vbl);

// Handle vblank event (same drmHandleEvent loop as page flip):
drmEventContext evctx = {
    .version = DRM_EVENT_CONTEXT_VERSION,
    .vblank_handler = on_vblank,    // (fd, seq, sec, usec, user_data)
};
drmHandleEvent(fd, &evctx);

// Query current scanout position (requires DRM_CAP_VBLANK_HIGH_CRTC)
// Use wp_presentation_feedback Wayland protocol for compositor-level timing
```

---

## 11. KMS Object Properties Reference

Properties are attached to KMS objects and queried via `drmModeObjectGetProperties`. Property IDs must be looked up by name at runtime.

### CRTC Properties

| Property name          | Type    | Description                                              |
|------------------------|---------|----------------------------------------------------------|
| `ACTIVE`               | RANGE   | 0 = CRTC off; 1 = CRTC on                               |
| `MODE_ID`              | BLOB    | Blob ID of `drmModeModeInfo` (0 = no mode)               |
| `DEGAMMA_LUT`          | BLOB    | Pre-CTM LUT (for HDR)                                    |
| `CTM`                  | BLOB    | 3×3 colour transform matrix                              |
| `GAMMA_LUT`            | BLOB    | Post-CTM LUT                                             |
| `VRR_ENABLED`          | RANGE   | Enable Variable Refresh Rate (1 = enabled)               |
| `OUT_FENCE_PTR`        | SIGNED_RANGE | Write ptr for out-fence fd (explicit sync)          |

### Connector Properties

| Property name          | Type    | Description                                              |
|------------------------|---------|----------------------------------------------------------|
| `CRTC_ID`              | OBJECT  | ID of the driving CRTC (0 = disconnected)                |
| `DPMS`                 | ENUM    | `On`/`Standby`/`Suspend`/`Off`                           |
| `EDID`                 | BLOB    | Raw EDID blob                                            |
| `PATH`                 | BLOB    | MST DisplayPort path string                              |
| `link-status`          | ENUM    | `Good`/`Bad` — link training result                      |
| `non-desktop`          | RANGE   | 1 = HMD / VR display, not for normal output              |
| `max bpc`              | RANGE   | Maximum bits per colour channel                          |
| `Colorspace`           | ENUM    | `Default`/`BT2020_RGB`/`BT709_YCC`/... (HDR)            |
| `HDR_OUTPUT_METADATA`  | BLOB    | HDMI HDR metadata (SMPTE ST 2086 + CTA 861.3)           |
| `scaling mode`         | ENUM    | `None`/`Full`/`Center`/`Full aspect`                     |
| `subconnector`         | ENUM    | VGA/DVI/HDMI subtype                                     |

### Plane Properties

| Property name          | Type    | Description                                              |
|------------------------|---------|----------------------------------------------------------|
| `CRTC_ID`              | OBJECT  | Driving CRTC                                             |
| `FB_ID`                | OBJECT  | Attached framebuffer (0 = disabled)                      |
| `SRC_X/Y/W/H`          | RANGE   | Source crop in 16.16 fixed-point (subpixel precision)    |
| `CRTC_X/Y/W/H`         | SIGNED_RANGE / RANGE | Destination rectangle on CRTC                |
| `type`                 | ENUM    | `Primary`/`Overlay`/`Cursor`                             |
| `IN_FENCE_FD`          | SIGNED_RANGE | Acquire fence fd (-1 = none; explicit sync input)   |
| `IN_FORMATS`           | BLOB    | Supported format+modifier combinations                   |
| `rotation`             | BITMASK | `rotate-0/90/180/270`, `reflect-X/Y`                     |
| `zpos`                 | RANGE   | Z-order (higher = on top)                                |
| `alpha`                | RANGE   | 0 = transparent; 0xFFFF = opaque (per-plane alpha)       |
| `pixel blend mode`     | ENUM    | `None`/`Pre-multiplied`/`Coverage`                       |
| `COLOR_ENCODING`       | ENUM    | `ITU-R BT.601`/`BT.709`/`BT.2020`                       |
| `COLOR_RANGE`          | ENUM    | `YCbCr limited range`/`YCbCr full range`                 |

---

## Cross-References

- **Chapter 1** — DRM Architecture: GEM, KMS, render nodes, driver model
- **Chapter 3** — KMS and Display Pipeline: CRTC/encoder/connector/plane model, atomic modesetting
- **Chapter 4** — GEM and DMA-BUF: buffer lifecycle, PRIME sharing, format modifiers
- **Chapter 75** — Explicit GPU Sync: syncobj, timeline syncobj, `linux-drm-syncobj-v1`
- **Appendix G** — Synchronisation Primitives: syncobj in context of the full Linux sync stack
- **Appendix H** — DRM Format Modifier Reference: `DRM_FORMAT_MOD_*` values and tiling layouts
- **Appendix N** — Vulkan Linux Platform Extensions: Vulkan WSI extensions that sit above the DRM/KMS ioctls

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
