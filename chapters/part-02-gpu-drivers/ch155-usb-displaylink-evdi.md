# Chapter 155: USB DisplayLink and the evdi Kernel Module

**Target audiences**: Systems engineers adding USB display support to embedded or desktop Linux systems; driver developers studying evdi as an example of a user-space DRM driver; and laptop users debugging multi-monitor setups over DisplayLink docks.

---

## Table of Contents

1. [Introduction](#introduction)
   - [1.1 What is DisplayLink?](#11-what-is-displaylink)
   - [1.2 What is evdi?](#12-what-is-evdi)
   - [1.3 What is the DisplayLink Manager (DLM)?](#13-what-is-the-displaylink-manager-dlm)
2. [DisplayLink Hardware and Protocol](#displaylink-hardware-and-protocol)
3. [evdi: Extensible Virtual Display Interface](#evdi-extensible-virtual-display-interface)
4. [evdi Kernel Module Architecture](#evdi-kernel-module-architecture)
5. [evdi GEM Memory Model](#evdi-gem-memory-model)
6. [The evdi Painter: Damage Tracking and Update Flow](#the-evdi-painter-damage-tracking-and-update-flow)
7. [Cursor Emulation](#cursor-emulation)
8. [libevdi: The Userspace Library](#libevdi-the-userspace-library)
9. [DisplayLink Manager (DLM) and the Full Stack](#displaylink-manager-dlm-and-the-full-stack)
10. [DRM Integration and Atomic Modesetting](#drm-integration-and-atomic-modesetting)
11. [USB4, Thunderbolt, and the DL-7400 Generation](#usb4-thunderbolt-and-the-dl-7400-generation)
12. [Limitations and Performance](#limitations-and-performance)
13. [Open-Source Alternatives, Reverse Engineering, and the Ecosystem](#open-source-alternatives-and-the-ecosystem)
14. [Troubleshooting and Diagnostics](#troubleshooting-and-diagnostics)
15. [Roadmap](#roadmap)
16. [Integrations](#integrations)

---

## Introduction

DisplayLink is a USB-to-display technology by Synaptics that lets a USB 3.0 or USB-C connection drive an external monitor. It is widely used in USB docking stations from Dell, Lenovo, CalDigit, Plugable, and many other vendors. On Linux, DisplayLink depends on the **evdi** (Extensible Virtual Display Interface) kernel module, which creates a virtual DRM device that the userspace DisplayLink Manager (DLM) can write pixels into.

The architecture is unusual: evdi is a DRM driver whose "GPU" is actually a userspace daemon that compresses the framebuffer and sends it over USB. This makes it a valuable case study in virtual DRM devices and the boundary between kernel DRM and userspace display processing.

Understanding evdi requires decomposing it into four distinct layers:

- **kernel module** — presents a standard DRM interface to compositors
- **GEM memory model** — backs framebuffers with shmem-allocated pages
- **painter subsystem** — tracks dirty regions and delivers pixel data to userspace
- **libevdi wrapper** — mediates between the kernel module and the proprietary DLM daemon

Each layer interacts with upstream DRM infrastructure in ways that illuminate the broader Linux graphics stack.

Sources: [evdi on GitHub](https://github.com/DisplayLink/evdi) | [DisplayLink Linux driver](https://www.synaptics.com/products/displaylink-graphics/downloads/ubuntu)

### 1.1 What is DisplayLink?

DisplayLink is a USB-to-display technology by Synaptics that enables a USB 3.0 or USB-C connection to drive one or more external monitors. It is found in USB docking stations and adapters from many vendors. Rather than exposing a GPU or hardware-accelerated display pipeline, DisplayLink delegates all rendering work to the host processor: the host captures the framebuffer, compresses it using the proprietary DL3 codec, and streams the result over USB bulk transfers to a DisplayLink chip in the dock, which decodes the stream and drives the physical display output (HDMI or DisplayPort).

On Linux, DisplayLink support requires the **evdi** kernel module and the proprietary **DisplayLink Manager (DLM)** daemon working in tandem. The DisplayLink chip itself contains no GPU and performs no compositing; its role is limited to decoding the compressed stream and managing display timing. This software-defined architecture makes DisplayLink broadly compatible across host systems but introduces constraints around CPU overhead, encoding latency, and USB bandwidth that distinguish it from native GPU-connected displays. Current chip generations span from DL-3xxx (USB 2.0/3.0, up to 1080p) through DL-6xxx (USB 3.2, up to 5K) to the DL-7400 (USB4/Thunderbolt, quad 4K@120Hz or dual 8K@60Hz). [Source](https://www.synaptics.com/products/displaylink-graphics)

### 1.2 What is evdi?

evdi (Extensible Virtual Display Interface) is a Linux kernel module that presents a virtual DRM (Direct Rendering Manager) device to the rest of the graphics stack. It is distributed as an out-of-tree module installed via DKMS and creates one or more `/dev/dri/cardN` device nodes that compositors treat as ordinary DRM modesetting devices. The key distinction from a conventional DRM driver is that evdi's "hardware" is not a physical GPU but a userspace consumer: the module captures committed framebuffers, tracks dirty regions via the `FB_DAMAGE_CLIPS` DRM plane property, and delivers pixel data to whichever userspace process connects through the custom ioctl interface.

evdi is transport-agnostic — any client that speaks the `EVDI_CONNECT` / `EVDI_REQUEST_UPDATE` / `EVDI_GRABPIX` ioctl protocol can consume its output. In the DisplayLink stack that client is the proprietary DLM daemon, but the same mechanism works for network-based virtual displays, screen capture pipelines, or automated test harnesses. The module source lives at [github.com/DisplayLink/evdi](https://github.com/DisplayLink/evdi) and implements the DRM subsystem conventions defined in `drivers/gpu/drm/`. Understanding evdi is an exercise in how the DRM atomic modesetting API can be implemented for non-physical hardware and how the kernel/userspace boundary is drawn in an unusual driver architecture. [Source](https://github.com/DisplayLink/evdi)

### 1.3 What is the DisplayLink Manager (DLM)?

The DisplayLink Manager (DLM) is the proprietary userspace daemon that connects the evdi kernel module to the physical DisplayLink USB hardware. It is the only component in the stack that speaks both the evdi ioctl protocol and the DisplayLink USB wire format; without it, evdi creates a visible DRM device that compositors can drive but no pixels reach the monitor.

At runtime the DLM opens the evdi DRM device node, sends the monitor EDID via the `EVDI_CONNECT` ioctl to make the virtual connector appear plugged in to the compositor, and then enters a frame update loop: it calls `EVDI_REQUEST_UPDATE` to wait for a frame commit, calls `EVDI_GRABPIX` to copy dirty pixel regions into a client buffer, compresses the result using the DL3 codec, and sends the compressed stream to the DisplayLink chip via USB bulk transfers through libusb. Internally the DLM is organized as a plugin pipeline with distinct source (evdi pixel capture), transform (DL3 encoding), and sink (libusb USB transfer) stages, which allows the same core logic to target different hardware generations. The DLM is distributed as a closed binary by Synaptics and is the only proprietary component in an otherwise open stack. [Source](https://www.synaptics.com/products/displaylink-graphics/downloads/ubuntu)

---

## DisplayLink Hardware and Protocol

### USB Display Architecture

The DisplayLink architecture is best understood as a software-defined display pipeline. Unlike a conventional GPU that drives a display via a direct electrical interface (HDMI, DisplayPort), a DisplayLink adapter relies entirely on host-side software to encode the framebuffer before it ever reaches hardware:

```
GPU (Intel/AMD/Nouveau)
  │ renders to framebuffer (DMA-BUF)
  ▼
evdi virtual DRM device (/dev/dri/cardN)
  │ exports damage regions + pixel data to userspace
  ▼
DisplayLink Manager (DLM) — proprietary daemon
  │ encodes using DL3 codec
  ▼
libusb → USB 3.0/3.2/USB4 host controller
  │ USB bulk transfers
  ▼
DisplayLink chip (DL-5xxx / DL-6xxx / DL-7400) in dock
  │ decodes DL3 stream, drives pixel clock
  ▼
HDMI/DisplayPort output to monitor
```

The key implication of this architecture is that **all computation happens on the host CPU**: frame capture, damage detection, encoding, and USB scheduling. The DisplayLink chip in the dock is purely a decoder and display interface; there is no GPU in the adapter capable of compositing, rendering, or handling hardware overlays.

### DisplayLink USB Protocol

The DisplayLink chip communicates with the host via USB bulk transfers carrying a proprietary encoded stream. DisplayLink uses their **DL3** adaptive codec across all current product generations. The DL3 codec adapts compression to content type — applying heavier compression for video-like content and higher quality (closer to lossless) for static UI and text. The specific algorithm is proprietary and not publicly documented; DisplayLink describes it as providing "low latency, pixel-perfect graphics" that "adapt automatically to bandwidth constraints." [Source](https://www.synaptics.com/products/displaylink-graphics/integrated-chipsets/dl-6000)

> **Note: needs verification.** The exact internal structure of DL3 (whether it uses JPEG, intra-frame DCT, or a custom scheme for lossy passes, and what lossless path it uses) is not publicly disclosed. Claims of JPEG or RLE should be treated as unverified speculation.

The DLM daemon uses a plugin-based pipeline, analogous to GStreamer:
1. **Source plugin**: receives raw pixel regions from evdi
2. **Transform plugin**: encodes via DL3 into a compressed bytestream
3. **Sink plugin**: writes to the USB bulk endpoint via libusb

This architecture allows the same DLM core to run on Windows, macOS, Chrome OS, and Linux with platform-specific source and sink plugins. [Source](https://support.displaylink.com/knowledgebase/articles/1104056)

A critical asymmetry: the DLM only sends *changed* tiles. Because evdi reports dirty rectangles via the DRM `FB_DAMAGE_CLIPS` plane property, the DLM receives exactly which screen regions changed per frame. For a typical desktop with mostly static content, only a fraction of the screen changes per frame — effective USB bandwidth consumption is typically far lower than worst-case.

### Supported Chipsets

| Chip | Max Resolution | USB Speed | Notes |
|---|---|---|---|
| DL-3xxx | 1920×1080@60 (4K@30 reported for some variants) | USB 2.0/3.0 | Legacy, DL3 codec. Note: needs verification — marketing materials conflict on max resolution. |
| DL-5xxx | 2560×1440@60 | USB 3.0 | First USB 3.0 gen; common in docks |
| DL-6xxx / DL-6950 | 5120×2880@60 (5K) or dual 4K@60 | USB 3.0/3.2 | DL3 codec, SoC with GbE + audio |
| DL-7400 | Quad 4K@120Hz or dual 8K@60Hz HDR10 | USB4 / Thunderbolt / DP 2.1 | Current gen, DP 2.1 Alt Mode |

Sources: [DL-6000 product page](https://www.synaptics.com/products/displaylink-graphics/integrated-chipsets/dl-6000) | [DL-7400 announcement](https://www.synaptics.com/company/news/synaptics-introduces-displayport-21-and-displaylink-quad-display)

---

## evdi: Extensible Virtual Display Interface

### What evdi Provides

evdi ([github.com/DisplayLink/evdi](https://github.com/DisplayLink/evdi)) is an out-of-tree DRM driver built and installed via DKMS. It:

- Creates one or more virtual `/dev/dri/cardN` DRM devices, each representing a potential virtual display
- Implements the full DRM modesetting stack: CRTC, connector, encoder, primary plane
- Tracks framebuffer damage using the `FB_DAMAGE_CLIPS` DRM plane property and the `drm_atomic_helper_dirtyfb` callback
- Delivers pixel data and damage rectangles to userspace via DRM ioctls from the `evdi_painter_ioctls` table
- Supports DDC/CI monitor control passthrough from userspace (via I²C over the DRM connector) [Source](https://github.com/DisplayLink/evdi/tree/main/module)

evdi does **not** perform USB transfers — it is a transport-agnostic virtual display. Any userspace program that speaks the evdi ioctl protocol can consume its output, whether the target is USB, network, Wi-Fi, or a test harness.

### evdi Architecture Overview

```
Compositor (Mutter/KWin/sway)
  │ opens /dev/dri/cardN (evdi virtual device)
  │ sets mode, creates GEM-backed framebuffer
  │ commits planes via drmModeAtomicCommit()
  ▼
evdi kernel module
  │ on each commit: copies GPU DMA-BUF to GEM pages (CPU copy)
  │ tracks damage via FB_DAMAGE_CLIPS property
  │ stores dirty rects in evdi_painter; wakes update waiter
  ▼
DLM (DisplayLink Manager) userspace daemon
  │ opens same /dev/dri/cardN DRM node
  │ calls EVDI_CONNECT ioctl with EDID
  │ calls EVDI_REQUEST_UPDATE → waits for update_ready event
  │ calls EVDI_GRABPIX → kernel copies dirty rects to client buffer
  │ encodes with DL3 → USB bulk transfer → DisplayLink chip
```

The critical insight is that the DLM talks to evdi through the **same DRM device node** (`/dev/dri/cardN`) as the compositor, using custom DRM ioctls registered in the `evdi_painter_ioctls` table. There is no separate character device.

---

## evdi Kernel Module Architecture

### Module Structure

The evdi module is structured as a standard out-of-tree DRM platform driver. Its source lives in `module/` within the evdi repository, with these key files:

| File | Purpose |
|---|---|
| `evdi_drm_drv.c` | `drm_driver` definition, device init/teardown |
| `evdi_platform_drv.c` | Platform driver probe/remove |
| `evdi_platform_dev.c` | sysfs-triggered device creation |
| `evdi_painter.c` | Damage tracking, ioctl handling, update flow |
| `evdi_gem.c` | GEM object allocation, shmem page management, mmap |
| `evdi_fb.c` | `drm_framebuffer` wrapper and dirty callback |
| `evdi_modeset.c` | CRTC, encoder, connector, atomic helpers |
| `evdi_cursor.c` | Software cursor compositing |
| `evdi_sysfs.c` | `/sys/bus/platform/devices/evdi.*` interface |
| `evdi_i2c.c` | DDC/CI I²C adapter for monitor control |

### The drm_driver Definition

The evdi DRM driver declares support for modesetting, GEM memory management, and atomic commits. Abridged from `module/evdi_drm_drv.c` at `main`:

```c
/* module/evdi_drm_drv.c (abridged from DisplayLink/evdi main) */
static const struct drm_driver driver = {
    .driver_features = DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC,
    .open            = evdi_driver_open,
    .postclose       = evdi_driver_postclose,
    .ioctls          = evdi_painter_ioctls,
    .num_ioctls      = ARRAY_SIZE(evdi_painter_ioctls),
    .fops            = &evdi_driver_fops,
    /* GEM object functions provided via evdi_gem_object_funcs */
};
```

The driver advertises `DRIVER_ATOMIC` — compositors that support the atomic modesetting API (Mutter ≥ 3.36, KWin ≥ 5.20, sway) commit frames atomically. Legacy `drmModeSetCrtc` still works via the DRM atomic emulation layer.

### Device Lifecycle

Virtual evdi devices are created on demand via sysfs, not by hardware enumeration. The DLM writes to `/sys/bus/platform/driver/evdi/add` to instantiate a new virtual device, which triggers `evdi_platform_probe()`. The resulting device appears as `/dev/dri/cardN`. The `evdi_drm_device_init()` function sets up the painter, cursor state, and vblank tracking:

```c
/* module/evdi_drm_drv.c (abridged) */
int evdi_drm_device_init(struct drm_device *dev)
{
    struct evdi_device *evdi = to_evdi_device(dev);

    evdi_modeset_init(dev);     /* CRTC, connector, encoder, planes */
    evdi_painter_init(evdi);    /* dirty-rect tracking + ioctl state */
    evdi_cursor_init(evdi);     /* software cursor compositor */
    drm_vblank_init(dev, 1);    /* single CRTC vblank */
    return 0;
}
```

### Custom ioctl Table

evdi registers five custom DRM ioctls via `evdi_painter_ioctls[]`:

```c
/* module/evdi_painter.c (abridged) */
const struct drm_ioctl_desc evdi_painter_ioctls[] = {
    DRM_IOCTL_DEF_DRV(EVDI_CONNECT,
        evdi_painter_connect_ioctl,       DRM_UNLOCKED),
    DRM_IOCTL_DEF_DRV(EVDI_REQUEST_UPDATE,
        evdi_painter_request_update_ioctl, DRM_UNLOCKED),
    DRM_IOCTL_DEF_DRV(EVDI_GRABPIX,
        evdi_painter_grabpix_ioctl,        DRM_UNLOCKED),
    DRM_IOCTL_DEF_DRV(EVDI_DDCCI_RESPONSE,
        evdi_painter_ddcci_response_ioctl, DRM_UNLOCKED),
    DRM_IOCTL_DEF_DRV(EVDI_ENABLE_CURSOR_EVENTS,
        evdi_painter_enable_cursor_events_ioctl, DRM_UNLOCKED),
};
```

Connection is established via `EVDI_CONNECT` with `connected=1` in `struct drm_evdi_connect`; disconnection reuses the same ioctl with `connected=0`. There is no separate disconnect ioctl. Cursor events are **opt-in**: by default the kernel composites the cursor into the grabbed framebuffer; after calling `EVDI_ENABLE_CURSOR_EVENTS`, the client receives `cursor_set` / `cursor_move` kernel events and is responsible for blending the cursor itself.

### DRM Connector and EDID

The evdi connector is a stub until the DLM connects. Once `EVDI_CONNECT` is called with EDID data, the painter stores the EDID, triggers a hotplug detection event via `drm_helper_hpd_irq_event()`, and the DRM core runs the connector's `get_modes` callback, which parses the stored EDID:

```c
/* module/evdi_connector.c (abridged) */
static int evdi_connector_get_modes(struct drm_connector *connector)
{
    struct evdi_connector *evdi_con = to_evdi_connector(connector);
    struct evdi_painter *painter = evdi_con->evdi->painter;

    if (!painter->edid)
        return 0;
    drm_connector_update_edid_property(connector, painter->edid);
    return drm_add_edid_modes(connector, painter->edid);
}
```

This means the compositor sees a fully populated monitor — correct resolutions, preferred mode, HDCP flags — without knowing it is talking to a virtual device.

---

## evdi GEM Memory Model

### GEM Object Structure

evdi manages framebuffer memory via a custom GEM object type, `evdi_gem_object`, that wraps standard DRM GEM. Backing store is provided by shmem (anonymous pageable kernel memory), allocated lazily on first `mmap()`. Abridged from `module/evdi_gem.c` at `main`:

```c
/* module/evdi_gem.c (abridged) */
struct evdi_gem_object {
    struct drm_gem_object base;       /* must be first */
    struct page         **pages;      /* shmem pages, NULL until pinned */
    struct sg_table      *sg;         /* scatter-gather table for DMA */
    void                 *vmapping;   /* kernel vmap address */
    bool                  vmap_is_iomem;
    bool                  allow_sw_cursor_rect_updates;
    int                   pages_pin_count;
    struct mutex          pages_lock;
};
```

Allocation uses `drm_gem_object_init()`, which creates an shmem-backed file of the requested size. Pages are not faulted in until `mmap()` is called or until `evdi_pin_pages()` is explicitly invoked:

```c
struct evdi_gem_object *evdi_gem_alloc_object(struct drm_device *dev,
                                               size_t size)
{
    struct evdi_gem_object *obj = kzalloc(sizeof(*obj), GFP_KERNEL);
    if (!obj)
        return NULL;
    if (drm_gem_object_init(dev, &obj->base, size)) {
        kfree(obj);
        return NULL;
    }
    mutex_init(&obj->pages_lock);
    return obj;
}
```

### Page Pinning and Cache Coherency

Before pixel data can be read by the DLM, the GEM pages must be pinned in memory and the CPU cache must be flushed to ensure coherency. evdi uses `drm_clflush_pages()` after obtaining the page array, which issues cache-line flush instructions on x86 (CLFLUSH). This is the mechanism that makes CPU-written pixels visible to subsequent reads:

```c
/* module/evdi_gem.c (abridged) */
static int evdi_gem_get_pages(struct evdi_gem_object *obj, gfp_t gfpmask)
{
    struct page **pages;
    if (obj->pages)
        return 0;  /* already have pages */

    pages = drm_gem_get_pages(&obj->base);
    if (IS_ERR(pages))
        return PTR_ERR(pages);

    obj->pages = pages;
    /* Flush CPU caches so subsequent reads see up-to-date pixel data: */
    drm_clflush_pages(obj->pages,
        DIV_ROUND_UP(obj->base.size, PAGE_SIZE));
    return 0;
}
```

### mmap and VM Flags

The userspace DLM allocates its own pixel buffer and registers it with `evdi_register_buffer()`. When evdi grabs pixels (`EVDI_GRABPIX`), it copies from the GEM pages into the client-registered buffer using `copy_primary_pixels()`. The module also exposes `evdi_drm_gem_mmap()` for direct mapping with adjusted VM flags:

```c
/* module/evdi_gem.c (abridged) */
int evdi_drm_gem_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret = drm_gem_mmap(filp, vma);
    if (ret)
        return ret;
    /* Replace VM_PFNMAP with VM_MIXEDMAP to allow struct-page access: */
    vm_flags_mod(vma, VM_MIXEDMAP, VM_PFNMAP);
    return ret;
}
```

This distinction matters because CLFLUSH requires struct-page pointers, not raw PFNs — `VM_MIXEDMAP` enables the kernel to obtain page references for cache operations on mapped memory.

---

## The evdi Painter: Damage Tracking and Update Flow

### evdi_painter Structure

The `evdi_painter` is the heart of evdi's userspace interaction model. It maintains connection state, the dirty-rectangle ring, the currently displayed framebuffer, and the DRM event queue for delivering `update_ready` signals to the DLM. Abridged from `module/evdi_painter.c` at `main`:

```c
/* module/evdi_painter.c (abridged) */
#define MAX_DIRTS 16

struct evdi_painter {
    bool                     is_connected;
    struct edid             *edid;
    unsigned int             edid_length;
    struct mutex             lock;

    /* Dirty rectangle ring: */
    struct drm_clip_rect     dirty_rects[MAX_DIRTS];
    int                      num_dirts;

    struct evdi_framebuffer *scanout_fb;   /* current displayed FB */
    struct drm_file         *drm_filp;     /* the DLM's DRM file handle */
    struct drm_device       *drm_device;

    bool                     was_update_requested;
    bool                     needs_full_modeset;
    struct drm_crtc         *crtc;
    struct drm_pending_vblank_event *vblank;
    struct list_head         pending_events;
    struct delayed_work      send_events_work;

    /* DDC-CI (I²C monitor control) passthrough: */
    struct completion        ddcci_response_received;
    char                    *ddcci_buffer;
    unsigned int             ddcci_buffer_length;

    /* VT switching: pause updates when a VT is active: */
    struct notifier_block    vt_notifier;
    int                      fg_console;
};
```

### Damage Tracking: Mark Dirty

When the compositor commits a new frame to evdi, the `drm_atomic_helper_dirtyfb` callback fires, which ultimately calls `evdi_painter_mark_dirty()`. This function sanitizes each damage rectangle against framebuffer bounds and adds it to the ring:

```c
/* module/evdi_painter.c (abridged) */
void evdi_painter_mark_dirty(struct evdi_device *evdi,
                              const struct drm_clip_rect *dirty_rect)
{
    struct evdi_painter *painter = evdi->painter;
    struct drm_clip_rect sanitized =
        evdi_framebuffer_sanitize_rect(painter->scanout_fb, dirty_rect);

    mutex_lock(&painter->lock);

    if (painter->num_dirts < MAX_DIRTS) {
        painter->dirty_rects[painter->num_dirts++] = sanitized;
    } else {
        /* Ring full: merge two rects into their bounding box */
        merge_dirty_rects(painter->dirty_rects, &painter->num_dirts);
        if (painter->num_dirts == MAX_DIRTS)
            collapse_dirty_rects(painter->dirty_rects, &painter->num_dirts);
        painter->dirty_rects[painter->num_dirts++] = sanitized;
    }

    mutex_unlock(&painter->lock);
}
```

When the ring overflows 16 slots, evdi tries pairwise bounding-box merging before falling back to collapsing all rects into a single full-framebuffer update. This design trades precision for bounded memory: at worst, the DLM re-encodes the entire framebuffer.

### Update Ready Notification

After marking dirty, evdi delivers a `DRM_EVDI_EVENT_UPDATE_READY` event to the DLM's event queue if an update has been requested:

```c
/* module/evdi_painter.c (abridged) */
void evdi_painter_send_update_ready_if_needed(struct evdi_painter *painter)
{
    if (painter->was_update_requested && painter->num_dirts > 0) {
        evdi_painter_send_event(painter, &painter->update_ready_event);
        painter->was_update_requested = false;
    }
}
```

The DLM reads this event via the standard `poll()` / `read()` interface on the DRM device file descriptor — the same mechanism used by compositors to receive vblank completion events.

### EVDI_GRABPIX: Pixel Data Delivery

When the DLM receives an `update_ready` event, it calls `EVDI_GRABPIX`. The kernel copies dirty pixels from the GEM pages into the client's registered buffer:

```c
/* Abridged flow for EVDI_GRABPIX ioctl */
struct drm_evdi_grabpix {
    uint32_t mode;        /* EVDI_GRABPIX_MODE_DIRTY */
    uint32_t width;
    uint32_t height;
    uint32_t stride;
    uint64_t buffer;      /* userspace buffer pointer */
    uint32_t num_rects;   /* in/out: max rects / actual rects returned */
    struct drm_clip_rect rects[MAX_DIRTS];
};
```

The `copy_primary_pixels()` function iterates over dirty rectangles and copies row-by-row from pinned GEM pages to the client buffer, with `drm_clflush_virt_range()` ensuring cache coherency before each copy on x86. This is a standard CPU memcpy — there is no DMA, no zero-copy, and no GPU involvement.

**This is the fundamental performance bottleneck**: for 1920×1080 XRGB8888, a full-frame grab copies ~8 MB. At memory bandwidth of ~30 GB/s (typical DDR4), this takes ~270 µs minimum — but in practice, cache pressure and page-fault handling push typical latencies to 1–5 ms per frame.

---

## Cursor Emulation

### Hardware Cursor Impossibility

A DisplayLink adapter has no GPU, no 2D overlay hardware, and no compositor. All cursor rendering must happen before pixels reach USB. evdi offers two strategies: **automatic cursor compositing** (default) and **cursor event forwarding** (opt-in via `EVDI_ENABLE_CURSOR_EVENTS`).

### Software Cursor Compositor

In default mode, evdi composites the cursor into the grabbed framebuffer pixel-by-pixel. The `evdi_cursor` state tracks the current cursor shape, dimensions, hotspot, and position:

```c
/* module/evdi_cursor.c (abridged) */
struct evdi_cursor {
    bool              enabled;
    int32_t           x, y;
    uint32_t          width, height;
    int32_t           hot_x, hot_y;
    uint32_t          pixel_format;   /* ARGB8888 */
    uint32_t          stride;
    struct evdi_gem_object *obj;      /* cursor bitmap in GEM memory */
    struct mutex      lock;
};
```

The `evdi_cursor_compose_and_copy()` function blends cursor pixels onto the framebuffer using per-pixel alpha: for each cursor pixel within the framebuffer bounds, it calculates the blended output as:

```c
/* Per-channel alpha blend: dest = (dest*(255-a) + src*a) / 255 */
static uint8_t blend_component(uint8_t dest, uint8_t src, uint8_t alpha)
{
    return (uint16_t)(dest * (255 - alpha) + src * alpha) / 255;
}
```

This approach generates an `update_ready` event on every cursor movement — even a mouse pan generates a dirty rect containing the old and new cursor positions and triggers a full grab-compress-send cycle.

### Cursor Event Mode

When the DLM calls `EVDI_ENABLE_CURSOR_EVENTS`, evdi stops compositing the cursor. Instead, each `evdi_cursor_set` or `evdi_cursor_move` generates a `DRM_EVDI_EVENT_CURSOR_SET` or `DRM_EVDI_EVENT_CURSOR_MOVE` event carrying cursor metadata. The DLM then blends the cursor into its own output buffer before encoding. This reduces CPU overhead for high-cursor-activity scenarios by decoupling cursor position updates from full framebuffer grabs.

---

## libevdi: The Userspace Library

### Library API

libevdi (in `library/` within the evdi repository) provides a C API used by the DLM to interact with the kernel module. It wraps the DRM ioctl interface with a callback-based event model:

```c
/* library/evdi_lib.h (abridged) */
#include <evdi_lib.h>

/* Open or create an evdi device: */
evdi_handle handle = evdi_open(0);  /* first evdi device */
if (handle == EVDI_INVALID_HANDLE) {
    evdi_add(1);                    /* create one new device via sysfs */
    handle = evdi_open(0);
}

/* Connect with monitor EDID: */
evdi_connect(handle,
    edid_data, edid_size,
    pixel_area_limit,         /* max pixels per frame (bandwidth limit) */
    pixel_per_second_limit);  /* max pixels per second (throughput cap) */

/* Register a pixel buffer (client-allocated): */
evdi_buffer buf = {
    .id     = 0,
    .buffer = pixels_ptr,     /* userspace-allocated framebuffer */
    .width  = 1920,
    .height = 1080,
    .stride = 1920 * 4,
};
evdi_register_buffer(handle, buf);

/* Register event handlers: */
struct evdi_event_context ctx = {
    .update_ready_handler   = my_update_ready_cb,
    .mode_changed_handler   = my_mode_changed_cb,
    .dpms_handler           = my_dpms_cb,
    .cursor_set_handler     = my_cursor_set_cb,
    .cursor_move_handler    = my_cursor_move_cb,
    .ddcci_data_handler     = my_ddcci_cb,
};

/* Main event loop: */
while (running)
    evdi_handle_events(handle, &ctx);
```

Source: [library/evdi_lib.c at main](https://github.com/DisplayLink/evdi/blob/main/library/evdi_lib.c)

### Render Loop: Request → Update Ready → Grab

The render loop follows a request-notify-grab pattern. Unlike a polling model, the DLM does not busy-loop; it sleeps on `poll()` until the kernel signals readiness:

```c
/* Typical DLM render loop (abridged) */
static void my_update_ready_cb(int buf_id, void *user_data)
{
    evdi_handle handle = user_data;
    struct drm_clip_rect rects[EVDI_HANDLES_MAX];
    int num_rects = EVDI_HANDLES_MAX;

    /*
     * EVDI_GRABPIX: kernel copies dirty pixels into our registered
     * buffer and fills rects[] with the damage rectangles.
     * This is a kernel→user CPU copy, not mmap of the GEM buffer.
     */
    evdi_grab_pixels(handle, rects, &num_rects);

    /* For each dirty region: encode with DL3, send via USB bulk: */
    for (int i = 0; i < num_rects; i++) {
        uint8_t *region_pixels = pixels_ptr_for_rect(&rects[i]);
        displaylink_encode_and_send(handle, &rects[i], region_pixels);
    }

    /* Re-arm: EVDI_REQUEST_UPDATE tells kernel we're ready for next frame */
    evdi_request_update(handle, buf_id);
}
```

`evdi_request_update()` corresponds to the `EVDI_REQUEST_UPDATE` ioctl. It sets `painter->was_update_requested = true` in the kernel, so the next `evdi_painter_mark_dirty()` call will immediately deliver a `DRM_EVDI_EVENT_UPDATE_READY` event.

### evdi_connect2 and Bandwidth Limiting

The `evdi_connect2()` variant (preferred in DLM ≥ 5.x) accepts separate `pixel_area_limit` (pixels per frame) and `pixel_per_second_limit` (pixels per second) constraints. The kernel uses these to reject mode setting requests that would exceed the transport's bandwidth:

```c
/* struct drm_evdi_connect (abridged) */
struct drm_evdi_connect {
    int      connected;             /* 1=connect, 0=disconnect */
    int      dev_index;
    uint8_t __user *edid;           /* pointer to EDID data in userspace */
    uint32_t edid_length;
    uint32_t pixel_area_limit;      /* max screen area in pixels */
    uint32_t pixel_per_second_limit; /* bandwidth in pixels/second */
};
```

For a USB 3.0 DL-5xxx device, a typical `pixel_per_second_limit` is 2560 × 1440 × 60 ≈ 221 million pixels/second, preventing the user from accidentally setting 4K@60 which would saturate the USB link.

---

## DisplayLink Manager (DLM) and the Full Stack

### DLM Architecture

The DisplayLink Manager (proprietary, distributed as `displaylink-driver-*.run` from Synaptics) is a userspace daemon that bridges evdi and the DisplayLink USB hardware. It is distributed as a self-extracting installer that bundles:

- `evdi.ko` DKMS source (open source, MIT/GPLv2)
- `DisplayLinkManager` binary (proprietary)
- udev rules (`74-displaylink.rules`)
- systemd service (`displaylink-driver.service`)
- Firmware blobs for the DisplayLink chip

Installation on Ubuntu/Debian:

```bash
# Download from Synaptics:
# https://www.synaptics.com/products/displaylink-graphics/downloads/ubuntu
chmod +x displaylink-driver-6.0.run
sudo ./displaylink-driver-6.0.run

# This installs:
# /usr/lib/displaylink/DisplayLinkManager   (proprietary daemon)
# /usr/lib/displaylink/libevdi.so           (libevdi shared library)
# /etc/udev/rules.d/74-displaylink.rules    (udev hotplug rules)
# /lib/systemd/system/displaylink-driver.service
# /usr/src/evdi-*/                          (DKMS kernel module source)
```

The installer runs `dkms install evdi/<version>` to compile and register the kernel module. Each kernel update triggers a DKMS rebuild automatically.

### udev Integration

The `74-displaylink.rules` file matches DisplayLink USB devices by vendor ID (`17e9`) and calls a helper that signals the DLM:

```bash
# /etc/udev/rules.d/74-displaylink.rules (excerpt)
ACTION=="add", SUBSYSTEM=="usb", \
    ATTR{idVendor}=="17e9", \
    RUN+="/usr/lib/displaylink/displaylink-installer.sh attach %k"

ACTION=="remove", SUBSYSTEM=="usb", \
    ATTR{idVendor}=="17e9", \
    RUN+="/usr/lib/displaylink/displaylink-installer.sh detach %k"
```

When a DisplayLink dock is plugged in, udev fires the `attach` helper, which triggers the DLM to write to `/sys/bus/platform/driver/evdi/add` (creating a new virtual evdi device) and then open the resulting `/dev/dri/cardN` node.

### systemd Service

```bash
# Check DLM status:
systemctl status displaylink-driver

# Follow log output:
journalctl -u displaylink-driver -f --since "1 minute ago"

# Restart DLM (required after evdi module reload):
sudo systemctl restart displaylink-driver
```

The service runs as `root` to access USB bulk endpoints and DRM device nodes. It does not require `CAP_SYS_ADMIN` for normal operation but does require read/write access to `/dev/dri/cardN` and `/dev/bus/usb/<bus>/<device>`.

### Why DisplayLink Manager Is Closed Source

The question of why Synaptics will not open-source the DLM has a short official answer and several unstated reasons that are more technically interesting.

**The official answer.** DisplayLink's support knowledgebase article on this topic states: *"The vast majority of the application's code is in fact reused on every operating system we support, meaning that it's also used in our drivers for Windows, macOS, Android, Chrome OS, etc."* [[Source](https://support.displaylink.com/knowledgebase/articles/1104056)]. The plugin architecture allows a single core codebase to run on all platforms with only the source/sink plugins differing per OS. Open-sourcing the Linux DLM would open the Windows and macOS code too.

**The codec as competitive IP.** The DL3 adaptive codec is DisplayLink's core technical differentiator. Synaptics holds over 76 patents covering the compression algorithms, bandwidth-adaptation logic, and display-over-USB protocol. The codec is what allows a USB 3.0 connection to drive a 4K@60Hz display at acceptable quality; without it, raw pixel transmission would require ≈16 Gbps for uncompressed 4K@60Hz XRGB8888 — far beyond USB 3.2 Gen 2's 10 Gbps ceiling. Publishing the codec source would allow competitors to implement compatible decoders in alternative USB display chips, collapsing the value of the chipset licensing business.

**HDCP 2.0 — the hard legal barrier.** Modern DL-3xxx and later chips include HDCP 2.0 (High-bandwidth Digital Content Protection). HDCP 2.0 licensing from Digital Content Protection LLC explicitly prohibits open-source implementations because they cannot enforce private key secrecy: the HDCP specification requires that device keys remain secret, which is technically unenforceable in open-source software. This is the same barrier that prevents open-source implementations of HDCP in the Linux DRM subsystem — the kernel only exposes HDCP state machine hooks (`drm_hdcp.c`) while actual key management remains in proprietary blobs. Any open-source DLM that included HDCP 2.0 support would be in licence violation with DCP LLC, meaning an open DLM would either omit HDCP (making protected content unplayable) or remain proprietary by legal necessity. **Note**: DisplayLink has not publicly cited HDCP as a reason; this is a structural inference from DCP LLC licensing terms.

**Third-party licensed components.** The DLM's USB communication layer may incorporate codecs or protocol components licensed from third parties whose licences prohibit source disclosure. Synaptics acquired DisplayLink in 2020 and has a history of incorporating licensed IP from third-party silicon vendors into its touch and display IC firmware stacks. The official statement does not address this.

**Community pressure and the upstream block.** The feature request for an open-source DLM on the DisplayLink forum has accumulated over 2,000 votes — making it the highest-voted item — with no substantive official response beyond automated acknowledgment [[Source](https://support.displaylink.com/forums/287786-displaylink-feature-suggestions/suggestions/17798941-make-the-linux-driver-open-source-and-get-it-into)]. The practical consequence: this split architecture — open kernel interface, closed userspace — is precisely why evdi cannot be upstreamed into mainline Linux. The DRM subsystem maintainers (Daniel Vetter's documented position, echoed in LKML review threads) require an open userspace companion before accepting drivers that define new DRM uAPI. Without an open DLM, the `EVDI_CONNECT`, `EVDI_GRABPIX`, and `EVDI_REQUEST_UPDATE` ioctls will never be in-tree, evdi will never be DKMS-free, and every kernel update will continue to require a DKMS rebuild.

---

## DRM Integration and Atomic Modesetting

### Compositor View

From the compositor's perspective, evdi presents as a normal DRM device with standard KMS objects. The compositor discovers it via `drmModeGetResources()` and programs it identically to a real GPU output:

```bash
# Inspect the evdi DRM device:
modetest -M evdi -c
# Output: Connector HDMI-A-1, 1920x1080@60, connected

# Verify with drm-info:
drm-info /dev/dri/card2   # substitute actual evdi card number
```

The compositor (Mutter, KWin, sway) performs an atomic commit to evdi exactly as it would to a real GPU:

```c
/* Compositor atomic commit to evdi device (standard DRM): */
drmModeAtomicAddProperty(req, crtc_id, mode_id_prop, mode_blob_id);
drmModeAtomicAddProperty(req, conn_id, crtc_id_prop, crtc_id);
drmModeAtomicAddProperty(req, plane_id, fb_id_prop, fb_id);
/* Attach damage clips for efficient dirty tracking: */
drmModeAtomicAddProperty(req, plane_id, fb_damage_clips_prop,
    damage_blob_id);
drmModeAtomicCommit(evdi_fd, req,
    DRM_MODE_ATOMIC_FLIP_EVENT | DRM_MODE_ATOMIC_NONBLOCK, NULL);
```

The `FB_DAMAGE_CLIPS` plane property is crucial for performance: it tells evdi precisely which rectangles changed so only those regions need to be encoded and transmitted.

### DMA-BUF Import and the CPU Copy

When the primary GPU renders to a DMA-BUF and the compositor attaches that buffer to the evdi plane, evdi must perform a CPU copy. The adapter chip cannot perform PCIe DMA — it has no host memory access. The copy path is:

```
GPU renders → DMA-BUF (GPU VRAM or system memory)
  ↓ CPU: kmap() / vmap() the DMA-BUF pages
  ↓ CPU: copy to evdi GEM shmem pages (with clflush)
  ↓ EVDI_GRABPIX: copy GEM pages to DLM client buffer
  ↓ DLM: DL3 encode → USB bulk transfer
```

On modern hardware (DDR5, high-bandwidth integrated graphics), this double-copy typically takes 3–10 ms for 1080p. It is a hard architectural constraint: until a zero-copy DMA-engine path exists (see Roadmap), all DisplayLink data must transit the CPU.

### No Hardware Overlays

evdi implements a single primary plane. The DRM plane model allows hardware to composite multiple layers (cursor plane, overlay planes) without a full GPU pass. On a real GPU, this is done in scanout hardware. On evdi, there is no scanout hardware — the adapter chip receives a pre-composited stream. This means:

- No cursor plane (cursor is software-composited, see Section 7)
- No overlay planes for video or UI layers
- No YUV hardware conversion in the adapter
- Every pixel presented to the display must flow through the DLM encode path

---

## USB4, Thunderbolt, and the DL-7400 Generation

### USB4 and DisplayPort 2.1

The DL-7400 is Synaptics's current-generation DisplayLink chip, announced in 2024. It represents a fundamental architectural advance over the DL-6xxx series:

**Key DL-7400 capabilities** [Source](https://www.synaptics.com/company/news/synaptics-introduces-displayport-21-and-displaylink-quad-display):
- Quad 4K@120Hz HDR10 display support
- Dual 8K@60Hz HDR10 display support
- DisplayPort 2.1 Alt Mode (UHBR10 / UHBR20 signaling)
- Compatible with USB4, Thunderbolt 3/4, and DisplayPort Alt Mode over USB-C
- 2.5 Gbps Ethernet integrated
- Explicit support for Variable Refresh Rate (VRR/G-Sync/FreeSync)

### USB4 vs. USB 3.2 Bandwidth

The DL-7400 communicates with the host over USB4 tunneled links:

| Interface | Raw Bandwidth | Practical Video Bandwidth |
|---|---|---|
| USB 3.2 Gen 1 (5 Gbps) | 500 MB/s | ~200–250 MB/s for display |
| USB 3.2 Gen 2 (10 Gbps) | 1,000 MB/s | ~400–500 MB/s |
| USB4 Gen 2×2 (20 Gbps) | 2,000 MB/s | ~800 MB/s |
| USB4 Gen 3×2 (40 Gbps) | 4,000 MB/s | ~1.5 GB/s |
| Thunderbolt 4 (40 Gbps) | 4,000 MB/s | ~1.5 GB/s |

With USB4 / Thunderbolt 4 at 40 Gbps, the DL-7400 has sufficient bandwidth to transmit quad-4K@120Hz streams after DL3 compression. Compare this to the DL-6xxx on USB 3.0 (5 Gbps), where 4K@30Hz was achievable but 4K@60Hz required high compression ratios.

### USB4 Alt Mode vs. DP Tunneling

USB4 supports two mechanisms for display data:

1. **DisplayPort Tunneling**: the USB4 fabric creates a DP tunnel that carries raw DisplayPort packets. The endpoint is a standard DP sink (no DisplayLink chip needed). This is native DP over USB4 — supported by the Linux `thunderbolt` / `typec` kernel subsystem.

2. **DisplayLink over USB4**: the DL-7400 chip uses the USB4 fabric as a high-bandwidth USB device — it enumerates as a USB device (VID 17e9) and carries DL3-encoded streams over USB protocol. The higher USB4 bandwidth enables the higher resolutions.

These are distinct and non-interchangeable. A USB4 dock with a DL-7400 chip requires the full evdi + DLM stack. A USB4 dock using native DP tunneling requires only the kernel's built-in DP Alt Mode / DRM connector code.

### Linux Support for DL-7400

As of Linux 6.10 and evdi ≥ 1.14, the DL-7400 is supported by the standard evdi + DLM stack. The VID/PID is 17e9 (DisplayLink family), and the same udev rule triggers DLM association. The chip appears over USB4 but the software path is identical to DL-6xxx — only bandwidth and resolution limits change.

> **Note: needs verification.** Specific evdi version requirements for DL-7400 and minimum kernel version for reliable USB4 enumeration should be confirmed against release notes.

---

## Limitations and Performance

### Latency

DisplayLink adds measurable latency to the display path that does not exist on a direct-attach GPU output:

1. **Compositor commit → evdi dirty event**: ≈ 0–2 ms (kernel scheduling)
2. **EVDI_GRABPIX CPU copy** (1080p): ≈ 1–5 ms
3. **DLM DL3 encoding** (1080p, full frame): ≈ 5–15 ms (CPU-dependent)
4. **USB bulk transfer** (USB 3.0, 1080p compressed): ≈ 2–8 ms
5. **Chip DL3 decode + display output**: ≈ 1–3 ms

Total round-trip latency from compositor commit to display pixel: **≈ 10–35 ms**, compared to ≈ 1–5 ms for a directly connected display with vsync. This latency is imperceptible for office work and web browsing but is noticeable during interactive gaming or low-latency applications.

### CPU Overhead

All encoding is done on the host CPU in the DLM daemon. Observed CPU utilization:

- **Static desktop** (no movement): ≈ 1–3% of one CPU core
- **Mouse cursor movement** (default cursor mode): ≈ 10–15% of one CPU core (every mouse position triggers a dirty event for the cursor bounding box)
- **Video playback (1080p)**: ≈ 25–40% of one CPU core
- **4K@60Hz full-motion content**: ≈ 60–100% of one CPU core (may not be achievable in real-time)

Source: community measurements via Arch Linux forums and DisplayLink's own product documentation. [Source](https://displaylink.org/forum/archive/index.php/t-66909.html)

Using `EVDI_ENABLE_CURSOR_EVENTS` reduces cursor-motion CPU overhead by decoupling cursor position updates from framebuffer grabs. This is the single most impactful optimization for typical desktop use.

### Why Gaming Through DisplayLink Is Impractical

Three factors combine to make DisplayLink unsuitable for gaming:

1. **Frame latency**: The 10–35 ms encode-transfer pipeline adds to input lag on top of game engine latency and display response time. Competitive gaming targets < 5 ms total display pipeline latency.

2. **No hardware overlays**: Games that use the DRM overlay plane (PRIME offload output) or direct scanout (DRM_MODE_PAGE_FLIP) cannot bypass the evdi CPU copy path. Every frame must be captured, encoded, and transmitted.

3. **Variable encode time**: The DL3 encode time is content-dependent. Complex scenes (many edges, high spatial frequency) encode slower than simple ones, introducing frame-rate jitter that manifests as stuttering.

DisplayLink documentation explicitly targets "productivity applications" and notes it is "not recommended for high-motion content." [Source](https://kb.plugable.com/docking-stations-and-video/displaylink-based-displays-running-slow-heres-how-to-improve-performance)

### No GPU Acceleration

There is no GPU in the DisplayLink adapter. This means:

- No hardware video decode on the display side (no VDPAU/VAAPI offload to the dock)
- No 2D/3D acceleration on the adapter (all rendering on host GPU)
- No hardware compositor in the dock (all compositing on host CPU/GPU, with result transmitted to dock)
- No HDR tone mapping on the adapter side (host must produce the correct signal before encoding)

### Interaction with NVIDIA Proprietary Driver

DisplayLink on Linux requires an open-source GPU driver on the primary display adapter. The NVIDIA proprietary driver does not expose DMA-BUFs in a form compatible with evdi's CPU-copy path; additionally, NVIDIA's proprietary modesetting stack does not support the atomic `FB_DAMAGE_CLIPS` property in a form evdi can consume. As a result, DisplayLink officially does not support NVIDIA proprietary driver configurations on Linux. Systems with Intel or AMD primary GPUs using Mesa (i915, amdgpu, nouveau) work correctly. [Source](https://support.displaylink.com/knowledgebase/articles/615714)

---

## Open-Source Alternatives and the Ecosystem

### udlfb: The Legacy fbdev Driver

The original DisplayLink USB 2.0 driver in the Linux kernel was `udlfb`, a framebuffer device (`/dev/fb*`) driver for the older "Alex" and "Ollie" DisplayLink chips (USB VID 17e9 with USB 2.0). It is documented in the kernel at `Documentation/fb/udlfb.rst` [Source](https://docs.kernel.org/fb/udlfb.html).

`udlfb` has two fundamental limitations:
1. **fbdev's mmap model** assumes a real hardware framebuffer. For USB, all writes to the mapped region must be detected and encoded into USB bulk transfers by the CPU — typically via deferred I/O (`fb_defio`), which uses the page-dirty bits to detect which pages changed.
2. **No DRM integration**: fbdev lacks the atomic modesetting, damage clip properties, and plane model that enable efficient dirty-region tracking.

A patch to remove `udlfb` was proposed in 2020 ([dri-devel, November 2020](https://lists.freedesktop.org/archives/dri-devel/2020-November/289574.html)), and the driver is considered legacy. Note: `fb_defio` (deferred framebuffer I/O) is the **udlfb** damage-tracking mechanism — not evdi's. evdi uses `drm_atomic_helper_dirtyfb` and the `FB_DAMAGE_CLIPS` DRM plane property instead.

### udl: The In-Kernel DRM DisplayLink Driver

`udl` (`drivers/gpu/drm/udl/`) is the in-kernel DRM driver for USB 2.0-era DisplayLink hardware. It is the DRM successor to `udlfb` and is fully open-source and mainline. `udl` supports only USB 2.0 DisplayLink chips (DL-1xx, DL-1x5) and is limited to 1920×1080@60. It does not use evdi or the DLM.

For users with legacy USB 2.0 DisplayLink docks, `udl` is the preferred driver: no proprietary daemon required. [Source](https://www.kernel.org/doc/html/latest/gpu/)

### evdi + DLM: Current Stack for USB 3.0+

For USB 3.0 and newer DisplayLink hardware (DL-3xxx through DL-7400), the required stack is evdi (open source, out-of-tree) + DLM (proprietary). The split is intentional: evdi defines the kernel ↔ userspace interface, and the DLM provides the proprietary DL3 codec and USB protocol implementation.

### Reverse Engineering Efforts

The reverse-engineering landscape for DisplayLink breaks cleanly into two eras: the USB 2.0 DL-1xx chips (fully reverse-engineered and open) and the USB 3.0+ DL-3xxx/5xxx/6xxx generation (largely undocumented, with only recent meaningful progress).

**libdlo and the udl precedent.** Around 2009, DisplayLink voluntarily published **libdlo** (LGPL), a reference library for their DL-1x5 USB 2.0 chips. This deliberate disclosure formed the basis for the `udl` kernel DRM driver and established the only period in DisplayLink's history where community development of an open driver was explicitly sanctioned. The libdlo codebase is archived at [github.com/freddy77/libdlo](https://github.com/freddy77/libdlo); the `udl` driver that descends from it covers DL-120, DL-160, DL-1x5 class hardware and is fully mainline at `drivers/gpu/drm/udl/`. No equivalent disclosure has ever been made for DL-3xxx or later hardware.

**Vino: The First Serious DL-3xxx Reverse Engineering Effort (2026).** The most significant development in the DisplayLink open-source landscape in years arrived in June 2026: **Mike Lothian** published an experimental Rust DRM kernel driver called **Vino** targeting the **DL-3xxx** chipset family, tested on a Dell Universal Dock D6000. Uniquely, Lothian acknowledged that the driver was developed with AI assistance (including Claude/Opus) for protocol analysis and code generation [[Source](https://www.phoronix.com/news/Experimental-Vino-DRM-Driver)].

Vino's approach is a clean-room reverse engineering: USB bulk transfers are captured with `usbmon` and analysed to infer the command/response structure of the DL-3xxx protocol, without reference to the DLM binary. The driver is written in Rust using the in-kernel Rust DRM abstractions introduced in Linux 6.7+. As of mid-2026 it is experimental — tested on a single device, not merged upstream, and its handling of DL3 compression (whether it decodes compressed frames or limits itself to uncompressed modes) had not been publicly clarified. **Note: needs verification** — the GitHub repository URL and exact compression support status require confirmation.

The significance of Vino is architectural: if successful, it would be the first mainline-eligible DRM driver for USB 3.0-era DisplayLink hardware that requires no proprietary userspace. The evdi maintainers have not commented publicly on Vino's prospects.

**Protocol analysis tools.** Several small projects target the USB protocol layer without attempting a full driver:

- **dlresp** ([github.com/kjy00302/dlresp](https://github.com/kjy00302/dlresp)): A Linux FunctionFS gadget that emulates a DisplayLink device from the device side — useful for observing how the DLM initialises and communicates with hardware without needing a physical chip.
- **dlemu-rs**: A Rust bulk-transfer stream viewer for observing USB captures — useful for initial protocol analysis but not a driver.
- `usbmon` + Wireshark: The standard approach for capturing DisplayLink USB traffic. The DisplayLink USB VID is `17e9`; bulk transfers on endpoint `0x02` (OUT) carry the encoded stream. The DL-3xxx frame structure reportedly begins with a fixed-length command header followed by tile data, but no complete specification has been published.

**The DL3 codec: still opaque.** Despite multiple RE attempts, the internal structure of the DL3 compression codec for DL-3xxx through DL-6xxx remains publicly unknown. The USB 2.0 DL-1xx protocol was simpler (run-length encoding over raw RGB565 was documented in the libdlo source); the DL3 codec on DL-3xxx is substantially more complex. Community speculation includes JPEG-like DCT for lossy passes and LZ-family compression for lossless paths, but no disassembly of the dlm binary or successful packet decode has been published. The HDCP key material in the firmware makes disassembly legally hazardous under DMCA §1201 in the United States.

### Hardware Alternatives: Avoiding DisplayLink Entirely

For users and system designers willing to choose different hardware, several approaches avoid the DisplayLink software stack entirely:

**USB-C DisplayPort Alt Mode** routes raw DisplayPort signal lanes over the USB-C physical connector without any compression or USB protocol encapsulation. A dock implementing DP Alt Mode drives displays as a native DP sink — the kernel's `typec` / `drm_dp` subsystem handles everything, and no proprietary daemon is required. Compatible with any GPU that exposes DP Alt Mode pins. Limitations: single display output per DP Alt Mode cable; no USB data on the same lanes while DP is active (unless the cable supports lane multiplexing); max resolution limited by DP Alt Mode bandwidth (DP 1.4a = 32.4 Gbps, DP 2.1 = 80 Gbps).

**USB4 and Thunderbolt DisplayPort Tunneling** extends the same concept: the USB4 fabric creates a DP tunnel that carries raw DisplayPort packets end-to-end. Linux's `thunderbolt` kernel driver negotiates the tunnel; the display appears as a standard DRM connector. This is distinct from DisplayLink's DL-7400 — which uses USB4 *as a transport for DL3-encoded data* and still requires the evdi + DLM stack. A USB4 dock that uses DP tunneling (vs. DisplayLink) is driver-free on Linux.

**Multi-display DP hubs (MST)** use DisplayPort Multi-Stream Transport to drive multiple monitors from a single DP connection. Linux's `drm_dp_mst_topology.c` MST topology manager handles hub enumeration natively; no userspace daemon is required. Limited to the GPU's native DP bandwidth and the MST hub's configuration.

The practical guidance for Linux desktop users: if your dock supports native DP Alt Mode or USB4 DP tunneling, use it — you get lower latency, lower CPU overhead, and no proprietary daemon dependency. DisplayLink docks are appropriate when you need more displays than your GPU has native DP outputs, when connecting over a port that only has USB 3.x (not DP Alt Mode), or when using hardware that specifically ships DisplayLink docks (many business docking stations).

**Wayland and the DisplayLink problem.** Under Wayland, DisplayLink has an additional systemic problem beyond the closed DLM: screen capture and recording via the `xdg-desktop-portal` / `pipewire-capture-desktop` path does not work correctly through evdi virtual displays. Applications that query Wayland screen-capture (OBS Studio, video conferencing via WebRTC pipewire capture) cannot capture evdi outputs. The root cause is that evdi virtual displays are not visible to the Wayland compositor's DMA-BUF export paths used by the portal API. This is an architectural gap with no short-term fix, since fixing it requires changes to both the compositor and potentially evdi itself [[Source](https://github.com/DisplayLink/evdi/issues)].

### Community Open-Source Efforts

Several community projects extend or package components of the evdi stack (these do not replace the proprietary DLM):

**pyevdi** ([github.com/DisplayLink/pyevdi](https://github.com/DisplayLink/pyevdi)): Python bindings for libevdi. Enables scripting and testing of the evdi ioctl interface without the proprietary DLM — useful for writing test harnesses or alternative evdi clients.

**evdipp**: A minimal C++ example client for evdi/libevdi demonstrating the complete connection-update-grab loop. Useful as a reference for developers implementing custom evdi clients.

**displaylink-rpm** ([github.com/displaylink-rpm/displaylink-rpm](https://github.com/displaylink-rpm/displaylink-rpm)): Community-maintained RPM build scripts that repackage the DisplayLink `.run` installer for Fedora, CentOS Stream, Rocky Linux, and AlmaLinux [[Source](https://github.com/displaylink-rpm/displaylink-rpm)].

**displaylink-debian** ([github.com/AdnanHodzic/displaylink-debian](https://github.com/AdnanHodzic/displaylink-debian), ~1,400 stars): A shell script that extracts and repackages the DisplayLink `.run` installer as a proper Debian package with correct DKMS integration. These packaging scripts are critically dependent on the proprietary dlm binary and break with each kernel update that changes DRM internal APIs.

### USB CDC Display Class: Not a Current Reality

There is no standardized USB Display Device Class analogous to USB Audio or USB Mass Storage. Each USB display vendor implements a proprietary protocol. The DisplayLink protocol (DL3 + USB bulk) is proprietary; the `udl` driver was reverse-engineered and disclosed by DisplayLink for USB 2.0 hardware, but the modern DL3 protocol for DL-3xxx/5xxx/6xxx is not publicly specified. Proposals for a USB display class have been discussed in standards bodies but have not resulted in a standardized class code as of 2026. Note: needs verification.

---

## Troubleshooting and Diagnostics

### Common Issues

**Black screen after plug-in:**
```bash
# Check DLM is running:
systemctl status displaylink-driver

# Check evdi module is loaded:
lsmod | grep evdi

# Check USB device is enumerated:
lsusb | grep -i displaylink
# e.g.: Bus 002 Device 003: ID 17e9:6006 DisplayLink

# Check new DRM device appeared:
ls -la /dev/dri/
# A new cardN should appear when DLM connects to evdi

# Check DLM log for connection errors:
journalctl -u displaylink-driver --since "5 minutes ago"
```

**Low performance / stuttering:**
```bash
# Verify USB speed (should be SuperSpeed 5 Gbps or better):
lsusb -t | grep -A 3 17e9
# Look for: speed=5000Mbps or higher

# Check DLM CPU usage:
top -p $(pgrep -x DisplayLinkMana)  # actual process name may vary

# Enable cursor event mode in DLM config to reduce cursor-drag CPU:
# (Depends on DLM version; check Synaptics documentation)

# Disable compositor effects (e.g., Kwin effects) for the evdi output
```

**EDID not recognized / wrong resolution:**
```bash
# Check EDID is loaded by evdi connector:
drm-info /dev/dri/card2  # substitute actual evdi card number

# Force a mode:
xrandr --addmode HDMI-1 "1920x1080"
xrandr --output HDMI-1 --mode "1920x1080"

# Under Wayland, use wlr-randr or KScreen to force mode
```

**evdi module fails to build with DKMS:**
```bash
# Install kernel headers for current kernel:
sudo apt install linux-headers-$(uname -r)

# Trigger DKMS rebuild:
sudo dkms autoinstall

# Check DKMS status:
dkms status
# Expected: evdi/<version>, <kernel-version>, <arch>: installed

# If build fails, check error log:
cat /var/lib/dkms/evdi/<version>/<kernel>/<arch>/log/make.log
```

**Module not compatible with kernel 6.x:**

evdi is an out-of-tree module and must track kernel DRM API changes. Known breakage points:
- Kernel 6.0: `drm_irq.h` removed as legacy
- Kernel 6.6: atomic helper API changes
- Kernel 6.17+: ongoing compatibility maintained in evdi `main` branch

Always use the evdi version bundled with the DLM installer, or the latest tag from the evdi GitHub repo that lists your kernel version in its compatibility matrix. [Source](https://github.com/DisplayLink/evdi/issues/433)

### Debug Logging

```bash
# Enable evdi kernel debug messages via dynamic debug:
echo 'module evdi +p' | sudo tee /sys/kernel/debug/dynamic_debug/control

# Watch kernel messages:
dmesg -w | grep -i evdi

# Verbose DLM log:
journalctl -u displaylink-driver -f

# Check evdi sysfs state:
ls /sys/bus/platform/devices/ | grep evdi
cat /sys/bus/platform/devices/evdi.0/count  # number of virtual displays

# Trace USB bulk transfers (requires usbmon):
sudo modprobe usbmon
sudo tcpdump -i usbmon2 -w /tmp/usb-capture.pcap  # substitute bus number
```

### Performance Profiling

```bash
# Measure DLM CPU cost with perf:
sudo perf top -p $(pgrep -x DisplayLinkMana)

# Trace evdi ioctl call rates:
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_ioctl
  /comm == "DisplayLinkMana"/
  { @[args->cmd] = count(); }
  interval:s:5 { print(@); clear(@); }
'
```

---

## Roadmap

### Near-term (6–12 months)

- **Kernel 6.x compatibility maintenance**: evdi is out-of-tree and must be updated for each DRM API change; the project has tracked upstream through kernel 6.15 as of mid-2026. [Source](https://github.com/DisplayLink/evdi)
- **Wayland compositor session handoff fixes**: Active bugs cause module state corruption when Wayland compositor instances restart (e.g., log out and log back in). Patches addressing teardown and re-connection sequencing are under discussion. [Source](https://github.com/DisplayLink/evdi/issues/540)
- **DL-7400 Linux driver maturation**: USB4/Thunderbolt enumeration edge cases and multi-4K configuration stability are being worked on.
- **HDR metadata passthrough**: As Wayland compositors gain HDR support via `color-management-v1`, evdi will need to propagate `HDR_OUTPUT_METADATA` connector properties through the painter to the DLM. Note: needs verification.

### Medium-term (1–3 years)

- **Open-source DLM**: The most-requested community goal — Synaptics opening the DLM — would unblock evdi upstreaming. Without an open userspace, the DRM maintainers cannot accept evdi into mainline. [Source](https://support.displaylink.com/forums/287786-displaylink-feature-suggestions/suggestions/17798941-make-the-linux-driver-open-source-and-get-it-into)
- **Upstream kernel merge**: If the DLM is opened, a formal RFC patchset for mainlining evdi under `drivers/gpu/drm/evdi/` becomes viable, analogous to how `udl` covers USB 2.0 hardware. Note: needs verification for timeline.
- **`FB_DAMAGE_CLIPS` adoption in more compositors**: Broader compositor support for damage clips reduces the frequency of full-framebuffer grabs. KWin and Mutter have relevant work underway.
- **GPU-side encode offload**: Offloading DL3 encode to the host GPU's video encoder (VAAPI/NVENC) could reduce CPU overhead significantly for 4K streams. Note: needs verification.

### Long-term

- **DMA-BUF zero-copy path**: The fundamental latency and CPU overhead arise from the double CPU copy (GPU→evdi GEM, evdi GEM→DLM buffer). A kernel DMA-engine path scattering dirty tiles directly into USB transfer buffers would bypass both copies. This requires a redesign of the evdi ↔ DLM interface. Note: needs verification.
- **USB4 native DP tunneling displacement**: USB4 native DisplayPort tunneling (built into the Linux kernel `thunderbolt`/`typec` subsystem) may displace DisplayLink for high-end docks on USB4/Thunderbolt 4 hosts, since it requires no proprietary codec and achieves lower latency. DisplayLink's value proposition may shift toward lower-cost USB 3.x multi-display scenarios.
- **Wireless docking**: The evdi architecture (virtual DRM + userspace transport) is transport-agnostic. Future directions include Wi-Fi 7 (IEEE 802.11be) wireless docking, with the DLM replaced by an open transport daemon. Note: needs verification.

---

## Integrations

- **Ch01 (DRM Architecture)** — evdi is a DRM driver; it registers a `drm_driver` with `DRIVER_MODESET | DRIVER_GEM | DRIVER_ATOMIC` and creates device nodes via `drm_dev_register()`, identically to any real GPU driver
- **Ch04 (Framebuffer and Planes)** — evdi implements a single primary plane; damage tracking uses the `FB_DAMAGE_CLIPS` DRM plane property and `drm_atomic_helper_dirtyfb`; `udlfb` used the older `fb_defio` page-dirty mechanism
- **Ch06 (KMS Atomic Modesetting)** — compositors commit frames to evdi via the standard atomic commit path; evdi's connector/CRTC appear in `drmModeGetResources()` and are programmed identically to physical display outputs
- **Ch11 (DMA-BUF)** — evdi must CPU-copy imported DMA-BUFs since the USB chip cannot perform PCIe DMA; this double-copy (GPU→evdi GEM, evdi GEM→DLM buffer) is the core performance and latency limitation
- **Ch49 (Multi-GPU)** — DisplayLink monitors appear as additional DRM devices; DRI_PRIME is not needed since evdi has no 3D capability — it is display-only
- **Ch139 (DRM Hardware Planes)** — evdi supports only a single primary plane; hardware overlay planes are not possible since the DisplayLink chip has no compositor and no GPU
- **Ch147 (VA-API and Video Decode)** — GPU-side video decode offload (VA-API) cannot be tunnelled through evdi; decoded frames must be read back to system memory and re-encoded via DL3 before USB transmission
- **Ch172 (eGPU / Thunderbolt / USB4)** — USB4 native DisplayPort tunneling (built into the kernel's `thunderbolt` subsystem) is an alternative to DisplayLink for high-end USB4 docks that does not require a proprietary stack

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
