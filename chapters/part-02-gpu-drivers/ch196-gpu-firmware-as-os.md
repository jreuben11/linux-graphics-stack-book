# Chapter 196: The GPU as Embedded Computer — Firmware-as-OS Architecture Across Vendors

**Target audiences**: Systems developers, kernel/driver authors, and security engineers working on GPU trust models. This chapter assumes familiarity with the mechanics of firmware loading (see Ch. 129) and focuses instead on the *architectural classification* of each vendor's firmware model and the strategic implications of the industry-wide trend toward firmware-as-OS.

---

## Table of Contents

1. [The GPU as Embedded Computer: Architecture Overview](#1-the-gpu-as-embedded-computer-architecture-overview)
2. [NVIDIA GSP-RM: The Most Complete Firmware-OS](#2-nvidia-gsp-rm-the-most-complete-firmware-os)
3. [AMD's Tiered Firmware Architecture](#3-amds-tiered-firmware-architecture)
4. [Intel's GuC/HuC/GSC Model: Firmware-Assisted Submission](#4-intels-guchucgsc-model-firmware-assisted-submission)
5. [Qualcomm Adreno: The Minimal Firmware Model](#5-qualcomm-adreno-the-minimal-firmware-model)
6. [Other Vendors: Broadcom, ARM Mali, and Apple AGX](#6-other-vendors-broadcom-arm-mali-and-apple-agx)
7. [The Spectrum: Accelerator to Firmware-OS](#7-the-spectrum-accelerator-to-firmware-os)
8. [Security and Trust Model Comparison](#8-security-and-trust-model-comparison)
9. [Open-Source Driver Feasibility at the Firmware Layer](#9-open-source-driver-feasibility-at-the-firmware-layer)
10. [The Convergence Trajectory](#10-the-convergence-trajectory)
11. [Strategic Outlook](#11-strategic-outlook)
12. [Integrations](#12-integrations)
13. [References](#13-references)

---

## 1. The GPU as Embedded Computer: Architecture Overview

The prevailing mental model of a GPU — silicon that a CPU drives by writing registers — has been obsolete for nearly a decade. Today's discrete and integrated GPUs are better understood as networked embedded computers that happen to share a PCIe bus (or an SoC interconnect) with the host processor. The CPU-side kernel driver is increasingly a firmware *client*, not a hardware *programmer*.

A modern GPU die contains two categories of silicon. The first category is what most people think of when they say "GPU": shader engines, raster/ROPs, texture units, fixed-function video encode/decode blocks, and display timing engines. The second category is a collection of dedicated management microprocessors that govern the first. These embedded processors are diverse:

- **NVIDIA**: Falcon v6 or RISC-V (GSP), additional Falcons for PMU/SEC2/FECS/GPCCS
- **AMD**: ARM Cortex-A5 (PSP/AMD-SP), a second power-management microprocessor (SMU), an Xtensa-family DSP (DMCUB for display), and GFX ME/MEC microcode running on the CP (Command Processor) itself
- **Intel**: i486/Pentium-compatible microcontroller (GuC), a dedicated video co-processor (HuC), and a separate Graphics Security Controller (GSC/CSE) for security operations
- **ARM Mali (CSF generation)**: ARM Cortex-M7 inside the CSF frontend for job queue scheduling
- **Apple AGX**: ARM64 application-class coprocessor running Apple's proprietary RTKit firmware OS

These embedded processors run firmware that manages hardware init, clock and power domains, resource allocation, security enforcement, and — in the most complete cases — every aspect of GPU virtualisation. The host CPU driver submits work through structured interfaces defined by the firmware: message queues, command transport rings, or doorbell-driven ring buffers whose protocol is specified by the firmware, not the hardware documentation.

The historical contrast is sharp. A pre-2012 GPU driver directly programmed hardware registers to manage clocks, set up context descriptors, and program command FIFOs. Bugs were a matter of writing the wrong value to the right address at the wrong time. A modern driver bug may be a malformed RPC payload, a missing firmware feature flag, or an incorrect sequence of firmware-defined state machine transitions. The hardware registers are still there, but many of them are now owned by the firmware and off-limits to the host driver.

This architectural shift has consequences at every level of the stack:

- **Driver development**: The host driver must implement a firmware-defined protocol, not a hardware specification. The firmware version becomes a compatibility matrix dimension alongside the kernel version and GPU generation.
- **Debugging**: Bugs that manifest as GPU hangs may be inside the firmware, unreachable by the community's debugging tools.
- **Security**: The firmware is an independent trusted execution environment with DMA access to system memory. Its attack surface is separate from — and sometimes larger than — the host driver.
- **Open-source feasibility**: A complete open-source driver now requires either firmware source code, documented firmware protocols, or reverse-engineered protocol knowledge. Without at least one of these, the driver is at best a partial implementation.

---

## 2. NVIDIA GSP-RM: The Most Complete Firmware-OS

The NVIDIA GPU System Processor Resource Manager (GSP-RM) represents the industry's most complete realisation of the firmware-as-OS pattern. This section characterises it architecturally; the implementation details (RPC boot sequence, nvkm integration, open-gpu-kernel-modules source structure) are covered in Ch. 9.

### Processor Evolution: Falcon to RISC-V

NVIDIA's embedded processor story begins with the Falcon microcontroller, a proprietary in-house design used since the Tesla generation for security-sensitive firmware. On Turing and Ampere GPUs, the GSP itself is built on a Falcon v6 variant extended with RISC-V support. On later datacenter silicon (Hopper GH100, Blackwell GB100/GB202), NVIDIA transitioned the GSP fully to a capable RISC-V configuration, with a different boot path called FMC (First Stage Microcode). [Source](https://github.com/NVIDIA/open-gpu-kernel-modules)

The transition from Falcon to RISC-V is architecturally significant: Falcon is a NVIDIA-proprietary ISA with limited public tooling. RISC-V is a standard open ISA, which in principle makes the firmware more amenable to community analysis — though NVIDIA's firmware remains signed and the signing key is not public.

### The Resource Manager Moves to Silicon

In NVIDIA's architecture, the Resource Manager (RM) is the software layer responsible for everything outside rendering: hardware initialisation, clock and voltage programming, power domain management, thermal monitoring, engine context management, and virtualisation multiplexing. Historically, RM ran on the CPU as part of `nvidia.ko` — a several-million-line body of code whose sensitivity derived precisely from the intimate hardware knowledge it encoded.

With GSP-RM, NVIDIA relocated RM off the CPU and onto the GSP processor itself. The CPU-side driver becomes a thin client: it sends structured RPC calls over a shared-memory message queue, and the firmware's RM executes the actual hardware operations. The split is clean: the GPU's engines, clock trees, and power domains are managed exclusively by the GSP.

```
Host CPU                          GPU Die
─────────────────                ─────────────────────────────────
nvidia.ko / nova-core            ┌─ GSP (Falcon/RISC-V) ─────────┐
  GEM / TTM / DRM scheduling     │  GSP-RM firmware               │
  KMS / display                  │  ├─ Hardware init               │
  drm_syncobj / fences           │  ├─ Clock / voltage control     │
  DMA-BUF                        │  ├─ Power domain management     │
       │                         │  ├─ Context / channel alloc     │
       │ RPC over shared-memory  │  └─ vGPU resource partitioning  │
       ╰────────────────────────►│                                 │
                                 └── DMA access to system RAM ─────┘
```

### What the CPU Driver Actually Does

With GSP-RM, the CPU-side kernel module's responsibilities are: loading and booting the GSP firmware (verified by hardware signature check), maintaining the shared-memory message queue, performing GEM/TTM memory management (the GPU memory allocator stays on the CPU side), implementing KMS/display paths, and handling `drm_syncobj` synchronisation primitives. It does *not* directly program clock trees, voltage rails, or engine initialisation sequences — those are RPC calls to the firmware.

The Nova driver (merged in Linux 6.15, written in Rust) was designed from scratch around this model: `nova-core` handles the GSP boot and RPC infrastructure, while `nova-drm` implements the DRM interfaces on top. The explicit design assumption is that the hardware is opaque except through the GSP protocol. [Source](https://www.kernel.org/doc/html/v6.16-rc6/gpu/nova/index.html)

### Trust Model

The GSP firmware is cryptographically signed with NVIDIA's key. The GSP hardware verifies this signature before executing the firmware; a build from the open-gpu-kernel-modules source cannot boot on real hardware without NVIDIA's signing key. [Source](https://github.com/NVIDIA/open-gpu-kernel-modules)

The GSP has DMA access to system memory, isolated from the main GPU engines by IOMMU mapping. In vGPU deployments, each virtual function has a separate channel to GSP-RM; the hypervisor driver does not need direct hardware RM access. vGPU licensing enforcement is also mediated through GSP-RM, making the firmware a policy enforcement point for commercial features.

### Openness Classification

- Kernel module (CPU side): open (MIT/GPLv2) since May 2022
- GSP-RM firmware binary: proprietary, signed, in `linux-firmware`
- Protocol (RPC encoding): partially documented via `open-gpu-doc` headers
- Userspace (CUDA, libVulkan, libcuda): proprietary, no open equivalent

---

## 3. AMD's Tiered Firmware Architecture

AMD's approach to firmware is fundamentally different from NVIDIA's: rather than moving a single monolithic resource manager to a firmware processor, AMD uses *tiered* firmware — separate processors for separate concerns — while retaining most GPU operation in open kernel code.

### PSP: Platform Security Processor (Root of Trust)

The AMD Platform Security Processor (also called AMD-SP, rebranded from PSP) is a dedicated ARM Cortex-A5 processor embedded in the same die as the GPU (on APUs) or on the CPU die (on discrete GPU+CPU systems). [Source](https://en.wikipedia.org/wiki/AMD_Platform_Security_Processor) It runs TrustZone firmware and serves as the root of trust for the entire platform:

- It boots before the x86 cores, verifying the BIOS/UEFI image
- For the GPU: every other firmware blob (GFX, SDMA, VCN, DMCUB) must be authenticated by the PSP before the corresponding GPU IP block can initialise
- It handles AMD SEV (Secure Encrypted Virtualization) for confidential computing
- It manages XGMI (cross-GPU interconnect) security setup on multi-GPU systems
- RAS (Reliability, Availability, Serviceability) error correction is PSP-mediated

The PSP is the hardest firmware dependency for amdgpu: without PSP authentication, no GPU engine can start. The PSP firmware itself is binary-only and closed, shipped in `linux-firmware` as `amdgpu/psp_*_sos.bin` (Secure OS) and `psp_*_ta.bin` (Trusted Application binaries). [Source](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)

### SMU: System Management Unit (Power/Thermal OS)

The System Management Unit is a separate embedded processor (on modern AMD GPUs, also ARM-based or a proprietary core) that runs the power and thermal management firmware. It handles:

- Voltage/frequency scaling (DVFS)
- TDP enforcement and power limit management
- Thermal throttling with microsecond granularity
- Deep power state transitions (GFX off, memory retention modes)
- Fan control and sensor aggregation

The amdgpu driver communicates with the SMU through a set of message registers (`amdgpu_smu_*` functions in `drivers/gpu/drm/amd/pm/swsmu/`). The SMU firmware is binary-only in `linux-firmware`.

### DMCUB: Display Microcontroller (Display Engine Firmware)

Starting with DCN 2.0 (Navi10/Renoir era), AMD added the DMCUB (Display MicroController Unit B) — an embedded Tensilica Xtensa-class DSP that runs display management firmware. DMCUB handles:

- Panel Self Refresh (PSR) state machine
- Adaptive Backlight Management (ABM)
- Real-time link training responses during monitor hotplug
- Display Clock (DCLK) programming sequences that are too timing-sensitive for the CPU driver

The DMCUB firmware ships as `amdgpu/dcn*_dmcub.bin` in `linux-firmware`. DMCUB updates for RDNA 2/3 ASICs (DCN 3.1, DCN 3.2, etc.) are ongoing as AMD fixes display compatibility issues — a notable pattern where display firmware bugs manifest as kernel-level `[drm] dc_dmub_srv_log_diagnostic_data` errors. [Source](https://phoronix.com/news/Renoir-DMCUB-AMDGPU-Patches)

### GFX ME/MEC: Command Processor Firmware

The main GPU graphics engine is controlled by the CP (Command Processor), which reads PM4 ring buffers. The CP runs GFX ME (Micro Engine) firmware for graphics queues and MEC (Micro Engine Compute) firmware for compute queues. These are traditional microcode blobs that implement PM4 packet decoding, indirect buffer chaining, and context switch sequencing. They ship as `amdgpu/gfx*_mec.bin` and `amdgpu/gfx*_me.bin`.

Crucially, unlike NVIDIA's GSP-RM, the GFX ME/MEC firmware is *not* a resource manager. It implements the low-level command stream processing, but the `amdgpu` driver still does direct register programming for engine initialisation, context setup, and page table management. The CP firmware is a narrow protocol layer, not a general-purpose OS.

### MES: Micro Engine Scheduler (Hardware Queue Scheduler)

The Micro Engine Scheduler (MES) is a firmware component introduced with RDNA 2/3 that runs on the CP hardware and handles dynamic mapping of software work queues to hardware queue descriptors (HQDs). It supports oversubscription — more software queues than hardware HQDs — by implementing preemption and time-slicing at the hardware queue level. [Source](https://docs.kernel.org/gpu/amdgpu/gc/mes.html)

AMD published MES firmware documentation for RDNA 3 without NDA in 2024, and has announced plans to open-source MES firmware for select GPU generations. [Source](https://www.phoronix.com/news/AMD-MES-Firmware-Docs) MES applies to compute and GFX scheduling; it is an RDNA-era feature and distinct from the display-oriented DMCUB or the security-oriented PSP.

### VCN: Video Core Next Firmware

The Video Core Next (VCN) block implements hardware video encode/decode (H.264, H.265/HEVC, AV1 on RDNA 3+). It runs its own firmware (`amdgpu/vcn_*_unified.bin`) and has a separate command ring managed by the amdgpu driver through `amdgpu_vcn.c`.

### Key Contrast with NVIDIA

The critical architectural difference from NVIDIA is this: **amdgpu still performs direct register programming for the main GFX/3D engine operation, memory management, and KMS.** The firmware is present and required for specific subsystems (PSP for security, SMU for power, DMCUB for display, MES for queue scheduling), but the GPU's core rendering pipeline is initialised through the IP block chain in open kernel code. The `amdgpu_device_ip_init()` path in `drivers/gpu/drm/amd/amdgpu/amdgpu_device.c` calls `hw_init()` on each IP block, most of which program hardware registers directly. [Source](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/amd/amdgpu/amdgpu_device.c)

### Openness Classification

- GFX kernel driver: fully open (RADV, radeonsi, OpenCL written with vendor cooperation)
- PSP firmware: binary-only, closed source, required for boot
- SMU firmware: binary-only
- DMCUB firmware: binary-only
- GFX ME/MEC/MES firmware: binary-only; AMD has begun open-sourcing select MES firmware
- Protocol (IP block register interfaces): documented in AMD's GPUOpen and kernel headers

---

## 4. Intel's GuC/HuC/GSC Model: Firmware-Assisted Submission

Intel's GPU firmware architecture sits between AMD's tiered model and NVIDIA's full firmware-OS. The GPU's hardware initialisation and memory management are firmly in the open kernel driver; the firmware handles *scheduling policy* and *security operations*.

### GuC: The Scheduling Co-Processor

The Graphics Micro-Controller (GuC) is implemented using an i486DX4-compatible processor (Intel calls it "Minute IA" or P24C) running a microkernel called μOS written in C. [Source](https://igor-blue.github.io/2021/02/10/graphics-part1.html) It runs in 32-bit protected mode and is responsible for:

- Workload submission scheduling (which engine processes which batch buffer)
- Power management assist (C6/RC6 power gating coordination)
- HuC firmware authentication (GuC verifies HuC's signature before HuC activates)
- Performance monitoring and telemetry

Critically: the submission model on Intel Xe hardware is **GuC-mandatory**. All workloads are submitted through the GuC CT (Command Transport) protocol: the host driver writes H2G (Host-to-GuC) messages, and GuC sends G2H (GuC-to-Host) completions. [Source](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/xe/xe_guc_ct.c) This means a GuC firmware bug blocks all GPU rendering — the driver has no fallback path to submit work without firmware.

The CT protocol is fully documented in the xe/i915 kernel source headers, making it the most transparent firmware submission protocol in the industry.

### HuC: Video Authentication

The HuC firmware is a video co-processor for H.265/HEVC and H.264 decode and encode acceleration. Its distinctive property is that HuC **requires GuC to authenticate it** before it can activate: GuC verifies HuC's RSA signature and only then enables HuC's execution. Without GuC authentication, HuC remains disabled and video codec acceleration falls back to software. [Source](https://igor-blue.github.io/2021/02/10/graphics-part1.html)

### GSC: Graphics Security Controller (Xe Generation)

The GSC (Graphics System Controller) is an Intel IP block embedded in the media/display subsystem of discrete Xe-generation GPUs. It replaces the function previously handled by the platform-level CSME (Converged Security and Management Engine) for GPU-specific security operations. [Source](https://edc.intel.com/content/www/us/en/design/products/platforms/details/arrow-lake-s/core-ultra-200s-series-processors-datasheet-volume-1-of-2/intel-graphics-system-controller-intel-gsc/)

GSC responsibilities include:
- HDCP (High-bandwidth Digital Content Protection) authentication
- PXP (Protected Xe Path) sessions for premium 4K/HDR content DRM — encrypted GPU memory access from a hardware TEE [Source](https://www.phoronix.com/news/Intel-PXP-Protected-Xe-Path)
- HuC attestation pipeline on Xe (via `mei_pxp`)
- GPU firmware update management via the `igsc` library

PXP is Intel's L1 hardware DRM implementation: the GPU firmware establishes an encrypted session, and only that session can decode and scan out protected premium content. This is the Intel equivalent of NVIDIA's vGPU attestation — firmware enforcing a security policy that the CPU-side driver cannot bypass.

### DMC: Display Microcontroller

The Display Micro-Controller (DMC) firmware handles display clock gating and deep power states (DC5/DC6) during display idle periods. It is loaded by the kernel driver and runs autonomously to gate display engine clocks when no display activity is occurring, resuming on demand.

### Key Contrast with NVIDIA and AMD

Intel's model: GuC handles *scheduling*, not resource management. The `xe.ko` and `i915.ko` drivers still do all hardware initialisation, memory management via TTM/GEM, KMS modesetting, and GPU page table management through direct register access. GuC is a policy layer inserted between the driver and the hardware *execution units*, not between the driver and the hardware *management infrastructure*. This is a materially smaller firmware footprint than NVIDIA's GSP-RM.

### Openness Classification

- Xe/i915 kernel driver: fully open
- GuC firmware: binary-only in `linux-firmware`; CT submission protocol fully documented in open kernel source
- HuC firmware: binary-only
- GSC firmware: binary-only
- DMC firmware: binary-only
- PXP protocol: documented in i915/xe kernel source

---

## 5. Qualcomm Adreno: The Minimal Firmware Model

Among the major GPU vendors with significant Linux desktop/laptop presence, Qualcomm's Adreno GPU family has the most register-programming-centric driver model — closer to the pre-firmware era than to NVIDIA's GSP-RM.

### freedreno Architecture

The `msm` DRM kernel driver (`drivers/gpu/drm/msm/`) programs Adreno GPU hardware directly: ring buffer construction, register writes for engine state, GPU MMU (IOMMU/SMMU) management, and GPU scheduler operation are all in-kernel open code. There is no AMD-PSP-equivalent authentication chain, no NVIDIA-GSP-equivalent resource manager, and no Intel-GuC-equivalent scheduling co-processor.

The CP (Command Processor) in Adreno GPUs executes PM4-like microcode from ring buffers. The kernel driver builds these rings using the `msm_ringbuffer` infrastructure. Engine initialisation involves direct register writes to Adreno control registers. [Source](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/msm/)

### Zap Shader: Security Firmware (Required on Mobile SoCs)

The one mandatory firmware dependency for Adreno on mobile SoCs is the **zap shader** — a small Qualcomm TrustZone-mediated firmware image that transitions the GPU out of its post-boot "secure mode". On mobile SoCs, the GPU powers up in a hardware-locked secure state where it can only render to protected memory. The zap shader (loaded via `qcom_scm_pas_init_image()`) interacts with the ARM TrustZone Secure Monitor Call (SMC) interface to remove this restriction. [Source](https://www.phoronix.com/news/MSM-A6xx-Zap-Shader)

The zap shader firmware (`a*_zap.mbn`) is loaded once at driver init. On desktop SoCs like the Snapdragon X Elite (SC8380), some bootloaders do not enforce the secure boot restriction, making the zap shader optional.

### CP Microcode: Partially Open

The Adreno CP microcode (ring buffer instruction set microcode, distinct from the host PM4 protocol) has historically been proprietary. Unlike Intel GuC or AMD MEC firmware, which run as general-purpose processors with their own C code, Adreno CP microcode is narrowly scoped — it is closer to a fixed-function ring buffer interpreter than an embedded OS.

Qualcomm has provided community documentation for newer GPU generations (Adreno 6xx and 7xx register specs have been published), enabling the freedreno community to write the driver without reverse engineering. The Turnip Mesa Vulkan driver achieves near-parity with the proprietary Adreno driver on A7xx hardware, something that would be impossible without documentation. [Source](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/msm/)

### Why the Minimal Firmware Model Persists

Qualcomm's low firmware dependency is not accidental. It reflects the mobile SoC ecosystem where the GPU is a tightly integrated subsystem, the bootloader controls secure boot transitions, and there is no virtualisation use case demanding firmware-mediated resource partitioning. DVFS is handled by the SoC's PMIC firmware and the Linux `devfreq` framework, not a GPU-internal SMU equivalent.

This makes the Adreno model the closest surviving example of the "old" GPU driver paradigm — and the existence proof that the firmware-as-OS trend is not technologically inevitable. It is a vendor choice.

### Openness Classification

- MSM/freedreno kernel driver: fully open
- Zap shader firmware: binary-only, TrustZone-mediated, required on mobile
- CP microcode: binary-only for older generations; register/protocol documentation available for 6xx/7xx
- Turnip Vulkan: fully open Mesa implementation

---

## 6. Other Vendors: Broadcom, ARM Mali, and Apple AGX

### Broadcom V3D: Minimal Firmware (Raspberry Pi 4/5)

The V3D driver (`drivers/gpu/drm/v3d/`) for the VideoCore VI/VII GPU in Raspberry Pi 4 and 5 operates without firmware: the kernel driver communicates directly with V3D hardware registers and the QPU (Quad Processing Unit) shader engines through a ring buffer. The V3D driver was written by Broadcom engineers and open-sourced with full register documentation. [Source](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/v3d/)

The older VideoCore IV (vc4 driver, Raspberry Pi 1-3) also has no GPU firmware dependency. There is a separate VPU (Video Processing Unit) firmware for the BCM2835/2836/2837 VideoCore that handles display, codec, and camera processing, but this is a platform-level firmware (loaded before Linux starts, separate from the GPU driver), not GPU engine firmware in the sense discussed in this chapter. The Mesa driver explicitly avoids the closed VPU firmware for its 3D rendering path.

Broadcom V3D therefore sits at the register-programming end of the spectrum: essentially zero firmware dependency for 3D rendering.

### ARM Mali CSF: Approaching Firmware-as-OS for Scheduling

The most significant recent shift in the ARM Mali ecosystem is the introduction of CSF (Command Stream Frontend) in the Valhall GPU generation (Mali-G57/G68/G77/G78 in the second Valhall iteration, and all subsequent generations). CSF fundamentally changes how the GPU receives work:

**Pre-CSF (Job Manager, Midgard/Bifrost/first Valhall)**: The CPU driver writes job descriptors to hardware Job Slots (JS) registers. The hardware polls these registers for new work. Preemption is software-managed by the kernel driver.

**CSF (second Valhall / third generation onward, Mali-G310/G510/G610/G710)**: An embedded ARM Cortex-M7 microcontroller inside the GPU runs the CSF firmware (`mali_csffw.bin`). This microcontroller manages command stream queues: software submits command stream buffers to firmware-defined queue interfaces, and the Cortex-M7 schedules them onto GPU hardware queues, handles preemption, and manages the tiler heap. [Source](https://www.collabora.com/news-and-blog/news-and-events/pancsf-a-new-drm-driver-for-mali-csf-based-gpus.html)

The Panthor kernel driver (mainlined in Linux 6.10) targets CSF-generation Mali GPUs. The submission path — `DRM_IOCTL_PANTHOR_GROUP_SUBMIT` — sends command streams to firmware-managed groups, not to hardware register slots. From the driver's perspective, the GPU's execution model is now defined by the firmware protocol, not the hardware register specification. [Source](https://lwn.net/Articles/953784/)

The CSF firmware (`mali_csffw.bin`) is binary-only and essential: without it, the GPU will not execute any work. ARM ships this blob in `linux-firmware`.

The implication is significant: Mali CSF GPUs now require reverse-engineered *firmware protocol* knowledge to write an open-source driver, not just reverse-engineered register knowledge. Panthor was developed with some ARM cooperation, but the detailed CSF protocol spec is not public. This is a deliberate architectural convergence toward the NVIDIA/Intel model.

### Apple AGX: Adversarial Firmware RE

Apple Silicon GPUs present the most extreme case: every aspect of GPU operation — hardware initialisation, command submission, shader execution, display output — is mediated through Apple's proprietary firmware OS running on a dedicated ARM64 coprocessor called the ASC (Application System Controller, or GPU firmware processor). [Source](https://asahilinux.org/docs/hw/soc/agx/)

Communication follows the RTKit framework: the CPU driver exchanges messages with the firmware over shared DRAM using a mailbox-and-ring-buffer protocol. The firmware handles work queue management, tiler buffer allocation, context management, and event signalling — essentially a complete GPU resource manager equivalent to NVIDIA's GSP-RM, but undocumented and not intended to be replaced.

Apple Silicon GPU generations and their firmware architecture:
- **M1 (G13G/G13X)**: First Linux-targeted generation; RTKit firmware protocol reverse-engineered by Asahi team
- **M2 (G14G/G14X)**: Extended ISA with new compute features; Asahi driver extended to cover
- **M3 (G15G/G15X)**: Hardware ray tracing and mesh shaders; ISA reverse engineering ongoing
- **M4 (G16G/G16X)**: Dynamic Caching GPU architecture; Asahi team working on support

[Source: dougallj/applegpu](https://github.com/dougallj/applegpu)

The Asahi team's methodology combines: macOS IOKit call interception using `agxdecode`, GPU trace comparison between macOS and Linux execution paths, shader disassembly via open-source ISA tools, and inference from hardware behaviour. It is the most sophisticated GPU firmware reverse-engineering project currently active. The resulting `drm/asahi` driver (written in Rust) was submitted for upstream inclusion with the UAPI header landing in Linux 6.13.

The firmware write-protection enforced by Apple's bootloader means the Asahi team cannot run community-built firmware even if they had the source; they must implement a compatible Linux driver that communicates with Apple's unmodified firmware binary using the reverse-engineered protocol.

### All-Vendor Summary Table

| Vendor/Driver | Firmware Scope | CPU Driver Role | Firmware Openness | Open Driver Viable? |
|---|---|---|---|---|
| **Broadcom V3D** | None (3D) | Full HW programming | N/A | Yes (fully open) |
| **Qualcomm Adreno (freedreno)** | Zap shader (security only) | Full HW programming | Zap: binary-only; registers documented | Yes (fully open) |
| **AMD amdgpu** | PSP (root of trust), SMU (power), DMCUB (display), MES (queue sched) | GFX direct register programming | Binary-only; MES docs released | Yes (amdgpu is fully open) |
| **Intel Xe/i915** | GuC (scheduling), HuC (video), GSC (security), DMC (display power) | HW init, MM, KMS direct | Binary-only; CT protocol documented | Yes (Xe/i915 fully open) |
| **ARM Mali CSF (Panthor)** | CSF firmware (job scheduling) | Command stream construction | Binary-only; protocol partially RE | Harder (Panthor + limited ARM cooperation) |
| **Apple AGX (Asahi)** | Full GPU OS (RTKit) | Firmware protocol client | Binary-only; fully RE by Asahi | Extremely hard (full RE required) |
| **NVIDIA (Nouveau/Nova)** | Full Resource Manager (GSP-RM) | RPC client; GEM/KMS/display | Binary-only; RPC protocol via open-gpu-doc | Yes, demonstrated by Nova/NVK |

---

## 7. The Spectrum: Accelerator to Firmware-OS

The vendors above distribute across a meaningful spectrum. Understanding where each falls — and why — frames the strategic analysis that follows.

```
Register-Programming          Firmware-Assisted           Firmware-as-OS
      │                              │                          │
      ▼                              ▼                          ▼
  Broadcom V3D          Intel GuC (scheduling)        NVIDIA GSP-RM (full RM)
  Qualcomm Adreno       AMD PSP+SMU (security+power)  Apple AGX (full GPU OS)
  (minimal firmware)    ARM Mali CSF (job sched)
```

**Register-programming pole**: The CPU driver reads and writes hardware registers directly. Firmware is absent or limited to a narrow transition helper (zap shader). Driver correctness is a matter of register specification compliance. Community reverse engineering is feasible because the observable state is hardware state. Representative: V3D, Adreno.

**Firmware-assisted**: The CPU driver retains most hardware control, but a firmware co-processor handles one or two specific concerns (scheduling policy, security authentication, power management). The firmware is narrow-scope and the protocol is often documented. Bugs in the firmware affect those concerns but do not prevent the driver from programming hardware directly. Representative: Intel GuC, AMD amdgpu.

**Firmware-as-OS**: The firmware owns the hardware management plane entirely. The CPU driver cannot meaningfully operate without the firmware, and cannot bypass it even for debugging. The firmware protocol is the driver's only hardware interface. Hardware register access is either absent or redirected through the firmware. Representative: NVIDIA GSP-RM, Apple AGX.

**An important nuance**: AMD is not uniformly in one category. For the GFX/3D engine, AMD remains in the firmware-assisted or even register-programming camp. For security (PSP) it is closer to firmware-as-OS for that specific subsystem. AMD's architecture is a deliberate hybrid, preserving register-level control where open-source driver development benefits most, while using firmware OS patterns where security or power management justify them.

ARM Mali CSF is transitional: it moved job scheduling from register-based job slots into firmware, but the firmware's scope is limited to that scheduling layer. The driver still constructs command streams. Whether ARM expands the CSF firmware scope in future generations is an open question with significant implications for Panthor's architecture.

---

## 8. Security and Trust Model Comparison

### IOMMU and Firmware DMA

Every GPU firmware processor has DMA access to system memory for message queues, firmware scratch areas, and firmware-managed GPU memory. This DMA access creates a security boundary: if the firmware is compromised, it can read and write arbitrary system memory within its IOMMU aperture.

All major vendors use the host IOMMU (Intel VT-d, AMD-Vi, ARM SMMU) to constrain firmware DMA to a defined memory region. The GSP processor's DMA aperture is mapped through the IOMMU; PSP firmware access to system memory is similarly constrained. However, the firmware is trusted to operate within these constraints: the IOMMU prevents *accidental* out-of-bounds access but does not protect against a firmware that deliberately targets its allowed aperture to reconstruct data from other processes' mapped regions.

### Hardware Signature Verification

All major vendors enforce cryptographic signature verification on GPU firmware:
- **NVIDIA**: GSP hardware verifies RSA signature before executing firmware. NVIDIA's signing key; source-built firmware cannot boot.
- **AMD**: PSP verifies all other firmware images using AMD's PKI. The PSP boot ROM is immutable.
- **Intel**: GuC boot ROM verifies firmware using SHA-256 + PKCS v2.1 RSA. [Source](https://igor-blue.github.io/2021/02/10/graphics-part1.html)
- **ARM Mali CSF**: ARM signs the CSFFW blob; the CSF hardware verifies it.
- **Apple AGX**: Apple's bootloader loads the firmware and enforces write protection; the ASC does not execute unsigned firmware.

The security implication is double-edged: signature verification prevents running tampered firmware, but it also prevents security researchers from running patched firmware to investigate vulnerabilities. The community cannot build and test a fixed version of GSP-RM firmware even if a vulnerability is identified in the published Falcon/RISC-V binary.

### vGPU and SR-IOV Isolation

GPU virtualisation requires firmware-enforced isolation between virtual functions:
- **NVIDIA**: GSP-RM is the virtualisation arbiter. Each VF has a separate channel to GSP-RM; the firmware enforces resource partitioning. vGPU licensing checks are also GSP-RM-mediated, making the firmware a commercial policy enforcement point.
- **AMD**: PSP manages SRIOV attestation and memory encryption (SEV-SNP) for confidential computing. The GFX driver handles VF submission rings, but PSP is the security authority.
- **Intel**: GuC handles multi-engine scheduling across VFs. GSC enforces PXP session isolation for DRM content.

### Attestation

Firmware-as-OS architectures enable hardware-backed attestation:
- **NVIDIA**: GSP-RM provides CUDA attestation for confidential computing (CC mode); the firmware signs attestation reports that cannot be forged by a compromised CPU driver.
- **Intel**: PXP session attestation for premium content playback. GSC is the TEE anchor.
- **AMD**: PSP provides SEV-SNP VM attestation for confidential computing workloads.

None of this is available in the register-programming model (V3D, Adreno) — attestation requires a trusted firmware executor.

### Threat Surface Analysis

The firmware layer introduces a distinct attack surface:
- Firmware vulnerabilities are harder to discover (no source, limited dynamic analysis tools)
- Firmware bugs may persist longer (firmware update requires distribution shipping new blobs + user rebooting)
- The firmware-CPU interface (message queues, shared memory) can be attacked from both sides: a compromised host driver could send malformed RPC payloads; a compromised firmware could corrupt host memory
- Physical security: with DMA access to system RAM, a compromised GPU firmware is equivalent to a DMA attack; IOMMU mitigates but does not eliminate this

Community security research is possible for register-programming drivers (the kernel driver is open and attackable by standard audit means) and partially possible for firmware-assisted drivers where the protocol is documented. For firmware-as-OS GPUs, security research on the firmware itself requires binary analysis expertise and specialised tooling.

---

## 9. Open-Source Driver Feasibility at the Firmware Layer

The fundamental question for each vendor: can a correct, complete open-source GPU driver be written without access to the firmware source?

### What "Without Firmware Source" Means

A community driver may need:
1. **Firmware binary** (to boot the GPU) — this is nearly universal; all vendors provide binaries in `linux-firmware`
2. **Firmware protocol documentation** (to exchange messages correctly) — the differentiating factor
3. **Hardware register documentation** (for the portions the driver programs directly) — relevant where the driver still does register programming

The question of feasibility turns on (2): does the vendor document the firmware protocol well enough that a driver can be written without reverse engineering?

### NVIDIA: Demonstrated Feasible

Nova and NVK demonstrate that a complete open-source NVIDIA driver is achievable. The key enabler is `open-gpu-doc`: NVIDIA publishes C header files describing the RPC message structures and class method encodings that the CPU driver uses to communicate with GSP-RM. [Source](https://github.com/NVIDIA/open-gpu-doc)

The firmware binary is required; its source is not. The CPU-side protocol is documented (or can be inferred from the open-gpu-kernel-modules source); the hardware register access is GSP-mediated and not needed directly. Nova's explicit architectural premise is that GSP-RM is a well-defined hardware abstraction layer: the driver writes to a firmware API, not to hardware.

The remaining gap: some GFX command encoding (class methods for specific engine operations) is not fully documented, requiring inference from the open nvidia-open module source. NVK has addressed this incrementally as Vulkan conformance testing exercises more of the command space.

### AMD: Demonstrably Feasible (and Deployed)

amdgpu is a fully open-source driver for all AMD GPU generations from GCN forward, co-developed by AMD. The PSP and SMU firmware are binary-only, but the driver communicates with them through documented interfaces (`amdgpu_psp.c`, `amdgpu_smu.c`). The hardware register interfaces for the GFX engine are documented in AMD's GPUOpen and `amd/display` hardware specs. [Source](https://gpuopen.com/amd-gpu-architecture-programming-documentation/)

AMD's decision to document hardware registers while keeping security/power firmware binary-only is the template for a sustainable open driver model: the community can audit, debug, and extend the driver's core logic without firmware source.

### Intel: Demonstrably Feasible (and Deployed)

xe.ko and i915.ko are fully open. The GuC CT protocol is documented within the kernel source (`drivers/gpu/drm/xe/xe_guc_ct.c`, `xe_guc_submit.c`). HuC's role is narrow and its interface simple (authenticated by GuC, then receives video commands through hardware registers). PXP is a niche feature for premium content and does not affect general GPU operation. [Source](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/xe/)

### Qualcomm: Demonstrably Feasible

freedreno and Turnip are fully open with near-parity performance. The zap shader is a small binary with a simple interface. Qualcomm's documentation of Adreno 6xx/7xx register interfaces enabled this parity. [Source](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/msm/)

### ARM Mali CSF: Harder, Not Impossible

The Panthor driver is open source and functional, but it required ARM cooperation for key protocol details. The CSF firmware protocol is not publicly documented as a standalone specification. The Panthor team worked with ARM to understand the command stream group interface, queue scheduling semantics, and tiler heap protocol.

The risk for future Mali CSF generations: if ARM expands the CSF firmware's scope (more hardware management moved into the Cortex-M7 firmware) without updating its community engagement, Panthor's architecture will require deeper reverse-engineering effort to track. The Tyr Rust re-implementation of Panthor inherits the same protocol constraints.

### Apple AGX: Extremely Hard, Achieved Anyway

The Asahi team's accomplishment is that a functional open-source driver *does* exist for Apple AGX — despite full firmware opacity, no hardware documentation, and active write-protection on the firmware. The cost is years of reverse-engineering work by a small team of experts, incomplete feature coverage (ray tracing on M3/M4 is not yet supported), and permanent maintenance burden as Apple updates its firmware with each macOS release.

The conclusion from the Asahi case is not that firmware-as-OS is insurmountable for community drivers — it is that it raises the barrier to entry by roughly two orders of magnitude relative to a documented hardware register interface.

### Summary: Feasibility as a Function of Protocol Openness

```
Driver Feasibility vs. Firmware Transparency

High feasibility  │ amdgpu     Xe/i915     freedreno     Nova/NVK
                  │    (open HW)  (CT documented) (registers)  (open-gpu-doc)
                  │
Medium feasibility│                    Panthor (partial ARM cooperation)
                  │
Low feasibility   │                              Asahi AGX (full RE, no docs)
                  │
                  └─────────────────────────────────────────────────────────
                   Register-programming   Firmware-assisted   Firmware-as-OS
```

The pattern is clear: feasibility tracks with protocol openness, not with firmware scope per se. A firmware-as-OS GPU with a well-documented RPC protocol (NVIDIA with open-gpu-doc) is more tractable than a firmware-assisted GPU whose firmware protocol is completely opaque.

---

## 10. The Convergence Trajectory

### Evidence for Convergence Toward Firmware-as-OS

Several data points suggest the industry is converging toward NVIDIA's model:

**AMD MES expansion**: MES is present and active in RDNA 2/3 and takes on more queue scheduling responsibility with each generation. AMD's RDNA 3 ROCm compute frustrations (Tiny Corp's public criticism in 2024) centered in part on MES scheduling issues, suggesting MES has become a critical execution path that was not fully hardened. [Source](https://videocardz.com/newz/amd-releases-micro-engine-scheduler-mes-firmware-documentation-for-rdna3-gpus)

**Intel GSC expanding**: The GSC started as HDCP-focused and has grown to encompass PXP, HuC attestation, and GPU firmware update management. The `drm/xe` PXP HWDRM support (submitted November 2024) extends GSC's role further into the runtime execution path.

**ARM Mali CSF scope**: The CSF model's Cortex-M7 scheduler is the first step. ARM has not published a roadmap for CSF evolution, but the architectural trajectory — moving scheduling policy from hardware job slots into firmware — matches the industry trend.

**RISC-V as firmware substrate**: The adoption of RISC-V for the NVIDIA GSP on Hopper/Blackwell, and its appearance as the management core in Imagination BXE (the `drm/imagination` driver target), suggests the industry views RISC-V as a good platform for embedded GPU management processors. RISC-V's open ISA makes the management processor design more flexible, enabling richer firmware OS capabilities without hardware ISA licensing concerns.

### Evidence Against Convergence

**Qualcomm stays minimal**: Despite expanding into datacenter AI accelerators (Qualcomm Cloud AI 100, handled by `accel/qaic`), Qualcomm's mobile/laptop Adreno GPU family continues its register-programming model with community-documented protocols. The Snapdragon X Elite's Adreno X1-85 is driven by the same MSM/freedreno infrastructure as Snapdragon 865 phones.

**RISC-V GPU projects stay open**: The RISC-V community's long-horizon goal is a fully open-source GPU, including firmware. The Vortex GPGPU research project targets exactly this. If community RISC-V GPUs reach sufficient capability, they will be entirely register-programming architecture by design.

**Vendor incentive structure**: The firmware-as-OS model has costs: firmware bugs block the entire driver, firmware updates require distribution packaging, and firmware complexity grows maintenance burden. AMD's mixed model (firmware for security/power, open code for GFX) manages these costs by limiting firmware scope.

### The Deeper Driver

The primary economic driver for firmware-as-OS is not driver simplicity — it is *virtualisation* and *confidential computing*. NVIDIA's vGPU product (and CUDA CC mode) directly requires firmware-enforced resource partitioning. AMD's PSP is required for SEV-SNP confidential VMs. Intel's GuC enables preemptive multi-tenant scheduling; GSC enables PXP content DRM.

As GPU virtualisation and AI workload isolation become more commercially important (cloud GPU rentals, confidential AI inference), vendor incentives to push more policy into firmware increase. The firmware is the enforcement point that cannot be bypassed by a compromised kernel. For cloud vendors, this is a strong requirement.

The risk for the open-source ecosystem is that the commercial imperative for firmware-enforced isolation drives firmware scope expansion beyond what is needed for security, into areas (like general resource management) where community debugging access matters. NVIDIA's current model — open CPU-side protocol, documented headers, binary firmware — shows that these goals need not conflict. But it required deliberate vendor investment (open-gpu-doc, open-gpu-kernel-modules) to achieve.

---

## 11. Strategic Outlook

### For Kernel Driver Developers

The architectural shift from hardware programmer to firmware client changes what kernel driver code *does*. In a register-programming driver, the interesting code is hardware state machine management, register sequencing, and fence/synchronisation logic tied to hardware events. In a firmware-client driver, the interesting code is protocol correctness, message queue management, shared-memory layout, and firmware version compatibility handling.

This shift has practical skill implications: firmware-client drivers require understanding the firmware's state machine (often only inferable from the firmware's observable behaviour) alongside the hardware's state machine. Debugging tools must be firmware-aware: `umr` (AMD GPU debugger) and `intel_gpu_top` have been extended to query firmware state, but no cross-vendor equivalent exists.

The introduction of Rust for new firmware-client drivers (Nova, Asahi, Tyr) reflects this: Rust's type system can enforce protocol state transitions at compile time, reducing the risk of malformed firmware messages that silently corrupt GPU state.

### For Security Researchers

The firmware layer is the new GPU attack surface. Traditional GPU security research focused on the kernel driver (privilege escalation via ioctl bugs) and the compiler (shader JIT attacks). With firmware-as-OS GPUs, two additional attack surfaces exist:

1. **Firmware binary vulnerabilities**: Buffer overflows, integer errors, and logic bugs in the firmware itself. These are reachable via malformed RPC payloads from a compromised kernel driver. A firmware vulnerability with DMA access to system RAM is a potential kernel bypass.

2. **Host-firmware interface attacks**: The message queue protocol between CPU driver and firmware can be attacked from both sides. A malicious userspace process that compromises the kernel driver (or a kernel driver with a vulnerability) can send malformed messages to the firmware.

The community's ability to audit these surfaces is limited: firmware source is generally unavailable, and dynamic analysis requires specialised tools (RISC-V debugging via JTAG, Falcon disassemblers in Envytools). The `open-gpu-kernel-modules` publication helps for NVIDIA — the CPU-side code is auditable — but the firmware binary remains opaque.

Distribution security teams should factor in the firmware version as a security maintenance concern: `linux-firmware` updates for GSP-RM, GuC, and AMD PSP may include security fixes that require user-facing firmware-update prompts, not just kernel updates.

### For Distribution Maintainers

GPU firmware blobs are a practical packaging challenge. Key policy points:

- **Debian**: non-free-firmware section (`firmware-nvidia-gsp`, `firmware-amd-graphics`, `intel-media-va-driver`). The bookworm release (2023) created non-free-firmware as a distinct component, acknowledging that modern hardware is increasingly non-functional without these blobs. [Source](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)
- **Fedora**: Ships most GPU firmware in `linux-firmware` (enabled by default) with the view that binary firmware for community-supported open drivers is a practical necessity, not a free software violation.
- **Arch Linux**: Ships `linux-firmware` with broad coverage; users must explicitly opt into proprietary NVIDIA userspace.
- **Trisquel / Parabola**: Exclude non-free firmware entirely, meaning AMD PSP firmware, Intel GuC, NVIDIA GSP-RM blobs, and Mali CSFFW are absent — resulting in broken or severely degraded GPU functionality on all modern discrete GPUs.

The distribution policy question sharpens as firmware scope grows: excluding GPU firmware blobs now means not just missing optional features, but broken GPU initialisation (AMD, Intel), no GPU rendering at all (NVIDIA Turing+), or degraded power management. Distributions committed to free-software-only principles face increasing hardware support gaps as firmware-as-OS becomes the default.

### For the Open-Source GPU Ecosystem

The risk to the open-source GPU ecosystem is not that vendors will refuse to ship binary firmware — they already do, and the community has adapted. The risk is that *the documented protocol boundary* that enables community drivers shrinks to a thin RPC stub with no stable specification.

The positive scenario — demonstrated by NVIDIA's open-gpu-doc and AMD's GPUOpen — is that vendors maintain stable, documented CPU-firmware interfaces even as they move more hardware management into firmware. A well-documented RPC protocol is an engineering contract that benefits both the vendor (stable CPU-driver API surface) and the community (a clear target for open driver implementation).

The negative scenario is what Apple and (partially) ARM Mali represent: firmware that is opaque, whose protocol changes with each hardware generation, with no published specification, requiring community reverse-engineering to track. In this scenario, open-source driver maintenance becomes a continuous archaeological effort rather than principled engineering.

The architectural trajectory of AMD's MES, Intel's GSC, and ARM's CSF all require monitoring. If AMD documents MES scheduling semantics to the same standard as PSP/SMU interfaces in `amdgpu_smu.c`, the community driver remains viable. If MES becomes an undocumented black box that the driver must reverse-engineer, amdgpu's open character becomes constrained.

The NVIDIA case offers an existence proof: structured vendor community engagement — open CPU-side source, header-documented protocols, firmware upstreaming to `linux-firmware` — can sustain an open driver ecosystem even with firmware opacity. The Nova/NVK stack demonstrates that a new Rust driver, written without firmware source and with access only to documented headers, can achieve Vulkan 1.4 conformance and near-native performance.

The sustainability of open GPU drivers at the firmware-as-OS frontier depends on vendors choosing to publish protocol documentation alongside firmware binaries, treating the CPU-firmware interface as a public API rather than an implementation detail.

---

## 12. Integrations

- **Ch. 5 (x86 GPU Drivers)**: amdgpu's IP block chain, PSP authentication sequence, and GuC CT submission are the deployment context for the firmware architectures described here. The `amdgpu_device_ip_init()` path and `xe_guc_ct.c` are starting points for hands-on exploration.

- **Ch. 9 (GSP-RM and the nvidia-open Connection)**: Deep dive into the NVIDIA GSP-RM implementation — RPC message queue layout, `nvkm_gsp` subdevice hierarchy, boot sequence via `tu102_gsp_booter_load()`, and the Nova Rust driver architecture. This chapter's Section 2 deliberately summarises rather than duplicates that material.

- **Ch. 73 (Asahi Linux and Apple Silicon AGX)**: The adversarial firmware reverse-engineering case. The `drm/asahi` Rust driver, RTKit firmware protocol, UAT GPU MMU, and DCP display coprocessor are covered there; Section 6 here places them in the cross-vendor spectrum.

- **Ch. 90 (Open ARM GPU Drivers: Lima, Panfrost, Panthor)**: The ARM Mali CSF model and Panthor's `DRM_IOCTL_PANTHOR_GROUP_SUBMIT` interface are covered in depth there. This chapter's classification of CSF as "approaching firmware-as-OS for scheduling" is the architectural summary; Ch. 90 provides the implementation detail.

- **Ch. 129 (GPU Firmware Deep Dive)**: The technical mechanics this chapter deliberately does not cover: `request_firmware` API, specific blob file naming (`psp_*_sos.bin`, `gfx*_mec.bin`, `guc_*_guc.bin`), AMD PSP authentication ring, Intel GuC boot sequence, and the `linux-firmware` packaging structure.

- **Ch. 160 (freedreno and Turnip: Qualcomm Adreno Open Drivers)**: The zap shader firmware, ring buffer construction, and Adreno CP microcode are detailed there. Section 5 here contextualises Adreno as the register-programming anchor of the vendor spectrum.

---

## 13. References

- [NVIDIA open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules)
- [NVIDIA open-gpu-doc (RPC header documentation)](https://github.com/NVIDIA/open-gpu-doc)
- [Nova driver documentation — Linux kernel 6.16](https://www.kernel.org/doc/html/v6.16-rc6/gpu/nova/index.html)
- [AMD MES kernel documentation](https://docs.kernel.org/gpu/amdgpu/gc/mes.html)
- [AMD MES firmware documentation release — Phoronix](https://www.phoronix.com/news/AMD-MES-Firmware-Docs)
- [AMD MES specification (GPUOpen, April 2024)](https://gpuopen.com/amd-gpu-architecture-programming-documentation/)
- [AMD Platform Security Processor — Wikipedia](https://en.wikipedia.org/wiki/AMD_Platform_Security_Processor)
- [AMD Secure Processor reverse engineering — Dayzerosec](https://dayzerosec.com/blog/2023/04/17/reversing-the-amd-secure-processor-psp.html)
- [Intel GSC — Intel EDC documentation](https://edc.intel.com/content/www/us/en/design/products/platforms/details/arrow-lake-s/core-ultra-200s-series-processors-datasheet-volume-1-of-2/intel-graphics-system-controller-intel-gsc/)
- [Intel GSC (igsc library)](https://github.com/intel/igsc)
- [Intel PXP Protected Xe Path — Phoronix](https://www.phoronix.com/news/Intel-PXP-Protected-Xe-Path)
- [Intel GPU security — Igor's Lab (2021)](https://igor-blue.github.io/2021/02/10/graphics-part1.html)
- [Intel GuC CT interface — drm-tip](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/xe/xe_guc_ct.c)
- [Panthor CSF driver — LWN](https://lwn.net/Articles/953784/)
- [Collabora Panthor / PanCSF announcement](https://www.collabora.com/news-and-blog/news-and-events/pancsf-a-new-drm-driver-for-mali-csf-based-gpus.html)
- [Arm Mali Gen10 firmware in linux-firmware — Phoronix](https://www.phoronix.com/news/Arm-Mali-Panthor-Firmware-Git)
- [Apple AGX GPU architecture — Asahi Linux docs](https://asahilinux.org/docs/hw/soc/agx/)
- [Apple GPU (G13) architecture tools — dougallj/applegpu](https://github.com/dougallj/applegpu)
- [Adreno A6xx zap shader — Phoronix](https://www.phoronix.com/news/MSM-A6xx-Zap-Shader)
- [AMD DMCUB support — Phoronix](https://phoronix.com/news/Renoir-DMCUB-AMDGPU-Patches)
- [V3D driver source — drm-tip](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/v3d/)
- [amdgpu IP block device init — drm-tip](https://cgit.freedesktop.org/drm/drm-tip/tree/drivers/gpu/drm/amd/amdgpu/amdgpu_device.c)
- [linux-firmware repository](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)
- [AMD GPU architecture programming documentation — GPUOpen](https://gpuopen.com/amd-gpu-architecture-programming-documentation/)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
