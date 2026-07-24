# Chapter 220: GPU Image Processing Algorithms

*Part XXIX — Graphics Algorithms*

**Audiences:** Systems developers working with libcamera, V4L2, and GStreamer image pipelines who need to understand how ISP algorithms execute on GPU or DSP hardware; browser and web platform engineers using Canvas 2D, WebCodecs, and WebGPU for image operations; graphics application developers using OpenCV GPU backends, Halide GPU targets, or writing Vulkan compute shaders for image processing.

This chapter surveys the principal classes of GPU image processing algorithms — spatial filtering, morphological operations, edge detection, frequency-domain methods, optical flow, stereo matching, super-resolution, color processing, and ISP pipelines — and grounds each in real Linux-stack implementation paths. The chapter closes with a comparative look at the OpenCV GPU backend, Halide's GPU code-generation pipeline, and VkComputePipeline as the low-level Vulkan alternative.

---

## Table of Contents

1. [GPU Image Processing Architecture](#1-gpu-image-processing-architecture)
2. [Spatial Filtering](#2-spatial-filtering)
3. [Morphological Operations](#3-morphological-operations)
4. [Edge and Feature Detection](#4-edge-and-feature-detection)
5. [Frequency-Domain Processing](#5-frequency-domain-processing)
6. [Optical Flow](#6-optical-flow)
7. [Stereo Matching and Depth](#7-stereo-matching-and-depth)
8. [Super-Resolution](#8-super-resolution)
9. [Color Processing and Histograms](#9-color-processing-and-histograms)
10. [ISP Pipeline on Linux](#10-isp-pipeline-on-linux)
11. [OpenCV and Halide GPU Backends on Linux](#11-opencv-and-halide-gpu-backends-on-linux)
12. [Integrations](#12-integrations)

---

## 1. GPU Image Processing Architecture

### 1.1 Why Images Map Naturally to GPU Compute

Image processing is overwhelmingly *embarrassingly parallel*: the output value at pixel (x, y) depends either solely on the input at (x, y) — a pointwise operation such as gamma correction or color space conversion — or on a bounded neighborhood around (x, y) — a convolution, morphological operation, or histogram window. Because these dependencies are spatially local and uniform, a single compute shader can process millions of pixels concurrently with no inter-thread communication beyond what shared memory explicitly provides.

Beyond raw parallelism, GPU architectures expose two hardware paths that image processing exploits directly:

**Texture samplers.** The GPU's fixed-function texture unit provides bilinear or bicubic interpolation, clamping or wrapping address modes, and automatic mip-level selection with a single instruction — operations that a compute shader would need many ALU cycles to replicate. For read-only, interpolated access (scaling, rotation, Lanczos upsampling) texture fetch is almost always faster than manual implementation.

**Image load/store.** Vulkan's `VK_IMAGE_USAGE_STORAGE_BIT` flag enables a shader to read and write image texels as if they were a typed array (`imageLoad` / `imageStore`). Unlike a sampler, image load/store skips the interpolation hardware, gives integer-coordinate access, and allows write-back to the same image in a single pass. This is the correct path for compute-intensive algorithms (bilateral filter, NLM, FFT butterfly stages) where the shader computes the interpolation itself or needs random write access.

### 1.2 VkImage vs. Buffer for 2D Data

A Vulkan compute shader can hold a 2D image in either a `VkImage` (with `VkImageView` bound as a storage image or combined image sampler) or a plain `VkBuffer` (bound as an SSBO). The choice matters for performance:

- **VkImage** stores texels in an implementation-defined *tiled* layout (Z-order, swizzled, or vendor-specific) that improves spatial locality on cache lines whose shape more nearly matches 2D neighborhoods than a row-major flat buffer. On most discrete GPUs this gives a meaningful speedup for stencil-access patterns (convolutions, morphological ops, bilateral).
- **VkBuffer** (linear row-major) is simpler to construct from CPU-mapped data, integrates cleanly with `V4L2_MEMORY_DMABUF` or DMA-BUF shared buffers, and is required when another device (camera ISP, video codec, display controller) has written the data without producing a tiled image.

For pure GPU-to-GPU pipelines, prefer `VkImage` with `VK_IMAGE_TILING_OPTIMAL`. For camera or codec-sourced frames arriving via DMA-BUF, import the buffer into a `VkBuffer` or use `VkImageCreateInfo.tiling = VK_IMAGE_TILING_LINEAR` with the external memory extension.

### 1.3 Dispatch Sizing: One Thread Per Pixel vs. Tile-Based

The two canonical dispatch strategies are:

**One thread per output pixel.** `gl_GlobalInvocationID.xy` directly indexes the output pixel. No synchronization is needed. Works well for pointwise operations and when the per-pixel work is large enough to amortize the thread launch overhead.

**Tile-based with shared memory.** A tile of threads (commonly 8×8 or 16×16) cooperates to load a padded block of input pixels into `shared` memory, then each thread accesses its neighborhood from shared memory rather than performing repeated global memory reads. This is the standard pattern for convolutions where the kernel radius is significant.

```glsl
// tile_gaussian.comp — tile-based 2D Gaussian with shared memory
layout(local_size_x = 16, local_size_y = 16) in;
layout(set=0, binding=0, rgba8) uniform readonly  image2D src;
layout(set=0, binding=1, rgba8) uniform writeonly image2D dst;
layout(push_constant) uniform PC {
    int   radius;      // kernel half-width (e.g. 3 for 7x7)
    float sigma;
} pc;

shared vec4 tile[16 + 2*3][16 + 2*3];   // padded for radius=3

void main() {
    ivec2 gid   = ivec2(gl_GlobalInvocationID.xy);
    ivec2 lid   = ivec2(gl_LocalInvocationID.xy);
    ivec2 base  = ivec2(gl_WorkGroupID.xy) * ivec2(16,16) - ivec2(pc.radius);
    ivec2 sz    = imageSize(src);

    // cooperative load into shared memory
    for (int dy = int(lid.y); dy < 16 + 2*pc.radius; dy += 16)
        for (int dx = int(lid.x); dx < 16 + 2*pc.radius; dx += 16)
            tile[dy][dx] = imageLoad(src, clamp(base + ivec2(dx,dy), ivec2(0), sz-1));
    barrier();

    // Gaussian accumulation from shared memory
    vec4  acc = vec4(0.0);
    float wt  = 0.0;
    for (int ky = -pc.radius; ky <= pc.radius; ky++) {
        for (int kx = -pc.radius; kx <= pc.radius; kx++) {
            float g  = exp(-(kx*kx + ky*ky) / (2.0*pc.sigma*pc.sigma));
            acc     += g * tile[lid.y + ky + pc.radius][lid.x + kx + pc.radius];
            wt      += g;
        }
    }
    imageStore(dst, gid, acc / wt);
}
```

For a separable kernel (Gaussian, box, Lanczos) the two-pass (horizontal then vertical) approach reduces the per-output-pixel work from O(k²) to O(2k) and is strongly preferred for large radii.

---

## 2. Spatial Filtering

### 2.1 Separable Gaussian Blur

A 2D isotropic Gaussian G(x,y;σ) = G(x;σ)·G(y;σ) separates into two 1D kernels. The standard GPU implementation is two compute passes:

1. **Horizontal pass:** reads `src`, writes `tmp`, one dispatch per row-group.
2. **Vertical pass:** reads `tmp`, writes `dst`, one dispatch per column-group.

The 1D kernel weights fit in push constants for σ ≤ 3 (radius ≤ 9, 19 floats). For larger σ, store the kernel in a 1D storage buffer and fetch with a loop.

```glsl
// gauss_horiz.comp — horizontal pass
layout(local_size_x = 256, local_size_y = 1) in;
layout(set=0, binding=0, rgba16f) uniform readonly  image2D src;
layout(set=0, binding=1, rgba16f) uniform writeonly image2D dst;
layout(push_constant) uniform PC { float kernel[19]; int radius; } pc;

void main() {
    ivec2 p  = ivec2(gl_GlobalInvocationID.xy);
    vec4  acc = vec4(0.0);
    for (int k = -pc.radius; k <= pc.radius; k++)
        acc += pc.kernel[k + pc.radius] *
               imageLoad(src, clamp(p + ivec2(k,0), ivec2(0), imageSize(src)-1));
    imageStore(dst, p, acc);
}
```

### 2.2 Box Filter via Summed-Area Table

A box filter of any radius can be computed in O(1) per pixel once a *summed-area table* (SAT, also called integral image) is precomputed. The SAT can be built in O(N) with two prefix-sum passes (horizontal then vertical), each needing a scan across one dimension.

On GPU, prefix scans are performed using parallel reduction with shared memory — the standard Kogge-Stone or Blelloch scan pattern. Once the SAT is in a storage image, a box filter of width 2r+1 is a four-corner lookup:

```
box_avg(x,y,r) = [SAT(x+r,y+r) - SAT(x-r-1,y+r) - SAT(x+r,y-r-1) + SAT(x-r-1,y-r-1)] / (2r+1)²
```

The SAT approach is the backbone of the *guided filter* (§2.4) and per-tile CLAHE histograms (§9.3).

### 2.3 Bilateral Filter

The bilateral filter preserves edges by combining spatial proximity and intensity similarity:

```
BF[I](p) = (1/W_p) Σ_q G_s(||p-q||) · G_r(||I(p)-I(q)||) · I(q)
```

where G_s is the spatial Gaussian (std σ_s) and G_r the range Gaussian (std σ_r). Each output pixel requires O(k²) reads of global memory. For a 5×5 window (radius 2) on a 4K image this is ≈33 million texture fetches — manageable with good caching. For large radii (radius ≥ 8) the cost is prohibitive and approximate bilateral methods (domain transform, guided filter) are preferred.

The per-pixel cost structure means a tile-based shared-memory approach (§1.3) is beneficial even for moderate window sizes.

### 2.4 Guided Filter

The guided filter approximates the bilateral filter with O(N) complexity by expressing the output as a locally linear function of a guidance image G:

```
q_i = a_k · G_i + b_k,  ∀i ∈ ω_k
```

where ω_k is a square window of radius r and the coefficients are solved in closed form using local means and variances. The algorithm reduces to five box-filter passes (mean_G, mean_I, corr_GI, corr_GG, then the output linear combination), all computable with the SAT approach. The guided filter has become a standard building block in ISP denoising, depth-map refinement, and alpha matting.

### 2.5 Non-Local Means

NLM computes the denoised value at pixel p as a weighted average over all pixels q whose *patches* P(q) are similar to P(p):

```
NLM[I](p) = (1/Z_p) Σ_q exp(-||P(p)-P(q)||² / h²) · I(q)
```

The naïve O(N²·k²) cost is reduced to O(N·S·k²) by restricting the search to a window of radius S around p. Each thread loads a (2S+1)×(2S+1) search neighborhood and a (2k+1)×(2k+1) patch from shared memory. For 7×7 patches and a 21×21 search window, this requires 49 + 441 = 490 shared floats per channel per thread, fitting comfortably within the shared-memory budget of modern GPUs.

---

## 3. Morphological Operations

### 3.1 Erosion and Dilation

Binary morphology operates on sets of foreground pixels. Given a structuring element B, erosion of image I at pixel p is the logical AND of I over the B-translated neighborhood; dilation is the OR. For *grey-level morphology*, AND/OR become min/max:

```
Erosion:   (I ⊖ B)(p) = min_{b ∈ B} I(p + b)
Dilation:  (I ⊕ B)(p) = max_{b ∈ B} I(p + b)
```

Both are trivially implementable as compute shaders. For a disc structuring element of radius r, the kernel iterates over the 2D neighborhood and accumulates the min or max:

```glsl
// erode.comp
layout(local_size_x = 16, local_size_y = 16) in;
layout(set=0, binding=0, r8)  uniform readonly  image2D src;
layout(set=0, binding=1, r8)  uniform writeonly image2D dst;
layout(push_constant) uniform PC { int radius; } pc;

void main() {
    ivec2 p   = ivec2(gl_GlobalInvocationID.xy);
    float val = 1.0;
    for (int ky = -pc.radius; ky <= pc.radius; ky++) {
        for (int kx = -pc.radius; kx <= pc.radius; kx++) {
            if (kx*kx + ky*ky <= pc.radius*pc.radius)   // disc structuring element
                val = min(val, imageLoad(src, p + ivec2(kx,ky)).r);
        }
    }
    imageStore(dst, p, vec4(val));
}
```

### 3.2 Opening, Closing, and Non-Flat Structuring Elements

Morphological opening (erosion followed by dilation) removes small bright objects while preserving large structures. Closing (dilation then erosion) fills small holes. Both are implemented as two sequential compute passes.

**Non-flat structuring elements** assign a grey-level weight h(b) to each element position b. Erosion becomes:

```
(I ⊖ B)(p) = min_{b ∈ B} [I(p+b) - h(b)]
```

This models operations such as top-hat transforms and grey-level gradient estimation. The GPU implementation adds a single subtraction inside the inner loop.

**Separable approximations.** For large rectangular structuring elements, the 2D morphological op separates into 1D horizontal then 1D vertical passes, each computable with O(N) running-minimum/maximum using a sliding-window deque. This is the correct approach for rectangular elements with radius > 8; for disc elements, the direct GPU implementation typically wins up to radius ≈ 20 due to shared-memory efficiency.

---

## 4. Edge and Feature Detection

### 4.1 Sobel/Scharr Gradient

The Sobel operator approximates image gradients via 3×3 convolution:

```
Gx = [-1 0 +1; -2 0 +2; -1 0 +1],  Gy = Gxᵀ
```

The Scharr operator uses slightly different weights `[-3 0 +3; -10 0 +10; -3 0 +3]` and `[-3 -10 -3; 0 0 0; +3 +10 +3]` for better rotational symmetry. Both fit in push constants. The gradient magnitude is `||G|| = sqrt(Gx²+Gy²)` and angle `θ = atan(Gy, Gx)`.

```glsl
// sobel.comp
layout(local_size_x = 16, local_size_y = 16) in;
layout(set=0, binding=0, r8)    uniform readonly  image2D src;
layout(set=0, binding=1, rg16f) uniform writeonly image2D dst;  // Gx, Gy

float px(int dx, int dy) {
    return imageLoad(src, ivec2(gl_GlobalInvocationID.xy) + ivec2(dx,dy)).r;
}
void main() {
    float gx = -px(-1,-1) + px(1,-1) - 2.0*px(-1,0) + 2.0*px(1,0)
               -px(-1,1)  + px(1,1);
    float gy = -px(-1,-1) - 2.0*px(0,-1) - px(1,-1)
               +px(-1, 1) + 2.0*px(0,1)  + px(1,1);
    imageStore(dst, ivec2(gl_GlobalInvocationID.xy), vec4(gx, gy, 0, 0));
}
```

### 4.2 Canny Edge Detection

Canny is a three-pass pipeline:

**Pass 1 — Gradient computation.** Sobel or Scharr (§4.1), producing magnitude and angle images.

**Pass 2 — Non-maximum suppression (NMS).** For each pixel, quantize the gradient direction θ to one of four canonical directions (0°, 45°, 90°, 135°). Suppress the pixel if its magnitude is not a local maximum along that direction:

```glsl
// nms.comp — non-maximum suppression
layout(local_size_x = 16, local_size_y = 16) in;
layout(set=0, binding=0, rg16f) uniform readonly  image2D grad;  // Gx, Gy
layout(set=0, binding=1, r16f)  uniform writeonly image2D nms;

void main() {
    ivec2 p    = ivec2(gl_GlobalInvocationID.xy);
    vec2  g    = imageLoad(grad, p).xy;
    float mag  = length(g);
    float ang  = atan(g.y, g.x);  // [-pi, pi]

    // quantise angle to 4 directions
    float a    = mod(ang + 3.14159, 3.14159);  // [0, pi)
    ivec2 d    = (a < 0.3927 || a >= 2.7489) ? ivec2(1,0) :
                 (a < 1.1781)                ? ivec2(1,1) :
                 (a < 1.9635)                ? ivec2(0,1) : ivec2(-1,1);

    float m1   = length(imageLoad(grad, p +  d).xy);   // neighbour magnitudes
    float m2   = length(imageLoad(grad, p + -d).xy);
    imageStore(nms, p, vec4(mag > m1 && mag > m2 ? mag : 0.0));
}
```

**Pass 3 — Hysteresis thresholding.** Two thresholds `tLow`, `tHigh` define strong edges (mag > tHigh), weak edges (tLow < mag ≤ tHigh), and non-edges (mag ≤ tLow). A connected-component pass keeps weak edges only if they are 8-connected to a strong edge. On GPU this is typically implemented as an iterative flood-fill or parallel BFS over the NMS image, converging after O(diameter) iterations.

### 4.3 Harris Corner Score

The Harris corner detector computes the second-moment matrix M at each pixel using smoothed products of Sobel gradients, then scores it:

```
M = [Ix²_smooth  IxIy_smooth]
    [IxIy_smooth Iy²_smooth ]

R = det(M) - k · trace(M)²   (k ≈ 0.04–0.06)
```

All three components (Ix², IxIy, Iy²) are computed per-pixel in one pass and then Gaussian-smoothed over a window (typically 5×5) in a second pass. The Harris score R is computed in a third pass. Strong corners have large positive R; edges have large negative R.

### 4.4 FAST Corner Detector

FAST (Features from Accelerated Segment Test) tests 16 pixels on a Bresenham circle of radius 3 around each candidate pixel p. A corner is detected if N consecutive pixels (typically N=12) are all brighter than `I(p)+t` or all darker than `I(p)-t`. On GPU each pixel independently tests all 16 circle positions in parallel, with no data dependency between pixels:

```glsl
// fast_corners.comp — simplified FAST-12
const ivec2 circle[16] = {
    ivec2( 0,-3), ivec2( 1,-3), ivec2( 2,-2), ivec2( 3,-1),
    ivec2( 3, 0), ivec2( 3, 1), ivec2( 2, 2), ivec2( 1, 3),
    ivec2( 0, 3), ivec2(-1, 3), ivec2(-2, 2), ivec2(-3, 1),
    ivec2(-3, 0), ivec2(-3,-1), ivec2(-2,-2), ivec2(-1,-3)
};
// ... test 16 positions, count consecutive bright/dark pixels
```

The FAST test is the basis of ORB and BRISK feature detectors used in visual odometry pipelines.

---

## 5. Frequency-Domain Processing

### 5.1 Cooley-Tukey Radix-2 DFT in Compute

A 1D DFT of size N = 2^p decomposes into log₂(N) butterfly stages. Each stage operates on pairs of complex values separated by a fixed stride and multiplies by twiddle factors ω = e^(-2πij/N). In a compute shader, one thread group processes a segment of the array stored in shared memory, performing all butterfly stages in-place with barrier synchronisation between stages:

```glsl
// fft_radix2.comp — single-workgroup 1D FFT for N=1024
layout(local_size_x = 512) in;   // N/2 threads, each handles 2 elements
layout(set=0, binding=0) buffer FFTBuf { vec2 data[]; };  // complex as (re,im)

shared vec2 smem[1024];

vec2 cmul(vec2 a, vec2 b) { return vec2(a.x*b.x-a.y*b.y, a.x*b.y+a.y*b.x); }

void main() {
    uint tid = gl_LocalInvocationID.x;
    uint N   = 1024u;

    // bit-reverse permutation load
    uint rev = bitfieldReverse(tid) >> (32u - 10u);   // 10 = log2(1024)
    smem[tid]       = data[rev];
    smem[tid + 512] = data[rev + 512];
    barrier();

    for (uint s = 1u; s <= 10u; s++) {
        uint m   = 1u << s;
        uint m2  = m >> 1u;
        float ang = -6.283185307 / float(m);
        uint k   = tid % m2;
        uint j   = (tid / m2) * m + k;

        vec2 twiddle = vec2(cos(float(k)*ang), sin(float(k)*ang));
        vec2 t = cmul(twiddle, smem[j + m2]);
        vec2 u = smem[j];
        smem[j]      = u + t;
        smem[j + m2] = u - t;
        barrier();
    }

    data[tid]       = smem[tid];
    data[tid + 512] = smem[tid + 512];
}
```

For images larger than a single workgroup's shared memory, the FFT proceeds row-by-row then column-by-column (Stockham auto-sort variant), each row or column fitting in one workgroup dispatch. Production 2D FFTs use out-of-place transforms alternating between two buffers to maximise memory access efficiency.

### 5.2 VkFFT — Production Vulkan FFT Library

Writing and tuning a full FFT kernel for arbitrary sizes, multiple radices, and FP16/FP32/FP64 precision is complex engineering. The open-source *VkFFT* library provides a header-only solution that generates and JIT-compiles optimised FFT compute shaders at runtime.
[Source: VkFFT repository, https://github.com/DTolm/VkFFT]

VkFFT supports Vulkan, CUDA (via PTX), HIP (AMD ROCm), OpenCL, Level Zero (Intel), and Metal backends, all from a single configuration struct:

```c
// Minimal VkFFT 2D FFT setup for Vulkan (conceptual — see VkFFT_TestSuite.cpp)
VkFFTConfiguration cfg = {};
cfg.FFTdim      = 2;          // 2D transform
cfg.size[0]     = 1920;       // width
cfg.size[1]     = 1080;       // height
cfg.numberBatches = 1;
cfg.device      = &vkDevice;
cfg.queue       = &vkQueue;
cfg.fence       = &vkFence;
cfg.commandPool = &vkCommandPool;
cfg.physicalDevice = &vkPhysDevice;
cfg.buffer      = &inputBuffer;     // VkBuffer* with data
cfg.bufferSize  = &bufferBytes;     // uint64_t*

VkFFTApplication app = {};
VkFFTResult res = initializeVkFFT(&app, cfg);
// dispatch forward transform:
VkFFTLaunchParams params = {};
params.commandBuffer = cmdBuf;
VkFFTAppend(&app, -1, &params);   // -1 = forward, 1 = inverse
deleteVkFFT(&app);
```

VkFFT implements radix-2/3/4/5/7/8/11/13 decomposition, Rader's algorithm for prime factors ≥ 17, and Bluestein's algorithm for arbitrary sizes. The library handles 2D images up to ~4 billion elements and supports single (FP32), double (FP64), half (FP16), and quad-precision via double-double.

### 5.3 FFT Convolution and Wiener Deconvolution

Convolution with a large kernel h (large PSF in deblurring, large filter in template matching) is O(N log N) in the frequency domain versus O(N·k²) spatially. The pointwise complex multiply in frequency domain is:

```glsl
// fft_conv_pointwise.comp — element-wise complex multiply
layout(local_size_x = 256) in;
layout(set=0, binding=0) buffer ImageFFT  { vec2 H[]; };  // FFT of image
layout(set=0, binding=1) buffer KernelFFT { vec2 K[]; };  // FFT of kernel (precomputed)

vec2 cmul(vec2 a, vec2 b) { return vec2(a.x*b.x-a.y*b.y, a.x*b.y+a.y*b.x); }
void main() {
    uint i = gl_GlobalInvocationID.x;
    H[i] = cmul(H[i], K[i]);
}
```

*Wiener deconvolution* recovers an image from a blurred, noisy observation by applying a frequency-domain filter:

```
W(f) = H*(f) / (|H(f)|² + SNR⁻¹)
```

where `H*(f)` is the conjugate of the blur PSF's FFT and `SNR` is the signal-to-noise ratio (often estimated from the flat regions of the image). This degenerates to plain inverse filtering when noise → 0. The pointwise application of W is a second complex multiply pass identical in structure to the convolution pass above.

### 5.4 Frequency-Selective Filtering

With the image in the frequency domain, spatially-global filters become pointwise mask operations on the FFT coefficients:

- **Low-pass filter:** zero coefficients with frequency radius `r > r_cut` (brick-wall) or multiply by `exp(-r²/2σ²)` for a Gaussian smooth roll-off. Removes high-frequency noise while blurring edges.
- **High-pass filter:** `H(r) = 1 − LP(r)`, or `H(r) = r² / (r² + r₀²)` for a Butterworth variant. Enhances edges and detail; used in sharpening and focus scoring.
- **Band-pass / notch filter:** zeroes a narrow ring of frequencies (band-reject / notch). Useful for removing periodic interference patterns (moire, scan-line artefacts): identify the spike in the power spectrum, set a small disc of radius ε around it to zero, and inverse FFT. On GPU the notch mask is precomputed as a storage image and applied in a single multiply pass using the same `fft_conv_pointwise.comp` shader.

All three are single-pass variants of the pointwise complex multiply (§5.3) with a different mask `K[i]`.

---

## 6. Optical Flow

### 6.1 TV-L1 Primal-Dual Solver

TV-L1 optical flow minimises a functional combining a data term (L1 difference between warped frames) and a total-variation regulariser on the flow field (u,v). A primal-dual iterative algorithm alternates two coupled per-pixel update rules:

```
p^{n+1}  = proj_{ ||p||≤1 } (p^n + σ ∇(Ku^n))   // dual variable update
u^{n+1}  = prox_{λ/τ||·||_1} (u^n - τ K^T p^{n+1})  // primal variable update
```

where K is the linearised data term operator. The GPU implementation dispatches one compute pass per dual update and one per primal update, each independent over all pixels. Because convergence typically requires 50–200 outer iterations, the cost is dominated by memory bandwidth rather than ALU; using FP16 images and 16×16 tiles with shared memory load significantly reduces bandwidth.

The standard Linux open-source reference for TV-L1 on GPU is the OpenCV `cv::cuda::OpticalFlowDual_TVL1` module [Source: OpenCV CUDA optical flow, https://docs.opencv.org/4.x/d2/d75/namespacecv_1_1cuda.html], which wraps a CUDA implementation. A portable Vulkan compute equivalent requires hand-writing the warp + data term computation, then reusing the standard bilateral primal-dual update kernels.

### 6.2 RAFT and Deep Optical Flow

RAFT (Recurrent All-Pairs Field Transforms) replaced hand-designed optical flow with a learned architecture:

1. **Feature encoder** — a shared-weight CNN extracts dense feature maps at 1/8 resolution from both frames.
2. **Context encoder** — a second CNN encodes context from the first frame.
3. **Correlation volume** — an all-pairs dot product between every pair of feature vectors across the search space, producing a 4D volume of shape `[H/8, W/8, H/8, W/8]`.
4. **Iterative GRU refinement** — a ConvGRU hidden state is updated at each iteration by indexing the correlation volume around the current flow estimate, then predicting a flow delta.

For GPU inference on Linux, RAFT models are typically served through ONNX Runtime with a CUDA or TensorRT execution provider, or via LibTorch (PyTorch C++ API) using `torch::jit::load`. The correlation volume construction (step 3) is the most memory-intensive stage — for 480×640 input it requires ≈5 GB at FP32, which motivated later architectures (GMFlow, FlowFormer) to use coarser matching or attention-based alternatives.

### 6.3 Motion Compensation in Video Encoding

GPU-computed optical flow feeds directly into video codec motion estimation. GStreamer's `nvh264enc` and `vaapih264enc` elements accept external motion vectors when operating in `_LowLatency` or `_custom_ME` mode. The flow is:

1. Run a GPU optical flow pass (TV-L1 or a lightweight CNN) on the inter-frame pair, producing an MV field at macroblock (16×16) resolution.
2. Quantise sub-pixel MV estimates to the codec's motion vector precision (quarter-pixel for H.264 / H.265).
3. Pass the MV field to the codec via VA-API `VAEncMiscParameterTypeROI` or an extension surface depending on the driver. Note: the exact VA-API path for external MV injection is implementation-specific; verify against the driver's capabilities via `vaQueryConfigAttributes`.

This allows quality-aware encoding — giving higher motion-vector budget to regions detected as high-motion — and enables low-latency screen capture where intra-frame re-use of GPU rasterisation results (depth buffer, velocity buffer) avoids a redundant motion search.

---

## 7. Stereo Matching and Depth

### 7.1 Semi-Global Matching (SGM)

SGM remains the dominant real-time stereo algorithm, combining per-pixel cost computation with scanline-level smoothness constraints across multiple directions. The GPU pipeline consists of three compute stages:

**Stage 1 — Cost volume construction.** For each pixel (x, y) and disparity hypothesis d (0 to D−1), compute a matching cost C(x, y, d). Two standard cost functions are:

- **SAD (Sum of Absolute Differences):** sum of |I_L(x,y) − I_R(x−d,y)| over a small patch.
- **Census transform:** encodes the sign of each pixel's difference from the patch centre into a bit-vector; Hamming distance between left and right census codes gives the cost. Census is more robust to illumination changes.

The cost volume has shape W×H×D and for 1080p with D=256 requires W×H×256×1 byte ≈ 530 MB, suggesting that FP8/uint8 quantisation or a reduced disparity range is needed in practice.

**Stage 2 — Cost aggregation.** For each of 8 (or 16) scan-line directions, an accumulated cost `Lr(p, d)` is computed along the scanline:

```
Lr(p,d) = C(p,d) + min[Lr(p-r,d), Lr(p-r,d±1)+P1, min_k Lr(p-r,k)+P2] - min_k Lr(p-r,k)
```

where `P1 < P2` are smoothness penalties. Each direction can be processed independently, but within a direction the update is sequential. A practical GPU approach dispatches one thread per scanline, iterating along the direction. The 8 direction passes run sequentially (or in parallel for opposing directions) and accumulate into a shared aggregation volume.

**Stage 3 — Winner-Takes-All (WTA) selection.** Sum the aggregated costs across all directions and select the disparity with minimum cost:

```glsl
// wta_disparity.comp
layout(local_size_x = 16, local_size_y = 16) in;
layout(set=0, binding=0) readonly buffer CostVol { uint16_t cost[]; };  // W*H*D
layout(set=0, binding=1, r16ui) uniform writeonly image2D disp;
layout(push_constant) uniform PC { int W, H, D; } pc;

void main() {
    ivec2 p    = ivec2(gl_GlobalInvocationID.xy);
    int   base = (p.y * pc.W + p.x) * pc.D;
    uint  best_d = 0u;
    uint  best_c = 0xFFFFFFFFu;
    for (int d = 0; d < pc.D; d++) {
        uint c = uint(cost[base + d]);
        if (c < best_c) { best_c = c; best_d = uint(d); }
    }
    imageStore(disp, p, uvec4(best_d));
}
```

**Confidence filtering.** WTA alone produces noisy disparities at textureless regions. A confidence measure derived from the cost volume — e.g., the ratio between the best and second-best matching cost, or the left-right consistency check (`|disp_L(x,y) − disp_R(x − disp_L, y)| > 1`) — marks unreliable pixels. A second compute pass writes a confidence mask into a `r8` image, and subsequent depth-completion or smoothing passes use the mask to skip or de-weight unreliable pixels.

**Depth completion** from sparse disparity maps (LIDAR+stereo fusion) is typically performed via bilateral upsampling — a guided filter (§2.4) with the colour image as guidance — or the IP-Basic algorithm, which fills holes with morphological operations and median filtering.

---

## 8. Super-Resolution

### 8.1 Classical Upscaling: Lanczos and Bicubic

**Lanczos** upscaling uses a windowed sinc kernel `L(x) = sinc(x) · sinc(x/a)` (a=2 or 3 for Lanczos-2 or -3). For upscaling factor `s`, each output pixel samples `2a` input positions along each axis:

```glsl
// lanczos_horiz.comp — horizontal Lanczos-3 pass
layout(local_size_x = 256) in;
layout(set=0, binding=0) uniform sampler2D src;  // texture for hardware fetch
layout(set=0, binding=1, rgba16f) uniform writeonly image2D dst;
layout(push_constant) uniform PC { float scale; int a; } pc;

float lanczos(float x) {
    if (abs(x) < 1e-6) return 1.0;
    if (abs(x) >= float(pc.a)) return 0.0;
    float px = 3.14159265 * x;
    return sin(px) * sin(px / float(pc.a)) / (px * px / float(pc.a));
}

void main() {
    ivec2 out_p = ivec2(gl_GlobalInvocationID.xy);
    vec2  sz    = vec2(imageSize(dst));
    float src_x = (float(out_p.x) + 0.5) / pc.scale;
    int   x0    = int(floor(src_x)) - pc.a + 1;
    vec4  acc   = vec4(0.0);
    float wt    = 0.0;
    for (int k = 0; k < 2*pc.a; k++) {
        float w   = lanczos(src_x - float(x0 + k));
        vec2  uv  = vec2(float(x0+k) + 0.5, float(out_p.y) + 0.5) / sz;
        acc      += w * texture(src, uv);
        wt       += w;
    }
    imageStore(dst, out_p, acc / wt);
}
```

**Bicubic** uses a cubic polynomial kernel with parameter `a = −0.5` (Catmull-Rom) or `a = −0.75` (Mitchell), similarly separable.

### 8.2 Neural Super-Resolution via ONNX Runtime on Linux

ESRGAN (Enhanced Super-Resolution GAN) and Real-ESRGAN use a Residual-in-Residual Dense Block (RRDB) network that maps a low-resolution input to a 4× upscaled output in a single forward pass. Inference on Linux can be performed through ONNX Runtime with the CUDA execution provider:

```cpp
// ort_sr_inference.cpp — Real-ESRGAN via ONNX Runtime (conceptual)
#include <onnxruntime_cxx_api.h>

Ort::Env env(ORT_LOGGING_LEVEL_WARNING, "sr");
Ort::SessionOptions opts;
OrtCUDAProviderOptions cuda_opts;
cuda_opts.device_id = 0;
opts.AppendExecutionProvider_CUDA(cuda_opts);

Ort::Session session(env, L"realesrgan_x4.onnx", opts);
// Input: float32 tensor [1, 3, H, W], normalized [0,1] in RGB
// Output: float32 tensor [1, 3, 4H, 4W]
```

The ONNX Runtime TensorRT execution provider can further accelerate inference by compiling the model to NVIDIA TensorRT on first run, caching the engine for subsequent invocations.

The proprietary analogue is NVIDIA DLSS (Deep Learning Super Sampling), which uses a transformer-based architecture trained on high-quality content and exposed via the DLSS SDK. Unlike FSR 2, DLSS is NVIDIA-only and closed-source; FSR 2 is the open-source, GPU-vendor-agnostic equivalent that integrates natively into Vulkan on AMD, Intel, and NVIDIA hardware under Linux.

### 8.3 AMD FidelityFX Super Resolution 2 (FSR 2)

FSR 2 is AMD's open-source temporal upscaling algorithm, available under MIT licence and Vulkan-native.
[Source: FidelityFX-FSR2 repository, https://github.com/GPUOpen-Effects/FidelityFX-FSR2]

FSR 2 is a composite effect comprising six sequential compute passes:

1. **Luminance pyramid** — SPD (Single Pass Downsampler) produces exposure and 1/2-resolution luminance.
2. **Reconstruct and dilate** — depth and motion vectors are reconstructed and dilated into display resolution.
3. **Depth clip** — detects disoccluded areas using reprojected depth.
4. **Create locks** — establishes per-pixel convergence locks to protect temporally stable regions.
5. **Reproject and accumulate** — reprojects the previous high-resolution frame and blends with the current render.
6. **RCAS (Robust Contrast-Adaptive Sharpening)** — post-process sharpening pass applied to the accumulated output.

The public API is three functions:

```c
#include "ffx-fsr2-api/ffx_fsr2.h"
#include "ffx-fsr2-api/vk/ffx_fsr2_vk.h"

// 1. Initialise the Vulkan backend
size_t scratch_sz = ffxFsr2GetScratchMemorySizeVK(physDevice);
void  *scratch    = malloc(scratch_sz);
FfxFsr2Interface vk_interface;
ffxFsr2GetInterfaceVK(&vk_interface, scratch, scratch_sz,
                       physDevice, vkGetDeviceProcAddr);

// 2. Create FSR2 context
FfxFsr2ContextDescription ctx_desc = {};
ctx_desc.flags       = FFX_FSR2_ENABLE_HIGH_DYNAMIC_RANGE |
                       FFX_FSR2_ENABLE_DEPTH_INVERTED;
ctx_desc.maxRenderSize.width  = 1920;   // render resolution
ctx_desc.maxRenderSize.height = 1080;
ctx_desc.displaySize.width    = 3840;   // output (display) resolution
ctx_desc.displaySize.height   = 2160;
ctx_desc.callbacks   = vk_interface;
ctx_desc.device      = ffxGetDeviceVK(vkDevice);

FfxFsr2Context ctx;
ffxFsr2ContextCreate(&ctx, &ctx_desc);

// 3. Per-frame dispatch
FfxFsr2DispatchDescription disp = {};
disp.commandList     = ffxGetCommandListVK(cmdBuf);
disp.color           = ffxGetTextureResourceVK(&ctx, colorImage, colorView, ...);
disp.depth           = ffxGetTextureResourceVK(&ctx, depthImage, depthView, ...);
disp.motionVectors   = ffxGetTextureResourceVK(&ctx, mvImage, mvView, ...);
disp.output          = ffxGetTextureResourceVK(&ctx, outputImage, outputView, ...);
disp.jitterOffset.x  = jitter_x;
disp.jitterOffset.y  = jitter_y;
disp.renderSize      = { 1920, 1080 };
disp.enableSharpening = true;
disp.sharpness       = 0.5f;
disp.frameTimeDelta  = delta_ms;
disp.reset           = camera_cut;
ffxFsr2ContextDispatch(&ctx, &disp);
```

FSR 2 supports four quality presets: Quality (1.5× per dimension), Balanced (1.7×), Performance (2.0×), and Ultra Performance (3.0×).
[Source: ffx_fsr2.h, https://github.com/GPUOpen-Effects/FidelityFX-FSR2/blob/master/src/ffx-fsr2-api/ffx_fsr2.h]

---

## 9. Color Processing and Histograms

### 9.1 Color Space Conversion in Compute

**sRGB to linear.** The IEC 61966-2-1 standard defines a piecewise transfer function. The common approximation `linear = pow(sRGB, 2.2)` is acceptable for most processing; the exact form is:

```glsl
float srgb_to_linear(float c) {
    return (c <= 0.04045) ? c / 12.92
                          : pow((c + 0.055) / 1.055, 2.4);
}
```

**YCbCr (BT.601) to RGB:**

```glsl
vec3 ycbcr_to_rgb(vec3 yuv) {
    float y  = yuv.x - 16.0/255.0;
    float cb = yuv.y - 128.0/255.0;
    float cr = yuv.z - 128.0/255.0;
    return clamp(mat3(1.164,  0.0,    1.596,
                      1.164, -0.392, -0.813,
                      1.164,  2.017,  0.0  ) * vec3(y, cb, cr), 0.0, 1.0);
}
```

For NV12 (Y plane + interleaved UV plane at half resolution), a compute shader reads the Y value from binding 0 and the UV pair from binding 1 using integer coordinates, then applies the matrix above. This is the standard path for camera preview rendering from V4L2 NV12 output.

### 9.2 Histogram Computation with Shared-Memory Atomics

A global histogram requires atomic writes that serialise on the same bin. The standard GPU approach uses per-tile partial histograms in shared memory, then merges them:

```glsl
// histogram.comp — 256-bin luminance histogram
layout(local_size_x = 16, local_size_y = 16) in;
layout(set=0, binding=0, r8) uniform readonly image2D src;
layout(set=0, binding=1) buffer Histo { uint bins[256]; };

shared uint local_bins[256];

void main() {
    uint lid = gl_LocalInvocationID.x + gl_LocalInvocationID.y * 16u;
    if (lid < 256u) local_bins[lid] = 0u;
    barrier();

    ivec2 p   = ivec2(gl_GlobalInvocationID.xy);
    float val = imageLoad(src, p).r;
    uint  bin = uint(val * 255.0);
    atomicAdd(local_bins[bin], 1u);
    barrier();

    if (lid < 256u)
        atomicAdd(bins[lid], local_bins[lid]);
}
```

The shared-memory atomic reduces contention from O(N) global atomics to O(N/T) per workgroup (T = 256 threads), followed by 256 global atomic adds — a factor-T improvement.

### 9.3 CLAHE (Contrast-Limited Adaptive Histogram Equalization)

CLAHE divides the image into a grid of non-overlapping tiles (e.g., 8×8 or 16×16 tiles), computes a histogram per tile, clips each histogram at `clipLimit` (redistributing the excess evenly across all bins), computes the CDF per tile, then bilinearly interpolates between the four surrounding tile CDFs for each output pixel.

The GPU pipeline is four passes:

1. **Tile histogram compute** — one workgroup per tile, using shared-memory atomics as above.
2. **Clip and redistribute** — one thread per tile, serial per tile but parallel across tiles. The surplus `sum_over_clipLimit` is redistributed as `surplus / 256` per bin; residual is spread from bin 0 upward.
3. **CDF normalisation** — exclusive prefix scan within each tile's 256 bins.
4. **Bilinear interpolation** — each pixel identifies the four surrounding tile CDFs and interpolates using its fractional tile-coordinate position.

### 9.4 3D LUT Application (Tetrahedral Interpolation)

A 3D lookup table of size 33×33×33 (or 65³ for high-accuracy colour grading) maps input RGB triplets to output RGB. Direct trilinear interpolation uses 8 samples from the enclosing cube; tetrahedral interpolation subdivides the cube into 6 tetrahedra based on the dominant fractional coordinate, requiring only 4 samples with no multiplication artefacts on the cube diagonals.

```glsl
// apply_3dlut.comp
layout(set=0, binding=0) uniform sampler3D lut;  // 33^3 LUT in hardware
layout(set=0, binding=1, rgba8) uniform readonly  image2D src;
layout(set=0, binding=2, rgba8) uniform writeonly image2D dst;
layout(push_constant) uniform PC { float lutSize; } pc;  // e.g. 32.0 (0..32 index)

void main() {
    ivec2 p = ivec2(gl_GlobalInvocationID.xy);
    vec3  c = imageLoad(src, p).rgb;
    // scale to [0, lutSize] index range, then normalise for sampler
    vec3  tc = (c * pc.lutSize + 0.5) / (pc.lutSize + 1.0);
    // hardware trilinear; tetrahedral requires manual 4-corner lookup
    imageStore(dst, p, vec4(texture(lut, tc).rgb, 1.0));
}
```

For tetrahedral interpolation use a storage buffer holding the LUT as a flat array and perform the manual 4-sample computation; hardware trilinear on a `sampler3D` provides only trilinear accuracy and is acceptable for most colour grading workloads.

---

## 10. ISP Pipeline on Linux

### 10.1 libcamera Architecture

libcamera separates camera *pipeline handling* (kernel driver interaction, DMA-BUF buffer management, format negotiation) from *image processing algorithm* (IPA) logic. IPA modules run in a sandboxed process and implement a generic algorithm interface that operates on ISP statistics and returns parameter updates to the pipeline handler.
[Source: libcamera repository, https://github.com/libcamera-org/libcamera]

The core `Algorithm` template is defined in `src/ipa/libipa/algorithm.h`:

```cpp
// src/ipa/libipa/algorithm.h (libcamera-org/libcamera, main branch)
template<typename _Module>
class Algorithm {
public:
    using Module = _Module;
    virtual int  init(typename Module::Context &ctx,
                      const ValueNode &tuningData)          { return 0; }
    virtual int  configure(typename Module::Context &ctx,
                           const typename Module::Config &c) { return 0; }
    virtual void queueRequest(typename Module::Context &ctx,
                              const uint32_t frame,
                              typename Module::FrameContext &fc,
                              const ControlList &controls)   {}
    virtual void prepare(typename Module::Context &ctx,
                         const uint32_t frame,
                         typename Module::FrameContext &fc,
                         typename Module::Params *params)    {}
    virtual void process(typename Module::Context &ctx,
                         const uint32_t frame,
                         typename Module::FrameContext &fc,
                         const typename Module::Stats *stats,
                         ControlList &metadata)               {}
};
```

The `prepare()` method runs *before* each frame is captured, writing ISP hardware registers via the `Params` structure. The `process()` method runs *after* the frame's statistics are available, updating the algorithm's internal state for the next frame.

### 10.2 IPA Algorithm Interfaces

The Raspberry Pi IPA (`src/ipa/rpi/controller/`) defines abstract interfaces for each ISP algorithm:
[Source: libcamera rpi controller, https://github.com/libcamera-org/libcamera/tree/main/src/ipa/rpi/controller]

**Auto White Balance (`awb_algorithm.h`):**
```cpp
class AwbAlgorithm : public Algorithm {
    virtual unsigned int getConvergenceFrames() const = 0;
    virtual void setMode(std::string const &modeName) = 0;
    virtual void setManualGains(double manualR, double manualB) = 0;
    virtual void setColourTemperature(double temperatureK) = 0;
    virtual void enableAuto() = 0;
    virtual void disableAuto() = 0;
};
```

**Auto Gain/Exposure Control (`agc_algorithm.h`):**
```cpp
class AgcAlgorithm : public Algorithm {
    virtual unsigned int getConvergenceFrames() const = 0;
    virtual void setEv(unsigned int channel, double ev) = 0;
    virtual void setFlickerPeriod(libcamera::utils::Duration period) = 0;
    virtual void setMeteringMode(std::string const &name) = 0;
    virtual void setExposureMode(std::string const &name) = 0;
    virtual void enableAutoExposure() = 0;
    virtual void enableAutoGain() = 0;
};
```

**Denoise (`denoise_algorithm.h`):**
```cpp
enum class DenoiseMode { Off, ColourOff, ColourFast, ColourHighQuality };
class DenoiseAlgorithm : public Algorithm {
    virtual void setMode(DenoiseMode mode) = 0;
    virtual void setConfig(std::string const &name) {}  // optional
};
```

**Sharpening (`sharpen_algorithm.h`):**
```cpp
class SharpenAlgorithm : public Algorithm {
    virtual void setStrength(double strength) = 0;
};
```

### 10.3 Controller and ISP Pipeline Order

The `Controller` class (`src/ipa/rpi/controller/controller.h`) owns a list of `AlgorithmPtr` objects and runs them in declaration order on each frame. The typical pipeline order on the Raspberry Pi ISP is:

1. **Black level subtraction** — removes sensor dark current.
2. **Lens shading correction (ALSC)** — compensates vignetting with a per-channel gain map.
3. **Debayer / demosaic** — converts Bayer-pattern RAW (e.g., RGGB) to RGB.
4. **Denoise** — spatial or temporal noise reduction (ColourFast uses HW chroma NR; ColourHighQuality adds luma NR).
5. **Auto White Balance (AWB) / Colour Correction Matrix (CCM)** — converts sensor RGB to sRGB with a 3×3 CCM adjusted for scene colour temperature.
6. **Gamma / tone mapping** — applies the display transfer function.
7. **Sharpening** — unsharp mask or Laplacian enhancement.

The Controller's `prepare()` is called once per frame before capture, aggregating algorithm register writes into the ISP `Params` struct. After capture, `process()` is called with the ISP `Stats` struct (AE histogram, AWB colour statistics, focus phase-detection data).

### 10.4 V4L2 Format Negotiation

The libcamera pipeline handler negotiates formats between the sensor, ISP, and output using V4L2 subdevice API calls. A typical negotiation for a 10-bit Bayer sensor with NV12 ISP output looks like:

```bash
# V4L2 subdev format negotiation (via v4l2-ctl or media-ctl)
# Sensor output: RAW10 Bayer RGGB
media-ctl --set-v4l2 "'imx477 4-001a':0 [fmt:SRGGB10_1X10/4056x3040]"
# ISP input: same format
media-ctl --set-v4l2 "'bcm2835-isp0-output0':0 [fmt:SRGGB10_1X10/4056x3040]"
# ISP main output: NV12 (Y plane + interleaved UV)
media-ctl --set-v4l2 "'bcm2835-isp0-capture1':0 [fmt:NV12/1920x1080]"
```

The resulting NV12 DMA-BUF can be imported into a Vulkan pipeline as a `VkImage` with `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM` (or the equivalent `VkSamplerYcbcrConversion` path) for further GPU processing.

**P010 and HDR formats.** HDR cameras (HLG or PQ transfer function) and 10-bit video codecs output P010 format: a 16-bit-per-component NV12 variant where the 10-bit value occupies the most significant 10 bits of each 16-bit word. Mesa handles P010 via `VK_FORMAT_G10X6_B10X6R10X6_2PLANE_420_UNORM_3PACK16`. In a compute shader, P010 values must be right-shifted by 6 bits after loading to recover the 10-bit linear value, then scaled to [0,1] by dividing by 1023. The ISP pipeline on Raspberry Pi exposes P010 output via the `bcm2835-isp` V4L2 device with format `V4L2_PIX_FMT_NV12_10`.

---

## 11. OpenCV and Halide GPU Backends on Linux

### 11.1 OpenCV Transparent API (UMat / OpenCL)

OpenCV's Transparent API allows most OpenCV operations to execute on GPU via OpenCL with no change to calling code beyond switching `cv::Mat` to `cv::UMat`:

```cpp
#include <opencv2/core/ocl.hpp>
#include <opencv2/imgproc.hpp>

cv::ocl::setUseOpenCL(true);
if (!cv::ocl::haveOpenCL()) { /* fallback to CPU */ }

cv::UMat src, blurred, edges;
cv::imread("frame.png").copyTo(src);   // uploads to OpenCL device memory on first use
cv::GaussianBlur(src, blurred, {7,7}, 1.5);
cv::Canny(blurred, edges, 50, 150);
// Results remain on device; download only when needed:
cv::Mat cpu_edges;
edges.copyTo(cpu_edges);
```

`UMat` operations dispatch OpenCL kernels via `clEnqueueNDRangeKernel`. The kernel source files are compiled JIT on first use and cached. For operations not yet ported to OpenCL, OpenCV silently falls back to CPU.

### 11.2 OpenCV CUDA Backend (cv::cuda)

The `cv::cuda` namespace provides explicit CUDA-backed image processing with `GpuMat` as the device buffer type:

```cpp
#include <opencv2/cudafilters.hpp>
#include <opencv2/cudaimgproc.hpp>

cv::cuda::GpuMat d_src, d_blur, d_edges;
d_src.upload(cpu_src);

// Separable Gaussian filter
auto gauss = cv::cuda::createGaussianFilter(
    CV_8UC1, CV_8UC1, {7,7}, 1.5);
gauss->apply(d_src, d_blur);

// Canny: all three passes (gradient, NMS, hysteresis) on GPU
auto canny = cv::cuda::createCannyEdgeDetector(50.0, 150.0);
canny->detect(d_blur, d_edges);

d_edges.download(cpu_edges);
```

The `cv::cuda::Filter` base class provides `apply(InputArray src, OutputArray dst, Stream &stream = Stream::Null())` for stream-ordered execution. Multiple operations can be pipelined onto a single `cv::cuda::Stream` to overlap H2D uploads and kernel execution.

[Source: OpenCV CUDA module documentation, https://docs.opencv.org/4.x/d2/d75/namespacecv_1_1cuda.html]

### 11.3 Halide GPU Scheduling

Halide separates algorithm definition from schedule, allowing the same image processing pipeline to target CPU, CUDA, OpenCL, or Vulkan by changing only the schedule and target:

```cpp
// halide_gaussian.cpp
#include "Halide.h"
using namespace Halide;

Func gauss_5x5(Func input) {
    Func blur_x("blur_x"), blur_y("blur_y");
    Var x("x"), y("y"), c("c");
    Var xo("xo"), yo("yo"), xi("xi"), yi("yi");
    Var block("block"), thread("thread");

    blur_x(x,y,c) = (input(x-2,y,c) + input(x-1,y,c) + input(x,y,c) +
                     input(x+1,y,c) + input(x+2,y,c)) / 5;
    blur_y(x,y,c) = (blur_x(x,y-2,c) + blur_x(x,y-1,c) + blur_x(x,y,c) +
                     blur_x(x,y+1,c) + blur_x(x,y+2,c)) / 5;

    // GPU schedule: 16x16 tiles, blur_x computed in shared memory
    blur_y.gpu_tile(x, y, xo, yo, xi, yi, 16, 16);
    blur_x.compute_at(blur_y, xo).gpu_threads(x, y);
    return blur_y;
}

// Compile for Vulkan:
Target target = get_host_target();
target.set_feature(Target::Vulkan);
// or for CUDA: target.set_feature(Target::CUDA);
// or for OpenCL: target.set_feature(Target::OpenCL);
gauss_pipeline.compile_jit(target);
gauss_pipeline.realize(output_buffer);
```

`gpu_tile` marks the loop variables as GPU block/thread dimensions. `compute_at(..., xo)` schedules `blur_x` inside the block, causing Halide to automatically allocate the padded shared-memory region for the tile and insert `barrier()` calls where needed.

[Source: Halide GPU tutorial, https://halide-lang.org/tutorials/tutorial_lesson_12_using_the_gpu.html]

### 11.4 Pure Vulkan Compute Pipeline

For algorithms that don't fit the Halide or OpenCV abstractions, or where precise control over barriers, synchronisation, and resource layouts is needed, a pure Vulkan compute pipeline is the correct choice:

```cpp
// Minimal Vulkan compute pipeline setup for image processing
VkDescriptorSetLayoutBinding bindings[2] = {
    {0, VK_DESCRIPTOR_TYPE_STORAGE_IMAGE, 1, VK_SHADER_STAGE_COMPUTE_BIT, nullptr},
    {1, VK_DESCRIPTOR_TYPE_STORAGE_IMAGE, 1, VK_SHADER_STAGE_COMPUTE_BIT, nullptr},
};
VkDescriptorSetLayoutCreateInfo dsl_info = {
    VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
    nullptr, 0, 2, bindings
};
VkDescriptorSetLayout dsl;
vkCreateDescriptorSetLayout(device, &dsl_info, nullptr, &dsl);

VkPipelineLayoutCreateInfo pl_info = {
    VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO,
    nullptr, 0, 1, &dsl, 0, nullptr
};
VkPipelineLayout pl;
vkCreatePipelineLayout(device, &pl_info, nullptr, &pl);

// Load SPIR-V, create VkShaderModule, then:
VkComputePipelineCreateInfo cp_info = {
    VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO, nullptr, 0,
    {VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO, nullptr, 0,
     VK_SHADER_STAGE_COMPUTE_BIT, shader_module, "main", nullptr},
    pl, VK_NULL_HANDLE, -1
};
VkPipeline pipeline;
vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &cp_info, nullptr, &pipeline);

// Dispatch
vkCmdBindPipeline(cmd, VK_PIPELINE_BIND_POINT_COMPUTE, pipeline);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_COMPUTE, pl, 0, 1, &ds, 0, nullptr);
vkCmdDispatch(cmd, (width+15)/16, (height+15)/16, 1);
```

Image barriers (`VkImageMemoryBarrier2`) must transition layouts and stages between passes — for example, from `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` to `VK_IMAGE_LAYOUT_GENERAL` before writing with `imageStore`, and back after. The synchronisation2 extension (`VK_KHR_synchronization2`) provides cleaner barrier syntax on modern drivers.

---

## 12. Integrations

This chapter connects to several related chapters in the book:

- **Chapter 25 (GPU Compute)** — The Vulkan compute pipeline setup and dispatch model used throughout this chapter (workgroups, push constants, storage images) is introduced in detail in Chapter 25.
- **Chapter 26 (Hardware Video)** — V4L2 DMA-BUF buffer sharing between the camera ISP (§10.4) and Vulkan storage images uses the same import mechanism described for VA-API and V4L2 codec output in Chapter 26.
- **Chapter 133 (Vulkan Compute Queues)** — Multi-pass image processing pipelines (bilateral filter, Canny, SGM) require careful queue selection and pipeline barrier placement; Chapter 133 covers compute queue families and async compute overlap.
- **Chapter 148 (Vulkan Synchronisation)** — The `VkImageMemoryBarrier2` layout transitions between passes (§11.4) are explained in full in Chapter 148, including pipeline stage flags and access masks for compute shaders.
- **Chapter 157 (Vulkan Descriptor Binding)** — The storage image descriptor bindings used throughout this chapter connect to Chapter 157's treatment of bindless resources and push descriptor updates.
- **Chapter 212 (GPU Neural and Specialized Primitives)** — RAFT optical flow (§6.2), ESRGAN super-resolution (§8.2), and neural ISP denoise methods build directly on the GPU neural inference infrastructure (ONNX Runtime, LibTorch, CUDA graph execution) described in Chapter 212.
- **Part XXVIII (Linux Multimedia)** — libcamera's interaction with PipeWire and GStreamer, the V4L2 media controller pipeline, and NV12/P010 format handling across the multimedia stack are covered in Part XXVIII; this chapter focuses only on the ISP algorithm layer within libcamera.
- **Chapter 208 (GPU Geometry Algorithms)** — Morphological operations in §3 of this chapter share the GPU min/max neighborhood pattern with the mesh voxelisation and SDF generation algorithms in Chapter 208.
