# Chapter 204a: Physically Based Rendering — Theory and Foundations (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Target audiences**: Graphics application developers who need the derivation behind the PBR parameters they already tune, not just a checklist of them; browser and web platform engineers implementing or evaluating WebGPU/Three.js/Babylon.js material systems; systems and driver developers assessing the per-pixel shader cost of physically based shading.

This chapter derives the theoretical foundation that the rest of the book's PBR-adjacent material assumes but does not individually re-derive: Chapter 205's "Physically Based Shading — BRDF Models" catalog entry lists the D/G/F building blocks as a menu; Chapter 83 (Filament) shows a production implementation consuming them; Chapter 64 (glTF 2.0) shows the asset schema that serializes their parameters. This chapter is the connective layer — the rendering equation these techniques approximate, the microfacet derivation behind the Cook-Torrance formula, why energy conservation is not optional, why the industry converged on the metallic-roughness parameterization, and how the split-sum approximation turns an intractable integral into two texture lookups.

## Table of Contents

- [1. The Rendering Equation](#1-the-rendering-equation)
- [2. Microfacet Theory and the Cook-Torrance BRDF](#2-microfacet-theory-and-the-cook-torrance-brdf)
- [3. Energy Conservation and Multi-Scatter Compensation](#3-energy-conservation-and-multi-scatter-compensation)
- [4. Metallic-Roughness vs. Specular-Glossiness Workflows](#4-metallic-roughness-vs-specular-glossiness-workflows)
- [5. Image-Based Lighting and the Split-Sum Approximation](#5-image-based-lighting-and-the-split-sum-approximation)
- [6. Authoring and Validating PBR Materials](#6-authoring-and-validating-pbr-materials)
- [7. Integrations](#7-integrations)

---

## 1. The Rendering Equation

Every real-time PBR technique in this book — Cook-Torrance specular, split-sum IBL, screen-space GI — is a structured approximation to a single integral equation. Formalizing it first makes clear *what* is being approximated and *why* each shortcut is taken.

The rendering equation, in its steady-state (non-emissive-transport) form, states that the outgoing radiance `L_o` leaving a surface point `p` in direction `ω_o` equals the point's emitted radiance plus the integral, over the hemisphere `Ω` above the surface normal, of incoming radiance `L_i` weighted by the surface's BRDF `f_r` and a cosine foreshortening factor:

```
L_o(p, ω_o) = L_e(p, ω_o) + ∫_Ω f_r(p, ω_i, ω_o) · L_i(p, ω_i) · (N·ω_i) dω_i
```

[Kajiya: The Rendering Equation (SIGGRAPH 1986)](https://dl.acm.org/doi/10.1145/15922.15902) introduced this formulation as a generalization that subsumes ray tracing, radiosity, and earlier ad hoc shading models as special cases, and proposed a Monte Carlo solution using hierarchical (importance) sampling — the ancestor of the path-tracing techniques cataloged in Chapter 206. The equation is recursive: `L_i(p, ω_i)` is itself the outgoing radiance `L_o` of whatever surface is visible along `ω_i`, which is why an exact solution requires following light transport paths of unbounded length.

**Why real-time rendering cannot solve this directly.** The integral has no closed form for arbitrary BRDFs and arbitrary incoming radiance distributions; stochastically sampling it at real-time frame budgets (sub-millisecond per light per pixel) produces unacceptable noise without denoising. Every PBR technique in this book is therefore a way to make one part of the equation tractable by constraining another:

- **The BRDF `f_r`** is replaced with a closed-form analytic model (Cook-Torrance/GGX, §2) instead of a measured or procedural function, so it can be evaluated in closed form rather than sampled.
- **The incoming radiance `L_i`** is split into a small number of analytic punctual/area lights (Chapter 204, Category II — evaluated exactly, since a delta or near-delta light needs no integration) plus a low-frequency *environment* term (this chapter, §5 — pre-integrated offline so the real-time cost is two texture fetches) plus, optionally, a full path-traced or screen-space GI term (Chapter 205, Category IV; Chapter 206) for the mid-frequency indirect light that neither analytic lights nor a static environment map capture.
- **The hemisphere integral itself** is pre-computed and stored in a lookup structure — spherical harmonics coefficients, a prefiltered mip chain, or a 2D BRDF-response LUT — whenever the integrand does not depend on per-frame state, trading a large offline or load-time cost for a near-zero runtime cost. This is the general pattern behind every "pre-integration" or "split-sum" technique in this chapter and in Chapter 205.

The energy-conservation constraint enforced on every analytic BRDF (§3) exists precisely so that these decompositions — direct + indirect, diffuse + specular, near light + distant environment — can be summed independently without double-counting or leaking energy, which is what makes an approximate, decomposed solution visually convincing despite not evaluating the actual integral.

---

## 2. Microfacet Theory and the Cook-Torrance BRDF

### 2.1 Rough surfaces as statistical facet distributions

Microfacet theory models a rough surface not as a smooth interface but as an uncountable collection of microscopic, perfectly smooth mirror facets, each independently reflecting light according to the law of specular reflection. At any macroscopic scale visible to a camera, an individual facet is far below pixel resolution; what the eye or sensor perceives instead is the *statistical distribution* of facet orientations relative to the macroscopic surface normal `N`. A surface where all facets are aligned with `N` looks like a mirror (very low roughness); a surface where facet orientations are widely scattered looks matte-glossy (high roughness). This statistical framing is what lets a single roughness scalar stand in for a surface's entire micro-geometry.

[Cook & Torrance: A Reflectance Model for Computer Graphics (1982)](https://doi.org/10.1145/357290.357293) adapted an earlier physically based microfacet model from optics ([Torrance & Sparrow 1967](https://doi.org/10.1364/JOSA.57.001105)) into computer graphics, producing the specular BRDF term that remains, essentially unmodified, the specular core of every production PBR pipeline covered in this book:

```
f_specular(ω_i, ω_o) = D(h) · G(ω_i, ω_o, h) · F(ω_o, h) / (4 · (N·ω_i) · (N·ω_o))
```

where `h` is the half-vector `normalize(ω_i + ω_o)` — the microfacet orientation that would specularly reflect `ω_i` into `ω_o` — and three independently swappable terms account for the physics:

- **`D`, the normal distribution function (NDF)** — the statistical density of microfacets oriented along `h`. Only facets aligned with `h` contribute to the reflection from `ω_i` to `ω_o`.
- **`G`, the geometry (masking-shadowing) term** — the fraction of those facets that are *not* occluded from the light direction (shadowing) or the view direction (masking) by neighboring facets.
- **`F`, the Fresnel term** — the fraction of light a single smooth facet reflects versus refracts, as a function of the angle between `ω_o` and `h`.

The `4·(N·ω_i)·(N·ω_o)` denominator is a Jacobian correction converting the facet-orientation probability density (measured with respect to `h`) into a probability density with respect to the actual reflected direction `ω_o`.

### 2.2 The GGX / Trowbridge-Reitz normal distribution

The NDF in near-universal production use is GGX, originally published as the Trowbridge-Reitz distribution in optics ([Trowbridge & Reitz: Average irregularity representation of a rough surface for ray reflection, JOSA 1975](https://doi.org/10.1364/JOSA.65.000531)) and reintroduced to graphics, evaluated against measured BRDF data, and shown to fit real materials better than the previously dominant Beckmann and Blinn-Phong distributions by [Walter et al.: Microfacet Models for Refraction through Rough Surfaces (EGSR 2007)](https://www.graphics.cornell.edu/~bjw/microfacetbsdf.pdf):

```glsl
// Trowbridge-Reitz / GGX normal distribution function
// alpha = roughness^2 (the standard perceptual remap, see §4)
float D_GGX(float NdotH, float alpha) {
    float a2    = alpha * alpha;
    float d     = NdotH * NdotH * (a2 - 1.0) + 1.0;
    return a2 / (3.14159265 * d * d);
}
```

GGX's defining visual characteristic relative to Beckmann is a longer, more gradual falloff tail away from the highlight peak — a better match to measured metals and dielectrics, particularly at grazing viewing angles, which is why it displaced Beckmann and Blinn-Phong as the default NDF industry-wide during the early-2010s PBR transition (§4). The `alpha = roughness²` remap is not part of the original Trowbridge-Reitz derivation; it is an artist-facing convention (popularized in the same SIGGRAPH course series cited in §4) chosen because linear interpolation of `roughness` then produces a perceptually more linear change in highlight sharpness than linearly interpolating `alpha` directly.

### 2.3 The Smith masking-shadowing term

The geometry term `G` must estimate, for a given microfacet orientation, what fraction of facets are simultaneously visible to both the light and the viewer. The Smith formulation ([Smith: Geometrical Shadowing of a Random Rough Surface, IEEE Trans. Antennas Propag. 1967](https://doi.org/10.1109/TAP.1967.1138991)) derives this statistically from the same normal distribution used for `D`, under the assumption that shadowing and masking are governed by the same facet-slope statistics as the NDF itself — so `G` is not an independent free parameter but is mathematically coupled to whichever `D` is chosen.

The separable form factors `G` into independent light- and view-direction masking functions, `G(ω_i, ω_o) = G1(ω_i) · G1(ω_o)`; the height-correlated form accounts for the fact that a facet high enough to be unmasked from the view direction is statistically also more likely to be unshadowed from the light direction, which the separable form ignores. [Heitz: Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs (JCGT 2014)](https://jcgt.org/published/0003/02/03/) is the standard reference and shows the height-correlated form is measurably more accurate at grazing angles for negligible extra cost — it is the form used in Filament (Chapter 83) and virtually all current production renderers:

```glsl
// Smith height-correlated visibility term (folds G and the 4*NdotV*NdotL
// denominator together, following Heitz 2014, eq. 72)
float V_SmithGGXCorrelated(float NdotV, float NdotL, float alpha) {
    float a2      = alpha * alpha;
    float lambdaV = NdotL * sqrt(NdotV * NdotV * (1.0 - a2) + a2);
    float lambdaL = NdotV * sqrt(NdotL * NdotL * (1.0 - a2) + a2);
    return 0.5 / max(lambdaV + lambdaL, 1e-5);
}
```

### 2.4 The Schlick Fresnel approximation

The exact Fresnel equations for an unpolarized dielectric-conductor interface require complex refractive indices and are too expensive to evaluate per-pixel for real-time use. [Schlick: An Inexpensive BRDF Model for Physically-Based Rendering (Eurographics 1994)](https://doi.org/10.1111/1467-8659.1330233) proposed a polynomial approximation parameterized only by `F0`, the reflectance at normal incidence (`ω=0°`):

```glsl
vec3 F_Schlick(float VdotH, vec3 F0) {
    return F0 + (vec3(1.0) - F0) * pow(clamp(1.0 - VdotH, 0.0, 1.0), 5.0);
}
```

`F0` is where the metallic-roughness workflow's `metallic` parameter enters the BRDF math directly: dielectrics (wood, plastic, skin, stone) have a narrow, near-achromatic `F0` around 0.02–0.05 (a default of 0.04, corresponding to an index of refraction of ~1.5, is the near-universal engine default); conductors (metals) have a high, often strongly chromatic `F0` — 0.95/0.64/0.54 (linear RGB) for gold, for instance — because a metal's free electrons absorb and re-radiate the refracted component almost immediately, leaving no diffuse term at all. This physical distinction, not an artist convention, is why the metallic-roughness workflow drives diffuse albedo to near-zero and colors `F0` with the base-color texture as `metallic` approaches 1 (§4).

### 2.5 Assembling the full BRDF

A complete direct-lighting PBR evaluation sums the microfacet specular term against a diffuse term (Lambertian, `albedo/π`, is the near-universal real-time default; Chapter 205 covers Oren-Nayar and other retroreflective alternatives) and weights the diffuse contribution by `(1 - F)` and `(1 - metallic)` so that specular and diffuse light are never double-counted at the same surface point:

```glsl
vec3 pbr_direct(vec3 N, vec3 V, vec3 L, vec3 albedo, float metallic, float roughness, vec3 radiance) {
    vec3  H       = normalize(V + L);
    float NdotV   = max(dot(N, V), 1e-4);
    float NdotL   = max(dot(N, L), 0.0);
    float NdotH   = max(dot(N, H), 0.0);
    float VdotH   = max(dot(V, H), 0.0);
    float alpha   = roughness * roughness;

    vec3 F0 = mix(vec3(0.04), albedo, metallic);       // §2.4 — dielectric vs. conductor F0
    vec3  F = F_Schlick(VdotH, F0);
    float D = D_GGX(NdotH, alpha);
    float Vis = V_SmithGGXCorrelated(NdotV, NdotL, alpha);

    vec3 specular = D * Vis * F;                        // 4*NdotV*NdotL folded into Vis (§2.3)
    vec3 kd       = (vec3(1.0) - F) * (1.0 - metallic);  // energy split, §3
    vec3 diffuse  = kd * albedo / 3.14159265;

    return (diffuse + specular) * radiance * NdotL;
}
```

This function — or a near-identical variant — is what Chapter 205 catalogs tersely as "Cook-Torrance Specular BRDF" and what Chapter 83 shows instantiated inside Filament's `standardModelFragment.fs`.

---

## 3. Energy Conservation and Multi-Scatter Compensation

A BRDF is physically valid only if it satisfies non-negativity, Helmholtz reciprocity (`f_r(ω_i, ω_o) = f_r(ω_o, ω_i)`), and energy conservation — the directional-hemispherical reflectance, `∫_Ω f_r(ω_i, ω_o)·(N·ω_o) dω_o`, must not exceed 1 for any incoming direction. The `kd = (1 - F) * (1 - metallic)` term in §2.5 is the mechanism that keeps the *single-scatter* diffuse+specular sum from exceeding energy conservation: at grazing angles `F` approaches 1, so `kd` correctly drives the diffuse contribution to zero exactly where the specular (Fresnel) contribution takes over.

**Where the single-scatter model still loses energy.** The Cook-Torrance formula in §2 models only *single* scattering — a photon reflecting off one microfacet and leaving. On a rough surface, a substantial fraction of light instead bounces between two or more microfacets before escaping; the single-scatter GGX model does not account for this energy at all, so it is simply lost rather than redirected. The visual symptom is that rough, bright metals and dielectrics render measurably darker than reference (path-traced or photographed) ground truth, with the darkening most visible at high roughness (>0.5) where inter-facet bouncing is most frequent.

[Kulla & Conty: Revisiting Physically Based Shading at Imageworks (SIGGRAPH 2017)](https://blog.selfshadow.com/publications/s2017-shading-course/) formalized a practical multi-scatter compensation: pre-integrate, for a given roughness and viewing angle, how much energy the single-scatter GGX BRDF fails to reflect (`1 - E(roughness, NdotV)`, where `E` is the directional albedo of the single-scatter lobe), store the result in a small 2D LUT, and add back a compensating energy-preserving term at shading time. [Fdez-Agüera: A Multiple-Scattering Microfacet Model for Real-Time Image-Based Lighting (2019)](https://jcgt.org/published/0008/01/03/) extends the same correction specifically to the IBL split-sum path (§5), which is where the missing energy is most visually apparent since environment lighting integrates over the entire rough lobe rather than a single light direction. Chapter 205 catalogs this as "Multiscattering BRDF Correction (Kulla-Conty)"; the underlying reason it is needed, rather than an optional polish pass, is this energy-conservation argument.

---

## 4. Metallic-Roughness vs. Specular-Glossiness Workflows

Two parameterizations of the same underlying Cook-Torrance/GGX BRDF circulated during the industry's PBR transition (roughly 2012–2016), and understanding why one displaced the other clarifies why glTF, Filament, Unreal, and Unity's newer render pipelines all converged on a single artist-facing schema.

**Specular-glossiness** (the earlier of the two, following directly from how offline renderers had long parameterized specular reflectance) exposes three independent texture inputs: diffuse color, a full RGB specular color (`F0`), and glossiness (inverse roughness). It is strictly more expressive than metallic-roughness — it can represent non-physical `F0` values freely — which is also its central problem: nothing in the parameterization prevents an artist from painting a diffuse-colored dielectric with a metal-bright specular color, producing a combination no real-world material exhibits, and every such combination has to be caught by validation tooling rather than being structurally impossible.

**Metallic-roughness** collapses the same physical degrees of freedom into a smaller, harder-to-misuse set: a single `baseColor`, a scalar `metallic` factor, and a scalar `roughness`. Because §2.4 established that dielectric `F0` is a narrow, nearly material-independent constant (~0.04) while metals have no diffuse term and colored `F0` equal to their base color, `metallic` can act as a single interpolation switch between two well-defined physical regimes rather than requiring the artist to hand-paint a plausible `F0`:

```
F0      = mix(vec3(0.04), baseColor, metallic)
diffuse = mix(baseColor, vec3(0.0), metallic)
```

This parameterization traces to the Disney "principled" BRDF's design goals — an artist-facing, perceptually intuitive parameter set built on top of a physically based core rather than exposing raw physical quantities — described in [Burley: Physically Based Shading at Disney (SIGGRAPH 2012)](https://blog.selfshadow.com/publications/s2012-shading-course/), and to Epic Games' contemporaneous account of the same tradeoff during Unreal Engine 4's PBR adoption in [Karis: Real Shading in Unreal Engine 4 (SIGGRAPH 2013)](http://blog.selfshadow.com/publications/s2013-shading-course/karis/s2013_pbs_epic_notes_v2.pdf), and DICE/Frostbite's independent but convergent account of the same production tradeoff in [Lagarde & de Rousiers: Moving Frostbite to Physically Based Rendering (SIGGRAPH 2014 course notes)](https://seblagarde.files.wordpress.com/2015/07/course_notes_moving_frostbite_to_pbr_v32.pdf). By the time the Khronos Group formalized glTF 2.0 as the cross-engine interchange format for real-time 3D assets, metallic-roughness was the de facto convergence point across the industry, and glTF adopted it as the *core* material model — [glTF 2.0 Appendix B: BRDF Implementation](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#appendix-b-brdf-implementation) specifies exactly the GGX/Smith/Schlick combination derived in §2 as the reference BRDF a conformant glTF renderer must implement — while retaining `KHR_materials_pbrSpecularGlossiness` only as a legacy extension for content migrated from older pipelines. Chapter 64 covers the resulting texture-channel packing convention (`ORM`: occlusion in R, roughness in G, metallic in B) as an asset-format concern; the physical argument for *why* that specific triple of channels is sufficient is the one made in this section.

---

## 5. Image-Based Lighting and the Split-Sum Approximation

Section 1 identified the environment (distant, low-frequency) lighting term as one of the pieces the rendering equation is split into for tractability. Evaluating it correctly still requires, per pixel, integrating the product of the BRDF and the full incoming radiance environment over the hemisphere — intractable at real-time rates if done naively for every shaded pixel every frame. **Image-based lighting (IBL)** makes this tractable by observing that the integral separates into two independent parts, each of which is expensive to precompute once but cheap to evaluate per pixel afterward.

### 5.1 Diffuse irradiance: convolution and spherical-harmonic projection

The Lambertian diffuse term's contribution from an environment is `irradiance(N) = ∫_Ω L_i(ω_i)·(N·ω_i) dω_i` — a cosine-weighted convolution of the environment map that depends only on the surface normal, not on roughness or view direction. Because a cosine lobe is extremely low-frequency, this convolution can be represented almost exactly with only the first few bands of a spherical-harmonic (SH) expansion (order 2, 9 RGB coefficients) rather than a full texture, following the derivation in [Ramamoorthi & Hanrahan: An Efficient Representation for Irradiance Environment Maps (SIGGRAPH 2001)](https://graphics.stanford.edu/papers/envmap/). The 9 SH coefficients are computed once per environment map (offline or at load time) and evaluated per pixel as a simple polynomial in `N` — cataloged in Chapter 205 as "Spherical Harmonics Lighting."

### 5.2 Specular IBL and the split-sum approximation

The specular case is harder: the integral now depends on both the surface normal *and* the roughness (which controls the width of the GGX lobe being convolved) *and* the view direction (because the GGX lobe is not radially symmetric about `N` except at normal incidence). A brute-force precomputation would need a separate convolved environment map per roughness value *and* per view angle — infeasible to store.

[Karis: Real Shading in Unreal Engine 4 (SIGGRAPH 2013)](http://blog.selfshadow.com/publications/s2013-shading-course/karis/s2013_pbs_epic_notes_v2.pdf) popularized the **split-sum approximation** for real-time use: under the simplifying assumption that `N = V` (a reasonable approximation for the environment term specifically, since it only introduces visible error at extreme grazing angles and high roughness), the integral factors into the product of two independent sums, each of which *can* be precomputed cheaply:

```
∫_Ω f_r(l,v)·L_i(l)·(N·l) dl  ≈  ( ∫_Ω L_i(l)·D(l,roughness)·(N·l) dl )  ×  ( ∫_Ω f_r(l,v)·(N·l) dl / ∫_Ω D(l,roughness)·(N·l) dl )
                                        prefiltered environment map                    BRDF integration ("split-sum") LUT
```

The first factor — the environment prefiltered against the GGX lobe for a given roughness — is stored as a mip chain: mip 0 is the original (unfiltered) environment, and each successively coarser mip is convolved with a progressively wider GGX lobe corresponding to a higher roughness, so a runtime lookup is `textureLod(prefiltered_env, R, roughness * max_mip)`. This is generated once per environment map by a compute or fragment pass that importance-samples the GGX distribution per mip level:

```glsl
// Prefiltered specular environment map generation (one dispatch per mip level)
// roughness increases with mip level; SAMPLE_COUNT importance samples per texel
vec3 prefilter_env_ggx(vec3 N, float roughness, samplerCube env, uint SAMPLE_COUNT) {
    vec3 V = N; vec3 R = N;                    // split-sum assumption: N == V == R
    vec3 prefiltered_color = vec3(0.0);
    float total_weight = 0.0;

    for (uint i = 0u; i < SAMPLE_COUNT; ++i) {
        vec2 Xi = hammersley(i, SAMPLE_COUNT);          // low-discrepancy 2D sample
        vec3 H  = importance_sample_ggx(Xi, N, roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);   // reflect V about H

        float NdotL = max(dot(N, L), 0.0);
        if (NdotL > 0.0) {
            prefiltered_color += texture(env, L).rgb * NdotL;
            total_weight      += NdotL;
        }
    }
    return prefiltered_color / max(total_weight, 1e-4);
}
```

The second factor — the BRDF's own response, integrated over the hemisphere as a function of `roughness` and `NdotV` only, independent of any specific environment — is *environment-independent*, so it is precomputed exactly once (not per environment map, not per frame) into a single 2D LUT of `(scale, bias)` pairs applied to `F0`:

```glsl
// BRDF integration LUT — precomputed once, indexed by (NdotV, roughness)
vec2 integrate_brdf(float NdotV, float roughness) {
    vec3 V = vec3(sqrt(1.0 - NdotV * NdotV), 0.0, NdotV);
    float A = 0.0, B = 0.0;
    const uint SAMPLE_COUNT = 1024u;

    for (uint i = 0u; i < SAMPLE_COUNT; ++i) {
        vec2 Xi = hammersley(i, SAMPLE_COUNT);
        vec3 H  = importance_sample_ggx(Xi, vec3(0,0,1), roughness);
        vec3 L  = normalize(2.0 * dot(V, H) * H - V);

        float NdotL = max(L.z, 0.0), NdotH = max(H.z, 0.0), VdotH = max(dot(V, H), 0.0);
        if (NdotL > 0.0) {
            float Vis = V_SmithGGXCorrelated(NdotV, NdotL, roughness * roughness) * NdotL * NdotV * 4.0 * VdotH / NdotH;
            float Fc  = pow(1.0 - VdotH, 5.0);
            A += (1.0 - Fc) * Vis;
            B += Fc * Vis;
        }
    }
    return vec2(A, B) / float(SAMPLE_COUNT);
}
```

At shading time, the runtime cost of the entire specular IBL term is two texture fetches — a trilinear mip fetch against the prefiltered environment and a bilinear fetch against the 2×2 (per-channel) BRDF LUT — reconstructing the full integral as `prefiltered_color * (F0 * lut.x + lut.y)`. This is exactly the pattern behind Filament's IBL implementation (Chapter 83, consuming a KTX2-packaged prefiltered cubemap and a baked BRDF LUT) and Chapter 205's "Image-Based Lighting" catalog entry.

**Where the approximation breaks down.** The `N = V` assumption discards all anisotropic-lobe stretching that a true view-dependent convolution would capture, which is visible as an over-rounded highlight at high roughness and grazing view angles; it also assumes the environment is infinitely distant, so it produces incorrect parallax for reflections of nearby geometry (addressed by the parallax-corrected cubemap and screen-space reflection techniques in Chapter 205's Category IV and V). The single-scatter energy loss described in §3 applies to the specular IBL term as well and is the case [Fdez-Agüera 2019](https://jcgt.org/published/0008/01/03/) specifically targets.

---

## 6. Authoring and Validating PBR Materials

The physical constraints derived in §2–§4 translate into concrete authoring rules that texture and material tools enforce or visualize:

- **Albedo range clamping.** Real-world diffuse albedo rarely exceeds ~0.9 reflectance (coal is ~0.02–0.04; fresh snow, one of the brightest natural materials, is ~0.85–0.9) and the metallic-roughness workflow additionally requires `baseColor` to represent diffuse-only reflectance for dielectrics (`metallic = 0`) or `F0`-only reflectance for metals (`metallic = 1`) — a texture painted with out-of-range albedo values (near-black or near-white with intermediate metalness) is the most common non-physical authoring mistake, and both Substance 3D Painter and Blender's Principled BSDF preview include an albedo-range validation overlay for exactly this reason.
- **No painted `F0` for dielectrics.** Because §2.4 establishes that dielectric `F0` is a near-constant 0.04, metallic-roughness tooling intentionally does *not* expose a paintable specular-color channel for `metallic = 0` materials (unlike specular-glossiness, §4) — the `KHR_materials_ior` and `KHR_materials_specular` glTF extensions exist specifically for the minority of cases (gems, some plastics/liquids) where the default 0.04 is measurably wrong, rather than as a general-purpose specular-color control.
- **Roughness texture linearity.** Roughness (and metallic) textures must be sampled without sRGB decoding — they are linear scalar data, not display-referred color — a mistake that silently darkens mid-roughness values and is one of the most common glTF import bugs; Chapter 64 covers the corresponding image-format and colorspace metadata.
- **Cross-tool material authoring.** GIMP, Krita, and darktable (Chapter 241) are used in PBR pipelines primarily for procedural or hand-painted texture-map authoring and channel-packing (combining separate occlusion, roughness, and metallic grayscale passes into a single packed ORM texture) rather than for physically calibrated albedo painting, which more commonly happens in dedicated substance-authoring tools; this chapter's energy-conservation and F0 constraints (§2.4, §6) are what such texture-authoring workflows need to respect regardless of which tool produces the maps.

---

## 7. Integrations

- **Chapter 205** (Shader Algorithm Catalog — Global Illumination and Materials) — this chapter's §2–§5 are the derivations behind Chapter 205's terse Category IV/V entries ("Cook-Torrance Specular BRDF," "GGX/Trowbridge-Reitz NDF," "Smith Masking-Shadowing Function," "Schlick Fresnel Approximation," "Multiscattering BRDF Correction," "Image-Based Lighting," "Spherical Harmonics Lighting"); read this chapter first for the *why*, Chapter 205 for the full catalog of variants and named alternatives this chapter does not cover (anisotropic GGX, clear coat, sheen, iridescence, subsurface scattering).
- **Chapter 83** (Filament) — a concrete production implementation of the exact BRDF (§2.5) and split-sum IBL (§5) derived in this chapter, including its KTX2-based prefiltered-cubemap and BRDF-LUT asset pipeline.
- **Chapter 64** (glTF 2.0 — The 3D Asset Pipeline Standard) — the metallic-roughness asset schema (§4) that serializes the parameters this chapter derives, including the ORM texture-packing convention and the `KHR_materials_ior`/`KHR_materials_specular`/`KHR_materials_pbrSpecularGlossiness` extensions referenced in §4 and §6.
- **Chapter 206** (Shader Algorithm Catalog — Ray Tracing and Procedural Content) — the same BRDFs derived here are evaluated at ray-hit points rather than rasterized fragments; the rendering-equation framing in §1 is also the direct theoretical basis for the path-tracing techniques cataloged there.
- **Chapter 234** (GPU Spectral Rendering and Colorimetric Algorithms) — extends the RGB Fresnel/albedo treatment in §2.4 and §6 to spectral rendering, where `F0` and dispersion effects (thin-film, iridescence) require wavelength-resolved evaluation.
- **Chapter 241** (GIMP, Krita, and darktable — Open-Source Photo Editing) — the texture-authoring and channel-packing tools referenced in §6.
- **Chapter 135** (Vulkan ray tracing) — provides the `VkAccelerationStructureKHR` and ray-query infrastructure that Chapter 206's path-traced evaluation of these BRDFs runs on.
