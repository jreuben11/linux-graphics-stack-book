# Chapter 163: VKMS and Virtual Display Drivers for Testing

**Target audiences**: Kernel DRM developers writing tests and CI pipelines for graphics code; compositors and display stack engineers needing a headless virtual display; and CI infrastructure engineers running GPU tests without physical hardware.

---

## Table of Contents

1. [Introduction](#introduction)
   - [1.1 What is Kernel Mode Setting (KMS)?](#11-what-is-kernel-mode-setting-kms)
   - [1.2 What is a Virtual Display Driver?](#12-what-is-a-virtual-display-driver)
   - [1.3 What is a DRM Writeback Connector?](#13-what-is-a-drm-writeback-connector)
2. [VKMS: Virtual Kernel Modesetting Driver](#vkms-virtual-kernel-modesetting-driver)
3. [VKMS Architecture and Source Layout](#vkms-architecture-and-source-layout)
4. [Vblank Simulation and the Composition Worker](#vblank-simulation-and-the-composition-worker)
5. [Software Composition Engine](#software-composition-engine)
6. [Cursor Plane Support](#cursor-plane-support)
7. [EDID Emulation](#edid-emulation)
8. [VKMS Writeback Connector](#vkms-writeback-connector)
9. [CRC Testing Infrastructure](#crc-testing-infrastructure)
10. [KUnit Tests for VKMS Internals](#kunit-tests-for-vkms-internals)
11. [VKMS in Mesa and Vulkan CI](#vkms-in-mesa-and-vulkan-ci)
12. [Configfs Runtime Reconfiguration](#configfs-runtime-reconfiguration)
13. [Comparison with Alternative Headless Backends](#comparison-with-alternative-headless-backends)
14. [Other Virtual Display Drivers](#other-virtual-display-drivers)
15. [DRM Leases and VR Direct Mode](#drm-leases-and-vr-direct-mode)
16. [GPU Testing with LLVMpipe and softpipe](#gpu-testing-with-llvmpipe-and-softpipe)
17. [Container and VM Display Stacks](#container-and-vm-display-stacks)
18. [Roadmap](#roadmap)
19. [Integrations](#integrations)

---

## Introduction

Testing graphics code without a physical display is essential for CI pipelines. Linux provides several virtual display solutions:

- **VKMS** (Virtual Kernel Modesetting) — a software DRM driver in the kernel that implements full KMS without hardware
- **evdi** — userspace-backed virtual display (Ch155, also used by DisplayLink)
- **virtio-gpu** — paravirtualized GPU for VMs
- **DRM writeback connector** — captures composed output to a buffer
- **LLVMpipe/softpipe** — software Mesa rasterisers for GPU-less rendering

Together, these enable running Wayland compositors, IGT tests, and Vulkan/OpenGL applications in a CI environment with no GPU.

VKMS is the most important of these for kernel-level graphics testing. It was first merged in Linux 4.14 and has grown steadily into a reference implementation of the DRM atomic modesetting API, a substrate for compositor CI, and a platform for developing and validating new DRM subsystem features before they reach real hardware drivers.

Sources: [VKMS in kernel](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/vkms) | [IGT GPU tests](https://gitlab.freedesktop.org/drm/igt-gpu-tools) | [VKMS kernel documentation](https://docs.kernel.org/gpu/vkms.html)

### 1.1 What is Kernel Mode Setting (KMS)?

Kernel Mode Setting is the Linux DRM subsystem's mechanism for configuring display hardware from within the kernel. Before KMS, mode-setting—selecting display resolution, refresh rate, and pixel format—was handled by userspace graphics servers, which caused flickering during boot and mode transitions. KMS moves this responsibility into the kernel, giving the DRM layer exclusive ownership of display hardware configuration.

The KMS API centers on four object types: CRTCs (scanout controllers), encoders (signal converters), connectors (physical display outputs), and planes (pixel sources blended together for scanout). A modern "atomic" KMS path bundles property changes across all these objects into a single atomic commit that the driver validates and applies in one operation. The atomic path eliminates the partial-update races inherent in legacy modesetting and is the foundation for tear-free display. VKMS implements the full atomic KMS interface (`DRIVER_MODESET | DRIVER_ATOMIC`), making it a complete software reference of the DRM atomic modesetting API.

Kernel documentation for the DRM/KMS API: [https://www.kernel.org/doc/html/latest/gpu/drm-kms.html](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html)

### 1.2 What is a Virtual Display Driver?

A virtual display driver is a DRM driver that exposes the full display infrastructure—connector, CRTC, encoder, and planes—without connecting to physical scanout hardware. It exists to satisfy the DRM API surface so that compositors, userspace clients, and test suites can operate as if a real display were attached.

Virtual display drivers are essential for CI pipelines, containerized environments, and headless servers where no GPU or physical monitor is present. They allow a Wayland compositor to start, allocate framebuffers, submit atomic commits, and receive vblank events entirely in software—enabling pixel-accurate rendering tests and compositor integration tests in automated environments with no display hardware.

Linux includes several virtual display solutions suited to different parts of the stack. VKMS is a software-only DRM driver that lives entirely in the kernel and simulates every aspect of display hardware, from vblank timing to pixel composition. The evdi driver provides a userspace-backed virtual display used by DisplayLink devices. The virtio-gpu driver implements a paravirtualized GPU interface for virtual machines. The DRM writeback connector extends any CRTC with an output-capture path. This chapter covers all of these, with VKMS as the primary focus.

### 1.3 What is a DRM Writeback Connector?

A DRM writeback connector is a special connector type (`DRM_MODE_CONNECTOR_WRITEBACK`) that routes the pixel output of a CRTC into a userspace-provided framebuffer instead of driving a physical display. It is defined in the DRM core (`drivers/gpu/drm/drm_writeback.c`) and exposed to userspace through the standard KMS property interface using a `WRITEBACK_FB_ID` property that the client sets before an atomic commit.

The writeback connector enables pixel-exact verification: a test submits a frame via atomic commit, reads the composed output back through the writeback path, and compares it against a reference image byte for byte. VKMS implements a writeback connector alongside its virtual display connector, giving tests two complementary verification paths—per-frame CRC values via the DRM debugfs CRC interface for fast pass/fail assertions, and full pixel readback via writeback for detailed comparison or golden-image diffing. The writeback path also drives the DRM content protection test suite and is used by compositors for screenshot capture and frame streaming pipelines.

---

## VKMS: Virtual Kernel Modesetting Driver

### What VKMS Provides

VKMS (`drivers/gpu/drm/vkms/`) is a software-only DRM driver that:

- Creates a virtual `DRM_MODE_CONNECTOR_VIRTUAL` connector (and, with the writeback feature enabled, a `DRM_MODE_CONNECTOR_WRITEBACK` connector)
- Implements a full atomic modesetting stack (CRTC, encoder, connector, planes)
- Simulates vertical blank interrupts via a high-resolution kernel timer (`hrtimer`)
- Provides a CPU-only pixel pipeline: composition of primary, overlay, and cursor planes into a framebuffer
- Exposes per-frame CRC values via the DRM debugfs CRC interface for pixel-exact test assertions
- Optionally captures the composed output to a userspace-provided framebuffer via the writeback connector

```bash
# Load VKMS:
sudo modprobe vkms
# Check:
ls /dev/dri/  # new card* appears
dmesg | grep -i vkms
# Output: vkms: loaded

# Run Weston on VKMS:
WLR_BACKENDS=drm WLR_DRM_DEVICES=/dev/dri/card0 sway &

# Or use weston's own headless backend (not VKMS):
weston --backend=headless-backend.so --width=1920 --height=1080 &
```

### Module Parameters

```bash
# VKMS module options (defaults in parentheses):
modprobe vkms enable_cursor=1          # hardware cursor plane (default: 1)
modprobe vkms enable_overlay=1         # overlay plane (default: 0)
modprobe vkms enable_writeback=1       # writeback connector (default: 1)
modprobe vkms enable_plane_pipeline=1  # plane pipeline (default: 0)
modprobe vkms create_default_dev=1     # create the default device (default: 1)
```

Setting `create_default_dev=0` allows VKMS to be loaded purely as a configfs-managed provider without creating any DRM device immediately; additional instances are then created by writing to configfs.

---

## VKMS Architecture and Source Layout

### Source Structure

As of Linux 6.12 and later, the VKMS source tree is:

```
drivers/gpu/drm/vkms/
├── vkms_drv.c          # Module init, drm_driver, default device creation
├── vkms_drv.h          # Shared types: vkms_device, vkms_output, vkms_plane_state
├── vkms_crtc.c         # CRTC: vblank simulation, atomic check/flush
├── vkms_plane.c        # Primary, overlay, cursor plane creation and atomic update
├── vkms_connector.c    # Virtual connector: modes, EDID, plug/unplug
├── vkms_connector.h
├── vkms_output.c       # Output initialisation (crtc + encoder + connector linkage)
├── vkms_writeback.c    # Writeback connector: drm_writeback_connector setup
├── vkms_composer.c     # Software composition: blend loop, CRC, writeback output
├── vkms_composer.h
├── vkms_colorop.c      # DRM color pipeline operators (1D LUT, CTM, 3D LUT)
├── vkms_formats.c      # Per-format pixel read/write callbacks
├── vkms_formats.h
├── vkms_config.c       # Configurable output description (planes, CRTCs, connectors)
├── vkms_config.h
├── vkms_configfs.c     # configfs interface for runtime reconfiguration
├── vkms_configfs.h
├── vkms_luts.c         # Color LUT management
├── vkms_luts.h
├── tests/              # KUnit tests for VKMS internals
├── Kconfig
└── Makefile
```

### Driver Feature Flags

The `drm_driver` structure is declared with:

```c
/* drivers/gpu/drm/vkms/vkms_drv.c */
static const struct drm_driver vkms_driver = {
    .driver_features = DRIVER_MODESET | DRIVER_ATOMIC | DRIVER_GEM,
    .fops            = &vkms_driver_fops,
    .name            = "vkms",
    .desc            = "Virtual Kernel Mode Setting",
    .major = 1, .minor = 0,
    DRM_GEM_SHMEM_DRIVER_OPS,
    DRM_FBDEV_SHMEM_DRIVER_OPS,
};
```

`DRIVER_MODESET | DRIVER_ATOMIC` ensures VKMS registers the full atomic KMS path, and `DRIVER_GEM` with `DRM_GEM_SHMEM_DRIVER_OPS` gives it shared-memory GEM backing so framebuffers can be allocated from system RAM using `drm_gem_shmem_helper`.

[Source: torvalds/linux drivers/gpu/drm/vkms/vkms_drv.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/vkms/vkms_drv.c)

---

## Vblank Simulation and the Composition Worker

### hrtimer-based Vblank

VKMS has no real scanout hardware. Instead, a high-resolution timer fires at the configured refresh period to simulate vertical blanking. The core is `vkms_vblank_simulate()`:

```c
/* drivers/gpu/drm/vkms/vkms_crtc.c (simplified) */
static bool vkms_crtc_handle_vblank_timeout(struct drm_crtc *crtc)
{
    struct vkms_output *out = drm_crtc_to_vkms_output(crtc);

    spin_lock(&out->composer_lock);
    /* record the frame range this vblank covers */
    crtc_state->frame_end = drm_crtc_vblank_count(crtc);
    if (!crtc_state->frame_start)
        crtc_state->frame_start = crtc_state->frame_end;
    crtc_state->crc_pending = true;
    spin_unlock(&out->composer_lock);

    /* wake up the ordered workqueue */
    queue_work(out->composer_wq, &crtc_state->composer_work);

    return true;
}
```

The hrtimer period is derived from the mode's `vrefresh` field. For a 60 Hz mode, the timer fires every ~16.67 ms. Each fire increments the vblank counter via `drm_crtc_handle_vblank()` and enqueues the composition work.

### Ordered Composition Workqueue

The composition work runs in an ordered single-threaded workqueue (`alloc_ordered_workqueue`). This ensures frames are composed in sequence even if the timer fires faster than the worker can drain:

```c
/* vkms_composer_worker — scheduled by every hrtimer vblank */
void vkms_composer_worker(struct work_struct *work)
{
    struct vkms_crtc_state *crtc_state =
        container_of(work, struct vkms_crtc_state, composer_work);
    struct drm_crtc *crtc = crtc_state->base.crtc;
    struct vkms_output *out = drm_crtc_to_vkms_output(crtc);
    struct vkms_writeback_job *active_wb = NULL;
    u64 frame_start, frame_end;
    bool crc_pending, wb_pending;
    u32 crc32 = 0;
    int ret;

    spin_lock_irq(&out->composer_lock);
    frame_start    = crtc_state->frame_start;
    frame_end      = crtc_state->frame_end;
    crc_pending    = crtc_state->crc_pending;
    wb_pending     = crtc_state->wb_pending;
    crtc_state->crc_pending = false;
    crtc_state->frame_start = 0;
    spin_unlock_irq(&out->composer_lock);

    if (!crc_pending)
        return;

    ret = compose_active_planes(wb_pending ? active_wb : NULL,
                                crtc_state, &crc32);
    if (ret)
        return;

    /* deliver one CRC entry per frame in the window */
    while (frame_start <= frame_end)
        drm_crtc_add_crc_entry(crtc, true, frame_start++, &crc32);

    /* signal writeback completion */
    if (wb_pending)
        drm_writeback_signal_completion(&out->wb_connector, 0);
}
```

The `while (frame_start <= frame_end)` loop handles the case where the worker catches up on multiple missed frames. [Source: torvalds/linux vkms_composer.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/vkms/vkms_composer.c)

---

## Software Composition Engine

### Line-by-Line Compositing

VKMS composes planes in software on the CPU. The algorithm iterates over every scanline in the output framebuffer, building a composited line from all active planes. The inner representation is `pixel_argb_u16` — four 16-bit channels that avoid precision loss during accumulation:

```c
/* vkms_composer.h */
struct pixel_argb_u16 {
    u16 a, r, g, b;   /* 16-bit fixed-point, 0x0000–0xffff */
};
/* static_assert(sizeof(struct pixel_argb_u16) == 8); */

struct line_buffer {
    struct pixel_argb_u16 *pixels;
    size_t n_pixels;
};
```

### Premultiplied-Alpha Blend Loop

The blend step for a single plane reads source pixels (converting from the plane's native format into `pixel_argb_u16` via a per-format callback), then composites them over the running output using the standard premultiplied-alpha formula:

```c
/* drivers/gpu/drm/vkms/vkms_composer.c */
static u16 pre_mul_blend_channel(u16 src, u16 dst, u16 alpha)
{
    /* out = src + dst * (1 - alpha), with rounding */
    u32 new_color = (u32)src * 0xffff + (u32)dst * (0xffff - alpha);
    return DIV_ROUND_CLOSEST_ULL(new_color, 0xffff);
}

static void pre_mul_alpha_blend(const struct line_buffer *stage_buffer,
                                struct line_buffer *output_buffer,
                                int x_start, int pixel_count)
{
    struct pixel_argb_u16 *out = output_buffer->pixels + x_start;
    const struct pixel_argb_u16 *in  = stage_buffer->pixels;

    for (int x = 0; x < pixel_count; x++) {
        out[x].a = 0xffff;   /* output is always fully opaque */
        out[x].r = pre_mul_blend_channel(in[x].r, out[x].r, in[x].a);
        out[x].g = pre_mul_blend_channel(in[x].g, out[x].g, in[x].a);
        out[x].b = pre_mul_blend_channel(in[x].b, out[x].b, in[x].a);
    }
}
```

The choice of 16-bit intermediate precision means that compositing eight overlapping planes does not accumulate the rounding error that 8-bit intermediate math would produce, keeping VKMS's output suitable as a reference for CRC comparisons.

### Plane Z-Ordering and Overlay Planes

Planes are sorted by their `zpos` DRM property before compositing begins. For a typical configuration with a primary plane at `zpos=0`, an overlay at `zpos=1`, and a cursor at `zpos=MAX_INT`, the composition loop visits them in ascending order:

```c
/* compose_active_planes — per-scanline loop */
for (int y = 0; y < output_height; y++) {
    /* clear output scanline to opaque black */
    memset(output_buffer.pixels, 0, line_size);

    list_for_each_entry(plane_state, &crtc_state->active_planes, head) {
        if (!plane_visible_on_row(plane_state, y))
            continue;
        /* read a row of source pixels into stage_buffer */
        plane_state->pixel_read_line(plane_state, y, &stage_buffer);
        /* blend stage onto output */
        pre_mul_alpha_blend(&stage_buffer, &output_buffer,
                            plane_state->frame_info.dst.x1,
                            plane_state->frame_info.dst.x2 -
                            plane_state->frame_info.dst.x1);
    }

    /* accumulate CRC32 over this scanline */
    *crc32 = crc32_le(*crc32, (void *)output_buffer.pixels, line_size);

    /* if writeback: write scanline to the writeback framebuffer */
    if (active_wb)
        active_wb->pixel_write_line(active_wb, y, &output_buffer);
}
```

The `pixel_read_line` and `pixel_write_line` callbacks are resolved at plane-update time from the `vkms_formats.c` dispatch table, one callback per pixel format. This avoids a per-pixel branch on the inner loop.

### Pixel Format Dispatch in vkms_formats.c

`vkms_formats.c` is the format-specific layer that decouples the composition loop from pixel encoding details. It exposes two function-pointer types:

```c
/* drivers/gpu/drm/vkms/vkms_formats.h */

/**
 * pixel_read_line_t - read one scanline from a plane framebuffer
 * @plane_state: the VKMS plane state carrying the GEM bo and frame_info
 * @y:           output scanline index (in destination coordinates)
 * @dst:         line_buffer to fill with pixel_argb_u16 values
 */
typedef void (*pixel_read_line_t)(struct vkms_plane_state *plane_state,
                                  int y, struct line_buffer *dst);

/**
 * pixel_write_line_t - write one composited scanline to a writeback buffer
 * @wb_job:  the active writeback job carrying the destination GEM bo
 * @y:       output scanline index
 * @src:     line_buffer holding the composited pixel_argb_u16 row
 */
typedef void (*pixel_write_line_t)(struct vkms_writeback_job *wb_job,
                                   int y, const struct line_buffer *src);
```

These function pointers are resolved once — at plane-update time in `vkms_plane_atomic_update()` and at writeback-prepare time in `vkms_wb_prepare_job()` — via `get_pixel_read_line_function()` and `get_pixel_write_line_function()`. The inner composition loop then calls them without any format-dependent branching:

```c
/* vkms_formats.c — dispatch table entry (one per supported format) */
static void read_line_xrgb8888(struct vkms_plane_state *plane_state,
                               int y, struct line_buffer *dst)
{
    const u8 *src_pixels = plane_fb_addr_at_row(plane_state, y);
    struct pixel_argb_u16 *out = dst->pixels;
    int n = dst->n_pixels;

    for (int i = 0; i < n; i++) {
        const u32 px = le32_to_cpu(((const u32 *)src_pixels)[i]);
        out[i].a = 0xffff;
        out[i].r = (u16)(((px >> 16) & 0xff) * 0x0101);
        out[i].g = (u16)(((px >>  8) & 0xff) * 0x0101);
        out[i].b = (u16)(((px >>  0) & 0xff) * 0x0101);
    }
}
```

The `* 0x0101` idiom replicates 8-bit channel values into 16-bit range (e.g., `0xff → 0xffff`) without a division. Formats with fewer bits (RGB565) use appropriate shift/mask sequences; ARGB16161616 reads 64-bit words directly. For multi-planar formats (NV12, P010), the read function handles both the Y-plane and UV-plane stride calculations internally.

The line-by-line approach was reintroduced by Louis Chauvet in 2024 (reverted from an earlier per-plane-at-once scheme) because it achieves 5–10% better cache utilisation: each scanline fits in L1 cache, so the output line buffer stays hot across the full plane stack, whereas the earlier scheme would evict the output buffer between planes.

[Source: LWN — drm/vkms: Reimplement line-per-line pixel conversion](https://lwn.net/Articles/998618/) | [PATCH series — dri-devel](https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg493762.html)

### Color Operations (colorop) and the DRM Color Pipeline API

`vkms_colorop.c` implements the DRM color pipeline operators that can be chained on a CRTC or plane. VKMS is the kernel reference driver for the DRM Color Pipeline API, a major patchset (v5 at time of writing, 44 patches) led by Harry Wentland at AMD and upstream-merged to `drm-misc-next`.

The supported colorop types mirror what is available on AMD DCN 3+ hardware, making VKMS a faithful software model:

| Colorop type | Kernel enum | Effect |
|---|---|---|
| 1D EOTF curve | `DRM_COLOROP_1D_CURVE` | Electro-Optical Transfer Function (e.g., sRGB, PQ) |
| 3×4 CTM | `DRM_COLOROP_CTM_3X4` | Color transformation matrix (gamut conversion) |
| HDR multiplier | `DRM_COLOROP_MULTIPLIER` | Peak luminance scaling |
| 1D inverse EOTF | `DRM_COLOROP_1D_CURVE` | OETF / inverse EOTF (linear → display) |
| 3D LUT | `DRM_COLOROP_3D_LUT` | 17³ tetrahedral interpolated LUT |

Supported 1D curve subtypes (set in `DRM_COLOROP_1D_CURVE_TYPE` property) include:
- `DRM_COLOROP_1D_CURVE_SRGB_EOTF` / `DRM_COLOROP_1D_CURVE_SRGB_INV_EOTF`
- `DRM_COLOROP_1D_CURVE_PQ_EOTF` / `DRM_COLOROP_1D_CURVE_PQ_INV_EOTF` (scaled to [0, 125 cd/m²])
- `DRM_COLOROP_1D_CURVE_BT2020_INV_OETF` and its inverse

The VKMS implementation in `vkms_colorop.c` applies these via `apply_colorop()` after the plane-blend loop and before writing to the writeback buffer (or computing the CRC). This placement mirrors the real hardware pipeline, where CRTC color operations occur post-composition. Because VKMS is software, it applies all operations with full floating-point precision, producing a deterministic reference output that real drivers can compare against during conformance testing.

```c
/* vkms_colorop.c — apply one colorop node to a scanline (simplified) */
static void apply_colorop(const struct drm_colorop *colorop,
                          struct pixel_argb_u16 *pixels, size_t n)
{
    switch (colorop->type) {
    case DRM_COLOROP_1D_CURVE:
        apply_1d_curve(colorop, pixels, n);
        break;
    case DRM_COLOROP_CTM_3X4:
        apply_ctm(colorop->data.ctm_3x4, pixels, n);
        break;
    case DRM_COLOROP_MULTIPLIER:
        apply_multiplier(colorop->data.mult, pixels, n);
        break;
    case DRM_COLOROP_3D_LUT:
        apply_3d_lut(colorop, pixels, n);
        break;
    }
}
```

The color pipeline operators are exposed through DRM properties created in `vkms_colorop.c`. Their IGT validation uses a pixel-by-pixel comparison with a configurable delta tolerance, because floating-point rounding in software makes exact CRC matching impractical for non-linear curves. The patchset was merged to drm-misc-next with VKMS as the sole in-tree reference implementation, ahead of the amdgpu backend. [Source: LWN Color Pipeline API](https://lwn.net/Articles/1033931/) | [PATCH v5 00/44](https://lore.freedesktop.org/wayland-devel/20240819205714.316380-12-harry.wentland@amd.com/T/)

[Source: torvalds/linux vkms_composer.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/vkms/vkms_composer.c)

---

## Cursor Plane Support

### Creating Planes

`vkms_plane_init()` creates a plane of the requested type using `drmm_universal_plane_alloc()`:

```c
/* drivers/gpu/drm/vkms/vkms_plane.c */
struct vkms_plane *vkms_plane_init(struct vkms_device *vkmsdev,
                                   struct vkms_config_plane *plane_cfg)
{
    struct drm_device *dev = &vkmsdev->drm;
    const u32 *formats;
    int nformats;

    /* select format list by plane type */
    switch (plane_cfg->type) {
    case DRM_PLANE_TYPE_CURSOR:
        formats  = vkms_cursor_formats;
        nformats = ARRAY_SIZE(vkms_cursor_formats);
        break;
    default:
        formats  = vkms_formats;
        nformats = ARRAY_SIZE(vkms_formats);
    }

    plane = drmm_universal_plane_alloc(dev, struct vkms_plane, base,
                   possible_crtcs, &vkms_plane_funcs,
                   formats, nformats, NULL,
                   plane_cfg->type, NULL);
    drm_plane_helper_add(&plane->base, &vkms_plane_helper_funcs);
    drm_plane_create_rotation_property(&plane->base, DRM_MODE_ROTATE_0,
                                       DRM_MODE_ROTATE_MASK |
                                       DRM_MODE_REFLECT_MASK);
    return plane;
}
```

The cursor plane supports `ARGB8888` and `XRGB8888` formats by default; it accepts asynchronous updates via the legacy `DRM_IOCTL_MODE_CURSOR` ioctl as well as the atomic path.

### Atomic Cursor Update

The cursor plane's `atomic_update` callback:

```c
static void vkms_plane_atomic_update(struct drm_plane *plane,
                                     struct drm_atomic_commit *state)
{
    struct vkms_plane_state *vkms_state = to_vkms_plane_state(plane->state);
    struct drm_framebuffer *fb = plane->state->fb;

    if (!plane->state->visible || !fb) {
        vkms_state->frame_info = NULL;
        return;
    }

    /* populate frame_info: src/dst rects, rotation, format */
    vkms_state->frame_info.fb           = fb;
    vkms_state->frame_info.src          = plane->state->src;
    vkms_state->frame_info.dst          = plane->state->dst;
    vkms_state->frame_info.rotation     = plane->state->rotation;
    /* resolve format-specific pixel_read_line callback */
    vkms_state->pixel_read_line =
        get_pixel_conversion_function(fb->format->format);
}
```

In the composition worker, the cursor plane appears as just another entry in the sorted `active_planes` list — its blend is identical to overlay blend, with the `zpos` property set to the maximum value (`INT_MAX`) by default. This ensures the cursor renders on top of all other content.

[Source: torvalds/linux vkms_plane.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/vkms/vkms_plane.c)

---

## EDID Emulation

### Why EDID Matters in Testing

Compositors like Weston and sway call `drm_connector_get_modes()` on startup and pick a display mode based on the EDID blob. Without a well-formed EDID the connector may report no modes at all, causing the compositor to fail to set up its output. VKMS must therefore provide a synthetic EDID so that a headless instance behaves exactly like a connected monitor.

Beyond mode selection, EDID encodes display capabilities that trigger specific code paths in compositors and color managers:
- **HDR static metadata** (CTA-861 HDR/WCG metadata extension, EDID byte 7 flags): a compositor that sees `HDR_OUTPUT_METADATA` in the EDID will enable its HDR color pipeline; without it, the path is never exercised in CI.
- **Color primaries and white point**: the colorimetry data block reports the display's native gamut (DCI-P3, BT.2020, sRGB), which compositors use to choose the ICC profile for color management.
- **Supported VRR/FreeSync range** (EDID Monitor Range Limits): compositors check this to decide whether to enable variable refresh rate output.

### Default Synthetic EDID

The virtual connector in `vkms_connector.c` uses `drm_edid_build_fake()` (or a static EDID constructed with `drm_mode_duplicate`) to expose a small set of standard modes. The default configuration exposes:

- 1920×1080 at 60 Hz (preferred mode)
- 1280×720 at 60 Hz
- 1024×768 at 60 Hz

```c
/* drivers/gpu/drm/vkms/vkms_connector.c (conceptual) */
static int vkms_conn_get_modes(struct drm_connector *connector)
{
    struct drm_display_mode *mode;
    int count = 0;

    /* preferred mode */
    mode = drm_cvt_mode(connector->dev, 1920, 1080, 60, false, false, false);
    mode->type |= DRM_MODE_TYPE_PREFERRED;
    drm_mode_probed_add(connector, mode);
    count++;

    /* additional modes */
    mode = drm_cvt_mode(connector->dev, 1280, 720, 60, false, false, false);
    drm_mode_probed_add(connector, mode);
    count++;

    return count;
}
```

### Custom EDID via Configfs

The configfs interface (see section below) allows injecting an arbitrary raw EDID binary into the connector at runtime. This is important for testing compositor code paths that depend on EDID-reported features such as:

- HDR static metadata (Luminance values, EOTF flags in the HDR/WCG metadata extension)
- Audio or video format capabilities
- Non-standard resolutions or refresh rates (e.g., 2560×1440 at 165 Hz)
- Multiple-monitor topology (using the configfs multi-connector support)

EDID can be changed while the connector is live; a hot-plug detect (HPD) event must be triggered afterward for the compositor to re-query modes:

```bash
# inject a custom EDID via configfs (see Configfs section):
cp my-monitor.edid /config/vkms/my-vkms/connectors/conn0/edid
# trigger HPD:
echo connected > /config/vkms/my-vkms/connectors/conn0/status
```

[Source: LKML Louis Chauvet VKMS configfs EDID patch 2025](https://lkml.org/lkml/2025/10/18/67)

---

## VKMS Writeback Connector

### DRM Writeback Connector Overview

A DRM writeback connector is a special connector type (`DRM_MODE_CONNECTOR_WRITEBACK`) that, instead of driving a physical display, writes the composed CRTC output into a userspace-provided framebuffer. The kernel writeback infrastructure lives in `drivers/gpu/drm/drm_writeback.c` and is used by display IP blocks (Arm Mali, Rockchip VOP, MediaTek MDP) that have hardware writeback engines.

VKMS implements writeback in software as a convenience: the same composed scanlines that go to the CRC computation also feed the writeback output buffer. This allows test harnesses to capture pixel-exact compositor output without a physical display.

The key structures are:

```c
/* include/drm/drm_writeback.h */
struct drm_writeback_connector {
    struct drm_connector base;
    struct drm_encoder   encoder;     /* internal encoder */
    struct list_head     job_queue;   /* pending writeback jobs */
    spinlock_t           job_lock;
    u64                  fence_context;
    spinlock_t           fence_lock;
    char                 timeline_name[32];
    struct drm_property_blob *pixel_formats_blob_ptr;
};

struct drm_writeback_job {
    struct drm_writeback_connector *connector;
    struct drm_framebuffer         *fb;       /* destination buffer */
    struct dma_fence               *out_fence;/* completion fence */
    struct kref                     refcount;
    struct list_entry               list_entry;
    struct work_struct              cleanup_work;
};
```

[Source: torvalds/linux drm_writeback.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_writeback.c)

### VKMS Writeback Initialisation

`vkms_writeback.c` registers the writeback connector using `drm_writeback_connector_init()`:

```c
/* drivers/gpu/drm/vkms/vkms_writeback.c */
static const u32 vkms_wb_formats[] = {
    DRM_FORMAT_ARGB8888,
    DRM_FORMAT_XRGB8888,
    DRM_FORMAT_ABGR8888,
    DRM_FORMAT_XRGB16161616,
    DRM_FORMAT_ARGB16161616,
    DRM_FORMAT_RGB565,
};

static const struct drm_connector_helper_funcs vkms_wb_conn_helper_funcs = {
    .get_modes           = vkms_wb_get_modes,
    .prepare_writeback_job = vkms_wb_prepare_job,
    .cleanup_writeback_job = vkms_wb_cleanup_job,
    .atomic_commit       = vkms_wb_atomic_commit,
};

int vkms_enable_writeback_connector(struct vkms_device *vkmsdev,
                                    struct vkms_output *vkms_output)
{
    struct drm_writeback_connector *wb = &vkms_output->wb_connector;

    vkms_output->wb_encoder.possible_crtcs =
        drm_crtc_mask(&vkms_output->crtc);

    return drm_writeback_connector_init_with_encoder(
               &vkmsdev->drm, wb, &vkms_output->wb_encoder,
               &vkms_wb_connector_funcs,
               vkms_wb_formats, ARRAY_SIZE(vkms_wb_formats));
}
```

### Writeback Job Lifecycle

When userspace submits an atomic commit with a framebuffer attached to the writeback connector property `WRITEBACK_FB_ID`, the kernel calls `prepare_writeback_job` to set up the pixel output callback:

```c
static int vkms_wb_prepare_job(struct drm_writeback_connector *wb_connector,
                               struct drm_writeback_job *job)
{
    struct vkms_writeback_job *vkms_job;

    vkms_job = kzalloc(sizeof(*vkms_job), GFP_KERNEL);
    if (!vkms_job)
        return -ENOMEM;

    /* map the destination framebuffer into kernel address space */
    ret = drm_gem_fb_vmap(job->fb, vkms_job->data.map,
                          vkms_job->data.data);
    if (ret) {
        kfree(vkms_job);
        return ret;
    }

    /* resolve pixel-write callback for the destination format */
    vkms_job->pixel_write_line =
        get_pixel_write_function(job->fb->format->format);

    job->priv = vkms_job;
    return 0;
}
```

The `cleanup_writeback_job` callback unmaps the framebuffer and frees the private data. The `atomic_commit` callback sets `wb_pending = true` in the CRTC state so the composition worker knows it must write output to the buffer.

### Reading Back Pixels in a Test Harness

From userspace, a test program uses the atomic API:

```c
/* test harness: capture one frame via writeback */
uint32_t writeback_fb_id = create_framebuffer(drm_fd, 1920, 1080,
                                              DRM_FORMAT_XRGB8888);

drmModeAtomicReqPtr req = drmModeAtomicAlloc();

/* attach writeback framebuffer */
drmModeAtomicAddProperty(req, writeback_connector_id,
    writeback_fb_id_prop, writeback_fb_id);

/* non-blocking commit: returns before composition is done */
drmModeAtomicCommit(drm_fd, req, DRM_MODE_ATOMIC_NONBLOCK, NULL);

/* read the out-fence from the connector state */
uint64_t fence_fd_val;
drmModeGetPropertyValue(drm_fd, writeback_connector_id,
    "WRITEBACK_OUT_FENCE_PTR", &fence_fd_val);
int fence_fd = (int)(intptr_t)fence_fd_val;

/* wait for composition to complete */
sync_wait(fence_fd, -1);   /* or poll()/dma_buf_sync_file */
close(fence_fd);

/* writeback_fb now contains the pixel-exact composed frame */
compare_pixels(writeback_fb_mmap, reference_image);
```

The out-fence (`WRITEBACK_OUT_FENCE_PTR`) is a DMA-fence backed sync file; userspace passes in a pointer to a 64-bit integer which the kernel fills with a file descriptor. The composition worker calls `drm_writeback_signal_completion()` when it finishes writing the frame, which signals the fence.

### Out-Fence Mechanics and Explicit Synchronization

`WRITEBACK_OUT_FENCE_PTR` integrates writeback into the standard Linux explicit-synchronization framework. The value that userspace sets for this property is the address (cast to `u64`) of a `int32_t` variable — the kernel writes a `sync_file` file descriptor into that address before the `drmModeAtomicCommit()` call returns. The sequence at the kernel level is:

1. `drm_writeback_connector_init_with_encoder()` allocates a DMA-fence timeline for the connector during initialization.
2. On each atomic commit that carries a `WRITEBACK_FB_ID`, `drm_writeback_queue_job()` creates a `drm_writeback_job`, attaches the framebuffer, and allocates a new fence on the connector's timeline.
3. The kernel exports that fence as a `sync_file` and fills in the userspace-provided pointer before returning from the ioctl.
4. In `vkms_composer_worker()`, after the last composited scanline is written to the destination buffer, `drm_writeback_signal_completion(&out->wb_connector, 0)` advances the timeline and signals all waiters.

This means the out-fence is a reliable guarantee: any process that calls `poll()` or `sync_wait()` on the file descriptor will unblock only after the CPU has finished writing the last pixel of the frame. A test harness can therefore safely mmap the destination buffer and compare pixels immediately after the fence signals, without any additional synchronization primitive.

An important constraint: requesting `WRITEBACK_OUT_FENCE_PTR` without simultaneously setting `WRITEBACK_FB_ID` on the same commit is rejected by the kernel with `EINVAL`. The two properties are logically inseparable — there is no meaningful out-fence if there is no destination buffer to write into.

[Source: VKMS writeback kernel docs](https://docs.kernel.org/gpu/vkms.html) | [drm_writeback.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_writeback.c) | [DRM writeback — LWN introduction](https://lwn.net/Articles/704647/)

---

## CRC Testing Infrastructure

### DRM CRTC CRC API

The DRM subsystem exposes per-frame CRC values from a CRTC through a debugfs file. After loading the driver and setting a mode, enabling CRC capture:

```bash
# Enable CRC capture on CRTC 0:
echo auto > /sys/kernel/debug/dri/0/crtc-0/crc/control

# Read per-frame CRC values (one line per vblank):
cat /sys/kernel/debug/dri/0/crtc-0/crc/data
# Output format: frame_number  source  crc0  [crc1 ...]
# e.g.:          00000001      auto    0xab12cd34
```

Real hardware drivers provide CRC from scanout hardware (Intel pipe CRC, AMD DCE CRC); VKMS computes it in software over the CPU-composited output.

### VKMS CRC Computation

VKMS computes CRC32 over every composited scanline. Within `compose_active_planes()` the inner loop accumulates the CRC over the 16-bit `pixel_argb_u16` output buffer:

```c
/* drivers/gpu/drm/vkms/vkms_composer.c */
*crc32 = crc32_le(*crc32, (void *)output_buffer->pixels,
                  output_buffer->n_pixels * sizeof(struct pixel_argb_u16));
```

This produces a 32-bit CRC of the full frame (all scanlines concatenated), which is then reported via `drm_crtc_add_crc_entry()`. Because VKMS is deterministic (same planes + same pixel data = same CRC on every run), these values are suitable as golden references for regression tests.

### IGT Pipe CRC Infrastructure

IGT provides a high-level CRC capture API that wraps the debugfs interface:

```c
/* IGT test: typical CRC workflow */
igt_pipe_crc_t *pipe_crc;
igt_crc_t crc, reference_crc;

pipe_crc = igt_pipe_crc_new(drm_fd, pipe_idx,
                             INTEL_PIPE_CRC_SOURCE_AUTO);
igt_pipe_crc_start(pipe_crc);

/* render the test pattern */
draw_test_pattern(primary_fb);
drmModePageFlip(drm_fd, crtc_id, fb_id, DRM_MODE_PAGE_FLIP_EVENT, NULL);
wait_for_vblank(drm_fd);

/* capture CRC */
igt_pipe_crc_get_current(drm_fd, pipe_crc, &crc);
igt_pipe_crc_stop(pipe_crc);
igt_pipe_crc_free(pipe_crc);

igt_assert_crc_equal(&crc, &reference_crc);
```

The key function `igt_pipe_crc_get_current()` reads from `crtc-N/crc/data`, parses the CRC words, and returns them as `igt_crc_t`. `igt_assert_crc_equal()` compares all CRC words; a mismatch fails the test with a diagnostic message.

### kms_cursor_crc Test

The `kms_cursor_crc` IGT test validates cursor plane rendering by comparing two frames:

1. **Cursor plane enabled**: the cursor framebuffer is placed at a given (x, y) position; a CRC is captured.
2. **Cursor plane disabled, content blended on primary**: the same cursor image is drawn into the primary framebuffer using Cairo at the same position; a second CRC is captured.

If both CRCs match, the kernel's cursor plane blend is pixel-exact. This test covers positions including fully on-screen, partially off the left/top/right/bottom edges, and fully off-screen. VKMS's cursor blend implementation matches the reference because both paths use the same `pre_mul_alpha_blend()` function.

```bash
# Run kms_cursor_crc on VKMS:
sudo IGT_FORCE_DRIVER=vkms \
    ./build/tests/kms_cursor_crc \
    --run-subtest cursor-64x64-onscreen
```

The `IGT_FORCE_DRIVER=vkms` environment variable tells IGT's device-selection layer to open the VKMS DRM device rather than any real GPU present in the system.

### kms_plane_multiple Test

`kms_plane_multiple` stresses the composition engine with multiple overlapping planes at varying z-positions and pixel formats. On VKMS it exercises the software blending path with combinations like:

- Primary XRGB8888 + overlay ARGB8888 at zpos=1 (transparent overlay)
- Primary NV12 (YUV) + overlay ARGB8888 (compositing YUV → RGB conversion)

```bash
sudo IGT_FORCE_DRIVER=vkms \
    ./build/tests/kms_plane_multiple \
    --run-subtest pipe-A-tiling-none
```

### The DRM CRC Debugfs Callbacks

Beyond the userspace interface, the kernel side requires two CRTC helper callbacks before a driver can expose CRC values through debugfs:

```c
/* A driver that supports CRTC CRC must implement: */
static const struct drm_crtc_funcs my_crtc_funcs = {
    /* ... */
    .verify_crc_source = vkms_verify_crc_source,
    .set_crc_source    = vkms_set_crc_source,
    .get_crc_sources   = vkms_get_crc_sources,
};
```

`verify_crc_source()` is called when userspace writes to `crc/control`; it validates the requested source name and sets `values_cnt` to the number of 32-bit CRC words this driver produces per frame (VKMS uses 1; Intel pipe CRC produces 5). `set_crc_source()` starts or stops CRC generation by setting a flag that the composition worker checks:

```c
/* drivers/gpu/drm/vkms/vkms_crtc.c */
static int vkms_set_crc_source(struct drm_crtc *crtc, const char *src_name)
{
    struct vkms_output *out = drm_crtc_to_vkms_output(crtc);
    bool enabled = src_name && strcmp(src_name, "auto") == 0;

    spin_lock_irq(&out->composer_lock);
    out->crc_enabled = enabled;
    vkms_composer_toggle(out, enabled);
    spin_unlock_irq(&out->composer_lock);
    return 0;
}
```

`vkms_composer_toggle()` increments or decrements a `composer_enabled` reference count shared with the writeback path; when it reaches zero the hrtimer is stopped to save CPU cycles.

### kms_writeback IGT Test

The `kms_writeback` IGT test exercises the writeback connector directly. It submits an atomic commit with `WRITEBACK_FB_ID` set and then waits on the out-fence before comparing the captured framebuffer pixels against a reference pattern drawn with Cairo:

```bash
sudo IGT_FORCE_DRIVER=vkms \
    ./build/tests/kms_writeback \
    --run-subtest writeback-check-output
```

The test library call sequence mirrors what a real compositor would do:

1. Allocate a destination GEM buffer and import it as a framebuffer (`drmModeAddFB2`).
2. Get the writeback connector's `WRITEBACK_FB_ID` and `WRITEBACK_OUT_FENCE_PTR` property IDs.
3. Build an atomic request: set `WRITEBACK_FB_ID` to the destination framebuffer, set `WRITEBACK_OUT_FENCE_PTR` to the address of a `int32_t` where the kernel will write the sync-file descriptor.
4. Call `drmModeAtomicCommit()` with `DRM_MODE_ATOMIC_NONBLOCK`. The kernel fills in the fence fd before returning.
5. Call `sync_wait(fence_fd, timeout_ms)` to block until the composition worker has finished writing the frame.
6. Map the destination buffer with `mmap()` and compare pixels.

An important nuance: `WRITEBACK_FB_ID` is a write-only property that is *not* preserved across commits. Userspace must re-set it every commit in which it wants a writeback frame captured. If it is omitted from a commit, no writeback occurs for that frame even if the writeback connector is active.

[Source: IGT reference manual](https://drm.pages.freedesktop.org/igt-gpu-tools/igt-kms-tests.html) | [kms_cursor_crc overview](https://melissawen.github.io/blog/2020/06/03/overview_kms_cursor_crc)

---

## KUnit Tests for VKMS Internals

VKMS includes an in-tree KUnit test suite at `drivers/gpu/drm/vkms/tests/`. These are kernel unit tests that run without loading the full DRM stack — they exercise isolated C functions directly, catching regressions in numerical routines that would otherwise only surface as wrong pixels in an IGT test.

### What the Tests Cover

The initial focus of the KUnit suite (as of Linux 6.13 and the Color Pipeline API patchset) is the LUT and color math:

- **`get_lut_index` tests**: verify that the function mapping a 16-bit channel value to a LUT segment index returns the correct index at boundaries (0, mid-point, maximum), at power-of-two segment counts, and at non-power-of-two sizes.
- **`lerp_u16` tests**: verify the linear interpolation function used inside 1D LUT evaluation. Edge cases include saturation at 0xffff, zero-span segments, and rounding behavior.
- **Format conversion round-trips**: unit tests that write a pixel in a given format, read it back through the `pixel_read_line` callback, and assert the `pixel_argb_u16` fields round-trip correctly — ensuring no channels are swapped and no precision is lost for exact values.

### Running KUnit Tests

KUnit tests run inside a stripped-down kernel image via `kunit.py`:

```bash
# Run all VKMS KUnit tests (no hardware required):
tools/testing/kunit/kunit.py run \
    --kunitconfig=drivers/gpu/drm/vkms/tests

# Sample output:
#     [PASSED] vkms_formats_test: xrgb8888_read_roundtrip
#     [PASSED] vkms_lut_test: get_lut_index_boundary_max
#     [PASSED] vkms_lut_test: lerp_u16_midpoint
#     [PASSED] vkms_lut_test: lerp_u16_saturation
# Test suite summary: 12 passed, 0 failed
```

The `.kunitconfig` at `drivers/gpu/drm/vkms/tests/.kunitconfig` specifies:

```
CONFIG_KUNIT=y
CONFIG_DRM=y
CONFIG_DRM_VKMS=y
CONFIG_DRM_VKMS_KUNIT_TEST=y
```

The `DRM_VKMS_KUNIT_TEST` Kconfig option depends on both `DRM_VKMS` and `KUNIT`, so it can only be enabled in test builds and is never compiled into a production kernel image. This makes the tests safe to ship in the kernel tree without bloating production builds.

### Integration with CI

The VKMS KUnit tests run in Mesa CI (via the kernel CI pipeline) and in the Linux kernel's own CI system (KernelCI / LAVA). Because they need no physical device, they run on any architecture — x86-64, arm64, and RISC-V — providing broad coverage of numerical bugs that would otherwise be platform-specific.

[Source: PATCH 7/7 drm/vkms — Add how to run the KUnit tests](https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg484731.html) | [PATCH V13 02/51 — Add KUnit tests for VKMS LUT handling](http://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg132423.html)

---

## VKMS in Mesa and Vulkan CI

### The VKMS + Software Renderer Stack

Mesa's CI infrastructure for KMS/display-coupled tests combines three components:

1. **VKMS** — provides a DRM device and synthetic display for Wayland compositors
2. **LLVMpipe / lavapipe** — software OpenGL 4.5 / Vulkan 1.3 rasteriser, no GPU needed
3. **sway or weston** — Wayland compositor running on the VKMS DRM device

This stack runs inside ephemeral VMs or containers on bare-metal CI runners at freedesktop.org. The display server boots against VKMS, enabling Wayland socket creation, and then dEQP or other test clients connect through the Wayland socket or via direct EGL on GBM.

### Boot-to-VKMS Pattern

A typical CI VM setup script:

```bash
#!/bin/bash
# ci/setup-vkms.sh — boot-to-VKMS for Mesa CI

# Load the virtual display driver
modprobe vkms enable_cursor=1 enable_writeback=1

# Set up a minimal Wayland compositor over VKMS
export WLR_BACKENDS=drm
export WLR_DRM_DEVICES=/dev/dri/card0
export WLR_RENDERER=pixman          # software renderer in compositor
export LIBGL_ALWAYS_SOFTWARE=1      # force LLVMpipe for EGL clients
export XDG_RUNTIME_DIR=/tmp/wayland-runtime
mkdir -p "$XDG_RUNTIME_DIR"

sway -c /dev/null &                 # minimal sway config
SWAY_PID=$!

# Wait for the Wayland socket to appear
until [ -S "$XDG_RUNTIME_DIR/wayland-0" ]; do sleep 0.1; done

echo "Wayland socket ready on $XDG_RUNTIME_DIR/wayland-0"
```

### Running dEQP via VKMS

For Vulkan CTS (`dEQP-VK.*`) the display backend is typically `EGL_PLATFORM=surfaceless` or a Wayland surface backed by the VKMS compositor:

```bash
# Run VK-GL-CTS against lavapipe on the VKMS-backed Wayland session
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json \
WAYLAND_DISPLAY=wayland-0 \
    deqp-runner run \
        --deqp VK-GL-CTS/build/deqp-vk \
        --caselist test-cases.txt \
        --output results/ \
        --jobs 8
```

The `deqp-runner` utility (part of Mesa's CI tooling) parallelises CTS execution and reports flakiness, filtering failures against a known-bad list. Tests that pass on lavapipe confirm that the Vulkan frontend is correct independently of GPU hardware.

### Weston DRM-Backend Testing with virtme

The Weston project uses VKMS together with `virtme` (a QEMU wrapper for running the host kernel in a light VM) to test the Weston DRM backend:

```bash
# .gitlab-ci/virtme-scripts/run-weston-tests.sh (simplified)
export XDG_RUNTIME_DIR=/tmp/tests
export WESTON_TEST_SUITE_DRM_DEVICE=card0  # VKMS card

cd weston
meson build
ninja -C build test
```

The virtme environment boots a real (host) kernel image so VKMS tests run against the same kernel under test, not a distribution kernel. This catches regressions where a kernel change breaks the DRM-backend path in Weston.

[Source: Collabora blog — Testing Weston DRM/KMS backends with virtme and VKMS](https://www.collabora.com/news-and-blog/blog/2020/08/07/testing-weston-drm-kms-backends-with-virtme-and-vkms/) | [Mesa CI documentation](https://docs.mesa3d.org/ci/index.html)

---

## Configfs Runtime Reconfiguration

### The Old Module-Parameter Approach

Before the configfs work, VKMS was configured entirely via module parameters at load time. Changing the number of connectors or enabling overlay planes required `modprobe -r vkms && modprobe vkms enable_overlay=1`. This was inconvenient for complex CI scenarios requiring multiple independent VKMS instances or dynamic hot-plug simulation.

### Configfs-Based Multi-Instance Management

Work merged in Linux 6.13 (led by Louis Chauvet at Bootlin) adds a configfs interface under `/config/vkms/`. Each subdirectory is an independent VKMS device instance:

```bash
# mount configfs if not already mounted
mount -t configfs none /config

# load VKMS without creating the default device
modprobe vkms create_default_dev=0

# create a new VKMS instance named "test-compositor"
mkdir /config/vkms/test-compositor

# configure planes
mkdir /config/vkms/test-compositor/planes/plane0
echo 1 > /config/vkms/test-compositor/planes/plane0/type   # primary

mkdir /config/vkms/test-compositor/planes/plane1
echo 0 > /config/vkms/test-compositor/planes/plane1/type   # overlay

mkdir /config/vkms/test-compositor/planes/cursor0
echo 2 > /config/vkms/test-compositor/planes/cursor0/type  # cursor

# configure a CRTC
mkdir /config/vkms/test-compositor/crtcs/crtc0
echo 1 > /config/vkms/test-compositor/crtcs/crtc0/writeback  # enable writeback

# configure encoder and connector
mkdir /config/vkms/test-compositor/encoders/enc0
mkdir /config/vkms/test-compositor/connectors/conn0
echo 1 > /config/vkms/test-compositor/connectors/conn0/status  # connected

# wire up the pipeline via symlinks
ln -s /config/vkms/test-compositor/planes/plane0  \
      /config/vkms/test-compositor/crtcs/crtc0/planes/plane0
ln -s /config/vkms/test-compositor/encoders/enc0  \
      /config/vkms/test-compositor/crtcs/crtc0/encoders/enc0
ln -s /config/vkms/test-compositor/connectors/conn0 \
      /config/vkms/test-compositor/encoders/enc0/connectors/conn0

# enable the device (registers the DRM device)
echo 1 > /config/vkms/test-compositor/enabled
# /dev/dri/cardN is now available
```

### Runtime EDID Injection

EDID can be written at any time to the connector's `edid` attribute (maximum size: `PAGE_SIZE` = 4096 bytes on x86, enough for extended EDID):

```bash
# inject a 1920×1200 EDID:
cp custom-1920x1200.edid \
    /config/vkms/test-compositor/connectors/conn0/edid

# trigger hot-plug detect so the compositor re-reads modes:
echo connected > /config/vkms/test-compositor/connectors/conn0/status
```

### Multi-Connector Topology

CI tests for multi-monitor compositor code paths can instantiate multiple connectors on a single VKMS device:

```bash
mkdir /config/vkms/multi-head/connectors/hdmi-0
mkdir /config/vkms/multi-head/connectors/dp-1
echo 1 > /config/vkms/multi-head/connectors/hdmi-0/status
echo 1 > /config/vkms/multi-head/connectors/dp-1/status
# now the compositor sees two connected outputs
```

[Source: Bootlin ELC Europe 2025 — VKMS new ConfigFS](https://bootlin.com/pub/conferences/2025/elce/chauvet-vkms.pdf) | [LKML PATCH 18/22 — Introduce config for connector EDID](https://lkml.org/lkml/2025/10/18/66)

---

## Comparison with Alternative Headless Backends

When setting up a CI pipeline, several approaches are available for headless graphics testing. Each occupies a different point in the trade-off between realism, performance, and complexity.

### VKMS (recommended for DRM/KMS testing)

VKMS sits inside the kernel and implements the full DRM subsystem API. It is the most realistic virtual display option for:
- Testing DRM atomic modesetting code paths
- Exercising Wayland compositor DRM backends
- CRC-based pixel regression tests
- New DRM feature development (color pipelines, HDR, VRR)

Limitations: CPU-only composition is slow; no GPU acceleration; a compositor must be willing to drive a DRM backend (not all compositors support this in headless mode without patches).

### Xvfb (X Virtual Framebuffer)

`Xvfb` is an X11 server backed by an in-memory framebuffer. It predates DRM/KMS and operates entirely in the X11 protocol layer.

```bash
Xvfb :99 -screen 0 1920x1080x24 &
DISPLAY=:99 legacy-gl-app
DISPLAY=:99 import -window root screenshot.png
```

**Use when**: testing legacy X11 applications that have not been ported to Wayland; running GL applications that use GLX rather than EGL. Xvfb does not implement DRM/KMS, so it is not suitable for testing compositor code that calls `drmModeSetCrtc` or uses atomic commits.

### GBM Offscreen Rendering (no display)

For pure rendering tests without any display, applications can create a GBM device directly:

```c
int drm_fd = open("/dev/dri/renderD128", O_RDWR);
struct gbm_device *gbm = gbm_create_device(drm_fd);
EGLDisplay egl_dpy = eglGetPlatformDisplayEXT(
    EGL_PLATFORM_GBM_KHR, gbm, NULL);
/* create a surfaceless EGL context or a GBM surface */
```

This avoids any display server entirely. It is appropriate for benchmarks, compute workloads, and tests that compare rendered textures by reading them back via `glReadPixels`. It does not test compositor or display-pipeline code paths at all.

### virtio-gpu in QEMU

`virtio-gpu` is the paravirtualized display driver for QEMU guests. With `virgl` enabled, OpenGL commands are forwarded to the host GPU:

```bash
qemu-system-x86_64 \
    -device virtio-gpu-gl \
    -display sdl,gl=on \
    -m 4G -smp 4 \
    -kernel guest-vmlinuz -initrd guest-initrd
```

**Use when**: the test requires a full VM (with network isolation, snapshot/restore, kernel versions different from the host); or when the guest needs approximate GPU acceleration via virgl. virtio-gpu is heavier than VKMS because it requires a running QEMU instance; it is appropriate for integration tests where the guest OS must differ from the host.

### Xvnc (VNC Virtual Display)

`Xvnc` starts an X11 server that also acts as a VNC server, rendering into an off-screen framebuffer and serving it over the network. It is useful for remote desktop CI where a human observer may want to view the running session, but it adds network overhead and VNC encoding latency compared to VKMS or Xvfb.

### Summary Comparison Table

| Backend | DRM/KMS | Wayland | CRC tests | GPU accel | Complexity |
|---------|---------|---------|-----------|-----------|------------|
| VKMS | Yes (full) | Yes | Yes | No (CPU) | Low |
| Xvfb | No | No | No | No | Very low |
| GBM offscreen | Partial | No | No | Yes | Medium |
| virtio-gpu | Yes | Yes | No | Approx (virgl) | High |
| Xvnc | No | No | No | No | Low |

---

## Other Virtual Display Drivers

### virtio-gpu

`virtio-gpu` is the paravirtualized display driver for QEMU/KVM virtual machines:

```bash
# QEMU with virtio-gpu:
qemu-system-x86_64 \
    -device virtio-gpu-pci \
    -display sdl,gl=on \
    -kernel vmlinuz -initrd initrd.img

# In the guest: /dev/dri/card0 is virtio-gpu
# Supports: virgl (OpenGL via host GPU), 2D display
```

virtio-gpu + virgl passes OpenGL commands to the host GPU, enabling guest VMs to use hardware acceleration without GPU passthrough.

### DRM Dummy Driver

The `drm_dummy` driver (proposed, not merged) creates a minimal DRM device for testing without any display output or pixel composition:

```c
/* Minimal DRM driver for testing: */
static const struct drm_driver dummy_driver = {
    .driver_features = DRIVER_MODESET | DRIVER_ATOMIC,
    .name = "drm-dummy",
};
```

### simpledrm

`simpledrm` (`drivers/gpu/drm/tiny/simpledrm.c`) claims a simple framebuffer from the UEFI boot screen and creates a DRM device over it — useful for booting to a display before GPU drivers load:

```bash
dmesg | grep simpledrm
# Output: simpledrm: bound to /dev/dri/card0 (efifb at ...)
```

---

## DRM Leases and VR Direct Mode

### DRM Lease Concept

A DRM lease allows one DRM master to delegate exclusive access to a subset of DRM resources (CRTCs, connectors, planes) to another process. Used for VR direct mode:

```
SteamVR / OpenXR runtime
  ├─ leases connector 0 (headset) from the desktop compositor
  ├─ runs its own modesetting on the leased connector
  └─ desktop compositor retains control of other connectors
```

### Creating a DRM Lease

```c
/* Lessor (compositor): grant lease to VR runtime: */
uint32_t objects[] = { connector_id, crtc_id, plane_id };
int lessee_fd = drmModeCreateLease(fd, objects, 3, 0, &lease_id);
/* Send lessee_fd to the VR runtime process (e.g. via SCM_RIGHTS) */

/* Lessee (VR runtime): use the leased resources exclusively: */
/* The lessee uses the received fd as if it were the DRM master: */
drmModeSetCrtc(lessee_fd, crtc_id, fb_id, 0, 0, &connector_id, 1, &mode);
/* VR headset renders at 90Hz without compositor involvement */
```

### Lease Lifecycle

```c
/* Compositor: revoke lease: */
drmModeRevokeLease(fd, lease_id);

/* List active leases: */
struct drm_mode_list_lessees list;
drmModeListLessees(fd, &list);
```

OpenXR runtimes (Monado) use DRM leases on Linux for direct-to-HMD rendering, bypassing the desktop compositor entirely for minimum latency.

---

## GPU Testing with LLVMpipe and softpipe

### LLVMpipe

LLVMpipe is Mesa's LLVM-JIT software rasteriser. It provides full OpenGL 4.5 and Vulkan (via lavapipe) on CPU:

```bash
# Force LLVMpipe:
LIBGL_ALWAYS_SOFTWARE=1 glxinfo | grep "OpenGL renderer"
# Output: OpenGL renderer string: llvmpipe (LLVM 17, 256 bits)

# For Vulkan:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json vulkaninfo
```

### lavapipe

lavapipe is Mesa's Vulkan software rasteriser (LLVMpipe + Vulkan frontend):

```bash
# Check lavapipe:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json \
    vulkaninfo --summary
# Shows: Vulkan 1.3 on llvmpipe

# CI use: run Vulkan CTS on lavapipe:
VK_ICD_FILENAMES=.../lvp_icd.x86_64.json \
    deqp-runner run --deqp deqp-vk --caselist dEQP-VK.api.txt \
    --output results/ --jobs 8
```

lavapipe supports Vulkan 1.3 + most extensions, making it suitable for CTS conformance testing without a GPU.

---

## Container and VM Display Stacks

### Running Wayland in Docker/Podman

```bash
# Run a Wayland compositor in a container with VKMS:
docker run --privileged \
    --device /dev/dri/card0 \
    -e XDG_RUNTIME_DIR=/tmp/runtime \
    -v /tmp/runtime:/tmp/runtime \
    ubuntu weston --backend=drm-backend.so --drm-device=card0

# For software-only (no DRI device needed):
docker run -e LIBGL_ALWAYS_SOFTWARE=1 \
    ubuntu weston --backend=headless-backend.so
```

The `--privileged` flag (or more targeted `--device /dev/dri`) is needed to access DRM device nodes from inside a container. The VKMS module must be loaded on the host before the container starts.

### X Virtual Framebuffer (Xvfb)

For legacy X11 applications:

```bash
# Start Xvfb:
Xvfb :99 -screen 0 1920x1080x24 &
DISPLAY=:99 some_x11_app
# Capture screenshot:
DISPLAY=:99 import -window root screenshot.png
```

### GPU Passthrough (VFIO)

For VMs needing real GPU performance:

```bash
# Bind GPU to VFIO:
echo "10de 2684" > /sys/bus/pci/drivers/vfio-pci/new_id  # NVIDIA A100
# QEMU command:
qemu-system-x86_64 \
    -device vfio-pci,host=01:00.0 \
    -m 8G -smp 8
# Guest uses the real GPU via VFIO passthrough
```

---

## Roadmap

### Near-term (6–12 months)

- **Color pipeline API integration**: An RFC patchset (v5, 44 patches) to implement the DRM Color Pipeline API is in progress, with VKMS as the reference implementation for validating the new `drm_color_pipeline` abstraction covering 1D EOTF curves, 3×4 CTM, HDR multiplier, 3D LUT, and inverse EOTF stages — mirroring the color pipeline used by gamescope. [Source](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg111875.html)
- **configfs-based reconfiguration**: The configfs interface (merged in Linux 6.13) allows VKMS instances to be reconfigured at runtime without reloading the module, enabling hotplug/hotremove of virtual connectors and dynamic EDID or refresh-rate changes — essential for compositor hot-plug CI tests. [Source](https://docs.kernel.org/gpu/vkms.html)
- **Multi-planar YUV format support**: Adding NV12 and other semi-planar YUV formats to the software compositor (`vkms_composer.c`) so that video decode pipelines (VA-API, V4L2) can be integration-tested headlessly. Note: needs verification of landing target kernel version.
- **Blend mode properties on overlay planes**: Implementing per-plane `pixel blend mode` properties (None, Pre-multiplied, Coverage) so IGT `kms_plane_alpha_blend` tests can run against VKMS, closing a coverage gap relative to real hardware drivers. [Source](https://www.phoronix.com/news/VKMS-Driver-2023)
- **Multiple CRTC support**: Extending VKMS to expose more than one CRTC/connector pair so multi-monitor compositor code paths (mirror, extended desktop) can be exercised in CI without physical hardware. [Source](https://docs.kernel.org/gpu/vkms.html)

### Medium-term (1–3 years)

- **HDR/wide-color-gamut testing surface**: Once the Color Pipeline API lands, VKMS is expected to become the canonical headless surface for testing HDR compositor paths (Wayland `color-management-v1` protocol) end-to-end, from surface color space through CRTC LUT to writeback output capture. [Source](https://www.mail-archive.com/wayland-devel@lists.freedesktop.org/msg42796.html)
- **Variable refresh rate / Adaptive-Sync emulation**: Planned support for VRR/FreeSync via VKMS by driving the hrtimer at variable periods, enabling IGT `kms_vrr` tests to run headlessly; this also requires PRIME buffer-sharing groundwork. Note: needs verification of RFC patchset status.
- **composer_enabled refcounting**: The current boolean `composer_enabled` flag shared between writeback and CRC capture paths needs to be replaced with proper reference counting so both features can co-exist without races — a prerequisite for reliable parallel CI test runs. [Source](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2154714.html)
- **PRIME / dma-buf import support**: Allowing VKMS to import dma-buf objects from software renderers (LLVMpipe, lavapipe) would enable zero-copy headless rendering pipelines in CI, removing the current CPU-memcpy path for composited frames. Note: needs verification.

### Long-term

- **Acceleration via DRM scheduler integration**: Longer-term proposals envision wiring VKMS into `drm_gpu_scheduler` so asynchronous flip and compute workloads can be stress-tested for scheduling correctness without requiring real GPU hardware.
- **Synthetic display feature flags**: Exposing configurable capability bits (rotation, scaling, cursor size, connector type) through configfs to simulate the exact display feature profile of a target GPU, allowing driver feature tests to be run deterministically in CI without that GPU being present.
- **Integration with kernel self-test (kselftest) framework**: Moving VKMS-driven tests into `tools/testing/selftests/drm/` so they run as part of the standard kernel CI suite (`kunit`/`kselftest`) rather than requiring the separate IGT harness. Note: needs verification of upstream consensus.
- **Virtualised display topology for VR/XR testing**: Coupling VKMS with DRM lease infrastructure (Ch121) to simulate multi-display direct-mode VR setups so OpenXR runtimes like Monado can run their headless CI without a physical HMD.

---

## Integrations

- **Ch01 (DRM Architecture)** — VKMS is a DRM driver implementing the same `drm_driver` and `drm_crtc_helper_funcs` interfaces as real GPU drivers; it serves as a clean reference implementation
- **Ch02 (KMS Display Pipeline)** — VKMS implements the full atomic modesetting path; IGT's `kms_atomic` test suite runs against VKMS to verify atomic commit semantics
- **Ch139 (DRM Hardware Planes)** — VKMS implements primary, overlay, and cursor planes; software composition in `vkms_composer.c` demonstrates how planes blend with premultiplied alpha
- **Ch155 (evdi)** — Both VKMS and evdi are virtual DRM drivers; VKMS uses hrtimer for vblank, evdi uses a userspace-driven update model
- **Ch34 (Wayland)** — Weston and sway can run on VKMS for headless CI testing; the display protocol stack is fully exercised without physical hardware
- **Ch121 (DRM Lease/VR Direct Display)** — DRM leases allow VR runtimes to take exclusive control of a VKMS connector for headless OpenXR testing
- **Ch125 (RenderDoc)** — RenderDoc captures work from lavapipe (software Vulkan) for shader debugging without a GPU; VKMS provides the display surface for the captured session
- **Ch147 (VA-API Video Decode)** — VKMS's planned YUV plane support will enable headless VA-API decode pipeline tests by providing a DRM display target for video composition

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
