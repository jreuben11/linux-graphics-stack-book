# Chapter 169 — Snapdragon X Elite on Linux: Adreno X1-85, freedreno, and the Arm Laptop Era

**Target audiences:** Systems and driver developers targeting Qualcomm Arm64 laptops; embedded Linux engineers adapting mobile-SoC expertise to the laptop form factor; kernel developers working in the `drivers/gpu/drm/msm/` and `media/platform/qcom/` subsystems.

---

## Table of Contents

1. [Introduction: Qualcomm Enters the Linux Laptop Market](#1-introduction-qualcomm-enters-the-linux-laptop-market)
   - [1.1 What is the Snapdragon X Elite?](#11-what-is-the-snapdragon-x-elite)
   - [1.2 What is freedreno?](#12-what-is-freedreno)
   - [1.3 What is Turnip?](#13-what-is-turnip)
2. [Snapdragon X Elite Hardware Architecture](#2-snapdragon-x-elite-hardware-architecture)
3. [Kernel Driver: DRM/MSM](#3-kernel-driver-drmmsm)
4. [Mesa Turnip Vulkan Driver](#4-mesa-turnip-vulkan-driver)
5. [freedreno OpenGL Driver](#5-freedreno-opengl-driver)
6. [Display Pipeline](#6-display-pipeline)
7. [Video Acceleration](#7-video-acceleration)
8. [NPU / Hexagon Integration](#8-npu--hexagon-integration)
9. [Qualcomm's Upstream Commitment](#9-qualcomms-upstream-commitment)
10. [Practical Setup: Linux on X Elite Laptops](#10-practical-setup-linux-on-x-elite-laptops)
11. [Integrations](#11-integrations)

---

## 1. Introduction: Qualcomm Enters the Linux Laptop Market

The Snapdragon X Elite marks a pivotal transition for Qualcomm's relationship with the Linux graphics stack. For more than a decade, freedreno and Turnip were squarely mobile-SoC technologies — living in the Android kernel's `msm` DRM driver and Mesa's Gallium3D layer, powering phones that most users never thought of as Linux machines. When Qualcomm launched the Snapdragon X Elite in late 2023 and shipped it in commercial laptops from mid-2024 onward, that same driver stack suddenly needed to work in a context it had never been designed for: x86-style UEFI boot, LPDDR5X shared memory attached to a multi-display laptop, Wayland compositors, Chromium, and productivity software that users expect to "just work."

The transition is not merely one of form factor. It is one of platform architecture. Mobile Adreno SoCs live in a world where the bootloader supplies a Device Tree Blob (DTB) that exactly describes every peripheral. The X Elite enters UEFI territory, where the Linux kernel discovers hardware through a combination of ACPI tables and a platform-specific device tree overlay; each OEM laptop model still requires its own `.dts` file to wire together the ACPI enumerations. This makes Linux bring-up per-laptop-model work rather than per-SoC work — a sharp contrast to the phone world where a single kernel tree images many handsets from the same chipset.

Why does this matter for graphics developers? Because the GPU and display drivers — the `msm` DRM driver, the `msm_dpu` KMS front-end, and the Mesa freedreno/Turnip stack — were already upstream. Qualcomm had been contributing them since around 2012 for mobile SoCs. The X Elite story is therefore less about writing new drivers from scratch and more about extending proven mobile-lineage code to handle the new platform environment: ACPI-enumerated PCI-style resources, panel-self-refresh on eDP panels, HDR-capable display pipelines, and the IRIS video decoder replacing the older Venus firmware interface. This chapter traces that extension work, documents what is upstream and what remains out-of-tree as of mid-2026, and equips driver and embedded developers to navigate the Snapdragon X Elite on Linux.

### 1.1 What is the Snapdragon X Elite?

The Snapdragon X Elite (internal designation X1E80100, also referenced as SC8380XP in pre-release documentation) is a heterogeneous system-on-chip from Qualcomm, manufactured on TSMC's 4nm process and targeting the thin-and-light laptop market. It integrates Qualcomm's Oryon CPU cores, the Adreno X1-85 GPU, the Hexagon NPU for neural-network inference, and an IRIS video processing unit — all sharing a unified LPDDR5X memory pool over a 128-bit bus. Unlike discrete GPU platforms, there is no separate VRAM; the CPU and GPU allocate from the same physical address space, with coherency assisted by a 6 MB System Level Cache on the SoC.

From a Linux platform perspective, the Snapdragon X Elite occupies a structurally novel position. It boots via UEFI rather than the raw bootloader of a smartphone, yet its SoC peripherals are described by a Device Tree Blob delivered through the EFI stub rather than by pure ACPI tables. Each OEM laptop model — Lenovo ThinkPad T14s Gen 6, Dell XPS 13 9345, ASUS Vivobook S 15, HP OmniBook X 14, and others — requires its own `.dts` entry under `arch/arm64/boot/dts/qcom/`, even though all share the same X1E80100 die. Core SoC support (pinctrl, clocks, power domains, SMMU, PCIe, USB PHY) landed across Linux 6.8 and 6.9; GPU and display support targeted Linux 6.11 and became practically usable around Linux 6.14 combined with Mesa 25.0. The platform represents Qualcomm's first SoC aimed squarely at a desktop-class Linux environment in which Wayland compositors, Chromium, and general productivity software are primary workloads alongside the mobile-derived GPU hardware.

### 1.2 What is freedreno?

freedreno is the open-source driver stack for Qualcomm Adreno GPUs on Linux. The name covers two coupled components: the `msm` DRM kernel driver at `drivers/gpu/drm/msm/` and the Mesa Gallium3D user-space driver at `src/gallium/drivers/freedreno/`. The kernel component manages GPU command-ring submission, the Graphics Management Unit (GMU) firmware interface for GPU power states, GPU memory mapping through the SoC's SMMU IOMMU, and the KMS display pipeline through the `msm_dpu` subsystem. The Mesa Gallium3D component translates OpenGL API calls through the NIR shader intermediate representation into the Adreno IR3 instruction set, managing render-pass state, buffer objects, and hardware queries.

freedreno has been an in-tree kernel driver since approximately 2012, initially supporting Adreno 3xx mobile SoCs. Successive kernel releases expanded coverage through the A4xx, A5xx, A6xx, and A7xx GPU generations. For the Snapdragon X Elite the existing driver was extended — not replaced — by adding the X185 catalog entry, a `gmxc.lvl` voltage rail, and the X1E80100 DPU configuration table. The Gallium3D path in Mesa gained initial A7xx support in Mesa 24.3. The freedreno name is sometimes used loosely to refer to the entire Qualcomm open-source GPU driver ecosystem, encompassing both the OpenGL path described here and the Vulkan driver, Turnip, covered in the next subsection and in detail in §4 and §5.

### 1.3 What is Turnip?

Turnip (`src/freedreno/vulkan/`) is Mesa's open-source Vulkan driver for Qualcomm Adreno GPUs. It is distinct from the freedreno Gallium3D driver — which serves OpenGL — in that Turnip implements the Vulkan API directly, without passing through a Gallium3D state tracker. Merged into Mesa in 2020 for Adreno A6xx hardware, it has since expanded to the A7xx family, which includes the Adreno X1-85 inside the Snapdragon X Elite.

Turnip exposes Vulkan 1.3 on the X1-85, matching Qualcomm's official API declaration for the platform. It compiles SPIR-V shaders through Mesa's NIR intermediate representation and then through the Adreno-specific IR3 compiler back-end, producing binary shader code submitted to the GPU via `CP_LOAD_STATE7` packets in the command stream. Turnip manages descriptor sets, render-pass tiling decisions, and the Adreno tile-based rendering model in which geometry is first binned across screen tiles before per-tile fragment shading occurs in the GPU's on-chip GMEM. The `VK_KHR_ray_query` extension became available for the A7xx family — including the X1-85 — in Mesa 25.0. Mesa 25.1 added Turnip to the default build list for AArch64 targets, making it the standard Vulkan path on ARM64 Linux systems using Mesa distribution packages, without requiring explicit build-time configuration. §4 covers the Turnip command buffer, shader compiler, and tiling implementation in detail.

---

## 2. Snapdragon X Elite Hardware Architecture

### 2.1 SoC Overview

The Snapdragon X Elite SoC carries the internal Qualcomm designation **X1E80100** (also seen as SC8380XP in earlier pre-announcement references). It is a heterogeneous compute platform built on TSMC 4nm, integrating:

- **CPU:** Up to 12 Oryon cores in a big-big cluster configuration (no LITTLE cores). The Oryon microarchitecture is derived from Qualcomm's Nuvia acquisition and represents a clean-sheet Arm ISA implementation.
- **GPU:** Marketed as the **Adreno X1-85** (see §2.2 below).
- **NPU:** Hexagon processor for neural-network inference acceleration.
- **Memory:** LPDDR5X unified memory, shared coherently between CPU, GPU, and the NPU. Official laptop SKUs ship with 16 GB or 32 GB; the memory controller supports up to 64 GB. The memory bus is 128-bit wide, enabling substantial bandwidth for both the CPU and the GPU's GMEM-bypass paths.
- **Connectivity:** Wi-Fi 7 (FastConnect 7800), Bluetooth 5.4, integrated USB 4 Gen 3x2 host controllers, NVMe over PCIe Gen 4.
- **Display:** Snapdragon Display Engine (SDE), driving eDP for the internal panel and DP Alt Mode over USB-C for external displays.
- **Video:** IRIS video processor, capable of hardware-accelerated H.264, H.265 (HEVC), VP9, and AV1 decode, and concurrent dual-stream encode.

The SKU matrix includes X1E-80-100 (the most common in laptops), X1E-78-100, and X1E-84-100, which differ primarily in GPU clock speed (1.25 GHz to 1.5 GHz) and CPU boost frequency.

### 2.2 GPU: Adreno X1-85

The GPU is marketed by Qualcomm as the **Adreno X1-85**. In the kernel driver it is registered under the internal designator **X185** (see §3.2). One well-sourced hardware analysis describes it as "a scaled-out version of the Adreno 730 from the Snapdragon 8+ Gen 1 phone chip," noting that it carries the A7xx internal microarchitecture family [Chips and Cheese, "The Snapdragon X Elite's Adreno iGPU," 2024](https://chipsandcheese.com/p/the-snapdragon-x-elites-adreno-igpu).

Published Qualcomm specifications and the above analysis establish the following GPU characteristics:

| Attribute | Value |
|---|---|
| Marketing name | Adreno X1-85 |
| Kernel identifier | X185 / ADRENO_7XX_GEN2 |
| Shader Processors | 6 |
| FP32 ALUs | 1,536 |
| Peak FP32 throughput | 4.6 TFLOPS |
| Texel rate | 96 texels/cycle |
| Pixel rate | 72 gigapixels/second |
| GMEM (local memory) | 3 MB |
| GPU virtual address space | 256 GB |
| GPU clock (X1E-80-100) | 1.25 GHz |
| GPU clock (X1E-84-100) | 1.5 GHz |
| Memory | 128-bit LPDDR5X (shared with CPU) |

API support, as declared by Qualcomm: DirectX 12.1 (Shader Model 6.7), Vulkan 1.3, OpenCL 3.0. On Linux, Vulkan 1.3 is provided by the Mesa Turnip driver (see §4).

The GPU memory model is unified: there is no discrete VRAM. The Adreno X1-85 allocates from the same LPDDR5X pool as the Oryon CPU, with a System Level Cache (SLC) of 6 MB, cluster caches of 128 KB, per-uSPTP texture caches of 2 KB, and the 3 MB of GMEM for tile-based rendering. This contrasts sharply with discrete GPU architectures and has implications for DMA-BUF sharing: CPU and GPU share the same physical backing store, making zero-copy import from the camera subsystem or the V4L2 capture path straightforward in principle.

### 2.3 Display Engine

The Snapdragon Display Engine (SDE) is exposed to Linux through the `msm_dpu` subsystem. The X1E80100 DPU configuration (catalogue entry `dpu_9_2_x1e80100.h`) supports:

- 4 VIG (video-capable) source planes with QSEED 3.3.2 scaler
- 6 DMA source planes (2 are cursor-capable)
- 6 layer mixers
- 4 DSPP (display stream post-processing) units
- 8 ping-pong interfaces
- 4 DSC (Display Stream Compression) encoders (2 support native 4:2:2)
- 2 DP output interfaces
- 2 DSI output interfaces
- 1 writeback output

Maximum line width is 5,120 pixels (5K resolution on the widest axis), with up to 11 blending stages per mixer. The platform supports source splitting and 3D merge for multi-LMTX configurations, and has idle power collapse.

### 2.4 Video Codec Hardware

The IRIS video processor on the X1E80100 supports the following codec capabilities [Qualcomm product brief]:

- **Decode:** H.264 up to 4K120 10-bit; HEVC (H.265) up to 4K120 10-bit; VP9 up to 4K; AV1 up to 4K120 10-bit
- **Encode:** Concurrent dual-stream 4K30 for H.264, HEVC, and AV1; single-stream 4K60 10-bit

On Linux, H.264 decode became available with the upstreaming of the IRIS V4L2 driver in kernel 6.15 (see §7). H.265 and AV1 hardware decode on Linux remain in progress as of mid-2026.

---

## 3. Kernel Driver: DRM/MSM

### 3.1 Driver Lineage: Mobile to Laptop

The `msm` DRM driver at `drivers/gpu/drm/msm/` is one of the older in-tree SoC DRM drivers, with origins tracing to Qualcomm-affiliated contributors around 2012. Rob Clark (then at Texas Instruments, later at Google/Qualcomm) was the primary author. The driver handles both the Adreno GPU (ring submission, memory management, power management) and the Display Processing Unit (display pipeline, KMS CRTC/plane abstraction). For the X Elite, no new driver was created; the X185 GPU and X1E80100 DPU were added as new entries within the existing driver. [Source: `lore.kernel.org`, 2024 patch series](https://lore.kernel.org/linux-kernel/20240701055103.srt6olauy7ux5um5@hu-akhilpo-hyd.qualcomm.com/T/)

### 3.2 GPU Catalog Entry: X185

Kernel support for the Adreno X1-85 GPU was targeted at Linux 6.11 through a patch series from Qualcomm engineer Akhil P Oommen, posted in June–July 2024. [Source: `[PATCH v1 0/3] Support for Adreno X1-85 GPU`](https://lore.kernel.org/linux-kernel/20240701055103.srt6olauy7ux5um5@hu-akhilpo-hyd.qualcomm.com/T/)

The GPU is registered in the driver's `adreno_devices` table as `X185`, belonging to the `ADRENO_7XX_GEN2` family (internal A7xx generation, second tier). Key catalog parameters are:

```c
/* drivers/gpu/drm/msm/adreno/a6xx_gpu_state.c and adreno/adreno_device.c
 * (illustrative — derived from [PATCH v2 2/5] drm/msm/adreno: Add support for X185 GPU,
 *  https://lkml.iu.edu/2406.3/08138.html) */
.chip_ids = { 0x43050c01 },   /* chip ID: C512v2 */
.family  = ADRENO_7XX_GEN2,
.gmem    = SZ_3M,              /* 3 MB GMEM */
.va_size = SZ_256G,            /* 256 GB GPU VA */
/* firmware (gen70500 series filenames per linux-firmware-qcom package) */
.fw[ADRENO_FW_SQE] = "qcom/gen70500_sqe.fw",
.fw[ADRENO_FW_GMU] = "qcom/gen70500_gmu.bin",
/* ZAP shader security firmware: qcom/x1e80100/gen70500_zap.mbn */
/* power */
/* gmxc.lvl rail preferred; falls back to mx.lvl */
```

**Note:** The above is a reconstructed illustration from patch summaries; do not treat it as a verbatim kernel source listing. For the authoritative text see the `freedreno@lists.freedesktop.org` patch archive.

Power management adds a `gmxc.lvl` voltage rail specific to the X185; the driver falls back to `mx.lvl` if the former is absent, for compatibility with compute reference designs that may not populate the rail separately. A later patch (September 2024) switched the `is_x185` predicate to an `is_x1xx_family` check to accommodate additional Snapdragon X variants. [Source: `patches.linaro.org`](https://patches.linaro.org/project/linux-arm-msm/patch/20240921204237.8006-1-john.schulz1@protonmail.com/)

An important early caveat: **the X1E80100 device tree initially disabled the GPU node by default**, pending firmware extraction solutions for end-user systems. [Source: Phoronix reporting, "Linux Patch To Disable The Snapdragon X Elite GPU By Default"](https://www.phoronix.com/news/Linux-Disabling-X-Elite-GPU) Boards and laptops need the firmware to be extracted from the co-installed Windows partition before the GPU driver can probe successfully (see §10).

### 3.3 GPU Ring Submission and the GMU

The Adreno X1-85 uses the same Graphics Management Unit (GMU) firmware-based power management as the A6xx/A7xx phone GPUs. The GMU is a small Arm Cortex-M3-class co-processor embedded in the SoC; it manages GPU power states, frequency scaling, and idle collapsing independently of the CPU. Communication between the host driver and GMU uses a pair of shared-memory rings.

Ring buffer submission follows the pattern established for A6xx and A7xx. The real implementation lives in `drivers/gpu/drm/msm/adreno/a6xx_gpu.c` (`a6xx_submit()`). The key concepts are:

- **CP_INDIRECT_BUFFER_PFE** — PM4 packet type 7 that dispatches a command buffer by IOVA address and size
- **OUT_PKT7 / OUT_RING** — macros that emit PM4 header words and data words into the ring buffer
- **Fence signalling** — completed via a CP event write packet that the kernel waits on through a `msm_fence`

The X185 inherits this A7xx (A6xx derivative) command processor model unchanged. The main X185-specific additions in the kernel driver are the catalog registration, GMU firmware binding, and the `gmxc.lvl` rail handling. For the authoritative implementation, see `drivers/gpu/drm/msm/adreno/a6xx_gpu.c` in the kernel tree.

### 3.4 Platform Boot: Device Tree vs. ACPI

The X Elite laptops boot via UEFI, and unlike a pure x86 box, they rely on a **Device Tree Blob** delivered through UEFI's EFI stub rather than on bare ACPI tables for SoC peripheral description. This is the "ACPI + DT overlay" model that Qualcomm has standardized across its Compute Reference Design (CRD) and OEM laptops: ACPI handles power button, thermal zones, and PSCI CPU topology; the DTB handles everything below — interconnects, clocks, GPIOs, the GPU, display, IOMMU, and so on.

This is consequential for bring-up: each new laptop SKU requires a per-board `.dts` entry, even if it uses the same X1E80100 SoC, because OEMs choose different display panels, Wi-Fi card strapping, USB multiplexers, and camera topologies. Kernel patches for the CRD, Lenovo ThinkPad T14s Gen 6, ASUS Vivobook S 15, Dell XPS 13 9345, HP OmniBook X 14, and others are separate commits that add entries under `arch/arm64/boot/dts/qcom/`. [Source: Qualcomm upstream blog, May 2024](https://www.qualcomm.com/developer/blog/2024/05/upstreaming-linux-kernel-support-for-the-snapdragon-x-elite)

### 3.5 Kernel Version Timeline

| Kernel Version | What Landed |
|---|---|
| 6.8 / 6.9 | Core SoC support: pinctrl, clocks (GCC/RPMHCC), interconnects, powerdomains (RPMh), SMMU, QUP (I2C/SPI/UART), PCIe, eDP/USB PHY, PMIC8380 |
| 6.11 | Adreno X185 GPU driver targeted; DPU (display) support for X1E80100 |
| 6.14 | Functional GPU bring-up reported; usable with firmware extraction from Windows |
| 6.15 | IRIS H.264 video decode driver upstreamed; additional laptop DT entries |
| 6.17 | Significant GPU performance improvements: speedbin support, IFPC (Inter-Frame Power Collapse), further decoupling of GPU and KMS code |

Note that "targeted" for 6.11 does not equal "working in 6.11"; end-to-end GPU acceleration required firmware and additional Mesa work that solidified around 6.14 + Mesa 25.0. [Source: Mesa 25.0 benchmarks on Phoronix; Linaro blog](https://www.linaro.org/blog/linux-on-snapdragon-x-elite/)

---

## 4. Mesa Turnip Vulkan Driver

### 4.1 Turnip Architecture

Turnip (`src/freedreno/vulkan/`) is Mesa's open-source Vulkan driver for Qualcomm Adreno GPUs. It was first merged into Mesa in 2020 for A6xx hardware. Like all Mesa Vulkan drivers it implements the Vulkan 1.x API on top of the NIR shader IR, feeding a hardware-specific back-end for command-buffer emission, descriptor management, and render-pass tiling logic.

For the Adreno X1-85, Turnip maps into the same A7xx (a7xx) code paths used for the Snapdragon 8-series phone GPUs that share the generation. Turnip exposes **Vulkan 1.3** on this hardware, matching the API level Qualcomm declared for the platform. [Source: Mesa docs, freedreno](https://docs.mesa3d.org/drivers/freedreno.html)

### 4.2 Shader Compiler: NIR → IR3

Turnip compiles SPIR-V shaders through the following pipeline:

```text
SPIR-V
  │
  ▼
Mesa NIR (via spirv_to_nir())
  │  [NIR lowering and optimisation passes]
  ▼
IR3 (freedreno's internal IR)
  │  [register allocation, scheduling, spill/reload]
  ▼
Binary IR3 machine code
  │
  ▼
CP_LOAD_STATE7 packets → GPU command stream
```

The **IR3** intermediate representation is the shader compiler used for all Adreno hardware from A3xx onward (the "ir3" name comes from the third-generation ISA that debuted with A3xx). [Source: Mesa IR3 notes](https://docs.mesa3d.org/drivers/freedreno/ir3-notes.html) For A7xx the IR3 back-end adds support for the expanded instruction encoding (64-wide scalar FP32 per scheduler partition) and hardware features like bindless descriptors and the extended texture cache.

Relevant source files:

- `src/freedreno/ir3/` — IR3 compiler proper (instruction selection, register allocation)
- `src/freedreno/vulkan/tu_cmd_buffer.c` — Turnip command buffer recording
- `src/freedreno/vulkan/tu_shader.c` — SPIR-V → NIR → IR3 dispatch
- `src/freedreno/common/freedreno_devices.py` — hardware device table (chip IDs, features)

### 4.3 Mesa Version History for X1-85

The Adreno X1-85 falls within the a7xx family that Mesa 24.3 began supporting in the Gallium3D (OpenGL) path. For Turnip (Vulkan), the critical milestones are:

- **Mesa 24.3** — Initial Freedreno a7xx Gallium3D support merged. [Source: Phoronix, "Mesa's Freedreno Gallium3D Driver Lands Initial Adreno 700 Series Support"](https://www.phoronix.com/news/Mesa-24.3-Freedreno-A7xx)
- **Mesa 25.0** — Working end-to-end Vulkan (Turnip) + OpenGL (Gallium3D/freedreno) on X1-85 with firmware extracted from the Windows partition; Linux 6.14 + Mesa 25.0 was the first configuration where 3D graphics acceleration was practically usable on X Elite laptops. Ray query support (`VK_KHR_ray_query`) merged for Adreno 740 and newer (which includes the X1-85 family). [Source: Phoronix]
- **Mesa 25.1** — Turnip added to the default Mesa build list for AArch64, meaning ARM64 Linux systems building Mesa from source or using distribution packages will now get the Vulkan driver by default without extra configuration. [Source: Phoronix, "Open-Source Qualcomm Adreno Vulkan Driver Matures To Default AArch64 Mesa Driver List"](https://www.phoronix.com/news/Mesa-25.1-Turnip-ARM64-Default)

Vulkan extensions supported on X1-85 via Turnip include (non-exhaustive, subject to version):

- `VK_KHR_ray_query` (mesh shaders still under development as of mid-2026)
- `VK_KHR_dynamic_rendering`
- `VK_EXT_descriptor_indexing`
- `VK_KHR_buffer_device_address`
- `VK_KHR_timeline_semaphore`
- `VK_EXT_memory_budget`
- `VK_KHR_16bit_storage`

For a full, version-specific extension list consult the Vulkan hardware database entry for `freedreno` or run `vulkaninfo` on the target system.

### 4.4 Performance vs. Proprietary Driver

As of mid-2026 there is no production-ready proprietary Qualcomm Vulkan driver available for Linux on the X Elite. The Windows ARM driver stack is separate and not portable. The Turnip driver is therefore the only viable accelerated Vulkan path. Phoronix benchmarks on Ubuntu with the X1E80100 GPU placed Turnip-accelerated gaming workloads behind the Windows driver but competitive with AMD Radeon 780M iGPU at 1080p in a subset of OpenGL and Vulkan tests. **Note: consult current Phoronix benchmark data for up-to-date comparisons, as driver performance is evolving rapidly.**

---

## 5. freedreno OpenGL Driver

### 5.1 Gallium3D Driver Overview

The `freedreno` Gallium3D driver at `src/gallium/drivers/freedreno/` provides the OpenGL implementation path for Adreno hardware. It shares the NIR lowering infrastructure and much of the IR3 shader compiler with Turnip. For the X1-85 / A7xx family, Rob Clark landed initial Gallium3D support in **Mesa 24.3**, reworking approximately 700 lines to add the A7xx code paths on top of the A6xx implementation. [Source: Phoronix, "Mesa's Freedreno Gallium3D Driver Lands Initial Adreno 700 Series Support"](https://www.phoronix.com/news/Mesa-24.3-Freedreno-A7xx)

The Gallium3D driver targets OpenGL 4.5 and OpenGL ES 3.2 on hardware that supports it.

### 5.2 Zink as an Alternative OpenGL Path

For applications requiring OpenGL on systems where `freedreno`'s native OpenGL implementation is incomplete or not yet available for the A7xx generation, **Zink** provides an alternative path. Zink is a Gallium3D driver that translates OpenGL calls to Vulkan, using the Turnip Vulkan driver as its back-end:

```text
OpenGL Application
  │
Mesa Gallium3D API
  │
Zink (OpenGL→Vulkan translation layer)
  │
Turnip Vulkan Driver
  │
Adreno X1-85 Hardware
```

Zink is available in Mesa 21.x and later; with Turnip reaching Vulkan 1.3 status, Zink-on-Turnip provides a practical OpenGL 4.6 path even where native freedreno OpenGL support for a particular A7xx subvariant lags. The environment variable `MESA_LOADER_DRIVER_OVERRIDE=zink` or the Vulkan device selection mechanism can be used to select it.

### 5.3 Mesa 25.x / 26.x Status

By **Mesa 25.1** both the native freedreno OpenGL path and Turnip Vulkan are functional for the Adreno X1-85 on Linux with the x1e80100 platform. Mesa 26.0 introduced initial support for the next-generation **Adreno X2-85** (Gen 8 architecture) found in the Snapdragon X2 Elite, while X1-85 support continued maturing in the 25.x stable series. [Source: Phoronix, "Adreno Gen 8 Vulkan Graphics Merged For Mesa 26.0"](https://www.phoronix.com/news/Mesa-26.0-Adreno-Gen-8-Graphics)

---

## 6. Display Pipeline

### 6.1 msm_dpu: Architecture

The Display Processing Unit (DPU) on the X1E80100 is driven by the `msm_dpu` sub-driver inside `drivers/gpu/drm/msm/disp/dpu1/`. The DPU is a fixed-function display compositor that manages scanout from LPDDR5X through the display engine to the panel or DP output; the CPU-side KMS driver exposes it through the standard DRM plane/CRTC/encoder/connector model to user space and to Wayland compositors.

The X1E80100-specific DPU catalog is defined in `drivers/gpu/drm/msm/disp/dpu1/catalog/dpu_9_2_x1e80100.h` and was upstreamed in a patch series posted in early 2024 targeting Linux 6.11. [Source: `[PATCH 5/5] drm/msm/dpu: Add X1E80100 support`](https://www.mail-archive.com/freedreno@lists.freedesktop.org/msg29593.html)

The key DPU capability parameters (verified from patch content):

```c
/* dpu_9_2_x1e80100.h — excerpt (illustrative from patch archive summary) */
.max_line_width = 5120,
.max_mixer_blendstages = 0xb,   /* 11 stages */
.has_src_split = true,
.has_3d_merge = true,
.has_idle_pc = true,
.has_dither = true,
/* SSPP: 4 VIG + 6 DMA planes */
/* LM: 6 layer mixers, SDM845-class caps */
/* DSPP: 4 display post-processing units */
/* DSC: 4 encoders, 2 native 4:2:2 */
/* WB: 1 writeback interface */
```

### 6.2 eDP and External Display

The internal laptop display is connected via **eDP** (embedded DisplayPort). eDP on the X1E80100 is handled through a dedicated PHY and connector; the DPU kernel patch notes that the eDP connector is enabled via an `is-edp` Device Tree property rather than from driver match data, distinguishing it from the DP Alt Mode connectors. [Source: LWN summary, drm/msm display support for X1E80100](https://lwn.net/Articles/962320/)

External display output uses USB-C DP Alt Mode, with the PHY upstreamed in the 6.8/6.9 timeframe. HDMI out (through an on-board HDMI encoder) was not fully functional in early kernels on some laptop models.

### 6.3 HDR and Wide Color Gamut

The DPU supports HDR metadata passthrough and wide color gamut processing through the DSPP units. As of mid-2026, KMS HDR support on Linux is still stabilizing across the stack; Wayland compositors (KDE Plasma 6.x, GNOME 47+) have growing HDR support, but per-pixel HDR workflows on the X Elite's display pipeline require both correct DRM HDR property support and a compositor that populates them. **Note: the practical HDR status on specific X Elite laptop panels under Wayland requires verification per-configuration.**

### 6.4 DMA-BUF and Wayland Compositor Integration

Because the Adreno X1-85 uses unified system memory, DMA-BUF sharing between the camera (`v4l2-capture`), video decoder (IRIS), and the compositor is zero-copy: a buffer allocated by the GPU driver can be exported as a DMA-BUF file descriptor and imported by any other driver that speaks the same `dma_buf_ops`. Mesa's Turnip allocates GPU memory through the `msm_gem` allocator, which participates in the DMA-BUF framework.

DMA-BUF modifier negotiation between Turnip and a Wayland compositor proceeds as follows:

1. The compositor queries `wl_drm` or `zwp_linux_dmabuf_v1` for supported format/modifier pairs.
2. Turnip advertises modifiers for its tiled (UBWC — Universal Bandwidth Compression) and linear memory layouts.
3. The compositor selects a compatible modifier and allocates a shared buffer using `gbm_bo_create_with_modifiers()`.
4. The buffer's DMA-BUF fd is passed back; the KMS driver imports it through `drm_prime_fd_to_handle` and the DPU scanout plane imports it with the agreed modifier.

**UBWC (Universal Bandwidth Compression)** is Qualcomm's proprietary tiled memory layout that packs compressed tile metadata alongside the tile data. The kernel MSM driver received a cleanup pass in Linux 6.17 that consolidated UBWC configuration into a single canonical location, improving maintainability. [Source: Phoronix, "Big Improvements For Qualcomm GPU Driver With Linux 6.17"](https://www.phoronix.com/news/Qualcomm-MSM-DRM-Linux-6.17)

### 6.5 Panel Self-Refresh

eDP Panel Self-Refresh (PSR) allows the display panel to continue updating from its own framebuffer while the GPU and display controller enter low-power states. This is essential for laptop battery life. The DPU's `has_idle_pc` flag enables the KMS driver to enter display idle power collapse, which works in conjunction with PSR. Practical PSR support on specific X Elite laptop panels depends on per-panel firmware and Device Tree configuration; as of kernel 6.15 it was functional on some panel models and still in progress on others.

---

## 7. Video Acceleration

### 7.1 IRIS V4L2 Driver

The Snapdragon X Elite does **not** use the Venus video decoder that is common on older Qualcomm mobile SoCs (Venus is used on Snapdragon 845 and earlier). The X1E80100 uses the **IRIS** video processor, which communicates with the application processor through a Host Firmware Interface (HFI) protocol over shared memory. IRIS is a multi-pipe stateful decoder: the host driver submits stream data and receives decoded frames, with the hardware managing codec state internally.

The Qualcomm IRIS V4L2 driver was **upstreamed in Linux 6.15** as a V4L2 video driver with initial H.264 decode capability. [Source: Phoronix forums, "Qualcomm Iris Video Decode Driver & DesignWare HDMI Input Support Ready For Linux 6.15"](https://www.phoronix.com/forums/forum/hardware/general-hardware/1536294-qualcomm-iris-video-decode-driver-designware-hdmi-input-support-ready-for-linux-6-15) The driver path in the kernel tree is `drivers/media/platform/qcom/iris/`.

Initial platform-specific Device Tree enablement for the IRIS hardware on X Elite laptop models (CRD and ThinkPad T14s Gen 6) was posted by Linaro engineers alongside the core driver. [Source: Phoronix, "Select Qualcomm X Elite Laptops Seeing IRIS Video Acceleration On Linux"](https://www.phoronix.com/news/Qualcomm-X-Elite-IRIS-Video)

Because IRIS is a **stateful** V4L2 decoder (using V4L2 M2M with the firmware managing codec state), the application-facing model is:

```c
/* Typical V4L2 M2M stateful decode flow */
fd = open("/dev/video0", O_RDWR);
/* OUTPUT queue: compressed stream input */
/* CAPTURE queue: decoded YUV frames output */
struct v4l2_format fmt = {
    .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
    .fmt.pix_mp.pixelformat = V4L2_PIX_FMT_H264,
};
ioctl(fd, VIDIOC_S_FMT, &fmt);
/* stream on, queue/dequeue buffers */
```

This is different from the stateless V4L2 decoder model (used on e.g. Hantro/Cedrus) where the host driver is responsible for feeding individual decoded slices.

### 7.2 VA-API

VA-API is an Intel-originated API for hardware video acceleration. On Qualcomm/Linux the path to VA-API is through a VA-API driver backend that translates VA-API calls to V4L2. The `libva-v4l2-request` project provides such a backend for stateless V4L2 decoders; however the IRIS driver is **stateful**, meaning a different adaptation layer is required. As of mid-2026, **a fully functional VA-API backend for IRIS on the X Elite is not available in upstream**. Users wishing to test hardware decode should use GStreamer's `v4l2h264dec` element or FFmpeg with `v4l2m2m`, which speak the V4L2 M2M interface directly.

### 7.3 Hardware Encode and AV1/H.265 Status

The IRIS hardware supports H.265 decode and AV1 decode at the silicon level. As of mid-2026 the upstream driver supports **H.264 decode only**; H.265 and AV1 decode drivers are under active development. Hardware encode support was likewise not yet in mainline. **Note: check the linux-media mailing list and Qualcomm's developer blog for the most current status on these codec paths.**

---

## 8. NPU / Hexagon Integration

### 8.1 Architecture: On-Die Hexagon DSP

Every Snapdragon SoC since the Snapdragon 845 includes one or more **Hexagon** DSP cores. On the X Elite, the Hexagon NPU (also called the Hexagon processor or CDSP — Compute DSP) is Qualcomm's primary hardware for on-device ML inference, accelerating operations like matrix multiply, convolution, and activation functions far more efficiently than the Oryon CPU cores.

The Hexagon DSP is not a general-purpose compute device in the GPU sense. It runs firmware loaded by the kernel's remote processor framework (`remoteproc`), and the host only communicates with it through well-defined QMI/FastRPC channels. On Android this is abstracted by the HAL layer; on Linux the substrate is different and the situation is substantially more constrained.

### 8.2 Kernel Driver: remoteproc / CDSP

In the Linux kernel, the Hexagon CDSP on Qualcomm SoCs is managed through the `remoteproc` subsystem (`drivers/remoteproc/qcom_*.c`). The CDSP firmware is loaded as a firmware blob by the kernel, the processor is held in reset, and then released. From the host's perspective, the CDSP is an opaque co-processor running proprietary firmware; the only host-visible interface is the shared-memory ring used for QMI messaging.

The `drivers/accel/qaic/` path **does not apply here**. The `qaic` driver is for Qualcomm's **Cloud AI 100 (AIC100)** discrete PCIe card, a datacenter accelerator product that is a completely separate hardware product from the integrated Hexagon NPU in the X Elite. [Source: Linux kernel docs, `docs.kernel.org/accel/qaic/aic100.html`](https://docs.kernel.org/accel/qaic/aic100.html) Conflating the two is a common error.

There is **no upstream compute-capable DRM/accel driver for the Hexagon CDSP on the X Elite** as of mid-2026. An RFC patch series titled "accel/qda: Introduce Qualcomm DSP Accelerator driver" was posted to the kernel mailing list in early 2026, proposing a DRM-based accelerator implementation for Qualcomm Hexagon DSPs, but it had not been merged and remained under review. [Source: `[PATCH RFC 00/18] accel/qda: Introduce Qualcomm DSP Accelerator driver`, lkml.iu.edu]

### 8.3 SNPE, QNN, and Linux

On Windows ARM, Qualcomm provides:

- **SNPE** (Snapdragon Neural Processing Engine) — the older SDK, now deprecated.
- **QNN** (Qualcomm AI Engine Direct / QAIRT) — the current ML inference SDK, the recommended successor to SNPE.

Neither is available for upstream Linux on the X Elite SoC in a form that accesses the Hexagon NPU. The situation is compounded by a deliberate Qualcomm decision: in response to a community request to open-source the Hexagon DSP headers (which would enable third-party ML runtimes to target the NPU on Linux), Qualcomm closed the GitHub issue with the statement that there are **"no plans to open source DSP headers as of now."** [Source: VideoCardz, "Qualcomm shuts door on Snapdragon X DSP headers open-sourcing, Linux support hopes fade"](https://videocardz.com/newz/qualcomm-shuts-door-on-snapdragon-x-dsp-headers-open-sourcing-linux-support-hopes-fade)

The practical consequence: **on Linux, the Hexagon NPU of the Snapdragon X Elite cannot be used for ML inference as of mid-2026.** CDSP firmware is loaded by the kernel remoteproc driver (needed for audio offload and sensor hub functionality), but no open compute interface exists above it.

### 8.4 WebNN NPU Backend

The Web Neural Network API (WebNN, see Chapter 168) provides a browser-facing interface for hardware ML acceleration. On Linux with the X Elite, a WebNN NPU backend is effectively blocked by the same Hexagon driver gap described above. WebNN in Chromium currently falls back to CPU (XNNPACK) or GPU (WebGPU/OpenCL) backends on this platform.

---

## 9. Qualcomm's Upstream Commitment

### 9.1 Historical Freedreno Contributions

Qualcomm's relationship with open-source Linux graphics drivers is longer and more substantive than its laptop Linux story might suggest. Rob Clark, who wrote the original `msm` DRM driver and the freedreno Mesa Gallium3D driver, worked at Qualcomm (after stints at TI and Linaro). The freedreno project — encompassing the `msm` kernel driver, Mesa freedreno Gallium3D, and Mesa Turnip Vulkan — represents more than a decade of Qualcomm-affiliated contribution.

More recently, Akhil P Oommen at Qualcomm has been a key upstream contributor for X Elite GPU bring-up patches, including the X185 GPU catalog entry, the IFPC (Inter-Frame Power Collapse) series for GPU power management, and the `gmxc.lvl` power rail handling. [Source: freedreno mailing list archives, 2024–2025]

### 9.2 Upstream Status as of 2026

The kernel side of the X Elite stack — SoC infrastructure, GPU driver, display driver — is substantially in mainline Linux, a fact that distinguishes Qualcomm favorably from some GPU vendors. The following components are upstream as of kernel 6.17:

**In mainline:**
- Core SoC: pinctrl, clocks, interconnect, SMMU, QUP, power domains
- DRM/MSM GPU: Adreno X185 GPU support, IFPC, speedbin
- DRM/MSM display: X1E80100 DPU catalog, DP output, eDP
- IRIS V4L2 decoder: H.264 decode
- remoteproc: ADSP, CDSP firmware loading
- Wi-Fi: QCA6490 / WCN785x via `ath11k` / `ath12k`

**Downstream / out-of-tree or proprietary:**
- NPU/Hexagon compute interface (no upstream driver; no open headers)
- Fan control on several OEM laptops
- USB4 full-bandwidth operation (in progress)
- H.265/AV1 IRIS decode (in development)
- Firmware blobs for GPU and IRIS (proprietary; must be extracted from Windows or obtained via linux-firmware.git for the ThinkPad only)

### 9.3 Firmware Distribution

The Adreno X185 GPU requires firmware blobs at probe time. The linux-firmware-qcom package (as seen in the Arch Linux `linux-firmware-qcom` file list) ships these under the `gen70500` naming scheme used for the X1E80100's GPU generation: `gen70500_sqe.fw` (SQE command processor microcode), `gen70500_gmu.bin` (GMU power management firmware), and `x1e80100/gen70500_zap.mbn` (ZAP shader security firmware). [Source: Arch Linux `linux-firmware-qcom` package file listing](https://archlinux.org/packages/core/any/linux-firmware-qcom/files/) These are proprietary Qualcomm binaries. On most X Elite laptops, the only way to obtain them on a Linux installation is to extract them from the co-installed Windows 11 partition using the `qcom-firmware-extract` tool or equivalent. The one notable exception: **Lenovo ThinkPad T14s Gen 6 firmware is distributed through `linux-firmware.git`**, making it the most "native Linux" X Elite laptop from a firmware-distribution perspective. [Source: community reporting]

This firmware situation echoes the broader open-source challenge: the GPU hardware is well-described in open drivers, but the firmware that boots it remains proprietary.

---

## 10. Practical Setup: Linux on X Elite Laptops

### 10.1 Supported Laptops

As of Ubuntu 25.04 (the first mainstream distribution to declare Snapdragon X Elite "supported"), the following laptop models had functional graphics acceleration on Linux: [Source: PCWorld, "Ubuntu and Tuxedo duke it out for Linux on Snapdragon X Elite laptops"](https://www.pcworld.com/article/2831829/ubuntu-and-tuxedo-duke-it-out-for-linux-on-snapdragon-x-elite-laptops.html)

- **Lenovo ThinkPad T14s Gen 6** (best supported; firmware in linux-firmware.git)
- **Lenovo Yoga Slim 7x**
- **ASUS Vivobook S 15 S5507**
- **Dell XPS 13 9345**
- **HP OmniBook X 14**
- **Acer Swift 14 AI**
- **Microsoft Surface Laptop 7**

### 10.2 TUXEDO's Cancelled Project

TUXEDO Computers invested approximately 18 months in a bespoke Snapdragon X Elite Linux laptop (codename "Drako"), demonstrating a prototype at Computex 2024. The project was ultimately cancelled. TUXEDO cited: battery runtimes falling short of expectations under Linux, no practical path for BIOS updates from Linux, missing fan control, and USB4 ports not reaching expected transfer rates. TUXEDO stated the platform was "less suitable than expected" and announced open-sourcing of all their driver and integration work. They indicated they would revisit the project with the Snapdragon X2 Elite. [Source: VideoCardz, "TUXEDO halts Snapdragon X Elite Linux laptop plans"](https://videocardz.com/newz/tuxedo-halts-snapdragon-x-elite-linux-laptop-plans-may-revisit-with-snapdragon-x2)

### 10.3 Recommended Kernel and Distribution

For end users wanting the best Linux experience on Snapdragon X Elite hardware as of mid-2026:

| Component | Recommendation |
|---|---|
| Kernel | 6.17 or later (speedbin GPU performance, IFPC, IRIS video) |
| Mesa | 25.1 or later (Turnip default on AArch64, X1-85 working) |
| Distribution | Ubuntu 25.04+ (official ARM64 laptop support) or Fedora Rawhide |
| Display server | Wayland with GNOME 47 / KDE Plasma 6.x recommended |
| Firmware | Extract via `qcom-firmware-extract` from Windows partition (or use ThinkPad with linux-firmware.git blobs) |

### 10.4 Known-Working Configuration

A working graphics stack on the ThinkPad T14s Gen 6 looks like:

```bash
# Verify kernel Adreno driver is loaded
$ dmesg | grep -i adreno
[    5.123] msm 990000.mdss: [drm] Initialized msm 1.12.0 ...
[    5.891] adreno 3d000000.gpu: Adreno X185 (0x43050c01) identified

# Check Vulkan device via Turnip
$ vulkaninfo --summary 2>/dev/null | grep -A2 "GPU"
GPU0:
    apiVersion = 1.3.xxx
    driverVersion = xxx
    deviceName   = Turnip Adreno (TM) X1-85

# Verify DRI device
$ ls /dev/dri/
card0  card1  renderD128

# Check Mesa GL renderer
$ glxinfo -B | grep "OpenGL renderer"
OpenGL renderer string: Adreno (TM) X1-85 (LLVM 18.x, DRM 3.xx, ...)
```

### 10.5 Known Limitations as of Mid-2026

- **NPU/Hexagon**: No ML inference acceleration on Linux.
- **USB4**: Transfer rates below rated spec on several laptop models; still maturing.
- **Audio**: Headset jack functional on some models; digital audio paths still work-in-progress.
- **Fan control**: Platform-specific; not functional on all laptops.
- **Hardware video decode**: H.264 only (via IRIS); H.265 and AV1 in development.
- **Suspend/resume**: Functional on some laptops; reliability varies by model.
- **BIOS updates**: No fwupd/LVFS path available on most models.

### 10.6 Community Resources

- **Linaro Snapdragon X Elite blog**: [linaro.org/blog/linux-on-snapdragon-x-elite](https://www.linaro.org/blog/linux-on-snapdragon-x-elite/) — primary upstream integration reference
- **freedreno mailing list**: `freedreno@lists.freedesktop.org` — kernel and Mesa patches
- **Ubuntu ARM community hub**: `discourse.ubuntu.com` — distribution-level support discussions
- **Qualcomm developer blog**: `qualcomm.com/developer/blog` — official upstream status posts
- **linux-arm-msm mailing list**: for SoC-level kernel patches

---

## Roadmap

### Near-term (6–12 months)

- **Snapdragon X2 Elite (X1E80200 / Adreno X2-85) kernel and Mesa support:** Qualcomm has posted patch series targeting Linux 6.19 to add the Adreno X2-85 (`a8xx` family) to the `msm` DRM driver, with freedreno Gallium3D (OpenGL) support landing first and Turnip Vulkan support to follow. [Source: Phoronix, "Qualcomm Upstreaming Initial GPU Support For Snapdragon X2 Elite In Linux 6.19"](https://www.phoronix.com/news/Qualcomm-X2-Elite-GPU-Linux-619)
- **Turnip Vulkan for Adreno Gen 8 (Mesa 26.0+):** Adreno Gen 8 Vulkan support — covering both the Snapdragon X2 and Snapdragon 8 Elite Gen 5 platforms — was merged for Mesa 26.0. Additional conformance work (full dEQP-VK coverage, ray-tracing extensions) is expected across subsequent Mesa releases. [Source: Phoronix, "Adreno Gen 8 Vulkan Graphics Merged For Mesa 26.0"](https://www.phoronix.com/news/Mesa-26.0-Adreno-Gen-8-Graphics)
- **IRIS video decode expansion (H.265, VP9, AV1) on X Elite:** The IRIS V4L2 driver that landed in kernel 6.15 initially covers H.264 decode only on the X1E80100. Qualcomm's upstream driver development (v5/v6 patchsets on `linux-media`) lists H.265 and AV1 decode capabilities in the hardware; bringing these codecs to VA-API and GStreamer on the laptop platform is expected within the near-term window. Note: needs verification for exact target kernel version. [Source: kernel.org Patchwork, `[v6,00/28] Qualcomm iris video decoder driver`](https://patchwork.kernel.org/project/linux-media/cover/20241120-qcom-video-iris-v6-0-a8cf6704e992@quicinc.com/)
- **Per-OEM device-tree and ACPI overlay maturation:** Linaro's Snapdragon X Elite Linux effort continues to land board-specific device trees for additional laptop models (Lenovo ThinkPad T14s Gen 6, HP EliteBook, Dell XPS 13 9345, and others). Each model requires display, USB-C DP alt-mode, and suspend/resume validation. [Source: Linaro blog, "Linux on Snapdragon X Elite"](https://www.linaro.org/blog/linux-on-snapdragon-x-elite/)
- **Panel Self-Refresh (PSR) and eDP power management:** Improvements to `msm_dpu`'s PSR implementation for eDP panels on the X Elite — reducing idle display power toward what Windows achieves — are an active focus area among freedreno contributors. Note: needs verification for specific patch landing target.

### Medium-term (1–3 years)

- **Same-day upstream support for Snapdragon X-series successors:** Qualcomm demonstrated same-day upstream Linux kernel support for the Snapdragon 8 Elite Gen 5 mobile platform, signalling an intent to extend this commitment to future X-series compute platforms. If maintained, Snapdragon X3 or successor SoCs would have day-one kernel support. [Source: Qualcomm developer blog, "Same-day upstream Linux support for Snapdragon 8 Elite Gen 5"](https://www.qualcomm.com/developer/blog/2025/10/same-day-snapdragon-8-elite-gen-5-upstream-linux-support)
- **Hexagon/CDSP NPU driver (`accel/qda` RFC):** The RFC `accel/qda` kernel driver for on-die Hexagon inference acceleration remains the most-watched pending piece of the X Elite Linux stack. Qualcomm's decision not to open-source the DSP firmware headers has stalled progress; re-engagement — perhaps through a firmware-blob model similar to the Raspberry Pi's VPU — remains possible if enterprise Linux demand grows sufficiently. [Source: VideoCardz, "Qualcomm shuts door on Snapdragon X DSP headers open-sourcing"](https://videocardz.com/newz/qualcomm-shuts-door-on-snapdragon-x-dsp-headers-open-sourcing-linux-support-hopes-fade)
- **OpenCL 3.0 conformance via Rusticl or Clover on Turnip:** Mesa's Rusticl (Rust-based OpenCL implementation on top of Gallium3D NIR) is expanding hardware support; bringing X1-85 to a conformant OpenCL 3.0 implementation on Linux would enable compute workloads (image processing, physics simulation) without the proprietary Adreno Control Panel. Note: needs verification for timeline.
- **HDR and wide-colour-gamut display pipeline:** The DPU on the X1E80100 includes hardware DSPP post-processing units capable of HDR tone-mapping. Full HDR support in KMS/KWin/Mutter under the Wayland HDR protocol extension (`color-management-v1`) is a multi-year effort involving both the `msm_dpu` driver and compositor stack changes. [Source: Phoronix, "Big Improvements For Qualcomm GPU Driver With Linux 6.17"](https://www.phoronix.com/news/Qualcomm-MSM-DRM-Linux-6.17)
- **Firmware/fwupd LVFS support for OEM BIOS updates:** No LVFS path exists for X Elite laptop models as of mid-2026; Linaro and OEM partners are expected to work toward this for commercial Linux laptop viability.

### Long-term

- **Full Hexagon NPU open-source stack:** If Qualcomm or the community achieves an open firmware ABI for the Hexagon CDSP, a complete open-source ML inference path — Hexagon kernel driver → ONNX Runtime / OpenVINO backend → WebNN in Chromium — becomes achievable, analogous to what Intel has achieved with IVPU on Meteor Lake.
- **Ray-tracing and mesh-shader extensions on Turnip (A7xx/A8xx):** Qualcomm's A7xx hardware has documented support for hardware ray-tracing acceleration. Turnip's development roadmap is expected to expose `VK_KHR_ray_tracing_pipeline` and mesh shaders as driver maturity increases, making the X Elite viable for Vulkan ray-tracing workloads on Linux.
- **Integration with upcoming Arm laptop ecosystem convergence:** As more Arm-based laptops (Apple M-series via Asahi, Snapdragon X, NVIDIA Thor) target Linux, shared infrastructure for UEFI/DT-overlay boot, Wayland compositing, and Vulkan ICD management is likely to converge. The `msm` driver's approach to ACPI+DT co-existence may inform a broader Arm laptop driver model.
- **Snapdragon X-series in commercial Linux laptop offerings:** TUXEDO halted its X Elite Linux laptop plans citing driver maturity gaps; longer-term re-engagement (possibly with Snapdragon X2 or beyond) would validate the upstream stack at a commercial QA level, accelerating kernel and Mesa development velocity. [Source: VideoCardz, "TUXEDO halts Snapdragon X Elite Linux laptop plans"](https://videocardz.com/newz/tuxedo-halts-snapdragon-x-elite-linux-laptop-plans-may-revisit-with-snapdragon-x2)

---

## 11. Integrations

This chapter connects to several other parts of the book:

**Chapter 6 — ARM and Embedded GPU Drivers**
The `msm` DRM kernel driver that powers the Snapdragon X Elite is the same driver used for Qualcomm's phone SoCs. Chapter 6 covers the general architecture of ARM SoC GPU drivers including the remoteproc model, SMMU integration, and the Device Tree vs. ACPI split that makes the X Elite an interesting transitional case. The X Elite adds UEFI/DT-overlay boot, per-OEM device trees, and ACPI co-existence — all absent from the pure-embedded world.

**Chapter 160 — freedreno/Turnip/Adreno**
Chapter 160 covers the freedreno kernel/Mesa stack in depth: the A6xx and A7xx architectures, Turnip's command-buffer internals, the IR3 compiler, and the UBWC tiling format. The X Elite's Adreno X1-85 is an A7xx-family GPU; everything in Chapter 160 about A7xx applies, with the X Elite adding the ACPI platform environment and the laptop-class use cases (eDP PSR, multi-display DP, Wayland compositor integration at desktop scale).

**Chapter 88 — NPU and AI Accelerators**
Chapter 88 surveys the landscape of hardware ML accelerators on Linux including Apple Neural Engine, Intel NPU (Meteor Lake), and AMD XDNA. The Hexagon NPU on the X Elite is a prominent gap in the Linux accelerator ecosystem: unlike the Intel NPU (which has an upstream IVPU driver and OpenVINO integration) and unlike the Qualcomm Cloud AI 100 (which has the `qaic` driver), the X Elite's on-die Hexagon NPU has no open Linux driver and no published ABI as of mid-2026. The RFC QDA driver may eventually close this gap.

**Chapter 168 — WebNN**
Chapter 168 covers the Web Neural Network API and its browser-level hardware backends. The X Elite's Hexagon NPU limitation directly affects WebNN's acceleration story on Linux: WebNN's NPU backend cannot target the Hexagon on this platform, falling back to WebGPU (which itself uses Turnip) or XNNPACK CPU. When and if a Hexagon/CDSP driver becomes available on Linux, it would unlock a new WebNN backend path — making the `accel/qda` RFC an item to watch.

---

*Sources cited in this chapter:*

- [Chips and Cheese, "The Snapdragon X Elite's Adreno iGPU," 2024](https://chipsandcheese.com/p/the-snapdragon-x-elites-adreno-igpu)
- [Qualcomm developer blog, "Upstreaming Linux kernel support for the Snapdragon X Elite," May 2024](https://www.qualcomm.com/developer/blog/2024/05/upstreaming-linux-kernel-support-for-the-snapdragon-x-elite)
- [LKML, `[PATCH v2 0/5] Support for Adreno X1-85 GPU`, June 2024](https://lore.kernel.org/linux-kernel/20240701055103.srt6olauy7ux5um5@hu-akhilpo-hyd.qualcomm.com/T/)
- [LKML, `[PATCH v2 2/5] drm/msm/adreno: Add support for X185 GPU`](https://lkml.iu.edu/2406.3/08138.html)
- [Patchwork, drivers/gpu: Switching Adreno x1-85 device check to family check, Sep 2024](https://patches.linaro.org/project/linux-arm-msm/patch/20240921204237.8006-1-john.schulz1@protonmail.com/)
- [LWN.net, "drm/msm: Add display support for X1E80100"](https://lwn.net/Articles/962320/)
- [freedreno mailing list, `[PATCH 5/5] drm/msm/dpu: Add X1E80100 support`](https://www.mail-archive.com/freedreno@lists.freedesktop.org/msg29593.html)
- [Mesa docs, Freedreno](https://docs.mesa3d.org/drivers/freedreno.html)
- [Mesa docs, IR3 notes](https://docs.mesa3d.org/drivers/freedreno/ir3-notes.html)
- [Phoronix, "Mesa's Freedreno Gallium3D Driver Lands Initial Adreno 700 Series Support"](https://www.phoronix.com/news/Mesa-24.3-Freedreno-A7xx)
- [Phoronix, "Open-Source Qualcomm Adreno Vulkan Driver Matures To Default AArch64 Mesa Driver List"](https://www.phoronix.com/news/Mesa-25.1-Turnip-ARM64-Default)
- [Phoronix, "Adreno Gen 8 Vulkan Graphics Merged For Mesa 26.0"](https://www.phoronix.com/news/Mesa-26.0-Adreno-Gen-8-Graphics)
- [Phoronix, "Big Improvements For Qualcomm GPU Driver With Linux 6.17"](https://www.phoronix.com/news/Qualcomm-MSM-DRM-Linux-6.17)
- [Phoronix forums, "Qualcomm Iris Video Decode Driver Ready For Linux 6.15"](https://www.phoronix.com/forums/forum/hardware/general-hardware/1536294-qualcomm-iris-video-decode-driver-designware-hdmi-input-support-ready-for-linux-6-15)
- [Linux kernel docs, `accel/qaic` (Cloud AI 100)](https://docs.kernel.org/accel/qaic/aic100.html)
- [VideoCardz, "Qualcomm shuts door on Snapdragon X DSP headers open-sourcing"](https://videocardz.com/newz/qualcomm-shuts-door-on-snapdragon-x-dsp-headers-open-sourcing-linux-support-hopes-fade)
- [VideoCardz, "TUXEDO halts Snapdragon X Elite Linux laptop plans"](https://videocardz.com/newz/tuxedo-halts-snapdragon-x-elite-linux-laptop-plans-may-revisit-with-snapdragon-x2)
- [PCWorld, "Ubuntu and Tuxedo duke it out for Linux on Snapdragon X Elite"](https://www.pcworld.com/article/2831829/ubuntu-and-tuxedo-duke-it-out-for-linux-on-snapdragon-x-elite-laptops.html)
- [Linaro blog, "Linux on Snapdragon X Elite"](https://www.linaro.org/blog/linux-on-snapdragon-x-elite/)
- [Tweaktown, "Qualcomm Adreno X1 GPU detailed: specs, performance"](https://www.tweaktown.com/news/98810/qualcomm-adreno-x1-gpu-detailed-specs-performance-adreno-control-panel-shown-off/index.html)
- [Arch Linux, linux-firmware-qcom package file list (firmware naming reference)](https://archlinux.org/packages/core/any/linux-firmware-qcom/files/)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
