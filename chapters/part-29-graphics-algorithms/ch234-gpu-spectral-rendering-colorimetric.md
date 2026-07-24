# Chapter 234: GPU Spectral Rendering and Colorimetric Algorithms (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Target audiences**: Graphics application developers implementing physically accurate colour in path tracers or display pipelines who need to understand how wavelength-resolved light transport differs from RGB rendering and how to connect spectral output to Linux HDR display infrastructure; systems developers building wide-colour and HDR rendering pipelines on AMD/NVIDIA/Intel hardware under Linux.

---

## Table of Contents

1. [Spectral vs Tristimulus Rendering — Why RGB Fails](#1-spectral-vs-tristimulus-rendering--why-rgb-fails)
2. [Hero-Wavelength Path Tracing](#2-hero-wavelength-path-tracing)
3. [Spectral Upsampling](#3-spectral-upsampling)
4. [Dispersion and Chromatic Effects](#4-dispersion-and-chromatic-effects)
5. [Fluorescence Simulation](#5-fluorescence-simulation)
6. [Polarised Light Rendering](#6-polarised-light-rendering)
7. [Spectral Sky and Atmosphere](#7-spectral-sky-and-atmosphere)
8. [CIE Colour Science on GPU](#8-cie-colour-science-on-gpu)
9. [Gamut Mapping and HDR Colorimetry](#9-gamut-mapping-and-hdr-colorimetry)
10. [Colour Space Conversion on GPU](#10-colour-space-conversion-on-gpu)
11. [Spectral Rendering in Production](#11-spectral-rendering-in-production)
12. [Integration with the Linux Display Pipeline](#12-integration-with-the-linux-display-pipeline)
13. [Integrations](#13-integrations)

---

## 1. Spectral vs Tristimulus Rendering — Why RGB Fails

### 1.1 Physical Light as a Spectral Power Distribution

Physical light is not a three-number tuple. A lamp, a diffuse reflectance, a flame — each emits or reflects energy continuously across wavelengths. A **spectral power distribution (SPD)** describes this: a function `L(λ)` giving radiant power per unit wavelength, measured in W·sr⁻¹·m⁻³ across the visible range (conventionally 360–830 nm). The human visual system collapses `L(λ)` to a three-number tristimulus value `[X, Y, Z]` through integration against the CIE 1931 2° standard observer colour matching functions — an irreversible projection.

### 1.2 Metamerism

Two SPDs `L₁(λ)` and `L₂(λ)` are **metamers** if they produce identical XYZ (and therefore identical RGB) under one illuminant but diverge under another. A pair of sRGB textures painted to match under D65 may appear visibly different under an incandescent lamp `A`. RGB path tracers relighting a scene can produce physically incorrect colour shifts for metameric pairs. Real-world situations where this matters: fabric/paint colour matching across daylight and tungsten, gem colours (alexandrite changes from green to red across illuminant), automotive finish appearance.

### 1.3 Fluorescence

Fluorescence involves **energy redistribution across wavelengths**: a molecule absorbs a photon at `λ_in` and emits at `λ_out > λ_in` (Stokes shift). An RGB renderer treats fluorescent materials as ordinary reflectors; it cannot represent the energy cross-talk between wavebands. Optical brighteners in paper and laundry detergent convert UV radiation into visible blue emission — making paper appear whiter than 100% reflectance. An RGB renderer cannot model this.

### 1.4 Dispersion

Glass disperses white light because its refractive index `n(λ)` varies with wavelength. A single-wavelength (RGB) refraction angle cannot simultaneously satisfy Snell's law at all wavelengths. Prisms, gemstones, and water droplets (rainbows) require wavelength-resolved IOR.

### 1.5 When Spectral Rendering Is Worth the Cost

Spectral rendering is computationally more expensive — a 4-wavelength path tracer costs roughly 4× the BSDF evaluations, though some overhead amortises across the path. The fidelity premium is visible when:

- Dispersive materials (glass, gems, water) are present
- Fluorescent materials appear under varying illuminants
- Accurate material appearance under mixed or coloured lighting is required
- The output will be tone-mapped or colour-graded with illuminant-relative colour accuracy

Film VFX and physically-based product visualisation regularly warrant spectral rendering. Interactive games under rasterisation generally do not. [Source: "Hero Wavelength Spectral Sampling", Wilkie et al., Eurographics 2014, https://cgg.mff.cuni.cz/~wilkie/Website/Hero_Wavelength_Spectral_Sampling.html]

### 1.6 Linux GPU Spectral Rendering Landscape

On Linux, spectral rendering runs primarily through:

- **PBRT-v4** — GPU path tracer using CUDA (NVIDIA) or wavefront optix backend; spectral mode is the default
- **Mitsuba 3** — Dr.Jit JIT-compiled renderer with `cuda_ad_spectral` and `llvm_ad_spectral` variants
- Custom Vulkan/GLSL or HIP/CUDA compute kernels implementing hero-wavelength sampling

No mainline Linux rasterisation pipeline performs spectral rendering; spectral algorithms live exclusively in offline or research-oriented GPU compute.

---

## 2. Hero-Wavelength Path Tracing

### 2.1 Single-Wavelength Sampling

**Hero-wavelength path tracing** (Wilkie et al. 2014) assigns one primary wavelength `λ` per ray, sampled uniformly from [360, 830] nm or importance-sampled from the CIE luminous efficiency function `V(λ)`. Each ray carries a scalar radiance estimate for that wavelength. The SPD estimator integrates to a full spectral image across many samples: pixel `p`'s SPD is reconstructed by accumulating `L(λ_i)` at each sample's wavelength into a histogram or dense spectral buffer.

### 2.2 Multi-Wavelength SIMD Extension

PBRT-v4 extends this to **four simultaneous wavelengths** (`NSpectrumSamples = 4`, range 360–830 nm), exploiting SIMD width. `SampledWavelengths` stores two parallel arrays of length 4: `lambda[4]` and `pdf[4]`. `SampledSpectrum` stores the corresponding radiance values `values[4]`. All BSDF evaluations, transmittance queries, and emission lookups produce `SampledSpectrum` objects evaluated at all four wavelengths simultaneously. [Source: pbrt-v4 `src/pbrt/util/spectrum.h`, https://github.com/mmp/pbrt-v4/blob/master/src/pbrt/util/spectrum.h]

### 2.3 Wavelength Sampling Strategies

```cpp
// pbrt-v4: sample wavelengths once per path, uniform across [360, 830] nm
SampledWavelengths lambda = film.SampleWavelengths(sampler.Get1D());
// lambda.lambda[] contains 4 wavelengths; lambda.pdf[] contains their PDFs
```

`SampleUniform()` draws wavelengths from a uniform distribution on [360, 830] nm. `SampleVisible()` importance-samples from the CIE V(λ) luminous efficiency, concentrating samples where the eye is most sensitive; this reduces luminance variance but requires reweighting by the ratio `V(λ)/pdf_visible(λ)`.

### 2.4 MIS Across Wavelength Samples

In a multi-wavelength path tracer with `N` simultaneous hero wavelengths `{λ₁, ..., λ_N}`, each wavelength sample can be treated as an independent Monte Carlo estimator. The **balance heuristic** applied across wavelengths weights each sample's contribution by its share of the total wavelength PDF:

```
w_j(λ) = p_j(λ) / Σ_k p_k(λ)
```

where `p_j(λ)` is the probability that the `j`-th sampling strategy would have selected wavelength `λ`. For uniform sampling, all `p_k` are equal and the weights collapse to `1/N`. For a mixed strategy (some hero wavelengths sampled uniformly, others importance-sampled from V(λ)), the balance heuristic correctly downweights the importance-sampled wavelengths at regions where they are over-represented relative to the full PDF.

This extends Multiple Importance Sampling (Ch227) from spatial/directional strategies to the wavelength dimension: each spectral channel is treated as a separate sampling technique, and their estimators are combined with the balance heuristic to reduce variance across the spectrum. [Source: Wilkie et al., "Hero Wavelength Spectral Sampling", Eurographics 2014, https://cgg.mff.cuni.cz/~wilkie/Website/Hero_Wavelength_Spectral_Sampling.html]

### 2.5 Hero Wavelength Correlation Across Bounces

The key insight of hero-wavelength sampling is **wavelength consistency across path bounces**: once `λ` is drawn at the camera, the same four wavelengths propagate through every bounce, scattered by the same geometry, filtered by the same textures. This is necessary for unbiased estimation — if wavelengths changed at each bounce the estimator would be biased. The `lambda` object is passed by reference through every `Li()` recursive call in pbrt-v4 unchanged.

### 2.6 Secondary Wavelength Termination

At specularly transmitted surfaces (e.g. inside a dispersive glass), only one wavelength can follow the exact refracted direction. PBRT-v4 calls `lambda.TerminateSecondary()` in volumetric integrators to drop secondary wavelengths and focus estimator power on the primary hero wavelength, reducing variance from secondary-wavelength contamination.

### 2.7 SPD Storage Formats

| Format | Size | GPU access | Use case |
|---|---|---|---|
| Dense tabulated (1 nm step) | 471 floats (360–830 nm) | 1D texture, linear interp | Measured illuminants, materials |
| Piecewise linear | N×2 floats | CPU-side, upload as buffer | Sparse measured data |
| Sigmoid polynomial | 3 floats | Register arithmetic | RGB-upsampled spectra (§3) |
| Gaussian mixture | 2K floats (K peaks) | Shader arithmetic | Fluorescent emission |

PBRT-v4's `DenselySampledSpectrum` stores 471 floats at 1 nm resolution; at runtime the GPU queries it via interpolated texture fetch.

---

## 3. Spectral Upsampling

### 3.1 The Problem

Most asset pipelines deliver surface colours as sRGB or linear RGB triples. A spectral path tracer requires a physically plausible SPD for every material. **Spectral upsampling** converts an RGB triple to a smooth SPD that integrates back to the original RGB under the given illuminant and observer. Because the projection from spectrum to XYZ is many-to-one, infinitely many spectra are valid; the goal is to choose one that is smooth (physically plausible) and GPU-friendly.

### 3.2 Jakob–Hanika Sigmoid Basis

Jakob and Hanika (2019) represent the upsampled spectrum as a **sigmoid polynomial**:

```
S(λ) = 1 / (1 + exp(-(c₀·λ² + c₁·λ + c₂)))
```

Three coefficients `(c₀, c₁, c₂)` parameterise a smooth, bounded spectrum in [0, 1]. Offline optimisation finds the coefficient triple that minimises the L² error between the tristimulus values of `S(λ)` and the target RGB. The resulting coefficients are stored in a **64×64×64×3 float 3D LUT** indexed by the sorted RGB components (max-channel remapping), enabling GPU trilinear lookup:

```glsl
// GLSL: evaluate sigmoid polynomial at wavelength lambda (nm)
vec3 c = texture(rgbToSpectrumLUT, normalizedRGB).rgb;
float x = (c.x * lambda + c.y) * lambda + c.z; // Horner form
float S = 1.0 / (1.0 + exp(-x));
```

[Source: "A Low-Dimensional Function Space for Efficient Spectral Upsampling", Jakob & Hanika, Eurographics 2019, https://rgl.epfl.ch/publications/Jakob2019Spectral]

PBRT-v4's `RGBToSpectrumTable` implements this: `res = 64`, `CoefficientArray` typed as `float[3][64][64][64][3]`, with four colour-space variants (sRGB, DCI-P3, Rec.2020, ACES2065-1). The GPU upload path (`Init()`) allocates the table in GPU-accessible memory.

### 3.3 Mallett–Yuksel Smooth Metamers

Mallett and Yuksel (2019) take a different approach: they define a **smooth basis spectrum** for each primary colour (red, green, blue) such that their linear combination is always a smooth metamer of the input RGB. The basis functions are constrained to be positive and to integrate correctly under the CIE 1931 observer. This approach is closed-form at runtime (no LUT required), making it suitable for GPU immediate evaluation:

```glsl
float spectralUpMallettYuksel(float r, float g, float b, float lambda) {
    // Linear combination of smooth basis functions R(λ), G(λ), B(λ)
    return r * basisR(lambda) + g * basisG(lambda) + b * basisB(lambda);
}
```

The basis functions are tabulated as 1D textures (471 samples at 1 nm resolution). The method guarantees smooth output spectra but does not minimise upsampling error under specific illuminants. [Source: "Spectral Primary Decomposition for Rendering with sRGB Reflectance", Mallett & Yuksel, EGSR 2019, https://doi.org/10.1111/cgf.13332]

### 3.4 Accuracy Trade-off

Both methods produce spectra that integrate to the correct XYZ under D65. They diverge under other illuminants — this residual error is the fundamental cost of spectral upsampling. For measured materials (gemstones, paints, textiles) where measured SPDs are available, using the measured data directly is always preferable.

---

## 4. Dispersion and Chromatic Effects

### 4.1 Cauchy Dispersion Equation

The wavelength-dependence of refractive index in optical glass is approximated by the **Cauchy equation**:

```
n(λ) = A + B/λ²
```

where `λ` is in micrometres. Constants `A` and `B` characterise the glass type. For BK7 (borosilicate crown): `A ≈ 1.5220`, `B ≈ 0.00459 μm²`. In a GPU spectral path tracer, `A` and `B` are uploaded as push constants; each wavelength sample computes its own IOR at shader invocation time.

### 4.2 Sellmeier Equation

The **Sellmeier equation** gives higher accuracy for precision optics:

```
n²(λ) = 1 + B₁λ²/(λ²-C₁) + B₂λ²/(λ²-C₂) + B₃λ²/(λ²-C₃)
```

Three term-pairs `(Bᵢ, Cᵢ)` are tabulated for each glass type in optical databases (Schott, Ohara). The GPU evaluates this per wavelength as six multiply-add operations. [Source: Born & Wolf, "Principles of Optics", 7th ed., §2.3]

### 4.3 GPU Refraction with Per-Hero-Wavelength Angle

In a hero-wavelength path tracer, each wavelength component in `SampledSpectrum` refracts at a **different angle** via Snell's law:

```glsl
// Per-wavelength refraction inside a dispersive material
float ior = cauchyA + cauchyB / (wavelength_um * wavelength_um);
vec3 refractDir = refract(rayDir, normal, eta_prev / ior);
```

For four simultaneous wavelengths, this yields four different refracted directions. In practice, a single geometric ray traces one direction (the hero wavelength), and secondary wavelengths accumulate error unless explicit multi-ray tracing is used. High-quality dispersion requires separate ray paths per wavelength, multiplying ray count by `NSpectrumSamples`.

### 4.4 Chromatic Aberration as Post-Process Approximation

For rasterisation or offline post-processing, chromatic aberration is approximated without spectral rays:

**Lateral CA** (colour fringing at image edges): shift red and blue channels in opposite screen-space directions proportional to distance from image centre:

```glsl
float dist = length(uv - 0.5);
vec2 caOffset = (uv - 0.5) * caStrength * dist;
float r = texture(hdrBuffer, uv + caOffset).r;
float g = texture(hdrBuffer, uv).g;
float b = texture(hdrBuffer, uv - caOffset).b;
```

**Longitudinal CA** (channel-dependent focus): apply per-channel depth-of-field blur with slightly different circle-of-confusion radius per colour channel. Both are approximations with no physical accuracy for strongly dispersive media.

---

## 5. Fluorescence Simulation

### 5.1 Bispectral BRDF and the Reradiation Matrix

Standard BRDFs map incident radiance at wavelength `λ` to reflected radiance at the same `λ`. Fluorescence couples wavelengths: energy arriving at `λ_in` may be re-emitted at `λ_out`. The **bispectral BRDF** (or reradiation matrix) `R(λ_in, λ_out)` encodes this coupling. For a purely fluorescent material:

```
L_r(λ_out) = ∫ R(λ_in, λ_out) · L_i(λ_in) dλ_in
```

For non-fluorescent reflectors, `R` is diagonal: `R(λ_in, λ_out) = f_r(λ) · δ(λ_in - λ_out)`. Fluorescence populates off-diagonal terms where `λ_out > λ_in` (the Stokes shift — emission at longer, lower-energy wavelength).

[Source: Hullin et al., "Fluorescent Immersion Range Scanning", SIGGRAPH 2008; Wilkie et al., "An Analytical Model for the Scattering of Polarised Light in Biological Tissue", EGWR 2001]

### 5.2 GPU Bispectral Path Tracer

A full bispectral implementation requires integrating over `λ_in` for each outgoing `λ_out`, turning the BSDF evaluation into a 1D integral. For a spectral path tracer with `NSpectrumSamples = 4` discrete wavelengths, this inner integral becomes a 4×4 matrix-vector product:

```glsl
// Fluorescent contribution: R[lambda_out][lambda_in] * L_in[lambda_in]
vec4 L_fluor = mat4(R) * L_incident; // R is 4x4 bispectral coupling matrix
```

The coupling matrix `R` is built offline from measured bispectral data (typically measured with a bispectrophotometer) and stored as a 4×4 float texture or push constant.

### 5.3 Two-Level Practical Simplification

A widely adopted simplification models a fluorescent material as:
1. An **absorption SPD** `A(λ)` describing what wavelengths are absorbed
2. An **emission SPD** `E(λ)` describing the re-emitted distribution (typically a smooth peak at the Stokes-shifted wavelength)
3. A **quantum yield** `Φ ∈ [0, 1]` representing the fraction of absorbed energy that is re-emitted

```glsl
float absorbed = A(lambda_in) * L_incident;
float emitted  = Phi * absorbed * E(lambda_out); // summed over lambda_in
```

This is not fully bispectral but captures the primary perceptual effect: optical brighteners in paper emit strongly in blue (~440 nm) when excited by near-UV (~360 nm), producing apparent reflectance above 100% in blue.

---

## 6. Polarised Light Rendering

### 6.1 Stokes Vector Representation

The polarisation state of a light beam is described by the **Stokes vector**:

```
S = [I, Q, U, V]ᵀ
```

where `I` is total intensity, `Q` measures linear polarisation along the horizontal/vertical axis, `U` measures linear polarisation at ±45°, and `V` measures circular polarisation (right/left hand). All components have units of radiance. Unpolarised light has `Q = U = V = 0`.

### 6.2 Mueller Matrix Calculus

Each optical interaction transforms the Stokes vector via a **4×4 Mueller matrix** `M`:

```
S_out = M · S_in
```

Key Mueller matrices:
- **Dielectric Fresnel reflection** (at angle of incidence `θᵢ`, IOR ratio `η`): entries derived from Fresnel `rs` and `rp` coefficients; see Born & Wolf §1.5
- **Metallic reflection**: complex IOR `ñ = n + ik`; Mueller matrix is non-diagonal due to phase shift between s/p components
- **Rayleigh scattering** (atmosphere): produces strong polarisation perpendicular to the scattering plane

### 6.3 GPU Polarisation Path Tracer

A polarised path tracer carries a 4-component Stokes vector instead of a scalar or RGB radiance. Per-bounce overhead is one 4×4 matrix-vector multiply (16 FMAs):

```glsl
// Per-bounce polarisation update
vec4 stokesOut = mullerMatrix * stokesIn;
float I = stokesOut.x; // extract intensity for next-event estimation
```

The Mueller matrix for each surface is constructed from Fresnel coefficients at runtime. Mitsuba 3 provides the `cuda_ad_spectral_polarized` variant which combines spectral sampling with full Mueller calculus. [Source: Mitsuba 3 documentation, https://mitsuba.readthedocs.io/en/stable/; Collett, "Field Guide to Polarization", SPIE 2005]

### 6.4 Applications

Polarisation rendering is needed for: glare from wet roads (Brewster angle suppresses s-polarisation), LCD screen simulation (crossed polariser + liquid crystal), rainbow simulation (Rayleigh/Mie scattering polarisation controls bow brightness), and thin-film iridescence (See Ch205 for thin-film interference; Mueller calculus extends it to handle polarisation correctly).

---

## 7. Spectral Sky and Atmosphere

### 7.1 Preetham Analytic Sky Model

The **Preetham model** (1999) gives analytic sky luminance as a function of sun-observer-zenith geometry, parameterised by **turbidity** `T` (1 = clear, 10 = hazy). Five Perez coefficients `(A, B, C, D, E)` are fitted as linear functions of `T`. GPU evaluation is a few multiply-add operations per sky pixel:

```glsl
float perezF(float theta, float gamma, float A, float B, float C, float D, float E) {
    return (1.0 + A*exp(B/cos(theta))) * (1.0 + C*exp(D*gamma) + E*cos2(gamma));
}
float Y = Yz * perezF(theta_s, gamma, A_Y, B_Y, C_Y, D_Y, E_Y)
             / perezF(0.0,     gamma, A_Y, B_Y, C_Y, D_Y, E_Y);
```

The model is tristimulus-only (Yxy), not spectral. [Source: Preetham et al., "A Practical Analytic Model for Daylight", SIGGRAPH 1999, https://doi.org/10.1145/311535.311553]

### 7.2 Hosek–Wilkie Sky Model

Hosek and Wilkie (2012) extend the Preetham model with improved solar disc representation and optionally a **spectral dataset** of discrete wavelength channels (precomputed as a large C array). The CPU-side reference API allocates a state struct, queries per-channel sky radiance at a given sun elevation, turbidity, and ground albedo, and must be freed after use. For GPU use, the coefficient dataset is uploaded as a texture (turbidity × albedo × 9 Perez-like coefficients per channel); the shader interpolates and evaluates the polynomial at each hero wavelength. [Source: Hosek & Wilkie, "An Analytic Model for Full Spectral Sky-Dome Radiance", SIGGRAPH 2012, https://doi.org/10.1145/2185520.2185564; reference implementation: https://cg.mff.cuni.cz/projects/SkylightModelling/]

> **Note:** The exact channel count, wavelength range, and current `ArHosekSkyModelState` API function signatures of the reference implementation should be verified against the published source at the URL above before use in production code.

### 7.3 Nishita Atmosphere (Ray Marching)

Nishita et al. (1993) model atmospheric scattering by ray-marching a ray through the atmosphere, accumulating **Rayleigh scattering** (from air molecules, λ⁻⁴ wavelength dependence) and **Mie scattering** (from aerosols, weakly wavelength-dependent) at each step:

```glsl
vec3 rayleighCoeff = vec3(5.8e-6, 13.5e-6, 33.1e-6); // per nm, RGB channels approximate wavelengths
// Spectral version uses per-wavelength β_R(λ) = (8π³(n²-1)²) / (3Nλ⁴)
```

The λ⁻⁴ Rayleigh term makes blue sky: short wavelengths scatter far more than red. A spectral path tracer evaluates this at each hero wavelength, naturally reproducing the reddening of sunsets.

[Source: Nishita et al., "Display of the Earth Taking into Account Atmospheric Scattering", SIGGRAPH 1993, https://doi.org/10.1145/166117.166140]

### 7.4 Bruneton–Neyret Precomputed Scattering

Bruneton and Neyret (2008) precompute atmospheric scattering into several LUTs:
- **Transmittance LUT** `T(h, cosθ)`: how much light reaches altitude `h` at angle `cosθ` to zenith
- **Single-scattering LUT**: in-scattered radiance from sun, indexed by (height, view angle, sun angle)
- **Multiple-scattering accumulation**: iterative passes accumulate 2nd, 3rd, ... order scattering

The precomputation runs as GPU compute passes once per set of atmosphere parameters. Runtime sky rendering is a LUT lookup with interpolation:

```glsl
vec3 transmittance = texture(transmittanceLUT, vec2(uHeight, uCosTheta)).rgb;
vec3 scatter = texture(scatteringLUT, vec3(uHeight, uViewAngle, uSunAngle)).rgb;
vec3 L = scatter + transmittance * sunRadiance;
```

[Source: Bruneton & Neyret, "Precomputed Atmospheric Scattering", EGSR 2008, https://doi.org/10.1111/j.1467-8659.2008.01245.x; open-source implementation: https://github.com/ebruneton/precomputed_atmospheric_scattering]

A spectral extension samples Rayleigh and Mie coefficients per wavelength, requiring precomputed LUTs per wavelength channel — typically 10–30 channels for acceptable spectral accuracy.

---

## 8. CIE Colour Science on GPU

### 8.1 CIE 1931 2° Standard Observer

The **CIE 1931 2° standard observer** defines three colour matching functions `x̄(λ)`, `ȳ(λ)`, `z̄(λ)` tabulated at 1 nm intervals from 360–830 nm (471 samples). On GPU, these are stored as three float16 1D textures (or one RGB texture), queried with linear interpolation at the path tracer's hero wavelengths.

### 8.2 Spectral-to-XYZ Quadrature on GPU

To convert a sampled spectral estimate to XYZ, the path tracer uses Monte Carlo quadrature. For each hero wavelength sample `λᵢ` with probability `p(λᵢ)`:

```
X ≈ (1/N) Σᵢ [ x̄(λᵢ) · L(λᵢ) / p(λᵢ) ] · CIE_Y_integral
```

In PBRT-v4, `SampledSpectrum::ToXYZ()` implements this: it multiplies the four spectrum values element-wise by the CMF values at the four wavelengths, divides by the PDF, and sums, scaled by `CIE_Y_integral = 106.856895`.

```cpp
// pbrt-v4: SampledSpectrum::ToXYZ()
XYZ xyz;
for (int i = 0; i < NSpectrumSamples; ++i) {
    xyz.x += Xbar(lambda[i]) * values[i] / pdf[i];
    xyz.y += Ybar(lambda[i]) * values[i] / pdf[i];
    xyz.z += Zbar(lambda[i]) * values[i] / pdf[i];
}
return xyz * CIE_Y_integral / NSpectrumSamples;
```

[Source: pbrt-v4 `src/pbrt/util/spectrum.cpp`, https://github.com/mmp/pbrt-v4/blob/master/src/pbrt/util/spectrum.cpp]

### 8.3 CIE L\*a\*b\* and L\*u\*v\*

From XYZ, perceptual colour spaces are computed:

```glsl
// CIE L*a*b* (D50 whitepoint: Xn=0.9642, Yn=1.0, Zn=0.8249)
float f(float t) { return t > 0.008856 ? pow(t, 1.0/3.0) : 7.787*t + 16.0/116.0; }
float L = 116.0 * f(Y/Yn) - 16.0;
float a = 500.0 * (f(X/Xn) - f(Y/Yn));
float b = 200.0 * (f(Y/Yn) - f(Z/Zn));
```

L\*u\*v\* uses a different chromatic encoding based on u' v' chromaticity coordinates, better suited for luminance-based colour difference metrics.

### 8.4 CIE 2015 Cone Fundamentals and Metamerism Index

The CIE 2015 standard (10° observer) updates the CMFs with cone fundamentals based on physiological measurements. The **metamerism index** `MI` quantifies how much two metameric pairs diverge under a test illuminant relative to a reference; GPU evaluation computes the CMF-weighted integrals for both spectra and measures their CIELAB distance.

### 8.5 Colour Temperature on GPU

**Planckian radiator** spectral emission at temperature `T` (K):

```glsl
float planckian(float lambda_nm, float T) {
    float h = 6.626e-34, c = 2.998e8, k = 1.381e-23;
    float l = lambda_nm * 1e-9;
    return (2.0 * h * c * c) / (pow(l, 5.0) * (exp(h * c / (l * k * T)) - 1.0));
}
```

**Correlated Colour Temperature (CCT)** is found via Robertson's method: project the chromaticity `(u', v')` onto the Planckian locus and interpolate the two nearest tabulated isothermal lines. [Source: Robertson, "Computation of Correlated Color Temperature and Distribution Temperature", JOSA 1968]

---

## 9. Gamut Mapping and HDR Colorimetry

### 9.1 Colour Appearance Models: ZCAM and CAM16

Colour appearance models (CAMs) predict perceived colour attributes — lightness `J`, colourfulness `C`, hue `h` — under varying viewing conditions (adapting luminance, surround, background). **CAM16** (Li et al. 2017) and **ZCAM** (Safdar et al. 2021) extend CIECAM02 with improved hue uniformity and HDR range.

For gamut mapping, these models provide **perceptually uniform** coordinates in which to apply compression: compressing chroma `C` at constant lightness `J` and hue `h` produces gamut-mapped colours that preserve appearance better than clipping in RGB.

[Source: Li et al., "Comprehensive colour appearance model (CAM16)", Color Research & Application, 2017; Safdar et al., "ZCAM, a colour appearance model based on a high dynamic range uniform colour space", Optics Express 2021]

### 9.2 Gamut Boundary Descriptor

The **gamut boundary descriptor (GBD)** maps out the maximum achievable chroma at each lightness–hue combination for a target colour space (BT.2020, DCI-P3, sRGB). On GPU, the GBD is stored as a 2D texture indexed by `(J, h)`, returning the maximum `C_max`. Perceptual gamut mapping then scales chroma:

```glsl
float C_max = texture(gamutBoundary, vec2(J / J_max, h / 360.0)).r;
float C_mapped = C_max * (1.0 - exp(-C / C_max)); // soft-knee compression
```

### 9.3 HDR Tone Mapping with Gamut Mapping

Production pipelines combine tone mapping (scene-linear → display-referred) with gamut mapping (wide gamut → display gamut). Common GPU implementations:

- **ACES RRT+ODT**: reference rendering transform from scene-linear through the ACES AP0/AP1 gamut into display-specific output device transforms. Implemented as a sequence of 3×3 matrix multiplications and a filmic tone curve.
- **AgX** (Blender's current tone mapper): maps scene-linear into a log encoding, applies a sigmoid-shaped tone curve per channel, then converts to sRGB. Minimal gamut mapping; relies on the sigmoid to desaturate naturally at high luminance.
- **Filmic** (older Blender): polynomial filmic curve in scene-linear.

[Source: ACES documentation, https://docs.acescentral.com/; Blender AgX implementation, https://github.com/blender/blender/blob/main/release/datafiles/colormanagement/config.ocio]

### 9.4 PQ and HLG Transfer Functions on GPU

```glsl
// ST.2084 Perceptual Quantizer (PQ) EOTF: nits → linear
float pqEOTF(float V) {
    const float m1 = 0.1593017578125, m2 = 78.84375;
    const float c1 = 0.8359375, c2 = 18.8515625, c3 = 18.6875;
    float Vp = pow(V, 1.0/m2);
    return pow(max(0.0, Vp - c1) / (c2 - c3*Vp), 1.0/m1) * 10000.0; // nits
}

// HLG OETF: linear → encoded
float hlgOETF(float E) {
    const float a=0.17883277, b=0.28466892, c_hlg=0.55991073;
    return E <= 1.0/12.0 ? sqrt(3.0*E) : a*log(12.0*E-b)+c_hlg;
}
```

[Source: SMPTE ST.2084-2014; ITU-R BT.2100-2]

---

## 10. Colour Space Conversion on GPU

### 10.1 Matrix Transforms

All linear colour space conversions are 3×3 matrix multiplications. Common matrices:

```glsl
// sRGB (D65) → CIE XYZ
const mat3 sRGBtoXYZ = mat3(
    0.4124564, 0.3575761, 0.1804375,
    0.2126729, 0.7151522, 0.0721750,
    0.0193339, 0.1191920, 0.9503041
);
// BT.2020 (D65) → XYZ
const mat3 BT2020toXYZ = mat3(
    0.6369580, 0.1446169, 0.1688810,
    0.2627002, 0.6779981, 0.0593017,
    0.0000000, 0.0280727, 1.0609851
);
// DCI-P3 (D65) → XYZ
const mat3 P3D65toXYZ = mat3(
    0.4865709, 0.2656677, 0.1982173,
    0.2289746, 0.6917385, 0.0792869,
    0.0000000, 0.0451134, 1.0439444
);
```

These are uploaded as push constants or UBO members; the GPU applies them as 9-FMA operations per pixel. [Source: IEC 61966-2-1 (sRGB), ITU-R BT.2020, SMPTE ST 431-2 (DCI-P3)]

### 10.2 Non-Linear Transfer Functions

```glsl
// sRGB piecewise EOTF (encoded → linear)
float srgbEOTF(float C) {
    return C <= 0.04045 ? C / 12.92 : pow((C + 0.055) / 1.055, 2.4);
}
// Gamma 2.2 approximation (fast, no branch)
float gamma22(float C) { return pow(C, 2.2); }
```

For PQ and HLG see §9.4. These are pure arithmetic; prefer the piecewise form over polynomial approximations when precision matters.

### 10.3 OKLab and OKLCH

**OKLab** (Ottosson, 2020) provides a perceptually uniform opponent-colour space optimised for interpolation and blending:

```glsl
vec3 linearSRGBtoOKLab(vec3 c) {
    // Step 1: approximate cone response (LMS)
    vec3 lms = mat3(0.4122214708, 0.5363325363, 0.0514459929,
                    0.2119034982, 0.6806995451, 0.1073969566,
                    0.0883024619, 0.2817188376, 0.6299787005) * c;
    // Step 2: cube root non-linearity
    lms = sign(lms) * pow(abs(lms), vec3(1.0/3.0));
    // Step 3: opponent encoding
    return mat3(0.2104542553, 0.7936177850, -0.0040720468,
                1.9779984951, -2.4285922050, 0.4505937099,
                0.0259040371, 0.7827717662, -0.8086757660) * lms;
}
```

**OKLCH** adds polar coordinates `(L, C, h)` in OKLab. [Source: Björn Ottosson, "A perceptual color space for image processing", 2020, https://bottosson.github.io/posts/oklab/]

### 10.4 ICtCp

**ICtCp** (BT.2100 Annex 2) is designed for HDR perceptual uniformity, separating intensity `I` from chromatic components `Ct` (tritan), `Cp` (protan). It uses the PQ transfer function as its non-linearity and is the recommended space for HDR colour difference measurement.

### 10.5 Bradford Chromatic Adaptation

**Chromatic adaptation** converts tristimulus values between different whitepoints. The **Bradford transform** (ICC standard) maps XYZ under source illuminant to XYZ under destination illuminant:

```
XYZ_D = M_Bradford⁻¹ · diag(ρ_D/ρ_S, γ_D/γ_S, β_D/β_S) · M_Bradford · XYZ_S
```

where `(ρ, γ, β)` are the Bradford sharpened cone responses of each whitepoint. On GPU this collapses to a single precomputed 3×3 matrix. CAT16 (used in CAM16 and ZCAM) uses an updated adaptation matrix with improved performance for saturated colours.

### 10.6 GPU 3D LUT Tetrahedral Interpolation

Complex colour transforms (camera LUTs, display calibration) are stored as 3D LUTs (e.g. 33×33×33 RGB entries). **Tetrahedral interpolation** (Sakamoto, 1973) is more accurate than trilinear for colour transforms:

```glsl
// Determine which tetrahedron the input falls in from fractional rgb offsets
vec3 f = fract(rgb_indexed);
// Six tetrahedral regions based on ordering of f.r, f.g, f.b
// Each computes output as weighted sum of 4 LUT corner values
```

Tetrahedral interpolation is the standard in GPU-side colour management (OpenColorIO's GPU path, DaVinci Resolve's GPU LUT engine). [Source: OpenColorIO GPU renderer, https://github.com/AcademySoftwareFoundation/OpenColorIO/blob/main/src/OpenColorIO/GpuShaderUtils.cpp]

---

## 11. Spectral Rendering in Production

### 11.1 PBRT-v4 Spectral Architecture

PBRT-v4 is the reference spectral path tracer on Linux. Its spectral architecture:

- **`SampledWavelengths`**: four wavelengths + PDFs per path; allocated once per pixel sample and propagated through the full path
- **`SampledSpectrum`**: four-component array of radiance values, one per wavelength
- **`SpectrumTexture`**: evaluates any spectrum representation (tabulated, blackbody, RGB-upsampled via `RGBToSpectrumTable`) at the four hero wavelengths
- **`RGBToSpectrumTable`**: 64×64×64 × 3 float coefficient array stored in GPU-accessible memory, queried to produce `RGBSigmoidPolynomial` coefficients for asset upsampling

Four colour space tables are precomputed (sRGB, DCI-P3, Rec.2020, ACES2065-1). The GPU wavefront integrator (CUDA or OptiX backend) processes full path throughput as `SampledSpectrum` at every bounce.

[Source: pbrt-v4 source, https://github.com/mmp/pbrt-v4; Pharr, Jakob, Humphreys, "Physically Based Rendering: From Theory to Implementation", 4th ed., 2023]

### 11.2 Mitsuba 3 Spectral Variants

**Mitsuba 3** uses **Dr.Jit** for JIT-compiled kernel generation. Spectral variants:

| Variant | Backend | Autodiff |
|---|---|---|
| `llvm_ad_spectral` | LLVM / CPU vectorised | Yes |
| `llvm_ad_spectral_polarized` | LLVM / CPU vectorised | Yes |
| `cuda_ad_spectral` | NVIDIA CUDA | Yes |
| `cuda_ad_spectral_polarized` | NVIDIA CUDA | Yes |

The `_polarized` variants carry a full Mueller matrix instead of a scalar transmittance. Dr.Jit traces Python-level rendering code into GPU kernels at first execution, then caches the compiled result. For spectral rendering, each variable (spectrum, wavelength PDF) becomes a traced tensor over wavelength channels. [Source: Mitsuba 3 GitHub, https://github.com/mitsuba-renderer/mitsuba3]

### 11.3 Blender Cycles

Blender Cycles is an RGB-based Monte Carlo path tracer. **It does not have a mainline spectral mode** as of 2026. Experimental community patches implementing hero-wavelength sampling exist but are not merged into upstream. Cycles uses CUDA and HIP for GPU rendering (NVIDIA and AMD respectively), oneAPI for Intel Arc, and Metal on macOS; OpenCL support was removed in Blender 3.0. For spectral-accurate rendering in Blender, third-party renderers (Mitsuba 3 via a Blender bridge, or LuxCoreRender which has spectral support) must be used.

[Source: Blender documentation, https://docs.blender.org/manual/en/latest/render/cycles/; LuxCoreRender spectral mode, https://luxcorerender.org/]

### 11.4 When Spectral Rendering Visibly Differs

Spectral rendering produces visibly different output from RGB in these scenarios:

1. **Dispersive glass and gemstones** — RGB cannot separate wavelength-dependent refraction; colour fringes and caustic rainbows are unphysical without spectral tracing
2. **Fluorescent paints and optical brighteners** — RGB clamps reflectance to [0, 1]; fluorescence can emit more than it receives in a waveband
3. **Coloured glass under white light** — sequential filters multiply spectral transmittance; RGB multiplication is an approximation that holds only for broadband spectra
4. **Illuminant metamerism** — fabrics and paints matched under D65 may diverge under A; visible only when illuminant changes

---

## 12. Integration with the Linux Display Pipeline

### 12.1 Spectral Renderer Output to Display

A spectral path tracer accumulates per-pixel spectral histograms or reconstructs the full SPD. The final step converts to a display-referred signal:

1. **Spectral → XYZ**: Monte Carlo quadrature over wavelength samples (§8.2)
2. **XYZ → display primaries** (BT.2020 for HDR10): 3×3 matrix (§10.1)
3. **Tone mapping**: map scene-linear XYZ or display-linear values into the display peak luminance range
4. **EOTF encoding**: apply PQ (for HDR10/ST.2084) or HLG OETF to produce encoded signal

### 12.2 DRM/KMS HDR Metadata

The encoded HDR frame is passed to the compositor, which sets the KMS connector property **`hdr_output_metadata`**. This property carries a `hdr_output_metadata` struct populated with:
- `hdr_metadata_type` = `HDMI_STATIC_METADATA_TYPE1`
- Mastering display primaries and white point (SMPTE ST 2086)
- `max_cll` (MaxCLL, maximum content light level, cd/m²) — CTA 861.3
- `max_fall` (MaxFALL, maximum frame-average light level, cd/m²) — CTA 861.3

These values inform the display's internal tone mapping. For a spectral path tracer, `max_cll` should be derived from the maximum luminance in the rendered frame. See Ch158 for the full `drm_hdr_output_metadata` kernel struct layout and wire signalling over HDMI/DisplayPort.

### 12.3 Wayland Colour Management Protocol

The **`wp_color_management_v1`** Wayland protocol (staged in wayland-protocols, tracked in the KWin and Mutter compositor implementations) allows applications to declare their output colour space and transfer function. A client application creates a parametric image description by specifying named primaries (e.g. BT.2020) and a named transfer function (e.g. ST2084 PQ), then attaches that description to its Wayland surface. The compositor uses this declaration to perform colour-aware compositing and to program the KMS hardware colour pipeline accordingly.

> **Note:** The exact Wayland protocol interface names, enum values, and C client-side call sequence for `wp_color_management_v1` should be verified against the staged protocol XML in wayland-protocols before use. The protocol was still evolving as of mid-2026.

See Ch74 (the authoritative chapter for the full KMS colour pipeline, `wp_color_management_v1`, and compositor HDR architecture) for the complete end-to-end flow.

### 12.4 ICC Profile Awareness in Compositors

For SDR spectral rendering output, the compositor may apply an ICC display profile (managed by colord, see Ch101) to transform from the rendering colour space to the display's colour space. An sRGB-output spectral renderer feeding a wide-gamut display should embed the sRGB ICC profile in its output buffer metadata; the compositor applies the display's profile chain.

---

## 13. Integrations

**Ch101 — Color Science and the ICC Profile Pipeline**: CIE XYZ/Lab/xyY foundations used throughout §8; ICC profile pipeline for SDR spectral renderer output in §12.4; Bradford chromatic adaptation (§10.5) derives from ICC standard matrices.

**Ch158 — HDR Display Hardware and Signaling on Linux**: §12.2 references the `drm_hdr_output_metadata` connector property, SMPTE ST 2086 mastering display metadata, and CTA 861.3 MaxCLL/MaxFALL values that Ch158 covers in full, including the wire signalling to displays.

**Ch205 — Shader Algorithm Catalog: Global Illumination and Materials**: Spectral BRDF evaluation (§2) extends the BRDF models surveyed in Ch205; bispectral BRDF in §5 is the fluorescence extension of the standard BRDF framework; thin-film iridescence from Ch205 interacts with polarisation in §6.

**Ch207 — Shader Algorithm Catalog: VFX, Post-Processing, and GPU Compute**: Colour grading and 3D LUT interpolation in §10.6 complement Ch207's coverage of colour grading algorithms; the PQ/HLG transfer functions in §9.4 and §10.2 appear in Ch207's post-processing section.

**Ch227 — GPU Random Number Generation and Monte Carlo Methods**: Hero-wavelength sampling in §2 is a specialised application of importance sampling and MIS from Ch227; wavelength PDF construction follows the same balance heuristic framework; spectral quadrature in §8.2 is a Monte Carlo estimator.
