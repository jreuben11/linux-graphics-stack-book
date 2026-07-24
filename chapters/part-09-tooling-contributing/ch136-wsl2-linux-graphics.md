# Chapter 136: Linux Graphics on WSL2

**Audiences**: Developers running Linux graphics workloads on Windows via WSL2; GPU compute engineers deploying ML inference pipelines on Windows hardware; cross-platform developers targeting both Linux and Windows from a single machine; anyone deploying GPU-accelerated Linux tools — PyTorch, ROCm, Vulkan applications, OpenGL workloads — without leaving the Windows desktop.

---

## Table of Contents

1. [Introduction](#1-introduction)
   - [1.1 What is WSL2?](#11-what-is-wsl2)
   - [1.2 What is GPU Paravirtualisation (GPU-P)?](#12-what-is-gpu-paravirtualisation-gpu-p)
   - [1.3 What is dxgkrnl?](#13-what-is-dxgkrnl)
2. [WSL2 Architecture Overview](#2-wsl2-architecture-overview)
3. [dxgkrnl: The Linux Paravirtual GPU Driver](#3-dxgkrnl-the-linux-paravirtual-gpu-driver)
4. [Mesa D3D12 Gallium Driver](#4-mesa-d3d12-gallium-driver)
5. [Vulkan on WSL2](#5-vulkan-on-wsl2)
6. [CUDA on WSL2](#6-cuda-on-wsl2)
7. [Display and GUI in WSL2 — WSLg](#7-display-and-gui-in-wsl2--wslg)
8. [GPU Compute Workloads in WSL2](#8-gpu-compute-workloads-in-wsl2)
9. [Limitations and Comparison with Native Linux](#9-limitations-and-comparison-with-native-linux)
10. [Future Directions](#10-future-directions)
11. [Integrations](#integrations)

---

## 1. Introduction

**Windows Subsystem for Linux 2 (WSL2)**, introduced with Windows 10 Build 2004 (May 2020) and substantially expanded with the **21H2** release (November 2021), runs a real Linux kernel inside a lightweight **Hyper-V** virtual machine on Windows 10 and Windows 11. Unlike its predecessor WSL1 — which was a translation layer mapping Linux syscalls to Windows NT calls — WSL2 boots an actual Linux kernel binary, providing true system call compatibility at the cost of a thin VM boundary.

The 21H2 release added a landmark capability: **GPU acceleration via a paravirtualized GPU driver stack** that bridges the Windows GPU driver to Linux applications. This enables:

- **OpenGL** and **Vulkan** workloads inside Linux using the physical Windows GPU, mediated by Mesa's D3D12 and `dzn` drivers.
- **OpenCL** and **CUDA** for GPU compute — NVIDIA CUDA runs at near-native performance; AMD ROCm and Intel OneAPI also function via this path.
- **DirectML** for vendor-neutral ML inference across any GPU that has a D3D12-capable Windows driver.
- **WSLg** — a full Wayland compositor that lets Linux GUI applications render on the Windows desktop.

The architecture is unique in the GPU driver ecosystem: the Linux kernel's conventional DRM/KMS driver stack is **entirely absent**. There are no `/dev/dri/` nodes, no GEM buffers, no KMS pipelines. Instead, a purpose-built paravirtual driver called `dxgkrnl` forwards GPU work across a **Hyper-V VMBus channel** to the Windows GPU driver, which does the real scheduling, memory management, and command submission.

This chapter maps the complete technical stack — from the Hyper-V transport layer through the kernel module, Mesa driver, Vulkan implementation, CUDA runtime, and the WSLg display system — so that developers understand both what the stack provides and what it fundamentally cannot provide when compared to native Linux.

The WSL2 kernel is maintained by Microsoft at [github.com/microsoft/WSL2-Linux-Kernel](https://github.com/microsoft/WSL2-Linux-Kernel). As of June 2026, the active branch is **linux-msft-wsl-6.18.y**, tracking upstream Linux 6.18 with Microsoft-specific patches — including `dxgkrnl` — layered on top. The latest release at time of writing is **linux-msft-wsl-6.18.35.2** (June 19, 2026).

### 1.1 What is WSL2?

Windows Subsystem for Linux 2 (WSL2) is a compatibility layer in Windows 10 version 2004 and later that boots a real Linux kernel inside a lightweight Hyper-V virtual machine. Unlike its predecessor WSL1, which translated Linux system calls into Windows NT equivalents, WSL2 runs a genuine Linux kernel binary — currently a Microsoft-maintained fork tracking upstream Linux 6.18 — giving true POSIX and system call compatibility within the constraints of a VM boundary.

WSL2 integrates tightly with the Windows host: the user's home directory is accessible across the boundary, Windows executables can be called from Linux, and Linux processes see host GPU resources via a paravirtualised driver. This design makes WSL2 attractive for workflows that require a Linux toolchain (gcc, gdb, perf, Vulkan SDK, CUDA toolkit) against hardware that has only a Windows driver. From a graphics perspective, WSL2 is notable for what it omits: there is no DRM/KMS stack, no `/dev/dri/card*` device node, and no traditional display pipeline. The GPU is accessed exclusively through a purpose-built paravirtual path described in the rest of this chapter. The WSL2 kernel is maintained at [github.com/microsoft/WSL2-Linux-Kernel](https://github.com/microsoft/WSL2-Linux-Kernel).

### 1.2 What is GPU Paravirtualisation (GPU-P)?

GPU Paravirtualisation (GPU-P) is the mechanism WSL2 uses to expose host GPU capabilities to Linux applications without giving the virtual machine direct hardware access. In contrast to SR-IOV — which partitions the GPU at the hardware level into Virtual Functions — or VFIO passthrough, which grants the VM exclusive PCI ownership, GPU-P keeps the physical GPU under full Windows driver control at all times. The VM submits D3D12 command streams over a VMBus channel; the Windows kernel driver executes them on the GPU hardware and returns results over the same channel. This arrangement allows the host and guest to share the GPU simultaneously — Windows desktop composition, games, and WSL2 compute workloads coexist on the same physical device with time-sliced hardware access managed by the Windows scheduler.

GPU-P requires a Windows Display Driver Model version 2.9 (WDDMv2.9) or later driver on the host. Any discrete GPU with a driver released from 2020 onwards typically satisfies this requirement. From the Linux application developer's perspective, the GPU appears as a render-only device with no display outputs, accessed through the `/dev/dxg` miscdevice node. GPU memory management, command scheduling, and hardware fault recovery remain opaque — handled entirely by the Windows-side driver stack.

### 1.3 What is dxgkrnl?

`dxgkrnl` is the Linux kernel driver that implements the guest side of GPU-P inside the WSL2 virtual machine. It lives at `drivers/hv/dxgkrnl/` in the Microsoft WSL2 kernel fork and registers a miscdevice at `/dev/dxg`. Rather than implementing the DRM subsystem's standard interfaces — there are no GEM buffers, no KMS pipelines, no `drm_driver` struct registration — `dxgkrnl` exposes a WDDM D3DKMT (Kernel Mode Thunk) compatible IOCTL interface that mirrors the API used by `d3d12.dll` on Windows. This deliberate API correspondence means the same D3D12 runtime source can target either the Windows kernel or the Linux VM by changing only the underlying IOCTL dispatch path.

Userspace libraries mounted from the host at `/usr/lib/wsl/lib/` — including `libd3d12.so` and `libdxcore.so` — speak this IOCTL interface, forwarding D3D12 operations as VMBus messages to the Windows kernel component (`dxgkrnl.sys`), which in turn submits work to the physical GPU driver. Mesa's D3D12 Gallium driver and the `dzn` Vulkan driver link against these host-provided libraries, allowing standard OpenGL and Vulkan applications to run on WSL2 without modification. As of mid-2026, `dxgkrnl` is under active review for mainline Linux inclusion (v4 patch series on LKML) but ships only in the Microsoft WSL2 kernel tree.

---

## 2. WSL2 Architecture Overview

### 2.1 The Hyper-V Foundation

WSL2 uses **Hyper-V**, the Windows hypervisor, as its virtualisation substrate. Hyper-V is a Type-1 (bare-metal) hypervisor built into Windows 10/11 and Windows Server. The WSL2 VM is an **Enlightened VM** — it runs synthetic Hyper-V devices rather than emulated legacy hardware, which reduces exit overhead dramatically. The **VMBus** (Virtual Machine Bus) is the high-bandwidth, low-latency transport between the VM and the Windows host for synthetic devices including network, storage, and GPU.

The key Windows components on the host side are:

- **`lxss.sys`**: The Windows kernel driver that manages the WSL2 VM lifecycle — creating, pausing, and terminating the Hyper-V VM on demand.
- **`dxgkrnl.sys`** (the *Windows* `dxgkrnl`, not the Linux one): The DirectX Graphics Kernel scheduler on Windows, which arbitrates GPU access among all processes on the host including WSL2's paravirtualised channel.
- **`dxcore.dll`** / **`libdxcore.so`**: A cross-platform GPU enumeration API (a stripped-down DXGI) designed for headless compute use cases — it does not require a display or swap chain. `dxcore.dll` runs on Windows; `libdxcore.so` is the same API compiled for Linux and mounted into the WSL2 VM.

### 2.2 GPU Paravirtualisation (GPU-P)

WSL2 uses **GPU Paravirtualisation (GPU-P)**, not SR-IOV or full device passthrough. The distinction is important:

| Mechanism | Hardware partitioning | Who schedules? | Requirements |
|---|---|---|---|
| **SR-IOV** | Hardware Virtual Functions (VFs) on PCIe | GPU hardware per VF | GPU SR-IOV support |
| **VFIO passthrough** | Exclusive PCI ownership | VM owns GPU | Only one VM at a time |
| **GPU-P** | Software partition | Windows host driver | WDDMv2.9+ |

With GPU-P, the physical GPU is never handed directly to the VM. The Windows GPU driver (e.g., `nvlddmkm.sys` for NVIDIA, `amdkmdag.sys` for AMD) retains full control of hardware scheduling and memory management. The VM submits D3D12 command streams over VMBus, the Windows driver executes them on the GPU, and results are returned. GPU-P requires **WDDMv2.9** (Windows Display Driver Model version 2.9) or higher — essentially any GPU driver released from 2020 onwards for a discrete GPU.

[Source: DirectX Developer Blog — DirectX ❤ Linux](https://devblogs.microsoft.com/directx/directx-heart-linux/)

All host GPUs running WDDMv2.9 drivers are automatically visible inside WSL2, including multi-GPU systems.

### 2.3 The Two `dxgkrnl` Components

The name `dxgkrnl` appears on both sides of the VM boundary and the distinction is a perennial source of confusion:

- **Windows `dxgkrnl.sys`**: The DirectX Graphics Kernel on Windows — a core OS component that exists on every Windows machine since Vista. It manages GPU command scheduling, GPU VA allocation, and D3D state on the host.
- **Linux `dxgkrnl`** (`drivers/hv/dxgkrnl/`): The paravirtual GPU driver Microsoft wrote for WSL2. It runs inside the Linux VM, creates `/dev/dxg`, and communicates with the Windows side via VMBus. This is not a general-purpose DRM driver; it has no KMS, no GEM, no display pipeline.

The rest of this chapter refers exclusively to the **Linux `dxgkrnl`** unless otherwise noted.

### 2.4 Library Stack at `/usr/lib/wsl/`

When WSL2 boots, the Windows host mounts closed-source, pre-compiled binaries into the VM at two paths:

```
/usr/lib/wsl/lib/          # GPU runtime libraries
  libdxcore.so             # DxCore — GPU enumeration, D3DKMT IOCTLs
  libd3d12.so              # D3D12 runtime for Linux
  libd3d12core.so          # Core D3D12 state
  libDirectML.so           # DirectML ML acceleration
  libcuda.so               # NVIDIA CUDA stub (NVIDIA GPUs only)
  libcuda.so.1             # Symlink
  libnvwgf2umx.so          # NVIDIA UMD (NVIDIA GPUs only)

/usr/lib/wsl/drivers/      # Vendor user-mode drivers (UMDs)
  <vendor-specific .so files compiled for Linux>
```

These libraries are compiled from the **same source code** as their Windows counterparts (`d3d12.dll`, `dxcore.dll`, `directml.dll`) but target Linux ABI (ELF, glibc). They implement the D3D12 and DxCore APIs in Linux userspace, talking to `dxgkrnl` via `/dev/dxg` IOCTLs. Mesa's D3D12 Gallium driver links against `libd3d12.so` and `libdxcore.so` to invoke D3D12.

The `LD_LIBRARY_PATH` must include `/usr/lib/wsl/lib` for these libraries to be found by userspace applications:

```bash
export LD_LIBRARY_PATH=/usr/lib/wsl/lib:$LD_LIBRARY_PATH
```

Modern WSL2 distributions (Ubuntu 22.04 LTS and later) configure this automatically via `/etc/ld.so.conf.d/ld.wsl.conf`.

---

## 3. dxgkrnl: The Linux Paravirtual GPU Driver

### 3.1 Source Location and Upstreaming Status

The Linux `dxgkrnl` driver lives at **`drivers/hv/dxgkrnl/`** in the WSL2 kernel fork. Microsoft has submitted patch series to upstream the driver into the mainline Linux kernel; as of March 2026 the **v4** patch series is under review on the kernel mailing list ([LKML dxgkrnl v4](https://lkml.iu.edu/2603.2/09609.html)), but it has not yet been merged. The driver depends on `CONFIG_HYPERV` and ships only in Microsoft's WSL2 kernel tree.

[Source: Phoronix — Microsoft's DXGKRNL Driver Updated For Linux](https://www.phoronix.com/news/Microsoft-DXGKRNL-Linux-v4)

### 3.2 Source Files

The driver comprises 19 files (~17,400 lines):

```
drivers/hv/dxgkrnl/
  dxgkrnl.c        # Legacy module init entrypoint
  dxgmodule.c      # Driver initialization and loading (971 lines)
  dxgvmbus.c       # VMBus message protocol, channel init (3,992 lines)
  ioctl.c          # IOCTL implementations (5,648 lines)
  dxgadapter.c     # Virtual GPU adapter management (1,367 lines)
  dxgprocess.c     # Per-process GPU context tracking
  dxgsyncfile.c    # DMA fence / sync_file integration (481 lines)
  hmgr.c           # Handle manager (567 lines)
  include/uapi/misc/d3dkmthk.h  # Userspace API definitions (1,794 lines)
```

The `d3dkmthk.h` header defines the IOCTL structures. The API closely mirrors the native **WDDM D3DKMT** (Kernel Mode Thunk) interface on Windows — the same ABI that `d3d12.dll` uses on Windows is re-implemented by `libd3d12.so` on Linux, pointing at `/dev/dxg` instead of the Windows kernel.

### 3.3 Device Node and IOCTL Interface

`dxgkrnl` registers a **miscdevice** at `/dev/dxg`. All GPU operations flow through IOCTLs on this node:

```c
/* d3dkmthk.h — GPU adapter enumeration */
struct d3dkmt_enumadapters3 {
    struct d3dkmt_adaptertype filter;
    __u32 adapter_count;
    struct d3dkmt_adapterinfo adapters[D3DKMT_MAX_ENUM_ADAPTERS];
};
#define LX_DXENUMADAPTERS3 \
    _IOWR('G', 0x01, struct d3dkmt_enumadapters3)
```

Key IOCTLs include:

- **`LX_DXENUMADAPTERS3`**: Enumerate GPU adapters by LUID (Locally Unique Identifier). Each adapter visible on Windows appears here.
- **`LX_DXCREATEDEVICE`**: Create a GPU device context within a virtual adapter.
- **`LX_DXSUBMITCOMMAND`**: Submit a pre-compiled D3D12 command buffer for execution.
- **`LX_DXCREATEALLOCATION`**: Allocate GPU-accessible memory.
- **`LX_DXLOCK2`**: Map GPU memory into CPU-accessible address space.
- **`LX_DXESCAPE`**: Vendor-specific pass-through for GPU vendor IOCTLs (the `IOCTL_DXGK_ESCAPE` mechanism).

The **LUID** (a 64-bit pair `{HighPart, LowPart}`) identifies each GPU adapter and is used consistently between the Windows and Linux sides of the VMBus channel to refer to the same physical device.

Adapters in WSL2 are **render-only** — they have no display outputs (`VidPnSourceId` is zero for all). There is no KMS path, no scanout, and no `DRM_IOCTL_MODE_*`. This is by design: display presentation in WSL2 goes through a completely separate RDP-based path (see Section 7).

### 3.4 VMBus Communication

The heart of `dxgkrnl` is `dxgvmbus.c`, which implements the VMBus channel protocol. Each IOCTL that requires real GPU work translates into a **VMBus message** sent to the Windows host:

```
Linux userspace
  ↓  ioctl(/dev/dxg, LX_DXSUBMITCOMMAND, ...)
dxgkrnl (Linux kernel)
  ↓  VMBus ring buffer write → dxgvmbus.c
Hyper-V VMBus channel
  ↓  host-side message receive
dxgkrnl.sys (Windows kernel)
  ↓  D3D12 command list submission
nvlddmkm.sys / amdkmdag.sys / igdkmd64.sys
  ↓  GPU hardware
Physical GPU
```

The ring buffer implementation in `dxgvmbus.c` includes retry logic for ring buffer saturation — a condition that arises under heavy GPU workloads where the Linux side produces messages faster than the Windows side consumes them. The DMA fence integration in `dxgsyncfile.c` bridges the Windows-side GPU timeline (exposed via Hyper-V channel notifications) to Linux's `SYNC_FILE` infrastructure, enabling proper CPU–GPU synchronisation for compute applications.

### 3.5 Memory Model

GPU memory allocation via `LX_DXCREATEALLOCATION` requests the Windows driver to allocate `ID3D12Resource` objects in the VM's share of GPU memory. Physical memory is managed entirely by the Windows driver — the Linux side never maps GPU VRAM pages directly. CPU access to GPU buffers (for readback, staging) uses `LX_DXLOCK2`, which triggers a Windows-side map of the allocation into a CPU-visible region accessible via `pin_user_pages()` on the Linux side.

Checking dxgkrnl status:

```bash
# Verify dxgkrnl is loaded
dmesg | grep -i dxg
# Expected: [dxgkrnl] DXGK: dxgkrnl adapter detected

# Verify WSL2 kernel
cat /proc/version
# Linux version 6.18.35-microsoft-standard-WSL2 ...

# Check the device node
ls -la /dev/dxg
# crw------- 1 root root 10, 126 Jun 19 2026 /dev/dxg
```

---

## 4. Mesa D3D12 Gallium Driver

### 4.1 Architecture

Microsoft contributes a **Gallium driver for D3D12** to Mesa that provides OpenGL (and OpenCL) by translating Gallium state tracker calls into Direct3D 12 API calls. The driver targets WSL2 as its primary use case — where there is no native Linux GPU driver — but also functions on Windows itself when no native OpenGL driver is available.

[Source: Mesa D3D12 driver documentation](https://docs.mesa3d.org/drivers/d3d12.html)

The architecture layers are:

```
Application (OpenGL API calls)
  ↓
Mesa OpenGL state tracker (st/mesa)
  ↓
D3D12 Gallium driver (src/gallium/drivers/d3d12/)
  ↓  NIR → DXIL shader translation
libd3d12.so  (D3D12 runtime for Linux)
  ↓  D3DKMT IOCTLs
/dev/dxg (dxgkrnl kernel driver)
  ↓  VMBus
Windows GPU driver
  ↓
Physical GPU
```

### 4.2 Key Source Locations in Mesa

```
src/gallium/drivers/d3d12/   # Gallium driver implementation
  d3d12_screen.cpp           # pipe_screen — adapter/caps discovery
  d3d12_context.cpp          # pipe_context — draw/compute dispatch
  d3d12_resource.cpp         # Buffers and textures → ID3D12Resource
  d3d12_query.cpp            # GPU queries (occlusion, timestamp, ...)
  d3d12_draw.cpp             # Draw call translation
  d3d12_compute.cpp          # Compute dispatch

src/microsoft/compiler/      # NIR → DXIL compiler (shared with dzn)
  nir_to_dxil.c              # Main NIR → DXIL translation
  nir_to_dxil.h
  dxil_container.c           # DXIL RIFF container packaging
  dxil_signature.c           # Shader signature generation

src/microsoft/vulkan/        # dzn — Vulkan on D3D12 (see Section 5)
```

### 4.3 Shader Translation: NIR → DXIL

Mesa's standard intermediate representation is **NIR** (the New Intermediate Representation). The D3D12 and dzn drivers share the **`nir_to_dxil`** compiler in `src/microsoft/compiler/` that translates NIR to **DXIL** (DirectX Intermediate Language) — the bytecode format consumed by D3D12 drivers.

The translation path for a vertex shader:

```
GLSL source
  ↓  glsl_to_nir()
NIR (Mesa IR)
  ↓  nir_to_dxil()       (src/microsoft/compiler/nir_to_dxil.c)
DXIL bytecode blob
  ↓  dxil_wrap_module()  — RIFF container
ID3D12PipelineState
```

D3D12 requires all shaders to be cryptographically signed before use. Mesa includes `dxil_sign_blob()` which performs the signing step required by the D3D12 runtime before a shader can be compiled into a pipeline state object. Without signing, D3D12 will reject the DXIL blob.

[Source: Mesa nir_to_dxil.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/a9b6a54a8cce0aab44c81ea4821ee564b939ea51/src/microsoft/compiler/nir_to_dxil.c)

### 4.4 Core Data Structures

```c
/* d3d12_screen.cpp — the Gallium pipe_screen for D3D12 */
struct d3d12_screen {
    struct pipe_screen base;           /* Gallium base */
    ID3D12Device2 *dev;                /* D3D12 device */
    IDXGIAdapter *adapter;             /* DxCore adapter */
    struct d3d12_descriptor_pool *rtv_pool;  /* Render target views */
    struct d3d12_descriptor_pool *dsv_pool;  /* Depth/stencil views */
    struct d3d12_descriptor_pool *srv_pool;  /* Shader resource views */
    struct d3d12_descriptor_pool *sampler_pool;
    struct d3d12_memory_info memory_info;
};

/* d3d12_resource.cpp — buffer/texture → ID3D12Resource */
struct d3d12_resource {
    struct threaded_resource base;     /* Gallium base */
    ID3D12Resource *bo;                /* The D3D12 allocation */
    DXGI_FORMAT dxgi_format;
    D3D12_RESOURCE_STATES first_state;
    D3D12_RESOURCE_DESC desc;
};
```

The `d3d12_screen` calls `D3D12CreateDevice()` via `libd3d12.so` at Mesa driver initialization. It uses `DxCoreAdapterFactory` from `libdxcore.so` to enumerate available adapters — the same enumeration path available to any D3D12 application on Linux or WSL2.

### 4.5 Runtime Configuration

```bash
# Select a specific GPU by name (case-insensitive substring match)
export MESA_D3D12_DEFAULT_ADAPTER_NAME="NVIDIA"

# Fallback to software (llvmpipe/softpipe) if D3D12 unavailable
export LIBGL_ALWAYS_SOFTWARE=1

# Debug: dump DXIL bytecode during shader compilation
export D3D12_DEBUG=dxil

# Debug: show DXIL disassembly
export D3D12_DEBUG=dxil,disass

# Debug: verbose resource tracking
export D3D12_DEBUG=res,blit

# Debug DXIL shader compiler internals
export DXIL_DEBUG=dump_blob,trace
```

### 4.6 OpenGL Version and Limitations

Mesa's D3D12 Gallium driver achieves **OpenGL 4.6** on D3D12 Feature Level 12.0+ hardware, though some extensions have coverage gaps due to D3D12 semantics differences. Specific limitations:

- No `GL_ARB_sparse_buffer` — D3D12 sparse resources use a different commitment model.
- `glTextureView` requires careful format aliasing support in D3D12, which varies by vendor UMD.
- `glBegin`/`glEnd` immediate mode is emulated via softpipe fallback inside WSL2 (rare in practice).
- No `GL_NV_*` or `GL_AMD_*` vendor extensions — the D3D12 translation layer has no path to vendor-specific hardware features.

### 4.7 OpenCL on WSL2

Mesa also provides an **OpenCL** Gallium driver via `clover` and the `CLOn12` path (using `libOpenCL.so` → Mesa Clover → D3D12). The `libOpenCL.so` in `/usr/lib/wsl/lib/` implements the OpenCL ICD loader that dispatches to the D3D12 compute path, enabling OpenCL workloads without native Linux compute drivers.

---

## 5. Vulkan on WSL2

WSL2 supports three distinct Vulkan paths, each with different capabilities and use cases.

### 5.1 Path A: Microsoft's `dzn` (Dozen) Driver

**`dzn`** is Mesa's Vulkan implementation built on top of D3D12, written by Collabora under contract for Microsoft. It lives at `src/microsoft/vulkan/` in Mesa and was merged into Mesa 22.1.

[Source: Phoronix — "Dozen" Merged Into Mesa](https://www.phoronix.com/news/Vulkan-On-Direct3D-12-Dzn-Merge)

Key source files:

```
src/microsoft/vulkan/
  dzn_device.c         # VkDevice, VkPhysicalDevice, VkInstance
  dzn_cmd_buffer.c     # VkCommandBuffer — maps Vulkan cmds to D3D12
  dzn_pipeline.c       # Graphics and compute pipelines
  dzn_image.c          # VkImage → ID3D12Resource
  dzn_buffer.c         # VkBuffer → ID3D12Resource
  dzn_wsi.c            # WSI — swapchain via WSLg/RDP path
```

The adapter enumeration calls `dxcore` to list D3D12 adapters:

```c
/* dzn_device.c — enumerate D3D12 adapters as VkPhysicalDevices */
static VkResult
dzn_physical_device_enumerate(struct dzn_instance *instance)
{
    IDXCoreAdapterFactory *factory;
    DXCoreCreateAdapterFactory(IID_IDXCoreAdapterFactory, &factory);

    IDXCoreAdapterList *adapter_list;
    factory->CreateAdapterList(1, &DXCORE_ADAPTER_ATTRIBUTE_D3D12_GRAPHICS,
                               IID_IDXCoreAdapterList, &adapter_list);

    uint32_t count = adapter_list->GetAdapterCount();
    for (uint32_t i = 0; i < count; i++) {
        IDXCoreAdapter *adapter;
        adapter_list->GetAdapter(i, IID_IDXCoreAdapter, &adapter);
        dzn_physical_device_create(instance, adapter, ...);
    }
}
```

Vulkan command buffers in dzn translate to D3D12 command lists:

```c
/* dzn_cmd_buffer.c — Vulkan draw → D3D12 DrawInstanced */
void
dzn_cmd_buffer_record_draw(struct dzn_cmd_buffer *cmdbuf,
                           uint32_t vertex_count,
                           uint32_t instance_count,
                           uint32_t first_vertex,
                           uint32_t first_instance)
{
    cmdbuf->cmdlist->DrawInstanced(vertex_count, instance_count,
                                   first_vertex, first_instance);
}
```

**dzn Vulkan version and conformance** (as of mid-2026):

- Achieves **99.75%+** of Vulkan 1.0 CTS conformance.
- Targets **Vulkan 1.2**, with ongoing work on 1.3 features.
- The driver displays a conformance warning in `vulkaninfo` output: `"dzn is not a conformant Vulkan implementation, testing use only"` for non-shipped configurations.
- `VK_KHR_external_memory` and `VK_KHR_external_semaphore` work via D3D12 shared handles.

**D3D12-induced Vulkan limitations**:

- No `VkDisplayKHR` — D3D12 has no display enumeration in headless mode.
- No `VK_KHR_present_mode_fifo_relaxed` — presentation timing is managed by WSLg.
- `VkPhysicalDeviceVulkan12Features.shaderSubgroupExtendedTypes` requires D3D12 SM 6.0 wave ops.
- Sparse binding (`VK_EXT_buffer_device_address` sparse) limited by D3D12 sparse tier.

### 5.2 Path B: Vendor-Native Vulkan via GPU-P

GPU vendors provide their own Linux-compiled Vulkan ICDs for WSL2, which bypass Mesa entirely and communicate with the Windows Vulkan driver directly through `dxgkrnl`:

**NVIDIA**: Provides `libvulkan_nvidia.so` as part of the CUDA WSL2 driver package. The NVIDIA Vulkan ICD in WSL2 is the full NVIDIA Linux Vulkan driver compiled to use `/dev/dxg` as its backend instead of `/dev/nvidiactl`. It supports the complete NVIDIA Vulkan extension set.

**AMD**: Provides a WSL2-specific Vulkan ICD (`libvulkan_amdgpu.so`) as part of the ROCm for WSL2 stack (AMD Software: Adrenalin Edition 26.1.1+). It communicates through `dxgkrnl` to `amdkmdag.sys` on the Windows side.

**Intel**: Intel's Vulkan driver for WSL2 is available via the Intel OneAPI toolkit and targets Intel Arc and Iris Xe GPUs.

### 5.3 Path C: SwiftShader Software Fallback

When no GPU or GPU-P is available (older Windows versions, or Hyper-V configurations without GPU-P), Mesa's `libvk_swiftshader.so` (the CPU-based Vulkan implementation) provides a software fallback. This is the same SwiftShader used by Chrome's WebGPU for software rendering.

### 5.4 Examining Vulkan in WSL2

```bash
# List all Vulkan ICDs found
vulkaninfo --summary

# Typical output on NVIDIA WSL2:
# GPU id : 0 (NVIDIA GeForce RTX 4090)
#   apiVersion = 1.3.260
#   driverVersion = 560.35.03
# GPU id : 1 (Microsoft Direct3D12 (NVIDIA GeForce RTX 4090))
#   apiVersion = 1.2.0
#   driverVersion = 22.1.0
```

The duplication (native NVIDIA ICD + dzn ICD for the same GPU) is expected. The `VK_ICD_FILENAMES` environment variable controls which ICD is used; the Vulkan loader normally selects the highest-capability one.

---

## 6. CUDA on WSL2

### 6.1 Architecture

NVIDIA CUDA on WSL2 provides **full CUDA support** via a purpose-built library stack. The key insight is that the Linux CUDA runtime (`libcuda.so`) in WSL2 is **not** the native Linux CUDA driver — it is a **stub** compiled by NVIDIA that translates CUDA Driver API calls into D3DKMT IOCTLs on `/dev/dxg`, which are forwarded via VMBus to `nvlddmkm.sys` on the Windows host.

[Source: NVIDIA CUDA on WSL User Guide](https://docs.nvidia.com/cuda/wsl-user-guide/index.html)

```
Python (torch.cuda) / C (libcuda.so API)
  ↓
libcuda.so  (NVIDIA's WSL2 stub — NOT the native Linux CUDA driver)
  ↓  D3DKMT IOCTLs
/dev/dxg (dxgkrnl)
  ↓  VMBus
nvlddmkm.sys (Windows NVIDIA driver)
  ↓
GPU hardware
```

**Critical rule**: **Do not install the native NVIDIA Linux GPU driver inside WSL2.** Doing so overwrites `libcuda.so` with the native Linux version, which expects `/dev/nvidiactl` (absent in WSL2) and will fail. The `libcuda.so` at `/usr/lib/wsl/lib/libcuda.so` is mounted by Windows and must not be replaced.

### 6.2 Library Setup

```bash
# The WSL2 NVIDIA stack lives at:
ls /usr/lib/wsl/lib/
# libcuda.so  libcuda.so.1  libnvcuvid.so  libnvidia-encode.so
# libcuda.so.1 → libcuda.so (symlink)
# libdxcore.so  libd3d12.so  ...

# Confirm LD_LIBRARY_PATH includes WSL libraries
echo $LD_LIBRARY_PATH
# /usr/lib/wsl/lib:...

# Install CUDA Toolkit (NOT the full Linux NVIDIA driver)
# Use the WSL-Ubuntu variant:
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-4

# Verify CUDA works
nvidia-smi
# +-----------------------------------------------------------------------------------------+
# | NVIDIA-SMI 565.90                 Driver Version: 565.90         CUDA Version: 12.7    |
# |-------------------------------+----------------------+----------------------+
# | GPU  Name           Persistence-M | Bus-Id          Disp.A | Volatile Uncor. ECC |
# |===============================+======================+======================|
# |   0  NVIDIA GeForce RTX ...  Off  | 00000000:00:00.0 Off  |                  N/A |
```

### 6.3 NVML and nvidia-smi in WSL2

**NVML** (NVIDIA Management Library) works in WSL2 via `libdxcore.so` bridging NVML queries to the Windows D3D12 GPU management API. However, several NVML capabilities are unavailable:

- GPU utilization query (`nvmlDeviceGetUtilizationRates`) — **not available** in WSL2.
- Active process list (`nvmlDeviceGetComputeRunningProcesses`) — **not available**.
- ECC, persistence mode, compute mode modifications — **not available** (Windows driver controls these).
- GPU reset (`nvidia-smi --gpu-reset`) — **not available** (Windows controls GPU lifecycle).
- Clock lock (`nvidia-smi -lgc`) — **not available**.

What works: reading GPU temperature, memory usage (with reduced granularity), total/free VRAM, driver version, and CUDA version.

### 6.4 PyTorch and Deep Learning

```bash
# Install PyTorch with CUDA support for WSL2
pip install torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/cu124

# Verify GPU access
python3 -c "import torch; print(torch.cuda.is_available())"
# True

python3 -c "import torch; print(torch.cuda.get_device_name(0))"
# NVIDIA GeForce RTX 4090

# Run a quick benchmark
python3 -c "
import torch, time
x = torch.randn(4096, 4096, device='cuda')
t = time.perf_counter()
for _ in range(100):
    y = x @ x.T
torch.cuda.synchronize()
print(f'GEMM: {(time.perf_counter() - t)*10:.1f} ms/iter')
"
```

**Known CUDA limitations in WSL2**:

- **Unified Memory (UVM)** is not fully supported. `cudaMallocManaged()` works on some configurations but concurrent CPU/GPU access is unavailable.
- **Pinned memory** (`cudaMallocHost`) is available but the pool is limited compared to native Linux.
- **Pascal (GTX 10xx)** is the minimum; Maxwell and older are unsupported.
- **GPU Direct RDMA** — not available in WSL2 (requires direct PCIe DMA access).
- Individual GPU selection via device index works (`CUDA_VISIBLE_DEVICES=0`); filtering by UUID works with native NVIDIA ICD but not through the D3D12 path.

### 6.5 Performance Characteristics

For compute-bound workloads (GEMM, convolutions, FFTs) the WSL2 overhead is in the 0–5% range compared to native Linux, because the command submission path is the primary overhead and VMBus adds only microseconds of latency. Performance comparisons from 2025–2026 testing on RTX 4090:

| Workload | Native Linux | WSL2 | Overhead |
|---|---|---|---|
| cuBLAS SGEMM 8192×8192 | 100% | 97–100% | 0–3% |
| PyTorch ResNet-50 training | 100% | 95–99% | 1–5% |
| CUDA graph workloads | 100% | 90–95% (before WSL2 2.7) / 97% (after) | 3–10% |
| Video encode (NVENC) | 100% | ~90% | ~10% |

WSL2 2.7.0 shipped targeted dxgkrnl improvements for NVIDIA Blackwell (RTX 5000 series) that reduced CUDA graph capture time dramatically — from ~90s to ~25s — in workloads using `torch.compile`.

---

## 7. Display and GUI in WSL2 — WSLg

### 7.1 WSLg Architecture

**WSLg** (Windows Subsystem for Linux GUI) provides a full graphical environment for Linux applications running in WSL2. It was integrated into Windows 11 and backported to Windows 10 (21H2). WSLg runs as a **separate system distribution** (based on CBL-Mariner) inside the same Hyper-V VM, isolated from user distributions, providing consistent display services regardless of which Linux distribution the user runs.

[Source: WSLg Architecture — Windows Command Line Blog](https://devblogs.microsoft.com/commandline/wslg-architecture/)

The WSLg stack:

```
Linux GUI Application (Wayland or X11)
  ↓  Wayland protocol / XWayland
Weston (Wayland compositor) — runs in WSLg system distro
  ↓  RDP backend (FreeRDP-based)
RDP server → Hyper-V socket (AF_VSOCK, no network)
  ↓  
mstsc.exe / RAIL+VAIL RDP client on Windows
  ↓
Windows DWM (Desktop Window Manager)
  ↓
Windows GPU (display output)
```

Key components:

- **Weston**: The reference Wayland compositor, with Microsoft's custom RDP backend. The Wayland frontend is largely unmodified — it implements standard Wayland protocols, handling both Wayland-native and XWayland (X11) clients.
- **XWayland**: Runs inside Weston to support X11 applications. `DISPLAY=:0` points to this XWayland server.
- **FreeRDP**: Microsoft extended Weston's RDP backend using FreeRDP to support **RAIL** (Remote Application Integrated Locally) and **VAIL** (Virtualized Application Integrated Locally) modes, which allow individual application windows to appear on the Windows desktop rather than in a full virtual desktop.
- **Hyper-V socket (AF_VSOCK)**: The RDP traffic between Weston and the Windows RDP client flows over a `vsock` connection through the Hyper-V fabric — not the network stack — minimising latency.

### 7.2 GPU Acceleration in WSLg

Linux applications using OpenGL in WSLg use the **Mesa D3D12 Gallium driver** for rendering. The display pipeline then involves:

```
GL application → Mesa D3D12 → GPU render (via dxgkrnl/VMBus)
  ↓  Rendered framebuffer
  GPU VRAM → PCIe → System RAM (readback for discrete GPU)
  ↓
Weston compositor → RDP encoder (FreeRDP)
  ↓  AF_VSOCK RDP stream
Windows RDP client → DWM → Screen
```

This introduces a notable cost for **discrete GPUs**: rendered content must be read back from VRAM to system memory (a PCIe transfer) before being fed to the RDP encoder. At 60 FPS this is 60 readbacks per second. **Integrated GPUs** (Intel Iris Xe, AMD iGPU in Ryzen) share memory between CPU and GPU, so no PCIe readback occurs, making WSLg performance better relative to native for integrated GPU workloads.

[Source: GPU Acceleration — WSLg DeepWiki](https://deepwiki.com/microsoft/wslg/5.3-gpu-acceleration)

### 7.3 Environment Setup

```bash
# Wayland-native applications
export WAYLAND_DISPLAY=wayland-0
export XDG_RUNTIME_DIR=/mnt/wslg/runtime-dir   # Set by WSL2

# X11 applications (via XWayland inside Weston)
export DISPLAY=:0

# Test Wayland display
wayland-info   # Shows compositor capabilities

# Test OpenGL acceleration
glxinfo | grep "OpenGL renderer"
# OpenGL renderer string: D3D12 (NVIDIA GeForce RTX 4090)

# Test Vulkan
vulkaninfo --summary

# Software fallback (if GPU issues occur)
export LIBGL_ALWAYS_SOFTWARE=1

# Test a Wayland application
weston-terminal &

# GTK4 Wayland application
GDK_BACKEND=wayland gedit &
```

### 7.4 WSLg Limitations

- **No VSync / tearing control**: The RDP path does not expose `VK_KHR_present_mode_fifo` or equivalent control to applications. Frame pacing is managed by Weston and RDP timing, not by the application.
- **No hardware overlay planes**: All compositing happens in software (Weston + RDP). There is no direct scanout path.
- **Frame rate cap**: RDP encoding typically caps at 60 FPS for interactive use; high-frame-rate gaming (144+ FPS) is not meaningful through WSLg.
- **Audio**: PulseAudio runs in the WSLg system distro and routes audio through an RDP audio channel to the Windows audio stack.
- **Clipboard**: Bidirectional clipboard (text, HTML, image) is supported via RDP clipboard extension.
- **Multi-monitor**: Supported via per-monitor DPI scaling through the RDP multi-monitor extension.
- **`gbm_device`**: Not available. EGL with `EGL_PLATFORM_GBM` fails — headless EGL must use `EGL_PLATFORM_SURFACELESS_MESA` or `EGL_PLATFORM_DEVICE_EXT` via the D3D12 path.

---

## 8. GPU Compute Workloads in WSL2

### 8.1 DirectML — Vendor-Neutral ML Acceleration

**DirectML** is Microsoft's ML acceleration library (`libDirectML.so`, mounted at `/usr/lib/wsl/lib/`) that works on **any GPU with a D3D12 driver** — NVIDIA, AMD, Intel, and Qualcomm. It provides a set of GPU-accelerated ML primitives (convolution, matrix multiply, attention) implemented via D3D12 compute shaders.

```bash
# Install ONNX Runtime with DirectML execution provider
pip install onnxruntime-directml

# Run inference with DirectML EP
python3 - <<'EOF'
import onnxruntime as ort
providers = ['DmlExecutionProvider', 'CPUExecutionProvider']
sess = ort.InferenceSession("model.onnx", providers=providers)
print(sess.get_providers())
# ['DmlExecutionProvider', 'CPUExecutionProvider']
EOF

# PyTorch via DirectML (all GPUs, including AMD and Intel)
pip install torch-directml
python3 - <<'EOF'
import torch, torch_directml
dml = torch_directml.device()
x = torch.randn(1024, 1024, device=dml)
y = x @ x.T
print(y.shape)  # torch.Size([1024, 1024])
EOF
```

DirectML is particularly valuable for AMD and Intel GPU users on WSL2 who cannot use CUDA or ROCm — any D3D12-capable GPU works without additional vendor-specific drivers inside Linux.

### 8.2 AMD ROCm on WSL2

AMD provides WSL2-compatible ROCm as part of **AMD Software: Adrenalin Edition 26.1.1** (and later). Installation:

```bash
# Install AMD GPU driver for WSL2 on the Windows host first:
# AMD Software: Adrenalin Edition 26.1.1 or later

# Inside WSL2 (Ubuntu 22.04/24.04):
sudo amdgpu-install --usecase=wsl,rocm --no-32

# Verify ROCm sees the GPU
rocm-smi
# ========================= ROCm System Management Interface =========================
# ============================== GPU Info =======================================
# GPU[0]  : Device Name: AMD Radeon RX 7900 XTX
# GPU[0]  : ...

# HIP program compilation
hipcc -o hello_hip hello_hip.cpp
./hello_hip
```

ROCm on WSL2 uses the same dxgkrnl → VMBus → `amdkmdag.sys` path as D3D12 OpenGL and Vulkan. The `rocm-smi` tool shows GPU metrics via NVML-equivalent AMD SMI APIs routed through `libdxcore.so`.

### 8.3 Intel OneAPI on WSL2

Intel OneAPI (including DPC++, oneMKL, and OpenCL) works on WSL2 with Intel Iris Xe or Arc GPUs:

```bash
# Install Intel OneAPI toolkit
# Follow: https://www.intel.com/content/www/us/en/docs/oneapi/installation-guide-linux/2025-0/configure-wsl-2-for-gpu-workflows.html
source /opt/intel/oneapi/setvars.sh

# Intel GPU Top (performance monitoring)
intel_gpu_top

# Performance event access (perf_event_paranoid for VTune)
cat /proc/sys/kernel/perf_event_paranoid
# 1   (WSL2 allows perf events at non-root level)
sudo sysctl -w kernel.perf_event_paranoid=0
```

### 8.4 Containers with GPU in WSL2

Docker Desktop's WSL2 backend supports GPU acceleration:

```bash
# NVIDIA GPU in Docker via WSL2
docker run --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi

# GPU selection (note: per-GPU selection limited in WSL2)
docker run --gpus all \
    -e NVIDIA_VISIBLE_DEVICES=0 \
    pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime \
    python -c "import torch; print(torch.cuda.is_available())"

# NVIDIA Container Toolkit must be v2.6.0+ for WSL2
nvidia-ctk --version
```

The `nvidia-container-toolkit` in WSL2 injects `/dev/dxg` (rather than `/dev/nvidiactl`) into containers, along with the WSL2 library paths. Individual GPU device filtering (`--gpus device=0`) is **not available** in WSL2 multi-GPU configurations — `--gpus all` is the supported form.

### 8.5 Performance Monitoring

```bash
# GPU utilization (NVIDIA — limited in WSL2)
nvidia-smi dmon -s u

# GPU memory usage
nvidia-smi --query-gpu=memory.used,memory.total \
    --format=csv,noheader

# AMD GPU monitoring
rocm-smi --showuse --showmemuse

# Linux perf (works inside WSL2 with some PMU restrictions)
perf stat -e cycles,instructions,cache-misses -- ./benchmark

# Note: Hardware PMU counters may have reduced access in WSL2
# Some events require: echo 0 > /proc/sys/kernel/perf_event_paranoid
```

---

## 9. Limitations and Comparison with Native Linux

### 9.1 Structural Absences

WSL2's GPU stack lacks several fundamental Linux graphics subsystem components that native Linux takes for granted:

| Feature | Native Linux | WSL2 |
|---|---|---|
| `/dev/dri/card0` (KMS primary node) | Yes | **No** |
| `/dev/dri/renderD128` (render node) | Yes | **No — use `/dev/dxg`** |
| `/dev/kfd` (AMD compute) | Yes | No (ROCm uses `/dev/dxg`) |
| `/dev/input/event*` (evdev input) | Yes | **No** |
| GEM buffer objects | Yes | No (D3D12 resources) |
| DRM/KMS display pipeline | Yes | **No** |
| `gbm_device` (GBM for EGL) | Yes | **No** |
| `VkDisplayKHR` | Yes | No |
| Direct scanout / page flip | Yes | No |
| Hardware overlay planes | Yes | No |
| GPU reset (wedge recovery) | Yes | No |
| SR-IOV VF isolation | Configurable | No |
| `perf` hardware PMU events | Full access | Restricted |
| `nvidia-smi --gpu-reset` | Yes (nvidia) | **No** |

### 9.2 GPU Memory Sharing with Windows

GPU memory (VRAM) is **shared** between Windows and the WSL2 VM. Windows reserves memory for:

- The Windows DWM compositor.
- All other Windows applications using the GPU.
- Driver overhead.

What remains is available to WSL2. On a GPU with 8 GB VRAM, WSL2 may see only 4–6 GB available depending on Windows-side usage. This is fundamentally different from native Linux where the OS boots with GPU-only use cases having nearly full VRAM.

There is no VRAM reservation guarantee for the WSL2 partition. If Windows starts a GPU-intensive task (a game, 3D application, or GPU-accelerated browser), it competes with the WSL2 workload for VRAM. Out-of-memory on the GPU side manifests as D3D12 allocation failures inside `libd3d12.so`, surfaced to the application as `CUDA_ERROR_OUT_OF_MEMORY` or `VK_ERROR_OUT_OF_DEVICE_MEMORY`.

### 9.3 GPU-P vs SR-IOV

GPU-P (what WSL2 uses) and SR-IOV (used by some cloud hypervisors and Xe) differ in isolation level:

- **SR-IOV**: Hardware-enforced Virtual Function with dedicated VRAM partition and compute time-slice. One VF cannot interfere with another at the hardware level.
- **GPU-P**: Software virtualisation. The Windows GPU driver schedules all GPU contexts (Windows apps + WSL2). A misbehaving Windows application can starve WSL2 GPU time. GPU resets triggered by a GPU hang on the Windows side terminate all GPU contexts including WSL2.

### 9.4 Diagnosing WSL2 GPU Issues

```bash
# Verify dxgkrnl is active
dmesg | grep -i dxg
# [    2.314] dxgkrnl: DXGK: dxgkrnl adapter 0 detected
# [    2.315] dxgkrnl: DXGK: using compute-only adapter

# Check WSL2 kernel version
cat /proc/version
# Linux version 6.18.35-microsoft-standard-WSL2 (...)

# Enumerate GPU adapters via libdxcore
python3 -c "
import ctypes, ctypes.util
# Or use a test tool:
" 

# List GPU adapters with d3d12test (if installed)
d3d12test --list-adapters

# Check WSL2 version on Windows side (PowerShell):
# wsl --version
# WSL version: 2.7.0
# Kernel version: 6.18.35.2

# Common: Mesa D3D12 driver not finding GPU
MESA_D3D12_DEFAULT_ADAPTER_NAME="" LIBGL_DEBUG=verbose glxinfo 2>&1 | head -20

# Check if GPU is visible to Mesa
glxinfo -B | grep Device
# Device: D3D12 (NVIDIA GeForce RTX 4090) (0x...)
```

---

## 10. Future Directions

### 10.1 dxgkrnl Mainline Upstreaming

The v4 dxgkrnl patch series ([LKML discussion](https://lwn.net/Articles/881311/)) represents Microsoft's most substantial attempt to upstream the driver into mainline Linux. Key concerns from kernel maintainers include: the driver's tight coupling to the Windows-specific WDDM D3DKMT ABI, the limited utility outside Hyper-V/WSL2 environments, and the absence of DRM integration. Whether dxgkrnl will be accepted into the mainline tree remains open as of mid-2026.

[Source: LWN.net — drivers: hv: dxgkrnl overview](https://lwn.net/Articles/881311/)

### 10.2 Virtio-GPU Rutabaga

Intel's **virtio-gpu rutabaga** framework — a standards-based paravirtualised GPU transport using the VirtIO specification — has been merged into upstream Linux and QEMU. Rutabaga provides a path for any hypervisor (not just Hyper-V) to expose GPU capabilities to VMs using a vendor-neutral protocol. Microsoft's Patrick Wu acknowledged at Build 2026 that rutabaga could eventually replace dxgkrnl, though no timeline was committed. If adopted, WSL2 would gain a GPU driver present in the mainline kernel without custom Microsoft patches, improving maintenance and distribution compatibility.

[Source: Windows News — Microsoft Revives DXGKRNL After Years of Quiet Development](https://windowsnews.ai/article/microsoft-revives-dxgkrnl-for-linux-gpu-virtualization-in-wsl2-after-years-of-quiet-development.406197)

### 10.3 dzn Vulkan Conformance Progress

Microsoft's dzn Vulkan driver continues to improve toward Vulkan 1.3 full conformance. The trajectory suggests that within the 2026–2027 timeframe, dzn may achieve the status of a conformant Vulkan implementation, removing the testing-only caveat and enabling Vulkan applications to rely on dzn in WSL2 without per-application workarounds.

---

## Roadmap

### Near-term (6–12 months)

- **dxgkrnl v4 mainline review resolution**: The v4 patch series (55 patches) submitted to LKML in March 2026 is under active review. Key reviewer concerns — WDDM D3DKMT ABI coupling, absence of DRM integration, and Hyper-V-only utility — must be addressed before acceptance. A v5 reroll addressing these issues is expected; outcome could be conditional merge under `drivers/hv/` or continued deferral. [Source](https://lkml.iu.edu/2603.2/09609.html)
- **WSL2 kernel 6.18.y stability series**: The active `linux-msft-wsl-6.18.y` branch continues to ship stable-kernel backports (e.g., the dxgkrnl sync fix in 6.18.33.1). Point releases are expected to continue at roughly monthly cadence through late 2026. [Source](https://windowsnews.ai/article/wsl-kernel-618331-delivers-critical-dxgkrnl-sync-fix-and-linux-61833-update.423194)
- **dzn Vulkan 1.3 conformance push**: Microsoft's dzn ("Dozen") Vulkan-on-D3D12 driver continues accumulating extensions and conformance test fixes. The trajectory targets full Vulkan 1.3 conformance in Khronos CTS, removing the testing-only advisory and making dzn a first-class Vulkan ICD for WSL2 workloads. Note: no official milestone date is published.
- **Mesa 26.x D3D12 Gallium improvements**: Mesa 26.1 (May 2026) shipped additional Vulkan and OpenGL extension coverage across the D3D12 backend; Mesa 26.2 and 26.3 are expected to close remaining OpenGL 4.6 feature gaps and improve shader compilation throughput via the NIR→DXIL pipeline. [Source](https://www.linuxcompatible.org/story/mesa-2610-released)
- **WSLg RDP presentation path optimisation**: WSLg's architecture routes rendered frames through an RDP stream back to the Windows compositor, incurring a VRAM→system-memory copy. Near-term work is expected to reduce this overhead via shared-memory presentation on unified-memory architectures (integrated GPUs, NPUs). [Source](https://github.com/microsoft/wslg)

### Medium-term (1–3 years)

- **Virtio-GPU rutabaga as a potential dxgkrnl successor**: The Rust-based `rutabaga_gfx` framework — a standards-conformant VirtIO-GPU transport supporting Vulkan via host-visible memory and gfxstream — is tracked in upstream QEMU and the Linux kernel. Microsoft engineers have discussed rutabaga as a longer-term replacement for the bespoke dxgkrnl VMBus protocol, which would give WSL2 a mainline-kernel GPU driver without vendor-specific patches. [Source](https://github.com/magma-gpu/rutabaga_gfx/issues/24)
- **SR-IOV GPU partitioning for WSL2**: Current GPU-P uses software partitioning (WDDMv2.9); Microsoft and GPU vendors are evaluating hardware SR-IOV virtual functions as a higher-isolation, lower-latency GPU access path for enterprise WSL2 deployments. This would allow per-VM GPU quotas and stronger scheduling guarantees than the current shared-scheduler model. Note: needs verification from public Microsoft roadmap sources.
- **Improved multi-GPU and device selection**: Current WSL2 multi-GPU support does not expose per-GPU container assignment (`--gpus device=N` semantics). Medium-term work aims to expose individual `DxCore` adapter handles as selectable compute devices in ROCm, CUDA, and DirectML runtimes, matching the per-device control available natively. Note: needs verification.
- **ROCm and oneAPI parity with native Linux**: AMD ROCm 7.2 introduced WSL2 support via `libdxcore.so`; further ROCm releases are expected to close the gap on features requiring `/dev/kfd` (e.g., HSA topology queries, P2P NVLink-equivalent transfers). Intel oneAPI DPC++ WSL2 support is similarly maturing. [Source](https://windowsnews.ai/article/microsoft-revives-dxgkrnl-for-linux-gpu-virtualization-in-wsl2-after-years-of-quiet-development.406197)
- **VirtIO-GPU virgl removal and native Vulkan-only path**: Mesa 26.1 officially dropped VirGL support; the industry is converging on native Vulkan guest drivers (e.g., NVK, RADV, ANV) paired with virtio-gpu for non-WSL2 VM guests, which indirectly pushes WSL2 toward a cleaner Vulkan-native stack on both the Mesa and runtime sides. [Source](https://kernel-recipes.org/en/2025/schedule/modernizing-virtio-gpu/)

### Long-term

- **Unified memory model across VM boundary**: The fundamental performance ceiling in WSL2 GPU use is the VM memory boundary — GPU allocations in the Linux guest are shadowed on the Windows host, and UVM (Unified Virtual Memory) semantics are unavailable for CUDA. Long-term architectural work on Hyper-V enlightened IOMMU and coherent cross-partition memory could enable true zero-copy GPU memory sharing, unlocking CUDA UVM and enabling peer-to-peer transfers between WSL2 and Windows GPU processes.
- **DRM integration for dxgkrnl**: Kernel maintainers have consistently requested that dxgkrnl integrate with the DRM subsystem (exposing a `drm_device`, GEM buffer objects, DMA-fence interop). Long-term, such integration would allow standard Linux GPU tooling (`nvidia-smi`, `radeontop`, `intel_gpu_top`, Vulkan WSI via GBM) to work inside WSL2, significantly closing the gap with native Linux. Note: this is a major architectural rework of the current design.
- **WSL2 as a first-class Linux desktop environment**: Microsoft's stated long-term direction for WSLg is to reduce perceptible latency and visual fidelity gaps between WSLg and a native Wayland desktop. Speculative architectural goals include Wayland protocol compositing directly over the Windows Desktop Window Manager (DWM) without the intermediate RDP hop, and support for DRM leasing for VR/XR headsets connected to Windows systems.
- **CUDA and ROCm containerisation maturity**: As Kubernetes and OCI container runtimes mature their WSL2 GPU backend support, long-term goals include full `nvidia-container-toolkit` feature parity (MIG partitioning, GPU time-slicing, NVML telemetry) and matching AMD ROCm container semantics under WSL2-backed Docker Desktop. Note: needs verification from vendor container roadmap announcements.

---

## Integrations

**Chapter 1 (DRM Architecture)**: DRM is notably absent in WSL2. `dxgkrnl` is a complete replacement: there is no `drm_driver`, no GEM, no KMS, no `drm_device`. Developers expecting `/dev/dri/` nodes will find them absent; all GPU access goes through `/dev/dxg`.

**Chapter 5 (x86 GPU Drivers — AMDGPU)**: The `amdgpu` kernel module does not run in WSL2. AMD GPU access uses `dxgkrnl` → VMBus → `amdkmdag.sys` (Windows AMDGPU driver). ROCm 7.2+ is compiled to use this path via `libdxcore.so` instead of `/dev/kfd`.

**Chapter 14 (NIR — Mesa's Shader IR)**: Both the D3D12 Gallium driver and dzn share the `nir_to_dxil` compiler in `src/microsoft/compiler/`. NIR is the internal representation; DXIL is the output format accepted by D3D12. The `nir_to_dxil.c` code is the single biggest technical contribution in Mesa's WSL2 support.

**Chapter 17 (Software Renderers)**: `llvmpipe` and `softpipe` remain available as fallbacks when D3D12 initialisation fails. Setting `LIBGL_ALWAYS_SOFTWARE=1` forces software rendering. This is useful for debugging shader issues that may be D3D12-specific versus Mesa-generic.

**Chapter 24 (Vulkan — Application Development)**: The dzn `VkPhysicalDevice` exposes standard Vulkan caps but with D3D12-imposed gaps (no `VkDisplayKHR`, limited sparse binding). Application developers should check `vkGetPhysicalDeviceFeatures2` and avoid assuming `VkPhysicalDeviceVulkan13Features.maintenance4` or similar features are present on the dzn path.

**Chapter 25 (GPU Compute)**: CUDA, ROCm HIP, and Intel OneAPI DPC++ all function in WSL2, routed through `dxgkrnl`. Unified Memory (UVM) is the primary compute limitation; most inference and training workloads that use explicit allocations (`cudaMalloc`, `hipMalloc`) run without modification.

**Chapter 48 (ROCm and Machine Learning)**: AMD ROCm 7.2 officially supports WSL2. The HIP runtime communicates through `libdxcore.so` → `dxgkrnl` → `amdkmdag.sys`. The `rocm-smi` tool and HIP device enumeration work with `ROCR_VISIBLE_DEVICES` for device selection.

**Chapter 55 (GPU Containers and Cloud Compute)**: Docker Desktop with the WSL2 backend injects `/dev/dxg` and WSL2 library paths instead of `/dev/nvidiactl` into containers. The `nvidia-container-toolkit` v2.6.0+ is WSL2-aware. Individual GPU selection (`--gpus device=0`) is unavailable in multi-GPU WSL2 configurations.

**Chapter 89 (GPU Virtualisation)**: GPU-P is the software virtualisation mechanism WSL2 uses — weaker isolation than SR-IOV but compatible with any WDDMv2.9+ driver. This chapter covers the full spectrum from SR-IOV through VFIO, MxGPU, and GVT-g for comparison.

**Chapter 104 (DXVK and VKD3D-Proton)**: DXVK (D3D9/D3D11 → Vulkan) and VKD3D-Proton (D3D12 → Vulkan) both function on WSL2's Vulkan ICDs. This makes WSL2 a viable platform for testing Windows game compatibility via Wine + Proton without leaving Windows, using the native NVIDIA or AMD Vulkan ICD.

**Chapter 107 (Headless Rendering)**: Headless EGL in WSL2 cannot use `EGL_PLATFORM_GBM` (no GBM). Use `EGL_PLATFORM_SURFACELESS_MESA` (Vulkan) or `EGL_PLATFORM_DEVICE_EXT` with the D3D12 backend. The `DISPLAY` variable should be unset for truly headless operation; setting it to `:0` (WSLg XWayland) makes applications window-based.

**Chapter 108 (ROCm)**: ROCm for WSL2 is installed via `amdgpu-install --usecase=wsl,rocm` and requires AMD Software: Adrenalin Edition 26.1.1 on the Windows host. GPU-P makes ROCm workloads first-class citizens on AMD hardware in WSL2, with the important caveat that `/dev/kfd` is absent and the compute path goes exclusively through `/dev/dxg`.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
