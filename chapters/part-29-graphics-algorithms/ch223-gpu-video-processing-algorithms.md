# Chapter 223: GPU Video Processing Algorithms

**Audiences:** Systems developers building GPU-accelerated video transcode or analysis pipelines on
Linux; browser and web platform engineers working with WebCodecs and GPU-backed canvas video;
application developers using GStreamer, FFmpeg, or VA-API for video processing beyond simple
decode/encode.

**Scope note.** Chapter 26 covers the VA-API and V4L2 *API layer* for hardware video
decode/encode; Chapter 165 (Chapter 50 in the published numbering) covers the Vulkan Video
*extensions*. This chapter covers the **algorithms** executed during video processing — the
mathematical and computational techniques that operate on video data — independent of which API
exposes them. The API chapters explain the system calls and Vulkan extension structs; this chapter
explains what the hardware is computing behind those calls and how to replicate or augment those
computations in general-purpose Vulkan compute shaders.

---

## Table of Contents

- [1. Video as a GPU Algorithm Domain](#1-video-as-a-gpu-algorithm-domain)
- [2. Motion Estimation and Compensation](#2-motion-estimation-and-compensation)
- [3. Intra Prediction](#3-intra-prediction)
- [4. Transform Coding on GPU](#4-transform-coding-on-gpu)
- [5. Quantisation and Rate Control](#5-quantisation-and-rate-control)
- [6. Deblocking and Loop Filters](#6-deblocking-and-loop-filters)
- [7. Temporal Processing: Deinterlacing and Frame Interpolation](#7-temporal-processing-deinterlacing-and-frame-interpolation)
- [8. Video Super-Resolution](#8-video-super-resolution)
- [9. HDR and Wide Color Gamut Video Processing](#9-hdr-and-wide-color-gamut-video-processing)
- [10. Video Analysis Algorithms](#10-video-analysis-algorithms)
- [11. 360° Video and Immersive Formats](#11-360-video-and-immersive-formats)
- [12. GStreamer and FFmpeg GPU Pipeline Integration](#12-gstreamer-and-ffmpeg-gpu-pipeline-integration)
- [Integrations](#integrations)

---

## 1. Video as a GPU Algorithm Domain

Video differs from static image processing in three fundamental ways that shape every algorithmic
decision: **temporal redundancy**, **inter-frame causal dependencies**, and the need for
**rate-distortion optimisation** across an entire sequence rather than a single frame.

### Temporal Redundancy

A typical broadcast scene at 1080p/60 changes fewer than 15% of its pixels per frame; a static
camera shot may change fewer than 2%. Codecs exploit this by partitioning the signal into spatial
components that are encoded once per intra period and residual components that are transmitted as
differences from a prediction. The GPU excels at computing those residuals in parallel across
thousands of macroblocks simultaneously, but the benefit is conditional: residuals are only cheap
to store if prediction is accurate, and accurate prediction requires motion search — itself
expensive.

### The Codec Algorithm Stack

Every block-based video codec processes data through five layered stages:

1. **Prediction** — intra (spatial) or inter (temporal motion-compensated)
2. **Residual transform** — DCT or integer approximation thereof
3. **Quantisation** — lossy scalar quantisation of transform coefficients
4. **Entropy coding** — CABAC, CAVLC, or ans/range coding
5. **In-loop filtering** — deblocking, SAO, CDEF, Wiener restoration

GPU parallelism applies well to stages 1–3 and stage 5: these are data-parallel over independent
blocks. Stage 4 (entropy coding) is fundamentally sequential within each slice or tile; CABAC
arithmetic coding maintains a probability context that depends on each previously coded symbol.
Modern codecs break this dependency by partitioning into independent tiles (HEVC, AV1) or
independent segments (VP9), allowing one entropy-coding thread per tile — at the cost of a few
percent compression efficiency compared to a single-pass coder.

### GPU vs. Fixed-Function ASIC Trade-offs

All three major GPU vendors on Linux ship fixed-function video encode/decode blocks:

- **AMD VCN** (Video Core Next): present in RDNA2 and later discrete GPUs and APUs; exposes
  H.264, HEVC, AV1 encode and decode via VA-API (`radeonsi` driver) and Vulkan Video (RADV).
  VCN 4.0 (RDNA3) adds AV1 encode. [Source: AMD open-source driver documentation,
  https://github.com/GPUOpen-Drivers/pal]
- **Intel Quick Sync**: available on all Intel Xe-series integrated graphics; exposed through
  VA-API (`iHD` driver) with AV1 decode from Xe (Tiger Lake) and AV1 encode from Alder Lake
  onwards. [Source: Intel Media SDK / oneVPL, https://github.com/intel/libvpl]
- **NVIDIA NVENC / NVDEC**: binary firmware blocks on all post-Fermi NVIDIA GPUs; no open driver
  exposes these on Linux (Nouveau/NVK lacks firmware access). The proprietary driver exposes them
  through CUDA Video (NVCUVID), NVENC API, and increasingly through Vulkan Video extensions on
  driver ≥535. [Source: NVIDIA Codec SDK, https://developer.nvidia.com/nvidia-video-codec-sdk]

Shader-based encode (i.e., running the codec in Vulkan compute without VCN/Quick Sync/NVENC)
trades 3–10× higher power consumption and 2–5× higher latency for architectural flexibility:
arbitrary bit depths, non-standard block sizes, algorithm experimentation, and operation on GPUs
that lack dedicated codec hardware (e.g., Mali, PowerVR, or any GPU in a system where the
fixed-function block is occupied). The sections below describe the shader-side algorithms
regardless of whether they run on a dedicated block or on shader cores.

---

## 2. Motion Estimation and Compensation

Motion estimation (ME) finds, for each block in the current frame, the best-matching region in
one or more reference frames. Motion compensation (MC) then uses the found vectors to reconstruct
a prediction, which is subtracted from the current frame to yield a residual.

### Block-Matching Search Strategies

The reference algorithm is **full search** (exhaustive): for a block at position `(bx, by)` of
size `BW×BH`, every candidate `(cx, cy)` within a search window `[-R, R]` is tested. The cost
metric is Sum of Absolute Differences (SAD):

```c
uint32_t sad_4x4(const uint8_t *src, int src_stride,
                 const uint8_t *ref, int ref_stride) {
    uint32_t s = 0;
    for (int y = 0; y < 4; y++)
        for (int x = 0; x < 4; x++)
            s += abs((int)src[y*src_stride+x] - (int)ref[y*ref_stride+x]);
    return s;
}
```

Full search over a ±32-pixel window for a 16×16 macroblock requires 65×65 = 4,225 SAD evaluations
each touching 256 pixel pairs. GPU parallelisation assigns one thread block per macroblock
candidate and computes SAD in shared memory using parallel reduction:

```glsl
// Vulkan compute shader: one workgroup per (candidate_x, candidate_y) pair
// workgroup size: 16x16 threads, one per pixel in the macroblock
layout(local_size_x = 16, local_size_y = 16) in;

layout(binding = 0) uniform sampler2D src_frame;
layout(binding = 1) uniform sampler2D ref_frame;
layout(binding = 2, std430) buffer SadOutput { uint sad_values[]; };

shared uint partial_sad[256];  // 16x16 = 256 threads

layout(push_constant) uniform PC {
    ivec2 block_origin;   // top-left of macroblock in source
    ivec2 candidate;      // candidate displacement
    int   mb_index;       // flat index into sad_values
} pc;

void main() {
    ivec2 local = ivec2(gl_LocalInvocationID.xy);
    ivec2 src_p = pc.block_origin + local;
    ivec2 ref_p = src_p + pc.candidate;

    float s = texelFetch(src_frame, src_p, 0).r;
    float r = texelFetch(ref_frame, ref_p, 0).r;
    partial_sad[local.y * 16 + local.x] = uint(abs(s - r) * 255.0);

    // Parallel tree reduction
    barrier();
    for (uint stride = 128; stride > 0; stride >>= 1) {
        uint idx = local.y * 16 + local.x;
        if (idx < stride)
            partial_sad[idx] += partial_sad[idx + stride];
        barrier();
    }
    if (local.x == 0 && local.y == 0)
        sad_values[pc.mb_index] = partial_sad[0];
}
```

Practical encoders use faster search patterns. **Three-step search** (TSS) starts with step size
`S = 4`, evaluates eight neighbours plus centre, moves to the best, halves step size, and repeats.
This reduces evaluations from O(R²) to O(log₂R). **Diamond search** uses a large diamond
(five points at distance 2) until no improvement, then a small diamond (four orthogonal points at
distance 1), matching the probability distribution of motion vectors in natural video. x264 uses
the Uneven Multi-Hexagon (UMH) search pattern: a fixed set of 5×1 and 1×5 rectangles before
switching to exhaustive near the best candidate. [Source: x264 source, `common/me.c`,
https://github.com/mirror/x264]

### Sub-Pixel Motion Estimation

Integer-pel motion vectors produce blocking artefacts at block boundaries. Half-pel and
quarter-pel refinement reconstruct fractional-position reference samples by filtering. H.264 uses
a **6-tap Wiener filter** `[-1, 5, 20, 20, 5, -1] / 32` for half-pel horizontal and vertical
positions, then bilinear averaging for quarter-pel positions. AV1 uses an 8-tap and a 6-tap
filter depending on the interpolation filter mode (`EIGHTTAP`, `EIGHTTAP_SMOOTH`, `BILINEAR`).
[Source: AV1 specification §7.11.3, https://aomediacodec.github.io/av1-spec/]

Sub-pixel interpolation can be run in a compute shader by pre-generating an expanded reference
frame at half-pel and quarter-pel positions (a "fractional reference pyramid"), then reading from
it during SAD computation.

### Bidirectional Prediction (B-frames)

B-frames use two reference frames: one in the past (forward reference L0) and one in the future
(backward reference L1). The encoder searches for an L0 motion vector, an L1 motion vector, and
can additionally merge them as a bi-prediction: `pred = (pred_L0 + pred_L1 + 1) / 2`. GPU
parallelisation dispatches L0 search and L1 search concurrently on separate compute queues (or
as separate dispatches), then a merge pass computes the bi-prediction SAD.

### Optical Flow for Screen-Content Coding

Screen-content coding (SCC) in HEVC and AV1 adds intra block copy (IBC), which is equivalent to
motion estimation *within the current frame* — matching identical regions such as a scrolled UI
element. Dense optical flow (e.g., Lucas-Kanade or Farneback variants) can feed IBC mode
decisions: a GPU sparse-to-dense flow field is computed at low resolution, and the resulting
integer displacements seed block-match verification at full resolution.

---

## 3. Intra Prediction

When no reference frame is available (I-frames or intra-refreshed regions), prediction is derived
from already-decoded neighbouring samples. The dependency on left and top neighbours creates a
**wavefront** constraint: block `(bx, by)` can only be processed after `(bx-1, by)` and
`(bx, by-1)` are complete. This forces GPU parallelism to operate along anti-diagonals rather than
row-by-row.

### H.264 Intra Modes

H.264 defines **nine 4×4 luma modes** (DC, planar, and seven directional modes) and **four 16×16
luma modes** (Vertical, Horizontal, DC, Plane). The plane mode extrapolates a linear gradient from
the top-left corner pixel and the rightmost top/bottom-most left samples:

```c
// H.264 16x16 Plane intra prediction (simplified)
int h = 0, v = 0;
for (int i = 1; i <= 8; i++) {
    h += i * (top[7+i] - top[7-i]);
    v += i * (left[7+i] - left[7-i]);
}
int a = 16 * (left[15] + top[15]);
int b = (5 * h + 32) >> 6;
int c = (5 * v + 32) >> 6;
for (int y = 0; y < 16; y++)
    for (int x = 0; x < 16; x++)
        pred[y][x] = clip(((a + b*(x-7) + c*(y-7) + 16) >> 5), 0, 255);
```

[Source: ITU-T H.264 specification §8.3.3, https://www.itu.int/rec/T-REC-H.264]

### HEVC/H.265 Intra Prediction

HEVC supports up to **35 intra modes** (DC, planar, and 33 angular modes) across CTU sizes of
4×4 to 64×64 (in extensions, up to 128×128). Angular modes specify an angle in units of
`intraAngle` steps; the implementation generates prediction samples by projecting along that
angle and interpolating between two reference samples. The angular reference array is first
filtered (using a bilinear or three-tap filter depending on the mode angle and block size) to
reduce high-frequency noise at block boundaries. [Source: ITU-T H.265 specification §8.4.4,
https://www.itu.int/rec/T-REC-H.265]

### AV1 Intra Prediction

AV1 defines **13 luma intra modes** (`INTRA_MODES = 13`), comprising DC, Paeth predictor, smooth
(vertical, horizontal, bi-directional), and **8 directional modes** (`DIRECTIONAL_MODES = 8`)
that can be steered by up to ±3 steps (`MAX_ANGLE_DELTA = 3`) of 3° each.
[Source: AV1 Specification constants, https://aomediacodec.github.io/av1-spec/]

The **Paeth predictor** picks from the left, above, or top-left sample whichever minimises the
absolute difference from `left + above - top_left`:

```c
// AV1 Paeth predictor for a single sample
static inline int paeth(int left, int above, int top_left) {
    int base  = left + above - top_left;
    int dl    = abs(base - left);
    int da    = abs(base - above);
    int dtl   = abs(base - top_left);
    if (dl <= da && dl <= dtl) return left;
    if (da <= dtl)              return above;
    return top_left;
}
```

AV1 also includes a **recursive intra mode** (`FILTER_INTRA_MODE`) that applies a learned 7-tap
filter to reconstruct samples from the already-predicted block interior rather than the raw
boundary, allowing smoother predictions across block boundaries.

### GPU-Parallel Intra Mode Decision

The encoder must evaluate all candidate modes and select the minimum-cost one. The cost metric
uses the **Hadamard transform** (a fast Walsh-Hadamard Transform on the residual) as a SATD
(Sum of Absolute Transformed Differences) metric — cheaper than a full DCT but better correlated
with bitrate than plain SAD. On GPU, one thread per mode computes the SATD concurrently. The
wavefront constraint limits concurrency to the number of blocks on the current anti-diagonal.

---

## 4. Transform Coding on GPU

After prediction, the residual (difference between current block and prediction) is transformed
to the frequency domain to decorrelate coefficients. Quantisation is then applied in the frequency
domain, where perceptual weighting is straightforward.

### DCT-II and Integer Approximations

H.264 uses a scaled integer **4×4 DCT-II** that avoids floating-point arithmetic. The transform
decomposes into butterfly stages that can be expressed entirely in 16-bit integer arithmetic with
shifts. HEVC adds 8×8, 16×16, and 32×32 transforms; AV1 extends to 4×4–64×64.

The 8-point DCT butterfly network:

```
Stage 0: s[0..7] inputs
Stage 1: butterfly pairs (s[0]+s[7], s[0]-s[7]), (s[1]+s[6], s[1]-s[6]), ...
Stage 2: further butterfly combinations with rotation constants
Stage 3: final butterfly, scale output by √(2/N)
```

On GPU, each butterfly stage maps to a single compute dispatch where each thread handles one
butterfly pair. For an 8×8 block, the 2D DCT separates into a row pass (eight 8-point 1D DCTs)
and a column pass (another eight 1D DCTs). **Warp shuffles** (`subgroupShuffleXor` in Vulkan /
GLSL) allow butterfly exchange between threads in the same warp without shared memory:

```glsl
// Vulkan/GLSL butterfly exchange using subgroup shuffles (warp size 32)
// Each thread holds one element of the 1D DCT
layout(local_size_x = 8, local_size_y = 8) in;

shared float row_temp[8][8];

float butterfly(float val, uint partner, float cos_factor) {
    float peer = subgroupShuffleXor(val, partner);
    return val * cos_factor + peer;
}
```

[Source: Vulkan GLSL subgroup operations, KHR_shader_subgroup specification,
https://github.com/KhronosGroup/GLSL/blob/master/extensions/khr/GL_KHR_shader_subgroup.txt]

### AV1 Transform Types

AV1 defines **16 transform types** (`TX_TYPES = 16`) formed by independent row and column
1D transforms chosen from four primitives: DCT, ADST (Asymmetric Discrete Sine Transform),
FLIPADST (time-reversed ADST), and IDTX (identity transform, bypassing the 1D transform in that
axis). [Source: AV1 Specification, transform type definitions, https://aomediacodec.github.io/av1-spec/]

Transform sizes span **5 square sizes** (`TX_SIZES = 5`: 4×4, 8×8, 16×16, 32×32, 64×64) and
**19 sizes including non-square** (`TX_SIZES_ALL = 19`, adding 4×8, 8×4, 8×16, 16×8, …, 32×64,
64×32). Non-square transforms exist because the encoder may partition rectangular CTU regions
into thin rectangular transform units.

### Inverse Transform for Decode (IDCT)

For decode, the inverse transform (IDCT) is applied identically in parallel per block. The
inverse butterfly network mirrors the forward one: each butterfly stage can be dispatched as a
Vulkan compute workgroup, with one workgroup per block in the frame. For a 1920×1080 frame with
16×16 transform units, there are 8,100 independent IDCT operations, all parallelisable without
any synchronisation between them.

---

## 5. Quantisation and Rate Control

### Uniform Scalar Quantisation with Dead Zone

Transform coefficients `C[i]` are quantised to indices `Q[i] = sign(C[i]) * max(0, (|C[i]| + dz/2) / step)`,
where `step` is the quantisation step size (derived from QP — Quantisation Parameter) and `dz`
is the dead-zone width, which is typically `step/2` for DC and `step/3` for AC coefficients.
This asymmetric dead zone produces a slight bias toward zero for small coefficients, improving
rate-distortion performance. [Source: H.264 specification §8.5.8; x264 `encoder/quant.c`]

### Rate-Distortion Optimised Quantisation (RDOQ)

RDOQ runs trellis optimisation over the quantised coefficients to minimise `D + λ·R` where `D`
is mean squared error between original and reconstructed coefficients and `R` is the estimated
bitrate. For each non-zero coefficient position, the trellis evaluates the cost of rounding up
vs. down vs. forcing to zero. On GPU, trellis paths for different coefficient sub-blocks can be
evaluated in parallel (one warp per 4×4 sub-block), though the Viterbi traceback for the optimal
path within a sub-block remains sequential.

### Rate Control Algorithms

Three modes dominate production pipelines:

- **CQP (Constant Quantisation Parameter)**: QP is fixed per frame type (I/P/B may have
  different QPs). No feedback loop; throughput is maximised and one GPU pass is sufficient.
- **CBR (Constant Bitrate)**: A feedback controller adjusts QP per frame based on the actual
  bits produced by the previous frame. The controller runs on CPU, but GPU assists through a
  *look-ahead pass*: encode a down-scaled version of upcoming frames at coarse QP to estimate
  scene complexity (sum of SAD across all blocks), then feed the complexity estimate into a
  proportional-integral controller.
- **VBR (Variable Bitrate)**: Similar to CBR but the instantaneous bitrate may vary within a
  buffer size constraint; average bitrate is the target. VBR requires two-pass encoding or a
  look-ahead window.

### GPU Look-Ahead Complexity Estimation

A single compute dispatch over a 1/4-resolution frame computes intra SAD for each 8×8 block,
accumulates the sums into a buffer, and the CPU reads back one `uint32_t` sum per scene unit.
This is orders of magnitude faster than full encode and gives the rate controller a reliable
signal for scene cuts and high-motion segments.

---

## 6. Deblocking and Loop Filters

Quantisation introduces ringing at block boundaries. In-loop filters remove blocking artefacts
before the reconstructed frame is stored as a reference for future frames — meaning the
*encoder* must run them too, matching decoder behaviour exactly.

### H.264 Deblocking Filter

The H.264 deblocking filter operates on 4×4 pixel edges, both horizontal (between block rows)
and vertical (between block columns). For each 4-pixel edge it computes a **boundary strength**
(BS) in `{0, 1, 2, 3, 4}`:

- BS=0: both blocks are inter-coded with the same motion vector and reference frame — no filtering
- BS=1–3: inter blocks with different motion or reference; filtered with a weak filter
- BS=4: intra block boundary; filtered with a strong filter that can modify up to four pixels
  on each side

The filter parameters `α` and `β` are derived from the average QP of the two blocks and the
filter offset parameters. GPU parallelism over edges: all vertical edges in a row can be filtered
simultaneously (they are non-overlapping); then all horizontal edges. This two-pass (vertical,
horizontal) approach ensures that each pixel is only read once per pass.

### HEVC Sample Adaptive Offset (SAO)

HEVC's SAO filter classifies each sample into a category and applies an offset to reduce
distortion. Two SAO modes exist:

- **Band Offset (BO)**: samples are classified into one of 32 intensity bands; offsets are
  additive per band. Useful for flat regions.
- **Edge Offset (EO)**: samples are classified based on the relative magnitude of two neighbours
  in one of four directions (horizontal, vertical, 135°, 45°). The edge categories are
  {both neighbours smaller, one smaller one larger, equal, one larger one smaller, both larger},
  each with its own signed offset.

GPU parallelism is straightforward for SAO: classification and offset addition for each pixel are
independent once the category parameters (stored in the bitstream) are known. One thread per
pixel computes `category = classify(pixel, neighbours); output = clamp(pixel + offset[category], 0, maxVal)`.

### AV1 Loop Filter

AV1's loop filter applies a deblocking filter similar to H.264's but uses four independent
strength values (`FRAME_LF_COUNT = 4` in the AV1 specification: luma horizontal, luma vertical,
Cb, Cr). Filter strength at each edge is derived from the loop filter delta values for each
segment and each reference frame. Maximum filter level `MAX_LOOP_FILTER = 63`.
[Source: AV1 Specification constants, https://aomediacodec.github.io/av1-spec/]

### AV1 CDEF (Constrained Directional Enhancement Filter)

CDEF is unique to AV1 and designed to remove ringing artefacts while preserving edges. It
operates in two stages:
1. **Direction detection**: for each 8×8 block, compute the direction (0–7, in 22.5° steps) that
   maximises the variance of sums projected along that direction. The best direction identifies
   the dominant edge orientation.
2. **Filtering**: apply a non-linear filter along and across the detected direction. The filter
   uses a secondary strength for the perpendicular direction to avoid smearing edges. The
   non-linearity is `clamp(d, -threshold, threshold)` applied to each `diff = sample - filtered_sum`.

GPU parallelism: direction detection is independent per 8×8 block, making it trivially parallel.
The filtering pass requires the detected direction from stage 1, so a barrier separates the two
stages, but both are embarrassingly parallel across blocks.

### AV1 Restoration Filter

AV1 adds a **restoration filter** applied after CDEF. Two restoration filter types are available:

- **Wiener filter**: a separable 7-tap filter with coefficients signalled in the bitstream and
  constrained to be symmetric. Applied per restoration unit (64×64 or 128×128 pixels).
- **Self-guided restoration**: a non-linear filter derived from a guidance image computed from
  the frame itself at two different scales, combined with a linear weighting.

The overlap-add tiling on GPU: restoration units are processed with 8-pixel overlap on each
border; the overlap region is blended to avoid seam artefacts. One workgroup per restoration unit
is sufficient for non-overlapping units; overlapping borders require a second pass with blending
weights.

---

## 7. Temporal Processing: Deinterlacing and Frame Interpolation

### Deinterlacing

Broadcast legacy content is often interlaced: each frame contains two fields (top/odd lines,
bottom/even lines) captured at half the vertical resolution at double the frame rate.
Deinterlacing reconstructs full-resolution progressive frames.

**Bob** (field duplication): each field is vertically scaled to full height by duplicating lines.
Zero computation; output suffers from visible combing on moving objects.

**Weave**: interleave the two fields directly. Correct for static scenes; produces combing
artefacts on motion.

**Motion-adaptive deinterlacing**: compute a motion mask per pixel (binary or soft), blend weave
output for static pixels and bob output for moving pixels. A simple motion mask:
`motion[y][x] = |current_field[y][x] - prev_field[y][x]| > threshold`. This is trivially
parallel over all pixels.

### YADIF — Yet Another Deinterlacing Filter

YADIF ([Source: FFmpeg `libavfilter/vf_yadif.c`,
https://github.com/FFmpeg/FFmpeg/blob/master/libavfilter/vf_yadif.c]) combines spatial and
temporal information. For each missing line (the one not present in the current field), it:

1. Computes a **spatial prediction** `sp = (above + below) / 2`, where `above` and `below` are
   the pixels one row up and down in the same field.
2. Evaluates a **spatial score** from 14 neighbouring pixel pairs along diagonal directions to
   select the best interpolation angle, analogous to intra-prediction mode selection.
3. Constrains the spatial prediction using **temporal differences** `temporal_diff0/1/2` computed
   from the previous and next frames, clamping `sp` to `[d - diff, d + diff]` where `d` is the
   directly corresponding reference pixel.

FFmpeg ships a CUDA-accelerated YADIF (`yadif_cuda`) that assigns one thread per output pixel
and parallelises both the spatial and temporal checks. The CUDA kernels are
`yadif_uchar` and `yadif_ushort` for 8-bit and 10/12-bit video respectively, with dual-channel
variants `yadif_uchar2`/`yadif_ushort2` for packed chroma. [Source: FFmpeg
`libavfilter/vf_yadif_cuda.cu`,
https://github.com/FFmpeg/FFmpeg/blob/master/libavfilter/vf_yadif_cuda.cu]

FFmpeg also ships a Vulkan-accelerated `bwdif_vulkan` (Better Weave Deinterlace with Interpolation
Filter) that processes previous, current, and next frames as three input bindings. The filter uses
precompiled SPIR-V, dispatches with workgroup dimensions `(1, 64, planes)` using an
`FFVkExecPool` on a compute queue obtained via `ff_vk_qf_find(VK_QUEUE_COMPUTE_BIT)`, and passes
per-frame parity and field state through push constants. [Source: FFmpeg
`libavfilter/vf_bwdif_vulkan.c`,
https://github.com/FFmpeg/FFmpeg/blob/master/libavfilter/vf_bwdif_vulkan.c]

### Frame Interpolation

Frame interpolation generates synthetic frames between two real frames, increasing apparent frame
rate or enabling slow-motion. The key challenge is handling occlusions: regions visible in the
target time but occluded in one or both reference frames.

A GPU frame interpolation pipeline:

1. **Optical flow**: compute forward flow `F_{t→t+1}` and backward flow `F_{t+1→t}` between
   the two reference frames using a CNN or classical Lucas-Kanade variant.
2. **Flow reversal**: estimate the intermediate flow as `F_{t→τ} ≈ -(1-τ)·F_{t→t+1}` and
   `F_{t+1→τ} ≈ τ·F_{t+1→t}` for target time `τ ∈ [0,1]`.
3. **Forward warp**: project pixels from each reference frame to the target time using the
   intermediate flows via scatter (each source pixel votes into its target position).
4. **Occlusion mask**: detect occlusions where forward and backward warped images disagree;
   weight contributions accordingly.
5. **Blend**: combine the two warped images: `output = mask * warp_0 + (1-mask) * warp_1`.

The **FILM** (Frame Interpolation for Large Motion) architecture
[Source: arXiv:2202.04901, https://arxiv.org/abs/2202.04901] uses a multi-scale feature pyramid
and a bidirectional flow estimator specifically trained for large displacements between frames far
apart in time. It is deployed as a TensorFlow SavedModel and can run on Linux via the TF ROCm
build or TF GPU build.

---

## 8. Video Super-Resolution

Video super-resolution (VSR) upscales low-resolution video to higher resolution while using
temporal information from multiple frames to recover detail that no single frame contains.

### Classical and CNN-based VSR

The first DNN approach to VSR (**SRCNN**, 2014) operated on single frames and is best treated as
a spatial post-processor. Multi-frame VSR requires alignment of neighbouring frames to the target
frame before fusion.

### BasicVSR++ Architecture

BasicVSR++ [Source: arXiv:2104.13371, https://arxiv.org/abs/2104.13371] improves upon the
original BasicVSR by introducing two mechanisms: **second-order grid propagation** (propagating
features from frames two steps away in addition to one step away) and **flow-guided deformable
alignment** (using optical flow to initialise deformable convolution offsets for sub-pixel-accurate
feature alignment).

The deformable alignment module computes a flow field `F` between a reference frame and the
target frame, then uses `F` as an initialiser for the offsets of a deformable convolutional
layer. The deformable convolution samples the feature map at the flow-displaced locations plus
learned residual offsets, giving the network freedom to correct small flow estimation errors.

Temporal propagation runs in two directions (forward and backward passes over the sequence),
accumulating aligned features from past and future frames at the target frame's position. The
fused features are then upscaled by a pixel-shuffle layer (sub-pixel convolution) with scale
factor 4.

### Deployment on Linux via ONNX Runtime

Production deployments export the trained model to ONNX and run it via ONNX Runtime, which
supports three GPU execution providers on Linux:

- **CUDA EP**: `CUDAExecutionProvider` using cuBLAS and cuDNN. Requires CUDA 11.6+ and
  corresponding libcudnn.
- **ROCm EP**: `ROCMExecutionProvider` for AMD GPUs via HIP. Available in ONNX Runtime ≥1.13
  built with ROCm 5.x support.
- **TensorRT EP**: `TensorrtExecutionProvider` for NVIDIA GPUs; JIT-compiles inference kernels
  optimised for the target GPU's capabilities.

Approximate call from Python to verify GPU path:

```python
import onnxruntime as ort

# List available providers
print(ort.get_available_providers())
# -> ['TensorrtExecutionProvider', 'CUDAExecutionProvider', 'CPUExecutionProvider']

sess = ort.InferenceSession(
    "basicvsr_plusplus_x4.onnx",
    providers=["CUDAExecutionProvider"],
    provider_options=[{"device_id": "0"}]
)
```

### Integration Points in GStreamer and FFmpeg

- **FFmpeg `scale_cuda`** with a super-resolution filter expression can be chained to an NVENC
  encode session, keeping frames in GPU memory throughout. `scale_cuda` uses a bicubic kernel by
  default; a custom CUDA super-resolution kernel can replace it in a filter graph.
- **GStreamer `vvas:superres`** (part of Xilinx/AMD Vitis Video Analytics SDK) wraps an
  ONNX or TensorFlow Lite super-resolution model into a GStreamer element, enabling zero-copy
  inference on AMD data centre GPUs. [Source: Vitis Video Analytics SDK,
  https://xilinx.github.io/video-sdk/] Note: verify element availability in your specific VVAS
  version, as naming has changed across releases.

---

## 9. HDR and Wide Color Gamut Video Processing

### HDR Metadata Extraction

HDR10 content embeds metadata in the bitstream via SEI (Supplemental Enhancement Information) NAL
units in H.264/HEVC or via OBU (Open Bitstream Unit) metadata in AV1. The key metadata types are:

- `SEI_TYPE_MASTERING_DISPLAY_COLOUR_VOLUME` (HEVC SEI 137): display primary chromaticities,
  white point, and max/min display luminance.
- `SEI_TYPE_CONTENT_LIGHT_LEVEL_INFO` (HEVC SEI 144): MaxCLL (Maximum Content Light Level,
  in nits) and MaxFALL (Maximum Frame-Average Light Level, in nits).

These are parsed by libavcodec and exposed in `AVFrame` side data of type
`AV_FRAME_DATA_MASTERING_DISPLAY_METADATA` and `AV_FRAME_DATA_CONTENT_LIGHT_LEVEL`.
[Source: FFmpeg `libavutil/hdr_dynamic_metadata.h`,
https://github.com/FFmpeg/FFmpeg/blob/master/libavutil/hdr_dynamic_metadata.h]

### EOTF Evaluation in Compute

HDR content uses non-linear transfer functions (EOTFs) to map code values to absolute luminance.

**PQ (Perceptual Quantiser, SMPTE ST 2084)**: used by HDR10 and Dolby Vision. Code value `v ∈
[0,1]` maps to linear luminance `L` (in cd/m²):

```glsl
// GLSL: PQ EOTF (SMPTE ST 2084)
const float m1 = 0.1593017578125;
const float m2 = 78.84375;
const float c1 = 0.8359375;
const float c2 = 18.8515625;
const float c3 = 18.6875;
const float L_max = 10000.0; // peak nits

float pq_eotf(float v) {
    float xp = pow(v, 1.0 / m2);
    float num = max(xp - c1, 0.0);
    float den = c2 - c3 * xp;
    return pow(num / den, 1.0 / m1) * L_max;
}
```

**HLG (Hybrid Log-Gamma, ITU-R BT.2100)**: code value maps differently for values above and
below 0.5:

```glsl
// GLSL: HLG EOTF (ITU-R BT.2100)
float hlg_eotf(float v) {
    const float a = 0.17883277;
    const float b = 0.28466892;
    const float c = 0.55991073;
    if (v <= 0.5)
        return (v * v) / 3.0;
    else
        return (exp((v - c) / a) + b) / 12.0;
}
```

[Source: ITU-R BT.2100, https://www.itu.int/rec/R-REC-BT.2100]

### Tone Mapping Operators

Tone mapping converts absolute scene luminance to display-referred values for a target SDR display.

**Reinhard (global)**: `L_out = L_in / (1 + L_in / L_white)` — simple, avoids clipping but
compresses highlights.

**Hable (Uncharted 2)**: a shoulder-toe curve that preserves contrast in shadows and midtones
while rolling off highlights smoothly. The curve parameters are typically:

```glsl
float hable(float x) {
    const float A = 0.15, B = 0.50, C = 0.10, D = 0.20, E = 0.02, F = 0.30;
    return ((x*(A*x+C*B)+D*E) / (x*(A*x+B)+D*F)) - E/F;
}
float hable_tonemap(float L, float exposure) {
    float scale = hable(11.2) / hable(exposure * L); // normalise to reference
    return hable(exposure * L) * scale;
}
```

**ACES (Academy Colour Encoding System)**: a filmic S-curve approximation using the RRT (Reference
Rendering Transform) + ODT (Output Display Transform). A common GLSL approximation:

```glsl
// ACES filmic tone mapping approximation
vec3 aces(vec3 x) {
    const float a = 2.51, b = 0.03, c = 2.43, d = 0.59, e = 0.14;
    return clamp((x*(a*x+b)) / (x*(c*x+d)+e), 0.0, 1.0);
}
```

**Dynamic tone mapping** uses a per-frame luminance histogram to parameterise the operator. A
compute shader accumulates a 1,024-bin histogram in shared memory atomics, then a prefix-sum pass
extracts the 1st-percentile and 99th-percentile luminance values to clip the operator's input
range to the actual frame content.

### Colour Space Conversion: BT.2020 → BT.709

Wide colour gamut (WCG) content encoded in BT.2020 primaries must be converted to BT.709 for
SDR displays. The conversion is a 3×3 linear transform applied in linear-light domain:

```glsl
// BT.2020 to BT.709 colour space conversion matrix
// (Applied after EOTF linearisation, before OETF re-encoding)
const mat3 bt2020_to_bt709 = mat3(
     1.6605,  -0.5876,  -0.0728,
    -0.1246,   1.1329,  -0.0083,
    -0.0182,  -0.1006,   1.1187
);

vec3 convert_gamut(vec3 rgb_2020) {
    return bt2020_to_bt709 * rgb_2020;
}
```

Note: the exact matrix coefficients depend on the source/destination primaries and white point
adaptation method (chromatic adaptation transform). The values above use the standard
chromaticity coordinates from ITU-R BT.2020 and BT.709 without explicit chromatic adaptation
[Source: ITU-R BT.2087 Recommendation, https://www.itu.int/rec/R-REC-BT.2087].

Dolby Vision RPU (Reference Processing Unit) metadata processing involves proprietary per-frame
and per-region tone-mapping curves. Note: needs verification on the specific GPU shader path for
RPU application; the open-source `dovi_tool` ([Source: https://github.com/quietvoid/dovi_tool])
handles RPU parsing on CPU but GPU-accelerated RPU application is vendor-specific.

### VA-API Tone Mapping

FFmpeg's `tonemap_vaapi` filter offloads HDR-to-SDR tone mapping to the VA-API fixed-function
path on Intel and AMD GPUs. It accepts a `tonemap` parameter selecting the algorithm (none,
linear, gamma, reinhard, hable, mobius) and uses `VA_PROC_PIPELINE_SUBPICTURES` under the hood
to set up the hardware post-processor. [Source: FFmpeg `libavfilter/vf_tonemap_vaapi.c`,
https://github.com/FFmpeg/FFmpeg/blob/master/libavfilter/vf_tonemap_vaapi.c]

---

## 10. Video Analysis Algorithms

### Scene Cut Detection

A scene cut produces a histogram difference spike. For each frame, compute a colour histogram
(e.g., 64 bins per channel) in a compute shader using `atomicAdd` into shared memory:

```glsl
layout(local_size_x = 16, local_size_y = 16) in;
layout(binding = 0) uniform sampler2D frame;
layout(binding = 1, std430) buffer Histogram { uint bins[192]; }; // 64*3 channels

shared uint local_hist[192];

void main() {
    // Zero shared histogram
    if (gl_LocalInvocationIndex < 192)
        local_hist[gl_LocalInvocationIndex] = 0;
    barrier();

    ivec2 p = ivec2(gl_GlobalInvocationID.xy);
    if (p.x < textureSize(frame, 0).x && p.y < textureSize(frame, 0).y) {
        vec3 rgb = texelFetch(frame, p, 0).rgb;
        atomicAdd(local_hist[uint(rgb.r * 63.0)],       1u);
        atomicAdd(local_hist[uint(rgb.g * 63.0) + 64u], 1u);
        atomicAdd(local_hist[uint(rgb.b * 63.0) + 128u],1u);
    }
    barrier();

    // Merge into global histogram
    if (gl_LocalInvocationIndex < 192)
        atomicAdd(bins[gl_LocalInvocationIndex],
                  local_hist[gl_LocalInvocationIndex]);
}
```

The histogram chi-squared distance or correlation between consecutive frames is computed in a
second small dispatch. A threshold (typically 0.25–0.4 in normalised distance) marks a hard cut.

**DCT-based scene change detection** computes the mean DC coefficient of each 8×8 block; a
sudden large change in the mean DC across the frame indicates a cut, and this metric is robust
to gradual dissolves which the histogram method misses.

### Motion Vector Magnitude Histogram

For encoded content, the decoded motion vectors from VA-API (`VAMotionVector` array attached to
each decoded frame) or from FFmpeg's `codecpar` side data can be read back to CPU cheaply (one
value per macroblock). The histogram of `|MV|` values characterises frame activity and is used
in thumbnail selection heuristics.

### Thumbnail Generation via Sharpness Metric

Best-frame selection for thumbnails uses the **Laplacian variance** as a sharpness metric:

```glsl
// Fragment/compute shader: Laplacian of each pixel
layout(local_size_x = 16, local_size_y = 16) in;
layout(binding = 0) uniform sampler2D frame;
layout(binding = 1, std430) buffer SharpOut { float sharpness; };

shared float local_sum[256];

void main() {
    ivec2 p = ivec2(gl_GlobalInvocationID.xy);
    float lap = abs(
        -4.0 * texelFetch(frame, p, 0).r
        + texelFetch(frame, p + ivec2(1,0), 0).r
        + texelFetch(frame, p - ivec2(1,0), 0).r
        + texelFetch(frame, p + ivec2(0,1), 0).r
        + texelFetch(frame, p - ivec2(0,1), 0).r
    );
    local_sum[gl_LocalInvocationIndex] = lap * lap;  // variance, not mean
    barrier();
    // tree reduction to sum
    for (uint s = 128; s > 0; s >>= 1) {
        if (gl_LocalInvocationIndex < s)
            local_sum[gl_LocalInvocationIndex] += local_sum[gl_LocalInvocationIndex + s];
        barrier();
    }
    if (gl_LocalInvocationIndex == 0)
        atomicAdd_f32(sharpness, local_sum[0]);  // Note: needs float atomics extension
}
```

A frame with higher Laplacian variance has more fine edge detail and less motion blur, making it
a better thumbnail candidate. `atomicAdd` for floating-point requires `GL_EXT_shader_atomic_float`
or equivalent.

### Video Deduplication via Perceptual Hashing

GPU-accelerated perceptual hashing (**pHash / DCT hash**):

1. Resize frame to 32×32 using bilinear downscale in a compute shader.
2. Convert to luma: `Y = 0.299R + 0.587G + 0.114B`.
3. Compute 32×32 DCT using a butterfly compute shader.
4. Take the top-left 8×8 DC coefficients (excluding position [0,0] which is the mean).
5. Compute the median of these 64 values; threshold each coefficient against the median to produce
   a 64-bit hash.

Hamming distance between hashes is computed as `popcount(hash_A XOR hash_B)` — a single GPU
instruction. A distance of ≤ 10 bits is typically considered a near-duplicate.

---

## 11. 360° Video and Immersive Formats

### Equirectangular Projection

360° video is typically stored in equirectangular projection (ERP): latitude maps to vertical
pixel position, longitude to horizontal. This introduces severe non-uniform sampling density: the
poles are over-sampled (stretched pixels) while the equator is sampled at the correct density.
The effective resolution at latitude `φ` is reduced by `cos(φ)` compared to equatorial.

### Equi-Angular Cubemap (EAC)

EAC projects the sphere onto six cube faces but distributes samples proportional to the solid
angle they cover rather than linearly in face texture coordinates. YouTube's 360° streaming
system uses EAC as its primary projection format. [Source: YouTube 360° video mapping,
https://blog.youtube/news-and-events/bringing-pixels-front-and-center-in-vr-video/]

Converting from ERP to EAC in a compute shader requires evaluating `atan()` per texel to remap
coordinates — a transcendental-function-heavy operation well suited to GPU.

### Viewport-Dependent Streaming and Tile Selection

In tile-based DASH for 360° video, the sphere is partitioned into N×M tiles (e.g., 6×4 = 24
tiles). Only tiles within the viewer's viewport need to be decoded at high quality. A compute
shader implements GPU frustum intersection: given a view quaternion and FOV, it computes the four
corner rays of the viewport frustum, projects them onto the sphere, and determines which EAC face
tiles they intersect. This intersection test is `O(N_tiles)` per frame and trivially parallel
across tiles.

The output is a tile priority vector used by the adaptive bitrate (ABR) client to request tiles
at appropriate quality levels before the playback position reaches them.

### Stereoscopic Layout Handling

Stereoscopic video encodes left and right eye views as:
- **Side-by-side (SbS)**: left eye occupies `x ∈ [0, W/2)`, right eye `x ∈ [W/2, W)`.
- **Top-bottom (TaB)**: left eye occupies `y ∈ [0, H/2)`, right eye `y ∈ [H/2, H)`.

A Vulkan `VkImageViewUsageCreateInfo` with a split viewport or a texture atlas lookup in the
fragment shader handles the remapping. For VR rendering, the two eye views are decoded from the
same surface and submitted to the compositor as separate texture layers.

### Projection-Aware Motion Estimation for 360° Encoding

Standard block-matching ME in ERP space fails near the poles because matching blocks that are
spatially close on the sphere may be far apart in ERP coordinates (wrap-around at longitude ±180°)
or geometrically distorted (equatorial vs. polar sampling density mismatch). Projection-aware ME
applies a weight to the SAD metric inversely proportional to `cos(φ)` (the pole-weight factor)
and enables search range wrap-around at the horizontal boundaries. This is implemented in the
AOM libaom encoder's 360° mode and in the JVET reference encoder for HEVC 360Lib.
[Source: JVET 360Lib, https://jvet.hhi.fraunhofer.de/]

---

## 12. GStreamer and FFmpeg GPU Pipeline Integration

### GStreamer VA-API Pipeline

A complete GStreamer pipeline decoding H.264, applying colour conversion, and encoding to HEVC
using VA-API, keeping frames in GPU-accessible VA surfaces throughout:

```bash
gst-launch-1.0 \
  filesrc location=input.mp4 ! qtdemux ! h264parse ! \
  vaapih264dec ! \
  vaapipostproc format=nv12 ! \
  vaapih265enc rate-control=cbr bitrate=4000 ! \
  h265parse ! mp4mux ! \
  filesink location=output.mp4
```

The `vaapih264dec` element produces frames in `VAMemoryType` (VA surfaces). `vaapipostproc`
accepts VA surfaces, runs colour conversion or scaling in the fixed-function VA post-processor,
and passes VA surfaces to `vaapih265enc` — zero CPU copies between decode, filter, and encode.

For more complex graphs, `cudaconvert` (on NVIDIA) or custom `vkimport` elements accept DMA-BUF
file descriptors exported from VA surfaces. The VA-API export path is:
`vaExportSurfaceHandle(dpy, surface, VA_SURFACE_ATTRIB_MEM_TYPE_DRM_PRIME_2, VA_EXPORT_SURFACE_READ_ONLY, &desc)`,
which returns a `VADRMPRIMESurfaceDescriptor` containing DMA-BUF `fd` values per plane.
A Vulkan `VkImage` is created from that DMA-BUF using `VK_EXT_external_memory_dma_buf` and
`VK_EXT_image_drm_format_modifier`. [Source: VA-API `va_drmcommon.h`,
https://github.com/intel/libva/blob/master/va/va_drmcommon.h]

GStreamer NVENC elements (`nvh264enc`, `nvh265enc`, `nvav1enc`) connect to the GStreamer CUDA
memory allocator; the `cudaconvert` element handles format conversion between NV12 and CUDA
surfaces:

```bash
gst-launch-1.0 \
  v4l2src ! video/x-raw,format=YUY2 ! \
  cudaupload ! cudaconvert ! video/x-raw(memory:CUDAMemory),format=NV12 ! \
  nvh264enc bitrate=8000 rc-mode=cbr ! \
  h264parse ! rtph264pay ! udpsink host=192.168.1.2 port=5004
```

### FFmpeg CUDA Hwaccel Filter Graph

FFmpeg's CUDA hardware acceleration keeps frames in CUDA device memory across filter operations.
Frames are uploaded to GPU with `hwupload_cuda`, processed through CUDA filters, and downloaded
for CPU-side muxing with `hwdownload`:

```bash
ffmpeg -i input.mp4 \
  -vf "hwupload_cuda,scale_cuda=1920:1080,hwdownload,format=yuv420p" \
  -c:v h264_nvenc -b:v 4M output.mp4
```

`scale_cuda` uses a CUDA bicubic or bilinear kernel for resizing. The filter graph context
manages a CUDA stream shared across all filter stages, avoiding host-device synchronisation
between filters. Adding `yadif_cuda` for deinterlacing:

```bash
ffmpeg -i interlaced.ts \
  -vf "hwupload_cuda,yadif_cuda=mode=0,scale_cuda=1280:720,hwdownload,format=yuv420p" \
  -c:v h264_nvenc output.mp4
```

### FFmpeg Vulkan Filter Graph

FFmpeg's Vulkan filter graph (`-vf hwupload,scale_vulkan`) creates an `AVHWFramesContext` backed
by Vulkan image memory and passes `AVVkFrame` objects between filters.

`AVVkFrame` fields relevant to filter authors:
- `img[AV_NUM_DATA_POINTERS]`: `VkImage` handles for each plane
- `tiling`: image tiling (`VK_IMAGE_TILING_OPTIMAL` or `VK_IMAGE_TILING_LINEAR`)
- `mem[AV_NUM_DATA_POINTERS]`: `VkDeviceMemory` backing each image
- `layout[AV_NUM_DATA_POINTERS]`: current `VkImageLayout` for each image
- `sem[AV_NUM_DATA_POINTERS]`: `VkSemaphore` (timeline) for synchronisation
- `sem_value[AV_NUM_DATA_POINTERS]`: timeline semaphore signal/wait value
- `access[AV_NUM_DATA_POINTERS]`: `VkAccessFlags` for barrier specification
- `queue_family[AV_NUM_DATA_POINTERS]`: owning queue family per image

[Source: FFmpeg `libavutil/hwcontext_vulkan.h`,
https://github.com/FFmpeg/FFmpeg/blob/master/libavutil/hwcontext_vulkan.h]

Available Vulkan filters in FFmpeg's `libavfilter`:
`avgblur_vulkan`, `blend_vulkan`, `bwdif_vulkan`, `chromaber_vulkan`, `color_vulkan`,
`gblur_vulkan`, `hflip_vulkan`, `vflip_vulkan`, `flip_vulkan`, `nlmeans_vulkan`,
`overlay_vulkan`, `transpose_vulkan`.
[Source: FFmpeg filter documentation, https://ffmpeg.org/ffmpeg-filters.html]

A complete Vulkan filter graph example for HDR tone mapping and resize:

```bash
ffmpeg -init_hw_device vulkan=vk:0 -filter_hw_device vk \
  -i hdr10_input.mkv \
  -vf "hwupload,scale_vulkan=1280:720,hwdownload" \
  -c:v libx265 -x265-params "hdr10=1:max-cll=1000,400" \
  output.mkv
```

VA-API Vulkan interop (the emerging zero-copy path from VA-API decode to Vulkan filter) uses
`VK_EXT_external_memory_dma_buf` to import VA surface DMA-BUFs directly into Vulkan images,
bypassing any CPU copy. Mesa's RADV and ANV drivers both support this extension; the FFmpeg
Vulkan hwaccel path performs the import automatically when both `vaapi` and `vulkan` hwaccel
contexts share the same DRM device node.

### VA-API Filters

FFmpeg's VA-API filter set:
`overlay_vaapi`, `tonemap_vaapi`, `hstack_vaapi`, `vstack_vaapi`, `xstack_vaapi`, `pad_vaapi`,
`drawbox_vaapi`. [Source: FFmpeg `libavfilter`, https://ffmpeg.org/ffmpeg-filters.html]

These filters run on the GPU's fixed-function video post-processor (VDPAU, Intel EU shader, or
AMD VPE), and are accessed via `vaCreateContext` with `VA_PROC_PIPELINE_SUBPICTURES` attributes.

---

## Integrations

This chapter sits within Part XXIX (Graphics Algorithms) and is read alongside the following
chapters from the rest of the book:

**Chapter 26 (Hardware Video)** covers the VA-API and V4L2 API layer — the system calls,
`VADisplay`, `VASurface`, and buffer lifecycle that this chapter assumes. The DMA-BUF export
mechanism described in §12 relies on `vaExportSurfaceHandle` documented in Ch26.

**Chapter 165 / Chapter 50 (Vulkan Video Extensions)** covers the `VK_KHR_video_decode_*` and
`VK_KHR_video_encode_*` extension structures, session objects, and query pools. The transform,
quantisation, and in-loop filter algorithms in §4–6 are what Vulkan Video's `vkCmdDecodeVideoKHR`
triggers inside the video decode block; §12's Vulkan filter graph interoperates with the decoded
`AVVkFrame`.

**Chapter 207 (Shader VFX and Post-Processing)** covers general post-processing algorithms —
TAA, DoF, motion blur, bloom — that are commonly chained after GPU video decode in a display
pipeline. The colour grading and 3D LUT entries in Ch207 complement the HDR tone-mapping
operators in §9 of this chapter.

**Chapter 25 (GPU Compute)** covers the Vulkan compute pipeline, `VkComputePipeline`, descriptor
sets, and push constants that are the underlying mechanism for every compute shader shown in this
chapter. Readers unfamiliar with compute dispatch should start there.

**Chapter 38 (PipeWire and the Video Session Layer)** covers how decoded GPU video surfaces are
shared with the compositor via DMA-BUF, a mechanism this chapter relies on in §12 for the
zero-copy GStreamer pipeline. The `wp_linux_drm_syncobj` protocol for GPU timeline semaphore
propagation is the synchronisation primitive that AVVkFrame's `sem`/`sem_value` fields ultimately
map onto when a frame reaches the compositor.

**Chapter 114 (OpenCV and GPU-Accelerated Computer Vision)** covers GPU-accelerated feature
extraction, histogram computation, and optical flow implementations in OpenCV's CUDA backend —
algorithms that overlap with §10 (video analysis) and §7 (frame interpolation) in this chapter.
The `cv::cuda::calcOpticalFlowFarneback` function is a practical alternative to writing a custom
compute shader for the optical flow stage of frame interpolation.

**Chapter 212 (Neural Geometry — Specialized Techniques and GPU Primitives)** covers GPU radix
sort and other parallel primitives (§XIII in that chapter) that underpin the prefix-sum histogram
operations in §9 and §10 of this chapter, and discusses ONNX Runtime GPU deployment patterns
relevant to §8's super-resolution inference.
