# Chapter 172: eGPU on Linux — Thunderbolt, USB4, and PCIe Hot-Plug

This chapter targets three overlapping audiences: **systems and driver developers** who need to understand how DRM handles PCIe device arrival and removal at runtime; **laptop users and creative professionals** connecting external GPU enclosures to gain desktop-class graphics performance; and **kernel contributors** working on PCIe hot-plug, Thunderbolt authorization, or GPU driver unplug paths. Understanding eGPU support exercises nearly every layer of the Linux graphics stack simultaneously — the Thunderbolt/USB4 kernel subsystem, PCIe resource allocation, DRM device lifecycle, PRIME DMA-BUF sharing, and Wayland compositor hot-plug recovery.

---

## Table of Contents

1. [Introduction: External GPUs on Linux](#1-introduction-external-gpus-on-linux)
2. [Thunderbolt and USB4 Architecture](#2-thunderbolt-and-usb4-architecture)
3. [The bolt Daemon: Device Authorization](#3-the-bolt-daemon-device-authorization)
4. [Linux Kernel Thunderbolt/USB4 Subsystem](#4-linux-kernel-thunderboltusb4-subsystem)
5. [DRM PCIe Hot-Plug](#5-drm-pcie-hot-plug)
6. [PRIME Reverse Offload with eGPU](#6-prime-reverse-offload-with-egpu)
7. [AMD eGPU Support](#7-amd-egpu-support)
8. [NVIDIA eGPU Support](#8-nvidia-egpu-support)
9. [USB4 eGPU: Current Status](#9-usb4-egpu-current-status)
10. [Practical Setup Guide](#10-practical-setup-guide)
11. [Integrations](#11-integrations)

---

## 1. Introduction: External GPUs on Linux

An external GPU (eGPU) is a full desktop graphics card mounted in an enclosure that connects to a laptop or compact PC via a Thunderbolt or USB4 cable. The connection tunnels a PCIe link over the cable, so from the driver's perspective, the GPU appears as a standard PCIe device — but one that can arrive and depart at runtime while the system is running.

### Use Cases

The two dominant use cases reflect the system's architecture:

**Developer workstation augmentation.** A software engineer using a thin laptop for travel attaches an eGPU at their desk to accelerate GPU compute workloads, Vulkan-based CI pipelines, or machine-learning training jobs. In this scenario, the eGPU typically drives no display at all; compute results are copied back over PCIe.

**Creative professional with an eGPU dock.** A video editor or 3D artist uses the eGPU to accelerate rendering and compositor operations. An external monitor is plugged directly into the eGPU enclosure, bypassing the laptop's internal display pipeline entirely. When no external monitor is present, the eGPU renders its output and the result must be transferred back over the Thunderbolt link to the iGPU's scanout engine — a technique called *reverse PRIME*.

### Why eGPU Is Architecturally Interesting

eGPU on Linux exercises the following mechanisms simultaneously:

- **DRM hot-plug model**: `drm_dev_register` on PCIe probe, `drm_dev_unplug` on surprise removal, all while userspace file descriptors remain open.
- **PCIe topology changes at runtime**: bus numbers, BAR allocations, and MMIO windows must be assigned dynamically when the device appears.
- **PRIME reverse offload**: DMA-BUF sharing and blitting between two independently-managed DRM devices.
- **Display pipeline switching**: Wayland compositors must discover the new DRM device, migrate rendering state, and optionally redirect scanout.
- **Thunderbolt/USB4 security model**: the kernel must authorize a PCIe-capable device before creating tunnels, because Thunderbolt PCIe devices are DMA masters that can read system memory.

---

## 2. Thunderbolt and USB4 Architecture

### PCIe Tunneling

Thunderbolt was originally designed by Intel and Apple as a high-bandwidth interconnect that carries both DisplayPort and PCIe signals over a single cable. Rather than exposing raw PCIe signals electrically (as in OCuLink), Thunderbolt *encapsulates* PCIe TLPs inside Thunderbolt protocol packets and routes them over the serial link. At the receiving end, the Thunderbolt controller decapsulates them and presents a standard PCIe interface to the downstream device.

The net result is that a GPU inside a Thunderbolt enclosure registers as a PCI device under a Thunderbolt-managed PCIe root port. The kernel's PCI subsystem enumerates it normally, assigns bus numbers and BARs, and loads the appropriate GPU driver (`amdgpu`, `nvidia`, etc.) exactly as it would for an internally-soldered GPU.

### Thunderbolt 3, 4, and USB4 Bandwidth

| Protocol | Total link bandwidth | PCIe payload available | PCIe width |
|---|---|---|---|
| Thunderbolt 3 | 40 Gbps | ~22 Gbps (bandwidth shared with DP) | PCIe 3.0 ×4 equivalent |
| Thunderbolt 4 | 40 Gbps | up to 32 Gbps (guaranteed minimum) | PCIe 3.0 ×4 |
| USB4 Gen 2×2 | 20 Gbps | ~10 Gbps | PCIe 3.0 ×2 |
| USB4 Gen 3×2 / USB4 v1 40G | 40 Gbps | ~22–32 Gbps | PCIe 3.0 ×4 |
| USB4 v2 / Thunderbolt 5 | 80 Gbps (120 Gbps Bandwidth Boost) | up to 64 Gbps PCIe | PCIe 4.0 ×4+ |

Thunderbolt 3's 40 Gbps is shared between DisplayPort tunnels and PCIe tunnels. When a display is connected to the enclosure, the DisplayPort tunnel consumes ~18 Gbps, leaving roughly 22 Gbps for PCIe — equivalent to a PCIe 3.0 ×4 link at reduced efficiency. [Source](https://egpu.io/forums/thunderbolt-enclosures/technical-questions-on-tb3-pcie-tunnelling-bandwidth/) Thunderbolt 4 guarantees a minimum 32 Gbps for PCIe data even with a simultaneous display. [Source](https://docs.kernel.org/admin-guide/thunderbolt.html)

USB4 is the public specification derived from the Thunderbolt 3 protocol. The Thunderbolt 3 specification was contributed to the USB Implementers Forum and became USB4, with some differences at the register level but the same fundamental PCIe tunneling mechanism. [Source](https://docs.kernel.org/admin-guide/thunderbolt.html)

USB4 v2 (also marketed as Thunderbolt 5) doubles the link to 80 Gbps bidirectional, with an asymmetric "Bandwidth Boost" mode reaching 120 Gbps downstream. It enables up to 64 Gbps of PCIe throughput, making it equivalent to a PCIe 4.0 ×4 link — substantially reducing the performance gap between internal and external GPUs. [Source](https://www.notebookcheck.net/Thunderbolt-5-0-unveiled-with-240-Watts-fast-charging-and-up-to-three-times-more-bandwidth-than-Thunderbolt-4-0.748684.0.html)

### Thunderbolt Security Levels

Thunderbolt devices are DMA masters: once authorized, they can read and write arbitrary system RAM unless constrained by an IOMMU. The Linux kernel exposes six security levels via sysfs, readable from `/sys/bus/thunderbolt/devices/domainX/security`:

| Level | Meaning |
|---|---|
| `none` | All connected devices are authorized automatically (firmware-managed, no user interaction needed). |
| `user` | User or `bolt` must authorize each device by writing to the `authorized` attribute before PCIe tunnels are created. |
| `secure` | Like `user`, but the kernel also performs a cryptographic challenge using a 32-byte key stored with the device. |
| `dponly` | Firmware only permits DisplayPort and USB tunnels; PCIe tunneling is blocked. |
| `usbonly` | Limits tunneling to USB controller and DisplayPort in docking scenarios. |
| `nopcie` | BIOS-enforced prohibition of PCIe tunneling (USB4 systems only, Linux 5.12+). [Source](https://www.phoronix.com/news/Linux-5.12-USB4-SL5) |

[Source: Linux kernel Thunderbolt documentation](https://docs.kernel.org/admin-guide/thunderbolt.html)

Systems manufactured from approximately 2018 onwards may also support IOMMU-based DMA protection (`iommu_dma_protection = 1`), which constrains what memory any Thunderbolt PCIe device can access via DMA. When this is enabled, the security level becomes partially redundant for memory safety — though device identity verification remains important. [Source](https://www.guyrutenberg.com/2021/01/09/checking-thunderbolt-security-on-linux/)

### Physical Topology

The PCIe topology created by a Thunderbolt eGPU looks like this:

```
CPU
 └── Root Complex
      └── PCIe Root Port (to Thunderbolt controller)
           └── Thunderbolt Host Controller (NHI — Native Host Interface)
                └── [Thunderbolt cable]
                     └── Thunderbolt Switch (inside enclosure)
                          └── PCIe Endpoint (the GPU)
```

The Thunderbolt controller on the host side appears as a PCIe device itself, and uses the NHI to communicate ring buffers for control messages. Downstream of the switch, the GPU appears as a standard PCIe endpoint at a dynamically assigned bus number.

---

## 3. The bolt Daemon: Device Authorization

### Overview

`bolt` (also known as `boltd`) is the freedesktop.org daemon that manages Thunderbolt device enrollment on Linux systems using the `user` or `secure` security levels. It exposes the `org.freedesktop.bolt` D-Bus name on the system bus and stores an enrollment database of previously authorized devices and their cryptographic keys. [Source](https://github.com/gicmo/bolt)

The daemon is autostarted by systemd and udev when a Thunderbolt device is connected, eliminating the need for the user to manually write to sysfs.

### bolt Architecture

At its core, `boltd` is a C daemon (with some Python tooling using the Meson build system) that:

1. Watches `/sys/bus/thunderbolt/devices/` via udev for device additions.
2. On device arrival, checks its local database for a stored enrollment record.
3. If the device is enrolled with policy `auto`, writes `1` to the device's `authorized` sysfs attribute, creating PCIe tunnels.
4. If the device requires a cryptographic challenge (`secure` mode), it writes the challenge key and then writes `2` to `authorized`.
5. Exposes new unknown devices on D-Bus so that a GUI or CLI can prompt the user.

`boltd` also adapts its behavior when IOMMU DMA protection is detected — when `iommu_dma_protection` reads `1`, it can relax authorization requirements because hardware-enforced DMA isolation is active. [Source](https://github.com/gicmo/bolt)

### boltctl Commands

The `boltctl` command-line client interacts with `boltd` over D-Bus:

```bash
# List all known Thunderbolt devices (connected and stored)
boltctl list

# Authorize a specific device immediately (without storing it)
boltctl authorize <uuid>

# Enroll a device with a policy so it is auto-authorized on future connections
boltctl enroll --policy auto <uuid>

# Remove a stored enrollment
boltctl forget <uuid>

# Monitor udev events in real time
boltctl monitor
```

The `--policy` flag accepts `auto` (always authorize on connect), `manual` (require explicit authorization each time), and `iommu` (auto-authorize only when IOMMU DMA protection is active).

### Kernel sysfs Interface

`bolt` ultimately drives the kernel's sysfs interface. For a device at path `/sys/bus/thunderbolt/devices/0-1/`:

```bash
# Read the current authorization state (0 = unauthorized, 1 = authorized)
cat /sys/bus/thunderbolt/devices/0-1/authorized

# Manually authorize the device (creates PCIe tunnels)
echo 1 | sudo tee /sys/bus/thunderbolt/devices/0-1/authorized

# Read device identity attributes
cat /sys/bus/thunderbolt/devices/0-1/vendor_name
cat /sys/bus/thunderbolt/devices/0-1/device_name
cat /sys/bus/thunderbolt/devices/0-1/unique_id
```

Writing `0` to `authorized` de-authorizes the device and tears down PCIe tunnels. Writing `2` triggers a secure connect challenge (only valid in `secure` security mode).

### udev Rules for Automatic Authorization

For systems where the security level is `none`, or where IOMMU protection is trusted, a udev rule can authorize devices automatically:

```udev
# /etc/udev/rules.d/99-thunderbolt-auto.rules
# Auto-authorize Thunderbolt devices when IOMMU DMA protection is active
ACTION=="add", SUBSYSTEM=="thunderbolt", ATTR{iommu_dma_protection}=="1", \
    RUN+="/bin/sh -c 'echo 1 > /sys%p/authorized'"
```

This approach is simpler than bolt for fully trusted environments but provides no enrollment database or per-device policy management.

### Security Implications: DMA Attack Surface

A Thunderbolt PCIe device is a full DMA master on the system bus. Before authorization (writing `1` to `authorized`), the device has no PCIe tunnel and cannot generate DMA transactions. After authorization, it can issue PCIe Memory Read/Write requests to any physical address unless an IOMMU constrains it.

The practical attack scenario (explored in the 2019 Thunderclap research [Source](https://www.ndss-symposium.org/ndss-paper/thunderclap-exploring-vulnerabilities-in-operating-system-iommu-protection-via-dma-from-untrustworthy-peripherals/)) is that a malicious device mimics a legitimate peripheral, gets authorized, and then reads credentials or cryptographic keys from RAM. The mitigations are:

1. Enable Kernel DMA Protection (`iommu_dma_protection`) in UEFI — available on most systems shipped after 2019.
2. Use `secure` security level with bolt to verify device identity cryptographically before authorization.
3. Use `bolt`'s per-device `iommu` policy to only auto-authorize when hardware DMA isolation is active.

Never use security level `none` on a machine that handles sensitive data and is carried to untrusted locations.

---

## 4. Linux Kernel Thunderbolt/USB4 Subsystem

### Driver Overview

The unified Thunderbolt/USB4 driver lives in `drivers/thunderbolt/` and is selected by `CONFIG_USB4` (formerly `CONFIG_THUNDERBOLT`). It was introduced as a unified driver in Linux 5.6, replacing earlier generation-specific drivers. [Source](https://cateee.net/lkddb/web-lkddb/USB4.html) The module is named `thunderbolt` regardless of the underlying protocol.

Key files in `drivers/thunderbolt/`:

| File | Purpose |
|---|---|
| `nhi.c` | Native Host Interface — manages the host controller's ring buffers, PCI probe/remove, and interrupt handling. |
| `tb.c` | Software connection manager — device enumeration, PCIe/DP/USB3 tunnel lifecycle, bandwidth management. |
| `switch.c` | Router/switch abstraction — reads and writes router configuration space over the ring buffer protocol. |
| `tunnel.c` | Tunnel creation and teardown for PCIe, DisplayPort, USB3, and network paths. |
| `usb4.c` | USB4-specific register access layer — handles differences between USB4 and Thunderbolt register layouts. |
| `xdomain.c` | Cross-domain protocol for peer-to-peer connections (e.g., Thunderbolt networking). |
| `dma_port.c` | DMA port for NVM firmware upgrades. |

The original Thunderbolt driver was authored by **Andreas Noever**, whose work laid the foundation for the subsystem. The unified Thunderbolt + USB4 driver that is current today was largely driven by **Mika Westerberg** (Intel), who merged USB4 support into the existing Thunderbolt driver framework and has continued driving USB4 v2 additions. [Source](https://lore.kernel.org/linux-usb/20230612082145.62218-20-mika.westerberg@linux.intel.com/T/)

### Connection Manager Models

The kernel driver supports two connection manager implementations detected at runtime:

**Firmware Connection Manager (ICM)**: Implemented in the Thunderbolt controller's firmware (common on Intel Thunderbolt 3 PCs). The firmware handles device enumeration and tunnel negotiation; the kernel driver issues authorization commands to the firmware over the NHI ring buffer.

**Software Connection Manager**: Implemented entirely in the kernel (`tb.c`). Used by Apple systems and newer USB4 compliant devices. In this mode, the kernel reads the router configuration registers directly and creates tunnels itself. The software CM exclusively advertises `user` security level and is expected to run alongside IOMMU-based DMA protection.

### USB4 Support: usb4.c

`drivers/thunderbolt/usb4.c` implements the USB4 register access layer. USB4 uses the same fundamental PCIe tunneling concept as Thunderbolt 3/4 but differs at the configuration register level. Key differences handled in `usb4.c`:

- **USB4 router configuration**: USB4 routers expose configuration registers via a defined USB4 router configuration space, distinct from Thunderbolt 2/3 registers.
- **USB4 v2 extended encapsulation**: USB4 v2 (80G) introduces PCIe TLP/DLLP extended encapsulation to support higher link widths and speeds. [Source](https://lore.kernel.org/linux-usb/20230612082145.62218-20-mika.westerberg@linux.intel.com/T/)
- **Asymmetric link modes**: USB4 v2 and Thunderbolt 5 support 120/40 Gbps asymmetric bandwidth allocation.

USB4 v2 initial support was added in the kernel ~6.4 timeframe through a 20-patch series from Westerberg and Gil Fine. [Source](https://lore.kernel.org/linux-usb/20230612082145.62218-20-mika.westerberg@linux.intel.com/T/)

### PCIe Tunnel Creation

When a new device is authorized (by `bolt` or manual sysfs write), the software connection manager in `tb.c` creates a PCIe tunnel through the function `tb_tunnel_alloc_pci()` in `tunnel.c`. The tunnel negotiation:

1. Allocates bandwidth from the domain's available pool.
2. Configures the adapter registers on both the host-side and device-side router ports.
3. Sets up the PCIe adapter paths — one path for upstream traffic (device to host) and one for downstream (host to device).
4. Activates the tunnel, enabling PCIe TLP forwarding.

Once the tunnel is active, the kernel's PCI subsystem detects the new PCIe endpoint via the standard hot-plug mechanism (`pciehp` or the ACPI hot-plug subsystem) and begins enumeration: assigning bus numbers, reading BARs, and calling the device driver's `probe()` callback.

### Hot-Plug Notifications

The Thunderbolt controller interrupts the CPU when a device is connected or removed. The NHI driver processes these events on a work queue and calls the connection manager's `hotplug` callback. For device arrival, this triggers the authorization check and eventually tunnel creation. For device removal, the connection manager tears down tunnels first, then signals the PCI subsystem to remove the PCIe device.

The kernel broadcasts `KOBJ_CHANGE` udev events for tunnel state changes through the `TUNNEL_EVENT` environment variable, allowing userspace tools to react to bandwidth changes or tunnel failures.

### Kernel Configuration

To enable Thunderbolt/USB4 and PCIe hot-plug support:

```kconfig
# Unified USB4 and Thunderbolt driver (required)
CONFIG_USB4=m

# PCIe hot-plug support (required for runtime device enumeration)
CONFIG_HOTPLUG_PCI=y
CONFIG_HOTPLUG_PCI_PCIE=y   # PCIe native hot-plug
CONFIG_HOTPLUG_PCI_ACPI=y   # ACPI-based hot-plug (fallback)

# IOMMU support (strongly recommended for security)
CONFIG_INTEL_IOMMU=y        # for Intel systems
CONFIG_AMD_IOMMU=y          # for AMD systems
CONFIG_IOMMU_DEFAULT_DMA_STRICT=y
```

Most distribution kernels ship with all of these enabled or modular.

---

## 5. DRM PCIe Hot-Plug

### The DRM Device Lifecycle

When a GPU driver's `probe()` callback is called after PCIe enumeration, the standard sequence is:

```c
/* In amdgpu_pci_probe() or xe_pci_probe() */
adev = drm_dev_alloc(&amdgpu_drm_driver, &pdev->dev);
/* ... hardware initialization ... */
ret = drm_dev_register(adev->ddev, 0);
```

`drm_dev_register()` creates the `/dev/dri/cardN` and `/dev/dri/renderDN` device nodes, advertises the device to userspace via udev, and makes the DRM file operations available. For a hot-plugged eGPU, this happens at the moment the PCIe tunnel is established and the driver completes initialization — the device node appears within seconds of connecting the cable and authorizing the enclosure.

[Source: Linux DRM internals documentation](https://docs.kernel.org/gpu/drm-internals.html)

### Hot-Unplug: drm_dev_unplug

The standard `drm_dev_unregister()` function expects no open file descriptors; calling it while users hold the device open causes a deadlock. For hot-pluggable devices, DRM provides `drm_dev_unplug()` instead:

```c
/* In the PCI remove callback */
static void amdgpu_pci_remove(struct pci_dev *pdev)
{
    struct drm_device *dev = pci_get_drvdata(pdev);
    drm_dev_unplug(dev);
    /* ... hardware teardown ... */
}
```

`drm_dev_unplug()` marks the DRM device as unplugged and wakes any threads blocked in `drm_dev_enter()`. The device node remains in `/dev/dri/` but returns `ENODEV` on subsequent ioctls. Existing file descriptors remain valid (they can be closed cleanly) but no new operations succeed.

[Source](https://docs.kernel.org/gpu/drm-internals.html)

### drm_dev_enter / drm_dev_exit

Driver code that accesses hardware registers must protect against the device being unplugged mid-operation. The pattern is:

```c
int amdgpu_some_ioctl(struct drm_device *dev, void *data,
                      struct drm_file *file_priv)
{
    struct amdgpu_device *adev = drm_to_adev(dev);
    int idx;

    if (!drm_dev_enter(dev, &idx))
        return -ENODEV;   /* device unplugged, bail out */

    /* Safe to access hardware here */
    amdgpu_ring_emit_fence(...);

    drm_dev_exit(idx);
    return 0;
}
```

`drm_dev_enter()` acquires a reference that prevents `drm_dev_unplug()` from completing until `drm_dev_exit()` is called. This serialization ensures that hardware accesses complete cleanly before the device is torn down.

### amdgpu Hot-Unplug History

Before Linux 5.14, amdgpu had no proper surprise-removal support. When a Thunderbolt eGPU was unplugged while in use, the system would typically freeze or produce oopses. The root cause was that applications had GPU buffer objects (BOs) mapped into their virtual address space via DMA-BUF; when the backing PCIe device disappeared, those mappings pointed to non-existent memory and any CPU access to them faulted.

A multi-version patch series by **Andrey Grodzovsky** (AMD) culminating in approximately v7 during 2021 addressed this by:

- Using `drm_dev_unplug()` instead of `drm_dev_unregister()` in `amdgpu_pci_remove()`.
- Propagating the `drm_dev_enter()`/`drm_dev_exit()` guards through critical command submission paths.
- Modifying the GPU scheduler (`drm_gpu_scheduler`) to drain pending jobs before teardown.
- Handling TTM (Translation Table Manager) eviction of BOs whose backing store disappeared.

[Source](https://lore.kernel.org/amd-gfx/20210510163625.407105-1-andrey.grodzovsky@amd.com/) [Source: Phoronix report](https://www.phoronix.com/news/AMDGPU-eGPU-Hot-Unplug-RFC)

The work landed in Linux 5.14, allowing hot-unplug of AMD Radeon GPUs used as eGPUs without system freezes. Some edge cases with active Vulkan device-local memory mappings and open DRM render nodes remain fragile.

### The DRM Device Node Lifecycle for Compositors

When a Wayland compositor opens `/dev/dri/cardN` for KMS (modesetting) and `/dev/dri/renderD128` for rendering, it holds file descriptors to those nodes. On eGPU removal:

1. `drm_dev_unplug()` is called by the driver's `remove()` callback.
2. Pending ioctls return `ENODEV`.
3. The compositor receives `ENODEV` (or a `DRM_EVENT_VBLANK` never arrives, causing a timeout) and must handle it.
4. The udev subsystem generates a `remove` event for the DRM device.
5. The compositor can react to the udev remove event and tear down its DRM backend.

### Compositor Recovery: wlroots

wlroots gained initial GPU hot-plug support through community contributions targeting Nvidia Optimus and eGPU use cases. [Source](https://github.com/swaywm/wlroots/pull/2423) The implementation monitors udev events:

- **Device add**: wlroots detects the udev `add` event for the DRM subsystem and creates a new DRM sub-backend, adding it to the multi-backend arrangement.
- **Device remove**: wlroots listens for udev `remove` events and destroys the corresponding DRM sub-backend.

The key architectural challenge is that removing a *rendering* device (as opposed to a display-only output) requires tearing down all GPU command buffers, descriptor sets, and framebuffers — not merely disconnecting an output. wlroots handles this by destroying the entire DRM backend instance and falling back to another available GPU. [Source](https://github.com/swaywm/wlroots/issues/1697)

GNOME's Mutter compositor has independent handling for GPU hot-plug. Most compositors can *tolerate* eGPU insertion (they detect the new DRM node via udev and can optionally migrate rendering to it) but surprise removal remains more fragile in practice.

---

## 6. PRIME Reverse Offload with eGPU

### The Typical eGPU Topology

When a laptop with an integrated GPU connects an eGPU, there are two DRM devices: the iGPU (e.g., `card0` for Intel UHD or AMD Radeon integrated) and the eGPU (e.g., `card1` for an AMD Radeon RX 7900 XT in a Thunderbolt enclosure). The laptop's internal display is wired to the iGPU's display engine. The eGPU may have an external monitor connected to its own display outputs, or it may render to a buffer that must be transferred to the iGPU for display.

> **Terminology note:** The term *reverse PRIME* is used in this chapter — and widely in the Linux eGPU community — to mean the case where the eGPU (the render provider) transfers frames to the iGPU (the display sink) for scanout. Some Mesa and DRM documentation uses the same term to describe the opposite direction (primary dGPU renders, iGPU scans out to built-in display). The meaning is consistent within this chapter; Chapter 49 discusses the full taxonomy.

### Scenario A: Direct eGPU Scanout (Preferred)

When an external monitor is connected directly to the eGPU enclosure, there is no need for cross-GPU buffer copies. The Wayland compositor detects both DRM devices, assigns the external monitor to the eGPU's KMS pipeline, and the eGPU renders and scans out independently. This is the highest-performance configuration and avoids all PRIME overhead.

### Scenario B: Reverse PRIME (Internal Display Only)

When the eGPU must render content for the laptop's internal display (which is wired to the iGPU), reverse PRIME is used:

1. The eGPU renders the frame into a DRM-managed buffer (GEM object).
2. A DMA-BUF file descriptor is exported from the eGPU's DRM device using `DRM_IOCTL_PRIME_HANDLE_TO_FD`.
3. The DMA-BUF is imported into the iGPU's DRM device using `DRM_IOCTL_PRIME_FD_TO_HANDLE`.
4. The iGPU blits or scans out the imported buffer to the display.

The blit transfer consumes part of the Thunderbolt link's available PCIe bandwidth. For a 4K display at 60 Hz, the uncompressed framebuffer is approximately 33 MB per frame (3840×2160×4 bytes); at 60 fps that is roughly 2 GB/s of data, or approximately **~16 Gbps** when expressed in bits. Against Thunderbolt 3's remaining PCIe budget of ~22 Gbps (after DisplayPort tunnel overhead), this transfer consumes roughly 70% of available bandwidth. On Thunderbolt 4 or USB4 with its higher PCIe guarantee, the ratio improves to roughly 50%, and on Thunderbolt 5 (USB4 v2) it drops to below 25%.

### Performance Overhead

Reverse PRIME introduces latency (the blit must complete before scanout) and bandwidth overhead (the framebuffer crosses the PCIe link on every frame). The large bandwidth consumption — particularly on Thunderbolt 3 where it can approach 70% of the PCIe budget — means that GPU-bound workloads have substantially less bandwidth available for vertex/texture streaming and command submission while a reverse-PRIME display is active. **Note: needs verification** — quantitative benchmarks of the PCIe bandwidth impact on specific workloads vary by GPU, workload, and Thunderbolt generation; consult current eGPU.io community benchmarks for empirical data.

For compute-only workloads (ML training, GPGPU), reverse PRIME is irrelevant — results are copied back over PCIe regardless.

### X11: xrandr --setprovideroffloadsink

Under X11 (Xorg), PRIME requires explicit configuration:

```bash
# List available providers (GPU devices)
xrandr --listproviders

# Configure the eGPU as the render provider, with the iGPU as the sink (display)
xrandr --setprovideroffloadsink 1 0

# Then run an application on the eGPU
DRI_PRIME=1 glxgears
```

On modern Xorg with DDX drivers that have DRI3 enabled by default, the `--setprovideroffloadsink` step is often handled automatically.

### Wayland: DRI_PRIME and Compositor Configuration

Under Wayland, `DRI_PRIME` is the standard environment variable for selecting which GPU a specific application uses:

```bash
# Use eGPU (by index) for a specific application
DRI_PRIME=1 vulkaninfo

# Use eGPU by PCI device ID (more stable than index across reboots)
DRI_PRIME=pci-0000_0c_00_0 glxinfo
```

The PCI device path form (`pci-XXXX_XX_XX_X`) is preferred because device indices can shift when hot-plugging. The PCI path is stable for a given hardware configuration.

However, `DRI_PRIME` does not tell the Wayland *compositor* to use the eGPU as its primary renderer. Compositors use the GPU that controls the `boot_vga` display output by default. The `all-ways-egpu` community script provides workarounds for GNOME (Mutter), KDE Plasma (KWin), and wlroots-based compositors:

```bash
# Install and configure all-ways-egpu
# Method 2: Flip the boot_vga indicator flag
sudo all-ways-egpu setup

# Method 3: Set compositor-specific environment variables
# For wlroots (Sway, Hyprland): sets WLR_DRM_DEVICES
# For Mutter (GNOME): sets MUTTER_DEBUG_FORCE_KMS_MODE
# For KWin (KDE Plasma): sets KWIN_DRM_DEVICES
```

[Source](https://github.com/ewagner12/all-ways-egpu)

There is currently no universal Wayland protocol extension for compositor-level GPU selection; the situation varies by compositor. **Note: needs verification** that all three compositors support the variables listed above as of kernel 6.10+.

---

## 7. AMD eGPU Support

### Which GPU Generations Work

`amdgpu` supports all GCN 1.0+ (Radeon HD 7000 series, 2012+), RDNA, and RDNA2/3 GPUs connected over Thunderbolt. In practice, eGPU use is most common with:

- **RDNA 2**: Radeon RX 6000 series — well-supported, hot-plug works with kernel ≥ 5.14.
- **RDNA 3**: Radeon RX 7000 series — supported, but some enclosures require kernel boot parameters (see below).
- **Older GCN**: Functional but less tested; surprise removal is more likely to cause issues.

### Known Issues

**Surprise removal**: Despite the improvements in Linux 5.14, abrupt physical disconnection (not gracefully authorized-0'd via bolt first) can still produce `amdgpu` errors in dmesg and occasionally stall the rendering thread. The recommended procedure for removing an eGPU gracefully:

```bash
# Step 1: Tear down applications using the eGPU

# Step 2: De-authorize via boltctl (tears down PCIe tunnel first)
boltctl authorize --set-policy manual <uuid>
# or directly:
echo 0 | sudo tee /sys/bus/thunderbolt/devices/0-1/authorized

# Step 3: The GPU driver will receive PCI remove notification
# and call amdgpu_pci_remove() cleanly
```

**DRI3/Present interaction**: Some Xorg configurations with DRI3 and `Present` extension enabled on reverse PRIME setups have exhibited tearing or synchronization failures with amdgpu eGPUs. These are typically resolved by using Wayland instead.

**Wayland compositor rebinding**: After an amdgpu eGPU is removed and re-inserted, most Wayland compositors do not automatically reclaim and initialize the new DRM device. A compositor restart is often required. This is a compositor-level limitation, not a kernel limitation.

**Power management across link suspend**: When the Thunderbolt link enters a low-power state (e.g., because the host enters system suspend), the GPU must enter D3cold. On resume, the Thunderbolt tunnel must be re-established before the GPU can transition back to D0. `amdgpu` handles this through the standard PCI PM callbacks (`amdgpu_pm_suspend`/`amdgpu_pm_resume`), but the tunnel re-establishment adds latency to system resume.

### Runtime PM

amdgpu supports runtime PM on eGPUs — the GPU can enter D3cold over the Thunderbolt link when idle. The system must have `CONFIG_PM_RUNTIME=y` and the device's runtime PM must be enabled:

```bash
# Check current runtime PM status
cat /sys/bus/pci/devices/0000:0c:00.0/power/runtime_status

# Enable runtime autosuspend (2 second idle timeout)
echo auto | sudo tee /sys/bus/pci/devices/0000:0c:00.0/power/control
echo 2000 | sudo tee /sys/bus/pci/devices/0000:0c:00.0/power/autosuspend_delay_ms
```

---

## 8. NVIDIA eGPU Support

### Proprietary Driver

The NVIDIA proprietary driver supports eGPU configurations on Linux, but historically this support has been fragile and poorly documented. The key requirement is enabling `AllowExternalGpus` in the X11 configuration:

```xorg
# /usr/share/X11/xorg.conf.d/10-nvidia.conf
Section "OutputClass"
    Identifier "nvidia"
    MatchDriver "nvidia-drm"
    Driver "nvidia"
    Option "AllowExternalGpus" "True"
EndSection
```

[Source: NVIDIA developer blog](https://developer.nvidia.com/blog/accelerating-machine-learning-on-a-linux-laptop-with-an-external-gpu/)

Some systems require early-loading the `thunderbolt` kernel module to ensure it is initialized before `nvidia_drm` during boot:

```bash
# /etc/modules-load.d/thunderbolt-early.conf
thunderbolt
```

The standard boot parameters to ensure PCIe resources are allocated correctly:

```bash
# /etc/default/grub GRUB_CMDLINE_LINUX_DEFAULT
pci=assign-busses,hpbussize=0x33,realloc,hpmmiosize=128M,hpmmioprefsize=16G pcie_ports=native
```

**Note: needs verification** that all these parameters remain necessary on recent kernels (≥ 6.8); PCIe resource allocation has improved substantially since these recommendations were first published.

### Known Issues with NVIDIA Proprietary Driver and Thunderbolt

A confirmed hard-lock issue exists as of 2025 where an RTX 5080 connected via Thunderbolt 5 (Sonnet Breakaway Box 850T5) causes immediate system hard-lock on any CUDA operation, despite `nvidia-smi` functioning normally at idle. The PCIe link negotiates successfully at PCIe 4.0 ×4 (16 GT/s), but CUDA memory operations trigger a fault with no kernel panic or recoverable error. [Source](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/979) Without `pcie_ports=native`, the GPU enters D3cold and the driver cannot transition it back to D0.

The NVIDIA proprietary driver does not support `drm_dev_unplug()` — it uses its own device lifecycle management. Surprise removal with an NVIDIA eGPU on Linux is likely to produce unrecoverable errors.

### Nova / nvidia-open Kernel Module

The `nvidia-open` kernel module (NVIDIA's open-source kernel module, distinct from the community Nouveau driver) began shipping with Linux kernel integration starting with the 515.x driver series. As of the kernel 6.15 timeframe, the **Nova** Rust-based replacement for `nvidia-open` is in very early stages, with initial code having landed in DRM-next. [Source](https://www.phoronix.com/news/NVIDIA-Linux-2025-Highlights)

Neither `nvidia-open` nor Nova has documented or verified eGPU/Thunderbolt-specific support. **Note: needs verification** — the `nvidia-open` module likely uses the same `AllowExternalGpus` path as the binary blob, but Thunderbolt hot-plug has not been publicly confirmed as supported. Nova is far too early in development to have eGPU-specific code.

### NVK (Open-Source Vulkan Driver)

NVK, the Mesa open-source Vulkan driver for NVIDIA GPUs, operates in userspace on top of either the Nouveau kernel driver or eventually Nova. NVK itself has no Thunderbolt-specific code — hot-plug support depends entirely on the kernel driver. Given that Nouveau has limited eGPU support (the `nouveau` driver on modern GPUs requires GSP firmware that complicates hot-plug), **NVK eGPU support is not verified as of 2026**. **Note: needs verification.**

---

## 9. USB4 eGPU: Current Status

### USB4 Gen 3×2 vs. Thunderbolt 4

USB4 Gen 3×2 (40 Gbps) and Thunderbolt 4 (40 Gbps) are electrically and bandwidth-equivalent for PCIe tunneling purposes. Both use the same Thunderbolt 3 physical layer at 40 Gbps. The Linux `thunderbolt` driver handles both transparently via the `usb4.c` register access layer. A USB4 Gen 3×2 host port can connect to a Thunderbolt 4 enclosure and vice versa.

The difference that matters for eGPU use: Thunderbolt 4 *certifies* that PCIe tunneling works and mandates minimum bandwidth guarantees. USB4 Gen 3×2 *permits* PCIe tunneling but does not require the host or enclosure to implement it. Some USB4 hosts (particularly those on AMD platforms prior to Ryzen 7000) and some USB4 enclosures do not support PCIe tunneling.

### AMD USB4 Support

AMD integrated USB4 support into its SoCs beginning with the Ryzen 7000 series (Zen 4, 2022). Earlier Ryzen systems with USB4 on the Zen 3 platform have partial PCIe tunneling support. The AMD USB4 driver patch series added proper handling for PCIe bridge resume after USB4 switch discovery. [Source](https://lkml.iu.edu/hypermail/linux/kernel/2209.1/00801.html)

### Intel USB4 (Meteor Lake and later)

Intel Meteor Lake (Core Ultra 1st Gen, 2023) integrated Thunderbolt 4 / USB4 Gen 3×2 into the SoC fabric directly, replacing the previous discrete Thunderbolt controller. Thunderbolt 5 (USB4 v2, 80 Gbps) debuted in Intel Arrow Lake and Lunar Lake platforms. These hosts support full PCIe tunneling to USB4 and Thunderbolt 4/5 enclosures.

### USB4 v2 / Thunderbolt 5 eGPU

USB4 v2 and Thunderbolt 5 double the available PCIe bandwidth to approximately 64 Gbps, equivalent to PCIe 4.0 ×4. This is sufficient to deliver most of a modern GPU's command and data throughput without being the bottleneck. Commercial enclosures supporting Thunderbolt 5 include the Razer Core X V2 and the Sonnet Breakaway Box 850T5. [Source](https://www.trebleet.com/product-page/80gbps-egpu-enclosure-compatible-with-thunderbolt-5-usb4-v2)

Linux kernel USB4 v2 support (including 80G link initialization and extended encapsulation) was added through Westerberg's patch series targeting approximately the 6.4–6.5 merge windows. [Source](https://lore.kernel.org/linux-usb/20230612082145.62218-20-mika.westerberg@linux.intel.com/T/)

### Known Working USB4 eGPU Enclosures on Linux

The following have been reported working by Linux users as of 2025–2026. Compatibility depends on the host's PCIe resource allocation, the specific GPU, and kernel version.

| Enclosure | Interface | Linux Status |
|---|---|---|
| Razer Core X | Thunderbolt 3/4 | Generally working (AMD GPUs); NVIDIA problematic |
| Razer Core X Chroma | Thunderbolt 3/4 | Generally working |
| Razer Core X V2 | Thunderbolt 5 | Emerging support; NVIDIA CUDA lock-up bug (#979) |
| Sonnet Breakaway Box 350W | Thunderbolt 3/4 | Widely used; AMD GPUs work well |
| Sonnet Breakaway Box 750 | Thunderbolt 3/4 | Same |
| Sonnet Breakaway Box 850T5 | Thunderbolt 5 | Hardware works; NVIDIA driver issues |
| ADT-Link UT3G | USB4 40 Gbps | AMD/NVIDIA reported working; PCIe Gen 4 slot |

**Note: needs verification** for all entries above — compatibility changes with kernel and driver versions. Consult the eGPU.io community forums for current reports. [Source](https://egpu.io/forums/thunderbolt-linux-setup/)

---

## 10. Practical Setup Guide

### Step 1: Verify Thunderbolt Hardware Support

```bash
# Check that a Thunderbolt controller is present
lspci -vvv | grep -i thunderbolt

# Expected output (example, Intel Titan Ridge):
# 00:1d.0 PCI bridge: Intel Corporation ... Thunderbolt 3 Bridge (rev 02)

# Check if the thunderbolt module is loaded
lsmod | grep thunderbolt

# Load manually if needed
sudo modprobe thunderbolt

# Verify the Thunderbolt domain security level
cat /sys/bus/thunderbolt/devices/domain0/security
# Output: user   (common) or none (some OEMs)

# Verify IOMMU DMA protection status
cat /sys/bus/thunderbolt/devices/domain0/iommu_dma_protection
# Output: 1 (protected) or 0 (unprotected)
```

### Step 2: Connect the Enclosure and Authorize

```bash
# Connect the enclosure via Thunderbolt cable
# Watch dmesg for device detection
dmesg -w | grep -i thunderbolt

# List detected devices
boltctl list
# Output shows the device UUID, vendor, name, and current status

# Enroll with auto-authorization policy
boltctl enroll --policy auto <uuid>

# Confirm authorization succeeded
cat /sys/bus/thunderbolt/devices/0-1/authorized
# Output: 1
```

### Step 3: Verify PCIe Device Visibility

```bash
# List PCI devices before and after authorization
# Before authorization:
lspci | grep -i "VGA\|3D\|Display"
# Shows: iGPU only (e.g., Intel UHD Graphics)

# After authorization:
lspci | grep -i "VGA\|3D\|Display"
# Shows: iGPU + eGPU (e.g., AMD Radeon RX 7900 XT)

# Confirm DRM device nodes created
ls -la /dev/dri/
# Output: card0 (iGPU), card1 (eGPU), renderD128, renderD129

# Check which GPU is which
cat /sys/class/drm/card1/device/vendor   # 0x1002 = AMD
cat /sys/class/drm/card1/device/device   # Device ID
```

### Step 4: Kernel Boot Parameters (if needed)

Some systems require kernel boot parameters for correct PCIe resource allocation, particularly for large GPU BARs:

```bash
# /etc/default/grub
# Necessary on some systems where the Thunderbolt root port
# is not assigned enough MMIO window space at boot
GRUB_CMDLINE_LINUX_DEFAULT="... pci=assign-busses,hpbussize=0x33,realloc,hpmmiosize=128M,hpmmioprefsize=16G pcie_ports=native"
```

After editing, run `sudo update-grub` (Debian/Ubuntu) or `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` (Fedora/RHEL) and reboot.

The `pcie_ports=native` parameter is particularly important: without it, some systems leave PCIe port services under ACPI control, which can prevent the driver from transitioning the GPU out of D3cold.

### Step 5: Wayland Session with eGPU

```bash
# For a specific application: use eGPU for rendering
DRI_PRIME=1 vulkan-smoketest

# Use PCI path (more stable)
DRI_PRIME=pci-0000_0c_00_0 glxgears

# Verify which GPU an application is using
DRI_PRIME=1 glxinfo | grep "OpenGL renderer"
# Output: OpenGL renderer string: AMD Radeon RX 7900 XT (...)

# For compute-only workloads on AMD (ROCm):
HIP_VISIBLE_DEVICES=1 python3 rocm_workload.py

# For the eGPU to drive an external monitor directly,
# connect the monitor to the enclosure — the compositor
# will detect it as a new output on card1 automatically
```

### Step 6: Surprise Removal Recovery

If the eGPU cable is disconnected while in use:

```bash
# Check dmesg for what the kernel reports
dmesg | tail -50
# Look for: "thunderbolt: PCIe tunnel ... destroyed"
#           "amdgpu: GPU removed while running"
#           "pciehp: Slot(...): adapter removed"

# If the compositor is frozen, try restarting it
# For GNOME:
systemctl restart gdm

# For sway:
swaymsg exit  # then re-login

# After reconnecting and re-authorizing:
boltctl authorize <uuid>

# Check if the GPU was re-enumerated
ls /dev/dri/
```

### Step 7: Using lstbt (Intel thunderbolt-utils)

Intel provides `thunderbolt-utils` with a `lstbt` tool for inspecting the Thunderbolt/USB4 topology:

```bash
# Install thunderbolt-utils (package name varies by distro)
# On Fedora: sudo dnf install thunderbolt-tools
# On Ubuntu: may need to build from source

# List topology
lstbt

# Verbose output with bandwidth info
lstbt -v
```

[Source: Phoronix coverage of thunderbolt-utils](https://www.phoronix.com/news/Intel-Linux-thunderbolt-utils)

---

## Roadmap

### Near-term (6–12 months)

- **NVIDIA open kernel module Thunderbolt 4/5 and USB4 eGPU detection**: A community PR (#981) to NVIDIA's open-gpu-kernel-modules targets extending eGPU recognition beyond Thunderbolt 3 bridges, adding proper hotplug timeout handling and stability improvements for TB4/TB5/USB4 connections that currently hard-lock during CUDA operations. [Source](https://github.com/NVIDIA/open-gpu-kernel-modules/pull/981)
- **AMDGPU hot device unplug hardening**: The long-running RFC patch series by Andrey Grodzovsky for graceful eGPU unplug handling in `amdgpu` (avoiding application crashes and GEM buffer object corruption on surprise removal) continues to be refined toward merge readiness. [Source](https://www.phoronix.com/news/AMDGPU-Hot-Unplug-V2)
- **NVIDIA thunderbolt hotplug issue resolution**: Issue #842 in NVIDIA's open-gpu-kernel-modules tracks a recurring problem where reconnecting a Thunderbolt eGPU after sleep/suspend leaves the module in a "fallen off the bus" state requiring a reboot. Active community attention and upstream triage are ongoing. [Source](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/842)
- **Wider USB4 v2 / Thunderbolt 5 enablement**: The USB4 v2 host controller driver support (merged for basic operation) is expected to gain more complete PCIe bandwidth allocation and Bandwidth Boost mode control, reducing the performance gap between eGPU and internal PCIe 4.0 ×16 slots. Note: needs verification of exact kernel milestone.

### Medium-term (1–3 years)

- **DRM device lifecycle formalisation**: The `drm_dev_enter`/`drm_dev_exit` reference-counting infrastructure needs to be uniformly adopted across all GPU drivers (not just `amdgpu`) to allow clean surprise-removal without hung fds. A generalised DRM hot-unplug helper layer is under informal discussion on dri-devel. Note: needs verification of current RFC status.
- **Wayland compositor eGPU hot-plug recovery**: The wlroots GPU hot-plug PR (#2423) merged basic device-arrival handling; graceful renderer migration on mid-session GPU removal (compositor surviving an eGPU unplug while a client is rendering to it) remains an open design problem. [Source](https://egpu.io/forums/thunderbolt-linux-setup/linux-egpus-and-wayland-in-2026/)
- **IOMMU / PCIe ACS enforcement for Thunderbolt**: Tightening Access Control Services (ACS) policy for Thunderbolt PCIe subtrees to guarantee peer-to-peer DMA isolation on multi-GPU eGPU systems is being discussed as part of broader vIOMMU/PCIe security hardening work in the kernel. Note: needs verification.
- **`thunderbolt-utils` lstbt bandwidth reporting**: Intel's `thunderbolt-utils` tooling is expected to gain structured bandwidth query support for USB4 v2 Bandwidth Boost mode, enabling userspace tools and compositors to make better scheduling decisions. [Source](https://www.phoronix.com/news/Intel-Linux-thunderbolt-utils)
- **Bolt 2.0 / unified USB4 device management**: The `bolt` daemon may be extended or superseded by a unified USB4 device management layer that covers both Thunderbolt-legacy and native USB4 host controllers under a single D-Bus API. Note: needs verification.

### Long-term

- **Native USB4 PCIe tunneling without Thunderbolt firmware**: As USB4-native (non-Thunderbolt) host controllers proliferate on ARM and RISC-V SoCs, the kernel USB4 subsystem may need to manage PCIe tunnel allocation entirely in software without relying on Thunderbolt NHI firmware, enabling eGPU connectivity on a broader range of hardware. Note: speculative direction.
- **eGPU as a first-class DRM render node with live migration**: Long-term, the graphics stack could treat eGPU attach/detach like a live VM migration: freezing client rendering, migrating GEM objects and Vulkan state to a new device, and resuming without application restart. This requires cooperation between Mesa, the kernel DRM core, and compositor protocols not yet designed. Note: speculative direction.
- **Standardised USB4 eGPU bandwidth negotiation API**: A userspace API (likely via a new `ioctl` on the Thunderbolt/USB4 character device) for applications to query and reserve PCIe + DisplayPort bandwidth shares across the USB4 fabric would allow display servers and compute runtimes to avoid bandwidth starvation conflicts dynamically. Note: speculative direction.

---

## 11. Integrations

**Chapter 4 (GPU Memory Management)** — DMA-BUF PRIME file descriptors are the mechanism by which the eGPU's GEM buffer objects are shared with the iGPU for reverse PRIME blit. The DMA-BUF attachment, mapping, and synchronization mechanisms described in Chapter 4 underpin every eGPU-to-iGPU frame transfer.

**Chapter 49 (Multi-GPU and PRIME Render Offload)** — The PRIME framework that eGPU reverse offload relies on is described in detail in Chapter 49, including `DRI_PRIME`, `xrandr --setprovideroffloadsink`, and the Mesa-level driver dispatch that selects between GPUs. eGPU is architecturally a special case of the multi-GPU PRIME topology.

**Chapter 5 (x86 GPU Drivers)** — The `amdgpu` and `i915`/`xe` PCIe probe and remove callbacks, and their interaction with the DRM hot-plug infrastructure, are covered in the context of the full driver architecture in Chapter 5. The `drm_dev_register`/`drm_dev_unplug` pattern described here is implemented by those drivers.

**Chapter 51 (GPU Power Management and Thermal)** — Runtime PM interactions between the eGPU and the Thunderbolt link (D3cold transitions, tunnel suspend on system sleep) connect to the broader GPU PM story in Chapter 51.

**Chapter 80 (GPU Security)** — The DMA attack surface of Thunderbolt PCIe tunneling, IOMMU-based mitigation, and the Thunderclap research referenced in this chapter are covered in depth in Chapter 80's section on hardware-level GPU security threats.

---

## References

- [USB4 and Thunderbolt — The Linux Kernel Documentation](https://docs.kernel.org/admin-guide/thunderbolt.html)
- [bolt — Thunderbolt 3 Device Manager (freedesktop mirror)](https://github.com/gicmo/bolt)
- [Linux DRM Internals — drm_dev_unplug / drm_dev_enter](https://docs.kernel.org/gpu/drm-internals.html)
- [Mika Westerberg: Initial USB4 v2 support patch series](https://lore.kernel.org/linux-usb/20230612082145.62218-20-mika.westerberg@linux.intel.com/T/)
- [Andrey Grodzovsky: RFC Support hot device unplug in amdgpu v6](https://lore.kernel.org/amd-gfx/20210510163625.407105-1-andrey.grodzovsky@amd.com/)
- [wlroots GPU hotplugging PR #2423](https://github.com/swaywm/wlroots/pull/2423)
- [all-ways-egpu: Configure eGPU as primary under Linux Wayland desktops](https://github.com/ewagner12/all-ways-egpu)
- [NVIDIA RTX 5080 Thunderbolt 5 hard-lock issue #979](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/979)
- [NVIDIA developer blog: Accelerating ML with eGPU on Linux](https://developer.nvidia.com/blog/accelerating-machine-learning-on-a-linux-laptop-with-an-external-gpu/)
- [CONFIG_USB4 — Linux Kernel Driver DataBase](https://cateee.net/lkddb/web-lkddb/USB4.html)
- [Thunderclap: IOMMU DMA vulnerability research (NDSS 2019)](https://www.ndss-symposium.org/ndss-paper/thunderclap-exploring-vulnerabilities-in-operating-system-iommu-protection-via-dma-from-untrustworthy-peripherals/)
- [Checking Thunderbolt security on Linux — Guy Rutenberg](https://www.guyrutenberg.com/2021/01/09/checking-thunderbolt-security-on-linux/)
- [Intel thunderbolt-utils release — Phoronix](https://www.phoronix.com/news/Intel-Linux-thunderbolt-utils)
- [NVIDIA Linux 2025 highlights (Nova driver progress) — Phoronix](https://www.phoronix.com/news/NVIDIA-Linux-2025-Highlights)
- [LWN: The modernization of PCIe hotplug in Linux](https://lwn.net/Articles/767904/)
- [eGPU.io community Linux forums](https://egpu.io/forums/thunderbolt-linux-setup/)
