# Chapter 229: GPU Machine Learning Inference Algorithms (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers integrating neural super-resolution, denoising, or style transfer into GPU rendering pipelines; systems developers building ML inference runtimes on Linux with ROCm, CUDA, or Vulkan compute. Readers of this chapter are assumed to know GPU compute fundamentals (Ch221) and GEMM tiling (Ch226). For the runtime and deployment layer — GGUF weight loading, vLLM PagedAttention internals, ONNX Runtime execution provider configuration, llama.cpp Vulkan startup — see Ch124 (Local LLM Inference on Linux GPUs), which covers those topics in full; this chapter focuses on the underlying algorithms, their GPU mapping, and their mathematical structure.

---

## Table of Contents

1. [ML Inference as a GPU Algorithm Domain](#1-ml-inference-as-a-gpu-algorithm-domain)
2. [Dense Layer and GEMM Inference](#2-dense-layer-and-gemm-inference)
3. [Transformer Attention Mechanisms](#3-transformer-attention-mechanisms)
4. [Quantization Algorithms](#4-quantization-algorithms)
5. [Speculative Decoding and Decode Acceleration](#5-speculative-decoding-and-decode-acceleration)
6. [Convolutional Neural Network Inference](#6-convolutional-neural-network-inference)
7. [Graph Optimisation and Kernel Fusion](#7-graph-optimisation-and-kernel-fusion)
8. [Neural Rendering Inference on GPU](#8-neural-rendering-inference-on-gpu)
9. [Inference Runtimes on Linux](#9-inference-runtimes-on-linux)
10. [Vulkan Compute for ML Inference](#10-vulkan-compute-for-ml-inference)
11. [Memory Management for Inference](#11-memory-management-for-inference)
12. [Profiling and Optimising Inference](#12-profiling-and-optimising-inference)
13. [Integrations](#13-integrations)

---

## 1. ML Inference as a GPU Algorithm Domain

### 1.1 Training vs. Inference

Training and inference differ in their arithmetic structure in ways that change almost every algorithm decision. Training computes forward and backward passes; every activation must be materialised and retained for the backward gradient computation. Inference executes only the forward pass, so intermediate activations can be freed immediately after they are consumed, allowing aggressive memory reuse. The batch size is also different: training uses large batches (32–512) to amortise the per-element DRAM cost of weight loading and to saturate FLOP units, whereas interactive inference often serves a single user request at a time (batch size 1).

### 1.2 Arithmetic Intensity at Inference Time

The **roofline model** (Ch221 §1) maps workloads onto two axes: arithmetic intensity (FLOPs per byte of DRAM traffic) and attainable performance. Inference falls into two regimes determined by batch size:

**Batch-1 decode (latency-bound).** Each token generation step is a matrix–vector multiply: the query/key/value projections multiply a single token's hidden state (shape `[1, d_model]`) against each weight matrix (shape `[d_model, d_head]`). The weight matrix must be loaded from DRAM once per step; there is almost no data reuse. For a 7B-parameter model with FP16 weights, generating each token loads roughly 14 GB of data (2 bytes × 7 × 10⁹ parameters), performing approximately 14 × 10⁹ multiply-adds — an arithmetic intensity of ~1 FLOP/byte. This places the workload far left of every modern GPU's ridge point (typically 60–160 FLOPs/byte), making token generation memory-bandwidth-bound at batch 1. Larger models are proportionally worse; quantisation to INT4 (0.5 bytes/parameter) quadruples the effective bandwidth available per FLOP and is the principal technique for recovering throughput.

**Large-batch prefill (throughput-bound).** During the prefill phase (processing the prompt), all tokens are available simultaneously. The computation becomes a matrix–matrix multiply (GEMM) of shape `[seq_len, d_model] × [d_model, d_ff]`, where `seq_len` may be thousands. With seq_len ≥ 512, the arithmetic intensity rises above most GPU ridge points and the workload becomes compute-bound, saturating tensor cores at 80–95% of peak throughput. Prefill is the regime where quantisation provides less benefit because weight-loading cost is amortised across many tokens.

### 1.3 Linux ML Inference Runtime Landscape

The Linux ecosystem for GPU inference as of mid-2026 spans several distinct stacks:

| Runtime | Primary GPU backend | Quantization | Continuous batching |
|---|---|---|---|
| llama.cpp | Vulkan, ROCm HIP, CUDA | GGUF Q4–Q8, K-quants | Partial (llama-server) |
| vLLM | CUDA, ROCm | GPTQ, AWQ, FP8 | Full (PagedAttention) |
| ONNX Runtime | CUDA EP, MIGraphX EP, OpenVINO EP | INT8 static/dynamic | No |
| TensorRT | CUDA only | INT8 calibration, FP8 | Engine-level |
| mlc-llm | TVM/ROCm, Vulkan, Metal | GGUF, NF4 | Emerging |
| MIGraphX | ROCm | INT8 | No |
| OpenVINO | Intel GPU (Level Zero/OpenCL) | INT8, INT4 | No |

See Ch124 for deployment configuration of each runtime. The remainder of this chapter covers the algorithms these runtimes implement.

---

## 2. Dense Layer and GEMM Inference

### 2.1 Fully-Connected Layer as Batched GEMM

A fully-connected (dense) layer computes `Y = X W^T + b` where X has shape `[N, C_in]`, W has shape `[C_out, C_in]`, and Y has shape `[N, C_out]`. This is a standard GEMM with M=N, K=C_in, N=C_out. On ROCm the operation dispatches through `rocblas_gemm_ex`, which accepts separate input, compute, and output types — enabling INT8 multiply with INT32 accumulate and FP16 output in a single call.

```cpp
// Mixed-precision GEMM for INT8 inference on ROCm
rocblas_gemm_ex(
    handle,
    rocblas_operation_none,        // op_a
    rocblas_operation_transpose,   // op_b (W stored row-major, transposed for GEMM)
    N, C_out, C_in,                // M, N, K
    &alpha,
    X_int8,  rocblas_datatype_i8_r,  C_in,   // A, type, lda
    W_int8,  rocblas_datatype_i8_r,  C_in,   // B, type, ldb
    &beta,
    Y_fp16,  rocblas_datatype_f16_r, C_out,  // C, type, ldc
    Y_fp16,  rocblas_datatype_f16_r, C_out,  // D (output)
    rocblas_datatype_i32_r,                  // compute type (INT32 accumulate)
    rocblas_gemm_algo_standard,
    0, 0
);
```

[Source: rocBLAS API reference, https://rocm.docs.amd.com/projects/rocBLAS/en/latest/]

### 2.2 im2col Conv2D Lowering to GEMM

Convolutional layers are traditionally lowered to GEMM via the **im2col** transformation. For a convolution with input `[N, C_in, H, W]`, kernel `[C_out, C_in, R, S]`:

1. **im2col**: expand each `R×S` input patch at every spatial position into a single row of a matrix, producing shape `[N×H'×W', C_in×R×S]`.
2. **GEMM**: multiply by the weight matrix `[C_out, C_in×R×S]`, producing `[N×H'×W', C_out]`.
3. **col2im** (implicit): reshape output to `[N, C_out, H', W']`.

The im2col step duplicates data by a factor of `R×S` (for a 3×3 kernel, 9×), increasing the memory working set. For large models this cost is acceptable because it enables the use of highly optimised BLAS GEMM kernels. For small kernels running at inference batch 1, the copy overhead dominates; Winograd is preferred.

### 2.3 Winograd Convolution for Small Kernels

Winograd's minimal filtering algorithm, applied to convolutional neural networks by Lavin and Gray [Source: "Fast Algorithms for Convolutional Neural Networks", Lavin & Gray, CVPR 2016, https://arxiv.org/abs/1509.09308], reduces the number of multiplications needed for a 3×3 convolution at the cost of additional additions.

For the 2D tile F(m×m, 3×3) — producing an m×m output tile from a 3×3 filter — the algorithm transforms the filter and input into a domain where element-wise multiplication replaces the convolution, then inverse-transforms the result:

```
Y = A^T [(G g G^T) ⊙ (B^T d B)] A
```

where `g` is the filter, `d` is the input tile, `⊙` is element-wise multiplication, and A, B, G are fixed transform matrices.

**Multiplication counts per 2D output tile:**

| Algorithm | Output tile | Transform domain size | Multiplications | Direct convolution (reference) | Reduction |
|---|---|---|---|---|---|
| F(2×2, 3×3) | 2×2 = 4 outputs | 4×4 = 16 elements | **16** | 4 × 9 = 36 | **2.25×** |
| F(4×4, 3×3) | 4×4 = 16 outputs | 6×6 = 36 elements | **36** | 16 × 9 = 144 | **4×** |

F(4×4, 3×3) achieves 4× fewer multiplications than direct convolution at the cost of larger transform matrices and increased additions. In practice it is preferred for inference on GPU because modern GPUs have higher FLOP throughput than memory bandwidth; the additional additions cost less than the multiplications saved. For large kernels (5×5, 7×7) Winograd transform matrices become numerically ill-conditioned and the approach is not used; direct convolution or FFT-based methods are preferred there.

### 2.4 Depthwise Separable Convolution

The MobileNet family decomposes a standard convolution into:

1. **Depthwise convolution**: apply one filter per input channel independently (shape `[C, 1, R, S]`).
2. **Pointwise convolution**: 1×1 convolution that mixes channels (shape `[C_out, C_in, 1, 1]`).

A standard 3×3 convolution on a C-channel feature map performs `C_out × C_in × H × W × 9` multiply-adds. The depthwise+pointwise factorisation performs `C × H × W × 9 + C_out × C_in × H × W`, a reduction by factor `1/C_out + 1/9 ≈ 1/9` for typical architectures.

On GPU the pointwise 1×1 convolution is a standard GEMM. The depthwise convolution has very low arithmetic intensity (each filter has only R×S weights shared across all spatial positions of one channel) and is typically memory-bound even at large batch sizes. Optimised depthwise kernels store the filter in registers and stream the input, achieving close to memory bandwidth saturation.

---

## 3. Transformer Attention Mechanisms

### 3.1 Scaled Dot-Product Attention

The standard attention computation for a single head is:

```
Attention(Q, K, V) = softmax(Q K^T / sqrt(d_k)) V
```

For seq_len N and head dimension d_k, the full pipeline is:

1. **Q/K/V projection GEMMs**: shape `[N, d_model] × [d_model, d_head]` → three outputs of shape `[N, d_head]`. This is compute-bound for large N.
2. **Score matrix**: `S = Q K^T`, shape `[N, N]`. Arithmetic intensity = N/6 FLOPs/byte, compute-bound for N ≥ ridge_point × 6.
3. **Softmax**: row-wise `exp(S) / sum(exp(S))` — memory-bound, requires a two-pass scan.
4. **Context GEMM**: `O = softmax(S) V`, shape `[N, N] × [N, d_head]` — the N×N score matrix must be read from HBM, which is the primary memory cost of standard attention.

Steps 2–4 require materialising the N×N score matrix, which costs O(N²) HBM space. For N=8192 and FP16, this is 8192² × 2 bytes = 128 MB per head. This is the problem FlashAttention solves.

### 3.2 FlashAttention: Tiled Online Softmax

FlashAttention [Source: "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness", Dao, Fu, Ermon, Rudra, Ré, NeurIPS 2022, https://arxiv.org/abs/2205.14135] avoids materialising the full N×N score matrix by using **tiled online softmax**: the attention computation is split into tiles that fit in SRAM, and the softmax normalisation is applied incrementally across tiles.

The key mathematical insight is the **online softmax** algorithm. For a row of scores `[s₁, s₂, ..., sₙ]`, instead of computing the full softmax in two passes (first find max, then normalise), maintain running statistics:
- `m_i = max(s_1, ..., s_i)` (running maximum)
- `l_i = sum(exp(s_j - m_i) for j=1..i)` (running normalised sum)

When processing a new block `[s_{i+1}, ..., s_{i+B}]`, update:
```
m_new = max(m_i, max(s_{i+1..i+B}))
l_new = exp(m_i - m_new) * l_i + sum(exp(s_j - m_new))
O_new = exp(m_i - m_new) * O_i + exp(S_block - m_new) V_block
```

This allows the output O to be accumulated block by block in SRAM, with the score matrix tiles computed and discarded immediately. The HBM traffic is reduced from O(N²) to O(N), enabling attention on sequences that would otherwise not fit.

```glsl
// Conceptual FlashAttention tile loop — see real SPIR-V in llama.cpp
// For each Q tile (BLOCK_M rows of Q):
//   Initialise m = -inf, l = 0, O = 0
//   For each KV tile (BLOCK_N rows of K, V):
//     S = Q_tile @ K_tile^T / sqrt(d_k)   // [BLOCK_M, BLOCK_N] in SRAM
//     m_new = max(m, rowmax(S))
//     l_new = exp(m - m_new) * l + rowsum(exp(S - m_new))
//     O = O * exp(m - m_new) + exp(S - m_new) @ V_tile
//     m, l = m_new, l_new
//   O = O / l   // final normalisation
```

### 3.3 FlashAttention-2: Improved Work Partitioning

FlashAttention-2 [Source: "FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning", Tri Dao, ICLR 2024, https://arxiv.org/abs/2307.08691] achieves roughly 2× speedup over FlashAttention-1 through three changes:

1. **Reduced non-matmul FLOPs**: the algorithm is restructured to minimise the rescaling operations needed at each tile boundary, reducing the number of element-wise multiply operations on the running sum `l`.
2. **Parallelism over sequence length**: the outer loop over Q tiles is distributed across thread blocks even for a single attention head, increasing occupancy on heads with small d_k.
3. **Improved warp work partitioning**: within each thread block, warps are partitioned to share Q in registers and tile over KV, reducing shared-memory traffic.

FlashAttention-2 reaches 50–73% of theoretical peak FLOPs/s on A100, compared to roughly 30% for FlashAttention-1.

### 3.4 Grouped-Query Attention and Multi-Query Attention

Standard multi-head attention (MHA) uses H independent K/V projection heads for H query heads. **Multi-query attention (MQA)** uses a single K/V head shared across all H query heads, reducing the KV-cache size by H× at the cost of some model quality. **Grouped-query attention (GQA)** [Source: "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints", Ainslie et al., EMNLP 2023, https://arxiv.org/abs/2305.13245] interpolates: G groups each share one K/V head, so H/G K/V matrices total. Llama-3 (8B) uses 8 KV heads for 32 query heads (G=4). Falcon-7B is an example of true MQA (1 KV head for all query heads).

**KV-cache layout for autoregressive decode.** During generation, K and V tensors accumulate one entry per generated token. The layout must support appending one token at a time (O(1)) and reading the full cache history for attention (linear scan). Contiguous row-major layout `[max_seq_len, num_kv_heads, d_head]` works for fixed-size caches but wastes memory when sequences vary in length. See Ch124 §9 for the PagedAttention approach to fragmentation.

---

## 4. Quantization Algorithms

### 4.1 Post-Training Quantization Basics

Post-training quantization (PTQ) maps floating-point weights (or activations) to lower-bit integers after training. The two primary schemes are:

**Symmetric INT8**: the range `[-max, +max]` is mapped uniformly to `[-127, 127]`. A single scale factor `s = max / 127` converts between domains. No zero-point offset needed; INT8 multiply reduces to shift operations.

**Asymmetric INT8**: the range `[min, max]` is mapped to `[0, 255]` with a scale `s` and zero-point `z = round(-min/s)`. Supports non-zero-centred distributions (e.g., ReLU outputs that are always ≥ 0) without wasting half the dynamic range.

**Per-tensor vs. per-channel vs. per-group:**

- *Per-tensor*: one scale per weight matrix. Fast but imprecise when weight magnitudes vary across output channels.
- *Per-channel*: one scale per output channel of a weight matrix. Standard for INT8 conv/linear inference; used in TensorRT and ONNX Runtime INT8 mode.
- *Per-group*: one scale per `G` consecutive weights within a row (commonly G=64 or G=128). Enables INT4 weight storage with fine-grained scales stored in FP16; used in GGUF K-quant formats.

### 4.2 GPTQ: Hessian-Based Weight Rounding

GPTQ [Source: "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers", Frantar, Ashkboos, Hoefler, Alistarh, ICLR 2023, https://arxiv.org/abs/2210.17323] is a one-shot weight quantization method based on second-order (Hessian) information from the Optimal Brain Compression framework. The core idea: when quantizing weight `w_i`, choose the nearest quantized value that minimises the reconstruction error on the layer output, given the layer's input calibration data. Remaining unquantized weights in the same row are adjusted to compensate.

The Hessian H = 2 X^T X (where X is the calibration input to the layer) captures how sensitive the output is to each weight. GPTQ processes weights column by column, updating a Cholesky decomposition of H^{-1} as each column is quantized:

```
ΔW = (w_j - quant(w_j)) / [H^{-1}]_{jj} * H^{-1}_{:,j}
```

This error-compensation step ensures that quantizing one weight propagates a corrective update to subsequent weights. GPTQ can quantize a 175B parameter model to 4-bit in approximately 4 GPU hours, enabling it to run on a single high-end GPU that would otherwise require multiple nodes for FP16 inference.

### 4.3 AWQ: Activation-Aware Weight Quantization

AWQ [Source: "AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration", Lin, Tang et al., MLSys 2024 Best Paper, https://arxiv.org/abs/2306.00978] observes that not all weight channels are equally important for model output. By examining activation magnitudes across calibration data, AWQ identifies the 1% of weight channels with the highest corresponding activation values — these are the "salient" channels. Rather than quantizing them at higher precision (which breaks hardware uniformity), AWQ scales each salient weight channel before quantization:

```
quant(w * s) / s ≈ w for salient channels
```

where `s = mean(|X_channel|)^α` (α typically 0.5) is computed from calibration activation statistics. This mathematically equivalent scaling operation reduces the per-channel quantization error without mixed-precision layouts. AWQ achieves state-of-the-art INT4 quality without retraining and is hardware-friendly since the actual memory layout is uniform INT4 throughout.

### 4.4 SmoothQuant: Migrating Outliers from Activations to Weights

SmoothQuant [Source: "SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models", Xiao, Lin, Seznec, Wu, Demouth, Han, ICML 2023, https://arxiv.org/abs/2211.10438] targets **activation quantization**, which is harder than weight quantization because activations have per-token outlier values that can be 100× larger than typical values, causing high quantization error.

The key observation: if activation channel `c` has large values, the corresponding weight channel can be made correspondingly small via a scale migration:

```
Y = (X diag(s)^{-1}) (diag(s) W) = X̃ W̃
```

where `s_c = max(|X_c|)^α / max(|W_c|)^{1-α}`. The smoothed activations X̃ = X diag(s)^{-1} have smaller outliers and quantize accurately to INT8; the smoothed weights W̃ = diag(s) W absorb the scaling and are quantized offline. This enables W8A8 quantization (both weights and activations in INT8), allowing use of INT8 tensor core units in hardware that supports INT8 input tensors. SmoothQuant demonstrated up to 1.56× speedup and 2× memory reduction.

### 4.5 INT4 Packing and Mixed-Precision GEMM

INT4 weights store two values per byte. On NVIDIA and AMD hardware, a mixed-precision GEMM with INT4 weights proceeds as:

1. **Dequantize** INT4 → FP16 or BF16 in a fast register-level unpack operation (each group of 8 INT4 values → 8 FP16 values, with scale applied).
2. **FP16/BF16 GEMM** using tensor cores.

True INT4 multiply-accumulate hardware (performing 8-bit INT4 GEMM natively without dequantization) exists on NVIDIA Hopper (FP8/INT8 native) and upcoming CDNA4, but most 2024–2026 inference pipelines use the dequantize-then-FP16-GEMM path. The bandwidth advantage of INT4 (0.5 bytes/weight vs. 2 bytes for FP16) is retained because the decompression is done in registers on-chip; only the INT4 bytes cross the HBM interface.

---

## 5. Speculative Decoding and Decode Acceleration

### 5.1 Speculative Decoding

Speculative decoding [Source: "Fast Inference from Transformers via Speculative Decoding", Leviathan, Kalman, Matias, ICML 2023, https://arxiv.org/abs/2211.17192] uses a small **draft model** to generate K candidate tokens in parallel, then runs the full **verifier model** once across all K+1 positions simultaneously. The verifier accepts or rejects each draft token through a rejection sampling procedure that preserves the exact verifier output distribution. Because the verifier processes K tokens in one forward pass (a matrix–matrix operation with favorable arithmetic intensity) instead of K sequential single-token steps (K matrix–vector operations, each memory-bound), the throughput can improve 2–3× when most draft tokens are accepted.

The practical bottleneck: the draft model must be small enough that its K forward passes cost less than the saved verifier passes. A draft model 10–20× smaller than the verifier is typical; the pair must share a vocabulary.

### 5.2 Medusa: Multiple Draft Heads

Medusa [Source: "Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads", Cai, Li, Geng, Peng, Lee, Chen, Dao, https://arxiv.org/abs/2401.10774] eliminates the separate draft model by appending multiple prediction heads to the base model. Each head predicts one additional future token position independently. The set of predictions from all heads forms a **token tree**, which the base model verifies using tree attention (masking the attention matrix so each candidate sees only its causal ancestor in the tree). Accepted paths are extracted, and the KV cache is updated for the chosen continuation.

Medusa-1 (heads fine-tuned with frozen backbone) achieves 2.2× speedup; Medusa-2 (heads and backbone jointly fine-tuned) achieves 2.3–3.6×.

### 5.3 Continuous Batching and Iteration-Level Scheduling

Standard batching groups requests that start and end together, leaving GPUs idle while short requests in a batch finish early. **Iteration-level scheduling** (Orca, Yu et al., OSDI 2022 [Source: https://www.usenix.org/conference/osdi22/presentation/yu]) inserts new requests at any generation step: when request A finishes token K, a new request B can be added to the active batch immediately for step K+1 rather than waiting for the full batch to complete. This raises GPU utilisation from ~40% to >90% under mixed-length workloads. vLLM implements iteration-level scheduling alongside PagedAttention; see Ch124 §9 for the full BlockManager implementation.

### 5.4 Lookahead Decoding

Lookahead decoding [Source: "Break the Sequential Dependency of LLM Inference Using Lookahead Decoding", Fu et al., ICML 2024, https://arxiv.org/abs/2402.02057] maintains a 2D window of candidate token n-grams drawn from a Jacobi iteration, allowing parallel verification of multiple hypotheses without a draft model. The algorithm has higher per-step cost than standard decoding but generates multiple verified tokens per step. Performance advantage depends on the n-gram hit rate, which varies with model type and prompt style.

---

## 6. Convolutional Neural Network Inference

### 6.1 Batch Normalisation Fusion

Batch normalisation computes:

```
y = (x - μ_B) / sqrt(σ_B² + ε) * γ + β
```

At inference time, `μ_B` and `σ_B` are fixed (computed from the training population statistics). This means the BN transformation reduces to a per-channel affine operation: `y = x * scale + bias` where `scale = γ / sqrt(σ² + ε)` and `bias = β - μ * scale`. **Folding BN into the preceding convolution** absorbs these constants into the convolution weight and bias offline:

```
W_fused[c] = W[c] * scale[c]
b_fused[c] = b[c] * scale[c] + bias[c]
```

The result: the BN layer disappears entirely from the inference graph, saving one pass over the activation tensor per fused layer. For a ResNet-50 with 53 BN layers, this eliminates 53 global memory read-write passes. TensorRT performs this fusion automatically during engine build; ONNX Runtime applies it via the graph optimizer before dispatching to an execution provider.

### 6.2 Activation Fusion

ReLU, GELU, SiLU, and similar pointwise activations have arithmetic intensity ≈ 0.5 FLOPs/byte and are purely memory-bandwidth-bound. Fusing them into the preceding GEMM or convolution kernel — executing the activation inline on register values before writing the result to HBM — eliminates one read-write round trip per fused layer. This is universally done in inference kernels; TensorRT, MIGraphX, and cuDNN all expose fused conv+bias+activation primitives.

### 6.3 Pooling and Reduction

**Global average pooling (GAP)**, used at the end of most CNN backbones before the classifier, reduces a `[N, C, H, W]` activation tensor to `[N, C]` by averaging spatially. On GPU this is a parallel reduction: each channel's H×W values are summed in shared memory using a tree reduction (log₂(H×W) steps), then divided by H×W. For typical feature map sizes (7×7 = 49 elements) the reduction fits in a single thread block warp and completes in a handful of synchronisations.

### 6.4 Depthwise Conv Tiling

Depthwise convolutions (§2.4) have one filter of shape `[R, S]` per input channel. With a 3×3 filter:

```glsl
// Depthwise conv compute shader (simplified)
layout(local_size_x=8, local_size_y=8) in;
layout(set=0,binding=0) readonly buffer Input  { float x[]; }; // [N,C,H,W]
layout(set=0,binding=1) readonly buffer Filter { float w[]; }; // [C,3,3]
layout(set=0,binding=2) writeonly buffer Out   { float y[]; }; // [N,C,H',W']

shared float tile[10][10]; // 8×8 output + 1-pixel halo for 3×3 filter

void main() {
    ivec2 out_xy = ivec2(gl_GlobalInvocationID.xy);
    int c = int(gl_GlobalInvocationID.z);
    // Load 10×10 input tile into shared memory (8×8 output + halo)
    // ...barrier()...
    float acc = 0.0;
    for (int ky=0; ky<3; ky++)
        for (int kx=0; kx<3; kx++)
            acc += tile[gl_LocalInvocationID.y+ky][gl_LocalInvocationID.x+kx]
                 * w[c*9 + ky*3 + kx];
    y[/*index*/] = acc;
}
```

The filter `w` (9 floats per channel) fits in L1 cache across the entire thread group and is broadcast to all 64 threads, making this kernel's bottleneck the input tile loads from shared memory.

---

## 7. Graph Optimisation and Kernel Fusion

### 7.1 Constant Folding and Dead Code Elimination

Before any kernel fusion, an inference runtime traverses the operator graph and:

- **Constant folds**: any operator whose inputs are all compile-time constants (e.g., a `Reshape` of a fixed weight tensor) is evaluated offline and replaced with the constant output.
- **Dead code eliminates**: any operator whose output is not consumed by any reachable node (e.g., training-only logging ops left in an exported ONNX model) is removed.

These passes reduce the graph size and expose fusion opportunities that were blocked by intervening non-fuseable ops.

### 7.2 Vertical Fusion (Elementwise Chains)

A chain of elementwise ops — `add → multiply → clip → cast` — can be fused into a single kernel: one global read of the input, the chain of ALU operations, one global write. The arithmetic intensity of the chain is still O(1) FLOPs/byte, but HBM bandwidth is used only once instead of 4×. On modern GPUs with 500–1000 GB/s HBM bandwidth, each saved round trip is ~1 ms per GB of activation data.

TVM and XLA perform vertical fusion by pattern matching chains of fuseable op types. In MLIR-based pipelines (IREE), the Linalg dialect represents convolutions and element-wise ops uniformly, and the `linalg-fusion-on-tensors` pass fuses producer–consumer pairs where the producer's output is consumed by exactly one consumer.

### 7.3 Horizontal Fusion and Sub-graph Pattern Matching

**Horizontal fusion** (also called *operator batching* or *widening*) merges independent ops of the same type that share an input into a single wider kernel. Example: in the Q/K/V projection of attention, three GEMMs `[N,d] × [d,d_head]` share the same input X. A fused kernel computes all three projections with a single read of X and three writes, trading one read pass for two.

**Sub-graph pattern matching** identifies recurring structural patterns — Conv+BN+ReLU, Attention+LayerNorm, SkipLayerNorm — and replaces them with hand-optimised fused implementations. TensorRT's graph optimizer has an extensive library of such patterns; ONNX Runtime TransformerOptimizer similarly targets BERT-type `SkipLayerNorm`, `FusedMatMul`, `FastGelu`, and `MultiHeadAttention` patterns.

### 7.4 MLIR and IREE for Inference Graph Compilation

IREE (Intermediate Representation Execution Environment) is an MLIR-based ML compiler that targets multiple GPU backends from a unified compilation pipeline. The compilation path from a model to GPU kernels:

1. **Import**: TensorFlow, PyTorch, JAX, or ONNX models are converted to MLIR in the `torch` or `stablehlo` dialect.
2. **Dispatch region formation**: the `iree-global-opt` and `flow` dialect passes segment the graph into regions that will become GPU dispatch calls.
3. **Lowering to `linalg` / `vector` dialect**: convolutions and GEMMs become tiled vector operations.
4. **Backend codegen**: the `hal` (hardware abstraction layer) dialect generates SPIR-V for Vulkan or HSA kernels for ROCm.

[Source: IREE project documentation, https://iree.dev/guides/ml-frameworks/; IREE supports GPU-ROCm, GPU-Vulkan, GPU-CUDA, and GPU-Metal backends.]

### 7.5 TVM AutoTVM and MetaSchedule on ROCm

Apache TVM compiles ML models to optimised kernels for a target backend using a search-based scheduler. The workflow for ROCm:

```bash
# Tune a ResNet-50 for AMD MI250X
python -m tvm.driver.tvmc tune \
    --target "rocm -model=gfx90a" \
    --output resnet50_rocm.json \
    resnet50.onnx

python -m tvm.driver.tvmc compile \
    --target "rocm -model=gfx90a" \
    --tuning-records resnet50_rocm.json \
    --output resnet50_rocm.tar \
    resnet50.onnx
```

TVM's **MetaSchedule** (successor to AutoTVM) uses a cost model trained on hardware traces to propose tile sizes, vectorisation widths, and unroll factors without exhaustive enumeration. On ROCm/GFX targets it generates HIP C++ kernels that are compiled through `hipcc`. mlc-llm uses TVM as its compilation backend, targeting both `rocm` and `vulkan` devices.

### 7.6 XLA/HLO on ROCm

JAX uses XLA as its compilation backend. AMD contributes a ROCm backend to the upstream OpenXLA project, enabling JAX's `jax.jit`-compiled inference to target AMD GPUs. The compilation path goes through XLA's HLO (High-Level Optimizer) IR, which applies `kFusion` to merge element-wise ops around a root op and lowers `kWhile` loops to persistent GPU kernels. The ROCm backend generates HIP kernels via the XLA GPU codegen pipeline. [Source: OpenXLA/XLA repository, https://github.com/openxla/xla; ROCm support: https://rocm.docs.amd.com/en/latest/how-to/jax-install.html]

---

## 8. Neural Rendering Inference on GPU

### 8.1 NeRF Volume Rendering

Neural Radiance Fields represent a scene as a function `(x,y,z,θ,φ) → (RGB, σ)` encoded in an MLP. Rendering a single pixel requires marching a ray through the scene, evaluating the MLP at N sample points along the ray, and compositing the resulting colours using volume rendering:

```
C = Σ_i T_i (1 - exp(-σ_i δ_i)) c_i
where T_i = exp(-Σ_{j<i} σ_j δ_j)   (accumulated transmittance)
```

The MLP evaluation at N sample points per ray dominates compute. For a small MLP (8 layers, 256 hidden units) the bottleneck at inference is launching N×pixels_per_batch separate MLP evaluations — a heavily batched dense GEMM workload. Hash-encoded NeRF variants (Instant-NGP) replace some MLP layers with a multi-resolution feature hash grid stored in GPU memory, trading compute for memory lookups; Instant-NGP achieves interactive frame rates for trained scenes; exact throughput depends on scene complexity, network size, and hardware. [Source: "Instant Neural Graphics Primitives", Müller et al., SIGGRAPH 2022, https://nvlabs.github.io/instant-ngp/]

### 8.2 3D Gaussian Splatting: Spherical Harmonic Evaluation

3DGS rasterisation is covered in Ch212 (§91–94). This section focuses on the spherical harmonic (SH) colour evaluation step, which is pure MLP-free inference. Each Gaussian stores SH coefficients for view-dependent RGB. Degree-3 SH has 16 coefficients per colour channel (48 floats total per Gaussian). Evaluating the colour for a given view direction `d` is:

```glsl
vec3 eval_sh(vec3 d, vec4 sh[12]) { // 12 vec4s = 48 coefficients
    // Degree 0 (1 coeff)
    vec3 c = 0.2820948 * sh[0].rgb;
    // Degree 1 (3 coeffs)
    c += 0.4886025 * (-d.y * sh[1].rgb + d.z * sh[2].rgb - d.x * sh[3].rgb);
    // Degree 2 (5 coeffs) ...
    // Degree 3 (7 coeffs) ...
    return max(c + 0.5, vec3(0.0)); // sigmoid-like offset
}
```

This evaluation is a dot product of 48 coefficients against 48 pre-computed SH basis values — 48 multiply-adds. For 1M Gaussians visible in a frame, the SH evaluation is 48M multiply-adds, negligible compared to the sorting and tile rasterisation.

### 8.3 Diffusion Model UNet Inference

Latent diffusion models (Stable Diffusion, SDXL) perform T denoising steps (typically 20–50 for DDIM, 10–25 for DPM-Solver++) each executing a UNet forward pass. The UNet has:

- **Encoder blocks**: conv+attention at multiple spatial resolutions.
- **Bottleneck**: full cross-attention between latent tokens and text conditioning (CLIP embeddings).
- **Decoder blocks**: mirror of encoder with skip connections.

At 512×512 output (64×64 latent), the bottleneck cross-attention has Q dimension 4096 (64×64), K/V dimension 77 (CLIP context). This is a small-N attention that fits entirely in SRAM and does not require FlashAttention tiling. The dominant cost is the 2D convolutions in encoder/decoder blocks.

**VAE encode/decode** converts pixel images to/from the latent space. The VAE decoder is a pure convolutional network with upsampling; it runs as a standard CNN inference graph amenable to BN folding and activation fusion (§6.1–§6.2).

### 8.4 DDIM and DPM-Solver++ Schedulers

The denoising schedulers are CPU-side algorithms that select the noise levels `σ_t` for each step and compute the Euler update to the latent. DPM-Solver++ [Note: algorithm described in "DPM-Solver++: Fast Solver for Guided Sampling of Diffusion Probabilistic Models", Lu et al., 2022, https://arxiv.org/abs/2211.01095] reduces the required number of UNet evaluations from 50+ (DDPM) to 10–25 through a higher-order ODE solver that exploits the semi-linear structure of the diffusion SDE. The scheduler computation itself is negligible (a few floating-point updates per step); the GPU inference cost is entirely in the UNet forward passes.

---

## 9. Inference Runtimes on Linux

*This section provides an algorithm-focused summary; see Ch124 for runtime configuration, GGUF loading, and execution provider setup.*

### 9.1 ROCm MIGraphX

MIGraphX is AMD's native inference graph compiler for ROCm. It accepts ONNX models and applies a pipeline of graph optimisations — constant propagation, dead code elimination, operator fusion, layout optimisation — before generating ROCm kernels via MIOpen (convolutions) and rocBLAS (GEMMs). Key capabilities:

- **INT8 quantization**: calibration-based per-tensor INT8 using a calibration dataset; fused INT8 conv+relu.
- **Operator fusion**: Conv+BN+Activation, Attention+LayerNorm fused patterns.
- **hiprtc JIT**: for small specialised kernels not covered by MIOpen, MIGraphX uses `hiprtc` (runtime compilation) to generate and compile HIP C++ at model load time.

As of ONNX Runtime 1.23, the ROCm EP is deprecated; users are directed to the MIGraphX EP instead. [Source: ONNX Runtime documentation, https://onnxruntime.ai/docs/execution-providers/ROCm-ExecutionProvider.html]

### 9.2 TensorRT

NVIDIA TensorRT applies layer fusion, precision calibration (INT8, FP8 on Hopper), and kernel auto-selection at engine build time. The engine is serialised to a `.trt` file and deserialized for repeated inference without rebuilding. Key algorithms: overlap of DMA and compute using CUDA Graphs, tactic selection (choosing among cuDNN/cuBLAS/custom kernels for each layer by benchmarking all candidates), and sparsity exploitation (2:4 structured sparsity on Ampere+). TensorRT is CUDA-only; for AMD, MIGraphX is the functional equivalent.

### 9.3 OpenVINO for Intel GPUs

OpenVINO targets Intel GPUs via the Level Zero backend (XeGPU execution units) and Intel's NPU via a separate driver. The compilation pipeline: OpenVINO IR → Intel Graphics Compiler (IGC) → GPU binary. The `OrtOpenVINOProviderOptions` allows setting `"device_type": "GPU"` or `"NPU"` with heterogeneous fallback. INT8 quantization via Post-Training Optimization (POT) or NNCF achieves 2–4× memory reduction and 1.5–2× throughput improvement on Arc GPUs.

---

## 10. Vulkan Compute for ML Inference

### 10.1 VK_KHR_cooperative_matrix

`VK_KHR_cooperative_matrix` (Vulkan 1.3 extension) exposes matrix-multiply-accumulate (MMA) operations on subgroup-level cooperative matrices — the Vulkan equivalent of CUDA's WMMA/MMA or ROCm's `__builtin_amdgcn_mfma_*`. A SPIR-V shader using cooperative matrices declares matrix operands with `OpTypeCooperativeMatrixKHR` and issues a multiply-add with `OpCooperativeMatrixMulAddKHR`.

Tile sizes exposed depend on the driver; query with:

```cpp
uint32_t propCount = 0;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(physicalDevice, &propCount, nullptr);
std::vector<VkCooperativeMatrixPropertiesKHR> props(propCount);
for (auto& p : props) p.sType = VK_STRUCTURE_TYPE_COOPERATIVE_MATRIX_PROPERTIES_KHR;
vkGetPhysicalDeviceCooperativeMatrixPropertiesKHR(physicalDevice, &propCount, props.data());
// Each VkCooperativeMatrixPropertiesKHR has: MSize, NSize, KSize, AType, BType,
//   CType, ResultType, saturatingAccumulation, scope
```

NVIDIA Ampere/Ada drivers expose FP16 A/B with FP32 C at 16×16×16; INT8 A/B with INT32 C at 16×16×32. AMD RDNA3+ drivers expose similar sizes via the extension. The `llama.cpp` Vulkan backend detects cooperative matrix support at startup and selects the `matmul_id_f16` or `matmul_q4_f32_coopmat` shader variants.

### 10.2 Push Constants and Descriptor Layout for Weight Tensors

ML inference kernels bind weight tensors as storage buffers. A minimal descriptor set layout for a GEMM dispatch:

```cpp
// Descriptor set layout: A (weights, read-only SSBO), B (activation), C (output)
VkDescriptorSetLayoutBinding bindings[] = {
    {0, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, 1, VK_SHADER_STAGE_COMPUTE_BIT, nullptr}, // W
    {1, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, 1, VK_SHADER_STAGE_COMPUTE_BIT, nullptr}, // X
    {2, VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, 1, VK_SHADER_STAGE_COMPUTE_BIT, nullptr}, // Y
};
// Push constants carry M, N, K and per-inference parameters
struct PushConstants { uint32_t M, N, K; float scale; };
VkPushConstantRange pcRange = {VK_SHADER_STAGE_COMPUTE_BIT, 0, sizeof(PushConstants)};
```

Weights are loaded once into a device-local `VkBuffer` at model load time and bound to binding 0 for every inference call. Activation tensors are re-bound each call. This avoids per-inference device memory allocation.

### 10.3 Specialization Constants for Tile Sizes

Vulkan pipeline specialization constants allow baking tile sizes into a compiled shader at pipeline creation time without SPIR-V recompilation:

```glsl
layout(constant_id=0) const int BLOCK_M = 64;
layout(constant_id=1) const int BLOCK_N = 64;
layout(constant_id=2) const int BLOCK_K = 16;
layout(local_size_x_id=0, local_size_y_id=1) in;
```

At pipeline creation, `VkSpecializationInfo` bakes in the chosen tile sizes. The llama.cpp Vulkan backend creates multiple pipeline variants per quantisation format (Q4_0, Q4_1, Q8_0, F16) and selects the best based on the query device's subgroup size and cooperative matrix support.

---

## 11. Memory Management for Inference

*This section focuses on algorithmic aspects; see Ch124 §4 for GGUF mmap loading and DMA-BUF integration details.*

### 11.1 Activation Memory Planning

During inference the activation tensors from one layer are consumed by the next and then become dead. An offline **liveness analysis** of the inference graph determines the live range of each tensor (first write to last read). The runtime allocates a single GPU memory pool upfront and assigns each tensor a sub-range using a bump or buddy allocator, with offsets chosen to minimise peak memory usage. Tensors with non-overlapping live ranges share physical memory.

For a ResNet-50 at batch 1 (FP32), liveness-aware memory planning substantially reduces peak activation memory compared to naive allocation, because residual-layer activations can alias with later layers' scratch space once the residual connection has been consumed. [Note: exact figures vary with implementation; verify with your target runtime's memory planner.]

### 11.2 GPU Memory Pools

Repeated `cudaMalloc`/`hipMalloc` calls are expensive (100 µs–1 ms per call) due to driver synchronisation. Production inference runtimes allocate one large pool at startup:

```cpp
// Allocate 4 GB inference pool on ROCm
size_t pool_size = 4ULL << 30;
void* pool_base;
hipMalloc(&pool_base, pool_size);

// Sub-allocate with a simple arena allocator
void* alloc(size_t size) {
    void* ptr = (char*)pool_base + offset;
    offset = (offset + size + 255) & ~255; // 256-byte align
    return ptr;
}
```

This reduces per-inference allocation overhead to a pointer bump. Pool recycling for activation tensors is done by resetting the arena offset between inference calls.

### 11.3 GGUF Format and Quantized Weight Layout

GGUF [Source: GGUF specification in ggml repository, https://github.com/ggerganov/ggml/blob/master/docs/gguf.md] is a binary container format with:
- A header with magic, version, and tensor count.
- A key-value metadata section (architecture name, context length, embedding size, etc.).
- Per-tensor info (name, type, shape, byte offset into data section).
- A 32-byte-aligned tensor data section.

K-quant formats (Q4_K_M, Q6_K) store weights in blocks of 256 values. Each block has a super-scale (FP16), a scale (FP16), and the quantized weights. Loading proceeds via `mmap(2)` with `MADV_SEQUENTIAL` prefetch; the GPU upload copies only the blocks needed for the layers assigned to that GPU.

### 11.4 DMA-BUF for Zero-Copy Neural SR

When inference output (e.g., super-resolved frame from a neural SR network) is consumed by a Vulkan graphics pipeline for display, copying through CPU memory wastes bandwidth. Using `VK_EXT_external_memory_dma_buf`, the inference output buffer can be exported as a DMA-BUF file descriptor and imported as a Vulkan `VkImage` without any CPU-side copy:

```cpp
// Export inference output buffer as DMA-BUF
VkMemoryGetFdInfoKHR fdInfo = {
    VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR, nullptr,
    inferenceOutputMemory, VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT
};
int dmabuf_fd;
vkGetMemoryFdKHR(device, &fdInfo, &dmabuf_fd);

// Import into display/compositor Vulkan instance
VkImportMemoryFdInfoKHR importInfo = {
    VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR, nullptr,
    VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT, dmabuf_fd
};
```

The DMA-BUF fd can also be passed to a KMS display plane directly (Ch207, Ch220), enabling the compositor to scan out neural SR output without staging through CPU memory.

---

## 12. Profiling and Optimising Inference

### 12.1 Roofline for Inference Kernels

Applying the roofline model (Ch221 §1) to inference workloads:

| Operation | Arithmetic intensity | Bottleneck |
|---|---|---|
| Batch-1 token GEMM (7B model) | ~1 FLOPs/byte | Memory bandwidth |
| Batch-32 prefill GEMM | ~32 FLOPs/byte | Transitions to compute |
| INT4 dequantize + GEMM | ~2 FLOPs/byte | Still memory |
| Softmax | ~2 FLOPs/byte | Memory bandwidth |
| ReLU/GELU (elementwise) | ~0.5 FLOPs/byte | Memory bandwidth |
| FlashAttention (SRAM-resident) | High (data reused) | Compute |

Any optimisation that reduces the working set so data stays in L2/SRAM dramatically improves performance.

### 12.2 Profiling Tools on Linux

**NVIDIA (CUDA)**: `nsys profile` captures GPU kernel timelines; `ncu` (Nsight Compute) profiles individual kernels for occupancy, memory transactions, and FLOP utilisation.

```bash
nsys profile --trace=cuda,nvtx -o inference_trace python run_inference.py
nsys stats inference_trace.nsys-rep --report gputrace
```

**AMD (ROCm)**: `rocprof` captures hardware counters; `rpp` (ROCm Profiler Python) profiles kernel durations.

```bash
rocprof --stats -o profile.csv ./inference_binary
# Or for per-kernel hardware counters:
rocprof --hsa-trace --hip-trace ./inference_binary
```

`rocm-smi --showclockfreq` verifies the GPU is running at rated clock, not throttled. `radeontop` shows live GPU utilisation.

### 12.3 Kernel Occupancy and Attention

Attention kernels benefit from high occupancy (many active warps/wavefronts per CU) to hide memory latency during KV cache loads. On AMD RDNA3, each CU has 5 SIMD32 units and 256 KB of LDS. A FlashAttention kernel with BLOCK_M=64, BLOCK_N=64, d_head=128 uses:
- Registers: ~64 per thread for Q/K/V tiles → limits active wavefronts.
- LDS: ~64 KB for shared K/V tiles → limits occupancy on low-LDS CUs.

Tuning tile sizes via TVM MetaSchedule or hand-tuning with `vkGetPhysicalDeviceProperties2` → `maxComputeSharedMemorySize` allows matching tile sizes to specific GPU memory hierarchies.

### 12.4 Quantization Accuracy vs. Latency

| Format | VRAM (7B) | Perplexity Δ | Tokens/s (RX 7900 XTX) |
|---|---|---|---|
| FP16 | ~14 GB | 0 (baseline) | ~20 |
| Q8_0 | ~7.7 GB | +0.1 | ~35 |
| Q4_K_M | ~4.2 GB | +0.3 | ~70 |
| Q3_K_M | ~3.3 GB | +0.8 | ~90 |
| Q2_K | ~2.9 GB | +2.5 | ~100 |

*Representative values from llama-bench; exact numbers vary with model architecture and hardware.*

The Q4_K_M format (4-bit weights with per-group 6-bit scales) is the standard recommendation: it roughly triples token throughput versus FP16 while maintaining acceptable perplexity. Moving to Q2_K risks visible quality degradation in long-form generation.

### 12.5 torch.compile on ROCm

PyTorch's `torch.compile` (using TorchInductor as backend) generates Triton kernels that are compiled to HIP via the `triton-rocm` package. The compilation path:

```python
import torch
model = load_model().to("cuda")  # ROCm uses "cuda" device alias
model = torch.compile(model, backend="inductor", mode="reduce-overhead")
# First call triggers JIT compilation; subsequent calls use cached kernels
output = model(input_ids)
```

On ROCm, `TORCH_COMPILE_DEBUG=1` and `TORCHINDUCTOR_COMPILE_THREADS=1` help diagnose compilation failures. The `"reduce-overhead"` mode uses CUDA Graphs (HIP Graph on ROCm) to capture and replay the kernel launch sequence, eliminating Python dispatch overhead for subsequent calls.

---

## 13. Integrations

This chapter covers the algorithm layer of GPU ML inference. The following chapters address adjacent topics in the Linux graphics stack:

- **Ch207** (TAA, DLSS, FSR, and Neural Upscaling VFX/Postprocess Compute): discusses how neural super-resolution is integrated into the graphics pipeline as a post-process pass; this chapter provides the underlying CNN/attention inference algorithms used by those super-resolution networks.
- **Ch212** (GPU Geometry Algorithms — Neural Geometry and Specialized Primitives): covers 3D Gaussian Splatting rasterisation, NeRF volume rendering as a geometry technique, and geometric deep learning. This chapter provides the MLP/SH evaluation algorithms that implement the colour computation inside those representations.
- **Ch220** (GPU Image Processing Algorithms): covers super-resolution and image enhancement from a signal-processing perspective; neural SR is the ML inference path to the same outputs.
- **Ch221** (GPU Algorithm Performance and Optimization): the roofline model and profiling methodology applied throughout this chapter; read Ch221 §1–3 before interpreting the arithmetic intensity values in §1 and §12 above.
- **Ch223** (GPU Video Processing Algorithms): video super-resolution pipelines apply the CNN inference algorithms of §6 frame-by-frame; temporal consistency techniques (optical flow warping) are covered there.
- **Ch226** (GEMM as Shared Substrate): provides the detailed GPU tiling and shared-memory implementation of GEMM that underlies §2–3 of this chapter; read Ch226 before implementing custom GEMM kernels.
- **Ch228** (Graph Neural Network Inference): GNN inference shares the graph-optimisation techniques of §7; sparse message-passing GEMMs have different arithmetic intensity profiles from dense transformer GEMMs.
- **Ch124** (Local LLM Inference on Linux GPUs): the runtime counterpart to this chapter — covers GGUF loading, llama.cpp Vulkan initialisation, vLLM PagedAttention implementation, ONNX Runtime EP configuration, and ROCm KFD/MIOpen deployment in full.
- **Ch48** (ROCm and ML on Linux) and **Ch108** (ROCm HIP): ROCm driver layer and HIP programming model underlying the ROCm-backend algorithms described here.
