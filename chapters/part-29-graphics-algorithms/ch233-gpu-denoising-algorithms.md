# Chapter 233: GPU Denoising Algorithms (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers implementing path-traced or ray-traced rendering pipelines on Linux; systems developers integrating denoising into Vulkan compute workloads. This chapter assumes familiarity with Monte Carlo rendering fundamentals (Ch206), Vulkan compute basics (Ch25), and GPU performance analysis (Ch221). For temporal anti-aliasing history buffer architecture shared with denoisers, see Ch207.

---

## Table of Contents

1. [Denoising as a Rendering Stage](#1-denoising-as-a-rendering-stage)
2. [Spatio-Temporal Variance-Guided Filtering (SVGF)](#2-spatio-temporal-variance-guided-filtering-svgf)
3. [Adaptive SVGF (ASVGF)](#3-adaptive-svgf-asvgf)
4. [À-Trous Wavelet Filter Standalone](#4-à-trous-wavelet-filter-standalone)
5. [NRD — NVIDIA Real-Time Denoisers](#5-nrd--nvidia-real-time-denoisers)
6. [OIDN — Intel Open Image Denoise](#6-oidn--intel-open-image-denoise)
7. [History Rejection and Ghosting Prevention](#7-history-rejection-and-ghosting-prevention)
8. [Signal-Specific Denoising Strategies](#8-signal-specific-denoising-strategies)
9. [Neural Denoising on Linux GPU](#9-neural-denoising-on-linux-gpu)
10. [Integration with Vulkan Render Graphs](#10-integration-with-vulkan-render-graphs)
11. [Profiling and Tuning](#11-profiling-and-tuning)
12. [Open-Source Implementations and References](#12-open-source-implementations-and-references)
13. [Integrations](#13-integrations)

---

## 1. Denoising as a Rendering Stage

### 1.1 The Monte Carlo Variance Problem

Monte Carlo path tracing is an unbiased estimator of the rendering equation, but its per-pixel variance decreases only as 1/N, where N is the number of samples. At interactive frame rates, real-time path tracers budget 1–4 samples per pixel (spp). The resulting images are visually intolerable without post-processing: high-frequency noise dominates smooth surfaces, specular highlights flicker, and shadow boundaries dissolve. Denoising — the process of estimating the converged, noise-free image from a low-sample-count input — is the mechanism that makes real-time path tracing viable.

The fundamental tradeoff is between sample count and filter quality. More samples reduce variance but cost ray-budget; a stronger filter reduces noise but may over-smooth detail (albedo loss) or ghost moving objects. Modern denoising pipelines combine both: 1–4 spp input to a spatio-temporal filter with per-pixel confidence weighting, achieving perceptual quality equivalent to 32–128 spp on static scenes.

### 1.2 Input Buffer Layout

A real-time denoiser requires more than the noisy radiance buffer. The standard input set is:

| Buffer | Format | Notes |
|---|---|---|
| Noisy HDR radiance | `R16G16B16A16_SFLOAT` | Pre-tonemapping; diffuse and specular may be separated |
| Albedo (base colour) | `R8G8B8A8_UNORM` | Material colour for de-modulation |
| World-space normals | `R16G16_SFLOAT` (oct-encoded) or `R16G16B16A16_SFLOAT` | First-hit surface normal |
| View-space depth | `R32_SFLOAT` | Linearised depth, not NDC |
| Screen-space motion vectors | `R16G16_SFLOAT` | Pixel displacement from current to previous frame |
| Variance estimate | `R16_SFLOAT` | Per-pixel luminance variance, if available |

Normal and depth buffers serve as edge-stopping guides: the denoiser blurs radiance only between pixels that share the same surface. Motion vectors drive temporal reprojection, aligning history with the current frame before accumulation.

### 1.3 Denoiser Placement in the Frame Graph

Denoising must occur before tonemapping but after the raw path-trace output has been accumulated. Operating in linear HDR space preserves the correct luminance relationships that the edge-stopping functions depend on. The typical sequence:

```
Ray trace (1 spp) → [separate diffuse / specular] → Denoiser → Recombine → Tonemap → TAA / FSR2 / DLSS → Display
```

Some pipelines de-modulate albedo before denoising (`irradiance = radiance / albedo`) and re-modulate after, preventing the denoiser from blurring albedo boundaries. NRD formalises this as a mandatory pre/post-processing step (§5.4).

### 1.4 Linux GPU Denoising Runtime Landscape

Three software stacks provide denoising on Linux as of mid-2026:

- **NRD (NVIDIA Real-Time Denoisers)**: open-source C++ SDK, ships compute shaders as HLSL/SPIRV, Vulkan-native via NRI abstraction. Hardware-agnostic at the API level; tuned primarily for NVIDIA RTX but runs on any Vulkan 1.2 device. [Source: NRD SDK, https://github.com/NVIDIAGameWorks/RayTracingDenoiser]
- **OIDN (Intel Open Image Denoise)**: open-source neural denoiser, GPU execution via SYCL (Intel), HIP (AMD ROCm), and CUDA (NVIDIA). Vulkan interop through external memory import. Primary use-case is offline and semi-interactive rendering. [Source: OpenImageDenoise/oidn, https://github.com/OpenImageDenoise/oidn]
- **Hand-rolled SVGF/ASVGF**: the Schied et al. technique [Source: Schied et al., SIGGRAPH 2017, https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced] is freely implementable in Vulkan compute; reference code is available in Falcor and Q2RTX.

---

## 2. Spatio-Temporal Variance-Guided Filtering (SVGF)

SVGF [Source: Schied et al., "Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination", SIGGRAPH 2017, https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced] decomposes denoising into three sequential compute passes: temporal accumulation, variance estimation, and iterative à-trous bilateral filtering.

### 2.1 Temporal Accumulation Pass

The first pass reprojects the previous frame's filtered output into the current frame using screen-space motion vectors:

```glsl
// Pseudo-code: temporal reprojection and EMA blending
vec2 prev_uv = current_uv + motion_vector;
vec4 history  = texture(prev_frame, prev_uv);
vec4 current  = texture(noisy_radiance, current_uv);

// Exponential moving average; alpha ∈ [0.1, 0.2] is typical
float alpha   = 0.1;  // "Alpha" in Falcor SVGF controls this
vec4 blended  = mix(history, current, alpha);
```

The Falcor reference implementation exposes `Alpha` (range 0–1, controls illumination history weight) and `MomentsAlpha` (separate alpha for moment accumulation) as tunable parameters. [Source: Falcor SVGFPass.cpp, https://github.com/NVIDIAGameWorks/Falcor]

History rejection — discarding samples from the wrong surface — is controlled by depth and normal thresholds (§7).

### 2.2 Variance Estimation

The second pass computes per-pixel luminance variance using two strategies:

**Temporal variance** comes from accumulating the first and second moments of luminance over time: `m1 = E[L]`, `m2 = E[L²]`. Variance is then `σ² = m2 − m1²`. This is reliable only once sufficient history is accumulated (typically 4+ frames). For newly disoccluded pixels with little history, a **spatial variance** estimate over a 7×7 neighbourhood is substituted.

The moment buffer stores `(m1, m2)` per pixel and is updated with the same EMA alpha as the primary radiance buffer:

```
m1_new = mix(m1_history, lum(current), alpha_moments)
m2_new = mix(m2_history, lum(current)^2, alpha_moments)
variance = m2_new - m1_new^2
```

### 2.3 À-Trous Wavelet Bilateral Filter Passes

The à-trous (French: "with holes") filter applies a bilateral kernel with exponentially increasing step sizes — at level `i`, each sample is offset by multiples of `step = 2^i` pixels, skipping the intermediate pixels ("holes"). This yields a coarser spatial footprint at each level without increasing the tap count.

**SVGF uses a 5×5 B3-spline kernel** [Source: Dammertz et al., "Edge-Avoiding À-Trous Wavelet Transform for fast Global Illumination Filtering", HPG 2010] at each pass — 25 taps per output pixel, with weights drawn from the separable outer product of `(1/16, 1/4, 3/8, 1/4, 1/16)`, located at `step·{−2,−1,0,1,2}` along each axis. The step doubles each level, so levels 0–4 cover spatial footprints of 5, 9, 17, 33, 65 pixels respectively.

**Q2RTX ASVGF uses a 3×3 kernel** (9 taps, `for(yy=-1..1) for(xx=-1..1)`) with the same exponentially growing step, trading spatial coverage for lower ALU cost per pass [Source: Q2RTX asvgf_atrous.comp, https://github.com/NVIDIA/Q2RTX]. Both approaches share the same bilateral edge-stopping structure; the footprint choice is an implementation tradeoff, not an algorithm difference.

The Falcor reference implementation drives the pass loop with `gStepSize = 1 << i` and runs 2–10 iterations controlled by `mFilterIterations`:

```cpp
// Falcor SVGFPass.cpp — atrous pass loop (SVGF 5×5 B3-spline kernel)
for (int i = 0; i < mFilterIterations; i++) {
    perImageCB["gStepSize"] = 1 << i;   // step: 1, 2, 4, 8, ...
    mpAtrous->execute(pRenderContext, curTargetFbo);
    std::swap(mpPingPongFbo[0], mpPingPongFbo[1]);
}
```
[Source: Falcor SVGFPass.cpp, https://github.com/NVIDIAGameWorks/Falcor]

Each tap's bilateral weight combines three edge-stopping functions. The Q2RTX 3×3 implementation illustrates the practical form (the same functions apply to SVGF's 5×5 kernel):

```glsl
// From Q2RTX asvgf_atrous.comp — edge stopping functions
// Luminance: suppress blur across high-contrast boundaries
float sigma_l_hf = min(hist_len_hf, flt_atrous_lum_hf)
                   / (2.0 * lum_variance_hf);
w_hf *= exp(- dist_l_hf * dist_l_hf * sigma_l_hf);

// Normal: suppress blur across geometric edges
float NdotN = max(0.0, dot(normal_center, normal_sample));
w_hf *= pow(NdotN, normal_weight_hf);

// Depth: suppress blur across depth discontinuities
float dist_z = abs(depth_center - depth_sample)
               * fwidth_depth * flt_atrous_depth;
w *= exp(-dist_z / float(step_size));
```
[Source: Q2RTX asvgf_atrous.comp, https://github.com/NVIDIA/Q2RTX]

The variance-guided aspect is in `sigma_l_hf`: when variance is high (noisy region), the luminance stopping function weakens, allowing more blur. When variance is low (converged region), the function is sharp, preserving detail.

### 2.4 GPU Dispatch Pattern

SVGF maps to a sequence of compute dispatches with the same thread group dimensions (typically 8×8 or 16×16 threads):

1. `temporal_accumulate` — one dispatch, reads motion vectors and history
2. `estimate_variance` — one dispatch, reads moments and produces per-pixel σ²
3. `atrous_level_0` through `atrous_level_N-1` — N dispatches, ping-ponging between two intermediate framebuffers

Ping-pong is mandatory because each atrous level reads the output of the previous level. Memory layout: all intermediates should be `R16G16B16A16_SFLOAT` to avoid precision loss; the variance channel is packed into the alpha component.

### 2.5 Firefly Pre-Pass

Before temporal accumulation, a firefly clamping pass suppresses sample values that exceed a neighbourhood luminance threshold by a large factor. A common heuristic: clamp if `lum(pixel) > k * mean_lum(3×3_neighbourhood)` where k ∈ [4, 8]. This prevents single bright samples (caused by paths sampling a very small light source) from "burning in" to the history buffer and requiring many frames to decay.

SVGF achieves ~5–47% SSIM improvement and roughly 10× temporal stability improvement over simpler filters at approximately 10 ms for 1920×1080 on modern GPUs [Source: Schied et al. 2017 abstract, https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced].

---

## 3. Adaptive SVGF (ASVGF)

ASVGF [Source: Schied et al., "Gradient Estimation for Real-Time Adaptive Temporal Filtering", HPG 2018] extends SVGF with two mechanisms: a gradient image that detects disocclusions and temporally unstable regions, and adaptive spatial filter width driven by per-pixel sample count.

### 3.1 Gradient Image Computation and Reprojection

The gradient pass (`asvgf_gradient_img.comp` in Q2RTX) identifies pixels where the lighting has changed significantly since the previous frame. For each 3×3 block, it selects the pixel with the highest combined diffuse+specular luminance as a "gradient sample." The gradient sample is reprojected to the previous frame, and the ratio of luminance values yields a temporal gradient:

```glsl
// Conceptual: temporal luminance gradient
float lum_curr = luminance(current_sample);
float lum_prev = luminance(reprojected_history);
float gradient = abs(lum_curr - lum_prev) / max(lum_curr, lum_prev);
```

The gradient reprojection pass (`asvgf_gradient_reproject.comp`) validates reprojected gradient samples using the same depth and normal thresholds as temporal accumulation, plus an additional constraint: a pixel that was itself a gradient sample in the previous frame is excluded to prevent bias propagation. [Source: Q2RTX asvgf_gradient_reproject.comp, https://github.com/NVIDIA/Q2RTX]

### 3.2 Disocclusion Detection and Adaptive History Length

ASVGF's gradient image drives the antilag mechanism in the temporal accumulation pass. The `asvgf_temporal.comp` shader computes:

```glsl
// Adaptive history length based on temporal gradient
new_hist_len = old_hist_len * pow(1.0 - antilag, 10.0) + 1.0;
new_hist_len = min(new_hist_len, 256.0);
```
[Source: Q2RTX asvgf_temporal.comp, https://github.com/NVIDIA/Q2RTX]

When the gradient is large (lighting change or disocclusion), `antilag` is set high, rapidly reducing `hist_len`, which in turn increases the EMA `alpha = 1/hist_len` and weights current-frame samples more heavily. This recovers from ghosting at the cost of temporarily increased noise.

### 3.3 Comparison with SVGF

| Aspect | SVGF | ASVGF |
|---|---|---|
| Disocclusion response | Relies on depth/normal test failure | Gradient-driven antilag on luminance changes |
| Lighting change reaction | Slow (history must dilute) | Fast (gradient directly resets history) |
| Per-pixel sample count | Fixed 1 spp | Gradient-based selection of "best" pixel per 3×3 |
| Additional passes | None beyond SVGF | Gradient compute, gradient atrous, gradient reproject |
| Overhead vs. SVGF | Baseline | +1–2 ms at 1920×1080 (needs verification) |

ASVGF is the algorithm used in Q2RTX [Source: https://github.com/NVIDIA/Q2RTX], Quake 2's real-time path-tracing port, making it one of the most thoroughly exercised open-source denoiser implementations available.

---

## 4. À-Trous Wavelet Filter Standalone

The à-trous bilateral filter can be applied independently of temporal accumulation for signals that either don't benefit from temporal history (single-frame renders, lightmaps) or where temporal stability is already provided by a different mechanism (e.g., a separate TAA pass).

### 4.1 Wavelet Coefficient Thresholding

In the standalone case, noise suppression uses hard or soft thresholding on wavelet detail coefficients rather than variance guidance. **Hard threshold**: set coefficient to zero if `|c| < λ`. **Soft threshold**: `sign(c) * max(0, |c| − λ)`. Soft thresholding avoids discontinuities in the filtered output at the cost of slight signal attenuation.

For real-time use, the wavelet decomposition is implicit in the à-trous step structure: each filter level corresponds to one wavelet scale. Storing residuals between levels is optional; most real-time implementations simply accumulate the bilateral-filtered output directly.

### 4.2 Kernel Footprint Choices

Two footprints are common in practice for standalone à-trous denoising:

**5×5 B3-spline (SVGF standard)**: 25 taps at offsets `step·{−2,−1,0,1,2}` per axis. This is separable in the *non-bilateral* case (horizontal then vertical), but the bilateral depth/normal weights couple the two axes, requiring a full 2D pass. Cost: 25 texture reads per output pixel.

**3×3 (Q2RTX / compact variant)**: 9 taps at `step·{−1,0,1}` per axis. Lower cost; achieves similar effective filtering range over multiple levels at the cost of less accurate per-level reconstruction. Used when budget is tight.

```glsl
// 3×3 à-trous kernel body (Q2RTX style, step = 1 << level)
for (int yy = -1; yy <= 1; yy++) {
    for (int xx = -1; xx <= 1; xx++) {
        ivec2 offset = ivec2(xx, yy) * step;
        // sample radiance, depth, normal at (ipos + offset)
        // accumulate weighted contribution
    }
}
```

The choice between footprints is a quality-versus-cost tradeoff; both use the same luminance/normal/depth edge-stopping functions.

### 4.3 GPU Shared-Memory Optimisation

At small step sizes (level 0, step = 1 pixel), threads in a workgroup read overlapping neighbourhoods. Loading the 3×3 region of the 16×16 tile with halo into shared memory reduces global memory reads from 9 per thread to ~1.2 per thread amortised:

```glsl
// Workgroup 16×16, halo of 1 pixel → 18×18 shared memory load
shared vec4 s_radiance[18][18];
shared float s_depth[18][18];

// Load phase: each thread loads its pixel plus border threads load halo
// Compute phase: read from shared memory
```

At larger step sizes (levels 2+), the sample spacing exceeds the tile width and shared memory provides no benefit; direct global reads are used.

### 4.4 Ambient Occlusion and Soft Shadow Denoising

For standalone AO denoising, the normal edge-stopping function is dominant: AO varies smoothly within a flat surface and sharply only at geometric edges. Depth-based stopping alone is insufficient for curved surfaces. The 5-tap sparse filter run for 4–5 levels produces a clean AO signal from 1-spp input in under 1 ms on modern hardware.

For soft shadow denoising, a hit-distance channel (the distance to the shadow-casting surface sampled per pixel) guides the blur radius to match the penumbra width, preventing over-blur in the umbra region (§8.1).

---

## 5. NRD — NVIDIA Real-Time Denoisers

NRD [Source: NRD SDK, https://github.com/NVIDIAGameWorks/RayTracingDenoiser] is an open-source C++ library shipping three production-quality denoisers: **ReBLUR** (recurrent blur), **ReLAX** (à-trous least squares), and **SIGMA** (shadow-only). The library does not issue GPU API calls directly; instead it returns resource descriptions and compute dispatch parameters that the application submits through its own graphics API layer.

### 5.1 ReBLUR — Recurrent Blur-Based Denoiser

ReBLUR uses a three-phase blur strategy: pre-blur → main blur → post-blur. Each blur pass scales its radius based on a per-pixel hit-distance histogram. The intuition is that in a diffuse environment, the correct blur radius is proportional to the distance to the first bounce surface: nearby geometry needs a smaller filter (more detail visible), distant geometry needs a larger filter (high variance, little fine detail).

Hit distances are normalised by the denoiser before processing using `REBLUR_FrontEnd_GetNormHitDist`, a helper that maps raw ray hit distances into a unit range using a scene-scale parameter. The normalised hit distance feeds directly into the blur radius formula (exact formula is internal to NRD shaders; see `NRD.hlsli` in the SDK).

**Specular lobe tracking**: ReBLUR's specular variant tracks the BRDF lobe width across frames. As roughness decreases (sharper specular), the filter narrows; for mirror-like surfaces it avoids blurring entirely. The lobe width is derived from the roughness parameter stored in the G-buffer.

### 5.2 ReLAX — À-Trous Least Squares

ReLAX is designed specifically for use with RTXDI (RTX Direct Illumination, NVIDIA's reservoir-based direct lighting system). It employs à-trous spatial filtering in the **YCoCg colour space**, which separates luminance (Y) from chroma (Co, Cg). This decomposition allows luminance-based edge stopping to operate independently of chroma, reducing colour bleeding at surface boundaries.

**History clamping via variance**: instead of discarding bad history, ReLAX clamps the reprojected history value to the AABB (axis-aligned bounding box) of the current frame's neighbourhood in YCoCg space — the same neighbourhood-clamping strategy used by modern TAA implementations (§7.2).

**Firefly suppression**: ReLAX applies a dedicated anti-firefly pass that computes the local neighbourhood luminance range and clamps outliers. This is more expensive than SVGF's pre-pass clamping but handles temporal fireflies that survive into history.

### 5.3 SIGMA — Shadow Denoiser

SIGMA is a single-purpose denoiser for binary (0/1) and semi-transparent shadow signals from both directional (sun, moon) and local (point, spot) light sources. It processes hit distances separately from radiance to recover penumbra structure, then reconstructs a smooth shadow mask. Performance is significantly lower than ReBLUR/ReLAX: ~0.40–0.45 ms at 1440p on RTX 4080. [Source: NRD SDK README, https://github.com/NVIDIAGameWorks/RayTracingDenoiser]

### 5.4 Signal Normalisation: Albedo De-Modulation

NRD requires callers to de-modulate albedo before denoising and re-modulate after. The helper `NRD_MaterialFactors` computes the separation:

```glsl
// NRD pre-processing: irradiance = radiance / albedo
// (albedo clamped to avoid division by zero)
irradiance = radiance / max(albedo, 0.01);

// Post-processing: radiance = denoised_irradiance * albedo
output_radiance = denoised_irradiance * albedo;
```

This prevents the denoiser from treating albedo colour boundaries as noise to be smoothed. A green-to-red material transition should remain sharp; only the illumination signal should be filtered.

### 5.5 NRD SDK Integration: Vulkan Workflow

The application drives NRD through a stateless query interface:

```cpp
// 1. Query library and denoiser capabilities
nrd::LibraryDesc libraryDesc = nrd::GetLibraryDesc();

// 2. Create denoiser instance (ReBLUR or ReLAX)
nrd::InstanceCreationDesc instanceDesc = {};
instanceDesc.requestedDenoisers    = &denoiserDesc;
instanceDesc.requestedDenoisersNum = 1;
nrd::CreateInstance(instanceDesc, m_NrdInstance);

// 3. Each frame: set per-frame parameters
nrd::CommonSettings commonSettings = {};
commonSettings.motionVectorScale[0] = 1.0f / screenWidth;
commonSettings.motionVectorScale[1] = 1.0f / screenHeight;
nrd::SetCommonSettings(m_NrdInstance, commonSettings);

// 4. Retrieve dispatch list and submit to Vulkan
const nrd::DispatchDesc* dispatchDescs;
uint32_t dispatchDescNum;
nrd::GetComputeDispatches(m_NrdInstance, &dispatchDescs, dispatchDescNum);
// Application submits each dispatch via vkCmdDispatch
```
[Source: NRD SDK integration guide, https://github.com/NVIDIAGameWorks/RayTracingDenoiser]

For Vulkan, SPIRV binding offsets are retrieved from `libraryDesc.spirvBindingOffsets` to configure the descriptor set layouts correctly. The NRI (NVIDIA Rendering Interface) integration layer (`NRDIntegration.hpp`) wraps this flow and handles intermediate texture allocation, descriptor heap management, and pipeline state restoration.

**Performance at 1440p on RTX 4080:**

| Denoiser | Mode | Time |
|---|---|---|
| ReBLUR | Diffuse + Specular | 2.55 ms |
| ReBLUR | Diffuse + Specular (SH) | 3.40 ms |
| ReLAX | Diffuse + Specular | 3.25 ms |
| ReLAX | Diffuse + Specular (SH) | 4.80 ms |
| SIGMA | Shadow | 0.40–0.45 ms |

[Source: NRD SDK README performance table, https://github.com/NVIDIAGameWorks/RayTracingDenoiser]

---

## 6. OIDN — Intel Open Image Denoise

OIDN [Source: Intel Open Image Denoise, https://github.com/OpenImageDenoise/oidn] is a neural image denoiser based on a U-Net convolutional encoder-decoder architecture. Unlike SVGF and NRD (which are designed for real-time), OIDN targets semi-interactive offline rendering, lightmap baking, and production path tracing. The network is trained on HDR Monte Carlo renders and generalises across scene types without scene-specific tuning.

### 6.1 U-Net Architecture

The filter uses an encoder-decoder network with skip connections (U-Net topology). The encoder progressively downsamples the input through convolutional blocks, building a hierarchy of feature maps. The decoder upsamples back to full resolution, with skip connections from matching encoder layers preventing information loss. The network operates on 3-channel colour plus optional auxiliary inputs (albedo, normals). Architecture specifics (layer counts, channel widths, kernel sizes) are not publicly documented by Intel; the above is inferred from the "RT filter" description and OIDN's published training methodology [Source: https://www.openimagedenoise.org/documentation.html].

### 6.2 GPU Execution Backends

OIDN selects the execution backend from the device type at creation:

```c
// C99 API — backend selection
OIDNDevice device = oidnNewDevice(OIDN_DEVICE_TYPE_DEFAULT); // auto-select
// Or explicitly:
OIDNDevice cuda_dev = oidnNewCUDADevice(&cuda_device_id, &cuda_stream, 1);
OIDNDevice hip_dev  = oidnNewHIPDevice(&hip_device_id, &hip_stream, 1);
OIDNDevice sycl_dev = oidnNewSYCLDevice(sycl_queues, 1);
```

Backend support matrix:

| Backend | Hardware | Runtime requirement |
|---|---|---|
| SYCL / Level Zero | Intel Xe, Xe2, Xe3 | oneAPI DPC++ Compiler |
| CUDA | NVIDIA Turing–Blackwell | CUDA Toolkit 12.8+ |
| HIP | AMD RDNA 2–4 | ROCm 6.4.2+ |
| Metal | Apple M1+ | (not relevant to Linux) |
| CPU | x86, ARM | Default fallback |

[Source: OIDN documentation, https://www.openimagedenoise.org/documentation.html]

### 6.3 Core API Usage

```c
OIDNDevice device = oidnNewDevice(OIDN_DEVICE_TYPE_DEFAULT);
oidnCommitDevice(device);

OIDNFilter filter = oidnNewFilter(device, "RT");  // or "RTLightmap"

// Assign image buffers (device-accessible memory)
oidnSetSharedFilterImage(filter, "color",  color_ptr,
    OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0);
oidnSetSharedFilterImage(filter, "albedo", albedo_ptr,
    OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0);
oidnSetSharedFilterImage(filter, "normal", normal_ptr,
    OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0);
oidnSetSharedFilterImage(filter, "output", output_ptr,
    OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0);

oidnSetFilterBool(filter, "hdr", true);    // HDR input
oidnCommitFilter(filter);
oidnExecuteFilter(filter);

// Check for errors
const char* errorMsg;
if (oidnGetDeviceError(device, &errorMsg) != OIDN_ERROR_NONE)
    fprintf(stderr, "OIDN error: %s\n", errorMsg);
```
[Source: OIDN API documentation, https://www.openimagedenoise.org/documentation.html]

### 6.4 Vulkan Interop via External Memory

OIDN imports GPU buffers allocated by Vulkan through POSIX file descriptor handles (Linux) using `VK_KHR_external_memory_fd`:

```c
// Export VkDeviceMemory as an opaque FD
VkMemoryGetFdInfoKHR fdInfo = {
    .sType      = VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR,
    .memory     = vk_device_memory,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT,
};
int fd;
vkGetMemoryFdKHR(device, &fdInfo, &fd);

// Import into OIDN
OIDNBuffer oidn_buf = oidnNewSharedBufferFromFD(
    oidn_device,
    OIDN_EXTERNAL_MEMORY_TYPE_FLAG_OPAQUE_FD,
    fd,
    buffer_size_bytes
);
```
[Source: OIDN documentation §External Memory, https://www.openimagedenoise.org/documentation.html]

This eliminates host-side staging copies. External semaphores (`oidnWaitSemaphoresAsync`, `oidnSignalSemaphoresAsync`) synchronise Vulkan timeline semaphores with OIDN execution.

### 6.5 Filter Types

- **RT**: General-purpose ray-traced image denoiser. Accepts LDR or HDR input with optional albedo and normal auxiliary channels. Three quality presets: `fast`, `balanced`, `high`.
- **RTLightmap**: Variant optimised for HDR lightmaps and directional lightmaps (spherical harmonics). Designed for offline baking workflows where the whole lightmap is denoised as a large texture.

---

## 7. History Rejection and Ghosting Prevention

### 7.1 Disocclusion Detection

Temporal reprojection fails when a pixel's previous-frame surface is occluded in the current frame (disocclusion). The standard two-test detection:

**Depth test**: compare reprojected depth against current depth. A depth difference exceeding a threshold (in Q2RTX: `dist_depth < 0.1` in normalised view-space units) indicates the reprojected sample hits a different surface. The threshold must be scene-scale dependent — a fixed 0.1 m threshold fails in both micro-scale and planetary-scale renderers.

**Normal test**: `dot(normal_prev, normal_curr) > 0.5` (cosine threshold, ~60°). Surfaces at steep angles to each other are rejected even if their depths happen to match.

Q2RTX uses `dot_geo_normals > 0.5` for primary surfaces and relaxes depth sensitivity (`×0.25`) for secondary surfaces because reflection and refraction vectors carry higher positional uncertainty [Source: Q2RTX asvgf_temporal.comp, https://github.com/NVIDIA/Q2RTX].

### 7.2 Fast-Moving Object Ghosting

When an object moves faster than the denoiser's rejection thresholds, history colour "smears" across the current frame position — the classic ghosting artifact. Two mitigations:

**Neighbourhood colour clamping (AABB clamp)**: compute the colour AABB of the 3×3 neighbourhood of the current pixel in YCoCg space. Clamp the reprojected history to this AABB before blending. History values outside the current-frame neighbourhood range are pulled inward, eliminating smear at the cost of slightly increased noise for fast-moving content:

```glsl
// YCoCg neighbourhood AABB clamp
vec3 ycocg_hist = RGBtoYCoCg(history_colour);
vec3 ycocg_min  = /* 3×3 neighbourhood min in YCoCg */;
vec3 ycocg_max  = /* 3×3 neighbourhood max in YCoCg */;
ycocg_hist = clamp(ycocg_hist, ycocg_min, ycocg_max);
history_colour = YCoCgtoRGB(ycocg_hist);
```

**Velocity-based history weight reduction**: pixels with high motion-vector magnitude receive a lower history weight (higher effective alpha). A common heuristic: `alpha_final = mix(alpha, 0.5, saturate(motion_magnitude * k))`.

### 7.3 Confidence and Validity Mask

A per-pixel confidence value (0–1 float or quantised to uint8) tracks how many valid frames have accumulated. On disocclusion, confidence resets to 0 and ramps back up over subsequent frames. The confidence mask:
- Drives the transition from spatial-only variance estimation (low confidence) to temporal variance estimation (high confidence)
- Can be exposed to downstream TAA to suppress TAA history blending in unstable regions
- Is propagated through the denoiser pass chain and written alongside the denoised output

### 7.4 Flicker vs. Ghosting Tradeoff

Reducing ghosting (more aggressive history rejection, lower history weights) increases per-frame noise variance and flicker. The operating point is set by two parameters:

| Parameter | Effect when increased |
|---|---|
| `alpha` (temporal blend) | More current-frame weight → less ghosting, more flicker |
| Depth threshold | More rejections → less ghosting on depth edges, more noise on moving flat surfaces |
| Normal threshold | Fewer rejections for curved moving objects → more ghosting |

Practical tuning starts with the reference values in the Falcor or Q2RTX configurations and adjusts based on visual inspection of the specific scene's motion characteristics. Automated quality metrics (FLIP §11.2) can guide this tuning systematically.

---

## 8. Signal-Specific Denoising Strategies

### 8.1 Soft Shadows

Soft shadow denoising benefits from a hit-distance-guided blur radius. The penumbra width at a shadow receiver is proportional to the distance between receiver and blocker and inversely proportional to the light size. Storing the average hit distance to the shadow-casting surface as a per-pixel auxiliary channel enables the denoiser to widen the blur in deep penumbra (large hit distance spread) and narrow it in the umbra (zero hit distance for fully-occluded pixels).

**Contact hardening**: the blur radius should approach zero for pixels in full umbra (`hit_distance = 0`) and grow smoothly into the penumbra. A simple mapping: `blur_radius = k * mean_hit_distance / scene_scale`.

NRD's SIGMA denoiser implements this strategy explicitly for shadow signals. For SVGF-style pipelines, the hit distance is packed into an auxiliary buffer and referenced in the spatial filter's weight computation.

### 8.2 Ambient Occlusion

AO is a low-frequency signal: it varies slowly across flat surfaces and sharply only at creases and contact shadows. Aggressive temporal accumulation (small alpha, long history) is safe for static scenes. The bent-normal (average unoccluded direction) must be preserved during filtering — denoising the AO scalar separately from the bent-normal vector and recombining avoids the averaging artifacts that arise from directly denoising the product.

### 8.3 Indirect Diffuse

Indirect diffuse (global illumination bounce light) is the lowest-frequency radiance signal and tolerates the most aggressive spatial blur. A temporal accumulation history length of 32–64 frames with alpha ≈ 0.03–0.05 is common, recovering quality equivalent to several hundred effective samples. The cost of over-blurring is subtle irradiance washing at material boundaries rather than visible geometric aliasing.

### 8.4 Indirect Specular

Indirect specular is challenging because its spatial frequency scales inversely with roughness: near-mirror surfaces require almost no spatial blur (the signal is view-dependent and sharp) while rough specular tolerates significant blur. A lobe-width-adaptive filter scales the spatial kernel by the BRDF specular lobe half-angle, using the roughness value from the G-buffer:

```glsl
// Roughness-adaptive spatial filter width
float ggx_alpha = roughness * roughness;  // GGX remapping
float lobe_angle = atan(ggx_alpha);        // approximate lobe half-angle
float spatial_scale = max(0.0, lobe_angle / MAX_LOBE_ANGLE);
int filter_radius = int(MAX_FILTER_RADIUS * spatial_scale);
```

ReBLUR in NRD implements specular lobe tracking by accumulating history only from pixels whose reprojected specular direction is within the BRDF lobe of the current pixel — rejecting history from wrong-angle specular paths.

### 8.5 Subsurface Scattering

SSS radiance (light transmitted through skin, wax, leaves) should be denoised on the raw sub-surface scattered signal *before* albedo multiplication. The SSS transport is a low-frequency diffusion process; its radiance varies smoothly across the surface even at material edges. Denoising the pre-albedo SSS term allows aggressive spatial blur without smearing albedo colour boundaries into the subsurface signal.

---

## 9. Neural Denoising on Linux GPU

### 9.1 OptiX Denoiser (CUDA-Only Reference)

NVIDIA's OptiX SDK [Source: https://developer.nvidia.com/optix] ships a neural denoiser that has served as the neural reference since OptiX 5 (2018). The denoiser runs as an OptiX pass and requires CUDA-capable hardware. It is not available as a standalone Vulkan component; integration requires the OptiX CUDA runtime. On Linux, OptiX requires the proprietary NVIDIA driver (not the open-source GSP path). The OptiX denoiser supports AOV (arbitrary output variable) denoising and flow-based temporal coherence, but its closed-source nature limits Linux open-source rendering pipeline use.

### 9.2 Weighted À-Trous MLP

Research has combined traditional à-trous spatial structure with neural-predicted per-pixel filter weights. Rather than using fixed edge-stopping functions, a lightweight MLP (multi-layer perceptron) runs per-tile and outputs filter weight maps that are applied by a standard à-trous pass. This gives the filter adaptivity that learned features can provide while keeping the spatial filter's cache-friendly access pattern. [Source: Bako et al., "Kernel-predicting convolutional networks for denoising Monte Carlo renderings", SIGGRAPH 2017; and follow-on work.] Shipping implementations in open-source Linux renderers are not yet widespread as of mid-2026.

*Note: the chapter outline referenced "IRCIS (Intel real-time compute image synthesis)" as a distinct Intel neural denoiser. This term could not be confirmed against Intel's public documentation or research publications as of mid-2026; it may refer to an internal project or an alternate name for OIDN's SYCL inference path. Readers should consult current Intel graphics SDK documentation for any newly public API under that name.*

### 9.3 OIDN as the Neural Path on ROCm HIP

OIDN's HIP backend enables the U-Net neural denoiser on AMD RDNA 2–4 GPUs under Linux with ROCm 6.4.2+. The ROCm path does not require CUDA; it runs natively on the AMD compute stack. Installation:

```bash
# Install OIDN with HIP support (ROCm must be installed)
pip install openimagedenoise   # Python bindings if available
# Or build from source with HIP backend enabled:
cmake -DOIDN_DEVICE_HIP=ON -DOIDN_DEVICE_CUDA=OFF ..
```
[Source: OIDN build documentation, https://github.com/OpenImageDenoise/oidn]

Quality is equivalent across backends (the same trained weights are used); throughput varies by hardware. Intel Xe hardware benefits from the SYCL path's hardware-specific kernel optimisations.

### 9.4 Performance Comparison

Approximate performance for a 1920×1080 denoising pass (single diffuse+specular combined):

| Method | Hardware | Approximate time | Notes |
|---|---|---|---|
| À-Trous (4 levels, no temporal) | RX 7900 XTX | ~1.0 ms | No temporal history |
| SVGF (4 atrous levels + temporal) | RX 7900 XTX | ~3–5 ms | Falcor reference config |
| NRD ReBLUR | RTX 4080 | 2.55 ms | At 1440p [NRD SDK] |
| NRD ReLAX | RTX 4080 | 3.25 ms | At 1440p [NRD SDK] |
| OIDN RT (neural, balanced) | RTX 4080 | ~8–15 ms | Higher quality, no temporal |
| OIDN RT (neural, fast) | RX 7900 XTX (HIP) | ~12–20 ms | ROCm path |

*Non-NRD figures are approximations from publicly reported benchmarks; verify against your target hardware.*

---

## 10. Integration with Vulkan Render Graphs

### 10.1 Render Pass Dependency and Barrier Chain

The denoiser sits downstream of the ray tracing pass and upstream of tonemapping. The barrier chain:

```cpp
// After ray trace: transition noisy colour + G-buffer to SHADER_READ
VkImageMemoryBarrier2 barriers[] = {
    {
        .sType         = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER_2,
        .srcStageMask  = VK_PIPELINE_STAGE_2_RAY_TRACING_SHADER_BIT_KHR,
        .srcAccessMask = VK_ACCESS_2_SHADER_WRITE_BIT,
        .dstStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
        .dstAccessMask = VK_ACCESS_2_SHADER_READ_BIT,
        .oldLayout     = VK_IMAGE_LAYOUT_GENERAL,
        .newLayout     = VK_IMAGE_LAYOUT_GENERAL,
        .image         = noisy_radiance_image,
        // subresourceRange ...
    },
    // Repeat for albedo, normals, depth, motion vectors
};

VkDependencyInfo depInfo = {
    .sType                   = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .imageMemoryBarrierCount = ARRAY_SIZE(barriers),
    .pImageMemoryBarriers    = barriers,
};
vkCmdPipelineBarrier2(cmd, &depInfo);
```

Each à-trous level requires a barrier between successive compute dispatches because each level reads the output of the previous.

### 10.2 Descriptor Set Layout for Denoiser Compute Shaders

A minimal descriptor set for a single denoiser compute pass:

```cpp
VkDescriptorSetLayoutBinding bindings[] = {
    {0, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 1,
     VK_SHADER_STAGE_COMPUTE_BIT},  // noisy radiance (read)
    {1, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 1,
     VK_SHADER_STAGE_COMPUTE_BIT},  // albedo (read)
    {2, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 1,
     VK_SHADER_STAGE_COMPUTE_BIT},  // normals (read)
    {3, VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 1,
     VK_SHADER_STAGE_COMPUTE_BIT},  // depth (read)
    {4, VK_DESCRIPTOR_TYPE_STORAGE_IMAGE, 1,
     VK_SHADER_STAGE_COMPUTE_BIT},  // denoised output (write)
    {5, VK_DESCRIPTOR_TYPE_STORAGE_IMAGE, 1,
     VK_SHADER_STAGE_COMPUTE_BIT},  // history buffer (read/write)
    {6, VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, 1,
     VK_SHADER_STAGE_COMPUTE_BIT},  // per-frame settings UBO
};
```

NRD's NRI integration layer handles descriptor management internally and only exposes the set of `VkCommandBuffer` dispatches to the application.

### 10.3 Timeline Semaphore Chaining

For async compute overlap between the ray trace queue and denoiser queue:

```cpp
// Signal from graphics queue (ray trace complete)
VkSemaphoreSubmitInfo signal_info = {
    .semaphore = timeline_semaphore,
    .value     = frame_index,          // frame N
    .stageMask = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT,
};

// Wait on compute queue (before denoiser dispatch)
VkSemaphoreSubmitInfo wait_info = {
    .semaphore = timeline_semaphore,
    .value     = frame_index,          // wait for frame N from RT
    .stageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
};
```

On hardware with separate async compute queues (AMD RDNA, NVIDIA Ampere+), the denoiser can partially overlap with the subsequent frame's geometry and rasterisation work.

### 10.4 Double-Buffering History Buffers

Temporal accumulation requires two history buffers in a ping-pong configuration: the denoiser reads from the previous frame's history and writes to the current frame's history:

```
Frame N-1: write → history_buffer[0]
Frame N:   read  ← history_buffer[0], write → history_buffer[1]
Frame N+1: read  ← history_buffer[1], write → history_buffer[0]
```

Each history buffer must be a `VkImage` in `VK_IMAGE_LAYOUT_GENERAL` (storage image for write, sampled image for read). At minimum, the following channels need double-buffering: filtered radiance, moments (m1, m2), confidence/history length, and optionally the specular lobe direction for ReBLUR-style tracking.

---

## 11. Profiling and Tuning

### 11.1 GPU Timestamp Queries

Per-pass timing with `VK_EXT_calibrated_timestamps`:

```cpp
// At command buffer record time:
vkCmdWriteTimestamp2(cmd,
    VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT,
    query_pool, QUERY_DENOISER_BEGIN);

// ... dispatch denoiser passes ...

vkCmdWriteTimestamp2(cmd,
    VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT,
    query_pool, QUERY_DENOISER_END);

// After frame: read results
uint64_t timestamps[2];
vkGetQueryPoolResults(device, query_pool,
    QUERY_DENOISER_BEGIN, 2, sizeof(timestamps),
    timestamps, sizeof(uint64_t),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);

float ms = (timestamps[1] - timestamps[0])
           * timestamp_period_ns / 1e6f;
```

Wrap each à-trous level in its own timestamp pair to identify which filter level dominates. The temporal accumulation pass is typically 0.3–0.8 ms; individual à-trous levels at full resolution are 0.5–1.5 ms each.

### 11.2 Quality Metrics

**SSIM (Structural Similarity Index)**: measures luminance, contrast, and structural similarity between the denoised output and a high-spp reference render. SVGF achieves 5–47% SSIM improvement over unfiltered 1-spp input [Source: Schied et al. 2017].

**FLIP (Perceptual Difference Metric)**: a metric from Andersson et al. [Source: Andersson et al., "FLIP: A Difference Evaluator for Alternatively Rendered Images", HPG 2020, https://github.com/NVIDIAGameWorks/FLIP] that models human visual perception at a given display resolution and viewing distance. FLIP is better calibrated to perceptual quality than SSIM for HDR content and is the preferred metric for denoiser evaluation in modern rendering research. The open-source FLIP tool runs on CPU and can be integrated into automated regression testing.

### 11.3 Sample-Count Sweep vs. Quality

A systematic tuning methodology: fix the denoiser configuration, vary the spp budget (0.5, 1, 2, 4 spp), and plot FLIP error against spp. This curve determines the minimum spp that meets the quality target and identifies whether the bottleneck is denoiser quality or sample count. For most scenes, SVGF-class denoisers saturate quality at 2–4 spp; adding more samples produces diminishing returns compared to spending the ray budget on importance sampling improvements.

### 11.4 Vendor-Specific Tuning: RDNA Wave32

AMD RDNA 2/3 GPUs execute compute shaders in **Wave32** mode by default (32 lanes per wavefront). An à-trous compute shader with a 16×16 thread group (256 threads) launches 8 wavefronts. The 9-sample bilateral kernel requires careful register allocation to avoid spilling to VRAM — each sample requires 4-component radiance plus depth and normal, totalling ~10 VGPRs per sample plus temporaries. Exceeding 32 VGPRs per lane halves occupancy.

The two primary tools for diagnosing this on Linux:

- **Radeon GPU Profiler (RGP)**: capture a frame via the Radeon Developer Panel or `RadeonDeveloperServiceCLI` on Linux, then load the `.rgp` file in the RGP GUI. The "Pipeline State" view shows wave occupancy and VGPR usage per shader stage. [Source: https://gpuopen.com/rgp]
- **Radeon GPU Analyzer (RGA)**: compile the denoiser SPIR-V offline using the `rga` CLI with the Vulkan offline backend, which emits the RDNA ISA and a VGPR/SGPR resource report. Consult the RGA documentation for the exact flags for offline SPIR-V compilation. [Source: https://gpuopen.com/rga]

Strategies: use 8×8 thread groups to halve register pressure per dispatch, or split the kernel into a horizontal pass followed by a vertical pass (at the cost of correctness for non-separable bilateral weights).

---

## 12. Open-Source Implementations and References

### 12.1 Primary Papers

- **SVGF**: Schied et al., "Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination", SIGGRAPH 2017. [Source: https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced]
- **ASVGF**: Schied et al., "Gradient Estimation for Real-Time Adaptive Temporal Filtering", HPG 2018. Reference implementation in Q2RTX.
- **FLIP metric**: Andersson et al., "FLIP: A Difference Evaluator for Alternatively Rendered Images", HPG 2020. [Source: https://github.com/NVIDIAGameWorks/FLIP]

### 12.2 Production SDK References

- **NRD SDK** (ReBLUR, ReLAX, SIGMA): https://github.com/NVIDIAGameWorks/RayTracingDenoiser — includes HLSL/SPIRV compute shaders, NRI integration layer, and performance documentation
- **OIDN** (U-Net neural denoiser): https://github.com/OpenImageDenoise/oidn — SYCL, HIP, and CUDA GPU backends, Vulkan external memory interop

### 12.3 Research Renderers with Reference Implementations

- **Falcor** (NVIDIA research renderer): https://github.com/NVIDIAGameWorks/Falcor — contains `SVGFPass` with the canonical SVGF Falcor implementation in `Source/RenderPasses/SVGFPass/`; tunable `Alpha`, `MomentsAlpha`, `PhiColor`, `PhiNormal`, and `FilterIterations` parameters
- **Q2RTX** (Quake 2 path-traced port): https://github.com/NVIDIA/Q2RTX — production ASVGF implementation in `src/refresh/vkpt/shader/`; shaders include temporal accumulation, gradient reprojection, and à-trous passes with actual GLSL edge-stopping functions
- **Vulkan Samples** (Khronos Group): https://github.com/KhronosGroup/Vulkan-Samples — contains ray tracing samples with denoising post-processing demonstrating VkImageMemoryBarrier2 and timeline semaphore patterns

### 12.4 Supplementary Tools

- **RGP (Radeon GPU Profiler)**: https://gpuopen.com/rgp — GPU wavefront occupancy and timestamp analysis for RDNA targets
- **Nsight Graphics**: https://developer.nvidia.com/nsight-graphics — frame debugging and compute shader profiling for CUDA/Vulkan on NVIDIA hardware under Linux

---

## 13. Integrations

- **Ch206 — Shader Raytracing and Procedural**: The noisy Monte Carlo estimator output from the path-tracing pass (1–4 spp) is the primary input to every denoiser described in this chapter. G-buffer contents (albedo, normals, depth) produced by the ray-tracing G-buffer pass feed the denoiser's edge-stopping functions.

- **Ch207 — Shader VFX and Post-Process Compute**: Temporal anti-aliasing and the denoiser share the same history buffer infrastructure (ping-pong VkImage pairs, motion vector reprojection, exponential moving average blending). The neighbourhood clamping (AABB in YCoCg) used by TAA to prevent ghosting is structurally identical to ReLAX's history clamping. When TAA follows the denoiser in the frame graph, the two systems must coordinate to avoid double-ghosting artifacts.

- **Ch221 — GPU Algorithm Performance**: À-trous wavelet passes are memory-bandwidth-bound at small step sizes and transition toward compute-bound at larger step sizes as the sample reuse ratio drops. The roofline analysis from Ch221 directly applies to diagnosing denoiser pass bottlenecks; occupancy analysis and Wave32 tuning strategies apply to RDNA hardware running SVGF or standalone à-trous shaders.

- **Ch229 — GPU ML Inference Algorithms**: OIDN's U-Net inference is a specialised instance of the convolutional neural network inference workload covered in Ch229. The GPU kernel types (convolution, element-wise, upsampling) and the memory management strategies (activation reuse, weight caching) follow the same patterns. The HIP and SYCL backend selection for OIDN parallels the runtime selection strategies for general ML inference on Linux.
