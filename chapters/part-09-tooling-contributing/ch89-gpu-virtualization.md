# Chapter 89: GPU Virtualization in Depth

This chapter targets:

- **Systems developers** — building or integrating virtualization stacks
- **Cloud infrastructure engineers** — designing multi-tenant GPU services
- **VM and hypervisor developers** — implementing GPU sharing
- **DevOps practitioners** — deploying GPU-accelerated workloads inside containers and virtual machines

Readers are assumed to be comfortable with Linux kernel internals, PCI architecture, and basic Vulkan/OpenGL concepts.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [IOMMU and VFIO Foundations](#2-iommu-and-vfio-foundations)
3. [VFIO GPU Passthrough: Practical Setup](#3-vfio-gpu-passthrough-practical-setup)
4. [Intel SR-IOV: GVT-g to Xe SR-IOV](#4-intel-sr-iov-gvt-g-to-xe-sr-iov)
5. [AMD MxGPU SR-IOV](#5-amd-mxgpu-sr-iov)
6. [NVIDIA vGPU and MIG](#6-nvidia-vgpu-and-mig)
7. [virtio-gpu: The Paravirtual Display Stack](#7-virtio-gpu-the-paravirtual-display-stack)
8. [Venus: Vulkan Over virtio-gpu](#8-venus-vulkan-over-virtio-gpu)
9. [WSL2 GPU Architecture](#9-wsl2-gpu-architecture)
10. [GPU Containers: cgroups and Device Isolation](#10-gpu-containers-cgroups-and-device-isolation)
11. [Performance Comparison and Use-Case Guide](#11-performance-comparison-and-use-case-guide)
12. [Integrations](#12-integrations)

---

## 1. Introduction

Virtualising a GPU is fundamentally harder than virtualising a CPU or network card. A modern discrete GPU is a cache-coherent NUMA node with dozens of memory domains, a custom command-stream processor, hardware-managed scheduling queues, display engines, and video codecs — all exposed through a proprietary command ring whose semantics change with every generation. Getting those resources into a VM without sacrificing either isolation or performance is an unsolved problem in the general case, and the field has converged on four distinct strategies, each with a different point on the compatibility/performance/isolation triangle.

**Software emulation** (**virtio-gpu** in **`DRIVER_MODESET`**-only mode, or the legacy **QXL** driver) converts display output to a CPU-rendered framebuffer. Compatibility is essentially universal — any guest kernel sees a standard PCI device — but performance is limited to what the host CPU can push through a **virtio** ring. Emulation is sufficient for **VDI** text desktops or headless workloads with software rendering.

**Paravirtualisation / API forwarding** lifts the abstraction one level: the guest exports a stream of 3D API commands (**OpenGL** via **VirGL**, **Vulkan** via **Venus**) over virtio, and a host-side renderer translates them to real GPU calls. This gives near-native throughput for graphics-API-centric workloads without dedicating hardware to a single VM. The **virtio-gpu** driver implements a full **DRM/KMS** stack including atomic modesetting, **GEM** memory management, and blob resources for zero-copy buffer sharing. **VirGL** serialises a **Gallium3D**-derived state-object stream using **TGSI** shaders and host-side **virglrenderer**, while **Venus** transmits **Vulkan** commands nearly verbatim as **SPIR-V** binaries over a shared-memory **`vn_ring`** command ring, decoded by **virglrenderer** and forwarded to the host **Vulkan** **ICD** (such as **RADV** or **ANV**).

**SR-IOV and mediated devices** partition one physical GPU into multiple hardware Virtual Functions (**VF**s) or software-mediated slices. **Intel GVT-g** (covering Gen8–Gen12, implemented via the kernel **mdev** framework and the **`kvmgt`** module) and its successor **Xe SR-IOV** (introduced in Linux 6.8 for **Intel Arc**/**Battlemage**, implemented under **`drivers/gpu/drm/xe/`**), and **AMD MxGPU** (using spatial partitioning managed by the **GIM** host module and the in-kernel **`amdgpu_virt`** subsystem) use these mechanisms to run multiple independent guests with hardware-accelerated compute and display simultaneously. **SR-IOV** is the technology of choice for **VDI** and multi-tenant ML training.

**VFIO passthrough** assigns the entire GPU to a single VM at the **IOMMU** boundary. The guest driver runs on bare-metal register access; there is effectively zero virtualisation overhead. Passthrough is the gold standard for cloud gaming and single-tenant dedicated GPU compute, but consumes an entire card per guest and requires careful **IOMMU group** design on the host. The foundations of **VFIO** — including **VT-d** and **AMD-Vi** hardware, **IOMMU groups** under **`/sys/kernel/iommu_groups/`**, **`vfio-pci`** stub binding, **interrupt remapping**, and **PCIe FLR** (Function Level Reset) for safe device reassignment — underpin all hardware-isolation approaches in this chapter. Practical **VFIO** passthrough involves **QEMU** with **OVMF** firmware and **`-mem-path /dev/hugepages`**, the **ACS** (Access Control Services) override patch for consumer-platform **IOMMU group** constraints, and **Looking Glass** (**KVMFR**) for low-latency framebuffer capture via an **IVSHMEM** shared-memory device.

**NVIDIA** offers two proprietary virtualisation technologies: **vGPU**, a time-sliced implementation requiring the **NVIDIA vGPU Manager** and a per-GPU annual licence, and **MIG** (Multi-Instance GPU), available from the **A100** through **H100**, **H200**, and **Blackwell**, which partitions the physical GPU into **GPU Instances** (**GI**s) and **Compute Instances** (**CIs**) with dedicated **SM**s, **L2** partitions, and **HBM** channels. **MIG** partitions are exposed as **`/dev/nvidia-caps/`** capability files and targeted via the **`NVIDIA_VISIBLE_DEVICES`** environment variable.

**WSL2** (Windows Subsystem for Linux 2) provides GPU access through a Microsoft-proprietary path: the **`dxgkrnl`** kernel module exposes **`/dev/dxg`** and forwards **WDDM D3DKMT** ioctls over **Hyper-V VMBUS** to the Windows GPU driver. The **Mesa** **`d3d12`** Gallium driver translates **OpenGL**/**Gallium** calls into **Direct3D 12** via **`libdxcore.so`**, and **VA-API** video decode is bridged through the same stack. **WSLg** provides GUI support via a **Weston** compositor with a **RAIL**/**VAIL** **RDP** backend. **DirectML** is the recommended **ONNX Runtime** inference path in this environment.

GPU containers and orchestration are covered in depth: the **NVIDIA Container Toolkit** (supporting both legacy **OCI** hook mode and modern **CDI** — Container Device Interface — via **`/etc/cdi/`** specs), **AMD ROCm** containers using **`/dev/kfd`** and **DRM** render nodes, **Intel** GPU containers via **`/dev/dri/renderD128`**, the **Kubernetes** device plugin framework (**`nvidia.com/gpu`**, **`amd.com/gpu`**, **`intel.com/gpu`**), **Dynamic Resource Allocation** (**DRA**) as of **Kubernetes** 1.31, and **cgroup v2** **eBPF**-based device access control via **`BPF_CGROUP_DEVICE`**.

The chapter closes with a qualitative performance comparison across all approaches — **VFIO passthrough**, **SR-IOV VF**, **NVIDIA MIG**, **Intel GVT-g**, **Venus**, **VirGL**, and **virtio-gpu** display-only — and a decision guide covering cloud gaming, multi-user **VDI**, multi-tenant ML training and inference, development environments, **CI/CD** pipelines, and frame timing jitter in cloud gaming scenarios.

An emerging fifth category — **confidential computing enclaves** (**AMD SEV-SNP**, **Intel TDX** with **GPU TEE** extensions) — encrypts VM memory such that even the hypervisor cannot read GPU buffers. That topic is examined in Ch80.

### 1.1 What is GPU Virtualization?

GPU virtualization is the collection of techniques that allow a physical GPU to be shared among multiple virtual machines or containers while maintaining isolation, security, and acceptable performance. Unlike CPU virtualization, where a hypervisor can trap and emulate privileged instructions with relatively low overhead, GPUs expose many thousands of hardware registers, proprietary command streams, and memory apertures that cannot be efficiently emulated in software. The challenge is compounded by the fact that GPU driver stacks are large, complex, and tightly coupled to specific hardware generations, making clean abstraction boundaries difficult to define. The Linux kernel addresses this through a layered set of mechanisms: IOMMU hardware for DMA isolation, the VFIO framework for safe device assignment, the virtio-gpu paravirtual device model for portable API forwarding, and vendor-specific SR-IOV or mediated-device schemes for hardware partitioning. Each strategy trades compatibility, isolation, and performance differently, and the choice of mechanism depends on whether the goal is maximum throughput for a single workload, fair sharing across many tenants, or broad OS compatibility in a cloud or VDI environment.

### 1.2 What is VFIO?

VFIO (Virtual Function I/O) is a kernel framework, introduced in Linux 3.6, that allows userspace programs — primarily hypervisors such as QEMU — to directly control PCI devices with full hardware access while the IOMMU enforces memory isolation between the device and the rest of the system. A device bound to the `vfio-pci` stub driver is removed from all other host kernel drivers and becomes accessible through a layered set of character devices: `/dev/vfio/vfio` represents an IOMMU domain (container), `/dev/vfio/<N>` represents an IOMMU group, and individual device file descriptors expose BAR memory regions via `mmap()` and interrupt delivery via eventfd. The core abstraction lives in `drivers/vfio/` in the kernel source tree. VFIO is the foundation for GPU passthrough, where an entire physical GPU is dedicated to a single VM at the IOMMU boundary, and is also used to expose SR-IOV Virtual Functions to guest operating systems. The framework handles interrupt remapping, PCIe Function Level Reset for safe device reassignment, and DMA address translation through IOMMU page tables.

### 1.3 What is SR-IOV?

SR-IOV (Single Root I/O Virtualization) is a PCIe specification that allows a single physical device to present multiple independent virtual endpoints on the PCI bus. The device exposes one Physical Function (PF) managed by the host and up to 256 Virtual Functions (VFs) that are each assignable to a separate virtual machine. Each VF has its own PCIe configuration space, its own MSI-X interrupt vectors, and at least one dedicated DMA queue, but shares the physical compute resources, caches, and memory fabric of the underlying silicon. From the IOMMU's perspective each VF is an independent endpoint with its own IOMMU group, so a VF can be passed to a VM via VFIO without exposing the PF or other VFs to that guest. In the GPU context, SR-IOV requires the hardware scheduler to arbitrate between VF workloads in the compute and display engines — a capability that consumer GPUs historically lacked but that datacenter and newer consumer GPU lines now implement. Intel Xe SR-IOV (Linux 6.8 and later, under `drivers/gpu/drm/xe/`) and AMD MxGPU are the two actively maintained open-source SR-IOV GPU stacks covered in this chapter.

### 1.4 What is virtio-gpu?

virtio-gpu is the paravirtual GPU device model defined in the VirtIO specification (section 5.7). Unlike passthrough or SR-IOV, it does not require any specific GPU hardware in the host: the guest sees a standard virtio PCI device and communicates with it by writing command descriptors to a VirtIO ring, while a host-side renderer — typically inside QEMU or a dedicated GPU process — translates those commands to real GPU API calls. The Linux guest driver (`drivers/gpu/drm/virtio/`) implements a full DRM/KMS stack including atomic modesetting, GEM buffer management, and blob resources for zero-copy sharing via DMA-BUF. Two distinct forwarding protocols run over virtio-gpu: VirGL transmits a Gallium3D-derived OpenGL state stream processed by `virglrenderer` on the host, while Venus (§8) transmits Vulkan commands nearly verbatim, decoded by `virglrenderer` and forwarded to the host Vulkan ICD. The virtio-gpu model provides strong portability — it works on any hypervisor that implements the VirtIO transport, including QEMU, crosvm, and cloud-hypervisor — at the cost of some API overhead compared to direct hardware access.

---

## 2. IOMMU and VFIO Foundations

### 2.1 IOMMU Hardware

An Input-Output Memory Management Unit (IOMMU) sits between the PCI bus and physical RAM, performing address translation for device DMA in the same way the CPU's MMU protects process address spaces. On Intel platforms the technology is called **VT-d** (Virtualization Technology for Directed I/O); AMD calls theirs **AMD-Vi** (also written IOMMU or Vi). [Source](https://www.kernel.org/doc/html/latest/driver-api/vfio.html)

When a PCI device issues a DMA write to guest-physical address `0x10000000`, the IOMMU intercepts the transaction, consults its page tables (the IOMMU domain), and maps that I/O Virtual Address (IOVA) to the actual host-physical page the hypervisor pinned for the VM. Without the IOMMU a rogue device (or a compromised guest driver) could perform arbitrary DMA to any host physical memory — the classic DMA attack vector.

Enabling IOMMU requires both BIOS/firmware support and a kernel boot parameter:

```bash
# Intel
intel_iommu=on iommu=pt

# AMD
amd_iommu=on iommu=pt
```

The `iommu=pt` ("passthrough") flag tells the IOMMU to only protect devices that have been explicitly assigned to domains, avoiding overhead for host-owned devices.

### 2.2 IOMMU Groups

The kernel groups PCI devices that share DMA isolation capabilities into **IOMMU groups** (`/sys/kernel/iommu_groups/`). Every device in a group shares the same IOMMU translation context, which means the IOMMU cannot isolate them from each other at the hardware level. The cardinal rule of VFIO is: **all devices in an IOMMU group must be assigned to the same VM, or stubbed with `vfio-pci`**. Passing only the GPU from a group that also contains the PCIe root port leaves the host CPU able to access the VM's memory via that root port's DMA path.

Consumer motherboards frequently place multiple unrelated devices in the same IOMMU group, defeating the requirement. The ACS (Access Control Services) override discussed in §3 is a workaround with security implications.

### 2.3 VFIO Kernel Objects

VFIO exposes three layered abstractions to userspace [Source](https://kernel-internals.org/virtualization/vfio/):

**Container** (`/dev/vfio/vfio`): represents a single IOMMU domain. Multiple groups can share a container if they need coherent DMA mappings to the same address space. The container holds a reference to an `iommu_domain`.

**Group** (`/dev/vfio/N`): wraps the kernel's `iommu_group` object. The character device at `/dev/vfio/<N>` (where N is the IOMMU group number) exposes ioctls to attach the group to a container and to open individual device fds.

**Device**: opened via `VFIO_GROUP_GET_DEVICE_FD` on the group fd. The core PCI device implementation lives in `vfio_pci_core_device`:

```c
/* drivers/vfio/pci/vfio_pci_core.h (simplified) */
struct vfio_pci_core_device {
    struct vfio_device      vdev;
    struct pci_dev         *pdev;
    void __iomem           *barmap[PCI_STD_NUM_BARS];
    struct eventfd_ctx     *err_trigger;
    struct eventfd_ctx     *req_trigger;
    /* interrupt contexts, INTx, MSI, MSIX */
};
```

MMIO BAR regions are made accessible to userspace via `mmap()` on the device fd; the IOMMU ensures writes from the VM are translated correctly.

### 2.4 Interrupt Remapping

Without interrupt remapping a device under VFIO could inject arbitrary interrupts into the host kernel. VT-d and AMD-Vi implement interrupt remapping tables that translate device MSI/MSI-X vectors to per-VM interrupt descriptors. The vfio-pci driver uses `vfio_pci_intx_mask()` to block INTx-style level interrupts while QEMU polls for them via eventfd, avoiding spurious host interrupts.

### 2.5 PCIe FLR

Before a GPU can be safely re-assigned between VMs it must be reset to a known-clean state. PCIe **Function Level Reset** (FLR) is the standard mechanism: writing `1` to the FLR bit in the device's PCIe capability register forces all pending DMA and MMIO transactions to complete, flushes caches, and returns registers to power-on defaults. VFIO enforces FLR on device close. Not all GPUs implement FLR correctly; a defective FLR leaving stale DMA mappings is a security vulnerability (see Ch80). [Source](https://www.kernel.org/doc/html/latest/driver-api/vfio.html)

---

## 3. VFIO GPU Passthrough: Practical Setup

### 3.1 Binding to vfio-pci

The `vfio-pci` kernel module is a stub driver whose sole purpose is to bind a PCI device to VFIO. Once bound, the device is invisible to all other host kernel drivers:

```bash
# Find the GPU's PCI ID
lspci -nn | grep -i nvidia
# 01:00.0 VGA compatible controller [0300]: NVIDIA TU102 [10de:1e04]
# 01:00.1 Audio device [0403]: NVIDIA TU102 [10de:10f7]

# Load VFIO modules early in initramfs (add to /etc/mkinitcpio.conf MODULES=)
modprobe vfio vfio_iommu_type1 vfio_pci

# Bind GPU and its companion audio device at boot via modprobe option
echo "options vfio-pci ids=10de:1e04,10de:10f7" > /etc/modprobe.d/vfio.conf
```

The same IOMMU group rule applies: both the GPU (`01:00.0`) and its HDMI audio device (`01:00.1`) must be bound to `vfio-pci` because they share an IOMMU group. [Source](https://www.pchardwarepro.com/en/complete-guide-to-gpu-passthrough-with-vfio-on-linux/)

### 3.2 QEMU Command Line

A minimal QEMU invocation for single-GPU passthrough with OVMF:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 16G \
  -mem-path /dev/hugepages \
  -cpu host,kvm=off,hv_vendor_id=GenuineIntel \
  -machine q35,accel=kvm \
  -bios /usr/share/ovmf/OVMF.fd \
  -device vfio-pci,host=01:00.0,multifunction=on,x-vga=on \
  -device vfio-pci,host=01:00.1 \
  ...
```

Key parameters:

- **`-mem-path /dev/hugepages`**: backs VM RAM with 2 MiB hugepages, reducing TLB pressure for IOMMU page walks.
- **`-machine q35`**: required for PCIe topology that matches a real platform; older `i440fx` lacks the PCIe root complex geometry VFIO expects.
- **`-bios OVMF.fd`**: OVMF is an open-source UEFI firmware implementation. Most modern GPU ROMs require UEFI GOP initialisation; OVMF satisfies this requirement where SeaBIOS cannot.
- **`x-vga=on`**: tells QEMU to shadow the VGA legacy I/O range (ports `0x3c0`–`0x3df`, memory `0xa0000`–`0xbffff`) from the passed-through device, enabling the ROM to run at boot.
- **`kvm=off`**: prevents the CPUID `KVM` hypervisor leaf from being visible; required by some NVIDIA drivers that refuse to initialise in a VM (the so-called anti-VM check). Note: recent NVIDIA drivers relax this restriction, but `hv_vendor_id=GenuineIntel` is still commonly used to spoof the vendor string.

### 3.3 PCIe ACS Override Quirks

On consumer motherboards many PCIe slots share an IOMMU group with the chipset's PCIe switch, making clean passthrough impossible without the **ACS override patch** — an out-of-tree kernel patch that forces ACS (Access Control Services) peer-to-peer isolation bits to be reported as enabled even when the hardware does not implement them. This broadens isolation enough for VFIO but does so by lying about hardware capabilities, which introduces a real (if typically theoretical) security risk: another device in the same group could still issue peer-to-peer DMA bypassing the IOMMU. Use only in single-owner environments.

### 3.4 Looking Glass: Low-Latency Framebuffer Capture

When the GPU is passed through, its display output is only available on the physical connector. **Looking Glass** (project codename KVMFR, KVM FrameRelay) solves this by sharing the GPU's rendered framebuffer with the host via a shared-memory IVSHMEM device:

1. The Windows guest installs the KVMFR capture driver, which uses the Desktop Duplication API to capture frames and writes them — uncompressed — into the IVSHMEM shared region.
2. The `kvmfr` kernel module on the host maps that shared region, allowing the host Looking Glass client to read and display frames without any video encoding/decoding round-trip.
3. Frame capture rates above 300 UPS have been reported, with end-to-end display latency comparable to native. [Source](https://github.com/gnif/LookingGlass)

QEMU exposes the shared memory via:

```bash
-device ivshmem-plain,memdev=ivshmem,bus=pcie.0 \
-object memory-backend-file,id=ivshmem,share=on,mem-path=/dev/kvmfr0,size=128M
```

---

## 4. Intel SR-IOV: GVT-g to Xe SR-IOV

### 4.1 GVT-g (Gen8–Gen12): Mediated Passthrough

**GVT-g** is Intel's GPU virtualisation technology for Gen8 through Gen12 (8th–12th generation, Broadwell through Tiger Lake/Alder Lake for the i915 driver). It implements **mediated passthrough** — the GPU is not partitioned at the PCIe level but multiplexed in software via the kernel's **mdev** (mediated device) framework. [Source](https://github.com/intel/gvt-linux/wiki/GVTg-New-Architecture-Introduction-Update)

The `kvmgt` kernel module (`CONFIG_DRM_I915_GVT_KVMGT`) registers i915 as an mdev parent. Each vGPU instance is created by writing a UUID to the sysfs interface:

```bash
# List available vGPU types
ls /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/
# i915-GVTg-V5-1  i915-GVTg-V5-2  i915-GVTg-V5-4  i915-GVTg-V5-8

# Create a vGPU of type V5-4 (4 slices, ~256 MB aperture)
echo "$(uuidgen)" > /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg-V5-4/create
```

The type number encodes the resource allocation: `V5-8` gives 8 virtual engines and a large aperture but limits the host to one vGPU; `V5-1` shrinks the per-vGPU footprint to allow up to eight concurrent instances.

Internally, GVT-g maintains per-VM **shadow page tables** that mirror the guest's GPU page table (GGTT and PPGTT). A write to the guest GGTT causes a vmexit; the hypervisor validates the mapping and updates a shadow GGTT entry that points to the actual host-physical page. Workload scheduling between vGPUs is time-sliced by a per-engine scheduler running in the i915 driver. [Source](https://github.com/intel/gvt-linux/wiki/GVTg_Setup_Guide)

### 4.2 Xe SR-IOV (Intel Arc and Beyond)

The new **Xe** kernel driver (introduced in Linux 6.8 for Intel Arc/Battlemage, replacing i915 for discrete GPUs) natively supports PCIe **SR-IOV** as defined by the PCIe specification — genuine hardware Virtual Functions, not software mediation. [Source](https://www.phoronix.com/news/Intel-Xe-Linux-6.17-PF-4K)

Key kernel code lives under `drivers/gpu/drm/xe/xe_gt_sriov_pf_*` (Physical Function management) and `xe_gt_sriov_vf_*` (Virtual Function guest driver). VF creation follows the standard Linux SR-IOV path:

```bash
# Enable SR-IOV on the Xe Physical Function with up to 7 VFs
echo 7 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# Verify VFs appeared
lspci | grep -i "Virtual Function"
# 03:00.1 VGA compatible controller: Intel Corporation Device [VF] ...
```

The `xe.max_vfs` module parameter caps the number of VFs the driver will ever activate. As of Linux 6.17, SR-IOV PF mode is enabled by default for Battlemage (BMG) and Panther Lake (PTL) platforms. [Source](https://forum.proxmox.com/threads/issue-enabling-sr-iov-on-intel-alder-lake-p-igpu-proxmox-6-8-xe-driver.154381/)

QEMU assignment reuses the standard VFIO path — the VF appears as a distinct PCI device with its own IOMMU group:

```bash
qemu-system-x86_64 \
  -device vfio-pci,host=03:00.1 \
  ...
```

The critical difference from GVT-g: the Xe VF has its own PCI config space, its own BAR regions, and (on sufficiently capable hardware) its own fixed slice of the GPU's memory channels. The hypervisor is not involved in command submission after assignment.

**Performance ordering (qualitative)**: Full VFIO passthrough ≈ SR-IOV VF > GVT-g time-sliced > virtio-gpu/Venus. The SR-IOV advantage over GVT-g is primarily lower latency (no vmexit on GGTT write) and better isolation.

---

## 5. AMD MxGPU SR-IOV

AMD's GPU virtualisation technology, branded **MxGPU**, uses hardware SR-IOV and is supported from the Fiji/Tonga generation through the current RDNA3/CDNA3 generations. Unlike Intel's time-sliced GVT-g, AMD MxGPU uses **spatial partitioning**: compute units, memory channels, and fixed bandwidth are statically allocated per VF — there is no time-sharing scheduler. [Source](https://github.com/amd/MxGPU-Virtualization)

### 5.1 GIM Module

On the hypervisor, AMD ships the **GIM** (GPU-IOV Module) driver (open-source on GitHub at `amd/MxGPU-Virtualization`) which runs alongside the amdgpu kernel module. GIM is responsible for:

- SR-IOV physical-function initialisation and VF provisioning
- PF/VF mailbox-based communication
- Hang detection and FLR-based recovery
- Managing the world switch (context save/restore across VF scheduling epochs)

On Instinct MI300X for multi-tenant inference, multiple VFs can be instantiated on the same physical function, each with a fixed share of the 192 GB HBM pool. [Source](https://instinct.docs.amd.com/projects/virt-drv/en/latest/userguides/Getting_started_with_MxGPU.html)

### 5.2 amdgpu_virt in the Kernel

The in-tree Linux kernel support for the guest side of AMD SR-IOV lives in `drivers/gpu/drm/amd/amdgpu/amdgpu_virt.c` and its header `amdgpu_virt.h`. [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.c) Key types and functions:

```c
/* amdgpu_virt.h – core virtualization structures */

struct amdgpu_virt_ops {
    int (*req_full_gpu)(struct amdgpu_device *adev, bool init);
    int (*rel_full_gpu)(struct amdgpu_device *adev, bool init);
    int (*reset_gpu)(struct amdgpu_device *adev);
    int (*wait_reset)(struct amdgpu_device *adev);
    void (*trans_msg)(struct amdgpu_device *adev, u32 req, u32 data1,
                      u32 data2, u32 data3);
};

struct amdgpu_vf_error_buffer {
    struct mutex lock;
    int read_count;
    int write_count;
    uint16_t code[AMDGPU_VF_ERROR_ENTRY_SIZE];
    uint16_t flags[AMDGPU_VF_ERROR_ENTRY_SIZE];
    uint64_t data[AMDGPU_VF_ERROR_ENTRY_SIZE];
};
```

The `amdgpu_sriov_vf(adev)` macro is the canonical guard throughout the driver to execute VF-specific code paths:

```c
/* Used throughout amdgpu to branch on whether we are a VF */
#define amdgpu_sriov_vf(adev)   ((adev)->virt.caps & AMDGPU_SRIOV_CAPS_IS_VF)
#define amdgpu_sriov_enabled(adev) \
    ((adev)->virt.caps & (AMDGPU_SRIOV_CAPS_IS_VF | AMDGPU_SRIOV_CAPS_RUNTIME))
```

Key runtime functions [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_virt.c):

- `amdgpu_virt_request_full_gpu()` — asks the hypervisor for exclusive GPU access during GPU initialisation/reset.
- `amdgpu_virt_release_full_gpu()` — releases the exclusive token.
- `amdgpu_virt_reset_gpu()` — triggers an FLR via hypervisor mailbox.
- `amdgpu_virt_exchange_data()` — synchronises the PF↔VF shared data structure (`amd_sriov_msg_pf2vf_info` / `amd_sriov_msg_vf2pf_info`) used for capability negotiation and error reporting.

RLCG (Register List Control via GRBM) indirect register access error codes (from `amdgpu_virt.h`):

```c
#define AMDGPU_RLCG_VFGATE_DISABLED       0x4000000
#define AMDGPU_RLCG_WRONG_OPERATION_TYPE  0x2000000
#define AMDGPU_RLCG_REG_NOT_IN_RANGE      0x1000000
```

These codes surface when a VF attempts a privileged MMIO access that should be mediated through RLCG; the driver returns them to indicate misconfiguration or security policy violations.

### 5.3 World Switch Overhead

AMD MxGPU's spatial-partitioning model means that, unlike GVT-g, there is **no per-command world switch** — each VF has its own hardware queue. However, context switches between VF scheduling epochs (where the hardware moves from one VF's compute context to another) still incur a world-switch cost: saving and restoring shader state, wavefront registers, and LDS. On MI300X this overhead is sub-millisecond for typical ML inference contexts, but it is not zero. The GIM scheduler controls epoch duration. [Source](https://instinct.docs.amd.com/projects/virt-drv/en/latest/userguides/Host_configuration.html)

---

## 6. NVIDIA vGPU and MIG

### 6.1 NVIDIA vGPU (Proprietary)

NVIDIA's traditional virtualisation product, **vGPU**, is a proprietary time-sliced implementation similar in concept to GVT-g. It requires the NVIDIA vGPU Manager kernel module on the host and a per-GPU annual licence. vGPU profiles (`Q`-series for workstation, `C`-series for compute) carve the GPU into named instances by memory size (e.g., `NVIDIA-A100-80C-40C` gives 40 GB to a vGPU). Because this is a proprietary product with a licencing wall, the open-source Linux ecosystem does not support it natively; deployment requires the NVIDIA-provided closed hypervisor components.

### 6.2 MIG: Multi-Instance GPU

**MIG** (Multi-Instance GPU), introduced with the A100 and continued through H100, H200, and Blackwell, is NVIDIA's **hardware SR-IOV-like** partitioning — but implemented in NVIDIA's proprietary driver stack rather than via standard PCIe SR-IOV. MIG uses separate hardware engines (SMs, L2 partitions, HBM channels) to provide genuine compute isolation between instances. [Source](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

**Hierarchy**: A GPU Instance (GI) is carved from the physical GPU (e.g., `7g.80gb` uses all 7 GPCs and 80 GB HBM on an A100). Inside a GI, one or more **Compute Instances** (CIs) share the GI's resources (e.g., three `2c` CIs in a `7g.80gb` GI, each with ~2 GPCs). A CUDA application sees a CI + its parent GI as a single CUDA device.

```bash
# Enable MIG mode on GPU 0 (requires reboot or driver reload on some systems)
nvidia-smi -i 0 -mig 1

# List available GPU instance profiles on A100
nvidia-smi mig -lgip

# Create a 3g.40gb GPU instance (3 GPCs, 40 GB HBM)
nvidia-smi mig -cgi 9 -i 0      # profile 9 = 3g.40gb on A100
# Output: Successfully created GPU instance ID  1 ...

# Create a compute instance inside that GPU instance
nvidia-smi mig -cci 0 -gi 1 -i 0

# List all MIG devices
nvidia-smi -L
# GPU 0: NVIDIA A100 (MIG 3g.40gb)
#   MIG 3g.40gb     Device  0: (UUID: MIG-GPU-<uuid>/1/0)
```

Device nodes for MIG appear under `/dev/nvidia-caps/`:

```
/dev/nvidia0          # full GPU (inaccessible in MIG mode for CUDA)
/dev/nvidia-caps/nvidia-cap1   # GI 1 capability file
/dev/nvidia-caps/nvidia-cap2   # CI 0 in GI 1 capability file
```

Access to a CI is gated by the capability file. Containers and cgroups achieve isolation via the device cgroup (v1: `devices.allow/deny`; v2: eBPF-based) on the capability file descriptor. A container is exposed to a specific MIG partition via:

```bash
NVIDIA_VISIBLE_DEVICES=MIG-GPU-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/1/0
```

where the UUID identifies the physical GPU, the first integer the GI, and the second the CI. [Source](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

The NVIDIA Container Toolkit (§10) reads `NVIDIA_VISIBLE_DEVICES` at container start and configures the device cgroup and driver library mounts accordingly.

**No graphics APIs** are supported in MIG mode; the A100/H100 are exclusively compute GPUs. RTX Pro 6000 Blackwell is the first GPU where NVIDIA is extending MIG to graphics workloads (Note: this feature is still evolving as of mid-2026; verify current documentation).

---

## 7. virtio-gpu: The Paravirtual Display Stack

### 7.1 Driver Overview

**virtio-gpu** is the standard paravirtual GPU for QEMU/KVM VMs, implemented in the Linux kernel under `drivers/gpu/drm/virtio/`. The driver conforms to the Virtio GPU specification (section 5.7 of the OASIS virtio-1.2 specification) and implements a full DRM/KMS driver stack, including atomic modesetting and GEM memory management. [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/virtio/virtgpu_drv.c)

The driver advertises the following feature flags (negotiated with the host VMM at probe time):

```c
/* From virtgpu_drv.c */
static unsigned int features[] = {
    VIRTIO_GPU_F_VIRGL,           /* Gallium/VirGL 3D command stream */
    VIRTIO_GPU_F_EDID,            /* Extended display information */
    VIRTIO_GPU_F_RESOURCE_UUID,   /* UUID-addressed resources */
    VIRTIO_GPU_F_RESOURCE_BLOB,   /* Blob (zero-copy) resources */
    VIRTIO_GPU_F_CONTEXT_INIT,    /* Context initialisation with named type */
    VIRTIO_GPU_F_BLOB_ALIGNMENT,  /* Blob alignment hints */
};
```

The DRM driver capabilities include `DRIVER_MODESET | DRIVER_GEM | DRIVER_RENDER | DRIVER_ATOMIC | DRIVER_SYNCOBJ | DRIVER_SYNCOBJ_TIMELINE`.

### 7.2 Command Protocol

Guest userspace or Mesa submits GPU work via DRM ioctls that the kernel driver serialises into virtio virtqueue messages. Key commands in the protocol include:

| Command | Purpose |
|---|---|
| `VIRTIO_GPU_CMD_RESOURCE_CREATE_2D` | Allocate a 2D scanout framebuffer |
| `VIRTIO_GPU_CMD_RESOURCE_CREATE_3D` | Allocate a 3D (VirGL) resource |
| `VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB` | Allocate a blob resource (zero-copy) |
| `VIRTIO_GPU_CMD_SUBMIT_3D` | Submit a VirGL or Venus command stream |
| `VIRTIO_GPU_CMD_SET_SCANOUT` | Bind a resource to a display output |
| `VIRTIO_GPU_CMD_RESOURCE_FLUSH` | Flush a resource to the host display |

### 7.3 Blob Resources: Zero-Copy Path

The traditional virtio-gpu path copies pixel data from a guest GEM object into a host-side resource using `VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D`. For high-resolution or high-framerate display this is prohibitively expensive.

**Blob resources** (`VIRTIO_GPU_F_RESOURCE_BLOB`) solve this by backing a resource with host-visible memory that both the guest and host map simultaneously. A `VIRTIO_GPU_CMD_RESOURCE_CREATE_BLOB` request with `VIRTIO_GPU_BLOB_MEM_HOST3D_GUEST` tells the host to allocate a buffer visible in the guest's physical address space. The guest kernel maps it into its address space (via `udmabuf` or a dedicated carveout), and both sides can access it without any copy. [Source](https://patchwork.kernel.org/project/dri-devel/patch/20200814024000.2485-11-gurchetansingh@chromium.org/)

This mechanism is the foundation for the Venus Vulkan zero-copy swapchain (§8).

### 7.4 VirGL: OpenGL over virtio-gpu

VirGL (Virtual GL) is the 3D acceleration layer for virtio-gpu's OpenGL workloads. It is not an API translation layer in the OpenGL-to-Vulkan sense; rather, it uses a Gallium3D-derived serialised state-object stream. The guest Mesa Gallium driver (`virgl`) encodes OpenGL state (draw calls, shader programs in TGSI format, textures, samplers) into a command buffer and submits it via `VIRTIO_GPU_CMD_SUBMIT_3D`. The host-side **virglrenderer** library decodes these commands and issues native OpenGL calls to the host GPU driver. [Source](https://www.collabora.com/news-and-blog/blog/2025/01/15/the-state-of-gfx-virtualization-using-virglrenderer/)

VirGL supports up to OpenGL 4.3 / GLES 3.2 (or 4.4+ with blob resources enabled). Shaders must be compiled twice — once in the guest to produce TGSI, and once in the host to compile TGSI to the host driver's native IR — which is a significant CPU overhead for shader-heavy applications.

---

## 8. Venus: Vulkan Over virtio-gpu

### 8.1 Architecture

**Venus** is a Vulkan command-serialisation protocol layered on top of virtio-gpu. Rather than translating API calls (as VirGL does for OpenGL), Venus transmits the Vulkan command stream nearly verbatim: the guest driver serialises Vulkan API calls into a binary format defined by the Venus protocol (XML-described, with generated encoder/decoder code), and a host-side renderer deserialises and re-issues them to the host Vulkan driver. [Source](https://docs.mesa3d.org/drivers/venus.html)

This approach has several advantages over VirGL for Vulkan workloads:

- Shaders are transmitted as SPIR-V binaries without any intermediate format conversion.
- The protocol is versioned with the Vulkan spec; adding a new extension requires adding its serialisation to the protocol, not reimplementing the semantic behaviour.
- As of Mesa 26.0, Venus supports Vulkan 1.3 and exposes extensions including `VK_EXT_mesh_shader` and `VK_KHR_ray_query` (where the host driver supports them). [Source](https://www.phoronix.com/news/Venus-Vulkan-Mesh-Shader)

### 8.2 Guest Driver: vn_* Types

The Venus guest driver lives in `src/virtio/` in the Mesa tree. Core object types mirror their Vulkan counterparts:

```c
/* src/virtio/vulkan/vn_instance.h (schematic) */
struct vn_instance {
    struct vk_instance          base;       /* Mesa Vulkan common */
    struct vn_renderer         *renderer;   /* transport to host */
    struct vn_ring             *ring;       /* command ring */
    /* ... */
};

struct vn_device {
    struct vk_device            base;
    struct vn_instance         *instance;
    struct vn_physical_device  *physical_device;
    /* queues, command pools, memory type maps... */
};

struct vn_queue {
    struct vk_queue             base;
    struct vn_device           *device;
    uint32_t                    family;
    uint32_t                    index;
};
```

[Source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/virtio)

### 8.3 The vn_ring Command Ring

The `vn_ring` is Venus's primary transport: a shared-memory ring buffer that the guest writes encoded Vulkan commands into and the host renderer reads out of. It is conceptually similar to a Vulkan command buffer, but at the transport layer rather than the API layer. The ring is backed by a blob resource so both guest and host share the same physical pages with no copy.

```c
/* Schematic of vn_ring submission */
struct vn_ring_submit_command {
    uint32_t *cs_data;       /* encoded Venus protocol bytes */
    size_t    cs_size;
    uint32_t  ring_seqno;   /* monotonic sequence number */
};
```

Larger, less-frequent commands (e.g., `vkCreateGraphicsPipelines` with large pipeline state) are submitted via `VIRTIO_GPU_CMD_SUBMIT_3D` with the Venus context type rather than through the ring, to avoid ring stalls. [Source](https://deepwiki.com/arehnman/virtio-win-mesa/7.7-virtualized-and-layered-drivers-(virtio-gpu-venus-gfxstream))

### 8.4 Host Renderer: vkrenderer / virglrenderer

On the host, the **virglrenderer** library (version ≥ 0.10) handles both VirGL OpenGL and Venus Vulkan contexts. When a Venus context (`VIRTIO_GPU_CONTEXT_TYPE_VENUS`) is created, virglrenderer spawns a per-context Vulkan instance on the host and forwards decoded Vulkan calls to the host's Vulkan driver (RADV for AMD, ANV for Intel, or any conformant ICD).

The QEMU side uses `-device virtio-vga-gl` or `-device virtio-gpu-gl` combined with `-display gtk,gl=on` to activate the virglrenderer backend. CrosVM uses a similar mechanism with its `gfxstream` backend as an alternative.

### 8.5 Zero-Copy Swapchain

For presentation, Venus uses blob resources to avoid the copy that VirGL requires for framebuffer flush. The guest allocates a `VkDeviceMemory` backed by a `VIRTIO_GPU_BLOB_MEM_HOST3D_GUEST` blob. When `vkQueuePresentKHR` is called, the Venus ring submits a `VIRTIO_GPU_CMD_SET_SCANOUT_BLOB` command, telling the host VMM to display the blob resource directly on the host's display without any pixel copy. This path achieves display latency approaching native when the host display driver can import the blob as a DMA-BUF. [Source](https://www.collabora.com/news-and-blog/blog/2025/01/15/the-state-of-gfx-virtualization-using-virglrenderer/)

**Latency vs VirGL**: Venus Vulkan approaches native speeds for many workloads because there is no shader re-compilation overhead and the protocol is thinner. VirGL experiences serialised guest-host communication affecting concurrent application performance, and dual compilation costs for every shader program. For real-world workloads the gap is 10–30% throughput in favour of Venus on graphics-centric tests; compute-heavy tests show smaller differences.

---

## 9. WSL2 GPU Architecture

### 9.1 Overview

**WSL2** (Windows Subsystem for Linux 2) runs a real Linux kernel inside a lightweight Hyper-V VM but provides GPU access via a Microsoft-proprietary protocol rather than virtio or VFIO. The design goal was zero-driver-porting cost: Linux applications should see hardware-accelerated GPU access without requiring Linux GPU driver code for the host (Windows) GPU.

### 9.2 dxgkrnl: The Virtual GPU DRM Driver

The `dxgkrnl` module (`drivers/gpu/dxgkrnl/` in the WSL2 Linux kernel branch) creates the `/dev/dxg` device node. It implements a set of ioctls that closely mirror the Windows WDDM D3DKMT (DirectX Graphics Kernel) API — the same kernel interface used by Windows GPU drivers. These ioctls are **not a standard upstream Linux interface**: they are Microsoft-proprietary and differ structurally from DRM ioctls; they do not use GEM objects or DMA-BUF. [Source](https://devblogs.microsoft.com/commandline/wslg-architecture/)

When the Linux application calls `dxgkrnl`, the driver either handles the request locally (for bookkeeping) or forwards it as a Hyper-V VMBUS message to the Windows host, where the request is executed by the Windows GPU driver (WDDM). GPU work happens on the host; results are returned to the guest.

### 9.3 Mesa d3d12 Gallium Driver

Microsoft contributed the `d3d12` Gallium driver to Mesa (first landed in Mesa 21.0). It translates Gallium/OpenGL calls into Direct3D 12 API calls via `libdxcore.so` (a userspace library provided by Microsoft alongside WSL2). The stack is:

```
Linux OpenGL app
    ↓  Mesa EGL/GLX
    ↓  Gallium state tracker
    ↓  d3d12 Gallium driver
    ↓  libdxcore.so (Microsoft proprietary)
    ↓  /dev/dxg ioctl
    ↓  dxgkrnl → Hyper-V VMBUS
    ↓  Windows WDDM GPU driver
    ↓  Physical GPU
```

Hardware-accelerated video decoding was also added by integrating the VAAPI frontend with the d3d12 Gallium driver, enabling Linux media applications using VA-API to reach Windows hardware decode engines. [Source](https://devblogs.microsoft.com/commandline/d3d12-gpu-video-acceleration-in-the-windows-subsystem-for-linux-now-available/)

### 9.4 WSLg: The Display Architecture

**WSLg** provides GUI application support. Its compositor is a **Weston** instance running inside the WSL2 VM (in a dedicated CBL-Mariner "system distro" container), extended with a heavily customised RDP backend implementing the RAIL (Remote Application Integrated Locally) and VAIL (Virtual App Integration Layer) protocols. [Source](https://devblogs.microsoft.com/commandline/wslg-architecture/)

The display path:

```
Linux GUI app (Wayland or X11)
    ↓  Weston compositor (wayland-server in WSL2 VM)
    ↓  VAIL/RDP backend (shared-memory frame capture)
    ↓  RDP stream → Hyper-V VMBUS
    ↓  RDP plugin in Windows Desktop Window Manager
    ↓  Appears as a native Windows window
```

The shared-memory optimisation (VAIL) avoids encoding frames to a video stream when both endpoints are on the same physical machine, giving acceptable latency for development use.

### 9.5 Limitations

The WSL2 GPU path has structural limitations compared to native Linux or VFIO:

- **No DMA-BUF export**: `dxgkrnl` does not implement the DRM prime/DMA-BUF interfaces, so zero-copy sharing with other Linux subsystems (V4L2, VA-API native, Vulkan external memory) is not available via standard mechanisms.
- **No display scanout**: there is no KMS/DRM connector; the Weston compositor always renders to system memory and transfers via RDP.
- **Vulkan compute limitations**: Vulkan ICD support depends on Microsoft's layering through D3D12; not all Vulkan extensions map cleanly. As of 2026, Vulkan 1.3 is supported on most WDDM 3.x drivers but with an incomplete extension set.
- **DirectML**: The DirectML execution provider for ONNX Runtime is the recommended ML inference path in WSL2, using the d3d12 backend to reach the hardware ML engines (Tensor Cores, Matrix Cores) without native ROCm or CUDA support.

---

## 10. GPU Containers: cgroups and Device Isolation

### 10.1 NVIDIA Container Toolkit

The **NVIDIA Container Toolkit** is the standard mechanism for exposing NVIDIA GPUs to OCI containers (Docker, Podman, containerd). Its architecture has evolved from a single OCI hook to a CDI-based model. [Source](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html)

**Legacy mode** (`nvidia-container-runtime-hook`): an OCI `prestart` hook executable that inspects `NVIDIA_VISIBLE_DEVICES` in the container environment, calls `libnvidia-container` to mount the necessary device nodes (`/dev/nvidiaX`, `/dev/nvidia-uvm`, `/dev/nvidia-modeset`) and driver libraries into the container's filesystem namespace.

**CDI mode** (Container Device Interface, currently preferred): uses pre-generated YAML specifications stored in `/etc/cdi/` or `/var/run/cdi/`. The specification declaratively lists the device nodes, environment variables, and bind mounts required for a GPU. Container runtimes that implement CDI (containerd ≥ 1.7, crio, Docker ≥ 25.0) read these specs directly:

```bash
# Generate CDI spec for all NVIDIA GPUs
nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Run a container referencing a GPU by CDI name
docker run --device=nvidia.com/gpu=0 ubuntu nvidia-smi
```

As of NVIDIA Container Toolkit v1.18.0, a systemd service `nvidia-cdi-refresh.service` automatically regenerates CDI specs after driver updates. [Source](https://deepwiki.com/NVIDIA/nvidia-container-toolkit)

```bash
# Expose a specific MIG partition
docker run -e NVIDIA_VISIBLE_DEVICES=MIG-GPU-<uuid>/1/0 \
           nvcr.io/nvidia/pytorch:24.01-py3 python train.py
```

### 10.2 AMD ROCm in Containers

AMD GPU access in containers uses the `/dev/kfd` (the ROCm kernel fusion driver interface) and `/dev/dri/renderD128` (the DRM render node) devices. No proprietary toolkit is required:

```bash
docker run --device /dev/kfd --device /dev/dri \
           --group-add video \
           rocm/rocm-terminal rocminfo
```

The kernel `kfd` driver enforces process-level isolation; multiple containers can share the same physical GPU without interfering with each other's command streams because the KFD tracks per-process queue contexts and enforces memory protection via the IOMMU.

### 10.3 Intel GPU in Containers

Intel GPU access uses the DRM render node only:

```bash
docker run --device /dev/dri/renderD128 \
           intel/intel-extension-for-pytorch:latest
```

For the Intel GPU plugin in Kubernetes, the `intel.com/gpu` resource is advertised per render node.

### 10.4 Kubernetes Device Plugin Framework

Kubernetes exposes GPUs through the **device plugin** API (`DevicePlugin` gRPC interface on `/var/lib/kubelet/device-plugins/`). Each vendor ships a DaemonSet that registers with the kubelet and advertises resources such as `nvidia.com/gpu`, `amd.com/gpu`, or `intel.com/gpu`. [Source](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

```yaml
# Pod requesting one GPU
resources:
  limits:
    nvidia.com/gpu: "1"   # GPU limits must equal requests
```

As of Kubernetes 1.31, **Dynamic Resource Allocation (DRA)** is GA, enabling finer-grained GPU partitioning: a `ResourceClaim` can request a MIG partition, a specific SR-IOV VF, or a time-slice share without a custom device plugin per capability. [Source](https://blog.aks.azure.com/2025/11/17/dra-devices-and-drivers-on-kubernetes)

### 10.5 Device cgroup v2 (eBPF-based)

Linux cgroup v2 replaces the legacy `devices` cgroup subsystem with an eBPF-based policy attached via `BPF_CGROUP_DEVICE`. The container runtime attaches a BPF program to the container's cgroup that allows or denies `open()` calls on specific device major:minor numbers. For MIG, each `nvidia-cap*` capability file has a unique major:minor, so the cgroup policy precisely isolates a container to a single MIG partition without exposing other partitions or the whole GPU.

---

## 11. Performance Comparison and Use-Case Guide

### 11.1 Qualitative Ordering

The table below gives a qualitative ordering of the main approaches across the axes that matter most for deployment decisions. Hard benchmark numbers depend on GPU generation, workload type (graphics vs. compute vs. display), and host configuration; treat this as a guide rather than precise measurements.

| Approach | Compute Throughput | Display Latency | Isolation | Guests per GPU | Use Case |
|---|---|---|---|---|---|
| **VFIO passthrough** | Native (≈100%) | Native (native connector or Looking Glass) | Hardware IOMMU | 1 | Cloud gaming, single-tenant compute |
| **SR-IOV VF** (Xe/AMD) | Near-native (90–98%) | Near-native | Hardware VF isolation | Up to 7 (Xe) / varies (AMD) | VDI, multi-tenant ML training |
| **NVIDIA MIG** | Native per partition | N/A (compute only) | Hardware SM/HBM partition | Up to 7 (A100) | Multi-tenant ML inference |
| **Intel GVT-g** | Moderate (60–80%) | Moderate | Software mediation | Up to 8 | Legacy VDI, Gen8–12 iGPU |
| **Venus (Vulkan)** | 70–90% of native | Moderate | Virtio protocol | Many (VMM-limited) | Dev environments, CI with Vulkan |
| **VirGL (OpenGL)** | 50–70% of native | Moderate | Virtio protocol | Many | OpenGL dev, legacy 3D workloads |
| **virtio-gpu (display only)** | CPU-limited | High (copy-based) | Virtio | Many | Headless servers, VDI text |

Sources: [Collabora VirGL/Venus performance analysis](https://www.collabora.com/news-and-blog/blog/2025/01/15/the-state-of-gfx-virtualization-using-virglrenderer/); [Intel SR-IOV contention measurements](https://www.michaelstinkerings.org/gpu-virtualization-with-intel-12th-gen-igpu-uhd-730/).

The only concrete latency numbers from verified testing: Intel UHD 730 SR-IOV with a single encoding guest ran at <4 ms per frame; adding a second workload raised this to 10–12 ms; a third pushed latency above 30 ms while maintaining throughput. [Source](https://www.michaelstinkerings.org/gpu-virtualization-with-intel-12th-gen-igpu-uhd-730/)

### 11.2 Decision Guide

**Cloud gaming / interactive VDI for a single user**: VFIO passthrough + Looking Glass or a physical display connector. Maximum compatibility with Windows GPU drivers; no multi-tenancy. Pair with OVMF and hugepages.

**Multi-user VDI (workstation graphics)**: Intel Xe SR-IOV (Linux 6.8+) or AMD MxGPU. SR-IOV gives each user a hardware VF with near-native performance. Intel's `i915.max_vfs=7` (or Xe `xe.max_vfs`) enables up to 7 VDI sessions per discrete GPU.

**Multi-tenant ML training**: AMD MxGPU (spatial partition, deterministic memory bandwidth) or NVIDIA MIG (for guaranteed SM+HBM isolation). Both provide fault isolation: a poorly-behaved tenant cannot corrupt another's GPU context.

**Multi-tenant ML inference**: NVIDIA MIG on A100/H100/H200/Blackwell is purpose-built for this, allowing different customers to run inference simultaneously with QoS guarantees. AMD MI300X with MxGPU is the open-driver alternative.

**Development environments**: Venus (Vulkan) or VirGL (OpenGL) inside QEMU gives a reproducible test environment on any host GPU without dedicating hardware per developer. Venus is preferred for Vulkan workloads (lower overhead, no shader re-compilation, Vulkan 1.3 support).

**CI/CD pipelines**: virtio-gpu with virglrenderer suffices for shader compilation and API-level testing. For GPU-timing-sensitive performance tests, use a dedicated VFIO or SR-IOV VM to avoid jitter from time-slicing.

**WSL2 development on Windows**: Use the d3d12 Mesa stack for OpenGL/GLES; DirectML for ONNX Runtime inference. Do not expect DMA-BUF interop, DRM modesetting, or complete Vulkan extension coverage.

### 11.3 Frame Timing Jitter in Cloud Gaming

VFIO passthrough eliminates GPU-side jitter by definition — the guest drives the GPU identically to bare metal. Jitter sources shift entirely to the VM's CPU scheduling (use realtime priorities and CPU pinning), IOMMU page-walk latency (mitigated by hugepages), and the PCIe interrupt path (use MSI-X over INTx). Looking Glass adds a fixed software overhead of ~1 ms for the frame-relay copy; with D12 capture on Windows this is largely eliminated by DMA from the shared memory region.

SR-IOV VF jitter is dominated by the hardware world-switch between VFs and any PCIe contention on shared root ports. For cloud gaming with SR-IOV, schedule only one gaming VF per GPU (using the remaining VFs for lower-priority workloads) to minimise world-switch preemption.

---

## Roadmap

### Near-term (6–12 months)

- **Intel Xe3P SR-IOV on Nova Lake**: Linux 7.2 is expected to enable SR-IOV for Intel Nova Lake S/P integrated graphics via the `xe` driver, extending hardware VF partitioning to the next-generation Xe3P iGPU. [Source](https://www.phoronix.com/news/Intel-Nova-Lake-Graphics-SR-IOV)
- **`VIRTIO_GPU_CAPSET_ROCM` capability set**: A patch series in review on LKML adds a new ROCm capset to `virtio-gpu`, enabling AMD compute workloads (HIP/ROCm) to run accelerated inside VMs using the paravirtual path — the first non-display, non-Vulkan compute forwarding mechanism in the virtio-gpu protocol. [Source](https://lkml.iu.edu/2601.1/09980.html)
- **Venus per-context fence migration**: The virglrenderer Venus renderer is expected to transition to per-context fences only and mandate renderserver initialisation; this removes global fence serialisation and improves multi-VM throughput at the cost of a small API break for integrators. [Source](https://www.collabora.com/news-and-blog/blog/2025/01/15/the-state-of-gfx-virtualization-using-virglrenderer/)
- **Kubernetes DRA and GPU fractional resources**: Dynamic Resource Allocation (DRA, GA in Kubernetes 1.31) is seeing active driver development for NVIDIA MIG partitions and Intel SR-IOV VFs, allowing `ResourceClaim`-based GPU scheduling without per-capability custom device plugins. [Source](https://blog.aks.azure.com/2025/11/17/dra-devices-and-drivers-on-kubernetes)
- **KubeVirt + Intel Graphics SR-IOV Enablement Toolkit**: Intel's `kubevirt-gfx-sriov` project is actively being extended to support Xe-based discrete GPUs alongside existing 12th/13th-gen iGPU SR-IOV, enabling cloud/edge orchestration of graphics VFs inside KubeVirt VMs. [Source](https://github.com/intel/kubevirt-gfx-sriov)

### Medium-term (1–3 years)

- **Unified VFIO/SR-IOV abstraction for NPU devices**: AMD XDNA and Intel Meteor Lake NPU virtualization are expected to follow the VFIO SR-IOV template established for GPUs; IOMMU group isolation, FLR, and VF assignment should map directly once kernel driver support matures. Note: needs verification — no public RFC patchset exists at time of writing.
- **NVIDIA open-kernel vGPU on Blackwell**: As the NVIDIA open-source GPU kernel modules (`nvidia-open`) have achieved feature parity with the proprietary modules for Hopper/Ada and newer, the expectation is that future vGPU Manager releases will be based on the open-source tree, enabling community-auditable virtualisation for Blackwell (`GB100`/`GB200`) architectures. [Source](https://eunomia.dev/blog/2025/10/14/nvidia-open-gpu-kernel-modules-comprehensive-source-code-analysis/)
- **Confidential GPU Computing integration with vGPU**: NVIDIA's CC (Confidential Computing) mode — hardware VRAM encryption, PCIe bus encryption, and attestation — is currently a bare-metal or VFIO-passthrough feature; roadmap work is underway to integrate CC mode with MIG and eventually vGPU Manager so multi-tenant MIG partitions can provide encrypted VRAM per tenant. [Source](https://www.spheron.network/blog/confidential-gpu-computing-nvidia-tee-encrypted-vram/)
- **VirGL replacement / Zink-over-Venus**: As Venus (Vulkan) matures and reaches wider driver coverage, the VirGL (OpenGL/TGSI) path faces deprecation in favour of routing OpenGL through Zink (OpenGL-on-Vulkan) on top of Venus, eliminating the TGSI translation tier. This path is architecturally preferred but requires Venus Vulkan coverage on all host drivers. Note: needs verification on concrete timeline.
- **AMD MxGPU live migration**: Stateful SR-IOV VF migration (live VM migration without dropping the GPU context) is an active research area for AMD MxGPU; AMD has published initial GIM driver changes toward checkpoint/restore of VF state, analogous to CPU live migration. Note: needs verification on upstream patch status.

### Long-term

- **Hardware-native multi-tenant GPU architectures**: Future GPU generations from AMD, Intel, and NVIDIA are expected to expose more granular hardware partitioning — finer than today's MIG 7-way split — potentially allowing O(100) concurrent tenants per die as chiplet-based designs (AMD MI300X-style) become mainstream in datacenter GPUs.
- **GPU TEE interoperability across hypervisors**: Long-term, the GPU TEE ecosystem (AMD SEV-SNP + GPU CC, Intel TDX + GPU extensions) is expected to converge on a common attestation format and VMPL/SEAM integration, allowing portable confidential GPU workloads that can be attested independently of the hypervisor vendor. [Source](https://arxiv.org/html/2408.11601v2)
- **Full virtio-gpu display protocol unification**: The current split between display-only virtio-gpu, VirGL (OpenGL), Venus (Vulkan), and ROCM capsets may converge toward a unified extensible command stream modelled on the Vulkan extension mechanism, with negotiated capability sets replacing the current hard-coded protocol types.
- **Standardised GPU virtualisation at the Khronos level**: Khronos working groups have discussed whether Vulkan Portability or a future "Vulkan Virtualisation" extension family could provide a standardised guest-to-host serialisation protocol, reducing fragmentation between Venus, gfxstream (Android), and potential future implementations. Note: needs verification — no formal Khronos WG proposal exists at time of writing.

---

## 12. Integrations

This chapter builds directly on foundations established across the book. The connections below cross-reference the chapters most relevant to extending or applying the material here.

**Chapter 1 (DRM Architecture)**: `virtio-gpu` is a full DRM driver registering the same DRM/KMS interfaces as `amdgpu` or `i915`. Its `virtio_gpu_driver` probe, GEM object lifecycle, and atomic modesetting callbacks follow exactly the model described in Ch1. Understanding how a "real" driver works is prerequisite to reading the virtio-gpu shim.

**Chapter 3 (GPU Memory — GEM objects across VM boundaries)**: Blob resources in virtio-gpu are GEM objects (`drm_gem_shmem_object`) whose backing store is visible to the host. The memory model for cross-VM-boundary buffer sharing is an extension of the GEM handle/prime model introduced in Ch3.

**Chapter 4 (DMA-BUF)**: The blob resource scanout path in Venus (`VIRTIO_GPU_CMD_SET_SCANOUT_BLOB`) relies on the host VMM importing the blob as a DMA-BUF to hand to the host display driver. The DMA-BUF import/export protocol described in Ch4 is the mechanism that makes zero-copy swapchains possible.

**Chapter 18 (RADV, Vulkan drivers)**: Venus's host-side renderer (virglrenderer) forwards decoded Vulkan calls to whatever host Vulkan ICD is present. On AMD hosts this is RADV; understanding RADV's memory types, queue families, and extension support directly predicts which Venus capabilities are available to the guest.

**Chapter 19 (ANV, Intel Vulkan)**: Same relationship as Ch18 for Intel hosts using ANV. The `VIRTIO_GPU_CONTEXT_TYPE_VENUS` context type selects the host ICD at virglrenderer level; on Intel platforms ANV is the default target.

**Chapter 24 (Vulkan and EGL for application developers)**: Venus guest applications use standard Vulkan and EGL APIs — the virtualization is transparent. Understanding the Vulkan object lifecycle (instances, devices, swapchains) from Ch24 is prerequisite to understanding why Venus serialises those objects across a ring.

**Chapter 55 (GPU Containers and Cloud Compute)**: Chapter 55 covers the operational layer — container runtimes, NVIDIA Container Toolkit, ROCm in Docker, and Kubernetes scheduling. This chapter provides the kernel-level foundation (VFIO, MIG, SR-IOV, device cgroups) that those operational mechanisms rely on. Together they form a complete picture from hardware assignment to pod scheduling.

**Chapter 80 (GPU Security: Isolation, Content Protection, and Confidential Computing)**: IOMMU groups, FLR requirements, and DMA attack vectors covered in §2 of this chapter connect directly to the threat model in Ch80. MIG's hardware isolation guarantees and VFIO's dependence on correct FLR implementation are security properties, not just performance ones. Confidential computing extensions (AMD SEV-SNP + GPU TEE, Intel TDX + NVIDIA CC) extend the trust boundary through the VFIO/MIG boundary.

**Chapter 87 (LLM Inference)**: The NVIDIA MIG partitioning model (§6.2) is the dominant mechanism for multi-tenant LLM inference serving — one H100 can host multiple concurrent inference processes with guaranteed memory and compute isolation. The `NVIDIA_VISIBLE_DEVICES=MIG-GPU-...` environment variable described here is what production inference frameworks use to target a specific MIG partition.

**Chapter 88 (NPU Acceleration)**: As of mid-2026, dedicated NPU accelerators (Intel Meteor Lake NPU, AMD XDNA) do not have virtualisation support equivalent to GPU SR-IOV or MIG. They are typically exposed to a single host process or, in limited cases, passed through to a single VM via VFIO. This is an active area of development; this chapter's VFIO foundations (§2–§3) will be the template for NPU virtualisation as it matures.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
