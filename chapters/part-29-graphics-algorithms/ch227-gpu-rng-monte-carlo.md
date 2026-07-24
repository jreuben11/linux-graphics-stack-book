# Chapter 227: GPU Random Number Generation and Monte Carlo Methods (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Target audiences**: Graphics application developers implementing path tracers, stochastic global illumination, or GPU-based simulation who need reliable, high-quality random number streams on AMD, Intel, and NVIDIA hardware under Linux; systems developers building random-number infrastructure for GPU workloads using ROCm/HIP, oneAPI, or Vulkan compute.

This chapter covers the full stack from hardware-appropriate RNG selection through quasi-random low-discrepancy sequences, distribution sampling transforms, multiple importance sampling, reservoir-based resampling (ReSTIR), Markov chain Monte Carlo on GPU, and integration into production Vulkan pipelines. It draws on the rocRAND/hipRAND API (AMD ROCm), oneMKL PRNG (Intel oneAPI), PBRT-v4 sampler implementations, and GLSL/HLSL GPU-native patterns. API signatures reference ROCm 6.x and Vulkan 1.3 documentation; verify against current upstream before use.

---

## Table of Contents

1. [RNG on the GPU — Why CPU RNG Doesn't Scale](#1-rng-on-the-gpu--why-cpu-rng-doesnt-scale)
2. [Pseudorandom Number Generators](#2-pseudorandom-number-generators)
3. [Counter-Based RNGs for GPU](#3-counter-based-rngs-for-gpu)
4. [Quasi-Random Low-Discrepancy Sequences](#4-quasi-random-low-discrepancy-sequences)
5. [Sampling Distributions](#5-sampling-distributions)
6. [Multiple Importance Sampling and Sampling Strategies](#6-multiple-importance-sampling-and-sampling-strategies)
7. [Reservoir Sampling and ReSTIR](#7-reservoir-sampling-and-restir)
8. [GPU MCMC](#8-gpu-mcmc)
9. [Monte Carlo Integration on the GPU](#9-monte-carlo-integration-on-the-gpu)
10. [Production Libraries](#10-production-libraries)
11. [Reproducibility and Debugging](#11-reproducibility-and-debugging)
12. [Integration with Rendering Pipelines](#12-integration-with-rendering-pipelines)
13. [Integrations](#13-integrations)

---

## 1. RNG on the GPU — Why CPU RNG Doesn't Scale

### 1.1 The SIMT Problem

Classical CPU PRNGs — the Mersenne Twister (MT19937), the Xoshiro family, C standard `rand()` — maintain **global mutable state**. A single thread advances the state register with each call. On a GPU running 16,384 simultaneous threads across 128 compute units, serialising on a shared state register is impossible without catastrophic synchronisation overhead. The problem is architectural: SIMT (Single Instruction, Multiple Threads) execution means each thread must independently compute its next random value without communicating with its neighbours.

The conventional CPU solution — seeding one instance of a PRNG per thread — introduces new failure modes. If every thread receives the same seed, the streams are identical. If each thread receives `seed + thread_id`, many RNG families produce **correlated streams**: the sequence `{seed+0, seed+1, ..., seed+N}` is not an independent family for most LCGs or Mersenne Twisters. Statistical correlation between adjacent threads causes visible patterns (structured noise, banding) in path-traced images and biased Monte Carlo estimators.

### 1.2 Requirements for GPU RNG

Four properties are necessary for GPU-appropriate random number generation:

| Property | Requirement | Typical target |
|---|---|---|
| **Statistical independence** | Streams for different threads must pass independence tests | Failing rate < 1% across BigCrush battery |
| **Period** | Longer than the total samples generated | ≥ 2^64 per stream, ideally 2^128 |
| **Throughput** | Low instruction count per sample | ≤ 10 instructions/sample on GPU |
| **Reproducibility** | Same seed, same input, same output | Exactly bit-identical given same seed/thread-index |

A fifth property — **jump-ahead** (the ability to skip to the N-th output without computing the first N-1) — becomes important for dynamic parallelism and for partitioning a single long sequence across heterogeneous compute units.

### 1.3 GPU RNG Architecture Patterns

Three architectural patterns handle per-thread state on GPU:

1. **Per-thread state in registers or LDS**: each thread holds a small state struct (32–256 bits). Fast generation, but requires seeding each thread with an independent starting point. Works for LCG, PCG, and XORSHIFT.

2. **Counter-based (stateless)**: the thread ID *is* the input; the RNG maps `(seed, counter, offset) → value` via a keyed hash. No per-thread state stored; any output can be computed independently. Works for Philox, Threefry, and ARS.

3. **Shared-memory table RNG**: the generator (e.g. MTGP32) stores a shared state in LDS, cooperating across a threadgroup. High statistical quality at the cost of LDS pressure and a fixed group size.

[Source: Salmon, J. K. et al., "Parallel Random Numbers: As Easy as 1, 2, 3", SC'11, https://dl.acm.org/doi/10.1145/2063384.2063405]

---

## 2. Pseudorandom Number Generators

### 2.1 Linear Congruential Generator

The **multiplicative LCG** advances state via:

```
x_{n+1} = a · x_n mod m
```

With `m = 2^32` or `m = 2^64`, the modulo operation is free (integer overflow). A full-period multiplicative LCG requires `m = 2^k`, `a ≡ 5 (mod 8)` for the full period of `2^{k-2}`. The minimal requirement for GPU use is 64-bit state.

**Strengths**: one multiply, one add. **Weaknesses**: serious spectral defects in high dimensions (lattice structure in sequential output); low-order bits are short-period. LCG alone is unsuitable as a general-purpose GPU RNG; it appears in Philox's key-schedule only.

### 2.2 XORSHIFT and XORSHIFT128+

Marsaglia (2003) showed that combining three XOR-shift operations on a 32- or 64-bit integer produces period-`2^n - 1` sequences with good empirical properties. XORSHIFT32:

```c
uint32_t xorshift32(uint32_t *state) {
    uint32_t x = *state;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    return *state = x;
}
```

**XORSHIFT128+** (Vigna 2014) uses a 128-bit state (two 64-bit registers) and passes BigCrush with only marginal failures in the low-order bit. It is the RNG used inside JavaScript engines (V8, SpiderMonkey) for `Math.random()`. On GPU it maps naturally to two `uint64_t` registers per thread.

[Source: Marsaglia, G., "Xorshift RNGs", Journal of Statistical Software 8(14), 2003, https://doi.org/10.18637/jss.v008.i14]

### 2.3 PCG (Permuted Congruential Generator)

O'Neill (2014) demonstrated that a 64-bit LCG state, passed through a **permutation function** (output: xorshift then random rotation), produces 32-bit output that passes all of BigCrush and most of PractRand to at least 32 TB. PCG-XSH-RR (the default PCG32):

```c
uint32_t pcg32(uint64_t *state, uint64_t increment) {
    uint64_t oldstate = *state;
    *state = oldstate * 6364136223846793005ULL + (increment | 1u);
    uint32_t xorshifted = (uint32_t)(((oldstate >> 18u) ^ oldstate) >> 27u);
    uint32_t rot = (uint32_t)(oldstate >> 59u);
    return (xorshifted >> rot) | (xorshifted << ((-rot) & 31));
}
```

`increment` controls which of 2^63 distinct streams is selected; setting `increment = 2 * stream_id + 1` gives independent per-thread streams. PCG has 128-bit variants for longer periods.

[Source: O'Neill, M. E., "PCG: A Family of Simple Fast Space-Efficient Statistically Good Algorithms for Random Number Generation", HMC-CS-2014-0905, https://www.pcg-random.org/paper.html]

### 2.4 Philox and Threefry (Counter-Based)

Both are described in §3 below; they are placed here for statistical context. Philox-4×32-10 passes all of BigCrush and PractRand to 512 GB without a single failure. Threefry-4×64-20 has equivalent quality with longer output. These are the **AMD hipRAND defaults** for GPU work. [Source: Salmon et al., SC'11]

### 2.5 MTGP32 — Mersenne Twister on GPU

Saito & Matsumoto (2010) adapted the Mersenne Twister for GPU via **MTGP32**, which uses shared LDS to maintain a state array of 351 32-bit words cooperatively across a 256-thread group. Period is 2^{11213} − 1; the output passes BigCrush. The GPU trade-off is significant: the 351-word state requires 1.4 KB of LDS per group, occupancy is reduced, and the initialisation phase runs serially. MTGP32 is best suited to workloads that generate millions of samples from a single group (Monte Carlo particle transport, turbulence) rather than path tracers where each pixel is a separate thread.

[Source: Saito, M., "A Variant of Mersenne Twister Suitable for Graphic Processors", ACM TOMACS 23(2), 2013, https://doi.org/10.1145/2427530.2427531 — Note: verify DOI and publication year against the ACM DL page before citing.]

### 2.6 Statistical Quality Summary

| Generator | Period | BigCrush failures | PractRand limit | GPU state size |
|---|---|---|---|---|
| LCG (64-bit) | 2^62 | Many | < 1 GB | 8 B |
| XORSHIFT128+ | 2^128 − 1 | 1–2 (low bit) | 32 TB | 16 B |
| PCG-XSH-RR | 2^128 | 0 | ≥ 32 TB | 16 B |
| Philox-4×32-10 | 2^128 per key | 0 | ≥ 512 GB | 16 B (counter) |
| Threefry-4×64-20 | 2^256 per key | 0 | ≥ 512 GB | 32 B (counter) |
| MTGP32 | 2^11213 − 1 | 0 | ≥ 512 GB | 1.4 KB (LDS) |

[Source: TestU01 BigCrush battery, L'Ecuyer & Simard, http://simul.iro.umontreal.ca/testu01/tu01.html; PractRand, http://pracrand.sourceforge.net/]

---

## 3. Counter-Based RNGs for GPU

### 3.1 The Key Advantage

Counter-based RNGs (CBRNGs) define a function:

```
output = RNG(key, counter)
```

where **counter** is an arbitrary integer (typically the thread index or sample number) and **key** is a seed. There is no sequential dependency: any thread can compute its N-th output in O(1) without evaluating outputs 0 through N-1. This is the property that makes CBRNGs ideal for GPU parallelism and for reproducible rendering (where thread X in frame F at bounce B needs a unique, independent sample without coordination).

### 3.2 Philox-4×32-10

Philox uses a **Feistel network** with multiplication as the mixing operation. The 4×32 variant produces four 32-bit outputs from a 128-bit counter and a 64-bit key, using 10 rounds. The mixing constants are chosen to maximise avalanche:

```c
// Philox-4x32-10 round constants
#define PHILOX_M4x32_0  0xD2511F53u
#define PHILOX_M4x32_1  0xCD9E8D57u
#define PHILOX_W4x32_0  0x9E3779B9u  // key schedule constant (golden ratio)
#define PHILOX_W4x32_1  0xBB67AE85u

typedef struct { uint32_t v[4]; } philox4x32_ctr_t;
typedef struct { uint32_t v[2]; } philox4x32_key_t;

static inline void philox4x32_round(philox4x32_ctr_t *c, const philox4x32_key_t *k) {
    uint64_t prod0 = (uint64_t)PHILOX_M4x32_0 * c->v[0];
    uint64_t prod1 = (uint64_t)PHILOX_M4x32_1 * c->v[2];
    c->v[0] = (uint32_t)(prod1 >> 32) ^ c->v[1] ^ k->v[0];
    c->v[1] = (uint32_t)prod1;
    c->v[2] = (uint32_t)(prod0 >> 32) ^ c->v[3] ^ k->v[1];
    c->v[3] = (uint32_t)prod0;
}
```

After 10 such rounds (with key schedule `key[r+1] = key[r] + {W0, W1}`), the four 32-bit outputs pass all known statistical tests.

[Source: Salmon et al., SC'11; Random123 library reference implementation, https://github.com/DEShawResearch/random123]

### 3.3 Threefry-4×64-20

Threefry applies the **Threefish** block cipher's core operation (word XOR, rotation, word permutation) without the key schedule, using only the initial key. The 4×64-20 variant produces four 64-bit outputs from a 256-bit counter and a 256-bit key, using 20 rounds; period is 2^{256} per key. Salmon et al. estimate the effective security from cryptanalysis of reduced-round variants, but the exact bit-security of Threefry-4×64-20 as a PRNG (as opposed to a cipher) is not formally established — treat it as statistically strong rather than cryptographically certified.

On current AMD RDNA3 and NVIDIA Ada hardware, Threefry-4×64 is slightly slower than Philox-4×32 because 64-bit multiply is less native than 32-bit. For applications requiring double-precision random inputs, Threefry-4×64 avoids the reassembly step.

### 3.4 ARS — AES Round-Based

ARS uses hardware AES encryption rounds (`vaesenc` on x86, `AESENC` on NVIDIA via inline PTX) as the keyed hash. One or two rounds suffice for most sampling applications; ten rounds (full AES) gives cryptographic quality. ARS is only available on hardware with AES acceleration units — not all GPU shader cores have these.

### 3.5 Implementation Pattern: (seed, sequence, offset) → Values

The canonical three-level indexing maps naturally to graphics workloads:

```glsl
// GLSL GPU-native counter-based RNG pattern
// seed    = per-frame or per-render constant
// seq_id  = gl_GlobalInvocationID (unique per thread)
// offset  = dimension index within this thread's sample

struct RNGState {
    uvec4 counter;  // Philox 128-bit counter
    uvec2 key;      // Philox 64-bit key
};

RNGState rng_init(uint seed, uint seq_id, uint offset) {
    RNGState s;
    s.counter = uvec4(seq_id, offset, 0u, 0u);
    s.key     = uvec2(seed, seed >> 16u);
    return s;
}

// Generate next float in [0,1)
// Uses umulExtended() for the 32x32->64 bit multiply, available in GLSL core 4.0+
float rng_next(inout RNGState s) {
    // Apply Philox-2x32 round (simplified for GLSL core — no uint64_t extension needed)
    const uint M0 = 0xD2511F53u;
    const uint W0 = 0x9E3779B9u;
    uvec2 ctr = s.counter.xy;
    uint key  = s.key.x;
    uint hi, lo;
    for (int r = 0; r < 7; r++) {  // 7 rounds
        umulExtended(M0, ctr.x, hi, lo);  // hi:lo = M0 * ctr.x (GLSL core)
        ctr = uvec2(hi ^ ctr.y ^ key, lo);
        key += W0;
    }
    s.counter.x++;  // advance counter for next call
    return float(ctr.x) * (1.0 / 4294967296.0);
}
```

*Note: `umulExtended()` is GLSL core since version 4.0 and is available in GLSL ES 3.1 compute shaders. The full Philox-4×32-10 with 10 rounds is available in rocRAND device headers for HIP kernels.*

[Source: rocRAND device API, https://rocm.docs.amd.com/projects/rocRAND/en/latest/]

---

## 4. Quasi-Random Low-Discrepancy Sequences

### 4.1 Discrepancy and Convergence Rate

Monte Carlo integration converges at O(1/√N). **Quasi-Monte Carlo** (QMC) replaces pseudo-random samples with **low-discrepancy sequences** (LDS) whose samples fill space more evenly, achieving convergence rates of O((log N)^s / N) for s-dimensional integrals, where s is the number of dimensions. In practice this means 10–100× fewer samples needed to reach the same variance.

**Star discrepancy** D* measures how evenly a set of N points covers the unit hypercube:

```
D*(x_1,...,x_N) = sup_{J ⊆ [0,1]^s} |A(J,N)/N - Vol(J)|
```

where A(J,N) counts points inside J. Random points have D* ≈ O(1/√N); Sobol sequences achieve D* = O((log N)^s / N).

### 4.2 van der Corput and Halton

The **van der Corput sequence** in base b maps integer n to a fraction by reversing its base-b digits. In base 2:

```glsl
// Radical inverse base 2 — fast bit-reversal implementation
float radicalInverseBase2(uint bits) {
    bits = (bits << 16u) | (bits >> 16u);
    bits = ((bits & 0x55555555u) << 1u)  | ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u)  | ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u)  | ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u)  | ((bits & 0xFF00FF00u) >> 8u);
    return float(bits) * 2.3283064365386963e-10; // 1 / 2^32
}
```

The **Halton sequence** in s dimensions uses base-p_d for dimension d, where p_1=2, p_2=3, p_3=5, ... (consecutive primes). It requires no precomputed tables, but correlation between dimensions grows for high bases: Halton is typically limited to 10–15 dimensions before scrambling is needed.

### 4.3 Sobol Sequences and Joe-Kuo Tables

Sobol sequences use **direction numbers** v_{d,i} (one per dimension d and bit position i) to construct a sequence where:

```glsl
// GPU Sobol: O(log N) per sample using XOR accumulation
uint sobolSample1D(uint n, uint dim, uint direction_numbers[32]) {
    uint result = 0u;
    for (int i = 0; n != 0u; i++, n >>= 1u) {
        if ((n & 1u) != 0u) result ^= direction_numbers[i];
    }
    return result;
}
// Convert to float: float(result) * (1.0 / 4294967296.0)
```

The Joe-Kuo direction number tables (2010) provide good two-dimensional projections for up to 21,201 dimensions. They are the reference tables used by PBRT-v4's `ZSobolSampler`.

[Source: Joe, S. & Kuo, F. Y., "Constructing Sobol Sequences with Better Two-Dimensional Projections", SIAM J. Sci. Comput. 30(5), 2010, https://doi.org/10.1137/070709359; direction number tables at https://web.maths.unsw.edu.au/~fkuo/sobol/]

### 4.4 Scrambled Sobol and Owen Scrambling

Unscrambled Sobol sequences are **deterministic**: the same samples appear in the same positions every render, leading to structured correlations across the image. **Owen scrambling** (1995) applies a random permutation at each tree level of the radical inverse construction, randomising the sequence while preserving its stratification properties. The scrambled estimator is unbiased and variance-equal to the unscrambled version, but eliminates systematic patterns.

**Faure-Tezuka scrambling** applies a linear transform over GF(2) to the direction number matrix — cheaper than full Owen scrambling but slightly less effective. PBRT-v4's `ZSobolSampler` uses Owen scrambling via a fast hash.

[Source: PBRT-v4, §8 "Sampling and Reconstruction", https://www.pbr-book.org/4ed/Sampling_and_Reconstruction]

### 4.5 GPU Sobol Generation via Parallel Prefix

For parallel generation of all N Sobol samples, note that samples n and n+1 differ only in the bits corresponding to the trailing zeros of n (the **Gray code** property). On GPU this enables a parallel prefix approach: compute direction number XOR contributions from the Gray code of each thread's index independently, then prefix-XOR to accumulate.

For a 1024-thread dispatch generating sample indices 0..1023:
1. Each thread i computes `gray = i ^ (i >> 1)` (Gray code index).
2. XOR direction numbers for set bits of `gray`.
3. This is embarrassingly parallel; no inter-thread communication needed for the basic case.

---

## 5. Sampling Distributions

### 5.1 Uniform to Gaussian: Box-Muller

Given two independent uniform samples u₁, u₂ ∈ (0,1], the **Box-Muller transform** produces two independent standard normal samples:

```glsl
vec2 boxMuller(float u1, float u2) {
    float r     = sqrt(-2.0 * log(u1));
    float theta = 6.28318530718 * u2;
    return r * vec2(cos(theta), sin(theta));
}
```

The log and trigonometric functions make this moderately expensive on GPU (~8 instructions). For high-throughput normal sampling, the **Ziggurat method** is faster: it partitions the normal distribution into horizontal rectangular layers and uses table lookups to accept/reject without transcendentals in the common path. The Ziggurat tail probability is small enough that the tail handler is rarely invoked.

[Source: Box, G. E. P. & Muller, M. E., "A Note on the Generation of Random Normal Deviates", Annals of Mathematical Statistics 29(2), 1958, https://doi.org/10.1214/aoms/1177706645]

### 5.2 Inverse CDF for Standard Distributions

For exponential distribution with rate λ: `x = -ln(1 - u) / λ`. Since 1-u is also uniform when u is uniform, `x = -ln(u) / λ` is equivalent and avoids a subtraction.

For Poisson(λ): sum exponential inter-arrival times until they exceed 1 (Knuth's method) or use the inversion by binary search on the CDF table for large λ.

### 5.3 Rejection Sampling

Rejection sampling draws from a proposal q(x) and accepts with probability f(x) / (c · q(x)) where c = sup(f/q). The expected number of draws is c. On GPU, divergence between accepted and rejected threads creates wavefront efficiency issues; rejection sampling is acceptable only when the acceptance rate is high (> 70%) or when samples are generated in bulk and the rejected-thread overhead is amortised by other active threads.

### 5.4 Alias Method for Arbitrary Discrete Distributions

Walker's **Alias method** (1977) preprocesses a discrete distribution P into two tables `Prob[i]` and `Alias[i]` such that sampling requires only:
1. Draw u₁ ∈ [0,N), identify bucket `i = floor(u₁)`.
2. Draw u₂ ∈ [0,1); if `u₂ < Prob[i]`, return i; else return Alias[i].

This is O(1) per sample. **GPU-parallel alias table construction**: normalise weights, label each bucket as "small" (scaled weight < 1) or "large" (≥ 1), then pair small with large via a parallel worklist algorithm using atomic compaction. The table fits in a structured buffer (SSBO) for shader access.

[Source: Walker, A. J., "An Efficient Method for Generating Discrete Random Variables with General Distributions", ACM TOMS 3(3), 1977, https://doi.org/10.1145/355744.355749]

---

## 6. Multiple Importance Sampling and Sampling Strategies

### 6.1 The Multiple Importance Sampling Framework

When combining samples drawn from k different PDFs p₁,...,pₖ to estimate an integral, **multiple importance sampling** (MIS) assigns a weight w_i to each sample from pᵢ such that the combined estimator is unbiased and variance is minimised.

The **balance heuristic** sets:
```
w_i(x) = n_i p_i(x) / Σ_j (n_j p_j(x))
```

The **power heuristic** with exponent β=2 (Veach's recommendation) is more effective:
```glsl
float powerHeuristic(float nf, float fPdf, float ng, float gPdf) {
    float f = nf * fPdf;
    float g = ng * gPdf;
    return (f * f) / max(1e-10, f * f + g * g);
}
```

In a path tracer, the two strategies are BSDF sampling and light sampling; MIS combines both without requiring one to dominate.

[Source: Veach, E. & Guibas, L., "Optimally Combining Sampling Techniques for Monte Carlo Rendering", SIGGRAPH 1995, https://doi.org/10.1145/218380.218498]

### 6.2 Cosine-Weighted Hemisphere Sampling

For Lambertian surfaces the BRDF is `f = albedo/π` and the ideal PDF is cosine-weighted:

```glsl
// Sample hemisphere with pdf = cos(theta)/pi
vec3 cosineSampleHemisphere(float u1, float u2, out float pdf) {
    float phi      = 6.28318530718 * u2;
    float cosTheta = sqrt(u1);      // u1 = cos^2(theta)
    float sinTheta = sqrt(1.0 - u1);
    pdf = cosTheta * 0.31830988;    // / pi
    return vec3(sinTheta * cos(phi), sinTheta * sin(phi), cosTheta);
}
```

### 6.3 GGX VNDF Sampling

For GGX microfacet BRDFs, sampling the **visible normal distribution function** (VNDF) rather than the full NDF dramatically reduces dark-region samples. Heitz (2018) gives the algorithm:

```glsl
// GGX VNDF sampling (Heitz, JCGT 2018)
vec3 sampleGGXVNDF(vec3 Ve, float alphaX, float alphaY, float u1, float u2) {
    // Stretch view direction
    vec3 Vh = normalize(vec3(alphaX * Ve.x, alphaY * Ve.y, Ve.z));

    // Orthonormal basis
    float lensq = Vh.x * Vh.x + Vh.y * Vh.y;
    vec3 T1 = (lensq > 0.0) ? vec3(-Vh.y, Vh.x, 0.0) / sqrt(lensq)
                             : vec3(1.0, 0.0, 0.0);
    vec3 T2 = cross(Vh, T1);

    // Disk sample
    float r   = sqrt(u1);
    float phi = 6.28318530718 * u2;
    float t1  = r * cos(phi);
    float t2  = r * sin(phi);
    float s   = 0.5 * (1.0 + Vh.z);
    t2 = mix(sqrt(1.0 - t1 * t1), t2, s);

    // Compute half-vector and unstretch
    vec3 Nh = t1 * T1 + t2 * T2 + sqrt(max(0.0, 1.0 - t1*t1 - t2*t2)) * Vh;
    return normalize(vec3(alphaX * Nh.x, alphaY * Nh.y, max(1e-6, Nh.z)));
}
```

[Source: Heitz, E., "Sampling the GGX Distribution of Visible Normals", JCGT 7(4), 2018, http://jcgt.org/published/0007/04/01/]

### 6.4 Stratified and Blue-Noise Sampling

**Jittered stratification** divides the 2D sample domain into an N×N grid and places one random sample per cell: `(i + u1) / N, (j + u2) / N` for cell (i,j). This reduces variance relative to pure random sampling but couples samples spatially.

**Blue-noise sampling** targets a power spectrum that is flat at high frequencies (no low-frequency clustering). The **void-and-cluster algorithm** (Ulichney 1993) iteratively moves the point with the highest cluster density to the largest void, converging to a blue-noise distribution. Pre-computed blue-noise textures (128×128 arrays) are the practical GPU approach: sample from `bluenoise_tex[px % 128, py % 128]` with per-frame offset scrambling to avoid temporal repetition.

[Source: Ulichney, R., "The Void-and-Cluster Method for Dither Array Generation", SPIE Human Vision, Visual Processing, and Digital Display IV, 1993, https://doi.org/10.1117/12.152707]

---

## 7. Reservoir Sampling and ReSTIR

### 7.1 Weighted Reservoir Sampling

**Weighted reservoir sampling** (WRS) selects one element from a stream of N elements with probability proportional to weight w_i, using O(1) memory. The **A-Res algorithm** (Efraimidis & Spirakis 2006) assigns key `k_i = u_i^{1/w_i}` to each element and keeps the maximum-key element. The equivalent one-pass incremental form is:

```glsl
struct Reservoir {
    uint  y;      // selected sample index
    float w_sum;  // accumulated weight sum
    uint  M;      // number of candidates seen
    float W;      // unbiased contribution weight
};

bool reservoirUpdate(inout Reservoir r, uint x, float w, float u) {
    r.w_sum += w;
    r.M     += 1u;
    bool chosen = (u < w / r.w_sum);
    if (chosen) r.y = x;
    return chosen;
}

void reservoirFinalise(inout Reservoir r, float pHatY) {
    r.W = (pHatY > 0.0) ? r.w_sum / (float(r.M) * pHatY) : 0.0;
}
```

The reservoir stores only the current best sample and accumulated weight, enabling streaming updates without storing the full candidate set.

**A-ExpJ** is an improved variant from the same paper that avoids a uniform draw per candidate by drawing the next-skip-count from an exponential distribution, reducing expected draw count for skewed weight distributions. In practice on GPU — where branch divergence between threads is the primary cost and the number of candidates per thread is small (8–32) — the simpler A-Res incremental form above is preferred for its predictable control flow.

**GPU-parallel WRS with prefix-sum weight accumulation**: when the full weight array is known up-front (not streaming), all-parallel WRS can use a **parallel prefix sum** over weights to build a cumulative distribution, then binary-search into it with a single uniform sample per thread. This requires O(N) global memory but enables concurrent sampling without sequential accumulation. In rendering, this pattern appears in light BVH importance sampling: precompute prefix-sum of per-light weights into an SSBO, then each pixel independently binary-searches the CDF with its random sample.

[Source: Efraimidis, P. S. & Spirakis, P. G., "Weighted Random Sampling with a Reservoir", Information Processing Letters 97(5), 2006, https://doi.org/10.1016/j.ipl.2005.11.003]

### 7.2 ReSTIR DI — Spatiotemporal Reservoir Resampling

**ReSTIR** (Bitterli et al., SIGGRAPH 2020) uses reservoir resampling to achieve many-light path tracing at real-time frame rates. The core idea: for each pixel, maintain a reservoir that accumulates candidates from the current and previous frames, biased toward candidates with high contribution (`p_hat`).

The pipeline has three compute passes:

```
Pass 1: Initial sampling
  For each pixel p:
    Sample INITIAL_M light candidates from the light BVH
    Weighted-reservoir-sample them into R_p (w = p_hat(y) / p_source(y))
    Finalise R_p.W

Pass 2: Temporal reuse
  For each pixel p:
    Project previous R_{t-1}[q] where q = motionVector(p)
    Combine R_p and R_{t-1}[q] via reservoir merge:
      R_combined = merge(R_p, R_{t-1}[q])
    Cap M to M_max (e.g. 20) to prevent temporal overconfidence

Pass 3: Spatial reuse
  For each pixel p:
    Sample k spatial neighbours q_1...q_k
    Merge R_{spatial}[q_i] into R_p after visibility weighting
    Apply MIS weight to account for different p_hat at neighbour location
```

The reservoir merge operation:

```glsl
void mergeReservoirs(inout Reservoir r1, Reservoir r2, float pHat2, float u) {
    reservoirUpdate(r1, r2.y, pHat2 * r2.W * float(r2.M), u);
    r1.M += r2.M;
}
```

[Source: Bitterli, B., Wyman, C., Pharr, M., Shirley, P., Lefohn, A., Jarosz, W., "Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting", SIGGRAPH 2020, https://research.nvidia.com/publication/2020-07_spatiotemporal-reservoir-resampling]

### 7.3 ReSTIR Vulkan Compute Layout

```glsl
// GLSL compute shader bindings for ReSTIR
layout(set = 0, binding = 0, std430) buffer ReservoirBuffer {
    Reservoir cur_reservoirs[];
};
layout(set = 0, binding = 1, std430) buffer ReservoirBufferPrev {
    Reservoir prev_reservoirs[];
};
layout(set = 0, binding = 2) uniform accelerationStructureEXT tlas;
layout(set = 0, binding = 3) uniform sampler2D gbuffer_depth;
layout(set = 0, binding = 4) uniform sampler2D motion_vectors;

layout(push_constant) uniform PC {
    uint  frame_seed;
    uint  width;
    uint  height;
    uint  num_lights;
} pc;

layout(local_size_x = 8, local_size_y = 8) in;

void main() {
    ivec2 px = ivec2(gl_GlobalInvocationID.xy);
    if (px.x >= int(pc.width) || px.y >= int(pc.height)) return;
    uint idx = px.y * pc.width + px.x;
    uint seed = hash_combine(pc.frame_seed, idx);
    // ... perform initial sampling + temporal merge
}
```

Dispatch: `vkCmdDispatch(cmd, ceil(width/8), ceil(height/8), 1)` for each of the three passes, with `VkMemoryBarrier` between passes (stage `COMPUTE_SHADER_BIT`, access `SHADER_READ_BIT | SHADER_WRITE_BIT`).

---

## 8. GPU MCMC

### 8.1 Metropolis-Hastings on Parallel Independent Chains

For Bayesian inference over GPU-resident data (material parameter estimation, shape reconstruction), Metropolis-Hastings (MH) runs best as multiple **independent chains** rather than one long chain, exploiting GPU parallelism. Each chain evolves independently:

```glsl
// MH step for one chain (invocation = one chain)
bool metropolisStep(float current_log_target, float proposed_log_target,
                    float log_proposal_ratio, float u) {
    float log_accept = proposed_log_target - current_log_target + log_proposal_ratio;
    return (u < exp(min(0.0, log_accept)));
}
```

The SSBO layout stores current state `x_n` and log-target `log π(x_n)` for each chain; after `steps_per_dispatch` proposals, the host reads convergence statistics via an output buffer.

### 8.2 Hamiltonian Monte Carlo with Leapfrog

**HMC** augments the target distribution with momentum p and integrates Hamiltonian dynamics using the **leapfrog integrator**, producing proposals with high acceptance rates even in high-dimensional spaces. Each leapfrog step:

```
p(t + ε/2) = p(t) − (ε/2) ∇U(q(t))
q(t + ε)   = q(t) + ε M⁻¹ p(t + ε/2)
p(t + ε)   = p(t + ε/2) − (ε/2) ∇U(q(t + ε))
```

where U(q) = −log π(q) is the potential energy. On GPU, each chain invocation runs L leapfrog steps (typically L=10–50), then performs an MH accept/reject. The gradient ∇U is computed analytically in the shader (for simple targets) or via a finite-difference approximation.

**Parallel tempering** runs K chains at inverse temperatures β_1 > β_2 > ... > β_K and periodically proposes swaps between adjacent temperature levels. It improves exploration in multimodal targets. The swap acceptance probability `min(1, exp((β_i − β_{i+1})(log π(x_j) − log π(x_i))))` requires inter-chain reads via SSBO, so swaps are performed between compute dispatches, not inside a single kernel.

### 8.3 Convergence Diagnostics on GPU

**R-hat** (Gelman-Rubin potential scale reduction factor) compares between-chain variance B to within-chain variance W:

```
B = N/(K-1) Σ_k (θ̄_k − θ̄)²      // between-chain
W = 1/K Σ_k s²_k                  // within-chain mean of sample variances
R-hat = sqrt((N-1)/N + B/(N·W))
```

Values R-hat < 1.01 indicate convergence. Computing R-hat on GPU requires a reduce pass over the chain buffer: compute per-chain mean and variance in a parallel reduction, then combine. **Effective sample size** (ESS) is estimated via autocorrelation of each chain's samples, requiring an FFT-based autocorrelation or a truncated running average.

---

## 9. Monte Carlo Integration on the GPU

### 9.1 The Basic Estimator and Variance

The Monte Carlo estimator for `∫ f(x) dx` over domain Ω with sampling PDF p:

```
F_N = (1/N) Σᵢ f(xᵢ) / p(xᵢ),   xᵢ ~ p
```

has variance `Var[F_N] = Var[f/p] / N`. Variance decreases as 1/N regardless of dimension, which is the key advantage over quadrature in high-dimensional rendering integrals (path-space is 2k-dimensional for k-bounce paths).

### 9.2 Variance Reduction Techniques

**Control variates**: if g(x) has known expectation E[g], replace `f` with `f − c(g − E[g])`. The optimal coefficient c = Cov(f,g)/Var(g) halves variance when f and g are strongly correlated. In path tracing, the direct illumination estimator (analytically integrable for simple lights) is a natural control variate for the indirect estimator.

**Antithetic variates**: draw sample pairs (x, 1−x) from the unit interval; if f is monotone in x, f(x) + f(1-x) has lower variance than 2f(x) from independent draws. Antithetic sampling is compatible with the van der Corput radical-inverse construction.

### 9.3 Environment Map Importance Sampling

Sampling from an HDR environment map with probability proportional to radiance requires a **hierarchical 2D CDF**:

```
1. Compute row sums: marginal_cdf[v] = integral of row v over u
2. Normalise marginal_cdf to CDF
3. For each row v: compute conditional_cdf[v][u] = prefix sum of row v pixels
4. Sample: draw u1 → binary-search marginal_cdf → v; draw u2 → conditional_cdf[v] → u
```

On GPU, steps 1–3 are a prefix-sum pass over the environment texture (2D dispatch). The resulting CDF tables fit in SSBOs of size proportional to the map resolution. The Alias method applied to the 2D raster of environment pixels avoids binary search and gives O(1) per sample, with O(WHR) preprocessing (W×H map, R=4 bytes per entry).

### 9.4 Path Tracing Estimator Structure

A path tracer decomposes the rendering integral into a **direct illumination** estimator (next event estimation, NEE) and an **indirect illumination** estimator:

```
L_direct(p,ωo) ≈ (1/N_lights) Σᵢ f(p,ωo,ωᵢ) L_e(p',ωᵢ) G(p,p') V(p,p') / p_light(ωᵢ)
L_indirect(p,ωo) ≈ f(p,ωo,ω_sample) L(p',−ω_sample) / p_bsdf(ω_sample)
```

MIS combines NEE and BSDF sampling for direct illumination using the power heuristic (§6.1). Indirect illumination uses a single BSDF sample per bounce. Each path bounce consumes two random dimensions (direction sampling), plus additional dimensions for NEE (light position, visibility).

**Bidirectional path tracing** (BDPT) samples paths from both the camera and light simultaneously, forming connection paths. Sampling from two independent sub-path generators (one seeded from camera pixels, one from light sources) is natural on GPU; the sub-path count per dispatch is limited by SSBO memory for storing intermediate path vertices.

---

## 10. Production Libraries

### 10.1 rocRAND / hipRAND (AMD ROCm)

rocRAND provides both a host API (generates into device memory via HIP kernels) and a device API (per-thread state in shader code). The host API:

```c
#include <rocrand/rocrand.h>

rocrand_generator gen;
rocrand_create_generator(&gen, ROCRAND_RNG_PSEUDO_PHILOX4_32_10);
rocrand_set_seed(gen, 42ULL);

float *d_buf;
hipMalloc((void**)&d_buf, N * sizeof(float));

// Generate N uniform floats in (0,1]:
rocrand_generate_uniform(gen, d_buf, N);

// Generate N normally distributed floats (mean=0, stddev=1):
rocrand_generate_normal(gen, d_buf, N, 0.0f, 1.0f);

rocrand_destroy_generator(gen);
hipFree(d_buf);
```

Available generator types via `ROCRAND_RNG_*`: `PSEUDO_XORWOW`, `PSEUDO_MRG32K3A`, `PSEUDO_MTGP32`, `PSEUDO_MT19937`, `PSEUDO_PHILOX4_32_10`, `QUASI_SOBOL32`, `QUASI_SCRAMBLED_SOBOL32`, `QUASI_SOBOL64`.

The **hipRAND compatibility shim** (`hiprand.h`) provides a cuRAND-compatible API using the same camelCase naming convention — `hiprandCreateGenerator`, `hiprandSetPseudoRandomGeneratorSeed`, `hiprandGenerateUniform`, `hiprandGenerateNormal` — making CUDA code portable to HIP with only header and prefix changes. Note that rocRAND itself uses snake_case (`rocrand_create_generator`, `rocrand_generate_uniform`); hipRAND wraps rocRAND behind the cuRAND-style camelCase façade.

The device API allows per-thread generation inside HIP/CUDA kernels:

```c
// Device-side rocRAND (Philox per thread)
#include <rocrand/rocrand_kernel.h>

__global__ void sample_kernel(float *out, unsigned long long seed, int N) {
    int tid = blockIdx.x * blockDim.x + threadIdx.x;
    if (tid >= N) return;

    rocrand_state_philox4x32_10_t state;
    rocrand_init(seed,       // seed (same for all threads)
                 (uint64_t)tid,  // sequence: unique per thread
                 0ULL,           // offset within sequence
                 &state);

    float4 r4 = rocrand_uniform4(&state);  // 4 independent floats
    out[4*tid + 0] = r4.x;
    out[4*tid + 1] = r4.y;
    out[4*tid + 2] = r4.z;
    out[4*tid + 3] = r4.w;
}
```

[Source: rocRAND API reference, https://rocm.docs.amd.com/projects/rocRAND/en/latest/; hipRAND compatibility, https://rocm.docs.amd.com/projects/hipRAND/en/latest/]

### 10.2 oneMKL PRNG (Intel oneAPI)

Intel's oneAPI Math Kernel Library exposes PRNG through `oneapi::mkl::rng`:

```cpp
#include <oneapi/mkl/rng.hpp>

sycl::queue q;

// Create Philox engine (oneAPI uses Philox4x32x10 internally)
oneapi::mkl::rng::philox4x32x10 engine(q, /* seed = */ 12345ULL);

// Uniform distribution [0.0, 1.0)
oneapi::mkl::rng::uniform<float> uniform_dist(0.0f, 1.0f);
sycl::buffer<float, 1> buf_u(N);
oneapi::mkl::rng::generate(uniform_dist, engine, N, buf_u);

// Normal distribution N(0,1)
oneapi::mkl::rng::gaussian<float> gauss_dist(0.0f, 1.0f);
sycl::buffer<float, 1> buf_n(N);
oneapi::mkl::rng::generate(gauss_dist, engine, N, buf_n);
```

The **VSL (Vector Statistical Library)** C API remains available for compatibility:

```c
VSLStreamStatePtr stream;
vslNewStream(&stream, VSL_BRNG_PHILOX4X32X10, 12345);
vsRngUniform(VSL_RNG_METHOD_UNIFORM_STD, stream, N, buf, 0.0f, 1.0f);
vslDeleteStream(&stream);
```

[Source: oneMKL Developer Reference, PRNG chapter, https://www.intel.com/content/www/us/en/docs/onemkl/developer-reference-dpcpp/]

### 10.3 PBRT-v4 Sampler Implementations

PBRT-v4 (Pharr, Jakob, Humphreys 2023) provides several GPU-compatible sampler implementations as reference:

- **`IndependentSampler`**: per-pixel PCG state, one PCG advance per dimension. Simplest; highest variance.
- **`HaltonSampler`**: Halton sequence with per-pixel digit-scrambling. Good for low sample counts.
- **`ZSobolSampler`**: scrambled Sobol using the hashed Owen-scrambling variant, with the Z-order curve mapping 2D pixel positions to sequence indices. This is the recommended high-quality sampler.
- **`PMJ02BNSampler`**: progressive multi-jittered samples (Christensen et al. 2018); blue-noise properties across the image plane.

The PBRT Sobol direction numbers are embedded as a 10MB table constant (all 1024 dimensions). The `ZSobolSampler` GPU implementation runs as a CUDA/HIP kernel; each thread is one pixel, each sample dimension calls `SobolSample(mortonIndex, dim, scramble)`.

[Source: PBRT-v4, https://github.com/mmp/pbrt-v4; https://www.pbr-book.org/4ed/Sampling_and_Reconstruction/Halton_Sampler and /ZSobol_Sampler]

### 10.4 Vulkan-Native RNG Patterns

When a full library is unavailable or overhead is unacceptable, path tracers commonly embed a lightweight RNG directly in GLSL/HLSL. A robust pattern uses a PCG hash with push-constant seed:

```glsl
layout(push_constant) uniform PC {
    uint frame_seed;
} pc;

uint pcg_hash(uint v) {
    uint state = v * 747796405u + 2891336453u;
    uint word  = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
    return (word >> 22u) ^ word;
}

uint rng_seed_for_pixel(uvec2 px, uint frame) {
    return pcg_hash(px.x + pcg_hash(px.y + pcg_hash(frame + pc.frame_seed)));
}

float rand01(inout uint seed) {
    seed = pcg_hash(seed);
    return float(seed >> 8u) * (1.0 / float(1u << 24u));  // 24-bit float precision
}
```

**Subgroup shuffle for correlated sampling**: when adjacent pixels intentionally share the same sample (e.g. for checkerboard rendering or temporal upsampling), `subgroupShuffleXor(value, 1)` exchanges values between pairs of invocations within a subgroup without memory traffic.

---

## 11. Reproducibility and Debugging

### 11.1 Deterministic Replay with Fixed Seeds

A path tracer that maps `(pixel_x, pixel_y, frame_index, bounce, dimension)` to a deterministic RNG output is fully reproducible. The canonical mapping for counter-based RNGs:

```
key   = global_seed XOR frame_hash
count = pixel_y * width + pixel_x   (spatial index)
offset = bounce * MAX_DIMS + dimension_within_bounce
```

Store the global seed as a push constant or UBO. Changing the seed changes the noise pattern but not the expected value, enabling noise-based verification: render with seed=0 and seed=1; if the expected image is the same (within statistical error) the estimator is unbiased.

### 11.2 Thread-Index-to-Seed Mapping

A common error is hashing a 32-bit thread index with an identity function, giving sequential seeds {0,1,2,...}. For LCG and XORSHIFT, sequential seeds produce highly correlated streams. The fix: always pass the thread index through a mixing function before use as a seed:

```glsl
uint good_seed = pcg_hash(gl_GlobalInvocationID.x
                 + pcg_hash(gl_GlobalInvocationID.y
                 + pcg_hash(frame_index)));
```

### 11.3 Floating-Point Rounding in Sampling Transforms

`float(bits) / float(0xFFFFFFFFu)` returns a value in [0,1] inclusive — the value 1.0 is achievable and will cause `log(0)` = −∞ in Box-Muller. Use `float(bits >> 8u) * (1.0f / float(1u << 24u))` (24-bit mantissa precision) to guarantee the open interval (0,1). Alternatively, `max(1e-7, float(bits) * 2.3283064365386963e-10)` clamps away from zero.

### 11.4 NaN/Inf Guards in Path Tracers

Path tracers accumulate variance from rare outlier paths. `isnan()` or `isinf()` checks on the radiance sample before accumulation prevent fireflies from corrupting the reservoir weight or framebuffer. A maximum radiance clamp (e.g. 10× the scene average) is a common production guard; it introduces bias but eliminates fireflies. PBRT-v4 uses `MaxSafeAdd` logic to detect and discard NaN samples.

---

## 12. Integration with Rendering Pipelines

### 12.1 Feeding rocRAND Output into a Vulkan SSBO

rocRAND writes into HIP device memory; Vulkan can import that memory via `VK_KHR_external_memory`:

```c
// Allocate HIP memory with exportable handle
hipExternalMemoryHandleDesc extDesc = {};
extDesc.type = hipExternalMemoryHandleTypeOpaqueFd;
// (Use hipMallocManaged or VkDeviceMemory with external flag)

// Import in Vulkan
VkImportMemoryFdInfoKHR importInfo = {
    .sType      = VK_STRUCTURE_TYPE_IMPORT_MEMORY_FD_INFO_KHR,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT,
    .fd         = hip_fd
};
```

The practical alternative is to run rocRAND on the HIP side and map the SSBO backing store to a `hipDeviceptr_t` using `hipMemcpy` with `hipMemcpyDeviceToDevice` (zero-copy on unified memory systems such as APUs). On discrete GPUs without unified addressing, a single HIP-to-HIP copy of the random buffer suffices; at 4 bytes × 8M samples = 32 MB, this takes < 0.1 ms on PCIe 4×16.

### 12.2 Blue-Noise Texture Array for TAA

Temporal anti-aliasing jitters the subpixel sample each frame. A blue-noise texture array avoids temporal repetition:

```glsl
layout(set=0, binding=5) uniform sampler2DArray blue_noise_tex;  // 128x128x64 R8G8 UNORM

vec2 getJitter(ivec2 px, uint frame) {
    // Toroidal addressing with per-frame slice offset (modulo 64 slices)
    ivec3 coord = ivec3(px.x & 127, px.y & 127, frame & 63);
    vec2 bn = texelFetch(blue_noise_tex, coord, 0).rg;
    // Owen-scramble the blue-noise value with the frame index
    return fract(bn + float(frame) * 0.61803398875);  // golden ratio offset
}
```

[Cross-reference: Ch207 §Anti-Aliasing, TAA, and ML Upscaling covers the TAA pipeline that consumes this jitter.]

### 12.3 Sobol Sampler in a Vulkan Ray Tracing Pipeline

The Sobol direction numbers are stored in a read-only SSBO:

```glsl
layout(set=1, binding=0, std430) readonly buffer SobolDirectionNumbers {
    uint direction_numbers[1024][32];  // 1024 dimensions, 32 bits each
};

// In the ray generation shader:
uint sobol_sample(uint n, uint dim) {
    uint v = 0u;
    uint gray = n ^ (n >> 1u);  // Gray code for fast incremental update
    for (int i = 0; gray != 0u; i++, gray >>= 1u)
        if ((gray & 1u) != 0u) v ^= direction_numbers[dim][i];
    return v;
}

// Sample: dimension 0 for x, dimension 1 for y, dimensions 2+ for bounce, NEE, etc.
float u_x = float(sobol_sample(sample_index, 0)) * 2.3283064365386963e-10;
float u_y = float(sobol_sample(sample_index, 1)) * 2.3283064365386963e-10;
```

[Cross-reference: Ch206 §Complete Path Tracer Pipeline describes the full ray tracing shader structure that wraps these sample calls.]

### 12.4 Reservoir Buffer Layout for ReSTIR

ReSTIR requires two reservoir buffers (current and previous frame) and one temporal-accumulation buffer. Minimum SSBO layout:

```c
// Host-side VkBuffer creation
VkDeviceSize reservoir_size = width * height * sizeof(Reservoir);
// Reservoir = {uint y; float w_sum; uint M; float W} = 16 bytes

VkBufferCreateInfo buf_info = {
    .size  = 2 * reservoir_size,  // ping-pong
    .usage = VK_BUFFER_USAGE_STORAGE_BUFFER_BIT
};
```

The two reservoir buffers are ping-ponged between frames by swapping descriptor set bindings rather than copying memory. A third SSBO holds per-pixel visibility samples from the previous frame for spatial reuse bias correction.

---

## 13. Integrations

- **Ch205 (Shader Algorithm Catalog — Global Illumination and Materials)**: The GI techniques in Ch205 — stochastic lightmap baking, screen-space GI, world-space radiance cache — all consume the sampling infrastructure described here. The Sobol-based sampler (§4.3–4.4) is the recommended low-discrepancy source for multi-bounce lightmap baking; the ZSobolSampler pattern (§10.3) maps directly onto PBRT-v4 and Blender Cycles path-tracer samplers used for offline GI.

- **Ch206 (Shader Algorithm Catalog — Ray Tracing and Procedural Content)**: The complete path tracer pattern in Ch206 §Complete Path Tracer Pipeline uses the sampling infrastructure described here — cosine-weighted hemisphere sampling (§6.2), GGX VNDF (§6.3), NEE with MIS (§9.4), and ReSTIR (§7.2) — as its direct and indirect illumination estimators.

- **Ch221 (GPU Algorithm Performance and Optimization)**: RNG throughput is often memory-bandwidth-limited rather than compute-limited. The roofline analysis from Ch221 §1 applies directly: Philox-4×32-10 performs ~20 arithmetic operations per 16 output bytes, placing it near the ridgeline of current GPUs. Direction-number table lookups for Sobol are random-access reads and benefit from L1 texture cache described in Ch221 §3.

- **Ch226 (Linear Algebra for MCMC — matrix operations for HMC)**: the Hamiltonian Monte Carlo leapfrog integrator in §8.2 requires gradient computation `∇U(q)` which for Gaussian likelihood targets reduces to matrix-vector products; these are efficiently mapped to rocBLAS GEMV calls on AMD hardware, as described in Ch226.

- **Ch229 (Dropout and Data Augmentation Sampling)**: GPU data augmentation pipelines (random crops, flips, colour jitter) use the same per-thread PRNG infrastructure described here — specifically PCG per-thread state (§2.3) or Philox counter-based indexing (§3.2) with the data-item index as the counter, ensuring each augmented sample is independently and reproducibly randomised.
