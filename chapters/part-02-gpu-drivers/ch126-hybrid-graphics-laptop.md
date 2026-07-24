# Chapter 126: Hybrid Graphics and Laptop Power Management

> **Part**: Part II — GPU Drivers
> **Audience**: Laptop users and system integrators; kernel driver developers; power management engineers; anyone dealing with NVIDIA Optimus or AMD dual-GPU configurations.
> **Status**: First draft — 2026-06-19

## Table of Contents

- [Overview](#overview)
- [1. Introduction: The Dual-GPU Problem](#1-introduction-the-dual-gpu-problem)
  - [1.3 What is Hybrid Graphics?](#13-what-is-hybrid-graphics)
  - [1.4 What is PRIME Render Offload?](#14-what-is-prime-render-offload)
  - [1.5 What is Runtime Power Management (Runtime PM)?](#15-what-is-runtime-power-management-runtime-pm)
- [2. The Hardware Model: ACPI, MUX, and PCIe](#2-the-hardware-model-acpi-mux-and-pcie)
- [3. vga_switcheroo: The Kernel Switching Subsystem](#3-vga_switcheroo-the-kernel-switching-subsystem)
- [4. bbswitch: Forcible dGPU Power Cutoff](#4-bbswitch-forcible-dgpu-power-cutoff)
- [5. PRIME Render Offload: DRI_PRIME and Beyond](#5-prime-render-offload-dri_prime-and-beyond)
- [6. NVIDIA Optimus on Linux](#6-nvidia-optimus-on-linux)
- [7. envycontrol: Modern Hybrid GPU Management](#7-envycontrol-modern-hybrid-gpu-management)
- [8. TLP and power-profiles-daemon](#8-tlp-and-power-profiles-daemon)
- [9. AMD SmartShift and Radeon Laptops](#9-amd-smartshift-and-radeon-laptops)
- [10. Practical Diagnostics](#10-practical-diagnostics)
- [Integrations](#integrations)
- [References](#references)

---

## Overview

Hybrid graphics—shipping two GPUs in a single laptop—is now ubiquitous, but the Linux software stack to manage them spans the ACPI firmware layer, multiple kernel subsystems, several competing user-space tools, and non-trivial interactions with display servers. This chapter is a vertical tour from the PCIe topology and ACPI power methods down through the `vga_switcheroo` kernel subsystem and PRIME DMA-BUF buffer-sharing framework, up to user-space tools such as `envycontrol`, TLP, and `power-profiles-daemon`. Each layer is examined with real sysfs paths, module parameters, and environment variables so that both driver developers and system integrators can understand why the laptop's battery drains, why the dGPU does not spin down, or why a game ignores the NVIDIA card.

The chapter is organized from hardware upward:

- **Section 2** — grounds the discussion in ACPI power methods and the critical distinction between MUX and MUX-less display routing.
- **Sections 3 and 4** — cover the two historical kernel-side power control mechanisms (`vga_switcheroo` and `bbswitch`).
- **Section 5** — explains PRIME render offload and the Mesa `DRI_PRIME`/`MESA_VK_DEVICE_SELECT` environment variables in detail, including how the Mesa loader matches a GPU by sysfs bus address.
- **Sections 6 and 7** — cover NVIDIA's proprietary RTD3 runtime PM path and the `envycontrol` tool that automates it.
- **Sections 8 and 9** — turn to system-wide power management policy via TLP and `power-profiles-daemon`, and to AMD's open-stack hybrid approach with SmartShift.
- **Section 10** — is a practical diagnostics reference.

---

## 1. Introduction: The Dual-GPU Problem

Modern consumer laptops routinely include two graphics processors: an **integrated GPU (iGPU)** built into the CPU package—Intel Iris Xe / Arc, AMD Radeon 680M/890M, or Qualcomm Adreno—and a **discrete GPU (dGPU)** such as an NVIDIA GeForce RTX or AMD Radeon RX sitting on a separate PCIe endpoint. The iGPU is always powered because it is physically wired to the panel's eDP connector and must drive the display during idle use. The dGPU provides ten to fifty times more rendering throughput but draws thirty to one hundred watts of additional power.

The Linux challenge is fourfold:

1. **Display routing**: the panel is wired to the iGPU on most laptops, so the dGPU cannot simply scan out to the screen; its framebuffers must be copied through PRIME to the iGPU's display engine.
2. **Runtime power gating**: without explicit power management, the dGPU remains in D0 (powered on, clock gated) even at idle, adding five to fifteen watts to the platform's idle power draw—a serious battery penalty.
3. **Application targeting**: user-space applications must explicitly request the dGPU; by default the iGPU handles all rendering.
4. **Driver diversity**: NVIDIA's proprietary driver, the open `amdgpu` driver, and the open NVIDIA kernel modules each implement power management and PRIME offload differently.

Getting this wrong means either a laptop that discharges in two hours instead of eight (dGPU never powers down) or a gaming application that silently runs on the iGPU at one-tenth expected performance.

### 1.1 Brief History

The history of hybrid graphics on Linux divides into three eras:

**Era 1 (2009–2014): ATI/Intel switchable graphics.** AMD and Intel shipped laptops with a hardware MUX and `vga_switcheroo` support in the kernel. Switching required closing the X session and was a significant hassle. NVIDIA launched "Optimus" in 2010 for Windows with seamless switching, but Linux support was minimal—the proprietary NVIDIA driver did not register with `vga_switcheroo`.

**Era 2 (2013–2019): Bumblebee.** The community-driven Bumblebee project (bbswitch + optirun) provided the only practical way to use the NVIDIA dGPU on Linux. It worked for many users but was fragile on firmware updates and required manual module management.

**Era 3 (2019–present): PRIME render offload.** NVIDIA and the Mesa project both converged on the DMA-BUF buffer-sharing PRIME model. NVIDIA added RTD3 support for Turing GPUs (kernel 4.18+, driver 435+), delivering automatic runtime power gating without external tools. Modern distributions (Ubuntu 22.04+, Fedora 38+, Arch Linux) ship everything needed for on-demand PRIME offload out of the box, though the interaction of multiple power management layers still requires care.

### 1.2 Current Recommendation Matrix

| Platform | Best practice |
|----------|---------------|
| Intel + NVIDIA (Turing/Ampere/Ada) | `envycontrol --switch hybrid` or `prime-select on-demand`; `NVreg_DynamicPowerManagement=0x02` |
| Intel + NVIDIA (pre-Turing) | bbswitch for power gating; PRIME offload for rendering |
| AMD APU + AMD dGPU | `DRI_PRIME=1`; `amdgpu.runpm=1`; SmartShift via sysfs |
| AMD APU + NVIDIA | Same as Intel + NVIDIA; `amdgpu` drives iGPU natively |
| Laptop with hardware MUX | Vendor-specific tool (`supergfxctl`, BIOS setting) + above PM |

### 1.3 What is Hybrid Graphics?

Hybrid graphics refers to a laptop design in which two physically distinct GPU dies share a single display and power envelope. The integrated GPU (iGPU) is fabricated inside the CPU package and accesses system DRAM over a high-bandwidth internal fabric; the discrete GPU (dGPU) is a separate PCIe endpoint device with its own dedicated VRAM. The iGPU is always powered because it is hardwired to the internal display panel's eDP connector, while the dGPU is optional and can be power-gated when idle. On Linux, both devices appear as separate PCI endpoints with their own `/dev/dri/cardN` and `/dev/dri/renderDN` nodes under separate kernel drivers. The central challenge is that the operating system must simultaneously minimize dGPU idle power draw—which can exceed fifteen watts on modern parts—while allowing applications to explicitly target the more capable hardware for compute-intensive workloads. Getting the balance wrong leads to either battery drain when the dGPU is left in D0 permanently, or performance shortfalls when applications are silently routed to the iGPU. This chapter covers every software layer that participates in that balance, from ACPI firmware tables through the kernel runtime PM framework to the Mesa GPU-selection environment variables that applications and compositors rely on.

### 1.4 What is PRIME Render Offload?

PRIME is the kernel and Mesa framework that allows one GPU to produce a rendered frame and have it displayed through a second GPU's display engine. The mechanism is built on DMA-BUF buffer sharing: the render GPU allocates a framebuffer in its address space, exports it as a DMA-BUF file descriptor, and the display GPU imports it for scanout. On the kernel side, this requires both DRM devices to implement `drm_prime_fd_to_handle` and `drm_prime_handle_to_fd` so that buffer handles can cross driver boundaries. On the user-space side, the Mesa loader selects the render GPU based on the `DRI_PRIME` environment variable for OpenGL and `MESA_VK_DEVICE_SELECT` for Vulkan enumeration. The `DRI_PRIME=pci-0000_01_00_0` form pins a specific PCIe device by sysfs bus address; `DRI_PRIME=1` selects the first non-default GPU. PRIME offload eliminates the need for a hardware MUX on MUX-less laptops because the iGPU's display engine handles all panel scanout regardless of which GPU produced the pixels. Section 5 covers the Mesa loader code paths, the `DRM_CLIENT_CAP_ATOMIC` offload coordinator, and the Vulkan device selection mechanism in detail.

### 1.5 What is Runtime Power Management (Runtime PM)?

Runtime power management (Runtime PM) is the Linux kernel framework that allows individual device drivers to power down a device when it is idle and restore power transparently when a new request arrives. For hybrid GPU laptops, Runtime PM is the mechanism that puts the discrete GPU into D3cold—the state in which the PCIe slot's VCC power rail is fully cut via the ACPI `_PS3` method—achieving the lowest possible idle power draw. The framework is implemented in `drivers/base/power/runtime.c` and exposes per-device callbacks (`runtime_suspend`, `runtime_resume`, `runtime_idle`) that a driver registers through `dev_pm_ops`. A device can only reach D3cold when three conditions hold: the ACPI table declares a `_PR3` power resource for the device, the PCIe bridge above it also allows D3cold, and the sysfs file `d3cold_allowed` reads `1`. For NVIDIA GPUs, the proprietary driver implements Runtime PM through a mechanism called RTD3 (Runtime D3), controlled by the `NVreg_DynamicPowerManagement` module parameter. For AMD discrete GPUs, `amdgpu` uses the `runpm` module parameter. Both paths ultimately invoke the same kernel Runtime PM core. Sections 3 through 7 examine how each driver and user-space tool layer interacts with this framework, and Section 10 provides diagnostic commands for verifying that each condition is met.

---

## 2. The Hardware Model: ACPI, MUX, and PCIe

### 2.1 PCIe Topology

Both GPUs are PCIe endpoint devices visible to `lspci`. On a typical Intel/NVIDIA hybrid:

```bash
# Identify GPU PCI addresses
lspci -nn | grep -i vga
# 00:02.0 VGA compatible controller [0300]: Intel Corporation Alder Lake-P GT2 [Iris Xe Graphics] [8086:46a6]
# 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA107M [GeForce RTX 3050 Mobile] [10de:25a2]
```

The iGPU at `00:02.0` is integrated into the CPU package. The dGPU at `01:00.0` hangs off the CPU's PCIe root port (`00:06.0` in many Intel designs) via a PCIe ×4 or ×8 Gen 4 link. Each device gets its own `/dev/dri/cardN` and `/dev/dri/renderDN` nodes under separate kernel drivers (`xe`/`i915` for Intel, `nvidia` or `nouveau` for NVIDIA, `amdgpu` for AMD).

### 2.2 ACPI Power Control

The laptop firmware (UEFI + ACPI tables) owns the dGPU's power rail. Two critical ACPI methods, `_PS0` and `_PS3`, apply D0 (full power) and D3 (power off) to the device:

```
Device (PEGP) {          // typically \_SB.PCI0.PEG0.PEGP on Intel systems
    Method (_PS0) { ... }  // assert power, release PCIe reset
    Method (_PS3) { ... }  // cut power rail to dGPU
    Method (_PR0) { Package { P3MO } }  // power resource for D0
    Method (_PR3) { Package { P3MO } }  // power resource for D3cold
}
```

`_PR3` is the key method that enables **D3cold**: if the ACPI table declares `_PR3`, the kernel's runtime PM framework (and NVIDIA's RTD3 logic) can cut the PCIe slot's VCC power entirely, achieving the lowest possible idle draw. Without `_PR3`, the best achievable power state is D3hot (clock and logic gated but power rail live). [Source: NVIDIA RTD3 README](https://download.nvidia.com/XFree86/Linux-x86_64/435.17/README/dynamicpowermanagement.html)

### 2.3 Display Architecture: MUX vs. MUX-less

**MUX-less (most laptops)**: The iGPU's display engine is hard-wired to the panel. The dGPU has no direct path to the display connector. To render a frame on the dGPU and show it on screen, the result must be shared via a DMA-BUF with the iGPU and composited there. This is the PRIME render offload model (Section 5).

**MUX (hardware multiplexer)**: A small switch chip physically selects which GPU's eDP signal drives the panel. ASUS ROG, Razer Blade, MSI gaming laptops, and similar "MUX switch" designs expose a firmware setting that controls the mux. On Linux, toggling the MUX requires either a vendor ACPI method (exposed via tools like `supergfxctl` on ASUS) or a firmware/BIOS change—there is no universal kernel abstraction. Switching the MUX typically requires a logout or reboot to transfer display ownership cleanly between drivers.

```bash
# Check vendor-specific MUX capabilities (ASUS example)
cat /sys/bus/platform/devices/asus-nb-wmi/dgpu_disable
# 0 = dGPU enabled, 1 = dGPU disabled (integrated-only mode)
```

### 2.3.1 D3cold Allowed Flag

The kernel exposes a sysfs file `d3cold_allowed` per PCI device that gates whether the device may enter D3cold. This must be `1` for any runtime PM path (NVIDIA RTD3 or `amdgpu runpm`) to achieve full power cutoff:

```bash
cat /sys/bus/pci/devices/0000:01:00.0/d3cold_allowed
# 1  ← required for D3cold; 0 means stuck at D3hot at best

# If 0, the ACPI _PR3 method is likely absent or the bridge forbids it
# Check parent bridge
cat /sys/bus/pci/devices/0000:00:06.0/d3cold_allowed
# 1  ← bridge must also allow D3cold
```

The kernel sets `d3cold_allowed` to 0 if the ACPI table lacks `_PR3`, if the PCIe bridge above the device does not support D3cold, or if `pci=nod3cold` is on the kernel command line. All three must be correct for the full power-off path to work.

### 2.4 The ACPI Device Path

The dGPU's ACPI path varies by OEM and firmware. Common forms:

| OEM style | ACPI path |
|-----------|-----------|
| Intel PEG slot | `\_SB.PCI0.PEG0.PEGP` |
| Intel RP06 | `\_SB.PCI0.RP06.PXSX` |
| AMD platforms | `\_SB.PCI0.GPP1.PEGP` |

To inspect: `cat /sys/bus/pci/devices/0000:01:00.0/firmware_node/path` (or `acpi_path` in older kernels).

---

## 3. vga_switcheroo: The Kernel Switching Subsystem

### 3.1 What It Does

`vga_switcheroo` is the kernel subsystem for managing GPU switching on laptops with a hardware multiplexer or driver-controlled power rails. It lives in `drivers/gpu/vga/vga_switcheroo.c` and is controlled by a debugfs interface. [Source: kernel doc](https://www.kernel.org/doc/html/latest/gpu/vga-switcheroo.html)

The subsystem models two roles:

- **Handler**: provided by a platform driver (e.g., an Apple SMU driver or an ACPI-based handler for AMD laptops); owns the MUX switch and power rail.
- **Clients**: GPU drivers that register themselves as switchable. Currently `radeon`, `amdgpu`, and `nouveau` register as clients; `nvidia` (proprietary) does **not**.

### 3.2 Registration API

```c
/* Register the platform handler (mux + power control) */
vga_switcheroo_register_handler(const struct vga_switcheroo_handler *handler,
                                enum vga_switcheroo_handler_flags_t handler_flags);

/* Register a GPU driver as a switcheroo client */
vga_switcheroo_register_client(struct pci_dev *pdev,
                               const struct vga_switcheroo_client_ops *ops,
                               bool driver_power_control);

/* Register the HDA audio device on the dGPU */
vga_switcheroo_register_audio_client(struct pci_dev *pdev,
                                     const struct vga_switcheroo_client_ops *ops,
                                     struct vga_switcheroo_client_id id);
```

The `driver_power_control` flag (true on modern AMD dGPU + iGPU setups) means the driver itself manages power state transitions rather than delegating to the handler.

### 3.3 debugfs Interface

The control file is `/sys/kernel/debug/vga_switcheroo/switch`. Valid commands to write:

| Command | Effect |
|---------|--------|
| `IGD`   | Immediate switch to iGPU. Power on iGPU if needed, power off dGPU. |
| `DIS`   | Immediate switch to dGPU. |
| `DIGD`  | Delayed switch to iGPU. Deferred until all user-space processes have closed dGPU device files and the audio client releases. |
| `DDIS`  | Delayed switch to dGPU (same deferral logic). |
| `MIGD`  | Mux-only switch to iGPU. Does not remap console or change power state. |
| `MDIS`  | Mux-only switch to dGPU. |
| `OFF`   | Power off the device not currently in use. |
| `ON`    | Power on the device not currently in use. |

```bash
# Power off the dGPU (must be on iGPU already)
echo OFF | sudo tee /sys/kernel/debug/vga_switcheroo/switch

# Read current state
cat /sys/kernel/debug/vga_switcheroo/switch
# 0:IGD:+:Pwr:0000:00:02.0
# 1:DIS: :Off:0000:01:00.0
```

The `+` marks the active GPU; `Pwr` means powered, `Off` means powered down.

### 3.4 The Two-Stage Switch Sequence

When `vga_switcheroo` performs an immediate switch (e.g., `IGD`), it executes a two-stage sequence internally:

1. **Stage 1 (`vga_switchto_stage1()`)**: Powers on the target GPU if it is off, sets it to VGAARB (VGA arbitration) state, and begins draining existing rendering. If `mux_id` is non-null, the physical MUX line is toggled here.
2. **Stage 2 (`vga_switchto_stage2()`)**: Completes the handoff — remaps the VGA console to the new GPU, sets the old GPU's power state to off (calling the handler's `power_state` callback), and notifies DRM of the change.

Delayed switches (`DIGD`/`DDIS`) skip to stage 2 via `vga_switcheroo_process_delayed_switch()` only after all DRM clients on the source GPU have released their file descriptors and the reference count falls to zero:

```c
/* Called by DRM on last fd close of a switcheroo-registered device */
void vga_switcheroo_process_delayed_switch(void)
{
    mutex_lock(&vgasr_mutex);
    if (!vgasr_priv.delayed_switch_active)
        goto err;
    /* only switch once all clients have quiesced */
    if (vgasr_priv.delayed_client_id == VGA_SWITCHEROO_IGD)
        vga_switchto_stage2(vgasr_priv.client_id_list);
    ...
}
```

This prevents the display from going black mid-session if the compositor is still rendering to the old GPU.

### 3.5 Driver Power Control vs. Handler Power Control

The `driver_power_control` boolean in `vga_switcheroo_register_client()` has important implications. When `false` (legacy mode, used by older `radeon`/ATI drivers), the switcheroo handler itself calls the platform ACPI power methods to power the GPU on or off. When `true` (modern AMD `amdgpu` hybrid mode), the driver's own `pm_runtime_*` callbacks manage the power transitions—the switcheroo subsystem coordinates state but defers to the driver for the actual PCIe power state change. This is why on modern AMD APU + dGPU laptops, the `amdgpu` driver's `runpm` path (Section 9) integrates cleanly with `vga_switcheroo` registration without requiring a separate ACPI handler.

### 3.6 DDC Lock for EDID Probing

A subtle capability: `vga_switcheroo_lock_ddc()` / `vga_switcheroo_unlock_ddc()` temporarily routes DDC (display data channel) lines to an inactive GPU so its driver can read the panel's EDID without powering on the full GPU. Handlers declare support via `VGA_SWITCHEROO_CAN_SWITCH_DDC`.

### 3.7 Scope and Limitations

`vga_switcheroo` was designed for pre-PRIME era laptops (2009–2014 ATI/AMD hybrid designs and some Intel/NVIDIA laptops). NVIDIA's proprietary driver never registered with it. Modern Intel+NVIDIA MUX-less systems use PRIME offload (Section 5) combined with NVIDIA's runtime D3 power management (Section 6) rather than `vga_switcheroo`. The subsystem remains relevant for:

- AMD APU + discrete AMD Radeon ("SmartShift" tier-1 laptops) running the open `amdgpu` driver on both devices.
- Older ATI/AMD switchable graphics laptops still running `radeon` + `amdgpu`.

Enable with `CONFIG_VGA_SWITCHEROO=y`.

---

## 4. bbswitch: Forcible dGPU Power Cutoff

### 4.1 Origin and Purpose

`bbswitch` is an out-of-tree Linux kernel module from the Bumblebee Project ([https://github.com/Bumblebee-Project/bbswitch](https://github.com/Bumblebee-Project/bbswitch)). Its purpose is to cut power to the NVIDIA dGPU by invoking the ACPI `_DSM` (Device-Specific Method) that controls the PCIe slot's power rail. On laptops where the NVIDIA driver's own runtime PM cannot achieve D3cold, `bbswitch` provides a manual override.

### 4.2 The procfs Interface

After loading the module (`modprobe bbswitch`), the control interface appears at `/proc/acpi/bbswitch`:

```bash
# Cut power to NVIDIA dGPU
echo OFF | sudo tee /proc/acpi/bbswitch

# Restore power
echo ON | sudo tee /proc/acpi/bbswitch

# Check current state
cat /proc/acpi/bbswitch
# 0000:01:00.0 OFF
```

Before issuing `OFF`, the `nouveau` or `nvidia` module must be unloaded (`rmmod nvidia_uvm nvidia_drm nvidia_modeset nvidia`), as a live driver holds an active device reference that would prevent power cutoff.

The bbswitch module exposes two module parameters that control GPU power state at module load/unload time:

```
# /etc/modprobe.d/bbswitch.conf
# load_state: -1=don't change, 0=turn card OFF, 1=turn card ON
# unload_state: same semantics
options bbswitch load_state=0 unload_state=1
```

With `load_state=0 unload_state=1`, the dGPU powers off when `bbswitch` loads (at boot) and returns to powered-on state when the module is removed. This automates the dGPU power cut without requiring manual writes to `/proc/acpi/bbswitch`. [Source: bbswitch README](https://github.com/Bumblebee-Project/bbswitch)

### 4.3 ACPI Mechanism

bbswitch calls the `_DSM` method at `\_SB.PCI0.PEG0.PEGP` (or whichever path contains the dGPU) with a specific GUID and function index that corresponds to "power off PCIe device." On many Optimus laptops this is the ACPI Optimus DSM set with GUID `{A486D8F8-0BDA-471B-A72B-6042A6B5BEE0}`. The ACPI `_PS3` method then cuts the power rail.

### 4.4 The Bumblebee Project

Bumblebee combined bbswitch with the `optirun` / `primusrun` command-line wrappers to route application rendering to the NVIDIA GPU via VirtualGL or DRI2:

```bash
# Run an application on the dGPU via Bumblebee
optirun glxgears
primusrun %command%   # Steam launch option
```

The `bumblebeed` daemon maintained the dGPU power state and managed module loading. Bumblebee was the primary way to use Optimus on Linux before kernel 5.5's PRIME render offload matured.

### 4.5 Status and Caveats

Bumblebee and bbswitch are effectively superseded by PRIME render offload + NVIDIA RTD3 (Sections 5 and 6) for Turing (RTX 20xx) and newer GPUs. However, bbswitch remains a fallback for:

- Pre-Turing NVIDIA mobile GPUs that lack RTD3 `_PR3` ACPI support.
- Systems where the NVIDIA proprietary driver's runtime PM fails to reach D3cold due to firmware bugs.

**Risk**: Not all firmware implementations gracefully handle forced power cutoff. On some systems, bbswitch leaves the PCIe link in a wedged state that prevents the dGPU from coming back up without a cold reboot. The `acpi_call` module (a general-purpose ACPI method invoker at `/proc/acpi/call`) is an alternative for systems where bbswitch's hard-coded GUID does not match the OEM's DSM.

---

## 5. PRIME Render Offload: DRI_PRIME and Beyond

### 5.1 What PRIME Is

PRIME is the DRM subsystem's framework for cross-device buffer sharing via DMA-BUF file descriptors. On a MUX-less hybrid laptop, the dGPU renders into a GEM buffer object, exports it as a DMA-BUF fd via `DRM_IOCTL_PRIME_HANDLE_TO_FD`, and the iGPU imports it via `DRM_IOCTL_PRIME_FD_TO_HANDLE`. The Wayland compositor running on the iGPU then composites it onto the desktop. This path incurs a PCIe DMA read but avoids a CPU copy on systems with coherent IOMMU topology. Chapter 49 covers the kernel internals in depth.

### 5.2 The PCIe Copy Cost

Before diving into the API, it is worth understanding the performance and power cost of MUX-less PRIME offload. When the dGPU finishes rendering a frame into its VRAM:

1. The DMA-BUF fd is passed to the iGPU compositor (e.g., Mutter or KWin).
2. The iGPU maps the fd via `DRM_IOCTL_PRIME_FD_TO_HANDLE`, which calls `drm_gem_prime_import()` in the iGPU's driver.
3. If the iGPU and dGPU share a coherent IOMMU domain (common on AMD platforms with the same dGPU as the iGPU brand), the DMA can be done without a CPU copy—the iGPU's display engine reads directly from the dGPU's VRAM via PCIe.
4. If coherent DMA is not possible (Intel iGPU + NVIDIA dGPU without P2P DMA support), the data must bounce through system RAM, adding a CPU copy step.

At 4K 60 Hz, a single uncompressed frame is approximately 33 MB. Transferring it over PCIe Gen 4 ×4 (~8 GB/s) takes roughly 4 ms, close to the 16.7 ms frame budget. At 60 fps the effective PCIe bandwidth requirement is ~2 GB/s. In practice, compressed tile formats (AFRC, DCC) reduce this, and the transfer is pipelined with rendering on the dGPU. But the PCIe copy is a real cost that MUX-capable laptops eliminate by letting the dGPU scan out directly.

### 5.3 Mesa: DRI_PRIME

For OpenGL and Vulkan applications using Mesa (i.e., AMD, Intel, and Nouveau open-stack):

```bash
# Run on the secondary (non-default) GPU — most commonly the dGPU
DRI_PRIME=1 glxgears

# Target a specific GPU by PCIe bus address (underscores replace colons/dots)
DRI_PRIME=pci-0000_01_00_0 glxinfo -B

# Target by vendor:device hex IDs
DRI_PRIME=1002:687f glxgears   # AMD Vega 56 by PCI IDs

# Vulkan: expose only the selected GPU
DRI_PRIME=1! vulkaninfo
```

The `DRI_PRIME` env var is parsed by the Mesa loader in `src/loader/loader.c`. The logic is:

1. **Integer form** (`DRI_PRIME=1`): Mesa opens `/dev/dri/renderD128` for device 0 (the default), then iterates `renderD129`, `renderD130`, ... returning the Nth non-default device.
2. **PCI bus form** (`DRI_PRIME=pci-0000_01_00_0`): Mesa reads `/sys/dev/char/<major>:<minor>/device/` for each `renderD*` node and compares the symlink target's basename against the requested string. Non-alphanumeric characters in the PCI address are replaced with underscores to form the sysfs match (e.g., `0000:01:00.0` becomes `pci-0000_01_00_0`).
3. **ID form** (`DRI_PRIME=vendor_id:device_id`): Mesa reads the `vendor` and `device` sysfs attributes under `/sys/bus/pci/devices/<addr>/` and selects the first GPU whose hex IDs match.

After device selection, Mesa opens the appropriate `renderD*` node and proceeds through the normal DRI3 driver loading path. The `!` suffix (`DRI_PRIME=1!`) causes the Mesa Vulkan implicit layer to additionally filter `vkEnumeratePhysicalDevices` so only the selected device is returned, preventing non-PRIME-aware Vulkan applications from accidentally reverting to the iGPU. [Source: Mesa env vars](https://docs.mesa3d.org/envvars.html)

### 5.4 Mesa Vulkan: MESA_VK_DEVICE_SELECT

For Vulkan applications running on Mesa (RADV, ANV, Lavapipe), a separate Vulkan implicit layer handles device selection:

```bash
# List all Vulkan physical devices and their vid:did
MESA_VK_DEVICE_SELECT=list vulkaninfo

# Select AMD GPU by vendor:device (hex)
MESA_VK_DEVICE_SELECT=1002:687f %command%

# Force: expose ONLY the selected GPU to the application
MESA_VK_DEVICE_SELECT=8086:46a6! %command%
```

The format is `vendorid:deviceid` in hexadecimal (e.g., `1002:747e` for an AMD RX 7900 XT). Appending `!` causes `vkEnumeratePhysicalDevices` to return only the selected device. [Source: Mesa env vars](https://docs.mesa3d.org/envvars.html)

### 5.5 PRIME on X11: xrandr Provider Model

On X11, the RandR 1.4 provider model is required for PRIME offload. The iGPU's DDX (modesetting driver) is the **sink** (has display outputs); the dGPU's DDX is the **source** (render offload provider):

```bash
# List providers
xrandr --listproviders
# Providers: number : 2
# Provider 0: id: 0x1b8 cap: 0xf, Source Output, Sink Output, Source Offload, Sink Offload name:modesetting
# Provider 1: id: 0x1b7 cap: 0x2, Sink Offload name:nouveau

# Wire the dGPU as render offload source for the iGPU sink
xrandr --setprovideroffloadsink 1 0

# Now DRI_PRIME=1 routes to provider 1
DRI_PRIME=1 glxgears
```

### 5.6 PRIME on Wayland

Under Wayland, the compositor handles device selection. For wlroots-based compositors (Sway, Hyprland, river):

```bash
# Force wlroots to use a specific DRM device as its primary
WLR_DRM_DEVICES=/dev/dri/card0:/dev/dri/card1 sway
```

KDE Plasma 6 and GNOME 47+ handle PRIME device selection internally; applications can still use `DRI_PRIME` to request the secondary GPU. The Wayland protocol itself has no GPU selection primitive—the compositor imports DMA-BUF buffers from whichever device the client renders on.

### 5.7 prime-run Helper

The `prime-run` shell script (shipped by many distributions as part of the `nvidia-prime` package) assembles the correct environment variables for Mesa-based offload:

```bash
#!/bin/sh
# /usr/bin/prime-run (Mesa / open-driver variant)
DRI_PRIME=1 exec "$@"
```

For NVIDIA, a similar wrapper sets `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia`.

---

## 6. NVIDIA Optimus on Linux

### 6.1 Architecture

NVIDIA Optimus is NVIDIA's trade name for their hybrid graphics implementation. The key distinction from AMD's approach: NVIDIA's proprietary driver (`nvidia.ko`) does **not** register with `vga_switcheroo`. Power management and buffer sharing are handled entirely within the NVIDIA driver stack and the GLVND (GL Vendor-Neutral Dispatch) library layer.

### 6.2 PRIME Offload Environment Variables

```bash
# Required for OpenGL offload (both vars needed together)
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxgears

# Vulkan offload: __NV_PRIME_RENDER_OFFLOAD automatically loads VK_LAYER_NV_optimus
__NV_PRIME_RENDER_OFFLOAD=1 vulkaninfo

# Control which GPUs the Vulkan layer exposes
__VK_LAYER_NV_optimus=NVIDIA_only    # hide non-NVIDIA GPUs from the app
__VK_LAYER_NV_optimus=non_NVIDIA_only  # hide NVIDIA GPUs

# Target a specific NVIDIA GPU when multiple are present
__NV_PRIME_RENDER_OFFLOAD_PROVIDER="NVIDIA-G0"  # RandR provider name
```

[Source: NVIDIA PRIME render offload README](https://download.nvidia.com/XFree86/Linux-x86_64/535.98/README/primerenderoffload.html)

### 6.3 Prime Select Modes

The `nvidia-prime` package provides `prime-select` for toggling between three modes:

```bash
# Select integrated graphics only (best battery life)
sudo prime-select intel    # or: prime-select amd

# Select NVIDIA as primary GPU (drives the panel directly via MUX, if present)
sudo prime-select nvidia

# Hybrid: iGPU drives panel, dGPU available for offload on-demand
sudo prime-select on-demand
```

`prime-select` writes a flag to `/etc/prime-discrete` and reconfigures `/etc/X11/xorg.conf.d/` appropriately. A display server restart is required to switch modes (logout on Wayland, server restart on X11).

### 6.4 RTD3: Runtime D3cold Power Management

NVIDIA's **RTD3** (Runtime D3) feature lets the proprietary driver cut PCIe power to the dGPU when no application is using it, achieving D3cold. Requirements: [Source: NVIDIA RTD3 README](https://download.nvidia.com/XFree86/Linux-x86_64/435.17/README/dynamicpowermanagement.html)

- Notebook platform only.
- ACPI firmware must declare `_PR3` on the dGPU device.
- Intel Coffee Lake (8th gen) chipset or newer, or AMD equivalent.
- Turing GPU (RTX 20xx Mobile) or newer.
- Linux kernel ≥ 4.18 with `CONFIG_PM=y`.

The `NVreg_DynamicPowerManagement` module parameter controls RTD3 aggressiveness:

| Value | Behaviour |
|-------|-----------|
| `0x00` | Default — no runtime power management, GPU stays in D0. |
| `0x01` | Power down when no applications hold the GPU open. |
| `0x02` | Coarse-grained: power down when all applications are idle, even if some hold the device open. Recommended for most laptops. |
| `0x03` | Fine-grained (Ampere/Ada and newer): deeper power cuts during idle periods within an active workload. |

Configure in `/etc/modprobe.d/nvidia.conf`:

```
# /etc/modprobe.d/nvidia.conf
options nvidia NVreg_DynamicPowerManagement=0x02
```

Enable runtime PM for all NVIDIA PCI functions:

```bash
# Enable runtime PM on GPU (repeat for audio/USB/serial functions if present)
echo auto | sudo tee /sys/bus/pci/devices/0000:01:00.0/power/control
echo auto | sudo tee /sys/bus/pci/devices/0000:01:00.1/power/control  # HDA audio
```

Verify D3cold state:

```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# suspended

cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_active_time
# 1843521   (milliseconds GPU has been in D0)

nvidia-smi --query-gpu=power.draw --format=csv,noheader,nounits
# 5.2  (watts; low idle value confirms near-suspended state)
```

### 6.5 GLVND: GL Vendor-Neutral Dispatch

NVIDIA's proprietary OpenGL relies on **GLVND** (GL Vendor-Neutral Dispatch, [https://github.com/NVIDIA/libglvnd](https://github.com/NVIDIA/libglvnd)), the dispatch library that routes OpenGL calls to the correct vendor ICD. When `__GLX_VENDOR_LIBRARY_NAME=nvidia` is set, GLVND loads `libGLX_nvidia.so.0` as the GLX vendor for the offloaded application context instead of `libGLX_mesa.so.0`. Without this variable, GLVND defaults to the vendor configured for the X screen (the iGPU's modesetting DDX, backed by Mesa), and the application renders on the iGPU regardless of `__NV_PRIME_RENDER_OFFLOAD`.

The GLVND `libEGL.so.1` dispatch uses a similar mechanism: NVIDIA provides `libEGL_nvidia.so.0`, and the ICD is selected by the `EGL_VENDOR` attribute of the device. For Wayland native applications using EGL, the correct ICD is automatically selected when the Wayland compositor has handed the application a `wl_drm` or `zwp_linux_dmabuf_v1` surface tied to the NVIDIA device fd.

### 6.6 nvidia-settings and PowerMizer

`nvidia-settings` exposes PowerMizer modes and NV-CONTROL attributes via the NV-CONTROL X extension:

```bash
# GpuPowerMizerMode values: 0=Adaptive, 1=Prefer Maximum Performance, 2=Auto (default)
nvidia-settings -a "[gpu:0]/GpuPowerMizerMode=0"  # Adaptive: driver-managed clock scaling

# Query driver version via NV-CONTROL attribute NV_CTRL_STRING_NVIDIA_DRIVER_VERSION
nvidia-settings -q NvidiaDriverVersion

# Query GPU utilization
nvidia-settings -q GPUUtilization

# Per-second power/utilization monitoring via nvidia-smi
nvidia-smi dmon -s pu -d 1   # power (w) + utilization (%)
```

The `NV_CTRL_ATTR_NV_MAJOR_VERSION` NV-CONTROL attribute (integer) reports the major version component of the NVIDIA driver. This is typically used by display managers and session tools (like `prime-select`) to check whether the NVIDIA driver is active and correctly loaded before switching modes.

---

## 7. envycontrol: Modern Hybrid GPU Management

### 7.1 What It Is

`envycontrol` ([https://github.com/bayasdev/envycontrol](https://github.com/bayasdev/envycontrol)) is a Python-based command-line tool that supersedes the scattered combination of `bbswitch` + manual `prime-select` + hand-edited udev rules. It supports GNOME, KDE Plasma, SDDM, GDM, and LightDM display managers and handles all three hybrid modes: integrated, hybrid (PRIME on-demand), and dedicated NVIDIA.

### 7.2 Modes

```bash
# Mode 1: Integrated only — dGPU completely powered off, best battery life
sudo envycontrol --switch integrated

# Mode 2: Hybrid — iGPU drives panel, dGPU available via PRIME offload with RTD3
sudo envycontrol --switch hybrid

# Mode 3: NVIDIA — dGPU drives panel (requires hardware MUX or PRIME reverse sync)
sudo envycontrol --switch nvidia

# Reset all changes
sudo envycontrol --reset

# Query current mode
envycontrol --query
```

A reboot is required after switching modes.

### 7.3 What Each Mode Writes

**Integrated mode** (`--switch integrated`):

- Creates `/etc/modprobe.d/blacklist-nvidia.conf` — blacklists `nvidia`, `nvidia_drm`, `nvidia_modeset`, `nvidia_uvm` modules.
- Writes `/lib/udev/rules.d/50-remove-nvidia.rules` — removes the NVIDIA device from the PCI bus via `power/remove` or `power/control`.
- Removes any `/etc/X11/xorg.conf.d/10-nvidia.conf` previously created.

**Hybrid mode** (`--switch hybrid`):

- Removes the module blacklist.
- Writes `/lib/udev/rules.d/80-nvidia-pm.rules` — sets `power/control=auto` on the NVIDIA PCI device to enable RTD3 at boot.
- Creates `/etc/modprobe.d/nvidia.conf` with `NVreg_DynamicPowerManagement=0x02`.

**NVIDIA mode** (`--switch nvidia`):

- Removes blacklist and PM udev rules.
- Writes `/etc/X11/xorg.conf.d/10-nvidia.conf` configuring the NVIDIA driver as the primary.
- On Wayland, creates `/etc/udev/rules.d/61-gdm.rules` overrides to force GDM into X11 mode if needed (GDM historically disabled Wayland when NVIDIA was primary).

### 7.4 Cache and State

`envycontrol` stores its current mode in `/var/cache/envycontrol/cache.json` so subsequent invocations and `--query` can report the active configuration without parsing system state.

---

## 8. TLP and power-profiles-daemon

### 8.1 TLP

TLP ([https://linrunner.de/tlp/](https://linrunner.de/tlp/)) is a comprehensive user-space power management daemon that applies dozens of kernel tunables at boot and on AC/battery transitions. For GPU power management, the relevant settings in `/etc/tlp.conf`:

```ini
# Enable PCIe runtime PM for all devices when on battery (includes dGPU)
RUNTIME_PM_ON_BAT=auto
# Keep everything always-on when on AC
RUNTIME_PM_ON_AC=on

# Exclude the NVIDIA dGPU from autosuspend if RTD3 is managed by the NVIDIA driver itself
# (avoid conflicting with NVreg_DynamicPowerManagement)
RUNTIME_PM_DRIVER_DENYLIST="amdgpu mei_me nouveau nvidia xhci_hcd"

# Or exclude by PCI address
RUNTIME_PM_DENYLIST="0000:01:00.0 0000:01:00.1"

# Force autosuspend for a specific device (override denylist)
RUNTIME_PM_ENABLE="0000:01:00.0"

# CPU governor on battery
CPU_SCALING_GOVERNOR_ON_BAT=powersave

# PCIe Active State Power Management
PCIE_ASPM_ON_BAT=powersupersave
```

> **Note**: When NVIDIA's `NVreg_DynamicPowerManagement=0x02` is active, TLP's `RUNTIME_PM_ON_BAT=auto` combined with `RUNTIME_PM_DRIVER_DENYLIST` containing `nvidia` is the correct configuration—TLP steps back and lets the NVIDIA driver handle its own PM, while TLP handles the rest of the PCIe hierarchy.

Diagnostic commands:

```bash
# Show all GPU power states
tlp-stat -g

# Show runtime PM status for PCI devices
tlp-stat -e | grep -i nvidia
```

### 8.2 power-profiles-daemon

`power-profiles-daemon` ([https://gitlab.freedesktop.org/upower/power-profiles-daemon](https://gitlab.freedesktop.org/upower/power-profiles-daemon)) is the D-Bus service backing GNOME Settings' and KDE's "Power Mode" toggle. It exposes three profiles over D-Bus:

```bash
# List available profiles and current setting
powerprofilesctl list
#   performance:
#     Driver:     amd_pstate
#     Degraded:   no
# * balanced:
#     Driver:     amd_pstate
#   power-saver:
#     Driver:     amd_pstate

# Switch profiles
powerprofilesctl set power-saver
powerprofilesctl set performance
```

The D-Bus interface:

```
org.freedesktop.UPower.PowerProfiles
  .ActiveProfile  (read-write string property)
  .Profiles       (array of profile dictionaries)
  .PerformanceDegraded (string, empty if not degraded)
```

**GPU interaction**: In `performance` mode, `power-profiles-daemon` sets the platform `amd_pstate` or `intel_pstate` driver to maximum performance bias, and since kernel 6.9 can also set `amdgpu`'s power profile via `power_dpm_force_performance_level`. In `power-saver` mode, it lowers CPU/GPU clocks and—on AMD Ryzen laptops—enables the AMDGPU panel power-saving feature (APU display panel brightness reduction). The `performance` profile does not itself disable runtime PM for the NVIDIA dGPU; that remains under `NVreg_DynamicPowerManagement`.

### 8.3 Battery Threshold Control

Many modern laptops expose battery charge limits via sysfs:

```bash
# Set charge limit to 80% to preserve long-term battery health
echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold

# TLP equivalent (applies on every boot)
# In /etc/tlp.conf:
START_CHARGE_THRESH_BAT0=60
STOP_CHARGE_THRESH_BAT0=80
```

---

## 9. AMD SmartShift and Radeon Laptops

### 9.1 Architecture

AMD's hybrid laptop story differs from Intel/NVIDIA in an important way: both the iGPU (inside the Ryzen APU) and the dGPU (discrete Radeon RX) run the same `amdgpu` kernel driver. This means:

- No cross-vendor DMA-BUF compatibility issues.
- `vga_switcheroo` works (both register as clients with `driver_power_control=true`).
- Both devices are enumerable by `DRI_PRIME` and Mesa without any proprietary driver involvement.

### 9.2 PRIME on AMD Hybrid Laptops

```bash
# Verify both AMD GPUs are visible
DRI_PRIME=list vulkaninfo 2>/dev/null | grep deviceName
# AMD Radeon 680M (iGPU)
# AMD Radeon RX 7600M XT (dGPU)

# Run on dGPU
DRI_PRIME=1 glxinfo -B | grep "OpenGL renderer"
# OpenGL renderer string: AMD Radeon RX 7600M XT (radeonsi, navi33, LLVM 19.1...)

# Vulkan device select by vid:did
MESA_VK_DEVICE_SELECT=1002:7480 vkcube   # RX 7600M XT device ID
```

### 9.3 Runtime Power Management: amdgpu.runpm

The `amdgpu` driver's `runpm` module parameter controls whether the dGPU can power down at runtime:

```bash
# Check current runpm state
cat /sys/module/amdgpu/parameters/runpm
# 1  (enabled)

# Enable via modprobe configuration
echo 'options amdgpu runpm=1' | sudo tee /etc/modprobe.d/amdgpu.conf

# Check dGPU power state
cat /sys/bus/pci/devices/0000:03:00.0/power/runtime_status
# suspended  (D3cold achieved)

cat /sys/class/drm/card1/device/power_state
# D3cold
```

When `runpm=1` and the dGPU has no active DRM clients, `amdgpu` calls `pci_set_power_state(PCI_D3cold)` and the firmware's `_PS3` method cuts the power rail. The device re-enters D0 within a few tens of milliseconds (exact latency is firmware-dependent; it includes PCIe link retraining time) when a client opens `/dev/dri/renderD129` and the runtime PM `get_sync` callback triggers `_PS0`.

### 9.4 AMD SmartShift

SmartShift (AMD's marketing name for dynamic CPU+GPU TDP sharing) is supported in the `amdgpu` driver via sysfs starting from kernel 5.14 and refined through 6.x. [Source: AMD kernel misc docs](https://dri.freedesktop.org/docs/drm/gpu/amdgpu/driver-misc.html)

The sysfs interface (on APU + dGPU systems that declare SmartShift support in firmware):

```bash
# APU power shift: percentage of APU budget borrowed by/from dGPU
cat /sys/class/drm/card0/device/smartshift_apu_power
# 15  (15% of APU headroom shifted to dGPU)

# dGPU power shift percentage
cat /sys/class/drm/card1/device/smartshift_dgpu_power
# 85

# Bias: -100 (full APU preference) to +100 (full dGPU preference), default 0
cat /sys/class/drm/card1/device/smartshift_bias
# 0

# Shift bias toward dGPU for gaming
echo 50 | sudo tee /sys/class/drm/card1/device/smartshift_bias
```

SmartShift bias is also accessible via `rocm-smi`:

```bash
rocm-smi --showpower    # per-GPU power draw
rocm-smi --showperflevel  # current performance level
```

### 9.4.1 Idle Autosuspend Delay

By default, the `amdgpu` driver uses a 5-second autosuspend delay before entering D3cold: the GPU must be idle for 5 seconds after the last client closes before `runpm` fires `_PS3`. This can be tuned:

```bash
# Reduce autosuspend delay to 1 second
echo 1000 | sudo tee /sys/bus/pci/devices/0000:03:00.0/power/autosuspend_delay_ms
```

A delay that is too short can cause perceptible latency when switching from desktop idle to a game. A delay that is too long leaves the dGPU in D0 longer than necessary during brief idle periods. The default 5 seconds is a reasonable balance for most workloads.

### 9.5 amd_pstate Integration

On Ryzen platforms, the `amd_pstate` CPU frequency driver and the `amdgpu` DPM (Dynamic Power Management) share the platform's thermal budget via the **SMU** (System Management Unit) firmware. The SMU arbitrates between the CPU's STAPM (Skin Temperature Aware Power Management) envelope and the GPU's TDP allocation. `power-profiles-daemon` with `amd_pstate` as its backing driver adjusts the `energy_performance_preference` EPP hint, which the SMU uses to rebalance CPU/GPU headroom.

```bash
# Check current AMD P-state preference
cat /sys/devices/system/cpu/cpufreq/policy0/energy_performance_preference
# balance_power

# Check per-GPU DPM performance level
cat /sys/class/drm/card1/device/power_dpm_force_performance_level
# auto
```

---

## 10. Practical Diagnostics

### 10.1 Is the dGPU Powered Down?

```bash
# Check runtime PM status for all PCI devices
for dev in /sys/bus/pci/devices/*/power/runtime_status; do
    echo "$dev: $(cat $dev)"
done | grep -E "active|suspended|error"

# Specifically for the NVIDIA dGPU (PCI address 01:00.0)
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
# suspended  ← good

# D3cold allowed flag
cat /sys/bus/pci/devices/0000:01:00.0/d3cold_allowed
# 1  ← must be 1 for D3cold to be achievable

# PCIe D3cold events in dmesg
dmesg | grep -i "d3cold\|pci pm"
# [   10.342] pci 0000:01:00.0: PME# disabled
# [  123.891] pci 0000:01:00.0: Refused to change power state from D0 to D3hot
```

### 10.2 Is the Application Using the Right GPU?

```bash
# Mesa / open drivers
DRI_PRIME=1 glxinfo 2>/dev/null | grep "OpenGL renderer"
# OpenGL renderer string: NVIDIA GeForce RTX 3050 (inside host Mesa/nouveau, or...)

# NVIDIA proprietary
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo 2>/dev/null | grep "OpenGL renderer"
# OpenGL renderer string: NVIDIA GeForce RTX 3050/PCIe/SSE2

# Vulkan (NVIDIA)
__NV_PRIME_RENDER_OFFLOAD=1 vulkaninfo 2>/dev/null | grep "deviceName"
# deviceName = NVIDIA GeForce RTX 3050

# Watch live NVIDIA GPU utilization
nvidia-smi dmon -s u -d 1   # per-second utilisation
```

### 10.3 Power Draw Measurement

```bash
# powertop: live power estimate and auto-tune
sudo powertop --auto-tune   # one-shot enable all power-saving settings
sudo powertop               # interactive view

# turbostat: CPU + GPU package power (Intel systems)
sudo turbostat --interval 1 --show PkgWatt,GFXWatt,RAMWatt

# upower: battery discharge rate
upower -i $(upower -e | grep BAT) | grep -E "state|rate|energy"
# state:               discharging
# energy-rate:         8.41 W      ← good (idle, NVIDIA suspended)
```

### 10.4 ACPI Events and Firmware Paths

```bash
# Watch ACPI events (lid close, AC unplug)
acpi_listen

# Find dGPU ACPI firmware path
cat /sys/bus/pci/devices/0000:01:00.0/firmware_node/path
# \_SB.PCI0.PEG0.PEGP

# Kernel messages about PRIME, NVIDIA, and runtime PM
journalctl -k | grep -iE "PRIME|nvidia|runtime pm|d3cold"
```

### 10.5 envycontrol Status Query

```bash
envycontrol --query
# integrated

# After switching, verify udev rules are in place
ls /lib/udev/rules.d/ | grep -i nvidia
# 50-remove-nvidia.rules
# 80-nvidia-pm.rules
```

### 10.5.1 Battery Discharge Rate as a Power Sanity Check

The most direct way to verify dGPU power state from a user perspective is battery discharge rate:

```bash
# Real-time energy rate via upower
upower -i $(upower -e | grep BAT) | grep -E "state|rate|energy"
# state:               discharging
# energy-rate:         8.41 W      ← good (dGPU suspended)

# Versus dGPU stuck in D0:
# energy-rate:         22.3 W      ← dGPU burning ~14W at idle
```

If the discharge rate drops by 10–15W after applying `envycontrol --switch integrated` or setting `NVreg_DynamicPowerManagement=0x02`, that is confirmation the dGPU power-gating path is working.

### 10.6 Common Failure Modes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| dGPU always in D0 (never `suspended`) | Missing `_PR3` in ACPI, or `NVreg_DynamicPowerManagement=0x00` | Set `NVreg_DynamicPowerManagement=0x02`; verify `d3cold_allowed=1` |
| `DRI_PRIME=1` still uses iGPU | NVIDIA proprietary driver requires `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia` instead | Use correct NVIDIA env vars |
| bbswitch `OFF` leaves dGPU stuck | Driver not fully unloaded, or firmware `_DSM` GUID mismatch | `rmmod nvidia*` fully; try `acpi_call` with correct GUID |
| envycontrol `hybrid` mode: dGPU still in D0 | RTD3 requires `auto` in `power/control`; udev rule may not have triggered | Check `80-nvidia-pm.rules`; force `echo auto > .../power/control` |
| `prime-select on-demand` app ignores NVIDIA | Missing GLVND or Vulkan ICD for NVIDIA | Install `libgl1-nvidia-glvnd-glx`, `vulkan-icd-loader` |

---

## Roadmap

### Near-term (6–12 months)

- **NVIDIA open kernel module as the default for Optimus laptops.** Arch Linux switched its `nvidia` packages to use NVIDIA's open-source kernel modules by default for Turing and newer GPUs in late 2025, with other distributions (Fedora, Ubuntu) expected to follow. The open modules expose cleaner interfaces for RTD3 and PRIME that the proprietary closed modules could not. [Source: Phoronix — Arch Linux NVIDIA Open Kernel Modules](https://www.phoronix.com/news/Arch-LInux-NVIDIA-Open-Default)
- **RTD3 monitor hotplug fix in NVIDIA open modules.** A known regression in the open kernel module prevents the dGPU from re-entering D3cold after an external monitor is plugged and then unplugged during a PRIME offload session. Active upstream issue tracking a fix. [Source: NVIDIA open-gpu-kernel-modules issue #759](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/759)
- **power-profiles-daemon battery-state awareness for hybrid GPUs.** Power Profiles Daemon 0.22 introduced AMD-specific improvements; near-term work targets tighter integration between the `performance`/`balanced`/`power-saver` profile transitions and dGPU `power/control` sysfs writes so that switching to `power-saver` automatically triggers RTD3 suspend on the dGPU without requiring manual `envycontrol` invocation. [Source: Phoronix — Power-Profiles-Daemon 0.22](https://www.phoronix.com/news/Power-Profiles-Daemon-0.22)
- **AMD AMDGPU User Mode Queues (UserQ) improvements in Linux 6.19.** The Linux 6.19 merge window (February 2026) brought significant UserQ work from AMD engineers; this infrastructure underlies future SmartShift-aware scheduling where the driver can more dynamically balance workloads between the APU and dGPU. [Source: WebProNews — AMD Linux 6.19 Kernel Upgrades](https://www.webpronews.com/amds-linux-graphics-surge-decoding-the-6-19-kernel-upgrades/)
- **envycontrol 4.x: Wayland-native switching without reboot.** The `envycontrol` maintainer has discussed eliminating the logout/reboot requirement for switching between integrated and hybrid modes by hooking into compositor restart mechanisms on Wayland. Note: needs verification against upstream issue tracker.

### Medium-term (1–3 years)

- **Unified sysfs power control API across NVIDIA and AMD.** Currently, NVIDIA RTD3 is governed by `NVreg_DynamicPowerManagement` module parameters and udev rules, while `amdgpu` uses `amdgpu.runpm` and per-device sysfs `power/control`. A proposed kernel-level abstraction would allow `power-profiles-daemon` and TLP to manage both drivers through a single interface without driver-specific knowledge. Note: needs verification — concept discussed in freedesktop.org power management mailing threads.
- **Advanced Optimus MUX abstraction in the kernel.** NVIDIA RTX 50 Series laptops ship with "Advanced Optimus" hardware that can switch the MUX in-session without a reboot. Exposing this through a standardised kernel interface (extending `vga_switcheroo` or a new `drm_mux` subsystem) is a medium-term goal so that compositors can initiate seamless MUX switching transparently. Note: needs verification.
- **AMD SmartShift 2.0 (SS2.0) bias control integration with schedulers.** The `amdgpu` sysfs `smartshift_bias` knob (range −100 to +100) is currently only settable by privileged user-space tools. Integration with the kernel's power-aware CPU scheduler (`schedutil`) and `power-profiles-daemon` to adjust the bias dynamically based on workload type is a stated direction. [Source: kernel.org amdgpu driver-misc documentation](https://docs.kernel.org/gpu/amdgpu/driver-misc.html)
- **PRIME implicit sync via kernel fence timelines.** As Wayland moves to explicit sync (`linux-drm-syncobj-v1`), the DMA-BUF fence-passing path used by PRIME render offload needs corresponding updates to avoid implicit sync fallbacks that reduce throughput on multi-GPU compositing paths. RFC patches have circulated on the dri-devel mailing list. Note: needs verification of current RFC status.
- **bbswitch retirement.** The `bbswitch` out-of-tree DKMS module is in maintenance mode; medium-term expectation is that NVIDIA RTD3 in the open modules will render it obsolete and distributions will drop bbswitch from repositories once RTD3 D3cold reliability reaches parity. [Source: Bumblebee-Project/bbswitch GitHub](https://github.com/Bumblebee-Project/bbswitch)

### Long-term

- **Kernel DRM MUX subsystem replacing `vga_switcheroo`.** The `vga_switcheroo` code is a legacy subsystem that predates modern DRM. Long-term architectural plans discussed at XDC suggest a proper `drm_mux` abstraction integrated with DRM device links, enabling in-kernel MUX switching events to propagate to compositors via DRM hotplug notifications. Note: needs verification.
- **Wayland display server-side PRIME elimination.** In the long term, if hardware MUX switching becomes reliable and standardised, MUX-less PRIME offload (which requires copying framebuffers through the iGPU) may become optional: the dGPU could scan out directly when the MUX is in dGPU mode and fall back to PRIME only in iGPU mode. This would reduce PRIME-induced memory bandwidth and latency overhead on gaming workloads.
- **AI/NPU-directed power gating.** NVIDIA RTX 50 Series laptops already include on-chip AI that learns usage patterns to anticipate when the dGPU should wake from D3cold before the user launches a workload. Future Linux kernel support for NPU-directed PM hints (communicating via a standardised kernel interface) could replicate this on open-stack AMD and NVIDIA hardware. Note: needs verification — framed as vendor direction in public CES 2026 materials.
- **Cross-vendor hybrid graphics without vendor-specific tools.** The long-term goal of initiatives like `envycontrol`, `supergfxctl`, and `system76-power` converging on a shared D-Bus API (potentially absorbed into `power-profiles-daemon`) would mean a laptop user never needs to install vendor-specific tooling to achieve correct hybrid GPU power management regardless of whether the dGPU is NVIDIA, AMD, or a future architecture.

---

## Integrations

- **Ch5 (x86 GPU Drivers — amdgpu / NVIDIA)**: The `amdgpu` driver's `runpm` and SmartShift sysfs described here are implemented in the driver internals covered in Ch5. NVIDIA driver module parameters including `NVreg_DynamicPowerManagement` are rooted in the proprietary driver architecture covered there.

- **Ch49 (Multi-GPU and PRIME Render Offload)**: Ch49 covers the kernel DMA-BUF / PRIME infrastructure—`DRM_IOCTL_PRIME_HANDLE_TO_FD`, `drm_gem_prime_export()`, `dma_resv` fence sharing—that this chapter relies on at the user-space level. The `DRI_PRIME` env var invokes that kernel machinery.

- **Ch51 (GPU Power Management)**: The kernel runtime PM framework (`pm_runtime_get_sync()`, `pm_runtime_put_autosuspend()`, `CONFIG_PM`) that `amdgpu runpm` and NVIDIA RTD3 both use is covered in Ch51. The ACPI `_PR0`/`_PR3` power resources and D3cold entry path are examined there in detail.

- **Ch106 (Multi-GPU PRIME — detailed)**: Sibling chapter to Ch49, covering Reverse PRIME, RandR 1.4 provider model, and MUX switch vendor tooling (`supergfxctl`, `system76-power`) in depth. The `xrandr --setprovideroffloadsink` and `xrandr --setprovideroutputsource` calls mentioned here are detailed there.

- **Ch117 / Ch122 (DKMS and the NVIDIA proprietary module)**: `bbswitch`, `envycontrol`, and the NVIDIA RTD3 path all depend on the NVIDIA proprietary DKMS module being correctly installed. Ch117 and Ch122 cover DKMS build mechanics and the NVIDIA module's integration with the kernel signing and lockdown infrastructure.

---

## References

- [Linux kernel VGA Switcheroo documentation](https://www.kernel.org/doc/html/latest/gpu/vga-switcheroo.html)
- [NVIDIA RTD3 (Dynamic Power Management) README](https://download.nvidia.com/XFree86/Linux-x86_64/435.17/README/dynamicpowermanagement.html)
- [NVIDIA PRIME Render Offload README (535.98)](https://download.nvidia.com/XFree86/Linux-x86_64/535.98/README/primerenderoffload.html)
- [Mesa environment variables documentation](https://docs.mesa3d.org/envvars.html)
- [AMD amdgpu driver misc documentation](https://dri.freedesktop.org/docs/drm/gpu/amdgpu/driver-misc.html)
- [envycontrol GitHub repository](https://github.com/bayasdev/envycontrol)
- [bbswitch GitHub repository (Bumblebee Project)](https://github.com/Bumblebee-Project/bbswitch)
- [TLP runtime PM settings](https://linrunner.de/tlp/settings/runtimepm.html)
- [power-profiles-daemon GitLab](https://gitlab.freedesktop.org/upower/power-profiles-daemon)
- [NVIDIA libglvnd (GL Vendor-Neutral Dispatch)](https://github.com/NVIDIA/libglvnd)
- [bbswitch README (Bumblebee Project)](https://github.com/Bumblebee-Project/bbswitch/blob/master/README.md)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
