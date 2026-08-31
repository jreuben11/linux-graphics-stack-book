# Chapter 124: Local LLM Inference on Linux GPUs

**Target audiences**: Systems developers and GPU driver engineers who need to understand how inference runtimes talk to the kernel and hardware; ML engineers deploying large language models locally on Linux without cloud dependencies.

---

## Table of Contents

1. [Introduction](#1-introduction)
   - [1.1 What is LLM Inference?](#11-what-is-llm-inference)
   - [1.2 What is GGUF and GGML?](#12-what-is-gguf-and-ggml)
   - [1.3 What is Quantization for LLM Inference?](#13-what-is-quantization-for-llm-inference)
2. [GGML Architecture and the Vulkan Backend](#2-ggml-architecture-and-the-vulkan-backend)
3. [llama.cpp Vulkan Path in Depth](#3-llamacpp-vulkan-path-in-depth)
4. [Memory-Mapped Weights and DMA-BUF](#4-memory-mapped-weights-and-dma-buf)
5. [Ollama: GPU Dispatch and Model Management](#5-ollama-gpu-dispatch-and-model-management)
6. [ONNX Runtime with GPU Execution Providers](#6-onnx-runtime-with-gpu-execution-providers)
7. [ONNX Runtime OpenVINO EP](#7-onnx-runtime-openvino-ep)
8. [ROCm MIOpen and HIP for LLM Inference](#8-rocm-miopen-and-hip-for-llm-inference)
9. [KV Cache Management Strategies](#9-kv-cache-management-strategies)
10. [Performance Tuning and Benchmarking](#10-performance-tuning-and-benchmarking)
11. [VRAM Capacity Planning: Will This Model Fit?](#11-vram-capacity-planning-will-this-model-fit)
12. [MacBook vs. Gaming Laptop for Local Inference](#12-macbook-vs-gaming-laptop-for-local-inference)
14. [Production Serving Engines: vLLM and SGLang](#14-production-serving-engines-vllm-and-sglang)
    - [14.1 vLLM: PagedAttention as a Serving Engine](#141-vllm-pagedattention-as-a-serving-engine)
    - [14.2 SGLang: RadixAttention and Structured Generation](#142-sglang-radixattention-and-structured-generation)
15. [Disaggregated Prefill-Decode Serving](#15-disaggregated-prefill-decode-serving)
16. [Speculative Decoding](#16-speculative-decoding)
17. [Quantization for Serving Engines: GPTQ, AWQ, bitsandbytes, and FP8](#17-quantization-for-serving-engines-gptq-awq-bitsandbytes-and-fp8)
18. [NVIDIA NGC Catalog and NIM Microservices](#18-nvidia-ngc-catalog-and-nim-microservices)
19. [Docker Model Runner](#19-docker-model-runner)
20. [Kubernetes-Native Agent Orchestration: kagent](#20-kubernetes-native-agent-orchestration-kagent)
21. [Running Hugging Face Models Locally via the CLI](#21-running-hugging-face-models-locally-via-the-cli)
22. [Fine-Tuning Acceleration with Unsloth](#22-fine-tuning-acceleration-with-unsloth)
23. [Compiled-Engine Serving: TensorRT-LLM and LMDeploy](#23-compiled-engine-serving-tensorrt-llm-and-lmdeploy)
24. [Model-Swapping Proxies: llama-swap and LiteLLM](#24-model-swapping-proxies-llama-swap-and-litellm)
25. [Structured Output, Grammars, and Function Calling](#25-structured-output-grammars-and-function-calling)
26. [Multi-LoRA Serving](#26-multi-lora-serving)
27. [ExLlamaV2 and ExLlamaV3](#27-exllamav2-and-exllamav3)
28. [llm-d: Kubernetes-Native Distributed Inference](#28-llm-d-kubernetes-native-distributed-inference)
29. [Integrations](#29-integrations)

---

## 1. Introduction

By mid-2026, local LLM inference on Linux has evolved from a hobbyist curiosity into a production-grade concern. A **Llama-3-70B** model quantised to **Q4\_K\_M** fits in 40 GB of VRAM on a single AMD **RX 7900 XTX** or a pair of **NVIDIA RTX 4090**s. **Mixtral-8×7B** runs at interactive speed on a single high-end consumer card. Smaller 7B and 8B models run at 80–150 tokens/second on almost any recent discrete GPU. The diversity of Linux GPU hardware has never been greater: NVIDIA's **CUDA** stack, AMD **ROCm** on **RDNA3**/4 and **CDNA**, Intel **Arc** (**Xe-HPG**) with **Level Zero** and **OpenVINO**, Qualcomm **Adreno** via **Vulkan** compute, and Apple Silicon accessed through **Asahi Mesa** or Metal.

Three advantages motivate running inference locally rather than via a cloud API:

- **Privacy**: prompt tokens never leave the machine. Medical, legal, and enterprise use cases often require this.
- **Latency**: round-trip to a remote inference endpoint adds 50–200 ms of network overhead before the first token; local VRAM latency is sub-millisecond.
- **Cost at scale**: marginal cost of a generated token on owned hardware approaches the electricity cost per watt, typically orders of magnitude cheaper than API pricing at high token volumes.

This chapter examines the full software path from a **GGUF** file on disk to generated tokens on screen.

- **Section 2 — GGML Architecture and Vulkan Backend**
  - the **GGML** tensor engine including its **ggml_cgraph** DAG executor
  - the **ggml_tensor** and **ggml_backend_i** abstraction layer
  - quantisation types ranging from **GGML_TYPE_F16** and **GGML_TYPE_BF16** through K-quant block formats (**Q4_K_M**, **Q6_K**) and I-quant formats (**IQ4_XS**)
  - the **Vulkan** backend in **ggml-vulkan.cpp** — including the **vk_device_struct** device abstraction, the **SPIR-V** compute shader pipeline compiled via **glslc**, **vk_matmul_pipeline_struct** variants, push-constant dispatch, and **VK_KHR_cooperative_matrix** support
- **Section 3 — llama.cpp Vulkan Path**
  - startup initialisation via **vkEnumeratePhysicalDevices** and **vkCreateDevice** (with the **GGML_VK_VISIBLE_DEVICES** override)
  - KV cache allocation with **grouped-query attention** (**GQA**) accounting
  - the **build_llama** compute graph including fused **RoPE** via **ggml_rope_custom** and **Flash Attention** via **ggml_flash_attn_ext** with **coopMatMulAdd** on cooperative-matrix hardware
  - multi-GPU layer splitting with **--n-gpu-layers**, **--split-mode row**, and **--tensor-split**
  - representative **llama-bench** throughput numbers across **CUDA**, **ROCm**, and **Vulkan** backends
- **Section 4 — Memory-Mapped Weights and DMA-BUF**
  - the **GGUF** binary container format (header, KV metadata, tensor info, and 32-byte-aligned tensor data sections)
  - memory-mapped loading via **mmap(2)** with **MAP_SHARED** and **madvise** prefetch hints
  - host-to-GPU weight transfer through **vkCmdCopyBuffer** and **VK_ACCESS_TRANSFER_WRITE_BIT** pipeline barriers
  - handling models larger than VRAM via split mode and **MADV_WILLNEED** prefetch
  - zero-copy loading on systems with **Resizable BAR** (**AMD SAM**) using the **VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT** memory type
- **Section 5 — Ollama**
  - its Go **ollama serve** HTTP server and **ollama_llama_server** runner subprocess
  - GPU detection via **NVML** (**libnvidia-ml.so**), **KFD** sysfs parsing for AMD, and **Level Zero** / **OpenCL** queries for Intel
  - environment overrides (**CUDA_VISIBLE_DEVICES**, **ROCR_VISIBLE_DEVICES**, **OLLAMA_GPU_OVERHEAD**)
  - the content-addressed model library under **~/.ollama/models/**
  - the **REST API** endpoints (**/api/generate**, **/api/chat**, **/api/embeddings**)
  - parallel request handling via **OLLAMA_NUM_PARALLEL** and llama.cpp's batched-decode path
- **Section 6 — ONNX Runtime with GPU Execution Providers**
  - **ONNX Runtime** (**ORT**) and its **Execution Provider** (**EP**) plugin architecture
  - the **CUDA EP** using **cuDNN** and **cuBLAS** configured via **OrtCUDAProviderOptionsV2** (including **CUDA Graph** capture to eliminate per-iteration kernel launch overhead, **TF32** on Ampere+, and **TransformerOptimizer** graph fusions such as **SkipLayerNorm** and **FusedMatMul**)
  - ORT's quantisation tooling for **INT8** via **quantize_dynamic** and **FP16** conversion
- **Section 7 — ONNX Runtime OpenVINO EP**
  - the **OpenVINO** Intermediate Representation (**IR**) format
  - **OrtOpenVINOProviderOptions** and its V2 key-value configuration
  - the **Level Zero** backend routing **Intel Arc** compute through the **Intel Graphics Compiler** (**IGC**)
  - heterogeneous execution via **HETERO:NPU,GPU,CPU** across Intel **NPU**, **iGPU**, and CPU
- **Section 8 — ROCm MIOpen and HIP**
  - the **HIP** programming model and runtime (**hipMalloc**, **hipMemcpy**, **hipLaunchKernelGGL**)
  - GEMM via **rocBLAS** (**rocblas_gemm_ex**) and **hipBLASLt** with **TunableOp** auto-selection
  - unified memory via **hipMallocManaged** and **HMM** on AMD APUs such as **Strix Halo** (**Ryzen AI Max+ 395**)
  - the **PyTorch** CUDA-compatibility shim that allows ROCm-unmodified inference
  - **vLLM** device isolation via **ROCR_VISIBLE_DEVICES** and **AITER** (**AI Tensor Engine for ROCm**) attention kernels (**VLLM_ROCM_USE_AITER**)
  - **MIOpen** auto-tuning via **MIOPEN_USER_DB_PATH** and **MIOPEN_FIND_MODE**
- **Section 9 — KV Cache Management**
  - the quadratic VRAM scaling problem
  - **PagedAttention** in **vLLM** with fixed-size physical blocks and the **BlockManager** free-block pool achieving under 4% fragmentation
  - Automatic Prefix Caching (**APC**) using Merkle-chain block hashes (**xxhash**, **sha256**) with LRU eviction
  - llama.cpp's ring-buffer KV cache and **Sliding Window Attention** (**SWA**) for **Llama-3.1** models plus coarse CPU swap via **ggml_backend_cpu_buffer_type**
  - vLLM's preemption strategies (swap via **cudaMemcpyAsync** / **hipMemcpy** and recompute) managed by the hybrid KV cache manager
- **Section 10 — Performance Tuning**
  - measuring prompt-processing (**pp**) and token-generation (**tg**) throughput with **llama-bench**
  - quantisation impact on model size, generation speed, and perplexity across **F16**, **Q8_0**, **Q4_K_M**, **Q3_K_M**, and **Q2_K** formats
  - GPU utilisation monitoring with **nvtop**, **nvidia-smi dmon**, **radeontop**, **rocm-smi**, and **intel_gpu_top**
  - power and thermal considerations including **TJmax** throttling and **nvidia-smi -pl** / **rocm-smi --setpoweroverdrive** power capping
  - roofline arithmetic-intensity analysis that explains why token generation at batch=1 is deeply memory-bandwidth-bound

### 1.1 What is LLM Inference?

Large language model (LLM) inference is the process of executing a pre-trained transformer-based neural network to generate text token-by-token from a given prompt. Unlike training — which adjusts the model's billions of weight parameters over large datasets using backpropagation and gradient descent — inference uses the trained weights in a forward-only pass, computing attention over a key-value (KV) cache that grows with sequence length. The core computational primitive is dense matrix multiplication: for every generated token, the runtime multiplies the current hidden-state vector (dimension D, typically 4096–8192 for 7B–70B models) by each layer's query, key, value, and MLP projection weight matrices. On a GPU, those matrix–vector products map to GEMM (General Matrix Multiply) operations scheduled on the GPU's compute units via a kernel driver. On Linux, the kernel exposes GPU compute through the DRM subsystem: NVIDIA uses its proprietary kernel module paired with the CUDA user-space stack; AMD uses the `amdgpu` DRM driver with the ROCm stack and the Kernel Fusion Driver device node at `/dev/kfd`; Intel Arc GPUs use the `i915` or `xe` DRM driver with Level Zero. The inference runtime — llama.cpp, vLLM, Ollama, or ONNX Runtime — sits between the model weights on disk and those kernel interfaces, translating the transformer graph into backend-specific GEMM calls and memory management commands. Memory bandwidth, not arithmetic throughput, is the dominant performance constraint at batch size 1: every generated token requires reading the full weight tensor from GPU memory, making VRAM bandwidth the ceiling on tokens-per-second.

### 1.2 What is GGUF and GGML?

GGML (Georgi Gerganov Machine Learning) is a C tensor library whose primary design goal is efficient quantised inference with optional GPU offload. It defines a tensor type (`ggml_tensor`), a directed acyclic graph executor (`ggml_cgraph`), and a pluggable backend abstraction (`ggml_backend_i`) that lets the same compute graph dispatch to CPU SIMD, CUDA, ROCm, Vulkan, or SYCL backends without changing the calling code. Each backend registers capability queries (`supports_op`), buffer allocation routines, and a `graph_compute` entry point; the scheduler assigns tensor operations to the highest-priority backend that reports support for that operation type. GGUF (GGML Unified Format) is the binary container format that stores quantised model weights on disk. A GGUF file opens with a magic number and version header, followed by a key-value metadata section describing model architecture and tokeniser configuration, a tensor index listing each tensor's name and byte offset into the data region, and then the raw tensor data aligned to 32-byte boundaries. The data section is designed for `mmap(2)` with `MAP_SHARED`: the OS page cache becomes the backing store, and the kernel loads weight pages on demand as GPU transfer commands reference them, avoiding an explicit read-into-RAM stage. This also enables operating on models larger than physical VRAM by splitting layers across GPU and CPU memory. GGUF replaced the earlier GGML binary format in 2023 and is now the standard interchange format for quantised open-weight models distributed through model repositories and llama.cpp-compatible tools.

### 1.3 What is Quantization for LLM Inference?

Quantization in the context of LLM inference means representing weight tensors using fewer bits than the 32-bit (FP32) or 16-bit (FP16/BF16) floating-point formats used during training. The motivation is memory bandwidth: a Llama-3-70B model stored in FP16 requires approximately 140 GB to be read from GPU memory for each generated token, far exceeding the VRAM of any consumer GPU. Reducing weights to 4 bits per parameter shrinks that figure to roughly 35 GB, fitting on a single high-end GPU and reducing per-token bandwidth demand proportionally. Quantization introduces error by mapping a range of floating-point values onto a small number of discrete integer codes, but for transformer weight matrices this error is well-tolerated because individual weights rarely dominate a computation. GGML implements three families of quantisation. The legacy integer block formats (`Q4_0`, `Q4_1`, `Q8_0`) quantise fixed 32-element blocks with a per-block floating-point scale stored alongside the integer codes. The K-quant block formats (`Q2_K`, `Q3_K_S/M/L`, `Q4_K_S/M`, `Q5_K_S/M`, `Q6_K`) use 256-element super-blocks with learned scale and minimum values, achieving better perplexity at equivalent bit-width because the larger context captures the weight distribution more accurately. The I-quant formats (`IQ4_XS`, `IQ2_XXS`) use an importance matrix derived from calibration data to allocate bits non-uniformly, concentrating precision where weights most affect output quality. GPU backends decode quantised blocks back to FP16 or BF16 in-shader — in SPIR-V compute kernels for the Vulkan path, or in CUDA/HIP kernels for the NVIDIA and AMD paths — immediately before feeding values into the GEMM pipeline.

---

## 2. GGML Architecture and the Vulkan Backend

Linux offers multiple GPU compute backends for LLM inference, each with different hardware requirements, quantisation support, and framework ecosystems. Understanding these trade-offs up front helps engineers choose the right backend before diving into implementation details. The table below summarises the major options as of mid-2026, followed by an in-depth look at GGML's own architecture and its Vulkan backend.

| **Backend** | **Hardware** | **API / driver** | **Quantisation support** | **Batching / continuous batching** | **Memory efficiency** | **Framework examples** | **When to use** |
|---|---|---|---|---|---|---|---|
| CUDA 12 | NVIDIA RTX/Tesla/H100 | proprietary nvidia driver or nvidia-open | GGUF Q4/Q8, GPTQ, AWQ, FP8 (Hopper) | Full (vLLM, TGI) | Excellent (unified virtual addressing, NVLink P2P) | llama.cpp (CUDA), vLLM, TGI, Ollama | NVIDIA GPU; maximum performance; broadest framework support |
| ROCm / HIP 6 | AMD RDNA 2+, CDNA (MI-series) | AMDGPU + KFD | GGUF Q4/Q8, GPTQ (via AutoGPTQ-ROCm) | Full (vLLM-ROCm, TGI-ROCm) | Good (xGMI on MI-series) | llama.cpp (hipBLAS), vLLM-ROCm, Ollama | AMD GPU; comparable to CUDA on supported hardware |
| Vulkan Compute (GGML) | Any Vulkan 1.3 GPU (NVIDIA, AMD, Intel, ARM) | Mesa Vulkan or nvidia-open or Intel ANV | GGUF Q4_0, Q4_1, Q5_K, Q8_0 (growing) | Limited (single inference; no KV cache batching) | Good (explicit allocation) | llama.cpp (--n-gpu-layers), koboldcpp | Cross-vendor; Intel Arc; AMD on ROCm-unsupported hardware |
| OpenCL | AMD (ROCm OpenCL), Intel (NEO), NVIDIA (legacy) | ICD loader + vendor runtime | GGUF via clblast (limited) | Very limited | Moderate | llama.cpp (clblast; deprecated in favour of Vulkan) | Legacy; older AMD GPUs before ROCm support |
| CPU (SIMD / AVX-512) | Any x86-64 with AVX2/AVX-512; ARM NEON/SVE | No GPU driver needed | GGUF Q4/Q8 (excellent; GGML designed for CPU Q) | Supported | Limited by RAM bandwidth | llama.cpp (CPU), Ollama CPU | No GPU available; quantised 7B–13B models on capable CPUs; power-constrained |

### LLM Inference Pipeline Paths

The backend choice determines the entire data-flow path from model weights to output tokens. Each path has different memory bandwidth characteristics and GPU driver requirements — choices made at backend selection time ripple through every layer from kernel driver to token throughput.

```mermaid
flowchart LR
    weights["GGUF model weights on disk"]

    subgraph pathA ["Path A: CUDA (NVIDIA)"]
        A1["llama.cpp/vLLM CUDA backend"]
        A2["cuBLAS/cuDNN Tensor Core GEMM"]
        A3["NVIDIA GPU SM compute"]
        A4["Output tokens\nKV cache in VRAM"]
        A1 --> A2 --> A3 --> A4
    end

    subgraph pathB ["Path B: ROCm / HIP (AMD)"]
        B1["llama.cpp hipBLAS or vLLM-ROCm"]
        B2["rocBLAS/hipBLAS Matrix Core GEMM"]
        B3["AMD GPU /dev/kfd + XNACK"]
        B4["Output tokens\nKV cache in HBM"]
        B1 --> B2 --> B3 --> B4
    end

    subgraph pathC ["Path C: Vulkan Compute (cross-vendor)"]
        C1["llama.cpp ggml-vulkan backend"]
        C2["SPIR-V compute shaders\nVK_KHR_storage_buffer"]
        C3["Any Vulkan GPU\n(RADV/ANV/NVK/Turnip)"]
        C4["Output tokens\nKV cache in VRAM"]
        C1 --> C2 --> C3 --> C4
    end

    subgraph pathD ["Path D: CPU SIMD (no GPU)"]
        D1["llama.cpp CPU backend"]
        D2["AVX2/AVX-512 or ARM NEON"]
        D3["CPU DRAM\nbandwidth limited"]
        D4["Output tokens\nKV cache in RAM"]
        D1 --> D2 --> D3 --> D4
    end

    subgraph pathE ["Path E: IREE / MLIR (research/edge)"]
        E1["Python model\nMLIR frontend"]
        E2["IREE compiler\nSPIR-V emit"]
        E3["Vulkan dispatch\nor CPU fallback"]
        E4["Output\nportable runtime"]
        E1 --> E2 --> E3 --> E4
    end

    weights --> A1
    weights --> B1
    weights --> C1
    weights --> D1
    weights --> E1
```

The critical bottleneck in all paths is memory bandwidth, not compute. A 70B model at Q4_0 quantisation requires approximately 35 GB of weight reads per output token, making memory bandwidth the dominant performance constraint at batch=1. This explains why Paths A and B are dramatically faster than Path D: HBM3 on an AMD MI300X delivers over 5 TB/s of bandwidth, GDDR7 on an NVIDIA RTX 5090 provides over 3 TB/s, while CPU DDR5 is limited to 50–100 GB/s — a 30–100× deficit that translates directly into token throughput. Path C (Vulkan compute) achieves approximately 60–80% of the throughput of Paths A and B on equivalent hardware, due to Vulkan dispatch overhead and the absence of hand-tuned vendor kernels like cuBLAS's Tensor Core GEMM implementations. However, Path C's cross-vendor compatibility makes it the only GPU-accelerated option on hardware excluded from CUDA or ROCm support, including Intel Arc GPUs, older AMD GPUs predating ROCm compatibility, and any device with a conformant Vulkan 1.2 driver. Path E (IREE/MLIR) is primarily a research and edge-deployment path; its portable SPIR-V emission trades peak throughput for compiler portability and reproducible deployment artifacts.

### 2.1 The GGML Tensor Engine

GGML is a C library that provides a tensor type system and a DAG-based compute graph executor. Every inference operation — matrix–vector multiply, RoPE, softmax, layer normalisation — is expressed as a node in a `ggml_cgraph`. Nodes hold `ggml_tensor` structs that record shape (up to four dimensions), data type, a pointer to backing storage, and a `backend_buffer` reference that describes where the data lives. [Source](https://github.com/ggml-org/llama.cpp)

Data types include:
- Floating-point formats: `GGML_TYPE_F32`, `GGML_TYPE_F16`, `GGML_TYPE_BF16`
- Integer quantisation types: `GGML_TYPE_Q4_0`, `GGML_TYPE_Q4_1`, `GGML_TYPE_Q8_0`
- K-quant block formats: `GGML_TYPE_Q2_K`, `GGML_TYPE_Q3_K_S/M/L`, `GGML_TYPE_Q4_K_S/M`, `GGML_TYPE_Q5_K_S/M`, `GGML_TYPE_Q6_K`
- I-quant formats (importance-matrix-guided): `GGML_TYPE_IQ2_XXS`, `GGML_TYPE_IQ4_XS`, etc.

K-quant types are block-quantised: a `Q4_K_M` tensor stores weights in 256-element super-blocks, each containing two `Q4_0` sub-blocks of 32 elements sharing a per-block scale and minimum value. This yields approximately 4.5 bits/weight while preserving more of the weight distribution than plain `Q4_0`.

The backend abstraction (`ggml_backend_i`) decouples tensor storage from the executor. Each backend registers:
- `get_name` — returns `"Vulkan"`, `"CUDA"`, `"ROCm"`, `"CPU"`, etc.
- `free` — releases device resources
- `buffer_type` — returns a `ggml_backend_buffer_type_t` that knows how to allocate and zero tensors on this device
- `graph_plan_compute` / `graph_compute` — executes a previously built plan
- `supports_op` — returns whether this backend can natively handle a given op

Backends are discovered and registered at startup via `ggml_backend_load_all`, which walks a list of compiled-in backends in priority order (CUDA > ROCm > Vulkan > SYCL > CPU). The first backend that reports a usable device wins tensor assignment for GPU layers. [Source](https://deepwiki.com/ggml-org/llama.cpp)

### 2.2 The Vulkan Backend Architecture

The Vulkan backend (`ggml-vulkan.cpp` in the GGML source tree) enables inference on any Vulkan 1.2-capable GPU: NVIDIA, AMD, Intel Arc, Qualcomm Adreno, ARM Mali, and IMG GPUs on Linux. [Source](https://deepwiki.com/ggml-org/llama.cpp/5.3-vulkan-backend-(cross-platform))

#### Core Device Abstraction

The central structure is `vk_device_struct` (typedef `vk_device`):

```cpp
// Simplified from ggml-vulkan.cpp
struct vk_device_struct {
    vk::PhysicalDevice physical_device;
    vk::Device         device;
    vk::Queue          compute_queue;
    vk::Queue          transfer_queue;

    uint32_t  subgroup_size;     // wavefront/warp size
    uint32_t  vendor_id;         // PCI vendor: 0x10DE NVIDIA, 0x1002 AMD, 0x8086 Intel
    bool      fp16_support;
    bool      bf16_support;
    bool      coopmat_support;   // VK_KHR_cooperative_matrix
    bool      coopmat2_support;  // newer coopmat variant

    std::unordered_map<std::string, vk_pipeline> pipeline_cache;
    vk::PipelineCache vk_pipeline_cache;
    // ... pinned host memory pool, descriptor pool, command pools
};
```

The backend wraps a `vk_device` into a `ggml_vk_context`:

```cpp
struct ggml_vk_context {
    vk_device device;
    // per-context state: in-flight command buffers, semaphores
    std::vector<vk_context_struct> contexts;
    vk::Fence fence;
};
```

#### Pipeline and Shader System

Each compute shader is compiled from GLSL source (`.comp` files under `ggml/src/ggml-vulkan-shaders/`) to SPIR-V at build time by the `vulkan-shaders-gen` tool using `glslc` from the Vulkan SDK. The compiled SPIR-V blobs are embedded as C arrays in the generated header `ggml-vulkan-shaders.hpp` and loaded at library initialisation via `vkCreateShaderModule`. No runtime shader compilation is required; the backend simply calls `vkCreateComputePipeline` once per pipeline variant on first use.

Pipeline variants are registered in a `vk_matmul_pipeline_struct`:

```cpp
struct vk_matmul_pipeline_struct {
    vk_pipeline l;   // "large": for M >= 32 (batch decode or prompt)
    vk_pipeline m;   // "medium": M 8–31
    vk_pipeline s;   // "small": M 1–7 (single-token generation)
    vk_pipeline a;   // aligned variant requiring 4-byte-aligned strides
};
```

More than 200 pipeline variants exist across quantisation types, precision combinations (FP16 accumulators vs FP32), and hardware capability tiers (with/without cooperative matrix extensions). The selection logic in `ggml_vk_get_pipeline` uses the tensor's data type, the batch dimension, and device capability flags to choose the fastest variant at runtime.

#### Push Constants and Matrix–Vector Multiply

Shape parameters are passed as Vulkan push constants rather than descriptor-set bindings to minimise CPU overhead per dispatch. For the core GEMM shader:

```glsl
// From mul_mm.comp (simplified)
layout(push_constant) uniform PushConstants {
    uint M;       // rows of A / rows of output
    uint N;       // rows of B (columns of output)
    uint K;       // shared dimension (model hidden size)
    uint stride_a;
    uint stride_b;
    uint stride_d;
    float scale;
} pcs;
```

Each workgroup loads a tile of A and a tile of B into shared memory (`shared float As[BM][BK]; shared float Bs[BK][BN];`), performs the tile multiply using a standard blocked GEMM loop, and accumulates into registers. On devices with `VK_KHR_cooperative_matrix`, the shader uses `coopMatMulAdd` to exploit tensor core hardware.

#### Memory Buffer Types

Three buffer allocation strategies serve different GPU configurations:

| Buffer type | `vk::MemoryPropertyFlags` | Use case |
|---|---|---|
| Device-local | `eDeviceLocal` | Discrete GPU VRAM (weights, KV cache, activations) |
| Host-visible device-local | `eDeviceLocal \| eHostVisible \| eHostCoherent` | Resizable BAR / SAM-enabled systems (zero-copy staging) |
| Host pinned | `eHostVisible \| eHostCoherent \| eHostCached` | Staging buffers for discrete GPU weight upload |

On discrete GPUs without Resizable BAR, weights travel through a pinned staging buffer: `vkCmdCopyBuffer` submits the transfer on the `transfer_queue`, with a `vkQueueWaitIdle` (or timeline semaphore) before the compute queue reads the tensor. On systems with ReBAR enabled, the device-local buffer is simultaneously host-visible, and `memcpy` directly fills GPU VRAM without staging [Source](https://github.com/ggml-org/llama.cpp/issues/21590).

---

## 3. llama.cpp Vulkan Path in Depth

### 3.1 Initialisation

```cpp
// High-level call sequence at startup
llama_backend_init();          // registers all ggml backends via ggml_backend_load_all()
llama_model *model = llama_load_model_from_file("model.gguf", params);
llama_context *ctx  = llama_new_context_with_model(model, ctx_params);
```

`llama_backend_init` triggers `ggml_backend_vk_init(device_index)` for each discovered Vulkan physical device. The Vulkan backend enumerates physical devices via `vkEnumeratePhysicalDevices`, filters out devices without the required compute queue family and without Vulkan 1.2 support, then calls `vkCreateDevice` to create a logical device. Vendor-specific heuristics adjust the device selection:

- AMD GPUs are preferred over Intel integrated graphics when both are present.
- A device with `VK_KHR_cooperative_matrix` (tensor cores) is preferred over one without.
- The user can force a device index via `GGML_VK_VISIBLE_DEVICES`.

### 3.2 KV Cache and Buffer Allocation

The KV cache is allocated on the GPU backend chosen for layer 0 by default, via `ggml_backend_vk_buffer_type`. For a Llama-3-8B model at context length 8192, the K and V tensors across 32 layers at F16 precision occupy:

```
32 layers × 2 (K+V) × 8192 tokens × 8 heads × 128 head_dim × 2 bytes = ~4 GB
```

With grouped-query attention (GQA) as used in Llama-3-8B (8 KV heads vs 32 Q heads), this shrinks to:

```
32 × 2 × 8192 × 8 × 128 × 2 = ~512 MB
```

The KV cache is backed by a `vk::Buffer` allocated with `eDeviceLocal` flags. The `llama_kv_cache` struct tracks `cache_k_l%d` and `cache_v_l%d` tensors per layer, stored as contiguous slabs in VRAM. A `cell_data` array on the CPU tracks which context slots are occupied, and `find_slot()` assigns positions to incoming tokens.

### 3.3 Compute Graph: `build_llama`

For each decode step, `llama_build_graph` calls the architecture-specific graph builder. For LLaMA-family models this is `build_llama`:

```
Input token IDs → ggml_get_rows(embedding table) → residual stream
  for each transformer layer:
    ggml_norm → Q/K/V projection (ggml_mul_mat) → Q: ggml_rope, K: ggml_rope
    K/V → write into KV cache slab (ggml_cpy)
    Attention: ggml_flash_attn_ext (if Flash Attention) or manual QK^T → softmax → V
    FFN: gate·up projections → SiLU activation (ggml_silu) → down projection
    Residual add
  Final norm → logits projection
```

The entire graph is a pure DAG of `ggml_tensor` nodes. No loops or conditionals appear at the GGML level; branching is handled by building different graphs for prefill (full prompt processing) versus decode (single-token generation).

**RoPE** is applied as a fused kernel: `ggml_rope_custom` takes the Q or K tensor and computes `q_rotated[i] = q[i]*cos(θ) - q[i+1]*sin(θ)` for each head dimension pair, where `θ = pos / base^(2i/d)`. The Vulkan shader does this in-place over a workgroup of threads mapped to the batch×head×dim axes.

**Flash Attention** in GGML's Vulkan backend is implemented in `flash_attn.comp` and its cooperative-matrix variants. The shader computes the attention output in tiles, accumulating in FP32 with online softmax normalisation (the Dao–Milovanovic algorithm). This avoids materialising the full `N×N` attention matrix in global memory, saving VRAM especially at long contexts. On cooperative-matrix hardware, `flash_attn_cm1.comp` and `flash_attn_cm2.comp` use `coopMatMulAdd` for the QK^T and (softmax weights × V) multiplies. [Source](https://deepwiki.com/ggml-org/llama.cpp/5.3-vulkan-backend-(cross-platform))

### 3.4 Multi-GPU Layer Split

The `--n-gpu-layers N` flag (short: `-ngl N`) tells llama.cpp to assign the bottom N transformer layers to the GPU backend (Vulkan or CUDA), leaving the remainder on CPU. When N equals the total layer count, the full model including the embedding table and final norm runs on GPU.

For models that exceed a single GPU's VRAM, `--split-mode row` distributes rows of the weight matrices across up to four GPUs via the RPC backend or direct multi-device scheduling. Each GPU receives a contiguous slab of weight rows; partial results are summed across devices with an `ggml_all_reduce`-style reduction. The `--tensor-split` argument accepts a comma-separated ratio list (e.g., `--tensor-split 0.6,0.4`) to bias the split toward a larger-VRAM device. [Source](https://github.com/ggml-org/llama.cpp/discussions/7678)

### 3.5 Representative Benchmarks

The following figures are from community benchmarks using `llama-bench` on Llama-3-8B with Q4\_0 quantisation (pp512 = prompt processing at batch 512, tg128 = token generation at batch 1). Q4\_K\_M generation speed is comparable; prompt processing is typically 5–15% slower due to dequantisation overhead. [Source](https://knightli.com/en/2026/04/23/llama-cpp-gpu-benchmark-cuda-rocm-vulkan-scoreboard/)

| Hardware | Backend | pp512 (t/s) | tg128 (t/s) |
|---|---|---|---|
| NVIDIA RTX 4090 24 GB | CUDA | ~11,993 | ~186 |
| AMD RX 7900 XTX 24 GB | ROCm | ~3,552 | ~167 |
| Intel Arc A770 16 GB | Vulkan | ~1,074 | ~53 |

Note: prompt processing speed scales roughly with compute (tensor core TFLOPS); generation speed scales with memory bandwidth. The RTX 4090's 1 TB/s bandwidth advantage drives its generation edge over the RX 7900 XTX (960 GB/s). These numbers represent single-stream inference; multi-user batching changes the picture significantly. Note: verify against your specific build tag and driver version.

---

## 4. Memory-Mapped Weights and DMA-BUF

### 4.1 GGUF File Format

The GGUF (GGML Universal File Format) binary container packs model weights, tokeniser vocabulary, and hyperparameters into a single self-describing file. Its structure: [Source](https://deepwiki.com/ggml-org/llama.cpp/7.1-gguf-file-format)

```
┌─────────────────────────────────────────────┐
│ Header (fixed, 16 bytes minimum)            │
│   magic:          0x46554747  ("GGUF")      │
│   version:        uint32   (currently 3)    │
│   n_tensors:      uint64                    │
│   n_kv:           uint64                    │
├─────────────────────────────────────────────┤
│ KV Metadata Section (variable length)       │
│   key-value pairs: tokenizer.model,         │
│   llama.context_length, llama.block_count,  │
│   llama.attention.head_count, etc.          │
├─────────────────────────────────────────────┤
│ Tensor Info Section                         │
│   per tensor: name, n_dims, dims[], type,   │
│               offset (from data start)      │
├─────────────────────────────────────────────┤
│ Padding to GGUF_DEFAULT_ALIGNMENT = 32 bytes│
├─────────────────────────────────────────────┤
│ Tensor Data Section                         │
│   raw weight bytes, each tensor at its      │
│   declared offset, aligned to 32 bytes      │
└─────────────────────────────────────────────┘
```

The 32-byte alignment guarantee means that the tensor data section can be `mmap`-ped directly into process address space and individual tensor pointers handed to Vulkan or CUDA without any byte-copying for host-side CPU inference.

### 4.2 Memory-Mapped Loading

llama.cpp opens the GGUF file and calls `mmap(2)` with `MAP_SHARED | MAP_POPULATE` when `--mmap` is enabled (the default). This maps the entire file into the process address space. The kernel's page cache backs the mapping: the physical pages remain on disk until accessed, and the OS transparently faults them in on first read.

```c
// Simplified from llama.cpp src/llama-mmap.hpp
void * mmap_file(int fd, size_t file_size) {
    void *addr = mmap(nullptr, file_size,
                      PROT_READ, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        throw std::runtime_error("mmap failed");
    }
    madvise(addr, file_size, MADV_SEQUENTIAL);  // hint for prefetcher
    return addr;
}
```

During the **first inference pass**, each weight tensor not yet in VRAM triggers a page fault as the CPU accesses the mmap region. The kernel reads pages from disk (or SSD) into the page cache. This produces the characteristic "slow first run, fast subsequent runs" behaviour: models smaller than available RAM are fully cached in the page cache after the first load and subsequent llama.cpp invocations (even in a new process) skip disk I/O entirely.

### 4.3 Host-to-GPU Transfer

For GPU-offloaded layers, the weight tensors must move from host memory (page cache) to GPU VRAM. The transfer path in the Vulkan backend:

1. `ggml_backend_vk_buffer_from_ptr(ctx, ptr, size)` — wraps a host pointer in a `VkBuffer` backed by `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT` staging memory. On discrete GPUs, this is a separate allocation in host-visible DRAM (typically BAR space or system RAM pinned by the driver).
2. `vkCmdCopyBuffer(cmd, staging_buf, device_buf, 1, &region)` — enqueues a DMA transfer from the staging buffer to device-local VRAM.
3. A pipeline barrier with `VK_ACCESS_TRANSFER_WRITE_BIT → VK_ACCESS_SHADER_READ_BIT` ensures compute shaders see the updated data.

The transfer is not lazy: `llama_new_context_with_model` uploads all GPU-assigned layers up front, before any inference request. The mmap pages are then released (or kept in the page cache) but no longer needed for compute; the authoritative copy lives in VRAM.

```cpp
// Pseudocode: uploading one layer's weight tensor to Vulkan VRAM
vk::Buffer staging = create_host_visible_buffer(device, tensor->nbytes());
void *mapped = device.mapMemory(staging_mem, 0, VK_WHOLE_SIZE);
memcpy(mapped, tensor->data, tensor->nbytes());   // page fault happens here
device.unmapMemory(staging_mem);

vk::CommandBuffer cmd = begin_one_shot_cmd(device, transfer_pool);
vk::BufferCopy region{0, 0, tensor->nbytes()};
cmd.copyBuffer(staging, vram_buf, region);
end_and_submit_cmd(cmd, transfer_queue);
device.waitIdle();  // synchronous for initialisation
```

### 4.4 Models Larger than VRAM: Split Mode and Lazy Layers

When the model does not fit in VRAM, `--split-mode row` or `-sm none` (CPU offload) assigns some layers to the CPU backend. CPU layers use the mmap pointer directly — no VRAM transfer — so their page-fault cost is deferred until those layers execute during inference. This produces per-token latency spikes proportional to the number of CPU layers accessing cold pages; `madvise(MADV_WILLNEED)` on the tensor data range during model load mitigates this by prefetching ahead of time.

### 4.5 Resizable BAR and Zero-Copy

On modern systems with Resizable BAR (AMD SAM, Intel RST) enabled in UEFI, the CPU can address the full GPU VRAM aperture over PCIe. The Vulkan memory type `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT` becomes available. When detected, the GGML Vulkan backend allocates weight buffers in this type and maps them persistently. `memcpy` from the mmap region directly fills VRAM, eliminating the staging buffer and halving the memory bandwidth consumed during model loading. [Source](https://github.com/ggml-org/llama.cpp/issues/21590)

Resizable BAR availability can be confirmed via `drm_device` sysfs:

```bash
cat /sys/bus/pci/devices/0000:09:00.0/resource | awk 'NR==2{size=$2-$1; printf "BAR1: %d MB\n", size/1024/1024}'
# With ReBAR: BAR1: 24576 MB (full VRAM)
# Without:    BAR1: 256 MB  (legacy 256 MB aperture)
```

---

## 5. Ollama: GPU Dispatch and Model Management

### 5.1 Architecture Overview

Ollama is a Go service that wraps llama.cpp (and, since v0.30, builds on llama.cpp directly without a separate GGML layer). The service exposes a REST API and manages model download, storage, lifecycle, and GPU dispatch. Its architecture comprises: [Source](https://github.com/ollama/ollama)

- **`ollama serve`**: The main Go HTTP server. Listens on `127.0.0.1:11434` by default.
- **Runner subprocess**: For each loaded model, a separate process (`ollama_llama_server`) runs the llama.cpp inference loop. Communication between server and runner uses JSON-RPC over stdin/stdout or a local TCP port.
- **GPU detection layer** (`gpu/gpu.go`, `gpu/amd_linux.go`, `gpu/nvidia_linux.go`): Probes hardware at startup.

### 5.2 GPU Detection

Ollama's GPU detection follows a layered probe strategy:

**NVIDIA**: Dynamically loads `libnvidia-ml.so` (NVML) at runtime to avoid a hard dependency. Calls `nvmlInit()`, `nvmlDeviceGetCount()`, and `nvmlDeviceGetMemoryInfo()` to enumerate GPUs and read free/total VRAM. Falls back to `cudaMemGetInfo()` if NVML is unavailable. For Jetson/Tegra unified memory devices, reads `/proc/meminfo` to determine available system RAM. Device UUIDs from `nvmlDeviceGetUUID()` allow stable identification across reboots.

**AMD**: Parses `/sys/class/kfd/kfd/topology/nodes/*/properties` to enumerate KFD (Kernel Fusion Driver) GPU nodes. Maps each KFD node to a DRM render node via `/sys/class/drm/card*/device/` symlinks for memory reporting. Validates ROCm library compatibility by checking for TensileLibrary files in the ROCm installation. [Source](https://deepwiki.com/13rac1/ollama/5.3-gpu-discovery-and-hardware-acceleration)

**Intel**: Queries OpenCL or Level Zero for Intel GPU devices; falls back to sysfs `i915` entries when OpenCL is absent.

**Vulkan fallback**: When neither CUDA nor ROCm is detected, Ollama can enumerate Vulkan devices and route inference to the Vulkan backend.

Environment overrides:
```bash
CUDA_VISIBLE_DEVICES=0,1         # limit to first two NVIDIA GPUs
ROCR_VISIBLE_DEVICES=0           # limit to first AMD GPU
OLLAMA_GPU_OVERHEAD=1073741824   # reserve 1 GB safety margin
OLLAMA_NUM_GPU=1                 # force single GPU even if multiple detected
```

Supported backend identifiers as of Ollama 0.30+ include `cuda_v12`, `cuda_v13`, `rocm_v7_1`, `rocm_v7_2`, `vulkan`, `cuda_jetpack5`, and `cuda_jetpack6`.

### 5.3 Model Library and Layer Caching

Downloaded models live in `~/.ollama/models/blobs/` as GGUF files (or OCI-layer-style content-addressed blobs). Manifests in `~/.ollama/models/manifests/registry.ollama.ai/library/` record the layer digests, media types, and sizes. Ollama's layer-addressed storage means that two models sharing a base checkpoint (e.g., `llama3:8b` and `llama3:8b-instruct`) share the base weight blobs on disk and in page cache.

### 5.4 REST API

The primary endpoints:

```
POST /api/generate
  Body: {"model": "llama3:8b", "prompt": "Hello", "stream": true}
  Response: newline-delimited JSON, one token per line

POST /api/chat
  Body: {"model": "llama3:8b", "messages": [{"role": "user", "content": "Hello"}]}

POST /api/embeddings
  Body: {"model": "nomic-embed-text", "prompt": "text to embed"}

GET  /api/ps         # list running models
POST /api/pull       # download a model
DELETE /api/delete   # remove a model
```

Ollama maintains a model LRU cache: models stay loaded in VRAM for `OLLAMA_KEEP_ALIVE` seconds (default 5 minutes) after the last request. Parallel requests to the same model reuse the loaded runner; requests to a second model either evict the first (if VRAM is exhausted) or launch a second runner on remaining VRAM.

### 5.5 Parallel Request Handling

Within a single runner, Ollama serialises requests by default. When `OLLAMA_NUM_PARALLEL` is set, the runner uses llama.cpp's batched-decode path: multiple prompts are processed together in a single `llama_decode` call with a padded batch, sharing KV cache VRAM. Context slots are allocated round-robin and swapped to CPU when exhausted. This is the same mechanism as llama.cpp's `--parallel N` flag.

---

## 6. ONNX Runtime with GPU Execution Providers

### 6.1 Execution Provider Hierarchy

ONNX Runtime (ORT) is a cross-platform inference engine for ONNX models. It dispatches graph nodes to hardware via an **Execution Provider** (EP) plugin architecture. When a session is created with multiple EPs, ORT partitions the graph: each node goes to the highest-priority EP that declares `GetCapability` for that op. Remaining nodes fall back to the CPU EP. [Source](https://onnxruntime.ai/docs/execution-providers/)

Typical priority ordering for a Linux system with an NVIDIA GPU:

```
TensorRT EP → CUDA EP → ROCm EP → OpenVINO EP → CPU EP
```

For AMD hardware:

```
ROCm EP → CPU EP
```

For Intel:

```
OpenVINO EP → CPU EP
```

### 6.2 CUDA Execution Provider

The CUDA EP uses cuDNN for convolution and softmax, and cuBLAS for GEMM. Configuration via `OrtCUDAProviderOptionsV2`:

```cpp
#include <onnxruntime_cxx_api.h>

Ort::SessionOptions session_opts;

// Modern V2 API (preferred)
auto cuda_opts = OrtCUDAProviderOptionsV2();
cuda_opts.update({
    {"device_id",              "0"},
    {"gpu_mem_limit",          "8589934592"},  // 8 GB
    {"arena_extend_strategy",  "kNextPowerOfTwo"},
    {"enable_cuda_graph",      "1"},
    {"use_tf32",               "1"},
    {"cudnn_conv_algo_search", "HEURISTIC"},
});
session_opts.AppendExecutionProvider("CUDA", cuda_opts.GetOptions());

Ort::Session session(env, "model.onnx", session_opts);
```

Key configuration fields in `OrtCUDAProviderOptions` / `OrtCUDAProviderOptionsV2` [Source](https://onnxruntime.ai/docs/api/c/struct_ort_c_u_d_a_provider_options.html):

| Field | Purpose |
|---|---|
| `device_id` | GPU ordinal (0-indexed) |
| `gpu_mem_limit` | Hard cap on memory arena size in bytes |
| `arena_extend_strategy` | `kNextPowerOfTwo` (0) exponential growth; `kSameAsRequested` (1) exact |
| `cudnn_conv_algo_search` | `EXHAUSTIVE` (0), `HEURISTIC` (1), `DEFAULT` (2) |
| `enable_cuda_graph` | Capture the static graph as a CUDA Graph; eliminates per-iteration kernel launch overhead |
| `use_tf32` | Enable TensorFloat-32 on Ampere+ for GEMM (default on) |
| `user_compute_stream` | Custom `cudaStream_t` for pipelining with application streams |

**CUDA Graphs**: When `enable_cuda_graph=1`, the first `session.Run()` call captures all CUDA kernel launches into a CUDA Graph object. Subsequent calls replay the graph with a single `cudaGraphLaunch`, eliminating CPU-side kernel dispatch overhead. This is particularly effective for fixed-shape transformer inference. Constraint: all nodes must partition to CUDA EP, and input/output tensor addresses must be stable (use `IoBinding`).

**Graph optimisations**: ORT applies transformations before EP assignment: constant folding, identity elimination, layer fusion (e.g., fusing `Add + LayerNorm` → `SkipLayerNorm` kernel, `Gelu + MatMul` → `FusedMatMul`). The CUDA EP adds further fusions: attention heads are fused into a single `MultiHeadAttention` kernel when recognised by the `TransformerOptimizer` pass.

### 6.3 Model Quantisation

ORT's quantisation tooling converts FP32 ONNX models to INT8 or FP16:

```python
from onnxruntime.quantization import quantize_dynamic, QuantType

quantize_dynamic(
    "model.onnx",
    "model_int8.onnx",
    weight_type=QuantType.QInt8,
    per_channel=True,
    reduce_range=True,  # use 7-bit range to avoid VNNI saturation bugs
)
```

For GPU, FP16 conversion is more common:

```bash
python -m onnxruntime.tools.convert_onnx_models_to_fp16 model.onnx model_fp16.onnx
```

---

## 7. ONNX Runtime OpenVINO EP

### 7.1 Intel OpenVINO Architecture

Intel OpenVINO is an inference toolkit optimised for Intel hardware: CPUs (via oneDNN/AVX-512), integrated GPU (Gen12 / Xe), discrete Arc GPUs, and NPUs (Meteor Lake, Lunar Lake). The OpenVINO Model Optimizer converts ONNX/TensorFlow/PaddlePaddle graphs into the OpenVINO Intermediate Representation (IR) format: an XML topology file paired with a `.bin` weights file. Since OpenVINO 2022.1, direct ONNX ingestion is supported without explicit IR conversion. [Source](https://onnxruntime.ai/docs/execution-providers/OpenVINO-ExecutionProvider.html)

### 7.2 OrtOpenVINOProviderOptions

```cpp
// Legacy C API (deprecated but still widely used)
OrtOpenVINOProviderOptions ov_opts;
ov_opts.device_type = "GPU";          // or "CPU", "NPU", "AUTO", "HETERO:GPU,CPU"
ov_opts.num_of_threads = 8;
ov_opts.cache_dir = "/tmp/ov_cache";  // compiled model cache

session_opts.AppendExecutionProvider_OpenVINO(ov_opts);
```

The modern V2 API uses `load_config` with a JSON string:

```cpp
session_opts.AppendExecutionProvider("OpenVINO", {
    {"device_type",        "GPU"},
    {"cache_dir",          "/tmp/ov_cache"},
    {"PERFORMANCE_HINT",   "THROUGHPUT"},
    {"INFERENCE_PRECISION_HINT", "f16"},
    {"NUM_STREAMS",        "2"},
});
```

**Device type values**:
- `"CPU"` — Intel CPU via oneDNN kernels
- `"GPU"` — Intel integrated or discrete GPU (detected automatically)
- `"GPU.0"`, `"GPU.1"` — Explicit GPU selection by index
- `"NPU"` — Intel NPU (Meteor Lake/Lunar Lake integrated)
- `"AUTO"` — OpenVINO's Auto plugin selects the fastest available device at runtime
- `"HETERO:GPU,CPU"` — Heterogeneous execution: ops unsupported on GPU fall back to CPU

### 7.3 Level Zero Backend for Intel Arc

On Linux, OpenVINO's GPU plugin routes compute to Intel Arc (and earlier iGPU) through the Level Zero API. Level Zero (part of oneAPI) is Intel's low-level compute API analogous to Vulkan compute. The OpenVINO GPU plugin compiles OpenCL kernels (and increasingly SYCL) to Intel GPU ISA via the Intel Graphics Compiler (IGC) and dispatches them through Level Zero command queues. [Source](https://docs.openvino.ai/2025/get-started/install-openvino/configurations/configurations-intel-gpu.html)

For Intel Arc A770 on Linux, required kernel support:
- Linux kernel 6.2+ for the `xe` driver (recommended for Arc Alchemist) or `i915` with xe-specific patches
- Level Zero runtime: `intel-level-zero-gpu` package
- OpenCL runtime: `intel-opencl-icd`

The ORT session with `device_type="GPU"` will call `ov::Core::compile_model("model.onnx", "GPU")`, which invokes the Level Zero backend. The first compilation triggers IGC shader compilation and caches the result to `cache_dir`, making subsequent loads sub-second.

### 7.4 Heterogeneous Execution

For LLM inference where some attention operations are not yet supported on Intel NPU, `HETERO:NPU,GPU,CPU` distributes ops across all three devices. OpenVINO queries each plugin's `GetMetric(SUPPORTED_PROPERTIES)` to build an op compatibility map and partitions the graph accordingly. The overhead of cross-device tensor copies is amortised when long operations (e.g., large matrix multiplies) run on the faster device.

On Intel Arc A770 (512 EU, ~8 TFLOPS FP16), llama.cpp with Vulkan reaches approximately 1,074 tokens/s prompt processing and 53 tokens/s generation on Llama-3-8B Q4\_0 (see §3.5 benchmarks). ORT+OpenVINO on the same GPU with an FP16 ONNX model shows competitive generation speed for smaller models, with advantage in batch throughput scenarios due to OpenVINO's graph-level optimisations. Note: direct ORT-vs-llama.cpp numbers are architecture-dependent; verify against your model and driver version.

---

## 8. ROCm MIOpen and HIP for LLM Inference

### 8.1 The ROCm Stack

AMD's ROCm (Radeon Open Compute) platform provides the GPU compute stack for Linux on RDNA and CDNA hardware. The relevant components for LLM inference:

- **HIP**: CUDA-like programming model and runtime. `hipMalloc`, `hipMemcpy`, `hipLaunchKernelGGL` mirror CUDA equivalents. Most CUDA kernel code ports with `hipify` tooling.
- **rocBLAS / hipBLASLt**: BLAS and GEMM libraries. `rocblas_gemm_ex` dispatches to optimised RDNA3/CDNA3 matrix kernels.
- **MIOpen**: Deep learning primitive library (convolutions, attention, normalisation). On ROCm 7.1+, MIOpen ships TunableOp support for auto-selecting the fastest GEMM algorithm from rocBLAS and hipBLASLt across thousands of variants. [Source](https://github.com/ROCm/MIOpen)
- **RCCL**: ROCm equivalent of NCCL for multi-GPU collective operations.
- **AITER (AI Tensor Engine for ROCm)**: A repository of high-performance inference kernels from Triton, Composable Kernel, HIP, and hand-written assembly. [Source](https://rocm.blogs.amd.com/software-tools-optimization/vllm-0.9.x-rocm/README.html)

### 8.2 HIP GEMM Path

A typical GEMM call in ROCm-aware inference code:

```cpp
#include <rocblas/rocblas.h>

rocblas_handle handle;
rocblas_create_handle(&handle);

// Matrix A: M×K, B: K×N, C: M×N (all row-major)
float alpha = 1.0f, beta = 0.0f;
rocblas_gemm_ex(
    handle,
    rocblas_operation_none,    // A not transposed
    rocblas_operation_trans,   // B transposed (weights are stored K×N)
    M, N, K,
    &alpha,
    A_ptr, rocblas_datatype_f16_r, lda,
    B_ptr, rocblas_datatype_f16_r, ldb,
    &beta,
    C_ptr, rocblas_datatype_f32_r, ldc,   // accumulate in FP32
    C_ptr, rocblas_datatype_f32_r, ldc,
    rocblas_datatype_f32_r,                // compute type
    rocblas_gemm_algo_standard,
    0,   // solution index (0 = auto)
    0    // flags
);
```

hipBLASLt (available on RDNA3+) extends this with TunableOp: at first run, it benchmarks hundreds of algorithmic variants (tile sizes, pipeline depths, wave quantisation strategies) and caches the winner in a solution database. Subsequent runs use the cached solution, eliminating the tuning overhead.

### 8.3 Memory Allocation Strategies

```cpp
// Discrete GPU VRAM — standard path
void *d_weights;
hipMalloc(&d_weights, weight_size);
hipMemcpy(d_weights, h_weights, weight_size, hipMemcpyHostToDevice);

// Unified memory — for AMD APUs (Ryzen AI Max, Strix Halo) where CPU and GPU share DRAM
void *um_ptr;
hipMallocManaged(&um_ptr, weight_size);
// No explicit copy needed; HMM (Heterogeneous Memory Management) migrates pages
// on demand. On Strix Halo (Ryzen AI Max 395), the 128 GB shared pool allows
// running 70B models entirely in unified memory.
```

AMD's Strix Halo (Ryzen AI Max+ 395) represents a significant architectural development: a 40-CU RDNA3.5 iGPU sharing a 128 GB LPDDR5x pool with the CPU via HMM. The unified address space means `hipMallocManaged` pages do not migrate over PCIe; they reside in the shared DRAM pool, making 70B models viable on a 65 W APU at ~8–12 tokens/second generation speed. [Source](https://localaimaster.com/blog/amd-rocm-local-llm-setup)

### 8.4 PyTorch Compatibility Shim

ROCm ships a compatibility shim so that PyTorch code written for CUDA runs unmodified: `torch.cuda.is_available()` returns `True`, `tensor.cuda()` allocates HIP memory, and CUDA kernel extensions are hipified transparently. This enables frameworks like vLLM to run on AMD GPUs with minimal code changes:

```bash
# Install PyTorch for ROCm
pip install torch --index-url https://download.pytorch.org/whl/rocm6.2

# Verify
python -c "import torch; print(torch.cuda.get_device_name(0))"
# AMD Radeon RX 7900 XTX
```

### 8.5 Device Isolation and vLLM on ROCm

```bash
# Restrict vLLM to GPUs 0 and 1
ROCR_VISIBLE_DEVICES=0,1 vllm serve meta-llama/Llama-3-70B-Instruct \
    --tensor-parallel-size 2 \
    --max-model-len 8192

# Enable AITER kernels for maximum throughput on MI300X / RDNA3
VLLM_ROCM_USE_AITER=1 \
VLLM_ROCM_USE_AITER_MOE=1 \
VLLM_ROCM_USE_AITER_MHA=1 \
    vllm serve ...
```

The vLLM ROCm attention backend routes to one of three implementations depending on context length and model type: a Triton unified attention kernel (default), a prefill–decode split kernel optimised for MI300X, or AITER MHA (supports up to 8K context length as of vLLM 0.9.x). [Source](https://rocm.blogs.amd.com/software-tools-optimization/vllm-0.9.x-rocm/README.html)

### 8.6 MIOpen Auto-Tuning

MIOpen's `FindConvolution` (for convolutions) and TunableOp (for GEMM) cache optimal algorithm selections in `~/.config/miopen/` or a path set by `MIOPEN_USER_DB_PATH`. For LLM inference, the attention GEMM shapes are typically fixed (determined by model architecture and batch size), so pre-tuning at deployment time pays off:

```bash
# Pre-tune GEMM shapes for a specific model configuration
MIOPEN_FIND_MODE=NORMAL \
MIOPEN_DEBUG_CONV_IMPLICIT_GEMM_HIP_FWD=1 \
    python tune_gemm.py  # runs representative forward passes to populate cache
```

---

## 9. KV Cache Management Strategies

### 9.1 The KV Cache Problem

Transformer autoregressive generation stores the K and V projections for all previously seen tokens in a KV cache. Without management, KV cache size grows linearly with sequence length and number of concurrent requests. A Llama-3-70B model running 100 concurrent requests at 4096 context length requires:

```
80 layers × 2 × 100 seqs × 4096 tokens × 8 KV heads × 128 head_dim × 2 bytes (FP16)
= 80 × 2 × 100 × 4096 × 8 × 128 × 2 ≈ 84 GB
```

This exceeds the VRAM of all consumer GPUs and most server GPUs. Intelligent KV cache management is therefore not optional for production serving.

### 9.2 PagedAttention (vLLM)

vLLM's PagedAttention, introduced in the original Kwon et al. (2023) paper, treats the KV cache as a paged virtual memory system analogous to OS page tables. [Source](https://docs.vllm.ai/en/stable/design/prefix_caching/)

**Block structure**: KV cache is divided into fixed-size physical blocks (e.g., 16 tokens per block). Each block stores KV vectors for one block of tokens across all heads and layers:

```
Physical block size = block_size × n_kv_heads × head_dim × 2 bytes × 2 (K+V) × n_layers
```

Sequences maintain a **logical block table** mapping logical block index → physical block ID, analogous to a page table. Physical blocks are non-contiguous in VRAM; the attention kernel uses an extra indirection through the block table to gather K and V vectors.

**Block manager**: The `BlockManager` in vLLM allocates physical blocks from a free-block pool. On allocation, it assigns blocks to the requesting sequence. On completion or eviction, blocks are freed. Because blocks are fixed-size and reused, memory fragmentation is bounded to less than one block per sequence — vLLM achieves under 4% memory waste versus up to 60% for naive KV cache pre-allocation.

```python
# vLLM block structure (conceptual)
class KVCacheBlock:
    block_id: int            # physical index in VRAM slab
    block_hash: Optional[int]  # assigned when full, for prefix cache lookup
    ref_count: int           # number of sequences sharing this block
    # doubly-linked list for LRU free queue
    prev: Optional[KVCacheBlock]
    next: Optional[KVCacheBlock]
```

### 9.3 Automatic Prefix Caching

vLLM's Automatic Prefix Caching (APC) extends PagedAttention with hash-based block deduplication. When a sequence fills a block, vLLM computes a hash:

```
block_hash = H(parent_block_hash || token_ids_in_block [|| lora_id] [|| cache_salt])
```

The hash incorporates the parent block's hash (creating a Merkle-chain-like structure), making each block's hash unique to the entire prefix up to that point. Hashes are stored in a global hash map: `block_hash → physical_block_id`. [Source](https://docs.vllm.ai/en/stable/design/prefix_caching/)

When a new request arrives with a shared system prompt, vLLM hashes the prefix blocks and looks them up in the map. Matching blocks have their reference counts incremented; the sequence reuses their KV vectors without recomputation. This yields dramatic speedups for multi-turn chat (shared conversation history) and RAG pipelines (shared context chunks).

Hash algorithms available: `sha256` (default, cryptographically secure), `xxhash` (faster), `sha256_cbor` / `xxhash_cbor` (reproducible serialisation). A per-request `cache_salt` enables privacy isolation in multi-tenant deployments.

**Eviction**: When the free-block pool is empty, vLLM evicts the LRU block with `ref_count == 0`. Blocks with higher logical indices (nearer the end of a sequence) are evicted before earlier ones, as earlier blocks are more likely to be reused as shared prefixes. Evicted blocks lose their hash registration; a future request cannot reuse them and must recompute.

### 9.4 llama.cpp Sliding Window and Context Swapping

llama.cpp's KV cache uses a different approach suited to single-user scenarios:

**Ring buffer (default)**: The KV cache is a fixed-size ring buffer of `n_ctx` slots. When the context fills, the oldest tokens are overwritten. This truncates context but avoids any eviction complexity. For conversational applications, this is adequate because recent context dominates attention anyway.

**Sliding window attention (SWA)**: Llama-3.1 models use SWA in some layers: each token only attends to the previous `n_swa` tokens. The KV cache for SWA layers only needs to hold `n_swa` entries, reducing VRAM. llama.cpp detects the `n_swa` hyperparameter from the GGUF metadata and allocates appropriately.

**CPU swap**: When the KV cache fills during a conversation, llama.cpp can offload the oldest KV blocks to CPU RAM via `ggml_backend_cpu_buffer_type`. The swap is explicit: the application calls `llama_kv_cache_seq_rm` to evict a token range, and on the next decode call the relevant tensors are recomputed. This is coarse-grained compared to vLLM's block-level management.

### 9.5 Memory Pressure Handling

vLLM's scheduler handles VRAM pressure by **preemption**: when a new high-priority request cannot be allocated blocks, vLLM can either:
1. **Swap** the KV cache of a low-priority sequence to CPU RAM (if `cpu_swap_space` is configured)
2. **Recompute** the sequence from scratch (discarding KV cache, re-running prefill on resume)

The swapping path uses `hipMemcpy(host, device, ...)` or `cudaMemcpyAsync` on a dedicated transfer stream, avoiding blocking the compute stream. The hybrid KV cache manager introduced in vLLM 0.7+ provides a unified interface for CPU+GPU block pools, enabling per-request policies on swap behaviour. [Source](https://docs.vllm.ai/en/latest/design/hybrid_kv_cache_manager/)

---

## 10. Performance Tuning and Benchmarking

### 10.1 llama-bench

llama.cpp ships `llama-bench`, a benchmarking tool that measures prompt processing (pp) and token generation (tg) throughput:

```bash
# Benchmark Llama-3-8B Q4_K_M on Vulkan, 5 runs each
./llama-bench \
    -m models/llama-3-8b-q4_k_m.gguf \
    -b vulkan \
    -p 512 \      # prompt tokens (pp512)
    -n 128 \      # generation tokens (tg128)
    -r 5          # repetitions for averaging

# Output format:
# model | size | params | backend | ngl | test | t/s
# llama-3-8b-q4_k_m | 4.92 GiB | 8.03B | Vulkan | 99 | pp512 | 1073.85 ± 29.68
# llama-3-8b-q4_k_m | 4.92 GiB | 8.03B | Vulkan | 99 | tg128 | 52.56 ± 0.11
```

Key metrics:
- **pp (prompt processing)**: tokens/second for the initial prefill. Compute-bound. Scales with tensor core throughput.
- **tg (token generation)**: tokens/second for autoregressive decode. Memory-bandwidth-bound at batch=1. Scales with HBM/GDDR bandwidth.

### 10.2 Quantisation Impact

Quantisation affects three variables: model size, generation speed, and output quality.

| Quantisation | Bits/weight | Size (7B model) | tg speed (RTX 4090) | Perplexity delta (Llama-3-8B) |
|---|---|---|---|---|
| F16 | 16 | ~14.5 GB | ~100 t/s | 0 (baseline) |
| Q8_0 | 8.5 | ~7.7 GB | ~120 t/s | +0.001 |
| Q4_K_M | ~4.5 | ~4.9 GB | ~180 t/s | +0.054 |
| Q3_K_M | ~3.4 | ~3.7 GB | ~195 t/s | +0.25 |
| Q2_K | ~2.6 | ~2.7 GB | ~200 t/s | +0.87 |

Note: exact numbers vary by GPU, driver version, and batch size. Treat these as order-of-magnitude guidance. [Source](https://arxiv.org/html/2601.14277v1)

Generation speed is primarily memory-bandwidth-bound at batch=1: the GPU reads the full weight matrix (roughly 4.9 GB for Q4\_K\_M 7B) per token. Q4\_K\_M achieves 4× bandwidth savings vs F16, so it generates proportionally faster. Q8\_0 sits in an awkward middle ground: it saves 2× memory but its kernel implementation is less efficient on some GPU architectures — on Intel Arc, Q8\_0 achieves only 21–24% of theoretical bandwidth vs 53–64% for Q4\_K\_M, making Q8\_0 generation 4–5× slower than Q4\_K\_M despite only 1.7× more data. [Source](https://github.com/ggml-org/llama.cpp/issues/21517)

At batch sizes above 8, the workload becomes compute-bound (matrix–matrix multiply rather than matrix–vector). In this regime, F16 and Q8\_0 benefit from tensor core utilisation, and their throughput scales near-linearly with batch until TFLOPS saturation.

### 10.3 GPU Utilisation Monitoring

**NVIDIA**:
```bash
nvtop          # interactive, shows GPU%, VRAM, SM clock, power
nvidia-smi dmon -s pucvmet -d 1   # raw monitoring at 1-second intervals
```

**AMD**:
```bash
radeontop      # classic tool; shows GFX%, VRAM bus%, clocks
nvtop          # also supports AMD via libamdplugin.so (ROCm required)
rocm-smi --showuse --showmemuse --showpower  # scriptable
```

**Intel Arc**:
```bash
intel_gpu_top  # from intel-gpu-tools; shows Render/CS engine utilisation
nvtop          # supports Intel via i915/xe drivers (requires kernel ≥ 5.19)
```

During single-user Q4\_K\_M generation, GPU compute utilisation typically reads 15–35%: the GPU is stalled waiting for memory reads, not compute. This low utilisation figure is misleading — the GPU is actually running near 100% memory bandwidth. During prompt processing, compute utilisation reaches 80–95% on a well-tuned CUDA/ROCm path.

### 10.4 Power and Thermal Considerations

LLM inference is a sustained GPU workload. A RTX 4090 draws 350–380 W under inference load; a RX 7900 XTX draws 260–300 W. Power per generated token:

```
tokens/second = 150 (generation, RTX 4090)
power = 370 W
watt-seconds per token = 370 / 150 ≈ 2.5 J/token
```

Thermal throttling becomes relevant in sustained multi-hour inference runs. NVIDIA GPUs throttle when the junction temperature exceeds the TJmax (typically 83°C for throttle point on RTX 4090) or when power delivery is constrained. AMD RDNA3 throttles at ~110°C junction by default. Throttling reduces the GPU clock and thus reduces tokens/second progressively.

Monitoring thermal state:

```bash
# NVIDIA: watch temperature and throttle reason
nvidia-smi -q -d TEMPERATURE,PERFORMANCE | grep -E "GPU 0|Temp|Throttle"

# AMD: temperature and power cap
rocm-smi --showtemp --showpower --showclocks

# Reduce power limit to prevent throttling at cost of some performance
nvidia-smi -pl 300        # set 300W limit on RTX 4090 (from 450W)
rocm-smi --setpoweroverdrive 0 200000   # set 200W limit in milliwatts
```

Setting a power limit 10–15% below TDP often eliminates throttling while reducing generation speed by only 3–8%, since memory bandwidth scales weakly with GPU frequency. [Source](https://arxiv.org/html/2603.23640)

### 10.5 Compute vs Memory-Bandwidth Analysis

To distinguish memory-bandwidth-bound from compute-bound behaviour, compute the **arithmetic intensity** (AI) of the dominant op (token generation GEMM):

```
AI = 2 × M × N × K / (M×K + N×K + M×N) bytes of data
```

For a 4096→4096 linear layer at batch=1 (M=1, N=4096, K=4096):
```
FLOPs = 2 × 1 × 4096 × 4096 ≈ 33.5 MFLOP
Bytes = (1×4096 + 4096×4096 + 1×4096) × 2 bytes ≈ 33.6 MB
AI ≈ 1 FLOP/byte
```

RTX 4090 roofline: memory bandwidth = 1008 GB/s, compute = 82.6 TFLOP/s (FP16).
Crossover (ridge point) = 82600 / 1008 ≈ 82 FLOP/byte.

Since AI=1 << 82, token generation at batch=1 is deeply memory-bandwidth-bound. The GPU spends ~99% of its time waiting for VRAM reads, not computing. This is why quantisation that reduces model size accelerates generation proportionally: fewer bytes read → fewer memory stall cycles.

---

## 11. VRAM Capacity Planning: Will This Model Fit?

Before downloading a multi-gigabyte GGUF file or spinning up an Ollama pull, it is worth answering a cheaper question first: does this model, at this quantisation, at this context length, fit in the VRAM actually installed in the machine? This section covers the sizing math and the tools that automate it.

### 11.1 The Sizing Equation

Total VRAM demand for a single-request llama.cpp/Ollama deployment has three additive terms:

```
VRAM_total ≈ VRAM_weights + VRAM_kv_cache + VRAM_overhead

VRAM_weights   = n_params × bits_per_weight / 8
VRAM_kv_cache  = n_layers × 2 × n_ctx × n_kv_heads × head_dim × bytes_per_elem   (see §9.1)
VRAM_overhead  ≈ 200–800 MB (Vulkan/CUDA context, compute-graph scratch buffers, cuBLAS/rocBLAS workspace)
```

For Llama-3-8B at Q4_K_M (≈4.83 bits/weight effective, per the GGUF K-quant block layout) with an 8K context:

```
VRAM_weights  = 8.03e9 × 4.83 / 8            ≈ 4.85 GB
VRAM_kv_cache = 32 × 2 × 8192 × 8 × 128 × 2   ≈ 0.54 GB   (FP16 KV, GQA with 8 KV heads)
VRAM_overhead ≈ 0.5 GB
-----------------------------------------------
VRAM_total    ≈ 5.9 GB
```

This is the same arithmetic `llama-bench` and Ollama's scheduler perform internally; the tools below just save the hand calculation and read the true per-layer tensor shapes out of the GGUF header instead of assuming uniform layer sizes.

### 11.2 gguf-parser: Estimating Directly from GGUF Metadata

[`gguf-parser-go`](https://github.com/gpustack/gguf-parser-go) (MIT licensed, from the GPUStack project) reads a GGUF file's header — locally or via a ranged HTTP request against a remote URL, without downloading the full weights — and computes a memory/throughput estimate from the actual architecture metadata (layer count, head dimensions, quantisation type per tensor) rather than an approximation:

```bash
gguf-parser --path ~/.cache/lm-studio/models/.../DeepSeek-R1-Distill-Qwen-7B-Q4_K_M.gguf \
  --ctx-size 8192 --flash-attention
```

For a 7.62B-parameter model this reports roughly 7.3 GiB of VRAM under a unified-memory (UMA) device and up to 18.9 GiB under a discrete (NONUMA) device with full GPU offload — the UMA/NONUMA split matters because on Apple Silicon and integrated APUs, `mmap`-loaded weight pages can be shared between the CPU and GPU views of the same physical memory, while a discrete GPU needs its own resident copy. [Source](https://github.com/gpustack/gguf-parser-go)

Relevant flags for capacity planning: `--ctx-size` (context length, feeds the KV cache term), `--tensor-split` (per-GPU VRAM estimate when splitting layers across multiple cards, matching §3.4), `--flash-attention` (flash attention reduces the KV cache's intermediate buffer footprint), and `--mmap-load` (toggles whether weights are assumed memory-mapped, affecting the RAM-vs-VRAM split reported for §4.2's loading path).

### 11.3 Hugging Face Estimators for Transformers-Format Models

For models still in Hugging Face `safetensors`/PyTorch format — before any GGUF conversion — the `accelerate` library's memory estimator answers the same question for the ONNX Runtime and PyTorch/ROCm paths in §6–§8:

```bash
accelerate estimate-memory meta-llama/Llama-3-8B --dtypes float16 int8 int4
```

This loads the model's config onto Accelerate's `meta` device (no weights are actually downloaded) and reports largest-layer and total memory footprint per dtype, accurate to within a few percent of the real value. [Source](https://huggingface.co/docs/accelerate/en/usage_guides/model_size_estimator) The same logic backs the official [`hf-accelerate/model-memory-usage`](https://huggingface.co/spaces/hf-accelerate/model-memory-usage) Space, and a community fork, [`Vokturz/can-it-run-llm`](https://huggingface.co/spaces/Vokturz/can-it-run-llm), extends it to filter by a specific GPU's VRAM and LoRA fine-tuning overhead.

Caveat: Accelerate's estimator measures the cost of loading dense `float32`/`float16`/`int8`/`int4` weights, not llama.cpp's non-uniform K-quant block formats (Q4_K_M, Q5_K_S, etc.), and it does not model the KV cache at all — for the GGUF/llama.cpp/Ollama stack this chapter otherwise covers, gguf-parser (§11.2) is the more accurate tool; Accelerate is the right tool when the deployment target is ONNX Runtime or a ROCm/PyTorch `transformers` pipeline loading safetensors directly.

### 11.4 Runtime Introspection: `ollama ps`, `ollama show`, and GPU Memory Queries

Once a model is actually loaded, the most reliable numbers come from the runtime and the driver, not a pre-flight estimate:

```bash
ollama ps
# NAME              ID              SIZE      PROCESSOR    CONTEXT    UNTIL
# llama3:8b         365c0bd3c000    6.2 GB    100% GPU     8192       4 minutes from now

ollama show llama3:8b   # architecture, parameter count, quantisation, context length
```

`ollama ps`'s `PROCESSOR` column reports the actual split Ollama's scheduler chose: `100% GPU` means the whole model fit in VRAM, `100% CPU` means it did not fit at all, and a mixed percentage (e.g. `48%/52% CPU/GPU`) means Ollama partially offloaded layers. Ollama's scheduler evaluates the required VRAM against what the GPU library reports available and only places a model entirely on one GPU if it fits — otherwise it either splits across GPUs or falls back toward CPU. [Source](https://github.com/ollama/ollama/blob/main/docs/gpu.mdx) On Linux, VRAM is queried through each vendor's device library (NVML for NVIDIA, ROCm's SMI library for AMD); the Vulkan backend's VRAM reporting additionally requires elevated capabilities via `setcap`, and without them Ollama falls back to approximate model-size-based scheduling. [Source](https://github.com/ollama/ollama/blob/main/docs/faq.mdx)

For a direct driver-level reading, independent of any inference runtime:

```bash
nvidia-smi --query-gpu=memory.total,memory.used,memory.free --format=csv
rocm-smi --showmeminfo vram
```

These report the same free/used VRAM figures surfaced by the `nvtop`/`radeontop` monitors in §10.3, and are the ground truth against which any of the estimators above should be checked.

## 12. MacBook vs. Gaming Laptop for Local Inference

The GGML Vulkan/CUDA/ROCm/OpenVINO stack covered in this chapter targets Linux GPUs, but the two most common portable hardware choices for running LLMs locally — an Apple Silicon MacBook and an RTX-class gaming laptop — sit on opposite sides of a fundamental memory-architecture divide worth understanding before choosing either.

### 12.1 Unified Memory vs. Discrete VRAM

Apple Silicon uses a single pool of LPDDR5X, physically on-package, addressable by the CPU, GPU, and Neural Engine without a copy — Apple's unified memory architecture. A discrete gaming-laptop GPU instead has its own dedicated GDDR7/GDDR6 VRAM pool, separate from system RAM and reachable from the CPU only across PCIe.

This has a direct capacity consequence for LLM inference: Apple's M4 Max ships with up to 128 GB of unified memory at up to 546 GB/s of bandwidth, and the M3 Ultra scales to 512 GB at up to 819 GB/s — both figures from Apple's own announcements. [Source](https://www.apple.com/newsroom/2024/10/new-macbook-pro-features-m4-family-of-chips-and-apple-intelligence/) [Source](https://www.apple.com/newsroom/2025/03/apple-reveals-m3-ultra-taking-apple-silicon-to-a-new-extreme/) A Mac Studio with an M3 Ultra and 512 GB of unified memory has been demonstrated running the 671B-parameter DeepSeek-R1 model entirely resident in memory, something no single discrete consumer GPU can do at any quantisation. [Source](https://www.techradar.com/pro/apple-mac-studio-m3-ultra-workstation-can-run-deepseek-r1-671b-ai-model-entirely-in-memory-using-less-than-200w-reviewer-finds)

The top-tier RTX 5090 Laptop GPU, by contrast, is fixed at 24 GB of GDDR7 over a 256-bit bus, for 896 GB/s of bandwidth. [Source](https://www.notebookcheck.net/Nvidia-GeForce-RTX-5090-Laptop-Benchmarks-and-Specs.934947.0.html) That is comparable per-byte bandwidth to the M4 Max, but with roughly a fifth of the addressable capacity of even a mid-configuration Mac Studio, and no upgrade path once the laptop is purchased — VRAM is soldered, unlike a Mac's unified memory, which is also soldered but offered in much larger configurations at the point of purchase. Applying §11.1's sizing equation, 24 GB caps full-GPU-offload models at roughly the Q4-quantised 30B-parameter class before spilling into the CPU-offload (§4.4) or multi-GPU-split (§3.4) paths — neither of which a single laptop dGPU can use.

### 12.2 The Apple Silicon Software Stack: Metal, MLX, and llama.cpp

On macOS, llama.cpp's GPU backend is `ggml_backend_metal`, not Vulkan or CUDA — it translates GGML's compute graph into Metal compute shaders, a different code path from the `ggml_backend_vk` architecture covered in §2.2 of this chapter. Apple's own `MLX` framework is a separate project, built from scratch around the unified-memory architecture rather than adapted from a CUDA-shaped compute model; MLX and llama.cpp's Metal backend share no code. Independent benchmarks report MLX 20–87% faster token generation than llama.cpp/Metal for models under roughly 14B parameters, with the gap closing above ~27B parameters as generation becomes purely memory-bandwidth-bound (the same roofline argument as §10.5) rather than kernel-efficiency-bound. [Source](https://arxiv.org/html/2601.19139v1) Apple positioned MLX as its preferred LLM inference framework across multiple WWDC 2025 sessions, while llama.cpp/Metal remains the better choice for cross-platform portability and for models that only barely fit in available memory, where its more mature memory-mapping and partial-offload paths (§4.2–§4.4) have an edge.

None of this Metal/MLX stack is available on Linux; it is included here only because it is the software a MacBook actually runs by default, in contrast to this chapter's Vulkan/CUDA/ROCm/OpenVINO focus.

### 12.3 Discrete-GPU Gaming Laptops

A gaming laptop's RTX 50-series mobile GPU runs the same CUDA, Vulkan, and (via ROCm-adjacent tooling on the AMD side) driver stacks documented throughout this chapter — Ollama's GPU dispatch (§5), llama.cpp's Vulkan or CUDA backend (§2–§3), and ONNX Runtime's CUDA execution provider (§6.2) all work unmodified on a laptop dGPU exactly as they do on a desktop card, modulo the laptop's lower power/thermal envelope (§10.4) and its hard VRAM ceiling. The RTX 5090 Laptop's 896 GB/s is close to, but below, the RTX 4090 desktop's 1008 GB/s used as the §10.5 roofline reference — laptop cards trade a modest amount of bandwidth and a fixed 24 GB (or 16 GB, on the RTX 5080 Laptop) VRAM budget for portability. [Source](https://www.tomshardware.com/pc-components/gpus/nvidia-introduces-rtx-5090-rtx-5080-and-rtx-5070-laptop-gpus-rtx-50-blackwell-goes-mobile-with-up-to-24gb-of-gddr7-memory) Within that VRAM budget, however, a gaming laptop's dedicated GDDR7 delivers more consistent, driver-managed throughput than an Apple Silicon machine allocating a large unified-memory model, because it never contends with concurrent CPU-side memory traffic from the rest of the OS.

### 12.4 Running Linux on Apple Silicon: Asahi Linux Status

Because this chapter is scoped to Linux GPUs, a MacBook running macOS is out of scope by definition — but Asahi Linux, the reverse-engineered open-source driver stack for Apple Silicon, brings Apple GPUs into the same Linux DRM/Vulkan world this book otherwise covers. As of the project's 2025–2026 progress reports, Asahi's "Honeykrisp" Vulkan driver has reached full **Vulkan 1.3 conformance** on Apple GPUs (achieved in 2024, without relying on the Vulkan Portability waivers some non-native implementations require), alongside OpenGL 4.6 and OpenCL 3.0 support; more recent 2025–2026 progress reports (Linux 6.19 through 7.2) have focused on upstream kernel work and power-management infrastructure rather than further GPU driver feature milestones. [Source](https://asahilinux.org/blog/)

Because llama.cpp's Vulkan backend (§2.2) targets any Vulkan 1.2+ device, an Apple Silicon machine running Asahi Linux is, in principle, a valid target for the exact same `ggml_backend_vk` code path used for AMD and Intel GPUs elsewhere in this chapter — Vulkan compute shaders, not Metal or MLX, would be doing the work. **Note: needs verification** — this research did not turn up published benchmarks or user reports of llama.cpp or Ollama actually running LLM inference through Asahi's Vulkan driver; treat Apple-Silicon-under-Linux LLM inference as an untested extrapolation from Honeykrisp's general Vulkan 1.3 conformance rather than a proven deployment path, and expect the CPU-side memory bandwidth and unified-memory sharing behaviour under Linux's memory manager to differ from macOS's tuned Metal driver stack.

---

## 14. Production Serving Engines: vLLM and SGLang

Everything from §1–§13 covers single-user or small-scale local inference: one process, one or a few GPUs, one request at a time or a small handful. Production serving — many concurrent users, an OpenAI-compatible HTTP API, and throughput measured in aggregate tokens/second rather than single-stream latency — is a different engineering problem, and it is dominated by two open-source engines: **vLLM** and **SGLang**. Both build on the KV cache and PagedAttention concepts already introduced in §9, but add continuous batching schedulers, distributed execution, and quantized-serving support on top.

### 14.1 vLLM: PagedAttention as a Serving Engine

vLLM originated at UC Berkeley's Sky Computing Lab and is now a community-maintained project (2,000+ contributors) under the Apache 2.0 license. [Source](https://github.com/vllm-project/vllm) Where §9.2 described PagedAttention as a KV cache memory-management technique, vLLM the *engine* wraps that technique in a full serving stack: an `LLMEngine` that tokenizes requests, a **scheduler** that performs iteration-level continuous batching (admitting new requests into a running batch as soon as GPU slots free up, rather than only at batch boundaries), and GPU worker processes that execute the model. In the current V1 architecture, the API server, the engine-core process (scheduler + KV cache manager), and the GPU workers run as separate processes communicating over IPC. [Source](https://docs.vllm.ai/en/latest/design/arch_overview/)

**Installation on Linux.** For NVIDIA GPUs, vLLM ships pre-built CUDA wheels:

```bash
uv pip install vllm --torch-backend=auto
```

which resolves to a CUDA 12.9 build by default (extra wheel indices exist for CUDA 12.8/13.0; Blackwell B200/GB200 requires CUDA ≥12.8). For AMD GPUs, a dedicated ROCm wheel index is used instead:

```bash
uv pip install vllm --extra-index-url https://wheels.vllm.ai/rocm/ --upgrade
```

which requires ROCm ≥6.3 (MI350-series needs ROCm ≥7.0) and currently mandates Python 3.12; a `vllm/vllm-openai-rocm` Docker image is offered as an alternative to a bare-metal install. Intel XPU has its own nightly wheel index. [Source](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/)

**Launching an OpenAI-compatible server.** The current, documented entry point is the `vllm serve` subcommand — the older `python -m vllm.entrypoints.openai.api_server` invocation is not referenced in current docs:

```bash
vllm serve NousResearch/Meta-Llama-3-8B-Instruct \
    --gpu-memory-utilization 0.9 \
    --max-model-len 8192 \
    --dtype auto \
    --quantization awq
```

`--gpu-memory-utilization` (default 0.9) is the fraction of GPU memory vLLM's executor reserves for weights, activations, and the KV cache pool — the same VRAM-budgeting concern §11.1's sizing equation addresses, but here vLLM manages the allocation internally rather than the operator computing it by hand. `--max-model-len` caps the maximum sequence length (context + generation) the KV cache pool is sized for; `--dtype auto` selects FP16 or BF16 based on the checkpoint; `--quantization` loads a pre-quantized checkpoint format (§17 covers the formats vLLM accepts here in depth). [Source](https://docs.vllm.ai/en/latest/serving/online_serving/) [Source](https://docs.vllm.ai/en/v0.10.2/configuration/engine_args.html)

**Multi-GPU parallelism.** vLLM implements tensor parallelism following Megatron-LM's approach (splitting attention and MLP weight matrices across GPUs) and combines it with pipeline parallelism for multi-node scale-out. A common convention is tensor-parallel size equal to the GPU count within a node, pipeline-parallel size equal to the node count:

```bash
vllm serve gpt2 --tensor-parallel-size 4 --pipeline-parallel-size 2
```

`--distributed-executor-backend` selects `mp` (multiprocessing, the single-node default) or `ray` (the multi-node default); multi-node runs additionally take `--nnodes`, `--node-rank`, `--master-addr`. For large mixture-of-experts models, vLLM also supports combining data-parallel attention with expert/tensor-parallel MoE layers. [Source](https://docs.vllm.ai/en/v0.9.1/serving/distributed_serving.html) [Source](https://docs.vllm.ai/en/latest/serving/parallelism_scaling/)

As of this writing, the current release is v0.28.0 (2026-08-26). [Source](https://github.com/vllm-project/vllm/releases/tag/v0.28.0)

### 14.2 SGLang: RadixAttention and Structured Generation

SGLang ("Structured Generation Language") is a serving engine and Python-embedded DSL from LMSYS Org (the UC Berkeley-affiliated group behind Chatbot Arena), introduced in a January 2024 blog post and formalized in a NeurIPS 2024 paper. [Source](https://arxiv.org/abs/2312.07104) [Source](https://www.lmsys.org/blog/2024-01-17-sglang/) It is Apache 2.0 licensed and, as of this writing, at PyPI version 0.5.18. [Source](https://pypi.org/project/sglang/)

Where vLLM's PagedAttention indexes the KV cache by fixed-size blocks per sequence, SGLang's core mechanism, **RadixAttention**, indexes the KV cache for *all* sequences the server has ever processed as a single radix tree keyed on token sequences. This lets the scheduler automatically detect and reuse KV cache for any two requests sharing a token prefix — not just requests explicitly grouped by the application — evicting via an LRU policy over tree leaves when memory pressure requires it. SGLang's own documentation describes RadixAttention as compatible with, rather than a replacement for, block-based paged memory management. [Source](https://www.lmsys.org/blog/2024-01-17-sglang/) A v0.4 release (December 2024) added a cache-aware load balancer (reported up to 1.9× throughput and 3.8× higher cache-hit rate in SGLang's own benchmarks) and a "zero-overhead" batch scheduler. [Source](https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/)

The frontend DSL layers structured-generation primitives — `gen` (invoke generation into a variable), `fork` (parallel prompt branches), `choices` (constrained decoding over a fixed option set) — on top of the serving engine, targeting workloads like multi-step agent pipelines or JSON-schema-constrained output where naive prompting wastes tokens re-sending shared context. [Source](https://arxiv.org/pdf/2312.07104)

Install and launch on Linux:

```bash
pip install uv && uv pip install --prerelease=allow sglang
python -m sglang.launch_server \
    --model-path meta-llama/Meta-Llama-3-8B-Instruct \
    --tp-size 1 \
    --mem-fraction-static 0.7 \
    --quantization fp8
```

`--mem-fraction-static` plays the same role as vLLM's `--gpu-memory-utilization`; `--tp-size` is SGLang's tensor-parallel-degree flag; `--quantization` accepts over 20 values including `awq`, `fp8`, `gptq`, `marlin`, `bitsandbytes`, and `gguf`. [Source](https://docs.sglang.io/docs/advanced_features/server_arguments) NVIDIA GPUs are the primary target; AMD Instinct (MI300X, MI325X, MI355X) is supported via ROCm, distributed as a prebuilt Docker image (`rocm/sgl-dev:...-rocm7.14`). [Source](https://github.com/sgl-project/sglang/blob/main/docs/platforms/amd_gpu.md)

SGLang's own blog posts report throughput advantages over vLLM ranging from roughly 3× (structured generation workloads, Jan 2024) to 3–7× (DeepSeek MLA-optimized models, Sep 2024) — these figures are self-reported by the SGLang project against specific vLLM versions and hardware configurations, not independently audited, and should be read as directional rather than a fixed multiplier. [Source](https://www.lmsys.org/blog/2024-07-25-sglang-llama3/) [Source](https://www.lmsys.org/blog/2024-09-04-sglang-v0-3/)

---

## 15. Disaggregated Prefill-Decode Serving

§9.1 characterized prefill as compute-bound (large batched matrix multiplies over the full prompt) and decode as memory-bandwidth-bound (one token at a time, reading the entire KV cache and weight set per step). When both phases run on the same GPU, they interfere: a batch of large prefill jobs delays the small, latency-sensitive decode steps queued behind them — a form of head-of-line blocking that couples two workloads with fundamentally different latency objectives (prefill optimizes Time-To-First-Token; decode optimizes Time-Per-Output-Token). The DistServe paper (OSDI 2024) frames this explicitly and reports that separating the two phases onto independently-scaled GPU pools — with the KV cache computed during prefill transferred over the network to the decode pool — lets a cluster serve up to 7.4× more requests within the same latency SLO. [Source](https://arxiv.org/abs/2401.09670)

This pattern, disaggregated (or "P/D-disaggregated") serving, has since been implemented by several projects:

- **vLLM** exposes an explicitly-labeled **experimental** `--kv-transfer-config` flag (JSON payload, e.g. `{"kv_connector":"NixlConnector","kv_role":"kv_both"}`) built on a pluggable `Connector` abstraction. Documented connector backends include `NixlConnector` (described as "the primary connector for production disaggregated prefilling," using NVIDIA's NIXL transfer library), `MooncakeConnector`, `LMCacheConnectorV1`, and a ROCm-only `MoRIIOConnector`. vLLM's own docs caution that disaggregated prefilling targets *latency* goodput, not raw throughput, and that the feature "does not improve throughput" on its own. [Source](https://docs.vllm.ai/en/latest/features/disagg_prefill/)

- **NVIDIA Dynamo** (https://github.com/ai-dynamo/dynamo, Apache 2.0) is an orchestration layer sitting above existing engines — SGLang, TensorRT-LLM, and vLLM — rather than a replacement for any of them, adding disaggregated-serving orchestration, KV-aware request routing, and autoscaling. Its KV-transfer layer is **NIXL** (NVIDIA Inference Xfer Library, github.com/ai-dynamo/nixl, Apache 2.0), a modular transport abstracting GPU/CPU memory and storage with UCX and GPUDirect Storage backend plugins. Dynamo's own README reports up to 7× higher per-GPU throughput (DeepSeek-R1 on GB200 NVL72) and 2× faster time-to-first-token via KV-aware routing — self-reported figures, not independently benchmarked. [Source](https://github.com/ai-dynamo/dynamo)

- **Mooncake** (https://github.com/kvcache-ai/Mooncake, Apache 2.0, FAST 2025 Best Paper) is the KV-cache-centric disaggregated architecture backing Moonshot AI's production Kimi service. It separates prefill and decode clusters and pools DRAM/SSD/RDMA resources across the cluster into a shared KV cache store ("Mooncake Store"), moved by a **Transfer Engine** that aggregates multiple NICs and supports RDMA, TCP, and NVMe-oF. The project reports serving 75% more requests under the same SLOs in production, and paper-level results of 59–498% effective capacity increase over non-disaggregated baselines on real traces. [Source](https://arxiv.org/abs/2407.00079) [Source](https://github.com/kvcache-ai/Mooncake)

All three vLLM, Dynamo, and Mooncake connectors are Apache 2.0 licensed. As of this writing, vLLM's own implementation remains explicitly marked experimental in its documentation — disaggregated serving is a technique worth understanding for its architectural implications (it decouples GPU pool sizing for prefill from decode, and pushes the KV cache transfer problem onto the network/RDMA fabric rather than local VRAM), but it is not yet the default deployment path for a single-node local-inference setup of the kind §1–§13 describe.

---

## 16. Speculative Decoding

Speculative decoding attacks the memory-bandwidth bottleneck described in §10.5 from a different angle than quantization: instead of shrinking the weights read per token, it amortizes each expensive full-model forward pass over *several* output tokens. A small, fast **draft model** proposes several candidate tokens autoregressively; the large **target model** then verifies all of them in a single batched forward pass, and a rejection-sampling step accepts the longest matching prefix (resampling at the first divergence). Because rejection sampling is constructed to reproduce the target model's exact output distribution, this is a lossless speedup, not an approximation — the two foundational papers (Leviathan et al., Google, and Chen et al., DeepMind, both 2023) independently derived the same guarantee and reported 2–3× wall-clock speedups with byte-identical outputs to running the target model alone. [Source](https://arxiv.org/abs/2211.17192) [Source](https://arxiv.org/abs/2302.01318)

Three lines of follow-up work changed how the "draft" step is produced:

- **Medusa** attaches multiple extra decoding heads directly to the target model, each predicting a token several positions ahead in a single forward pass; candidates are combined into a tree and verified with tree-based attention in one pass, avoiding a separate draft model entirely. Reported speedups: 2.2× (heads only, base model frozen) to 3.6× (heads + fine-tuned base model). [Source](https://arxiv.org/abs/2401.10774) [Source](https://github.com/FasterDecoding/Medusa)
- **EAGLE** (and its successors EAGLE-2 and EAGLE-3) drafts in the target model's own *feature space* — the second-to-top-layer hidden states — rather than token space, on the premise that a separate small model's token-level predictions discard information the target model's own representations retain. EAGLE-2 adds confidence-based dynamic draft trees; EAGLE-3 fuses features from multiple layers using a training-time-test objective. EAGLE-1 reports 2.7–3.5× latency reduction on LLaMA2-Chat-70B. [Source](https://arxiv.org/abs/2401.15077) [Source](https://arxiv.org/pdf/2406.16858) [Source](https://arxiv.org/pdf/2503.01840) [Source](https://github.com/SafeAILab/EAGLE)
- **Lookahead decoding** drops the draft model concept entirely: it recasts autoregressive decoding as a system of nonlinear equations solved via a parallel Jacobi-iteration variant, extracting candidate n-grams from the trajectory of Jacobi iterations and verifying them in parallel — no auxiliary model or training required. [Source](https://arxiv.org/pdf/2402.02057) [Source](https://github.com/hao-ai-lab/LookaheadDecoding)

All three serving engines and llama.cpp expose speculative decoding as a configuration option rather than an architectural rewrite:

- **vLLM**: `--speculative-config`, a JSON payload selecting a `method` (`ngram`, `eagle`, `eagle3`, `medusa`, `draft_model`, among others) and `num_speculative_tokens`, e.g. `vllm serve <target> --speculative-config '{"method":"eagle3","model":"<draft>","num_speculative_tokens":4}'`. [Source](https://docs.vllm.ai/en/latest/features/speculative_decoding/)
- **SGLang**: `--speculative-algorithm {EAGLE,EAGLE3,STANDALONE,NGRAM}` plus `--speculative-draft-model-path`, `--speculative-num-steps`, `--speculative-eagle-topk`. [Source](https://docs.sglang.io/advanced_features/speculative_decoding.html)
- **llama.cpp**: `-md`/`--model-draft <FNAME>` supplies a draft-model GGUF file alongside the target model; n-gram-based draft-free modes are also available. Flag surfaces here have changed across releases more than the other two projects — pin the exact commit/tag before citing a specific flag name in production documentation. [Source](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md)

Reported speedups across all of the above (2–3.6×) are acceptance-rate-dependent: workloads with low output entropy (code completion, greedy/low-temperature decoding, repetitive structured output) see the draft model's guesses accepted far more often than open-ended creative generation, so the practical multiplier on any given deployment should be benchmarked against its actual traffic rather than assumed from a paper's number.

---

## 17. Quantization for Serving Engines: GPTQ, AWQ, bitsandbytes, and FP8

§1.3 covered GGUF's block-quantized K-quant formats (Q4_K_M and friends) as used by llama.cpp and Ollama. vLLM and SGLang instead center on a different family of post-training quantization methods, developed independently in the Hugging Face `transformers` ecosystem and adopted directly into the serving engines' `--quantization` flag.

**GPTQ** quantizes a model layer-by-layer, one-shot (no retraining), updating each not-yet-quantized weight to compensate for the error introduced by quantizing the weights processed just before it, using approximate second-order (Hessian) curvature information descended from the Optimal Brain Quantization line of work. The original paper quantized a 175B-parameter model to 3–4 bits/weight in about four GPU-hours with negligible accuracy loss. [Source](https://arxiv.org/abs/2210.17323) The `AutoGPTQ` library that popularized this in `transformers` has been superseded by **GPTQModel**, which adds hardware-accelerated inference kernels for `transformers`, vLLM, and SGLang across CUDA, ROCm, and other backends. [Source](https://github.com/ModelCloud/GPTQModel)

**AWQ** (Activation-aware Weight Quantization) starts from the observation that not all weight channels matter equally to output quality — a small fraction (roughly 1%) of "salient" channels, identified by observing *activation* magnitudes rather than weight magnitudes, dominate accuracy loss under quantization. AWQ searches for per-channel scaling factors that protect those salient channels before quantizing the rest, without backpropagation or calibration-set reconstruction (which reduces overfitting to whatever calibration data happens to be used). The paper won the MLSys 2024 Best Paper award. [Source](https://arxiv.org/abs/2306.00978) The reference `AutoAWQ` implementation is now archived; the project's own README directs users to vLLM, which adopted AWQ's kernels directly, and to LLM Compressor for continued maintenance. [Source](https://github.com/casper-hansen/AutoAWQ)

**bitsandbytes** implements two distinct schemes: **LLM.int8()**, a mixed-precision scheme that isolates the small number of "outlier" feature dimensions that emerge once a model exceeds roughly 6.7B parameters into a separate 16-bit matrix multiply while keeping the remaining >99.9% of values in 8-bit, and **NF4** (4-bit NormalFloat), an information-theoretically-optimal 4-bit encoding for normally-distributed weights (as opposed to a uniform-spacing INT4 grid), introduced as part of QLoRA and paired there with double quantization of the quantization constants themselves. [Source](https://arxiv.org/abs/2208.07339) [Source](https://arxiv.org/abs/2305.14314)

**FP8** — specifically the E4M3 (1 sign, 4 exponent, 3 mantissa bits; range ±448, favors precision) and E5M2 (1 sign, 5 exponent, 2 mantissa bits; wider dynamic range) formats — is a hardware-native quantization scheme rather than a software compression trick: it requires NVIDIA Ada Lovelace, Hopper, or Blackwell Tensor Cores (compute capability ≥8.9). On Blackwell, block-scaled MXFP8 is the default recipe; on Hopper, blockwise E4M3 (1×128 activation tiles, 128×128 weight tiles) is recommended. Unlike GPTQ/AWQ, an FP8 checkpoint can often skip calibration entirely — vLLM's `FP8_DYNAMIC` scheme statically quantizes weights per-channel with round-to-nearest and computes activation scales dynamically per-token at runtime, with zero calibration data required. [Source](https://docs.vllm.ai/en/latest/features/quantization/llm_compressor/fp8/)

vLLM's `--quantization` flag accepts, among others: `awq`, `gptq`, `fp8`, `bitsandbytes`, `gguf`, `llm-compressor` (which itself covers FP8 W8A8, INT4 W4A16, and INT8 W4A8/W8A8 recipes), `modelopt` (NVIDIA Model Optimizer), and `quark` (AMD). Hardware compatibility varies per scheme — Turing-generation GPUs (compute capability 7.5) support AWQ/GPTQ but not the faster Marlin kernel variants or FP8, while llm-compressor's FP8 W8A8 path requires Ada Lovelace or newer. [Source](https://docs.vllm.ai/en/latest/features/quantization/) In practice, kernel implementation (e.g., whether the Marlin GEMM kernel backs a given format) often affects throughput as much as the quantization algorithm choice itself; FP8 is generally described as the closest to BF16-equivalent quality with the least accuracy risk of the options here, while 4-bit GPTQ/AWQ push memory savings further at somewhat greater risk, a tradeoff comparable in shape to the K-quant bit-depth tradeoffs already covered for GGUF in §10.2.

---

## 18. NVIDIA NGC Catalog and NIM Microservices

**NGC** (catalog.ngc.nvidia.com) is NVIDIA's registry of GPU-optimized, security-scanned containers, pretrained models, and Helm charts — the same registry that supplies, for example, the `nvcr.io/nvidia/pytorch` training containers referenced elsewhere in this book. [Source](https://docs.nvidia.com/ngc/gpu-cloud/ngc-catalog-user-guide/index.html) Authenticating against it uses an NGC API key generated from the NGC account portal, presented to Docker as a password with the literal username `$oauthtoken`:

```bash
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
docker pull nvcr.io/nvidia/pytorch:20.03-py3
```

**NIM** (NVIDIA Inference Microservices, build.nvidia.com) is a narrower, LLM-specific layer on top of NGC: pre-built containers that wrap a specific model with an OpenAI-compatible API, chosen and optimized per model/GPU combination using TensorRT-LLM, vLLM, or SGLang as the underlying engine. [Source](https://developer.nvidia.com/nim) NVIDIA's own documentation describes the LLM NIM architecture as "an enterprise orchestration layer for vLLM": on first launch, it inspects the local GPU against a support matrix and, for supported GPU/model pairs, downloads a pre-compiled TensorRT engine and serves via TensorRT-LLM; on unsupported hardware it falls back to running the model through vLLM directly. [Source](https://docs.nvidia.com/nim/large-language-models/latest/about-nim-llm/overview.html) This makes NIM, in effect, a curated and pre-optimized deployment path over the same vLLM engine already covered in §14.1 — the value proposition is skipping the engine-flag tuning and quantization-format selection §14 and §17 walk through by hand, at the cost of using NVIDIA's chosen defaults and a container pull rather than a pip install.

A concrete local deployment (Llama 3-8B-Instruct, from NVIDIA's own getting-started guide):

```bash
export NGC_API_KEY=<value>
export LOCAL_NIM_CACHE=~/.cache/nim
mkdir -p "$LOCAL_NIM_CACHE"

docker run -it --rm --name=llama3-8b-instruct \
  --runtime=nvidia --gpus all --shm-size=16GB \
  -e NGC_API_KEY=$NGC_API_KEY \
  -v "$LOCAL_NIM_CACHE:/opt/nim/.cache" \
  -u $(id -u) -p 8000:8000 \
  nvcr.io/nim/meta/llama3-8b-instruct:1.2.1
```

which exposes `http://0.0.0.0:8000/v1/chat/completions` once the container logs "Uvicorn running." [Source](https://docs.nvidia.com/nim/large-language-models/1.3.0/getting-started.html)

Downloadable NIM containers are free to pull and run under an NVIDIA Developer Program account for development, testing, and research, capped at up to two nodes or 16 GPUs; production deployment requires an NVIDIA AI Enterprise license (a 90-day free trial is offered). [Source](https://developer.nvidia.com/blog/access-to-nvidia-nim-now-available-free-to-developer-program-members/) The catalog spans 100+ models at build.nvidia.com, including the Llama 3.x family, NVIDIA's own Nemotron line, Mistral/Mixtral, and entries from DeepSeek, Qwen, and Gemma; verify the exact image tag for a given model on its build.nvidia.com "Deploy" page before pulling, since tags update over time and the example above (`1.2.1`) will not remain current. [Source](https://developer.nvidia.com/nim)

---

## 19. Docker Model Runner

Docker's own answer to "run an LLM locally" ships inside Docker Desktop and, since general availability, Docker Engine on Linux: **Docker Model Runner** (DMR), introduced in beta in Docker Desktop 4.40 (April 2025) and reaching general availability on September 18, 2025. [Source](https://www.docker.com/blog/introducing-docker-model-runner/) [Source](https://www.docker.com/blog/announcing-docker-model-runner-ga/)

Architecturally, DMR is a thin wrapper: its own open-source repository confirms it vendors the Linux `llama-server` binary from the official `ghcr.io/ggml-org/llama.cpp` container image as its default backend (with vLLM and SGLang also listed as available backends), and runs the inference process *natively on the host* rather than inside a container — this is a deliberate design choice to avoid the GPU-passthrough overhead of running a full container/VM stack on macOS. [Source](https://github.com/docker/model-runner) This is the key difference from "Ollama running inside a Docker container" (the pattern documented in Chapter 55): DMR is Docker's own daemon-integrated runtime with `docker model` as a first-class CLI verb group, not an arbitrary containerized application.

**Linux support** is native via Docker CE (Community Edition), with CPU, NVIDIA CUDA, AMD ROCm, and Vulkan backends documented:

```bash
sudo apt-get install docker-model-plugin   # Debian/Ubuntu
docker model install-runner                # Docker CE only: starts the runner service
```

[Source](https://docs.docker.com/ai/model-runner/) [Source](https://docs.docker.com/ai/model-runner/get-started/)

Models are distributed as standard **OCI Artifacts** on Docker Hub under the `ai/` namespace (e.g., `ai/llama3.2:3B-Q4_K_M`), or pulled directly from Hugging Face by GGUF reference (`hf.co/...`), auto-packaged into an OCI artifact on the fly:

```bash
docker model pull ai/llama3.2:3B-Q4_K_M
docker model run ai/llama3.2:3B-Q4_K_M
docker model list
docker model rm ai/llama3.2:3B-Q4_K_M
```

[Source](https://docs.docker.com/reference/cli/docker/model/) [Source](https://hub.docker.com/r/ai/llama3.2)

A running model is reachable through an OpenAI-compatible endpoint — `http://model-runner.docker.internal/engines/v1` from inside another container, or `http://localhost:12434/engines/v1` from the host once TCP access is enabled — serving the same `/chat/completions`, `/completions`, `/models`, and `/embeddings` routes used throughout this chapter's other engines. [Source](https://docs.docker.com/ai/model-runner/api-reference/) The `docker/model-runner` core is Apache 2.0 licensed and free to use, bundled at no additional cost with Docker Desktop and Docker CE. [Source](https://github.com/docker/model-runner)

---

## 20. Kubernetes-Native Agent Orchestration: kagent

kagent (kagent.dev, github.com/kagent-dev/kagent) is worth a brief mention here for what it is *not*: it is not an inference engine, and it does not belong alongside vLLM, SGLang, or Ollama as a way to run model weights on a GPU. It is a Kubernetes-native framework, originally built at Solo.io and now a CNCF Sandbox project, for declaring AI *agents* — not models — as Kubernetes custom resources (`Agent`, `ModelConfig`, `ToolServer` CRDs under `kagent.dev/v1alpha2`), reconciled by a controller and executed through a runtime built on Google's Agent Development Kit, with tool connectivity via MCP. [Source](https://github.com/kagent-dev/kagent) [Source](https://kagent.dev/docs/kagent/concepts/architecture)

The relevant distinction for this chapter: a kagent `Agent` resource does not host or serve a model itself. Its `ModelConfig` resource points at an *already-running* inference endpoint — kagent's documented provider list includes the major cloud APIs alongside Ollama (via an `ollama.host` field addressing an in-cluster or external Ollama server) and a generic "bring your own OpenAI-compatible model" option, which covers pointing kagent at a self-hosted vLLM or SGLang deployment from §14 even without first-class provider support (a dedicated vLLM `ModelConfig` type was, as of this research, still an open feature request). [Source](https://kagent.dev/docs/kagent/supported-providers) In other words: kagent is the orchestration layer that *calls* the serving engines this chapter covers, deployed via Helm (`helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds`, followed by the main `kagent` chart) onto a cluster that already has an inference backend running somewhere. It is Apache 2.0 licensed and under active development. [Source](https://kagent.dev/docs/kagent/introduction/installation) A reader building a Kubernetes-hosted agent system on top of a self-hosted LLM deployed per this chapter's §14–§19 would reach for kagent at the orchestration layer, one level above everything else discussed here.

---

## 21. Running Hugging Face Models Locally via the CLI

Every model reference in this chapter — a GGUF file for llama.cpp/Ollama, a `transformers`-format checkpoint for vLLM/SGLang/ONNX Runtime, an AWQ- or GPTQ-quantized repo — most commonly starts life as a Hugging Face Hub repository, and the `huggingface_hub` package's CLI is the standard way to pull one down onto a Linux GPU box outside of a specific inference tool's built-in fetcher.

**The CLI was renamed.** As of `huggingface_hub` v0.34.0 (July 2025), the CLI is `hf`, not the older `huggingface-cli` — the command grammar changed to `hf <resource> <action>` (`hf auth login`, `hf download`, `hf cache`, `hf repo`, `hf jobs`), with `download`/`upload` kept at the top level as the most-used commands. `huggingface-cli` continued to work with deprecation warnings for a transition period and was removed outright in `huggingface_hub` v1.0. [Source](https://www.huggingface.co/blog/hf-cli) [Source](https://huggingface.co/docs/huggingface_hub/en/concepts/migration)

```bash
pip install -U huggingface_hub
hf auth login                                          # or: hf auth login --token $HF_TOKEN
hf download meta-llama/Llama-2-7b-chat-hf --local-dir models/llama-2-7b-chat/
```

`--include`/`--exclude` accept glob patterns, the mechanism for pulling a single GGUF quantization out of a repository that hosts several:

```bash
hf download Qwen/Qwen2-0.5B-Instruct-GGUF --include "*q5_k_m.gguf" --local-dir .
```

[Source](https://raw.githubusercontent.com/huggingface/huggingface_hub/main/docs/source/en/guides/download.md)

**Cache layout.** Without `--local-dir`, downloads land in the shared cache at `~/.cache/huggingface/hub` (relocatable via the `HF_HOME` environment variable, or more specifically `HF_HUB_CACHE`). Inside, each repo gets a `models--<namespace>--<name>` directory containing `blobs/` (file content, named by hash), `snapshots/` (one directory per revision, populated with symlinks into `blobs/`), and `refs/` (branch/tag → commit mapping) — the symlink indirection is what lets two revisions that happen to share an unchanged file (e.g., a tokenizer config across point releases) reference the same blob on disk rather than duplicating it, the same deduplication goal §4.2's `mmap`-based weight loading serves for a single file. [Source](https://raw.githubusercontent.com/huggingface/huggingface_hub/main/docs/source/en/guides/manage-cache.md)

**Transfer acceleration.** The historical `hf_transfer` Rust accelerator (enabled via `HF_HUB_ENABLE_HF_TRANSFER=1`) is now deprecated: current `huggingface_hub` routes all Hub transfers through the newer **Xet** storage backend instead, and the equivalent tuning knob is `HF_XET_HIGH_PERFORMANCE=1`. [Source](https://raw.githubusercontent.com/huggingface/huggingface_hub/main/docs/source/en/package_reference/environment_variables.md)

Programmatically, the same operations are `hf_hub_download(repo_id, filename, ...)` for a single file and `snapshot_download(repo_id, allow_patterns=..., ignore_patterns=..., ...)` for a filtered full-repo pull — the Python equivalents of `hf download`'s single-file form and its `--include`/`--exclude` flags, useful when a download needs to be triggered from inside a provisioning script rather than a shell one-liner. [Source](https://raw.githubusercontent.com/huggingface/huggingface_hub/main/docs/source/en/guides/download.md)

---

## 22. Fine-Tuning Acceleration with Unsloth

Every runtime covered so far in this chapter — llama.cpp, Ollama, ONNX Runtime, vLLM, SGLang — assumes a finished set of weights and optimizes *serving* them. Unsloth (github.com/unslothai/unsloth) sits one step earlier in the pipeline: it accelerates the *fine-tuning* step that produces a custom model in the first place, and it is included here because its output feeds directly back into the formats and runtimes already covered — a QLoRA fine-tune produced by Unsloth is typically exported straight to GGUF for llama.cpp/Ollama (§1–§5) or merged and served through vLLM (§14.1), making Unsloth the practical on-ramp from "fine-tune a model on a local GPU" to "serve it locally" using the tools this chapter already documents.

**What it does.** Unsloth reimplements the LoRA/QLoRA fine-tuning path — attention and MLP kernels, backward-pass gradient computation, and 4-bit `bitsandbytes` quantization handling (the same NF4 scheme covered in §17) — using hand-written Triton kernels and a manually-derived backpropagation graph in place of PyTorch autograd's generic (and more memory-hungry) computation graph, without approximating the math: training is claimed to be lossless relative to standard LoRA. The project reports this yields training roughly 2× faster with about 70% less VRAM, with per-model variation (e.g., its Gemma and gpt-oss support pages cite different multipliers such as 1.5× faster / 50% less memory for some models). These figures are self-reported by the Unsloth project on its own documentation and blog, not independently benchmarked, and should be treated the same way the SGLang-vs-vLLM throughput claims in §14.2 are: directionally credible, not a fixed guarantee for an arbitrary model/hardware pair. [Source](https://github.com/unslothai/unsloth) [Source](https://unsloth.ai/docs)

**Hardware support.** Unsloth's primary target has always been NVIDIA CUDA GPUs (from Turing to Blackwell, including RTX 50-series). Official AMD ROCm support (ROCm ≥6.0) was added as a dedicated, documented install path, alongside preliminary Intel GPU and CPU/Vulkan-backend support — a meaningfully more recent and less battle-tested addition than the NVIDIA path, worth flagging as such rather than presenting all three as equally mature. [Source](https://unsloth.ai/docs/basics/amd) [Source](https://www.amd.com/en/developer/resources/technical-articles/2026/train-and-run-models-on-amd-gpus-with-unsloth.html)

**Installation and license.** The core library is Apache 2.0 licensed and installs via pip; a separately-licensed (AGPL-3.0) desktop application and web-based "Unsloth Studio" UI wrap the same core for users who prefer a GUI over a notebook/script workflow — a licensing split worth noting since the two components are not interchangeable from a redistribution standpoint. [Source](https://pypi.org/project/unsloth/)

```bash
pip install unsloth   # NVIDIA CUDA
# AMD ROCm (≥6.0) — separate PyTorch wheel index, then unsloth on top:
uv pip install "torch>=2.4,<2.11.0" "torchvision<0.26.0" "torchaudio<2.11.0" \
    --index-url https://download.pytorch.org/whl/rocm7.1 --upgrade --force-reinstall
```

[Source](https://unsloth.ai/docs/get-started/install/amd)

**QLoRA fine-tune, then export for the runtimes already in this chapter.** A minimal fine-tuning loop loads a base model 4-bit-quantized and attaches LoRA adapters via `FastLanguageModel`:

```python
from unsloth import FastLanguageModel

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "meta-llama/Meta-Llama-3-8B",
    max_seq_length = 2048,
    load_in_4bit = True,
)
model = FastLanguageModel.get_peft_model(
    model, r = 16,
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                       "gate_proj", "up_proj", "down_proj"],
    lora_alpha = 16, lora_dropout = 0, bias = "none",
    use_gradient_checkpointing = "unsloth",
)
# ... train with a standard Hugging Face Trainer/TRL SFTTrainer loop ...
```

[Source](https://unsloth.ai/docs/basics/continued-pretraining)

Once trained, Unsloth exports directly to the two deployment paths already documented in this chapter. For llama.cpp/Ollama, `save_pretrained_gguf` builds llama.cpp internally and converts in one call, using the same K-quant naming scheme from §1.3:

```python
model.save_pretrained_gguf("out_dir", tokenizer, quantization_method = "q4_k_m")
```

For vLLM (§14.1) or SGLang (§14.2), the LoRA adapter is merged back into the base weights and saved as a standard 16-bit checkpoint that either engine's `--model` flag can load directly:

```python
model.save_pretrained_merged("out_dir_merged", tokenizer, save_method = "merged_16bit")
```

[Source](https://unsloth.ai/docs/basics/inference-and-deployment/saving-to-gguf) [Source](https://docs.unsloth.ai/basics/running-and-saving-models/saving-to-vllm)

As of this writing, the PyPI package is at version 2026.8.22. [Source](https://pypi.org/project/unsloth/)

---

## 23. Compiled-Engine Serving: TensorRT-LLM and LMDeploy

vLLM and SGLang (§14) build their execution graph at Python runtime and rely on the framework's scheduler for continuous batching. A second family of serving engines instead compiles a model into a fixed, hardware-specific execution plan ahead of time — trading a slower, more involved build step for a leaner, faster-starting runtime.

**TensorRT-LLM** (github.com/NVIDIA/TensorRT-LLM, Apache 2.0) is NVIDIA's own entry in this category, and the engine that NIM (§18) uses internally for the supported GPU/model pairs where it doesn't fall back to vLLM. Where NIM wraps a pre-built engine behind a container so the operator never sees the build step, using TensorRT-LLM directly means running that build step: `trtllm-build` compiles a model checkpoint into a TensorRT engine file for the exact target GPU, applying kernel fusion, layer fusion, and (since TensorRT-LLM's Model Optimizer integration) quantization — FP8, FP4 (Blackwell), INT4-AWQ, and INT8 SmoothQuant are all documented calibration/quantization paths applied before or during the build. [Source](https://nvidia.github.io/TensorRT-LLM/quick-start-guide.html) Serving the resulting engine uses `trtllm-serve`, an OpenAI-compatible server entry point (available since TensorRT-LLM 0.9.0), with `--tp_size`/`--pp_size` multi-GPU flags that must match the parallelism degree the engine was built with — unlike vLLM's `--tensor-parallel-size`, which is a pure runtime choice, TensorRT-LLM's parallelism is baked into the compiled engine and cannot be changed without rebuilding. [Source](https://docs.nvidia.com/tensorrt-llm/index.html) TensorRT-LLM is NVIDIA-GPU-only (no ROCm/Vulkan path) and, as of this writing, at PyPI version 1.2.1 (April 2026). [Source](https://pypi.org/project/tensorrt-llm/) The practical tradeoff versus §14.1's vLLM: TensorRT-LLM engines generally start faster and run with less scheduling overhead once built, at the cost of a build step that is per-GPU-model-specific (an engine built for one GPU SKU is not portable to another) and a more involved iteration loop when the model or quantization scheme changes.

**LMDeploy** (github.com/InternLM/lmdeploy, Apache 2.0), from the Shanghai AI Laboratory / InternLM group, occupies similar ground with a different lineage: it ships two inference engines, **TurboMind** (a compiled, CUDA-kernel-optimized C++ engine, analogous in spirit to TensorRT-LLM's ahead-of-time approach) and a pure-Python PyTorch eager-mode engine for broader model/hardware compatibility at lower performance. LMDeploy's own documentation reports up to 2.4× faster inference for AWQ 4-bit-quantized models on the TurboMind engine versus FP16, a self-reported figure that should be read with the same caveat applied to this chapter's other vendor benchmarks. [Source](https://github.com/InternLM/lmdeploy) Its own quantization module implements AWQ specifically (§17), while its TurboMind runtime can additionally load GPTQ-quantized checkpoints produced elsewhere:

```bash
pip install lmdeploy
lmdeploy serve api_server ./internlm2_5-7b-chat-4bit \
    --backend turbomind --model-format awq
```

which exposes an OpenAI-compatible endpoint and a Swagger UI on port 23333 by default. [Source](https://lmdeploy.readthedocs.io/en/stable/quantization/w4a16.html) LMDeploy documents an AMD ROCm installation path alongside its primary CUDA target, making it one of the few compiled-engine serving stacks (unlike TensorRT-LLM) with any AMD support. [Source](https://lmdeploy.readthedocs.io/en/latest/get_started/installation.html)

---

## 24. Model-Swapping Proxies: llama-swap and LiteLLM

Every serving engine in this chapter — llama.cpp's `llama-server`, Ollama's daemon aside, vLLM, SGLang, TensorRT-LLM, LMDeploy — is, at its core, a process that serves *one* loaded model (Ollama is the deliberate exception, already covered in §5, which manages multiple models within a single daemon). A workstation running several different local models for different tasks therefore needs something above the engine layer to route requests to the right model and, in the common case where VRAM can't hold every model simultaneously, to load and unload them on demand.

**llama-swap** (github.com/mostlygeek/llama-swap, MIT license) is a small Go proxy, distributed as a single binary or Docker image, purpose-built for this: it sits in front of any OpenAI/Anthropic-compatible backend — not only `llama-server`, despite the name, but vLLM, Ollama, or any other command-line-launchable server — and inspects the `model` field of each incoming request to decide which backend process should be running. If the currently-running process doesn't match, llama-swap stops it and launches the correct one before forwarding the request. Configuration is a single YAML file mapping model names to arbitrary shell launch commands, with additional features for automatic idle-timeout unloading (TTL), running multiple models concurrently when VRAM allows ("groups"), and request-modifying filters. [Source](https://github.com/mostlygeek/llama-swap) [Source](https://github.com/mostlygeek/llama-swap/blob/main/config.example.yaml) This is a narrower, single-host answer to the same problem §15's disaggregated-serving connectors and §20's kagent solve at cluster scale: all three route a request to the right place, but llama-swap does it with a YAML file and process supervision rather than a distributed KV-cache transfer fabric or Kubernetes CRDs.

**LiteLLM** (github.com/BerriAI/litellm, MIT core license with a separate enterprise-licensed `enterprise/` subdirectory) solves an adjacent but distinct problem: rather than swapping which model process is loaded, its proxy server (`litellm --config config.yaml`) presents one OpenAI-compatible endpoint in front of upstream providers that are already running — local backends like Ollama or vLLM via their own `api_base` URLs, alongside cloud APIs (OpenAI, Anthropic, Bedrock, Vertex, and around a hundred others) — adding cost tracking, load balancing across multiple upstream instances, and automatic failover if one backend is unreachable. [Source](https://github.com/BerriAI/litellm) [Source](https://docs.litellm.ai/docs/proxy_server) In a local-inference setup combining, say, a small always-on Ollama model with an occasionally-launched large vLLM deployment, LiteLLM is the layer an application talks to, while llama-swap (or Ollama's own daemon) is what actually manages which weights are resident in VRAM underneath it — the two compose rather than compete.

---

## 25. Structured Output, Grammars, and Function Calling

A recurring requirement in production LLM serving — return valid JSON matching a schema, pick from a fixed set of labels, emit a well-formed tool call — can be handled by re-prompting and retrying until the model's free-text output happens to parse, or it can be guaranteed at the decoding level. **Constrained** (or **guided**) **decoding** takes the second approach: a grammar (a JSON Schema, a regular expression, or a context-free EBNF grammar) is compiled into a state machine that tracks, at every generation step, which tokens in the vocabulary are valid continuations; the sampler then masks out (sets to −∞ logit) every invalid token before sampling, so the model is architecturally incapable of producing a token that would violate the schema. Because the constraint is enforced by masking valid vocabulary rather than by modifying weights or prompting, this composes with everything else in this chapter — quantization (§17), speculative decoding (§16), and continuous batching (§14) all operate underneath it unmodified.

Three implementations of the state-machine/grammar-compilation step are relevant here:

- **Outlines** (github.com/dottxt-ai/outlines, Apache 2.0) was among the first widely-adopted implementations, converting a regex or JSON Schema into a finite-state machine (precomputing, for every FSM state, which vocabulary tokens are valid transitions) so that per-token masking at generation time is a cheap lookup rather than a schema re-parse. It integrates as a logits processor across `transformers`, llama.cpp, and vLLM. The underlying technique is described in "Efficient Guided Generation for Large Language Models." [Source](https://arxiv.org/abs/2307.09702) [Source](https://github.com/dottxt-ai/outlines)
- **XGrammar** (github.com/mlc-ai/xgrammar, Apache 2.0), from the MLC-LLM team, is a newer grammar-execution engine that further splits vocabulary tokens into context-independent ones (precheckable once, ahead of time) and context-dependent ones (checked only when actually reached), and has since become the default guided-decoding backend in both vLLM and SGLang. [Source](https://arxiv.org/abs/2411.15100) [Source](https://github.com/mlc-ai/xgrammar)
- llama.cpp implements its own **GBNF** (GGML BNF) grammar format natively, letting `llama-server`/`llama-cli` constrain output to an arbitrary context-free grammar — including JSON Schema, which llama.cpp converts to GBNF internally — without depending on Outlines or XGrammar at all. [Source](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md)

**vLLM** exposes backend selection via `--structured-outputs-config.backend` (accepted values include `xgrammar`, `guidance`, `outlines`, and `lm-format-enforcer`; `auto`, the default, picks a backend per-request), and per-request constraints via OpenAI-compatible fields — `response_format` with a JSON Schema, or vLLM's own `extra_body` parameters `guided_json`, `guided_regex`, `guided_choice`, and `guided_grammar`. [Source](https://docs.vllm.ai/en/latest/features/structured_outputs/) **SGLang** takes the same three-backend approach via `--grammar-backend {xgrammar,outlines,llguidance}` (XGrammar is the default), with per-request `json_schema`, `regex`, or `ebnf` fields inside `sampling_params`. [Source](https://docs.sglang.io/advanced_features/structured_outputs.html)

**Function/tool calling** — the OpenAI-compatible `tools`/`tool_choice` API surface that lets a model emit a structured call into an application-defined function rather than free text — is a related but distinct feature layered on top of the same masking mechanism: the tool-call JSON itself is grammar-constrained to match the requested function's parameter schema. vLLM requires two flags to enable it: `--enable-auto-tool-choice` (opt-in, since letting a model autonomously decide to call a tool is a behavior change) and `--tool-call-parser <name>`, selecting a parser matched to the model family's native tool-call output format (e.g., `hermes`, `mistral`, `llama3_json`, `internlm`) — mismatching the parser to the model's actual output format is a common source of tool-calling failures in practice, since each model family was fine-tuned to emit tool calls in its own specific token sequence. [Source](https://docs.vllm.ai/en/latest/features/tool_calling/)

---

## 26. Multi-LoRA Serving

§14's vLLM and SGLang deployments both assume one model checkpoint per server. In practice, teams that fine-tune per-customer, per-task, or per-language LoRA adapters off a shared base model rarely want to run a separate GPU deployment per adapter — merging each adapter into its own full checkpoint (§22's `save_pretrained_merged` workflow, run once per adapter) multiplies both disk footprint and the number of processes to keep warm. The alternative both serving engines from §14 support directly is **multi-LoRA serving**: one base-model deployment, with a batch of adapters kept resident and swapped in per-request at the kernel level rather than per-process.

The kernel technique both engines build on traces to two 2023 papers written concurrently: **Punica** (Chen et al., arXiv:2310.18547) contributes the batched CUDA kernel — a "Segmented Gather Matrix-Vector multiplication" (SGMV) op that lets requests targeting *different* LoRA adapters be processed together in one batched GPU call instead of falling back to per-adapter sub-batches — and **S-LoRA** (Sheng et al., arXiv:2311.03285, MLSys 2024) contributes "Unified Paging," a shared memory pool that pages both adapter weights and KV cache blocks (of varying LoRA rank and sequence length) out of the same pool used elsewhere in this chapter for the KV cache itself (§9). S-LoRA's paper reports up to 4× higher throughput than earlier per-adapter-batch approaches when serving thousands of concurrent adapters — a research-lineage number describing the technique in general, not a measured figure for either engine's specific implementation below. [Source](https://arxiv.org/abs/2310.18547) [Source](https://arxiv.org/abs/2311.03285)

**vLLM.** Multi-LoRA is enabled with `--enable-lora`, alongside `--max-loras` (how many adapters may be resident in a single batch concurrently), `--max-lora-rank`, `--max-cpu-loras` (adapters cached in host memory beyond the GPU-resident set), and `--lora-target-modules`. A request selects an adapter by naming it in the OpenAI-API `model` field:

```bash
vllm serve meta-llama/Llama-3.2-3B-Instruct \
    --enable-lora \
    --lora-modules sql-lora=jeeejeee/llama32-3b-text2sql-spider

curl http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "sql-lora", "prompt": "San Francisco is a", "max_tokens": 7, "temperature": 0}'
```

Adapters can also be hot-loaded and unloaded without restarting the server: setting `VLLM_ALLOW_RUNTIME_LORA_UPDATING=True` exposes `POST /v1/load_lora_adapter` and `POST /v1/unload_lora_adapter` endpoints, with a pluggable `LoRAResolver` interface (built-in filesystem and Hugging Face Hub resolvers) for locating an adapter by name at load time. [Source](https://docs.vllm.ai/en/latest/features/lora.html)

**SGLang.** The equivalent flags are `--enable-lora` (implicitly set if `--lora-paths` is given), `--lora-paths` (accepting a bare path, a `name=path` pair, or a JSON object with `lora_name`/`lora_path`/`pinned` fields), `--max-loras-per-batch`, `--max-loaded-loras` (a CPU-resident cap, at least as large as `--max-loras-per-batch`), `--lora-backend` (`triton` or a chunked-SGMV `csgmv` kernel), and `--enable-lora-overlap-loading` (overlaps host-to-device adapter transfer with ongoing compute so a newly-loaded adapter doesn't stall the batch). SGLang's own documentation states its LoRA batching "incorporates methods from S-LoRA and Punica" directly:

```bash
python3 -m sglang.launch_server \
  --model-path meta-llama/Meta-Llama-3.1-8B-Instruct \
  --enable-lora \
  --lora-paths lora0=algoprog/fact-generation-llama-3.1-8b-instruct-lora \
               lora1=Nutanix/Meta-Llama-3.1-8B-Instruct_SFT_lora_4_alpha_16_humaneval_raw_json \
  --max-loras-per-batch 2
```

with a request selecting an adapter via `"model": "meta-llama/Meta-Llama-3.1-8B-Instruct:lora0"`, and the same load/unload pattern exposed as `POST /load_lora_adapter` / `POST /unload_lora_adapter`. [Source](https://docs.sglang.io/docs/advanced_features/lora)

The practical payoff of this scheme is GPU utilization: rather than a GPU sitting idle waiting for adapter-A traffic while adapter-B's dedicated deployment is overloaded, one base-model deployment's batch scheduler (§14.1, §14.2) can mix requests for many adapters into the same iteration, at the modest per-request cost of an extra batched LoRA kernel call. This makes multi-LoRA serving a direct complement to the fine-tune-many-small-adapters workflow §22 (Unsloth) describes — fine-tune per-task or per-tenant adapters cheaply, then serve all of them off one deployment instead of one GPU pool per adapter.

---

## 27. ExLlamaV2 and ExLlamaV3

A third local-inference lineage, distinct from both GGUF's K-quants (§1.2) and the GPTQ/AWQ/FP8 family §17 covers, comes from a single-author project by turboderp: **ExLlamaV2** and its successor **ExLlamaV3**, each pairing a CUDA-only inference engine with its own quantization format.

**ExLlamaV2** (github.com/turboderp-org/exllamav2, MIT license) introduced **EXL2**, a variable-bit-depth format descended from the same GPTQ-style layer-wise error-compensation approach as §17's GPTQ, but relaxing GPTQ's single-bit-width-per-model constraint: EXL2 mixes 2-, 3-, 4-, 5-, 6-, and 8-bit weight groups *within the same layer*, chosen per-group by sensitivity, so a checkpoint's effective compression is described as a non-integer average — "bits per weight" (bpw), e.g. 4.65bpw — rather than a fixed integer bit-width. [Source](https://github.com/turboderp-org/exllamav2/blob/master/README.md) [Source](https://docs.mistral.ai/resources/cookbooks/concept-deep-dive-quantization-methods-exl2) As of this writing, the project's own README states it is **archived, with development continuing on ExLlamaV3** — a maintenance-status distinction worth noting before adopting it for a new deployment. [Source](https://github.com/turboderp-org/exllamav2)

**ExLlamaV3** (github.com/turboderp-org/exllamav3, MIT license, currently in **beta**) introduces **EXL3**, described in the project's own documentation as "a streamlined variant of QTIP" — Quantization with Trellises and Incoherence Processing (Tseng et al., NeurIPS 2024), which uses a trellis-coded quantization scheme rather than GPTQ-style per-group scale/zero-point compensation. [Source](https://arxiv.org/abs/2406.11235) [Source](https://github.com/turboderp-org/exllamav3) EXL3 conversion uses dynamic Hessian computation and a fused Viterbi kernel, completing in minutes for small models and hours for 70B-class models on a single RTX 4090; unlike EXL2, EXL3 checkpoints preserve the original safetensors tensor naming rather than renaming tensors, a deliberate choice the project states is meant to keep the door open to future `transformers`/vLLM compatibility. The project reports Llama-3.1-70B remaining "coherent" quantized to 1.6 bpw, fitting under 16GB of VRAM — a self-reported extreme-compression claim, not an independently audited one.

Both engines are **NVIDIA CUDA-only**; ExLlamaV3's own documentation lists ROCm/AMD support as an unimplemented to-do item, not a supported backend, in contrast to the AMD coverage §8, §14, and §17 describe for ROCm-native tooling elsewhere in this chapter. [Source](https://github.com/turboderp-org/exllamav3)

```bash
pip install exllamav2
# or, for ExLlamaV3 (example pinned wheel; check the releases page for the current version):
pip install https://github.com/turboderp-org/exllamav3/releases/download/v0.0.6/exllamav3-0.0.6+cu128.torch2.8.0-cp313-cp313-linux_x86_64.whl
```

Neither project ships its own OpenAI-compatible HTTP server as a first-class citizen; the ecosystem-standard frontend is **TabbyAPI** (github.com/theroyallab/tabbyAPI, AGPLv3), a FastAPI wrapper around the ExLlamaV2/V3 Python API that ExLlamaV2's own documentation describes as its "official and recommended backend server," exposing the same `/v1/chat/completions` route this chapter's other engines use. [Source](https://github.com/theroyallab/tabbyAPI)

Pre-quantized checkpoints for both formats are distributed as ordinary Hugging Face safetensors repositories (no custom container format, unlike GGUF) — retrieved the same way as any other model with the `hf download` command from §21 — following a community naming convention that suffixes the repository name with `-exl2`/`-EXL3` and the target bpw, e.g. `Meta-Llama-3-8B-Instruct-4.0-bpw-exl2`. [Source](https://huggingface.co/alokabhishek/Meta-Llama-3-8B-Instruct-4.0-bpw-exl2) ExLlamaV2's README reports throughput gains over its own V1 predecessor (e.g., 257 vs. 217 tok/s for a 7B model at 3.0bpw on an RTX 4090); no independently audited, neutral-party benchmark comparing EXL2/EXL3 against GGUF or GPTQ at matched bit-depths was found during research, so any such comparison should be treated as community- or vendor-sourced rather than authoritative.

---

## 28. llm-d: Kubernetes-Native Distributed Inference

§20 drew a sharp line around kagent: it orchestrates *agents*, not inference — it points a `ModelConfig` at an already-running endpoint and never touches the serving path itself. **llm-d** (github.com/llm-d/llm-d, llm-d.ai) sits on the opposite side of that line: it is a Kubernetes-native orchestration layer for the model-serving pods themselves, sitting directly above vLLM/SGLang deployments rather than above agents that call them.

llm-d was launched in 2025 by Red Hat, Google Cloud, IBM Research, CoreWeave, and NVIDIA as founding contributors, later joined by AMD, Cisco, Hugging Face, Intel, Lambda, Mistral AI, and academic partners, and was accepted as a **CNCF Sandbox project on 2026-03-24**. [Source](https://www.cncf.io/blog/2026/03/24/welcome-llm-d-to-the-cncf-evolving-kubernetes-into-sota-ai-infrastructure/) [Source](https://research.ibm.com/blog/donating-llm-d-to-the-cloud-native-computing-foundation) It is Apache 2.0 licensed. As of this writing the project is at v0.9.0 (2026-08-17), with a steady roughly-monthly release cadence since its v0.2.0 debut in mid-2025 — an actively-maintained, fast-iterating project rather than a settled or dormant one.

llm-d is explicitly engine-agnostic rather than vLLM-specific: its own site describes it as running "vLLM, SGLang, and more across your cluster, turning single-node engines into production-grade serving," and its README frames the boundary precisely — "model servers like vLLM and SGLang handle efficiently running large language models on accelerators... llm-d provides state-of-the-art orchestration and optimizations above model servers." [Source](https://llm-d.ai/) In practice, the reference quickstart deploys vLLM pods via a Kustomize overlay, making §14.1's `vllm serve` invocation the unit llm-d schedules and routes traffic to at cluster scale, rather than something it replaces.

The orchestration itself builds on the **Kubernetes SIG Gateway API Inference Extension** — a set of CRDs and a gateway layer purpose-built for LLM traffic routing — installed as an explicit prerequisite before the router and model-server Helm charts:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/${GAIE_VERSION}/v1-manifests.yaml

helm install "$GUIDE_NAME" "$ROUTER_STANDALONE_CHART" \
    -f guides/recipes/router/base.values.yaml \
    -n "$NAMESPACE" --version "$ROUTER_CHART_VERSION"

kubectl apply -n "$NAMESPACE" -k guides/optimized-baseline/modelserver/gpu/vllm/base/
```

[Source](https://llm-d.ai/docs/getting-started/quickstart)

On top of that routing layer, llm-d implements its own prefill/decode disaggregation and prefix-cache-aware request routing — the same architectural problem §15 covers via vLLM's experimental connector, NVIDIA Dynamo, and Mooncake, but orchestrated here as a Kubernetes-native scheduling concern across many pods rather than a single-process connector config. llm-d's own README reports up to 70% higher tokens/sec with prefill/decode disaggregation versus standard vLLM, and 3× higher output throughput with 2× faster time-to-first-token from prefix-cache-aware routing versus round-robin load balancing — self-reported project figures, following the same non-independently-audited caveat already noted for Dynamo and Mooncake in §15. No primary source directly documents how llm-d relates to Dynamo or to kagent; the overlap with Dynamo (both orchestrate P/D disaggregation over vLLM/SGLang, via different vendor-backed implementations) and the layering relative to kagent (which could, in principle, target an llm-d-fronted endpoint as a `ModelConfig` backend, per §20) are architectural observations from reading both projects' scope, not claims either project states explicitly.

---

## 29. Integrations

This chapter draws on and extends topics covered across the book:

- **Chapter 3 (GPU Memory Management)**: The `drm_gem` object lifecycle, VRAM allocators, and BAR aperture management underpin the physical memory that Vulkan `VkDeviceMemory` and HIP `hipMalloc` allocate from. The Resizable BAR mechanism described in §4.5 is controlled at the DRM/KMS layer.

- **Chapter 6 (ARM GPU Drivers — Panfrost)**: The Vulkan backend in llama.cpp targets any Vulkan 1.2-capable device, including ARM Mali (via Turnip or vendor Vulkan) on Android or embedded Linux. Panfrost's compute path exposes the same `ggml_backend_vk` interface; local inference on mobile ARM SoCs (e.g., Snapdragon X Elite with Adreno 740) follows the same Vulkan compute path described in §2.

- **Chapter 14 (NIR — Mesa's Shader IR)**: Mesa's Vulkan drivers (RADV, ANV, Turnip) compile incoming SPIR-V through NIR before emitting ISA. The SPIR-V shaders baked into llama.cpp (§2.2) flow through Mesa's NIR passes on AMD and Intel hardware, where NIR's algebraic simplifications and late optimisations directly affect kernel efficiency.

- **Chapter 18 (RADV — AMD Vulkan Driver)**: RADV is the Vulkan driver that executes the GGML compute shaders on AMD RDNA hardware. The RADV pipeline compilation path, descriptor set layout, and push constant implementation described in Chapter 18 are the substrate for the `vk_pipeline_struct` operations in §2.2.

- **Chapter 24 (Vulkan Compute)**: The compute shader dispatch model — `vkCmdDispatch`, workgroup sizing, shared memory, subgroup operations — is the foundation for all GGML Vulkan kernels. The synchronisation primitives (pipeline barriers, semaphores) described in Chapter 24 are used in the weight-upload path of §4.3.

- **Chapter 25 (ROCm)**: The ROCm software stack (HIP, rocBLAS, MIOpen, RCCL) described in Chapter 25 is the foundation for §8. The ROCm kernel driver and KFD (Kernel Fusion Driver) expose the hardware that `hipMalloc` and `rocblas_gemm_ex` operate on.

- **Chapter 48 (ROCm Training vs Inference)**: Chapter 48 covers PyTorch's ROCm training path (gradient computation, mixed-precision, distributed training). This chapter complements it by focusing on inference-time concerns: quantisation, KV cache, throughput optimisation, and lightweight runtimes that skip the training graph entirely.

- **Chapter 55 (GPU Containers — Docker + Ollama)**: Ollama's GPU detection (§5.2) is the userspace complement to the container-level GPU passthrough described in Chapter 55. The `--device /dev/dri/renderD128` flag in a Docker run command exposes the same DRM device that Ollama's sysfs probe finds.

- **Chapter 61 (Vulkan Extensions)**: The `VK_KHR_cooperative_matrix` extension exploited by GGML's flash attention shaders (§2.2, §3.3) is profiled in Chapter 61, along with extension negotiation, capability queries, and the SPIR-V cooperative matrix instruction set.

- **Chapter 71 (Intel Arc / Level Zero)**: The Level Zero backend that OpenVINO uses for Arc inference (§7.3) is described in depth in Chapter 71, covering command list construction, memory residency, kernel submission, and the relationship between Level Zero and the `xe`/`i915` DRM drivers.

- **Chapter 88 (NPU Integration — Offloading Prefill to NPU)**: The next chapter examines how Intel's NPU (in Meteor Lake/Lunar Lake) and Qualcomm's Hexagon DSP accelerate the prefill phase of LLM inference while the GPU handles generation. The `HETERO:NPU,GPU` OpenVINO execution mode described in §7.4 is the bridge between these chapters.

- **Chapter 108 (ROCm and HIP — AMD's GPU Compute Stack)**: Chapter 108's own outline anticipates this chapter's §14.1 coverage of vLLM on AMD hardware, cross-referencing it explicitly as "Ch124 (Local LLM Inference — ROCm backend for llama.cpp/vllm)." §8 of this chapter (ROCm MIOpen and HIP) and §14.1's ROCm install path both build on the CDNA3/MI300X architecture, XGMI Infinity Fabric multi-GPU topology, and `ROCR_VISIBLE_DEVICES` device-selection mechanism detailed there.

- **Chapters 229 and 232 (GPU Machine Learning Inference Algorithms; GPU Generative AI and LLM Inference on Linux)**: Both chapters independently outline speculative decoding (draft/verify rejection sampling, token trees, Medusa, EAGLE) from an algorithmic-derivation perspective. §16 of this chapter covers the same techniques from a practical, framework-flag perspective — how to turn speculative decoding on in vLLM, SGLang, and llama.cpp — and should be read as a deployment-level complement to their theoretical treatment, not a duplicate.

- **Chapters 240 and 248 (NVIDIA Cosmos, OSMO, and Omniverse Farm; Render Farm Infrastructure — Nucleus, OpenCue, and Job Distribution)**: Both chapters use the NGC catalog and NIM microservices in a render-farm/production-pipeline context (container distribution, Helm-chart deployment, multi-node orchestration). §18 of this chapter covers the same NGC/NIM mechanics — `nvcr.io` authentication, the NIM container model, licensing — from the standpoint of a single-node local LLM deployment rather than a farm-scale one.

- **Chapter 48 (ROCm Training vs Inference), again**: §22's Unsloth coverage is this chapter's one deliberate excursion into training — Chapter 48's PyTorch-on-ROCm training path and Unsloth's Triton-kernel-accelerated LoRA/QLoRA path are two different answers to the same problem (fitting gradient computation into limited VRAM); §22 is included specifically because Unsloth's output (a GGUF file or a merged 16-bit checkpoint) re-enters this chapter's own serving stack via §1's GGUF format or §14.1's vLLM path, not because this chapter otherwise covers training.

- **§18 (NVIDIA NGC Catalog and NIM), again**: §23's TensorRT-LLM coverage is the "underneath the hood" complement to §18's NIM discussion — NIM pre-compiles and containerises exactly the `trtllm-build` step §23 walks through manually, so a reader who wants to understand what a NIM container is actually running, or who needs a GPU/model pairing NIM doesn't pre-package, follows §23's direct `trtllm-build`/`trtllm-serve` path instead.

- **§5 (Ollama) and §20 (kagent), again**: §24's llama-swap and LiteLLM sit at the same "route a request to the right backend" layer as Ollama's built-in multi-model daemon (§5) and kagent's Kubernetes CRDs (§20), but at opposite ends of the operational-complexity spectrum from kagent — a single YAML file and process supervision on one host, versus cluster-wide orchestration. All three solve the same underlying problem (which running model should serve this request) at different scales.

- **§14 (Production Serving Engines), again**: §25's structured-output and tool-calling coverage is a feature layered directly on top of §14's vLLM and SGLang serving engines — the `--structured-outputs-config.backend` and `--grammar-backend` flags discussed in §25 are configuration surfaces on the exact server processes `vllm serve` and `python -m sglang.launch_server` start in §14.1 and §14.2.

- **§9 (KV Cache Management Strategies) and §14, again**: §26's multi-LoRA serving builds directly on §9's KV cache concepts and §14's Punica/S-LoRA-derived batching — the "Unified Paging" scheme S-LoRA introduces pages adapter weights out of the same kind of shared memory pool this chapter's KV cache material (§9) already establishes, and §22's Unsloth fine-tuning workflow is the natural upstream source of the adapters §26 serves.

- **§1.3 and §17, again**: §27's EXL2/EXL3 formats are a third quantization lineage alongside §1.3's GGUF K-quants and §17's GPTQ/AWQ/bitsandbytes/FP8 family — all three attack the same VRAM-budgeting problem worked through numerically in §11, with different bit-allocation strategies and different engine ecosystems.

- **§15 (Disaggregated Prefill-Decode Serving) and §20 (kagent), again**: §28's llm-d sits at the intersection of both — it implements the same prefill/decode disaggregation problem §15 covers via vLLM's connector abstraction, NVIDIA Dynamo, and Mooncake, but orchestrated as a Kubernetes-native scheduling concern across pods rather than a single-process config, and it occupies the model-serving layer that §20's kagent explicitly sits above (kagent orchestrates agents that call an endpoint; llm-d orchestrates the endpoint itself).

---

*Sources referenced in this chapter:*
- [llama.cpp GitHub repository](https://github.com/ggml-org/llama.cpp)
- [GGML Vulkan Backend — DeepWiki](https://deepwiki.com/ggml-org/llama.cpp/5.3-vulkan-backend-(cross-platform))
- [GGUF File Format — DeepWiki](https://deepwiki.com/ggml-org/llama.cpp/7.1-gguf-file-format)
- [Ollama GitHub repository](https://github.com/ollama/ollama)
- [Ollama GPU Discovery — DeepWiki](https://deepwiki.com/13rac1/ollama/5.3-gpu-discovery-and-hardware-acceleration)
- [vLLM Automatic Prefix Caching](https://docs.vllm.ai/en/stable/design/prefix_caching/)
- [vLLM Hybrid KV Cache Manager](https://docs.vllm.ai/en/latest/design/hybrid_kv_cache_manager/)
- [ONNX Runtime CUDA Execution Provider](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html)
- [ONNX Runtime OpenVINO Execution Provider](https://onnxruntime.ai/docs/execution-providers/OpenVINO-ExecutionProvider.html)
- [OrtCUDAProviderOptions API Reference](https://onnxruntime.ai/docs/api/c/struct_ort_c_u_d_a_provider_options.html)
- [OpenVINO Intel GPU Configuration](https://docs.openvino.ai/2025/get-started/install-openvino/configurations/configurations-intel-gpu.html)
- [AMD ROCm MIOpen](https://github.com/ROCm/MIOpen)
- [vLLM 0.9.x ROCm AITER Integration](https://rocm.blogs.amd.com/software-tools-optimization/vllm-0.9.x-rocm/README.html)
- [AMD Instinct MI300X Workload Optimization](https://rocm.docs.amd.com/en/latest/how-to/rocm-for-ai/inference-optimization/workload.html)
- [llama.cpp GPU Benchmark Scoreboard](https://knightli.com/en/2026/04/23/llama-cpp-gpu-benchmark-cuda-rocm-vulkan-scoreboard/)
- [llama.cpp Q8_0 vs Q4_K_M Bandwidth Efficiency Issue](https://github.com/ggml-org/llama.cpp/issues/21517)
- [Vulkan ReBAR bypass staging buffer issue](https://github.com/ggml-org/llama.cpp/issues/21590)
- [Unified Evaluation of llama.cpp Quantisation](https://arxiv.org/html/2601.14277v1)
- [LLM Inference at the Edge: Power Efficiency](https://arxiv.org/html/2603.23640)
- [nvtop GPU monitor](https://github.com/Syllo/nvtop)
- [gguf-parser-go](https://github.com/gpustack/gguf-parser-go)
- [Hugging Face Accelerate: Model Memory Estimator](https://huggingface.co/docs/accelerate/en/usage_guides/model_size_estimator)
- [Hugging Face Model Memory Utility Space](https://huggingface.co/spaces/hf-accelerate/model-memory-usage)
- [Can It Run LLM? (community VRAM calculator)](https://huggingface.co/spaces/Vokturz/can-it-run-llm)
- [Ollama GPU Support Documentation](https://github.com/ollama/ollama/blob/main/docs/gpu.mdx)
- [Ollama FAQ](https://github.com/ollama/ollama/blob/main/docs/faq.mdx)
- [Apple: New MacBook Pro Features M4 Family of Chips](https://www.apple.com/newsroom/2024/10/new-macbook-pro-features-m4-family-of-chips-and-apple-intelligence/)
- [Apple Reveals M3 Ultra](https://www.apple.com/newsroom/2025/03/apple-reveals-m3-ultra-taking-apple-silicon-to-a-new-extreme/)
- [Mac Studio M3 Ultra Running DeepSeek-R1 671B In-Memory](https://www.techradar.com/pro/apple-mac-studio-m3-ultra-workstation-can-run-deepseek-r1-671b-ai-model-entirely-in-memory-using-less-than-200w-reviewer-finds)
- [Native LLM and MLLM Inference at Scale on Apple Silicon](https://arxiv.org/html/2601.19139v1)
- [NVIDIA RTX 5090 Laptop GPU Specs](https://www.notebookcheck.net/Nvidia-GeForce-RTX-5090-Laptop-Benchmarks-and-Specs.934947.0.html)
- [NVIDIA Introduces RTX 5090/5080/5070 Laptop GPUs](https://www.tomshardware.com/pc-components/gpus/nvidia-introduces-rtx-5090-rtx-5080-and-rtx-5070-laptop-gpus-rtx-50-blackwell-goes-mobile-with-up-to-24gb-of-gddr7-memory)
- [Asahi Linux Blog (GPU driver progress reports)](https://asahilinux.org/blog/)
- [vLLM GitHub repository](https://github.com/vllm-project/vllm)
- [vLLM V1 Architecture Overview](https://docs.vllm.ai/en/latest/design/arch_overview/)
- [vLLM GPU Installation Guide](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/)
- [vLLM Online Serving Guide](https://docs.vllm.ai/en/latest/serving/online_serving/)
- [vLLM Engine Arguments Reference](https://docs.vllm.ai/en/v0.10.2/configuration/engine_args.html)
- [vLLM Distributed Serving](https://docs.vllm.ai/en/v0.9.1/serving/distributed_serving.html)
- [vLLM Parallelism and Scaling](https://docs.vllm.ai/en/latest/serving/parallelism_scaling/)
- [vLLM v0.28.0 Release Notes](https://github.com/vllm-project/vllm/releases/tag/v0.28.0)
- [SGLang: Efficient Execution of Structured Language Model Programs (arXiv)](https://arxiv.org/abs/2312.07104)
- [SGLang Launch Blog](https://www.lmsys.org/blog/2024-01-17-sglang/)
- [SGLang v0.4 Release Blog (Zero-Overhead Batch Scheduler, Cache-Aware Load Balancer)](https://www.lmsys.org/blog/2024-12-04-sglang-v0-4/)
- [SGLang Server Arguments Reference](https://docs.sglang.io/docs/advanced_features/server_arguments)
- [SGLang AMD GPU Support](https://github.com/sgl-project/sglang/blob/main/docs/platforms/amd_gpu.md)
- [SGLang vs. vLLM Throughput (Llama3 blog)](https://www.lmsys.org/blog/2024-07-25-sglang-llama3/)
- [SGLang v0.3 Release Blog (DeepSeek MLA)](https://www.lmsys.org/blog/2024-09-04-sglang-v0-3/)
- [SGLang PyPI Package](https://pypi.org/project/sglang/)
- [DistServe: Disaggregating Prefill and Decoding (arXiv, OSDI'24)](https://arxiv.org/abs/2401.09670)
- [vLLM Disaggregated Prefilling](https://docs.vllm.ai/en/latest/features/disagg_prefill/)
- [NVIDIA Dynamo GitHub repository](https://github.com/ai-dynamo/dynamo)
- [Mooncake: KVCache-centric Disaggregated Architecture (arXiv, FAST'25)](https://arxiv.org/abs/2407.00079)
- [Mooncake GitHub repository](https://github.com/kvcache-ai/Mooncake)
- [Fast Inference from Transformers via Speculative Decoding (arXiv)](https://arxiv.org/abs/2211.17192)
- [Accelerating Large Language Model Decoding with Speculative Sampling (arXiv)](https://arxiv.org/abs/2302.01318)
- [Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads (arXiv)](https://arxiv.org/abs/2401.10774)
- [Medusa GitHub repository](https://github.com/FasterDecoding/Medusa)
- [EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty (arXiv)](https://arxiv.org/abs/2401.15077)
- [EAGLE-2 (arXiv)](https://arxiv.org/pdf/2406.16858)
- [EAGLE-3 (arXiv)](https://arxiv.org/pdf/2503.01840)
- [EAGLE GitHub repository](https://github.com/SafeAILab/EAGLE)
- [Break the Sequential Dependency of LLM Inference Using Lookahead Decoding (arXiv)](https://arxiv.org/pdf/2402.02057)
- [Lookahead Decoding GitHub repository](https://github.com/hao-ai-lab/LookaheadDecoding)
- [vLLM Speculative Decoding](https://docs.vllm.ai/en/latest/features/speculative_decoding/)
- [SGLang Speculative Decoding](https://docs.sglang.io/advanced_features/speculative_decoding.html)
- [llama.cpp Speculative Decoding Docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md)
- [GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers (arXiv)](https://arxiv.org/abs/2210.17323)
- [GPTQModel GitHub repository](https://github.com/ModelCloud/GPTQModel)
- [AWQ: Activation-aware Weight Quantization (arXiv)](https://arxiv.org/abs/2306.00978)
- [AutoAWQ GitHub repository (archived)](https://github.com/casper-hansen/AutoAWQ)
- [LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale (arXiv)](https://arxiv.org/abs/2208.07339)
- [QLoRA: Efficient Finetuning of Quantized LLMs (arXiv)](https://arxiv.org/abs/2305.14314)
- [vLLM FP8 Quantization (LLM Compressor)](https://docs.vllm.ai/en/latest/features/quantization/llm_compressor/fp8/)
- [vLLM Quantization Overview](https://docs.vllm.ai/en/latest/features/quantization/)
- [NGC Catalog User Guide](https://docs.nvidia.com/ngc/gpu-cloud/ngc-catalog-user-guide/index.html)
- [NVIDIA NIM Product Page](https://developer.nvidia.com/nim)
- [NVIDIA NIM for LLMs — About/Overview](https://docs.nvidia.com/nim/large-language-models/latest/about-nim-llm/overview.html)
- [NVIDIA NIM for LLMs — Getting Started](https://docs.nvidia.com/nim/large-language-models/1.3.0/getting-started.html)
- [NVIDIA NIM Free Access for Developer Program Members](https://developer.nvidia.com/blog/access-to-nvidia-nim-now-available-free-to-developer-program-members/)
- [Docker Model Runner Announcement](https://www.docker.com/blog/introducing-docker-model-runner/)
- [Docker Model Runner GA Announcement](https://www.docker.com/blog/announcing-docker-model-runner-ga/)
- [docker/model-runner GitHub repository](https://github.com/docker/model-runner)
- [Docker Model Runner Documentation](https://docs.docker.com/ai/model-runner/)
- [Docker Model Runner Get Started (Linux)](https://docs.docker.com/ai/model-runner/get-started/)
- [Docker CLI Reference: docker model](https://docs.docker.com/reference/cli/docker/model/)
- [Docker Hub ai/llama3.2 Model](https://hub.docker.com/r/ai/llama3.2)
- [Docker Model Runner API Reference](https://docs.docker.com/ai/model-runner/api-reference/)
- [kagent GitHub repository](https://github.com/kagent-dev/kagent)
- [kagent Architecture Concepts](https://kagent.dev/docs/kagent/concepts/architecture)
- [kagent Supported Providers](https://kagent.dev/docs/kagent/supported-providers)
- [kagent Installation Guide](https://kagent.dev/docs/kagent/introduction/installation)
- [Hugging Face: Introducing the hf CLI](https://www.huggingface.co/blog/hf-cli)
- [huggingface_hub Migration Guide](https://huggingface.co/docs/huggingface_hub/en/concepts/migration)
- [huggingface_hub Download Guide](https://raw.githubusercontent.com/huggingface/huggingface_hub/main/docs/source/en/guides/download.md)
- [huggingface_hub Cache Management Guide](https://raw.githubusercontent.com/huggingface/huggingface_hub/main/docs/source/en/guides/manage-cache.md)
- [huggingface_hub Environment Variables Reference](https://raw.githubusercontent.com/huggingface/huggingface_hub/main/docs/source/en/package_reference/environment_variables.md)
- [Unsloth GitHub repository](https://github.com/unslothai/unsloth)
- [Unsloth Documentation](https://unsloth.ai/docs)
- [Unsloth AMD GPU Support](https://unsloth.ai/docs/basics/amd)
- [AMD: Train and Run Models on AMD GPUs with Unsloth](https://www.amd.com/en/developer/resources/technical-articles/2026/train-and-run-models-on-amd-gpus-with-unsloth.html)
- [Unsloth Install Guide — AMD](https://unsloth.ai/docs/get-started/install/amd)
- [Unsloth Continued Pretraining Docs](https://unsloth.ai/docs/basics/continued-pretraining)
- [Unsloth Saving to GGUF Docs](https://unsloth.ai/docs/basics/inference-and-deployment/saving-to-gguf)
- [Unsloth vLLM Deployment & Inference Guide](https://docs.unsloth.ai/basics/running-and-saving-models/saving-to-vllm)
- [Unsloth PyPI Package](https://pypi.org/project/unsloth/)
- [TensorRT-LLM GitHub repository](https://github.com/NVIDIA/TensorRT-LLM)
- [TensorRT-LLM Quick Start Guide](https://nvidia.github.io/TensorRT-LLM/quick-start-guide.html)
- [TensorRT-LLM Documentation](https://docs.nvidia.com/tensorrt-llm/index.html)
- [TensorRT-LLM PyPI Package](https://pypi.org/project/tensorrt-llm/)
- [LMDeploy GitHub repository](https://github.com/InternLM/lmdeploy)
- [LMDeploy W4A16 Quantization Docs](https://lmdeploy.readthedocs.io/en/stable/quantization/w4a16.html)
- [LMDeploy Installation Guide](https://lmdeploy.readthedocs.io/en/latest/get_started/installation.html)
- [llama-swap GitHub repository](https://github.com/mostlygeek/llama-swap)
- [llama-swap Example Configuration](https://github.com/mostlygeek/llama-swap/blob/main/config.example.yaml)
- [LiteLLM GitHub repository](https://github.com/BerriAI/litellm)
- [LiteLLM Proxy Server Docs](https://docs.litellm.ai/docs/proxy_server)
- [Outlines: Efficient Guided Generation for Large Language Models (arXiv)](https://arxiv.org/abs/2307.09702)
- [Outlines GitHub repository](https://github.com/dottxt-ai/outlines)
- [XGrammar: Flexible and Efficient Structured Generation Engine for Large Language Models (arXiv)](https://arxiv.org/abs/2411.15100)
- [XGrammar GitHub repository](https://github.com/mlc-ai/xgrammar)
- [llama.cpp Speculative Decoding / GBNF Grammar Docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md)
- [vLLM Structured Outputs](https://docs.vllm.ai/en/latest/features/structured_outputs/)
- [vLLM Tool Calling](https://docs.vllm.ai/en/latest/features/tool_calling/)
- [SGLang Structured Outputs](https://docs.sglang.io/advanced_features/structured_outputs.html)
- [Punica: Multi-Tenant LoRA Serving (arXiv)](https://arxiv.org/abs/2310.18547)
- [S-LoRA: Serving Thousands of Concurrent LoRA Adapters (arXiv)](https://arxiv.org/abs/2311.03285)
- [vLLM LoRA Adapters Docs](https://docs.vllm.ai/en/latest/features/lora.html)
- [SGLang LoRA Docs](https://docs.sglang.io/docs/advanced_features/lora)
- [ExLlamaV2 GitHub repository](https://github.com/turboderp-org/exllamav2)
- [ExLlamaV3 GitHub repository](https://github.com/turboderp-org/exllamav3)
- [QTIP: Quantization with Trellises and Incoherence Processing (arXiv)](https://arxiv.org/abs/2406.11235)
- [Mistral AI: EXL2 Quantization Cookbook](https://docs.mistral.ai/resources/cookbooks/concept-deep-dive-quantization-methods-exl2)
- [TabbyAPI GitHub repository](https://github.com/theroyallab/tabbyAPI)
- [Example EXL2 checkpoint on Hugging Face](https://huggingface.co/alokabhishek/Meta-Llama-3-8B-Instruct-4.0-bpw-exl2)
- [CNCF: Welcome llm-d to the CNCF](https://www.cncf.io/blog/2026/03/24/welcome-llm-d-to-the-cncf-evolving-kubernetes-into-sota-ai-infrastructure/)
- [IBM Research: Donating llm-d to the CNCF](https://research.ibm.com/blog/donating-llm-d-to-the-cloud-native-computing-foundation)
- [llm-d project site](https://llm-d.ai/)
- [llm-d Quickstart Docs](https://llm-d.ai/docs/getting-started/quickstart)

## Roadmap

### Near-term (6–12 months)
- **VK_KHR_cooperative_matrix broader adoption**: AMD RDNA4 and Intel Xe2 (Battlemage) drivers are landing full `VK_KHR_cooperative_matrix` support in RADV and ANV, enabling GGML's cooperative-matrix flash-attention shaders to reach parity with CUDA Tensor Core throughput on those GPUs.
- **llama.cpp RPC backend stabilisation**: The RPC backend that distributes layers across machines over TCP is progressing toward stable API status, enabling multi-node consumer-GPU inference clusters without a full MPI stack.
- **ROCm 7.x hipBLASLt TunableOp expansion**: AMD is extending TunableOp auto-selection to cover grouped GEMM and mixture-of-experts (MoE) gate-and-expert dispatch, directly accelerating Mixtral-class models on RDNA3/CDNA3 hardware.
- **Ollama structured output and JSON schema enforcement**: Ollama's roadmap includes first-class grammar-constrained decoding (integrating llama.cpp's GBNF grammar engine into the REST API), reducing application-side post-processing for tool-call workflows.

### Medium-term (1–3 years)
- **FP8 inference on consumer GPUs**: As NVIDIA Ada Lovelace FP8 tensor-core support matures in CUDA and GGML adds `GGML_TYPE_FP8_E4M3`/`E5M2` quantisation types, 70B models will fit in 35 GB VRAM at near-F16 quality, eliminating the need for aggressive K-quant compression on high-VRAM cards.
- **DMA-BUF zero-copy weight streaming**: Proposals on the linux-media and DRM mailing lists aim to expose GGUF tensor data directly to GPU via DMA-BUF heaps, bypassing the staging-buffer upload path entirely on unified-memory APUs and systems with PCIe P2P support (AMD SAM + NVLink equivalent).
- **Speculative decoding and draft-model integration in Ollama/vLLM**: Both projects are integrating speculative decoding (a small draft model generates candidate tokens verified by the main model), which can multiply effective throughput by 2–4× on memory-bandwidth-bound generation workloads while keeping the API surface unchanged.
- **Kernel-side GPU scheduling for inference QoS**: Linux DRM scheduler patches targeting time-sliced GPU contexts and priority classes will allow inference daemons (Ollama, vLLM) to co-exist with interactive Wayland compositors on the same GPU without frame-time disruption.

### Long-term
- **Unified inference runtime over Linux ioctl API**: Longer-term kernel proposals (discussed in the DRM compute working group) envision a vendor-neutral `DRM_IOCTL_COMPUTE_SUBMIT` for structured tensor dispatch, allowing runtimes like GGML and ONNX Runtime to bypass the Vulkan/HIP/CUDA API layers and submit compute workloads directly to the DRM scheduler, reducing per-token dispatch overhead and enabling GPU resource accounting at the kernel level.
- **Near-memory and CXL-attached weight storage**: CXL 3.0 memory expansion modules with 1–8 TB capacity and 200–400 GB/s bandwidth are expected to become viable weight-streaming targets for very large models (>200B parameters), with Linux CXL subsystem drivers coordinating allocation between host RAM, CXL memory, and GPU VRAM as a unified inference memory hierarchy.
- **NPU + GPU heterogeneous inference as first-class Linux stack**: As Intel Lunar Lake, AMD XDNA2, and Qualcomm Hexagon NPUs gain upstream Linux kernel drivers (`amdxdna`, `qcom_npu`), frameworks like OpenVINO and ONNX Runtime are expected to converge on a common heterogeneous-dispatch API, making prefill-on-NPU / generation-on-GPU pipelines (the `HETERO:NPU,GPU` pattern of §7.4) routine rather than experimental.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
