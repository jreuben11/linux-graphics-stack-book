# Chapter 106: Multi-GPU Systems — PRIME Offloading and Vulkan Device Groups

> **Part**: Part II — GPU Drivers
> **Audiences**: Laptop users and developers with hybrid GPU (iGPU + dGPU); workstation engineers managing multiple discrete GPUs; Vulkan application developers targeting multi-GPU rendering and cross-device memory sharing.
> **Status**: First draft — 2026-06-19

---

## Table of Contents

- [1. Introduction: The Multi-GPU Landscape on Linux](#1-introduction-the-multi-gpu-landscape-on-linux)
- [2. PRIME Architecture](#2-prime-architecture)
- [3. iGPU+dGPU Topology Detection](#3-igpudgpu-topology-detection)
- [4. Reverse PRIME and PRIME Display Offload](#4-reverse-prime-and-prime-display-offload)
- [5. PRIME Render Offload](#5-prime-render-offload)
- [6. NVIDIA-Specific Considerations](#6-nvidia-specific-considerations)
- [7. Vulkan Multi-Device with Device Groups](#7-vulkan-multi-device-with-device-groups)
- [8. Peer-to-Peer GPU Memory](#8-peer-to-peer-gpu-memory)
- [9. Power Management for the dGPU](#9-power-management-for-the-dgpu)
- [10. Practical Configurations](#10-practical-configurations)
- [Integrations](#integrations)
- [References](#references)

---

## 1. Introduction: The Multi-GPU Landscape on Linux

Modern Linux systems routinely host two or more GPUs that must collaborate without being designed to do so. On the majority of laptops shipped since 2013, an integrated GPU (iGPU) embedded in the CPU die coexists with a discrete GPU (dGPU) connected via PCIe. The iGPU — driven by `i915`, `xe`, or `amdgpu` — is optimised for battery life and typically wired directly to the internal display and external connectors. The dGPU — driven by `amdgpu`, `nouveau`, or `nvidia` — carries fast GDDR VRAM, a wide shader array, and high-bandwidth memory controllers, but receives power from the motherboard's PCIe slot and therefore consumes several watts even at idle.

The fundamental difficulty is that the kernel treats each GPU as an entirely independent DRM device. There is no shared memory address space, no common fence domain, and no automatic mechanism to move a rendered frame from dGPU VRAM to the iGPU's display engine. At the same time, on the majority of laptop designs — the "MUXless" designs discussed in Section 6 — the display panel and external connectors are hardwired to the iGPU, so every frame the dGPU renders must eventually appear in iGPU-accessible memory before the display controller can scan it out.

Desktops and workstations face a different multi-GPU problem: a machine may host two or more discrete GPUs for parallel machine-learning training, multi-stream video transcoding, or split-screen rendering. In these cases, the GPUs may be from different vendors (a common ML configuration pairs an NVIDIA compute GPU with an AMD display GPU), or all from the same vendor but in a PCIe topology with no hardware interconnect.

The Linux answer to both problems is a two-layer solution. At the kernel level, the **PRIME** buffer-sharing framework uses the `dma_buf` subsystem to move GEM buffer handles across driver boundaries as UNIX file descriptors, enabling zero-copy (when IOMMU and PCIe topology permit) or DMA-assisted copies across devices. At the Vulkan API level, **device groups** (`VK_KHR_device_group`, core since Vulkan 1.1) allow a single logical `VkDevice` to span multiple physical devices and direct subsets of command buffers, memory allocations, and swapchain presentations to specific hardware.

This chapter traces the full stack: from the physical topology visible in sysfs, through the PRIME DRM API and its interaction with DRI3 and Wayland compositors, through NVIDIA's proprietary extensions and the open NVK driver, to the Vulkan device-group API and the peer-to-peer memory extensions that make cross-device zero-copy feasible. Power management — keeping the dGPU asleep until needed — closes the practical discussion.

Readers already familiar with basic DRM memory management (GEM, GBM, DMA-BUF) should consult Chapter 4 before this chapter; the DMA-BUF synchronisation primitives (`dma_fence`, `dma_resv`) are introduced there and assumed here.

---

## 2. PRIME Architecture

### 2.1 Origin and Purpose

PRIME — Peripheral Rendering Infrastructure for Multiple Engines — entered the Linux kernel in 2012 for DRM and became the standard cross-GPU buffer-sharing mechanism with kernel 3.12 (2013). [Source: DRM PRIME documentation — freedesktop.org](https://dri.freedesktop.org/docs/drm/gpu/drm-mm.html) Its design is elegant: rather than inventing a new shared-memory protocol between GPU drivers, PRIME reuses the existing `dma_buf` subsystem (introduced in 3.3) to export GEM buffer objects as UNIX file descriptors that cross driver and process boundaries.

PRIME defines two operational modes whose terminology has evolved:

- **PRIME display offload** (sometimes called "reverse PRIME"): the dGPU renders a frame, exports it as a `dma_buf`, and the iGPU imports the buffer and presents it through its KMS display engine. The copy or DMA transfer runs at each frame boundary.
- **PRIME render offload** (introduced with xrandr 1.6 / X.Org Server 1.20 in 2018): the X server or Wayland compositor remains on the iGPU for display, but individual applications are directed to render on the dGPU via per-process environment variables. The DRI3 protocol shuttles the resulting `dma_buf` fds from the application's dGPU context to the compositor's iGPU context.

Both modes are built on the same kernel primitives, described next.

### 2.2 The DRM PRIME API

The public PRIME interface exposed to userspace (via libdrm) consists of two ioctl wrappers:

```c
/* libdrm/xf86drm.h — PRIME buffer sharing API */

/**
 * drmPrimeHandleToFD - export a GEM handle as a dma-buf file descriptor
 * @fd:       open file descriptor of the DRM device that owns the GEM object
 * @handle:   GEM handle on that device
 * @flags:    DRM_CLOEXEC and/or DRM_RDWR
 * @prime_fd: output — the dma-buf file descriptor
 * Returns 0 on success, -1 on error (errno set).
 */
int drmPrimeHandleToFD(int fd, uint32_t handle, uint32_t flags, int *prime_fd);

/**
 * drmPrimeFDToHandle - import a dma-buf file descriptor as a GEM handle
 * @fd:       open file descriptor of the importing DRM device
 * @prime_fd: dma-buf file descriptor to import
 * @handle:   output — the new GEM handle on the importing device
 * Returns 0 on success, -1 on error (errno set).
 */
int drmPrimeFDToHandle(int fd, int prime_fd, uint32_t *handle);
```

[Source: libdrm xf86drm.h — grate-driver/libdrm on GitHub](https://github.com/grate-driver/libdrm/blob/master/xf86drm.h)

Internally, `drmPrimeHandleToFD` issues `DRM_IOCTL_PRIME_HANDLE_TO_FD` which calls the driver's `gem_prime_export` callback (default: `drm_gem_prime_export()`). The kernel wraps the GEM object's `sg_table` inside a `dma_buf` and returns a new anon-file fd. `drmPrimeFDToHandle` issues `DRM_IOCTL_PRIME_FD_TO_HANDLE`, calling `drm_gem_prime_import()` on the importing device. If the importing driver recognises the `dma_buf` as its own (same driver, same device), it returns a second handle to the existing GEM object without a copy. If the buffer is foreign, it calls `dma_buf_attach()` + `dma_buf_map_attachment()` to map the scatter-gather list into the importing GPU's IOMMU domain; a physical DMA transfer may be required.

### 2.3 The DMA-BUF Subsystem

The `dma_buf` kernel object is the backbone of PRIME. It consists of three layers [Source: Linux kernel DMA-BUF documentation](https://docs.kernel.org/driver-api/dma-buf.html]:

1. **`dma_buf`**: wraps a scatter-gather memory table as an anon-file. Export is via `dma_buf_export()` + `dma_buf_fd()`; import/attach is via `dma_buf_get()` + `dma_buf_attach()` + `dma_buf_map_attachment()`.
2. **`dma_fence`**: a single-shot callback object (`dma_fence_init()`, `dma_fence_signal()`) that a GPU driver signals when its command stream has finished accessing the buffer. The fence propagates across the `dma_resv` reservation object attached to each `dma_buf`, ensuring the importer does not read a buffer while the exporter is still writing it.
3. **`dma_resv`**: manages a set of fences (one exclusive write fence, zero or more shared read fences) for each `dma_buf`. Drivers call `dma_resv_lock()` before modifying the fence set and `dma_resv_add_fence()` to attach a new completion fence.

CPU access to a shared buffer requires bracketing with `dma_buf_begin_cpu_access()` and `dma_buf_end_cpu_access()` to trigger cache coherency operations; from userspace this is the `DMA_BUF_IOCTL_SYNC` ioctl with `DMA_BUF_SYNC_START`/`DMA_BUF_SYNC_END`.

### 2.4 Which GPU Owns the Display Output

The connector–encoder–CRTC chain is fixed to one DRM device. On a typical muxless laptop, `card0` is the iGPU and owns connectors `eDP-1` (internal panel), `HDMI-A-1`, and `DP-1`. The `card1` dGPU has a DRM primary node but no connectors — its `drm_mode_config.num_connector` is zero. This asymmetry is why all PRIME paths ultimately send the rendered frame back to the iGPU for display.

---

## 3. iGPU+dGPU Topology Detection

### 3.1 The `/sys/class/drm/` Hierarchy

The kernel exposes DRM devices through sysfs. On a hybrid laptop:

```
/sys/class/drm/card0 -> ../../devices/pci0000:00/0000:00:02.0/drm/card0
/sys/class/drm/card1 -> ../../devices/pci0000:00/0000:00:01.0/0000:01:00.0/drm/card1
/dev/dri/card0       # iGPU primary node (with connectors)
/dev/dri/card1       # dGPU primary node (no connectors on muxless)
/dev/dri/renderD128  # iGPU render node
/dev/dri/renderD129  # dGPU render node
```

The render nodes (`/dev/dri/renderDN`) are the correct path for compute and rendering access without display privileges. Starting from Linux 3.12, applications must open the render node to perform off-screen rendering; the primary node is reserved for display engine control.

Note that card numbering is not stable across reboots — it reflects driver registration order, which depends on module init ordering. Since kernel 6.4, `CONFIG_SYSFB_SIMPLEFB` loads `simpledrm` before `amdgpu`, which can cause the iGPU to land at `card1` rather than `card0`. Do not hardcode card numbers; use PCI path symlinks:

```bash
# Identify GPU devices by PCI address (stable across reboots)
ls -la /dev/dri/by-path/
# pci-0000:00:02.0-card  -> ../card0   (Intel iGPU)
# pci-0000:01:00.0-card  -> ../card1   (dGPU)
# pci-0000:01:00.0-render -> ../renderD129

# Identify GPU drivers
lspci -k | grep -A3 "VGA\|3D controller"
```

### 3.2 The `DRI_PRIME` Environment Variable

Mesa's loader reads `DRI_PRIME` to select which DRM device a process uses for rendering. Two forms are accepted:

```bash
DRI_PRIME=1              # Use the non-default GPU (by index, starting at 1)
DRI_PRIME=pci-0000_01_00_0   # Explicit PCI address (replace colons with underscores)
```

The Mesa device-selection logic walks `/sys/class/drm/renderDN` and scores devices by (a) the `DRI_PRIME` value, (b) the `drm_driver` name, and (c) whether the device is marked "boot VGA" by the firmware. The device with `DRI_PRIME=0` (or without the variable) gets the iGPU; `DRI_PRIME=1` forces the dGPU's render node.

For Vulkan, the `VK_LAYER_MESA_device_select` implicit layer reads `MESA_VK_DEVICE_SELECT` (analogous to `DRI_PRIME`) and re-orders `vkEnumeratePhysicalDevices()` results so the desired GPU appears first.

### 3.3 Enumerating Providers with xrandr

Under X11, the RandR 1.4 *provider* abstraction exposes each DRM device as a provider with roles:

```bash
xrandr --listproviders
# Providers: number : 2
# Provider 0: id: 0x43 cap: 0x9, Source Output, Sink Offload  crtcs: 4 outputs: 3 clones: 0 name:modesetting
# Provider 1: id: 0x44 cap: 0x6, Sink Output, Source Offload  crtcs: 0 outputs: 0 clones: 0 name:NVIDIA-G0
```

- **Sink Output** / **Source Output**: provider can receive frames from another for display, or send frames to another.
- **Source Offload** / **Sink Offload**: provider can render frames for another (offload source) or receive offloaded frames (offload sink).

The iGPU (modesetting DDX) typically advertises `Sink Offload` because it is the display sink. The dGPU (NVIDIA or amdgpu DDX) is the `Source Offload` — it can render frames but needs the iGPU to display them.

### 3.4 Verifying Active GPU

```bash
# OpenGL: check renderer string
glxinfo | grep "OpenGL renderer"

# Vulkan: list all physical devices with DRI node
vulkaninfo --summary | grep "deviceName\|driverName"

# Verify render node in use (Mesa Gallium)
MESA_LOADER_DRIVER_OVERRIDE=radeonsi glxinfo -B  # force specific driver
GALLIUM_HUD=GPU-load glxgears                    # GPU load overlay
```

---

## 4. Reverse PRIME and PRIME Display Offload

### 4.1 The Classical "Reverse PRIME" Topology

The original PRIME use case — sometimes called "reverse PRIME" to distinguish it from render offload — predates the modern render-offload protocol. In reverse PRIME:

1. The dGPU renders the desktop at its native resolution into a GEM buffer in VRAM.
2. The dGPU driver exports that GEM buffer as a `dma_buf` fd.
3. The X server or compositor running on the iGPU imports the `dma_buf`, creating a shadow GEM object backed by the dGPU memory.
4. The iGPU uses a copy engine or CPU blit to transfer pixel data from the dGPU's VRAM to iGPU VRAM.
5. The iGPU's KMS display engine scans out the local copy.

This path is required when an external monitor is physically connected to the dGPU's display output on a MUXed machine where the dGPU heads are active (uncommon), or on older ThinkPad/Dell workstation models where the Thunderbolt controller routes through the dGPU.

### 4.2 xrandr Provider Setup

The RandR 1.4 `setprovideroutputsource` command wires two providers together:

```bash
# Tell the modesetting (iGPU) provider to scan out from the NVIDIA provider
xrandr --setprovideroutputsource NVIDIA-G0 modesetting
# Now activate the NVIDIA virtual output
xrandr --auto
```

This is distinct from `--setprovideroffloadsink`, which is used for render offload (Section 5).

### 4.3 Performance Cost of Frame Copies

The copy inherent in reverse PRIME consumes memory bandwidth proportional to resolution and refresh rate:

| Resolution | Refresh | BPP | Bandwidth (one-way) |
|------------|---------|-----|---------------------|
| 1920×1080  | 60 Hz   | 32  | ~0.5 GB/s           |
| 2560×1440  | 144 Hz  | 32  | ~2.1 GB/s           |
| 3840×2160  | 60 Hz   | 32  | ~2.0 GB/s           |
| 3840×2160  | 120 Hz  | 32  | ~4.0 GB/s           |

On a PCIe 3.0 x4 link (theoretical 8 GB/s, practical ~6 GB/s), a 4K@120 Hz copy consumes roughly 67% of the available PCIe bandwidth — leaving little headroom for render data flowing in the other direction.

When the `dma_buf` modifier (Section 5.3) is compatible between the dGPU exporter and the iGPU's DRM scanout engine, the iGPU can import the `dma_buf` directly as a KMS plane without any copy. This "zero-copy PRIME" path requires both GPUs to agree on tiling layout, often only possible with the `DRM_FORMAT_MOD_LINEAR` modifier or when both GPUs use the same vendor's tiling scheme.

### 4.4 The DRI2/DRI3 Copy Path

Under DRI3, the X server receives the rendered buffer via `DRI3PixmapFromBuffer` or `DRI3FenceFromFD`. The modesetting DDX contains a `dri3_prime_pixmap_from_fd` path that, when the fd arrives from a foreign device:
- calls `drmPrimeFDToHandle()` to create an iGPU GEM handle for the buffer,
- if a zero-copy import fails (incompatible modifier), schedules a blit via the iGPU's 2D engine or fallback CPU memcpy,
- then promotes the resulting pixmap to a KMS framebuffer with `DRM_IOCTL_MODE_ADDFB2`.

---

## 5. PRIME Render Offload

### 5.1 Architecture

PRIME render offload separates where an application renders from where the compositor displays. The compositor process remains on the iGPU (the default GPU) and owns the display pipeline. Individual applications set environment variables that redirect Mesa's or NVIDIA's GL/Vulkan driver to open the dGPU's render node instead.

The flow under DRI3 and a Wayland compositor:

1. Application opens `/dev/dri/renderD129` (dGPU render node) because `DRI_PRIME=1` is set.
2. Application renders into a GEM buffer on the dGPU.
3. Application calls `eglSwapBuffers()` or `wl_surface.commit`. The EGL platform (GBM or EGLStream) calls `drmPrimeHandleToFD()` to obtain a `dma_buf` fd for the rendered buffer.
4. The `dma_buf` fd is sent to the Wayland compositor via `linux-dmabuf` protocol (`zwp_linux_buffer_params_v1`).
5. The compositor, running on the iGPU, imports the `dma_buf` via `drmPrimeFDToHandle()` and attaches it to a KMS plane.
6. If the format modifier is compatible, the iGPU KMS engine scans the buffer directly from dGPU VRAM (zero-copy). Otherwise, a blit copy runs on the iGPU.

### 5.2 Environment Variables for Open-Source Drivers

```bash
# Render this application on the non-default GPU (Mesa: i915, amdgpu, nouveau, NVK)
DRI_PRIME=1 glxgears

# Select by explicit PCI address (robust against card numbering changes)
DRI_PRIME=pci-0000_01_00_0 vulkaninfo

# Debug: print the GPU selected and the render node path
DRI_PRIME=1 MESA_LOADER_DRIVER_DEBUG=1 glxinfo 2>&1 | grep -i "using\|dri\|prime"
```

For Vulkan applications under Mesa, `DRI_PRIME` works when `vulkan-mesa-layers` (providing `VK_LAYER_MESA_device_select`) is installed. Without the layer, `DRI_PRIME` has no effect on Vulkan device enumeration; the application must call `vkEnumeratePhysicalDevices()` and select the desired device by name or PCI ID.

### 5.3 Zero-Copy Path: Format Modifiers

The zero-copy path is gated on buffer format modifier compatibility. A format modifier describes the tiling layout of a buffer (e.g., Intel's Y-tiling `I915_FORMAT_MOD_Y_TILED`, AMD's DCC `AMD_FMT_MOD_DCC_RETILE_ALLOW`, or the universal `DRM_FORMAT_MOD_LINEAR`). The KMS driver accepts a framebuffer only if its modifier matches what the hardware scanout engine supports.

Checking compatibility:

```bash
# List modifiers supported by the iGPU KMS for a given format
modetest -D /dev/dri/card0 -f XRGB8888 2>&1 | grep modifier
```

When a Wayland compositor receives a `zwp_linux_buffer_params_v1` buffer with a dGPU-specific modifier that the iGPU KMS does not support, it falls back to importing into a texture and blitting via the iGPU's GPU. This blit path is synchronised using `dma_resv` fences; the compositor waits for the dGPU's render fence before issuing the blit.

### 5.4 Per-Application Configuration

Shipping a `.desktop` file with PRIME enabled:

```ini
[Desktop Entry]
Name=My GPU-Intensive App
Exec=env DRI_PRIME=1 /usr/bin/myapp
```

Under Steam, per-game PRIME is set in "Properties → Launch Options":
```
DRI_PRIME=1 %command%
```

Under Gamescope:
```bash
gamescope -W 1920 -H 1080 -- env DRI_PRIME=1 /usr/bin/game
```

---

## 6. NVIDIA-Specific Considerations

### 6.1 NVIDIA PRIME with the Proprietary Driver

NVIDIA's proprietary driver implements PRIME through a different set of environment variables:

```bash
# Enable PRIME render offload to NVIDIA GPU (GLX)
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxgears

# Vulkan and EGL only need the first variable
__NV_PRIME_RENDER_OFFLOAD=1 vkgears

# Select a specific GPU provider (when multiple NVIDIA GPUs are present)
__NV_PRIME_RENDER_OFFLOAD_PROVIDER=NVIDIA-G0 __NV_PRIME_RENDER_OFFLOAD=1 app

# Restrict Vulkan enumeration to NVIDIA only
__VK_LAYER_NV_optimus=NVIDIA_only vkcube
```

[Source: NVIDIA PRIME Render Offload documentation](https://download.nvidia.com/XFree86/Linux-x86_64/440.31/README/primerenderoffload.html)

The `__GLX_VENDOR_LIBRARY_NAME=nvidia` variable is required for GLX because NVIDIA uses the GLVND (GL Vendor-Neutral Dispatch) dispatch library. Without this override, the iGPU's libGL implementation would be loaded even though `__NV_PRIME_RENDER_OFFLOAD=1` selects the NVIDIA device. EGL and Vulkan applications do not need `__GLX_VENDOR_LIBRARY_NAME` because their dispatch paths are already vendor-neutral.

The X server must be configured to expose NVIDIA GPU screens:

```
# /etc/X11/xorg.conf.d/10-nvidia.conf
Section "ServerLayout"
    Identifier     "layout"
    Option "AllowNVIDIAGPUScreens"
EndSection
```

### 6.2 `nvidia-drm` and Modesetting

The `nvidia-drm` kernel module provides a DRM primary node for the NVIDIA GPU, enabling KMS and DMA-BUF interoperability even with the closed-source driver. In muxless configurations, `nvidia-drm` acts as a render-only DRM device (no connectors), and the modesetting DDX on the iGPU handles all display configuration. The `nvidia-drm.modeset=1` kernel parameter (or `options nvidia-drm modeset=1` in `/etc/modprobe.d/`) must be set to enable full KMS support for Wayland compositors.

```bash
# Verify nvidia-drm is in KMS mode
cat /sys/module/nvidia_drm/parameters/modeset  # should print Y
```

### 6.3 On-Demand Mode and D3cold

The `nvidia-prime` package (Debian/Ubuntu) and `nvidia-settings` expose three modes:
- **Intel/AMD** (iGPU only): `nvidia-settings --prime-select intel`. The NVIDIA GPU is powered off entirely via ACPI `_PS3`.
- **NVIDIA** (dGPU only, MUXed): `nvidia-settings --prime-select nvidia`. Full performance, highest battery drain.
- **on-demand**: `nvidia-settings --prime-select on-demand`. NVIDIA GPU enters D3cold when idle; any application setting `__NV_PRIME_RENDER_OFFLOAD=1` transparently powers it on, activates it, and renders on it.

The on-demand mode relies on PCI runtime PM (`/sys/bus/pci/devices/<addr>/power/control = "auto"`) and ACPI D3cold support (the firmware's `_PR3` power resource). Power-on latency from D3cold is typically 400–700 ms; `nvidia-drm` caches context state to minimise this on successive wakeups.

### 6.4 MUXed vs MUXless Laptops

A **MUXless** laptop routes the display panel's eDP link permanently to the iGPU. The dGPU never directly drives the display; all frames must transit through PRIME. This is the default design for thin-and-light laptops (ThinkPad X1, Dell XPS, most consumer gaming laptops).

A **MUXed** laptop includes a hardware multiplexer (a PCIe switch or dedicated MUX IC) that can route the eDP link to either GPU. Engaging "dGPU direct" mode bypasses PRIME entirely: the dGPU's display engine scans out directly, eliminating copy latency and PCIe bandwidth consumption. Linux support for MUX switching is vendor-specific: ASUS ROG laptops expose the MUX through an ACPI device accessible via `supergfxctl`; MSI laptops use `MSI-WMI-PLATFORM`; some Lenovo Legion models use a firmware BIOS toggle with no runtime OS switch.

The practical implication: for sustained 4K gaming with an external monitor on a MUXed laptop, enable "dGPU direct" in firmware to get zero-copy performance. For battery-optimal daily driving, MUXless + PRIME on-demand is preferable.

### 6.5 NVK and PRIME

The open-source NVK Vulkan driver (Ch8) — merged into Mesa mainline in 2023 and declared production-ready for Turing and later in 2024 — participates in PRIME through the standard Mesa `DRI_PRIME` mechanism, identically to `amdgpu` and `iris`. NVK implements `VK_EXT_image_drm_format_modifier` and `VK_EXT_queue_family_foreign`, enabling a Wayland compositor to import NVK-rendered `dma_buf` objects without a copy when modifiers are compatible. [Source: NVK Mesa documentation](https://docs.mesa3d.org/drivers/nvk.html)

---

## 7. Vulkan Multi-Device with Device Groups

### 7.1 Overview

The `VK_KHR_device_group` and `VK_KHR_device_group_creation` extensions (both promoted to Vulkan 1.1 core) allow applications to enumerate physical device groups and create a single logical `VkDevice` that spans all physical devices in a group. Device group rendering is only possible between GPUs that the driver exposes as a group — in practice, this means same-vendor GPUs connected by a high-speed interconnect (NVLink, Infinity Fabric, or PCIe with P2P DMA support).

[Source: VK_KHR_device_group_creation specification — Khronos](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_device_group_creation.html)

### 7.2 Enumerating Device Groups

Device groups are enumerated at the instance level:

```c
/* Enumerate all physical device groups */
uint32_t groupCount = 0;
vkEnumeratePhysicalDeviceGroups(instance, &groupCount, NULL);

VkPhysicalDeviceGroupProperties *groups =
    calloc(groupCount, sizeof(VkPhysicalDeviceGroupProperties));
for (uint32_t i = 0; i < groupCount; i++) {
    groups[i].sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES;
    groups[i].pNext = NULL;
}
vkEnumeratePhysicalDeviceGroups(instance, &groupCount, groups);

for (uint32_t i = 0; i < groupCount; i++) {
    printf("Group %u: %u device(s), subsetAllocation=%s\n",
        i,
        groups[i].physicalDeviceCount,
        groups[i].subsetAllocation ? "yes" : "no");
    for (uint32_t j = 0; j < groups[i].physicalDeviceCount; j++) {
        VkPhysicalDeviceProperties props;
        vkGetPhysicalDeviceProperties(groups[i].physicalDevices[j], &props);
        printf("  Device %u: %s\n", j, props.deviceName);
    }
}
```

`VkPhysicalDeviceGroupProperties` fields:
- `physicalDeviceCount`: number of physical devices in this group (≥1).
- `physicalDevices[VK_MAX_DEVICE_GROUP_SIZE]`: handles for all devices in the group.
- `subsetAllocation`: if `VK_TRUE`, `VkMemoryAllocateFlagsInfo.deviceMask` can be used to allocate memory on a subset of devices. If `VK_FALSE`, all allocations span all devices.

On a typical Linux system with two GPUs from different vendors, each GPU is its own group of size 1. Multi-device groups of size >1 are advertised only when the driver has confirmed high-speed connectivity.

### 7.3 Creating a Multi-Device Logical Device

To create a logical device spanning a multi-device group:

```c
/* Assume groups[0] has physicalDeviceCount == 2 */
VkDeviceGroupDeviceCreateInfo groupCI = {
    .sType                = VK_STRUCTURE_TYPE_DEVICE_GROUP_DEVICE_CREATE_INFO,
    .pNext                = NULL,
    .physicalDeviceCount  = groups[0].physicalDeviceCount,
    .pPhysicalDevices     = groups[0].physicalDevices,
};

VkDeviceCreateInfo deviceCI = {
    .sType                = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO,
    .pNext                = &groupCI,          /* chain the group info */
    .queueCreateInfoCount = ...,
    .pQueueCreateInfos    = ...,
    .enabledExtensionCount = ...,
    .ppEnabledExtensionNames = ...,
};

/* physicalDevice must be one of the devices listed in pPhysicalDevices */
VkDevice device;
vkCreateDevice(groups[0].physicalDevices[0], &deviceCI, NULL, &device);
```

[Source: VkDeviceGroupDeviceCreateInfo — Vulkan Documentation Project](https://docs.vulkan.org/refpages/latest/refpages/source/VkDeviceGroupDeviceCreateInfo.html)

The resulting `VkDevice` has a device index for each physical device: index 0 is `physicalDevices[0]`, index 1 is `physicalDevices[1]`. These indices appear in device masks — bitmasks where bit `i` selects device `i`.

### 7.4 Rendering Strategies: SFR and AFR

Vulkan device groups do not define SFR (Split Frame Rendering) or AFR (Alternate Frame Rendering) as named API modes. They are *application-level rendering strategies* built on device masks and the structures below.

#### Split Frame Rendering (SFR)

In SFR, each physical device in the group renders a spatial region of the frame. Device 0 renders the left half, device 1 renders the right half:

```c
/* VkDeviceGroupRenderPassBeginInfo — pin render areas to specific devices */
VkRect2D deviceRenderAreas[2] = {
    { .offset = {0, 0},    .extent = {width/2, height} },  /* GPU 0: left */
    { .offset = {width/2, 0}, .extent = {width/2, height} }, /* GPU 1: right */
};
VkDeviceGroupRenderPassBeginInfo deviceGroupRP = {
    .sType              = VK_STRUCTURE_TYPE_DEVICE_GROUP_RENDER_PASS_BEGIN_INFO,
    .deviceMask         = 0b11,     /* both devices active */
    .deviceRenderAreaCount = 2,
    .pDeviceRenderAreas = deviceRenderAreas,
};

VkRenderPassBeginInfo rpBegin = {
    .sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO,
    .pNext = &deviceGroupRP,
    /* ... framebuffer, renderPass, etc. */
};
vkCmdBeginRenderPass(cmd, &rpBegin, VK_SUBPASS_CONTENTS_INLINE);
```

The image backing the framebuffer must be bound with `VkBindImageMemoryDeviceGroupInfo` to specify which physical device's memory holds each split region:

```c
VkRect2D splitRegions[2] = {
    { .offset = {0, 0},    .extent = {width/2, height} },
    { .offset = {width/2, 0}, .extent = {width/2, height} },
};
VkBindImageMemoryDeviceGroupInfo deviceGroupBind = {
    .sType = VK_STRUCTURE_TYPE_BIND_IMAGE_MEMORY_DEVICE_GROUP_INFO,
    .splitInstanceBindRegionCount = 2,
    .pSplitInstanceBindRegions    = splitRegions,
};
VkBindImageMemoryInfo bindInfo = {
    .sType  = VK_STRUCTURE_TYPE_BIND_IMAGE_MEMORY_INFO,
    .pNext  = &deviceGroupBind,
    .image  = image,
    .memory = deviceGroupMemory,
    .memoryOffset = 0,
};
vkBindImageMemory2(device, 1, &bindInfo);
```

#### Alternate Frame Rendering (AFR)

In AFR, device 0 renders even frames and device 1 renders odd frames. The device mask in command buffer submission alternates each frame:

```c
uint32_t frameIndex = 0;

/* VkDeviceGroupSubmitInfo controls which physical devices execute each command buffer */
VkDeviceGroupSubmitInfo deviceGroupSubmit = {
    .sType                        = VK_STRUCTURE_TYPE_DEVICE_GROUP_SUBMIT_INFO,
    .commandBufferCount           = 1,
    /* device mask: bit 0 = GPU0 (even frames), bit 1 = GPU1 (odd frames) */
    .pCommandBufferDeviceMasks    = (uint32_t[]){ 1u << (frameIndex & 1) },
    .signalSemaphoreCount         = 0,
    .waitSemaphoreCount           = 0,
};

VkSubmitInfo submitInfo = {
    .sType = VK_STRUCTURE_TYPE_SUBMIT_INFO,
    .pNext = &deviceGroupSubmit,
    .commandBufferCount = 1,
    .pCommandBuffers    = &cmdBuf,
};
vkQueueSubmit(queue, 1, &submitInfo, fence);
frameIndex++;
```

AFR can deliver near-linear throughput scaling for GPU-bound workloads on identical GPUs with a high-speed interconnect. In practice, without NVLink or Infinity Fabric, PCIe latency makes AFR stutter-prone for interactive applications; it is more useful for offline rendering farms.

### 7.5 Swapchain Presentation with Device Groups

For presentation, the application queries which modes the swapchain supports:

```c
VkDeviceGroupPresentCapabilitiesKHR caps = {
    .sType = VK_STRUCTURE_TYPE_DEVICE_GROUP_PRESENT_CAPABILITIES_KHR,
};
vkGetDeviceGroupPresentCapabilitiesKHR(device, &caps);

/* caps.presentMask[i]: bitmask of GPUs whose images device i can present */
/* caps.modes: bitfield of VkDeviceGroupPresentModeFlagBitsKHR */
```

The `modes` field is a bitmask of:
- `VK_DEVICE_GROUP_PRESENT_MODE_LOCAL_BIT_KHR` (`0x1`): each GPU presents its own swapchain images. Standard path; always supported.
- `VK_DEVICE_GROUP_PRESENT_MODE_REMOTE_BIT_KHR` (`0x2`): a GPU with a presentation engine can present swapchain images from another GPU in the group.
- `VK_DEVICE_GROUP_PRESENT_MODE_SUM_BIT_KHR` (`0x4`): a GPU with a presentation engine composites the sum of images from multiple GPUs (used for frame accumulation in path tracing).
- `VK_DEVICE_GROUP_PRESENT_MODE_LOCAL_MULTI_DEVICE_BIT_KHR` (`0x8`): multiple GPUs with presentation engines each present their own images simultaneously.

[Source: VkDeviceGroupPresentModeFlagBitsKHR — Vulkan Documentation Project](https://docs.vulkan.org/refpages/latest/refpages/source/VkDeviceGroupPresentModeFlagBitsKHR.html)

On a typical Linux system with a single display, only `LOCAL_BIT` is supported. `REMOTE_BIT` requires driver support for cross-GPU memory access, available on NVLink-connected NVIDIA GPUs and Infinity Fabric-connected AMD GPUs.

### 7.6 GLSL/SPIR-V Device Index Built-in

Within shaders, the active device can be queried using `SPV_KHR_device_group` and the `gl_DeviceIndex` built-in (GLSL) or `DeviceIndex` built-in (SPIR-V). This allows shaders to select different data paths per GPU:

```glsl
/* GLSL — requires GL_EXT_device_group */
#extension GL_EXT_device_group : require
void main() {
    /* GPU 0 renders left half, GPU 1 renders right half */
    float xOffset = float(gl_DeviceIndex) * 0.5;
    /* ... */
}
```

---

## 8. Peer-to-Peer GPU Memory

### 8.1 DMA-BUF as the Cross-Driver Sharing Mechanism

Every PRIME transfer — whether for display offload or render offload — ultimately passes through a `dma_buf` file descriptor. This is the only standard cross-driver buffer sharing mechanism in the Linux kernel. The `dma_buf` lifecycle:

```
Exporter (dGPU driver)          Importer (iGPU driver or compositor)
─────────────────────────────   ─────────────────────────────────────
dma_buf_export()                 │
dma_buf_fd() → fd               ──fd─→ dma_buf_get(fd)
                                        dma_buf_attach(dmabuf, dev)
                                        dma_buf_map_attachment() → sgt
                                        /* DMA into dev's IOMMU domain */
                                        dma_buf_unmap_attachment()
                                        dma_buf_detach()
                                        dma_buf_put()
```

### 8.2 Implicit vs Explicit Synchronisation

`dma_buf` supports two fence models:

**Implicit synchronisation** uses `dma_resv` objects attached to each `dma_buf`. A driver signals a `dma_fence` when its GPU has finished accessing the buffer; importers poll or wait on that fence before performing their own accesses. This is transparent to userspace but can create hidden pipeline stalls across GPU contexts.

**Explicit synchronisation** exports fences as `sync_file` objects via `dma_buf_sync_file_export()` and passes them to Vulkan via `VkImportFenceFdInfoKHR` / `VkImportSemaphoreFdInfoKHR`. Mesa 24.1 added Vulkan explicit sync support, making this the preferred model for Wayland compositors that use `linux-drm-syncobj-v1`. [Source: Mesa 24.1 explicit sync announcement](https://www.chicagovps.net/blog/mesa-24-1-linux-graphics-stack-now-with-vulkan-explicit-sync-support-released/)

### 8.3 Importing a DMA-BUF into Vulkan

An application can import a `dma_buf` file descriptor into Vulkan memory using `VK_EXT_external_memory_dma_buf`:

```c
/* Export a GEM buffer as dma_buf */
int dmaBufFd;
drmPrimeHandleToFD(drmFd, gemHandle, DRM_CLOEXEC | DRM_RDWR, &dmaBufFd);

/* Get memory requirements for an external image */
VkExternalMemoryImageCreateInfo extMemImg = {
    .sType       = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
VkImageCreateInfo imgCI = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext = &extMemImg,
    /* format, extent, usage, etc. */
};
VkImage image;
vkCreateImage(device, &imgCI, NULL, &image);

/* Import the dma_buf as VkDeviceMemory */
VkImportMemoryFdInfoKHR importInfo = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
    .fd         = dmaBufFd,   /* ownership transferred to Vulkan */
};
VkMemoryAllocateInfo allocInfo = {
    .sType           = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO,
    .pNext           = &importInfo,
    .allocationSize  = memReqs.size,
    .memoryTypeIndex = ...,
};
VkDeviceMemory importedMemory;
vkAllocateMemory(device, &allocInfo, NULL, &importedMemory);
vkBindImageMemory(device, image, importedMemory, 0);
```

[Source: VK_EXT_external_memory_dma_buf — Vulkan Documentation Project](https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_external_memory_dma_buf.html)

Note: after `vkAllocateMemory` with an imported fd, the Vulkan implementation takes ownership of the fd and the caller must not close it.

### 8.4 PCIe Peer-to-Peer DMA

For GPU-to-GPU copies without CPU involvement, the kernel's `p2pdma` framework (`CONFIG_PCI_P2PDMA`) enables PCIe peer-to-peer DMA. A GPU allocates PCI BAR memory (`pci_p2pdma_add_resource()`), and another GPU's DMA engine writes directly to that BAR. The kernel validates that both GPUs are behind the same PCIe root port or host bridge before permitting P2P DMA:

```c
/* Kernel driver: add GPU VRAM BAR as a p2pdma resource */
int ret = pci_p2pdma_add_resource(pdev, bar_index, offset, size);

/* Check whether two PCIe devices can do P2P DMA */
struct pci_dev *peers[] = { gpu0_pdev, gpu1_pdev };
int distance = pci_p2pdma_distance_many(gpu0_pdev, peers, 2, false);
if (distance >= 0)
    /* P2P is feasible; allocate shared memory */
    void *p2p_mem = pci_alloc_p2pmem(gpu0_pdev, size);
```

In practice, PCIe P2P DMA is blocked by many laptop IOMMUs and is most reliable on workstation/server platforms with x86 CPUs and non-virtualised PCIe topologies. NVIDIA NVLink provides a proprietary high-bandwidth alternative: NVLink 4.0 (Hopper/H100) delivers 900 GB/s bidirectional bandwidth vs. PCIe 4.0 x16's 64 GB/s.

### 8.5 Zero-Copy Display Path

The optimal path for PRIME display offload is:

1. dGPU renders into a `LINEAR` or compositor-compatible tiled buffer.
2. `drmPrimeHandleToFD()` exports as `dma_buf`.
3. Compositor imports `dma_buf` via `drmPrimeFDToHandle()`.
4. Compositor calls `DRM_IOCTL_MODE_ADDFB2` with the imported handle and the same format modifier.
5. iGPU KMS engine scans out from dGPU VRAM directly, with no CPU or GPU copy.

This requires that the iGPU's scanout engine can access dGPU VRAM via PCIe (IOMMU permitting) and that both sides agree on the `DRM_FORMAT_MOD_*` tiling modifier. On most muxless laptops, the iGPU's display engine has IOMMU translations that permit PCIe reads from the dGPU's BAR, so zero-copy scanout is feasible for `DRM_FORMAT_MOD_LINEAR` buffers.

---

## 9. Power Management for the dGPU

### 9.1 PCI D-States and Runtime PM

The dGPU's power consumption when idle is a primary battery concern. Linux's PCI power management defines four D-states:

- **D0**: fully active, clocks running.
- **D1/D2**: optional intermediate states (rarely used by GPU drivers).
- **D3hot**: device suspended but Vcc power maintained; memory state preserved; resume latency ~10 ms.
- **D3cold**: Vcc power removed; maximum power savings; resume requires full re-initialisation (~500 ms for AMD/NVIDIA dGPUs).

Runtime PM (`CONFIG_PM`, `CONFIG_RUNTIME_PM`) allows the dGPU to enter D3 automatically when all DRM file handles on the render node are closed:

```bash
# Enable runtime PM for the dGPU (amdgpu)
echo "auto" | sudo tee /sys/bus/pci/devices/0000:01:00.0/power/control

# Check current power state
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# possible values: active, suspended, suspending, resuming, unsupported

# Verify D3cold is reached (requires ACPI _PR3 support)
cat /sys/bus/pci/devices/0000:01:00.0/power_state
```

### 9.2 ACPI `_PR3` and D3cold

D3cold requires the firmware to expose an ACPI `_PR3` power resource for the GPU's PCIe slot. When the kernel transitions the device from D3hot → D3cold, it evaluates `_PR3._OFF` to cut power to the slot. On some AMD Renoir and later laptops, this ACPI method was missing or incorrect, preventing D3cold and causing the dGPU to idle in D3hot at ~3–5 W. A kernel fix in Linux 5.10 addressed the issue by enabling D3cold for hot-plug ports on laptops without Thunderbolt. [Source: AMD Radeon dGPU D3cold Linux 5.10 fix — Phoronix](https://www.phoronix.com/news/Linux-5.10-Power-Management)

```bash
# Check if D3cold is supported by the device
cat /sys/bus/pci/devices/0000:01:00.0/d3cold_allowed

# Force D3cold off (useful for debugging stability issues)
echo 0 | sudo tee /sys/bus/pci/devices/0000:01:00.0/d3cold_allowed
```

### 9.3 AMD GFXOFF

AMD GPUs implement **GFXOFF** — a fine-grained power state below D0 that powers down the graphics engine (GFX block) independently of the PCIe interface. GFXOFF engages automatically when the GFX engine is idle and the display engine does not need it. [Source: AMD GFXOFF documentation](https://docs.kernel.org/gpu/amdgpu/thermal.html)

```bash
# Monitor AMD GPU power state including GFXOFF
cat /sys/kernel/debug/dri/1/amdgpu_pm_info
# Output includes: GFX off, current clocks, temperature, power draw

# Disable GFXOFF for benchmarking (prevents variable latency)
echo "0" | sudo tee /sys/kernel/debug/dri/1/amdgpu_gfxoff
```

### 9.4 Platform Profile

The ACPI `platform_profile` sysfs interface allows userspace to request system-wide power/performance behaviour:

```bash
# List available profiles
cat /sys/firmware/acpi/platform_profile_choices
# quiet low-power balanced performance

# Set balanced power for the dGPU (reduces peak power budget)
echo "balanced" | sudo tee /sys/firmware/acpi/platform_profile
```

`platform_profile` interacts with the GPU driver through `amdgpu`'s `power_dpm_force_performance_level` and NVIDIA's `nvidia-smi -pl` (power limit). TLP and auto-cpufreq both hook into `platform_profile` when present.

### 9.5 Trigger Conditions and Latency

The dGPU powers on (D3cold → D0) when:
1. An application opens `/dev/dri/renderDN` (the dGPU render node).
2. `__NV_PRIME_RENDER_OFFLOAD=1` is set (for NVIDIA on-demand mode).
3. A kernel `drm_open()` on the primary node (e.g., hotplug event for a display connector on the dGPU).

From D3cold to D0 readiness, typical latencies:
- NVIDIA RTX: 400–600 ms
- AMD RX 7000 series: 200–400 ms
- Intel Arc (discrete): 100–300 ms

These latencies are imperceptible for long-running applications but noticeable for applications that open and close rapidly. The Mesa `DRI_PRIME` path keeps the render node open for the application's lifetime, so re-wakeup latency is only incurred at application launch.

---

## 10. Practical Configurations

### 10.1 Choosing a Laptop Configuration

| Scenario | Recommendation |
|---|---|
| Gaming with external monitor | MUXed laptop, dGPU direct mode — zero-copy presentation, full VRAM available |
| Gaming at laptop display | MUXless + PRIME render offload; accept ~1 frame of PRIME latency |
| Productivity / battery life | iGPU only; set `power/control = auto`; dGPU in D3cold |
| Mixed use (gaming + productivity) | on-demand PRIME; DRI_PRIME per application |
| Workstation multi-display | Check which connectors are wired to which GPU; may need reverse PRIME for dGPU-driven heads |

### 10.2 Wayland Hybrid GPU Setup (GNOME)

```ini
# /etc/gdm3/custom.conf
[daemon]
WaylandEnable=true
```

On muxless laptops, GNOME on Wayland selects the iGPU (the device with connectors) as the display server GPU automatically. Applications launched with `DRI_PRIME=1` render on the dGPU; Mutter imports their `dma_buf` via `linux-dmabuf` and composites on the iGPU.

```bash
# Verify which GPU Mutter is using for display
MUTTER_DEBUG_ENABLE_ATOMIC_KMS=1 gnome-shell --wayland 2>&1 | grep "drm device"
```

### 10.3 KDE Plasma 6 GPU Selection

KDE Plasma 6 reads environment variables from `~/.config/plasma-workspace/env/`:

```bash
# ~/.config/plasma-workspace/env/prime.sh
export DRI_PRIME=1  # force all KDE apps to render on dGPU
```

For per-application selection, right-click any `.desktop` file in KDE and set "Launch with dedicated graphics card" — this prepends `DRI_PRIME=1` to the Exec line.

### 10.4 Steam and Gamescope

```bash
# Steam: per-game launch options (right-click game → Properties → Launch Options)
DRI_PRIME=1 %command%

# Steam: NVIDIA PRIME for proprietary driver
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia %command%

# Gamescope with PRIME (composite on iGPU, game renders on dGPU)
gamescope -W 1920 -H 1080 -f -- env DRI_PRIME=1 %command%

# Gamescope: force specific render device by path
gamescope --prefer-output DP-1 -- env DRI_PRIME=pci-0000_01_00_0 game
```

### 10.5 Monitoring GPU Activity

```bash
# Check which GPU is rendering (Gallium HUD, AMD/Intel/Nouveau)
GALLIUM_HUD="GPU-load,VRAM-usage" DRI_PRIME=1 glxgears

# AMD GPU monitoring
radeontop                           # ncurses GPU load monitor
watch -n1 cat /sys/kernel/debug/dri/1/amdgpu_pm_info  # power state

# Intel GPU monitoring
intel_gpu_top                       # real-time engine utilisation

# NVIDIA GPU monitoring
nvidia-smi -l 1                     # 1-second refresh
nvtop                               # multi-GPU dashboard (AMD+NVIDIA+Intel)

# Verify a Vulkan app is using the intended GPU
VK_INSTANCE_LAYERS=VK_LAYER_MESA_device_select MESA_VK_DEVICE_SELECT_FORCE_DEFAULT_DEVICE=1 \
    vulkaninfo --summary 2>&1 | head -20
```

### 10.6 Debugging PRIME Issues

```bash
# Confirm DRI3 is enabled in the X server
xdpyinfo | grep -i dri3

# Check that the compositor can import dma_buf from the dGPU
WAYLAND_DEBUG=1 weston 2>&1 | grep -i dmabuf

# Identify modifier mismatches (requires kernel 5.7+)
DRM_DEBUG_PRIME=1 dmesg | grep -i prime

# List supported format+modifier combinations for each GPU
for card in /dev/dri/card*; do
    echo "=== $card ==="
    modetest -D $card -f 2>&1 | grep -A2 "XR24"
done
```

---

## Integrations

- **Chapter 1 (DRM Architecture)**: The PRIME API — `DRM_IOCTL_PRIME_HANDLE_TO_FD` and `DRM_IOCTL_PRIME_FD_TO_HANDLE` — is implemented within the DRM core. The connector–encoder–CRTC ownership model (which GPU has display output) drives the PRIME topology.

- **Chapter 4 (GPU Memory Management)**: GEM buffer objects and GBM BOs are the buffers shared via PRIME. The `dma_buf` framework and `dma_resv` fence objects described in Ch4 are the synchronisation primitives used in every PRIME transfer.

- **Chapter 5 (x86 GPU Drivers)**: The `i915`, `xe`, and `amdgpu` drivers implement `drm_gem_prime_export()` and `drm_gem_prime_import()` callbacks. The iGPU driver's capability to import foreign `dma_buf` objects and attach them to KMS framebuffers determines whether zero-copy PRIME is possible.

- **Chapter 8 (Nouveau and NVK)**: NVK implements `VK_EXT_image_drm_format_modifier` and `VK_EXT_queue_family_foreign`, enabling standard DRI_PRIME offload. The nvidia-drm module provides the DRM interface required for modesetting and DMA-BUF interoperability even with the closed-source NVIDIA driver.

- **Chapter 18 (RADV — AMD Vulkan)**: RADV on AMD dGPU systems participates in PRIME via the same DMA-BUF mechanism as Mesa Gallium drivers. AMD's `DRM_FORMAT_MOD_AMD_GFX9_` tiling modifiers are used for zero-copy PRIME on AMD+AMD hybrid systems.

- **Chapter 49 (Multi-GPU and PRIME Render Offload)**: Chapter 49 covers overlapping material including vga_switcheroo, MUX hardware design, AMD SmartShift, vendor switching tools (supergfxctl, system76-power), and the NVLink/Infinity Fabric P2P DMA framework. This chapter (106) focuses on the Vulkan device-group API, DMA-BUF import into Vulkan, and practical configuration. Readers interested in the kernel vga_switcheroo and AMD SmartShift sysfs interfaces should consult Ch49.

- **Chapter 51 (GPU Power Management)**: Runtime PM, D3cold, ACPI `_PR3`, and `platform_profile` are covered in detail in Ch51. This chapter summarises the key sysfs paths relevant to dGPU power-off in PRIME configurations.

- **Chapter 78 (Gamescope)**: Gamescope's DMA-BUF compositing path is the mechanism through which PRIME render offload works in Steam sessions. The Steam Deck uses a single AMD APU with no PRIME, but Gamescope's `linux-dmabuf` import logic is the same code path exercised by PRIME on hybrid-GPU systems.

- **Chapter 89 (GPU Virtualisation)**: VM GPU assignment via VFIO-PCI shares architectural similarities with PRIME: a guest VM's GPU (passed through via VFIO) exports buffers that the host compositor imports for display via PRIME. The `dma_buf` fence mechanism is extended in virtualisation scenarios to span guest→host context boundaries.

---

## References

- [Linux Kernel DRM Memory Management — PRIME](https://dri.freedesktop.org/docs/drm/gpu/drm-mm.html)
- [Linux Kernel DMA-BUF subsystem documentation](https://docs.kernel.org/driver-api/dma-buf.html)
- [libdrm xf86drm.h — drmPrimeHandleToFD / drmPrimeFDToHandle](https://github.com/grate-driver/libdrm/blob/master/xf86drm.h)
- [NVIDIA PRIME Render Offload — Chapter 34](https://download.nvidia.com/XFree86/Linux-x86_64/440.31/README/primerenderoffload.html)
- [VK_KHR_device_group_creation — Khronos Registry](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_device_group_creation.html)
- [VkDeviceGroupDeviceCreateInfo — Vulkan Documentation Project](https://docs.vulkan.org/refpages/latest/refpages/source/VkDeviceGroupDeviceCreateInfo.html)
- [VkDeviceGroupPresentModeFlagBitsKHR — Vulkan Documentation Project](https://docs.vulkan.org/refpages/latest/refpages/source/VkDeviceGroupPresentModeFlagBitsKHR.html)
- [VK_EXT_external_memory_dma_buf — Vulkan Documentation Project](https://docs.vulkan.org/refpages/latest/refpages/source/VK_EXT_external_memory_dma_buf.html)
- [VkDeviceGroupPresentCapabilitiesKHR — Vulkan Documentation Project](https://docs.vulkan.org/refpages/latest/refpages/source/VkDeviceGroupPresentCapabilitiesKHR.html)
- [NVK — Mesa driver documentation](https://docs.mesa3d.org/drivers/nvk.html)
- [AMD GPU thermal and power controls — Linux kernel](https://docs.kernel.org/gpu/amdgpu/thermal.html)
- [AMD dGPU D3cold fix — Linux 5.10 — Phoronix](https://www.phoronix.com/news/Linux-5.10-Power-Management)
- [Mesa 24.1 Vulkan explicit sync support](https://www.chicagovps.net/blog/mesa-24-1-linux-graphics-stack-now-with-vulkan-explicit-sync-support-released)
- [Vulkan Device Groups guide — docs.vulkan.org](https://docs.vulkan.org/guide/latest/extensions/device_groups.html)
