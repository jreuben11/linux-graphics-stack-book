# Chapter 91: MLIR and the Emerging GPU Compiler Infrastructure

> **Part**: Part IV — Mesa Architecture
> **Audience**: Compiler engineers who work on GPU code generation, ML framework developers who want to understand what happens below PyTorch/JAX's JIT boundary, and GPU driver developers who want to understand how MLIR-emitted SPIR-V arrives at Mesa's front door.
> **Status**: First draft — 2026-06-18

## Table of Contents

- [Scope](#scope)
- [1. Introduction: The Modern GPU Compilation Stack](#1-introduction-the-modern-gpu-compilation-stack)
- [2. MLIR Architecture for GPU Compilation](#2-mlir-architecture-for-gpu-compilation)
- [3. The GPU Dialect](#3-the-gpu-dialect)
  - [3.3 The NVVM Dialect](#33-the-nvvm-dialect)
- [4. OpenAI Triton: Python to GPU Compiler](#4-openai-triton-python-to-gpu-compiler)
- [5. Triton FlashAttention and Kernel Performance](#5-triton-flashattention-and-kernel-performance)
- [6. IREE: MLIR-Based Inference for Vulkan](#6-iree-mlir-based-inference-for-vulkan)
- [7. XLA: Accelerated Linear Algebra](#7-xla-accelerated-linear-algebra)
  - [7.0 OpenXLA, HLO, MHLO, StableHLO, CHLO, and IREE: The Ecosystem Map](#70-openxla-hlo-mhlo-stablehlo-chlo-and-iree-the-ecosystem-map)
- [8. SPIR-V as the Cross-Compiler Target](#8-spir-v-as-the-cross-compiler-target)
- [9. Cooperative Matrix and Tensor Core Access](#9-cooperative-matrix-and-tensor-core-access)
- [10. The Mesa Connection: SPIR-V to NIR to ISA](#10-the-mesa-connection-spir-v-to-nir-to-isa)
- [11. Why MLIR Stops at SPIR-V: The ISA Boundary](#11-why-mlir-stops-at-spir-v-the-isa-boundary)
- [Integrations](#integrations)
- [References](#references)

---

## Scope

This chapter is aimed at three overlapping audiences. **Compiler engineers** will find a detailed account of MLIR's dialect hierarchy, the progressive lowering philosophy, and how each stage maps onto real GPU hardware concepts. **ML framework developers** who use PyTorch, JAX, or similar tools will gain a clear picture of what happens between their `@torch.compile` or `jax.jit` call and the GPU executing instructions — a journey that increasingly passes through MLIR, Triton, or XLA. **GPU driver developers** will find an explanation of how MLIR-generated SPIR-V feeds into Mesa's existing `spirv_to_nir` pipeline and what constraints that places on the upstream compilers.

The chapter sits logically at the top of the Mesa architecture part because MLIR and its associated tools occupy a layer *above* Mesa: they produce SPIR-V, PTX, or AMD GCN assembly that Mesa or vendor drivers then consume. Understanding this layer is increasingly important for anyone working on the Linux graphics stack as ML workloads become first-class citizens on GPU hardware.

---

## 1. Introduction: The Modern GPU Compilation Stack

The path from a neural network layer to executed GPU instructions now passes through at least four distinct compiler systems on a typical Linux machine. A **PyTorch** `nn.Linear` forward pass might travel through **PyTorch**'s **TorchInductor** (which produces **Triton** IR), then through **Triton**'s **GPU**-dialect lowering to **LLVM** IR and **PTX**, then through **ptxas** to **CUDA** binary, all before the GPU ever sees a machine instruction. On **Vulkan** paths the journey is different but equally layered: a **JAX** model compiles via **XLA** to **StableHLO**, which **IREE** then lowers through its **Stream** and **HAL** dialects to **SPIR-V** binary, which **Mesa**'s **RADV** or **ANV** driver reads through **spirv_to_nir**, compiles through **NIR** passes and **ACO**, and emits as **RDNA** or **Gen12** ISA.

```
Application (Python / C++)
     │
     ▼
ML Framework IR (TorchScript / JAX trace / ONNX)
     │
     ▼
MLIR Dialects (linalg → vector → gpu / SPIR-V)
     │
     ▼
Backend IR (LLVM IR / SPIR-V binary)
     │
     ▼
Vendor compiler (ptxas / LLVM AMDGPU / Mesa spirv_to_nir)
     │
     ▼
GPU ISA (PTX → SASS / AMDGCN / Intel EU ISA)
```

**Why MLIR matters here.** Before **MLIR**, each ML framework carried its own bespoke intermediate representation. **TensorFlow** had its graph IR, **PyTorch** had **TorchScript**, **TVM** had **Relay**. Each required its own lowering path to **LLVM** and its own GPU back-end. **MLIR**, released by Google in 2020 and hosted under the **LLVM** project, introduced a common infrastructure of *dialects* — extensible sets of operations, types, and attributes with well-defined legality rules and rewrite patterns. A single piece of **MLIR** infrastructure (a lowering pass, an optimisation pass) can then be shared across multiple frameworks that target the same source dialect. Section 2 covers **MLIR**'s architecture in depth, including its core abstractions (**Operations**, **Values**, **Regions**, **Blocks**), the full dialect table (`**linalg**`, `**vector**`, `**arith**`, `**memref**`, `**scf**`, `**affine**`, `**gpu**`, `**nvgpu**`, `**rocdl**`, `**spirv**`, `**llvm**`), progressive lowering philosophy, and the **mlir-opt** and **mlir-translate** command-line tools.

**The gpu dialect.** Section 3 examines the `**gpu**` dialect, which provides a hardware-neutral abstraction layer above vendor-specific dialects. It defines `**gpu.module**`, `**gpu.func**`, `**gpu.launch**`, and `**gpu.launch_func**` operations, GPU intrinsics such as `**gpu.thread_id**`, `**gpu.block_id**`, and `**gpu.barrier**`, and memory operations including `**gpu.alloc**` and `**gpu.memcpy**`. The `**-convert-gpu-to-spirv**` pass (also called `**GpuToSpirvPass**`) lowers `**gpu**` dialect to the `**spirv**` dialect for **Vulkan** compute targets; the newer `**-gpu-module-to-binary**` pass invokes registered serialisers directly. Section 3 also covers how **IREE** uses the `**gpu**` dialect as its primary GPU representation.

**OpenAI Triton.** Section 4 covers **Triton**, the Python-to-GPU compiler developed by OpenAI. Programmers write kernels as Python functions decorated with `**@triton.jit**`; the **Triton** compiler lowers them through a progressive **MLIR** pipeline: first to the `**tt**` dialect (tile-level operations such as `**tl.load**`, `**tl.store**`, `**tl.dot**`), then to the `**ttgpu**` (**TritonGPU**) dialect which adds hardware layout annotations (`**#blocked**`, `**#shared**`, `**#nvidia_mma**`, `**#amd_mfma**`, `**#amd_wmma**`), and finally to **NVVM** or **ROCDL** intrinsics before **LLVM** emits **PTX** or **AMDGCN** assembly. The 2025 `**Gluon**` dialect extends **Triton** with lower-level architectural features including warp specialisation and **Hopper** **TMA** descriptors. `**@triton.autotune**` drives search over compile-time tile size configurations. Section 4 also covers the **Triton-ROCm** (**AMD**) backend targeting **MFMA** on **CDNA** and **WMMA** on **RDNA3**, and the experimental **Triton** **Vulkan**/**SPIR-V** backend.

**FlashAttention and Triton kernel performance.** Section 5 presents **FlashAttention-2** as the canonical high-performance **Triton** kernel, demonstrating how the tiling abstraction avoids materialising the N×N attention score matrix in **HBM** by streaming **K** and **V** tiles through shared memory and maintaining an online softmax with running maximum and sum accumulators. The section analyses block size tuning (`**BLOCK_M**`, `**BLOCK_N**`, `**BLOCK_DMODEL**`) on **A100** and **MI300X** hardware, adoption in **vLLM**, **llama.cpp**, and the **transformers** library, and current gaps in the **Vulkan** **SPIR-V** path — missing asynchronous copy instructions (**PTX** `**cp.async**`), warp shuffle, and **Hopper** **TMA** descriptors.

**IREE.** Section 6 covers **IREE** (Intermediate Representation Execution Environment), **Google**'s production **MLIR** compiler and runtime for neural network inference. Its compilation pipeline lowers framework models (**PyTorch**, **TFLite**, **ONNX**) through **StableHLO** → **IREE Flow** dialect → **IREE Stream** dialect → **IREE HAL** dialect → **SPIR-V** (or **PTX** / **LLVM CPU**), embedding the results in a **`.vmfb`** (**VM FlatBuffer**) binary. The `**iree-compile**` command-line tool drives this pipeline with flags such as `**--iree-hal-target-device=vulkan**` and `**--iree-vulkan-target=rdna3**`. At runtime, **IREE**'s **HAL** C API (`**iree_hal_driver_registry_try_create**`, `**iree_runtime_session_create_with_device**`) wraps `**vkCreateComputePipeline**` and `**vkCmdDispatch**`. Section 6 also covers **IREE**'s **Android** deployment path via `**libvulkan.so**`.

**XLA.** Section 7 covers **XLA** (Accelerated Linear Algebra), the deep learning compiler backend for **JAX** (`**jax.jit**`), **TensorFlow** (`**tf.function**`), and **PyTorch XLA**. Its primary representation is **StableHLO**. The **XLA** GPU backend pipeline lowers **StableHLO** through **HLO** graph optimisation passes — **SPMD** partitioner, layout assignment, fusion — to code generation via library calls (**cuBLAS**, **cuDNN**, **rocBLAS**, **MIOpen**), direct **LLVM** IR emission, or **Triton** emission for matmul-heavy fusions. **JAX** entry points (`**jax.jit**`, `**jax.ShapeDtypeStruct**`, `**lowered.compile()**`) are illustrated. The section covers **XLA** on **AMD** via `**jax[rocm]**` and **AMDGPU**, and the experimental **XLA** **Vulkan** backend via **IREE** integration.

**SPIR-V as cross-compiler target.** Section 8 examines **SPIR-V** as the interchange format between the high-level GPU compiler world and the driver world. It covers the **SPIR-V** binary format (32-bit word encoding, five-word header with magic `0x07230203`), the strict ordering of module-level instructions (`**OpCapability**`, `**OpExtension**`, `**OpMemoryModel**`, `**OpEntryPoint**`, `**OpExecutionMode**`), the **SPIRV-Tools** suite (`**spirv-val**`, `**spirv-opt**`, `**spirv-dis**`, `**spirv-as**`), and `**spirv-cross**` for reverse translation to **GLSL**, **HLSL**, or **MSL** (used by **Proton**, **DXVK**, and **MoltenVK**). The section concludes with the complete **MLIR** → `**mlir-translate --serialize-spirv**` → `**vkCreateShaderModule**` → `**spirv_to_nir()**` → **NIR** → **ACO** / **LLVM** → GPU ISA path.

**Cooperative matrix and Tensor Core access.** Section 9 covers the hardware matrix-multiply units — **Tensor Cores** on **NVIDIA**, **WMMA** on **RDNA3**, **MFMA** on **CDNA**, and **XMX** (**Xe Matrix Extensions**) on **Intel** — and their exposure through `**VK_KHR_cooperative_matrix**` and `**SPV_KHR_cooperative_matrix**`. It covers the **Vulkan** query API (`**vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR**`), the **SPIR-V** instructions `**OpTypeCooperativeMatrixKHR**` and `**OpCooperativeMatrixMulAddKHR**`, **RADV**'s mapping to **RDNA3** **WMMA** via **NIR** (`**nir_intrinsic_cooperative_matrix_mul_add**`) and **ACO**, **ANV**'s mapping to **DPAS** (Dot Product Accumulate Systolic) on **Gfx12.5+** / **DG2** (**Alchemist**) and **Xe2+**, **Triton**'s `**tl.dot**` lowering via `**#nvidia_mma**` and `**#amd_wmma**` layouts, and cooperative matrix use in **FlashAttention** attention computation.

**The Mesa connection.** **SPIR-V** is the lingua franca between the higher-level GPU compiler world and **Mesa**'s driver layer. When **IREE**, **Triton** (via its **Vulkan** backend), or any other **MLIR**-based system targets **Vulkan**, it serialises **MLIR**'s **SPIR-V** dialect to a binary **SPIR-V** module and hands it to `**vkCreateShaderModule**`. **Mesa**'s **SPIR-V** reader (`**src/compiler/spirv/spirv_to_nir.c**`) then translates this binary into **NIR** (Chapter 14), where **Mesa**'s own optimisation and lowering passes take over. Section 10 traces the full path including the `**spirv_to_nir_options**` struct for configuring **ML**-specific capabilities (`**cooperative_matrix**`, `**float16**`, `**int8**`), debugging environment variables (`**RADV_DEBUG**`, `**MESA_SPIRV_DUMP_PATH**`), and the precise sequence of calls by which an **IREE** workload reaches **RADV** or **ANV** and is ultimately dispatched via `**vkCmdDispatch**`. This chapter explains the upstream compilers; Chapters 14 and 15 explain what happens once the **SPIR-V** arrives.

---

## 2. MLIR Architecture for GPU Compilation

MLIR is part of the LLVM project ([source](https://mlir.llvm.org/)) and provides infrastructure for defining and lowering compiler intermediate representations. Its central abstractions are:

- **Operations** (`Op`): the unit of computation. Every instruction, function definition, module, and even the compilation unit itself is an Op. Each Op is defined in a dialect and has a name, a list of operands (values), a list of results (values), a list of regions (for ops that contain code), and a dictionary of attributes.
- **Values**: typed SSA values produced by ops or block arguments. Types are first-class and dialect-defined (e.g., `memref<4x4xf32>`, `vector<8xf16>`, `gpu.async.token`).
- **Regions and Blocks**: ops may contain regions, each containing a sequence of blocks. Blocks hold a sequence of ops and may have arguments, enabling structured control flow.
- **Dialects**: named namespaces that define a coherent set of ops, types, and attributes. The following dialects are central to GPU compilation:

| Dialect | Purpose |
|---------|---------|
| `linalg` | Named structured ops (matmul, conv, reduction) over tensors or memrefs |
| `vector` | SIMD vectors; tile-level computation mapped to hardware registers |
| `arith` | Scalar integer and floating-point arithmetic |
| `memref` | Typed buffer views with layout maps |
| `scf` | Structured control flow (for, if, while loops) |
| `affine` | Affine loop nests and memory access patterns |
| `gpu` | GPU abstractions: modules, kernels, barriers, memory ops |
| `nvgpu` | NVIDIA-specific extensions (tensor core ops, LDMATRIX, TMA) |
| `rocdl` | AMD ROCDL intrinsics |
| `spirv` | SPIR-V module, function, and instruction model |
| `llvm` | LLVM IR within MLIR; final target before `mlir-translate` |

[Source: MLIR Dialect list](https://mlir.llvm.org/docs/Dialects/)

**Progressive lowering.** The fundamental philosophy is that you lower code through a sequence of increasingly concrete dialects rather than in a single jump. A matrix multiply might start as `linalg.matmul` (a high-level named op over tensors), be tiled and vectorised into `vector.contract` ops (tile-level SIMD), then lowered to `gpu.launch` with explicit thread indexing, then lowered further to NVVM or ROCDL intrinsics, and finally emitted as LLVM IR by `mlir-translate`. Each transition is a *lowering pass* (`-convert-linalg-to-loops`, `-convert-vector-to-gpu`, `-convert-gpu-to-nvvm`, `-convert-gpu-to-spirv`). The critical advantage is that optimisation passes at each level can operate on the appropriate abstraction: loop fusion is cheapest at the `linalg` level; vectorisation is done at the `vector` level; warp-level scheduling at the `gpu` level.

**The `mlir-opt` and `mlir-translate` tools** are the standard MLIR command-line driver and translator. `mlir-opt` runs sequences of passes on MLIR textual IR:

```bash
mlir-opt \
  -convert-linalg-to-loops \
  -convert-scf-to-cf \
  -convert-cf-to-llvm \
  -convert-arith-to-llvm \
  input.mlir -o lowered.mlir
```

`mlir-translate` converts between MLIR and external formats. For SPIR-V, the relevant flags registered in `lib/Target/SPIRV/TranslateRegistration.cpp` are:

```bash
# Serialize MLIR SPIR-V dialect to binary
mlir-translate --serialize-spirv input.mlir -o shader.spv

# Deserialize SPIR-V binary back to MLIR
mlir-translate --deserialize-spirv shader.spv -o output.mlir

# Round-trip test
mlir-translate --test-spirv-roundtrip input.mlir
```

[Source: MLIR SPIR-V TranslateRegistration.cpp](https://mlir.llvm.org/doxygen/SPIRV_2TranslateRegistration_8cpp_source.html)

The resulting `shader.spv` is a valid SPIR-V binary that can be passed directly to `vkCreateShaderModule`. This is the bridge between the MLIR world and the Mesa driver world.

---

## 3. The GPU Dialect

The `gpu` dialect ([reference](https://mlir.llvm.org/docs/Dialects/GPU/)) provides a hardware-neutral GPU abstraction layer sitting above vendor-specific dialects. It was designed to be a common lowering target for `linalg` and `vector` before final lowering to NVVM, ROCDL, or SPIR-V.

### 3.1 Core Operations

**`gpu.module`** is a top-level container for GPU-side code, analogous to a CUDA module or a SPIR-V module. It carries a symbol name and may include target attributes that control serialisation.

**`gpu.func`** defines a function executing on the GPU. Kernel functions (callable from host) are tagged with `gpu.kernel` attribute. The function may declare workgroup-memory buffers via memory attribution arguments.

```mlir
gpu.module @my_kernels {
  gpu.func @vector_add(%a: memref<1024xf32>,
                        %b: memref<1024xf32>,
                        %c: memref<1024xf32>)
      kernel {
    %tid = gpu.thread_id x
    %bid = gpu.block_id x
    %bsz = gpu.block_dim x
    %idx = arith.addi %tid, arith.muli %bid, %bsz : index
    %va  = memref.load %a[%idx] : memref<1024xf32>
    %vb  = memref.load %b[%idx] : memref<1024xf32>
    %vc  = arith.addf %va, %vb : f32
    memref.store %vc, %c[%idx] : memref<1024xf32>
    gpu.return
  }
}
```

**`gpu.launch`** is the host-side kernel launch operation, embedding the kernel body inline. It takes grid dimensions (number of blocks) and block dimensions (threads per block) as operands, and the region body receives block IDs and thread IDs as block arguments. This is used when the kernel has not yet been outlined into a separate `gpu.func`.

**`gpu.launch_func`** launches an *already-outlined* kernel by symbol reference, with explicit kernel arguments. This is the form used after outlining:

```mlir
gpu.launch_func @my_kernels::@vector_add
    blocks in (%grid_x, %c1, %c1)
    threads in (%block_x, %c1, %c1)
    args(%a : memref<1024xf32>, %b : memref<1024xf32>, %c : memref<1024xf32>)
```

**Intrinsics:**
- `gpu.thread_id {x|y|z}` — returns the current thread's index within its block along the named dimension
- `gpu.block_id {x|y|z}` — returns the block's index within the grid
- `gpu.block_dim {x|y|z}` — returns the block size
- `gpu.barrier` — thread-block-scope synchronisation barrier (equivalent to CUDA `__syncthreads()` or GLSL `barrier()`)

**Memory operations:**
- `gpu.alloc` — allocates a GPU memory region, optionally with host visibility (`hostShared` flag)
- `gpu.memcpy` — asynchronous or synchronous copy between host and device memrefs

### 3.2 Lowering Targets

From the `gpu` dialect, code is lowered to one of three vendor-specific dialects:

| Target | Pass | Use case |
|--------|------|----------|
| `nvvm` | `-convert-gpu-to-nvvm` | NVIDIA PTX via LLVM NVPTX backend |
| `rocdl` | `-convert-gpu-to-rocdl` | AMD GCN via LLVM AMDGPU backend |
| `spirv` | `-convert-gpu-to-spirv` | Vulkan compute / OpenCL |

The `-convert-gpu-to-spirv` pass (also called `GpuToSpirvPass` in the MLIR C++ API) translates `gpu.func` kernel functions to `spirv.func` functions with appropriate `spirv.ExecutionMode` attributes, maps `gpu.thread_id` to `spirv.BuiltIn.LocalInvocationId`, and maps `gpu.barrier` to `spirv.ControlBarrier`.

The pass `-gpu-module-to-binary` is the modern (post-MLIR 17) approach: it invokes the registered serialiser for each attached target attribute, producing a binary blob rather than requiring an external `mlir-translate` invocation.

### 3.3 The NVVM Dialect

The `nvvm` dialect ([reference](https://mlir.llvm.org/docs/Dialects/NVVMDialect/)) is MLIR's typed representation of **NVIDIA's NVVM IR** — an LLVM IR subset extended with NVIDIA-specific intrinsics. When `-convert-gpu-to-nvvm` runs, every `gpu` dialect op is replaced by its `nvvm` equivalent; the result is valid LLVM IR that the LLVM NVPTX backend (`lib/Target/NVPTX`) can lower to PTX. The full NVIDIA lowering chain is:

```
gpu dialect
  └─ -convert-gpu-to-nvvm
       └─ nvvm dialect (LLVM IR + nvidia intrinsics)
            └─ mlir-translate --mlir-to-llvmir
                 └─ LLVM NVPTX backend
                      └─ PTX assembly
                           └─ ptxas  (proprietary NVIDIA assembler)
                                └─ SASS (binary cubin)
```

**Thread indexing.** The `-convert-gpu-to-nvvm` pass maps each `gpu.*` indexing op to the corresponding NVVM special-register read:

| `gpu` dialect | `nvvm` intrinsic | PTX equivalent |
|---------------|-----------------|----------------|
| `gpu.thread_id x` | `nvvm.read.ptx.sreg.tid.x` | `mov.u32 %r, %tid.x` |
| `gpu.block_id x` | `nvvm.read.ptx.sreg.ctaid.x` | `mov.u32 %r, %ctaid.x` |
| `gpu.block_dim x` | `nvvm.read.ptx.sreg.ntid.x` | `mov.u32 %r, %ntid.x` |
| `gpu.barrier` | `nvvm.barrier0` | `bar.sync 0` |
| `gpu.lane_id` | `nvvm.read.ptx.sreg.laneid` | `mov.u32 %r, %laneid` |

**Warp intrinsics.** The `nvvm` dialect exposes cooperative warp-level primitives that have no equivalent in the hardware-neutral `gpu` dialect:

```mlir
// Warp-level shuffle: broadcast lane 0's value to all lanes
%shuffled = nvvm.shfl.sync bfly %mask, %val, %delta, %c31 : i32

// Warp vote: any thread in warp satisfies predicate
%any = nvvm.vote.any.sync %mask, %pred : i1

// Active mask: bitmask of currently-executing lanes
%active = nvvm.activemask : i32
```

These intrinsics are used directly by Triton's NVIDIA backend after the `ttgpu` → `nvvm` lowering stage; they correspond to PTX `shfl.sync`, `vote.any.sync`, and `activemask` instructions.

**Asynchronous memory and tensor core ops.** For Ampere and later architectures, the `nvvm` dialect includes:

```mlir
// Ampere async copy: shared memory ← global without register staging
nvvm.cp.async.shared.global %dst, %src, 16 : !llvm.ptr<3>, !llvm.ptr<1>
nvvm.cp.async.commit.group
nvvm.cp.async.wait.group 0

// Warp-level matrix multiply-accumulate (pre-Hopper, WMMA API)
// D = A * B + C  for 16x16x16 f16 tiles
%d0, %d1, %d2, %d3, %d4, %d5, %d6, %d7 =
  nvvm.wmma.mma %a0, %a1, %a2, %a3, %a4, %a5, %a6, %a7,
                %c0, %c1, %c2, %c3, %c4, %c5, %c6, %c7
  {layout_a = #nvvm.mma_layout<row>,
   layout_b = #nvvm.mma_layout<col>,
   shape = #nvvm.shape<m = 16, n = 16, k = 16>,
   type_a = f16, type_b = f16, type_d = f32} : ...

// Hopper wgmma (warp-group MMA, replaces WMMA)
nvvm.wgmma.mma_async %desc_a, %desc_b, %accum
  {shape = #nvvm.shape<m = 64, n = 128, k = 16>,
   type_a = f16, type_b = f16, type_d = f32} : ...
```

**NVVM metadata conventions.** NVVM IR uses LLVM metadata to communicate kernel identity and launch constraints to `ptxas`. The `-convert-gpu-to-nvvm` pass attaches `nvvm.annotations` to mark functions as kernels and optionally set maximum thread counts:

```llvm
; Generated LLVM IR (after mlir-translate)
define void @vector_add(ptr %a, ptr %b, ptr %c) {
  ...
}

!nvvm.annotations = !{
  !0,               ; kernel entry point marker
  !1                ; optional maxntidx hint
}
!0 = !{ptr @vector_add, !"kernel", i32 1}
!1 = !{ptr @vector_add, !"maxntidx", i32 256}
```

This metadata is what tells `ptxas` to emit a `.entry` directive rather than a `.func` directive in the PTX output, and to respect any occupancy hints.

**NVVM vs ROCDL.** The AMD equivalent is the `rocdl` dialect (`-convert-gpu-to-rocdl`), which maps to LLVM AMDGPU backend intrinsics. The structural parallel is exact: `nvvm.read.ptx.sreg.tid.x` ↔ `rocdl.workitem.id.x`, `nvvm.barrier0` ↔ `rocdl.barrier`, `nvvm.wmma.*` ↔ `rocdl.mfma.*`. Both dialects are designed to be thin wrappers over their respective LLVM target intrinsics rather than portable abstractions — portability is the `gpu` dialect's job.

**Why Triton uses `nvvm` rather than going through SPIR-V.** Triton's NVIDIA backend targets `nvvm` (not `spirv`) because several performance-critical Triton primitives — warp shuffles, async copies, `wgmma` descriptors — have no standardised SPIR-V equivalent. The SPIR-V path is available (`-convert-gpu-to-spirv`) but loses these low-level primitives and is therefore the fallback for Vulkan compute (§4.4), not the primary NVIDIA path.

**NVCC does not use MLIR.** Despite sharing the name "NVVM", NVIDIA's `nvcc` compiler driver and the MLIR `nvvm` dialect are independent tools that share an IR format but have no pipeline relationship.

`nvcc` splits `.cu` source into host C++ (compiled by the system gcc or clang) and device code, which travels through its own pipeline entirely within LLVM:

```
CUDA C++ device code
    └─ nvcc front-end (CUDA language extensions → LLVM IR)
         └─ libNVVM  (NVIDIA's LLVM-based device compiler library)
              └─ LLVM NVPTX backend
                   └─ PTX assembly
                        └─ ptxas  (NVIDIA proprietary assembler)
                             └─ SASS cubin
```

`libNVVM` is NVIDIA's closed-source device compiler library that accepts NVVM IR (a documented LLVM IR subset) and emits PTX. It predates MLIR by several years and has no MLIR layer inside it. The MLIR `nvvm` dialect is an independent representation that happens to target the same NVVM IR format as its output — but it reaches that output through open MLIR→LLVM→NVPTX lowering rather than through `libNVVM`.

The MLIR-based path (Triton, XLA, IREE) and the `nvcc` path therefore converge at PTX but diverge in every step before it:

| Stage | nvcc path | MLIR path (Triton/XLA) |
|-------|-----------|------------------------|
| Source language | CUDA C++ | Python / StableHLO / JAX |
| Front-end | nvcc CUDA parser | Triton JIT / XLA tracer |
| Mid-level IR | LLVM IR via libNVVM | MLIR `nvvm` dialect |
| Backend | LLVM NVPTX (via libNVVM) | LLVM NVPTX (open) |
| Assembler | ptxas (proprietary) | ptxas (proprietary) |
| Output | SASS cubin | SASS cubin |

Both paths share only the last two steps. For Mesa's purposes — which targets open-source Vulkan/NIR rather than CUDA — neither `nvcc` nor `libNVVM` appears in the stack at all.

[Source: MLIR NVVM dialect docs](https://mlir.llvm.org/docs/Dialects/NVVMDialect/) [Source: LLVM NVPTX target](https://llvm.org/docs/NVPTXUsage.html) [Source: NVVM IR specification](https://docs.nvidia.com/cuda/nvvm-ir-spec/index.html)

### 3.4 IREE's Use of the GPU Dialect

IREE (Section 6) uses the `gpu` dialect as its primary GPU representation after lowering from the `flow` and `stream` dialects. IREE's CodeGen pipeline for Vulkan produces `gpu.launch_func` + `spirv.module` pairs, which are then serialised to SPIR-V and embedded in the compiled `.vmfb` (VM FlatBuffer) binary. [Source: IREE developer overview](https://iree.dev/developers/general/developer-overview/)

---

## 4. OpenAI Triton: Python to GPU Compiler

Triton ([GitHub](https://github.com/openai/triton)) occupies a distinctive niche: it is more abstract than CUDA (no explicit warp-level programming, no manual shared memory indexing) but more concrete than PyTorch (the programmer controls tile sizes, memory access patterns, and loop structure). Its core user-facing interface is a Python function decorated with `@triton.jit`:

```python
import triton
import triton.language as tl

@triton.jit
def add_kernel(x_ptr, y_ptr, out_ptr, n_elements, BLOCK_SIZE: tl.constexpr):
    pid = tl.program_id(axis=0)
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    tl.store(out_ptr + offsets, x + y, mask=mask)
```

[Source: Triton documentation](https://triton-lang.org/)

The `tl` namespace provides the programming model: `tl.load` and `tl.store` operate on *blocks* of memory (not individual elements), `tl.dot` computes a matrix product over blocked tensors (mapping to hardware tensor cores), and `tl.sum`, `tl.max` etc. reduce within a block. The `tl.constexpr` type annotation marks tile sizes as compile-time constants, enabling specialised code generation.

### 4.1 Triton Compiler Pipeline

The Triton compiler is structured as a progressive MLIR lowering pipeline through several dialects [Source: Triton GitHub architecture docs, AMD Triton compilation deep dive](https://medium.com/@nzhangnju/a-deep-dive-into-amd-triton-compilation-912d96e68e45):

```
Python kernel (decorated with @triton.jit)
         │
         ▼
  Triton IR (tt dialect)
  ── tile-level operations: tt.load, tt.store, tt.dot
  ── block pointer arithmetic
         │
         ▼
  TritonGPU IR (ttgpu dialect)
  ── hardware layout annotations added:
     #blocked (thread/warp layout), #shared (scratchpad),
     #nvidia_mma (tensor core layout), #amd_mfma / #amd_wmma
  ── warp-level distribution, shared memory planning,
     software pipelining of matmul loops
         │
         ▼
  LLVM Dialect (within MLIR)
  ── NVVM or ROCDL intrinsics for NVIDIA / AMD
         │
         ▼
  LLVM IR (emitted by mlir-translate)
         │
         ▼
  PTX (NVIDIA) via LLVM NVPTX backend  → ptxas → CUBIN
  AMDGCN (AMD) via LLVM AMDGPU backend → HSACO
```

The TritonGPU dialect is the centrepiece. Its layout annotations encode how a logical tile of data (e.g., a 128×64 matrix) is distributed across warps and lanes. The `#blocked` layout describes a simple strided distribution; `#nvidia_mma` describes the layout produced by NVIDIA Tensor Core `mma.sync` instructions; `#amd_mfma` and `#amd_wmma` describe AMD MFMA (Matrix Fused Multiply-Add on CDNA) and WMMA (RDNA) layouts. Triton's passes coerce loads, stores, and computes to honour these layouts, inserting shared memory transposes where necessary.

The 2025 Gluon dialect was added to Triton to expose lower-level architectural features (warp specialisation, NVIDIA Hopper TMA descriptors, cluster launches) that previously required custom intrinsics hacks.

### 4.2 Auto-tuning

`@triton.autotune` drives the search over compile-time constant configurations:

```python
@triton.autotune(
    configs=[
        triton.Config({'BLOCK_M': 128, 'BLOCK_N': 256, 'BLOCK_K': 64,
                       'num_stages': 3, 'num_warps': 8}),
        triton.Config({'BLOCK_M': 64,  'BLOCK_N': 128, 'BLOCK_K': 32,
                       'num_stages': 4, 'num_warps': 4}),
    ],
    key=['M', 'N', 'K'],
)
@triton.jit
def matmul_kernel(A, B, C, M, N, K,
                  BLOCK_M: tl.constexpr, BLOCK_N: tl.constexpr, BLOCK_K: tl.constexpr,
                  num_stages: tl.constexpr):
    ...
```

When a kernel is first invoked with a new `(M, N, K)` key, all candidate configurations are compiled and benchmarked; the fastest is cached for subsequent calls. Each configuration produces a separate compiled kernel binary because `tl.constexpr` parameters affect the generated code.

### 4.3 Triton on AMD (Triton-ROCm)

The AMD backend is built directly into the Triton repository. After TritonGPU IR, the AMD path lowers to ROCDL intrinsics (for MFMA on CDNA, or WMMA on RDNA3+) rather than NVVM. The LLVM AMDGPU backend then generates AMDGCN assembly, which is assembled via the `make_amdgcn` path into an HSACO ELF binary. [Source: AMD ROCm blogs — Triton on AMD GPUs](https://rocm.blogs.amd.com/artificial-intelligence/triton/README.html)

### 4.4 Triton on Vulkan / SPIR-V

An experimental Vulkan backend for Triton generates SPIR-V rather than PTX or AMDGCN. As of 2025, this backend lacks support for some performance-critical features that hardware vendors expose only through PTX or AMDGCN: asynchronous global-to-shared memory copies (PTX `cp.async`), warp shuffle instructions, and direct TMA descriptor manipulation. The SPIR-V path is therefore used primarily for portability to OpenCL and Vulkan Compute environments where CUDA/ROCm is unavailable. *Note: the Vulkan backend is under active development; check the Triton issue tracker for current status.*

---

## 5. Triton FlashAttention and Kernel Performance

FlashAttention-2 [Dao et al., 2023] is the most widely studied Triton kernel and a compelling demonstration of what the tiling abstraction enables. Its forward pass computes multi-head scaled dot-product attention:

```
O = softmax(Q Kᵀ / √d) V
```

without ever materialising the N×N attention score matrix in HBM (high-bandwidth memory). The Triton implementation iterates over tiles of the sequence dimension, keeping accumulators in registers and shared memory on-chip.

### 5.1 Kernel Structure

```python
@triton.jit
def _fwd_kernel(
    Q, K, V, Out,
    stride_qz, stride_qh, stride_qm, stride_qk,
    # ... strides for K, V, Out
    Z, H, N_CTX,
    BLOCK_M: tl.constexpr, BLOCK_DMODEL: tl.constexpr,
    BLOCK_N: tl.constexpr,
):
    # Each program handles one (batch, head, block_of_queries) tile
    start_m = tl.program_id(0)
    off_hz  = tl.program_id(1)

    # Load Q tile into SRAM (registers)
    q_ptrs = Q + off_hz * stride_qh + (start_m * BLOCK_M + tl.arange(0, BLOCK_M))[:, None] * stride_qm \
               + tl.arange(0, BLOCK_DMODEL)[None, :] * stride_qk
    q = tl.load(q_ptrs)    # shape: [BLOCK_M, BLOCK_DMODEL]

    # Initialise running statistics for online softmax
    m_i = tl.zeros([BLOCK_M], dtype=tl.float32) - float("inf")  # running max
    l_i = tl.zeros([BLOCK_M], dtype=tl.float32)                  # running sum
    acc = tl.zeros([BLOCK_M, BLOCK_DMODEL], dtype=tl.float32)    # output accumulator

    # Iterate over K, V tiles in the sequence dimension
    for start_n in range(0, N_CTX, BLOCK_N):
        k = tl.load(k_ptrs + start_n * stride_kn)  # [BLOCK_DMODEL, BLOCK_N]
        v = tl.load(v_ptrs + start_n * stride_vk)  # [BLOCK_N, BLOCK_DMODEL]

        # Compute attention scores: QKᵀ / sqrt(d)
        qk = tl.dot(q, k)                          # [BLOCK_M, BLOCK_N], uses Tensor Cores
        qk *= softmax_scale

        # Online softmax update
        m_ij  = tl.max(qk, axis=1)                # [BLOCK_M]
        p     = tl.exp(qk - m_ij[:, None])        # [BLOCK_M, BLOCK_N]
        l_ij  = tl.sum(p, axis=1)                 # [BLOCK_M]
        m_i_new = tl.maximum(m_i, m_ij)
        alpha   = tl.exp(m_i - m_i_new)
        beta    = tl.exp(m_ij - m_i_new)
        l_i   = alpha * l_i + beta * l_ij
        m_i   = m_i_new
        # Rescale accumulator and add new contribution
        acc   = alpha[:, None] * acc + tl.dot(p.to(tl.float16), v)

    # Final normalisation
    acc = acc / l_i[:, None]
    tl.store(out_ptrs, acc)
```

[Source: FlashAttention-2 Triton implementation walkthrough](https://nathanchen.me/public/Triton-Flash-Attention-Kernel-Walkthrough.html)

The kernel's performance advantage comes from three sources. First, the N×N score matrix is never materialised: each thread block computes a tile of the output by streaming K and V through shared memory. Second, `tl.dot` maps to Tensor Cores (NVIDIA) or MFMA (AMD), achieving the peak FLOP/s the hardware provides. Third, the online softmax — maintaining a running maximum and sum across tiles — avoids a second pass over the scores, reducing HBM reads by roughly 2×. The result is a kernel whose runtime is bounded by HBM bandwidth rather than by FLOP throughput, which is what theoretical roofline analysis predicts.

### 5.2 Block Size Tuning

The `BLOCK_M`, `BLOCK_N`, `BLOCK_DMODEL` constants must fit the data in the GPU's shared memory / L2. On an A100 (108 KB L1 per SM), a common configuration for FlashAttention-2 is `BLOCK_M=128, BLOCK_N=64, BLOCK_DMODEL=64`, consuming approximately 128×64×2 + 64×64×2 = 24 KB for FP16 operands. On an AMD MI300X, different MFMA tile sizes (16×16, 32×32, 16×4 for FP8) suggest different `BLOCK_M`, `BLOCK_N` configurations, which is where `@triton.autotune` earns its place.

### 5.3 Adoption

FlashAttention-2 Triton kernels are used in vLLM (a high-throughput LLM serving engine), llama.cpp's CUDA path, and the reference transformers library. The performance improvement over a naive PyTorch attention implementation on an H100 is typically 2–4× depending on sequence length and batch size, because the naive version reads and writes the full N×N score matrix from HBM.

### 5.4 Triton on Vulkan: Current Gaps

The Vulkan SPIR-V path currently lacks:
- **Asynchronous copies** (PTX `cp.async.bulk` / CUDA TMA): SPIR-V has no equivalent instruction; the `VK_EXT_shader_tile_image` extension helps for raster but not compute async prefetch.
- **Warp shuffle** (`tl.inline_asm_elementwise` using PTX shuffle can be used on CUDA, but there is no portable SPIR-V equivalent with the same performance characteristics).
- **TMA descriptors** (NVIDIA Hopper-specific hardware scatter/gather).

These gaps make Vulkan-Triton a lower-performance option compared to CUDA or ROCm Triton for attention-class kernels as of 2026.

---

## 6. IREE: MLIR-Based Inference for Vulkan

IREE (Intermediate Representation Execution Environment) is Google's production MLIR compiler and runtime for neural network inference, designed to target heterogeneous hardware from a single compilation pipeline. Its distinguishing feature relative to TensorRT or ROCm MIOpen is that it targets *multiple backends* from the same MLIR IR, with Vulkan SPIR-V as its primary open portable backend. [Reference: https://iree.dev/]

### 6.1 Compilation Pipeline

```
Framework model (PyTorch / TFLite / ONNX)
         │  (import tools: iree-import-tflite, torch-mlir, stablehlo)
         ▼
StableHLO / MHLO / linalg on tensors
         │  (iree-opt: input pipeline)
         ▼
IREE Flow dialect
  ── identifies dispatch regions (compute workloads)
  ── outlines workloads into executable function bodies
         │  (flow-to-stream lowering)
         ▼
IREE Stream dialect
  ── explicit asynchronous scheduling
  ── memory placement (host / device)
  ── concurrency extraction
         │  (stream-to-hal lowering)
         ▼
IREE HAL dialect
  ── target-independent command buffer operations
  ── hal.command_buffer.dispatch, hal.buffer_view
         │  (hal-to-spirv / hal-to-ptx / hal-to-llvm)
         ▼
Target binary (SPIR-V / PTX / LLVM CPU)
  embedded in .vmfb (VM FlatBuffer)
```

[Source: IREE Stream dialect reference](https://iree.dev/reference/mlir-dialects/Stream/), [HAL dialect reference](https://iree.dev/reference/mlir-dialects/HAL/)

### 6.2 The `iree-compile` Tool

The primary compilation command for a Vulkan SPIR-V target:

```bash
# Using the newer device-flag form (recommended as of IREE 3.x)
iree-compile \
  --iree-hal-target-device=vulkan \
  --iree-vulkan-target=rdna3 \
  --iree-opt-level=O3 \
  model.mlir -o model.vmfb

# Equivalent legacy form
iree-compile \
  --iree-hal-target-backends=vulkan-spirv \
  --iree-vulkan-target-triple=rdna3-unknown-linux \
  model.mlir -o model.vmfb
```

The `--iree-vulkan-target` flag controls which Vulkan extensions and SPIR-V capabilities are emitted. On RDNA3 this enables `VK_KHR_cooperative_matrix` and 16-bit storage. The compiled output is a `.vmfb` — a FlatBuffer file containing the SPIR-V shader modules, I/O layout metadata, and a bytecode VM program.

To list available backends:

```bash
iree-compile --iree-hal-list-target-backends
```

### 6.3 HAL Runtime Architecture

At runtime, IREE exposes a C API:

```c
// Create a Vulkan HAL device
iree_hal_driver_t* driver;
iree_hal_driver_registry_try_create(
    iree_make_cstring_view("vulkan"),
    iree_allocator_system(), &driver);

iree_hal_device_t* device;
iree_hal_driver_create_default_device(
    driver, iree_allocator_system(), &device);

// Load compiled module
iree_runtime_session_t* session;
iree_runtime_session_create_with_device(
    instance, &session_options, device,
    iree_allocator_system(), &session);

iree_runtime_session_append_bytecode_module_from_file(
    session, "model.vmfb");
```

Internally, the Vulkan HAL backend wraps `vkCreateComputePipeline` with the embedded SPIR-V modules and issues `vkCmdDispatch` calls driven by the HAL command buffer. From Mesa's perspective, a IREE Vulkan workload is indistinguishable from any other application submitting SPIR-V compute shaders.

### 6.4 Android Deployment

IREE's Vulkan path is used for on-device inference on Android because Vulkan is the only universally-available GPU compute API on Android (OpenCL is optional, NNAPI is limited, and ROCm/CUDA are vendor-specific). The `.vmfb` is deployed as an asset; the runtime links against `libvulkan.so`. [Source: IREE GPU Vulkan guide](https://iree.dev/guides/deployment-configurations/gpu-vulkan/)

---

## 7. XLA: Accelerated Linear Algebra

XLA is the deep learning compiler originally developed by Google for TPUs and subsequently extended to GPU and CPU [Reference: https://openxla.org/xla]. It is the compilation back-end for JAX (`jax.jit`), TensorFlow (via `tf.function`), and PyTorch XLA.

### 7.0 OpenXLA, HLO, MHLO, StableHLO, CHLO, and IREE: The Ecosystem Map

These five names refer to distinct layers of a single compilation ecosystem. Understanding how they relate is prerequisite to reading any of the following sections clearly.

**OpenXLA** is the open-source project umbrella announced by Google in October 2022, hosted at [openxla.org](https://openxla.org). It governs three formerly-separate projects as sibling sub-projects under one governance roof:

| Sub-project | Repository | Role |
|-------------|-----------|------|
| XLA | `github.com/openxla/xla` | The compiler: graph optimisation + code generation |
| StableHLO | `github.com/openxla/stablehlo` | The stable IR interchange format |
| IREE | `github.com/openxla/iree` | Alternative runtime + codegen for inference |

Before OpenXLA, IREE was a separate Google research project. Moving it under the same umbrella formalised StableHLO as the handoff contract between XLA-style frontends and IREE's codegen pipeline.

**The HLO lineage** evolved in three stages:

```
HLO  (2016–present)
 ├─ XLA's original, monolithic C++ IR
 ├─ HloInstruction / HloComputation / HloModule objects
 ├─ Not MLIR-based; no textual dialect; no serialization guarantee
 └─ Still the internal IR XLA optimises (fusion, layout, sharding)

MHLO  (2019–present, internal)
 ├─ MLIR-dialect translation of HLO ops ("Meta HLO" / "ML HLO")
 ├─ Lives in the mlir-hlo repo (github.com/tensorflow/mlir-hlo)
 ├─ Enables MLIR passes (canonicalisation, CSE) on HLO graphs
 ├─ Not stable: op signatures change with XLA internals
 └─ Bridge dialect — not a public API

StableHLO  (2022–present, public)
 ├─ Stable, versioned subset of MHLO (~105 ops as of v1.0)
 ├─ Backward and forward compatibility guarantees across versions
 ├─ Designed as the public serialization format for ML models
 ├─ Producer: JAX, TF, PyTorch XLA (via jax.export / tf.SavedModel)
 └─ Consumer: XLA itself, IREE, third-party runtimes
```

**CHLO** (Client HLO, `github.com/tensorflow/mlir-hlo/tree/master/mhlo/transforms/chlo`) sits one level *above* StableHLO. It contains higher-level ops corresponding to client-facing math operations — complex numbers, special functions (`chlo.bessel_i1e`, `chlo.erf`), broadcasting semantics — that do not map 1:1 to hardware primitives. The CHLO→StableHLO lowering (`-chlo-legalize-to-hlo`) decomposes these composite ops into the ~105 primitive StableHLO ops before serialisation.

**The handoff contract.** StableHLO's design rationale is the separation of concerns between XLA as a *training-optimised compiler* and IREE (or any third party) as an *inference-optimised runtime*:

```
JAX / TF / PyTorch XLA
    │
    │  [trace → jaxpr / TF graph → MHLO → StableHLO]
    ▼
StableHLO  ←── the stable public boundary ───────────────┐
    │                                                     │
    ├─ XLA path                        ├─ IREE path       │
    │  StableHLO → MHLO → HLO          │  StableHLO →     │
    │  → fusion/layout/sharding         │  Flow → Stream → │
    │  → cuBLAS/Triton/LLVM IR          │  HAL → SPIR-V /  │
    │  → PTX / AMDGCN / XLA CPU         │  PTX / CPU       │
    │                                   │                  │
    └───── both are OpenXLA siblings ───┘                  │
                                                           │
    Third-party runtimes (ORT, TFLite, custom) ────────────┘
    can also consume StableHLO without tracking XLA internals
```

This separation means an inference runtime can be validated against a frozen StableHLO model artifact without tracking XLA's internal MHLO evolution. The `jax.export` API (JAX ≥ 0.4.27) serialises a JIT-compiled function directly to a StableHLO `.mlirbc` (MLIR bytecode) file, which IREE can then compile independently. [Source: JAX export docs](https://jax.readthedocs.io/en/latest/export/export.html) [Source: OpenXLA StableHLO compatibility](https://github.com/openxla/stablehlo/blob/main/docs/compatibility.md)

**Summary table:**

| Name | Layer | MLIR? | Stable? | Who produces | Who consumes |
|------|-------|--------|---------|--------------|--------------|
| HLO | XLA internal graph IR | No | No | XLA tracing | XLA codegen |
| CHLO | Client ops above StableHLO | Yes | No | JAX/TF frontend | CHLO→HLO lowering |
| MHLO | MLIR dialect of HLO | Yes | No | XLA internals | XLA MLIR passes |
| StableHLO | Public interchange IR | Yes | **Yes** | JAX, TF, PyTorch XLA | XLA, IREE, 3rd parties |
| OpenXLA | Project umbrella | — | — | Google + community | — |
| IREE | Runtime + codegen | Yes (via dialects) | — | StableHLO input | SPIR-V / PTX / CPU |

[Source: OpenXLA announcement](https://opensource.googleblog.com/2022/11/introducing-openxla-project.html) [Source: StableHLO spec](https://github.com/openxla/stablehlo/blob/main/docs/spec.md) [Source: MHLO → StableHLO migration](https://github.com/openxla/stablehlo/blob/main/docs/status.md)

### 7.1 HLO and Compilation Pipeline

The XLA GPU backend pipeline is:

```
JAX Python (jax.jit decorated function)
     │  [tracing → jaxpr → StableHLO]
     ▼
StableHLO
     │  [StableHLO → HLO lowering]
     ▼
HLO graph
     │  [HLO optimisation passes]
     │   ├─ SPMD partitioner (multi-device sharding)
     │   ├─ Layout assignment (row/column-major)
     │   └─ Fusion (producer-consumer grouping)
     ▼
Code generation (per fused cluster)
     ├─ Library calls: cuBLAS, cuDNN, NCCL for NVIDIA
     │                 rocBLAS, MIOpen, RCCL for AMD
     ├─ Direct LLVM IR emission: reductions, transposes, elementwise
     └─ Triton emission: matmul-heavy fusions, softmax
          │
          ▼
         PTX / AMDGCN via LLVM, then ptxas / LLVM assembler
```

[Source: XLA GPU Architecture Overview](https://openxla.org/xla/gpu_architecture)

**Fusion** is XLA's most important optimisation. A producer-consumer pair such as `y = relu(matmul(A, B))` is fused into a single GPU kernel, preventing the `matmul` output from being written to HBM and immediately read back for the `relu`. XLA's fusion algorithm identifies such chains and produces a single HLO *fusion cluster* whose body is then compiled as one kernel.

**Triton Integration.** Since 2023, XLA:GPU uses Triton as an alternative code-generation path for matmul-heavy fusions. The HLO fusion is converted to Triton IR (the `tt` dialect), and the Triton compiler then performs tiling and layout assignment for tensor cores. This gives XLA:GPU access to Triton's auto-tuned matmul performance without duplicating Triton's tile scheduling logic inside XLA.

### 7.2 JAX Entry Points

```python
import jax
import jax.numpy as jnp

@jax.jit
def matmul(A, B):
    return jnp.dot(A, B)

# First call triggers XLA compilation
C = matmul(jnp.ones((1024, 1024)), jnp.ones((1024, 1024)))

# Ahead-of-time compilation
lowered = jax.jit(matmul).lower(
    jax.ShapeDtypeStruct((1024, 1024), jnp.float32),
    jax.ShapeDtypeStruct((1024, 1024), jnp.float32)
)
compiled = lowered.compile()
```

### 7.3 XLA on AMD

`jax[rocm]` packages XLA with the ROCm backend. HLO clusters are lowered to AMDGCN through either the ROCm library path (rocBLAS, MIOpen for standard ops) or direct LLVM AMDGPU emission for custom fusions. [Source: JAX ROCm support page](https://jax.readthedocs.io/en/latest/installation.html#amd-gpu-rocm)

### 7.4 XLA Vulkan Backend

An experimental Vulkan backend for XLA is under development via IREE integration: XLA lowers HLO to StableHLO, which IREE then ingests and compiles to SPIR-V. This path provides a fully open, vendor-neutral inference route. As of 2025–2026, it is not yet production-ready for all model architectures. *Note: needs verification for current production status.*

---

## 8. SPIR-V as the Cross-Compiler Target

SPIR-V ([Khronos Registry](https://registry.khronos.org/SPIR-V/)) serves as the interchange format between the high-level GPU compiler world (MLIR, Triton Vulkan, glslangValidator, DXC) and the driver world (Mesa, Vulkan ICD drivers). Its design makes it well-suited for this role: it is a binary format that can be validated offline (`spirv-val`), optimised with a standalone tool (`spirv-opt`), and disassembled to human-readable text (`spirv-dis`).

### 8.1 Binary Format

A SPIR-V binary is a sequence of 32-bit words. The first five words form the header:

```
Word 0: Magic number — 0x07230203
Word 1: Version    — e.g., 0x00010600 for SPIR-V 1.6
Word 2: Generator  — tool-specific magic (e.g., 0x000D0000 for glslang)
Word 3: Bound      — one more than the largest result ID used
Word 4: Reserved (schema) — currently 0
```

Following the header, instructions are encoded as `[length | opcode]` in the first word, followed by `length-1` words of operands. Each result-producing instruction has a result-type ID and a result ID as its first two operands.

[Source: SPIR-V Specification §2.3](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html)

### 8.2 Module Structure

A well-formed Vulkan SPIR-V module follows a strict ordering of instruction kinds:

```spirv
; Capabilities
OpCapability Shader
OpCapability CooperativeMatrixKHR

; Extension declarations
OpExtension "SPV_KHR_cooperative_matrix"

; Extended instruction imports (if using GLSL.std.450)
%glsl = OpExtInstImport "GLSL.std.450"

; Memory and addressing model
OpMemoryModel Logical GLSL450

; Entry points
OpEntryPoint GLCompute %main "main" %gl_GlobalInvocationID

; Execution modes
OpExecutionMode %main LocalSize 256 1 1

; Debug information (OpSource, OpName, OpMemberName)
; Decoration instructions (OpDecorate, OpMemberDecorate)
; Type declarations (OpTypeVoid, OpTypeInt, OpTypePointer, ...)
; Constant declarations
; Global variable declarations
; Function bodies
```

### 8.3 SPIR-V Tools

The [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools) project provides the standard toolchain:

```bash
# Validate a SPIR-V binary against Vulkan 1.3 semantics
spirv-val --target-env vulkan1.3 shader.spv

# Optimise: dead code elimination, constant folding, loop unrolling
spirv-opt -O shader.spv -o shader_opt.spv

# Disassemble to text
spirv-dis shader.spv -o shader.spvasm

# Assemble from text
spirv-as shader.spvasm -o shader.spv
```

`spirv-opt` is important for IREE and MLIR outputs: auto-generated SPIR-V tends to have redundant type declarations, unnecessary `OpUndef` references, and unoptimised constant expressions. A spirv-opt `-O` pass typically reduces module size by 10–30% and can expose peephole improvements to the driver's own compiler.

### 8.4 spirv-cross: Reverse Translation

`spirv-cross` ([GitHub](https://github.com/KhronosGroup/SPIRV-Cross)) reverse-translates SPIR-V to GLSL, HLSL, or MSL:

```bash
spirv-cross --glsl shader.spv --output shader_recovered.glsl
spirv-cross --hlsl shader.spv --output shader.hlsl
spirv-cross --msl  shader.spv --output shader.metal
```

This is used by Proton and DXVK to verify shaders during debugging, and by MoltenVK (which uses spirv-cross to produce Metal shaders from Vulkan SPIR-V on macOS).

### 8.5 MLIR → SPIR-V → Mesa Path

The complete translation chain from MLIR to Mesa:

```
MLIR SPIR-V dialect
  │   (mlir-translate --serialize-spirv)
  ▼
SPIR-V binary (.spv)
  │   (vkCreateShaderModule)
  ▼
VkShaderModule (driver-internal handle)
  │   (Mesa spirv_to_nir)
  ▼
NIR shader (Chapter 14)
  │   (Mesa NIR passes, driver lowering)
  ▼
ACO or LLVM backend (Chapter 15)
  │
  ▼
GPU ISA binary
```

The key interface in Mesa is `spirv_to_nir()` in `src/compiler/spirv/spirv_to_nir.c`. *Note: There is no Mesa function named `nir_from_spirv`; the correct entry point is `spirv_to_nir`.* The function accepts a SPIR-V word array, its length, an entry point name, an execution model, and a `spirv_to_nir_options` struct, and returns a `nir_shader*`. [Source: Mesa cgit spirv_to_nir.c](https://cgit.freedesktop.org/mesa/mesa/tree/src/compiler/spirv/spirv_to_nir.c)

---

## 9. Cooperative Matrix and Tensor Core Access

Machine-learning workloads require matrix-multiply operations in hardware: NVIDIA calls these Tensor Cores, AMD calls them WMMA (on RDNA3) or MFMA (on CDNA), Intel calls them XMX (Xe Matrix Extensions). Vulkan exposes these through the `VK_KHR_cooperative_matrix` extension; SPIR-V exposes them through `SPV_KHR_cooperative_matrix`.

### 9.1 Vulkan Extension

`VK_KHR_cooperative_matrix` ([Vulkan Docs](https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_cooperative_matrix.html)) was promoted to Vulkan 1.3 in 2023. The extension works by exposing *cooperative matrix types* that are implicitly distributed across the invocations of a subgroup. Each invocation holds a fragment of the matrix.

To query supported tile sizes and data types:

```cpp
uint32_t count;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(physDevice, &count, nullptr);
std::vector<VkCooperativeMatrixPropertiesKHR> props(count);
for (auto& p : props) p.sType = VK_STRUCTURE_TYPE_COOPERATIVE_MATRIX_PROPERTIES_KHR;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(physDevice, &count, props.data());

for (const auto& p : props) {
    // p.MSize, p.NSize, p.KSize: tile dimensions
    // p.AType, p.BType, p.CType, p.ResultType: VkComponentTypeKHR
    // p.scope: VkScopeKHR (typically Subgroup)
}
```

Common tile configurations reported by RDNA3 hardware: 16×16×16, 32×16×16 for FP16×FP16→FP32 and INT8×INT8→INT32. ANV on Intel DG2 (Xe) reports 8×16×16 and 16×16×16 for FP16 operations.

### 9.2 SPIR-V: SPV_KHR_cooperative_matrix

In SPIR-V, the cooperative matrix type and operations are:

```spirv
OpCapability CooperativeMatrixKHR
OpExtension  "SPV_KHR_cooperative_matrix"

; Declare a 16×16 FP16 A-matrix (Use=MatrixAKHR, scope=Subgroup)
%fp16       = OpTypeFloat 16
%coopmat_A  = OpTypeCooperativeMatrixKHR %fp16 %subgroup %const_16 %const_16 %matA

; Load from memory
%mat_a = OpCooperativeMatrixLoadKHR %coopmat_A %ptr %stride %colMajor

; Multiply-accumulate: D = A × B + C
%result = OpCooperativeMatrixMulAddKHR %coopmat_D %mat_a %mat_b %mat_c \
              MatrixASignedComponentsKHR|...
```

[Source: SPV_KHR_cooperative_matrix registry](https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_cooperative_matrix.html)

### 9.3 Mesa RADV: Mapping to RDNA WMMA

RADV added `VK_KHR_cooperative_matrix` support for RDNA3 and newer, mapping `OpCooperativeMatrixMulAddKHR` to RDNA's WMMA (Wave Matrix Multiply-Accumulate) instructions via NIR. RDNA2 does not have WMMA hardware and therefore does not support the extension. [Source: RADV cooperative matrix merge — Phoronix](https://www.phoronix.com/news/RADV-VK_KHR_cooperative_matrix)

The lowering path in Mesa is: SPIR-V `OpCooperativeMatrixMulAddKHR` → NIR `nir_intrinsic_cooperative_matrix_mul_add` → RADV's ACO or LLVM backend emitting WMMA instructions.

### 9.4 Mesa ANV: Mapping to Intel XMX

Intel ANV implemented `VK_KHR_cooperative_matrix` for Gfx12.5+ (DG2/Alchemist) using DPAS (Dot Product Accumulate Systolic) instructions, which are the hardware realisation of XMX. On future Xe2+ hardware the XMX units are larger and can support additional data types (FP8). [Source: Intel ANV cooperative matrix — Phoronix](https://www.phoronix.com/news/Intel-ANV-Cooperative-Matrix)

### 9.5 Triton's tl.dot Lowering

When Triton lowers `tl.dot(a, b)` on NVIDIA hardware, the TritonGPU backend assigns the `#nvidia_mma` layout to the result tensor and emits `nvgpu.wmma.matrix` ops (or the newer `nvgpu.mma.sync` form for Ampere+) that map to CUDA's `mma.sync.aligned.*` PTX instruction family — the same Tensor Core access point. On AMD, `tl.dot` with the `#amd_wmma` layout emits ROCDL WMMA intrinsics targeting RDNA3's WMMA instructions.

### 9.6 Use in Attention Mechanisms

Cooperative matrices are critical for efficient attention computation. The QK^T matmul in FlashAttention — a 128×64 × 64×128 multiplication with FP16 inputs and FP32 accumulation — maps directly onto cooperative matrix tile operations. Each subgroup (warp) computes one 16×16 tile of the output; multiple subgroups collaborate to produce a full BLOCK_M × BLOCK_N tile. This is how Triton's `tl.dot` and SPIR-V's `OpCooperativeMatrixMulAddKHR` both map to the same underlying hardware capability.

---

## 10. The Mesa Connection: SPIR-V to NIR to ISA

This section traces the complete path from MLIR output to executed GPU instructions, integrating the material from earlier sections with Mesa's internals.

### 10.1 The Full Path

```
MLIR SPIR-V dialect IR
       │
       │  mlir-translate --serialize-spirv
       ▼
SPIR-V binary (uint32_t words[])
       │
       │  Application calls vkCreateShaderModule(device, &createInfo, ...)
       │  createInfo.pCode = words; createInfo.codeSize = wordCount * 4;
       ▼
VkShaderModule (driver-held handle)
       │
       │  Application calls vkCreateComputePipeline or vkCreateGraphicsPipelines
       │  Mesa dispatches to RADV or ANV pipeline creation
       ▼
spirv_to_nir() [src/compiler/spirv/spirv_to_nir.c]
       │  Parameters: words, word_count, entry_point_name,
       │              stage (MESA_SHADER_COMPUTE, etc.),
       │              spirv_to_nir_options*
       │  Returns: nir_shader*
       ▼
nir_shader* (NIR — Chapter 14)
       │
       │  Driver NIR passes: nir_lower_io, nir_lower_alu, nir_opt_gcm, ...
       │  Cooperative matrix lowering in NIR
       ▼
ACO compiler (Chapter 15) or LLVM backend
       │
       ▼
GPU ISA binary (submitted via AMDGPU or i915 kernel driver)
```

### 10.2 spirv_to_nir Options for ML Workloads

ML compilers emit SPIR-V with specific capability sets. The `spirv_to_nir_options` struct lets drivers configure how to handle these:

```c
struct spirv_to_nir_options options = {
   .environment = NIR_SPIRV_VULKAN,
   .caps = {
      .cooperative_matrix        = true,  /* VK_KHR_cooperative_matrix */
      .float16                   = true,
      .int8                      = true,
      .storage_buffer_storage_class = true,
      .variable_pointers         = true,
   },
   .ubo_addr_format    = nir_address_format_vec2_index_32bit_offset,
   .ssbo_addr_format   = nir_address_format_vec2_index_32bit_offset,
};

nir_shader *nir = spirv_to_nir(
    spirv_words, spirv_word_count,
    NULL, 0,              /* specialisation constants */
    MESA_SHADER_COMPUTE,
    "main",
    &options,
    &nir_options
);
```

The cooperative matrix capability tells the SPIR-V reader to translate `OpTypeCooperativeMatrixKHR` and `OpCooperativeMatrixMulAddKHR` into NIR intrinsics rather than failing with an unsupported-capability error.

### 10.3 Debugging the Path

Mesa provides environment variables to inspect each stage of the SPIR-V → NIR → ISA pipeline for RADV:

```bash
# Dump the SPIR-V binary (write to MESA_SPIRV_DUMP_PATH)
MESA_SPIRV_DUMP_PATH=/tmp/shaders \
  RADV_DEBUG=spirv \
  ./my_vulkan_app

# Dump NIR after spirv_to_nir and after Mesa NIR passes
RADV_DEBUG=nir ./my_vulkan_app

# Dump backend IR (ACO IR or LLVM IR before ISA emission)
RADV_DEBUG=ir ./my_vulkan_app

# Dump final shader disassembly (RDNA assembly)
RADV_DEBUG=asm ./my_vulkan_app

# Multiple flags combined
RADV_DEBUG=spirv,nir,ir,asm ./my_vulkan_app
```

[Source: Mesa environment variables documentation](https://docs.mesa3d.org/envvars.html)

For fine-grained control of which shader stages to dump, add stage flags: `RADV_DEBUG=nir,cs` dumps only compute shaders' NIR.

For SPIR-V validation before Mesa sees the shader:

```bash
# Validate offline
spirv-val --target-env vulkan1.3 shader.spv

# Validate at driver entry (catches spec violations in generated SPIR-V)
VK_LAYER_ENABLES=VK_EXT_validation_features \
  VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation \
  ./my_vulkan_app
```

### 10.4 How IREE's Vulkan Path Feeds Mesa

When an IREE workload runs on a Linux system with RADV or ANV, the interaction is:

1. IREE's compiler has produced SPIR-V modules during offline compilation and embedded them in the `.vmfb`.
2. At runtime, IREE's Vulkan HAL calls `vkCreateComputePipeline` with the embedded SPIR-V.
3. Mesa's RADV or ANV pipeline creation calls `spirv_to_nir`, runs the NIR pipeline (which includes cooperative matrix lowering for ML shaders), and emits ISA via ACO or the Intel compiler backend.
4. IREE's HAL records `hal.command_buffer.dispatch` operations that become `vkCmdDispatch` calls submitted to the AMD or Intel GPU kernel driver.

From Mesa's perspective, IREE is just another Vulkan application. The SPIR-V it submits is held to exactly the same validation constraints as any other Vulkan shader, which is why IREE's offline compilation pipeline runs `spirv-val` and the Vulkan validation layers during testing.

---

## 11. Why MLIR Stops at SPIR-V: The ISA Boundary

MLIR is a meta-compiler: infrastructure for building compilers, not a compiler itself. Its dialects model GPU computation at the level of tiles, vector operations, thread hierarchy, and memory abstractions. None of those concepts have a direct mapping to machine instructions — the gap between `gpu.thread_id` and a SASS `S2R SR_TID.X` instruction is not one lowering pass but a chain of ISA-specific decisions about register class, instruction encoding width, scheduling control bits, and occupancy constraints. This section explains precisely where MLIR's responsibility ends, why the handoff occurs at SPIR-V, and why the ISA-level compilers below that boundary (NAK for NVIDIA, ACO for AMD, the Intel compiler for Xe) are structured the way they are rather than being implemented as MLIR backends.

### The Three-Layer Model

GPU compilation below the ML framework divides into three distinct concerns:

```
Layer 1: Tile / tensor abstraction     ← MLIR's domain
  (linalg, vector, gpu dialects)
         │
         │  progressive lowering
         ▼
Layer 2: Portable instruction set      ← SPIR-V's domain
  (typed SSA, explicit control flow,
   standardised intrinsics, no hardware assumptions)
         │
         │  spirv_to_nir() / driver compilation
         ▼
Layer 3: ISA-level compilation         ← NAK / ACO / Intel compiler's domain
  (register allocation, instruction
   scheduling, binary encoding, occupancy)
```

SPIR-V sits at the contract boundary between layers 2 and 3. It is ISA-neutral by construction: it has no concept of register banks, warp occupancy, instruction latency, or dual-issue. Everything below SPIR-V requires hardware-specific knowledge that is deliberately excluded from the SPIR-V specification.

### Why MLIR Cannot Reach the ISA Directly

**Occupancy-aware register allocation.** GPU register allocation is not a pure minimization problem. The NVIDIA SM allocates registers to *all warps in a thread block simultaneously* — if a shader uses 33 GPRs, the SM may schedule 50% fewer warps than if the shader used 32 GPRs, destroying parallelism that hides memory latency. The allocator must model this occupancy cliff. MLIR has no representation for "this ISA allocates N warps per SM as a function of register count" — that information lives in hardware documentation (or reverse-engineered tables) specific to each SM generation. NAK's allocator encodes this per-SM-generation as a lookup table; MLIR has no equivalent abstraction point.

**Instruction encoding width changes across generations.** NVIDIA SASS changed from 64-bit instruction words (Maxwell, Pascal) to 128-bit words (Volta, Turing, Ampere, Ada, Blackwell). The 64-bit extra word in 128-bit encoding is scheduling control: stall counts, yield bits, and read/write barrier indices. These control bits are what the instruction scheduler produces. There is no MLIR dialect that models stall counts or barrier indices — they are intrinsically tied to the specific pipeline latency model of a single GPU microarchitecture. An MLIR backend targeting NVIDIA would need to re-implement NAK's scheduler in terms of MLIR's infrastructure, gaining nothing.

**Uniform register files.** NVIDIA Turing (sm75) introduced a second register file: UR0–UR62, 63 scalar registers that hold values provably uniform across a warp. Accessing a constant buffer via a UGPR instruction costs one cycle; via a GPR plus `LDSM` costs many. NAK's compiler uses divergence analysis to promote values to UGPR; this is a GPU-microarchitecture-specific optimization with no MLIR analogue. The `nvgpu` dialect has warp-level MMA operations but no UGPR concept.

**ARM Valhall and non-NVIDIA architectures.** KRAID (Chapter 118) compiles to ARM Mali Valhall ISA. Valhall has native 16-bit arithmetic paths, a different memory model, and no concept analogous to UGPR. A single MLIR GPU dialect lowering to "GPU ISA" would require per-vendor forks at every decision point that involves ISA-specific concepts — which is to say, at nearly every decision below register pressure estimation. This is precisely the problem SPIR-V solves: it standardises the language at a level above ISA so each vendor's compiler can handle ISA-specific lowering independently.

### Why LLVM NVPTX Is Not the Answer

LLVM has `lib/Target/NVPTX`, which compiles LLVM IR to PTX. This is the path Triton uses for its NVIDIA backend (via the `nvvm` dialect → LLVM IR → PTX). PTX is then compiled to SASS by `ptxas` — NVIDIA's proprietary assembler. The LLVM/PTX/ptxas path has three structural problems for an open-source Linux driver:

1. **`ptxas` is proprietary.** It ships in the CUDA toolkit. An open-source NVIDIA driver using this path has a mandatory proprietary dependency at the terminal compilation step. NAK exists precisely to eliminate `ptxas`: it emits SASS directly, making the entire path from Vulkan shader to GPU binary open-source.

2. **ptxas controls occupancy.** The register allocation and scheduling decisions that determine occupancy are made inside `ptxas`, not in LLVM. An open-source driver using LLVM+NVPTX cedes control of the most latency-sensitive decisions to a component it cannot inspect or modify.

3. **Compile latency.** LLVM's pass pipeline has fixed overhead that is disproportionate for short shaders. ACO was built because LLVM AMDGPU was 3–4× slower than ACO at shader compile time for the per-draw compilation pattern GPU drivers use. The same argument applies to LLVM NVPTX for NAK's use case.

### NIR as MLIR's Peer at the Driver Layer

NIR (Chapter 14) and MLIR's `gpu` dialect occupy analogous positions in their respective layers — both are hardware-neutral SSA IRs with explicit control flow and typed values — but they serve different masters. The `gpu` dialect models *launch semantics* (thread blocks, grid dimensions, workgroup memory) at the framework layer. NIR models *lowered shader semantics* (I/O variables already resolved to `load_input`/`store_output`, memory access already in terms of SSBO offsets, cooperative matrix ops already expressed as NIR intrinsics). NIR sits closer to the ISA; the `gpu` dialect sits closer to the algorithm.

Crucially, NIR is much smaller. Its core type system is fixed-width integer and float scalars and small vectors. It has no tile abstraction, no layout annotation, no `affine_map`. This narrowness is intentional: the ISA-level compiler needs a precisely defined, small-surface IR so that every pass has a clear precondition and postcondition. MLIR's extensibility — the property that makes it useful as a meta-compiler — would be a liability inside NAK, where the register allocator must have complete knowledge of every value in the function.

```
MLIR gpu dialect     NIR
─────────────────    ─────────────────────────────
Tile/vector ops      Scalar SSA values (32-bit)
Thread hierarchy     Workgroup ID intrinsics
Layout attributes    I/O already lowered to load/store
Extensible types     Fixed int/float types only
Framework-facing     ISA-facing
```

### The Long-Term Convergence Direction

The Roadmap (below) notes a long-term goal: MLIR as a direct Mesa input, bypassing the SPIR-V serialization/deserialization round-trip. This would mean Mesa accepting an MLIR module directly from an ML runtime and running its own MLIR passes before calling `spirv_to_nir` — or replacing `spirv_to_nir` with an MLIR→NIR lowering. This is architecturally appealing: it would eliminate SPIR-V validation overhead, preserve higher-level information (tile sizes, layout annotations) that SPIR-V cannot carry, and enable joint optimization across the framework and driver layers.

The reason it remains long-term is the same reason the current division exists: SPIR-V is a *stable contract*. Every MLIR-based ML tool (IREE, Triton, XLA) targets it. Every Mesa driver consumes it. The interface is versioned, validated, and maintained by Khronos. Replacing it with a direct MLIR interface would require all producers and all consumers to coordinate on a new contract — a coordination cost that the existing SPIR-V ecosystem absorbs. Until the overhead of SPIR-V serialization is demonstrably the bottleneck, and until a stable MLIR ABI for this purpose is defined, the layer boundary remains at SPIR-V.

---

## Roadmap

### Near-term (6–12 months)

- **Intel XeVM dialect stabilisation in upstream LLVM.** Intel's `xevm` dialect — modelled after `nvvm` and `rocdl`, exposing Xe-architecture-specific features including 2D block loads/stores and XMX matrix multiply-add — was upstreamed to LLVM in mid-2025. Near-term work focuses on integration tests, lowering pipeline coverage, and enabling the `XeVM` target for full `mlir-opt` → ISA round-trips for Intel Arc and Battlemage GPUs. [Source](https://www.phoronix.com/news/Intel-XeVM-MLIR-In-LLVM) [Source](https://mlir.llvm.org/docs/Dialects/XeVMDialect/)

- **Triton Gluon dialect and automatic warp specialisation.** The `gluon` dialect exposes warp-group MMA, TMA descriptors, and explicit warp specialisation to expert kernel authors. Work is ongoing to generalise compiler heuristics so that auto-warp-specialisation applies to a broader class of kernels beyond attention and GEMM, and to stabilise the transformation passes for production use. [Source](https://pytorch.org/blog/warp-specialization-in-triton-design-and-roadmap/) [Source](https://triton-lang.org/main/getting-started/tutorials/gluon/index.html)

- **NVIDIA Blackwell and on-device TMA descriptor support in Triton.** Active upstream work adds support for Blackwell's Cluster Launch Control, distributed shared memory (DSMEM), multi-CTA tiling, and on-device TMA descriptor pipelines. These features are needed to match hand-written CUTLASS kernels on GB200 hardware. Note: needs verification for exact merge timeline.

- **IREE HAL NPU backend expansion.** IREE's hardware abstraction layer is being extended to target dedicated NPU backends (Qualcomm QNN, Samsung Eden, emerging RISC-V NPU targets), following the same MLIR → Stream → HAL → SPIR-V (or NPU binary) pipeline described in Section 6. [Source](https://iree.dev/)

- **SPIR-V cooperative matrix lowering improvements in Mesa.** Mesa's RADV and ANV drivers continue hardening their `VK_KHR_cooperative_matrix` paths as MLIR-generated SPIR-V that uses `OpCooperativeMatrixMulAddKHR` reaches production workloads via IREE and Triton's Vulkan backend. Note: needs verification for specific Mesa MR numbers.

### Medium-term (1–3 years)

- **Unified MLIR GPU dialect abstraction over vendor dialects.** The LLVM community is discussing a more principled separation between the hardware-neutral `gpu` dialect and vendor-specific dialects (`nvvm`, `rocdl`, `xevm`, `xegpu`). A proposed "GPU target" interface would allow a single `-convert-gpu-to-target` pass to dispatch to the correct vendor backend based on registered target attributes, eliminating the need for separate per-vendor lowering pipelines. Note: needs verification — track the LLVM discourse RFC threads.

- **Triton Vulkan/SPIR-V backend reaching feature parity.** The experimental Triton SPIR-V path currently lacks asynchronous copy instructions (`cp.async` equivalent), warp shuffle, and Hopper-class TMA. Bridging these gaps via new SPIR-V extensions (`SPV_KHR_non_uniform_group_operations`, async copy proposals in the Khronos working group) is a medium-term goal to enable Triton kernels on AMD and Intel GPUs without a PTX intermediate. [Source](https://github.com/triton-lang/triton)

- **StableHLO as the universal ML IR interchange.** The OpenXLA project is consolidating on StableHLO as the portable serialisation format across JAX, TensorFlow, and PyTorch XLA. Medium-term plans include a stable serialisation format with versioned compatibility guarantees, enabling IREE, XLA, and third-party runtimes to accept a single `.stablehlo` artifact and compile it independently. [Source](https://openxla.org/xla)

- **`SPV_KHR_cooperative_matrix` v2 and finer-grain precision support.** Khronos is working on extensions to `VK_KHR_cooperative_matrix` to expose FP8 (E4M3 / E5M2) matrix operands, INT4 quantised weights, and mixed-precision accumulation matching NVIDIA Hopper H100 and AMD CDNA3 hardware capabilities. MLIR's `spirv` dialect will need corresponding type and op additions. Note: needs verification for extension naming and timeline.

- **Structured kernel library approach in IREE.** IREE is developing a "codegen tuning" infrastructure where kernel tile configurations, vectorisation decisions, and pipeline depths are stored as external YAML specifications and applied at compile time, enabling model-specific tuning without recompilation of the full IREE stack. [Source](https://iree.dev/reference/mlir-dialects/HAL/)

### Long-term

- **MLIR as the universal GPU compiler IR across ML and graphics workloads.** A long-term architectural goal within LLVM/MLIR is to converge the graphics shader compilation path (GLSL → SPIR-V → Mesa NIR) and the ML compute path (linalg → gpu → SPIR-V) on a shared MLIR foundation. Mesa could in principle accept MLIR directly — bypassing the SPIR-V serialisation/deserialisation round-trip — for offline-compiled compute pipelines, reducing latency and debug friction.

- **Hardware-native ML tensor types in Vulkan and SPIR-V.** As FP8 and INT4 become first-class hardware types on RDNA4, Blackwell, and Xe3+, pressure will grow to expose them natively in SPIR-V and Vulkan rather than through cooperative matrix extensions. Long-term, MLIR's type system could drive the design of these extensions rather than trail hardware by multiple generations.

- **Automatic differentiation and training workloads on Vulkan via MLIR.** Current MLIR GPU infrastructure is optimised for inference (forward pass). Training requires reverse-mode automatic differentiation, gradient checkpointing, and cross-device collective communication (AllReduce). Long-term, frameworks such as JAX may route training through Vulkan on non-CUDA hardware via MLIR/IREE, requiring deep integration between the `linalg` / `scf` dialects and collective communication primitives. Note: needs verification — this is a speculative architectural direction.

- **Formal verification of MLIR lowering passes for safety-critical GPU compute.** As MLIR-compiled GPU code is deployed in automotive, medical, and safety-critical contexts, correctness guarantees for lowering passes (particularly tiling, vectorisation, and memory ordering) become important. Research into formally verified MLIR pass semantics is an active academic direction. Note: needs verification — track PLDI / CGO academic literature.

---

## Integrations

This chapter sits at the intersection of Mesa's compiler infrastructure and the emerging ML/inference stack. The following chapters provide essential context in both directions:

**Feeding into this chapter (upstream):**

- **Chapter 61 — The SPIR-V Ecosystem** (`chapters/part-14-khronos-ecosystem/ch61-spirv-ecosystem.md`): The SPIR-V binary format, tools (spirv-val, spirv-opt, spirv-cross), and the Khronos standardisation process. Chapter 61 covers SPIR-V in its full breadth across Vulkan, OpenCL, and SYCL; this chapter focuses on the specific sub-path from MLIR to SPIR-V to Mesa.

- **Chapter 77 — The Shader Toolchain** (`chapters/part-04-mesa-architecture/ch77-shader-toolchain.md`): glslangValidator, DXC, and the classical shader-to-SPIR-V path. MLIR is an additional producer that sits above this toolchain; both paths converge at `vkCreateShaderModule`.

**Fed by this chapter (downstream):**

- **Chapter 14 — NIR: Mesa's Shader Intermediate Representation** (`chapters/part-04-mesa-architecture/ch14-nir-shader-ir.md`): The NIR IR that `spirv_to_nir` produces, and the complete library of NIR optimisation and lowering passes. MLIR's SPIR-V dialect feeds NIR via exactly the `spirv_to_nir` path described in Section 10.

- **Chapter 15 — The ACO Compiler** (`chapters/part-04-mesa-architecture/ch15-aco-compiler.md`): ACO is the final compilation stage after NIR for RADV. Cooperative matrix NIR intrinsics produced from MLIR-generated SPIR-V are lowered to WMMA instructions by ACO's instruction selection.

- **Chapter 16 — Mesa Vulkan Common** (`chapters/part-04-mesa-architecture/ch16-mesa-vulkan-common.md`): The `vkCreateComputePipeline` entry point and the common Vulkan pipeline infrastructure that receives compiled SPIR-V and drives `spirv_to_nir`.

**Parallel tracks:**

- **Chapter 24 — Vulkan and EGL for Application Developers** (`chapters/part-07a-gpu-apis/ch24-vulkan-egl-application-developers.md`): The Vulkan compute pipeline API that IREE uses to submit SPIR-V compute shaders.

- **Chapter 25 — GPU Compute** (`chapters/part-07a-gpu-apis/ch25-gpu-compute.md`): OpenCL, ROCm HIP, and Vulkan compute as the three Linux GPU compute APIs. IREE and XLA target all three; this chapter provides the API-level context for what IREE submits.

- **Chapter 48 — ROCm and ML on Linux** (`chapters/part-07a-gpu-apis/ch48-rocm-ml-linux.md`): The AMD ROCm stack, including HIP and the AMDGPU kernel driver. XLA's AMD path (`jax[rocm]`) and Triton-ROCm both target this stack at the ISA level.

**Chapters using the results of this chapter:**

- **Chapter 87 — Local LLM Inference on Linux GPUs** (Part XX — planned): llama.cpp, vLLM, and Ollama; the Triton FlashAttention kernels from Section 5 are central to vLLM's performance on both NVIDIA and AMD hardware.

- **Chapter 88 — NPU and AI Accelerator Integration on Linux** (Part XX — planned): IREE as a compiler for NPU targets. The IREE HAL is designed to be extended with new target backends; an NPU HAL backend follows the same compilation pipeline described in Section 6 up to the HAL dialect.

---

## References

- MLIR Project: https://mlir.llvm.org/
- MLIR GPU Dialect: https://mlir.llvm.org/docs/Dialects/GPU/
- MLIR SPIR-V Dialect: https://mlir.llvm.org/docs/Dialects/SPIR-V/
- MLIR SPIR-V Serialization (TranslateRegistration.cpp): https://mlir.llvm.org/doxygen/SPIRV_2TranslateRegistration_8cpp_source.html
- OpenAI Triton: https://github.com/openai/triton
- Triton AMD backend: https://rocm.blogs.amd.com/artificial-intelligence/triton/README.html
- IREE: https://iree.dev/
- IREE Vulkan Deployment: https://iree.dev/guides/deployment-configurations/gpu-vulkan/
- IREE Stream Dialect: https://iree.dev/reference/mlir-dialects/Stream/
- IREE HAL Dialect: https://iree.dev/reference/mlir-dialects/HAL/
- OpenXLA / XLA: https://openxla.org/xla
- XLA GPU Architecture: https://openxla.org/xla/gpu_architecture
- SPIR-V Specification: https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html
- SPV_KHR_cooperative_matrix: https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_cooperative_matrix.html
- VK_KHR_cooperative_matrix Vulkan Feature Proposal: https://docs.vulkan.org/features/latest/features/proposals/VK_KHR_cooperative_matrix.html
- RADV VK_KHR_cooperative_matrix (Phoronix): https://www.phoronix.com/news/RADV-VK_KHR_cooperative_matrix
- Intel ANV Cooperative Matrix (Phoronix): https://www.phoronix.com/news/Intel-ANV-Cooperative-Matrix
- Mesa spirv_to_nir source: https://cgit.freedesktop.org/mesa/mesa/tree/src/compiler/spirv/spirv_to_nir.c
- Mesa environment variables: https://docs.mesa3d.org/envvars.html
- Mesa RADV documentation: https://docs.mesa3d.org/drivers/radv.html
- Mesa SPIR-V debugging: https://docs.mesa3d.org/spirv/index.html
- FlashAttention-2 Triton walkthrough: https://nathanchen.me/public/Triton-Flash-Attention-Kernel-Walkthrough.html
- AMD Triton compilation deep dive: https://medium.com/@nzhangnju/a-deep-dive-into-amd-triton-compilation-912d96e68e45
- Intel XMX overview: https://www.intel.com/content/www/us/en/support/articles/000091112/graphics.html
- AMD WMMA on RDNA3: https://gpuopen.com/learn/wmma_on_rdna3/
- StableHLO: https://github.com/openxla/stablehlo

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
