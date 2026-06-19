# Chapter 144: Boot Graphics Pipeline: From Firmware to KMS Handoff

**Part I — The Kernel Layer**

---

This chapter traces the complete lifecycle of the Linux display stack from the moment UEFI firmware illuminates the first pixels to the point at which a native GPU driver performs its first full KMS atomic modeset and hands control to a compositor or Plymouth boot splash. The audience is systems and driver developers integrating boot graphics on new platforms, driver authors adding native KMS support, and engineers debugging early-boot display regressions — including those working on embedded SBCs, UEFI x86 systems, and Secure Boot environments.

---

## Table of Contents

1. [The Boot Display Timeline](#the-boot-display-timeline)
2. [UEFI GOP: Graphics Output Protocol](#uefi-gop-graphics-output-protocol)
3. [simplefb: Device-Tree Framebuffers on ARM SBCs](#simplefb-device-tree-framebuffers-on-arm-sbcs)
4. [simpledrm: DRM Wrapper Before the Native Driver](#simpledrm-drm-wrapper-before-the-native-driver)
5. [The Kernel DRM Driver Probe Race](#the-kernel-drm-driver-probe-race)
6. [KMS Atomic Modesetting at First Modeset](#kms-atomic-modesetting-at-first-modeset)
7. [Plymouth Boot Splash](#plymouth-boot-splash)
8. [Boot-to-Compositor Handoff](#boot-to-compositor-handoff)
9. [Secure Boot and Early Graphics](#secure-boot-and-early-graphics)
10. [Embedded Considerations: U-Boot and Panel Sequencing](#embedded-considerations-u-boot-and-panel-sequencing)
11. [Raspberry Pi Boot Graphics](#raspberry-pi-boot-graphics)
12. [Debugging Boot Display Issues](#debugging-boot-display-issues)
13. [Integrations](#integrations)

---

## The Boot Display Timeline

The Linux boot display pipeline is a relay race with five distinct batons:

1. **UEFI firmware / GOP** — firmware identifies the physical display, sets a video mode, fills the GOP framebuffer in EFI memory, and populates `screen_info` via the EFI stub before `ExitBootServices()`.
2. **sysfb** — a `device_initcall` reads `screen_info` and registers a `"simple-framebuffer"` platform device (or falls back to `"efi-framebuffer"`/`"vesa-framebuffer"` for legacy paths).
3. **efifb / simplefb / simpledrm** — the platform device is claimed by one of these drivers; simpledrm (CONFIG_DRM_SIMPLEDRM, kernel ≥ 5.14) is the modern choice and provides a minimal DRM device backed by the firmware framebuffer, enabling fbcon output before any GPU driver loads.
4. **Plymouth** — started early from initramfs, Plymouth's DRM renderer (`/dev/dri/card0`) uses legacy DRM ioctls to display an animated boot splash, then calls `drmDropMaster()` at VT switch time.
5. **Native GPU driver** (i915, amdgpu, vc4, …) — probes via PCI or platform bus, calls `aperture_remove_conflicting_pci_devices()` to displace simpledrm/efifb, performs the first KMS atomic modeset, and thereafter provides the display device consumed by the Wayland compositor.

Understanding this chain is essential for anyone adding boot-splash support to a new SoC, debugging a black screen on a UEFI system with a discrete GPU, or tracing why simpledrm lingers too long after an amdgpu probe.

### Historical Evolution

Before kernel 5.14, the boot display situation was far messier. The standard approach was:

- x86/EFI: `efifb` driver (no DRM device, fbdev only); Plymouth fell back to `/dev/fb0`.
- ARM/DT: `simplefb` driver (no DRM device); early kernels sometimes had no console at all until the native driver loaded.
- No coherent handoff mechanism: native GPU drivers would simply overwrite the framebuffer, causing a visible flash and briefly corrupting the display.

The introduction of simpledrm in 5.14 (tracked via the `[PATCH v4] drm/simpledrm: Add simple display pipeline driver` series by Thomas Zimmermann) unified the boot display path:

- A real DRM device is available before the native driver loads.
- Plymouth can use the DRM renderer from the start.
- The aperture API provides a clean, race-free handoff.

Fedora 36 (kernel ~5.17, released April 2022) was the first major distribution to make `CONFIG_DRM_SIMPLEDRM=y` the default and actively replace `efifb`/`vesafb`/`simplefb` with simpledrm as the boot framebuffer driver. Critically, Fedora also set `CONFIG_DRM=y` (built-in rather than a module) to ensure the DRM console is available during early boot before initrd modules load — essential for displaying error messages if the initrd itself fails.

[Source: Fedora Change — Replace efifb+simplefb with simpledrm](https://fedoraproject.org/wiki/Changes/ReplaceFbdevDrivers)

---

## UEFI GOP: Graphics Output Protocol

### GOP Overview

The UEFI Graphics Output Protocol (GOP) is the UEFI-era replacement for VGA BIOS INT 10h and VESA VBE. The firmware exposes one or more `EFI_GRAPHICS_OUTPUT_PROTOCOL` handles; each handle represents a logical display output. The protocol provides three operations: `QueryMode`, `SetMode`, and `Blt`. The Linux EFI stub enumerates GOP handles and records the selected mode in the kernel's `screen_info` structure before calling `ExitBootServices()`.

[Source: UEFI Specification 2.10, §12.9](https://uefi.org/specs/UEFI/2.10/12_Protocols_Console_Support.html#graphics-output-protocol)

### EFI Stub GOP Enumeration

The relevant kernel file is `drivers/firmware/efi/libstub/gop.c`. The top-level entry point is:

```c
/* drivers/firmware/efi/libstub/gop.c */
efi_status_t efi_setup_graphics(struct screen_info *si, struct edid_info *edid)
```

[Source: linux/drivers/firmware/efi/libstub/gop.c](https://elixir.bootlin.com/linux/latest/source/drivers/firmware/efi/libstub/gop.c)

The function enumerates GOP handles via `efi_bs_call(locate_handle_buffer, EFI_GRAPHICS_OUTPUT_PROTOCOL_GUID, ...)`. When multiple handles are found — as is common on systems with a UEFI Console Splitter — `find_handle_with_primary_gop()` selects the handle that implements the `EFI_CONSOLE_OUT_DEVICE_GUID` (`ConOut`) protocol. This ensures the recorded framebuffer corresponds to the actual display rather than a virtual multiplexed output.

`setup_screen_info()` then fills the `struct screen_info` fields from the GOP mode pointer:

```c
/* drivers/firmware/efi/libstub/gop.c (simplified) */
si->lfb_base   = (u32)(u64)gop->mode->frame_buffer_base;
si->ext_lfb_base = (u32)((u64)gop->mode->frame_buffer_base >> 32);
si->lfb_width  = info->horizontal_resolution;
si->lfb_height = info->vertical_resolution;
si->orig_video_isVGA = VIDEO_TYPE_EFI;  /* 0x70 */
```

The 64-bit framebuffer address is split across `lfb_base` (low 32 bits) and `ext_lfb_base` (high 32 bits), controlled by the `VIDEO_CAPABILITY_64BIT_BASE` flag in `capabilities`. On systems with more than 4 GB of RAM the GOP framebuffer may lie above the 4 GB boundary; drivers must reconstruct the full address as `((u64)ext_lfb_base << 32) | lfb_base`.

### Pixel Format Handling

Three pixel formats are supported:

- `PIXEL_BGR_RESERVED_8BIT_PER_COLOR` — BGRX32; the most common format on contemporary UEFI firmware.
- `PIXEL_RGB_RESERVED_8BIT_PER_COLOR` — RGBX32; less common but handled symmetrically.
- `PIXEL_BIT_MASK` — custom bitmask; `find_bits()` computes bit positions for each channel.
- `PIXEL_BLT_ONLY` — no linear framebuffer; rejected by the stub with no `screen_info` population.

### screen_info Structure

```c
/* include/uapi/linux/screen_info.h */
struct screen_info {
    __u8  orig_x, orig_y;
    __u16 ext_mem_k;
    __u16 orig_video_page;
    __u8  orig_video_mode, orig_video_cols;
    __u8  flags, unused2;
    __u16 orig_video_ega_bx, unused3;
    __u8  orig_video_lines;
    __u8  orig_video_isVGA;   /* VIDEO_TYPE_EFI = 0x70 */
    __u16 orig_video_points;
    __u16 lfb_width, lfb_height, lfb_depth;
    __u32 lfb_base;           /* low 32 bits of FB physical address */
    __u32 lfb_size;
    __u16 cl_magic, cl_offset;
    __u16 lfb_linelength;
    __u8  red_size, red_pos;
    __u8  green_size, green_pos;
    __u8  blue_size, blue_pos;
    __u8  rsvd_size, rsvd_pos;
    __u16 vesapm_seg, vesapm_off, pages, vesa_attributes;
    __u32 capabilities;       /* VIDEO_CAPABILITY_64BIT_BASE etc. */
    __u32 ext_lfb_base;       /* upper 32 bits when 64-bit base */
    __u8  _reserved[2];
} __attribute__((packed));
```

[Source: linux/include/uapi/linux/screen_info.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/screen_info.h)

On x86 the `setup_graphics()` function in `drivers/firmware/efi/libstub/x86-stub.c` calls `efi_setup_graphics()` with pointers into `boot_params`, so the populated data becomes the global `screen_info` that persists after `ExitBootServices()` and is later read by sysfb.

[Source: linux/drivers/firmware/efi/libstub/x86-stub.c](https://elixir.bootlin.com/linux/latest/source/drivers/firmware/efi/libstub/x86-stub.c)

### sysfb: Converting screen_info to a Platform Device

`drivers/firmware/sysfb.c` implements the `sysfb_init()` device_initcall that bridges the firmware-populated `screen_info` into the Linux device model.

[Source: linux/drivers/firmware/sysfb.c](https://elixir.bootlin.com/linux/latest/source/drivers/firmware/sysfb.c)

```c
/* drivers/firmware/sysfb.c — simplified control flow */
static int __init sysfb_init(void)
{
    const struct screen_info *si = &screen_info;

    /* Try to create a generic "simple-framebuffer" device */
    pd = sysfb_create_simplefb(si, &dpy, parent);
    if (!IS_ERR(pd))
        goto done;

    /* Fall back to legacy devices based on video type */
    switch (si->orig_video_isVGA) {
    case VIDEO_TYPE_EFI:
        pd = platform_device_alloc("efi-framebuffer", 0);  break;
    case VIDEO_TYPE_VLFB:
        pd = platform_device_alloc("vesa-framebuffer", 0); break;
    /* ... other types ... */
    }
done:
    return platform_device_add(pd);
}
```

`CONFIG_SYSFB_SIMPLEFB=y` causes sysfb to prefer the `"simple-framebuffer"` path, which is then claimed by simpledrm. Without this Kconfig option, only the legacy `efi-framebuffer` platform device is created, which is claimed by `efifb`.

`sysfb_disable()` is called by native GPU drivers to prevent sysfb from creating a conflicting platform device if the native driver probes before `sysfb_init()` completes. A mutex protects against TOCTOU races: if a native driver sets the `disabled` flag before the device_initcall runs, `sysfb_init()` finds the flag set and skips device creation entirely.

### efifb: The Legacy EFI Framebuffer Driver

`drivers/video/fbdev/efifb.c` implements the fallback path for systems where `CONFIG_SYSFB_SIMPLEFB` is not set or where the framebuffer geometry is incompatible with the generic mode.

[Source: linux/drivers/video/fbdev/efifb.c](https://elixir.bootlin.com/linux/latest/source/drivers/video/fbdev/efifb.c)

efifb maps the GOP framebuffer with `ioremap_wc()` (write-combining) for reasonable performance, and calls `devm_aperture_acquire_for_platform_device()` to register aperture ownership so that a native GPU driver can later reclaim the region. The driver implements only `fb_setcolreg` and `fb_destroy` — there is no modesetting capability.

---

## simplefb: Device-Tree Framebuffers on ARM SBCs

On ARM and RISC-V embedded platforms, firmware (U-Boot, ARM Trusted Firmware, or the bootloader) does not use EFI. Instead, it writes a `simple-framebuffer` node into the kernel's device tree:

```dts
/* Example DT node written by U-Boot before booting Linux */
framebuffer0: framebuffer@0xff700000 {
    compatible = "simple-framebuffer";
    reg = <0x00 0xff700000 0x00 0x008ca000>;
    width = <1920>;
    height = <1080>;
    stride = <7680>;
    format = "x8r8g8b8";
    status = "okay";
};
```

[Source: Documentation/devicetree/bindings/display/simple-framebuffer.yaml](https://elixir.bootlin.com/linux/latest/source/Documentation/devicetree/bindings/display/simple-framebuffer.yaml)

The simplefb driver (`drivers/video/fbdev/simplefb.c`) parses these fields via `simplefb_parse_dt()`, maps the region with `ioremap_wc()`, optionally acquires clocks and power-domain regulators required to keep the display hardware alive, and registers a `struct fb_info`.

[Source: linux/drivers/video/fbdev/simplefb.c](https://elixir.bootlin.com/linux/latest/source/drivers/video/fbdev/simplefb.c)

Key simplefb limitations:

- **No modesetting.** simplefb implements only `simplefb_setcolreg()` for pseudo-palette. The resolution and format are fixed by whatever the bootloader set.
- **Clock/regulator lifecycle.** On SoCs where the display clocks are shared between the bootloader display path and the Linux display driver, simplefb acquires them in probe to prevent them from being gated off before the native DRM driver takes over. Without this, the screen goes dark at `ExitBootServices()` equivalent on embedded platforms.
- **Format support.** The `simplefb_formats[]` array maps format strings (`"x8r8g8b8"`, `"r5g6b5"`, etc.) to `fb_bitfield` structs; unknown format strings cause probe to fail with `-EINVAL`.
- **Power domain management.** The simplefb driver optionally acquires and enables power domains via `genpd` (generic power domain) using `simplefb_attach_genpds()`. This is critical on SoCs where the display subsystem power domain is separate from the CPU power domain and may otherwise be gated by runtime PM before the native driver initialises.

The DT binding is in `Documentation/devicetree/bindings/display/simple-framebuffer.yaml` and is stable; vendor-specific variants add a vendor prefix (`"allwinner,simple-framebuffer"`) before the generic compatible string.

### simplefb vs. simpledrm: Choosing the Right Driver

On modern kernels with `CONFIG_DRM_SIMPLEDRM=y`, simpledrm supersedes simplefb for most use cases:

| Feature | simplefb | simpledrm |
|---|---|---|
| DRM device (`/dev/dri/card0`) | No | Yes |
| fbdev interface (`/dev/fb0`) | Yes | Yes (via fbdev emulation) |
| Plymouth DRM renderer support | No | Yes |
| Aperture API integration | No | Yes |
| GEM buffer objects | No | Yes |
| Kernel version | 3.x+ | 5.14+ |

simplefb remains relevant on very old kernels, in configurations where `CONFIG_DRM` is not built-in, and as a reference implementation for understanding the minimal DT framebuffer interface that simpledrm builds upon.

---

## simpledrm: DRM Wrapper Before the Native Driver

### Motivation

Both efifb and simplefb present a `struct fb_info` to the Linux fbdev layer. This gives console output but no DRM device, which means:

- Plymouth's DRM renderer cannot open `/dev/dri/card0`.
- Compositor startup races: if the native driver is slow to probe, the compositor gets no DRM device.
- No atomic modesetting; no GEM buffer objects; no DMA-BUF export.

`CONFIG_DRM_SIMPLEDRM` (introduced in kernel 5.14) wraps the firmware framebuffer in a minimal DRM driver, providing a real `/dev/dri/card0` with a single CRTC, connector, and encoder, backed by a `drm_simple_display_pipe`.

[Source: linux/drivers/gpu/drm/tiny/simpledrm.c (≤6.15) / drivers/gpu/drm/sysfb/simpledrm.c (≥6.16)](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/tiny/simpledrm.c)

Note: In kernel 6.16 simpledrm was relocated to `drivers/gpu/drm/sysfb/` as part of a broader DRM sysfb reorganisation.

### Binding and Platform Device

On x86, simpledrm binds to the `"simple-framebuffer"` platform device registered by sysfb (requires `CONFIG_SYSFB_SIMPLEFB=y`). On ARM/DT, simpledrm also binds directly to `"simple-framebuffer"` device-tree nodes. An `fs_initcall` in simpledrm scans the `/chosen` OF node for `"simple-framebuffer"` compatible children and calls `of_platform_device_create()` on each — a workaround for the fact that the standard OF matching machinery does not process `/chosen` children. This path is critical for Apple Silicon, where the bootloader places the framebuffer descriptor in `/chosen` rather than a standard memory subtree.

### Aperture Acquisition

The core mechanism enabling safe handoff to a native driver is the aperture API:

```c
/* drivers/gpu/drm/tiny/simpledrm.c — probe */
ap = devm_aperture_acquire(dev, mem->start, resource_size(mem),
                           &simpledrm_aperture_funcs);
```

The `simpledrm_aperture_funcs.detach` callback is:

```c
/* Called when a native driver displaces simpledrm */
static void simpledrm_aperture_detach(struct drm_device *dev)
{
    /*
     * If simpledrm gets detached from the aperture,
     * it's like unplugging the device. So call drm_dev_unplug().
     */
    drm_dev_unplug(dev);
}
```

[Source: linux/drivers/gpu/drm/tiny/simpledrm.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/tiny/simpledrm.c)

Display pipe operations are guarded with `drm_dev_enter()/drm_dev_exit()` so that if a native driver triggers aperture removal while simpledrm is actively blitting a framebuffer, the operation returns safely rather than causing a use-after-free on the `ioremap`'d memory.

### DRM Capabilities

simpledrm provides:

- One `drm_connector` with a single fixed mode derived from the firmware framebuffer geometry.
- `drm_gem_shmem_*` buffer objects for scanout; pixel data is software-blitted from GEM to the firmware framebuffer.
- `drm_fbdev_generic_setup()` for fbcon console emulation.
- Support for `DRM_FORMAT_*` pixel format negotiation based on the firmware framebuffer's pixel layout.

### simpledrm Pixel Format and Conversion

The firmware framebuffer may use a pixel format that does not match what userspace sends. simpledrm handles this through software conversion during the shadow-framebuffer blit. The driver queries the firmware pixel format from `screen_info` (on x86) or from the DT `format` property (on ARM) and registers the corresponding `DRM_FORMAT_*` identifier with the display pipe. If the format is one of the common ones (XRGB8888, RGB565, etc.), DRM's built-in format conversion handles the blit transparently.

This software conversion is a performance trade-off: simpledrm is never intended for sustained high-performance rendering. Its purpose is to provide a valid DRM device during the boot window — typically 5–30 seconds — before the native GPU driver loads. Once the native driver probes and displaces simpledrm, all rendering goes through the native driver's hardware-accelerated paths.

### The /chosen Node Scan for Apple Silicon

Apple Silicon machines running Linux (via the Asahi Linux project) present a particularly interesting case. The Apple bootloader (m1n1) initialises the display hardware in a fixed mode and exports the framebuffer in the DT `/chosen` node as a `"simple-framebuffer"` compatible node. The standard Linux OF platform bus does not enumerate `/chosen` children, so simpledrm added an `fs_initcall` that explicitly scans `/chosen`:

```c
/* drivers/gpu/drm/tiny/simpledrm.c — Apple Silicon /chosen scan (simplified) */
static int __init simpledrm_chosen_init(void)
{
    struct device_node *chosen, *np;

    chosen = of_find_node_by_path("/chosen");
    if (!chosen)
        return 0;

    for_each_child_of_node(chosen, np) {
        if (of_device_is_compatible(np, "simple-framebuffer"))
            of_platform_device_create(np, NULL, NULL);
    }
    of_node_put(chosen);
    return 0;
}
fs_initcall(simpledrm_chosen_init);
```

This ensures that Apple Silicon machines get a DRM console device early in boot, before the AGX GPU driver (covered in Chapter 73) completes its substantially more complex probe.

[Source: linux/drivers/gpu/drm/tiny/simpledrm.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/tiny/simpledrm.c)

---

## The Kernel DRM Driver Probe Race

### Timeline of Driver Binding

The displacement of simpledrm by a native GPU driver follows a deterministic ordering rooted in Linux's initcall and PCI enumeration sequence:

1. `device_initcall` level: `sysfb_init()` runs, registers `"simple-framebuffer"` platform device.
2. `fs_initcall` (simpledrm ARM path): scans `/chosen`, creates platform devices.
3. simpledrm probe: calls `devm_aperture_acquire()`, registers DRM device as `/dev/dri/card0`.
4. PCI bus enumeration: `pci_register_driver()` for i915, amdgpu, etc.
5. Native driver probe: at the **top** of probe, the driver calls the aperture removal API.

### Aperture Removal Sequence

```c
/* Example: top of i915_pci_probe() or amdgpu_pci_probe() — simplified */
ret = aperture_remove_conflicting_pci_devices(pdev, "i915");
if (ret)
    return ret;
/* simpledrm has been unloaded; proceed with hardware init */
```

[Source: linux/drivers/video/aperture.c](https://elixir.bootlin.com/linux/latest/source/drivers/video/aperture.c)
[Source: Documentation/driver-api/aperture.rst](https://elixir.bootlin.com/linux/latest/source/Documentation/driver-api/aperture.rst)

The full sequence of `aperture_remove_conflicting_pci_devices()`:

1. Looks up registered apertures overlapping the PCI device's BAR.
2. Calls `platform_device_unregister()` on the `"simple-framebuffer"` platform device.
3. simpledrm's device-managed resources are freed; the aperture detach callback calls `drm_dev_unplug()`.
4. simpledrm's DRM device becomes unreachable; any open file descriptors see `ENODEV` on the next ioctl.
5. `drmfb` console output switches: `fb0: switching to i915drmfb from simpledrmdrmfb` appears in dmesg.
6. Native driver continues its probe, initialises hardware, reads EDID, performs first KMS modeset.

### Deferred Probe and Ordering

There is no explicit deferred-probe relationship between simpledrm and native GPU drivers. The displacement is purely aperture-based. If the native driver returns `-EPROBE_DEFER` (for example, because a required firmware file has not been loaded), simpledrm remains active; `driver_deferred_probe_trigger()` reschedules the native driver probe after any successful bind elsewhere in the system, eventually resolving the dependency.

[Source: linux/drivers/base/dd.c](https://elixir.bootlin.com/linux/latest/source/drivers/base/dd.c)

A subtle race: if the native driver probes so quickly that it runs before `sysfb_init()`, `sysfb_disable()` guards this case. The native driver calls `sysfb_disable()` which sets a `disabled` flag under a mutex; `sysfb_init()` checks the flag and skips device creation.

---

## KMS Atomic Modesetting at First Modeset

### Initialization Sequence

After the native driver displaces simpledrm, it proceeds through the DRM initialization sequence:

```c
/* Typical pattern in a KMS driver probe() */
drmm_mode_config_init(drm_dev);   /* sets up drm_device.mode_config */

/* Register encoder, connector, CRTC objects */
my_encoder_init(drm_dev);
my_connector_init(drm_dev);
my_crtc_init(drm_dev);

drm_mode_config_reset(drm_dev);   /* call ->reset callbacks; set default HW/SW state */

/* Set mode_config bounds, fb_formats, etc. */
drm_mode_config_validate(drm_dev);

/* Trigger first modeset */
drm_fbdev_generic_setup(drm_dev, 0);   /* fbdev emulation path */
/* OR: compositor will call DRM_IOCTL_MODE_ATOMIC directly */
```

[Source: linux/drivers/gpu/drm/drm_modeset_helper.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_modeset_helper.c)

### Mode Selection Heuristics

The `drm_client_modeset_probe()` function in `drivers/gpu/drm/drm_client_modeset.c` selects the initial display mode using a prioritised cascade:

[Source: linux/drivers/gpu/drm/drm_client_modeset.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_client_modeset.c)

```c
/* Simplified from drm_client_modeset_probe() */

/* 1. Kernel command line: highest priority */
mode = drm_connector_pick_cmdline_mode(connector);

/* 2. EDID preferred mode (native panel resolution) */
if (!mode)
    mode = drm_connector_preferred_mode(connector, width, height);

/* 3. Tiled display mode */
if (!mode)
    mode = drm_connector_get_tiled_mode(connector);

/* 4. First available in mode list */
if (!mode && !list_empty(&connector->modes))
    mode = list_first_entry(&connector->modes,
                            struct drm_display_mode, head);

/* 5. Ultimate fallback: 800×600 */
```

`drm_client_firmware_config()` inspects the atomic state to detect if the firmware left the display in a valid configuration, potentially skipping a redundant modeset and avoiding a flicker.

### EDID Read Timing

The EDID read happens inside `drm_helper_probe_single_connector_modes()` → driver's `.get_modes()` callback → `drm_get_edid()` → DDC I2C read. On the first modeset this is triggered either by `drm_fb_helper_initial_config()` (fbdev path) or the first userspace `DRM_IOCTL_MODE_GETCONNECTOR` ioctl (compositor path).

A practical issue: on eDP panels, the connector probe may run before the panel's HPD (Hot Plug Detect) line has asserted. If the EDID read fails, the driver must fall back to panel `drm_display_info` properties or hardcoded display timings from DT `display-timings`. This is handled by the `hpd_absent` timeout (the T3-max value from the eDP specification) in `drivers/gpu/drm/panel/panel-edp.c`.

[Source: linux/drivers/gpu/drm/panel/panel-edp.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/panel/panel-edp.c)

### Atomic Resume Pattern

The same infrastructure handles suspend/resume, which is structurally identical to the first-boot modeset:

```c
/* drivers/gpu/drm/drm_atomic_helper.c */
drm_mode_config_reset(dev);            /* reset hardware and software state */
drm_atomic_helper_resume(dev, state); /* restore saved atomic state */
```

[Source: linux/drivers/gpu/drm/drm_atomic_helper.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/drm_atomic_helper.c)

---

## Plymouth Boot Splash

### Architecture

Plymouth ([freedesktop.org/wiki/Software/Plymouth/](https://freedesktop.org/wiki/Software/Plymouth/)) is a boot splash daemon included in the initramfs. It starts early — before most kernel modules load, immediately after the initrd root pivot — and displays an animated splash screen while the system boots.

Plymouth uses a plugin architecture: renderer backends are shared libraries loaded from `/usr/lib/x86_64-linux-gnu/plymouth/renderers/` (Debian/Ubuntu) or `/usr/lib64/plymouth/renderers/` (Fedora). Three backends exist:

- `drm.so` — primary; uses KMS via direct DRM ioctls on `/dev/dri/card0`.
- `frame-buffer.so` — fallback; uses `/dev/fb0` directly.
- `x11.so` — debug only.

The DRM renderer is preferred when `/dev/dri/card0` exists (i.e., when simpledrm or a native driver has already registered a DRM device). If the card device is not yet available, Plymouth falls back to `frame-buffer.so`.

[Source: Plymouth source — src/plugins/renderers/drm/plugin.c](https://gitlab.freedesktop.org/plymouth/plymouth/-/blob/main/src/plugins/renderers/drm/plugin.c)

### DRM Renderer Internals

The DRM renderer opens `/dev/dri/card0` with `O_RDWR`, then uses legacy KMS ioctls to enumerate and configure CRTCs:

```c
/* src/plugins/renderers/drm/plugin.c — key structs (simplified) */
struct _ply_renderer_backend {
    int device_fd;             /* fd to /dev/dri/card0 */
    char *device_name;         /* defaults to "/dev/dri/card0" */
    drmModeRes *resources;
    ply_list_t *heads;
};

struct _ply_renderer_head {
    uint32_t controller_id;         /* CRTC id */
    uint32_t scan_out_buffer_id;    /* dumb buffer id */
    ply_pixel_buffer_t *pixel_buffer;
    ply_array_t *connector_ids;
};
```

The modeset call:

```c
/* src/plugins/renderers/drm/plugin.c */
drmModeSetCrtc(device_fd, controller_id, buffer_id, 0, 0,
               connector_ids, connector_count, mode_info);
```

Plymouth uses legacy `drmModeSetCrtc()` rather than atomic commits. This is intentional: Plymouth targets a minimal ABI available on any DRM driver, including simpledrm and efifb-backed devices that do not implement the full atomic modesetting interface.

Plymouth resets the hardware gamma LUT to linear before the first `drmModeSetCrtc()` call, correcting any non-linear gamma left by the firmware. The gamma table memory is freed after this single operation.

### VT Switch and DRM Master Handoff

When the Wayland compositor starts, logind triggers a VT switch. Plymouth's `on_active_vt_changed` handler fires:

```c
/* src/plugins/renderers/drm/plugin.c */
static void on_active_vt_changed(ply_renderer_backend_t *backend)
{
    if (ply_terminal_is_active(backend->terminal))
        activate(backend);   /* drmSetMaster(backend->device_fd) */
    else
        deactivate(backend); /* drmDropMaster(backend->device_fd) */
}
```

After `drmDropMaster()`, the compositor acquires DRM master through logind's `TakeDevice` D-Bus method (`org.freedesktop.login1.Session.TakeDevice`). The compositor opens the DRM device through logind rather than directly, enabling logind to hold a secondary file descriptor and orchestrate the VT switch by passing fds without requiring the compositor to close and re-open the device. This eliminates a race window where neither Plymouth nor the compositor holds master.

---

## Boot-to-Compositor Handoff

### The Handoff Sequence in Detail

The full DRM master transfer between Plymouth and the compositor:

1. Plymouth's DRM renderer holds `DRM_IOCTL_SET_MASTER` (via `drmSetMaster()`) on `/dev/dri/card0` while it is the active VT client.
2. Compositor process starts; opens the DRM device via logind's `TakeDevice` (receives a pre-opened fd with master suspended by logind).
3. logind triggers VT switch to the compositor's VT.
4. Plymouth's VT switch callback fires: `deactivate()` → `drmDropMaster()`.
5. logind resumes master on the compositor's fd.
6. Compositor calls its first atomic commit: `DRM_IOCTL_MODE_ATOMIC` with `DRM_MODE_ATOMIC_ALLOW_MODESET`.
7. Compositor configures planes, scanout buffers, and gamma LUT.
8. The brief black-frame window between step 4 and step 7 is the only visual discontinuity.

### LVDS/eDP Panel Power Sequencing

On laptops and embedded systems with eDP or LVDS panels, the compositor (or the native driver during its first modeset) must respect the panel power sequencing specification. The `drm_panel` framework abstracts this:

```c
/* include/drm/drm_panel.h */
int drm_panel_prepare(struct drm_panel *panel);   /* power on hardware; wait HPD */
int drm_panel_enable(struct drm_panel *panel);    /* enable backlight */
int drm_panel_disable(struct drm_panel *panel);   /* disable backlight */
int drm_panel_unprepare(struct drm_panel *panel); /* power off */
```

[Source: linux/include/drm/drm_panel.h](https://elixir.bootlin.com/linux/latest/source/include/drm/drm_panel.h)

The display controller driver calls `drm_panel_prepare()` before starting video transmission and `drm_panel_enable()` after the link is stable. The `pwm-backlight` driver (`drivers/video/backlight/pwm_bl.c`) supports a `boot_off` platform data field that prevents the backlight from enabling itself at probe time, deferring enable to the DRM panel framework. Without this field, the backlight illuminates before the panel has completed power sequencing, causing either a white flash or a corrupted image.

[Source: linux/drivers/video/backlight/pwm_bl.c](https://elixir.bootlin.com/linux/latest/source/drivers/video/backlight/pwm_bl.c)

### Plane State and Blank/Unblank

When the compositor performs its first atomic commit after acquiring DRM master, it submits a complete plane state including:

- CRTC mode (from EDID preferred mode or a user-selected mode).
- Primary plane framebuffer (compositor's scanout buffer, typically a `GBM_BO_USE_SCANOUT` GEM object).
- Overlay plane assignments if hardware composition is available.
- Gamma LUT (usually a linear table unless a colour profile is active).
- Connector properties (DPMS state: `DRM_MODE_DPMS_ON`).

If the atomic commit is submitted with `DRM_MODE_ATOMIC_ALLOW_MODESET`, the kernel may temporarily blank the display during the modeset operation. Compositors set up their initial framebuffer before this commit to minimise the blank window.

---

## Secure Boot and Early Graphics

### The UEFI Secure Boot Chain

On systems with Secure Boot enabled, every binary executed by UEFI firmware must be signed by a key in the firmware's authorized database (db):

```
UEFI firmware (OEM key in PK/KEK)
  → shim.efi (signed by Microsoft CA — "Microsoft Windows UEFI Driver Publisher")
      → grub.efi (signed by distro CA, enrolled via shim)
          → vmlinuz (signed by distro CA or user MOK)
```

[Source: rhboot/shim project](https://github.com/rhboot/shim)

Shim ≥ 15.3 enforces SBAT (Secure Boot Advanced Targeting). Each binary in the chain must carry a `.sbat` section with CSV generation metadata. If any component's SBAT generation is revoked (via UEFI `dbx` or shim's own SBAT revocation), shim refuses execution. This mechanism was deployed in response to BootHole (CVE-2020-10713).

From a graphics perspective: throughout this chain, the UEFI GOP framebuffer remains active. The shim, GRUB, and the kernel EFI stub all run in EFI Boot Services context and can invoke GOP `Blt()` or write directly to the GOP framebuffer. GRUB's graphical theme system uses GOP framebuffer blitting to display the boot menu.

### NVIDIA Open GPU Modules and Secure Boot Signing

When Secure Boot is enabled, `nvidia-open-gpu-kernel-modules` (the open-source NVIDIA kernel driver) must be signed with a key enrolled in the firmware's MOK (Machine Owner Key) database:

```bash
# Enroll a MOK key
sudo mokutil --import /etc/dkms/mok.pub

# Verify signing
modinfo nvidia | grep signer
```

DKMS systems on Debian/Ubuntu/Fedora automate this via `/etc/dkms/nvidia.conf`, which points to a MOK key+cert pair generated during driver installation. If signing fails or the key is not enrolled, the kernel refuses to load the NVIDIA module on Secure Boot systems, and the display falls back to simpledrm (on platforms where the EFI GOP framebuffer is visible at that address) or nouveau.

[Source: NVIDIA open-gpu-kernel-modules GitHub](https://github.com/NVIDIA/open-gpu-kernel-modules)

### efistub and console=ttyS0

When booting directly via the kernel's EFI stub (`efistub`) without shim or GRUB, the kernel receives control directly from firmware:

- With default boot parameters: `screen_info` is populated by the EFI stub GOP enumeration; the normal simpledrm/efifb path applies.
- With `console=ttyS0`: kernel console output is redirected to the serial port. No fbdev or DRM console is initialised unless a separate `video=` parameter is present. This path is used heavily in CI/embedded environments where a display is not required.
- With `console=ttyS0 video=efifb:mode=0`: forces a specific GOP mode before console initialisation.

Note: As of 2025, embedding a `.sbat` section into the kernel EFI zboot image (for direct efistub Secure Boot without shim) was an active RFC area. Deployments requiring both Secure Boot and direct efistub boot without shim should verify the current status of these patches in the upstream kernel.

---

## Embedded Considerations: U-Boot and Panel Sequencing

### U-Boot Video Framework

U-Boot provides its own display stack for embedded platforms. The driver model video layer (`CONFIG_DM_VIDEO`) supports a range of SoC display controllers (TI, Rockchip, Allwinner, i.MX, etc.). At U-Boot's startup it can:

- Display a splash screen loaded from storage or embedded in U-Boot's binary.
- Set a panel mode using the connected panel's timing data from DT.
- Remain in the configured mode until Linux takes over.

Relevant Kconfig options:

| Option | Purpose |
|---|---|
| `CONFIG_DM_VIDEO` | Driver model video layer |
| `CONFIG_SPLASH_SCREEN` | Boot splash loading support |
| `CONFIG_VIDEO_BMP_GZIP` | Compressed BMP splash support |
| `CONFIG_BMP_32BPP` / `CONFIG_BMP_24BPP` | Pixel depth for splash BMP |
| `CONFIG_VIDEO_LOGO` | Embed U-Boot logo |
| `CONFIG_VIDEO_TIDSS` | TI display subsystem driver |
| `CONFIG_VIDEO_ROCKCHIP` | Rockchip display driver |

### DT Framebuffer Handoff from U-Boot to Linux

U-Boot dynamically patches the Linux kernel device tree before executing `booti`. It writes `width`, `height`, `stride`, `format`, and `reg` into a `simple-framebuffer` DT node and sets `status = "okay"`:

```dts
/* DT after U-Boot patching, before Linux kernel receives it */
reserved-memory {
    framebuffer: framebuffer@ff700000 {
        reg = <0x00 0xff700000 0x00 0x008ca000>;
        no-map;
    };
};

framebuffer0: framebuffer@ff700000 {
    compatible = "simple-framebuffer";
    reg = <0x00 0xff700000 0x00 0x008ca000>;
    width = <1920>;
    height = <1080>;
    stride = <7680>;
    format = "x8r8g8b8";
    status = "okay";
};
```

The `no-map` reserved memory node prevents Linux from using that physical region for general allocation. Linux's simplefb or simpledrm picks up the node and maps it without needing to know how U-Boot initialised the hardware.

[Source: U-Boot Documentation — Framebuffer Handoff](https://docs.u-boot.org/en/latest/develop/bootstd/overview.html)

### panel-simple and PWM Backlight

`drivers/gpu/drm/panel/panel-simple.c` covers hundreds of LCD panels via DT `compatible` strings. Each panel entry specifies the timing, power-on delay, and backlight coupling:

[Source: linux/drivers/gpu/drm/panel/panel-simple.c](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/panel/panel-simple.c)

The PWM backlight driver (`drivers/video/backlight/pwm_bl.c`) is the generic backlight interface for panels driven by a PWM signal. Its DT node:

```dts
backlight: backlight {
    compatible = "pwm-backlight";
    pwms = <&pwm0 0 5000000>;
    brightness-levels = <0 4 8 16 32 64 128 255>;
    default-brightness-level = <6>;
    /* power-supply = <&vdd_bl>; */
};
```

The `boot_off` platform data field (or the absence of a `default-brightness-level` property in combination with specific driver logic) defers backlight enable to the `drm_panel_enable()` call rather than `probe()`. This prevents the white-flash problem on eDP panels where the backlight activates before the display link is trained.

---

## Raspberry Pi Boot Graphics

### VideoCore Firmware Initialisation

The Raspberry Pi 4 and 5 boot sequence is controlled by the VideoCore firmware blob (`start4.elf` on Pi 4, `start.elf` + `fixup.dat` on earlier models), loaded by the first-stage bootloader from SD card or SPI EEPROM. The VideoCore initialises the HDMI PHY, reads EDID from the connected display via DDC, selects a video mode (governed by `config.txt` settings), and begins driving the display before the ARM CPU is released from reset.

[Source: Raspberry Pi Documentation — config.txt](https://www.raspberrypi.com/documentation/computers/config_txt.html)

The key `config.txt` knobs for display:

```ini
display_auto_detect=1    # Auto-detect display type (DSI, HDMI, etc.)
dtoverlay=vc4-kms-v3d    # Load VC4 KMS DRM driver overlay (recommended)
# Historical alternative (deprecated):
# dtoverlay=vc4-fkms-v3d  # "fake" KMS: delegated modesetting to firmware
```

### simpledrm Before VC4

The Raspberry Pi boot display sequence under `vc4-kms-v3d`:

1. Firmware (`start4.elf`) initialises HDMI and drives display from VideoCore's framebuffer.
2. Kernel boots; the `/chosen` DT node contains a `simple-framebuffer` entry describing the VideoCore framebuffer.
3. simpledrm's `fs_initcall` scans `/chosen`, creates a platform device, probes, and registers `/dev/dri/card0`. fbcon uses simpledrm for early kernel messages.
4. The VC4 DRM driver (`drivers/gpu/drm/vc4/`) probes via the platform bus (bound to the `vc4-drm` DT node).
5. VC4 probe calls `aperture_remove_conflicting_devices()` to unregister the simpledrm device.
6. dmesg: `fb0: switching to vc4 from simpledrm`.
7. VC4 performs a full KMS modeset, reads EDID from HDMI DDC, and selects the native mode.

[Source: linux/drivers/gpu/drm/vc4/](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/vc4)

### fkms vs. kms

The historical `vc4-fkms-v3d` overlay ("fake KMS") used a firmware mailbox to delegate modesetting to the VideoCore. The Linux kernel spoke a simplified KMS interface backed by firmware calls. This avoided the complexity of implementing a full KMS driver but meant that certain DRM features (atomic modesetting, hardware overlays, correct EDID handling) were limited. The `vc4-kms-v3d` overlay implements genuine kernel-side KMS and is the recommended overlay as of Raspberry Pi OS Bookworm.

---

## Debugging Boot Display Issues

### Kernel Parameters

| Parameter | Effect |
|---|---|
| `drm.debug=0x4` | Enable KMS debug messages (`DRM_UT_KMS`) |
| `drm.debug=0x1f` | Enable all DRM debug categories (very verbose) |
| `drm.debug=0x40` | Enable atomic state dump (`DRM_UT_STATE`, kernel ≥5.13) |
| `efi=debug` | Verbose EFI boot output including GOP mode selection |
| `video=efifb:list` | List available GOP modes at boot |
| `video=efifb:mode=N` | Force a specific GOP mode |
| `console=ttyS0,115200` | Redirect console to serial; no video driver loaded |
| `nomodeset` | Disable all KMS drivers; fall back to efifb/simplefb only |

The `drm.debug` bitmask:

```
0x001  CORE      — DRM core operations
0x002  DRIVER    — driver-specific
0x004  KMS       — modesetting
0x008  PRIME     — DMA-BUF/PRIME
0x010  ATOMIC    — atomic commit paths
0x020  VBL       — vblank events
0x040  STATE     — atomic state dumps
0x080  LEASE     — DRM lease
0x100  DP        — DisplayPort
0x200  DRMRES    — device-managed resources
```

### Runtime Debugging

```bash
# Enable KMS debug logging on a running system
echo 0x4 | sudo tee /sys/module/drm/parameters/debug

# Filter dmesg for boot display activity
dmesg | grep -iE 'drm|simpledrm|efifb|simplefb|kms|aperture|efi-framebuffer'

# Dump current atomic state (requires CONFIG_DRM_DEBUG_DP_MST_TOPOLOGY_REFS)
cat /sys/kernel/debug/dri/0/state

# List active DRM framebuffer objects
cat /sys/kernel/debug/dri/0/framebuffers

# List all DRM devices
ls /sys/kernel/debug/dri/

# Check which driver owns the primary display DRM device
cat /sys/class/drm/card0/device/driver/module/srcversion

# Verify screen_info was populated by EFI GOP
dmesg | grep -i 'efifb\|EFI v\|Video: '

# Check aperture registration
cat /proc/iomem | grep -i framebuffer
```

### Plymouth Debugging

```bash
# Enable Plymouth debug logging
plymouth --debug

# View Plymouth debug log
cat /var/log/plymouth/debug.log

# Force Plymouth to use the framebuffer renderer
plymouth --set-splash-plugin=details

# Check which renderer Plymouth selected
grep -i renderer /var/log/plymouth/debug.log
```

### Common Failure Modes and Diagnostics

**Black screen after native driver probe:**
- Check `dmesg` for `aperture_remove_conflicting_pci_devices` — if absent, simpledrm was never displaced; native driver may have failed probe.
- Check for firmware signature errors: `dmesg | grep -i 'firmware\|sign\|rejected'`.
- `drm.debug=0x1f` combined with `efi=debug` provides the most comprehensive output.

**simpledrm persists after expected native driver probe:**
- Native driver may have returned `-EPROBE_DEFER` (firmware not loaded) or `-EINVAL` (missing device tree node).
- `dmesg | grep -i 'probe\|defer'` reveals deferred-probe cycles.

**Wrong resolution at boot:**
- `video=1920x1080@60` on the kernel command line overrides mode selection.
- EDID override: `drm_kms_helper.edid_firmware=HDMI-A-1:edid/1920x1080.bin` loads a pre-built EDID blob for displays that do not report a valid EDID.

**Plymouth fails to use DRM renderer:**
- `/dev/dri/card0` must exist when Plymouth starts; if simpledrm or efifb is not loaded (e.g., `nomodeset` on kernel command line), Plymouth falls back to `frame-buffer.so`.
- Plymouth requires DRM master at startup; if another process holds master, Plymouth's `drmSetMaster()` call fails silently and it falls back to the framebuffer renderer.

**eDP panel does not display during early boot:**
- Panel power sequencing may not be respected by simplefb/simpledrm (which inherit whatever state the bootloader left).
- Check if the native DRM panel driver is asserting HPD correctly: `drm.debug=0x4` and look for `drm_panel_prepare` and HPD timeout messages.

### Analysing the Boot Display Timeline with ftrace

For deep investigation of race conditions between driver probes, ftrace function tracing is invaluable:

```bash
# Trace the specific functions involved in the handoff
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'aperture_remove*:sysfb_disable:simpledrm_aperture*:drm_dev_unplug' \
    > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... trigger probe (or analyse trace from boot via trace_buf_size= and /proc/cmdline)
cat /sys/kernel/debug/tracing/trace | head -100
```

Alternatively, set `trace_buf_size=64M` and `ftrace=function` on the kernel command line to capture the entire boot sequence trace. The `trace_printk()` annotations in `drivers/video/aperture.c` and `drivers/firmware/sysfb.c` produce human-readable events in the trace output.

### Interpreting dmesg for Boot Display

A healthy x86 UEFI boot sequence produces dmesg entries in this approximate order:

```
[    0.000000] EFI v2.80 by AMI
[    0.453021] efifb: probing for EFI framebuffer
[    0.453025] efifb: framebuffer at 0x...
[    0.453030] efifb: mode is 1920x1080x32, linelength=7680
[    ...      ] ...
[    1.234567] simple-framebuffer simple-framebuffer.0: 1920x1080@32 framebuffer
[    1.234580] Console: switching to colour frame buffer device 240x67
[    1.234600] simpledrm simpledrm: [drm] Initialized simpledrm 1.0.0 20200625 ...
[    1.234605] simpledrm simpledrm: [drm] 1 display pipe(s)
[    ...      ] (native driver probe begins)
[    3.456789] i915 0000:00:02.0: [drm] aperture: removed conflicting framebuffer(s)
[    3.456800] fb0: switching to i915drmfb from simpledrmdrmfb
[    3.456900] i915 0000:00:02.0: [drm] Initialized i915 1.6.0 20201103 for 0000:00:02.0
```

The timestamp gap between the `simpledrm` initialization line and the `i915` aperture removal line is the window during which Plymouth and early console are served by simpledrm. On fast systems this is under 2 seconds; on systems that require firmware loading for the GPU, it can be 10–20 seconds.

---

## Integrations

**Chapter 1 — DRM Architecture and the Driver Model**
simpledrm is the minimal reference implementation of the Linux DRM driver model. It uses `drm_driver`, `drm_simple_display_pipe`, and `drm_gem_shmem_*` — exactly the interfaces described in Chapter 1. The aperture API (`devm_aperture_acquire`, `aperture_remove_conflicting_pci_devices`) is part of the DRM subsystem's device lifecycle framework and directly implements the device-model patterns discussed there.

**Chapter 2 — KMS and the Display Pipeline**
Boot is the first KMS client. `drm_client_modeset_probe()`, `drm_fb_helper_initial_config()`, and `drm_mode_config_reset()` are the boot-time entry points into the KMS subsystem. The atomic modesetting path described in Chapter 2 is exercised at the very first modeset after native driver probe. The EDID read that seeds mode selection flows through the connector probe path covered in Chapter 2.

**Chapter 3 — Advanced Display Features**
Panel power sequencing (`drm_panel_prepare`, `drm_panel_enable`) and eDP HPD handling are part of the advanced display infrastructure described in Chapter 3. The VT switch mechanism via which Plymouth hands off to the compositor is mediated by logind's `TakeDevice`/`PauseDevice` D-Bus interface, which itself relies on the DRM master and VT infrastructure.

**Chapter 5 — x86 GPU Drivers: i915 and amdgpu**
Both i915 and amdgpu call `aperture_remove_conflicting_pci_devices()` at the top of their probe functions to displace simpledrm/efifb. Chapter 5 covers the probe sequence in detail; this chapter provides the simpledrm and aperture-API context that explains why the displacement API call is necessary and what it does.

**Chapter 92 — The Raspberry Pi GPU Stack: VideoCore and V3D**
The VC4 driver's probe sequence (DT-driven, platform bus, `aperture_remove_conflicting_devices()`) is the embedded analogue of the PCI-based i915/amdgpu takeover described here. Chapter 92 describes the VC4 DRM driver in depth; this chapter provides the boot-side context (simpledrm `/chosen` scan, fkms vs. kms distinction, `config.txt` display settings).

**Chapter 129 — GPU Firmware Deep Dive**
The native GPU driver's probe depends on firmware loading (`request_firmware` API, `/lib/firmware/`, initramfs inclusion). If firmware is unavailable at probe time, the driver returns `-EPROBE_DEFER`, keeping simpledrm active longer than expected. Chapter 129 covers the firmware loading path that this chapter's deferred-probe discussion presupposes.

**Chapter 122 — DKMS and Out-of-Tree GPU Kernel Modules**
Secure Boot signing of out-of-tree modules (including NVIDIA's open-source driver) is managed by DKMS with MOK signing. Chapter 122 covers the DKMS build system; this chapter covers the Secure Boot chain and MOK enrollment that determine whether a DKMS-built module can load on a Secure Boot system.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
