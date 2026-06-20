# Chapter 129: GPU Firmware Deep Dive

This chapter targets kernel GPU driver developers, system integrators, and security engineers who need to understand how modern GPUs load, authenticate, and depend on proprietary firmware blobs — and who must maintain firmware update infrastructure for production systems. Browser and application developers may find the security and debugging sections useful when diagnosing GPU initialisation failures.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The `request_firmware` API](#the-request_firmware-api)
3. [AMD GPU Firmware Ecosystem](#amd-gpu-firmware-ecosystem)
4. [Intel GPU Firmware: GuC, HuC, DMC](#intel-gpu-firmware-guc-huc-dmc)
5. [NVIDIA GPU Firmware: GSP-RM and Falcon](#nvidia-gpu-firmware-gsp-rm-and-falcon)
6. [linux-firmware Package and Versioning](#linux-firmware-package-and-versioning)
7. [Firmware Authentication and Security](#firmware-authentication-and-security)
8. [fwupd and GPU Firmware Updates](#fwupd-and-gpu-firmware-updates)
9. [Debugging Firmware Loading Failures](#debugging-firmware-loading-failures)
10. [Integrations](#integrations)

---

## Introduction

Modern discrete and integrated GPUs are not mere silicon state machines driven entirely by CPU-side register writes. Each vendor embeds several independent microprocessors inside the GPU die itself. These processors run proprietary firmware blobs that are loaded by the kernel driver at device initialisation time. The firmware handles tasks that are either too latency-sensitive or too security-critical to delegate to the host CPU:

- **Command scheduling** — translating kernel ring-buffer commands into GPU micro-operations and multiplexing multiple application work queues across physical hardware queues.
- **Power management** — controlling voltage and frequency scaling, entering and exiting deep power states, and implementing thermal throttling with microsecond granularity.
- **Video decode acceleration** — running codec-specific state machines (H.264, H.265/HEVC, AV1) inside dedicated video processor cores that are governed by separate microcontrollers.
- **Display link management** — controlling DisplayPort AUX channel transactions, DPCD register access, multi-stream transport (MST) topology discovery, and link training during monitor hotplug.
- **Resource management** — on NVIDIA Turing and later hardware, the entire GPU resource manager (channel allocation, memory management, fault handling) has been migrated from the CPU-side driver into firmware running on a dedicated RISC-V processor embedded in the GPU.

The kernel driver cannot initialise the GPU hardware without these blobs. They are not built into the kernel image (with rare exceptions) but are instead distributed separately via the `linux-firmware` repository at [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git). On a running system they reside under `/lib/firmware/`. This separation exists for licence reasons: hardware vendors retain proprietary rights over the firmware binary and permit binary-only redistribution. Source code is generally not released.

This chapter covers the full lifecycle of GPU firmware: why it exists, how each major vendor structures its firmware ecosystem, how the kernel discovers and loads blobs at runtime, how updates are distributed and applied, and what security properties (and vulnerabilities) the firmware subsystem carries.

---

## The `request_firmware` API

The Linux kernel provides a generic firmware loading infrastructure declared in `include/linux/firmware.h`. All GPU drivers use this infrastructure rather than open-coding their own file I/O, because the generic layer handles the firmware search path, initramfs integration, caching, and optional userspace fallback uniformly.

### Core Data Structure

```c
/* include/linux/firmware.h */
struct firmware {
    size_t size;
    const u8 *data;
    /* private fields */
};
```

After a successful firmware request, the driver reads the blob bytes from `fw->data` (length `fw->size`), uploads them to the GPU, and then calls `release_firmware(fw)` to free the kernel mapping. The blob is never written back; the driver is responsible for keeping its own copy in GPU-accessible memory if the firmware needs to survive after release.

### Synchronous Variants

**`request_firmware()`** is the standard blocking call:

```c
/* include/linux/firmware.h */
int request_firmware(const struct firmware **fw, const char *name,
                     struct device *dev);
```

The call sleeps until the firmware is found (or fails). It must be called from a context that can sleep — process context, or at most in a work queue. For most GPU driver probe paths this is fine because `probe()` itself is called from process context.
[Source](https://docs.kernel.org/driver-api/firmware/request_firmware.html)

**`request_firmware_nowarn()`** suppresses the kernel log warning emitted when the firmware file is absent. Use this for optional firmware whose absence should be handled silently:

```c
int firmware_request_nowarn(const struct firmware **fw, const char *name,
                             struct device *dev);
```

**`request_firmware_direct()`** bypasses the usermode helper fallback and reads only from the filesystem. This is faster and appropriate when the rootfs is guaranteed to be mounted:

```c
int request_firmware_direct(const struct firmware **fw, const char *name,
                             struct device *dev);
```

### Asynchronous Variant

When the driver needs to avoid blocking probe:

```c
int request_firmware_nowait(struct module *module, bool uevent,
                             const char *name, struct device *dev,
                             gfp_t gfp, void *context,
                             void (*cont)(const struct firmware *fw,
                                          void *context));
```

The callback `cont` is invoked from a kernel work queue once the firmware is available or has failed. The driver must not assume the GPU is ready until the callback fires.

### Firmware Search Path

The kernel searches the following locations in order, stopping at the first match:
[Source](https://docs.kernel.org/driver-api/firmware/request_firmware.html)

1. `/lib/firmware/$(uname -r)/` — version-specific overrides; useful for testing a new blob without affecting other kernels installed on the same system.
2. `/lib/firmware/` — the system-wide firmware directory populated by the `linux-firmware` package.
3. The built-in initramfs — blobs embedded directly in the kernel image via `CONFIG_EXTRA_FIRMWARE` / `CONFIG_FIRMWARE_IN_KERNEL`. GPU blobs are almost never embedded this way due to their size (each AMD generation may ship 50+ MB of firmware across all firmware components).

The `CONFIG_EXTRA_FIRMWARE` Kconfig option (used in `make menuconfig → Device Drivers → Generic Driver Options → External firmware blobs`) allows specific files to be compiled into the kernel. For a system with a fixed GPU model (such as an embedded device), this eliminates the initramfs firmware dependency at the cost of kernel image size.

The firmware infrastructure also supports a userspace fallback path: if `CONFIG_FW_LOADER_USER_HELPER` is enabled and the firmware file is not found in the standard paths, the kernel creates a sysfs entry and waits for userspace (typically udev) to supply the data. This mechanism is largely unused for GPU firmware and disabled by default on most modern distributions.

### initramfs and Early Boot

Several GPU drivers initialise the hardware before the rootfs is mounted — for example when the GPU drives the primary display and Plymouth (the boot splash) needs a frame before `/` is available. In these cases the firmware must be present in the initramfs:

```bash
# Debian/Ubuntu — regenerate initramfs after linux-firmware update
update-initramfs -u -k all

# Fedora/RHEL — dracut must be told to include amdgpu firmware
dracut --add-drivers amdgpu --regenerate-all -f
```

The `dracut` `amdgpu` module (or `i915`, `nouveau`) calls `modinfo $driver | grep ^firmware:` to enumerate required blobs and copies them into the initrd image. The `firmware_request_cache()` API allows drivers to cache firmware during normal operation so it remains available across suspend/resume cycles without requiring the filesystem to be re-read:

```c
/* Cache firmware so it survives suspend/resume (filesystem may be frozen) */
int firmware_request_cache(struct device *device, const char *name);
```

This is particularly relevant for display drivers that may need to re-upload DMC firmware (Intel) or DMCUB firmware (AMD) after returning from S3 (suspend to RAM).
[Source](https://wiki.gentoo.org/wiki/Linux_firmware)

### Release

```c
void release_firmware(const struct firmware *fw);
```

Call this after the blob has been uploaded to the GPU. Holding the `struct firmware` beyond that point wastes kernel memory; the GPU firmware running on its embedded microcontroller does not require the host copy to remain valid.

---

## AMD GPU Firmware Ecosystem

AMD's AMDGPU driver (`drivers/gpu/drm/amd/amdgpu/`) loads the largest collection of discrete firmware blobs of any vendor's open-source driver. Each GPU generation (RDNA 2, RDNA 3, CDNA 2/3) ships a distinct set of blobs stored under `/lib/firmware/amdgpu/`.

### Firmware Component Overview

| Component | Example filename | Responsible hardware block |
|-----------|-----------------|---------------------------|
| GFX ME | `gfx10_me.bin` | Graphics ring Micro Engine |
| GFX MEC | `gfx11_mec.bin` | Compute Micro Engine |
| GFX RLC | `gfx11_rlc.bin` | Register Load Controller |
| VCN | `vcn_4_0_0.bin` | Video Core Next (encode/decode) |
| DMCUB | `dcn_3_1_4_dmcub.bin` | Display Micro-Controller Unit B |
| SMU | `smu_13_0_0.bin` | System Management Unit |
| PSP SOS | `psp_13_0_0_sos.bin` | PSP Secure OS |
| PSP ASD | `psp_13_0_0_asd.bin` | AMD Security Diagnostic |
| MES | `mes_v11_0_api.bin` | Micro Engine Scheduler API |
| MES KIQ | `mes_v11_0_kiq.bin` | MES Kernel Interface Queue |

[Source](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/amdgpu)

### PSP: The Root of Trust

The Platform Security Processor (PSP) is an ARM Cortex-A subsystem embedded alongside the GPU compute fabric on AMD APUs and discrete GPUs. It is the most critical firmware component because all other firmware blobs must be authenticated by the PSP before they may execute.

The PSP initialisation sequence in `drivers/gpu/drm/amd/amdgpu/amdgpu_psp.c` proceeds as follows:

1. The driver allocates a ring buffer in GPU-accessible VRAM for PSP command submission:

```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_psp.c */
static int psp_ring_init(struct psp_context *psp,
                          enum psp_ring_type ring_type)
{
    /* Allocate one 4 KB page for the ring */
    ring->ring_size = 0x1000;
    /* ... amdgpu_bo_create_reserved for ring BO ... */
}
```

2. The Secure OS (`sos.bin`) is loaded first. It is the PSP's own operating system and the entity that validates all subsequent blobs:

```c
static int psp_load_fw(struct amdgpu_device *adev)
{
    ret = psp_sos_load(psp);        /* load Secure OS */
    ret = psp_asd_load(psp);        /* AMD Security Diagnostic TA */
    ret = psp_rl_load(psp);         /* Ring List for XGMI/DTLS */
    ret = psp_rap_load(psp);        /* Runtime Anti-rollback Protection */
}
```

3. Each subsequent firmware blob is submitted to the PSP via `psp_cmd_submit_buf()`, which writes a `psp_gfx_cmd_resp` structure into the ring, rings the PSP doorbell, and polls for the response:

```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_psp.c */
int psp_cmd_submit_buf(struct psp_context *psp,
                        struct amdgpu_firmware_info *ucode,
                        struct psp_gfx_cmd_resp *cmd,
                        uint64_t fence_mc_addr)
{
    /* Write cmd into ring, ring doorbell, poll fence */
    /* PSP validates RSA-4096 signature on firmware blob */
    /* Returns -EPERM if signature invalid */
}
```

If the PSP rejects a blob (invalid signature, rolled-back version), the GPU will not initialise. This is by design: AMD uses the PSP to enforce a hardware-enforced anti-rollback policy on discrete GPU firmware.
[Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_psp.c)

### GFX Firmware: ME, MEC, RLC

The Graphics MicroEngine (ME) firmware runs the graphics ring command processor. On RDNA 3 (gfx11) it is split into:
- `me.bin` — the ME itself, handling 3D pipeline state packets on the GFX ring.
- `mec.bin` / `mec2.bin` — MEC (Micro Engine Compute), handling HSA compute queues (GFX10 onwards uses a unified MEC, GFX9 had two).
- `rlc.bin` — Register Load Controller, managing Compute Unit (CU) context save/restore for mid-thread preemption.
- `pfp.bin` — Pre-Fetch Parser, the front-end command stream parser that feeds ME.

The firmware upload sequence is gated on PSP availability. The generic upload helper:

```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_ucode.c */
void amdgpu_ucode_init_single_fw(struct amdgpu_device *adev,
                                   struct amdgpu_firmware_info *ucode,
                                   uint64_t mc_addr, void *kptr)
{
    /* Store blob's GPU MC address and CPU virtual address */
    ucode->mc_addr = mc_addr;
    ucode->kaddr   = kptr;
    /* ucode->fw already set by request_firmware earlier */
}
```

After `amdgpu_ucode_init_single_fw()` records the blob locations, the PSP loading path submits each blob to the PSP for authentication and upload to the appropriate internal SRAM on the command processor. The firmware must be allocated in a reserved VRAM BO (`amdgpu_bo_create_reserved`) accessible to both the CPU (for the initial copy) and the PSP DMA engine.

The RLC firmware is particularly important for compute workloads because it manages CU context save across preemption boundaries. Without a functioning RLC, mid-thread preemption is unavailable and long-running compute shaders cannot be preempted, degrading scheduling latency for interactive 3D workloads sharing the GPU.

These are initialised via `amdgpu_gfx_rlc_init()` and uploaded to the GPU via the PSP ring after the PSP SOS is running. The GFX firmware must be present in VRAM before any 3D or compute command submission.

### VCN Firmware

The Video Core Next (VCN) is AMD's unified encode/decode block, replacing the earlier UVD+VCE split. VCN firmware (`vcn_4_0_0.bin` for VCN 4.0 on RDNA 3) is loaded by `amdgpu_vcn_sw_init()`:

```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_vcn.c */
int amdgpu_vcn_sw_init(struct amdgpu_device *adev)
{
    r = request_firmware(&adev->vcn.fw,
                          adev->vcn.fw_name, adev->dev);
    /* Upload via PSP once PSP is running */
}
```

Without VCN firmware, hardware video decode acceleration (VA-API, V4L2) is unavailable and the codec falls back to software. See Ch26 for the full VA-API stack.

### DMCUB: Display Micro-Controller Unit B

The DMCUB is a Tensilica HiFi4 DSP embedded in AMD's DCN (Display Core Next) display engine. Its firmware (`dcn_3_1_4_dmcub.bin`) handles:
- DisplayPort AUX channel state machine (hotplug detection, EDID reads).
- DPCD register access for panel power sequencing.
- Multi-stream Transport (MST) hub management.
- Panel replay / Panel Self Refresh power state management.

The display driver initialises DMCUB in `dm_dmub_sw_init()` (within `amdgpu_dm.c`). If DMCUB firmware is absent, DisplayPort output may fail entirely on DCN 3.x hardware.

### SMU: System Management Unit

The System Management Unit firmware (`smu_13_0_0.bin`) runs on a dedicated Cortex-M microcontroller integrated on AMD SoCs and discrete GPUs. SMU firmware responsibilities include:
- Dynamic voltage and frequency scaling (DVFS) for the GPU core, memory, and display clocks.
- Thermal limit enforcement and emergency shutdown.
- Power cap enforcement for total board power (TBP).
- Communication of PPTable (power play table) settings from the driver.

The kernel driver communicates with SMU firmware via a set of SMU message registers. The `amdgpu_smu.c` layer abstracts per-generation differences. If SMU firmware is absent the GPU cannot regulate its own power consumption and may run at fixed (typically conservative) clocks.

### MES: Micro Engine Scheduler

The MicroEngine Scheduler (MES) is a newer addition to AMD's firmware stack, first appearing on GFX10 (RDNA 2) and documented publicly by AMD in April 2024 [Source](https://gpuopen.com/download/documentation/micro_engine_scheduler.pdf). MES runs on the GFX Command Processor and handles:
- Dynamic mapping of software queues (MQDs — Memory Queue Descriptors) to physical hardware queues (HQDs — Hardware Queue Descriptors).
- Queue preemption and priority management.
- Support for AMD's Hardware Scheduler model used by ROCm and the amdgpu HWS path.

The kernel driver submits commands to MES via a dedicated ring. Two MES images are required:
- `mes_v11_0_api.bin` — the MES API image (the scheduler logic).
- `mes_v11_0_kiq.bin` — the Kernel Interface Queue image (handles privileged kernel-side queue operations).

```bash
# Verify MES firmware is loaded on RDNA3 hardware
cat /sys/kernel/debug/dri/0/amdgpu_firmware_info | grep -i mes
```

---

## Intel GPU Firmware: GuC, HuC, DMC

Intel's i915 driver (Gen9–Gen12) and its successor, the Xe driver (Xe1/Xe2 era), both load multiple firmware blobs into the GPU at initialisation time. The blobs are stored under `/lib/firmware/i915/` for legacy hardware and `/lib/firmware/xe/` for Xe-class hardware.
[Source](https://docs.kernel.org/gpu/xe/xe_firmware.html)

### GuC: Graphics Microcontroller

The GuC (Graphics Microcontroller) is a small x86-compatible embedded core (later Arm-based on some platforms) inside the GPU's GT (Graphics Tile). Starting with Tiger Lake (Gen12+), GuC submission has replaced Execlist submission as the primary command submission path. GuC handles:

- **Command submission**: translating kernel `i915_request` objects into GPU work units and scheduling them across GPU engines.
- **Power gating**: coordinating GT power state transitions (RC6 render C-states).
- **GT power management** (via SLPC — Single Loop Power Conservation on Xe): frequency selection and power limit enforcement.

Firmware files follow the naming pattern `{platform}_{version}_guc.bin`:

```
/lib/firmware/i915/tgl_guc_70.bin       # Tiger Lake, GuC 70.x
/lib/firmware/i915/adlp_guc_70.bin      # Alder Lake-P
/lib/firmware/xe/lnl_guc_70.bin         # Lunar Lake (Xe2)
```

Initialisation path in i915:

```c
/* drivers/gpu/drm/i915/gt/uc/intel_guc_fw.c */
int intel_guc_fw_upload(struct intel_guc *guc)
{
    ret = guc_prepare_xfer(guc);     /* configure HW registers */
    ret = guc_xfer_rsa(guc);         /* transfer RSA signature */
    /* DMA transfer of microcode to GuC IMEM at 0x2000 */
    ret = guc_wait_ucode(guc);       /* poll status register */
}
```

The module parameter `i915.enable_guc=3` controls GuC behaviour:
- Bit 0 (`=1`): enable GuC submission (replaces execlist).
- Bit 1 (`=2`): enable HuC authentication via GuC.
- `=3`: both enabled (default on Gen12+).

On systems where GuC initialisation fails, the driver falls back to Execlist submission with a warning:
```
[drm] GuC firmware i915/tgl_guc_70.bin: load failed with error -110
[drm] GT0: GuC submission disabled
```

GuC status can be inspected at runtime:
```bash
cat /sys/kernel/debug/dri/0/i915_guc_info
```

### HuC: HEVC/H.264 Microcontroller

The HuC (HEVC Microcontroller) firmware handles video codec authentication — specifically, it allows the media driver to request hardware-enforced content protection for H.264 and H.265 decode paths. On Tiger Lake and later, HuC authentication is **required** for H.265 decode acceleration; without it the media driver falls back to the software path.

A critical aspect of HuC is that it must be **authenticated by GuC**, not directly by the driver. The firmware chain is:

1. Driver loads HuC firmware into GPU memory.
2. Driver loads and initialises GuC.
3. Driver calls `intel_huc_auth()`, which sends a GuC command to validate the HuC RSA signature.
4. Only after GuC confirms HuC authenticity does HuC become active.

```c
/* drivers/gpu/drm/i915/gt/uc/intel_huc.c */
int intel_huc_auth(struct intel_huc *huc,
                    enum intel_huc_authentication_type type)
{
    /* type = INTEL_HUC_AUTH_BY_GUC for Gen12+ */
    return intel_guc_auth_huc(gt->uc.guc,
                               intel_huc_ggtt_offset(huc));
}
```

On DG2 and later (Xe era), HuC firmware uses the GSC-based CPD (Code Partition Directory) layout rather than the older CSS binary format. The Xe driver's `xe_huc.c` handles both formats.

### DMC: Display Microcontroller

The Display Microcontroller (DMC) firmware controls display power gating during display idle periods. Without DMC, Intel GPUs cannot enter the DC5/DC6 (Display C-state) deep power states, resulting in significantly higher idle power consumption.

DMC firmware is loaded by `intel_dmc_init()` (i915) or `xe_dmc_init()` (Xe):

```c
/* drivers/gpu/drm/i915/display/intel_dmc.c */
void intel_dmc_init(struct drm_i915_private *i915)
{
    /* e.g.: "i915/adlp_dmc.bin" for Alder Lake-P */
    err = request_firmware(&fw, i915->dmc.fw_path, i915->drm.dev);
    intel_dmc_load_program(i915, fw);
}
```

DMC manages:
- Saving and restoring display engine registers across DC state transitions.
- Maintaining display output while the GT (render engine) is fully powered off.
- Waking the display pipe rapidly when the user moves the cursor.

DMC failure does not prevent the display from working at all, but power consumption in idle scenarios increases substantially (often by 1–3 W on laptop platforms).

### GSC: Graphics Security Controller (Xe)

On Xe-era hardware (DG2/Alchemist, Meteor Lake, Lunar Lake), Intel introduced the GSC (Graphics Security Controller), which subsumes the HuC authentication role and adds:

- Ownership verification for protected content (replacing HuC-based content protection).
- Attestation services for trusted execution environments.
- Interaction with Intel Platform Trust Technology (PTT).

GSC firmware: `/lib/firmware/xe/lnl_gsc.bin`

The GSC uses the CPD (Code Partition Directory) binary layout with a top-level directory header pointing to multiple partitions (manifest, metadata, firmware code). The Xe driver's `xe_gsc.c` parses the CPD header before loading.

---

## NVIDIA GPU Firmware: GSP-RM and Falcon

NVIDIA's firmware story is the most architecturally significant of the three major vendors, because the open-source kernel driver path (both `nvidia-open` and `nouveau`) fundamentally depends on a firmware blob running the entire GPU resource manager on dedicated RISC-V silicon inside the chip.

### Falcon: The Embedded Microcontroller Family

Falcon (FAst Logic Controller) is NVIDIA's family of small embedded RISC-like microcontroller cores. Modern GPUs contain multiple Falcon instances assigned to different sub-engines:
[Source](https://docs.kernel.org/gpu/nova/core/falcon.html)

| Falcon instance | Abbreviation | Role |
|----------------|-------------|------|
| Front-End Command Scheduler | FECS | GPC (Graphics Processing Cluster) scheduling |
| GPC Command Scheduler | GPCCS | Per-GPC work dispatch |
| Power Management Unit | PMU | Clock/power micro-management |
| Security Engine 2 | SEC2 | Secure boot loader / HS executor |
| GSP (GPU System Processor) | GSP | Full resource manager (Turing+) |

Falcon security operates in three privilege levels:

- **Heavy Secured (HS / PL3)**: firmware signed by NVIDIA hardware root-of-trust. Executed directly from Boot ROM validation. Has unrestricted chip access.
- **Light Secured (LS / PL2)**: firmware validated by an HS-mode Falcon before execution. Can access most resources but not HS-only registers.
- **Non-Secured (NS / PL0)**: no signature validation. Used for driver-generated microcode in older generations.

The trust chain on Turing and Ampere GPUs is:

```
SEC2 Boot ROM (hardware root of trust)
  └── Booter (HS, signed by NVIDIA)
        └── GSP-RM (LS, authenticated by Booter)
```
[Source](https://docs.kernel.org/gpu/nova/core/falcon.html)

### GSP-RM: The GPU Resource Manager Firmware

Starting with Turing (RTX 20xx, Quadro RTX), NVIDIA migrated the entire GPU Resource Manager from CPU-side code into firmware running on the GSP — a dedicated RISC-V processor with its own DRAM, caches, and interrupt controller embedded on the GPU die.
[Source](https://deepwiki.com/NVIDIA/open-gpu-kernel-modules/4.3-gsp-gpu-system-processor)

GSP-RM handles:
- GPU channel allocation and deallocation.
- VRAM memory management (GMMU page table management).
- Bar1/Bar2 mapping.
- Thermal and power management (previously done by CPU-side PMU firmware).
- Error containment and recovery.
- SR-IOV virtual function management.

The CPU-side kernel driver (`nvidia-open` or `nouveau`) communicates with GSP-RM via a shared-memory RPC (Remote Procedure Call) mechanism. Two circular buffers sit in system DRAM:

- **Command Queue** (256 KB): CPU writes RPC commands; GSP reads and executes them.
- **Status Queue** (256 KB): GSP writes responses and asynchronous events; CPU reads them.

```c
/* kernel-open/nvidia/nv-firmware.c (nvidia-open) */
/*
 * KernelGsp is the top-level kernel object managing GSP communication.
 * _kgspRpcSendMessage writes to command queue and rings GSP doorbell.
 * _kgspRpcRecvPoll polls the status queue for the response.
 */
```

GSP firmware file layout under `/lib/firmware/nvidia/`:

```
nvidia/tu102/gsp/gsp-535.113.01.bin     # Turing TU102 (RTX 2080 Ti)
nvidia/ga102/gsp/gsp-535.113.01.bin     # Ampere GA102 (RTX 3080)
nvidia/ad102/gsp/gsp-535.113.01.bin     # Ada Lovelace (RTX 4090)
nvidia/gh100/gsp/gsp-535.113.01.bin     # Hopper H100
nvidia/gb100/gsp/gsp-560.35.03.bin      # Blackwell B100
```

The version number in the filename matches the NVIDIA driver release version. Both `nvidia-open` and `nouveau` (when using GSP mode) load exactly the same firmware binary.
[Source](https://github.com/NVIDIA/open-gpu-kernel-modules/blob/main/nouveau/extract-firmware-nouveau.py)

### GSP Firmware Loading in nvidia-open

In `nvidia-open`, GSP firmware loading occurs during GPU initialisation:

1. The CPU-side driver loads the Booter (HS Falcon binary, `booter_load-*.bin`) via SEC2.
2. Booter validates and loads GSP-RM firmware from system RAM into GSP DRAM.
3. GSP boots, initialises all hardware, and signals readiness via the status queue.
4. All subsequent resource management operations are RPCs to GSP.

The `NVreg_EnableGpuFirmware=0` module parameter disables GSP mode on Turing GPUs (only), reverting to the legacy CPU-side resource manager. On Hopper (H100) and Blackwell, GSP mode is mandatory and cannot be disabled.

```bash
# Check if GSP firmware is in use
dmesg | grep -i "gsp\|firmware" | grep -i nvidia
# Output (GSP active):
# nvidia 0000:01:00.0: GSP firmware version: 570.144
```

### Falcon Memory Architecture

Each Falcon instance has a Harvard-style split between instruction memory (IMEM) and data memory (DMEM). An integrated DMA engine (FBDMA — FrameBuffer DMA) transfers firmware from system DRAM or GPU VRAM into these local memories via the FBIF (FrameBuffer Interface).

The IO-PMP (Input/Output Physical Memory Protection) unit controls which system memory addresses FBDMA may access, enforcing that HS firmware images are fetched only from authenticated locations. When an HS Falcon validates an LS image, it loads it into the target Falcon's IMEM and then sets the target's privilege level before releasing it from reset.

This chained loading model means a compromise in the HS Booter's own firmware (which is verified by SEC2's hardware Boot ROM) would give an attacker control over every other Falcon on the chip, which is why NVIDIA's hardware Boot ROM root-of-trust is so security-critical.

### Nouveau Firmware: Falcon and Devinit

The open-source Nouveau driver historically generated Falcon firmware ("FUC" — Falcon micro-code) at build time for pre-Maxwell GPUs, allowing redistribution without binary blobs. The FUC assembler was part of the `envytools` project (https://github.com/envytools/envytools) which also shipped `nvbios` for parsing NVIDIA video BIOS and `rnndb` for register documentation. This approach enabled Nouveau to run on Fermi and Kepler GPUs with zero proprietary blobs.

The situation changed fundamentally with Maxwell (2014). NVIDIA began requiring HS-mode Falcon signature validation for GPC (Graphics Processing Cluster) scheduling operations. Unsigned Falcon images could no longer access privileged registers needed for full 3D performance. This was explicitly communicated to the Nouveau community in [NVIDIA's 2014 Falcon security announcement](https://www.phoronix.com/news/MTc5ODA).

For Maxwell through Ampere, Nouveau requires firmware extracted from NVIDIA's proprietary driver package. NVIDIA open-sourced a Python extraction script alongside `nvidia-open`:

```bash
# Extract firmware from NVIDIA .run installer using the open-source script
python3 nouveau/extract-firmware-nouveau.py nvidia-driver-570.run
# Places files in /lib/firmware/nvidia/{arch}/gsp/
```
[Source](https://github.com/NVIDIA/open-gpu-kernel-modules/blob/main/nouveau/extract-firmware-nouveau.py)

The extracted files include:
- `bootloader-{ver}.bin` — the HS bootloader image for each GPU architecture class.
- `booter_load-{ver}.bin` / `booter_unload-{ver}.bin` — Booter images (authenticated by SEC2) that load and unload GSP-RM.
- `gsp-{ver}.bin` — GSP-RM firmware itself (runs in LS mode on the GSP RISC-V core).
- `gpu_scrubber-{ver}.bin` — secure memory scrubber (Ada Lovelace and later, for confidential computing setup).

On Ampere and later, Nouveau's `nouveau_gsp_init()` uses the same GSP-RM path as `nvidia-open`, communicating with NVIDIA's firmware via the RPC protocol. This means Nouveau on modern GPUs effectively runs the same proprietary resource manager as the closed driver — the "open" part is the kernel-to-firmware interface layer, not the GPU management logic itself.

On pre-Turing hardware where Nouveau handles scheduling entirely in software, the relevant module parameter is:

```bash
# Disable GSP mode in Nouveau (Turing only, for debugging)
modprobe nouveau config=NvGrFirmware=0
```

On Kepler and Fermi hardware where Falcon HS validation is not enforced for GPC scheduling, Nouveau can still operate without binary blobs by generating FUC images at module load time — though even here, `devinit` scripts (executed by PMU in LS mode to initialise clocks and memory on GPU power-up) are required and come from NVIDIA's VBIOS.
[Source](https://download.nvidia.com/open-gpu-doc/Falcon-Security/1/Falcon-Security.html)

---

## linux-firmware Package and Versioning

### Repository Structure

All three vendors' GPU firmware blobs (plus hundreds of other device firmware files) are maintained in the `linux-firmware` Git repository at:

```
https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git
```

The repository uses date-based version tags (`20240115`, `20241201`, etc.). The `WHENCE` file at the repository root is the authoritative licence manifest: it records the licence terms, upstream source URL, and contact information for every firmware file.

GPU firmware licence entries in `WHENCE` are typically marked as:
- **Proprietary** — binary redistribution permitted; source not available.
- Some AMD firmware (especially older generations) is dual-licensed or has been re-released under more permissive terms.

### Distribution Packages

| Distro | Package name | Typical installation path |
|--------|-------------|--------------------------|
| Debian/Ubuntu | `firmware-amd-graphics`, `firmware-misc-nonfree` | `/lib/firmware/` |
| Fedora | `linux-firmware` | `/lib/firmware/` |
| Arch Linux | `linux-firmware` | `/usr/lib/firmware/` |
| openSUSE | `kernel-firmware-amdgpu`, `kernel-firmware-i915` | `/lib/firmware/` |

### Discovering Required Firmware

To see which firmware files a given kernel module requires:

```bash
modinfo amdgpu | grep ^firmware
modinfo i915   | grep ^firmware
modinfo xe     | grep ^firmware
modinfo nouveau | grep ^firmware
```

This output lists the firmware filenames relative to `/lib/firmware/`. If any file is missing, the driver will fail to initialise the corresponding hardware block.

### Updating linux-firmware

After installing a new `linux-firmware` package version, the initramfs must be regenerated if firmware is embedded there:

```bash
# Debian/Ubuntu
update-initramfs -u -k all

# Fedora/RHEL
dracut --regenerate-all -f

# Arch Linux (with mkinitcpio)
mkinitcpio -P
```

Failure to regenerate the initramfs is a common cause of "firmware load failure" errors after a `linux-firmware` package update: the old initramfs still contains the old (or missing) blobs, while the new firmware is only in rootfs.

---

## Firmware Authentication and Security

### AMD PSP Security Chain

The AMD PSP establishes a hardware root of trust for all GPU firmware. The security architecture on RDNA 3 and CDNA 3 hardware works as follows:

1. **PSP Boot ROM** (immutable, in silicon) validates the PSP SOS (`psp_13_0_0_sos.bin`) using an RSA-4096 public key burned into fuses.
2. **PSP SOS** (the PSP operating system) validates all subsequent firmware blobs: GFX ME/MEC/RLC, SMU, DMCUB, VCN, MES.
3. Each firmware blob is signed with AMD's production private key. Validation is performed inside the PSP; the kernel driver submits the blob and receives an authenticated/rejected status.

Firmware signed for one GPU generation cannot be used on another, and AMD enforces an anti-rollback policy: firmware older than the version recorded in hardware fuses will be rejected. This prevents downgrade attacks that would reintroduce patched-out vulnerabilities.

For confidential computing (AMD SEV-ES, SEV-SNP), the PSP also manages encryption keys for guest VM memory isolation. The firmware thus sits at the intersection of graphics and platform security.

### Intel Content Protection

Intel's PAVP (Protected Audio-Video Path) is enforced by HuC firmware on Gen12 hardware. The HuC authentication step — where GuC validates HuC's RSA signature — is part of the content protection activation chain. Without HuC authentication, the driver cannot enable hardware content protection (DRM enforcement), though unprotected video decode continues normally.

On Xe-era hardware, the GSC takes over this role. The GSC communicates with Intel's MEI (Management Engine Interface) bus for attestation.

### NVIDIA Firmware Signing

NVIDIA signs all Falcon firmware images with its own private key. Starting with Maxwell, security-critical operations (mode transitions, power management registers) require HS-mode firmware. The hardware Boot ROM verifies the HS image signature before execution; unsigned or improperly signed images simply cannot run in HS mode, preventing third parties from replacing firmware with malicious alternatives.

The `NVreg_OpenRmEnableUnsupportedGpus=1` module parameter in `nvidia-open` bypasses some compatibility guards for older or experimental GPUs, but does not bypass signature enforcement — Falcon signature verification happens in hardware, not software.

### Confidential Computing and MIG

On NVIDIA H100 (Hopper), GSP firmware manages confidential computing (H100 CC) mode and Multi-Instance GPU (MIG) partitioning. In CC mode, GSP firmware handles the encryption/decryption of GPU-to-memory traffic, key provisioning, and attestation report generation. These operations require GSP firmware from NVIDIA specifically built with CC support; a standard GSP binary cannot be substituted.

### CVE Analysis: GPU Firmware Vulnerabilities

GPU firmware vulnerabilities are relatively rare in public CVE databases but have significant impact when they occur:

**CVE-2023-20579**: An improper access control vulnerability in AMD's SPI protection mechanism affecting multiple Ryzen processor families (3000–7000 series). An attacker with Ring 0 (kernel-mode) access could bypass SPI flash write protections. CVSS 3.1 score: 6.0 (Medium). Fixed via PSP firmware updates delivered through AMD's platform firmware (not through `linux-firmware`). The relevant AMD security bulletin is AMD-SB-7009.
[Source](https://nvd.nist.gov/vuln/detail/cve-2023-20579)

**GPU firmware supply chain security**: The `linux-firmware` repository GPG-signs its git tags but individual firmware blobs are not signed at rest on the filesystem. A compromised `/lib/firmware/` directory could allow a local attacker with write access to substitute a malicious firmware blob. The GPU's own secure boot chain (PSP, GuC RSA check, Falcon HS verification) is the last line of defence against such substitution.

**Secure Boot interaction**: GPU firmware is loaded *after* UEFI Secure Boot completes. The UEFI SHIM and kernel image may be verified, but `/lib/firmware/` contents are not — unless measured boot (TPM PCR extension) is in use. Measured boot records the firmware blob hashes into TPM PCRs, enabling attestation that the expected blobs were loaded.

### fwupd Security Attestation

The `fwupd` daemon's security command assesses the overall firmware security posture:

```bash
fwupdmgr security
```

This report includes HSI (Host Security ID) level indicators that cover UEFI Secure Boot, TPM PCR measurements, and firmware vendor signing quality. GPU firmware trust level is factored into the HSI score on systems where fwupd has visibility into the GPU firmware version.
[Source](https://fwupd.org/)

---

## fwupd and GPU Firmware Updates

### The fwupd Daemon

`fwupd` is the standard Linux firmware update daemon, providing a D-Bus API and CLI (`fwupdmgr`) for discovering and applying firmware updates. It connects to the LVFS (Linux Vendor Firmware Service at https://fwupd.org/), a vendor-neutral repository where hardware manufacturers publish signed firmware packages.
[Source](https://github.com/fwupd/fwupd)

```bash
# List all devices fwupd is aware of
fwupdmgr get-devices

# Check for available firmware updates
fwupdmgr get-updates

# Apply all pending updates (reboots if required)
fwupdmgr update

# Security attestation report
fwupdmgr security
```

### LVFS Firmware Package Format

Firmware packages on LVFS are distributed as `.cab` (Cabinet) archives. Each cabinet contains:
- One or more raw firmware blob files (`.bin`, `.efi`, etc.)
- A `metainfo.xml` file describing the device, firmware version, compatibility requirements, and update protocol.
- An optional `firmware.jcat` JCat signature file covering all other files in the cabinet.

LVFS metadata is signed with GPG; `fwupdmgr refresh` verifies the signature before trusting the package list. This prevents man-in-the-middle injection of malicious firmware packages.

### GPU-Specific LVFS Coverage

GPU firmware updates via LVFS are narrower than one might expect:

| GPU subsystem | Primary update path | LVFS coverage |
|--------------|---------------------|---------------|
| AMD dGPU GFX/SMU/PSP | `linux-firmware` package | Limited (some iGPU APU entries) |
| AMD iGPU (Renoir/Cezanne) | Platform firmware / `linux-firmware` | Partial |
| Intel DMC | `linux-firmware` package | Rare; mostly via kernel driver update |
| Intel GuC/HuC | `linux-firmware` package | No dedicated LVFS entry |
| NVIDIA GSP | NVIDIA driver `.run` package | No LVFS entry |

The practical reality is that most GPU firmware reaches users via the `linux-firmware` package (Debian/Ubuntu: `apt upgrade firmware-amd-graphics`) rather than through `fwupdmgr update`. Intel ME firmware (Management Engine, unrelated to GPU ME) *does* have LVFS entries from OEM vendors, but this is a platform component, not the GPU driver firmware.

### GPU Plugin

The `fwupd` codebase includes a GPU plugin (`plugins/gpu/fu-gpu-plugin.c`) that provides read-only inventory of loaded GPU firmware versions. This allows the security attestation report to include GPU firmware version information, enabling compliance monitoring without providing an update path.

```bash
# Examine fwupd's view of the GPU
fwupdmgr get-devices --filter=gpu
```

---

## Debugging Firmware Loading Failures

### First Line of Diagnosis: dmesg

Firmware load failures are always logged to the kernel ring buffer. Filter the relevant lines:

```bash
dmesg | grep -iE "firmware|amdgpu|i915|xe|nouveau|nvidia" | grep -iE "fail|error|missing|load"
```

Typical failure messages and their causes:

```
[drm] amdgpu: Failed to load firmware "amdgpu/navi10_sos.bin" (-2)
```
Error `-2` is `ENOENT` — the firmware file does not exist in `/lib/firmware/`. Verify:
```bash
ls -la /lib/firmware/amdgpu/navi10_sos.bin
```

```
[drm] amdgpu: psp_cmd_submit_buf: timeout, psp stall
```
The PSP received the firmware but failed to authenticate it. Causes: wrong firmware for this GPU revision, corrupted file, or hardware fault.

```
[drm] i915: Failed to load firmware i915/tgl_guc_70.bin (-22)
```
Error `-22` is `EINVAL` — the firmware file exists but failed validation (wrong size, wrong magic bytes, truncated download).

### Listing Loaded Firmware Versions

AMD:
```bash
cat /sys/kernel/debug/dri/0/amdgpu_firmware_info
```
Output example:
```
VCE feature version: 0, firmware version: 0x0
UVD feature version: 0, firmware version: 0x0
MC firmware version: 0x00000000
ME firmware version: 0x000000b0
PFP firmware version: 0x00000071
RLC firmware version: 0x000000b2
...
```

Intel GuC:
```bash
cat /sys/kernel/debug/dri/0/i915_guc_info
# or for Xe:
cat /sys/kernel/debug/dri/0/xe_guc_info
```

NVIDIA (open driver):
```bash
dmesg | grep "firmware version"
# nvidia 0000:01:00.0: GSP firmware version: 570.144
```

### Overriding Firmware for Testing

AMD provides a module parameter to force a specific firmware file path:

```bash
modprobe amdgpu fw_load_type=0   # 0=direct (bypass PSP), 1=SMU, 2=PSP (default)
```

Setting `fw_load_type=0` (direct load) causes the driver to upload firmware directly to the GPU registers without going through the PSP authentication path. This is useful for firmware development and debugging on pre-production hardware where signed firmware is not yet available, but should never be used in production.

For Intel, disabling GuC submission (fallback to Execlist) for debugging:
```bash
modprobe i915 enable_guc=0
```

For Nouveau, falling back to software GPC scheduling:
```bash
modprobe nouveau config=NvGrFirmware=0  # Turing only
```

### Verifying Firmware Presence Before Driver Load

Before loading the GPU driver (e.g., in a build environment or CI):

```bash
# Check all firmware files required by amdgpu are present
modinfo amdgpu | grep ^firmware | awk '{print "/lib/firmware/" $2}' | \
  xargs -I{} sh -c 'test -f "{}" && echo "OK: {}" || echo "MISSING: {}"'
```

### initramfs Verification

If the system boots before rootfs is mounted and the GPU is needed (e.g., EFI framebuffer or Plymouth):

```bash
# Inspect initramfs contents for firmware
lsinitramfs /boot/initrd.img-$(uname -r) | grep firmware/amdgpu

# For RHEL/Fedora initrd
lsinitrd /boot/initramfs-$(uname -r).img | grep firmware/amdgpu
```

If AMD firmware files are absent from the initramfs listing, re-run `update-initramfs -u -k all` after installing/updating the `firmware-amd-graphics` package.

### Firmware Version Mismatch

A subtle failure mode occurs when the `linux-firmware` package is updated but the kernel module (`amdgpu.ko`, `i915.ko`) is from a different kernel version. The firmware ABI is versioned: each firmware blob contains an embedded version header, and the driver checks that the loaded blob is within the supported version range. If the linux-firmware package ships a firmware version newer than the driver supports:

```
[drm] amdgpu: smu11_0: Unsupported SMU version 0x00402d00
[drm] amdgpu: SMU v11.0: Failed to check pptable version!
```

The fix is to keep the kernel and `linux-firmware` versions in sync — typically both come from the same distro package channel and update together. On distributions that ship kernel and firmware separately (e.g., Debian non-free firmware), version skew is a real operational concern.

### Stale Firmware Cache After modprobe

The kernel caches firmware blobs for the lifetime of the requesting module. If you update a firmware file on disk and want to reload it without rebooting:

```bash
# Unload and reload the GPU driver (warns: this disrupts all GPU operations)
modprobe -r amdgpu && modprobe amdgpu

# Or trigger a firmware cache flush via sysfs (kernel 5.10+)
echo 1 > /sys/class/firmware/timeout
```

Note that unloading `amdgpu` terminates all OpenGL/Vulkan contexts, kills compositor processes, and drops the framebuffer — only viable on a headless server or via SSH.

---

## Roadmap

### Near-term (6–12 months)

- **fwupd GPU firmware update coverage expansion**: fwupd 2.x (latest release 2.1.3, May 2026) is expanding AMD GPU firmware update support beyond Navi 3x to additional RDNA 4 SKUs, and Intel is contributing GuC/HuC update plugins. Broader fwupd integration means system tools like GNOME Software and KDE Discover will surface GPU firmware updates alongside UEFI and NIC firmware. [Source](https://en.wikipedia.org/wiki/Fwupd)
- **NVIDIA GSP-RM firmware consolidation in linux-firmware**: Following the large GSP binary firmware blob commits to linux-firmware.git, NVIDIA's open kernel module (nvidia-open) increasingly relies on signed GSP-RM blobs for all Turing and later GPUs; near-term work focuses on packaging these blobs in distro firmware packages so open-driver users no longer require manual firmware installation. [Source](https://www.phoronix.com/news/NVIDIA-Big-GSP-Firmware-Dump)
- **Firmware version mismatch detection improvements**: Kernel maintainers are working on better tooling to detect and warn when the linux-firmware package version diverges from the loaded kernel module's expected firmware ABI, reducing silent fallback-to-software-mode failures on rolling-release distros. Note: needs verification.
- **Rust-based firmware loading abstractions**: The Linux kernel's ongoing Rust integration (600,000+ lines of Rust in kernel as of early 2026) is expected to produce safe Rust wrappers around the `request_firmware` C API, initially for drivers being re-implemented in Rust (such as Nova, the Rust NVIDIA kernel driver). [Source](https://www.webpronews.com/linux-kernel-2026-rust-integration-and-security-advances/)
- **AMD PSP 14.0 firmware enablement for RDNA 4**: PSP 14.0 and associated GC/VCN/SDMA firmware components are being staged in linux-firmware for upcoming RDNA 4 hardware, with amdgpu driver patches in the `drm-next` tree adding matching IP block registration. [Source](https://www.phoronix.com/news/AMDGPU-Enabling-PSP-14.0)

### Medium-term (1–3 years)

- **Firmware authentication via kernel keyring and IMA**: A long-discussed direction for the firmware loader subsystem is integrating with the Integrity Measurement Architecture (IMA) so that firmware blobs are measured and their hashes recorded in the TPM PCR registers at load time. This would allow remote attestation frameworks to verify that a specific GPU firmware version was loaded, closing a gap in measured-boot coverage. Note: needs verification.
- **Nova (Rust NVIDIA driver) GSP-RM interface stabilisation**: Nova's Rust kernel driver (Ch10) is expected to reach feature parity with Nouveau's GSP-RM path within 1–2 years; this requires a stable kernel-to-GSP-RM RPC interface specification, which NVIDIA is developing in coordination with the Nouveau/Nova maintainers. [Source](https://forums.developer.nvidia.com/t/cannot-initialize-gsp-firmware-rm/351473)
- **Distributing GPU firmware via OCI/container registries**: There is ongoing discussion in the linux-firmware and fwupd communities about publishing firmware blobs as OCI container image layers, which would enable Kubernetes and container-native GPU workloads to pin exact firmware versions and update them via standard container tooling rather than host package managers. Note: needs verification.
- **Intel Xe2/Battlemage GuC firmware co-design**: The Xe driver's GuC submission path (introduced with Xe2) involves a rearchitected GuC firmware interface compared to i915; medium-term work focuses on aligning the GuC firmware ABI across both drivers and simplifying the dual-driver maintenance burden for Intel. [Source](https://docs.kernel.org/gpu/xe/xe_firmware.html)
- **Firmware signing key lifecycle management**: The UEFI Secure Boot certificate changes of 2026 (expiry of Microsoft's 2011 third-party signing key) have prompted discussion about formal lifecycle policies for GPU firmware signing keys — particularly for AMD's PSP, where the CPU-side firmware (SEV) has moved to open source but GPU PSP firmware signing keys remain proprietary. [Source](https://access.redhat.com/articles/7128933)

### Long-term

- **Open GPU firmware for compute-class hardware**: The open-source GPU movement (e.g., open NVIDIA kernel modules, AMD's open-source driver stack) creates long-term pressure to open-source at least the security-critical PSP/GSP firmware source under auditable licences, even if signing keys remain controlled. AMD has published SEV firmware source as a precedent. [Source](https://www.phoronix.com/forums/forum/hardware/processors-memory/1406519-amd-publishes-sev-firmware-as-open-source)
- **Unified GPU firmware manifest and dependency system**: As GPU firmware ecosystems grow (AMD ships 50+ MB of firmware per generation across SMU, PSP, GFX, VCN, SDMA, DMCUB components), there is architectural interest in a manifest-based dependency system (similar to UEFI Capsule Update manifests) so that the kernel can atomically load or reject a coherent set of firmware versions rather than loading each blob independently and discovering version mismatches at runtime.
- **Confidential computing integration for GPU workloads**: AMD's SEV-SNP and Intel TDX confidential VM technologies are being extended to cover GPU-attached workloads; long-term, GPU firmware will need to participate in attestation flows so that the guest VM can verify it is running a specific, unmodified firmware version inside a trusted execution environment. [Source](https://docs.nvidia.com/infra-controller/documentation/provisioning-day-0/measured-boot-attestation)
- **Elimination of initramfs firmware duplication**: For systems where the GPU drives the boot display, firmware must be duplicated into the initramfs. Long-term kernel infrastructure improvements (e.g., early firmware loader with compressed in-kernel cache, or EFI stub firmware hand-off from UEFI) could eliminate this duplication and reduce initramfs size for GPU-heavy embedded and set-top-box platforms. Note: needs verification.

---

## Integrations

The GPU firmware subsystem intersects with many other chapters in this book:

- **Ch1 (DRM Architecture)**: `request_firmware()` is called during the DRM driver's `probe()` path; the DRM core provides no special firmware loading facilities — drivers use the kernel firmware API directly.
- **Ch2 (KMS and the Display Pipeline)**: DMC firmware (Intel) and DMCUB firmware (AMD) are display-engine components that enable DC power states and DSC/MST features central to KMS operation.
- **Ch5 (AMDGPU Driver Internals)**: SMU, PSP, GFX ME/MEC/RLC, and MES firmware are all loaded during AMDGPU device initialisation, before any KMS or DRM scheduling begins.
- **Ch6 (i915 Driver Internals)** / **Ch7 (Xe Driver)**: GuC, HuC, and DMC firmware are loaded during i915/Xe driver probe; GuC is tightly coupled to the command submission path discussed in those chapters.
- **Ch8 (Nouveau)**: Nouveau's Falcon microcode and GSP-RM firmware are the foundation on which all Nouveau GPU operations rest; the chapter discusses extraction from NVIDIA proprietary packages.
- **Ch10 (Nova/NVK)**: Nova's Rust-language kernel driver also depends on Falcon HS firmware loaded through the same `request_firmware` pathway; the Falcon security model (HS/LS/NS privilege levels) is foundational to Nova's design.
- **Ch26 (Hardware Video Acceleration)**: VCN firmware (AMD) and HuC firmware (Intel) are prerequisites for hardware VA-API decode acceleration; their absence results in software codec fallback.
- **Ch51 (GPU Power Management)**: SMU firmware (AMD) and GuC SLPC (Intel) are the firmware agents that perform DVFS, RC6 power gating, and TBP enforcement described in that chapter.
- **Ch102 (DRM GPU Scheduler)**: GuC submission (Intel) and MES (AMD) are firmware-based schedulers that interact closely with the DRM GPU scheduler layer for multi-tenant queue management.
- **Ch122 / Ch117 (DKMS and Out-of-Tree Drivers)**: Drivers built via DKMS must have their required firmware installed separately; the `dkms.conf` `BUILT_MODULE_NAME` mechanism does not handle firmware installation, which must be handled by the package manager or by explicit `cp` into `/lib/firmware/`.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
