# Chapter 231: GPU Compression Algorithms

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers managing GPU texture memory, mesh streaming, or video codec integration who need to understand how compression formats affect bandwidth and quality; systems developers optimising GPU data pipelines with on-the-fly compression, zero-copy DMA-BUF transfers, and framebuffer compression integration with the DRM display engine.

This chapter surveys the principal GPU compression algorithm families — block texture compression, adaptive scalable texture compression, geometry compression, lossless buffer compression, entropy coding, and framebuffer/display compression — grounding each in real Linux-stack library paths, Vulkan extension usage, and kernel driver integration. Where GPU-accelerated encoding differs substantially from the fixed-function decode path, both are covered.

---

## Table of Contents

1. [Compression as a GPU Algorithm Domain](#1-compression-as-a-gpu-algorithm-domain)
2. [BC/DXT Texture Block Compression](#2-bcdxt-texture-block-compression)
3. [ASTC Texture Compression](#3-astc-texture-compression)
4. [ETC and PVRTC Formats](#4-etc-and-pvrtc-formats)
5. [GPU Texture Compression Encoders](#5-gpu-texture-compression-encoders)
6. [Geometry Compression](#6-geometry-compression)
7. [Lossless GPU Compression](#7-lossless-gpu-compression)
8. [Entropy Coding on GPU](#8-entropy-coding-on-gpu)
9. [Image and Video Codec Entropy Stages](#9-image-and-video-codec-entropy-stages)
10. [Framebuffer and Display Compression](#10-framebuffer-and-display-compression)
11. [GPU-Side Delta and Prediction Compression](#11-gpu-side-delta-and-prediction-compression)
12. [Integration with the GPU Pipeline](#12-integration-with-the-gpu-pipeline)
13. [Integrations](#integrations)

---

## 1. Compression as a GPU Algorithm Domain

GPU memory bandwidth is typically the binding constraint for large rendering workloads. A modern discrete GPU moves VRAM at 500–900 GB/s — impressive, but still finite. Compression trades compute cycles (cheap on modern GPUs) for fewer bytes transferred (expensive in energy, latency, and transistor area). Understanding where that trade-off falls for each algorithm class is the starting point for any GPU compression decision.

### 1.1 Read Bandwidth vs. Compute Trade-off

The relevant figure is not raw VRAM bandwidth but *effective* bandwidth: bytes transferred at the memory controller level after any compression has been applied. Compressed texture sampling does not traverse the shader pipeline at all — the texture unit decompresses the block in fixed-function hardware before interpolation. That path costs essentially zero shader cycles. Compute-shader compression for encoding is different: encoding is iterative and search-intensive, and runs on general shader execution units.

### 1.2 Fixed-Rate vs. Variable-Rate Compression

Block texture formats (BC, ETC, ASTC) are **fixed-rate**: every block of *n×m* texels encodes to exactly the same number of bits regardless of content. Fixed-rate formats support random access at any texel coordinate from a known base address, making them compatible with standard GPU texture sampling hardware. Display/framebuffer compression formats (AFBC, DCC) are **variable-rate**: each tile compresses to a different length, stored alongside a metadata table that records sizes and offsets. Variable-rate formats yield higher compression ratios but require indirection through the metadata table.

### 1.3 Lossy vs. Lossless

Texture block formats (BC, ASTC, ETC, PVRTC) are **lossy** by design — they reconstruct an approximation of the original signal from a quantised, parameterised block model. Framebuffer compression (AFBC, Intel FBC, AMD DCC in its basic colour-compression mode) is **lossless** — the display pixel sequence must be reproduced exactly. Geometry compression is typically **near-lossless**: quantisation is applied at encoding time, but the decoded positions are reproducible exactly from the compressed buffer.

### 1.4 Algorithm Categories

| Category | Examples | Encode where | Decode where |
|---|---|---|---|
| Texture block | BC1–BC7, ASTC, ETC2, PVRTC | CPU or GPU compute | Fixed-function texture unit |
| Geometry | Draco, meshoptimizer | CPU or GPU compute | CPU or GPU compute shader |
| Lossless buffer | LZ4, GDeflate, nvcomp | GPU compute | GPU compute |
| Entropy coding | Huffman, rANS, CABAC | GPU (partial) | GPU (partial) |
| Framebuffer/display | AFBC, DCC, FBC | Fixed-function GPU render | Fixed-function display engine |

---

## 2. BC/DXT Texture Block Compression

The BC (Block Compression) family, standardised for Direct3D 10+ and exposed in Vulkan via `VK_FORMAT_BC*_BLOCK`, is the dominant texture compression family on desktop Linux with discrete AMD, Intel, and NVIDIA GPUs.

### 2.1 BC1 (DXT1) — RGB and 1-bit Alpha

BC1 encodes a 4×4 block of texels into 64 bits. Two RGB565 endpoints `color0` and `color1` define a line segment in RGB colour space. Each of the 16 texels is assigned a 2-bit index selecting one of four colours: `color0`, `color1`, and two linearly interpolated intermediates. When `color0 > color1` (compared as uint16), the four entries are 0%, 33%, 67%, and 100%; when `color0 <= color1`, three entries plus a special transparent black entry allow a 1-bit alpha punch-through mode.

BC1 encodes to 4 bpp, giving a 6:1 ratio vs uncompressed 24bpp RGB8 input, or 4:1 vs 16bpp RGB565. [Source: Microsoft BC1-5 format reference, https://learn.microsoft.com/en-us/windows/win32/direct3d10/d3d10-graphics-programming-guide-resources-block-compression]

### 2.2 BC2 and BC3 — Alpha Channels

**BC2 (DXT3)** stores a 64-bit BC1 RGB block plus an explicit 4-bit alpha for each texel (64 bits total), for 128 bits per 4×4 block. The explicit alpha is useful for sharp alpha cutouts.

**BC3 (DXT5)** replaces the explicit alpha with a BC4-style interpolated alpha: two 8-bit endpoints and 3-bit indices for each texel selecting from 8 interpolated levels (or 6 levels plus 0 and 255 when the first endpoint is ≤ the second). BC3 achieves much smoother alpha gradients than BC2 at no additional bit cost.

### 2.3 BC4 and BC5 — Single and Dual Channel

**BC4** encodes a single channel using the same 2-endpoint, 3-bit-index scheme as BC3 alpha. 64 bits for a 4×4 block yields 4 bpp. Used for single-channel data such as height maps, roughness, or specular occlusion.

**BC5 (ATI2N)** concatenates two BC4 blocks for red and green channels, totalling 128 bits per 4×4 block at 8 bpp. This is the standard format for tangent-space normal maps, because only X and Y need to be stored; Z is reconstructed as `sqrt(1 - X² - Y²)` in the shader.

### 2.4 BC6H — HDR RGB

BC6H compresses 4×4 blocks of 16-bit half-float (FP16) RGB data to 128 bits — 8 bits per texel. FP16 is 2 bytes per component, so uncompressed RGB16F costs 6 bytes/texel (6:1 ratio) and RGBA16F costs 8 bytes/texel (8:1 ratio). There is no alpha channel.

BC6H defines 14 modes. Some modes describe a single-subset block with one pair of endpoints; others describe a two-subset block using one of 32 partition shapes from a shared table. Within each subset, the 16 texels are indexed by 3-bit indices selecting a linearly interpolated half-float RGB value. Both **unsigned** (`DXGI_FORMAT_BC6H_UF16`, Vulkan `VK_FORMAT_BC6H_UFLOAT_BLOCK`) and **signed** (`DXGI_FORMAT_BC6H_SF16`, Vulkan `VK_FORMAT_BC6H_SFLOAT_BLOCK`) variants exist; the signed form is necessary for data containing negative values (e.g., signed normal maps stored in FP16). [Source: Microsoft BC6H Format Reference, https://learn.microsoft.com/en-us/windows/win32/direct3d11/bc6h-format]

### 2.5 BC7 — High-Quality RGBA

BC7 is a 128-bit, 4×4-block format that encodes RGB or RGBA data with substantially higher quality than BC1/BC3 at the same bit rate. BC7 achieves this via **mode selection**: eight modes (0–7) offer different trade-offs between subset count, endpoint precision, index precision, rotation bits, and P-bits. [Source: Microsoft BC7 Format Reference, https://learn.microsoft.com/en-us/windows/win32/direct3d11/bc7-format]

The mode is identified by the position of the first `1` bit in the 128-bit block:

| Mode | Subsets | Partition bits | Endpoint precision | Index bits | P-bits | Notes |
|------|---------|---------------|-------------------|-----------|--------|-------|
| 0 | 3 | 4 | RGB 4.4.4 | 3 | per-endpoint | High quality RGB |
| 1 | 2 | 6 | RGB 6.6.6 | 3 | shared | Good balance |
| 2 | 3 | 6 | RGB 5.5.5 | 2 | none | |
| 3 | 2 | 6 | RGB 7.7.7 | 2 | per-endpoint | High precision |
| 4 | 1 | none | RGB 5.5.5 + A 6 | 2/3 | none | Separate alpha, rotation |
| 5 | 1 | none | RGB 7.7.7 + A 8 | 2 | none | Separate alpha, rotation |
| 6 | 1 | none | RGBA 7.7.7.7 | 4 | per-endpoint | Combined high quality |
| 7 | 2 | 6 | RGBA 5.5.5.5 | 2 | per-endpoint | Full RGBA, 2 subsets |

**Endpoint optimisation** in the encoder finds RGB565 or RGB888 pairs that minimise mean squared error for the texels assigned to each subset. The RGBP encoding adds a shared P-bit (one bit per subset or per endpoint) that expands each component by one bit, effectively providing 6-bit precision from 5-bit storage.

**Mode selection** in a GPU encoder evaluates candidate modes for each block and picks the one achieving the lowest rate-distortion cost. The BC7enc family of encoders uses a hierarchical approach: reject clearly suboptimal modes early (e.g., skip Mode 0 for blocks without sharp colour transitions), run endpoint optimisation for the surviving candidates, and commit to the best. [Source: bc7enc_rdo repository, https://github.com/richgel999/bc7enc_rdo]

---

## 3. ASTC Texture Compression

ASTC (Adaptive Scalable Texture Compression) is an open Khronos standard supported natively in OpenGL ES 3.2 and Vulkan `VK_FORMAT_ASTC_*_BLOCK`. Its defining property is a **variable block footprint**: the encoder can select any footprint from 4×4 (8 bpp) to 12×12 (≈0.89 bpp) in several steps (4×4, 5×4, 5×5, 6×5, 6×6, 8×5, 8×6, 8×8, 10×5, 10×6, 10×8, 10×10, 12×10, 12×12), all encoding to the same fixed 128-bit block. There are also 3D footprints (3×3×3 through 6×6×6) for volume textures. This flexibility allows one format family to serve everything from HDR skyboxes at high quality (6×6, 3.56 bpp) to low-bandwidth mobile UI (12×12, 0.89 bpp). [Source: ARM ASTC encoder documentation, https://github.com/ARM-software/astc-encoder]

### 3.1 ASTC Block Internals

Every 128-bit ASTC block encodes the following independently configured fields:

- **Weight grid.** A subsampled grid of weights, whose dimensions can be smaller than the block footprint, is bilinearly interpolated to produce per-texel weights. Weight precision ranges from 2 to 6 bits.
- **Partition count.** The block can be divided into 1–4 *partitions*, each with independent endpoint pairs. Partition shapes come from a parameterised hash function that maps (x, y, partition count, partition seed) → partition index.
- **Endpoint encoding.** Within each partition, a pair of endpoints specifies the colour line segment. Endpoints can be stored as LDR (0–255 per channel), HDR half-float, or a mix. Multiple endpoint formats are available (RGB, RGBA, luminance, luminance+alpha).
- **Dual-plane mode.** Two independent sets of indices allow one channel (e.g. alpha) to vary independently from the RGB vector. Not available with more than 2 partitions.

The decoder is deterministic and specified bit-precisely in the Khronos specification. GPU hardware implementations on all modern ARM Mali GPUs, AMD RDNA3+, and Qualcomm Adreno 500+ decode ASTC in fixed-function hardware.

### 3.2 ASTC Encoder: ARM astc-encoder

The reference implementation is `astc-encoder` from ARM, available at `https://github.com/ARM-software/astc-encoder`. The library exposes a C API:

```c
// Initialise codec context for a given block footprint and profile
astcenc_context *ctx;
astcenc_config config;
astcenc_config_init(ASTCENC_PRF_LDR, 6, 6, 1, ASTCENC_PRE_MEDIUM, 0, &config);
astcenc_context_alloc(&config, thread_count, &ctx);

// Compress one image
astcenc_image image = { .dim_x = width, .dim_y = height, .dim_z = 1,
                         .data_type = ASTCENC_TYPE_U8, .data = &row_ptrs };
astcenc_swizzle swizzle = ASTCENC_SWZ_RGBA;
astcenc_compress_image(ctx, &image, &swizzle, compressed_data, compressed_size, thread_index);
```

Quality presets map to `-fastest`, `-fast`, `-medium`, `-thorough`, and `-exhaustive` command-line flags. The encoder performs an iterative multi-candidate search: it evaluates an increasing number of candidate configurations (partition count, partition seed, endpoint mode, weight grid dimensions, weight precision), commits early if a sufficiently good result is found, and expands the search for harder blocks. The `ASTCENC_PRE_EXHAUSTIVE` preset evaluates all legal configurations within the block's search space.

The encoder supports a CPU-only path (NEON on ARM, AVX2 on x86) and an experimental OpenCL GPU backend that offloads the candidate evaluation loop to GPU compute. [Source: astc-encoder GitHub repository, https://github.com/ARM-software/astc-encoder]

---

## 4. ETC and PVRTC Formats

### 4.1 ETC1 and ETC2

**ETC1** (Ericsson Texture Compression) encodes 4×4 RGB blocks to 64 bits (4 bpp). Each block is split vertically or horizontally into two 4×2 or 2×4 sub-blocks. Two modes exist:

- **Individual**: each sub-block has an independent RGB444 base colour.
- **Differential**: the first sub-block has RGB555; the second sub-block stores a signed 3-bit delta per channel, giving a range of ±3 around the first sub-block's colour.

Each pixel receives a 2-bit modifier index selecting from one of eight 4-entry lookup tables (table selectors are per-sub-block), adding or subtracting a luminance offset to the base colour. ETC1 is RGB-only; it has no alpha channel.

**ETC2** extends ETC1 with three additional block modes selected when the differential would overflow:

- **T mode**: two RGB colours with luminance offsets; pixels assigned to either colour or a third tinted variant.
- **H mode**: two colours sorted by their luminance, with four decoded values.
- **Planar mode**: three RGB 676-precision endpoint colours (origin O, horizontal H, and vertical V) define a smooth bilinear plane across the block, encoding slowly varying colour gradients with zero error.

ETC2 also adds alpha support via a separate **EAC** (Ericsson Alpha Compression) block: the same architecture as BC4, with two 8-bit endpoints and 3-bit indices for 8 interpolated values. Combined, **ETC2 RGBA** = ETC2 RGB (64 bits) + EAC alpha (64 bits) = 128 bits, 8 bpp.

**EAC R11 and RG11** encode 1 or 2 channels with 11-bit precision per channel using the same 64-bit block structure. These are the ETC equivalent of BC4 and BC5. [Source: Khronos OpenGL ES 3.0 specification, Appendix C: Compressed Texture Image Formats, https://registry.khronos.org/OpenGL/specs/es/3.0/es_spec_3.0.pdf]

Hardware decode for ETC1/ETC2 is available on ARM Mali, Qualcomm Adreno, and PowerVR Series6 and later.

### 4.2 PVRTC

PVRTC (PowerVR Texture Compression) uses a fundamentally different architecture from block-based formats. Rather than encoding independent 4×4 blocks, PVRTC decomposes the image into two low-resolution **signal images** A and B, each at 1/4 the horizontal and vertical resolution. Each 32-bit signal word stores a modulated RGB colour value. The full-resolution image is reconstructed by bilinear interpolation of A and B across their 2×2-texel resampling grid, then applying a 2-bit **modulation** code per texel that selects between the A and B signals (or blended variants).

PVRTC exists in **4bpp** (4×4 block equivalent, 64-bit words on a 4×4 grid) and **2bpp** (8×4 block equivalent) variants. The overlapping block structure means that encoding one block affects neighbouring blocks, making PVRTC encoding significantly harder than BC or ETC — the encoder must solve a global optimisation problem. The PowerVR SDK provides a CPU-only offline encoder. GPU-side PVRTC decoding is fixed-function on PowerVR hardware (Raspberry Pi VideoCore VI, Pi 5's VideoCore VII).

PVRTC remains important for Raspberry Pi targets and older iOS hardware, but ASTC is preferred for all new PowerVR and ARM Mali deployments. [Source: Imagination Technologies PVRTC specification, https://docs.imgtec.com/reference-manuals/pvr-texture-compression-user-guide/html/topics/pvrtc-pvrtc2-compression-format.html]

---

## 5. GPU Texture Compression Encoders

Texture block compression was originally offline-only: a CPU tool pre-compressed artist assets before shipping. Real-time GPU-side encoding is now viable for several important use cases: compressing render-to-texture output before streaming, screenshot capture with immediate texture feed-back, and procedurally generated content that must be compressed on the fly.

### 5.1 Compressonator (AMD)

AMD's Compressonator SDK (`https://github.com/GPUOpen-Tools/Compressonator`) provides GPU-accelerated BC1–BC7, ETC1/ETC2, and ASTC encoding. The high-level entry point is:

```c
CMP_ERROR CMP_ConvertTexture(
    CMP_Texture *srcTexture,
    CMP_Texture *dstTexture,
    const CMP_CompressOptions *options,
    CMP_Feedback_Proc pFeedbackProc);
```

GPU compute is selected via `options->bDXT1UseAlpha` and the `nEncodeWith` field. The OpenCL backend dispatches kernels such as `BC1_Encode_Kernel.cpp` that process one 4×4 block per workgroup, computing endpoints via iterative PCA (principal component analysis) on the 16 texel RGB vectors and then exhaustive index assignment. The GPU path typically achieves 3–10× the throughput of a scalar CPU encoder for the same quality level. [Source: Compressonator SDK documentation, https://github.com/GPUOpen-Tools/Compressonator]

### 5.2 bc7enc_rdo — Rate-Distortion Optimised BC7

`bc7enc_rdo` (`https://github.com/richgel999/bc7enc_rdo`) provides CPU BC7 encoding with an optional **RDO (rate-distortion optimisation)** post-pass. After standard BC7 encoding, the RDO pass perturbs the compressed block data within a PSNR budget to maximise subsequent LZ compression ratios of the resulting stream. This is useful when BC7 textures will be stored in a compressed archive or transmitted over a network. The library also contains `rgbcx`, a fast BC1/BC3 encoder with parameterised quality levels, and a GPU-ready CPU reference.

### 5.3 ISPCTextureCompressor

Intel's ISPC Texture Compressor (`https://github.com/GameTechDev/ISPCTextureCompressor`) uses Intel ISPC (Implicit SPMD Program Compiler) to generate wide SIMD code for BC1, BC3, BC6H, and BC7. The resulting kernels run on AVX-512 (64-wide on Xeon Phi) but are not GPU compute shaders; they are vectorised CPU kernels. However, the approach has been ported to Vulkan compute by mapping ISPC's `foreach` over workgroup lanes, providing a compute shader encoder that achieves quality comparable to ISPCTextureCompressor's `ultrafast` quality at native GPU throughput.

### 5.4 Real-Time In-Frame BC7 Encoding

A practical real-time encoding pipeline for compressing a rendered VkImage:

```c
// 1. Render into uncompressed VkImage (VK_FORMAT_R8G8B8A8_UNORM)
// 2. Bind as storage image in compute shader
// 3. Dispatch: one workgroup per 4x4 block
vkCmdDispatch(cmd, (width + 3) / 4, (height + 3) / 4, 1);
// 4. Output: VkBuffer of BC7 blocks, 16 bytes per block
// 5. Copy buffer → VkImage with VK_FORMAT_BC7_UNORM_BLOCK
VkBufferImageCopy region = {
    .imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT,
    .imageExtent = { width, height, 1 },
};
vkCmdCopyBufferToImage(cmd, bc7_buffer, compressed_image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &region);
```

Quality is controlled by the shader's mode selection search depth. The fast-mode encoder (Mode 6 only, single subset) runs at ~1000 MP/s on a modern discrete GPU; the high-quality encoder (all modes) runs at ~200 MP/s.

---

## 6. Geometry Compression

### 6.1 Draco (Google)

Draco (`https://github.com/google/draco`) compresses triangle meshes via two independent algorithms: **connectivity compression** and **attribute compression**.

**Connectivity compression** uses the **Edgebreaker** algorithm. The encoder traverses the dual graph of the triangle mesh in a depth-first manner, tracking an active boundary (the "gate"). Each triangle's relationship to its parent gate is encoded as one of five symbols: C (continue — the new gate follows the left side), R (right — branch right), L (left — branch left), S (split — the boundary splits into two independent paths), or E (end — the current path merges with an already-visited vertex). This produces a compact symbol stream, typically achieving close to the theoretical optimum of approximately 1 bit per triangle for manifold meshes.

**Attribute compression** re-orders attribute data (position, normal, texture coordinate, colour) to match the traversal order. Within each connected region, attributes are predicted using the **parallelogram predictor**: given a triangle edge shared with an already-decoded triangle, the predicted attribute value for the opposite vertex is computed by reflecting the opposite vertex of the known triangle across the shared edge. The residual (actual minus predicted) is then entropy-coded.

The default quantisation levels are:

```
Position:   11 bits (default, range -1..1 mapped to ~4096 steps)
Normals:     8 bits
TexCoords:  10 bits
Colour:      8 bits
```

Residuals are entropy-coded using **rANS** (range-variant Asymmetric Numeral Systems). Compression levels 0–10 trade encoding speed for ratio, with the default at 7. [Source: Google Draco README, https://github.com/google/draco]

The C++ decode API:

```cpp
draco::DecoderBuffer buffer;
buffer.Init(compressed_data.data(), compressed_data.size());
draco::Decoder decoder;
auto statusor = decoder.DecodeMeshFromBuffer(&buffer);
std::unique_ptr<draco::Mesh> mesh = std::move(statusor).value();
```

### 6.2 meshoptimizer

`meshoptimizer` (`https://github.com/zeux/meshoptimizer`) provides a complementary set of optimisers and codecs oriented towards GPU cache efficiency and bandwidth.

**Pre-compression optimisers** run before encoding:

```c
// Vertex cache optimiser — reorders triangles for post-transform cache efficiency
meshopt_optimizeVertexCache(out_indices, in_indices, index_count, vertex_count);

// Overdraw optimiser — minimises overdraw for opaque geometry
meshopt_optimizeOverdraw(out_indices, in_indices, index_count,
    positions, vertex_count, vertex_stride, threshold);

// Vertex fetch optimiser — reorders vertices to match index buffer order
meshopt_optimizeVertexFetch(out_vertices, out_indices, index_count,
    in_vertices, vertex_count, vertex_size);
```

**Codec functions** compress the optimised buffers:

```c
// Encode vertex buffer: lossless, ~2-4x compression ratio
size_t bound = meshopt_encodeVertexBufferBound(vertex_count, vertex_size);
meshopt_encodeVertexBuffer(encoded, bound, vertices, vertex_count, vertex_size);

// Decode: ~3-6 GB/s throughput
meshopt_decodeVertexBuffer(vertices, vertex_count, vertex_size, encoded, encoded_size);

// Encode index buffer: ~1 byte per triangle
size_t ibound = meshopt_encodeIndexBufferBound(index_count, vertex_count);
meshopt_encodeIndexBuffer(encoded, ibound, indices, index_count);

// Decode: ~3-6 GB/s throughput
meshopt_decodeIndexBuffer(indices, index_count, sizeof(uint32_t), encoded, encoded_size);
```

[Source: meshoptimizer README, https://github.com/zeux/meshoptimizer]

The library also includes a GPU decode shader (`meshletdec.slang`) that decompresses vertex buffers in a compute shader, outputting 32-bit floats directly into GPU memory without a CPU readback step. This enables a streaming mesh pipeline where compressed meshlets arrive from the network and are decoded in-flight by a compute dispatch before the next draw call.

---

## 7. Lossless GPU Compression

### 7.1 nvcomp (NVIDIA)

nvcomp (`https://docs.nvidia.com/cuda/nvcomp/index.html`) is NVIDIA's GPU-accelerated lossless compression library. It supports multiple algorithm backends:

| Algorithm | Use case | GPU-friendliness |
|-----------|----------|-----------------|
| LZ4 | General-purpose byte-level | High |
| Snappy | Compatible with existing Snappy streams | High |
| GDeflate | DEFLATE-compatible, GPU-optimised | High |
| Zstandard (zstd) | High-ratio general compression | Moderate |
| Bitcomp | Scientific/sparse integer data | Very high |
| ANS | Entropy coding stage | High |
| Cascaded | Structured/columnar data | High |

The batch API processes multiple independent chunks in parallel, matching the independence structure of GPU computation:

```c
// LZ4 batch compress
nvcompBatchedLZ4CompressAsync(
    device_uncompressed_ptrs,   // array of device pointers
    device_uncompressed_bytes,  // array of uncompressed sizes
    max_uncompressed_chunk_bytes,
    batch_size,
    device_temp,                // temporary workspace
    temp_bytes,
    device_compressed_ptrs,     // output: array of device pointers
    device_compressed_bytes,    // output: array of compressed sizes
    format_opts,
    stream);

// LZ4 batch decompress
nvcompBatchedLZ4DecompressAsync(
    device_compressed_ptrs,
    device_compressed_bytes,
    device_uncompressed_bytes,
    device_actual_uncompressed_bytes,
    batch_size,
    device_temp,
    temp_bytes,
    device_uncompressed_ptrs,
    stream);
```

[Source: NVIDIA nvcomp documentation, https://docs.nvidia.com/cuda/nvcomp/index.html]

### 7.2 hipcomp (AMD ROCm)

hipcomp is an AMD ROCm port of the nvcomp API, replacing CUDA primitives with HIP equivalents. The API mirrors nvcomp exactly, with function prefix `hipcomp` substituting `nvcomp`. Supported backends include LZ4 and GDeflate. The HIP portability layer allows the same compression kernels to target both AMD RDNA3+ GPUs and NVIDIA hardware. [Source: hipcomp GitHub, https://github.com/ROCm/hipcomp]

### 7.3 Parallel LZ4 Decompress Architecture

LZ4 decompression on GPU is highly parallel because LZ4 frames can be split into independent **blocks** (default block size 64 KB or 4 MB). Within a block, decompression is sequential due to the back-reference chain; across blocks it is embarrassingly parallel. The GPU kernel assigns one CUDA/HIP thread block per LZ4 block, with threads cooperating to decode the match-copy sequences using shared memory for the sliding window. Throughput scales linearly with GPU SM count until memory bandwidth is saturated.

A primary application on Linux is compressing geometry and image buffers that need to cross the PCIe bus to the CPU or network. Compressing a 1 GB vertex buffer to ~400 MB with LZ4 at ~20 GB/s GPU encode throughput takes approximately 20 ms — often less than the PCIe transfer time for the uncompressed data.

---

## 8. Entropy Coding on GPU

### 8.1 Huffman Coding

GPU Huffman coding requires restructuring the inherently serial bit-packing step. The practical approach:

1. **Parallel histogram** (one thread per symbol, atomic increment into shared-memory frequency table).
2. **Canonical code construction** on the CPU from the histogram (Huffman tree, code lengths).
3. **Broadcast** the code table to GPU device memory.
4. **Parallel length computation** — each thread determines the output bit length of its symbol(s).
5. **Exclusive prefix sum** (`cub::DeviceScan::ExclusiveSum`) over bit lengths to produce bit offsets.
6. **Parallel scatter** — each thread writes its encoded bits to the output buffer at its assigned bit offset using atomic bit operations or byte-aligned output with later pack.

The prefix sum step dominates latency for small inputs. For large inputs (megabyte-scale), the parallel scatter achieves near-linear scaling. A CUDA implementation of GPU Huffman encoding targeting PNG-style byte streams can achieve ~5 GB/s on a modern GPU.

### 8.2 Asymmetric Numeral Systems (ANS) on GPU

ANS replaces arithmetic coding's interval subdivision with integer arithmetic on a single state variable. In **rANS** (range-variant ANS), encoding is:

```
new_state = (state / freq[s]) * M + cumfreq[s] + (state % freq[s])
```

where `M` is the total symbol frequency sum (power of 2), `freq[s]` is the quantised frequency of symbol `s`, and `cumfreq[s]` is its cumulative frequency. Decoding reverses this:

```
s = find_symbol(state % M)          // lookup table
state = freq[s] * (state / M) + (state % M) - cumfreq[s]
```

The encoding step is **inherently serial** because each state depends on the previous. The GPU parallelisation trick, developed for nvcomp's ANS backend and for the Oodle Data compressor, is **multi-stream interleaving**: divide the symbol sequence into *N* independent sub-sequences (one per warp lane, for example), encode each independently with its own rANS state, and interleave the output streams. Decoding reverses the interleave and runs *N* decoders in parallel (one per GPU thread). This achieves near-linear throughput scaling at the cost of a constant per-stream header (approximately 32 bits per stream for the final ANS state value that must be flushed at encode end and read at decode start).

**tANS** (table-based ANS) replaces the integer arithmetic with pre-computed state-transition tables, making it more cache-friendly for small alphabets. tANS is used internally by zstd and LZFSE. GPU implementations of tANS decode achieve ~15–20 GB/s on modern GPUs.

### 8.3 CABAC Acceleration (H.264 / HEVC)

CABAC (Context-Adaptive Binary Arithmetic Coding) used in H.264 and HEVC is the hardest entropy coder to parallelise: each symbol update modifies the probability context that governs the next symbol. GPU contributions are pre-computation rather than full parallel encode/decode:

- **Context initialisation**: at the start of each slice/tile, CABAC context tables are initialised from a small lookup table depending on the slice QP. This initialisation can be parallelised across tiles.
- **Multi-tile decode**: HEVC tiles reset the CABAC context at each tile boundary, allowing one CABAC decoder thread per tile.
- **Batch frame processing**: in a transcoding pipeline, multiple frames' CABAC streams can be decoded on independent GPU SM clusters.

Full CABAC decode on GPU is available in NVIDIA's NVDEC fixed-function hardware and AMD VCN, but is not exposed as a programmable compute shader.

---

## 9. Image and Video Codec Entropy Stages

### 9.1 JPEG GPU Pipeline

JPEG encoding on GPU proceeds in four parallelisable stages:

1. **DCT** — 8×8 blocks processed in parallel (see Ch223 for full treatment).
2. **Quantisation** — parallel element-wise division by the quantisation matrix.
3. **Zig-zag scan** — reorder DCT coefficients; parallel per-block.
4. **Huffman coding** — the GPU prefix-sum approach described in §8.1.

NVIDIA's `nvjpeg` library (`https://docs.nvidia.com/cuda/nvjpeg/index.html`) exposes a full GPU JPEG encode pipeline:

```c
nvjpegHandle_t handle;
nvjpegEncoderState_t state;
nvjpegEncoderParams_t params;

nvjpegCreateSimple(&handle);
nvjpegEncoderStateCreate(handle, &state, stream);
nvjpegEncoderParamsCreate(handle, &params, stream);
nvjpegEncoderParamsSetQuality(params, 90, stream);

nvjpegEncodeImage(handle, state, params,
    &nv_image, NVJPEG_INPUT_BGRI, width, height, stream);
```

AMD's `rocJPEG` library provides an equivalent AMD-GPU path targeting RDNA2 and later. [Source: NVIDIA nvjpeg documentation, https://docs.nvidia.com/cuda/nvjpeg/index.html]

### 9.2 PNG GPU Encoding

PNG encoding consists of filter selection followed by deflate compression:

**Filter selection**: each scanline of the image is transformed by one of five predictors — None, Sub (left pixel delta), Up (above pixel delta), Average (average of left and above), Paeth (Paeth predictor). The best filter for each scanline minimises the sum of absolute values of the filtered bytes. GPU parallelisation applies all five filters to each scanline simultaneously (five compute threads per scanline), then selects the minimum.

**Deflate**: LZ77 sliding-window matching followed by Huffman coding. The LZ77 step is problematic for GPU: finding the longest match in a sliding window of up to 32 KB requires searching backward through previous data, with each thread's search depending on previously processed bytes. Practical GPU deflate implementations (e.g., NVIDIA's GPU-Accelerated PNG encoder used internally in CUDA screenshot capture) use small fixed window sizes or segment the image into independently deflatable chunks.

### 9.3 AVIF/HEIF and WebP

**AVIF** encodes images as AV1 intra-only frames wrapped in ISOBMFF (`.avif` container). The AV1 intra coding stages (DCT/ADST transforms, CDEF filter, Wiener restoration) are partially accelerable on GPU (see Ch223), but the final entropy coding (ANS in AV1) remains a bottleneck for single-frame GPU encode.

**WebP lossless** uses a predictor coding stage followed by Huffman entropy coding on the prediction residuals. The predictor step is parallelisable; Huffman coding follows the prefix-sum pattern in §8.1.

---

## 10. Framebuffer and Display Compression

Framebuffer compression operates transparently between the GPU render engines and the display controller, requiring no application-level changes. It is configured through DRM KMS via **format modifiers** — a 64-bit modifier appended to the DRM pixel format that encodes compression type, tile geometry, and flags. [Source: Linux kernel drm_fourcc.h, https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_fourcc.h]

### 10.1 AFBC — Arm Frame Buffer Compression

AFBC is a proprietary lossless compression protocol used between ARM Mali GPUs (and recent Panfrost-driven hardware) and ARM display processors (including the ones in Rockchip RK3588 and similar SoCs). [Source: Linux kernel AFBC documentation, https://www.kernel.org/doc/html/latest/gpu/afbc.html]

**Block structure**: AFBC superblocks are 16×16 texels (or 32×8 in wide-block mode). Each superblock header is 16 bytes containing a pointer into the body region, a bitmask identifying which tiles within the superblock are empty (all-zero), and the compressed body size. The body contains independently compressed tiles of typically 4×4 texels using a simple run-length–based coding scheme. The header region is contiguous and addressable by the display controller using fixed-stride addressing.

**DRM format modifiers** for AFBC are formed as `DRM_FORMAT_MOD_ARM_AFBC(flags)` where the flags bitmask includes:

```c
AFBC_FORMAT_MOD_BLOCK_SIZE_16x16   // 16×16 superblock (default)
AFBC_FORMAT_MOD_BLOCK_SIZE_32x8    // 32×8 superblock
AFBC_FORMAT_MOD_SPARSE             // sparse allocation (incomplete superblocks)
AFBC_FORMAT_MOD_YTR                // lossless YCbCr-to-RGB transform pre-applied
AFBC_FORMAT_MOD_SPLIT              // body split into two planes for DRAM efficiency
AFBC_FORMAT_MOD_TILED              // wide superblocks for high-bandwidth displays
AFBC_FORMAT_MOD_CBR                // constant bit-rate mode (fixed body size)
```

In the Panfrost driver, AFBC is enabled when a KMS plane property signals that the display controller supports the modifier. The GPU writes AFBC-compressed render targets directly; the display controller reads them without decompression to CPU memory. [Source: Panfrost driver, https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/drivers/panfrost]

### 10.2 AMD DCC — Delta Color Compression

AMD GPUs from GCN3 onwards apply Delta Color Compression (DCC) to render targets. DCC exploits the observation that adjacent pixels in a rendered image often differ by small deltas. Each 256-byte tile of the render target is classified into one of several compression modes (constant colour, delta-1, delta-2, or uncompressed) and the result stored alongside a compact **DCC metadata** key in a separate surface.

The DRM format modifier is `DRM_FORMAT_MOD_AMD_DCC`, with subvariants for pipe-aligned (optimal for GPU access) and non-pipe-aligned (required for display engine scanout on some DCN generations). RDNA3's DCN 3.x display controller can scan out DCC-compressed buffers directly:

```c
// When allocating a scanout surface with DCC support:
// drm_mode_create_dumb() is insufficient — use GEM with explicit modifier:
uint64_t modifier = DRM_FORMAT_MOD_AMD_DCC;
// Pass modifier to drmModeAddFB2WithModifiers()
```

[Source: AMD GPU open documentation, https://gpuopen.com/]

### 10.3 Intel FBC — Frame Buffer Compression

Intel's Frame Buffer Compression (FBC) is implemented in the `i915` kernel driver (`drivers/gpu/drm/i915/display/intel_fbc.c`). FBC compresses the primary plane scanout buffer using a hardware compressor in the display engine. The compressor operates on 64×4 pixel tiles; tiles that compress well (smooth content) are stored in the FBC compression FIFO; tiles that do not compress (high-frequency content) are read uncompressed.

FBC is transparent to the application and Vulkan/Mesa layers — there is no DRM format modifier negotiation required. The i915 driver enables FBC automatically when the framebuffer is idle (no GPU writes in flight). [Source: i915 driver source, https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_fbc.c]

### 10.4 AMD DRAM Compression (RDNA3)

RDNA3 introduces an additional compression stage at the DRAM interface inside the GPU: the memory controller can apply a lossless delta compression scheme to DRAM transactions, transparent to the shader core and display engine. This reduces effective DRAM bandwidth consumption for any memory access pattern (not just render targets). There is no driver-level API to control this; it is microarchitecture-managed. [Source: AMD RDNA3 architecture whitepaper, https://gpuopen.com/rdna3-architecture/]

---

## 11. GPU-Side Delta and Prediction Compression

### 11.1 Animation Vertex Stream Delta Encoding

For animated meshes transmitted over a network or streamed from storage, per-frame vertex position delta encoding can achieve 5–10× compression over raw vertex data. The GPU compute approach:

```glsl
// delta_encode.comp
layout(local_size_x = 64) in;
layout(std430, binding = 0) readonly  buffer CurrFrame { vec3 curr[]; };
layout(std430, binding = 1) readonly  buffer PrevFrame { vec3 prev[]; };
layout(std430, binding = 2) writeonly buffer Deltas    { int  delta[]; }; // quantised

layout(push_constant) uniform PC { float quant_scale; } pc;

void main() {
    uint i = gl_GlobalInvocationID.x;
    vec3 d = (curr[i] - prev[i]) * pc.quant_scale;
    delta[3*i+0] = int(round(d.x));
    delta[3*i+1] = int(round(d.y));
    delta[3*i+2] = int(round(d.z));
}
```

The quantised integer deltas are then entropy-coded on CPU (or GPU using rANS) before transmission. The dominant cost is the GPU dispatch for quantisation, which runs at memory bandwidth speeds.

### 11.2 Sparse Update Compression

Remote desktop GPU compression (used in RDP RemoteFX and VNC with GPU capture) relies on detecting which screen regions changed between frames. The GPU computes a **dirty tile map** by XOR-comparing current and previous framebuffer tiles:

```glsl
// dirty_detect.comp
layout(local_size_x = 8, local_size_y = 8) in;
layout(set=0, binding=0, rgba8ui) readonly  uniform uimage2D curr;
layout(set=0, binding=1, rgba8ui) readonly  uniform uimage2D prev;
layout(std430, binding=2) writeonly buffer DirtyMap { uint dirty[]; };

void main() {
    ivec2 coord = ivec2(gl_GlobalInvocationID.xy);
    uvec4 c = imageLoad(curr, coord);
    uvec4 p = imageLoad(prev, coord);
    if (c != p) {
        uint tile_x = gl_WorkGroupID.x;
        uint tile_y = gl_WorkGroupID.y;
        uint stride  = gl_NumWorkGroups.x;
        atomicOr(dirty[tile_y * stride + tile_x], 1u);
    }
}
```

Dirty tiles are then compressed independently (JPEG or PNG for lossy/lossless screen content) and transmitted, while unchanged tiles are skipped. This approach reduces GPU-encode bandwidth by 80–95% for typical desktop workloads.

### 11.3 Depth Buffer Prediction (DPCM)

Depth buffers exhibit strong left-to-right and top-to-bottom correlation: neighbouring fragments at similar depths produce similar Z values. DPCM (Differential Pulse Code Modulation) prediction coding exploits this:

```
delta[x][y] = depth[x][y] - predict(depth[x-1][y], depth[x][y-1])
```

For GPU-side depth capture (e.g., before CPU readback for shadow map baking), a compute shader can apply DPCM prediction and quantisation, then feed the residuals to a deflate or ANS compressor, achieving 3–6× compression vs. raw 32-bit float depth.

---

## 12. Integration with the GPU Pipeline

### 12.1 VkImage Format Modifiers

The Vulkan extension `VK_EXT_image_drm_format_modifier` (EXT-only; not promoted to KHR or Vulkan core) allows a Vulkan `VkImage` to be created with a specific DRM format modifier, enabling zero-copy interop with the kernel DRM display pipeline.

To query which modifiers a physical device supports for a given format:

```c
VkImageDrmFormatModifierPropertiesEXT mod_props = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_PROPERTIES_EXT,
};
vkGetImageDrmFormatModifierPropertiesEXT(device, image, &mod_props);
// mod_props.drmFormatModifier now contains the modifier in use
```

To create an image with an explicit modifier list (letting the driver pick from the set):

```c
uint64_t modifiers[] = {
    DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_16x16 |
                             AFBC_FORMAT_MOD_YTR),
    DRM_FORMAT_MOD_LINEAR,
};
VkImageDrmFormatModifierListCreateInfoEXT mod_list = {
    .sType = VK_STRUCTURE_TYPE_IMAGE_DRM_FORMAT_MODIFIER_LIST_CREATE_INFO_EXT,
    .drmFormatModifierCount = 2,
    .pDrmFormatModifiers    = modifiers,
};
VkImageCreateInfo ici = {
    .sType  = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
    .pNext  = &mod_list,
    .tiling = VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT,
    /* ... */
};
vkCreateImage(device, &ici, NULL, &image);
```

[Source: Vulkan specification VK_EXT_image_drm_format_modifier, https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html]

### 12.2 DMA-BUF Zero-Copy Pipeline for Compressed Textures

The full zero-copy pipeline for a GPU-encoded compressed texture from render to display:

1. Application renders to a `VkImage` with `DRM_FORMAT_MOD_ARM_AFBC(...)` modifier — the Mali GPU writes AFBC-compressed data directly.
2. `vkGetImageDrmFormatModifierPropertiesEXT` confirms the modifier.
3. Export the image as a DMA-BUF: `vkGetMemoryFdKHR` or via `VK_EXT_external_memory_dma_buf`.
4. Import the DMA-BUF into a KMS plane: `drmModeAddFB2WithModifiers(fd, w, h, format, handles, pitches, offsets, modifiers, ...)`.
5. The KMS atomic commit presents the AFBC-compressed buffer directly to the display controller, which decodes on the fly as pixels are scanned out.

At no point does any compressed data pass through the CPU or require an intermediate linear copy. [Source: Linux kernel DRM AFBC documentation, https://www.kernel.org/doc/html/latest/gpu/afbc.html]

### 12.3 Texture Streaming with On-Demand BC7 Decode

A high-density world streaming system might keep assets in BC7-compressed tiles on storage and decode BC7 tiles on-demand into `VkImage` memory:

1. A background thread reads a 16-byte-per-block compressed tile from disk.
2. The tile is copied into a `VkBuffer` via staging.
3. A `vkCmdCopyBufferToImage` copies the raw BC7 block data into the `VkImage` with `VK_FORMAT_BC7_UNORM_BLOCK` format — no shader decode is needed because the GPU texture unit handles BC7 decode in fixed-function hardware.
4. The image is bound as a combined image sampler for subsequent rendering.

This is the most efficient texture streaming path: the GPU texture sampler performs BC7 decode at full texture bandwidth, and the memory footprint is 8 bpp vs 32 bpp for RGBA8.

---

## Integrations

- **Ch139 — DRM Hardware Plane Format Modifiers**: the DRM modifier system (`DRM_FORMAT_MOD_*`) provides the kernel interface through which AFBC and DCC modifiers are negotiated between the GPU driver and display controller. Format modifier caps and negotiation in the KMS atomic flow are covered there.

- **Ch162 — Framebuffer Compression in the Kernel**: the i915 FBC implementation, the AMD DCC metadata allocation path in amdgpu, and the Panfrost AFBC allocation flow are all covered in Ch162, which focuses on the driver-side kernel implementation rather than the algorithm.

- **Ch208 — Mesh Compression and Streaming**: Ch208 covers the full streaming pipeline for Draco- and meshoptimizer-compressed assets, including asset packaging, streaming protocols, and integration with Vulkan memory management for dynamically loaded meshes.

- **Ch220 — GPU Image Processing Algorithms**: DCT, bilateral filters, and the image ISP pipeline are covered in Ch220. This chapter's JPEG and PNG GPU encode sections build on those algorithms.

- **Ch223 — GPU Video Processing Algorithms**: motion estimation, DCT transform coding, quantisation, and AV1/HEVC in-loop filtering are covered in Ch223. This chapter's JPEG GPU pipeline and AVIF/entropy-stage sections extend those topics to still-image formats.

- **Ch226 — Entropy Model Computation**: GPU-side probability model computation for neural image codecs and learned compression (e.g., the scale-hyperprior model inference on GPU) is covered in Ch226, which this chapter's ANS and arithmetic coding sections complement from the classical-codec perspective.
