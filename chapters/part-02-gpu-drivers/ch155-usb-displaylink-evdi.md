# Chapter 155: USB DisplayLink and the evdi Kernel Module

**Target audiences**: Systems engineers adding USB display support to embedded or desktop Linux systems; driver developers studying evdi as an example of a user-space DRM driver; and laptop users debugging multi-monitor setups over DisplayLink docks.

---

## Table of Contents

1. [Introduction](#introduction)
2. [DisplayLink Hardware and Protocol](#displaylink-hardware-and-protocol)
3. [evdi: Extensible Virtual Display Interface](#evdi-extensible-virtual-display-interface)
4. [evdi Kernel Module Architecture](#evdi-kernel-module-architecture)
5. [libevdi: The Userspace Library](#libevdi-the-userspace-library)
6. [DisplayLink Manager (DLM) and the Full Stack](#displaylink-manager-dlm-and-the-full-stack)
7. [DRM Integration and Atomic Modesetting](#drm-integration-and-atomic-modesetting)
8. [Compression and USB Bandwidth](#compression-and-usb-bandwidth)
9. [Troubleshooting and Diagnostics](#troubleshooting-and-diagnostics)
10. [Integrations](#integrations)

---

## Introduction

DisplayLink is a USB-to-display technology by Synaptics that lets a USB 3.0 or USB-C connection drive an external monitor. It is widely used in USB docking stations (Dell, Lenovo, CalDigit, Plugable). On Linux, DisplayLink depends on the **evdi** (Extensible Virtual Display Interface) kernel module, which creates a virtual DRM device that the userspace DisplayLink Manager (DLM) can write pixels into.

The architecture is unusual: evdi is a DRM driver whose "GPU" is actually a userspace daemon that compresses the framebuffer and sends it over USB. This makes it a useful case study in virtual DRM devices and the boundary between kernel DRM and userspace display processing.

Sources: [evdi on GitHub](https://github.com/DisplayLink/evdi) | [DisplayLink Linux driver](https://www.synaptics.com/products/displaylink-graphics/downloads/ubuntu)

---

## DisplayLink Hardware and Protocol

### USB Display Architecture

```
GPU (Intel/AMD/NVIDIA)
  │ renders to framebuffer
  ▼
evdi virtual DRM device (kernel module)
  │ exports framebuffer updates via mmap + netlink
  ▼
DisplayLink Manager (DLM) userspace daemon
  │ compresses (JPEG/custom codec)
  ▼
libusb → USB 3.0 host controller
  │
  ▼
DisplayLink chip (DL-6xxx, DL-7xxx) in dock/adapter
  │
  ▼
HDMI/DP/DVI output to monitor
```

### DisplayLink USB Protocol

DisplayLink uses a proprietary USB bulk transfer protocol. The DLM daemon:
1. Receives raw pixel regions from evdi (where the framebuffer changed)
2. Compresses them (JPEG or DisplayLink's DLCL codec)
3. Sends compressed tiles to the DisplayLink chip via USB bulk transfers
4. The chip decompresses and drives the display's pixel clock

USB 3.0 provides ~400 MB/s practical throughput — sufficient for 1920×1080@60 with ~4:1 compression. USB 3.2 Gen 2 (10 Gbps) supports 4K@60.

### Supported Chipsets

| Chip | Max Resolution | USB Speed | Notes |
|---|---|---|---|
| DL-3xxx | 1920×1080@60 | USB 2.0 | Legacy, slow |
| DL-5xxx | 2560×1440@60 | USB 3.0 | Common in docks |
| DL-6xxx | 3840×2160@30 | USB 3.0 | 4K (30 Hz) |
| DL-7xxx | 3840×2160@60 | USB 3.2 Gen2 | 4K@60, current gen |

---

## evdi: Extensible Virtual Display Interface

### What evdi Provides

evdi (`drivers/gpu/drm/evdi/` in its out-of-tree repo) is a DRM driver that:
- Creates a virtual `/dev/dri/card*` DRM device
- Accepts framebuffer updates from a connected DRM master (the compositor)
- Notifies userspace about which regions of the framebuffer changed (damage tracking)
- Delivers those regions to userspace via a shared memory interface

evdi does not itself do USB — it is a generic virtual display. The DisplayLink Manager is the component that reads from evdi and sends over USB.

### evdi Architecture

```
Compositor (Mutter/KWin/sway)
  │ opens /dev/dri/cardN (evdi virtual device)
  │ sets mode, creates framebuffer
  ▼
evdi kernel module
  │ tracks framebuffer writes
  │ exports damage regions via update events
  ▼
DLM userspace (via /dev/evdi/cardN char device)
  │ reads pixel data + damage rects
  │ compresses + sends via USB
```

---

## evdi Kernel Module Architecture

### Module Initialization

```c
/* evdi/module/evdi_drm_drv.c */
static int __init evdi_init(void)
{
    /* Register as a platform driver: */
    platform_driver_register(&evdi_platform_driver);
    /* Register character device for userspace DLM: */
    evdi_register_char_dev();
    return 0;
}

/* Creates a new virtual DRM device on demand: */
static int evdi_platform_probe(struct platform_device *pdev)
{
    struct drm_device *drm;
    drm = drm_dev_alloc(&evdi_driver, &pdev->dev);
    evdi_modeset_init(drm);    /* modesetting hooks */
    evdi_fbdev_init(drm);      /* legacy fbdev compat */
    drm_dev_register(drm, 0);  /* expose as /dev/dri/cardN */
}
```

### DRM Connector and CRTC

evdi implements a minimal DRM modesetting stack:

```c
/* evdi/module/evdi_connector.c */
static const struct drm_connector_funcs evdi_connector_funcs = {
    .fill_modes          = drm_helper_probe_single_connector_modes,
    .detect              = evdi_connector_detect,
    .destroy             = drm_connector_cleanup,
    .reset               = drm_atomic_helper_connector_reset,
    .atomic_duplicate_state = drm_atomic_helper_connector_duplicate_state,
    .atomic_destroy_state   = drm_atomic_helper_connector_destroy_state,
};

/* Modes are reported by the DLM (which knows the real monitor's EDID): */
static int evdi_connector_get_modes(struct drm_connector *connector)
{
    struct evdi_connector *evdi_con = to_evdi_connector(connector);
    /* DLM has pushed EDID data; parse it: */
    drm_connector_update_edid_property(connector, evdi_con->edid);
    return drm_add_edid_modes(connector, evdi_con->edid);
}
```

### Framebuffer Damage Tracking

The critical evdi feature is damage tracking — knowing which pixels changed so evdi doesn't send the entire framebuffer over USB each frame:

```c
/* evdi/module/evdi_fb.c */
static void evdi_framebuffer_dirty(struct drm_framebuffer *fb,
    struct drm_file *file_priv, unsigned flags, unsigned color,
    struct drm_clip_rect *clips, unsigned num_clips)
{
    /* Merge damage rects and notify DLM: */
    evdi_painter_send_update(evdi_dev->painter, clips, num_clips);
}
```

With atomic modesetting, damage is tracked via `DRM_MODE_FB_DIRTY_MARK_DAMAGE` or the plane's `FB_DAMAGE_CLIPS` property.

### The Painter and Update Events

```c
/* evdi/module/evdi_painter.c */
struct evdi_painter {
    struct evdi_device *evdi;
    bool            is_connected;    /* DLM has opened char dev */
    struct mutex    lock;
    wait_queue_head_t updates_wq;
    /* Ring of pending update regions: */
    struct evdi_event_update updates[MAX_UPDATES];
    int update_ready;
};

void evdi_painter_send_update(struct evdi_painter *painter,
    struct drm_clip_rect *rects, int count)
{
    /* Copy damage rects into the painter's update queue: */
    mutex_lock(&painter->lock);
    /* ... store rects ... */
    painter->update_ready = 1;
    mutex_unlock(&painter->lock);
    wake_up_interruptible(&painter->updates_wq);
}
```

---

## libevdi: The Userspace Library

### Library API

libevdi provides a C API used by the DLM:

```c
#include <evdi_lib.h>

/* Open an evdi device: */
evdi_handle handle = evdi_open(0);  /* first evdi device */
if (handle == EVDI_INVALID_HANDLE) {
    /* Create a new evdi device: */
    evdi_add(1);  /* add 1 device via sysfs */
    handle = evdi_open(0);
}

/* Connect with monitor EDID: */
evdi_connect(handle, edid_data, edid_size,
    1920, 1080,       /* preferred resolution */
    60,               /* preferred refresh rate */
    DRM_FORMAT_XRGB8888);

/* Register update handler: */
struct evdi_event_context ctx = {
    .update_ready_handler = my_update_ready_callback,
};
evdi_register_event_handler(handle, &ctx);

/* Main loop: */
while (running) {
    evdi_handle_events(handle, &ctx);
}
```

### Grabbing Pixel Data

```c
static void my_update_ready_callback(int buf_id, void *user_data)
{
    evdi_handle handle = user_data;

    /* Acquire the grab buffer: */
    struct evdi_grab_pixels gp = {
        .rects     = damage_rects,
        .num_rects = MAX_RECTS,
    };
    /* Returns pixel data in gp.buffer for each damage rect: */
    evdi_grab_pixels(handle, gp.rects, &gp.num_rects);

    /* gp.rects[i].{x1,y1,x2,y2} = damage region */
    /* pixel data accessible via mmap of the evdi buffer */
    for (int i = 0; i < gp.num_rects; i++) {
        /* Compress and send the damaged region over USB: */
        displaylink_send_region(handle, &gp.rects[i]);
    }

    /* Release grab: */
    evdi_request_update(handle, buf_id);
}
```

### Buffer Registration

```c
/* Register a shared buffer between kernel and userspace: */
evdi_register_buffer(handle, 0,   /* buffer id */
    width, height,
    DRM_FORMAT_XRGB8888,
    pixels_ptr,                    /* userspace-allocated pixel buffer */
    stride);

/* evdi maps this buffer into kernel space; updates write directly here */
```

---

## DisplayLink Manager (DLM) and the Full Stack

### DLM Architecture

The DisplayLink Manager (closed-source, `displaylink-driver-*.run` from Synaptics) is a userspace daemon that:

1. Discovers DisplayLink USB devices via udev
2. Creates evdi virtual displays via `evdi_add()`
3. Connects each evdi handle to a USB device
4. Reads pixel updates from evdi and compresses them
5. Sends compressed tiles via libusb to the DisplayLink chip
6. Receives EDID from the chip and pushes it to evdi

```bash
# Install DLM:
# Download from: https://www.synaptics.com/products/displaylink-graphics/downloads/ubuntu
chmod +x displaylink-driver-6.0.run
sudo ./displaylink-driver-6.0.run   # installs evdi.ko + dlm.service

# Check status:
systemctl status displaylink-driver
journalctl -u displaylink-driver -f
```

### udev Integration

```bash
# DisplayLink USB devices:
lsusb | grep DisplayLink
# e.g.: Bus 002 Device 003: ID 17e9:6006 DisplayLink

# evdi devices after DLM starts:
ls /dev/dri/  # should show new card* device
```

---

## DRM Integration and Atomic Modesetting

### Compositor View

From the compositor's perspective, evdi presents as a normal DRM device:

```bash
# modetest on evdi device:
modetest -M evdi -c   # connectors
# Shows: HDMI-A-1 connected 1920x1080 ...
```

The compositor (Mutter/KWin/sway) sets modes and commits frames to the evdi DRM device exactly as with a real GPU output:

```c
/* Atomic commit to evdi (compositor internal): */
drmModeAtomicAddProperty(req, crtc_id, mode_id_prop, mode_blob_id);
drmModeAtomicAddProperty(req, conn_id, crtc_id_prop, crtc_id);
drmModeAtomicAddProperty(req, plane_id, fb_id_prop, fb_id);
drmModeAtomicCommit(evdi_fd, req, DRM_MODE_ATOMIC_FLIP_EVENT, NULL);
```

### Prime / DMA-BUF Import

When the GPU renders to a DMA-BUF and the compositor commits that buffer to evdi, evdi must CPU-copy the pixel data (since it can't do GPU DMA). This is the main performance limitation:

```
GPU renders → DMA-BUF → evdi: CPU maps DMA-BUF → copies to evdi pixel buffer → DLM reads
```

The copy is typically 5–15 ms for 1080p, meaning evdi displays are limited to ~30–40 fps practical maximum even on USB 3.2.

---

## Compression and USB Bandwidth

### Bandwidth Requirements

| Resolution | Refresh | Uncompressed | Required Compression |
|---|---|---|---|
| 1920×1080 | 60 Hz | ~380 MB/s | ~3× (USB 3.0 ~400 MB/s) |
| 2560×1440 | 60 Hz | ~670 MB/s | ~5× |
| 3840×2160 | 30 Hz | ~720 MB/s | ~5× |
| 3840×2160 | 60 Hz | ~1440 MB/s | ~4× (USB 3.2 Gen2) |

### Damage Region Compression

evdi only sends changed regions. For a typical desktop with mostly static content, only 10–20% of pixels change per frame — effective throughput is thus much lower than worst case.

```c
/* DLM compresses each damage rect independently: */
for each rect in damage_rects:
    uint8_t *pixels = evdi_get_pixels(rect);
    size_t compressed_size;
    uint8_t *compressed = displaylink_compress(pixels, rect.width, rect.height,
                                               &compressed_size);
    usb_bulk_send(compressed, compressed_size);
```

---

## Troubleshooting and Diagnostics

### Common Issues

**Black screen after plug-in:**
```bash
# Check DLM is running:
systemctl status displaylink-driver
# Check evdi loaded:
lsmod | grep evdi
# Check USB device seen:
lsusb | grep DisplayLink
# Check DRM device created:
ls -la /dev/dri/  # new card* should appear
```

**Low performance / stuttering:**
```bash
# Check USB speed (should be SuperSpeed 5 Gbps or better):
lsusb -t | grep -A2 DisplayLink
# Check CPU usage of DLM:
top -p $(pgrep dlm)
# Disable compositor effects if possible
```

**EDID not recognized:**
```bash
# Check monitor EDID via evdi:
drm-info /dev/dri/card2  # replace with evdi card number
# Force mode:
xrandr --addmode HDMI-1 "1920x1080" && xrandr --output HDMI-1 --mode "1920x1080"
```

**evdi module fails to build:**
```bash
# evdi requires matching kernel headers:
sudo apt install linux-headers-$(uname -r)
# Re-run DKMS build:
sudo dkms autoinstall
dkms status  # should show evdi/version, installed
```

### Debug Logging

```bash
# evdi kernel debug:
echo 'module evdi +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
dmesg | grep evdi

# DLM debug log:
journalctl -u displaylink-driver --since "5 minutes ago" -f
```

---

## Integrations

- **Ch01 (DRM Architecture)** — evdi is a DRM driver; it registers a `drm_driver` with `drm_dev_alloc`/`drm_dev_register` like any real GPU driver
- **Ch04 (Framebuffer and Planes)** — evdi implements a single primary plane; damage tracking uses the plane's `FB_DAMAGE_CLIPS` property in atomic mode
- **Ch06 (KMS Atomic Modesetting)** — compositors commit frames to evdi via the standard atomic commit path; evdi's connector/CRTC appear in `drmModeGetResources`
- **Ch11 (DMA-BUF)** — evdi must CPU-copy imported DMA-BUFs since the USB chip cannot do PCIe DMA; this is the core performance limitation
- **Ch49 (Multi-GPU)** — DisplayLink monitors appear as a third GPU output; DRI_PRIME is not needed since evdi has no 3D capability — it is display-only
- **Ch139 (DRM Hardware Planes)** — evdi supports only primary planes; hardware overlay planes are not possible since the USB chip has no compositor
