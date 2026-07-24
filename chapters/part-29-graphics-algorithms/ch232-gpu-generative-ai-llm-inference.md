# Chapter 232: GPU Generative AI and LLM Inference on Linux (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Systems developers deploying LLM or diffusion model inference on Linux with ROCm, Vulkan, or CUDA; graphics application developers integrating generative AI into rendering or creative pipelines; browser engineers understanding WebGPU ML inference paths.

This chapter focuses on the *algorithms and GPU kernels* that power generative AI inference on Linux — the compute primitives, memory management schemes, and quantization techniques that live beneath the runtime APIs. For deployment plumbing (GGML executor internals, Ollama server architecture, ONNX Execution Providers, memory-mapped weight loading, and llama.cpp Vulkan shader dispatch), see Chapter 124.

---

## Table of Contents

1. [Generative AI Inference on Linux — Scope and Hardware Reality](#1-generative-ai-inference-on-linux--scope-and-hardware-reality)
2. [LLM Architecture and the Inference Compute Graph](#2-llm-architecture-and-the-inference-compute-graph)
3. [KV-Cache and Memory Management](#3-kv-cache-and-memory-management)
4. [Quantization for LLM Inference](#4-quantization-for-llm-inference)
5. [Attention Kernel Optimizations](#5-attention-kernel-optimizations)
6. [Speculative Decoding Algorithms](#6-speculative-decoding-algorithms)
7. [Diffusion Model Inference Pipeline](#7-diffusion-model-inference-pipeline)
8. [Diffusion Model Quantization and Optimization](#8-diffusion-model-quantization-and-optimization)
9. [ROCm Inference Stack](#9-rocm-inference-stack)
10. [Vulkan Compute Inference](#10-vulkan-compute-inference)
11. [Linux Deployment Stack](#11-linux-deployment-stack)
12. [Performance Profiling and Optimization](#12-performance-profiling-and-optimization)
13. [Integrations](#13-integrations)

---

## 1. Generative AI Inference on Linux — Scope and Hardware Reality

Generative AI inference encompasses two dominant workload families: **autoregressive LLM decoding** (sequential token generation from transformer-based language models) and **diffusion model sampling** (iterative denoising of a latent variable through a UNet or transformer backbone). Both workloads have become primary consumers of discrete GPU compute on Linux systems, yet they differ fundamentally in their compute profiles, memory access patterns, and optimization strategies.

### Linux-Specific Hardware Landscape

Linux inference runs on three principal hardware paths, each with distinct driver stack implications:

- **CUDA (NVIDIA)**: proprietary kernel module plus open GSP firmware path (see Ch124 §8 and Ch15/NVIDIA chapters). Broadest framework support; FlashAttention-3 targets Hopper (H100/H200) specifically. RTX 4090 (24 GB GDDR6X) handles 13B INT4 with full KV cache; 70B INT4 requires an A100 80 GB or H100 80 GB.
- **ROCm (AMD)**: open-source `amdgpu` + `amdkfd` kernel path (see Ch48, Ch108). RDNA 3 (RX 7900 XTX, 24 GB) and CDNA (MI300X, 192 GB HBM3) are production targets as of mid-2026. The MI300X's unified 192 GB HBM3 pool makes 70B FP16 inference trivially fit on one device.
- **Vulkan compute**: universal fallback supported by all Vulkan 1.3 capable GPUs (RADV, ANV, NVK, Turnip, Mali). Lower peak throughput than CUDA/ROCm due to absence of cooperative matrix hardware on older GPUs, but enables inference on hardware excluded from ROCm's gfx target list.

### Memory Requirements

Model size in VRAM is the first engineering constraint:

| Model | FP16 | INT8 | INT4 (W4A16) |
|---|---|---|---|
| 7B parameters | ~14 GB | ~7 GB | ~3.5 GB |
| 13B parameters | ~26 GB | ~13 GB | ~6.5 GB |
| 70B parameters | ~140 GB | ~70 GB | ~35 GB |
| 70B (Q4_K_M GGUF) | — | — | ~41 GB (block overhead) |

KV cache consumes additional VRAM proportional to batch size and sequence length, discussed in §3. The inference-versus-training distinction is critical: inference does not maintain gradient tensors or optimizer states, so memory is dominated by weights plus KV cache, not activations. This enables single-GPU deployment of models that require multi-node training.

---

## 2. LLM Architecture and the Inference Compute Graph

A transformer decoder block — the repeated unit in GPT-style LLMs — executes the following sequence per layer:

```
x → LayerNorm → [Q, K, V projections] → Attention → Output projection → residual add
  → LayerNorm → FFN [gate/up GEMM → activation → down GEMM] → residual add
```

Each of the named steps maps to a specific GPU kernel type:

**LayerNorm / RMSNorm**: element-wise reduce-then-scale; memory-bandwidth-bound (AI ≈ 1–2 FLOPs/byte). Llama 3 uses RMSNorm: `x / sqrt(mean(x²) + ε) * weight`.

**QKV projection**: a batched GEMM of shape `[seq_len, hidden_dim] × [hidden_dim, 3*head_dim*n_heads]`. Arithmetic intensity rises with `seq_len`; for long prompts this is compute-bound. For batch=1 decode (one token per step), it degenerates to a matrix-vector product — bandwidth-bound.

**Attention**: scaled dot-product attention `softmax(QKᵀ / √d_head) V`. Naive implementation materializes the `seq_len × seq_len` score matrix; FlashAttention avoids this (§5).

**Output projection and FFN GEMMs**: same shape analysis as QKV projection. The FFN typically expands to `4 × hidden_dim` (or `8/3 × hidden_dim` for SwiGLU variants used in Llama).

### Architectural Variants

**Grouped-Query Attention (GQA)**, as used in Llama 3 [Source: Ainslie et al., "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints", https://arxiv.org/abs/2305.13245]: instead of one K/V head per Q head, multiple Q heads share a single K/V head group. This reduces KV-cache VRAM without proportional quality loss. Llama-3-70B uses 8 KV heads for 64 query heads (8:1 ratio), reducing KV-cache by 8×.

**Mixture-of-Experts (MoE)**, as in Mixtral-8×7B: a router network selects `top-k` expert FFN sub-networks per token. Only `k` of the 8 experts activate per token, so active parameter count per token is lower than total parameter count. GPU implementation dispatches expert GEMMs in a batched-scatter pattern — only populated experts receive tokens, making load balancing critical.

**RoPE (Rotary Positional Encoding)** [Source: Su et al., "RoFormer: Enhanced Transformer with Rotary Position Embedding", https://arxiv.org/abs/2104.09864]: applies a rotation matrix in each pair of embedding dimensions based on position. Implementation: split Q/K into pairs `(x_{2i}, x_{2i+1})`, apply `[cos(m*θ_i), -sin(m*θ_i); sin(m*θ_i), cos(m*θ_i)]` per pair, where `m` is the token position and `θ_i = 10000^{-2i/d}`. This fuses naturally with the Q/K projection in a single kernel.

### Prefill vs. Decode Phase

**Prefill** (processing the input prompt): all tokens are available simultaneously. QKV projection operates on `[seq_len, hidden_dim]`; attention is computed over the full sequence in parallel. Arithmetic intensity is proportional to `seq_len` — prefill is typically compute-bound for prompts longer than ~512 tokens.

**Decode** (autoregressive generation, one token at a time): only one new token's Q is projected; K/V are read from the KV cache. QKV projection degenerates to `[1, hidden_dim] × [hidden_dim, 3*head_dim*n_heads]` — a GEMV. Arithmetic intensity ≈ 1 FLOP/byte for the GEMV (2 FLOPs per FP16 weight element, 2 bytes per element), firmly memory-bandwidth-bound. This is the dominant bottleneck in interactive inference: the GPU reads the full weight matrix but performs trivial compute per byte.

---

## 3. KV-Cache and Memory Management

During decode, the key and value tensors for all prior tokens must be accessible. Storing them persistently avoids recomputing attention over the full context each step.

### KV-Cache Layout

The canonical layout is a pair of tensors per layer, shaped `[n_heads, max_seq_len, head_dim]` (or equivalently `[max_seq_len, n_heads, head_dim]` depending on implementation). Across all layers the total size is:

```
KV cache bytes = 2 × n_layers × n_kv_heads × max_seq_len × head_dim × dtype_bytes
```

For Llama-3-70B (80 layers, 8 KV heads, 128 head_dim) in FP16 with max_seq_len=8192: `2 × 80 × 8 × 8192 × 128 × 2 bytes = ~2.7 GB` per sequence. At a single sequence this is modest, but under continuous batching with 32 concurrent sequences at max length, aggregate KV cache reaches ~86 GB — comparable to full model weight storage and the primary constraint on batch size in production deployments.

### PagedAttention

The PagedAttention algorithm [Source: Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention", SOSP 2023, https://dl.acm.org/doi/10.1145/3600006.3613165] eliminates external fragmentation in KV-cache allocation by managing memory in fixed-size **physical blocks** of 16 tokens each.

Each request maintains a **block table** mapping logical block indices to physical block indices in a GPU memory pool:

```
Request A: logical blocks [0, 1, 2] → physical blocks [7, 3, 15]
Request B: logical blocks [0, 1]    → physical blocks [2, 9]
```

When a new token is generated, if the current physical block is full, a new free block is allocated from the pool. When a request finishes, its physical blocks are returned to the free list — no compaction needed.

**Copy-on-write for beam search**: multiple beam candidates can share physical blocks for their shared prefix. A block is duplicated only when a candidate must write a new token to a block also referenced by another candidate.

**Prefix caching**: system prompts shared across requests can be pre-computed and their KV blocks retained in a radix tree keyed by token hash. Subsequent requests with the same prefix reuse the cached blocks without recomputation, reducing time-to-first-token for common prefixes. vLLM implements this as Automatic Prefix Caching (APC) using a hash-chain per block.

The PagedAttention attention kernel itself differs from standard attention: K and V data is scattered across non-contiguous physical blocks. The kernel gathers blocks via the block table before computing attention — trading a gather indirection for elimination of memory fragmentation.

### Continuous Batching

Unlike static batching (fixed batch size per request), continuous batching (also called iteration-level scheduling) allows new requests to be inserted into an active batch at any decode step once earlier requests complete. This maximizes GPU utilization by keeping the batch full. The scheduler maintains a priority queue of waiting requests and slots them in as physical blocks become available.

---

## 4. Quantization for LLM Inference

Quantization reduces weight storage and memory bandwidth requirements during decode. The dominant strategies on Linux differ by algorithm and target hardware.

### Weight-Only Quantization (W4A16)

Weights are stored as INT4; activations remain FP16. The dequantization step — converting INT4 weights to FP16 before the GEMM — is fused into the GEMV/GEMM kernel. For decode-phase GEMV, the bottleneck is weight loading from HBM, so INT4 weights reduce memory traffic by ~4×, approximately quadrupling decode token throughput relative to FP16.

**GPTQ** [Source: Frantar et al., "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers", ICLR 2023, https://arxiv.org/abs/2210.17323]: layer-wise post-training quantization using the approximate second-order (Hessian) information of each layer's weight. The algorithm solves a block-wise least-squares problem to find INT4 weights that minimize the output error of each layer. GPTQ is applied offline and produces per-row scale factors stored alongside the INT4 weights.

**AWQ (Activation-aware Weight Quantization)** [Source: Lin et al., "AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration", MLSys 2024, https://arxiv.org/abs/2306.00978]: observes that a small fraction of weight channels (those corresponding to large-magnitude activations) disproportionately affect output quality. AWQ scales these salient channels up before quantization (and scales activations down correspondingly), preserving accuracy without requiring per-sample calibration data at the granularity GPTQ requires.

### GGUF Quantization Block Layout

The GGUF format, used by llama.cpp and Ollama, defines quantization types at the block level. Each block covers 32 or 256 weights and stores shared scale factors alongside the quantized values [Source: llama.cpp GGUF specification, https://github.com/ggerganov/llama.cpp/blob/master/docs/gguf.md]:

| Type | Bits | Block size | Notes |
|---|---|---|---|
| `Q4_0` | 4 | 32 weights | Single FP16 scale per block |
| `Q4_K_M` | 4 (mixed) | 256 weights | Super-block with nested sub-blocks; 4/6-bit mixed |
| `Q5_K_S` | 5 | 256 weights | K-quant, small quality tier |
| `Q8_0` | 8 | 32 weights | FP16 scale; near-lossless |
| `IQ4_NL` | 4 | 32 weights | Importance-aware; non-linear lookup table |

K-quant formats (`Q4_K`, `Q5_K`, `Q6_K`) use a two-level block structure: a super-block stores a coarser scale, and nested sub-blocks store finer scales. This reduces scale overhead and improves quantization accuracy at the cost of more complex dequantization logic in the GPU kernel.

### Mixed-Precision Layers

First and last layers of a transformer (embedding table, output logit projection) are typically kept at FP16 or BF16, as quantization error in these layers disproportionately affects output quality. Middle transformer layers use INT4. This mixed-precision strategy is implemented in GGUF by storing individual layer tensors with per-tensor quantization types.

### KV-Cache Quantization

KV-cache quantization stores the cached K/V tensors at reduced precision to save VRAM. INT8 KV cache (8-bit per element) halves KV cache size with minimal quality degradation. FP8 KV cache (on H100 and MI300X hardware supporting native FP8) halves it further. vLLM implements INT8 KV cache via a per-tensor quantization pass on each K/V block before storing to the physical block pool.

---

## 5. Attention Kernel Optimizations

The naive O(N²) materialization of the attention score matrix is the primary bottleneck for long sequences and high batch sizes.

### FlashAttention-2

FlashAttention-2 [Source: Dao, "FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning", ICLR 2024, https://arxiv.org/abs/2307.08691] eliminates the N×N score matrix by tiling Q, K, V into SRAM-resident tiles and computing attention in a single fused kernel:

```
For each tile of Q (size Br × d):
  Initialize: m = -∞ (running max), l = 0 (normalizer), O = 0
  For each tile of K, V (size Bc × d):
    S = Q_tile @ K_tile^T    # Br × Bc score tile
    m_new = max(m, rowmax(S))
    P = exp(S - m_new)       # numerically stable
    l_new = exp(m - m_new)*l + rowsum(P)
    O = (exp(m - m_new)*O + P @ V_tile) / l_new
    m = m_new; l = l_new
  Write O to HBM
```

The **online softmax** — maintaining a running maximum and normalizer — allows each tile pass to update O without storing intermediate attention scores. HBM reads are O(N × d); the N×N score matrix never leaves SRAM. FlashAttention-2 improves over FA1 by repartitioning work across warps to reduce non-matmul FLOPs and allow better parallelism across the sequence dimension.

### FlashAttention-3

FlashAttention-3 [Source: Shah et al., "FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low-precision", 2024, https://arxiv.org/abs/2407.08608] targets NVIDIA Hopper (H100/H200) specifically, exploiting hardware features unavailable on earlier architectures:

- **wgmma (warp-group matrix-multiply-accumulate)**: new Hopper ISA instructions for asynchronous GEMM at the warp-group granularity, allowing pipelining of GEMM and softmax operations.
- **TMA (Tensor Memory Accelerator)**: hardware unit that handles bulk SRAM ↔ HBM transfers asynchronously, allowing compute and memory operations to overlap.
- **FP8 precision**: FA3 introduces FP8 attention (E4M3 format) on Hopper for further speedup.

FA3 achieves approximately 75% of H100 FP16 peak on forward attention, versus ~35% for the naive implementation.

### ROCm FlashAttention

On AMD hardware, FlashAttention is implemented via `composable_kernel` (CK), AMD's high-performance tile-based GEMM/attention library [Source: AMD composable_kernel, https://github.com/ROCm/composable_kernel]. CK implements the online-softmax tiling algorithm using HIP kernel templates parameterized on tile sizes and data types. For GQA, CK implements the grouped-query attention kernel that broadcasts K/V across the Q head group. `hipBLASLt` provides the underlying GEMM used in non-fused attention paths.

### PagedAttention Kernel

The PagedAttention kernel [Source: vLLM source, https://github.com/vllm-project/vllm/tree/main/csrc] differs from standard FlashAttention in that K and V blocks are non-contiguous in physical memory. The kernel accepts a block table array and gathers K/V data from scattered physical blocks before computing the softmax-weighted sum. On NVIDIA, this is implemented in CUDA via a custom kernel in vLLM's C extension (`csrc/attention/attention_kernels.cu`). On AMD, vLLM-ROCm uses the AITER (AI Tensor Engine for ROCm) attention kernels.

### Speculative Decoding Attention

When verifying multiple draft tokens simultaneously (§6), attention must be computed over a **token tree** rather than a linear sequence. Tree attention extends the causal mask to allow each draft token to attend only to its ancestors in the token tree. This requires a custom attention kernel that accepts a non-standard mask pattern.

### Ring Attention for Long Context

For sequences exceeding a single GPU's VRAM capacity, **ring attention** distributes sequence chunks across multiple devices in a ring topology [Source: Liu et al., "Ring Attention with Blockwise Transformers for Near-Infinite Context", 2023, https://arxiv.org/abs/2310.01889]. Each device holds a shard of Q, K, V. In each ring step, each device computes local attention between its Q shard and the current K/V shard received from its neighbor, then passes K/V to the next device. The local FlashAttention outputs are combined using the online softmax reduction. Ring attention allows O(seq_len/n_devices) memory per device with O(n_devices) communication rounds.

---

## 6. Speculative Decoding Algorithms

Standard autoregressive decode generates one token per forward pass, making the GPU execute one large GEMV per layer per token. Speculative decoding breaks this bottleneck by generating and verifying multiple tokens per step.

### Draft Model + Target Verifier

The canonical speculative decoding algorithm [Source: Leviathan et al., "Fast Inference from Transformers via Speculative Decoding", ICML 2023, https://arxiv.org/abs/2211.17192]: a small **draft model** (e.g., 1B parameters) autoregressively generates `k` draft tokens. The large **target model** processes all `k` draft tokens in a single parallel forward pass and produces probability distributions for each position. **Rejection sampling** then accepts or rejects each draft token:

```
For i in 0..k:
  if uniform() < p_target(x_i) / p_draft(x_i):
    accept x_i
  else:
    resample from (p_target - p_draft)+ and stop
```

This procedure is **lossless**: the distribution of accepted tokens is identical to what the target model would produce alone. GPU implementation batches the `k`-token forward pass into a single decode call, amortizing the fixed overhead (kernel launch, memory access setup) across `k` tokens. Acceptance rate depends on draft model quality; typical values are 60–80% per token, yielding ~2–3× wall-clock speedup.

### Token Tree Speculation

Instead of a linear chain of `k` draft tokens, a **token tree** generates multiple branching candidate sequences. The target model verifies all branches simultaneously using tree attention (§5), selecting the longest accepted prefix along any branch. This improves utilization of the verification forward pass.

### Medusa

Medusa [Source: Cai et al., "Medusa: Simple Framework for Accelerating LLM Generation with Multiple Decoding Heads", 2024, https://arxiv.org/abs/2401.10774] eliminates the separate draft model by attaching multiple **draft heads** directly to the target model's penultimate hidden state. Each head `i` predicts the token `i+1` steps ahead. The heads are small (one linear layer per head) and trained with a custom tree-attention loss. GPU implementation: the target model forward pass produces the hidden state once; `k` draft heads run in parallel (k small GEMMs against the logit projection); tree attention verifies the resulting token tree in one pass.

### EAGLE

EAGLE [Source: Li et al., "EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty", 2024, https://arxiv.org/abs/2401.15077] trains a lightweight draft model that operates in **feature space** (embedding space of the target model's penultimate layer) rather than token space. The EAGLE draft model takes as input the target model's hidden states (already computed during the previous token's verification step) and predicts the next hidden state, from which draft tokens are decoded. By operating in the target model's representation space, EAGLE achieves higher acceptance rates than external draft models of comparable parameter count.

### GPU Implementation Notes

Both Medusa and EAGLE require modifications to the KV-cache management to handle the tree structure. In vLLM, speculative decoding is implemented as a `SpecDecodeWorker` that coordinates draft generation and target verification on separate GPU processes (or the same GPU with interleaved batches). The tree-attention verification step reuses the standard PagedAttention infrastructure with a modified causal mask.

---

## 7. Diffusion Model Inference Pipeline

Latent diffusion models [Source: Rombach et al., "High-Resolution Image Synthesis with Latent Diffusion Models", CVPR 2022, https://arxiv.org/abs/2112.10752] decompose image generation into three components:

1. **VAE encoder** (inference-time optional): compresses a 512×512 image to a 4×64×64 latent tensor (8× spatial downsampling, 4 channels).
2. **Denoising UNet**: iteratively refines the latent from pure noise to a clean latent through T denoising steps.
3. **VAE decoder**: expands the 4×64×64 clean latent back to a 512×512 image.

### UNet Architecture

The UNet backbone consists of:

- **Encoder path**: alternating ResNet blocks and spatial attention blocks, with stride-2 downsampling. Each ResNet block is two Conv2d + GroupNorm + SiLU layers with a residual connection.
- **Bottleneck**: ResNet + full self-attention at lowest spatial resolution (8×8 for 512 input).
- **Decoder path**: symmetric, with skip connections from encoder at matching resolution.
- **Cross-attention blocks**: inject text conditioning by computing attention between spatial feature maps (query) and CLIP/T5 text embeddings (key/value). Cross-attention shape: `[batch × spatial_tokens, d_model] × [d_model, n_text_tokens]`.

Text conditioning via cross-attention is the primary GEMM workload in diffusion inference: at each resolution level, every ResNet block is followed by a `SpatialTransformer` module that performs self-attention and cross-attention over the spatial feature map.

### Scheduler Algorithms

The scheduler controls how noise is removed across T steps:

**DDPM** [Source: Ho et al., "Denoising Diffusion Probabilistic Models", NeurIPS 2020, https://arxiv.org/abs/2006.11239]: stochastic; 1000 steps. Each step samples `x_{t-1} ~ p(x_{t-1}|x_t, x_0_pred)`. High quality but slow.

**DDIM** [Source: Song et al., "Denoising Diffusion Implicit Models", ICLR 2021, https://arxiv.org/abs/2010.02502]: deterministic; typically 20–50 steps. Reinterprets the reverse diffusion as a non-Markovian process that allows larger step sizes. Same UNet call per step, but the schedule skips intermediate noise levels.

**DPM-Solver++** [Source: Lu et al., "DPM-Solver++: Fast Solver for Guided Sampling of Diffusion Probabilistic Models", 2022, https://arxiv.org/abs/2211.01095]: second- or third-order ODE solver for the diffusion probability flow. Achieves high-quality results in 15–25 steps by using multi-step integration that leverages predictions from previous steps. The GPU implementation maintains a small history buffer (2–3 previous noise predictions) and applies the higher-order correction formula.

### SDXL Two-Stage Pipeline

Stable Diffusion XL operates at 1024×1024 resolution with a two-model pipeline:

1. **Base model**: UNet operating on 4×128×128 latents (1024/8 = 128). Text conditioning via dual text encoders (CLIP-L and CLIP-G). Generates a noisy intermediate latent at a fixed timestep (typically T=800).
2. **Refiner model**: continues denoising from the base model's intermediate latent, adding high-frequency detail at 4×128×128.

The base and refiner share the VAE decoder. VRAM requirement for sequential inference: ~6 GB for the VAE, ~5 GB for the text encoders, and ~6–10 GB per UNet model in FP16 — roughly 18–20 GB total for sequential offloading.

### ControlNet Conditioning

ControlNet [Source: Zhang et al., "Adding Conditional Control to Text-to-Image Diffusion Models", ICCV 2023, https://arxiv.org/abs/2302.05543] adds a parallel encoder path to the UNet. The control signal (edge map, depth map, pose skeleton) passes through a learned encoder identical to the UNet encoder, with an extra zero-initialized convolution at each resolution level. The encoder outputs are added to the corresponding UNet decoder skip connections. GPU implementation runs the ControlNet encoder in parallel with the UNet encoder when GPU memory permits, or sequentially with weight sharing.

---

## 8. Diffusion Model Quantization and Optimization

### INT8 Post-Training Quantization

SDXL and SD2.x UNets can be quantized to INT8 weights using PyTorch's `torch.ao` quantization toolkit:

```python
import torch
from torch.ao.quantization import quantize_dynamic

# Dynamic quantization: INT8 weights, FP32 activations at runtime
quantized_unet = quantize_dynamic(
    unet,
    {torch.nn.Linear, torch.nn.Conv2d},
    dtype=torch.qint8
)
```

Dynamic quantization computes per-tensor activation scales at runtime, avoiding the calibration dataset requirement of static quantization. Quality impact on image generation is typically acceptable for INT8 weights; INT4 weight quantization of diffusion UNets shows more visible degradation than for LLMs due to the convolution-heavy architecture.

### VRAM Optimization Strategies

**Attention slicing**: splits the attention computation along the batch or head dimension to reduce peak VRAM. Instead of computing all heads simultaneously, computes `n_heads / slice_size` heads per iteration and accumulates results. Reduces peak VRAM by `slice_size`× at the cost of `slice_size`× more kernel dispatches.

**CPU offload (accelerate)**: the Hugging Face `accelerate` library implements `cpu_offload()`, which moves model modules to system RAM and streams them to GPU one at a time as each block executes. This allows running SDXL on a GPU with only 4–8 GB VRAM at the cost of high CPU↔GPU transfer overhead (~5–10× slower than pure GPU).

**Sequential offload**: a more aggressive version that offloads individual layers rather than entire modules, minimizing peak GPU memory further.

**Model sharding**: for multi-GPU setups, `device_map="auto"` in `accelerate` partitions UNet layers across available GPUs using a greedy bin-packing algorithm.

### xformers Memory-Efficient Attention

The `xformers` library [Source: xformers, https://github.com/facebookresearch/xformers] provides a memory-efficient attention implementation for diffusion models based on FlashAttention-style tiling. On ROCm, xformers uses the `triton` backend which compiles tiled attention kernels via Triton's ROCm LLVM backend (targeting GFX targets for RDNA3/CDNA). xformers integration in diffusion pipelines: `pipe.enable_xformers_memory_efficient_attention()`.

### torch.compile on ROCm

`torch.compile` with the `inductor` backend emits Triton kernels, which Triton compiles to GFX-specific LLVM IR and then to AMDGPU ISA via the AMDGPU LLVM backend. This enables fusion of pointwise operations (elementwise activations, scaling, bias) with GEMM and convolution kernels, reducing memory traffic. For diffusion UNets, `torch.compile` typically delivers 10–30% throughput improvement on RDNA3 after the initial compilation overhead.

---

## 9. ROCm Inference Stack

### HIP BLAS Tier

**rocBLAS** provides the foundational GEMM primitives [Source: rocBLAS documentation, https://rocm.docs.amd.com/projects/rocBLAS/en/latest/]:

```c
rocblas_status rocblas_gemm_ex(
    rocblas_handle     handle,
    rocblas_operation  transA,
    rocblas_operation  transB,
    rocblas_int        m, n, k,
    const void*        alpha,
    const void*        a, rocblas_datatype a_type, rocblas_int lda,
    const void*        b, rocblas_datatype b_type, rocblas_int ldb,
    const void*        beta,
    const void*        c, rocblas_datatype c_type, rocblas_int ldc,
    void*              d, rocblas_datatype d_type, rocblas_int ldd,
    rocblas_datatype   compute_type,
    rocblas_gemm_algo  algo,
    int32_t            solution_index,
    uint32_t           flags
);
```

The `compute_type` parameter allows mixed-precision computation: `ROCBLAS_DATATYPE_F32_R` compute with `ROCBLAS_DATATYPE_F16_R` inputs realizes the W4A16 dequantization-fused GEMM on supported hardware. `rocblas_gemm_algo` selects between default heuristics and solution index tables produced by the rocBLAS auto-tuner.

**hipBLASLt** extends rocBLAS with support for fused epilogue operations (bias add, activation) and lower-precision types. It is the ROCm analog of cuBLASLt and is required for FP8 GEMMs on MI300X.

**MIOpen** [Source: MIOpen documentation, https://rocm.docs.amd.com/projects/MIOpen/en/latest/] provides convolution (for diffusion UNets), attention (for both LLM and diffusion), and normalization kernels. MIOpen auto-tuning builds a local database of optimal kernel configurations per problem shape; the database is stored under `MIOPEN_USER_DB_PATH`.

### MIGraphX Inference Graph Compiler

MIGraphX [Source: AMD MIGraphX documentation, https://rocm.docs.amd.com/projects/AMDMIGraphX/en/latest/] is AMD's inference graph compiler. It imports ONNX models, performs operator fusion, and compiles to AMD GCN/RDNA ISA:

```cpp
#include <migraphx/migraphx.hpp>
#include <migraphx/onnx.hpp>
#include <migraphx/gpu/target.hpp>

migraphx::program prog = migraphx::parse_onnx("llama_decoder.onnx");
migraphx::compile_options copts;
copts.set_offload_copy(true);        // auto host-device transfers
prog.compile(migraphx::make_target("gpu"), copts);

// Inference
migraphx::program_parameters params;
params.add("input_ids",   migraphx::argument{ids_shape, ids_data});
params.add("position_ids", migraphx::argument{pos_shape, pos_data});
auto results = prog.eval(params);
```

*Note: the exact MIGraphX C++ API has evolved across ROCm releases; verify method signatures against the installed ROCm version's headers.*

MIGraphX fusion passes include: LayerNorm fusion (reduce + scale + bias), QKV projection fusion, and ReLU/SiLU activation fusion with GEMM epilogues. INT8 calibration runs a representative dataset through the graph and records per-tensor histogram statistics to derive quantization scales.

### vLLM on ROCm

vLLM's ROCm support [Source: vLLM ROCm documentation, https://docs.vllm.ai/en/latest/getting_started/amd-installation.html] maps the CUDA-based implementation through HIP API compatibility. The key ROCm-specific components:

- `ROCR_VISIBLE_DEVICES` environment variable controls device visibility (analog to `CUDA_VISIBLE_DEVICES`).
- **AITER** (AI Tensor Engine for ROCm) provides ROCm-native attention kernels (`VLLM_ROCM_USE_AITER=1`), replacing the CUDA FA2 kernels with composable_kernel-based implementations.
- The `PagedAttentionImpl` class (in `vllm/attention/backends/`) selects the attention kernel backend at initialization; on ROCm, it dispatches to the HIP-compiled attention kernel. *Note: the exact class name in vLLM source should be verified against the installed version.*

### llama.cpp HIP/ROCm Path

llama.cpp's ROCm backend compiles via `make GGML_HIPBLAS=1`. The dispatch logic:

- For large GEMM shapes (prefill, large batches): routes to `hipblasGemmEx` with FP16 I/O.
- For GEMV shapes (decode batch=1): routes to a custom HIP GEMV kernel (`ggml-cuda/vec_dot.cuh` adapted for HIP) that dequantizes Q4_K_M blocks on-chip during the dot product.

Known deployment issues:
- Older ROCm versions (< 5.6) lack hipBLASLt; check `ROCM_PATH` is correctly set for shared library discovery.
- gfx target must match the installed GPU: `AMDGPU_TARGETS="gfx1100"` for RX 7900 XTX; `gfx90a` for MI250X; `gfx942` for MI300X.
- The `HSA_OVERRIDE_GFX_VERSION` environment variable can force gfx compatibility for unofficially supported GPUs.

---

## 10. Vulkan Compute Inference

Vulkan compute provides a vendor-neutral path for inference on GPUs excluded from ROCm's hardware support matrix, including older AMD RDNA1, Intel Arc (ANV driver), ARM Mali (Panfrost), and Qualcomm Adreno (Turnip).

### llama.cpp Vulkan Backend

The Vulkan backend in llama.cpp (`ggml/src/ggml-vulkan.cpp`) implements each quantization type's matrix-vector multiply as a dedicated SPIR-V compute shader [Source: llama.cpp Vulkan backend source, https://github.com/ggerganov/llama.cpp/tree/master/ggml/src/vulkan-shaders]:

```glsl
// mul_mat_q4_k.comp (simplified)
layout(local_size_x = 64) in;
layout(set = 0, binding = 0) readonly buffer WeightsBuf { Q4KBlock weights[]; };
layout(set = 0, binding = 1) readonly buffer ActBuf    { float16_t acts[]; };
layout(set = 0, binding = 2) writeonly buffer OutBuf   { float16_t output[]; };

layout(push_constant) uniform PushConst {
    uint M;   // output rows (weight matrix rows)
    uint K;   // weight cols / activation length
    uint stride;
} pc;

void main() {
    uint row = gl_GlobalInvocationID.x;
    // ... per-row dot product with Q4_K_M dequantization
}
```

Push constants carry matrix dimensions; descriptor sets bind weight, activation, and output buffers. Separate shader variants handle `Q4_0`, `Q4_1`, `Q4_K`, `Q5_K`, `Q8_0`, and `F16` types. Shader selection occurs at dispatch time based on the tensor's quantization type stored in the `ggml_tensor.type` field.

### VK_KHR_cooperative_matrix

On hardware and drivers that support `VK_KHR_cooperative_matrix` (NVIDIA Ampere+ via NVK or proprietary, Intel Arc via ANV, AMD RDNA3 via RADV — availability varies) [Source: Vulkan Specification VK_KHR_cooperative_matrix, https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_cooperative_matrix.html], llama.cpp can dispatch GEMM tiles using cooperative matrix instructions:

```glsl
#extension GL_KHR_cooperative_matrix : require
coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseA> fragA;
coopmat<float16_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseB> fragB;
coopmat<float32_t, gl_ScopeSubgroup, 16, 16, gl_MatrixUseAccumulator> fragC;
coopMatLoad(fragA, bufA, offsetA, strideA, gl_CooperativeMatrixLayoutRowMajor);
coopMatLoad(fragB, bufB, offsetB, strideB, gl_CooperativeMatrixLayoutColumnMajor);
coopMatMulAdd(fragA, fragB, fragC);
```

This maps to matrix-multiply hardware units (Tensor Cores on NVIDIA, Matrix Cores on AMD, XMX on Intel) and substantially outperforms scalar ALU for large GEMM tiles.

### Kompute ML Framework

Kompute [Source: Kompute documentation, https://kompute.cc/] provides a higher-level Vulkan compute abstraction oriented toward ML workloads:

```cpp
kp::Manager mgr;                           // Vulkan instance + device
std::vector<float> a_data(1024), b_data(1024), out_data(1024);
auto tensorA   = mgr.tensor(a_data);
auto tensorB   = mgr.tensor(b_data);
auto tensorOut = mgr.tensor(out_data);

// Compile SPIR-V shader for this workgroup config
auto algorithm = mgr.algorithm(
    {tensorA, tensorB, tensorOut},
    kp::OpAlgoDispatch::compile(spirv_bytes),
    kp::Workgroup{32, 1, 1}
);

mgr.sequence()
    ->record<kp::OpTensorSyncDevice>({tensorA, tensorB})
    ->record<kp::OpAlgoDispatch>(algorithm)
    ->record<kp::OpTensorSyncLocal>({tensorOut})
    ->eval();
```

`kp::Manager` wraps `VkInstance`, `VkDevice`, and `VkQueue` creation. `kp::Algorithm` compiles and caches a SPIR-V pipeline with associated descriptor set layout. `kp::Sequence` records and submits a `VkCommandBuffer`. Kompute is suitable for small inference workloads and prototype integrations; production LLM inference at scale uses llama.cpp's Vulkan backend or vLLM directly.

### WebGPU Inference via Dawn

For browser-side inference, the WebGPU path (Dawn on Linux) enables ML workloads in WGSL compute shaders [Source: Dawn WebGPU implementation, https://dawn.googlesource.com/dawn]:

```wgsl
// attention.wgsl — simplified scaled dot-product attention
@group(0) @binding(0) var<storage, read>       q      : array<f32>;
@group(0) @binding(1) var<storage, read>       k      : array<f32>;
@group(0) @binding(2) var<storage, read>       v      : array<f32>;
@group(0) @binding(3) var<storage, read_write> output : array<f32>;

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
    let head = gid.x;
    // ... tiled softmax + weighted sum
}
```

Dawn compiles WGSL to SPIR-V and dispatches via Vulkan on Linux. WebGPU inference frameworks (Transformers.js, ONNX Runtime Web with WebGPU EP) use this path. The WebNN API (Ch168) provides a higher-level ML graph API above WebGPU for browser applications.

---

## 11. Linux Deployment Stack

### Ollama

Ollama [Source: Ollama documentation, https://ollama.com/] packages llama.cpp as a background server with automatic GPU detection and a REST API:

```bash
# Start server (auto-detects CUDA/ROCm/Vulkan)
ollama serve &

# Pull and run a model
ollama pull llama3.1:70b-instruct-q4_K_M
ollama run llama3.1:70b-instruct-q4_K_M

# REST API (OpenAI-compatible)
curl http://localhost:11434/api/generate \
  -d '{"model": "llama3.1:70b-instruct-q4_K_M",
       "prompt": "Explain PagedAttention.",
       "stream": false}'
```

GPU detection proceeds in order: CUDA (via NVML/`libnvidia-ml.so`), ROCm (via KFD sysfs `/sys/class/kfd/kfd/topology/nodes`), Vulkan (via `vkEnumeratePhysicalDevices`), CPU fallback. Model files are stored content-addressed under `~/.ollama/models/blobs/`. The REST endpoints `/api/generate`, `/api/chat`, `/api/embeddings`, and `/api/tags` provide the full inference interface (see Ch124 §5 for detailed Ollama internals).

### LocalAI

LocalAI [Source: LocalAI documentation, https://localai.io/] provides an OpenAI-compatible HTTP API supporting multiple backends simultaneously: llama.cpp for LLMs, whisper.cpp for speech recognition, and stable-diffusion.cpp for image generation. A single LocalAI instance can serve multiple model types via backend-specific runners, selected by model configuration YAML files. LocalAI is typically deployed as a container:

```bash
podman run --rm -it \
  --device /dev/kfd \
  --device /dev/dri \
  --group-add video \
  -v ~/.local/share/localai:/models \
  -p 8080:8080 \
  quay.io/go-skynet/local-ai:latest \
  --models-path /models
```

The `--device /dev/kfd --device /dev/dri` flags pass the AMD GPU compute and render nodes into the container. For NVIDIA, use `--gpus=all` with the NVIDIA container runtime.

### ComfyUI on Linux ROCm

ComfyUI [Source: ComfyUI repository, https://github.com/comfyanonymous/ComfyUI] implements diffusion inference as a node graph executor. Each node in the graph is a Python callable; the execution engine topologically sorts nodes and dispatches them sequentially (or in parallel where graph structure permits). GPU memory management uses a LRU cache (`model_management.py`) that evicts models from VRAM when new models need to load. On ROCm, ComfyUI uses `torch.backends.cuda.matmul.allow_tf32 = False` (since TF32 is NVIDIA-specific) and routes through PyTorch's ROCm backend.

### systemd Service and D-Bus Activation

For production inference daemon deployment:

```ini
# /etc/systemd/system/llm-inference.service
[Unit]
Description=LLM Inference Daemon
After=network.target

[Service]
Type=simple
User=llm
Environment=OLLAMA_HOST=0.0.0.0:11434
Environment=ROCR_VISIBLE_DEVICES=0
ExecStart=/usr/local/bin/ollama serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

D-Bus activation allows the inference daemon to start on first API request and shut down after an idle timeout, conserving GPU memory when not in use. A `.service` file with `[D-Bus Service]` section registers the well-known name; a corresponding `ExecStop` timer releases VRAM.

### Podman + ROCm Device Passthrough

For containerized deployment without root:

```bash
# AMD GPU passthrough with Podman (rootless)
podman run --rm \
  --device /dev/kfd:/dev/kfd \
  --device /dev/dri/renderD128:/dev/dri/renderD128 \
  --group-add $(getent group video | cut -d: -f3) \
  -e ROCR_VISIBLE_DEVICES=0 \
  vllm/vllm-rocm:latest \
  python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3-70B-Instruct \
    --quantization awq
```

The `/dev/kfd` device provides the AMD compute path (via `amdkfd`); `/dev/dri/renderD128` provides the DRM render node for display or interop. The `video` group membership grants access to `/dev/dri` devices.

---

## 12. Performance Profiling and Optimization

### Key Inference Metrics

Two metrics characterize inference performance:

- **TTFT (Time to First Token)**: latency from request submission to first generated token. Dominated by the prefill pass over the prompt. Target: < 1 second for interactive use.
- **Throughput (tok/s)**: tokens generated per second during decode. Determined by memory bandwidth during GEMV, or by compute during large-batch GEMM.

### Roofline Analysis for Decode

Decode at batch size 1 is memory-bandwidth-bound. For a 70B INT4 model (35 GB weights) on a GPU with 1 TB/s HBM bandwidth: each decode step reads all 35 GB weights (approximately) once. Bandwidth-limited throughput: `1,000 GB/s ÷ 35 GB = ~28 tokens/second`. Actual throughput is lower due to non-weight memory accesses (KV cache, activations) and kernel launch overhead.

Increasing batch size amortizes weight reads across multiple simultaneous requests: at batch=16, the same 35 GB weight read produces 16 tokens, achieving 16× higher throughput — until the KV cache fills VRAM or compute becomes the new bottleneck.

**Prefill** is compute-bound for long prompts. At seq_len=4096, the QKV and FFN GEMMs have shapes on the order of `[4096, 8192] × [8192, 8192]`, giving arithmetic intensity ~seq_len/2 ≈ 2048 FLOPs/byte — well above the ridge point of any current GPU. Achieving high utilization on prefill requires using cuBLAS/rocBLAS GEMM (not GEMV) paths, which llama.cpp and vLLM both do automatically when `n_tokens > 1`.

### rocprof Tracing

AMD's `rocprof` captures kernel-level timing for HIP dispatch [Source: ROCm rocprof documentation, https://rocm.docs.amd.com/en/latest/tools/rocprof.html]:

```bash
rocprof --hip-trace --stats \
  python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3-8B-Instruct &

# After a sample decode step:
# rocprof outputs results.csv with kernel durations
```

The trace identifies the dominant kernel by accumulated duration. For decode at batch=1, the top kernel is typically `ggml_cuda_mul_mat_vec_q` (GEMV with dequant) or the rocBLAS GEMM kernel for larger batches. For prefill, the top kernel is `rocblas_gemm_ex` or hipBLASLt GEMM.

### Batch Size and Chunked Prefill

**Chunked prefill** splits long prompts into fixed-size chunks (e.g., 512 tokens) processed sequentially. This limits peak VRAM for the prefill KV cache and allows decode requests to interleave with prefill processing, reducing TTFT for waiting decode requests at the cost of higher total prefill time.

### Thermal Throttling and Power Limits

Sustained inference workloads push GPUs to TDP, triggering thermal throttling on air-cooled desktop cards. On AMD:

```bash
# Check current GPU frequency and power
rocm-smi --showclocks --showpower

# Set power limit (watts) — reduces throttling risk
rocm-smi --setpoweroverdrive 0 250  # 250W limit on device 0

# Monitor in real time
watch -n 1 rocm-smi
```

On NVIDIA:
```bash
nvidia-smi -pl 300           # 300W power limit
nvidia-smi dmon -s pucvmet   # continuous metric stream
```

For production inference servers, liquid cooling or high-airflow rack configurations are standard. The Linux thermal daemon (`thermald`) and ACPI thermal zones do not directly control GPU power limits; GPU power management is handled by the driver via `rocm-smi` or `nvidia-smi`.

---

## 13. Integrations

**Ch25 (GPU Compute)**: establishes Vulkan compute pipeline fundamentals — descriptor sets, push constants, compute shader dispatch — that the Vulkan inference path (§10) builds on directly. Kompute and llama.cpp Vulkan backend both use the Vulkan compute primitives described there.

**Ch141 (Vulkan Cooperative Matrices)**: `VK_KHR_cooperative_matrix` (§10) extends the cooperative matrix extension covered in Ch141. The GLSL cooperative matrix syntax shown in §10 follows the extension specification detailed in Ch141.

**Ch152 (Rust GPU Ecosystem)**: `wgpu` and `naga` (Rust's WGSL compiler) provide an alternative path to WebGPU inference. Rust-based inference frameworks targeting `wgpu` can reach the same Dawn/Vulkan backend discussed in §10, with Rust type safety over the GPU dispatch layer.

**Ch221 (GPU Algorithm Performance)**: the roofline analysis in §12 applies the performance model framework from Ch221 to LLM decode specifically. The arithmetic intensity argument (GEMV ≈ 2 FLOPs/byte → bandwidth-bound) uses the same roofline methodology developed there.

**Ch226 (GEMM for Projection Layers)**: the QKV and FFN projection GEMMs (§2) are the primary GEMM workloads in LLM inference. Ch226 covers GEMM algorithmic variants (Winograd, Strassen, tiled blocked GEMM) that underlie both cuBLAS and rocBLAS implementations.

**Ch227 (Sampling in Diffusion Schedulers)**: the DDIM and DPM-Solver++ scheduler algorithms (§7) are the diffusion-specific application of the numerical ODE solver methods covered in Ch227, which addresses general GPU-accelerated sampling and probability distribution operations.

**Ch229 (Inference Algorithms Substrate)**: the foundational inference compute graph concepts — operator fusion, graph rewriting for inference, quantization calibration pipelines — developed in Ch229 underlie the MIGraphX graph compiler (§9) and the torch.compile/inductor path for diffusion models (§8).

**Ch124 (Local LLM Inference on Linux)**: the deployment-layer counterpart to this chapter. Ch124 covers GGML executor internals, GGUF memory-mapped loading, Ollama server and GPU detection detail, ONNX Runtime Execution Providers, and KV-cache management strategies at the runtime API level. This chapter (Ch232) focuses on the algorithms and kernels that those runtimes implement: FlashAttention tiling, quantization schemes, speculative decoding, diffusion schedulers, and the ROCm compute library tier.
