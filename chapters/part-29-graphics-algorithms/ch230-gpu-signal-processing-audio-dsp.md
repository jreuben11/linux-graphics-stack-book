# Chapter 230: GPU Signal Processing and Audio DSP

*Part XXIX — Graphics Algorithms*

**Audiences:** Systems developers building GPU-accelerated audio or signal processing pipelines on
Linux; graphics application developers integrating audio spatialization or spectral analysis into
GPU workloads; multimedia engineers extending GStreamer and PipeWire pipelines with GPU DSP.

This chapter covers the mathematical and computational techniques for executing digital signal
processing workloads on the GPU — with particular emphasis on the audio domain. It bridges the
Linux audio entry points (PipeWire, JACK) with Vulkan and ROCm compute primitives, and surveys the
production libraries (rocFFT, VkFFT, cupyx.scipy.signal) that implement these techniques in practice. Chapter
220 covered 2D spatial-domain image filtering; this chapter treats 1D and nD signals in both
temporal and spectral domains, advancing from fundamental FFT strategies through FIR and IIR
filter implementations, wavelet transforms, filter banks, HRTF-based audio spatialization, and the
real-time pipeline integration patterns that tie GPU DSP output back into the Linux graphics and
multimedia stack.

---

## Table of Contents

1. [Signal Processing on the GPU](#1-signal-processing-on-the-gpu)
2. [FFT on the GPU](#2-fft-on-the-gpu)
3. [Overlap-Add and Overlap-Save Convolution](#3-overlap-add-and-overlap-save-convolution)
4. [FIR Filter Design and GPU Implementation](#4-fir-filter-design-and-gpu-implementation)
5. [IIR Filters on GPU](#5-iir-filters-on-gpu)
6. [Discrete Wavelet Transform](#6-discrete-wavelet-transform)
7. [Filter Banks and Subband Processing](#7-filter-banks-and-subband-processing)
8. [Audio Spatialization and HRTF](#8-audio-spatialization-and-hrtf)
9. [Spectral Audio Analysis](#9-spectral-audio-analysis)
10. [Real-Time Audio GPU Processing Pipeline on Linux](#10-real-time-audio-gpu-processing-pipeline-on-linux)
11. [Production Libraries](#11-production-libraries)
12. [Integration with the Graphics Pipeline](#12-integration-with-the-graphics-pipeline)
13. [Integrations](#integrations)

---

## 1. Signal Processing on the GPU

### 1.1 Signal Dimensionality and Domain

A *signal* is any discrete sequence of samples indexed along one or more dimensions. Audio is the
canonical 1D case: a sequence x[n] sampled at rate f_s (44100 Hz, 48000 Hz, or 96000 Hz are the
Linux-standard values from ALSA and PipeWire). Images are 2D signals. Volumetric sensor data
(sonar, LIDAR point-cloud projections, 3D MRI) are 3D. GPU compute shaders address all of these
through the same workgroup dispatch model; the signal dimensionality maps directly to the dispatch
dimension. For audio, a single 1D dispatch with `gl_GlobalInvocationID.x` indexing the sample
timeline suffices. For images the 2D spatial indices dominate (Chapter 220). For PDE solvers
operating on 3D grids a 3D dispatch is natural.

Signal processing algorithms divide by *domain*:

- **Temporal domain** (or spatial domain for images): operations computed directly on the
  x[n] sequence — convolution with a finite impulse response, sample-by-sample recursion (IIR),
  envelope following, peak detection.
- **Spectral domain**: the signal is first transformed to frequency (typically via the FFT),
  operations are performed on complex coefficients X[k], and the result is optionally
  transformed back. Spectral-domain operations include equalisation (multiply by a frequency
  response), pitch shifting (phase-vocoder), and convolution reverb (multiply by H[k]).
- **Joint time-frequency domain**: the Short-Time Fourier Transform (STFT) produces a matrix
  X[t, k] indexing both frame time and frequency bin simultaneously, enabling analysis of how
  spectra evolve over time.

### 1.2 Arithmetic Intensity of DSP Kernels

Most DSP kernels are *memory-bandwidth-limited* when operating on a single audio channel: the
computation per sample is modest (a handful of multiply-accumulate operations for a short FIR),
and the sample array must be loaded from and stored to DRAM. On an AMD Radeon RX 7900 XTX the
ridge point is approximately 64 FLOPs/byte (Chapter 221); a 64-tap FIR performing 128 FP32 MACs
per sample moves at minimum 8 bytes per sample (load + store), yielding about 16 FLOPs/byte —
well below the ridge point, so DRAM bandwidth is the bottleneck.

The arithmetic intensity rises dramatically under two conditions that make GPU acceleration
compelling:

1. **Large batch processing.** Processing B independent audio channels simultaneously. The DRAM
   traffic grows as O(B), while the workgroup-to-wavefront occupancy allows the GPU's memory
   subsystem to hide latency. At B ≈ 256 channels the effective utilisation of a discrete GPU
   becomes practical; professional audio gear with 512–1024 channels is a natural GPU workload.

2. **FFT-based convolution.** The per-sample FLOP count of FFT-domain multiplication is
   O(log N) rather than O(M) for a length-M filter, changing the arithmetic intensity
   substantially for large impulse responses. A convolution reverb with a 4-second room impulse
   response at 48 kHz has M ≈ 192,000 taps; FFT convolution replaces O(M) real MACs per sample
   with O(log N) complex operations — crossing the ridge point into compute-bound territory.

### 1.3 PipeWire and JACK as Linux Audio Entry Points

**PipeWire** ([https://pipewire.org](https://pipewire.org),
[https://gitlab.freedesktop.org/pipewire/pipewire](https://gitlab.freedesktop.org/pipewire/pipewire))
is the session manager and media routing daemon that has replaced PulseAudio and JACK for most
Linux desktop installations as of 2023. Its processing model is a *directed graph* of `spa_node`
objects connected through typed ports. Each graph iteration is driven by a *quantum* (the hardware
period, typically 128–1024 samples at 48 kHz). The `spa_node_methods::process()` callback is
invoked once per quantum for every node in dependency order. Inserting a GPU-based DSP node into a
PipeWire graph means implementing `struct spa_node_methods` with a `process()` function that
submits Vulkan or OpenCL work, waits on a timeline semaphore, and returns within the quantum
deadline. [Source: PipeWire SPA documentation,
https://docs.pipewire.org/group__spa__node.html]

**JACK2** ([https://github.com/jackaudio/jack2](https://github.com/jackaudio/jack2)) takes a
simpler contract: `jack_process_callback_t` is called once per period with pre-allocated float
buffers for all inputs and outputs. The period size (set at JACK startup with `-p`) directly
controls latency: 64 samples at 48 kHz is 1.3 ms; 1024 samples is 21.3 ms. GPU dispatch overhead
of 0.5–2 ms is non-trivial at short periods; GPU DSP therefore targets latency budgets of at least
one 256-sample period (5.3 ms at 48 kHz) or operates in an offline/batch mode. JACK's net backend
(`jack_net`) extends the period boundary across a network, which is compatible with the
GPU-dispatch latency.

---

## 2. FFT on the GPU

### 2.1 Algorithm Families

The Discrete Fourier Transform (DFT) of a length-N sequence has O(N²) direct complexity. The Fast
Fourier Transform (FFT) reduces this to O(N log N) through recursive factorisation of the DFT
matrix. Three factorisation strategies dominate GPU implementations:

**Radix-2 Cooley-Tukey.** Requires N = 2^m. The DFT of size N is split into two DFTs of size
N/2 combined via *butterfly* operations. Each butterfly computes:
```
A' = A + W * B
B' = A - W * B
```
where W = e^(−2πi k/N) is a *twiddle factor* that rotates B into the correct phase before
combination. Recursion proceeds for log₂N stages. The algorithm is memory-efficient and maps
well to GPU shared memory when the entire transform fits (N ≤ 4096 for float32 on typical GPUs
with 64 KB of LDS/shared memory).

**Radix-4.** Requires N = 4^m. Each butterfly combines four elements in a single stage, reducing
the total number of stages from log₂N to log₄N = (log₂N)/2. This halves the number of twiddle
factor memory loads, which is the primary bottleneck for large transforms. Radix-4 is the preferred
radix for GPU implementations when N is a power of 4.

**Mixed-radix (Bluestein / prime-factor).** Arbitrary N is handled by factoring N into smaller
sub-transforms (radix-2, radix-3, radix-5, or radix-7 stages). The Bluestein algorithm ("chirp-Z")
converts an arbitrary-length DFT into a convolution of power-of-2 length. Production libraries
(rocFFT, VkFFT) select the optimal decomposition automatically based on the plan size.
[Source: Cooley & Tukey 1965; Smith, "The Scientist and Engineer's Guide to Digital Signal
Processing", https://www.dspguide.com/ch12.htm]

### 2.2 Decimation in Time vs. Decimation in Frequency

The Cooley-Tukey factorisation has two dual forms:

**Decimation in Time (DIT).** Input is bit-reversed; output is in natural order. The twiddle
factors multiply the *lower* input of each butterfly before the butterfly combination. This is the
most common form because the output can be written directly to the destination array in natural
order.

**Decimation in Frequency (DIF).** Input is in natural order; output must be bit-reversed. The
twiddle factors multiply the *lower* output of each butterfly after combination. On GPU the
bit-reversal permutation is a gather operation that is relatively expensive; both DIT and DIF
therefore try to absorb the permutation into adjacent memory-access patterns rather than executing
it as a separate pass. VkFFT uses a pull-based approach where each output thread computes its
input address via a bit-reversal lookup table, avoiding an explicit scatter.

### 2.3 Multi-Batch 1D FFT

The most common GPU audio use case is a *batch* of many independent FFTs of the same size: one
per audio channel, per frame, per sensor, etc. Both rocFFT and VkFFT expose this as a plan
dimension.

**rocFFT** (AMD ROCm FFT library,
[https://github.com/ROCmSoftwarePlatform/rocFFT](https://github.com/ROCmSoftwarePlatform/rocFFT)):

```cpp
#include <rocfft/rocfft.h>

// Plan for 512 independent FFTs of length 4096, complex float
size_t lengths[1] = {4096};
rocfft_plan plan = nullptr;

rocfft_plan_create(
    &plan,
    rocfft_placement_inplace,       // in-place or rocfft_placement_notinplace
    rocfft_transform_type_complex_forward,
    rocfft_precision_single,
    1,         // number of dimensions
    lengths,   // length array
    512,       // batch count
    nullptr    // description (optional, for strides/offsets)
);

// Execute: device pointer to 512 × 4096 complex<float> contiguous array
rocfft_execute(plan, (void **)&device_ptr, nullptr, nullptr);

rocfft_plan_destroy(plan);
```

Non-contiguous batches, custom strides, and multiple GPUs are supported via
`rocfft_plan_description_set_data_layout()`. Real-to-complex and complex-to-real transforms
use `rocfft_transform_type_real_forward` / `rocfft_transform_type_real_inverse`.
[Source: rocFFT documentation, https://rocfft.readthedocs.io/en/latest/api.html]

**VkFFT** (Vulkan/OpenCL/CUDA/HIP/Metal,
[https://github.com/DTolm/VkFFT](https://github.com/DTolm/VkFFT)):

```cpp
#include "vkFFT.h"

VkFFTApplication app = {};
VkFFTConfiguration config = {};
config.FFTdim           = 1;
config.size[0]          = 4096;
config.numberBatches    = 512;
config.device           = &vkDevice;          // VkDevice
config.queue            = &vkQueue;
config.fence            = &fence;
config.commandPool      = &commandPool;
config.physicalDevice   = &physicalDevice;
config.isCompilerInitialized = 1;

// Attach buffer
config.buffer           = &vkBuffer;
config.bufferSize       = &bufferBytes;

initializeVkFFT(&app, config);

// Launch forward FFT
VkFFTLaunchParams launchParams = {};
launchParams.commandBuffer = cmdBuffer;
VkFFTAppend(&app, -1, &launchParams);   // -1 = forward; 1 = inverse
```

VkFFT compiles optimised SPIR-V at plan creation time for the target device, achieving
near-library-optimised throughput from a single header-only dependency.
[Source: VkFFT GitHub README, https://github.com/DTolm/VkFFT]

### 2.4 2D and 3D FFT

A 2D FFT of an M×N image decomposes into M independent row FFTs followed by N independent column
FFTs (separability of the 2D DFT). The GPU batch model applies directly: the M row FFTs form one
batch launch; the N column FFTs form a second. For a 4096×4096 complex image at 32 bits/element
the total data volume is 128 MB — exceeding L2 cache on most GPUs — so the column pass will
encounter more cache misses than the row pass (column stride = M × sizeof(complex) bytes; row
stride = sizeof(complex)).

rocFFT and VkFFT both support native 2D transforms where they internally manage the row/column
batching and may fuse passes to improve cache utilisation. The 2D real FFT of an M×N real image
produces an M×(N/2+1) complex half-spectrum exploiting conjugate symmetry; the full spectrum can
be reconstructed for the inverse transform.

### 2.5 Real-to-Complex FFT and Conjugate Symmetry

For a real input sequence x[n], the DFT satisfies X[N−k] = X[k]* (conjugate symmetry). This means
the full complex spectrum carries only N/2+1 independent complex values. Real-to-complex FFTs
exploit this to:
- Store only the half-spectrum (saving memory)
- Halve the arithmetic by operating on pairs of real signals packed as a single complex signal

In rocFFT, passing `rocfft_transform_type_real_forward` automatically produces an output buffer of
size N/2+1 complex elements per batch slot. The corresponding inverse transform
(`rocfft_transform_type_real_inverse`) accepts the half-spectrum and produces real output.

### 2.6 Twiddle Factor Precomputation and FFT Size Selection

Twiddle factors W_N^k = e^(−2πi k/N) for k = 0, …, N/2−1 are typically precomputed at plan
creation time and stored in a device-side lookup table read by each butterfly stage. For large N the
twiddle table is DRAM-resident and bandwidth-bound; for small N it fits in L1/shared memory.

FFT size selection matters significantly. Power-of-2 sizes are always efficient. Sizes with small
prime factors (e.g. 2^a × 3^b × 5^c) are handled well by mixed-radix decomposition. Arbitrary
sizes (primes) may force Bluestein's algorithm which is approximately 2–3× slower due to the
convolution overhead. In audio contexts: 48000 Hz × 0.1 s = 4800 samples, factored as
2^5 × 3 × 5^2 = 4800, which is handled efficiently. For spectral analysis prefer power-of-2 FFT
sizes (e.g. 4096 or 8192) and zero-pad the input signal as needed.

---

## 3. Overlap-Add and Overlap-Save Convolution

### 3.1 Linear vs. Circular Convolution

Direct application of the FFT to convolution has a subtlety: the FFT-based product of two
length-N spectra corresponds to *circular* convolution of period N, not *linear* convolution. If
x has length L and h has length M, their linear convolution has length L+M−1. To avoid circular
aliasing, the DFT size must be at least N ≥ L+M−1, and both sequences must be zero-padded to
length N. This constraint drives the blocking strategy for convolution engines.

### 3.2 Overlap-Add Algorithm

The **Overlap-Add (OLA)** method partitions the input into non-overlapping blocks of length L,
convolves each block with the filter h[n] of length M in the frequency domain using an FFT of
size N = L+M−1 (rounded up to the next efficient FFT size), and adds the overlapping tails of
successive output blocks:

```
for each block b:
    x_b = zero-pad(input[b*L : (b+1)*L], N)   // pad to length N
    X_b = FFT(x_b)
    Y_b = X_b * H                               // H = FFT(zero-pad(h, N))
    y_b = IFFT(Y_b)                             // length-N output
    output[b*L : b*L+N] += y_b                 // overlap-add tail
```

The output buffer must hold N−L = M−1 samples of tail from each block. OLA produces correct linear
convolution with latency of one block period (L samples), which is acceptable for batch processing
but introduces L/f_s seconds of latency in a real-time context.

### 3.3 Overlap-Save Algorithm

**Overlap-Save (OLS)** avoids the addition step by keeping M−1 samples from the *previous* input
block as a prefix of the current block, making the total block length N = L+M−1. The circular
convolution of this length-N block with H[k] produces N output samples, but the first M−1 are
corrupted by circular aliasing and are discarded; only the trailing L samples are saved:

```
x_prev = zeros(M-1)
for each block b:
    x_b = concat(x_prev, input[b*L : (b+1)*L])   // length N = L+M-1
    X_b = FFT(x_b)
    Y_b = X_b * H
    y_b = IFFT(Y_b)
    output[b*L : (b+1)*L] = y_b[M-1 : N]         // discard first M-1
    x_prev = input[(b+1)*L-M+1 : (b+1)*L]        // save tail
```

OLS has the same computational cost as OLA but avoids the write-back accumulation, making it
slightly simpler to parallelise: each output block is independent once the FFT of H is precomputed.

### 3.4 Latency vs. Throughput Tradeoff

Both OLA and OLS have block latency proportional to L. Choosing L = 64 at 48 kHz gives 1.3 ms
latency but requires more kernel launches (shorter FFTs, higher dispatch overhead per sample).
Choosing L = 4096 gives 85 ms latency but nearly saturates the FFT. For real-time convolution
reverb a latency of 10–40 ms is generally imperceptible; L = 512–2048 is a common compromise.

### 3.5 Partitioned Convolution for Long Impulse Responses

Concert hall room impulse responses last 1–4 seconds — 48,000–192,000 taps at 48 kHz. A single
FFT large enough for linear convolution of the entire IR (N ≥ 192,001) is expensive and introduces
enormous latency. **Partitioned convolution** splits the IR into B partitions of length P:

```
h = [h_0 | h_1 | ... | h_{B-1}]    // each h_i has length P
```

The output is:

```
y = sum_{i=0}^{B-1}  conv(x_delayed_by_i*P, h_i)
```

where each `conv` is computed with OLS using FFT size N = 2P. This keeps the per-block FFT size
small (P ≤ 512 is common for the *uniform partitioned* case), maintaining low latency for the
earliest partition (h_0) while amortising the later partitions — which can be processed with
longer blocks and delayed — across the frame budget. A *non-uniform* partition scheme uses short
blocks for the early part of the IR (where latency matters) and longer blocks for the reverberant
tail. [Source: Wefers, "Partitioned Convolution Algorithms for Real-Time Auralization", Logos Verlag 2015
— see https://publications.rwth-aachen.de/record/466561; note: verify availability]

---

## 4. FIR Filter Design and GPU Implementation

### 4.1 Windowed Sinc Design

The ideal lowpass filter has an impulse response of infinite length (sinc function). The windowed
sinc method truncates this to M+1 taps and applies a window w[n] to reduce spectral leakage
(Gibbs phenomenon):

```
h[n] = sinc(2 f_c / f_s * (n - M/2)) * w[n],   n = 0, ..., M
```

Common windows and their stopband attenuation (approximate):
- **Hamming** (−43 dB): w[n] = 0.54 − 0.46 cos(2πn/M)
- **Blackman** (−74 dB): w[n] = 0.42 − 0.5 cos(2πn/M) + 0.08 cos(4πn/M)
- **Kaiser** (adjustable via β): w[n] = I₀(β√(1−(2n/M−1)²)) / I₀(β), where β trades off
  transition width for stopband attenuation. β = 8.6 gives ~80 dB attenuation.

The Kaiser window is preferred for its adjustable tradeoff; scipy.signal.firwin (Python) and
remez (Parks-McClellan) implementations serve as offline design tools whose output coefficients
are then uploaded to GPU constant memory.

### 4.2 Parks-McClellan Equiripple Design

The Parks-McClellan algorithm uses the Remez exchange algorithm to compute the optimal Chebyshev
approximation to the desired frequency response. Unlike windowed sinc, it guarantees equal ripple
in both passband and stopband for a given filter length M, often achieving a shorter filter
(lower M) for the same stopband attenuation. Parks-McClellan filters are designed offline (e.g.
`scipy.signal.remez`) and loaded as constant coefficients into the GPU kernel.

### 4.3 Polyphase Decomposition for Sample Rate Conversion

Sample rate conversion by rational factor L/M (e.g. 44100→48000 Hz = 147/160) is implemented as
upsample by L, filter, downsample by M. The filter is a lowpass with cutoff at min(1/L, 1/M)/2.
The *polyphase decomposition* splits the length-N_h filter into L sub-filters:

```
E_i[k] = h[i + k*L],   k = 0, ..., N_h/L - 1,   i = 0, ..., L-1
```

For each output sample the appropriate polyphase branch E_i is selected and applied, avoiding
computation of the intermediate upsampled signal. On GPU, the L polyphase sub-filters are stored
in a 2D constant array `float poly[L][N_h/L]`. Each thread computes one output sample by reading
its polyphase branch and dotting against the input history.

### 4.4 Direct-Form GPU FIR Kernel

For a single channel the tapped delay line is stored as a ring buffer:

```glsl
// fir_compute.comp — GPU FIR for B independent channels
layout(local_size_x = 256) in;
layout(set=0, binding=0) buffer Coeffs  { float h[];  };   // M taps
layout(set=0, binding=1) buffer InBuf   { float x[];  };   // B * N samples
layout(set=0, binding=2) buffer OutBuf  { float y[];  };   // B * N samples
layout(push_constant) uniform PC {
    uint N;   // samples per channel per block
    uint M;   // filter length
    uint B;   // number of channels
} pc;

void main() {
    uint tid = gl_GlobalInvocationID.x;
    if (tid >= pc.B * pc.N) return;
    uint ch  = tid / pc.N;
    uint n   = tid % pc.N;

    float acc = 0.0;
    for (uint k = 0; k < pc.M; k++) {
        // x ring buffer: channel ch, sample (n - k), wrapped
        int   idx = int(n) - int(k);
        uint  src = ch * pc.N + uint((idx >= 0) ? idx : idx + int(pc.N));
        acc += h[k] * x[src];
    }
    y[tid] = acc;
}
```

This assigns one thread per output sample. For M = 128 and B = 256 channels at N = 512 samples
per block, the dispatch is 256×512 = 131,072 threads — adequate occupancy for a modern GPU. Each
thread performs M MAC operations (arithmetic intensity ≈ 2M / (4×3) ≈ 21 FLOPs/byte for M=128),
approaching the GPU ridge point for large M.

### 4.5 Overlap-Save FIR for Spectral Domain

For filter lengths M > 64 the FFT-domain overlap-save approach (Section 3.3) dominates. The
filter coefficients are precomputed to their half-spectrum FFT H[k] at plan time; only the input
FFT and a complex pointwise multiply are needed per block. This reduces per-sample cost from O(M)
to O(log N).

### 4.6 GPU Multichannel FIR: One Warp per Channel

A high-occupancy pattern for multichannel FIR assigns *one warp* (32 threads for NVIDIA, 64
for AMD) to process the FIR for one audio channel. The filter coefficients h[0..M-1] are loaded
into shared memory; each thread in the warp accumulates a portion of the dot product using warp-
level reductions (`subgroupAdd` in GLSL). For M = 512 this splits the coefficient dot product
into 16 warp passes of 32 elements each, keeping shared-memory bandwidth high and minimising
DRAM accesses per channel.

---

## 5. IIR Filters on GPU

### 5.1 Biquad Direct Forms

A biquad (second-order IIR section) has the transfer function:

```
H(z) = (b₀ + b₁z⁻¹ + b₂z⁻²) / (1 + a₁z⁻¹ + a₂z⁻²)
```

**Direct Form I** uses four state variables {x[n−1], x[n−2], y[n−1], y[n−2]}:

```c
y[n] = b0*x[n] + b1*x[n-1] + b2*x[n-2]
       - a1*y[n-1] - a2*y[n-2];
```

**Direct Form II transposed** (numerically preferred) uses two state variables {w₁, w₂}:

```c
y[n] = b0*x[n] + w1;
w1   = b1*x[n] - a1*y[n] + w2;
w2   = b2*x[n] - a2*y[n];
```

Direct Form II minimises coefficient sensitivity and is the standard for audio equaliser
implementations. Professional equalisers cascade 4–10 biquad sections; a parametric EQ with 10
bands uses 10 sections.

### 5.2 Data Dependency Challenge

The fundamental GPU challenge for IIR filters is *data dependency*: y[n] depends on y[n−1] and
y[n−2], so samples cannot be computed independently in parallel. A naïve single-threaded IIR
kernel wastes the GPU entirely. Three strategies exist:

1. **Parallelise across channels.** If there are B ≥ 256 independent channels each gets its own
   thread; within each channel the recursion is sequential. This is the standard approach for
   multichannel audio (B >> 1) and achieves full GPU utilisation. Each thread processes N samples
   sequentially — effectively a scalar IIR loop running in a GPU thread, occupying the thread for
   the entire block.

2. **Parallelise across cascaded sections.** A cascade of S biquad sections has S−1 stages of
   pipeline parallelism: while section s processes sample n, section s−1 can process sample n+1.
   This S-way pipeline parallelism requires S threads and is practical only for S ≥ 32 (one warp).
   In practice S ≤ 10 for audio EQ, so this strategy provides limited occupancy.

3. **Prefix-product look-ahead (parallel IIR).** The IIR recursion y[n] = a·y[n−1] + x[n] can
   be unrolled: y[n] = x[n] + a·x[n−1] + a²·x[n−2] + … + aⁿy[0]. The coefficients {1, a, a²,
   …} form a geometric series and the summation is a prefix operation computable with a parallel
   prefix scan. For a *general* second-order section the analogous state-transition matrix
   approach applies: the state vector [y[n], y[n−1]] = M * [y[n−1], y[n−2]] + [x[n], 0], where M
   is the 2×2 feedback matrix. Computing this for n=0..N−1 simultaneously requires N 2×2 matrix
   multiplications in a parallel prefix scan (O(N log N) work). This is beneficial only if N >>
   the thread synchronisation overhead and the filter must be applied to a *single* channel.
   [Source: Blelloch, "Prefix Sums and Their Applications",
   https://www.cs.cmu.edu/~guyb/papers/Ble93.pdf — note: verify exact title and year]

### 5.3 Applications: Equaliser, Crossover, Shelving

A 10-band parametric equaliser for B channels with N samples per block dispatches B threads, each
executing a sequential cascade of 10 Direct Form II biquad sections over N samples. The per-thread
arithmetic cost is 5×2×10×N = 100N FP32 MACs; at B=512, N=512 the total is 26 GMACs per block,
achievable in well under 1 ms on a modern GPU.

A loudspeaker crossover (Linkwitz-Riley 4th order = two cascaded 2nd-order Butterworth biquads per
band) applied to B speaker channels follows the same pattern. A shelving filter (bass/treble
boost/cut) uses a single biquad section with coefficients computed from the Audio EQ Cookbook
([Source: Zölzer, "DAFX: Digital Audio Effects", Wiley;
https://www.w3.org/TR/audio-eq-cookbook/]).

---

## 6. Discrete Wavelet Transform

### 6.1 Lifting Scheme

The **Lifting Scheme** ([Source: Sweldens 1995, "The lifting scheme: A new philosophy in
biorthogonal wavelet construction", SPIE]) decomposes the DWT computation into three in-place
stages:

1. **Split**: separate even-indexed samples (approximation) from odd-indexed (detail).
2. **Predict**: update odd (detail) coefficients by subtracting a prediction derived from the
   even samples. For the Haar wavelet: d[i] = x[2i+1] − x[2i].
3. **Update**: update even (approximation) coefficients using the detail coefficients. For Haar:
   a[i] = x[2i] + d[i]/2.

The lifting formulation allows in-place, integer-reversible computation and maps well to GPU
shared memory: a block of 2K input samples is loaded into shared memory, and the predict/update
steps are computed without additional global memory traffic.

### 6.2 Daubechies Wavelet Families

Daubechies D4 wavelet (4-coefficient filter, 2 vanishing moments) uses a two-step lifting
factorisation with predict/update coefficients derived from the D4 filter bank. The lifting steps
introduce more data dependencies than Haar (each predict step reads neighbours two samples apart),
but the wavelet achieves superior frequency selectivity. Daubechies D8 (4 vanishing moments) and
D16 (8 vanishing moments) use progressively longer support at the cost of wider dependency ranges.
[Source: Daubechies, "Ten Lectures on Wavelets", SIAM 1992]

For GPU implementation the dependency range must fit within the shared memory tile. A tile of W
input samples in shared memory handles a Daubechies D4 DWT step when W ≥ 2K + support_length.
The standard shared-memory tile of 256 samples per block is sufficient for D4 and D8 at a single
DWT level.

### 6.3 GPU DWT: In-Place Lifting in Shared Memory

```glsl
// dwt_haar.comp — in-place 1D Haar DWT, one level
layout(local_size_x = 128) in;
layout(set=0, binding=0) buffer Signal { float x[]; };
layout(push_constant) uniform PC { uint N; } pc;

shared float tile[256];   // 2 * local_size_x

void main() {
    uint gid = gl_GlobalInvocationID.x;
    uint lid = gl_LocalInvocationID.x;

    // Load two consecutive samples per thread
    tile[2*lid]   = (2*gid   < pc.N) ? x[2*gid]   : 0.0;
    tile[2*lid+1] = (2*gid+1 < pc.N) ? x[2*gid+1] : 0.0;
    barrier();

    // Predict (compute detail)
    float d = tile[2*lid+1] - tile[2*lid];
    // Update (compute approximation)
    float a = tile[2*lid]   + d * 0.5;

    // Write approximation to lower half, detail to upper half
    x[gid]          = a;
    x[pc.N/2 + gid] = d;
}
```

Multi-level DWT repeats the single-level transform on the approximation subband. Five levels of
a 65536-sample signal reduce the approximation to 2048 samples before reaching the lowest subband.

### 6.4 Separable 2D DWT

A 2D DWT applies the 1D transform first to all rows, then to all columns of the result — the
same separability that makes 2D FFT efficient. This produces four subbands (LL, LH, HL, HH) at
each level, matching the structure used by JPEG 2000 (which uses the biorthogonal CDF 9/7 wavelet
for lossy and CDF 5/3 for lossless compression).

The GPU row pass treats each image row as an independent 1D DWT (a batch of M transforms of
length N). The column pass is transposed: all column transforms run as a second batch. For a
4096×4096 image the column pass has poor L2 cache locality (stride-4096 access pattern); an
intermediate transpose of the 2D array before the column pass can improve throughput by 3–4×
on discrete GPUs with large L2 caches.

### 6.5 Applications: Denoising and Texture Analysis

DWT-based denoising (Donoho-Johnstone wavelet shrinkage) applies a pointwise threshold to the
detail coefficients, setting small-magnitude coefficients to zero (hard thresholding) or shrinking
them toward zero (soft thresholding). On GPU this is a single element-wise pass over the detail
subband — trivially parallelisable. Applications include audio noise suppression (DWT along the
time axis) and image denoising as used in libcamera's ISP pipelines.

---

## 7. Filter Banks and Subband Processing

### 7.1 Uniform DFT Filter Bank (WOLA)

A *Uniform DFT Filter Bank* produces K subband signals from a wideband input by applying K
bandpass filters whose centre frequencies are uniformly spaced at k/K of the sampling rate,
k = 0, …, K−1. The modulated window DFT filter bank (also called the Weighted Overlap-Add,
WOLA, analysis filter bank) implements this efficiently using the FFT: a single K-point FFT of a
windowed input block produces all K subbands simultaneously.

**Analysis:**
```
X[t, k] = FFT(w * x[t*H : t*H + K])    // windowed, hop size H
```

**Synthesis (Inverse WOLA):**
```
x_out[t*H : t*H + K] += IFFT(Y[t, :]) * w   // overlap-add
```

The hop size H determines the temporal resolution vs. computational load: H = K/2 (50% overlap)
is standard for audio and matches the Hann window perfect reconstruction condition.

### 7.2 Modulated Lapped Transform

The **Modulated Lapped Transform (MLT)**, also known as the Modified Discrete Cosine Transform
(MDCT), is the transform used in MP3, AAC, Vorbis, and Opus. It maps 2N overlapping input samples
to N spectral coefficients. The MDCT of a block of 2N samples is:

```
X[k] = sum_{n=0}^{2N-1} x[n] * cos(π/N * (n + N/2 + 1/2) * (k + 1/2))
```

On GPU, the MDCT is computed via the FFT: a pre-rotation step multiplies by complex exponentials,
an N/2-point complex FFT is applied, and a post-rotation extracts the real coefficients. This is
the strategy used in GPU-accelerated Opus encoders. [Source: Malvar, "Signal Processing with
Lapped Transforms", Artech House 1992]

### 7.3 Quadrature Mirror Filter Bank

A **QMF bank** splits a signal into two subbands using a lowpass filter H₀ and its
complementary highpass H₁ = H₀(−z), then downsamples each by 2. Two-band QMF banks are the
building block of the Discrete Wavelet Packet Transform. On GPU the two filters H₀ and H₁ are
applied simultaneously to the same input: each thread computes both outputs, halving the memory
access overhead relative to independent filter applications.

### 7.4 Polyphase Channeliser

A *polyphase channeliser* converts a wideband signal into K narrowband channels using the
polyphase decomposition of the prototype lowpass filter P(z). The channeliser applies K polyphase
sub-filters in parallel followed by a K-point IFFT. This structure — a bank of K matched filters
— is standard in software-defined radio (SDR) and is implemented in GPU DSP frameworks such as
cuDSP. [Source: Harris, "Multirate Signal Processing for Communication Systems", Prentice Hall 2004]

**GPU parallel channeliser:** one thread block per subband. Each block loads the polyphase
sub-filter coefficients into shared memory and dot-products against the corresponding input subset.
The K outputs feed an IFFT batch: one IFFT of length K per output time step. At K = 1024
subbands and an input sample rate of 20 Msps (SDR use case), the IFFT batch has 20,000/1024 ≈ 19
launches per second — trivially handled on GPU.

### 7.5 MPEG Psychoacoustic Subband

The MPEG Layer I/II psychoacoustic model divides audio into 32 uniform subbands using a 32-tap
polyphase filter bank. Each subband carries 1/32 of the bandwidth; the masking threshold is
computed per subband to determine bit allocation. The 512-point FFT of the same block feeds the
psychoacoustic model. Both operations can be launched concurrently on GPU with independent
workgroups: subbands on the compute queue; psychoacoustic FFT on a secondary async compute queue.

---

## 8. Audio Spatialization and HRTF

### 8.1 Head-Related Transfer Function

The **Head-Related Transfer Function (HRTF)** models how sound from a source at direction (θ, φ)
reaches the left and right eardrums after reflecting off the listener's head, torso, and pinnae.
Measured as a pair of impulse responses HRIR_L[n] and HRIR_R[n], typically of length 128–512
samples at 44.1 or 48 kHz, the HRTF fully characterises the binaural cues (interaural time
difference, interaural level difference, spectral notches) that the auditory system uses to
localise sound.

Binaural rendering of a single source produces:
```
y_L = conv(x, HRIR_L)    // left ear signal
y_R = conv(x, HRIR_R)    // right ear signal
```

For S simultaneous sources, 2S convolutions are required. At S = 64 sources with 256-tap HRIRs
this is 64 × 2 × 256 MACs per sample = 32,768 MACs/sample at 48 kHz → 1.6 GFLOPS sustained,
well within GPU capability but beyond CPU SIMD budgets. The HRTF database is loaded into
constant memory indexed by discretised azimuth/elevation bins; at runtime a lookup maps source
position to the nearest HRTF pair.

Common free HRTF databases: MIT KEMAR
([https://sound.media.mit.edu/resources/KEMAR.html](https://sound.media.mit.edu/resources/KEMAR.html)),
CIPIC
([https://www.ece.ucdavis.edu/cipic/](https://www.ece.ucdavis.edu/cipic/)),
and SADIE
([https://www.york.ac.uk/sadie-project/](https://www.york.ac.uk/sadie-project/)).

### 8.2 Binaural Rendering on GPU

The standard GPU binaural engine layout:
- **Input buffer:** `VkBuffer` containing S × N float32 samples (one contiguous block per source)
- **HRTF database:** `VkBuffer` containing D × 2 × M float32 HRIR coefficients (D = number of
  direction bins, 2 = ears, M = HRIR length), or pre-FFT'd as D × 2 × (M/2+1) complex32 for
  FFT convolution
- **Output buffer:** 2 × N float32 (left and right mix)

For FFT-domain convolution the pipeline per block is:
1. Batch FFT of S source blocks → S × (N/2+1) complex spectra (one FFT launch)
2. For each source: complex multiply S spectra by HRIR_L and HRIR_R spectra (S × 2 element-wise
   multiplications)
3. Accumulate S left spectra into Y_L, S right spectra into Y_R (two batch reductions)
4. IFFT Y_L and Y_R → left and right output signals

All steps are launched as Vulkan compute dispatches; the timeline semaphore gates the output
readback. For real-time use the 2 IFFT outputs feed a `VkBuffer` that is DMA-transferred to the
ALSA/PipeWire ring buffer.

**OpenAL Soft** ([https://github.com/kcat/openal-soft](https://github.com/kcat/openal-soft)),
the primary open-source OpenAL implementation on Linux, supports HRTF convolution in software
but does not have a native GPU acceleration path as of 2025. GPU binaural rendering in Linux
game engines is typically implemented at the engine level (e.g. Godot 4 spatial audio), not via
OpenAL.

### 8.3 Ambisonics

**Ambisonics** encodes a 3D sound field as a set of spherical harmonic (SH) coefficients rather
than per-listener binaural signals, allowing flexible decoding to any speaker layout or
to binaural at playback time.

**B-format** (1st order Ambisonics) uses 4 channels: W (omnidirectional), X, Y, Z (figure-of-8
in each axis). The SH encoder for a point source at unit direction vector **d** = (dx, dy, dz):

```
W = 1/√2
X = dx
Y = dy
Z = dz
```

**Higher-Order Ambisonics (HOA)** of order N uses (N+1)² channels total, providing directional
resolution of approximately 180°/N. Order 3 gives 16 channels; order 7 gives 64 channels.

**GPU SH encoder.** For S simultaneous sources, encoding to order-N Ambisonics requires
S × (N+1)² coefficient multiplications per sample. At S = 512, N = 5 (36 channels = 4608 ops/
sample), the SH encoder is the dominant cost. A GPU compute shader with one thread per
(source, channel) pair handles 512 × 36 = 18,432 threads, fitting comfortably in a single
dispatch.

**GPU Ambisonic rotation.** Rotating the sound field by a quaternion q (for head-tracking) maps
to a (N+1)²×(N+1)² block-diagonal rotation matrix in SH space. Computing the rotation matrix
coefficients and multiplying all HOA channels per sample is a natural GPU matrix-vector operation.
[Source: Rafaely, "Fundamentals of Spherical Array Processing", Springer 2015]

**Ambisonic decoder GPU shader.** Decoding from HOA to L loudspeakers applies a D matrix:

```glsl
// ambisonic_decode.comp — order-N HOA to L loudspeakers
layout(set=0, binding=0) readonly buffer HOA      { float hoa[];  };  // (N+1)^2 x T
layout(set=0, binding=1) readonly buffer DMatrix  { float D[];    };  // L x (N+1)^2
layout(set=0, binding=2) writeonly buffer Speakers { float spk[]; };  // L x T
layout(push_constant) uniform PC { uint T; uint C; uint L; } pc;      // T frames, C channels

void main() {
    uint sp = gl_GlobalInvocationID.x;  // speaker index
    uint t  = gl_GlobalInvocationID.y;  // time frame
    if (sp >= pc.L || t >= pc.T) return;
    float acc = 0.0;
    for (uint c = 0; c < pc.C; c++)
        acc += D[sp * pc.C + c] * hoa[c * pc.T + t];
    spk[sp * pc.T + t] = acc;
}
```

### 8.4 Room Acoustics: Image Source Method and Reverb Tail

The **Image Source Method** (ISM) models room reflections by placing virtual mirror-image sources
for each reflection order. For a rectangular room with reflection order R, the number of image
sources is O(R³). GPU ISM batch-evaluates contributions from all image sources in parallel:
each thread handles one image source, computing its delay (distance/c), attenuation, and adding
its contribution to the output at the appropriate sample offset. For R = 5 (≈370 image sources)
and S simultaneous real sources, the total thread count is S × 370, parallelising the acoustic
simulation that was previously O(R³) per source on CPU.

The late reverb tail (beyond the early reflections modelled by ISM) is conventionally handled
with statistical reverb: a convolution with a stochastic impulse response drawn from an
exponentially decaying noise model. GPU overlap-save convolution (Section 3.3) applies this
tail efficiently without the explicit O(M) cost.

---

## 9. Spectral Audio Analysis

### 9.1 Short-Time Fourier Transform

The **Short-Time Fourier Transform (STFT)** applies the FFT to successive windowed segments of
a signal, producing a time-frequency representation X[t, k]:

```
X[t, k] = sum_{n=0}^{N-1} x[t*H + n] * w[n] * e^{-2πi k n / N}
```

where w[n] is the analysis window (typically Hann), H is the hop size, and N is the FFT size.
For a 10-second audio segment at 48 kHz, N = 2048, H = 512 (75% overlap), the STFT produces
10×48000/512 ≈ 937 frames × 1025 frequency bins = a 937×1025 complex matrix.

On GPU, all T frame FFTs are a single batch launch of T FFTs of size N. The window multiplication
is fused into the batch: each FFT's input is formed by element-wise multiplication of the input
slice with the Hann window coefficients stored in a small constant buffer. VkFFT's
`config.performR2C = 1` produces the half-spectrum directly.

### 9.2 Mel Filterbank

The **Mel filterbank** applies M triangular filters to the STFT magnitude spectrum, spaced
uniformly on the Mel frequency scale:

```
mel(f) = 2595 * log10(1 + f/700)
```

The filterbank is precomputed as an M × (N/2+1) sparse matrix F; the Mel spectrum is:

```
S_mel[t, m] = sum_k F[m, k] * |X[t, k]|
```

On GPU this is a sparse matrix-vector multiplication (for one frame) or a batched SpMV (for all
T frames simultaneously). Because F is sparse and fixed, it is best stored in CSR format for
SSBO access. Alternatively, for M = 80 and N/2+1 = 1025 the dense matrix F fits in 320 KB and
can be stored as a plain 2D buffer for dense `matmul`: the T × 1025 magnitude matrix times the
1025 × M filterbank matrix, which maps to a GEMM call.

### 9.3 MFCC Extraction Pipeline on GPU

Mel-Frequency Cepstral Coefficients (MFCCs) are the standard features for audio classification:

```
STFT → |·| → Mel filterbank → log(·) → DCT → MFCC
```

The full GPU pipeline:
1. **STFT batch FFT**: T FFTs of size N → T × (N/2+1) complex (VkFFT or rocFFT)
2. **Magnitude**: element-wise `sqrt(re² + im²)` → T × (N/2+1) real (one compute dispatch)
3. **Mel multiply**: GEMM of T × 1025 times 1025 × M → T × M (rocBLAS SGEMM or Vulkan matmul)
4. **Log**: element-wise `log(·)` on T × M real matrix
5. **DCT**: M-point DCT-II per frame, computed via a radix-2 FFT with pre/post rotation
6. **Output**: T × K MFCC matrix (typically K = 13 coefficients retained)

All six stages run on the same command buffer with `vkCmdPipelineBarrier` between stages that
have write-after-read hazards. The total data volume is dominated by the STFT output (2 × T ×
(N/2+1) × 4 bytes = 2 × 937 × 1025 × 4 ≈ 7.6 MB for a 10-second clip), comfortably fitting in
GPU L2 cache on discrete hardware.

### 9.4 Chroma Features and CQT

**Chroma features** (Pitch Class Profile, PCP) represent the distribution of energy across the 12
pitch classes of the chromatic scale, independent of octave. They are computed from the Constant-Q
Transform (CQT) rather than the linear-frequency DFT:

```
X_CQT[b] = sum_n x[n] * w_b[n] * e^{-2πi Q n / N_b}
```

where the frequency resolution Q/N_b is constant in octave-relative terms. The CQT at octave b
has window length N_b = Q * f_s / f_b. On GPU the multi-resolution CQT is implemented as a
multi-resolution FFT: FFTs of different sizes for different octave bands, or via the efficient
sliding-window CQT of [Source: Schörkhuber & Klapuri, "Constant-Q Transform Toolbox for Music
Processing", SMC 2010]. Each octave requires a half-length input (downsampled by 2) and a smaller
FFT, creating a cascade of batch FFT launches of decreasing size.

### 9.5 Onset Detection: Spectral Flux

Onset detection marks the start of note or percussion events. The **Spectral Flux** measure
computes the positive-only change in log magnitude spectrum between adjacent STFT frames:

```
SF[t] = sum_k max(0, log|X[t,k]| - log|X[t-1,k]|)
```

On GPU this is a parallel reduction over frequency bins for each frame pair. The input is the
T × (N/2+1) log-magnitude matrix (already computed for MFCC). A 2D dispatch with one thread per
(frame, bin) pair computes the differences; a subsequent reduction per frame collapses bins to a
scalar SF[t] per frame.

### 9.6 Pitch Detection: YIN via Autocorrelation FFT

The **YIN algorithm** ([Source: de Cheveigné & Kawahara, JASA 2002,
https://doi.org/10.1121/1.1458024]) computes pitch from the autocorrelation function:

```
r[τ] = sum_{j=0}^{W-1} x[j] * x[j+τ]
```

via the FFT: r = IFFT(|FFT(x)|²), which avoids the O(W²) direct cost. The difference function
d[τ] = r[0] + r[0] − 2r[τ] is then computed per lag τ and normalised. The pitch period is the
lag τ_min corresponding to the first local minimum of the cumulative mean normalised difference
below a threshold (0.1 is typical).

GPU YIN for S pitch tracks: S parallel FFT autocorrelation launches, S parallel difference
function passes, S parallel argmin reductions. At S = 128 tracks the batch fits in one GPU
dispatch at each stage. The **McLeod Pitch Method** (MPM) is an alternative using normalised
autocorrelation that avoids the cumulative normalisation step, slightly simpler to implement
as a GPU kernel.

---

## 10. Real-Time Audio GPU Processing Pipeline on Linux

### 10.1 PipeWire Graph with a GPU Filter Node

A PipeWire filter node is implemented by instantiating an `spa_node` with `process()` performing
GPU work. The prototype structure:

```c
// Minimal spa_node implementation for a GPU DSP filter
static int impl_node_process(void *object) {
    struct gpu_node *n = object;

    // 1. Get input buffer pointer from spa_io_buffers
    struct spa_buffer *in_buf  = n->in_port->io->buffer;
    struct spa_buffer *out_buf = n->out_port->io->buffer;
    float *in_ptr  = (float *)in_buf->datas[0].data;
    float *out_ptr = (float *)out_buf->datas[0].data;
    uint32_t n_samples = in_buf->datas[0].chunk->size / sizeof(float);

    // 2. DMA: copy input to VkBuffer (or import dmabuf)
    memcpy(n->mapped_input, in_ptr, n_samples * sizeof(float));

    // 3. Submit Vulkan command buffer (pre-recorded FFT + filter pipeline)
    VkSubmitInfo2 submit = { /* ... configured with DSP command buffer */ };
    vkQueueSubmit2(n->queue, 1, &submit, n->fence);

    // 4. Wait on fence (GPU processing completes within graph quantum)
    vkWaitForFences(n->device, 1, &n->fence, VK_TRUE, quantum_ns);
    vkResetFences(n->device, 1, &n->fence);

    // 5. Copy GPU output to PipeWire output buffer
    memcpy(out_ptr, n->mapped_output, n_samples * sizeof(float));
    return SPA_STATUS_HAVE_DATA;
}
```

The quantum period is negotiated by PipeWire between the hardware source (ALSA period) and the
client graph. At 48 kHz with a 512-sample period, the GPU must complete in under 10 ms
(approximately 5× the period to leave headroom for system jitter).

Note: native GPU node support in PipeWire (using DMA-BUF import to avoid the CPU memcpy) is an
active development area as of 2025. Patches have been proposed that allow a PipeWire spa_node to
directly import and export DRM-prime DMA-BUF buffers, eliminating the CPU copy path.
[Source: PipeWire issue tracker, https://gitlab.freedesktop.org/pipewire/pipewire/-/issues]

### 10.2 Latency Budget

For real-time GPU audio with a graph period of P samples at f_s Hz, the latency budget breakdown
is:

| Component | Typical (48 kHz, P=512) |
|---|---|
| ALSA hardware period | P/f_s = 10.7 ms |
| DMA CPU→GPU upload | 0.1–0.5 ms (PCIe) |
| GPU processing (FFT convolution) | 0.2–1.0 ms |
| DMA GPU→CPU download | 0.1–0.5 ms |
| PipeWire graph scheduling jitter | ±0.5 ms |
| **Total one-way latency** | **~12–13 ms** |

Integrated GPUs (AMD APU, Intel iGPU) using shared LPDDR5 eliminate the PCIe DMA cost, reducing
the GPU roundtrip to 0.1–0.3 ms, making GPU DSP viable at 64-sample (1.3 ms) periods.

### 10.3 VkBuffer Ping-Pong for Audio Samples

Double-buffering avoids the synchronisation penalty of the CPU waiting for GPU completion before
refilling the input. Two input `VkBuffer`s alternate: while GPU processes buffer A, the CPU fills
buffer B from PipeWire; on the next period GPU processes B and CPU fills A:

```
Frame N:   GPU processes InputA → OutputA   |   CPU fills InputB
Frame N+1: GPU processes InputB → OutputB   |   CPU fills InputA
```

The GPU command buffers reference `InputA`/`OutputA` or `InputB`/`OutputB` via descriptor sets
that are updated before submission (using `vkUpdateDescriptorSets` or push descriptors).

### 10.4 Timeline Semaphore for Audio-Render Sync

When GPU audio processing and GPU graphics rendering share the same device, the audio DSP
pipeline and the graphics pipeline must be ordered. A `VkSemaphore` of type
`VK_SEMAPHORE_TYPE_TIMELINE` provides a monotonically increasing counter that both queues signal
and wait on. The audio DSP compute queue signals value T_audio after completing the audio block;
the graphics queue waits on T_audio before consuming visualisation data (e.g. the spectrogram
texture or MFCC matrix). This is the same timeline semaphore pattern described for Vulkan
synchronisation in Chapter 148.

### 10.5 AV Sync via PTS Timestamps

For multimedia applications (video + audio synchronisation), the GPU audio output carries a
Presentation Timestamp (PTS) in the `spa_buffer.datas[0].chunk->time` field (PipeWire) or as a
`jack_nframes_t` offset (JACK). The video renderer compares the current audio PTS with the video
frame PTS and adjusts the video present timing on the Wayland compositor. GPU audio processing
latency must be accounted for in the PTS adjustment: the audio PTS should lead the GPU audio
output by (GPU_processing_latency + DMA_download_latency) nanoseconds to preserve AV sync.

---

## 11. Production Libraries

### 11.1 rocFFT

**rocFFT** ([https://github.com/ROCmSoftwarePlatform/rocFFT](https://github.com/ROCmSoftwarePlatform/rocFFT))
is AMD's GPU FFT library for the ROCm compute stack. Key features:

- Supports 1D, 2D, and 3D transforms
- Complex-to-complex (`rocfft_transform_type_complex_forward/inverse`)
- Real-to-complex and complex-to-real (`rocfft_transform_type_real_forward/inverse`)
  producing/consuming half-spectrum buffers
- Single and double precision; half-precision experimental
- Arbitrary batch counts via `rocfft_plan_create` batch parameter
- Multi-GPU: plans can target specific HIP devices; multi-GPU overlap-save requires user-managed
  workload partitioning
- Non-uniform strides via `rocfft_plan_description_set_data_layout`
- Callback API (`rocfft_execution_info_set_load_callback`,
  `rocfft_execution_info_set_store_callback`) for fusing pre/post-processing into FFT kernel
  launches — set on the `rocfft_execution_info` object rather than the plan
  [Source: rocFFT header, https://github.com/ROCmSoftwarePlatform/rocFFT]

### 11.2 VkFFT

**VkFFT** ([https://github.com/DTolm/VkFFT](https://github.com/DTolm/VkFFT)) is a
header-only FFT library targeting Vulkan, OpenCL, CUDA, HIP, and Metal. It achieves close-to-peak
performance by generating optimised SPIR-V (or PTX/GCN) at plan initialisation for the exact
transform size, batch, and precision. Relevant `VkFFTConfiguration` fields:

| Field | Purpose |
|---|---|
| `FFTdim` | 1, 2, or 3 |
| `size[0..2]` | Transform dimensions |
| `numberBatches` | Independent transforms |
| `performR2C` | Enable half-spectrum real-to-complex mode |
| `normalize` | Apply 1/N normalisation on inverse |
| `makeForwardPlanOnly` / `makeInversePlanOnly` | Reduce compile time |
| `coalescedMemory` | Tune memory access pattern for target GPU |

VkFFT is particularly suited to projects that cannot take a ROCm or CUDA dependency; it runs on
any Vulkan 1.0+ device including Intel Arc, Mali, and Apple M-series. Benchmark comparisons
against cuFFT and rocFFT are published in the VkFFT repository at
[https://github.com/DTolm/VkFFT/tree/master/benchmark_scripts](https://github.com/DTolm/VkFFT/tree/master/benchmark_scripts).

### 11.3 FFTW3

**FFTW3** ([https://fftw.org](https://fftw.org)) is the reference CPU FFT library and defines the
API conventions (plan creation, execute, destroy) that rocFFT and other libraries deliberately
mirror. FFTW plans are created with `fftw_plan_dft_1d(N, in, out, FFTW_FORWARD, FFTW_MEASURE)`;
the `FFTW_MEASURE` flag runs timing tests at plan creation to select the fastest decomposition.
FFTW does not execute on GPU but serves as the ground-truth reference for correctness testing of
GPU FFT implementations.

### 11.4 cupyx.scipy.signal (formerly cuSignal)

The RAPIDS cuSignal project
([https://github.com/rapidsai/cusignal](https://github.com/rapidsai/cusignal)) provided CUDA
implementations of resampling (polyphase FIR), FIR filtering, spectral analysis, and waveform
generation. As of RAPIDS v23.08 (September 2023) cuSignal was archived and its functionality
migrated to **CuPy** (`cupyx.scipy.signal`,
[https://docs.cupy.dev/en/stable/reference/scipy_signal.html](https://docs.cupy.dev/en/stable/reference/scipy_signal.html)).
On Linux with an NVIDIA GPU and ROCm unavailable, `cupyx.scipy.signal` fills the same role as
rocFFT for Python-layer GPU signal processing. Its API mirrors SciPy.signal, lowering the barrier
for DSP engineers porting CPU-side signal processing pipelines to GPU.
[Source: cuSignal deprecation notice, https://github.com/rapidsai/cusignal]

### 11.5 GStreamer GPU Audio Elements

GStreamer ([https://gstreamer.freedesktop.org](https://gstreamer.freedesktop.org)) provides GPU
integration via two plugin families:

**gst-cuda** (for NVIDIA/CUDA) and **gst-vulkan** (backend-agnostic): both provide memory pool
integrations that let audio or video buffers be passed between GStreamer elements and GPU without
extra copies. For audio the standard pattern is:

```
pipewiresrc → audioresample → appsink       [CPU thread, fills GPU buffer]
       ↓
[GPU compute: FFT convolution, EQ, HRTF]
       ↓
appsrc → audioconvert → pipewiresink        [CPU thread, reads GPU output]
```

The `appsink` callback copies the audio buffer pointer to a mapped `VkBuffer`; the GPU pipeline
runs; the `appsrc` push-buffer call consumes the GPU output buffer. For high-channel-count
scenarios the memcpy overhead can be eliminated by importing the DMA-BUF from PipeWire directly
into the VkBuffer via `VK_EXT_external_memory_dma_buf`.

---

## 12. Integration with the Graphics Pipeline

### 12.1 Audio-Driven Animation: FFT Energy to Shader Uniform

Beat detection and frequency-band energy extraction provide natural inputs for GPU particle
systems, shader animations, and procedural effects. The standard pipeline:

1. Per-frame 512-point FFT of the audio block (already computed by the audio DSP pipeline)
2. Reduce to B = 8 frequency bands (bass, mid, high, etc.) by summing magnitudes within each
   band's bin range — a parallel reduction per band
3. Transfer band energies as push constants or a small SSBO to the graphics pipeline

The band energy buffer is written by the compute shader on the audio queue and consumed by the
graphics pipeline via a `VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT` → `VK_PIPELINE_STAGE_VERTEX_SHADER_BIT`
barrier with a timeline semaphore signal/wait on the transition from audio to render frame.

In GLSL:
```glsl
// fragment_audio.frag — audio-reactive shader uniform
layout(set=1, binding=0) uniform AudioBands {
    float bass;    // 20–250 Hz energy
    float lo_mid;  // 250–2000 Hz
    float hi_mid;  // 2–6 kHz
    float treble;  // 6–20 kHz
    float beat;    // binary beat detection flag
} audio;

// Particle emitter intensity: scale spawn rate by bass energy
float emitter_rate = audio.bass * 500.0;   // particles/second
```

### 12.2 Spectrogram Texture: GPU STFT to VkImage

A spectrogram is the visual representation of the STFT magnitude X[t, k] as an image. On GPU the
STFT output buffer (`VkBuffer` containing T × (N/2+1) complex values) is post-processed
in a compute shader that applies a log-magnitude colour map and writes the result as a `VkImage`:

```glsl
// spectrogram.comp — convert STFT row to texture column (scrolling spectrogram)
layout(local_size_x = 1024) in;
layout(set=0, binding=0) readonly  buffer STFT { vec2 X[]; };    // complex spectrum row
layout(set=0, binding=1, rgba8)    uniform writeonly image2D spec; // spectrogram texture
layout(push_constant) uniform PC { uint frame; uint N_bins; float min_db; float max_db; } pc;

void main() {
    uint k = gl_GlobalInvocationID.x;
    if (k >= pc.N_bins) return;
    vec2  c  = X[k];
    float db = 20.0 * log(length(c) + 1e-6) / log(10.0);
    float t  = clamp((db - pc.min_db) / (pc.max_db - pc.min_db), 0.0, 1.0);

    // Viridis-like colour map (simplified)
    vec4 colour = vec4(t, 0.5*t*(1.0-t), 1.0-t, 1.0);

    // Write to rolling column in spectrogram texture
    imageStore(spec, ivec2(pc.frame % imageSize(spec).x, int(pc.N_bins) - 1 - int(k)), colour);
}
```

The spectrogram `VkImage` is sampled by the fragment shader of a fullscreen quad in the
compositor overlay, creating a rolling spectrogram display updated every audio frame.

### 12.3 3D Audio Source Positions from Physics Simulation

Chapter 210 (GPU physics simulation) computes positions of rigid bodies and particle systems.
For audio sources attached to physical objects, the GPU physics output buffer
(`struct RigidBody { vec3 position; … }[]`) is read by the audio spatialization compute shader:

```glsl
// hrtf_lookup.comp — per-source HRTF selection from physics position
layout(set=0, binding=0) readonly buffer RigidBodies { RigidBody bodies[]; };
layout(set=0, binding=1) readonly buffer HRTFDatabase { vec2 hrir_db[]; };  // D x 2 x M

void main() {
    uint src = gl_GlobalInvocationID.x;
    vec3 rel_pos = bodies[src].position - listener_position;
    float azimuth   = atan(rel_pos.x, rel_pos.z);
    float elevation = asin(rel_pos.y / length(rel_pos));

    // Nearest-neighbour HRTF lookup (or bilinear interpolation)
    uint  dir_idx = nearest_hrtf_direction(azimuth, elevation);
    // ... load HRIR from database, write to per-source HRIR buffer for convolution
}
```

This pattern eliminates the CPU roundtrip for audio source positions: physics → GPU audio
spatialization → GPU convolution → PipeWire output, entirely on the GPU command stream.

### 12.4 GPU Audio + Vulkan Video AV Sync

For media playback applications the GPU decodes video frames (via Vulkan Video extensions,
Chapter 165), and the GPU processes audio (via the DSP pipeline above). Maintaining AV sync
requires that the presentation timestamp of each audio sample aligns with the corresponding video
frame.

The synchronisation point is the `vkQueuePresentKHR` call for the video frame and the DMA upload
of the corresponding audio block. Both are gated on the same `VkSemaphore` chain:

```
video_decode → [video semaphore] → composite + present
audio_dsp    → [audio semaphore] → DMA to ALSA ring buffer
```

The audio semaphore must signal no earlier than the video present semaphore, ensuring the audio
block is not audible before the corresponding video frame is visible. A drift correction loop
running on CPU adjusts the audio DMA offset if the GPU processing latency introduces measurable
AV drift.

---

## Integrations

- **Chapter 220 (GPU Image Processing Algorithms):** 2D FFT for image frequency-domain filtering
  (Section 2.4) is the image analogue of the 1D audio FFT; the GPU FFT library setup patterns
  (VkFFT, rocFFT) are shared directly.

- **Chapter 221 (GPU Algorithm Performance):** The roofline analysis for DSP kernels (arithmetic
  intensity of FIR vs. FFT convolution, occupancy patterns for multichannel processing) uses the
  performance models developed in Chapter 221. The IIR prefix-product approach maps directly to
  the parallel prefix scan primitives discussed there.

- **Chapter 223 (GPU Video Processing Algorithms):** Audio and video tracks share timeline
  semaphores for AV sync (Section 10.5). The GPU audio STFT pipeline feeds the spectrogram
  texture overlay displayed on video frames. The MDCT used in audio codecs (Section 7.2) is the
  1D analogue of the DCT used in video codecs (Chapter 223, Section 4).

- **Chapter 226 (Linear Algebra for GPU Algorithms):** Mel filterbank application (Section 9.2)
  and Ambisonic decoder (Section 8.3) reduce to GEMM; the HOA rotation matrix is a block-diagonal
  matrix multiply. rocBLAS / Vulkan cooperative matrices accelerate these operations.

- **Chapter 227 (Stochastic and Probabilistic GPU Algorithms):** Stochastic reverb tail
  generation (Section 8.4) samples exponential decay noise on GPU. MFCC-based audio
  classification (Section 9.3) feeds downstream GPU probabilistic models (HMM, GMM) for
  speech recognition — the stochastic decoding layer built on the deterministic GPU feature
  extraction pipeline of this chapter.
