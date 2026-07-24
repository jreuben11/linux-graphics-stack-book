# Chapter 80 — GPU Security: Isolation, Content Protection, and Confidential Computing

**Target audiences:** Systems and driver developers who need to reason about GPU attack surface, process isolation, and firmware trust; graphics application developers integrating content protection or confidential AI workloads; security researchers auditing the Linux DRM subsystem.

---

## Table of Contents

1. [GPU Attack Surface Overview](#1-gpu-attack-surface-overview)
2. [GPU Process Isolation](#2-gpu-process-isolation)
3. [IOMMU and DMA Attack Mitigation](#3-iommu-and-dma-attack-mitigation)
4. [HDCP 2.2 — Content Protection in the Kernel](#4-hdcp-22--content-protection-in-the-kernel)
5. [Secure Display Path](#5-secure-display-path)
6. [GPU Firmware Signing and Secure Boot](#6-gpu-firmware-signing-and-secure-boot)
7. [AMD SEV-SNP and GPU Confidential Computing](#7-amd-sev-snp-and-gpu-confidential-computing)
8. [NVIDIA H100 Confidential Computing](#8-nvidia-h100-confidential-computing)
9. [GPU Side-Channel Attacks](#9-gpu-side-channel-attacks)
10. [Driver Security Hardening](#10-driver-security-hardening)
11. [eBPF for GPU Access Control and Security Monitoring](#11-ebpf-for-gpu-access-control-and-security-monitoring)
12. [Integrations](#12-integrations)

---

## 1. GPU Attack Surface Overview

GPUs sit at an unusually exposed position in the system security model. Unlike a CPU, a GPU is a multi-tenant accelerator: many processes share the same physical die simultaneously, each with its own command stream and memory allocations but all executing on the same compute engines and memory buses. This creates an attack surface that spans several distinct categories, each addressed in depth in the sections that follow.

**Process isolation failures.** If the kernel driver does not correctly enforce per-process **GPU** virtual address spaces — or fails to scrub **GPU** memory and registers between contexts — a compromised or malicious process can read another process's framebuffer, texture data, model weights, or encryption keys that happen to reside in **GPU**-local **DRAM**. The **DRM** subsystem manages per-process **GPU** virtual address spaces through the **DRM_GPUVM** framework, associating each **drm_gem_object** with a list of **GPU** virtual address mappings. A critical moment is context teardown on process exit: **VRAM** pages must be scrubbed before being returned to the allocator, since **DRAM** retains its contents after deallocation. Research has shown that **CUDA**'s default allocator (**cudaMalloc**) does not zero **VRAM** between allocations. **AMD** implemented an architectural solution — the **cleaner shader** — for **CDNA2**/**GFX9.4.2** and later (**GFX11.5**), which clears **LDS** (Local Data Store), **VGPRs** (Vector General Purpose Registers), and **SGPRs** (Scalar General Purpose Registers) after each job, controllable via sysfs **enforce_isolation** and **run_cleaner_shader** interfaces.

**DMA attacks.** **GPU**s perform direct memory access to host **DRAM** in both directions. Without an **IOMMU** in the data path, a **GPU** driver vulnerability that allows an attacker to redirect **DMA** can compromise the entire host. On x86 the two **IOMMU** implementations are **Intel VT-d** and **AMD-Vi**; on **ARM** systems the **ARM SMMU** provides equivalent isolation, with the unified kernel **IOMMU** **API** in **include/linux/iommu.h**. The kernel groups **PCIe** devices that cannot be reliably isolated into **IOMMU groups**; the **ACS** (Access Control Services) **PCIe** feature enables per-function isolation at the switch level. The same risk applies to **GPU** passthrough in virtualisation environments: **VFIO** (Virtual Function I/O) is the Linux framework for assigning **PCIe** devices directly to virtual machines while maintaining host-kernel safety, requiring a properly scoped **IOMMU** group and refusing passthrough without one.

**Content protection bypass.** Premium 4K content distribution relies on **HDCP** (High-bandwidth Digital Content Protection) to prevent interception on the display link. **HDCP 1.4** uses a 56-bit symmetric key protocol; **HDCP 2.2** uses a modern three-phase exchange — **AKE** (Authentication and Key Exchange), **LC** (Locality Check), and **SKE** (Session Key Exchange) — incorporating the receiver's **RSA** public key certificate, a DCP LLC signature, and the global **LC128** constant. The kernel defines **HDCP_STREAM_TYPE0** and **HDCP_STREAM_TYPE1** content types: Type 1 (premium 4K) must only flow over **HDCP 2.2**-authenticated links. The **DRM** connector-level **API** exposes **DRM_MODE_CONTENT_PROTECTION_DESIRED** and **DRM_MODE_CONTENT_PROTECTION_ENABLED** properties via **drm_connector_attach_content_protection_property()**. **Intel**'s implementation routes **HDCP 2.2** authentication through the **GSC** (Graphics Security Controller) on **DG2** (Alchemist) and later via **intel_hdcp_gsc_message.c**; older platforms used the **MEI** (Management Engine Interface). A NULL pointer dereference in this path was assigned **CVE-2024-53050**.

**Secure display path.** When **HDCP 2.2** **Type 1** authentication is active, the display pipeline must be completely sealed from the **GPU** framebuffer to the panel, preventing even the **Wayland** compositor from capturing protected pixels. **Intel** platforms implement **PXP** (Protected Xe Path) — a hardware feature on **Gen12+** that creates a separate encrypted path through the **GPU**'s render engine to the display engine for protected content. The **Wayland** protocol extension **zwp_linux_content_type_manager_v1** / **zwp_content_type_v1** provides a minimal API for clients to request surface protection, with the compositor reporting back a **status** event.

**Firmware supply-chain trust.** Modern **GPU**s load multiple signed firmware blobs during initialisation: **GSP-RM** on **NVIDIA** (loaded via **nvkm_firmware_get()** in **Nouveau**, validated by the **GPU Boot ROM** with **PKC** signatures), **GuC** (Graphics Microcontroller) and **HuC** (HEVC Microcontroller) on **Intel** (loaded via **intel_uc_fw_fetch()**, carrying **CSS** headers for hardware boot validation), and **PSP** (**Platform Security Processor**) firmware on **AMD**. The **NVIDIA** **ACR** (Access Control Region) subsystem manages **HS Falcon** (Heavy-Secured) and **LS Falcon** (Light-Secured) microcontroller privilege levels enforced via a **WPR** (Write-Protected Region). The **AMD PSP** is an **ARMv7 Cortex-A5** core owning the root of trust, with a signing chain from **ARK** (AMD Root Key) through **ASK** (AMD SEV Signing Key) to **CEK** (Chip Endorsement Key). **UEFI Secure Boot** enforces that **GPU** kernel modules (**nvidia.ko**, **amdgpu.ko**, **i915.ko**) are signed via the **MOK** (Machine Owner Key) database; the **CONFIG_MODULE_SIG_FORCE** kernel option makes this mandatory.

**Side-channel attacks.** **GPU** memory systems exhibit timing and power characteristics that leak information across security boundaries. Shared **L1**/**L2** caches on **GPU**s are susceptible to **Prime+Probe** and **Flush+Reload** timing attacks. In 2025 researchers published **GPUHammer**, the first demonstrated **Rowhammer** attack on discrete **GPU** **GDDR6** memory, achieving bit-flips via standard **CUDA** compute kernels and demonstrating **AI** model accuracy degradation on an **NVIDIA A6000**. Power side channels are also relevant: **NVML**'s **nvmlDeviceGetPowerUsage()** and **AMD ROCm**'s **rsmi_dev_power_ave_get()** expose per-device wattage that could be exploited for coarse power analysis. Mitigations include the **kernel.perf_event_paranoid** sysctl, **AMDGPU**'s **amdgpu_perfcnt_ioctl** restricted by default, **Intel** **i915**/**Xe**'s **OA** (Observation Architecture) counters gated by process capabilities, and hardware **ECC** on datacenter parts (**A100**, **H100**, **AMD MI300**).

**Confidential computing.** As **AI** workloads migrate to the cloud, customers demand that their model weights and inference data remain encrypted even from the cloud operator. This requires a **GPU TEE** (Trusted Execution Environment) with hardware-enforced memory isolation and a remote attestation path. **NVIDIA**'s **Hopper** architecture (**H100**) delivered the first **GPU TEE** via three **CC** modes (**CC-Off**, **CC-On**, **CC-DevTools**), with a **CPR** (Compute Protected Region) in **HBM2e**, **AES-GCM**-encrypted bounce buffers in shared memory, and **SPDM** (Security Protocol and Data Model) attestation using a device-unique **ECC-384** key pair verified against **NVIDIA**'s certificate chain via **NRAS** (NVIDIA Remote Attestation Service) or the open-source **nvtrust** tooling. **AMD** is extending **SEV** (Secure Encrypted Virtualization) and **SEV-SNP** (Secure Nested Paging) toward **GPU** workloads, with the **HSMP** (Host System Management Port) and **SMI Transfer** interfaces relevant to confidential computing isolation.

**Driver security hardening.** The **DRM** subsystem controls access through per-ioctl permission flags — **DRM_AUTH**, **DRM_MASTER**, **DRM_ROOT_ONLY**, and **DRM_RENDER_ALLOW** — defined in **include/drm/drm_ioctl.h**. A key security boundary is the separation of **render nodes** (**\/dev\/dri\/renderD\***) from **primary nodes** (**\/dev\/dri\/card\***): render nodes expose only **DRM_RENDER_ALLOW** ioctl, while primary nodes gate display state changes. The **drm_ioctl()** function in **drivers/gpu/drm/drm_ioctl.c** enforces these checks before dispatching to driver handlers such as **amdgpu_info_ioctl** and **i915_gem_execbuffer2_ioctl**. The kernel fuzzer **syzkaller** has been the primary tool for discovering **DRM** security bugs — integer overflows, use-after-free in fence signalling, uninitialized memory in ioctl output structures, and missing capability checks. The **amdgpu** driver's history of **CVE**s in **VCN** (video decode/encode) paths and the **KFD** (Kernel Fusion Driver) ioctl surface illustrates the ongoing challenge of securing large, complex **GPU** drivers.

The remainder of this chapter addresses each category in depth.

### 1.1 What is GPU Process Isolation?

GPU process isolation is the mechanism by which the kernel and GPU hardware prevent one userspace process from accessing the GPU memory of another. Just as the CPU's Memory Management Unit (MMU) gives each process its own virtual address space backed by per-process page tables, modern GPUs include a GPU Memory Management Unit (GMMU) that enforces per-process GPU virtual address spaces. Each process receives its own set of GPU page tables, and the GPU translates virtual addresses through those tables before accessing physical VRAM. A failure to correctly set up or tear down these page tables — or to scrub GPU-local memory between process tenants — creates a path for data leakage across security boundaries.

In the Linux DRM subsystem, this isolation is managed through the DRM_GPUVM framework. Each `drm_gem_object` carries a list of GPU virtual address mappings (`gpuva` entries) that must be invalidated when the owning process exits. The `drm_file` release path is responsible for tearing down the GPU MMU context, unmapping all page table entries, and returning VRAM pages to the allocator in a state that cannot expose prior-tenant data to subsequent allocations. This chapter examines how different GPU vendors implement and enforce this boundary, including AMD's cleaner shader mechanism for clearing registers and Local Data Store between jobs.

### 1.2 What is an IOMMU?

An IOMMU (Input-Output Memory Management Unit) is a hardware unit that interposes on DMA (Direct Memory Access) transactions from peripheral devices — including GPUs — and enforces a device-visible page table called an IOMMU domain. Without an IOMMU, a GPU performing DMA has unrestricted access to all host physical memory. A single driver vulnerability that permits an attacker to redirect GPU DMA therefore bypasses the CPU MMU entirely, compromising the whole host.

On x86 systems, Intel provides this capability through VT-d (Virtualisation Technology for Directed I/O) and AMD through AMD-Vi. ARM systems rely on the ARM SMMU (System Memory Management Unit). The Linux kernel exposes a unified IOMMU API in `include/linux/iommu.h`, allowing drivers to create and attach IOMMU domains that restrict the physical addresses a device can reach. PCIe devices that cannot be isolated from each other — typically because they share a bridge lacking ACS (Access Control Services) capability — are grouped into IOMMU groups, and the kernel refuses to assign them individually to virtual machines via VFIO without complete group isolation. This chapter covers both the kernel IOMMU API and the VFIO framework that uses IOMMU isolation to enable secure GPU passthrough to virtual machines.

### 1.3 What is HDCP?

HDCP (High-bandwidth Digital Content Protection) is a cryptographic protocol that authenticates the display path from a GPU transmitter to a connected display sink — a monitor, television panel, or AV receiver — before permitting transmission of protected premium video content. Content licensing agreements require HDCP for 4K Ultra HD playback and for streams classified as "Type 1" by the DCP LLC, which may only traverse HDCP 2.2-authenticated links. HDCP 1.4 uses a 56-bit symmetric key exchange over the DDC (I²C) channel; HDCP 2.2 replaces this with a three-phase exchange — AKE (Authentication and Key Exchange), LC (Locality Check), and SKE (Session Key Exchange) — using RSA public-key certificates signed by the DCP LLC certificate authority and incorporating the shared LC128 global constant in key derivation.

In Linux, HDCP authentication is implemented in the DRM subsystem through driver-specific code in the Intel i915, Intel Xe, and AMD display drivers, with shared helper functions in `drivers/gpu/drm/display/drm_hdcp_helper.c`. The DRM connector-level API exposes content protection state to userspace via the `DRM_MODE_CONTENT_PROTECTION_DESIRED` and `DRM_MODE_CONTENT_PROTECTION_ENABLED` connector properties. On Intel platforms from DG2 (Alchemist) onward, HDCP 2.2 authentication messages are routed through the GSC (Graphics Security Controller) firmware, moving cryptographic operations out of the driver proper.

### 1.4 What is Confidential Computing on GPUs?

Confidential computing refers to hardware-enforced execution environments in which data and code remain encrypted and isolated even from the host operating system and cloud infrastructure operator. On CPUs, this capability is delivered by technologies such as AMD SEV-SNP (Secure Encrypted Virtualization with Secure Nested Paging) and Intel TDX. As AI inference and training workloads move to shared cloud platforms, the same threat model extends to GPU memory: a cloud customer may not wish to expose model weights, inference inputs, or activation data stored in GPU HBM or GDDR to the host kernel or hypervisor.

GPU Trusted Execution Environments (GPU TEEs) address this by combining hardware-enforced memory encryption within the GPU, a protected VRAM region inaccessible from the host, and a remote attestation path that allows a verifier to confirm the GPU is genuine and operating in the correct security mode before transmitting sensitive data. The NVIDIA H100 (Hopper) is the first GPU to deliver this via its Confidential Computing (CC) mode, using AES-GCM-encrypted bounce buffers for host-GPU data transfer, a Compute Protected Region (CPR) carved from HBM2e, and SPDM-based attestation against NVIDIA's certificate chain. AMD is extending SEV-SNP toward GPU workloads through the HSP (Host Security Processor) and related platform interfaces. This chapter covers both vendor implementations and the open-source attestation tooling available on Linux.

---

## 2. GPU Process Isolation

### 2.1 Per-Process GPU Virtual Address Spaces

On the CPU side, virtual memory gives every process its own address space backed by the MMU. GPUs replicate this architecture on the device. Each rendering or compute context is associated with a per-process GPU page table (GPUVA), so a process's GPU virtual addresses are translated by the GPU's own memory management unit (iommu on the device side, often called the GMMU or GPU MMU) before they reach physical VRAM addresses. A bug in the translation tables — a missing entry, a protection bit left off, or an uninitialized page — is required before one process can access another's VRAM via the GPU's own address translation.

The DRM subsystem exposes this through the `DRM_GPUVM` framework. A `drm_gem_object` carries a `gpuva` list of all GPU-virtual-address mappings pointing to that object, protected by either the GEM reservation lock or the driver's own mutex depending on whether the driver opts into `DRM_GPUVM_IMMEDIATE_MODE` [Source](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html). The GPU MMU context is torn down in the `drm_file` release path when the last file descriptor referencing the GPU context is closed.

```mermaid
graph TD
    GEM["drm_gem_object"]
    GPUVA["gpuva list\n(GPU-virtual-address mappings)"]
    LOCK["GEM reservation lock\nor driver mutex"]
    MODE["DRM_GPUVM_IMMEDIATE_MODE"]
    FILE["drm_file\n(GPU context)"]
    TEARDOWN["GPU MMU context teardown\n(on last fd close)"]

    GEM --> GPUVA
    GPUVA -- "protected by" --> LOCK
    MODE -- "determines lock type" --> LOCK
    FILE --> TEARDOWN
    TEARDOWN -- "unmaps" --> GPUVA
```

### 2.2 What Happens When a Process Exits

Context teardown is the most security-sensitive moment in GPU process lifecycle. The driver must:

1. Stop all pending command ring submissions from the dying context.
2. Wait for any in-flight GPU commands to drain or forcibly reset the engine.
3. Unmap all GPU page table entries referencing the dead context's memory.
4. Return VRAM allocations to the free pool — but crucially, ensure those pages cannot be read by the next allocator.

Step 4 is where VRAM scrubbing matters. DRAM (both host DRAM and GPU VRAM) retains its contents after deallocation. Without explicit zeroing, the next process to allocate a page will inherit the previous tenant's data. In the CPU world the kernel zeroes pages before handing them to userspace (`clear_page`); the GPU world has historically been weaker.

Research published by Barrack AI in 2024 confirmed that CUDA's default allocator (`cudaMalloc`) does not scrub VRAM between allocations, meaning tenants in a shared GPU cluster can sometimes read prior tenants' residues [Source](https://blog.barrack.ai/nvidia-cuda-never-clears-gpu-memory/). The Linux `nvidia-open` kernel driver and Nouveau do not currently implement VRAM zeroing on context exit at the kernel level (Note: needs verification for the latest open-driver releases).

### 2.3 AMDGPU Cleaner Shader

AMD implemented an architectural solution starting with CDNA2/GFX9.4.2 and extending through GFX11.5. The **cleaner shader** is a small GPU program that the kernel driver schedules after every process's last job completes, clearing:

- **LDS** (Local Data Store) — 32–64 KB of per-workgroup scratchpad
- **VGPRs** (Vector General Purpose Registers) — up to 256 × 32-bit per lane
- **SGPRs** (Scalar General Purpose Registers)

The driver exposes two sysfs interfaces under the GPU device node [Source](https://docs.kernel.org/gpu/amdgpu/process-isolation.html):

```bash
# Enable automatic isolation (serialize engine access + run cleaner shader between jobs)
echo 1 > /sys/bus/pci/devices/<BDF>/enforce_isolation

# Manually trigger the cleaner shader (useful at user logout)
echo 0 > /sys/bus/pci/devices/<BDF>/run_cleaner_shader
```

On multi-partition GPUs (SPX/DPX/QPX modes) each field accepts space-separated values, one per partition. The `enforce_isolation` interface serializes access to the graphics engine and schedules the cleaner shader automatically at context boundaries. A login manager such as GDM can call `run_cleaner_shader` on user logout to guarantee register hygiene before the next user's session.

The cleaner shader patches shipped via the `[PATCH v2 2/2] drm/amdgpu/gfx9: Enable Cleaner Shader for GFX9.4.2 Hardware` series on the amd-gfx mailing list [Source](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg114259.html) and expanded with GFX11.5 support in 2025 [Source](https://lists.freedesktop.org/archives/amd-gfx/2025-March/121647.html).

---

## 3. IOMMU and DMA Attack Mitigation

### 3.1 How the IOMMU Protects GPU DMA

Without an Input-Output Memory Management Unit (IOMMU), a GPU that can initiate DMA has unrestricted access to host physical memory — making it a potential tool for bypassing the CPU MMU entirely. The IOMMU interposes on DMA transactions and enforces a device-visible page table (an IOMMU domain) that restricts which host physical pages a device can access. A driver bug that lets a GPU or its attacker-controlled command stream issue DMA to an arbitrary physical address is blocked at the IOMMU boundary.

On x86 the two IOMMU implementations are **Intel VT-d** (Virtualisation Technology for Directed I/O) and **AMD-Vi** (AMD I/O Virtualisation). On ARM systems the **ARM SMMU** (System Memory Management Unit) provides equivalent isolation. The kernel's unified IOMMU API lives in `include/linux/iommu.h`.

```mermaid
graph LR
    GPU["GPU\n(DMA initiator)"]
    IOMMU["IOMMU\n(Intel VT-d / AMD-Vi / ARM SMMU)"]
    DOMAIN["IOMMU domain\n(device-visible page table)"]
    HOST["Host physical memory"]
    GROUP["IOMMU group\n(iommu_group_get)"]
    ACS["ACS\n(Access Control Services)"]

    GPU -- "DMA transactions" --> IOMMU
    IOMMU -- "enforces" --> DOMAIN
    DOMAIN -- "restricts access to" --> HOST
    GPU --> GROUP
    ACS -- "enables per-function\nisolation in" --> GROUP
```

The relevant current API includes:

```c
/* Allocate a paging-capable IOMMU domain for a device */
struct iommu_domain *iommu_paging_domain_alloc_flags(struct device *dev,
                                                      unsigned int flags);

/* Attach the domain to a device, so DMA from that device goes through it */
extern int iommu_attach_device(struct iommu_domain *domain,
                               struct device *dev);

/* Get the IOMMU group a device belongs to */
extern struct iommu_group *iommu_group_get(struct device *dev);
```

[Source: `include/linux/iommu.h`, torvalds/linux HEAD](https://github.com/torvalds/linux/blob/master/include/linux/iommu.h)

Note that `iommu_domain_alloc()` was replaced by `iommu_paging_domain_alloc_flags()` in recent kernel releases; downstream code still using the old API should be updated.

### 3.2 IOMMU Groups and ACS

The kernel groups PCIe devices that cannot be reliably isolated from each other into **IOMMU groups**. If two devices share an upstream PCIe bridge that lacks **ACS (Access Control Services)** capability, the IOMMU cannot prevent one device from issuing peer-to-peer DMA to the other — so they are placed in the same group and must be either both assigned to a VM or both kept on the host.

ACS is a PCIe specification feature that enables per-function isolation at the switch or root-port level. On systems where a GPU shares an upstream port with other devices and that port lacks ACS, the GPU and its neighbours cannot be cleanly separated for secure passthrough. The workaround, `pcie_acs_override=downstream,multifunction`, forces the kernel to treat certain ports as isolated — but this reduces IOMMU security guarantees and should not be used in production security-sensitive deployments [Source](https://github.com/benbaker76/linux-acs-override).

### 3.3 VFIO for GPU Passthrough

**VFIO** (Virtual Function I/O) is the Linux framework for assigning PCIe devices directly to virtual machines while maintaining host-kernel safety. VFIO requires an operational IOMMU and uses it to enforce DMA isolation. The sequence for GPU passthrough:

```bash
# Load VFIO driver
modprobe vfio-pci

# Bind the GPU to vfio-pci instead of the native driver
echo "10de 2684" > /sys/bus/pci/drivers/vfio-pci/new_id

# Pass the IOMMU group through to a QEMU VM
qemu-system-x86_64 -device vfio-pci,host=01:00.0 ...
```

Without a properly scoped IOMMU group, VFIO refuses to expose the device — protecting the host from a compromised guest that attempts to escape via GPU DMA.

---

## 4. HDCP 2.2 — Content Protection in the Kernel

### 4.1 Overview

**HDCP (High-bandwidth Digital Content Protection)** is the cryptographic protocol used to authenticate display sinks (monitors, TV panels, AV receivers) before transmitting protected premium video content. HDCP is required by content owners for 4K Ultra HD and for "Type 1" restricted streams that the HDCP specification limits to HDCP 2.2-capable paths only.

The Linux kernel implements HDCP primarily in:

- `drivers/gpu/drm/display/drm_hdcp_helper.c` — generic HDCP helper library
- `include/drm/display/drm_hdcp.h` — protocol constants and message structures
- `include/drm/display/drm_hdcp_helper.h` — helper API declarations
- `drivers/gpu/drm/i915/display/intel_hdcp.c` — Intel i915 implementation
- `drivers/gpu/drm/xe/display/xe_hdcp_gsc.c` — Intel Xe implementation
- `drivers/gpu/drm/amd/display/modules/hdcp/` — AMD display HDCP implementation

[Source: `drivers/gpu/drm/display/drm_hdcp_helper.c`, torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_hdcp_helper.c)

### 4.2 HDCP Protocol Versions

**HDCP 1.4** uses a 56-bit symmetric key protocol over DDC (I²C). The transmitter sends its key selection vector (AKSV) and a random nonce (An); the receiver responds with its BKSV and a challenge response. The HDCP 1.4 SRM (System Renewability Message) is a signed revocation list allowing the DCP LLC to revoke compromised receivers; the kernel's `drm_hdcp_check_ksvs_revoked()` function validates KSVs against the loaded SRM.

**HDCP 2.2** — the version implemented in the Linux DRM subsystem — uses a more modern authentication exchange. (Note: HDCP 2.3 is a subsequent revision of the specification; the kernel headers use `HDCP_2_2_*` naming throughout and there are no `HDCP_2_3_*` constants upstream. The same message set covers both 2.2 and 2.3 operation.)

- **AKE (Authentication and Key Exchange):** The transmitter reads the receiver's certificate (`HDCP_2_2_AKE_SEND_CERT`) containing the receiver's 1024-bit RSA public key (`HDCP_2_2_K_PUB_RX_MOD_N_LEN` = 128 bytes) and verifies the DCP LLC signature on that certificate (`HDCP_2_2_DCP_LLC_SIG_LEN` = 384 bytes, i.e., an RSA-3072 signature from DCP's Certificate Authority). The transmitter then establishes a shared master key `km`.
- **LC (Locality Check):** The transmitter sends `LC_INIT` with a 64-bit nonce `rn`; the receiver must respond with `L'` within 20 ms — proving physical locality and preventing routing attacks. Key derivation at this stage also incorporates **LC128**, a 128-bit global constant that all DCP-licensed HDCP 2.x devices share, used as an additional input to the key derivation functions. LC128 is obtained through the DCP licensing agreement and is not published publicly.
- **SKE (Session Key Exchange):** Derives the session encryption key `ks` from `km` and `riv`.
- **Repeater propagation:** For HDMI/DP repeaters, the receiver ID list is verified with `HDCP_2_2_REP_SEND_RECVID_LIST` and acknowledged.

```mermaid
graph TD
    subgraph "AKE (Authentication and Key Exchange)"
        AKE1["TX reads HDCP_2_2_AKE_SEND_CERT\n(receiver certificate, 1024-bit RSA pubkey)"]
        AKE2["TX verifies DCP LLC signature\n(HDCP_2_2_DCP_LLC_SIG_LEN = 384 bytes)"]
        AKE3["TX establishes shared master key km"]
        AKE1 --> AKE2 --> AKE3
    end

    subgraph "LC (Locality Check)"
        LC1["TX sends HDCP_2_2_LC_INIT\n(64-bit nonce rn)"]
        LC2["RX responds with L'\n(within 20 ms)"]
        LC3["Key derivation with LC128"]
        LC1 --> LC2 --> LC3
    end

    subgraph "SKE (Session Key Exchange)"
        SKE1["Derives session key ks\n(from km and riv)"]
    end

    subgraph "Repeater propagation"
        REP1["Verify HDCP_2_2_REP_SEND_RECVID_LIST"]
    end

    AKE3 --> LC1
    LC3 --> SKE1
    SKE1 --> REP1
```

The kernel defines the complete message ID table in `include/drm/display/drm_hdcp.h` [Source](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_hdcp.h):

```c
/* Protocol message definition for HDCP2.2 specification */
#define HDCP_2_2_AKE_INIT          2
#define HDCP_2_2_AKE_SEND_CERT    3
#define HDCP_2_2_AKE_NO_STORED_KM 4
#define HDCP_2_2_AKE_STORED_KM    5
#define HDCP_2_2_AKE_SEND_HPRIME  7
#define HDCP_2_2_LC_INIT           9
#define HDCP_2_2_LC_SEND_LPRIME   10
#define HDCP_2_2_SKE_SEND_EKS     11
```

### 4.3 Content Type: Type 0 vs. Type 1

The kernel introduces a **content type** property on HDCP 2.2 connections [Source](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_hdcp.h):

```c
/*
 * Protected content streams are classified into 2 types:
 * - Type0: Can be transmitted with HDCP 1.4+
 * - Type1: Can be transmitted with HDCP 2.2+
 */
#define HDCP_STREAM_TYPE0  0x00
#define HDCP_STREAM_TYPE1  0x01

#define DRM_MODE_HDCP_CONTENT_TYPE0  0
#define DRM_MODE_HDCP_CONTENT_TYPE1  1
```

Type 1 content — used for premium 4K titles — must only flow over HDCP 2.2-authenticated links. Even if a display supports HDCP 1.4, a Type 1 stream cannot be sent to it. The compositor and media pipeline are responsible for tagging content streams with the appropriate type.

### 4.4 DRM Content Protection Property

The connector-level API uses three UAPIs defined in `include/uapi/drm/drm_mode.h`:

```c
/* Content Protection Flags */
#define DRM_MODE_CONTENT_PROTECTION_UNDESIRED  0
#define DRM_MODE_CONTENT_PROTECTION_DESIRED    1
#define DRM_MODE_CONTENT_PROTECTION_ENABLED    2
```

A media player sets the "Content Protection" connector property to `DESIRED`; the driver's HDCP implementation completes authentication and transitions it to `ENABLED`. The `drm_connector_attach_content_protection_property()` function from `drm_hdcp_helper.h` registers this property on a connector:

```c
int drm_connector_attach_content_protection_property(
        struct drm_connector *connector,
        bool hdcp_content_type);
```

The `hdcp_content_type` flag, when true, also registers the "HDCP Content Type" property allowing userspace to select between Type 0 and Type 1.

### 4.5 Intel i915/Xe Implementation

Intel's HDCP support routes HDCP 2.2 authentication through the **GSC (Graphics Security Controller)**, a dedicated security microcontroller present on DG2 (Alchemist) and later. The file `drivers/gpu/drm/i915/display/intel_hdcp_gsc_message.c` manages the message exchange between the display driver and GSC over the GSCCS (Graphics Security Controller Command Streamer) engine. For older platforms (Gen10–Gen12), authentication passed through the **MEI (Management Engine Interface)** via `drivers/misc/mei/hdcp/mei_hdcp.c` [Source: `Documentation/driver-api/mei/hdcp.rst`](https://github.com/torvalds/linux/blob/master/Documentation/driver-api/mei/hdcp.rst).

A NULL pointer dereference in Intel i915's HDCP 2.x capability query path was assigned CVE-2024-53050 and fixed in the 6.12 stable series [Source: NVD CVE-2024-53050](https://nvd.nist.gov/vuln/detail/CVE-2024-53050).

---

## 5. Secure Display Path

### 5.1 Protected Content and the Compositor

When HDCP 2.2 Type 1 authentication is active, the display pipeline from GPU framebuffer to panel must be completely sealed. No intermediate stage — including the Wayland compositor — should be able to capture or redirect the pixel data. This creates a fundamental tension: the compositor is the entity that composites all windows, so it has structural access to every surface unless the hardware enforces an exception.

The solution used on Intel platforms is **PXP (Protected Xe Path)**, a hardware feature on Gen12+ that creates a separate encrypted path from the GPU's render engine to the display engine for protected content. PXP-enabled framebuffers are marked at allocation time and the hardware ensures they cannot be accessed by unprivileged command streams. The Intel Xe driver exposes this through `xe_hdcp_gsc.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/xe/display/xe_hdcp_gsc.c).

```mermaid
graph LR
    RENDER["GPU render engine\n(Gen12+)"]
    PXP["PXP\n(Protected Xe Path)\nencrypted path"]
    DISPLAY["Display engine"]
    FB["PXP-enabled framebuffer\n(marked at allocation time)"]
    UNPRIV["Unprivileged command streams"]
    PANEL["Panel / HDCP 2.2 sink"]

    FB --> RENDER
    RENDER -- "encrypted path via" --> PXP
    PXP --> DISPLAY
    DISPLAY --> PANEL
    UNPRIV -. "hardware blocks access" .-> FB
```

At the DRM level, "protected" framebuffers are not currently a generic kernel feature with a universal `FLAG_PROTECTED` flag — the protection is vendor-specific (Intel PXP, AMD Display Core's protected surface support). The compositor is expected to honour the protection by not compositing protected surfaces through unprotected paths.

### 5.2 Wayland Content Protection Protocol

The Wayland protocol extension for content protection provides a minimal API: a client requests protection for one of its surfaces and the compositor replies with a status event [Source](https://wayland.app/protocols/weston-content-protection):

```
zwp_linux_content_type_manager_v1 → zwp_content_type_v1
  protected_surface → status { disabled | enabled }
```

The compositor (trusted) decides where each surface appears. If all connected displays support the requested HDCP content type, the compositor sets HDCP up on those outputs and reports `enabled`. If any display on the output chain cannot authenticate, the compositor must either hide the protected surface or blank it.

The practical limitation is that this model trusts the compositor entirely. On traditional desktop systems where the compositor is owned by the logged-in user, this is acceptable. On set-top-box or kiosk deployments the compositor is a fixed trusted component and the model works correctly. Mixed untrusted compositor scenarios (malicious compositor, remote compositor) remain out of scope for this protocol.

---

## 6. GPU Firmware Signing and Secure Boot

### 6.1 NVIDIA GSP-RM Firmware

The NVIDIA open-source kernel driver (`nvidia-open`) loads the GPU System Processor firmware (`gsp.bin`) from the standard Linux firmware path (`/lib/firmware/nvidia/<chip>/`). The Nouveau driver uses `nvkm_firmware_get()` to locate and load firmware blobs [Source: `drivers/gpu/drm/nouveau/nvkm/core/firmware.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/nouveau/nvkm/core/firmware.c):

```c
/**
 * nvkm_firmware_get - load firmware from the official nvidia/chip/ directory
 * @subdev:  subdevice that will use that firmware
 * @fwname:  name of firmware file to load
 * @ver:     firmware version to load
 * @fw:      firmware structure to load to
 *
 * Use this function to load firmware files in the form nvidia/chip/fwname.bin.
 * Firmware files released by NVIDIA will always follow this format.
 */
int
nvkm_firmware_get(const struct nvkm_subdev *subdev, const char *fwname,
                  int ver, const struct firmware **fw)
```

The firmware itself is PKC (public-key cryptography) signed by NVIDIA; the GPU's Boot ROM validates the signature before granting the GSP execution privilege. (The exact padding scheme — PKCS#1 v1.5 vs PSS — has not been confirmed from public sources and needs verification.) The NVIDIA ACR (Access Control Region) subsystem in Nouveau (`nvkm/subdev/acr/`) manages the privilege levels for Light-Secured (LS) and Heavy-Secured (HS) Falcon microcontrollers. HS Falcons run code signed with NVIDIA's production key and execute at elevated Hardware-Secured privilege; LS Falcons run inside a WPR (Write-Protected Region) enforced by the HS ACR firmware at lower privilege. Drivers receive only LS-signed blobs; the GPU BootROM and SEC2 engine maintain the separation [Source: `include/nvkm/subdev/acr.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/nouveau/include/nvkm/subdev/acr.h). The Nova (Rust NVIDIA driver) documentation describes the HS/LS privilege model from an architectural standpoint [Source: Ch10 — Nova].

```mermaid
graph TD
    BOOTROM["GPU BootROM"]
    SEC2["SEC2 engine"]
    ACR["ACR\n(Access Control Region)\nnvkm/subdev/acr/"]
    HS["HS Falcon\n(Heavy-Secured)\nNVIDIA production key\nHardware-Secured privilege"]
    WPR["WPR\n(Write-Protected Region)"]
    LS["LS Falcon\n(Light-Secured)\nLS-signed blobs only\nlower privilege"]

    BOOTROM --> SEC2
    SEC2 --> ACR
    ACR -- "enforces" --> HS
    ACR -- "enforces" --> WPR
    WPR -- "contains" --> LS
```

### 6.2 Intel GuC and HuC Firmware

Intel's **GuC** (Graphics Microcontroller) handles GPU scheduling and power management; **HuC** (HEVC Microcontroller) accelerates video encode/decode. Both are loaded by `intel_uc_fw_fetch()` in the i915/Xe drivers [Source: `drivers/gpu/drm/i915/gt/uc/intel_uc_fw.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/gt/uc/intel_uc_fw.c):

```c
int intel_uc_fw_fetch(struct intel_uc_fw *uc_fw)
{
    struct intel_gt *gt = __uc_fw_to_gt(uc_fw);
    ...
    err = try_firmware_load(uc_fw, &fw);
    ...
    while (err == -ENOENT) { /* try older versions */ }
```

Both firmware blobs carry a Command Security System (CSS) header containing the firmware's signature; the GPU's hardware boot validation path uses this header to authenticate the firmware before execution. (The precise details of the CSS verification mechanism have not been confirmed from public kernel documentation and need further verification against Intel's firmware signing specifications.) Failure to authenticate results in GuC/HuC being disabled and GuC-based scheduling falling back to host scheduling. HuC specifically requires PXP support on DG2 and later, meaning the Intel Management Engine must be present and operational.

### 6.3 AMD Platform Security Processor (PSP)

The AMD **PSP** (Platform Security Processor, now "AMD Secure Processor") is an ARMv7 Cortex-A5 core embedded in every AMD APU and GPU. It owns the root of trust for the entire SoC. Its functions in the GPU security context include:

- Validating all GPU firmware blobs against AMD's ARK (AMD Root Key) certificate chain before allowing them to execute on the GPU's internal processors.
- Mediating SEV (Secure Encrypted Virtualization) key operations, including SEV-SNP page state management.
- Providing HDCP authentication services on AMD display hardware via `drivers/gpu/drm/amd/display/modules/hdcp/hdcp_psp.c`.

The firmware signing chain: ARK (2048-bit RSA) → ASK (AMD SEV Signing Key) → CEK (Chip Endorsement Key, per-device ECDSA) [Source](https://dayzerosec.com/blog/2023/04/17/reversing-the-amd-secure-processor-psp.html). GPU firmware files are compressed, signed, and for recent revisions also encrypted. The PSP 14.0 block added for RDNA4 continues this pattern [Source](https://www.phoronix.com/news/AMDGPU-Enabling-PSP-14.0).

```mermaid
graph TD
    ARK["ARK\n(AMD Root Key)\n2048-bit RSA"]
    ASK["ASK\n(AMD SEV Signing Key)"]
    CEK["CEK\n(Chip Endorsement Key)\nper-device ECDSA"]
    GPUFW["GPU firmware blobs\n(compressed, signed, encrypted)"]

    ARK -- "signs" --> ASK
    ASK -- "signs" --> CEK
    CEK -- "signs attestation for" --> GPUFW
```

### 6.4 UEFI Secure Boot and Kernel Module Signing

On a UEFI Secure Boot system the GPU kernel module (`nvidia.ko`, `amdgpu.ko`, `i915.ko`) must be signed by a key enrolled in the MOK (Machine Owner Key) database or the distribution's signing infrastructure. An unsigned module will be rejected by the kernel when `CONFIG_MODULE_SIG_FORCE=y` is set. Distributions such as RHEL and Ubuntu pre-sign the GPU modules with their distribution key; out-of-tree proprietary modules require users to enroll their own key with `mokutil`. The firmware blobs themselves are not subject to UEFI Secure Boot enforcement — they are validated by the GPU hardware or PSP independently.

---

## 7. AMD SEV-SNP and GPU Confidential Computing

### 7.1 SEV-SNP Background

**AMD SEV** (Secure Encrypted Virtualization) encrypts virtual machine memory using AES-128 with a per-VM key managed by the PSP. **SEV-SNP** (Secure Nested Paging) adds integrity protection, preventing a malicious hypervisor from injecting plaintext pages into the encrypted VM's address space. The guest attestation report, signed by the CEK, allows a remote party to verify the VM's identity and launch configuration.

### 7.2 GPU Workloads in a SEV-SNP VM

When a confidential VM (CVM) running under SEV-SNP wants to use a GPU, the interaction model depends on whether the GPU supports confidential computing mode itself. For legacy GPUs — or current non-CC-enabled configurations — the GPU is treated as an untrusted device:

- DMA buffers shared between the GPU driver and the CVM must be unencrypted (marked `C=0` in the SEV page attribute table) to allow the GPU to read/write them. This means data leaving the CVM for the GPU crosses a trust boundary.
- The CPU TEE boundary does not extend to the GPU's internal memory.

AMD is developing mechanisms to pass encrypted DMA pages to CC-capable GPUs, extending the trust domain. This work involves changes to the AMDGPU driver, the ROCm stack, and the PSP firmware to negotiate page encryption attributes. As of mid-2026 this work is still in progress for the open-source stack (Note: specific kernel commit references need verification against the upstream amd-gfx mailing list).

### 7.3 HSMP and SMI Transfer

The **HSMP** (Host System Management Port) interface allows privileged software to query and configure AMD processor features — including certain power and thermal parameters. **SMI Transfer** (System Management Interface Transfer) is used by the PSP to communicate with firmware components across isolation boundaries. These interfaces are relevant to confidential computing because a hypervisor that can manipulate HSMP or intercept SMI transfers can potentially interfere with SEV-SNP operations; mitigations are part of the broader AMD SNP-enabled host kernel patches. (Note: specific mitigation details in the Linux kernel need further verification against AMD's SEV-SNP host support patches in `arch/x86/kvm/svm/`.)

---

## 8. NVIDIA H100 Confidential Computing

### 8.1 The Hopper CC Model

The NVIDIA H100 (Hopper architecture) is the first GPU to provide a hardware-enforced Trusted Execution Environment (TEE). When operated in **CC-On** mode, the GPU partitions its memory into:

- **CPR (Compute Protected Region):** GPU-local HBM2e memory that is inaccessible from outside the GPU die. Code and data executing inside the TEE reside here.
- **Shared (unprotected) memory:** A shared region in system DRAM used for CPU-GPU communication, secured by encryption.

Three CC operational modes exist in the H100 driver:

| Mode | Description |
|------|-------------|
| **CC-Off** | Standard GPU operation; no TEE, full performance |
| **CC-On** | Full confidential computing; hardware firewalls active; performance counters disabled |
| **CC-DevTools** | Partial CC mode with security disabled; allows profiling for developer use |

[Source: "Confidential Computing on H100 GPUs for Secure and Trustworthy AI", NVIDIA Developer Blog](https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/)

### 8.2 Memory Protection and Bounce Buffers

The H100 in CC-On mode **does not encrypt its on-card HBM memory at rest** — the CPR isolation is enforced by hardware access control rather than encryption [Source](https://www.edgeless.systems/wiki/hardware/nvidia-hopper-h100). This distinguishes it from CPU-side SEV which encrypts DRAM. Instead:

- When the CPU's confidential VM writes data destined for the GPU (model inputs, activations), it encrypts the data using **AES-GCM** with a session key and places it in an **encrypted bounce buffer** in the shared memory region.
- The GPU reads from the bounce buffer, decrypts into the CPR, and processes the data entirely within the protected region.
- Results are encrypted with AES-GCM before leaving the CPR back through the shared bounce buffer.

```mermaid
graph LR
    CVM["CPU confidential VM\n(CVM)"]
    BB["encrypted bounce buffer\n(shared memory region)"]
    CPR["CPR\n(Compute Protected Region)\nGPU-local HBM2e"]

    CVM -- "AES-GCM encrypt\n(session key)" --> BB
    BB -- "GPU reads,\ndecrypts into" --> CPR
    CPR -- "AES-GCM encrypt\nresults" --> BB
    BB -- "CVM reads\ndecrypted results" --> CVM
```

The AES-GCM implementation conforms to NIST SP800-38D. The session key is established via the SPDM (Security Protocol and Data Model) protocol exchange.

### 8.3 SPDM Attestation and the ECC-384 Root of Trust

Every H100 GPU contains a **device-unique ECC-384 key pair** whose private component is burned into on-die fuses at production time. NVIDIA's CA issues a certificate binding the public key to the GPU's device identity. During confidential computing initialisation:

1. The GPU measures its firmware through a secure boot sequence — each stage's hash is extended into an on-die measurement register.
2. The GPU driver (within the CPU's TEE) opens an **SPDM session** to the GPU. SPDM is a DMTF standard for device authentication; the GPU signs its firmware measurements with its private ECC-384 key.
3. The driver verifies the SPDM attestation report against NVIDIA's certificate chain, establishing that the correct, unmodified firmware is running.
4. The SPDM session derives a symmetric session key used for all subsequent AES-GCM-encrypted bounce buffer traffic.

For remote attestation, the CUDA driver within the CVM can forward the attestation report to **NRAS (NVIDIA Remote Attestation Service)** for online verification, or perform offline verification in air-gapped environments [Source](https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/). The open-source tools for this are in the **nvtrust** repository on GitHub [Source](https://github.com/NVIDIA/nvtrust).

### 8.4 CUDA Device Attributes and CC Capability Probing

CUDA exposes device attributes that applications can query at runtime to determine GPU capabilities. One attribute relevant to shared-memory GPU workloads is `CU_DEVICE_ATTRIBUTE_GPU_DIRECT_RDMA_WITH_CUDA_VMM_SUPPORTED`, which reports whether the GPU supports GPUDirect RDMA combined with CUDA Virtual Memory Management. This is distinct from confidential computing: it relates to peer-to-peer RDMA access to GPU memory using the CUDA VMM API, not the H100 TEE mechanism. (Note: the relationship between this attribute and CC-mode operation has not been confirmed in public NVIDIA documentation; its presence in a CC context likely indicates a capability check for memory management interoperability, not a gate to enabling CC mode.)

### 8.5 Performance Overhead

Benchmark research (USENIX adjacent, arXiv:2409.03992, 2024) shows that TEE overhead for large language model inference typically remains below 5%, because the GPU's internal HBM computation is unaffected. Overhead concentrates in CPU-GPU I/O — particularly the encrypt/decrypt overhead on the bounce buffer path over PCIe. For I/O-bound workloads (small batch sizes, frequent host-device transfers) overhead can be higher.

In CC-On mode the H100 **disables GPU performance counters** to prevent side-channel information leakage through counter telemetry [Source](https://www.edgeless.systems/wiki/hardware/nvidia-hopper-h100). This means standard CUDA profiling tools (`nsight-compute`, `nvperf`) cannot profile CC-On workloads. The CC-DevTools mode re-enables counters at the cost of removing security guarantees.

---

## 9. GPU Side-Channel Attacks

### 9.1 Cache and Timing Attacks

GPU caches — L1/L2 data caches, the texture cache, the L2 shared between SMs — exhibit timing behaviour that can leak information across security boundaries. Classic Prime+Probe and Flush+Reload attacks assume shared physical memory and the ability to measure access latency; GPUs with shared L2 caches in multi-process mode are susceptible to similar attacks. Published research from 2016 onward demonstrated that GPU cache timing can leak AES round keys from co-located GPGPU workloads.

### 9.2 GPUHammer: Rowhammer on GDDR6

**Rowhammer** is a DRAM vulnerability where repeatedly reading the same memory row causes bit flips in adjacent rows due to electrical coupling. It was believed to apply only to CPU DRAM, but in 2025 researchers from the University of Toronto published **GPUHammer**, the first demonstrated Rowhammer attack on discrete GPU GDDR6 memory [Source: "GPUHammer: Rowhammer Attacks on GPU Memories are Practical", USENIX Security 2025](https://www.usenix.org/conference/usenixsecurity25/presentation/lin-shaopeng).

Key findings:

- GPUHammer achieves up to **8 bit-flips** across 4 DRAM banks on an NVIDIA A6000 with GDDR6 memory.
- The attack is mounted via standard CUDA compute kernels without elevated privilege.
- The primary demonstrated impact is **AI model accuracy degradation**: an image classification model's accuracy dropped from 80% to under 1% after targeted bit-flip injection into model weights.
- In shared GPU cloud environments, a malicious tenant can exploit GPUHammer to tamper with another tenant's model parameters.
- GDDR6's proprietary bank/row address mapping and refresh mitigations make the attack more complex than LPDDR4 Rowhammer but not impractical.

[Source: GPUHammer paper PDF](https://gururaj-s.github.io/assets/pdf/SEC25_GPUHammer.pdf)

### 9.3 Power Side Channels

Intel's RAPL (Running Average Power Limit) interface on CPUs has been exploited for cross-process AES key recovery (Platypus, 2020). NVIDIA does not expose a RAPL equivalent, but the `NVML` library's `nvmlDeviceGetPowerUsage()` call returns per-device wattage at ~10 Hz resolution — potentially usable for coarse power analysis in long-running workloads. AMD's ROCm `rsmi_dev_power_ave_get()` provides similar resolution. Whether this is practically exploitable for GPU-side key recovery from shared workloads has not been established in published research (Note: needs verification against forthcoming academic work).

### 9.4 Mitigations in the Linux Stack

**Performance counter access restriction.** The kernel's `kernel.perf_event_paranoid` sysctl limits which users can open perf events:

```bash
# Value 2: Only privileged users can access CPU perf counters
echo 2 > /proc/sys/kernel/perf_event_paranoid
```

For GPU-specific counters, each driver implements its own access model. AMDGPU exposes hardware performance counters through `amdgpu_perfcnt_ioctl`, restricted to root or DRM-master by default. Intel i915/Xe exposes OA (Observation Architecture) counters via `perf_open` with driver-side checks against the process's capabilities.

NVIDIA H100 in CC-On mode disables GPU performance counters entirely — this is a deliberate architectural decision rather than a software mitigation. On non-CC GPUs, NVIDIA's proprietary driver restricts raw hardware counter access to the process that owns the GPU context.

**ECC.** Hardware ECC on GPU DRAM (present on A100, H100, and AMD MI300 datacenter parts) corrects single-bit errors and detects double-bit errors, partially mitigating Rowhammer bit-flip injection — though sufficiently many flips or attacks targeting the ECC syndrome bits themselves may still succeed.

---

## 10. Driver Security Hardening

### 10.1 DRM ioctl Permission Model

The DRM subsystem controls access to sensitive operations through per-ioctl permission flags defined in `include/drm/drm_ioctl.h` [Source](https://github.com/torvalds/linux/blob/master/include/drm/drm_ioctl.h):

```c
/**
 * DRM_AUTH — for ioctl used for rendering; requires authenticated fd
 * or render node.
 */
#define DRM_AUTH        BIT(0)

/**
 * DRM_MASTER — must be set for any ioctl which can change modeset
 * or display state; requires active DRM master.
 */
#define DRM_MASTER      BIT(1)

/**
 * DRM_ROOT_ONLY — requires CAP_SYS_ADMIN; used for SETMASTER/DROPMASTER.
 */
#define DRM_ROOT_ONLY   BIT(2)

/**
 * DRM_RENDER_ALLOW — for render-only ioctl on drivers with render nodes.
 * Should be set alongside DRM_AUTH for all new render drivers.
 */
#define DRM_RENDER_ALLOW BIT(5)
```

The separation of **render nodes** (`/dev/dri/renderD*`) from **primary nodes** (`/dev/dri/card*`) is a key security boundary:

- Render nodes expose only rendering ioctl (those with `DRM_RENDER_ALLOW`). No modesetting, no master operations.
- Applications that only need GPU compute or rendering — Vulkan apps, CUDA, ROCm — should open the render node and never need access to the primary node.
- The primary node, which can change display state, should only be opened by the compositor.

```mermaid
graph TD
    subgraph "Render node (/dev/dri/renderD*)"
        RN["DRM_RENDER_ALLOW\nioctl only\nno modesetting"]
    end
    subgraph "Primary node (/dev/dri/card*)"
        PN["DRM_MASTER\nDRM_ROOT_ONLY\nDRM_AUTH\nmodesetting + display state"]
    end

    APPS["Vulkan apps / CUDA / ROCm"] --> RN
    COMPOSITOR["Compositor\n(trusted)"] --> PN
    COMPOSITOR -. "also opens" .-> RN
```

This separation prevents a compromised application from issuing a `DRM_IOCTL_SET_MASTER` and interfering with the display pipeline.

### 10.2 Fuzzing the DRM Subsystem

**syzkaller** is the kernel fuzzer developed at Google that has found thousands of kernel security bugs. Its GPU coverage includes the DRM core ioctl dispatch path, the GEM buffer management, and driver-specific ioctl handlers. Syzkaller generates random sequences of DRM ioctl calls with semi-valid arguments — exercising error paths, integer overflow checks, and NULL dereference guards that developers often overlook.

Notable areas that syzkaller and manual auditing have historically caught in GPU drivers include:

- **Integer overflow in buffer size arithmetic** — when `width * height * bpp` overflows 32 bits before bounds checks.
- **Use-after-free in fence signalling** — DRM fences are reference-counted objects; incorrect reference handling creates UAF windows.
- **Uninitialized memory in ioctl output structures** — kernel data leaking to userspace if padding bytes are not zeroed before copyout (`copy_to_user`).
- **Missing capability checks on privileged ioctl** — allowing unprivileged processes to change display parameters or access VRAM apertures.

CVE-2024-53050 (i915 HDCP NULL pointer dereference) is a recent example caught through code review of the HDCP 2.x path [Source: NVD CVE-2024-53050](https://nvd.nist.gov/vuln/detail/CVE-2024-53050).

### 10.3 The drm_ioctl Dispatch and Capability Checks

The DRM core dispatches ioctl through `drm_ioctl()` in `drivers/gpu/drm/drm_ioctl.c`, which checks the permission flags before calling the driver-specific handler:

```c
/* Simplified from drm_ioctl.c */
if (ioctl->flags & DRM_ROOT_ONLY && !capable(CAP_SYS_ADMIN))
    return -EACCES;

if (ioctl->flags & DRM_MASTER && !drm_is_current_master(file_priv))
    return -EACCES;

if (ioctl->flags & DRM_AUTH && !file_priv->authenticated)
    return -EACCES;
```

Driver-specific ioctl handlers (e.g., `amdgpu_info_ioctl`, `i915_gem_execbuffer2_ioctl`) add further validation: buffer handle validity, size range checks, and alignment requirements.

### 10.4 amdgpu CVE History

The amdgpu driver has accumulated a number of CVEs related to bounds checking in media accelerator paths (VCN video decode/encode), particularly in the argument validation for VCPU job submission. These vulnerabilities can result in out-of-bounds reads (kernel data leak) or, in some cases, privilege escalation if write primitives are available. The pattern is consistent: ioctls that accept user-provided array sizes and indices must validate every field before use. The AMDGPU KFD (Kernel Fusion Driver, the HPC/ML path) has a distinct ioctl surface and has also been the subject of VRAM attribute exposure vulnerabilities.

---

## 11. eBPF for GPU Access Control and Security Monitoring

The DRM subsystem's ioctl permission flags and render-node separation (Section 10) are necessary but not sufficient for fine-grained GPU access policy. They operate at the kernel's discretionary access control (DAC) level — anyone in the `render` group can open any render node. eBPF provides a second layer: programmable, per-process, in-kernel enforcement without requiring kernel patches, loadable modules, or daemon cooperation.

### 11.1 BPF LSM Hooks on DRM Device Opens

Linux 5.7 introduced **BPF LSM** — the ability to attach eBPF programs to Linux Security Module hooks [Source: `Documentation/bpf/prog_lsm.rst`](https://github.com/torvalds/linux/blob/master/Documentation/bpf/prog_lsm.rst). Enabling it requires `CONFIG_BPF_LSM=y` and `lsm=bpf` (or `lsm=...,bpf`) in the kernel boot parameters. Once active, BPF programs can enforce mandatory access control decisions on any LSM hook — including `file_open`, which fires every time a process opens a file descriptor.

For GPU access control the relevant moment is the `open(2)` of `/dev/dri/renderD128` (or any higher render-device minor). DRM_MAJOR is 226; render nodes use minors ≥ 128 [Source: `include/drm/drm_ioctl.h`, torvalds/linux](https://github.com/torvalds/linux/blob/master/include/drm/drm_ioctl.h). Rather than matching the file path as a string (fragile in BPF), the correct approach is to inspect the inode's device number via `file->f_inode->i_rdev`.

The following skeleton (using libbpf's CO-RE) attaches to `lsm/file_open` and denies GPU render-node access from processes outside a designated cgroup:

```c
/* gpu_lsm.bpf.c — deny /dev/dri/renderD* opens outside an allowed cgroup
 * Requires: CONFIG_BPF_LSM=y, kernel boot param lsm=...,bpf
 * Build: clang -O2 -target bpf -D__TARGET_ARCH_x86 -c gpu_lsm.bpf.c -o gpu_lsm.bpf.o
 */
#include "vmlinux.h"                    /* BTF-generated kernel types */
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>

#define DRM_MAJOR    226
#define RENDER_MINOR_BASE 128

/* cgroup id of the allowed GPU container hierarchy — set from userspace */
const volatile __u64 allowed_cgrp_id = 0;

SEC("lsm/file_open")
int BPF_PROG(gpu_access_control, struct file *file, int ret)
{
    /* Pass through if a previous hook already denied */
    if (ret != 0)
        return ret;

    struct inode *inode = BPF_CORE_READ(file, f_inode);
    unsigned int major = MAJOR(BPF_CORE_READ(inode, i_rdev));
    unsigned int minor = MINOR(BPF_CORE_READ(inode, i_rdev));

    /* Only intercept DRM render nodes */
    if (major != DRM_MAJOR || minor < RENDER_MINOR_BASE)
        return 0;

    /* Allow if process belongs to the permitted cgroup */
    struct cgroup *cgrp = bpf_get_current_cgroup_v2();
    if (cgrp && bpf_cgroup_id(cgrp) == allowed_cgrp_id)
        return 0;

    /* Deny all other processes — unprivileged containers, background jobs */
    return -EPERM;
}

char LICENSE[] SEC("license") = "GPL";
```

The userspace loader calls `bpf_map__set_value_size`, reads the target cgroup's id from `/sys/fs/cgroup/<path>/cgroup.id`, and writes it into the `allowed_cgrp_id` global before attaching. The program returns 0 (allow) or a negative errno (deny); the kernel LSM framework calls `security_file_open` which accumulates votes from all loaded LSM modules. BPF LSM hooks are composed: a `bpf` deny overrides SELinux or AppArmor allows unless `CONFIG_SECURITY_LOCKDOWN_LSM` is also active.

> Note: `bpf_get_current_cgroup_v2()` and `bpf_cgroup_id()` are available in kernel 5.14+. On earlier 5.7–5.13 kernels use `bpf_get_current_pid_tgid()` and a UID-keyed BPF map as an alternative policy store.

This approach lets a container runtime or cloud hypervisor enforce GPU access policies — "only containers with a specific cgroup label may open render nodes" — without any DAC group changes or capability grants.

### 11.2 Seccomp-BPF to Restrict DRM Ioctl Surface

`open(2)` controls which processes acquire a GPU file descriptor. Once they have one, seccomp-BPF can restrict which DRM ioctls they may invoke. seccomp filters intercept `ioctl(2)` at the syscall level, matching on the `request` argument (a scalar integer) before the kernel even dispatches the call. Critically, seccomp cannot dereference the `argp` pointer (third argument) — that is a kernel pointer and cannot be safely read from a BPF filter program. This means filtering is at ioctl-number granularity only, not by individual field values inside the ioctl struct.

DRM ioctls use `'d'` (0x64) as their type byte in the Linux `_IOWR`/`_IOW`/`_IOR` encoding [Source: `include/uapi/drm/drm.h`, torvalds/linux](https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/drm/drm.h). The numbers of interest:

| Ioctl | Request nr | Encoding |
|---|---|---|
| `DRM_IOCTL_VERSION` | 0x00 | `DRM_IOWR(0x00, struct drm_version)` |
| `DRM_IOCTL_GET_MAGIC` | 0x02 | `DRM_IOR(0x02, struct drm_auth)` |
| `DRM_IOCTL_MODE_SETCRTC` | 0xA2 | `DRM_IOWR(0xA2, struct drm_mode_crtc)` |
| `DRM_IOCTL_MODE_ADDFB2` | 0xB8 | `DRM_IOWR(0xB8, struct drm_mode_fb_cmd2)` |

Mode-setting ioctls (0xA0–0xD2) should never be called by sandboxed render-only clients. A seccomp-BPF fragment that allowlists only render-node operations:

```c
/* seccomp BPF fragment — allowlist DRM render-node ioctls only
 * Applied to a sandboxed GPU worker process after it has opened its fds.
 * Uses Linux seccomp BPF (not eBPF) — classic BPF instructions.
 */
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <sys/ioctl.h>

/* DRM ioctl type byte */
#define DRM_TYPE   0x64   /* 'd' */
/* nr range that is safe for render-only clients (pre-modesetting ioctls) */
#define DRM_NR_RENDER_MAX  0x3F
/* nr base of modesetting commands — deny entirely */
#define DRM_NR_MODE_BASE   0xA0

static struct sock_filter drm_ioctl_filter[] = {
    /* Load the syscall number */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, nr)),
    /* If not ioctl(2), jump past this block */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_ioctl, 0, 5),

    /* ioctl: load the request argument (args[0]) */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, args[1])),
    /* Extract the type byte (bits 8–15 in _IOC encoding) */
    BPF_STMT(BPF_ALU | BPF_RSH | BPF_K, 8),
    BPF_STMT(BPF_ALU | BPF_AND | BPF_K, 0xFF),
    /* Allow if type != 'd' (0x64) — non-DRM ioctl, pass to next rule */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, DRM_TYPE, 0, 1),
    /* For DRM ioctls: reload request, extract nr byte (bits 0–7) */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, args[1])),
    BPF_STMT(BPF_ALU | BPF_AND | BPF_K, 0xFF),
    /* Deny if nr >= DRM_NR_MODE_BASE (modesetting range) */
    BPF_JUMP(BPF_JMP | BPF_JGE | BPF_K, DRM_NR_MODE_BASE,
             0 /* fall through to KILL */, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ERRNO | EPERM),
    /* Non-DRM ioctls: allow (further rules may refine) */
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
};
```

Firefox's Remote Data Decoder (RDD) process sandbox uses exactly this class of approach: it explicitly permits DRM `'d'`-type ioctls in the RDD process sandbox — which handles hardware video decode via VA-API — while blocking them from the more exposed content process [Source: Mozilla bug 1698778](https://bugzilla.mozilla.org/show_bug.cgi?id=1698778). Flatpak uses bubblewrap's `--device=dri` option to grant access to `/dev/dri/*` device nodes inside the sandbox, paired with a seccomp blocklist that excludes dangerous system calls, but does not currently implement per-ioctl-number allowlisting at the DRM level [Source: Flatpak sandbox permissions](https://docs.flatpak.org/en/latest/sandbox-permissions.html).

### 11.3 GPU Memory Isolation: Tracing DMA-BUF Exports

DMA-BUF (DMA Buffer) sharing enables zero-copy GPU memory transfer between processes and devices. A GPU driver exports a buffer as a file descriptor via `dma_buf_fd()`; the recipient imports it via `dma_buf_get()`. This mechanism is legitimate for compositor-client shared surfaces, but in multi-tenant environments it can constitute a covert channel: two co-located processes sharing a DMA-BUF can communicate without any network socket or IPC, potentially bypassing seccomp or namespace isolation.

The kernel exposes DMA-BUF lifecycle events as tracepoints in `include/trace/events/dma_buf.h` [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/trace/events/dma_buf.h). The `dma_buf_fd` tracepoint fires when a buffer is exported to userspace as a new file descriptor (condition: `fd >= 0`), carrying the fields `exp_name` (exporter driver name), `size`, `ino` (unique inode identifier), and `fd`. A separate `dma_buf_export` tracepoint fires earlier in the export lifecycle and carries `exp_name`, `size`, and `ino` but does **not** carry the final `fd` value.

To audit all DMA-BUF exports and flag cross-process sharing in real time:

```bpftrace
/* Monitor DMA-BUF userspace export events
 * Run: sudo bpftrace -e '...'
 * Requires: CONFIG_TRACING=y, tracefs mounted
 */
bpftrace -e '
tracepoint:dma_buf:dma_buf_fd {
    printf("PID %-6d COMM %-16s exp=%-12s size=%-10zu ino=%-8lu fd=%d\n",
           pid, comm, args->exp_name, args->size, args->ino, args->fd);
}'
```

The `args->exp_name` field identifies which kernel driver created the buffer (e.g., `"amdgpu"`, `"i915"`, `"v4l2"`). Cross-referencing `ino` values across events allows detection of cases where the same buffer (same inode) is imported by a second process — a pattern that warrants audit in hardened multi-tenant deployments.

For tighter integration, a BPF program attached to `tracepoint:dma_buf:dma_buf_fd` can write alerts to a perf ring buffer, enabling real-time SIEM integration without polling `/proc`.

### 11.4 Anomaly Detection for Crypto-Jacking

Crypto-jacking — hijacking a host's GPU for unauthorised mining — is detectable because its signature differs from legitimate interactive workloads: a non-display process makes a high-rate stream of compute-submit ioctls on a render node but has no open primary node (no display output) and no user session association.

The GPU command submission path on AMD hardware passes through `amdgpu_cs_ioctl()` in `drivers/gpu/drm/amd/amdgpu/amdgpu_cs.c`, with signature:

```c
/* drivers/gpu/drm/amd/amdgpu/amdgpu_cs.c */
int amdgpu_cs_ioctl(struct drm_device *dev, void *data, struct drm_file *filp)
```

A kprobe on this function counts submissions per PID. An eBPF program can maintain a BPF hash map keyed by PID, increment on each `amdgpu_cs_ioctl` entry, and periodically scan for processes exceeding a threshold:

```bpftrace
/* Conceptual: count amdgpu command submissions per process
 * Adjust threshold to taste; 10 000 submissions/s is unusual for idle background work
 */
bpftrace -e '
kprobe:amdgpu_cs_ioctl {
    @submissions[pid, comm] = count();
}

interval:s:5 {
    print(@submissions);
    clear(@submissions);
}'
```

A production implementation would additionally check whether the PID has ever opened `/dev/dri/card*` (primary node) — a heuristic that excludes legitimate compositor and media workloads. If a process exceeds the submission rate without primary-node access and without a logged-in user in its cgroup, the anomaly score warrants an alert or automatic SIGSTOP.

> Note: kprobe stability is not guaranteed across kernel versions — function inlining or renaming may silently break attachment. Prefer tracepoints (stable ABI) where available; kprobes are appropriate here because `amdgpu_cs_ioctl` does not yet have a dedicated stable tracepoint in mainline.

### 11.5 Audit Trail for Privileged DRM Operations

In multi-user workstation or shared-display environments, privileged display operations — attaching framebuffers, reconfiguring CRTCs — represent a display-takeover capability. An attacker with access to `/dev/dri/card*` (the primary node) and DRM master status can redirect display output, capture screenshots via framebuffer reads, or disrupt a desktop session.

The kernel-level handlers for the two most impactful operations are:

- `drm_mode_setcrtc()` — dispatched by `DRM_IOCTL_MODE_SETCRTC` (nr 0xA2); reconfigures a display output to point at a new framebuffer.
- `drm_mode_addfb2()` — dispatched by `DRM_IOCTL_MODE_ADDFB2` (nr 0xB8); registers a new framebuffer object, typically backed by a GEM buffer.

kprobes on these functions provide an audit trail with no kernel modification:

```bpftrace
/* Audit trail for privileged DRM display operations
 * Run as root: sudo bpftrace drm_audit.bt
 */
kprobe:drm_mode_setcrtc {
    printf("[AUDIT] SETCRTC  pid=%-6d uid=%-6d comm=%s\n",
           pid, uid, comm);
}

kprobe:drm_mode_addfb2 {
    printf("[AUDIT] ADDFB2   pid=%-6d uid=%-6d comm=%s\n",
           pid, uid, comm);
}
```

Piping this output to a structured log (JSON via bpftrace's `-f json`) and forwarding to a SIEM produces a tamper-evident record of every display reconfiguration, attributable to PID and UID. On systems where the compositor is the only legitimate caller of these paths, any other PID appearing in the log warrants investigation.

For kernel-version stability, the `drm_mode_setcrtc` symbol has been stable since DRM atomic modesetting landed; `drm_mode_addfb2` has been present since Linux 3.3. Both are non-inlined functions in `drivers/gpu/drm/drm_mode_config.c` and `drivers/gpu/drm/drm_framebuffer.c` respectively, making them reliable kprobe targets [Source: `drivers/gpu/drm/drm_crtc.c`, torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_crtc.c).

---

## Roadmap

### Near-term (6–12 months)

- **AMD vIOMMU hardware acceleration upstream.** AMD revived 22 patches for hardware-accelerated Virtualized IOMMU (vIOMMU) support in March 2026, removing the RFC tag. When merged, guest IOMMUs will benefit from lower CPU overhead and lower latency for hypervisor intercepts, tightening GPU passthrough isolation without the performance cost of software emulation. [Source](https://www.phoronix.com/news/AMD-vIOMMU-2026-Linux)
- **AMDGPU isolation enforcement rework.** A patch series refactors how AMDGPU enforces per-partition isolation by tracking submissions in per-partition data structures and using DMA fences rather than VMID limits, improving correctness for multi-partition (SPX/DPX/QPX) deployments. [Source](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg119492.html)
- **NVIDIA open-kernel confidential computing integration.** NVIDIA's Confidential Computing deployment guide (April 2026 revision) tracks ongoing alignment between the open GPU kernel modules and upstream CC attestation flows; SPDM-based remote attestation via `nvtrust` is expected to stabilise against a standardised kernel interface. Note: needs verification for exact upstream merge timeline. [Source](https://docs.nvidia.com/cc-deployment-guide-tdx.pdf)
- **GPU Rowhammer mitigations hardening.** Following GPUHammer and the related GDDRHammer/GeForge attacks demonstrated in 2025 against GDDR6-equipped NVIDIA Ampere/Ada GPUs, hardware vendors are shipping GDDR7 with on-die ECC and scrubbing. Linux ECC enforcement paths in `amdgpu` and Nouveau are expected to be tightened for datacenter parts; NVIDIA advised enabling system-level ECC as an immediate mitigation. [Source](https://www.usenix.org/conference/usenixsecurity25/presentation/nazaraliyev)
- **Intel VT-d domain-ID reclamation.** A Google-authored RFC patchset (December 2025) for Intel VT-d reworks domain ID reclamation for preserved devices, closing a potential IOMMU domain aliasing window relevant to GPU passthrough security. [Source](https://lists.openwall.net/linux-kernel/2025/12/02/1681)

### Medium-term (1–3 years)

- **Unified GPU TEE framework in the Linux kernel.** The Confidential Computing microconference at LPC has identified the need to unify AMD SEV-SNP GPU extensions, Intel TDX+GPU composite attestation, and NVIDIA H100 CC into a single kernel-level `trusted_io` software architecture, avoiding per-vendor silos. Expect a unified attestation ABI proposal targeting Linux 6.18–6.22. Note: needs verification.
- **AMD SEV-SNP GPU memory extension.** AMD is extending SEV-SNP's Secure Nested Paging to cover GPU-resident memory (VRAM inside a confidential VM), requiring collaboration between the PSP firmware, the `amdgpu` DRM driver, and the HSMP interface. This closes the gap between CPU-side SEV isolation and GPU-side VRAM visibility. [Source](https://dl.acm.org/doi/pdf/10.1145/3793532)
- **BPF LSM hooks for DRM ioctls.** Expanding `bpf_lsm` attach points to cover DRM render-node and primary-node ioctl paths would let security-aware compositors and container runtimes enforce fine-grained GPU access policies without modifying the kernel. This is currently not gated by any LSM hook, requiring seccomp rules for sandboxed applications. Note: needs verification for specific patchset status.
- **HDCP 2.3 and Secure Display for Wayland.** The Wayland `zwp_linux_content_type_manager_v1` protocol covers content-type hints but lacks a full authenticated-pipeline status feedback path. A new Wayland protocol for protected-surface status is under informal discussion in freedesktop.org issues. Note: needs verification.
- **GDDR7 on-die ECC integration.** GDDR7 introduces on-die ECC and error-scrubbing comparable to DDR5, which will require kernel-side driver support to expose ECC error counts and trigger corrective actions. AMD and NVIDIA driver maintainers are expected to surface these via the `ras` (Reliability, Availability, Serviceability) sysfs interface already used for HBM2e/HBM3 on datacenter parts. [Source](https://www.tomshardware.com/pc-components/gpus/new-geforge-and-gddrhammer-attacks-can-fully-infiltrate-your-system-through-nvidias-gpu-memory-rowhammer-attacks-in-gpus-force-bit-flips-in-protected-vram-regions-to-gain-read-write-access)

### Long-term

- **Formal verification of DRM ioctl permission logic.** The DRM ioctl dispatch table (`drm_ioctl.c`) and permission flags (`DRM_AUTH`, `DRM_MASTER`, `DRM_ROOT_ONLY`, `DRM_RENDER_ALLOW`) are candidates for model-checking tools such as Kani (Rust) or CBMC (C) as the kernel's formal verification tooling matures. A verified permission model would close the category of ioctl-level privilege escalation bugs entirely.
- **GPU-native memory isolation via hardware security cells.** Future GPU architectures may expose hardware-enforced memory "cells" analogous to Intel MPX or SPARC ADI, providing sub-allocation isolation within a single process's VRAM without IOMMU granularity. Early research prototypes (e.g., NanoZone for Arm CCA) point in this direction. [Source](https://arxiv.org/pdf/2506.07034)
- **Open-source NVIDIA H100 CC attestation in Nouveau/NVK.** As NVK matures toward feature parity with proprietary drivers, adding confidential computing support (CPR management, SPDM attestation, GSP CC firmware paths) would enable fully open-source confidential GPU workloads. This requires upstream GSP-RM collaboration and remains a long-term goal contingent on NVIDIA's firmware openness trajectory.
- **Cross-vendor GPU security certification.** As confidential AI workloads enter regulated industries (finance, healthcare), demand for Common Criteria or FIPS 140-3 certification of GPU TEE paths will grow. This will drive standardisation of attestation report formats and kernel-level audit interfaces across AMD, Intel, and NVIDIA parts. Note: needs verification.

---

## 12. Integrations

This chapter connects to many other parts of the book:

- **Ch1 — DRM Architecture:** The DRM ioctl permission model, master/auth/render-node separation, and the `drm_file` lifecycle that triggers GPU context teardown are all foundational DRM architecture topics.

- **Ch4 — GPU Memory Management:** Per-process GPUVA spaces, the `DRM_GPUVM` framework, GEM buffer reference counting, and VRAM scrubbing all live in the GPU memory management layer.

- **Ch5 — x86 GPU Drivers:** Intel i915 and Xe are the primary HDCP implementors in the kernel; PXP (Protected Xe Path) and the GSCCS security engine are Xe-specific features.

- **Ch9 — GSP-RM Firmware:** NVIDIA's GSP firmware signing, the Falcon HS/LS security model, and the ACR (Access Control Region) mechanism for firmware privilege separation are covered in the GSP-RM chapter.

- **Ch22 — Production Compositors:** The Wayland content protection protocol, compositor trust model for HDCP, and the requirement that compositors honour protected surface requests without capturing them.

- **Ch25 — GPU Compute:** GPGPU context isolation, CUDA's lack of VRAM zeroing, and the GPU compute security model under multi-tenant workloads.

- **Ch30 — Debugging and Profiling:** GPU performance counters, the OA (Observation Architecture) interface, and the restriction of counter access to authorized processes are covered in the profiling chapter.

- **Ch32 — Contributing:** Security-sensitive patches to DRM must be sent via the `security@kernel.org` embargoed disclosure path; CVE assignment and stable-tree backport processes for GPU driver vulnerabilities.

- **Ch55 — GPU Containers and Cloud:** Container-level GPU isolation, the Kubernetes GPU plugin, VFIO-based passthrough for VMs, and the security model for shared GPU clusters where GPUHammer and VRAM leakage are most relevant.

- **Ch66 — CUDA:** CUDA's `cudaMalloc` and VRAM allocation lifecycle, the lack of memory scrubbing by default, and CC-mode cuMemcpy through encrypted channels.

- **Ch71 — Intel Xe:** PXP (Protected Xe Path), GSCCS (Graphics Security Controller Command Streamer), and HuC firmware authentication via the Management Engine are Xe-specific topics detailed in the Intel stack chapter.

- **Ch72 — AMD Radeon Tools:** AMD's ROCm stack, the AMDGPU KFD ioctl surface, and the `rsmi` (ROCm SMI) interface for power telemetry that could be used as a side channel.

---

*Sources consulted for this chapter:*

- [Linux kernel DRM HDCP helper (`drivers/gpu/drm/display/drm_hdcp_helper.c`)](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_hdcp_helper.c)
- [Linux kernel DRM HDCP header (`include/drm/display/drm_hdcp.h`)](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_hdcp.h)
- [Linux kernel IOMMU API (`include/linux/iommu.h`)](https://github.com/torvalds/linux/blob/master/include/linux/iommu.h)
- [AMDGPU Process Isolation — The Linux Kernel documentation](https://docs.kernel.org/gpu/amdgpu/process-isolation.html)
- [NVIDIA Confidential Computing on H100 — NVIDIA Developer Blog](https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/)
- [Creating the First Confidential GPUs — ACM Queue](https://queue.acm.org/detail.cfm?id=3623391)
- [GPUHammer: Rowhammer Attacks on GPU Memories are Practical — USENIX Security 2025](https://www.usenix.org/conference/usenixsecurity25/presentation/lin-shaopeng)
- [Confidential Computing on NVIDIA H100 GPU: A Performance Benchmark Study — arXiv:2409.03992](https://arxiv.org/html/2409.03992v2)
- [NVIDIA nvtrust open-source repository](https://github.com/NVIDIA/nvtrust)
- [Reversing the AMD Secure Processor (PSP) — Day Zero Security](https://dayzerosec.com/blog/2023/04/17/reversing-the-amd-secure-processor-psp.html)
- [Weston content protection protocol — Wayland Explorer](https://wayland.app/protocols/weston-content-protection)
- [CUDA Never Clears GPU Memory — Barrack AI](https://blog.barrack.ai/nvidia-cuda-never-clears-gpu-memory/)
- [PCI Passthrough via OVMF — ArchWiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [BPF LSM Programs — The Linux Kernel documentation](https://docs.kernel.org/bpf/prog_lsm.html)
- [Linux kernel BPF LSM implementation (`kernel/bpf/bpf_lsm.c`)](https://github.com/torvalds/linux/blob/master/kernel/bpf/bpf_lsm.c)
- [Linux kernel DMA-BUF tracepoints (`include/trace/events/dma_buf.h`)](https://raw.githubusercontent.com/torvalds/linux/master/include/trace/events/dma_buf.h)
- [Linux kernel DRM UAPI header (`include/uapi/drm/drm.h`)](https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/drm/drm.h)
- [Firefox RDD/VAAPI seccomp sandbox and DRM ioctl allowlist — Mozilla bug 1698778](https://bugzilla.mozilla.org/show_bug.cgi?id=1698778)
- [Flatpak sandbox permissions — Flatpak documentation](https://docs.flatpak.org/en/latest/sandbox-permissions.html)
- [Seccomp BPF — The Linux Kernel documentation](https://docs.kernel.org/userspace-api/seccomp_filter.html)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
