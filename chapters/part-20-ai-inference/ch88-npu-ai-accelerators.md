# Chapter 88: NPU and AI Accelerator Integration on Linux

> **Part**: Part XX — AI Inference on Linux
> **Audience**: Systems and driver developers working on AI PC platforms, embedded Linux engineers, GPU/NPU driver developers
> **Status**: First draft — 2026-06-18

---

## Table of Contents

1. [Introduction: The AI PC Inflection Point](#1-introduction-the-ai-pc-inflection-point)
2. [Intel NPU: The ivpu Kernel Driver](#2-intel-npu-the-ivpu-kernel-driver)
3. [Intel NPU and OpenVINO](#3-intel-npu-and-openvino)
4. [AMD XDNA: Ryzen AI NPU](#4-amd-xdna-ryzen-ai-npu)
5. [Qualcomm Hexagon DSP/NPU on Linux](#5-qualcomm-hexagon-dspnpu-on-linux)
6. [The DRM accel Subsystem](#6-the-drm-accel-subsystem)
7. [Heterogeneous Compute: CPU + GPU + NPU Dispatch](#7-heterogeneous-compute-cpu--gpu--npu-dispatch)
8. [Framework Integration: PyTorch, Transformers, and llama.cpp](#8-framework-integration-pytorch-transformers-and-llamacpp)
9. [Power and Thermal Management](#9-power-and-thermal-management)
10. [Developer Workflow: Tools and Debugging](#10-developer-workflow-tools-and-debugging)
11. [Integrations](#11-integrations)
12. [References](#12-references)

---

## 1. Introduction: The AI PC Inflection Point

This chapter targets systems engineers integrating **NPU** (Neural Processing Unit) hardware into Linux systems, driver developers extending the **DRM accel** subsystem, and embedded Linux engineers deploying edge inference. It assumes familiarity with the DRM driver model (Chapter 1), **DMA-BUF** buffer sharing (Chapter 4), Vulkan compute (Chapter 24), and **ROCm** fundamentals (Chapter 25).

The AI PC emerged as a distinct product category in 2023–2024 and by 2026 is the dominant form factor for x86 client hardware. Every major silicon vendor now ships a dedicated NPU tile alongside the CPU and iGPU:

- **Intel**: Meteor Lake (14th Gen, 2023) — 11 TOPS NPU; Lunar Lake (Core Ultra Series 2, 2024) — 48 TOPS **NPU 4**; Arrow Lake (Core Ultra Series 2 desktop, 2024) — 13 TOPS NPU (same IP as Meteor Lake); Panther Lake (2026) — next-generation **NPU 5**. [Source: Intel Lunar Lake NPU overview](https://www.intel.com/content/www/us/en/support/articles/000099574/processors/intel-core-ultra-processors.html)
- **AMD**: Ryzen AI 300 "Strix Point" (2024) — 55 TOPS **XDNA2** NPU; Krackan Point (2025) — **XDNA2** with 50 TOPS; Strix Halo (2025) — **XDNA2** with 50 TOPS.
- **Qualcomm**: Snapdragon X Elite / X Plus (2024) — **Hexagon** NPU at 45 TOPS, targeting Windows ARM laptops and, increasingly, Linux developer hardware.
- **Apple**: M-series (M1 through M4) — Apple Neural Engine (16 TOPS to 38 TOPS) on macOS/iOS; Linux support via Asahi Linux is limited to CPU/GPU.

### Why NPUs Exist Alongside GPUs

GPUs are throughput-optimised parallel processors designed around **SIMT** execution of wide vector workloads. They excel at training and batch inference where you have hundreds of inputs to process simultaneously. Their power envelope is proportional to throughput — a discrete GPU doing sustained inference at 100+ FP16 TFLOPS draws 200–300 W.

NPUs are purpose-built for *sustained, low-batch inference*. Their architectural characteristics differ in three key ways:

1. **Matrix-multiply acceleration at low power**: NPU tiles contain dense systolic array or **VLIW**+**SIMD** fabrics optimised for **INT8**/**INT4** matrix-multiply accumulate (**MMA**) operations — the dominant primitive in neural network inference. They achieve comparable throughput to a mid-tier GPU for quantised 1B–7B parameter models while consuming 3–15 W.

2. **Dedicated on-chip SRAM**: NPUs include large quantities of software-managed scratchpad **SRAM** (e.g., AMD **XDNA2**'s 4096 KB L2 pool) that eliminates DRAM bandwidth pressure for activation tensors, a critical bottleneck for transformer attention heads at small batch sizes.

3. **Always-on capability**: The NPU power domain can remain active and responsive while the CPU and GPU are in deep sleep states, enabling background AI tasks (voice recognition, camera processing, on-device LLM chat) without waking the main compute fabric.

### Linux NPU Support in 2026

The Linux kernel's **DRM accel** subsystem (**`drivers/accel/`**, introduced in Linux 6.2) provides the kernel-side foundation. As of Linux 6.14–6.15:

| Vendor | Hardware | Kernel Driver | Mainline Status |
|--------|----------|---------------|-----------------|
| Intel | Meteor Lake / Lunar Lake / Arrow Lake NPU | **`ivpu`** (**`drivers/accel/ivpu/`**) | Mainline since Linux 6.3 |
| AMD | Ryzen AI (Phoenix, Strix, Krackan) | **`amdxdna`** (**`drivers/accel/amdxdna/`**) | Mainline since Linux 6.14 |
| Qualcomm | Snapdragon X Elite Hexagon NPU | Out-of-tree / WIP | Not yet mainline |
| Habana Labs | Gaudi 2 | **`habanalabs`** + **`gaudi2`** | Mainline (training accelerator) |

Software stacks layer on top: **OpenVINO** for Intel, the **XRT**/**IRON**/Ryzen AI SDK for AMD, and **QNN SDK** for Qualcomm.

The chapter covers the following topics in depth. Section 2 examines the Intel **`ivpu`** kernel driver in detail: its **VPU**/**NPU** hardware generations (**Meteor Lake**, **Lunar Lake**, **Arrow Lake**), driver file structure, buffer object management via **`struct ivpu_bo`** and **GEM**, the full **ioctl** interface including **`DRM_IOCTL_IVPU_BO_CREATE`** and **`DRM_IOCTL_IVPU_SUBMIT`**, firmware loading via **`request_firmware()`** and the **JSM** (**Job Submission Module**) **IPC** channel, and the job submission model using command buffers and **MSI-X** completion. Section 3 covers Intel's **OpenVINO** software stack: the **`ov::Core`** API for device enumeration and model compilation via **`compile_model()`**, **`ov::intel_npu`** namespace properties for **NPU**-specific configuration including **MLIR** compiler type and **UMD** model caching, the **HETERO** plugin for heterogeneous graph splitting across **NPU**, **GPU**, and **CPU**, and tensor layout constraints (**NHWC** format) imposed by the NPU hardware. Section 4 covers AMD's **XDNA** (**AIE**) architecture — its 2D spatial tile array with **VLIW** compute tiles, **AXI4-Stream** switch fabric, and software-managed **L2 SRAM** — plus the **`amdxdna`** kernel driver (mainlined in Linux 6.14), buffer object types (**`AMDXDNA_BO_SHMEM`**, **`AMDXDNA_BO_DEV_HEAP`**, **`AMDXDNA_BO_CMD`**), the **xclbin**/**ctrlcode**/**ERT** workload execution model, and the **ONNX Runtime** **`VitisAIExecutionProvider`** integration. Section 5 addresses Qualcomm's **Hexagon** DSP/**NPU** hierarchy (**HTA**, **HTP**, **HVX**, **Adreno** GPU), the **`fastrpc`** kernel driver and **FastRPC** host-to-DSP **IPC** mechanism, the **QNN SDK** (**QAIRT**) with its **`libQnnHtp.so`** plugin and **`QNNExecutionProvider`** for **ONNX Runtime**, and the current (June 2026) Linux mainline support status including the closed-source **CDSP** firmware loader situation. Section 6 explains the **DRM accel** subsystem itself: its motivation relative to the older **`miscdevice`** approach, the **`/dev/accel/`** device node hierarchy with major number 261, driver registration via the **`DRIVER_COMPUTE_ACCEL`** feature flag and **`DEFINE_DRM_ACCEL_FOPS`** macro, current users (**`ivpu`**, **`amdxdna`**, **`habanalabs`**), and future candidates including **Arm Ethos-N** and the **MediaTek APU**. Section 7 covers heterogeneous **CPU**+**GPU**+**NPU** dispatch: the scheduling problem for **LLM** prefill vs. decode phases, **OpenVINO HETERO** graph partitioning, the **ONNX Runtime** execution provider priority and fallback chain, zero-copy **DMA-BUF** cross-device tensor sharing between GPU render nodes and NPU accel nodes via **`DRM_IOCTL_PRIME_HANDLE_TO_FD`**, and representative latency and power numbers for **Xe2** + NPU platforms running **Llama 3.2** INT4. Section 8 covers ML framework integration: **`ipex-llm`** (Intel Extension for **PyTorch** LLM) and the successor **OpenVINO GenAI** pipeline, **HuggingFace** **`optimum-amd`** and **`RyzenAIModel`** for Ryzen AI, the experimental **llama.cpp** **GGML** **XDNA** backend using the **`ggml_backend_reg`** API and **IRON** operator library, and **`torch.compile`** + **`torch.onnx.export()`** export paths for NPU targets. Section 9 covers power and thermal management: Intel NPU **ACPI** D-states and **`ivpu_pm.c`** runtime PM via **`dev_pm_ops`** and **`autosuspend_delay`**, NPU frequency visibility through **sysfs**, AMD **`xrt-smi`** power modes (**powersaver**, **balanced**, **performance**, **turbo**) and **SMU** thermal coordination, NPU vs. GPU power efficiency comparisons for 7B-parameter models, and thermal throttling behaviour across vendors. Section 10 presents the developer workflow and debugging tools: Intel's **`benchmark_app`** and **VTune** NPU profiling, AMD **`xrt-smi`** hardware context inspection and **GEMM** validation, **`ivpu`** driver tracing via **`trace_ivpu_*`** tracepoints defined in **`ivpu_trace.h`** and the **ftrace** subsystem, **sysfs** and **debugfs** inspection under **`/sys/kernel/debug/accel/`**, and **udev** rules for **`/dev/accel/accel0`** device node permissions.

---

## 2. Intel NPU: The ivpu Kernel Driver

### 2.1 Hardware Generations and the VPU/NPU Nomenclature

Intel originally called the accelerator tile a VPU (Vision Processing Unit, emphasising computer vision workloads). Starting with Lunar Lake's NPU 4, Intel shifted marketing to "NPU". The kernel driver retains the `ivpu` (Intel VPU) naming throughout. In the kernel source, the driver lives at:

```
drivers/accel/ivpu/
```

Hardware variants expose different `pci_device_id` values registered in `ivpu_drv.c`:

- `37xx` — Meteor Lake (MTL), 11 TOPS
- `40xx` — Lunar Lake (LNL), 48 TOPS; Arrow Lake (ARL), 13 TOPS

[Source: Linux kernel ivpu driver, torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/accel/ivpu/ivpu_drv.c)

The driver registers as a PCI driver:

```c
/* ivpu_drv.c (simplified) */
static const struct pci_device_id ivpu_pci_ids[] = {
    { PCI_DEVICE(PCI_VENDOR_ID_INTEL, PCI_DEVICE_ID_MTL) },
    { PCI_DEVICE(PCI_VENDOR_ID_INTEL, PCI_DEVICE_ID_LNL) },
    { 0 }
};
MODULE_DEVICE_TABLE(pci, ivpu_pci_ids);

static struct pci_driver ivpu_pci_driver = {
    .name     = DRIVER_NAME,
    .id_table = ivpu_pci_ids,
    .probe    = ivpu_probe,
    .remove   = ivpu_remove,
    .driver.pm = &ivpu_pm_ops,
};
module_pci_driver(ivpu_pci_driver);
```

The driver exposes the accelerator via the DRM accel subsystem as `/dev/accel/accel0` (see Section 6).

### 2.2 Driver File Structure

The `drivers/accel/ivpu/` directory contains:

| File | Purpose |
|------|---------|
| `ivpu_drv.c/h` | Main device initialisation, ioctl dispatch table, probe/remove |
| `ivpu_hw.c/h` | Hardware abstraction layer |
| `ivpu_hw_ip.c/h` | IP-layer register programming |
| `ivpu_hw_btrs_mtl_reg.h` | Meteor Lake register definitions |
| `ivpu_hw_btrs_lnl_reg.h` | Lunar Lake register definitions |
| `ivpu_gem.c/h` | Buffer object (BO) management via GEM |
| `ivpu_gem_userptr.c` | Userptr memory import |
| `ivpu_job.c/h` | Job submission and completion |
| `ivpu_mmu.c/h` | IOMMU/MMU context management |
| `ivpu_fw.c/h` | Firmware loading via `request_firmware()` |
| `ivpu_ipc.c/h` | IPC channel to NPU firmware |
| `ivpu_jsm_msg.c/h` | Job Submission Module message protocol |
| `ivpu_pm.c/h` | Runtime power management |
| `ivpu_ms.c/h` | Multi-stream context management |
| `ivpu_debugfs.c/h` | `/sys/kernel/debug/accel/` entries |
| `ivpu_trace.h` | `trace_ivpu_*` tracepoints |

### 2.3 Buffer Objects: ivpu_bo

Buffer objects in `ivpu_gem.c` extend the DRM GEM infrastructure. A `struct ivpu_bo` wraps a `struct drm_gem_object` and adds NPU-specific attributes:

```c
/* Simplified from ivpu_gem.h */
struct ivpu_bo {
    struct drm_gem_object base;
    struct mutex lock;
    struct ivpu_mmu_context *ctx;
    struct list_head bo_list_node;
    u32 flags;        /* IVPU_BO_* flags */
    u64 vpu_addr;    /* NPU-side virtual address */
    struct sg_table *sgt;
    void *kvaddr;    /* kernel virtual address (if mapped) */
};
```

The `DRM_IOCTL_IVPU_BO_CREATE` ioctl allocates a new buffer, maps it into the NPU MMU context, and returns a GEM handle:

```c
/* User-space usage via libze_intel_vpu (Level Zero NPU backend) */
struct drm_ivpu_bo_create bo_create = {
    .size  = tensor_size,
    .flags = DRM_IVPU_BO_SHAVE_MEM,  /* accessible to SHAVE cores */
};
ioctl(accel_fd, DRM_IOCTL_IVPU_BO_CREATE, &bo_create);
/* bo_create.handle now contains the GEM handle */
```

`DRM_IVPU_BO_SHAVE_MEM` marks the buffer for access by the NPU compute cores (SHAVE = Streaming Hybrid Architecture Vector Engine). Additional flag `DRM_IVPU_BO_HIGH_MEM` places the allocation in the upper 4 GB of the NPU address space, required for large tensor buffers on Lunar Lake.

### 2.4 The ioctl Interface

The full ioctl table in `ivpu_drv.c`:

```c
static const struct drm_ioctl_desc ivpu_drm_ioctls[] = {
    DRM_IOCTL_DEF_DRV(IVPU_GET_PARAM,              ivpu_get_param_ioctl,        0),
    DRM_IOCTL_DEF_DRV(IVPU_SET_PARAM,              ivpu_set_param_ioctl,        0),
    DRM_IOCTL_DEF_DRV(IVPU_BO_CREATE,              ivpu_bo_create_ioctl,        0),
    DRM_IOCTL_DEF_DRV(IVPU_BO_INFO,                ivpu_bo_info_ioctl,          0),
    DRM_IOCTL_DEF_DRV(IVPU_SUBMIT,                 ivpu_submit_ioctl,           0),
    DRM_IOCTL_DEF_DRV(IVPU_BO_WAIT,                ivpu_bo_wait_ioctl,          0),
    DRM_IOCTL_DEF_DRV(IVPU_METRIC_STREAMER_START,  ivpu_ms_start_ioctl,         0),
    DRM_IOCTL_DEF_DRV(IVPU_METRIC_STREAMER_STOP,   ivpu_ms_stop_ioctl,          0),
    DRM_IOCTL_DEF_DRV(IVPU_METRIC_STREAMER_GET_DATA, ivpu_ms_get_data_ioctl,    0),
    DRM_IOCTL_DEF_DRV(IVPU_METRIC_STREAMER_RESET,  ivpu_ms_reset_ioctl,         0),
    DRM_IOCTL_DEF_DRV(IVPU_CMDQ_CREATE,            ivpu_cmdq_create_ioctl,      0),
    DRM_IOCTL_DEF_DRV(IVPU_CMDQ_DESTROY,           ivpu_cmdq_destroy_ioctl,     0),
    DRM_IOCTL_DEF_DRV(IVPU_CMDQ_SUBMIT,            ivpu_cmdq_submit_ioctl,      0),
    DRM_IOCTL_DEF_DRV(IVPU_BO_CREATE_FROM_USERPTR, ivpu_bo_create_from_userptr_ioctl, 0),
};
```

### 2.5 Firmware Loading and IPC

The NPU runs a closed-source firmware blob provided by Intel. The driver loads it via the standard Linux firmware subsystem:

```c
/* ivpu_fw.c (simplified) */
static int ivpu_fw_request(struct ivpu_device *vdev)
{
    const char *fw_name = "intel/vpu/ivpu_fw.bin"; /* Meteor Lake */
    return request_firmware(&vdev->fw->file, fw_name, vdev->drm.dev);
}
```

The firmware binary is distributed in the `linux-firmware` package as `intel/vpu/ivpu_fw_<generation>.bin`. After loading, the driver copies the firmware into a reserved DMA region and signals the hardware via the JSM (Job Submission Module) IPC channel in `ivpu_ipc.c`.

### 2.6 Job Submission Model

Work is submitted as *command buffers* — sequences of NPU-native instructions that reference buffer object handles. The `DRM_IOCTL_IVPU_SUBMIT` ioctl takes a list of BO handles (inputs, outputs, and the command buffer itself):

```c
struct drm_ivpu_submit submit = {
    .buffers_ptr  = (u64)(uintptr_t)bo_handles,
    .buffer_count = num_bos,
    .commands_offset = 0,   /* byte offset into cmd BO */
    .engine       = DRM_IVPU_ENGINE_COMPUTE,
};
ioctl(accel_fd, DRM_IOCTL_IVPU_SUBMIT, &submit);
```

The driver validates the BO list, pins the pages, programs the MMU mappings, and writes the job descriptor to the firmware job queue via IPC. Completion is signalled by the firmware via MSI-X; the driver wakes waiters on `ivpu_bo_wait_ioctl`.

[Source: LWN article on the ivpu DRM accel driver](https://lwn.net/Articles/920233/)

---

## 3. Intel NPU and OpenVINO

### 3.1 OpenVINO Architecture Overview

OpenVINO is Intel's primary software stack for NPU, GPU, and CPU inference. Its plugin architecture abstracts device differences behind a unified `ov::Core` API. Three plugins are relevant:

- `intel_npu` — communicates with the NPU via the Level Zero NPU API, which in turn issues ioctls to the `ivpu` kernel driver
- `intel_gpu` — OpenCL/Level Zero path to Intel Xe iGPU and Arc dGPU (Chapter 71)
- `intel_cpu` — optimised CPU inference using AVX-512 / AMX intrinsics

[Source: OpenVINO NPU plugin README](https://github.com/openvinotoolkit/openvino/blob/master/src/plugins/intel_npu/README.md)

### 3.2 Device Enumeration and Model Compilation

```cpp
#include <openvino/openvino.hpp>

ov::Core core;

// List available devices
auto devices = core.get_available_devices();
// Typical output: ["CPU", "GPU", "NPU"]

// Load and compile a model for NPU
auto model = core.read_model("model.xml");
auto compiled = core.compile_model(model, "NPU");

// Create an inference request and run
auto infer_req = compiled.create_infer_request();
infer_req.set_input_tensor(input_tensor);
infer_req.infer();
auto output = infer_req.get_output_tensor();
```

Starting with OpenVINO 2026.1, the NPU plugin uses *Compiler-In-Plugin* by default: the OpenVINO model compiler (`intel_npu_compiler`) is bundled in the OpenVINO package rather than depending on the OEM driver's compiler. This decouples software stack updates from driver releases. [Source: Intel OpenVINO 2026 release](https://www.phoronix.com/news/Intel-OpenVINO-2026.0-Released)

### 3.3 NPU-Specific Configuration

The `ov::intel_npu` namespace exposes NPU-specific properties:

```cpp
// Set performance mode
compiled = core.compile_model(model, "NPU",
    ov::hint::performance_mode(ov::hint::PerformanceMode::THROUGHPUT),
    ov::intel_npu::compilation_mode_params("enable-dynamic-shapes=0"),
    ov::intel_npu::compiler_type(ov::intel_npu::CompilerType::MLIR));

// Query NPU capabilities
auto tops = core.get_property("NPU", ov::device::full_name.name());
auto npu_compiler = core.get_property("NPU", ov::intel_npu::compiler_type.name());
```

UMD (User-Mode Driver) model caching is enabled by default: after the first `compile_model()` call (which may take several seconds), the compiled NPU blob is stored in a hash-keyed cache, reducing subsequent time-to-first-inference from ~5s to ~100ms.

### 3.4 Heterogeneous Execution

For models containing operators not supported by the NPU (e.g., certain LSTM variants), OpenVINO's HETERO plugin splits the graph and dispatches subgraphs to different devices:

```cpp
// Prioritise NPU; fall back to GPU, then CPU
auto compiled = core.compile_model(model, "HETERO",
    ov::device::priorities("NPU,GPU,CPU"));
```

OpenVINO queries each device's *supported operations mask* (`ov::device::supported_properties`) and runs a graph-partitioning pass that assigns each node to the highest-priority device that supports its op type. Intermediate tensors between device subgraphs are allocated and transferred automatically. [Source: OpenVINO Heterogeneous Execution docs](https://docs.openvino.ai/2026/openvino-workflow/running-inference/inference-devices-and-modes/hetero-execution.html)

### 3.5 Tensor Layout Constraints

The Intel NPU imposes layout requirements on input tensors: activations must be in NHWC (batch, height, width, channels) order for CNN workloads; transformer models must present sequences in packed format. The NPU plugin inserts `Transpose` and `Reshape` nodes as needed during compilation, but explicitly specifying the correct layout at the `ov::Model` level avoids unnecessary transposes at runtime.

---

## 4. AMD XDNA: Ryzen AI NPU

### 4.1 XDNA Architecture

AMD's XDNA (formerly AIE — AI Engine) architecture is a 2D spatial array of compute and memory tiles connected by a programmable AXI4-Stream switch fabric. Each *column* in the array comprises:

- **4 compute tiles**: Each contains a VLIW processor (7-wide issue) with a 512-bit SIMD vector unit capable of INT8/INT4/FP16/BF16 MACs, plus dedicated program memory and data memory
- **1 memory tile**: 512 KB of L2 scratchpad SRAM shared among the column's compute tiles, software-managed via DMA

[Source: AMD NPU Linux kernel documentation](https://docs.kernel.org/accel/amdxdna/amdnpu.html)

Hardware generations:

| Generation | SoC | Array Topology | L2 SRAM | TOPS (INT8) | Max Contexts |
|------------|-----|----------------|---------|-------------|-------------|
| XDNA (AIE2) | Phoenix (Ryzen 7040) | 4×5 | 2560 KB | ~10 | 6 |
| XDNA (AIE2) | Hawk Point | 4×5 | 2560 KB | ~16 | 6 |
| XDNA2 (AIE4) | Strix Point (Ryzen AI 300) | 4×8 | 4096 KB | ~55 | 16 |
| XDNA2 | Krackan Point | 4×6 | 3072 KB | ~50 | 16 |
| XDNA2 | Strix Halo | 4×8 | 4096 KB | ~50 | 16 |

The NPU can be dynamically partitioned into spatial *contexts*: each context owns a contiguous subset of columns, enabling concurrent execution of multiple independent workloads with PASID-based isolation through the microcontroller MMU.

### 4.2 The amdxdna Kernel Driver

AMD's amdxdna driver was merged into mainline Linux in **Linux 6.14** (early 2025), living at `drivers/accel/amdxdna/`. [Source: Phoronix Linux 6.14 features](https://www.phoronix.com/review/linux-614-features)

The out-of-tree (OOT) driver at [github.com/amd/xdna-driver](https://github.com/amd/xdna-driver) supports both the upstream path and a legacy driver (`amdxdna_legacy.ko`). System requirements: Linux ≥ 6.10, `CONFIG_AMD_IOMMU=y`, `CONFIG_DRM_ACCEL=y`.

### 4.3 Buffer Object Types

The XDNA driver defines buffer object types reflecting the NPU memory hierarchy:

```c
/* amdxdna UAPI buffer object flags (from uapi/drm/amdxdna_accel.h) */
#define AMDXDNA_BO_SHMEM         0  /* Host DMA-accessible shared memory */
#define AMDXDNA_BO_DEV_HEAP      1  /* NPU device heap (large allocations) */
#define AMDXDNA_BO_DEV           2  /* NPU-local device memory */
#define AMDXDNA_BO_CMD           3  /* Command buffer for job submission */
```

User space allocates buffers via `DRM_IOCTL_AMDXDNA_CREATE_BO`:

```c
struct amdxdna_drm_create_bo create_bo = {
    .type  = AMDXDNA_BO_SHMEM,
    .size  = input_tensor_bytes,
    .flags = 0,
};
ioctl(accel_fd, DRM_IOCTL_AMDXDNA_CREATE_BO, &create_bo);
/* Returns .handle for subsequent ioctls */
```

### 4.4 Workload Execution Model

A workload comprises two AMD XDNA-specific binaries:

1. **Array overlay (xclbin)**: Configures the spatial partition — programs the AIE tile interconnect and loads compute kernels into each tile's program memory
2. **ctrlcode**: An orchestration sequence executed by the on-chip ERT (Embedded Runtime) microcontroller — commands DMA transfers from host DDR to L2 SRAM, triggers compute tiles, and signals completion

The `DRM_IOCTL_AMDXDNA_EXEC_CMD` ioctl submits a job:

```c
struct amdxdna_drm_exec_cmd exec = {
    .hwctx  = ctx_handle,   /* hardware context owning a column partition */
    .type   = AMDXDNA_CMD_SUBMIT_EXEC_BUF,
    .cmd_bo_handles = (u64)(uintptr_t)cmd_bos,
    .arg_bo_handles = (u64)(uintptr_t)arg_bos,
    .cmd_bo_count   = 1,
    .arg_bo_count   = num_tensors,
};
ioctl(accel_fd, DRM_IOCTL_AMDXDNA_EXEC_CMD, &exec);
```

### 4.5 ONNX Runtime XDNA Execution Provider

The Ryzen AI execution provider for ONNX Runtime links against AMD's XRT (Xilinx Runtime) and IRON (open-source NPU operator library) to compile ONNX model subgraphs to xclbin format at first run, caching the result on disk. Subsequent sessions load the cached xclbin, reducing first-inference latency from ~10s to ~200ms.

```python
import onnxruntime as ort

providers = [
    ('VitisAIExecutionProvider', {
        'config_file': '/opt/xilinx/xrt/share/vitis_ai_ep/vaip_config.json',
    }),
    'CPUExecutionProvider',   # fallback for unsupported ops
]
sess = ort.InferenceSession('model.onnx', providers=providers)
```

[Source: AMD Ryzen AI Software documentation](https://www.amd.com/en/developer/resources/ryzen-ai-software.html)

---

## 5. Qualcomm Hexagon DSP/NPU on Linux

### 5.1 Hexagon Architecture

Qualcomm's AI acceleration hierarchy in Snapdragon SoCs comprises three compute engines relevant to inference:

- **HTA (Hexagon Tensor Accelerator)**: Fixed-function accelerator for convolution and matrix multiply; the innermost "NPU" tile
- **HTP (Hexagon Tensor Processor)**: Programmable VLIW DSP with SIMD vector extensions (`HVX` — Hexagon Vector Extensions, 1024-bit SIMD); the main target for the QNN HTP backend
- **Adreno GPU**: Handles FP32/FP16 workloads not suited to the quantised HTP backend

In Snapdragon X Elite (Oryon CPU + Adreno 740 GPU + Hexagon NPU), the Hexagon NPU delivers ~45 TOPS INT8. [Source: Qualcomm Snapdragon X series](https://www.qualcomm.com/snapdragon/news/snapdragon-x-series--your-top-questions--answered)

### 5.2 FastRPC: Host-to-DSP IPC

The `fastrpc` kernel driver (`drivers/misc/fastrpc.c`, mainline since Linux 5.1) implements RPC from the application processor to the Hexagon DSP. It exposes `/dev/fastrpc-sdsp` (sensor DSP), `/dev/fastrpc-cdsp` (compute DSP), and `/dev/fastrpc-adsp` (audio DSP) as character devices. User space invokes DSP functions via:

```c
/* Open the compute DSP channel */
int fd = open("/dev/fastrpc-cdsp", O_RDWR);

/* Invoke a remote procedure — arguments serialised to shared memory */
struct fastrpc_ioctl_invoke invoke = {
    .handle   = dsp_function_handle,
    .sc       = FASTRPC_SCALARS(method_id, n_inputs, n_outputs),
    .pra      = remote_args,   /* array of struct remote_arg */
};
ioctl(fd, FASTRPC_IOCTL_INVOKE, &invoke);
```

Recent Qualcomm Linux releases added a shared-memory-optimised task queue path that reduces RPC overhead versus the legacy FastRPC interrupt path, with automatic fallback. [Source: Qualcomm QNN EP ONNX Runtime blog](https://www.qualcomm.com/developer/blog/2026/05/qualcomm-launches-the-first-onnx-runtime-plugin-execution-provider)

### 5.3 QNN SDK: The Software Stack

Qualcomm's *QNN (Qualcomm AI Engine Direct / QAIRT)* SDK supersedes the older SNPE (Snapdragon Neural Processing Engine). QNN provides:

- A model conversion and quantisation workflow (ONNX/TF/PyTorch → QNN DLC/bin)
- Runtime libraries for HTP, GPU, and CPU backends
- A plugin execution provider for ONNX Runtime (`onnxruntime-qnn`)

The QNN EP plugin architecture decouples QNN from ONNX Runtime core — it ships as `libQnnHtp.so` and is loaded at runtime:

```python
import onnxruntime as ort

# QNN HTP (Hexagon NPU) backend
ep_options = {
    'backend_type': 'HTP',
    'htp_performance_mode': 'burst',  # sustained high-performance
    'vtcm_mb': 8,                      # VTCM (vector TCM) allocation
    'htp_arch': '75',                  # Snapdragon X Elite Hexagon
}
sess = ort.InferenceSession('model_quantized.onnx',
    providers=[('QNNExecutionProvider', ep_options), 'CPUExecutionProvider'])
```

[Source: ONNX Runtime QNN EP documentation](https://onnxruntime.ai/docs/execution-providers/QNN-ExecutionProvider.html)

The QNN HTP backend requires INT8 quantised models; FP32 models must first be quantised via the QNN SDK's calibration tools or passed to the GPU backend.

### 5.4 Linux Support Status (June 2026)

Snapdragon X Elite Linux support in 2026 is actively developing but incomplete for NPU workloads. As of this writing:

- **Mainline Linux kernel**: FastRPC driver is mainline; the Hexagon NPU DSP firmware loader and CDSP driver for Snapdragon X are out-of-tree in Qualcomm's downstream BSP.
- **Open-source headers**: Qualcomm closed the GitHub issue requesting open-sourcing of Snapdragon X DSP headers in 2025, limiting third-party driver development. [Source: VideoCardz report on Qualcomm DSP headers](https://videocardz.com/newz/qualcomm-shuts-door-on-snapdragon-x-dsp-headers-open-sourcing-linux-support-hopes-fade)
- **Practical path**: QNN SDK targets Windows ARM natively; Linux x86 cross-compilation and Android are supported. Native Snapdragon X Linux NPU inference requires Qualcomm's proprietary Linux release or Windows + WSL2.
- **ai-hub-models**: Qualcomm's [ai-hub-models](https://github.com/quic/ai-hub-models) repository provides quantised model zoo and QNN EP integration examples that run on supported platforms.

**Note**: The upstream Linux Hexagon NPU driver situation for Snapdragon X Elite is fluid. Readers should consult the Qualcomm Linux BSP release notes and the `linux-arm-kernel` mailing list for current status.

---

## 6. The DRM accel Subsystem

### 6.1 Motivation and History

Before Linux 6.2, compute accelerator drivers (e.g., Habana Labs' original `habanalabs` driver) used the miscdevice framework — creating `/dev/hl0` etc. via `misc_register()` — with entirely custom ioctl namespaces. This meant no sharing of DRM infrastructure (GEM, fence synchronisation, debugfs patterns) and no common device model.

The DRM accel subsystem, merged in Linux 6.2 (November 2022), provides a middle ground: accelerator devices use DRM's infrastructure (GEM object management, `drm_sched` GPU scheduler, DMA-BUF export/import, debugfs) without exposing the display (KMS/CRTC) or render node machinery. [Source: LWN on new accel subsystem](https://lwn.net/Articles/915509/)

### 6.2 Device Nodes and Major Number

Accel devices use **major number 261** (distinct from DRM's 226 for `/dev/dri/`):

```
/dev/accel/accel0    ← first accelerator
/dev/accel/accel1    ← second (e.g., on dual-NPU systems)
```

Sysfs: `/sys/class/accel/accel0/`
Debugfs: `/sys/kernel/debug/accel/0/`

This separation means:

1. Accelerator access does not require the same privilege path as GPU rendering (`/dev/dri/renderD128` typically requires `video` group membership)
2. udev rules can independently control NPU device permissions
3. Container runtimes can mount NPU devices without granting GPU render access

### 6.3 Driver Registration

To register as an accel device rather than a GPU, a driver sets `DRIVER_COMPUTE_ACCEL` in `drm_driver.driver_features` — this flag is **mutually exclusive** with `DRIVER_RENDER` and `DRIVER_MODESET`:

```c
/* From drivers/accel/ivpu/ivpu_drv.c */
static const struct drm_driver ivpu_driver = {
    .driver_features    = DRIVER_GEM | DRIVER_COMPUTE_ACCEL,
    .open               = ivpu_open,
    .postclose          = ivpu_postclose,
    .ioctls             = ivpu_drm_ioctls,
    .num_ioctls         = ARRAY_SIZE(ivpu_drm_ioctls),
    .fops               = &ivpu_fops,   /* DEFINE_DRM_ACCEL_FOPS macro */
    .name               = DRIVER_NAME,
    .desc               = DRIVER_DESC,
    .major              = DRIVER_MAJOR,
    .minor              = DRIVER_MINOR,
};
```

The `DEFINE_DRM_ACCEL_FOPS` macro provides `accel_open` as the file operation open callback, which runs the accel-specific device open path rather than the DRM GPU path.

### 6.4 Current Users

As of Linux 6.15, the `drivers/accel/` directory contains:

| Subdirectory | Driver | Device |
|-------------|--------|--------|
| `ivpu/` | `intel_vpu` | Intel MTL/LNL/ARL NPU |
| `amdxdna/` | `amdxdna` | AMD Ryzen AI NPU (XDNA/XDNA2) |
| `habanalabs/` | `habanalabs` + `gaudi2` | Habana Labs Gaudi/Gaudi2 training accelerators |

The Habana Labs driver is a historical case study: it originally used the miscdevice approach (`/dev/hl0`) and was refactored to the accel subsystem, replacing the char device with the DRM accel infrastructure while maintaining ioctl ABI compatibility via a compatibility shim.

### 6.5 Future Directions

Accelerator types being discussed for the accel subsystem include:

- **MediaTek AI Processing Unit (APU)**: MTK's Dimensity mobile NPU; an out-of-tree driver exists
- **Arm Ethos-N**: Embedded neural network accelerator for Cortex-A SoCs; a staging driver (`drivers/staging/ethosu/`) exists as of Linux 6.8

[Source: Linux kernel accel subsystem documentation](https://docs.kernel.org/accel/introduction.html)

---

## 7. Heterogeneous Compute: CPU + GPU + NPU Dispatch

### 7.1 The Scheduling Problem

A deployed LLM inference pipeline involves multiple distinct operations:

- **Prefill (prompt processing)**: Compute-intensive parallel matrix multiplication — well-suited to GPU (high FLOPS, large batch)
- **Decode (token generation)**: Memory-bandwidth-bound, batch size 1 — suited to NPU (dedicated SRAM) or CPU (high memory bandwidth per watt)
- **Pre/post-processing**: Tokenisation, sampling — CPU-native

The goal is to partition the model graph across CPU, GPU, and NPU to minimise total latency and power. This is a combinatorial scheduling problem constrained by:

- Op-type support on each device (NPU supports quantised conv/matmul/attention; not all activation functions)
- Memory transfer overhead between devices (PCIe DMA for discrete GPU; shared LLC for integrated GPU/NPU)
- Power budget (NPU at ~10 W, iGPU at ~20 W, dGPU at 75–200 W)

### 7.2 OpenVINO HETERO Graph Splitting

OpenVINO's HETERO plugin implements graph partitioning as follows:

1. Query `ov::device::capabilities` on each device → set of supported op types
2. Traverse the `ov::Model` graph in topological order; assign each node to the first device in the priority list that supports it
3. Insert explicit `Assign`/`ReadValue` ops at device boundaries to materialise intermediate tensors
4. Each partition is compiled independently as an `ov::CompiledModel`

```cpp
// Priority: NPU first, GPU fallback, CPU last resort
auto compiled = core.compile_model(model, "HETERO",
    ov::device::priorities("NPU,GPU,CPU"),
    ov::device::properties("NPU",
        ov::hint::performance_mode(ov::hint::PerformanceMode::LATENCY)),
    ov::device::properties("GPU",
        ov::hint::performance_mode(ov::hint::PerformanceMode::THROUGHPUT)));
```

### 7.3 ONNX Runtime EP Priority and Fallback Chain

ONNX Runtime evaluates execution providers in registration order. Each EP claims the maximum subgraph it can support; remaining nodes fall through to the next EP:

```python
import onnxruntime as ort

# NPU via OpenVINO EP → GPU via OpenVINO EP → CPU fallback
sess_options = ort.SessionOptions()
providers = [
    ('OpenVINOExecutionProvider', {'device_type': 'NPU_FP16'}),
    ('OpenVINOExecutionProvider', {'device_type': 'GPU_FP32'}),
    'CPUExecutionProvider',
]
sess = ort.InferenceSession('model.onnx', sess_options, providers)
```

[Source: ONNX Runtime Execution Providers](https://onnxruntime.ai/docs/execution-providers/)

For NPU specifically: "if an op is not supported we fall back to CPU; if the model is not supported we fall back to CPU." This means NPU execution may be partial — some layers on NPU, others on CPU.

### 7.4 DMA-BUF Cross-Device Buffer Sharing

When the GPU produces feature tensors (e.g., image embeddings from an Adreno GPU) that are consumed by the NPU (e.g., an LLM decoder), zero-copy transfer via DMA-BUF is theoretically possible on integrated SoCs that share physical memory:

```c
/* GPU side: export a DMA-BUF from a render node buffer object */
struct drm_prime_handle prime = { .handle = gem_handle, .flags = DRM_CLOEXEC };
ioctl(gpu_fd, DRM_IOCTL_PRIME_HANDLE_TO_FD, &prime);
int dmabuf_fd = prime.fd;

/* NPU side: import the DMA-BUF as an ivpu_bo */
struct drm_ivpu_bo_create_from_dmabuf import_bo = {
    .fd    = dmabuf_fd,
    .flags = DRM_IVPU_BO_SHAVE_MEM,
};
ioctl(accel_fd, DRM_IOCTL_IVPU_BO_CREATE_FROM_USERPTR, &import_bo);
/* Note: DMA-BUF import path for ivpu is via userptr; direct prime import
   depends on driver version — verify against current kernel source */
```

**Note**: Direct DMA-BUF import between GPU render nodes and NPU accel nodes depends on both drivers implementing the DMA-BUF importer/exporter protocol correctly, and on the physical memory being accessible to both device MMUs. On Intel platforms with Xe iGPU + NPU sharing the same SoC interconnect, this zero-copy path is functional. On discrete GPU + NPU combinations, a PCIe DMA transfer occurs, and the performance benefit diminishes. Always measure before assuming zero-copy savings.

[Source: Linux kernel DMA-BUF documentation](https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html)

### 7.5 Representative Latency Numbers

The following figures are illustrative for a Lunar Lake (Core Ultra 200V, 48-TOPS NPU + Xe2 iGPU) platform running INT4-quantised Llama 3.2 3B (as of 2026 benchmarks):

| Configuration | Prefill (tokens/s) | Decode (tokens/s) | Power (W) |
|--------------|-------------------|------------------|---------:|
| CPU only (16 threads) | 180 | 12 | 28 |
| iGPU (Xe2, FP16) | 350 | 28 | 35 |
| NPU (INT4, OpenVINO) | 90 | 22 | 8 |
| NPU+iGPU (HETERO) | 320 | 24 | 22 |

The NPU wins decisively on power efficiency for decode; the iGPU wins on raw prefill throughput; HETERO execution approximates the iGPU throughput at lower power. For 7B+ parameter models, the NPU's on-chip SRAM is insufficient to hold full KV-cache, and the GPU becomes mandatory.

---

## 8. Framework Integration: PyTorch, Transformers, and llama.cpp

### 8.1 Intel Extension for PyTorch (IPEX-LLM)

`ipex-llm` (Intel Extension for PyTorch LLM) was Intel's primary library for accelerating LLM inference on Intel XPU targets including NPU, iGPU, and Arc dGPU. It integrated with llama.cpp via a portable zip distribution and supported HuggingFace Transformers via a patch. As of January 2026, the `intel/ipex-llm` repository was archived (read-only). Users are directed to the successor project at `github.com/ipex-llm/ipex-llm` or to the OpenVINO GenAI pipeline. [Source: IPEX-LLM GitHub](https://github.com/intel/ipex-llm)

For NPU-accelerated inference via OpenVINO GenAI:

```python
import openvino_genai as ov_genai

pipe = ov_genai.LLMPipeline("./Phi-3-mini-4k-instruct-int4-ov", "NPU")
result = pipe.generate("Explain NPUs in two sentences.", max_new_tokens=128)
```

OpenVINO GenAI handles KV-cache management, continuous batching, and the tokeniser; the NPU plugin executes the MatMul/Attention layers.

### 8.2 HuggingFace optimum-amd for Ryzen AI

HuggingFace's `optimum-amd` package provides a `RyzenAIModel` class that wraps ONNX Runtime's VitisAI EP:

```python
from optimum.amd.ryzenai import RyzenAIModelForImageClassification
from transformers import AutoFeatureExtractor

extractor = AutoFeatureExtractor.from_pretrained("microsoft/resnet-50")
model = RyzenAIModelForImageClassification.from_pretrained(
    "microsoft/resnet-50",
    vaip_config="./vaip_config.json",
    export=True,   # quantise and compile to xclbin on first run
)
```

[Source: HuggingFace optimum-amd](https://github.com/huggingface/optimum-amd)

### 8.3 llama.cpp GGML XDNA Backend

As of mid-2026, official XDNA backend support in llama.cpp is under development ([llama.cpp issue #21725](https://github.com/ggml-org/llama.cpp/issues/21725)). An experimental community fork (`OllamaAMDNPU`) implements matrix multiplication offload for Ryzen AI MAX (XDNA2) via XRT C++:

The GGML backend registration API defines the integration point:

```c
/* ggml/include/ggml-backend.h */
typedef struct ggml_backend_reg * ggml_backend_reg_t;

struct ggml_backend_reg {
    const char * name;
    ggml_backend_t (*init_backend)(void * params);
    ggml_backend_buffer_type_t (*get_default_buffer_type)(ggml_backend_reg_t reg);
    void * (*get_proc_address)(ggml_backend_reg_t reg, const char * name);
    void * context;
};

/* A hypothetical XDNA backend registers as: */
GGML_BACKEND_API ggml_backend_reg_t ggml_backend_xdna_reg(void);
```

The XDNA backend compiles GGML matrix-multiply operations to xclbin format via the IRON operator library, caches the results to disk, and dispatches via `DRM_IOCTL_AMDXDNA_EXEC_CMD`. Operations not yet mapped (e.g., RoPE, attention softmax) fall back to the CPU backend automatically.

### 8.4 PyTorch torch.compile and NPU Targets

PyTorch 2.x `torch.compile` uses the Inductor backend to generate optimised code. Direct NPU targeting via `torch.compile` is not yet first-class for Intel or AMD NPUs (2026). The recommended path remains:

1. Export to ONNX via `torch.onnx.export()`
2. Pass through ONNX Runtime with the appropriate NPU EP

or for Intel:

1. Export to OpenVINO IR via `ov.convert_model(torch_model)`
2. Compile with `ov.Core().compile_model(ir_model, "NPU")`

---

## 9. Power and Thermal Management

### 9.1 Intel NPU Power Management (ivpu_pm.c)

The Intel NPU power domain supports two mandatory ACPI D-states:

- **D0** (full power): NPU firmware running, ready to accept jobs via IPC
- **D3 Hot**: NPU clock-gated, firmware context saved; resumes in ~10ms
- **D3 Cold**: Full power removal; resume requires firmware reload (~200ms)

Runtime PM is managed in `ivpu_pm.c` via `dev_pm_ops`:

```c
/* ivpu_pm.c (simplified) */
static const struct dev_pm_ops ivpu_pm_ops = {
    .runtime_suspend = ivpu_pm_runtime_suspend,
    .runtime_resume  = ivpu_pm_runtime_resume,
    SET_SYSTEM_SLEEP_PM_OPS(ivpu_pm_suspend, ivpu_pm_resume)
};
```

The `autosuspend_delay` is set to 10 seconds by default: after the last job completes, the driver schedules a deferred D3 transition. This prevents power-thrashing when inference requests arrive in bursts. [Source: AMD Intel NPU power feature article](https://www.phoronix.com/news/AMD-Intel-NPU-Drivers-Power-7.2)

### 9.2 Intel NPU Frequency Limits via sysfs

The NPU driver exposes frequency control under the platform driver sysfs path:

```bash
# Read current NPU performance limits
cat /sys/bus/platform/drivers/intel_vpu/*/npu_busy_time_us
cat /sys/bus/platform/drivers/intel_vpu/*/fw_version

# Note: frequency control sysfs attributes vary by kernel version
# Check /sys/bus/platform/drivers/intel_vpu/ for available attributes
ls /sys/bus/platform/drivers/intel_vpu/*/
```

### 9.3 AMD XDNA Power Modes

The AMD XDNA driver exposes NPU power configuration via `xrt-smi configure`:

```bash
# Check current power mode
xrt-smi examine --report platform

# Set sustained performance mode (AC power required for turbo)
xrt-smi configure --pmode performance

# Available modes: default, powersaver, balanced, performance, turbo
```

Power consumption by mode:

| Mode | Typical NPU Power | Use Case |
|------|------------------|---------|
| `powersaver` | 0.5–2 W | Background always-on AI |
| `balanced` | 3–8 W | Interactive inference |
| `performance` | 8–15 W | Real-time video analysis |
| `turbo` | 15–25 W | Maximum throughput bursts |

[Source: AMD xrt-smi NPU management docs](https://ryzenai.docs.amd.com/en/latest/xrt_smi.html)

### 9.4 NPU vs GPU Power for 7B Model Inference

Based on current platform measurements (2026), the power-performance tradeoff for INT4-quantised 7B model inference:

| Device | Power Draw | Decode Speed | Power Efficiency |
|--------|-----------|-------------|----------------|
| Consumer NPU (XDNA2, 55T) | 8–12 W | 6–10 tok/s | ~1 tok/s/W |
| Intel Arc iGPU | 15–25 W | 15–25 tok/s | ~1 tok/s/W |
| Integrated GPU (AMD 890M) | 20–35 W | 18–30 tok/s | ~0.9 tok/s/W |
| RTX 4070 dGPU | 115 W | 80–120 tok/s | ~0.9 tok/s/W |
| RTX 5090 dGPU | 575 W | 350–450 tok/s | ~0.7 tok/s/W |

The NPU achieves comparable energy efficiency to a GPU for small (≤3B) quantised models but cannot match GPU throughput for 7B+ models due to SRAM capacity limits (the full KV-cache for a 7B model at 2K context exceeds 512 MB). The practical NPU sweet spot is 1B–3B parameter models with heavy INT4 quantisation for always-on, background applications. [Source: NPU vs GPU inference power analysis](https://www.solidaitech.com/2026/05/npu-vs-gpu-explained-ai-chips-difference.html)

### 9.5 Thermal Throttling Behaviour

All three major NPU vendors implement thermal throttling via the SoC's power management firmware:

- **Intel**: The NPU VPU tile shares a thermal zone with the CPU ring bus. Throttling reduces NPU clock frequency progressively at TJ thresholds. The `ivpu_pm.c` driver registers a notifier for system sleep transitions to flush in-flight jobs before D3.
- **AMD**: The NPU is managed by the SMU (System Management Unit) firmware, which coordinates with the CPU and iGPU thermal policies. The XDNA driver receives throttle notifications via MSI-X mailbox interrupts and suspends job submission.
- **Qualcomm**: Managed by the ThermalMonitor HAL; the Hexagon DSP receives thermal commands via FastRPC mailbox.

On sustained inference workloads, laptop NPUs typically settle at 60–70% of their peak rated TOPS after 10–15 minutes of continuous operation when operating at `performance` power mode without active cooling.

---

## 10. Developer Workflow: Tools and Debugging

### 10.1 OpenVINO benchmark_app

Intel's `benchmark_app` is the primary tool for measuring NPU latency and throughput:

```bash
# Install OpenVINO and convert a model
pip install openvino openvino-dev
mo --input_model yolov8n.onnx --output_dir ./yolov8n-ir

# Benchmark on NPU, 100 iterations, async mode (4 parallel requests)
benchmark_app -m ./yolov8n-ir/yolov8n.xml \
    -d NPU \
    -niter 100 \
    -nireq 4 \
    -api async

# Output:
# Throughput: 145.3 FPS
# Latency:    Mean: 27.4ms  Median: 26.8ms  95th%: 31.2ms
```

For VTune NPU profiling:

```bash
vtune -c npu -r ./vtune_results -- \
    benchmark_app -m yolov8n.xml -d NPU -niter 1000
```

[Source: OpenVINO benchmark tool docs](https://docs.openvino.ai/nightly/get-started/learn-openvino/openvino-samples/benchmark-tool.html)

### 10.2 AMD XDNA: xrt-smi

```bash
# Check NPU presence and readiness
xrt-smi examine

# View active hardware contexts and their resource usage
xrt-smi examine --report aie-partitions

# Run built-in GEMM TOPS benchmark (do not run with active workloads)
xrt-smi validate --run GEMM

# Configure power mode
xrt-smi configure --pmode performance

# JSON output for scripted monitoring
xrt-smi examine --report aie-partitions -f JSON -o /tmp/npu_status.json
```

[Source: AMD xrt-smi documentation](https://ryzenai.docs.amd.com/en/latest/xrt_smi.html)

### 10.3 Intel NPU Driver Tracing

The `ivpu` driver includes `trace_ivpu_*` tracepoints defined in `ivpu_trace.h`. Enable them via the ftrace subsystem:

```bash
# Enable all ivpu tracepoints
echo 1 > /sys/kernel/debug/tracing/events/ivpu/enable

# Capture a trace
cat /sys/kernel/debug/tracing/trace_pipe > /tmp/ivpu_trace.txt &
# ... run workload ...
kill %1

# Key tracepoints:
# trace_ivpu_job_submit    — job enters the driver queue
# trace_ivpu_job_done      — firmware signals completion
# trace_ivpu_ipc_send/recv — IPC message exchanges
# trace_ivpu_pm_suspend/resume — power state transitions
```

### 10.4 sysfs and debugfs Inspection

Intel NPU sysfs attributes:

```bash
# Find the NPU platform device
ls /sys/bus/platform/drivers/intel_vpu/

# Read device attributes (path varies by platform)
IVPU_DEV=$(ls /sys/bus/platform/drivers/intel_vpu/)
cat /sys/bus/platform/drivers/intel_vpu/${IVPU_DEV}/fw_version
cat /sys/bus/platform/drivers/intel_vpu/${IVPU_DEV}/npu_busy_time_us
```

DRM debugfs for accel devices:

```bash
# Intel NPU debugfs
ls /sys/kernel/debug/accel/0/
# Typical entries: bo_list, fw_log, ipc_stats, mmu_context, job_stats

# AMD XDNA debugfs
ls /sys/kernel/debug/accel/1/   # if NPU is accel1

# Read NPU memory allocations
cat /sys/kernel/debug/accel/0/bo_list
```

### 10.5 udev and Device Node Permissions

The NPU exposes as `/dev/accel/accel0` with major 261. Grant non-root access:

```
# /etc/udev/rules.d/99-npu.rules
SUBSYSTEM=="accel", KERNEL=="accel[0-9]*", GROUP="render", MODE="0660"
```

Add users to the `render` group (or a dedicated `npu` group) for NPU access without requiring GPU render node privileges.

---

## 11. Integrations

This chapter connects to the following chapters in this book:

- **Chapter 1 (DRM Architecture and the Driver Model)**: The DRM accel subsystem described in Section 6 is a direct extension of the DRM driver model — `drm_driver`, `drm_device`, GEM objects, and DMA-BUF are shared infrastructure. The `DRIVER_COMPUTE_ACCEL` feature flag and the `accel_open` file operation are the only accel-specific additions.

- **Chapter 4 (GPU Memory Management)**: Section 7.4 of this chapter directly depends on DMA-BUF cross-device sharing (Chapter 4 §DMA-BUF). The zero-copy GPU→NPU tensor transfer path requires both the GPU driver's DMA-BUF exporter and the NPU driver's DMA-BUF importer to agree on physical memory accessibility through the SoC interconnect.

- **Chapter 6 (ARM and Embedded GPU Drivers)**: On Snapdragon X Elite, the Adreno GPU (covered in Chapter 6) and the Hexagon NPU coexist in the SoC. The FastRPC IPC mechanism (Section 5.2) is the primary GPU→DSP communication path; tensor sharing between Adreno and Hexagon on Android uses `AHardwareBuffer` (Chapter 86) rather than Linux DMA-BUF on the current driver stack.

- **Chapter 24 (Vulkan and EGL for Application Developers)**: For GPU-heavy prefill phases, Vulkan compute (Chapter 24) remains the preferred API. The GPU→NPU handoff described in Section 7 involves passing tensors from Vulkan storage buffers (exported as DMA-BUF) to the NPU accel device.

- **Chapter 25 (GPU Compute — ROCm, CUDA, Level Zero)**: Chapter 25 covers Level Zero, which is Intel's low-level GPU compute API. The Intel NPU is also accessible via a Level Zero NPU backend (`libze_intel_vpu.so`); OpenVINO's NPU plugin uses this internally rather than calling `ivpu` ioctls directly.

- **Chapter 48 (ROCm and Machine Learning on Linux GPUs)**: Chapter 48 covers GPU-based training and inference. This chapter covers the complementary NPU path for *inference-only* workloads. The recommended hybrid strategy (Section 7) uses ROCm/HIP (Chapter 48) for training and NPU for deployed inference.

- **Chapter 55 (GPU Containers and Cloud Compute)**: Container runtime NPU access requires device node mounting. The `/dev/accel/accel0` device (major 261) must be explicitly added to container device lists. Kubernetes device plugins for Intel NPU (`intel-device-plugins-for-kubernetes`) and AMD Ryzen AI are under development with different maturity levels than the established GPU plugin ecosystem.

- **Chapter 71 (Intel Xe Kernel Driver, Arc GPU Architecture, and the Intel Open Stack)**: Chapter 71 covers Level Zero and the Xe kernel driver for Intel Arc GPU. The OpenVINO GPU plugin uses the same Level Zero path as Arc, while the NPU plugin uses a separate Level Zero NPU extension. Both share the OpenVINO `ov::Core` API described in Section 3.

- **Chapter 87 (LLM Inference on Linux — GPU+NPU Hybrid Dispatch)**: Chapter 87 covers the higher-level inference frameworks (vLLM, llama.cpp server, SGLang) that sit above the hardware layer described here. The NPU-GPU dispatch heuristics in Section 7 are what Chapter 87's frameworks implement when targeting AI PC hardware; this chapter provides the kernel and driver foundation that Chapter 87 builds on.

---

## 12. References

Key upstream sources used in this chapter:

- [Linux kernel ivpu driver source — torvalds/linux](https://github.com/torvalds/linux/tree/master/drivers/accel/ivpu)
- [LWN: New DRM accel driver for Intel VPU](https://lwn.net/Articles/920233/)
- [LWN: New DRM driver for Intel VPU](https://lwn.net/Articles/909077/)
- [LWN: New subsystem for compute accelerator devices](https://lwn.net/Articles/915509/)
- [Linux kernel accel subsystem documentation](https://docs.kernel.org/accel/introduction.html)
- [AMD NPU Linux kernel documentation](https://docs.kernel.org/accel/amdxdna/amdnpu.html)
- [AMD xdna-driver GitHub repository](https://github.com/amd/xdna-driver)
- [OpenVINO NPU device documentation (2026)](https://docs.openvino.ai/2026/openvino-workflow/running-inference/inference-devices-and-modes/npu-device.html)
- [OpenVINO HETERO execution documentation](https://docs.openvino.ai/2026/openvino-workflow/running-inference/inference-devices-and-modes/hetero-execution.html)
- [Intel OpenVINO 2026.0 release notes — Phoronix](https://www.phoronix.com/news/Intel-OpenVINO-2026.0-Released)
- [ONNX Runtime QNN Execution Provider](https://onnxruntime.ai/docs/execution-providers/QNN-ExecutionProvider.html)
- [Qualcomm QNN EP plugin launch blog (2026)](https://www.qualcomm.com/developer/blog/2026/05/qualcomm-launches-the-first-onnx-runtime-plugin-execution-provider)
- [Qualcomm Snapdragon X Elite Linux NPU status](https://mysupport.qualcomm.com/supportforums/s/question/0D5dK00000BcOw5SAF/where-can-i-find-the-linux-npu-driver-for-snapdragon-x-elite)
- [Intel Arc / NPU TOPS comparison — Tom's Hardware](https://www.tomshardware.com/pc-components/cpus/intels-arrow-lake-s-wont-be-an-ai-powerhouse-13-tops-npu-is-only-slightly-better-than-meteor-lake-much-less-than-lunar-lake)
- [AMD Ryzen AI software stack](https://www.amd.com/en/developer/resources/ryzen-ai-software.html)
- [AMD xrt-smi NPU management tool](https://ryzenai.docs.amd.com/en/latest/xrt_smi.html)
- [HuggingFace optimum-amd for Ryzen AI](https://github.com/huggingface/optimum-amd)
- [llama.cpp XDNA feature request — ggml-org](https://github.com/ggml-org/llama.cpp/issues/21725)
- [IPEX-LLM Intel NPU integration](https://github.com/intel/ipex-llm)
- [DMA-BUF kernel documentation](https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html)
- [OpenVINO benchmark_app documentation](https://docs.openvino.ai/nightly/get-started/learn-openvino/openvino-samples/benchmark-tool.html)
- [Qualcomm AI Hub Models](https://github.com/quic/ai-hub-models)
- [ONNX Runtime Execution Providers](https://onnxruntime.ai/docs/execution-providers/)
- [NPU vs GPU power efficiency analysis (2026)](https://www.solidaitech.com/2026/05/npu-vs-gpu-explained-ai-chips-difference.html)
- [Hybe: GPU-NPU Hybrid System for LLM Inference — ACM ISCA 2025](https://dl.acm.org/doi/10.1145/3695053.3731051)

## Roadmap

### Near-term (6–12 months)
- The `amdxdna` driver is expected to gain native DMA-BUF import/export support, enabling true zero-copy tensor sharing between the Radeon iGPU and the XDNA2 NPU via the shared SoC interconnect on Strix Point platforms.
- Intel's `ivpu` driver has pending patches (tracked in linux-accel mailing list discussions) to upstream the `DRM_IOCTL_IVPU_BO_CREATE_FROM_DMABUF` ioctl as a first-class PRIME import path, replacing the current workaround through the userptr code path.
- OpenVINO 2026.x releases are expected to complete the Compiler-In-Plugin transition for the NPU backend, fully removing the dependency on the OEM-supplied Level Zero NPU compiler shared library for Meteor Lake and Lunar Lake targets.
- Qualcomm is expected to submit the Snapdragon X Elite Hexagon CDSP firmware loader upstream to `drivers/accel/` following pressure from the linaro-kernel and linux-arm-kernel communities; early RFC patches were circulated in Q2 2026.

### Medium-term (1–3 years)
- The DRM accel subsystem will likely gain a standardised `drm_accel_sched` integration path built on `drm_gpu_scheduler`, allowing NPU drivers to share the same GPU scheduler infrastructure used by AMD and Intel GPU drivers and enabling cross-device fence synchronisation without bespoke IPC mechanisms.
- `torch.compile` is expected to gain a first-class NPU backend via the OpenVINO TorchDynamo frontend for Intel and via the MLIR-AIE compiler for AMD XDNA, eliminating the current two-step ONNX export → EP execution path for PyTorch 3.x workflows.
- As XDNA2 column-partition counts increase in future SoC generations (e.g., post-Strix Halo designs), the `amdxdna` driver will need to evolve a proper hardware context scheduling policy (analogous to GPU timeslicing) to fairly multiplex up to 32+ concurrent NPU contexts; early design work is visible in AMD kernel mailing list discussions.
- MediaTek APU and Arm Ethos-N drivers are likely to graduate from staging (`drivers/staging/`) to `drivers/accel/` following the pattern established by `amdxdna`, bringing embedded SoC NPUs under the same UAPI as PC-class NPUs.

### Long-term
- Convergence toward a unified NPU UAPI layer — analogous to how Vulkan unified GPU programming — is a stated goal of the DRM accel subsystem maintainers; a common set of buffer object types, fence primitives, and performance query ioctls shared across `ivpu`, `amdxdna`, and future drivers would allow userspace frameworks to target NPU hardware without vendor-specific code paths.
- As on-device LLM model sizes grow into the 30B–70B range and exceed what any single NPU's SRAM can accommodate, kernel-level NPU memory management will need to evolve toward demand-paged model weights analogous to GPU VRAM overcommit — early academic work (e.g., the Hybe paper) points toward collaborative GPU-NPU KV-cache management as the dominant architecture.
- The Qualcomm Hexagon NPU and future RISC-V-based custom accelerators (e.g., announced collaboration between SiFive and NPU IP vendors) may drive adoption of a vendor-neutral firmware ABI specification at the kernel level, reducing the proprietary firmware blob dependency that currently limits open-source reproducibility for Intel `ivpu` and AMD XDNA.
