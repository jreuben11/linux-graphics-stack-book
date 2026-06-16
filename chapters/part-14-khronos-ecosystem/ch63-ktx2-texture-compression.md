# Chapter 63: KTX2, Basis Universal, and GPU Texture Compression

> **Part**: Part XIV — Khronos Extended Ecosystem
> **Audience**: Graphics application developers building asset pipelines
> **Status**: First draft — 2026-06-15

---

## Table of Contents

1. [Overview](#overview)
2. [Why GPU Texture Compression](#why-gpu-texture-compression)
3. [Block Compression Format Families](#block-compression-format-families)
   - [BC1–BC7: Desktop S3TC/RGTC/BPTC](#bc1bc7-desktop-s3tcrgtcbptc)
   - [ETC1/ETC2: The Mobile Baseline](#etc1etc2-the-mobile-baseline)
   - [ASTC: Adaptive Scalable Texture Compression](#astc-adaptive-scalable-texture-compression)
   - [Hardware Support Matrix on Linux](#hardware-support-matrix-on-linux)
   - [Querying Format Support at Runtime](#querying-format-support-at-runtime)
4. [Basis Universal Supercompression](#basis-universal-supercompression)
   - [Architecture Overview](#architecture-overview)
   - [ETC1S: Global Codebook Encoding Pipeline](#etc1s-global-codebook-encoding-pipeline)
   - [UASTC LDR 4×4 Encoding](#uastc-ldr-44-encoding)
   - [UASTC HDR Variants](#uastc-hdr-variants)
   - [XUASTC: Latent-Space Supercompression](#xuastc-latent-space-supercompression)
   - [The basisu CLI Encoder](#the-basisu-cli-encoder)
   - [Runtime Transcoder C++ API](#runtime-transcoder-c-api)
5. [KTX2 Container Format](#ktx2-container-format)
   - [File Magic and Header Layout](#file-magic-and-header-layout)
   - [Supercompression Scheme Enum](#supercompression-scheme-enum)
   - [Level Index Table](#level-index-table)
   - [Data Format Descriptor](#data-format-descriptor)
6. [libktx C API](#libktx-c-api)
   - [ktxTexture2 Struct](#ktxtexture2-struct)
   - [Loading and Reading](#loading-and-reading)
   - [Transcoding Supercompressed Data](#transcoding-supercompressed-data)
   - [Creating and Writing KTX2 Files](#creating-and-writing-ktx2-files)
   - [ktxBasisParams: Full Encoder Control](#ktxbasisparams-full-encoder-control)
   - [The ktx Command-Line Tool](#the-ktx-command-line-tool)
7. [Vulkan Upload Path](#vulkan-upload-path)
   - [ktxVulkanDeviceInfo and ktxVulkanTexture Structs](#ktxvulkandeviceinfo-and-ktxvulkantexture-structs)
   - [Full Upload Workflow](#full-upload-workflow)
8. [glTF 2.0 and KHR_texture_basisu](#gltf-20-and-khr_texture_basisu)
   - [Extension Specification](#extension-specification)
   - [tinygltf Image Hook for KTX2](#tinygltf-image-hook-for-ktx2)
   - [cgltf Integration for KTX2](#cgltf-integration-for-ktx2)
   - [Full Pipeline: .glb to VkImage](#full-pipeline-glb-to-vkimage)
9. [Compression Quality Trade-offs](#compression-quality-trade-offs)
   - [Format Selection Matrix](#format-selection-matrix)
   - [PSNR Tiers](#psnr-tiers)
   - [ETC1S Global Codebook Size Effect](#etc1s-global-codebook-size-effect)
   - [CPU vs GPU Transcoder Throughput](#cpu-vs-gpu-transcoder-throughput)
   - [Real-World Compression Numbers](#real-world-compression-numbers)
10. [Integrations](#integrations)
11. [References](#references)

---

## Overview

This chapter is targeted at **graphics application developers** building production asset pipelines on Linux. It covers the full texture-compression stack: the hardware block-compression format families exposed via `VkFormat`, the Basis Universal supercompression system that provides a single intermediate codec transcoding to all of them, the KTX2 container format that wraps everything in a portable file format with embedded metadata, and the libktx C API that drives the entire load-transcode-upload sequence from a handful of function calls.

By the end of this chapter you will understand:

- Why BC, ETC2, and ASTC formats exist as separate hardware families and which Mesa driver exposes which on a Linux system
- The ETC1S global-codebook model and the UASTC per-block model, their quality/size trade-offs, and which to choose for a given texture type
- The KTX2 on-disk format down to the byte-level header, the Data Format Descriptor, and the supercompression global data block
- How to use the `ktx` CLI tool to author production-ready `.ktx2` files and how to use libktx (`KTX_TTF_*` / `ktxTexture2_TranscodeBasis` / `ktxTexture2_VkUploadEx`) to load them at runtime
- The `KHR_texture_basisu` glTF extension and how to wire a custom image loader in tinygltf for KTX2-backed textures

This chapter assumes you are already familiar with Vulkan memory management (Chapter 24), Mesa driver architecture (Chapters 18–19), and SPIR-V shader pipelines (Chapter 61). Chapter 64 extends the asset-pipeline story to glTF 2.0 as a whole.

---

## Why GPU Texture Compression

Texture bandwidth is one of the primary bottlenecks in real-time rendering. A 4K albedo texture uncompressed in RGBA8 occupies 33.5 MiB. Fetching 50 such textures per frame at 60 FPS demands 100 GB/s of memory bandwidth — a budget that dominates GDDR6X and LPDDR5X alike. Block-compressed textures reduce that footprint by a fixed ratio determined by the format:

| Format | Ratio vs. RGBA8 | On-chip decode cost |
|--------|----------------|---------------------|
| BC1 (4 bpp) | 8:1 | 1–2 ALU ops/texel |
| BC3, BC7, ASTC 4×4 (8 bpp) | 4:1 | 2–4 ALU ops/texel |
| ASTC 6×6 (3.56 bpp) | ~9:1 | 2–4 ALU ops/texel |
| ASTC 8×8 (2 bpp) | 16:1 | 2–4 ALU ops/texel |

Three properties make GPU block compression fundamentally different from CPU codecs like JPEG or PNG:

**Fixed decode throughput.** The texture sampling unit (TMU) decodes one or more blocks per clock in hardware. There is no variable-length Huffman parsing, no heap allocation, and no cache-coherency penalty from shared decompressor state. The decode cost is fully amortised across the rendering pipeline and is effectively free compared to fragment shader ALU.

**Random access.** Each 4×4 block is independently decodable. The GPU can sample from any mip level, any texel coordinate in a compressed image without decompressing neighbouring regions — enabling sparse residency (`VK_IMAGE_USAGE_SPARSE_BINDING_BIT`) and virtual texturing.

**Storage footprint.** A BC7-compressed 4K texture occupies 8 MiB instead of 33.5 MiB on disk and in VRAM, directly improving load times, streaming budgets, and GPU memory pressure.

The energy cost of texture fetch vs. TMU decode has been measured at roughly 2–4× in favour of compressed textures at the same effective resolution on ARM Cortex-A/Mali pairs — an important consideration on the mobile and embedded Linux SoCs covered in Chapters 6 and 19. [Source](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/arm-guide-for-unreal-engine-4-best-practices-for-mobile-developers)

---

## Block Compression Format Families

### BC1–BC7: Desktop S3TC/RGTC/BPTC

The BC family originates in S3 Texture Compression (S3TC, 1999), extended through RGTC (OpenGL 3.0) and BPTC (OpenGL 4.2/D3D11). All BC formats use 4×4 texel blocks. [Source](https://docs.vulkan.org/spec/latest/appendices/compressedtex.html)

**BC1** (`VK_FORMAT_BC1_RGB_UNORM_BLOCK`, `VK_FORMAT_BC1_RGBA_UNORM_BLOCK`, sRGB variants): 64 bits per block (4 bpp). Encodes two RGB565 endpoint colours and 16×2-bit selector indices. Interpolates linearly between two endpoints; the RGBA variant reserves one code point for a 1-bit punch-through alpha. Transcoding target for ETC1S→BC1 is accomplished by a simple 1D lookup table requiring no pixel recomputation — the key property enabling Basis Universal's fast transcoding. [Source](https://themaister.net/blog/2020/08/12/compressed-gpu-texture-formats-a-review-and-compute-shader-decoders-part-1/)

**BC2 / BC3** (`VK_FORMAT_BC2_UNORM_BLOCK`, `VK_FORMAT_BC3_UNORM_BLOCK`): 128 bits per block (8 bpp). BC2 prepends explicit 4-bit alpha per texel to BC1 RGB. BC3 replaces the alpha channel with BC4-style interpolated 8-bit alpha — the preferred choice for RGBA textures with smooth alpha gradients.

**BC4 / BC5 — RGTC** (`VK_FORMAT_BC4_UNORM_BLOCK`, `VK_FORMAT_BC5_UNORM_BLOCK`, signed variants): Single-channel (BC4, 64 bits per 4×4 block, 4 bpp) and dual-channel (BC5, 128 bits per 4×4 block, 8 bpp) formats. Each channel uses two 8-bit endpoints plus 16×3-bit selector indices. The decode is bit-exact across all GPU vendors — making BC5 the standard format for tangent-space normal maps (XY stored in RG; Z reconstructed as `sqrt(1 - x² - y²)` in the shader), and BC4 for single-channel AO or roughness textures. [Source](https://themaister.net/blog/2020/08/30/compressed-gpu-texture-formats-a-review-and-compute-shader-decoders-part-2/)

**BC6H** (`VK_FORMAT_BC6H_UFLOAT_BLOCK`, `VK_FORMAT_BC6H_SFLOAT_BLOCK`): BPTC floating-point, 128 bits per block (8 bpp). Encodes HDR RGB (no alpha) using 14 modes alternating between one-partition and two-partition configurations. Endpoints are stored as 16-bit integers bitcast to FP16; weight precision is 2 or 3 bits per texel. Perceptually near-lossless on HDR environment maps and light probe data. [Source](https://themaister.net/blog/2020/08/30/compressed-gpu-texture-formats-a-review-and-compute-shader-decoders-part-2/)

**BC7** (`VK_FORMAT_BC7_UNORM_BLOCK`, `VK_FORMAT_BC7_SRGB_BLOCK`): BPTC unorm, 128 bits per block (8 bpp). Eight modes with one to three colour subsets per block, variable endpoint precision, correlated and uncorrelated alpha encoding, and component rotation bits. Mode 6 (single subset, no alpha) is the primary target for ETC1S→BC7 and UASTC→BC7 transcoding. BC7 is mandatory on all D3D11-class and D3D12-class hardware (NVIDIA Fermi+, AMD GCN1+, Intel HD 4000+) and is the highest-quality LDR format available on desktop. [Source](https://www.khronos.org/opengl/wiki/BPTC_Texture_Compression)

### ETC1/ETC2: The Mobile Baseline

**ETC2** (`VK_FORMAT_ETC2_R8G8B8_UNORM_BLOCK`, `VK_FORMAT_ETC2_R8G8B8A8_UNORM_BLOCK`, and sRGB variants): 64 bits per 4×4 block (4 bpp for RGB; 128 bits for RGBA8). ETC2 extends ETC1 with T-mode, H-mode, and Planar mode, detected by overflow in the differential encoding path (maintaining full backwards compatibility). The EAC (Ericsson Alpha Compression) extension provides high-precision R11 and RG11 channels (`VK_FORMAT_EAC_R11_UNORM_BLOCK`, `VK_FORMAT_EAC_RG11_UNORM_BLOCK`) using an 8-bit base plus a 3-bit codebook selector.

ETC2/EAC is **mandated** on all Vulkan implementations targeting OpenGL ES 3.0 compatible hardware (`VkPhysicalDeviceFeatures::textureCompressionETC2 == VK_TRUE` on all Android and embedded Linux Vulkan devices). [Source](https://nicjohnson6790.github.io/etc2-primer/)

**Offline encoder: etc2comp.** Google's [`etc2comp`](https://github.com/google/etc2comp) (C++, multi-threaded) is the reference standalone encoder for ETC2/EAC. The command-line tool `EtcTool` converts PNG or other source images to all ETC2 format variants:

```bash
# Build: cmake . && cmake --build . -- -j$(nproc)
# Encode RGB8 (ETC2 opaque) at 75% effort, 8 threads
EtcTool source.png -format RGB8 -effort 75 -jobs 8 -output out.pkm

# Encode RGBA8 (ETC2+EAC full alpha) at maximum quality
EtcTool source.png -format RGBA8 -effort 100 -jobs 8 -output out.pkm

# Encode R11 (EAC single-channel, e.g. height maps)
EtcTool source.png -format R11 -effort 100 -output out.pkm

# With quality analysis: print PSNR after encoding
EtcTool source.png -format RGB8 -effort 75 -analyze -output out.pkm
```

The `-effort` value [1–100] controls what percentage of blocks are re-evaluated per compression pass — 100 gives the highest quality at the cost of encoding time. The `-jobs` flag enables multi-threaded block encoding, scaling near-linearly with core count. Note that `etc2comp` produces raw `.pkm` output (plain ETC2 header); for KTX2-wrapped output suitable for libktx pipelines, use `ktx create --format ETC2_R8G8B8_SRGB_BLOCK` after verifying format support. [Source: google/etc2comp README.md](https://github.com/google/etc2comp/blob/master/README.md)

### ASTC: Adaptive Scalable Texture Compression

ASTC (ARM/Khronos, 2012) is the most flexible block compression family. All ASTC blocks are exactly 128 bits regardless of block footprint, yielding variable bit rates:

| Block size | Bit rate | Typical use |
|-----------|----------|-------------|
| 4×4 | 8.00 bpp | High-quality albedo, matches BC7 |
| 5×5 | 5.12 bpp | Balanced quality/size |
| 6×6 | 3.56 bpp | Normal maps on mobile |
| 8×8 | 2.00 bpp | Distant environment maps |
| 10×10 | 1.28 bpp | Terrain macro-detail |
| 12×12 | 0.89 bpp | Aggressively compressed atlases |

ASTC supports 1–4 channels, LDR unorm, sRGB, and HDR (FP16) decoding selected per-block via the block header — all within the same `VkFormat`. 3D block footprints (3×3×3 to 6×6×6) are also defined for volumetric data. [Source](https://chromium.googlesource.com/external/github.com/ARM-software/astc-encoder/+/HEAD/Docs/FormatOverview.md)

The reference encoder `astcenc` (`ARM-software/astc-encoder`) exposes quality presets: `-fastest` (encoder time: 0.1×), `-fast` (0.5×), `-medium` (2×), `-thorough` (10×), `-exhaustive` (30×). The `thorough` preset is the recommended balance for offline asset pipelines. [Source](https://arm-software.github.io/vulkan-sdk/_a_s_t_c.html)

### Hardware Support Matrix on Linux

On desktop Linux via Mesa:

| Mesa Driver | GPU Family | BC | ETC2 | ASTC LDR | ASTC HDR |
|------------|-----------|-----|------|----------|----------|
| `radv` (RADV) | AMD RDNA/GCN (Ch18/19) | Yes | No | SW emulation | SW emulation |
| `anv` (Intel ANV) | Intel Xe (Ch19) | Yes | No | Yes (Xe2+) | No |
| `nvk` (NVK) | NVIDIA open (Ch10) | Yes | No | SW emulation | No |
| `panfrost` | ARM Mali (Ch6) | No | Yes | Yes (Mali-T624+) | Partial |
| `freedreno` | Qualcomm Adreno (Ch6) | No | Yes | Yes (Adreno 4xx+) | No |
| `etnaviv` | Vivante | No | Yes | No | No |

"SW emulation" means the driver exposes the `VkPhysicalDeviceFeatures` bit as `VK_TRUE` but decodes in shader — sampling cost is dramatically higher. Always query `vkGetPhysicalDeviceFormatProperties` and check `VK_FORMAT_FEATURE_SAMPLED_IMAGE_BIT` in `optimalTilingFeatures` to verify native hardware decode rather than relying solely on the feature bit. [Source](https://docs.vulkan.org/spec/latest/appendices/compressedtex.html)

### Querying Format Support at Runtime

```c
// Query hardware native decode support (not just feature bits)
// lib/include/ktxvulkan.h pattern, KhronosGroup/KTX-Software main branch

VkFormatProperties props;
vkGetPhysicalDeviceFormatProperties(physicalDevice,
    VK_FORMAT_BC7_SRGB_BLOCK, &props);

bool bc7_supported =
    (props.optimalTilingFeatures & VK_FORMAT_FEATURE_SAMPLED_IMAGE_BIT) != 0;

// Feature struct for three families at once
VkPhysicalDeviceFeatures features;
vkGetPhysicalDeviceFeatures(physicalDevice, &features);
// features.textureCompressionBC    — desktop: all NVIDIA/AMD/Intel
// features.textureCompressionETC2  — mobile: all Vulkan ES 3.0+ devices
// features.textureCompressionASTC_LDR — ARM Mali, Adreno 4xx+; SW on some desktop
```

---

## Basis Universal Supercompression

### Architecture Overview

Basis Universal ([BinomialLLC/basis_universal](https://github.com/BinomialLLC/basis_universal), v2.1.0r, June 2026) provides a **format-agnostic supercompression codec** — a single intermediate representation that transcodes at runtime to any native GPU block format. The core design principle: encode once offline, pay negligible CPU cost at runtime (≈1 ms per 4K texture on a modern core), upload a hardware-native block-compressed image to the GPU.

The architecture separates concerns cleanly:

- **Offline encoder** (`basisu`, `libbasisu.a`): applies the full VQ/RDO/Huffman compression pipeline; output is a `.basis` file or a KTX2 supercompressed with BasisLZ or UASTC.
- **Runtime transcoder** (`transcoder/basisu_transcoder.cpp`): a single C++ translation unit with no third-party dependencies (beyond optionally `zstddeclib.c` for KTX2+Zstandard). Initialised once via `basisu_transcoder_init()`, then invoked per-texture per-frame-load. [Source](https://github.com/BinomialLLC/basis_universal)

### ETC1S: Global Codebook Encoding Pipeline

ETC1S is a strict subset of ETC1. The constraint is precise: differential mode is always set, the differential colour delta is always (0,0,0), and the flip bit is never set. This reduces the meaningful per-block state from 64 bits to 50 bits and — critically — means ETC1S blocks can be transcoded to BC1 using only a 1D lookup table with no pixel recomputation. [Source](http://richg42.blogspot.com/2018/06/etc1s-texture-format-encoding.html)

The **global codebook** is the key innovation distinguishing ETC1S from plain ETC1/ETC2 compression. The encoder applies Vector Quantization (VQ) **independently** to two entities across the entire file — all mip levels, all array layers, all cubemap faces:

1. **Endpoint codebook**: clusters of (RGB555 + 3-bit intensity table) pairs, up to 16,128 entries
2. **Selector codebook**: clusters of 16×2-bit texel selection patterns, up to 16,128 entries

Both codebooks are then DPCM-encoded and Huffman-compressed. The result is that a texture atlas containing many sub-textures with shared colour palettes achieves dramatically better compression than independent per-image BC1 compression — the codebook amortises across the entire asset.

The `.basis` file header captures the complete layout:

```cpp
// transcoder/basisu_file_headers.h, BinomialLLC/basis_universal master branch
// Abridged to show primary fields; packed_uint<N> is an N-byte little-endian integer.
struct basis_file_header {
    enum { cBASISSigValue = ('B' << 8) | 's',  // 0x4273
           cBASISFirstVersion = 0x10 };
    packed_uint<2> m_sig;              // "Bs" = 0x4273
    packed_uint<2> m_ver;              // 0x10
    packed_uint<2> m_header_size;      // sizeof(basis_file_header)
    packed_uint<2> m_header_crc16;
    packed_uint<4> m_data_size;
    packed_uint<2> m_data_crc16;
    packed_uint<3> m_total_slices;     // per-image slice count
    packed_uint<3> m_total_images;
    packed_uint<1> m_tex_format;       // enum basis_tex_format (ETC1S/UASTC/etc.)
    packed_uint<2> m_flags;
    packed_uint<1> m_tex_type;         // 2D/array/cubemap/video
    packed_uint<3> m_us_per_frame;     // video framerate (microseconds)
    // ... reserved/userdata fields ...
    packed_uint<2> m_total_endpoints;
    packed_uint<4> m_endpoint_cb_file_ofs;
    packed_uint<3> m_endpoint_cb_file_size;
    packed_uint<2> m_total_selectors;
    packed_uint<4> m_selector_cb_file_ofs;
    packed_uint<3> m_selector_cb_file_size;
    packed_uint<4> m_tables_file_ofs;  // Huffman table offset
    packed_uint<4> m_tables_file_size;
    packed_uint<4> m_slice_desc_file_ofs;
};
```

Effective bit rates after VQ compression range from **0.3 to 3 bpp** for ETC1S — well below the 4 bpp floor of any native block format. The penalty at transcode time is minimal: ETC1S→BC1 loses approximately 0.3 dB Y PSNR via the lookup-table reconstruction. ETC1S→BC7 is nearly lossless (BC7 mode 6 used). [Source](https://github.com/BinomialLLC/basis_universal)

### UASTC LDR 4×4 Encoding

UASTC LDR 4×4 is a **19-mode subset of ASTC LDR 4×4** operating at a fixed 128 bits per 4×4 block (8 bpp). Unlike ETC1S, UASTC abandons global codebooks entirely. Each block is self-contained and carries **transcoding hint bits** — per-block metadata identifying which BC7 modes and ASTC modes are optimal, eliminating all search during decode. This allows runtime transcoding to BC7 or ASTC 4×4 in a handful of ALU operations per block on any CPU.

Quality is near-BC7 grade; UASTC→BC7 transcoding typically loses 0.75–1.5 dB RGB PSNR compared to direct ASTC encoding. UASTC handles normal maps, metallic-roughness channels, and HDR-style data textures far better than ETC1S, which degrades significantly on high-frequency data.

With Zstandard post-compression applied to the UASTC stream (KTX2 `supercompressionScheme = KTX2_SS_ZSTANDARD`), file sizes shrink by 10–40% while retaining fully lossless transcoding to native GPU formats. This is the recommended configuration for production deliverables targeting both desktop (BC7) and mobile (ASTC 4×4). [Source](https://github.com/BinomialLLC/basis_universal)

### UASTC HDR Variants

Basis Universal v2.0+ introduced HDR supercompression targets:

**UASTC HDR 4×4**: A 24-mode subset of ASTC HDR 4×4 at 8 bpp. Transcodes to `BC6H_UFLOAT` or `VK_FORMAT_ASTC_4x4_HDR` on supporting hardware. Recommended for HDR environment maps and emissive textures where FP16 precision is required.

**ASTC HDR 6×6**: Standard ASTC texture data at 3.56 bpp. Shares 27 partition patterns with BC6H for efficient transcoding. Lower quality than UASTC HDR 4×4 at a substantial storage advantage.

**UASTC HDR 6×6 Intermediate**: Supercompressed ASTC HDR 6×6 designed for rapid transcoding from a latent representation to ASTC HDR 6×6 or BC6H. Adds a small decode penalty but enables much smaller intermediate files.

### XUASTC: Latent-Space Supercompression

XUASTC LDR (introduced in basis_universal v2.0, not yet in KTX-Software as of 2026-06-15) applies a **Weight Grid DCT** — a JPEG-style 2D Discrete Cosine Transform over ASTC weight grids with adaptive quantization — to achieve effective bit rates from **0.3 to 5.7 bpp** while maintaining transcoding to ASTC LDR at arbitrary block sizes (4×4 through 12×12).

Three entropy profiles exist: full Zstandard (18 compressed sections), full arithmetic coding, and a hybrid mode. The arithmetic profile achieves 5–18% better compression than Zstandard on typical content. The proposed KTX2 supercompression scheme value is `KTX2_SS_XUASTC_LDR = 5`, currently being standardised with Khronos. [Source](https://github.com/BinomialLLC/basis_universal/wiki/XUASTC-LDR-Specification-v1.0)

**Note**: Do not use XUASTC in production KTX2 pipelines yet — the scheme ID is not ratified and `KTX-Software` does not support it as of this writing. Use UASTC LDR 4×4 + Zstandard instead.

### The basisu CLI Encoder

After building basis_universal (`cmake -B build && cmake --build build`), the `basisu` binary provides the full encoding pipeline. All commands below are verified against v2.1.0r: [Source: BinomialLLC/basis_universal README.md, v2.1.0]

```bash
# ETC1S (BasisLZ) — smallest files, best for diffuse/albedo atlases
basisu -quality 100 albedo.png

# UASTC LDR 4x4 — quality level 2 (range 0–4) with RDO post-compression
basisu -uastc -uastc_rdo_l 1.0 -mipmap albedo.png

# XUASTC LDR 6x6 with Zstd profile (experimental — not for production KTX2)
basisu -xuastc_ldr_6x6 -quality 75 -effort 4 -xuastc_zstd albedo.png

# XUASTC with arithmetic entropy for best compression
basisu -xuastc_ldr_6x6 -quality 75 -effort 4 -xuastc_arith albedo.png

# HDR workflow — format detected from .exr extension
basisu albedo.exr                # UASTC HDR 4x4 (default for HDR)
basisu -hdr_6x6 albedo.exr      # ASTC HDR 6x6 (3.56 bpp)
basisu -hdr_6x6i albedo.exr     # UASTC HDR 6x6 intermediate

# Normal maps: disable sRGB, use linear metrics
basisu -uastc -linear normalmap.png

# ETC1S with OpenCL acceleration (requires -DCMAKE_BUILD_TYPE=Release -DBASISD_SUPPORT_KTX2=1)
basisu -opencl -quality 128 large_atlas.png

# Inspection and validation
basisu -info texture.ktx2
basisu -validate texture.ktx2
basisu texture.ktx2              # decode to PNG/EXR/KTX/DDS for inspection
```

Key quality controls: `-quality X` [1–100] governs effective bit rate (100 = codebook maximised); `-effort X` [0–10] controls encoder search thoroughness; the legacy `-q X` [1–255] (`qualityLevel`) sets ETC1S codebook granularity directly.

### Runtime Transcoder C++ API

The transcoder API is defined in `transcoder/basisu_transcoder.h` ([BinomialLLC/basis_universal master](https://github.com/BinomialLLC/basis_universal)):

```cpp
// transcoder/basisu_transcoder.h — abridged key declarations
// Call once at startup; no argument in v2.x (removed selector_codebook parameter)
void basisu_transcoder_init();

// Primary transcoder for .basis files
class basisu_transcoder {
public:
    basisu_transcoder();
    bool start_transcoding(const void* pData, uint32_t data_size);
    bool get_image_info(const void* pData, uint32_t data_size,
                        basisu_image_info& image_info,
                        uint32_t image_index) const;
    bool get_image_level_info(const void* pData, uint32_t data_size,
                               basisu_image_level_info& level_info,
                               uint32_t image_index,
                               uint32_t level_index) const;
    bool transcode_image_level(const void* pData, uint32_t data_size,
                               uint32_t image_index, uint32_t level_index,
                               void* pOutput_blocks,
                               uint32_t output_blocks_buf_size_in_blocks_or_pixels,
                               transcoder_texture_format fmt,
                               uint32_t decode_flags = 0,
                               uint32_t output_row_pitch_in_blocks_or_pixels = 0,
                               basisu_transcoder_state* pState = nullptr,
                               uint32_t output_rows_in_pixels = 0);
};

// KTX2 container handler (preferred for .ktx2 workflows)
class ktx2_transcoder {
public:
    ktx2_transcoder();
    bool init(const void* pData, uint32_t data_size);
    bool start_transcoding();
    bool transcode_image_level(uint32_t level_index, uint32_t layer_index,
                               uint32_t face_index,
                               void* pOutput_blocks,
                               uint32_t output_blocks_buf_size_in_blocks_or_pixels,
                               transcoder_texture_format fmt,
                               uint32_t decode_flags = 0,
                               uint32_t output_row_pitch_in_blocks_or_pixels = 0,
                               uint32_t output_rows_in_pixels = 0);
    basis_tex_format get_basis_tex_format() const; // cETC1S, cUASTC_LDR_4x4, etc.
    bool is_uastc() const;
    bool is_hdr() const;
};
```

The `transcoder_texture_format` enum lists all supported transcode targets (verified from `transcoder/basisu_transcoder.h`, basis_universal master branch):

| Enum value | Value | Description |
|-----------|-------|-------------|
| `cTFETC1_RGB` | 0 | ETC1 opaque RGB |
| `cTFETC2_RGBA` | 1 | ETC2+EAC RGBA |
| `cTFBC1_RGB` | 2 | BC1 opaque (DXT1) |
| `cTFBC3_RGBA` | 3 | BC3 opaque+alpha (DXT5) |
| `cTFBC4_R` | 4 | BC4 single channel |
| `cTFBC5_RG` | 5 | BC5 dual channel |
| `cTFBC7_RGBA` | 6 | BC7 RGB/RGBA |
| `cTFASTC_LDR_4x4_RGBA` | 10 | ASTC LDR 4×4 |
| `cTFRGBA32` | 13 | Uncompressed 32 bpp (fallback) |
| `cTFETC2_EAC_R11` | 20 | EAC R11 unsigned |
| `cTFETC2_EAC_RG11` | 21 | EAC RG11 unsigned |
| `cTFBC6H` | 22 | BC6H unsigned HDR RGB |
| `cTFASTC_HDR_4x4_RGBA` | 23 | ASTC HDR 4×4 |
| `cTFRGB_HALF` | 24 | 48 bpp FP16 RGB |
| `cTFRGBA_HALF` | 25 | 64 bpp FP16 RGBA |
| `cTFASTC_HDR_6x6_RGBA` | 27 | ASTC HDR 6×6 |
| `cTFASTC_LDR_5x4_RGBA`–`cTFASTC_LDR_12x12_RGBA` | 28–40 | All other ASTC LDR block sizes |
| `cTFTotalTextureFormats` | 41 | Sentinel |

The `libktx` `KTX_TTF_*` enum in `lib/include/ktx.h` mirrors these with a `KTX_TTF_` prefix; numeric values are identical for the common subset. Low-level transcoders for direct block-level access without file-header parsing are also available: `basisu_lowlevel_etc1s_transcoder`, `basisu_lowlevel_uastc_ldr_4x4_transcoder`, `basisu_lowlevel_uastc_hdr_4x4_transcoder`, and the XUASTC and HDR 6×6 variants.

---

## KTX2 Container Format

### File Magic and Header Layout

KTX2 files begin with a 12-byte magic identifier: `{ 0xAB, 0x4B, 0x54, 0x58, 0x20, 0x32, 0x30, 0xBB, 0x0D, 0x0A, 0x1A, 0x0A }`, decoding to the printable prefix `«KTX 20»` followed by `\r\n`, the DOS end-of-file sentinel `\x1A`, and `\n`. The `\r\n\n` triplet detects line-ending corruption from text-mode transfers; `\x1A` terminates text-mode reads on legacy systems. IANA MIME type: `image/ktx2`. [Source](https://www.iana.org/assignments/media-types/image/ktx2)

The header occupies bytes 0–79, all little-endian: [Source](https://github.khronos.org/KTX-Specification/ktxspec.v2.html)

| Bytes | Field | Type | Notes |
|-------|-------|------|-------|
| 0–11 | `identifier` | `uint8[12]` | Magic (above) |
| 12–15 | `vkFormat` | `uint32` | `VkFormat`; 0 (`VK_FORMAT_UNDEFINED`) for BasisLZ/UASTC supercompressed |
| 16–19 | `typeSize` | `uint32` | Element size for endian conversion; 1 for block/byte formats |
| 20–23 | `pixelWidth` | `uint32` | Base level width; must be >0 |
| 24–27 | `pixelHeight` | `uint32` | 0 for 1D textures |
| 28–31 | `pixelDepth` | `uint32` | 0 for 2D textures/cubemaps |
| 32–35 | `layerCount` | `uint32` | 0 for non-array textures |
| 36–39 | `faceCount` | `uint32` | 6 for cubemaps, 1 otherwise |
| 40–43 | `levelCount` | `uint32` | Number of mip levels; 0 = generate at load |
| 44–47 | `supercompressionScheme` | `uint32` | See enum below |
| 48–55 | `dfdByteOffset` / `dfdByteLength` | `uint32×2` | Data Format Descriptor location |
| 56–63 | `kvdByteOffset` / `kvdByteLength` | `uint32×2` | Key-value metadata location |
| 64–71 | `sgdByteOffset` | `uint64` | Supercompression global data (codebooks) offset |
| 72–79 | `sgdByteLength` | `uint64` | Supercompression global data length |

### Supercompression Scheme Enum

```cpp
// transcoder/basisu_transcoder.h, BinomialLLC/basis_universal master branch
// Lines 942–965 (abridged)
enum ktx2_supercompression {
    KTX2_SS_NONE      = 0,  // uncompressed mip data; vkFormat is the native format
    KTX2_SS_BASISLZ   = 1,  // BasisLZ/ETC1S; vkFormat = VK_FORMAT_UNDEFINED
    KTX2_SS_ZSTANDARD = 2,  // Zstandard RFC 8478 frame per level
    KTX2_SS_DEFLATE   = 3,  // zlib RFC 1950 (not supported by basis_universal)
    // HDR 6x6 intermediate: KTX-Software as of 2/19/2026
    KTX2_SS_UASTC_HDR_6x6I = 4,
    // XUASTC LDR: coordinating with Khronos; NOT in KTX-Software yet (2/19/2026)
    KTX2_SS_XUASTC_LDR = 5
};
```

When `supercompressionScheme == KTX2_SS_BASISLZ` (ETC1S), `vkFormat` must be `VK_FORMAT_UNDEFINED`; the DFD `colorModel` field carries `KTX2_KDF_DF_MODEL_ETC1S = 163 (0xA3)`. The supercompression global data section (at `sgdByteOffset`) contains the compressed endpoint and selector codebooks.

For UASTC LDR 4×4, `supercompressionScheme` is 0 (none) or 2 (Zstandard), `vkFormat = VK_FORMAT_UNDEFINED`, and DFD `colorModel = KTX2_KDF_DF_MODEL_UASTC_LDR_4X4 = 166 (0xA6)`.

For natively block-compressed textures (BC7, ASTC, ETC2) stored without Basis supercompression, `vkFormat` holds the actual `VkFormat` value and `supercompressionScheme` is 0 or 2 (Zstd).

### Level Index Table

Immediately following the index section, an array of `levelCount` entries ordered smallest-to-largest (highest mip number first in the array — i.e., level 0 in the array is the smallest mip): [Source](https://github.khronos.org/KTX-Specification/ktxspec.v2.html)

```c
// KTX2 level index entry — from ktxspec.v2.html
// Stored at fileOffset = 80 (after 80-byte header)
struct KTX2LevelIndexEntry {
    uint64_t byteOffset;              // file offset to this level's data
    uint64_t byteLength;              // compressed (or raw) size in bytes
    uint64_t uncompressedByteLength;  // size after Zstd/BasisLZ decompression
                                      // equals byteLength when supercompressionScheme == 0
};
```

### Data Format Descriptor

The DFD block at `dfdByteOffset` provides format semantics independent of `vkFormat`. The KHR Data Format Specification (KhronosGroup/DataFormat) defines the layout: [Source](https://github.khronos.org/KTX-Specification/ktxspec.v2.html)

- `dfdTotalSize` (uint32): total descriptor block size including this field
- `descriptorType` (15 bits in descriptor block header): 0 for basic descriptor
- `vendorId` (16 bits): 0 for Khronos
- `colorModel`: `KHR_DF_MODEL_RGBSDA` for direct GPU formats; `KHR_DF_MODEL_ETC1S (0xA3)` for BasisLZ; `KHR_DF_MODEL_UASTC (0xA6)` for UASTC LDR; `KHR_DF_MODEL_BC7 (0x83)` for BC7
- `colorPrimaries`: `KHR_DF_PRIMARIES_BT709` for colour textures; `KHR_DF_PRIMARIES_UNSPECIFIED` for normal/data maps
- `transferFunction`: `KHR_DF_TRANSFER_SRGB` (0x02) or `KHR_DF_TRANSFER_LINEAR` (0x01)
- `flags`: `KHR_DF_FLAG_ALPHA_PREMULTIPLIED (0x01)` for pre-multiplied alpha blending
- `texelBlockDimension[4]`: block width/height/depth/planes minus 1 (so a 4×4 block → `{3,3,0,0}`)

---

## libktx C API

The `KTX-Software` library (KhronosGroup/KTX-Software, v4.4.2 stable; v5.0.0-rc1 pre-release May 2026) provides the reference C implementation for loading, transcoding, and writing KTX2 files. All API signatures below are verified from `lib/include/ktx.h`, main branch. [Source](https://github.com/KhronosGroup/KTX-Software/releases)

### ktxTexture2 Struct

```c
// lib/include/ktx.h — ktxTexture2 struct layout (KhronosGroup/KTX-Software main)
// KTXTEXTURECLASSDEFN macro expands to the fields shown inline below.
typedef struct ktxTexture2 {
    // Base class fields (from KTXTEXTURECLASSDEFN):
    ktx_bool_t    isArray;
    ktx_bool_t    isCubemap;
    ktx_bool_t    isCompressed;
    ktx_bool_t    generateMipmaps;
    ktx_uint32_t  baseWidth;
    ktx_uint32_t  baseHeight;
    ktx_uint32_t  baseDepth;
    ktx_uint32_t  numDimensions;     // 1, 2, or 3
    ktx_uint32_t  numLevels;
    ktx_uint32_t  numLayers;
    ktx_uint32_t  numFaces;          // 6 for cubemaps, 1 otherwise
    struct ktxOrientation orientation;
    ktxHashList   kvDataHead;
    ktx_uint32_t  kvDataLen;
    ktx_uint8_t*  kvData;
    ktx_size_t    dataSize;          // total pixel data size (after transcode)
    ktx_uint8_t*  pData;             // pointer to pixel data
    // ktxTexture2-specific:
    ktx_uint32_t          vkFormat;  // VkFormat (VK_FORMAT_UNDEFINED for supercompressed)
    ktx_uint32_t*         pDfd;      // pointer to Data Format Descriptor
    ktxSupercmpScheme     supercompressionScheme;
    ktx_bool_t            isVideo;
    ktx_uint32_t          duration;
    ktx_uint32_t          timescale;
    ktx_uint32_t          loopcount;
    struct ktxTexture2_private* _private;  // opaque; do not access directly
} ktxTexture2;
```

**v5.0 breaking change**: The `ktxSupercmpScheme` enum and `ktxBasisParams` struct have changed significantly. See the [ktxBasisParams section](#ktxbasisparams-full-encoder-control) below. Code targeting v4.4.x will require updates for v5.0. [Source](https://github.com/KhronosGroup/KTX-Software/releases)

### Loading and Reading

```c
// lib/include/ktx.h — loading signatures (KhronosGroup/KTX-Software main branch)
// createFlags: KTX_TEXTURE_CREATE_NO_FLAGS (lazy/load-on-demand)
//           or KTX_TEXTURE_CREATE_LOAD_IMAGE_DATA_BIT (load data immediately)

KTX_error_code ktxTexture2_CreateFromNamedFile(
    const char* const filename,
    ktxTextureCreateFlags createFlags,
    ktxTexture2** newTex);

KTX_error_code ktxTexture2_CreateFromMemory(
    const ktx_uint8_t* bytes, ktx_size_t size,
    ktxTextureCreateFlags createFlags, ktxTexture2** newTex);

KTX_error_code ktxTexture2_CreateFromStdioStream(
    FILE* stdioStream,
    ktxTextureCreateFlags createFlags, ktxTexture2** newTex);

// Check if the texture needs transcoding (ETC1S or UASTC supercompressed)
ktx_bool_t ktxTexture2_NeedsTranscoding(ktxTexture2* This);

// Cleanup
void ktxTexture2_Destroy(ktxTexture2* This);
```

### Transcoding Supercompressed Data

```c
// lib/include/ktx.h — transcode to native GPU format
// Must be called before VkUploadEx if NeedsTranscoding() returns KTX_TRUE.
KTX_error_code ktxTexture2_TranscodeBasis(
    ktxTexture2* This,
    ktx_transcode_fmt_e outputFormat,    // KTX_TTF_BC7_RGBA, KTX_TTF_ASTC_4x4_RGBA, etc.
    ktx_transcode_flag_bits_e transcodeFlags);  // 0 for default
```

The `ktx_transcode_fmt_e` enum values (`KTX_TTF_BC7_RGBA`, `KTX_TTF_ASTC_4x4_RGBA`, `KTX_TTF_ETC2_RGBA`, `KTX_TTF_RGBA32`, etc.) are numerically identical to the `transcoder_texture_format` values in `basisu_transcoder.h` — libktx delegates to the basis_universal transcoder internally.

### Creating and Writing KTX2 Files

```c
// lib/include/ktx.h — writing workflow

// 1. Describe the texture
ktxTextureCreateInfo createInfo = {
    .vkFormat       = VK_FORMAT_R8G8B8A8_SRGB,
    .baseWidth      = 1024,
    .baseHeight     = 1024,
    .baseDepth      = 1,
    .numDimensions  = 2,
    .numLevels      = 1,       // set generateMipmaps = KTX_TRUE for auto-mipmaps
    .numLayers      = 1,
    .numFaces       = 1,
    .isArray        = KTX_FALSE,
    .generateMipmaps = KTX_FALSE,
};

ktxTexture2* kTex;
ktxTexture2_Create(&createInfo, KTX_TEXTURE_CREATE_ALLOC_STORAGE, &kTex);

// 2. Copy pixel data into the texture
ktxTexture_SetImageFromMemory(ktxTexture(kTex), 0, 0, 0, pixels, pixelSize);

// 3a. Simple ETC1S compression
ktxTexture2_CompressBasis(kTex, 128 /* qualityLevel 1-255 */);

// 3b. Or full ASTC compression
ktxTexture2_CompressAstcEx(kTex, &astcParams);

// 4. Optionally add Zstandard supercompression to any KTX2 (even already BC-format)
ktxTexture2_DeflateZstd(kTex, 6 /* Zstd compression level 1-22 */);

// 5. Write output
ktxTexture2_WriteToNamedFile(kTex, "output.ktx2");

// Or to memory
ktx_uint8_t* dstBytes; ktx_size_t dstSize;
ktxTexture2_WriteToMemory(kTex, &dstBytes, &dstSize);
// ... free(dstBytes) when done ...

ktxTexture2_Destroy(kTex);
```

**Deprecation note**: `ktxTexture2_GetOETF()` and `ktxTexture2_GetOETF_e()` are deprecated in v4.4+. Use `ktxTexture2_GetTransferFunction_e()` instead. [Source: `lib/include/ktx.h` line 1173, KhronosGroup/KTX-Software main]

### ktxBasisParams: Full Encoder Control

The `ktxBasisParams` struct (verified from `lib/include/ktx.h`, main branch) exposes the full basis_universal encoder API surface:

```c
// lib/include/ktx.h — ktxBasisParams (KhronosGroup/KTX-Software main)
// v5.0 BREAKING CHANGE: ktx_bool_t uastc replaced by ktx_basis_codec_e codec;
//                        compressionLevel renamed to etc1sCompressionLevel.

typedef enum ktx_basis_codec_e {
    KTX_BASIS_CODEC_NONE                        = 0U,
    KTX_BASIS_CODEC_ETC1S                       = 1U,
    KTX_BASIS_CODEC_UASTC_LDR_4x4              = 2U,
    KTX_BASIS_CODEC_UASTC_HDR_4x4              = 3U,
    KTX_BASIS_CODEC_UASTC_HDR_6x6_INTERMEDIATE = 4U,
} ktx_basis_codec_e;

typedef struct ktxBasisParams {
    ktx_uint32_t structSize;           // must be set to sizeof(ktxBasisParams)
    ktx_basis_codec_e codec;           // v5.0: replaces bool uastc
    ktx_bool_t   verbose;
    ktx_bool_t   noSSE;
    ktx_uint32_t threadCount;          // default 1; set > 1 for parallel encoding

    // ETC1S/BasisLZ specific:
    ktx_uint32_t etc1sCompressionLevel; // v5.0 rename from compressionLevel; [0,6]
    ktx_uint32_t qualityLevel;          // [1,255]; default 128
    ktx_uint32_t maxEndpoints;          // [1,16128] endpoint codebook clusters
    ktx_uint32_t maxSelectors;          // [1,16128] selector codebook clusters
    float        endpointRDOThreshold;  // default 1.25
    float        selectorRDOThreshold;  // default 1.5
    ktx_bool_t   noEndpointRDO;
    ktx_bool_t   noSelectorRDO;

    // Input channel controls:
    char         inputSwizzle[4];       // e.g. "rrrg" for two-channel in RG
    ktx_bool_t   preSwizzle;
    ktx_bool_t   normalMap;             // optimise metrics for tangent-space normals

    // UASTC LDR quality:
    ktx_pack_uastc_flags uastcFlags;    // quality level 0-4 via cPackUASTCLevelDefault etc.
    ktx_bool_t   uastcRDO;
    float        uastcRDOQualityScalar; // [0.001, 50.0]; lower = better quality, larger file
    ktx_uint32_t uastcRDODictSize;
    float        uastcRDOMaxSmoothBlockErrScale;

    // UASTC HDR:
    ktx_uint32_t uastcHDRQuality;       // [0,4]
    ktx_bool_t   uastcHDRUberMode;
    float        uastcHDRLambda;        // RDO control for 6x6i
    ktx_uint32_t uastcHDRLevel;         // [0,12] for 6x6i
} ktxBasisParams;
```

Usage with `ktxTexture2_CompressBasisEx`:

```c
// lib/include/ktx.h — full UASTC LDR 4x4 encoding with RDO, verified against v5.0 API
ktxBasisParams params;
memset(&params, 0, sizeof(params));
params.structSize         = sizeof(params);
params.codec              = KTX_BASIS_CODEC_UASTC_LDR_4x4;
params.threadCount        = 8;
params.uastcFlags         = KTX_PACK_UASTC_LEVEL_DEFAULT; // quality 2 of 4
params.uastcRDO           = KTX_TRUE;
params.uastcRDOQualityScalar = 1.0f;  // higher = smaller file, lower quality
params.normalMap          = KTX_FALSE;

KTX_error_code result = ktxTexture2_CompressBasisEx(kTex, &params);
if (result != KTX_SUCCESS) { /* handle error */ }

// Then add Zstandard for ~20-30% additional size reduction:
ktxTexture2_DeflateZstd(kTex, 18);
```

### The ktx Command-Line Tool

The `toktx` tool was deprecated in KTX-Software v4.3 and will be removed in v4.5. The replacement is the unified `ktx` binary with subcommands. [Source](https://github.com/KhronosGroup/KTX-Software/releases)

```bash
# Create a KTX2 file from PNG — explicit VkFormat (no prefix)
ktx create --format R8G8B8A8_SRGB input.png output.ktx2

# Encode with BasisLZ/ETC1S supercompression in one step
ktx create --format R8G8B8A8_SRGB --encode basis-lz \
    --qlevel 128 --mipmap-filter lanczos4 input.png output.ktx2

# Encode UASTC LDR 4x4 with quality 2 and Zstandard supercompression
ktx encode --codec uastc-ldr-4x4 --uastc-quality 2 \
    --zstd 18 input.ktx2 output.ktx2

# UASTC with RDO for smaller files at slight quality cost
ktx encode --codec uastc-ldr-4x4 --uastc-quality 2 \
    --uastc-rdo --uastc-rdo-l 1.0 input.ktx2 output.ktx2

# ASTC 4x4 block compression (native hardware, no supercompression)
ktx create --format ASTC_4x4_SRGB_BLOCK --generate-mipmap \
    input.png output.ktx2

# Normal map — linear transfer function, UASTC for quality
ktx encode --codec uastc-ldr-4x4 --uastc-quality 3 --normal-mode \
    input.ktx2 normalmap_uastc.ktx2

# HDR from EXR: create uncompressed first, then encode
ktx create --format R16G16B16A16_SFLOAT hdr_input.exr hdr.ktx2
ktx encode --codec uastc-hdr-4x4 hdr.ktx2 hdr_compressed.ktx2

# PSNR comparison during encode
ktx encode --codec uastc-ldr-4x4 --compare-psnr input.ktx2 output.ktx2

# Validation and inspection
ktx validate output.ktx2
ktx info output.ktx2
```

Key `ktx encode` codec values: `basis-lz`, `uastc-ldr-4x4`, `uastc-hdr-4x4`, `uastc-hdr-6x6i`. The `--qlevel [1–255]` controls BasisLZ quality; `--uastc-quality [0–4]` controls UASTC; `--zstd [1–22]` and `--zlib [1–9]` add supercompression to non-BasisLZ KTX2. The `--astc-quality` flag accepts `{fastest|fast|medium|thorough|exhaustive}` and controls the `astcenc` backend. [Source: KTX-Software ktxtools reference]

Critical difference from `toktx`: `ktx create` requires an **explicit `--format`** (VkFormat name without `VK_FORMAT_` prefix) and performs no implicit colour-space conversion or resizing. Pre-process images with ImageMagick or FFmpeg before `ktx create` if dimensional constraints apply.

---

## Vulkan Upload Path

### ktxVulkanDeviceInfo and ktxVulkanTexture Structs

```c
// lib/include/ktxvulkan.h — Vulkan integration structs
// KhronosGroup/KTX-Software main branch (lines 107–205, abridged)

typedef struct ktxVulkanDeviceInfo {
    VkInstance       instance;          // added in CreateEx variant
    VkPhysicalDevice physicalDevice;
    VkDevice         device;
    VkQueue          queue;
    VkCommandBuffer  cmdBuffer;
    VkCommandPool    cmdPool;
    const VkAllocationCallbacks* pAllocator;
    VkPhysicalDeviceMemoryProperties deviceMemoryProperties;
    ktxVulkanFunctions vkFuncs;         // Vulkan function pointer table (for custom loaders)
} ktxVulkanDeviceInfo;

typedef struct ktxVulkanTexture {
    PFN_vkDestroyImage vkDestroyImage;  // captured at upload time for correct cleanup
    PFN_vkFreeMemory   vkFreeMemory;
    VkImage            image;
    VkFormat           imageFormat;     // the transcoded VkFormat (e.g. VK_FORMAT_BC7_SRGB_BLOCK)
    VkImageLayout      imageLayout;     // final layout after upload
    VkDeviceMemory     deviceMemory;    // unused when using a suballocator
    VkImageViewType    viewType;        // VK_IMAGE_VIEW_TYPE_2D, _CUBE, _2D_ARRAY, etc.
    uint32_t           width, height, depth;
    uint32_t           levelCount;      // mip levels
    uint32_t           layerCount;
    uint64_t           allocationId;    // for suballocator callbacks
} ktxVulkanTexture;
```

### Full Upload Workflow

```c
// lib/include/ktxvulkan.h — upload function signatures
KTX_API ktxVulkanDeviceInfo* KTX_APIENTRY
ktxVulkanDeviceInfo_Create(
    VkPhysicalDevice physicalDevice, VkDevice device,
    VkQueue queue, VkCommandPool cmdPool,
    const VkAllocationCallbacks* pAllocator);

// CreateEx exposes VkInstance for Vulkan 1.2+ functions and custom function tables
KTX_API ktxVulkanDeviceInfo* KTX_APIENTRY
ktxVulkanDeviceInfo_CreateEx(
    VkInstance instance, VkPhysicalDevice physicalDevice, VkDevice device,
    VkQueue queue, VkCommandPool cmdPool,
    const VkAllocationCallbacks* pAllocator,
    const ktxVulkanFunctions* pFunctions);

// Primary upload entry point — preferred over ktxTexture2_VkUpload
KTX_API KTX_error_code KTX_APIENTRY
ktxTexture2_VkUploadEx(
    ktxTexture2* This, ktxVulkanDeviceInfo* vdi,
    ktxVulkanTexture* vkTexture,
    VkImageTiling     tiling,       // VK_IMAGE_TILING_OPTIMAL (recommended) or LINEAR
    VkImageUsageFlags usageFlags,   // e.g. VK_IMAGE_USAGE_SAMPLED_BIT
    VkImageLayout     finalLayout); // e.g. VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL

// With suballocator support (for engines managing a single large VkDeviceMemory)
KTX_API KTX_error_code KTX_APIENTRY
ktxTexture2_VkUploadEx_WithSuballocator(
    ktxTexture2* This, ktxVulkanDeviceInfo* vdi,
    ktxVulkanTexture* vkTexture,
    VkImageTiling tiling, VkImageUsageFlags usageFlags, VkImageLayout finalLayout,
    ktxVulkanTexture_subAllocatorCallbacks* subAllocatorCallbacks);

// Format queries
KTX_API VkFormat KTX_APIENTRY ktxTexture2_GetVkFormat(ktxTexture2* This);
```

The following complete example shows the load → feature query → transcode → upload sequence:

```c
// Full KTX2 Vulkan upload workflow
// Pattern from: docs.vulkan.org/samples/latest/samples/performance/texture_compression_basisu/README.html
// and lib/include/ktxvulkan.h, KhronosGroup/KTX-Software main branch

// Step 1: Query device feature support for transcode target selection
VkPhysicalDeviceFeatures features;
vkGetPhysicalDeviceFeatures(physicalDevice, &features);

ktx_transcode_fmt_e transcode_target = KTX_TTF_RGBA32;  // safe fallback
if (features.textureCompressionBC) {
    // Desktop path: prefer BC7 for highest quality
    VkFormatProperties fp;
    vkGetPhysicalDeviceFormatProperties(physicalDevice,
        VK_FORMAT_BC7_SRGB_BLOCK, &fp);
    if (fp.optimalTilingFeatures & VK_FORMAT_FEATURE_SAMPLED_IMAGE_BIT)
        transcode_target = KTX_TTF_BC7_RGBA;
    else
        transcode_target = KTX_TTF_BC1_RGB;
} else if (features.textureCompressionASTC_LDR) {
    // ARM Mali / Adreno path
    transcode_target = KTX_TTF_ASTC_4x4_RGBA;
} else if (features.textureCompressionETC2) {
    // Embedded Linux fallback
    transcode_target = KTX_TTF_ETC2_RGBA;
}

// Step 2: Load the KTX2 file
ktxTexture2* kTexture;
KTX_error_code result = ktxTexture2_CreateFromNamedFile(
    "asset.ktx2",
    KTX_TEXTURE_CREATE_LOAD_IMAGE_DATA_BIT,
    &kTexture);
if (result != KTX_SUCCESS) { /* handle error */ }

// Step 3: Transcode if the file contains BasisLZ/ETC1S or UASTC supercompression
if (ktxTexture2_NeedsTranscoding(kTexture)) {
    result = ktxTexture2_TranscodeBasis(kTexture, transcode_target, 0);
    if (result != KTX_SUCCESS) { /* handle error */ }
}
// After TranscodeBasis: kTexture->vkFormat holds the transcoded VkFormat,
// kTexture->pData holds native block-compressed data, kTexture->dataSize is updated.

// Step 4: Set up Vulkan device info
ktxVulkanDeviceInfo* vdi = ktxVulkanDeviceInfo_Create(
    physicalDevice, device, queue, cmdPool, NULL);

// Step 5: Upload — allocates staging VkBuffer, copies data, creates VkImage,
// records vkCmdCopyBufferToImage for each mip level, submits, transitions layout
ktxVulkanTexture kvt;
result = ktxTexture2_VkUploadEx(kTexture, vdi, &kvt,
    VK_IMAGE_TILING_OPTIMAL,
    VK_IMAGE_USAGE_SAMPLED_BIT,
    VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
if (result != KTX_SUCCESS) { /* handle error */ }

// Step 6: Create VkImageView using KTX2 metadata for correct dimensionality
VkImageViewCreateInfo view_info = {
    .sType    = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO,
    .image    = kvt.image,
    .viewType = kvt.viewType,     // correctly set to CUBE, 2D_ARRAY, etc. by libktx
    .format   = kvt.imageFormat,
    .subresourceRange = {
        .aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT,
        .baseMipLevel   = 0,
        .levelCount     = kvt.levelCount,
        .baseArrayLayer = 0,
        .layerCount     = kvt.layerCount,
    },
};
VkImageView imageView;
vkCreateImageView(device, &view_info, NULL, &imageView);

// Step 7: Cleanup (after descriptor set is bound; free on device idle)
ktxVulkanTexture_Destruct(&kvt, device, NULL);
ktxVulkanDeviceInfo_Destroy(vdi);
ktxTexture2_Destroy(kTexture);
```

The `ktxTexture2_VkUploadEx` implementation internally: (a) creates a host-visible staging `VkBuffer` sized `kTexture->dataSize`, (b) maps and copies `kTexture->pData`, (c) creates the target `VkImage` with `imageType`/`extent`/`mipLevels`/`arrayLayers`/`format` from the transcoded ktxTexture2 fields, (d) records `vkCmdCopyBufferToImage` for each mip level using `ktxTexture2_GetImageOffset()` for byte offsets, (e) submits the command buffer, and (f) transitions to `finalLayout` via an image memory barrier. The staging buffer is destroyed after queue submission. [Source](https://docs.vulkan.org/samples/latest/samples/performance/texture_compression_basisu/README.html)

For DRM format modifier negotiation with tiled GPU memory layouts (relevant on Mali and some Intel iGPU configurations), see Chapter 4 — libktx does not currently expose DRM modifier selection but the resulting `VkImage` can be used with `VK_EXT_image_drm_format_modifier` by wrapping the upload in a custom command buffer. [Source](https://docs.vulkan.org/spec/latest/appendices/compressedtex.html)

---

## glTF 2.0 and KHR_texture_basisu

### Extension Specification

The `KHR_texture_basisu` extension (Khronos ratified) adds an alternative `source` property to glTF `texture` objects pointing to a `.ktx2` image entry (MIME type `image/ktx2`). The extension supports both a fallback PNG path (for renderers without KTX2 support) and a KTX2-required path: [Source](https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_texture_basisu/README.md)

```json
// With PNG fallback — renderer uses KTX2 if it can, PNG otherwise
{
  "extensionsUsed": ["KHR_texture_basisu"],
  "textures": [{
    "source": 0,
    "extensions": {
      "KHR_texture_basisu": { "source": 1 }
    }
  }],
  "images": [
    { "uri": "albedo.png" },
    { "uri": "albedo.ktx2" }
  ]
}
```

```json
// KTX2 required — no PNG fallback; extensionsRequired forces loader compliance
{
  "extensionsUsed": ["KHR_texture_basisu"],
  "extensionsRequired": ["KHR_texture_basisu"],
  "textures": [{
    "extensions": {
      "KHR_texture_basisu": { "source": 0 }
    }
  }],
  "images": [{ "uri": "albedo.ktx2" }]
}
```

Conformance requirements from the extension specification:

- Swizzle metadata: `rgba` or omitted (no channel reordering)
- Orientation: `rd` (right-down, standard 2D) or omitted
- Colour textures: BT.709 primaries + sRGB transfer function (DFD `transferFunction = KHR_DF_TRANSFER_SRGB`)
- Non-colour textures (normal maps, metallic-roughness): unspecified primaries + linear transfer function
- All dimensions must be multiples of 4 (BC/ETC/ASTC block alignment requirement)
- `KHR_DF_FLAG_ALPHA_PREMULTIPLIED` is disallowed except where the material model requires pre-multiplied alpha

### tinygltf Image Hook for KTX2

tinygltf (syoyo/tinygltf, header-only C++11) supports custom image loaders via `TinyGLTF::SetImageLoader`. This is the integration point for KTX2 handling: [Source](https://docs.vulkan.org/tutorial/latest/15_GLTF_KTX2_Migration.html)

```cpp
// Custom KTX2 image loader for tinygltf
// Pattern: docs.vulkan.org/tutorial/latest/15_GLTF_KTX2_Migration.html
#define TINYGLTF_IMPLEMENTATION
#define TINYGLTF_NO_STB_IMAGE      // disable default stb_image loader
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "tiny_gltf.h"
#include <ktx.h>                   // KhronosGroup/KTX-Software

struct VulkanContext {
    VkPhysicalDeviceFeatures features;
    ktx_transcode_fmt_e preferred_target;
};

bool LoadKTX2Image(tinygltf::Image* image, const int /*image_idx*/,
                   std::string* err, std::string* /*warn*/,
                   int /*req_width*/, int /*req_height*/,
                   const unsigned char* bytes, int size,
                   void* user_data) {
    // Only handle .ktx2 URIs; fall through for PNG/JPG
    if (image->uri.find(".ktx2") == std::string::npos &&
        image->mimeType != "image/ktx2")
        return false;

    VulkanContext* ctx = static_cast<VulkanContext*>(user_data);

    ktxTexture2* kTex;
    KTX_error_code result = ktxTexture2_CreateFromMemory(
        reinterpret_cast<const ktx_uint8_t*>(bytes),
        static_cast<ktx_size_t>(size),
        KTX_TEXTURE_CREATE_LOAD_IMAGE_DATA_BIT,
        &kTex);
    if (result != KTX_SUCCESS) {
        *err = "ktxTexture2_CreateFromMemory failed: " + std::to_string(result);
        return false;
    }

    if (ktxTexture2_NeedsTranscoding(kTex)) {
        result = ktxTexture2_TranscodeBasis(kTex, ctx->preferred_target, 0);
        if (result != KTX_SUCCESS) {
            ktxTexture2_Destroy(kTex);
            *err = "TranscodeBasis failed";
            return false;
        }
    }

    // Store the transcoded texture pointer in tinygltf::Image::extras
    // for later ktxTexture2_VkUploadEx in the asset-load phase.
    // Application-specific: store kTex pointer via image->extras or a side-table.
    // The caller is responsible for ktxTexture2_Destroy after GPU upload.

    // For simplicity here: extract RGBA32 fallback into tinygltf image buffer
    if (ctx->preferred_target == KTX_TTF_RGBA32) {
        image->width  = kTex->baseWidth;
        image->height = kTex->baseHeight;
        image->component = 4;
        image->bits    = 8;
        image->image.assign(kTex->pData, kTex->pData + kTex->dataSize);
    }
    // else: application stores kTex for Vulkan upload path

    ktxTexture2_Destroy(kTex);
    return true;
}

// Registration:
VulkanContext vk_ctx;
vkGetPhysicalDeviceFeatures(physicalDevice, &vk_ctx.features);
vk_ctx.preferred_target = select_transcode_target(vk_ctx.features); // as shown above

tinygltf::TinyGLTF loader;
loader.SetImageLoader(LoadKTX2Image, &vk_ctx);

tinygltf::Model model;
std::string err, warn;
loader.LoadBinaryFromFile(&model, &err, &warn, "scene.glb");
```

The `KHR_texture_basisu` extension source index is found via: `texture.extensions["KHR_texture_basisu"]["source"]` — in tinygltf, accessible as `texture.extensions.Get("KHR_texture_basisu").Get("source").GetNumberAsInt()`. When the fallback PNG source is present (primary `texture.source`), the loader should prefer the KTX2 source when the extension is available.

### cgltf Integration for KTX2

For C99 codebases, [cgltf](https://github.com/jkuhlmann/cgltf) (jkuhlmann/cgltf, single-header C99) provides a lower-level alternative to tinygltf with first-class `KHR_texture_basisu` support baked directly into the `cgltf_texture` struct — no extension-string lookup needed:

```c
// cgltf.h — cgltf_texture struct (jkuhlmann/cgltf master branch)
typedef struct cgltf_texture {
    char*           name;
    cgltf_image*    image;          // primary image source (PNG/JPG fallback)
    cgltf_sampler*  sampler;
    cgltf_bool      has_basisu;     // true when KHR_texture_basisu extension present
    cgltf_image*    basisu_image;   // points to the .ktx2 image entry
    cgltf_bool      has_webp;
    cgltf_image*    webp_image;
    cgltf_extras    extras;
    cgltf_size      extensions_count;
    cgltf_extension* extensions;
} cgltf_texture;
```

After calling `cgltf_parse_file()` and `cgltf_load_buffers()`, discovering and loading a KTX2 URI from a parsed glTF is a direct struct field access rather than an extension map traversal:

```c
// cgltf KHR_texture_basisu discovery pattern
// Source: jkuhlmann/cgltf README and cgltf.h struct definition
for (cgltf_size i = 0; i < data->textures_count; ++i) {
    cgltf_texture* tex = &data->textures[i];
    if (tex->has_basisu && tex->basisu_image != NULL) {
        // Prefer the KTX2 image URI over the PNG fallback
        const char* ktx2_uri = tex->basisu_image->uri;
        // Load ktx2_uri via ktxTexture2_CreateFromNamedFile or from
        // embedded buffer view (tex->basisu_image->buffer_view)
        load_ktx2_texture(ktx2_uri);
    } else if (tex->image != NULL) {
        // No KTX2 available — fall back to PNG/JPG
        load_png_texture(tex->image->uri);
    }
}
```

The key distinction from tinygltf: **cgltf** exposes the KTX2 URI via `cgltf_texture.basisu_image->uri` (or `->buffer_view` for embedded GLB assets), while **tinygltf** requires JSON traversal through `texture.extensions.Get("KHR_texture_basisu").Get("source")` to obtain the image index. Both parsers deliver the raw `.ktx2` bytes; the subsequent `ktxTexture2_CreateFromMemory` → `ktxTexture2_TranscodeBasis` → `ktxTexture2_VkUploadEx` sequence is identical regardless of which glTF parser is used. [Source: jkuhlmann/cgltf](https://github.com/jkuhlmann/cgltf)

### Full Pipeline: .glb to VkImage

```text
.glb/.gltf parse (tinygltf)
  ↓ discover KHR_texture_basisu extension on each texture object
  ↓ load image bytes from URI (embedded bufferView or external .ktx2 file)
  ↓ ktxTexture2_CreateFromMemory
      (parses KTX2 header, DFD, level index table, supercompression global data)
  ↓ ktxTexture2_NeedsTranscoding → ktxTexture2_TranscodeBasis
      (ETC1S→BC7 via lookup table; UASTC→BC7 via hint-guided block decompose+reencode;
       UASTC→ASTC 4x4 via nearly zero-cost bit manipulation)
  ↓ ktxVulkanDeviceInfo_Create / ktxVulkanDeviceInfo_CreateEx
  ↓ ktxTexture2_VkUploadEx
      (allocate staging VkBuffer → copy pData → create VkImage →
       vkCmdCopyBufferToImage per mip → pipeline barrier → SHADER_READ_ONLY layout)
  ↓ vkCreateImageView (using kvt.viewType, kvt.imageFormat, kvt.levelCount, kvt.layerCount)
  ↓ bind VkDescriptorSet for PBR shader (VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER)
```

This pipeline is covered in practical detail in the Vulkan Documentation Project's glTF+KTX2 migration guide. [Source](https://docs.vulkan.org/tutorial/latest/15_GLTF_KTX2_Migration.html)

---

## Compression Quality Trade-offs

### Format Selection Matrix

The choice of supercompression codec is driven by texture content type, not merely by available hardware support:

| Content Type | Recommended Codec | Rationale |
|---|---|---|
| Diffuse / albedo (sRGB) | ETC1S / BasisLZ | Smallest files; shared codebook amortises across texture atlases; smooth colour fields compress well via VQ |
| Normal maps (tangent-space) | UASTC LDR 4×4 | ETC1S degrades badly on high-frequency XY normal data; UASTC near-BC7 quality; use `--normal-mode` |
| Metallic-roughness, AO | UASTC LDR 4×4 | Data textures with precise gradient requirements; ETC1S unsuitable for precision |
| UI textures (flat colour, icons) | ETC1S | Flat colour regions are trivially clustered by VQ; very small codebooks suffice |
| HDR environment maps | UASTC HDR 4×4 | Preserves FP16 range; transcodes to BC6H on desktop |
| HDR with size constraints | ASTC HDR 6×6 | 3.56 bpp vs. 8 bpp for UASTC HDR 4×4; notable quality reduction |
| Large texture atlases | ETC1S | Global codebook shared across all sub-textures; better effective compression than per-image BC7 |
| Sprite sheets with many tiny images | ETC1S | VQ clusters colour endpoints across the entire atlas |

For atlases combining albedo (sRGB) and non-colour channels (normal, metallic-roughness), the pragmatic choice is: pack albedo into a separate ETC1S-encoded KTX2, and pack normal+metallic-roughness into a UASTC-encoded KTX2. This avoids applying UASTC's higher per-block cost to content where ETC1S suffices.

### PSNR Tiers

Quality measurements on photographic content (verified from public benchmarks): [Source](https://aras-p.info/blog/2020/12/08/Texture-Compression-in-2020/)

| Codec (target after transcode) | PSNR range | Bit rate |
|---|---|---|
| BC7 / ASTC 4×4 / UASTC 4×4 (direct) | >42 dB | 8 bpp |
| UASTC 4×4 → BC7 (transcoded) | >40.5–41.5 dB | 8 bpp after transcode |
| ETC1S → BC7 | ~39–41 dB | 0.3–3 bpp (effective) |
| ETC2 / BC3 / ASTC 6×6 | 35–40 dB | 4–3.56 bpp |
| ASTC 8×8 (2 bpp) | <35 dB | 2 bpp |
| ETC1S → BC1 | ~35–38 dB | 0.3–3 bpp (effective) |

UASTC→BC7 transcoding loses 0.75–1.5 dB RGB PSNR compared to direct BC7 encoding from the original source. ETC1S→BC7 using BC7 mode 6 is nearly lossless within the ETC1S quality envelope — the BC7 encoder simply matches the ETC1S block's endpoint colours. [Source](https://github.com/BinomialLLC/basis_universal)

### ETC1S Global Codebook Size Effect

`maxEndpoints` [1–16,128] and `maxSelectors` [1–16,128] directly control the VQ codebook granularity. At `qualityLevel=1`, the encoder auto-selects a codebook of roughly 128 endpoints and 128 selectors — yielding very small files but visible quantisation. At `qualityLevel=255`, codebooks of ~8,000–16,000 entries are used per dimension, approaching BC1 quality at a fraction of the per-block storage. [Source: `lib/include/ktx.h` ktxBasisParams documentation, KhronosGroup/KTX-Software main]

For texture atlases, the critical insight is that the **same codebook covers all sub-textures** in the file. An atlas of 1,024 individual 64×64 textures sharing a common art style (e.g., a pixel-art sprite sheet) will compress far better than 1,024 independent BC1-compressed images, because the VQ clusters find compact endpoint representations across the entire image set. Disable `noEndpointRDO` and `noSelectorRDO` for maximum effective compression — the RDO pass re-selects codebook entries per block to minimise perceptual error while maximising shared-entry reuse.

### CPU vs GPU Transcoder Throughput

The Basis Universal runtime transcoder runs on the **CPU** and is intentionally lightweight: the architecture (line-of-sight from ETC1S's 1D lookup tables and UASTC's per-block hint bits) avoids the decompressor state that would require heap allocations or coherency barriers. As noted in the Architecture Overview, a single core transcoding one 4K UASTC LDR 4×4 texture to BC7 takes approximately **1 ms** on a modern x86_64 core at release-build optimisation. ETC1S→BC1 is faster still — the lookup-table construction means a 4K ETC1S frame can transcode in roughly **0.5 ms** on the same core, and both operations scale near-linearly with thread count via per-image parallelism.

**GPU compute shader transcoding** (decode from Basis intermediate on the GPU) is a different trade-off: it offloads CPU time entirely, but requires uploading the compressed intermediate to GPU memory first, dispatching a compute shader, and synchronising before sampling. This is advantageous only when CPU cores are the bottleneck (e.g., streaming hundreds of textures simultaneously). For typical game-engine load-screen scenarios with 20–100 textures, the CPU path finishes before the GPU upload would even complete. Production engines (Godot 4, Bevy, Unreal) all default to the CPU transcoder path.

The practical guidance: for **foreground asset loads** (character models at camera), CPU transcode + immediate `ktxTexture2_VkUploadEx` delivers the best latency. For **background streaming** (terrain, distant LODs), multi-threaded CPU transcoding (`params.threadCount = N`) fills the GPU upload queue without blocking the render thread. Basis Universal's `basisu_transcoder_state` allows reuse across calls on the same thread without re-allocating decode state. If ETC1S→ASTC 4×4 is the target (embedded Linux / ARM Mali), the TMU's native ASTC decode means there is **no quality loss** relative to software-decoded RGBA32 — making UASTC→ASTC 4×4 transcoding the closest approximation to zero-cost transcoding available in the ecosystem.

> **Note**: Published absolute throughput numbers (MB/s) for the CPU transcoder vary significantly with texture resolution, codec target, and CPU microarchitecture. The figures above are sourced from the basis_universal README (which anchors "negligible CPU cost", ≈1 ms/4K texture) and ETC1S encoder benchmarks on the project wiki. Authoritative published benchmarks should be verified against the [basis_universal wiki](https://github.com/BinomialLLC/basis_universal/wiki) for the specific codec target and hardware of interest.

### Real-World Compression Numbers

A practical reference point: converting the [glTF FlightHelmet](https://github.com/KhronosGroup/glTF-Sample-Models) model from PNG textures to KTX2 (ETC1S for albedo maps, UASTC for normal maps) produced a **32% overall reduction**: 43.06 MB → 29.37 MB total. BaseColor PNG savings were the largest contributor (~4.75 MB per map). [Source](https://deepwiki.com/KhronosGroup/glTF-Sample-Models/5.1-texture-compression-with-ktx2-and-basis-universal)

Bevy's KTX2 asset loader (`bevy_render/src/texture/ktx2.rs`, using the `ktx2` Rust crate v0.4.0 as of PR #18411, 2025) now outputs `.ktx2` files instead of `.basis` from its asset processor. Two Zstandard feature flags control the backend: `"zstd_native"` (44% faster, native bindings via `libzstd`) vs `"zstd_rust"` (pure Rust `ruzstd`, no native dependency). [Source](https://github.com/bevyengine/bevy/pull/18411)

---

## Integrations

**Chapter 24 — Vulkan Memory and Resources**: The `ktxTexture2_VkUploadEx` path creates `VkImage` and `VkDeviceMemory` objects following exactly the staging-buffer upload pattern described in Chapter 24. Understanding `VkMemoryPropertyFlags`, `VkImageTiling`, and `vkCmdCopyBufferToImage` is prerequisite reading.

**Chapter 4 — DRM and KMS**: DRM format modifier negotiation (`VK_EXT_image_drm_format_modifier`) for tiled/swizzled GPU memory layouts connects to the `VkImageTiling` selection in `ktxTexture2_VkUploadEx`. On Mali SoCs, AFBC-compressed textures require modifier negotiation that libktx does not currently automate — applications targeting Panfrost must handle this separately.

**Chapters 6 and 19 — ARM Mali drivers (Panfrost/Panthor) and Mesa Vulkan drivers**: ASTC is the native block-compression format on all Mali GPUs. UASTC→ASTC 4×4 transcoding is the zero-cost path (essentially a bit-level copy plus header adjustment) and is the recommended production pipeline for ARM Linux targets (i.MX 8, Rockchip RK3588, Raspberry Pi 5 with VideoCore VII).

**Chapter 40 — Bevy**: Bevy's asset pipeline consumes KTX2 as its preferred texture format. The `ktx2` Rust crate wraps the C transcoder. Bevy assets encoded with `ktx encode --codec basis-lz --zstd 18` for diffuse and `ktx encode --codec uastc-ldr-4x4 --zstd 18` for normal maps are the production-recommended configuration.

**Chapter 41 — Godot**: Godot's `ResourceImporterTexture` supports KTX2 and Basis Universal import via its built-in compression pipeline. glTF files with `KHR_texture_basisu` are fully supported in Godot 4.2+.

**Chapter 42 — Blender**: Blender's glTF importer reads `KHR_texture_basisu` texture references; the exporter supports encoding textures as KTX2 via the `ktx` CLI tool in the asset export pipeline.

**Chapter 64 — glTF 2.0**: The `KHR_texture_basisu` extension described in this chapter is one of the most widely adopted glTF extensions. Chapter 64 covers the full glTF JSON/binary format and all officially ratified Khronos extensions. The PBR material model's five texture maps (base colour, metallic-roughness, normal, occlusion, emissive) each have distinct compression requirements as described in the Format Selection Matrix above.

**Chapter 61 — SPIR-V and Shader Compilation**: ASTC decode compute shaders (software decode fallback on GPUs without native ASTC hardware) compile to SPIR-V via the same glslang/DXC pipeline described in Chapter 61. The Mesa RADV and NVK drivers implement software ASTC decode in SPIR-V compute for Vulkan compliance on hardware without native ASTC support.

---

## References

1. [KhronosGroup/KTX-Software releases — v4.4.2 and v5.0.0-rc1](https://github.com/KhronosGroup/KTX-Software/releases)
2. [BinomialLLC/basis_universal — main repository, v2.1.0r](https://github.com/BinomialLLC/basis_universal)
3. [KTX2 File Format Specification v2.0 (Khronos Registry)](https://github.khronos.org/KTX-Specification/ktxspec.v2.html)
4. [IANA media type registration: image/ktx2](https://www.iana.org/assignments/media-types/image/ktx2)
5. [Maister's Graphics Adventures: Compressed GPU Texture Formats Part 1 (BC1–BC5)](https://themaister.net/blog/2020/08/12/compressed-gpu-texture-formats-a-review-and-compute-shader-decoders-part-1/)
6. [Maister's Graphics Adventures: Compressed GPU Texture Formats Part 2 (BC6H, BC7, ETC2, ASTC)](https://themaister.net/blog/2020/08/30/compressed-gpu-texture-formats-a-review-and-compute-shader-decoders-part-2/)
7. [Aras Pranckevičius: Texture Compression in 2020 (PSNR benchmarks)](https://aras-p.info/blog/2020/12/08/Texture-Compression-in-2020/)
8. [Vulkan Specification: Compressed Image Formats appendix](https://docs.vulkan.org/spec/latest/appendices/compressedtex.html)
9. [Khronos OpenGL Wiki: BPTC Texture Compression (BC6H, BC7)](https://www.khronos.org/opengl/wiki/BPTC_Texture_Compression)
10. [ARM Vulkan SDK: ASTC Texture Compression](https://arm-software.github.io/vulkan-sdk/_a_s_t_c.html)
11. [astc-encoder: ASTC Format Overview (ARM-software/astc-encoder)](https://chromium.googlesource.com/external/github.com/ARM-software/astc-encoder/+/HEAD/Docs/FormatOverview.md)
12. [ETC2 Primer (informal)](https://nicjohnson6790.github.io/etc2-primer/)
13. [Rich Geldreich: ETC1S texture format encoding (2018)](http://richg42.blogspot.com/2018/06/etc1s-texture-format-encoding.html)
14. [basis_universal: XUASTC LDR Specification v1.0](https://github.com/BinomialLLC/basis_universal/wiki/XUASTC-LDR-Specification-v1.0)
15. [glTF extension: KHR_texture_basisu (KhronosGroup/glTF)](https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_texture_basisu/README.md)
16. [Vulkan Documentation: glTF+KTX2 Migration Guide](https://docs.vulkan.org/tutorial/latest/15_GLTF_KTX2_Migration.html)
17. [Vulkan Samples: texture_compression_basisu performance sample](https://docs.vulkan.org/samples/latest/samples/performance/texture_compression_basisu/README.html)
18. [ARM Community Blog: Texture compression energy cost on Mali](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/arm-guide-for-unreal-engine-4-best-practices-for-mobile-developers)
19. [DeepWiki: glTF-Sample-Models KTX2 compression numbers (FlightHelmet)](https://deepwiki.com/KhronosGroup/glTF-Sample-Models/5.1-texture-compression-with-ktx2-and-basis-universal)
20. [Bevy PR #18411: ktx2 crate v0.4.0 with ASTC HDR and ETC1S support](https://github.com/bevyengine/bevy/pull/18411)
21. [KhronosGroup/KTX-Software `lib/include/ktx.h` — ktxTexture2, ktxBasisParams, ktx_transcode_fmt_e](https://github.com/KhronosGroup/KTX-Software/blob/main/lib/include/ktx.h)
22. [KhronosGroup/KTX-Software `lib/include/ktxvulkan.h` — ktxVulkanDeviceInfo, ktxVulkanTexture, upload functions](https://github.com/KhronosGroup/KTX-Software/blob/main/lib/include/ktxvulkan.h)
23. [BinomialLLC/basis_universal `transcoder/basisu_transcoder.h` — transcoder API, ktx2_supercompression enum](https://github.com/BinomialLLC/basis_universal/blob/master/transcoder/basisu_transcoder.h)
24. [BinomialLLC/basis_universal `transcoder/basisu_file_headers.h` — basis_file_header struct](https://github.com/BinomialLLC/basis_universal/blob/master/transcoder/basisu_file_headers.h)
25. [google/etc2comp — ETC2/EAC offline encoder tool and library](https://github.com/google/etc2comp)
26. [jkuhlmann/cgltf — single-file glTF 2.0 loader in C99 with KHR_texture_basisu support](https://github.com/jkuhlmann/cgltf)
