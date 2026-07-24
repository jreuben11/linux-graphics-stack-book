# Chapter 212: GPU Geometry Algorithms — Neural Geometry, Specialized Techniques, and GPU Primitives (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers working with learned geometry representations, specialized cross-domain geometry (acoustics, molecular, VR/XR), or fundamental GPU algorithm primitives; systems engineers selecting geometry computation libraries.

This chapter is the fifth and final volume of the GPU Geometry Algorithms series. It covers Category XI (Neural and Learned Geometry — 3D Gaussian Splatting, geometric deep learning, differentiable rendering, NeRF and neural implicit surfaces), Category XII (Specialized and Cross-Domain Applications — scientific visualization, decal projection, procedural texture synthesis, acoustic ray tracing, VR/XR reprojection and foveation, silhouette detection, micro-polygon displacement, SDF font rendering, and molecular surface computation), and Category XIII (GPU Algorithm Primitives for Geometry — Poisson disk and blue noise sampling, GPU convex hull, and GPU radix sort as a geometry primitive). The chapter closes with §108 (Library Landscape — a survey of production geometry libraries and their GPU integration patterns), §109 (Performance Reference — throughput and memory budgets for common geometry operations), and §110 (Integrations — cross-chapter reference map for the full GPU Geometry Algorithms series).

## Table of Contents

- **[XI. Neural and Learned Geometry](#xi-neural-and-learned-geometry)**
  - [91. 3D Gaussian Splatting and Neural Geometry](#91-3d-gaussian-splatting-and-neural-geometry)
  - [92. Geometric Deep Learning](#92-geometric-deep-learning)
  - [93. Differentiable Rendering and Inverse Geometry](#93-differentiable-rendering-and-inverse-geometry)
  - [94. 3D Gaussian Splatting](#94-3d-gaussian-splatting)
  - [95. Neural Geometry: NeRF and Neural Implicit Surfaces](#95-neural-geometry-nerf-and-neural-implicit-surfaces)
- **[XII. Specialized and Cross-Domain Applications](#xii-specialized-and-cross-domain-applications)**
  - [96. Scientific Visualization Geometry](#96-scientific-visualization-geometry)
  - [97. Decal and Surface Projection Geometry](#97-decal-and-surface-projection-geometry)
  - [98. GPU Texture Synthesis and Procedural Shading Geometry](#98-gpu-texture-synthesis-and-procedural-shading-geometry)
  - [99. Acoustic Ray Tracing and Sound Geometry](#99-acoustic-ray-tracing-and-sound-geometry)
  - [100. VR/XR Geometry: Reprojection and Foveation](#100-vrxr-geometry-reprojection-and-foveation)
  - [101. Silhouette Detection and Feature Lines](#101-silhouette-detection-and-feature-lines)
  - [102. Micro-Polygon Displacement and GPU Tessellation](#102-micro-polygon-displacement-and-gpu-tessellation)
  - [103. GPU SDF Font and Curve Rendering](#103-gpu-sdf-font-and-curve-rendering)
  - [104. Molecular Surface Computation](#104-molecular-surface-computation)
- **[XIII. GPU Algorithm Primitives for Geometry](#xiii-gpu-algorithm-primitives-for-geometry)**
  - [105. GPU Sampling: Poisson Disk and Blue Noise](#105-gpu-sampling-poisson-disk-and-blue-noise)
  - [106. GPU Convex Hull Computation](#106-gpu-convex-hull-computation)
  - [107. GPU Radix Sort as a Geometry Primitive](#107-gpu-radix-sort-as-a-geometry-primitive)




---

## XI. Neural and Learned Geometry

Neural geometry methods replace hand-designed algorithms with learned representations trained on image or geometry supervision. 3D Gaussian Splatting (§91 provides the conceptual overview; §94 covers the full implementation including the differentiable rasterizer and densification) renders scenes as oriented Gaussian splats fitted to multi-view images; Neural Radiance Fields (NeRF) and neural SDFs represent geometry as MLPs queried at continuous 3D positions; Geometric Deep Learning applies graph neural networks and SE(3)-equivariant architectures to mesh classification and segmentation; and differentiable rendering enables gradient-based inverse problems — recovering geometry, materials, and lighting from image observations. These workloads run as CUDA or Vulkan compute dispatches and typically require tensor core acceleration.

### 91. 3D Gaussian Splatting and Neural Geometry

*Audience: graphics application developers.*

3DGS (Kerbl et al. 2023) replaces triangle meshes with a set of 3D Gaussian primitives, each parameterised by position, covariance, opacity, and view-dependent colour encoded in spherical harmonics. Rendering is a differentiable rasterization pipeline that has overtaken NeRF for real-time novel-view synthesis. The GPU pipeline is fully Vulkan/compute-compatible.

### 91.1 3DGS Representation and Projection

Each Gaussian g has:

- **Position** μ ∈ ℝ³
- **Covariance** Σ = R S Sᵀ Rᵀ (rotation R from unit quaternion q, diagonal scale S)
- **Opacity** σ ∈ (0,1)
- **SH coefficients** for RGB colour (up to degree 3 → 48 floats)

Project to screen-space covariance Σ' = J W Σ Wᵀ Jᵀ, where W is the view transform and J is the Jacobian of the perspective projection at μ:

```glsl
// project_gaussians.comp
layout(set=0,binding=0) readonly buffer Gaussians { GaussianData g[]; };
layout(set=0,binding=1) writeonly buffer Projected { ProjectedG pg[]; };
layout(push_constant) uniform PC { mat4 view, proj; vec2 focal; } pc;
layout(local_size_x=64) in;

void main() {
    uint i   = gl_GlobalInvocationID.x;
    vec4 mu4 = pc.view * vec4(g[i].pos, 1.0);
    float tx = mu4.x, ty = mu4.y, tz = mu4.z;

    // Jacobian of perspective projection
    mat3 J = mat3(pc.focal.x/tz, 0, -pc.focal.x*tx/(tz*tz),
                  0, pc.focal.y/tz, -pc.focal.y*ty/(tz*tz),
                  0, 0, 0);
    mat3 W3 = mat3(pc.view);
    mat3 cov3d = buildCov3D(g[i].scale, g[i].rot);
    mat3 cov2d = J * W3 * cov3d * transpose(W3) * transpose(J);

    pg[i].pos2d  = (pc.proj * mu4).xy / mu4.w * 0.5 + 0.5;
    pg[i].cov2d  = vec3(cov2d[0][0], cov2d[0][1], cov2d[1][1]);
    pg[i].depth  = tz;
    pg[i].color  = evalSH(g[i].sh, normalize(-mu4.xyz));
    pg[i].opacity = g[i].opacity;
}
```

[Source: Kerbl et al. "3D Gaussian Splatting for Real-Time Novel View Synthesis", SIGGRAPH 2023; https://github.com/graphdeco-inria/gaussian-splatting]

### 91.2 Tile Rasterizer: Sort-by-Depth and Alpha Compositing

Rendering proceeds in four steps:

**Step 1 — Tile assignment.** Each Gaussian is assigned to one or more 16×16 screen tiles based on its 2D bounding box (3σ ellipse). A compact list of (tile_id << 32 | depth) sort keys is emitted — one per Gaussian-tile pair — via prefix scan:

```glsl
// tile_assign.comp — count tiles per Gaussian, prefix scan, then scatter keys
uint nTiles = countTilesInBbox(pg[i].pos2d, pg[i].cov2d);
uint base   = scanOffset[i];
for (uint t = 0; t < nTiles; t++)
    keys[base + t] = (tileID(t) << 32) | floatBitsToUint(pg[i].depth);
```

**Step 2 — Sort.** Radix sort keys by tile_id (high 32 bits) then depth (low 32 bits). Use the VkRadixSort library or a custom 4-pass GPU radix sort.

**Step 3 — Tile range.** A single scan over the sorted key array marks the start/end index in the sorted list for each tile.

**Step 4 — Alpha composite per tile.** Each tile gets one compute workgroup; it iterates its Gaussian list front-to-back and blends:

```glsl
// splat_forward.comp
layout(local_size_x=16, local_size_y=16) in;
void main() {
    ivec2 px = ivec2(gl_GlobalInvocationID.xy);
    vec3 col = vec3(0); float T = 1.0;   // transmittance
    for (uint k = tileStart[tileID]; k < tileEnd[tileID]; k++) {
        uint i     = sortedIdx[k];
        vec2 delta = vec2(px) - pg[i].pos2d * screenSize;
        float power = evalGaussian2D(delta, pg[i].cov2d);
        float alpha = min(0.99, pg[i].opacity * exp(-power));
        if (alpha < 1.0/255.0) continue;
        col += T * alpha * pg[i].color;
        T   *= (1.0 - alpha);
        if (T < 1e-4) break;
    }
    imageStore(outImage, px, vec4(col, 1.0 - T));
}
```

[Source: Kerbl et al. 2023 §4; GPU rasterizer implementation at https://github.com/graphdeco-inria/diff-gaussian-rasterization]

### 91.3 Mesh Extraction from 3DGS and NeRF

Converting a trained 3DGS scene to a triangle mesh enables traditional rendering pipelines and physics simulation. Two approaches:

**SuGaR** (Guédon & Lepetit 2023) adds a regularisation term during training that aligns Gaussians to surface sheets, then extracts a mesh by Poisson reconstruction (§3.12) from Gaussian centers weighted by opacity.

**NeRF → mesh** (using Instant NGP): marching cubes on the NeRF density field at a threshold σ_surface. The density grid is evaluated at each voxel center in a compute dispatch, then piped into the two-pass GPU MC pipeline from §3.2. Texture maps are baked by computing the NeRF colour at each surface point (§10.13 GPU texture baking pipeline).

**Occupancy networks** predict binary occupancy o(x) ∈ {0,1} for any query point. Run inference on a 256³ voxel grid in parallel (batched MLP evaluation in compute), then extract the isosurface with GPU marching cubes. [Source: Guédon & Lepetit "SuGaR: Surface-Aligned Gaussian Splatting", CVPR 2024; Müller et al. "Instant Neural Graphics Primitives", SIGGRAPH 2022]

---

### 92. Geometric Deep Learning

*Audience: graphics application developers, systems developers.*

Geometric deep learning runs neural network inference directly on point clouds and triangle meshes, enabling semantic segmentation, shape completion, and neural implicit surface extraction on the same GPU that renders the scene.

### 92.1 PointNet on GPU

PointNet (Qi et al. 2017) applies a shared MLP per point then aggregates via symmetric max-pool. The per-point MLP is fully parallel — one thread per point per layer:

```glsl
// pointnet_mlp.comp — one fully-connected layer
layout(set=0,binding=0) readonly buffer InFeat  { float inF[N_POINTS*IN_DIM]; };
layout(set=0,binding=1) readonly buffer Weights { float W[IN_DIM*OUT_DIM]; float B[OUT_DIM]; };
layout(set=0,binding=2) writeonly buffer OutFeat { float outF[N_POINTS*OUT_DIM]; };
layout(local_size_x=64) in;
void main() {
    uint p=gl_GlobalInvocationID.x;
    for(uint o=0; o<OUT_DIM; o++) {
        float acc=B[o];
        for(uint i=0; i<IN_DIM; i++) acc+=inF[p*IN_DIM+i]*W[i*OUT_DIM+o];
        outF[p*OUT_DIM+o]=max(acc,0.0);  // ReLU
    }
}
```

Global max-pool over all points aggregates per-point features into a global descriptor via a parallel reduce (same pattern as §48.3). [Source: Qi et al. "PointNet", CVPR 2017; https://github.com/charlesq34/pointnet]

### 92.2 Graph Convolution on Mesh Edge Graphs

Graph neural network message passing on mesh edges — structurally identical to the cotangent Laplacian evaluation (§34.1) with learned weights:

```glsl
// graph_conv.comp — one GNN message-passing layer
layout(set=0,binding=0) readonly buffer NodeFeat { float nf[N_VERTS*FEAT_DIM]; };
layout(set=0,binding=1) readonly buffer Edges    { uvec2 edges[N_EDGES]; };
layout(set=0,binding=2) readonly buffer EdgeW    { float ew[N_EDGES*FEAT_DIM*FEAT_DIM]; };
layout(set=0,binding=3) buffer AggFeat { float agg[N_VERTS*FEAT_DIM]; };
layout(local_size_x=64) in;
void main() {
    uint e=gl_GlobalInvocationID.x;
    uint i=edges[e].x, j=edges[e].y;
    for(uint f=0; f<FEAT_DIM; f++) {
        float msg=0.0;
        for(uint g=0; g<FEAT_DIM; g++)
            msg+=ew[e*FEAT_DIM*FEAT_DIM+f*FEAT_DIM+g]*nf[j*FEAT_DIM+g];
        atomicAdd(agg[i*FEAT_DIM+f], msg);   // VK_EXT_shader_atomic_float
    }
}
```

Supports ChebNet, GAT (Graph Attention Networks), and DGCNN with minor modifications. [Source: Kipf & Welling "Semi-Supervised Classification with GCNs", ICLR 2017; Wang et al. "Dynamic Graph CNN for Learning on Point Clouds", SIGGRAPH 2019]

### 92.3 Neural SDF Inference on GPU

Evaluate a two-layer MLP at each voxel of a 256³ grid in parallel, then extract the isosurface with GPU MC (§3.2):

```glsl
// neural_sdf.comp
layout(set=0,binding=0) readonly buffer W1 { float w1[IN_DIM*H_DIM]; float b1[H_DIM]; };
layout(set=0,binding=1) readonly buffer W2 { float w2[H_DIM];         float b2;        };
layout(set=0,binding=2, r32f) uniform writeonly image3D sdfGrid;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 vox=ivec3(gl_GlobalInvocationID);
    vec3  x=(vec3(vox)/vec3(GRID_SIZE))*2.0-1.0;
    float h[H_DIM];
    for(int j=0; j<H_DIM; j++) {
        float acc=b1[j];
        acc+=w1[j*3+0]*x.x+w1[j*3+1]*x.y+w1[j*3+2]*x.z;
        h[j]=max(acc,0.0);
    }
    float sdf=b2;
    for(int j=0; j<H_DIM; j++) sdf+=w2[j]*h[j];
    imageStore(sdfGrid, vox, vec4(sdf));
}
```

[Source: Mescheder et al. "Occupancy Networks", CVPR 2019; Müller et al. "Instant NGP", SIGGRAPH 2022]

### 92.4 Feature Distillation from 3DGS to Mesh Atlas

Bake semantic/appearance features from a trained 3DGS scene (§91) into a mesh UV atlas (§10.13) for downstream editing or segmentation:

```glsl
// feature_bake.comp
layout(set=0,binding=0) readonly buffer AtlasPos  { vec4 worldPos[];  };
layout(set=0,binding=1) readonly buffer Gaussians { GaussianData g[]; };
layout(set=0,binding=2, rgba32f) uniform writeonly image2D featureAtlas;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 texel=ivec2(gl_GlobalInvocationID.xy);
    vec3  p=worldPos[texel.y*ATLAS_W+texel.x].xyz;
    vec4  feat=vec4(0);
    for(uint i=0; i<N_GAUSSIANS; i++) {
        float d=length(g[i].pos-p);
        float alpha=g[i].opacity*exp(-0.5*d*d/(g[i].scale*g[i].scale));
        feat+=alpha*g[i].semanticFeature;
    }
    imageStore(featureAtlas, texel, feat);
}
```

[Source: Kerbl et al. SIGGRAPH 2023; Lerf "Language Embedded Radiance Fields", ICCV 2023]

---

### 93. Differentiable Rendering and Inverse Geometry

*Audience: graphics application developers, systems developers.*

Differentiable rendering computes the gradient ∂L/∂θ of a scalar loss L (e.g., image reconstruction error) with respect to scene geometry parameters θ (vertex positions, SDF values, Gaussian means). This enables gradient-based geometry optimisation: mesh fitting to photographs, geometry parameter tuning, neural inverse rendering.

### 93.1 Differentiable Rasterization: Soft Visibility

Standard rasterisation is not differentiable at triangle edges (step-function coverage). SoftRas (Liu et al. 2019) and NVDiffRast (Laine et al. 2020) smooth the triangle edge boundary:

```glsl
// soft_rast.frag — soft triangle coverage for differentiable rendering
in vec3 bary;   // barycentric coordinates
uniform float sigma;  // softness parameter (typ. 1e-4)
out float coverage;
void main() {
    float d = min(min(bary.x, bary.y), bary.z);  // signed distance to nearest edge
    coverage = 1.0 / (1.0 + exp(-d / sigma));    // sigmoid soft boundary
}
```

The backward pass accumulates ∂coverage/∂vertex_position via the chain rule; each edge contributes a gradient proportional to the sigmoid derivative. [Source: Liu et al. "Soft Rasterizer", ICCV 2019; Laine et al. "Modular Primitives for High-Performance Differentiable Rendering", SIGGRAPH Asia 2020]

### 93.2 Differentiable Ray Tracing: Geometry Gradients

For ray-traced images, the gradient of a pixel colour L w.r.t. a vertex position p is:

∂L/∂p = ∂L/∂x_hit · ∂x_hit/∂p

where x_hit is the intersection point. The second term comes from the barycentric interpolation:

```glsl
// grad_rchit.glsl — accumulate vertex gradient in ray tracing closest-hit shader
hitAttributeEXT vec2 bary;
layout(set=0,binding=3) buffer GradBuffer { vec3 vertGrad[]; };

void main() {
    vec3 grad_L = gradFromPayload();    // ∂L/∂shading from miss/recursion
    uint i0 = indices[3*gl_PrimitiveID+0];
    uint i1 = indices[3*gl_PrimitiveID+1];
    uint i2 = indices[3*gl_PrimitiveID+2];
    float w0 = 1.0 - bary.x - bary.y;
    // ∂x_hit/∂v_k = w_k * I (identity), so ∂L/∂v_k = w_k * grad_L
    atomicAdd_vec3(vertGrad[i0], grad_L * w0);
    atomicAdd_vec3(vertGrad[i1], grad_L * bary.x);
    atomicAdd_vec3(vertGrad[i2], grad_L * bary.y);
}
```

[Source: Laine et al. 2020; Li et al. "Differentiable Monte Carlo Ray Tracing through Edge Sampling", SIGGRAPH Asia 2018]

### 93.3 3DGS Gradient Flow

For 3D Gaussian splatting (§91), the backward pass of the tile rasterizer propagates image-space gradients back to per-Gaussian mean μ, covariance Σ, opacity α, and spherical harmonic coefficients. The forward-pass tile rasterizer stores per-pixel transmittance T for the backward pass:

```glsl
// gs_backward.comp — gradient w.r.t. Gaussian means and opacity
layout(set=0,binding=0) readonly buffer Transmittance { float T[];    };   // per-pixel, per-depth
layout(set=0,binding=1) readonly buffer GradImg       { vec3  dL_dC[]; };  // ∂L/∂pixel color
layout(set=0,binding=2) buffer GradMeans { vec3 dL_dmu[]; };
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    // iterate sorted Gaussians front-to-back, accumulate gradient
    for (int k = tileGaussEnd[pix] - 1; k >= tileGaussStart[pix]; k--) {
        uint g     = sortedIdx[k];
        float dL_dalpha = dot(dL_dC[pix.y*WIDTH+pix.x], color[g]) * T[pix.y*WIDTH+pix.x];
        // ∂alpha/∂mu via projected Gaussian PDF gradient
        vec2 d     = projPos[g] - vec2(pix);
        vec2 dmu2d = -alpha[g] * dL_dalpha * (cov2dInv[g] * d);
        atomicAdd_vec2(dL_dmu2d[g], dmu2d);
    }
}
```

[Source: Kerbl et al. "3D Gaussian Splatting for Real-Time Radiance Field Rendering", SIGGRAPH 2023 (supplemental training code); https://github.com/graphdeco-inria/gaussian-splatting]

### 93.4 Geometry Optimisation Loop

The full loop: forward render → compute loss → backward pass → gradient descent step on vertex positions/SDF values:

```python
# Host-side loop (pseudo-code; actual dispatch via Vulkan compute)
for iteration in range(MAX_ITERS):
    vkCmdDispatch(forward_render)      # §93.1 soft rasteriser or §75 RT
    vkCmdDispatch(loss_compute)        # MSE / perceptual loss
    vkCmdDispatch(backward_pass)       # §93.2 or §93.3 gradient accumulation
    vkCmdDispatch(adam_update)         # Adam: m = β1*m + (1-β1)*g; ...
    # After convergence: extract mesh from optimised SDF via §3.2 GPU MC
```

Adam per-parameter momentum/variance update also runs in compute — one thread per vertex/voxel/Gaussian. [Source: Kingma & Ba "Adam: A Method for Stochastic Optimization", ICLR 2015]

---

### 94. 3D Gaussian Splatting

*Audience: graphics application developers.*

3D Gaussian Splatting (3DGS, Kerbl et al. 2023) represents a scene as a set of anisotropic 3D Gaussians — each with a position, covariance matrix, opacity, and spherical harmonic colour coefficients — and rasterizes them via a GPU tile-based alpha compositing pipeline. It achieves real-time novel-view synthesis quality matching NeRF at 30–120 fps without ray marching.

### 94.1 Gaussian Representation and SH Coefficients

Each Gaussian stores: centre μ ∈ ℝ³, scale s ∈ ℝ³, rotation quaternion q ∈ ℝ⁴ (which together define the 3D covariance Σ = R·S·Sᵀ·Rᵀ), opacity α ∈ [0,1], and spherical harmonic coefficients for view-dependent colour (up to degree 3, 48 floats per Gaussian):

```glsl
// gaussian_project.comp — project 3D Gaussians to 2D screen-space ellipses
struct Gaussian3D {
    vec3  mu;       // centre
    vec4  q;        // rotation quaternion
    vec3  s;        // scale (log space)
    float alpha;
    float sh[48];   // SH colour coefficients (degree 0–3)
};
layout(set=0,binding=0) readonly buffer Gaussians { Gaussian3D g[]; };
layout(set=0,binding=1) writeonly buffer Projected { Gaussian2D proj[]; };
layout(set=0,binding=2) writeonly buffer Keys { uint64_t tileDepthKey[]; };
layout(push_constant) uniform PC { mat4 view; mat4 proj; vec2 focalLen; uint W; } pc;
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    Gaussian3D gi = g[i];
    // Build 3D covariance from scale+rotation
    mat3 S   = mat3(exp(gi.s.x),0,0, 0,exp(gi.s.y),0, 0,0,exp(gi.s.z));
    mat3 R   = quatToMat3(gi.q);
    mat3 Sig = R * S * transpose(S) * transpose(R);
    // Project to 2D: Σ' = J·W·Σ·Wᵀ·Jᵀ  (Zwicker et al. 2001 EWA splatting)
    mat3 W   = mat3(pc.view);
    vec3 tc  = (pc.view * vec4(gi.mu,1.0)).xyz;
    mat3x2 J = mat3x2(pc.focalLen.x/tc.z, 0,
                       0, pc.focalLen.y/tc.z,
                       -pc.focalLen.x*tc.x/(tc.z*tc.z), -pc.focalLen.y*tc.y/(tc.z*tc.z));
    mat2 cov2D = mat2(J * W * Sig * transpose(W) * transpose(J));
    // Tile key: sort by tile (high 32 bits) then depth (low 32 bits)
    vec4 clip  = pc.proj * vec4(tc,1.0);
    vec2 ndc   = clip.xy / clip.w;
    ivec2 tile = ivec2((ndc*0.5+0.5) * vec2(pc.W/TILE_SZ));
    tileDepthKey[i] = (uint64_t(tile.y*TILES_X+tile.x) << 32) | floatBitsToUint(clip.w);
    proj[i] = Gaussian2D(ndc, cov2D, evalSH(gi.sh, -normalize(tc)), gi.alpha);
}
```

[Source: Kerbl et al. "3D Gaussian Splatting for Real-Time Radiance Field Rendering", SIGGRAPH 2023; Zwicker et al. "EWA Splatting", IEEE TVCG 2002]

### 94.2 Tile-Based Sorting and Rasterization

After projection, sort Gaussians by tile+depth key using GPU radix sort (§25.4), then rasterize each tile in a compute shader doing back-to-front alpha compositing:

```glsl
// gaussian_raster.comp — tile-based alpha compositing of sorted Gaussians
layout(set=0,binding=0) readonly buffer SortedIdx  { uint idx[]; };
layout(set=0,binding=1) readonly buffer Projected  { Gaussian2D proj[]; };
layout(set=0,binding=2) readonly buffer TileRanges { uvec2 ranges[]; };  // [start,end) per tile
layout(set=0,binding=3, rgba16f) uniform writeonly image2D outImg;
layout(local_size_x=TILE_SZ,local_size_y=TILE_SZ) in;
shared Gaussian2D tileCache[MAX_GAUSSIANS_PER_TILE];
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    uint  tile = gl_WorkGroupID.y * TILES_X + gl_WorkGroupID.x;
    uvec2 rng  = ranges[tile];
    vec3  C    = vec3(0.0);
    float T    = 1.0;  // transmittance
    for (uint b=rng.x; b<rng.y && T>0.001; b++) {
        Gaussian2D gs = proj[idx[b]];
        vec2  d   = vec2(pix) - gs.center * 0.5 * vec2(imageSize(outImg));
        // Evaluate 2D Gaussian: power = exp(-0.5 * dᵀ Σ⁻¹ d)
        mat2  Si  = inverse(gs.cov2D + mat2(0.3));  // add low-pass filter
        float pwr = exp(-0.5 * dot(d, Si * d));
        float a   = min(0.99, gs.alpha * pwr);
        C  += T * a * gs.color;
        T  *= (1.0 - a);
    }
    imageStore(outImg, pix, vec4(C + T*BACKGROUND_COLOR, 1.0));
}
```

[Source: Kerbl et al. 2023; implementation reference: https://github.com/graphdeco-inria/gaussian-splatting]

### 94.3 Gaussian Culling and LOD

For large scenes (millions of Gaussians), cull those outside the view frustum and those below a screen-space size threshold in the projection pass:

```glsl
// gaussian_cull.comp — frustum cull + size cull before projection
layout(set=0,binding=0) readonly buffer Gaussians { Gaussian3D g[]; };
layout(set=0,binding=1) writeonly buffer Visible  { uint visIdx[]; };
layout(set=0,binding=2) buffer VisCount { uint count; };
layout(push_constant) uniform PC { mat4 viewProj; float minScreenArea; } pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    vec4 c = pc.viewProj * vec4(g[i].mu, 1.0);
    vec3 s = exp(g[i].s);
    float maxS = max(s.x, max(s.y, s.z));
    float screenSz = maxS / c.w;  // rough screen-space radius estimate
    // Frustum cull: NDC check with Gaussian radius
    if (c.w < 0.01 || abs(c.x/c.w) > 1.3 || abs(c.y/c.w) > 1.3) return;
    if (screenSz < pc.minScreenArea) return;
    uint slot = atomicAdd(count, 1u);
    visIdx[slot] = i;
}
```

[Source: Ye et al. "AbsGS: Recovering Fine Details for 3D Gaussian Splatting", 2024; Mallick et al. "Taming 3DGS: High-Quality Radiance Fields with Limited Resources", SIGGRAPH Asia 2024]

### 94.4 Gaussian Densification and Pruning

During training (offline, CUDA), over-reconstructed regions require splitting large Gaussians and cloning small ones; under-reconstructed regions require pruning transparent Gaussians. The GPU densification kernel:

```cuda
// gaussian_densify.cu — adaptive density control (CUDA training pass)
__global__ void densifyAndPrune(
    Gaussian3D* g, float* grad2D, float* maxRadii,
    uint* splitMask, uint* cloneMask, uint* pruneMask,
    float gradThresh, float sizeThresh, float alphaThresh, int N)
{
    int i = blockIdx.x*blockDim.x + threadIdx.x;
    if (i >= N) return;
    float g2  = grad2D[i];
    float sz  = max(exp(g[i].s.x), max(exp(g[i].s.y), exp(g[i].s.z)));
    if (g[i].alpha < alphaThresh) { pruneMask[i] = 1; return; }
    if (g2 > gradThresh) {
        if (sz > sizeThresh) splitMask[i] = 1;  // large + high grad → split
        else                 cloneMask[i] = 1;   // small + high grad → clone
    }
}
```

[Source: Kerbl et al. 2023 §5.2; https://github.com/graphdeco-inria/gaussian-splatting/blob/main/scene/gaussian_model.py]

---

### 95. Neural Geometry: NeRF and Neural Implicit Surfaces

*Audience: graphics application developers.*

Neural Radiance Fields (NeRF, Mildenhall et al. 2020) and neural signed distance functions (DeepSDF, Park et al. 2019) represent geometry implicitly as the weights of a neural network. At inference time, GPU shaders evaluate these networks along camera rays, producing novel views or surface geometry without an explicit triangle mesh. Instant-NGP (Müller et al. 2022) replaces the large MLP with a multiresolution hash grid, achieving real-time NeRF rendering.

### 95.1 NeRF: Volume Rendering with MLP Inference

A NeRF MLP maps (x, y, z, θ, φ) → (colour, density). Each camera ray is sampled at N stratified depths and the MLP is evaluated at each sample, then composited via the volume rendering equation (§67.1):

```glsl
// nerf_inference.comp — evaluate NeRF MLP for one camera ray
layout(set=0,binding=0) readonly buffer MLPWeights { float W0[]; float W1[]; float W2[]; };  // layer weights
layout(set=0,binding=1, rgba16f) uniform writeonly image2D renderOut;
layout(push_constant) uniform PC { mat4 invView; vec2 fov; uint W; uint H; uint N_SAMPLES; } pc;
layout(local_size_x=8,local_size_y=8) in;

vec4 evalMLP(vec3 pos, vec3 dir) {
    // Positional encoding: [sin(2^k π x), cos(2^k π x)] for k=0..9
    float feat[63];  // 3 + 2*3*10 = 63 input features (pos encode)
    posEncode(pos, dir, feat);
    // Forward pass: 8-layer ReLU MLP (256 hidden units)
    float h[256];
    for (int l=0; l<8; l++) matmulReLU(feat, W0 + l*256*256, h, 256);
    return vec4(sigmoid(h[0]), sigmoid(h[1]), sigmoid(h[2]),  // RGB
                relu(h[3]));                                    // density σ
}
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  ro   = (pc.invView * vec4(0,0,0,1)).xyz;
    vec3  rd   = normalize((pc.invView * vec4(pixelDir(pix, pc.fov),1)).xyz);
    vec3  C    = vec3(0); float T = 1.0;
    float tNear=0.1, tFar=10.0, dt=(tFar-tNear)/float(pc.N_SAMPLES);
    for (uint s=0; s<pc.N_SAMPLES && T>0.01; s++) {
        float t = tNear + (float(s)+0.5)*dt;
        vec4 rgbσ = evalMLP(ro+rd*t, rd);
        float a   = 1.0 - exp(-rgbσ.w*dt);
        C += T * a * rgbσ.rgb;
        T *= (1.0-a);
    }
    imageStore(renderOut, pix, vec4(C,1));
}
```

[Source: Mildenhall et al. "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis", ECCV 2020; https://www.matthewtancik.com/nerf]

### 95.2 Instant-NGP: Multiresolution Hash Grid

Instant-NGP (Müller et al. 2022) replaces the large positional encoding + deep MLP with a multi-resolution hash table of learnable feature vectors (16 levels, 2^19 entries each, 2 features per entry), followed by a tiny 2-layer MLP. GPU inference uses CUDA/Vulkan shared memory to batch hash lookups:

```glsl
// ngp_hash_encode.comp — lookup hash grid features for a batch of 3D positions
layout(set=0,binding=0) readonly buffer HashGrid { vec2 entries[N_LEVELS][HASH_SIZE]; };
layout(set=0,binding=1) readonly buffer Positions { vec3 pts[]; };
layout(set=0,binding=2) writeonly buffer Features { float feats[]; };  // N_LEVELS*2 per point
layout(push_constant) uniform PC { float baseRes; float perLevelScale; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    vec3 p  = pts[i];
    for (int lv=0; lv<N_LEVELS; lv++) {
        float res = pc.baseRes * pow(pc.perLevelScale, float(lv));
        vec3  psc = p * res;
        ivec3 c0  = ivec3(floor(psc));
        vec3  f   = fract(psc);
        // Trilinear interpolation of 8 hash entries
        vec2  feat = vec2(0.0);
        for (int dz=0;dz<=1;dz++) for(int dy=0;dy<=1;dy++) for(int dx=0;dx<=1;dx++) {
            ivec3 c  = c0 + ivec3(dx,dy,dz);
            uint  h  = (c.x*2654435761u ^ c.y*805459861u ^ c.z*3674653429u) % HASH_SIZE;
            float w  = mix(dx?f.x:1-f.x, 0, 0) * mix(dy?f.y:1-f.y,0,0)
                     * (dz ? f.z : 1.0-f.z);
            feat    += w * entries[lv][h];
        }
        feats[i*N_LEVELS*2 + lv*2 + 0] = feat.x;
        feats[i*N_LEVELS*2 + lv*2 + 1] = feat.y;
    }
}
```

[Source: Müller et al. "Instant Neural Graphics Primitives with a Multiresolution Hash Encoding", SIGGRAPH 2022; https://github.com/NVlabs/instant-ngp]

### 95.3 Neural SDF: DeepSDF Inference

DeepSDF (Park et al. 2019) represents a shape as φ(x) = MLP(x, z) where z is a learned latent code. At inference, evaluate the MLP at each query point. Combined with §65 marching cubes on the neural SDF grid, this produces a mesh:

```glsl
// deepsdf_infer.comp — evaluate DeepSDF MLP on a voxel grid for isosurface extraction
layout(set=0,binding=0) readonly buffer SDFWeights { float layers[N_LAYERS][512][512+3]; };
layout(set=0,binding=1) readonly buffer LatentCode { float z[256]; };
layout(set=0,binding=2, r32f) uniform writeonly image3D sdfGrid;
layout(push_constant) uniform PC { vec3 gridMin; float dx; } pc;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    ivec3 id = ivec3(gl_GlobalInvocationID);
    vec3  p  = pc.gridMin + (vec3(id)+0.5)*pc.dx;
    // Concatenate [p, z] as input to 8-layer 512-unit MLP
    float h[512]; buildInput(p, z, h);
    for (int l=0; l<8; l++) {
        if (l == 4) addSkip(p, z, h);  // skip connection at layer 4
        matmulReLU(h, layers[l], h, 512);
    }
    imageStore(sdfGrid, id, vec4(tanh(h[0])));  // output: signed distance
}
// Then run §65.1 marching cubes on the sdfGrid to extract the surface
```

[Source: Park et al. "DeepSDF: Learning Continuous Signed Distance Functions for Shape Representation", CVPR 2019]

### 95.4 NeuS: Neural Implicit Surface for Reconstruction

NeuS (Wang et al. 2021) learns an SDF-parameterised volume density function s(φ) = sigmoid(−φ/ε) that correctly renders the zero-level-set surface from multi-view images. The rendering integral uses the SDF value to bias the density toward the surface:

```glsl
// neus_render.comp — NeuS volume rendering with SDF-derived density
// After training, inference evaluates the same §95.3 MLP + colour network
// The key difference from NeRF (§95.1) is the density function:
// ρ(t) = max(-dφ/dt, 0) * sigmoid(-φ/ε) / (sigmoid(-φ/ε) + sigmoid(φ/ε))
// This makes the surface lie exactly on the SDF zero crossing.
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  ro  = cameraPos, rd = rayDir(pix);
    vec3  C   = vec3(0); float T = 1.0;
    float prevPhi = evalSDF_MLP(ro + rd*T_NEAR);
    for (uint s=0; s<N_SAMPLES; s++) {
        float t    = T_NEAR + float(s)*DT;
        float phi  = evalSDF_MLP(ro + rd*t);
        float dphi = (phi - prevPhi) / DT;
        float rho  = max(-dphi,0.0) * sigmoid(-phi/EPSILON) /
                     (sigmoid(-phi/EPSILON) + sigmoid(phi/EPSILON) + 1e-5);
        float a    = 1.0 - exp(-rho*DT);
        C += T * a * evalColor_MLP(ro+rd*t, rd);
        T *= (1.0-a); prevPhi=phi;
    }
    imageStore(renderOut, pix, vec4(C,1));
}
```

[Source: Wang et al. "NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction", NeurIPS 2021]

---


---

## XII. Specialized and Cross-Domain Applications

This category collects geometry algorithms that serve specific application domains rather than the general rendering pipeline. Scientific visualization renders isosurfaces and streamlines from simulation scalar and vector fields; acoustic ray tracing models sound propagation geometry; molecular surface computation (Connolly, SAS) supports biochemistry visualization; VR/XR reprojection geometry warps previously rendered frames to reduce latency; GPU SDF font rendering produces crisp text at all scales without rasterization artefacts; micro-polygon displacement adds film-quality surface detail via tessellation; silhouette detection drives both NPR rendering and shadow volume construction; and GPU texture synthesis generates tileable detail patterns. Each draws on core GPU geometry primitives but targets a narrow, well-defined application context.

### 96. Scientific Visualization Geometry

*Audience: systems developers, graphics application developers.*

Simulation output — velocity fields, scalar fields, tensor fields — requires geometry algorithms to extract, integrate, and render meaningful visual structures. All compute patterns here reuse infrastructure from earlier sections: RK4 integration, marching cubes, mesh shaders.

### 96.1 Streamline and Pathline Integration

Streamlines trace tangent curves of a vector field v(x); pathlines follow time-varying fields. 4th-order Runge-Kutta in compute — one thread per seed:

```glsl
// streamline.comp
layout(set=0,binding=0) uniform sampler3D velField;
layout(set=0,binding=1) writeonly buffer Lines { vec4 pts[MAX_SEEDS*MAX_STEPS]; };
layout(set=0,binding=2) writeonly buffer Counts{ uint len[]; };
layout(push_constant) uniform PC { float dt; uint maxSteps; vec3 fMin,fScale; } pc;
layout(local_size_x=64) in;

vec3 sample(vec3 p){ return texture(velField,(p-pc.fMin)/pc.fScale).xyz; }
vec3 rk4(vec3 p,float dt){
    vec3 k1=sample(p), k2=sample(p+0.5*dt*k1),
         k3=sample(p+0.5*dt*k2), k4=sample(p+dt*k3);
    return p+dt/6.0*(k1+2*k2+2*k3+k4);
}
void main() {
    uint lid=gl_GlobalInvocationID.x;
    vec3 pos=seedPoints[lid];
    uint base=lid*MAX_STEPS, step;
    for(step=0; step<pc.maxSteps && !outsideDomain(pos); step++){
        pts[base+step]=vec4(pos,0.0); pos=rk4(pos,pc.dt);
    }
    len[lid]=step;
}
```

Tube geometry (§4.3) converts streamlines to renderable ribbons. [Source: Jobard & Lefer "Creating Evenly-Spaced Streamlines", EuroVis 1997]

### 96.2 Empty-Space Skipping for Volume Ray Casting

Volume rendering casts one ray per pixel accumulating colour and opacity via front-to-back compositing. A min-max mipmap skips empty regions:

```glsl
// volume_cast.frag
uniform sampler3D vol; uniform sampler3D minMaxMip;
in vec3 entryPos, exitPos;
void main() {
    vec3 dir=normalize(exitPos-entryPos);
    float tMax=length(exitPos-entryPos);
    vec3 col=vec3(0); float T=1.0, t=0.0;
    while(t<tMax && T>0.01) {
        vec3 p=entryPos+t*dir;
        float maxD=textureLod(minMaxMip,p,SKIP_MIP).g;
        if(maxD<ISO_THRESHOLD){ t+=SKIP_STEP; continue; }
        float density=texture(vol,p).r;
        vec4 c=transferFunction(density);
        col+=(1.0-T)*c.a*c.rgb; T*=(1.0-c.a);
        t+=FINE_STEP;
    }
    fragColor=vec4(col,1.0-T);
}
```

[Source: Krueger & Westermann "Acceleration Techniques for GPU-based Volume Rendering", VIS 2003]

### 96.3 Tensor Field Glyphs in Mesh Shaders

Superquadric tensor glyphs (Kindlmann 2004) encode tensor eigenvalues/eigenvectors in glyph shape and orientation. Generated entirely in the mesh shader — no precomputed glyph mesh — one workgroup per grid point:

```glsl
// tensor_glyph.mesh.glsl
layout(triangles, max_vertices=96, max_primitives=64) out;
layout(local_size_x=64) in;
taskPayloadSharedEXT struct { uint count; uint glyphIdx[32]; } payload;
void main() {
    uint g=payload.glyphIdx[gl_WorkGroupID.x];
    vec3 evals=eigenvalues[g]; mat3 evecs=eigenvectors[g];
    // sample superquadric on (theta,phi) parametric grid
    // scale by eigenvalues, apply superquadric nonlinearity, rotate by evecs
    // emit vertices and triangle indices
    SetMeshOutputsEXT(96, 64);
}
```

[Source: Kindlmann "Superquadric Tensor Glyphs", EuroVis 2004]

### 96.4 Time-Varying Isosurface Extraction

For animated scalar fields, limit GPU marching cubes (§3.2) to **dirty voxels** where the field changed:

```glsl
// dirty_mark.comp — mark voxels changed between frames
layout(set=0,binding=0) readonly buffer FieldOld { float fOld[]; };
layout(set=0,binding=1) readonly buffer FieldNew { float fNew[]; };
layout(set=0,binding=2) buffer DirtyBits { uint dirty[]; };
layout(local_size_x=64) in;
void main() {
    uint v=gl_GlobalInvocationID.x;
    if(abs(fNew[v]-fOld[v])>CHANGE_THRESHOLD)
        atomicOr(dirty[v/32], 1u<<(v%32));
}
```

The MC dispatch skips voxels with `dirty[v/32] & (1<<(v%32)) == 0`. For slowly evolving simulations the dirty fraction is typically < 5% per frame, reducing MC compute by ~20×. [Source: Lorensen & Cline SIGGRAPH 1987; temporal delta technique from Kazhdan et al. "Streaming Multigrid", SIGGRAPH 2008]

---

### 97. Decal and Surface Projection Geometry

*Audience: graphics application developers.*

Decals project surface detail (bullet holes, dirt, emblems) onto arbitrary geometry without modifying the underlying mesh. GPU decal systems fall into two families: *deferred decals* (project in screen space using the deferred G-buffer) and *mesh decals* (clip decal polygon against scene triangles on GPU to generate a coplanar mesh).

### 97.1 Deferred Screen-Space Decals

A decal volume is an axis-aligned box in world space. In the deferred pass, each decal dispatches a full-screen quad; the fragment shader tests whether each G-buffer pixel falls inside the decal box and blends the decal texture:

```glsl
// decal.frag — deferred screen-space decal
uniform sampler2D depthBuffer;
uniform sampler2D decalTex;
uniform mat4 decalWorldToLocal;   // transforms world position into decal UVW [0,1]³
uniform mat4 invView, invProj;
in vec2 screenUV;
void main() {
    float z       = texture(depthBuffer, screenUV).r;
    vec4  ndc     = vec4(screenUV*2.0-1.0, z*2.0-1.0, 1.0);
    vec4  worldH  = invView * invProj * ndc;
    vec3  worldP  = worldH.xyz / worldH.w;
    vec3  local   = (decalWorldToLocal * vec4(worldP,1)).xyz;
    if (any(lessThan(local, vec3(0))) || any(greaterThan(local, vec3(1)))) discard;
    vec4 col = texture(decalTex, local.xz);
    fragColor = col;   // blend into albedo / normal GBuffer layer
}
```

For normal-map decals, also unproject the G-buffer normal and blend in tangent space. [Source: Nota "Rendering Wounds: Decal Projection in 'The Last of Us'", SIGGRAPH 2014; Valient "Killzone Shadow Fall: Deferred Rendering", GDC 2014]

### 97.2 Mesh Decal: GPU Polygon Clipping

For masked decals that must respect depth discontinuities or work in forward rendering, clip the decal polygon against each scene triangle using the Sutherland-Hodgman algorithm on GPU:

```glsl
// decal_clip.comp — clip decal quad against one scene triangle
layout(set=0,binding=0) readonly buffer SceneTris { vec3 triVerts[]; };
layout(set=0,binding=1) writeonly buffer DecalTris { vec3 outVerts[]; };
layout(set=0,binding=2) buffer OutCount { uint count; };
layout(local_size_x=64) in;

// Sutherland-Hodgman clip polygon against one half-space
uint clipAgainstPlane(vec3 poly[], uint n, vec4 plane, vec3 out[]) {
    uint outN=0;
    for (uint i=0; i<n; i++) {
        vec3 a=poly[i], b=poly[(i+1)%n];
        float da=dot(vec4(a,1),plane), db=dot(vec4(b,1),plane);
        if(da>=0.0) out[outN++]=a;
        if((da>=0.0)!=(db>=0.0)) out[outN++]=mix(a,b,-da/(db-da));
    }
    return outN;
}
void main() {
    uint t   = gl_GlobalInvocationID.x;
    vec3 tri[3] = {triVerts[t*3], triVerts[t*3+1], triVerts[t*3+2]};
    vec3 poly[8]; memcpy(poly, decalQuad, 4*12);
    uint n   = 4;
    // clip against all 6 decal box planes
    n = clipAgainstPlane(poly, n, decalPlane[0], poly);
    // ... repeat for planes 1-5 ...
    if (n >= 3) {
        uint slot = atomicAdd(count, n-2);
        for (uint i=0; i<n-2; i++) {
            outVerts[(slot+i)*3+0]=poly[0];
            outVerts[(slot+i)*3+1]=poly[i+1];
            outVerts[(slot+i)*3+2]=poly[i+2];
        }
    }
}
```

[Source: Sutherland & Hodgman "Reentrant Polygon Clipping", CACM 1974; Tatarchuk "Practical Parallax Occlusion Mapping with Self-Shadowing", GDC 2006]

### 97.3 Stencil-Volume Projection

A third approach uses the stencil buffer to mask the decal projection volume: draw back faces incrementing stencil, draw front faces decrementing. Pixels with stencil=1 are inside the volume. Then draw the decal with stencil test = equal(1):

```cpp
// Vulkan pseudo-code — stencil volume decal setup
VkStencilOpState backFace{VK_STENCIL_OP_KEEP, VK_STENCIL_OP_INCREMENT_AND_CLAMP, ...};
VkStencilOpState frontFace{VK_STENCIL_OP_KEEP, VK_STENCIL_OP_DECREMENT_AND_CLAMP, ...};
// Pass 1: fill stencil volume
// Pass 2: draw decal with VK_COMPARE_OP_EQUAL, reference=1
```

[Source: McGuire & Hughes "Fast, Practical and Robust Shadows", 2003 (stencil volume technique applied to decals)]

---

### 98. GPU Texture Synthesis and Procedural Shading Geometry

*Audience: graphics application developers.*

Procedural texturing generates surface detail without pre-authored texture assets — essential for infinite terrain, aged materials, and biological patterns. The geometric outputs (normal maps, displacement maps, colour masks) feed directly into the rendering pipeline as GPU images.

### 98.1 Reaction-Diffusion on GPU

Turing's reaction-diffusion system (1952) generates biological patterns — spots, stripes, labyrinth textures — by iterating two coupled PDEs on a 2D grid. The Gray-Scott model is GPU-friendly:

```glsl
// rdiff.comp — Gray-Scott reaction-diffusion step
layout(set=0,binding=0) uniform sampler2D uvIn;    // R=U chemical, G=V chemical
layout(set=0,binding=1, rg32f) uniform writeonly image2D uvOut;
layout(push_constant) uniform PC {
    float Du; float Dv; float F; float k; float dt;
} pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 p   = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = texelFetch(uvIn, p, 0).rg;
    float u   = uv.r, v = uv.g;
    // Discrete Laplacian (5-point stencil)
    float lapU = texelFetch(uvIn,p+ivec2(1,0),0).r + texelFetch(uvIn,p+ivec2(-1,0),0).r
               + texelFetch(uvIn,p+ivec2(0,1),0).r + texelFetch(uvIn,p+ivec2(0,-1),0).r
               - 4.0*u;
    float lapV = texelFetch(uvIn,p+ivec2(1,0),0).g + texelFetch(uvIn,p+ivec2(-1,0),0).g
               + texelFetch(uvIn,p+ivec2(0,1),0).g + texelFetch(uvIn,p+ivec2(0,-1),0).g
               - 4.0*v;
    float uvv  = u * v * v;
    float nu   = u + pc.dt*(pc.Du*lapU - uvv + pc.F*(1.0-u));
    float nv   = v + pc.dt*(pc.Dv*lapV + uvv - (pc.F+pc.k)*v);
    imageStore(uvOut, p, vec4(clamp(nu,0,1), clamp(nv,0,1), 0, 0));
}
```

Iterate 500–5000 steps offline or stream frames in real time for animated biological textures. [Source: Pearson "Complex Patterns in a Simple System", Science 1993; GPU RD: Rumpf & Strzodka "Numerical Methods for the Gray-Scott Model", 2000]

### 98.2 Wang Tiles for Non-Repeating Textures

Wang tiles (Wang 1961, Cohen et al. 2003) tile the plane aperiodically by ensuring adjacent tile edges match colour constraints — eliminating the visible repetition of tiled textures:

```glsl
// wang_sample.glsl — fragment shader texture lookup via Wang tile set
uniform sampler2DArray tileAtlas;  // N tiles × atlas
uniform sampler2D      tileIndex;  // pre-computed Wang tiling index map
in vec2 worldUV;
void main() {
    // Look up which tile occupies this texel's cell
    ivec2 cell  = ivec2(floor(worldUV));
    float tileID= texture(tileIndex, vec2(cell) / float(INDEX_SIZE)).r * NUM_TILES;
    // Within-tile UV
    vec2  localUV = fract(worldUV);
    fragColor = texture(tileAtlas, vec3(localUV, tileID));
}
```

GPU generation of valid Wang tilings: assign edge colours to a grid via a compute shader respecting the Wang constraint, then look up the corresponding tile from the set of 2^(2E) tiles for E edge colours. [Source: Cohen et al. "Wang Tiles for Image and Texture Generation", SIGGRAPH 2003]

### 98.3 Noise-Based Displacement and Normal Map Generation

Generate displacement maps from procedural noise for real-time terrain detail, without pre-authored assets. Combine fBm (§69.1) with domain warping:

```glsl
// displace_gen.comp — generate displacement + normal map from fBm + domain warp
layout(set=0,binding=0, r16f)  uniform writeonly image2D dispMap;
layout(set=0,binding=1, rgba8) uniform writeonly image2D normMap;
layout(push_constant) uniform PC { float scale; float warpStr; int octaves; vec2 offset; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(id)+0.5)/vec2(MAP_SIZE) + pc.offset;
    // Domain warp: offset UV by another fBm
    vec2  warp = vec2(fbm(uv + vec2(1.7,9.2), pc.octaves),
                      fbm(uv + vec2(8.3,2.8), pc.octaves)) * pc.warpStr;
    float h    = fbm(uv + warp, pc.octaves) * pc.scale;
    imageStore(dispMap, id, vec4(h));
    // Central-difference normal from displacement
    float hR = fbm((uv+warp)+vec2(1.0/MAP_SIZE,0), pc.octaves)*pc.scale;
    float hU = fbm((uv+warp)+vec2(0,1.0/MAP_SIZE), pc.octaves)*pc.scale;
    vec3  n  = normalize(vec3(h-hR, 1.0/MAP_SIZE, h-hU));
    imageStore(normMap, id, vec4(n*0.5+0.5, 1.0));
}
```

[Source: Quilez "Inigo Quilez — Articles: Domain Warping" https://iquilezles.org/articles/warp/]

---

### 99. Acoustic Ray Tracing and Sound Geometry

*Audience: systems developers, graphics application developers.*

Geometric acoustics treats sound as rays that reflect off surfaces according to the angle of incidence, attenuate with distance, and are absorbed by surface materials. GPU acceleration uses the same TLAS/BLAS infrastructure as visual ray tracing (§75) but with a much larger number of rays per source and a different termination criterion.

### 99.1 Acoustic Ray Emission

Each sound source emits N rays uniformly distributed over the unit sphere using Halton sampling (§105.1). Rays carry energy and terminate when energy falls below a threshold or when the maximum reflection count is reached:

```glsl
// acoustic_emit.raygen.glsl — emit acoustic rays from point source
#version 460
#extension GL_EXT_ray_tracing : require
struct AcousticRayPayload {
    float energy;        // current energy (starts at 1.0)
    float pathLength;    // accumulated path length
    uint  bounces;       // bounce count
    vec3  direction;
};
layout(location=0) rayPayloadEXT AcousticRayPayload payload;
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) writeonly buffer Impulse { vec2 impulse[]; };  // (energy, time)
layout(push_constant) uniform PC { vec3 sourcePos; float speedOfSound; uint nRays; } pc;
void main() {
    uint rayIdx  = gl_LaunchIDEXT.x;
    vec3 dir     = fibonacciSphere(rayIdx, pc.nRays);
    payload      = AcousticRayPayload(1.0, 0.0, 0u, dir);
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF,
                0, 1, 0, pc.sourcePos, 0.01, dir, MAX_RAY_DIST, 0);
}
```

### 99.2 Acoustic Closest-Hit: Reflection and Absorption

Each surface has a frequency-dependent absorption coefficient α. At each hit, attenuate the ray energy by (1-α) and reflect according to the surface normal:

```glsl
// acoustic_chit.glsl — acoustic closest hit shader
#extension GL_EXT_ray_tracing : require
layout(location=0) rayPayloadInEXT AcousticRayPayload payload;
hitAttributeEXT vec2 bary;
layout(set=0,binding=2) readonly buffer Materials { AcousticMaterial mats[]; };
void main() {
    AcousticMaterial m = mats[gl_InstanceCustomIndexEXT];
    payload.energy    *= (1.0 - m.absorption);
    payload.pathLength += gl_HitTEXT;
    payload.bounces++;
    // Record energy arrival at listener (if ray passes near listener position)
    float distToListener = length(listenerPos - gl_WorldRayOriginEXT + gl_HitTEXT*gl_WorldRayDirectionEXT);
    if (distToListener < LISTENER_RADIUS) {
        uint slot = atomicAdd(impulseCount, 1u);
        float arrivalTime = payload.pathLength / SPEED_OF_SOUND;
        impulse[slot] = vec2(payload.energy / (distToListener*distToListener), arrivalTime);
    }
    if (payload.energy < MIN_ENERGY || payload.bounces >= MAX_BOUNCES) return;
    // Specular reflection
    vec3 n = faceNormal();
    vec3 r = reflect(gl_WorldRayDirectionEXT, n);
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF,
                0, 1, 0, gl_WorldRayOriginEXT + gl_HitTEXT*gl_WorldRayDirectionEXT + n*0.001,
                0.001, r, MAX_RAY_DIST, 0);
}
```

[Source: Vorländer "Auralization: Fundamentals of Acoustics", 2008; GPU acoustic ray tracing: Schissler et al. "High-Order Diffraction and Diffuse Reflections for Interactive Sound Propagation in Large Environments", SIGGRAPH 2014]

### 99.3 Sound Occlusion Query via Ray Query

A cheaper occlusion-only query (no reflection) determines whether a sound source is audible from the listener position — equivalent to a shadow ray (§76.4) but in the audio domain:

```glsl
// sound_occlusion.comp
#extension GL_EXT_ray_query : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) readonly buffer Sources { SourceData sources[]; };
layout(set=0,binding=2) writeonly buffer Occlusion { float occl[]; };
layout(local_size_x=64) in;
void main() {
    uint s    = gl_GlobalInvocationID.x;
    vec3  dir = sources[s].pos - listenerPos;
    float len = length(dir);
    rayQueryEXT rq;
    rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT,
        ACOUSTIC_MASK, listenerPos, 0.01, normalize(dir), len);
    rayQueryProceedEXT(rq);
    occl[s] = (rayQueryGetIntersectionTypeEXT(rq,true)
               == gl_RayQueryCommittedIntersectionNoneEXT) ? 1.0 : 0.0;
}
```

[Source: Moeck et al. "Progressive Perceptual Audio Rendering of Complex Scenes", I3D 2007]

---

### 100. VR/XR Geometry: Reprojection and Foveation

*Audience: graphics application developers, systems developers.*

VR headsets have stringent per-eye frame rate requirements (90–120 Hz) and eye-tracking hardware that identifies the foveal region with sub-1° accuracy. GPU geometry algorithms support two XR-specific techniques: (1) reprojection (ATW/ASW) synthesises missing frames from depth and motion; (2) foveated rendering reduces triangle and shading work outside the fixation point.

### 100.1 Asynchronous TimeWarp (ATW)

ATW (Oculus 2014) re-projects the previous frame's colour+depth into the current head pose. For each output pixel, un-project using the previous frame's depth, transform to the new pose, and re-project:

```glsl
// atw.comp — Asynchronous TimeWarp reprojection
layout(set=0,binding=0) uniform sampler2D prevColor;
layout(set=0,binding=1) uniform sampler2D prevDepth;
layout(set=0,binding=2, rgba8) uniform writeonly image2D outColor;
layout(push_constant) uniform PC {
    mat4 prevViewProj;  // previous frame view-projection
    mat4 currViewProj;  // current head pose view-projection
    mat4 invPrevProj;   // inverse previous projection
} pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv   = (vec2(pix)+0.5)/vec2(imageSize(outColor));
    float d    = texture(prevDepth, uv).r;
    // Unproject from previous clip space to world space
    vec4  clip = vec4(uv*2.0-1.0, d*2.0-1.0, 1.0);
    vec4  world= pc.invPrevProj * clip;
    world /= world.w;
    // Reproject into current head pose
    vec4  curr = pc.currViewProj * world;
    vec2  nuv  = curr.xy/curr.w*0.5+0.5;
    vec4  col  = texture(prevColor, nuv);
    imageStore(outColor, pix, col);
}
```

[Source: Oculus "Asynchronous TimeWarp", Meta Developer Blog 2014; Valve "Low Latency VR Rendering" GDC 2015]

### 100.2 Application SpaceWarp (ASW)

ASW (Meta 2021) synthesises interleaved frames at half the application frame rate using optical flow. A motion vector field (from depth+pose) warps the previous frame to produce the synthesised frame:

```glsl
// asw_motionvec.comp — compute motion vectors from depth + pose change
layout(set=0,binding=0) uniform sampler2D depth;
layout(set=0,binding=1, rg16f) uniform writeonly image2D motionOut;
layout(push_constant) uniform PC { mat4 prevVP; mat4 currVP; mat4 invCurrProj; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(pix)+0.5)/vec2(imageSize(motionOut));
    float d   = texture(depth, uv).r;
    vec4  clip= vec4(uv*2.0-1.0, d*2.0-1.0, 1.0);
    vec4  ws  = pc.invCurrProj * clip; ws /= ws.w;
    vec4  prev= pc.prevVP * ws;
    vec2  prevUV = prev.xy/prev.w*0.5+0.5;
    imageStore(motionOut, pix, vec4(uv - prevUV, 0, 0));
}
// Warp pass: for each output pixel, backward-sample prevColor along motion vector
```

[Source: Meta "Asynchronous SpaceWarp" Developer Blog 2021; Liang et al. "NSFF: Neural Scene Flow Fields for Space-Time View Synthesis", CVPR 2021]

### 100.3 Fixed-Foveated Rendering: Variable Rate Shading

Fixed foveation divides the render target into centre (1×1 shading rate), mid-ring (1×2), and periphery (2×4) regions and uses `VK_KHR_fragment_shading_rate` to reduce shading work. The shading rate image is written by a compute pass using the eye-tracking gaze position:

```glsl
// foveated_sri.comp — write shading rate image from gaze centre
layout(set=0,binding=0, r8ui) uniform writeonly uimage2D shadingRateImg;
layout(push_constant) uniform PC { vec2 gazeUV; float innerR; float outerR; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 tile = ivec2(gl_GlobalInvocationID.xy);  // one thread per shading rate tile (8×8 px)
    vec2  uv   = (vec2(tile)+0.5)/vec2(imageSize(shadingRateImg));
    float d    = length(uv - pc.gazeUV);
    uint  rate;
    if      (d < pc.innerR) rate = VK_FRAGMENT_SHADING_RATE_1_INVOCATION_PER_PIXEL;
    else if (d < pc.outerR) rate = VK_FRAGMENT_SHADING_RATE_1_INVOCATION_PER_2X1_PIXELS;
    else                    rate = VK_FRAGMENT_SHADING_RATE_1_INVOCATION_PER_2X2_PIXELS;
    imageStore(shadingRateImg, tile, uvec4(rate));
}
```

[Source: Guenter et al. "Foveated 3D Graphics", SIGGRAPH Asia 2012; Vulkan `VK_KHR_fragment_shading_rate` specification]

### 100.4 Foveated Mesh LOD

For geometry workload, reduce triangle density in the periphery. A task shader computes the screen-space distance from the foveal centre and selects a meshlet LOD tier accordingly:

```glsl
// foveated_task.task.glsl — select meshlet LOD based on foveal distance
layout(local_size_x=32) in;
void main() {
    uint m = gl_WorkGroupID.x;
    vec4 clip = viewProj * vec4(meshletCentre[m],1.0);
    vec2 ndc  = clip.xy/clip.w;
    float d   = length(ndc - gazeNDC);      // distance from gaze in NDC
    uint lod  = (d < 0.15) ? 0 :            // full detail
                (d < 0.40) ? 1 :            // medium
                              2;             // coarse
    uint meshletBase = lodMeshletBase[m*3+lod];
    uint meshletCount= lodMeshletCount[m*3+lod];
    EmitMeshTasksEXT(meshletCount, 1, 1);
}
```

[Source: Stengel et al. "Adaptive Image-Space Sampling for Gaze-Contingent Real-time Rendering", EG 2016; Meta Quest Pro eye-tracking developer guide 2023]

---

### 101. Silhouette Detection and Feature Lines

*Audience: graphics application developers.*

Silhouette edges are mesh edges where one adjacent face is front-facing and the other is back-facing relative to the view direction. GPU detection operates on the half-edge data structure, parallelising over all edges. Feature lines (ridges, valleys, suggestive contours) require curvature information from §38 spectral processing.

### 101.1 Silhouette Edge Detection on Half-Edge Mesh

For each edge, test the dot product of the face normals with the view vector. Emit geometry for silhouette edges:

```glsl
// silhouette_detect.comp — detect silhouette edges from half-edge mesh
layout(set=0,binding=0) readonly buffer Pos       { vec3 pos[];   };
layout(set=0,binding=1) readonly buffer FaceNorm  { vec3 fnorm[]; };
layout(set=0,binding=2) readonly buffer HalfEdge  { HalfEdgeDS he[]; };
layout(set=0,binding=3) buffer SilEdges { uvec2 silEdges[]; uint count; };
layout(push_constant) uniform PC { vec3 viewPos; } pc;
layout(local_size_x=64) in;
void main() {
    uint e   = gl_GlobalInvocationID.x;
    if (he[e].twin == INVALID) return;  // boundary edge: always emit
    uint f0  = he[e].face, f1 = he[he[e].twin].face;
    vec3 eP  = pos[he[e].vert];
    float d0 = dot(fnorm[f0], pc.viewPos - eP);
    float d1 = dot(fnorm[f1], pc.viewPos - eP);
    if (d0 * d1 < 0.0) {  // sign change → silhouette
        uint slot = atomicAdd(count, 1u);
        silEdges[slot] = uvec2(he[e].vert, he[he[e].twin].vert);
    }
}
```

[Source: Gooch et al. "A Non-Photorealistic Lighting Model For Automatic Technical Illustration", SIGGRAPH 1998; Card & Mitchell "Non-Photorealistic Rendering with Pixel and Vertex Shaders", ShaderX 2002]

### 101.2 Silhouette Fattening via Mesh Shader

Expand silhouette edges into thin quads (fin geometry) in the mesh shader to produce anti-aliased outlines without geometry shader overhead:

```glsl
// silhouette_mesh.mesh.glsl — emit fin quad for each silhouette edge
layout(local_size_x=32) in;
layout(triangles, max_vertices=4, max_primitives=2) out;
void main() {
    uint e    = gl_WorkGroupID.x;
    vec3 v0   = worldPos[silEdges[e].x];
    vec3 v1   = worldPos[silEdges[e].y];
    vec3 view = normalize(cameraPos - (v0+v1)*0.5);
    vec3 edge = normalize(v1-v0);
    vec3 outDir = normalize(cross(edge, view));  // extrude direction
    // Emit 4 vertices: inner edge v0,v1; outer edge v0+w*out, v1+w*out
    SetMeshOutputsEXT(4, 2);
    gl_MeshVerticesEXT[0].gl_Position = viewProj*vec4(v0,1);
    gl_MeshVerticesEXT[1].gl_Position = viewProj*vec4(v1,1);
    gl_MeshVerticesEXT[2].gl_Position = viewProj*vec4(v0+outDir*OUTLINE_WIDTH,1);
    gl_MeshVerticesEXT[3].gl_Position = viewProj*vec4(v1+outDir*OUTLINE_WIDTH,1);
    gl_PrimitiveTriangleIndicesEXT[0] = uvec3(0,1,2);
    gl_PrimitiveTriangleIndicesEXT[1] = uvec3(1,3,2);
}
```

[Source: Raskar & Cohen "Image Precision Silhouette Edges on a GPU", I3D 1999; Isenberg et al. "A Developer's Guide to Silhouette Algorithms for Polygonal Models", IEEE CGA 2003]

### 101.3 Ridge and Valley Lines from Principal Curvature

Ridges are loci where the maximum principal curvature κ₁ is locally maximal along its curvature direction e₁. Detect them per-edge by sign change of the directional derivative of κ₁:

```glsl
// ridge_valley.comp — detect ridge/valley lines from principal curvature
layout(set=0,binding=0) readonly buffer Kappa  { vec2 kappa[]; };  // k1,k2 per vertex
layout(set=0,binding=1) readonly buffer E1dir  { vec3 e1[];    };  // max curvature direction
layout(set=0,binding=2) readonly buffer HalfEdge { HalfEdgeDS he[]; };
layout(set=0,binding=3) buffer RidgeEdges { uvec2 ridges[]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint e  = gl_GlobalInvocationID.x;
    uint vi = he[e].vert, vj = he[he[e].twin].vert;
    // Directional derivative of k1 along edge direction
    vec3  edgeDir = normalize(pos[vj]-pos[vi]);
    float dk1i    = dot(e1[vi], edgeDir) * kappa[vi].x;
    float dk1j    = dot(e1[vj], edgeDir) * kappa[vj].x;
    if (dk1i * dk1j < 0.0 && max(kappa[vi].x, kappa[vj].x) > RIDGE_THRESHOLD) {
        uint slot = atomicAdd(count, 1u);
        ridges[slot] = uvec2(vi, vj);
    }
}
```

[Source: Ohtake et al. "Ridge-Valley Lines on Meshes via Implicit Surface Fitting", SIGGRAPH 2004; DeCarlo et al. "Suggestive Contours for Conveying Shape", SIGGRAPH 2003]

### 101.4 Suggestive Contours

Suggestive contours are where the radial curvature κᵣ (curvature in the view direction projected onto the surface) crosses zero with negative derivative — predicting silhouettes that would appear under small view perturbations:

```glsl
// suggestive_contour.comp — find suggestive contour edges
layout(set=0,binding=0) readonly buffer Kappa  { vec2 kappa[]; };   // k1, k2
layout(set=0,binding=1) readonly buffer PrinDir { vec3 e1[]; vec3 e2[]; };
layout(set=0,binding=2) readonly buffer Pos    { vec3 pos[]; };
layout(set=0,binding=3) readonly buffer HalfEdge { HalfEdgeDS he[]; };
layout(set=0,binding=4) buffer SCEdges { uvec2 sc[]; uint count; };
layout(push_constant) uniform PC { vec3 viewPos; } pc;
layout(local_size_x=64) in;

float radialCurvature(uint v) {
    vec3  w    = normalize(pc.viewPos - pos[v]);
    vec3  wt   = normalize(w - dot(w, nor[v])*nor[v]);  // project to tangent
    float cosA = dot(wt, e1[v]);
    float sinA = dot(wt, e2[v]);
    return kappa[v].x*cosA*cosA + kappa[v].y*sinA*sinA;
}
void main() {
    uint e  = gl_GlobalInvocationID.x;
    uint vi = he[e].vert, vj = he[he[e].twin].vert;
    float ki = radialCurvature(vi), kj = radialCurvature(vj);
    if (ki * kj < 0.0)  {  // zero crossing of radial curvature
        uint slot = atomicAdd(count, 1u);
        sc[slot] = uvec2(vi, vj);
    }
}
```

[Source: DeCarlo et al. 2003; Judd et al. "Apparent Ridges for Line Drawing", SIGGRAPH 2007]

---

### 102. Micro-Polygon Displacement and GPU Tessellation

*Audience: graphics application developers.*

Displacement mapping (Cook 1984) adds fine surface detail by offsetting vertices along the normal by a scalar value sampled from a height texture. GPU tessellation via the hardware tessellator (Vulkan tessellation shaders) or mesh shaders generates the dense vertex grid at runtime; §1 subdivision surfaces and §33 virtual geometry represent related approaches. True micro-polygon displacement requires crack-free patch stitching and adaptive tessellation factor computation.

### 102.1 Adaptive Tessellation Factor Computation

Tessellation factors are set per-edge based on projected edge length or curvature, ensuring one tessellation sample per pixel:

```glsl
// tess_control.tesc — adaptive edge tessellation factors from projected length
#version 460
layout(vertices=3) out;
layout(push_constant) uniform PC { mat4 mvp; vec2 viewportRes; float targetEdgePx; } pc;
void main() {
    gl_out[gl_InvocationID].gl_Position = gl_in[gl_InvocationID].gl_Position;
    if (gl_InvocationID == 0) {
        // Project each edge midpoint, compute screen-space length
        for (int e=0;e<3;e++) {
            vec4 p0 = pc.mvp * gl_in[e].gl_Position;
            vec4 p1 = pc.mvp * gl_in[(e+1)%3].gl_Position;
            vec2 s0 = (p0.xy/p0.w*0.5+0.5)*pc.viewportRes;
            vec2 s1 = (p1.xy/p1.w*0.5+0.5)*pc.viewportRes;
            gl_TessLevelOuter[e] = clamp(length(s1-s0)/pc.targetEdgePx, 1.0, 64.0);
        }
        gl_TessLevelInner[0] = (gl_TessLevelOuter[0]+gl_TessLevelOuter[1]+gl_TessLevelOuter[2])/3.0;
    }
}
```

[Source: Schwarz & Stamminger "Bitmask Soft Shadows", EG 2007; Nießner et al. "Feature-Adaptive GPU Rendering of Catmull-Clark Subdivision Surfaces", SIGGRAPH 2012]

### 102.2 Displacement Evaluation in Tessellation Evaluation Shader

```glsl
// tess_eval.tese — displacement along interpolated normal
#version 460
layout(triangles, fractional_odd_spacing, ccw) in;
layout(push_constant) uniform PC { mat4 mvp; float dispScale; } pc;
uniform sampler2D heightTex;
in vec3 tcNormal[];
in vec2 tcUV[];
void main() {
    // Barycentric interpolation of position, normal, UV
    vec3 pos = gl_TessCoord.x*gl_in[0].gl_Position.xyz
             + gl_TessCoord.y*gl_in[1].gl_Position.xyz
             + gl_TessCoord.z*gl_in[2].gl_Position.xyz;
    vec3 nor = normalize(gl_TessCoord.x*tcNormal[0]
                        +gl_TessCoord.y*tcNormal[1]
                        +gl_TessCoord.z*tcNormal[2]);
    vec2 uv  = gl_TessCoord.x*tcUV[0]+gl_TessCoord.y*tcUV[1]+gl_TessCoord.z*tcUV[2];
    float h  = texture(heightTex, uv).r;
    pos     += nor * h * pc.dispScale;
    gl_Position = pc.mvp * vec4(pos, 1.0);
}
```

[Source: Cook "Shade Trees", SIGGRAPH 1984; Tatarchuk "Practical Parallax Occlusion Mapping for Highly Detailed Surface Rendering", ShaderX 5, 2006]

### 102.3 Crack-Free Patch Stitching

Adjacent patches with different tessellation factors produce T-junctions and cracks. The hardware tessellator handles this within a patch, but between patches the outer tessellation levels must match. Compute the outer tessellation factor for each shared edge identically from both patches:

```glsl
// tess_control_crackfree.tesc — crack-free outer tessellation via shared edge hash
layout(push_constant) uniform PC { mat4 mvp; vec2 res; float targetPx; } pc;
float edgeTessLevel(vec3 p0, vec3 p1) {
    vec4 c0=pc.mvp*vec4(p0,1), c1=pc.mvp*vec4(p1,1);
    vec2 s0=(c0.xy/c0.w*0.5+0.5)*pc.res, s1=(c1.xy/c1.w*0.5+0.5)*pc.res;
    return clamp(length(s1-s0)/pc.targetPx, 1.0, 64.0);
}
void main() {
    // Both patches sharing edge (v0,v1) must call edgeTessLevel(v0,v1)
    // in the same vertex order → sorted by vertex index
    uvec2 e = uvec2(min(vIdx[0],vIdx[1]), max(vIdx[0],vIdx[1]));
    gl_TessLevelOuter[0] = edgeTessLevel(pos[e.x],pos[e.y]);
    // ... similarly for other edges
}
```

[Source: Nießner et al. 2012; Walton "Tessellation on Any Budget", GDC 2011]

### 102.4 Mesh Shader Displacement for Nanite-Compatible Micro-Polygons

When using §33 virtual geometry, displacement can be applied in the mesh shader itself, offsetting cluster vertices along normals sampled from a virtual texture:

```glsl
// disp_mesh.mesh.glsl — apply displacement in mesh shader for cluster rendering
layout(local_size_x=128) in;
layout(triangles, max_vertices=128, max_primitives=126) out;
void main() {
    uint v = gl_LocalInvocationID.x;
    SetMeshOutputsEXT(N_VERTS, N_TRIS);
    if (v < N_VERTS) {
        vec3 pos = clusterVerts[v];
        vec3 nor = clusterNormals[v];
        vec2 uv  = clusterUVs[v];
        float h  = texture(heightMap, uv).r;
        pos     += nor * h * dispScale;
        gl_MeshVerticesEXT[v].gl_Position = mvp * vec4(pos,1.0);
        outUV[v] = uv; outNor[v] = nor;
    }
    if (v < N_TRIS) gl_PrimitiveTriangleIndicesEXT[v] = clusterTris[v];
}
```

[Source: Karis et al. 2021 §90; Kubisch "Turing Mesh Shaders", NVIDIA 2018]

---

### 103. GPU SDF Font and Curve Rendering

*Audience: graphics application developers.*

Signed distance field (SDF) fonts (Green 2007) pre-render each glyph to a low-resolution SDF texture; at runtime, the fragment shader reconstructs sharp edges regardless of scale by thresholding the interpolated SDF value. Multi-channel SDF (MSDF, Chlumský 2016) encodes the distance in three colour channels to preserve sharp corners. Loop-Blinn (2005) renders cubic Bézier curves exactly on the GPU without pre-baking.

### 103.1 Offline SDF Glyph Baking

Bake glyph outlines (from FreeType) into a low-resolution SDF texture by computing the signed distance from each texel to the nearest contour edge:

```glsl
// sdf_bake.comp — bake SDF for a single glyph from filled scanline bitmap
layout(set=0,binding=0) readonly buffer GlyphBitmap { uint bitmap[]; };  // 1-bit per pixel
layout(set=0,binding=1, r8_snorm) writeonly image2D sdfOut;
layout(push_constant) uniform PC { ivec2 bitmapDim; ivec2 sdfDim; int searchRadius; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 sdfPix = ivec2(gl_GlobalInvocationID.xy);
    if (any(greaterThanEqual(sdfPix,pc.sdfDim))) return;
    // Map SDF pixel to bitmap space (upsample factor)
    ivec2 bPix  = sdfPix * pc.bitmapDim / pc.sdfDim;
    bool inside = bitfieldExtract(bitmap[bPix.y*(pc.bitmapDim.x/32)+bPix.x/32], bPix.x%32, 1) != 0;
    float minD  = float(pc.searchRadius);
    for (int dy=-pc.searchRadius;dy<=pc.searchRadius;dy++)
    for (int dx=-pc.searchRadius;dx<=pc.searchRadius;dx++) {
        ivec2 nb  = bPix+ivec2(dx,dy);
        if (any(lessThan(nb,ivec2(0)))||any(greaterThanEqual(nb,pc.bitmapDim))) continue;
        bool nbIn = bitfieldExtract(bitmap[nb.y*(pc.bitmapDim.x/32)+nb.x/32], nb.x%32, 1) != 0;
        if (nbIn != inside) minD = min(minD, length(vec2(dx,dy)));
    }
    float sdf = (inside ? 1.0 : -1.0) * minD / float(pc.searchRadius) * 0.5;
    imageStore(sdfOut, sdfPix, vec4(sdf));
}
```

[Source: Green "Improved Alpha-Tested Magnification for Vector Textures and Special Effects", SIGGRAPH 2007]

### 103.2 Runtime SDF Font Rendering

In the fragment shader, threshold the SDF and apply anti-aliasing using the screen-space derivative of the SDF:

```glsl
// sdf_font.frag — render SDF glyph with smooth anti-aliased edges
uniform sampler2D sdfAtlas;
uniform vec4 textColor;
in vec2 texUV;
void main() {
    float d     = texture(sdfAtlas, texUV).r;
    // Width of the AA region in SDF units: ~0.5 / (screen pixels per SDF texel)
    float width = fwidth(d) * 0.7;
    float alpha = smoothstep(0.5 - width, 0.5 + width, d);
    fragColor   = vec4(textColor.rgb, textColor.a * alpha);
}
```

[Source: Green 2007; Behdad "State of Text Rendering 2", 2024]

### 103.3 Multi-Channel SDF (MSDF) for Sharp Corners

MSDF (Chlumský 2016) stores the minimum distance to the nearest contour edge in three separate colour channels (one per pseudo-distance direction), then takes the median at render time to reconstruct sharp corners that single-channel SDF blurs:

```glsl
// msdf_font.frag — render MSDF glyph: median of three channels for sharp corners
uniform sampler2D msdfAtlas;
uniform vec4 textColor;
in vec2 texUV;
float median(float r, float g, float b) { return max(min(r,g),min(max(r,g),b)); }
void main() {
    vec3  msd   = texture(msdfAtlas, texUV).rgb;
    float d     = median(msd.r, msd.g, msd.b);
    float width = fwidth(d) * 0.7;
    float alpha = smoothstep(0.5-width, 0.5+width, d);
    fragColor   = vec4(textColor.rgb, textColor.a * alpha);
}
```

[Source: Chlumský "Shape Decomposition for Multi-Channel Distance Fields", Master's thesis, Czech Technical University 2015]

### 103.4 Loop-Blinn Cubic Bézier Rendering

Loop & Blinn (2005) render cubic Bézier curves exactly on the GPU by encoding each bezier patch as a quad, computing a cubic discriminant texture, and discarding fragments outside the curve in the fragment shader:

```glsl
// loop_blinn.frag — inside/outside test for cubic Bézier via Loop-Blinn klmn coords
in vec3 klmn;  // per-vertex cubic texture coordinates from vertex shader
void main() {
    // Fragment is inside the loop/serpentine/cusp if klmn.x³ - klmn.y*klmn.z < 0
    float f = klmn.x*klmn.x*klmn.x - klmn.y*klmn.z;
    if (f > 0.0) discard;  // outside curve fill region
    fragColor = vec4(vec3(0), 1.0);  // inside: fill with text color
}
```

[Source: Loop & Blinn "Resolution Independent Curve Rendering Using Programmable Graphics Hardware", SIGGRAPH 2005; Nehab & Hoppe "Random-Access Rendering of General Vector Graphics", SIGGRAPH Asia 2008]

---

### 104. Molecular Surface Computation

*Audience: systems developers.*

Molecular surfaces (solvent-accessible surface, SAS; solvent-excluded surface, SES; Connolly surface) define the geometric boundary between a protein and its solvent. GPU algorithms compute these surfaces for real-time docking visualization, drug discovery pipelines, and volumetric electrostatics. Inputs are atom positions and van der Waals radii; outputs are triangle meshes or SDF grids.

### 104.1 Solvent-Accessible Surface via SDF Grid

The SAS is the union of spheres of radius rᵢ + rₛ centred on each atom (rᵢ = van der Waals, rₛ = solvent probe). Compute as a SDF grid and extract with marching cubes (§65):

```glsl
// molsurf_sas.comp — compute SAS SDF: min distance to expanded atom spheres
layout(set=0,binding=0) readonly buffer Atoms { vec4 atoms[]; };  // xyz + vdW radius
layout(set=0,binding=1) buffer SDF { float sdf[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; float probeR; uint N; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    vec3  xg = pc.gridMin + (vec3(c)+0.5)*pc.dx;
    float d  = 1e30;
    for (uint a=0; a<pc.N; a++) {
        float r = atoms[a].w + pc.probeR;  // expanded radius
        d = min(d, length(xg - atoms[a].xyz) - r);
    }
    sdf[c.z*pc.dim.y*pc.dim.x + c.y*pc.dim.x + c.x] = d;
}
// Then run §65 marching cubes on sdf[] at isovalue 0.0
```

[Source: Connolly "Analytical Molecular Surface Calculation", Journal of Applied Crystallography 1983; Krone et al. "GPU-Based Interactive Visualization Techniques for Molecular Dynamics Simulations", Computer Graphics Forum 2012]

### 104.2 Solvent-Excluded Surface via Rolling Probe

The SES (Connolly surface) accounts for where the probe sphere cannot fit between atoms — the re-entrant surface. GPU computation identifies contact circles and toroidal re-entrant patches:

```glsl
// molsurf_ses.comp — classify SES patch type per grid voxel
layout(set=0,binding=0) readonly buffer Atoms { vec4 atoms[]; };
layout(set=0,binding=1) buffer SDF { float sdf[]; };
layout(push_constant) uniform PC { vec3 gridMin; float dx; ivec3 dim; float probeR; uint N; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c  = ivec3(gl_GlobalInvocationID);
    vec3  xg = pc.gridMin + (vec3(c)+0.5)*pc.dx;
    // Probe-center-accessible surface: min distance from probe centre to SAS
    float dSAS = 1e30;
    for (uint a=0; a<pc.N; a++) dSAS = min(dSAS, length(xg-atoms[a].xyz) - (atoms[a].w+pc.probeR));
    // SES: subtract probe radius from probe-accessible SDF
    sdf[flatIdx(c,pc.dim)] = dSAS - pc.probeR;
    // Result: negative inside SES, positive outside; isovalue 0 gives the surface
}
```

[Source: Connolly 1983; Parulek & Viola "Implicit Representation of Molecular Surfaces", IEEE TVCG 2012]

### 104.3 Ambient Occlusion for Molecular Visualization

Molecular AO integrates over the hemisphere above each surface point whether other atoms occlude it — a proxy for solvent accessibility and electrostatic exposure:

```glsl
// mol_ao.comp — ambient occlusion for molecular surface via ray queries
layout(set=0,binding=0) readonly buffer SurfPts { vec3 spts[];  };
layout(set=0,binding=1) readonly buffer SurfNor  { vec3 snor[];  };
layout(set=0,binding=2) uniform accelerationStructureEXT tlas;  // BVH of atom spheres §31
layout(set=0,binding=3) writeonly buffer AO { float ao[]; };
layout(push_constant) uniform PC { int nRays; float maxDist; } pc;
layout(local_size_x=64) in;
void main() {
    uint v = gl_GlobalInvocationID.x;
    vec3 p = spts[v] + snor[v]*1e-3;
    float occ = 0.0;
    for (int r=0; r<pc.nRays; r++) {
        vec3 d = cosineWeightedDir(snor[v], halton2D(r));
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT, 0xFF, p, 0.0, d, pc.maxDist);
        rayQueryProceedEXT(rq);
        if (rayQueryGetIntersectionTypeEXT(rq,true)!=gl_RayQueryCommittedIntersectionNoneEXT)
            occ += 1.0;
    }
    ao[v] = 1.0 - occ / float(pc.nRays);
}
```

[Source: Tarini et al. "Ambient Occlusion and Edge Cueing to Enhance Real Time Molecular Visualization", IEEE TVCG 2006; §62 RTAO pattern]

### 104.4 Electrostatic Potential on Surface (Poisson-Boltzmann)

Map the Poisson-Boltzmann electrostatic potential onto the molecular surface by solving the linearised PB equation on the SDF grid with a conjugate gradient solver (§37.2 pattern):

```glsl
// poisson_boltzmann.comp — one Jacobi iteration of linearised PB equation
// ∇²φ = κ²φ - ρ/ε₀ (inside cavity: κ=0; outside: κ=Debye screening)
layout(set=0,binding=0) readonly buffer SDF     { float sdf[];   };  // §104.2 surface
layout(set=0,binding=1) readonly buffer Charges { vec4  charges[]; }; // xyz + charge
layout(set=0,binding=2) buffer Phi { float phi[]; };
layout(push_constant) uniform PC { ivec3 dim; float dx; float kappa2_out; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
void main() {
    ivec3 c   = ivec3(gl_GlobalInvocationID);
    if (any(greaterThanEqual(c,pc.dim))) return;
    uint idx  = flatIdx(c,pc.dim);
    float inside = step(sdf[idx], 0.0);  // 1=inside, 0=outside
    float kappa2  = pc.kappa2_out * (1.0-inside);  // screening only outside
    // 6-point Laplacian
    float lap = (phi[flatIdx(c+ivec3(1,0,0),pc.dim)] + phi[flatIdx(c-ivec3(1,0,0),pc.dim)]
               + phi[flatIdx(c+ivec3(0,1,0),pc.dim)] + phi[flatIdx(c-ivec3(0,1,0),pc.dim)]
               + phi[flatIdx(c+ivec3(0,0,1),pc.dim)] + phi[flatIdx(c-ivec3(0,0,1),pc.dim)]) / (pc.dx*pc.dx);
    float rho = atomChargeDensity(c, charges);
    phi[idx]  = (lap - rho) / (6.0/(pc.dx*pc.dx) + kappa2);
}
```

[Source: Baker et al. "Electrostatics of Nanosystems: Application to Microtubules and the Ribosome", PNAS 2001; APBS software; Sharp & Honig "Electrostatic Interactions in Macromolecules", Annual Review 1990]

---


---

## XIII. GPU Algorithm Primitives for Geometry

These are the low-level compute algorithms that serve as building blocks for the higher-level geometry techniques throughout this chapter. GPU Poisson disk and blue noise sampling underpin mesh point distribution, lightmap texel placement, and stochastic rendering; parallel convex hull computation (the Quickhull algorithm parallelized over GPU threads) appears inside GJK collision detection and computational geometry tools; and GPU radix sort is the enabling primitive for LBVH BVH construction, depth-sorting for OIT, and particle simulation step reordering. These three algorithms rarely appear in isolation — they are invoked implicitly inside the vast majority of the algorithms in the other twelve categories.

### 105. GPU Sampling: Poisson Disk and Blue Noise

*Audience: graphics application developers, systems developers.*

Low-discrepancy sampling, Poisson disk distributions, and blue noise are foundational primitives used throughout the chapter: AO hemisphere samples (§28), particle seeding (§72), LOD representative points (§10), and Monte Carlo integration in §77 IBL preprocessing. This section covers GPU-native generation of these distributions.

### 105.1 Low-Discrepancy Sequences: Halton and Sobol

Halton sequences in base b are computed per-sample with no state — fully parallel:

```glsl
// halton.glsl — single-sample Halton in base b
float halton(uint index, uint base) {
    float f = 1.0, r = 0.0;
    uint i = index;
    while (i > 0u) { f /= float(base); r += f * float(i % base); i /= base; }
    return r;
}
vec2 halton2D(uint i) { return vec2(halton(i,2), halton(i,3)); }
```

Sobol sequences require a precomputed direction-number table (max 21201 dimensions, 32 bits each) stored in a GPU buffer. [Source: Joe & Kuo "Constructing Sobol Sequences with Better Two-Dimensional Projections", SIAM J. Sci. Comput. 2010; GPU Sobol: Pharr et al. "Physically Based Rendering", 4th ed.]

### 105.2 Area-Weighted Triangle Sampling on Mesh Surfaces

Sample a uniform point on a mesh surface: select a triangle with probability ∝ area (CDF inversion on prefix-sum area array), then sample within the triangle using Osada's square-root method:

```glsl
// surface_sample.comp — sample N uniform points on mesh surface
layout(set=0,binding=0) readonly buffer AreaCDF  { float cdf[];   };  // prefix-sum areas
layout(set=0,binding=1) readonly buffer Tris     { vec3  tv[];    };  // 3 verts per tri
layout(set=0,binding=2) writeonly buffer Samples { vec3  samples[]; };
layout(local_size_x=64) in;
void main() {
    uint  i  = gl_GlobalInvocationID.x;
    vec2  xi = halton2D(i);
    // Binary search in CDF for triangle index
    uint lo=0u, hi=N_TRIS;
    while (lo<hi) { uint mid=(lo+hi)/2u; if(cdf[mid]<xi.x*cdf[N_TRIS]) lo=mid+1u; else hi=mid; }
    uint  t  = lo;
    float r1 = sqrt(xi.y), r2 = halton(i, 5);
    samples[i] = (1.0-r1)*tv[t*3] + r1*(1.0-r2)*tv[t*3+1] + r1*r2*tv[t*3+2];
}
```

[Source: Osada et al. "Shape Distributions", SIGGRAPH 2002]

### 105.3 GPU Poisson Disk Dart Throwing

Generate a Poisson disk distribution with minimum distance r via GPU dart throwing with spatial hash rejection:

```glsl
// poisson_dart.comp — one round of dart throwing
layout(set=0,binding=0) buffer Accepted { vec3 pts[]; };
layout(set=0,binding=1) buffer AccCount { uint count; };
layout(set=0,binding=2) buffer Grid     { uint gridPts[]; uint gridCnt[]; };  // §27.2 hash
layout(push_constant) uniform PC { float minDist; uint round; uint nDarts; } pc;
layout(local_size_x=64) in;
void main() {
    uint  i    = gl_GlobalInvocationID.x;
    vec3  dart = randomPointInDomain(i + pc.round * pc.nDarts);
    ivec3 ci   = ivec3(floor(dart / pc.minDist));
    bool  ok   = true;
    for (int dz=-2;dz<=2&&ok;dz++) for(int dy=-2;dy<=2&&ok;dy++) for(int dx=-2;dx<=2&&ok;dx++) {
        uint cell = hashCell3(ci+ivec3(dx,dy,dz));
        for (uint k=0; k<gridCnt[cell]&&ok; k++)
            if (length(dart - pts[gridPts[cell*MAX_PER_CELL+k]]) < pc.minDist) ok=false;
    }
    if (ok) {
        uint slot = atomicAdd(count, 1u);
        pts[slot] = dart;
        insertGrid(dart, slot);
    }
}
```

[Source: Bridson "Fast Poisson Disk Sampling in Arbitrary Dimensions", SIGGRAPH 2007 Sketches]

### 105.4 Blue Noise via Lloyd Relaxation on Mesh

Centroidal Voronoi tessellation (CVT) gives blue-noise-quality distributions with perfectly uniform coverage. Iterate: (1) partition surface into Voronoi cells (nearest-sample per surface point); (2) move each sample to the centroid of its Voronoi cell:

```glsl
// lloyd_relax.comp — one Lloyd iteration: find centroid of each Voronoi cell
layout(set=0,binding=0) buffer Samples { vec3 pts[]; };
layout(set=0,binding=1) readonly buffer Surface{ vec3 surf[]; };  // dense surface sample set
layout(set=0,binding=2) buffer CentAcc { vec3 acc[]; uint cnt[]; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;   // surface point index
    // Find nearest Poisson disk sample (GPU k-NN §87.1 or brute force for small N)
    uint nearest = findNearest(surf[p], pts, N_SAMPLES);
    atomicAdd_vec3(acc[nearest], surf[p]);
    atomicAdd(cnt[nearest], 1u);
}
// Post-pass: pts[i] = acc[i] / cnt[i] (project back to surface)
```

[Source: Du et al. "Centroidal Voronoi Tessellations: Applications and Algorithms", SIAM Review 1999]

---

### 106. GPU Convex Hull Computation

*Audience: systems developers, graphics application developers.*

Convex hulls are needed to bootstrap rigid body shapes (§50), compute GJK support functions (§24), and fit BVH leaf bounds (§31). GPU QuickHull operates on point clouds of up to tens of millions of points; GPU incremental insertion processes batches of points in parallel.

### 106.1 GPU QuickHull: Initial Simplex

QuickHull begins by finding the 6 extreme points (±X, ±Y, ±Z), forming an initial tetrahedron. Find extremes in parallel via a reduction:

```glsl
// qhull_extremes.comp — find 6 axis-aligned extremes via parallel reduction
layout(set=0,binding=0) readonly buffer Pts { vec3 pts[]; };
layout(set=0,binding=1) buffer Extremes { vec3 extr[6]; };  // ±x, ±y, ±z extremes
layout(local_size_x=256) in;
shared vec3 sMin[256], sMax[256];
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint li = gl_LocalInvocationID.x;
    sMin[li] = (i < N_PTS) ? pts[i] : vec3(1e30);
    sMax[li] = (i < N_PTS) ? pts[i] : vec3(-1e30);
    barrier();
    for (uint s=128u; s>0u; s>>=1) {
        if (li<s) { sMin[li]=min(sMin[li],sMin[li+s]); sMax[li]=max(sMax[li],sMax[li+s]); }
        barrier();
    }
    if (li==0) {
        atomicMin_vec3(extr[0], sMin[0]);  // -x extreme
        atomicMax_vec3(extr[1], sMax[0]);  // +x, etc.
    }
}
```

### 106.2 GPU QuickHull: Furthest-Point Assignment

For each face of the current hull, find the point furthest above it (positive distance). Points inside all faces are discarded:

```glsl
// qhull_assign.comp — assign outside points to faces, find furthest per face
layout(set=0,binding=0) readonly buffer Pts   { vec3 pts[];   };
layout(set=0,binding=1) readonly buffer Faces { vec4 planes[]; };  // normal + d
layout(set=0,binding=2) buffer Assignment { int assign[];  };  // face index or -1 (inside)
layout(set=0,binding=3) buffer FurthestD  { float maxDist[]; };  // per face
layout(set=0,binding=4) buffer FurthestI  { uint  maxIdx[];  };  // per face
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    if (p >= N_PTS) return;
    float bestD = 0.0; int bestF = -1;
    for (uint f=0; f<N_FACES; f++) {
        float d = dot(planes[f].xyz, pts[p]) + planes[f].w;
        if (d > bestD) { bestD=d; bestF=int(f); }
    }
    assign[p] = bestF;
    if (bestF >= 0) atomicMax_float_idx(maxDist[bestF], maxIdx[bestF], bestD, p);
}
```

[Source: Barber et al. "The Quickhull Algorithm for Convex Hulls", ACM TOMS 1996; GPU QuickHull: Cao et al. "Parallel Bounding Volume Hierarchies", SIGGRAPH 2010]

### 106.3 Horizon Edge and Face Expansion

Once the furthest point p for a face is found, compute the horizon ridge (edges visible from p), create new faces from p to each horizon edge, and remove the visible faces:

```glsl
// qhull_expand.comp — find horizon edges visible from furthest point
layout(set=0,binding=0) readonly buffer Faces { HullFace faces[]; };  // v0,v1,v2 + neighbours
layout(set=0,binding=1) readonly buffer Pts   { vec3 pts[]; };
layout(set=0,binding=2) buffer HorizonEdges { uvec2 hedges[]; uint hedgeCount; };
layout(push_constant) uniform PC { uint apexIdx; uint seedFace; } pc;
layout(local_size_x=1) in;  // serial BFS; parallelism across separate hull operations
void main() {
    // BFS from seedFace: mark faces visible from pts[apexIdx], collect horizon edges
    uint queue[MAX_FACES]; uint head=0, tail=0;
    queue[tail++] = pc.seedFace; visibleMask[pc.seedFace]=1u;
    while (head<tail) {
        uint f = queue[head++];
        for (int e=0;e<3;e++) {
            uint nb = faces[f].neighbours[e];
            float d  = dot(faces[nb].plane.xyz, pts[pc.apexIdx]) + faces[nb].plane.w;
            if (d > 0.0) { if (!visibleMask[nb]) { visibleMask[nb]=1u; queue[tail++]=nb; }}
            else {  // horizon edge: boundary between visible and non-visible
                uint slot = atomicAdd(hedgeCount, 1u);
                hedges[slot] = uvec2(faces[f].verts[e], faces[f].verts[(e+1)%3]);
            }
        }
    }
}
```

[Source: Barber et al. 1996; O'Brien "Fast and Simple Physics Using Sequential Impulses", GDC 2006]

### 106.4 Incremental Parallel Hull for Point Batches

For massive point clouds, process points in batches: run §106.1–82.3 serially on a random initial subset (1000 points), then for the remaining points, classify inside/outside in parallel (§106.2) and only add points outside the current hull:

```glsl
// qhull_batch_classify.comp — classify new batch against current hull
layout(set=0,binding=0) readonly buffer NewPts  { vec3 newPts[];  };
layout(set=0,binding=1) readonly buffer HullPlanes { vec4 planes[]; };
layout(set=0,binding=2) buffer Outside { vec3 outside[]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    for (uint f=0; f<N_HULL_FACES; f++)
        if (dot(planes[f].xyz, newPts[p]) + planes[f].w > 1e-5) {
            uint slot = atomicAdd(count, 1u);
            outside[slot] = newPts[p];
            return;
        }
    // Inside all planes → discard (already enclosed by hull)
}
```

[Source: Preparata & Shamos "Computational Geometry: An Introduction", 1985; GPU hull: Lauterbach et al. 2009 §70.1]

---

### 107. GPU Radix Sort as a Geometry Primitive

*Audience: systems developers.*

Radix sort underlies a surprising fraction of GPU geometry algorithms: Morton code sorting for BVH construction (§31), particle bucket assignment (§48), mesh edge/face key sorting for adjacency construction (§18, §106), histogram prefix sums for stream compaction (§26, §72), and order-independent transparency (§85). This section presents the canonical 4-pass GPU radix sort (Merrill & Grimshaw 2010 / CUB/Thrust pattern) in Vulkan compute, applicable as a geometry primitive throughout the chapter.

### 107.1 Histogram Pass: Count Digit Frequencies

For each 2-bit digit position, count the frequency of each of the 4 digit values within each thread block:

```glsl
// radix_histogram.comp — count 2-bit digit frequencies per workgroup
layout(set=0,binding=0) readonly buffer Keys   { uint keys[];  };
layout(set=0,binding=1) buffer Histograms { uint hist[N_WORKGROUPS][4]; };
layout(push_constant) uniform PC { uint N; uint bitOffset; } pc;
layout(local_size_x=256) in;
shared uint sharedHist[4];
void main() {
    if (gl_LocalInvocationID.x < 4u) sharedHist[gl_LocalInvocationID.x]=0u;
    barrier();
    uint i  = gl_GlobalInvocationID.x;
    if (i < pc.N) {
        uint digit = (keys[i] >> pc.bitOffset) & 3u;
        atomicAdd(sharedHist[digit], 1u);
    }
    barrier();
    if (gl_LocalInvocationID.x < 4u)
        hist[gl_WorkGroupID.x][gl_LocalInvocationID.x] = sharedHist[gl_LocalInvocationID.x];
}
```

[Source: Merrill & Grimshaw "Revisiting Sorting for GPGPU Stream Architectures", PACT 2010; CUB DeviceRadixSort]

### 107.2 Prefix Sum (Scan) Across Histograms

Exclusive prefix-sum the per-workgroup histograms to produce the global scatter offsets for each digit value:

```glsl
// radix_scan.comp — exclusive prefix sum of histograms across workgroups
layout(set=0,binding=0) readonly buffer Histograms { uint hist[N_WORKGROUPS][4]; };
layout(set=0,binding=1) writeonly buffer Offsets   { uint offsets[N_WORKGROUPS][4]; };
layout(local_size_x=N_WORKGROUPS) in;
shared uint sharedScan[N_WORKGROUPS];
void main() {
    uint digit = gl_WorkGroupID.x;  // one workgroup per digit value
    uint wg    = gl_LocalInvocationID.x;
    sharedScan[wg] = (wg < N_WORKGROUPS) ? hist[wg][digit] : 0u;
    barrier();
    // Hillis-Steele inclusive scan
    for (uint s=1u; s<N_WORKGROUPS; s<<=1) {
        uint v = (wg>=s) ? sharedScan[wg-s] : 0u;
        barrier(); sharedScan[wg] += v; barrier();
    }
    // Convert to exclusive by shifting right
    offsets[wg][digit] = (wg>0) ? sharedScan[wg-1] : 0u;
}
```

[Source: Harris et al. "Parallel Prefix Sum (Scan) with CUDA", GPU Gems 3 2007; Hillis & Steele "Data Parallel Algorithms", CACM 1986]

### 107.3 Scatter Pass: Place Keys at Computed Offsets

Each thread computes its output position from the workgroup-local prefix sum and the global offset, then scatters the key-value pair:

```glsl
// radix_scatter.comp — scatter keys and values to sorted positions
layout(set=0,binding=0) readonly buffer KeysIn   { uint keysIn[];   };
layout(set=0,binding=1) readonly buffer ValsIn   { uint valsIn[];   };
layout(set=0,binding=2) readonly buffer Offsets  { uint offsets[][4]; };
layout(set=0,binding=3) writeonly buffer KeysOut { uint keysOut[];  };
layout(set=0,binding=4) writeonly buffer ValsOut { uint valsOut[];  };
layout(push_constant) uniform PC { uint N; uint bitOffset; } pc;
layout(local_size_x=256) in;
shared uint sharedRanks[256];
void main() {
    uint i     = gl_GlobalInvocationID.x;
    uint li    = gl_LocalInvocationID.x;
    uint wg    = gl_WorkGroupID.x;
    uint digit = (i < pc.N) ? (keysIn[i] >> pc.bitOffset) & 3u : 0u;
    // Compute local rank within workgroup for this digit using shared ballot
    uint rank  = localRank(digit, li);  // count of same digit before li
    sharedRanks[li] = offsets[wg][digit] + rank;
    barrier();
    if (i < pc.N) { keysOut[sharedRanks[li]]=keysIn[i]; valsOut[sharedRanks[li]]=valsIn[i]; }
}
```

[Source: Merrill & Grimshaw 2010; Satish et al. "Designing Efficient Sorting Algorithms for Manycore GPUs", IPDPS 2009]

### 107.4 Full 32-Bit Sort and Application to Geometry

Compose 16 passes of 2-bit radix sort (or 8 passes of 4-bit) to sort 32-bit keys. Apply to geometry: sort triangle indices by Morton code (§31.1) for BVH construction, or sort vertex indices by material ID (§82.3 wavefront path tracing), or sort particle cell IDs for spatial hash (§27):

```glsl
// morton_sort_dispatch.comp — sort primitives by 30-bit Morton code for LBVH
layout(set=0,binding=0) readonly buffer Centers { vec3 cen[]; };
layout(set=0,binding=1) writeonly buffer MortonKeys { uint mortonKey[]; };
layout(set=0,binding=2) writeonly buffer SortVals   { uint triIdx[];    };
layout(push_constant) uniform PC { vec3 sceneMin; float invSceneSize; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint t   = gl_GlobalInvocationID.x;
    if (t >= pc.N) return;
    vec3  n  = (cen[t]-pc.sceneMin)*pc.invSceneSize;  // normalise to [0,1]³
    uvec3 q  = uvec3(n*1023.0);  // 10 bits per axis
    // Interleave bits: Morton code
    uint m = 0u;
    for(int b=0;b<10;b++) m|=((q.x>>b)&1u)<<(3*b)|((q.y>>b)&1u)<<(3*b+1)|((q.z>>b)&1u)<<(3*b+2);
    mortonKey[t]=m; triIdx[t]=t;
}
// Then run §107.1–107.3 for 15 passes (30-bit sort); result is LBVH input order (§31.2)
```

[Source: Lauterbach et al. "Fast BVH Construction on GPUs", EG 2009 §70; Karras "Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees", HPG 2012]

---


---

## 108. Library Landscape

**Coverage gap.** No single open-source library covers more than a narrow slice of the algorithms in this chapter. The table below maps the available libraries against the chapter's topic areas; empty cells represent functionality that must be implemented directly in shader/compute code using the patterns shown in the preceding sections. This fragmentation is the norm in GPU geometry programming: the field is young enough that GPU-native algorithmic libraries have not yet consolidated around a common abstraction layer comparable to what BLAS/LAPACK provide for linear algebra.

The closest approximation to a broad GPU geometry toolkit is **NVIDIA's** ecosystem when CUDA is available: cuSPARSE (sparse solvers for Poisson/LSCM), Thrust (prefix scans, radix sort, stream compaction), and NVCC-compiled versions of CPU libraries. On non-NVIDIA hardware the pattern in this chapter — implementing each algorithm as a self-contained Vulkan compute shader from first principles — remains the only portable path.

Two tables follow — one per group of nine sections — because a single 22-column table is unreadable. — means no coverage in that section.

**Table A — §1–§25 coverage**

| Library | Ver | GPU Backend | §1 Subdiv | §2 NURBS | §3 Implicit | §39 Skeletal | §40 IK | §4 Splines | §10 Mesh | §24 BVH/Coll | §25 SS | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [OpenSubdiv](https://github.com/PixarAnimationStudios/OpenSubdiv) | 3.6.0 | GL compute, CUDA, Metal | ✓ CC/Loop | — | — | — | — | — | — | — | — | Production subdivision |
| [CGAL](https://www.cgal.org) | 6.1 | None (CPU + TBB) | ✓ CC/Loop/DS | — | ✓ MC/DC | — | — | — | ✓ CDT/repair | ✓ AABB tree | — | CPU preprocessing |
| [GeometricTools](https://github.com/davideberly/GeometricTools) | 6.x | None | ✓ CC/DS | ✓ NURBS | partial | — | — | ✓ | — | ✓ BVH | — | CPU offline reference |
| [libigl](https://github.com/libigl/libigl) | 2.5.0 | None (CPU + Eigen) | ✓ | — | ✓ MC/DC/MT | ✓ LBS/DQS | — | — | ✓ UV/Bool | — | — | Skinning weights, UV |
| [OpenCASCADE](https://www.opencascade.com/) | 8.0.0p1 | None (OpenGL vis only) | — | ✓ BRep | — | — | — | — | ✓ tess | — | — | CAD NURBS → VkBuffer |
| [meshoptimizer](https://github.com/zeux/meshoptimizer) | 0.22 | None (CPU → GPU output) | — | — | — | — | — | — | ✓ QEM/cache | — | — | LOD chain, compression |
| [NanoVDB](https://github.com/AcademySoftwareFoundation/openvdb) | OVDBv9 | CUDA / Vulkan SSBO | — | — | ✓ sparse vol | — | — | — | — | — | — | Sparse volume GPU |
| [xatlas](https://github.com/jpcy/xatlas) | 2.x | None | — | — | — | — | — | — | ✓ UV atlas | — | — | UV layout for baking |
| [PoissonRecon](https://github.com/mkazhdan/PoissonRecon) | 13.x | None (CPU MT) | — | — | ✓ recon | — | — | — | — | — | — | Point cloud → mesh |
| [V-HACD](https://github.com/kmammou/v-hacd) | 4.0 | None (CPU) | — | — | — | — | — | — | — | ✓ decomp | — | Convex hull for GJK |
| [Triangle](https://www.cs.cmu.edu/~quake/triangle.html) | 1.6 | None | — | — | — | — | — | — | ✓ CDT | — | — | 2D CDT/quality meshing |

**Table B — §26–§70 coverage**

| Library | Ver | GPU Backend | §26 GPU-Driven | §48 Fluid | §49 Deform | §69 Proc | §34 Geo | §41 Shape | §91 3DGS | §27 Spatial | §35 Spectral | §70 NavMesh | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [meshoptimizer](https://github.com/zeux/meshoptimizer) | 0.22 | None (CPU → GPU output) | ✓ meshlets | — | — | — | — | — | — | — | — | — | Meshlet build + bounds |
| [NanoVDB](https://github.com/AcademySoftwareFoundation/openvdb) | OVDBv9 | CUDA / Vulkan SSBO | — | — | — | — | — | — | — | ✓ SVO | — | — | Sparse voxel GPU traversal |
| [CGAL](https://www.cgal.org) | 6.1 | None (CPU + TBB) | — | — | — | — | partial cotan | — | — | ✓ k-d tree | partial | — | CPU k-d tree, Laplacian |
| [libigl](https://github.com/libigl/libigl) | 2.5.0 | None (CPU + Eigen) | — | — | partial FEM | — | ✓ cotan/geo | ✓ ARAP/MVC | — | — | ✓ spectral | — | Geodesics, deformation, spectral |
| [SPlisHSPlasH](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH) | 2.13 | CUDA | — | ✓ SPH/PBF/FLIP | — | — | — | — | — | — | — | — | GPU fluid simulation |
| [PhysX 5](https://github.com/NVIDIA-Omniverse/PhysX) | 5.4 | CUDA | — | ✓ PBD fluid | ✓ FEM/cloth | — | — | — | — | — | — | — | GPU rigid + soft body + fluid |
| [geometry-central](https://github.com/nmwsharp/geometry-central) | 0.x | None (CPU) | — | — | — | — | ✓ heat/cotan | partial | — | — | ✓ spectral | — | Geodesics, DDG reference |
| [gsplat](https://github.com/nerfstudio-project/gsplat) | 1.x | CUDA / ROCm | — | — | — | — | — | — | ✓ 3DGS raster | — | — | — | GPU Gaussian splatting |
| [FastNoise2](https://github.com/Auburn/FastNoise2) | 0.10 | SIMD / CUDA node graph | — | — | — | ✓ GPU noise | — | — | — | — | — | — | Procedural terrain noise |
| [Recast/Detour](https://github.com/recastnavigation/recastnavigation) | 1.6 | None (CPU) | — | — | — | — | — | — | — | — | — | ✓ (CPU ref) | NavMesh voxelise + pathfind |

**OpenSubdiv** ([source](https://github.com/PixarAnimationStudios/OpenSubdiv)) is the right choice for any production subdivision pipeline. The lack of a Vulkan evaluator requires the custom SSBO stencil approach described in §1.5.

**CGAL** ([source](https://www.cgal.org), [subdivision](https://doc.cgal.org/latest/Subdivision_method_3/index.html)) provides `CGAL::Subdivision_method_3::CatmullClark_subdivision()` and `Loop_subdivision()` as single-call in-place operations on a `Surface_mesh`. It also covers 2D/3D Delaunay triangulation, constrained Delaunay, Ruppert refinement, a CPU k-d tree (`CGAL::Kd_tree<>`), and a partial cotangent Laplacian via `CGAL::Polygon_mesh_processing::compute_vertex_normals()` — the broadest algorithmic span of any library here, though entirely CPU-only:

```cpp
#include <CGAL/subdivision_method_3.h>
namespace Sub = CGAL::Subdivision_method_3;
Sub::CatmullClark_subdivision(mesh,
    CGAL::parameters::number_of_iterations(2));
```

**libigl** ([source](https://github.com/libigl/libigl)) provides `igl::lbs_matrix()` and `igl::dqs()` for skinning, `igl::bbw()` for bounded biharmonic weights, `igl::lscm()` and `igl::harmonic()` for UV parameterization, `igl::marching_tets()` for isosurface extraction, `igl::heat_geodesics()` and `igl::cotmatrix()` for geodesics and the cotangent Laplacian, `igl::arap()` for ARAP deformation, `igl::mvc()` for mean value coordinates, and `igl::eigs()` for spectral geometry. All CPU-side using Eigen; upload results to GPU as vertex buffers. libigl has the widest algorithmic breadth of any portable library in this chapter — it spans §1, §3, §39, §10, §34, §41, and §35 — but has no GPU execution path.

**GeometricTools** ([source](https://github.com/davideberly/GeometricTools), Boost license) provides `GTL::NURBSSurface<float, 3>` and `GTL::CatmullClarkSurface` as header-only CPU implementations. Use for offline asset preprocessing.

**meshoptimizer** ([source](https://github.com/zeux/meshoptimizer), MIT) provides `meshopt_simplify()`, `meshopt_optimizeVertexCache()`, and `meshopt_buildMeshlets()` with `meshopt_computeMeshletBounds()` — the last two are §26 GPU-Driven Rendering directly. Generate the full LOD chain and meshlet data offline, upload to a single `VkBuffer`, and dispatch via the task/mesh shader pipeline.

**NanoVDB** ([source](https://github.com/AcademySoftwareFoundation/openvdb), Apache 2.0, OpenVDB 9.0+) converts OpenVDB trees to flat GPU buffers via `nanovdb::openToNanoVDB()`. The companion GLSL header provides tree-descent accessors matching §27.3 (SVO traversal). Requires `VK_EXT_scalar_block_layout`.

**xatlas** ([source](https://github.com/jpcy/xatlas), MIT) generates UV atlases as a preprocessing step for texture baking (§10.13). CPU-only; outputs UV coordinates and a remapped index buffer ready for GPU upload.

**PoissonRecon** ([source](https://github.com/mkazhdan/PoissonRecon), MIT) is the reference screened Poisson surface reconstruction (§3.12). CPU multi-threaded (TBB); 10–60 s for 1M-point clouds.

**V-HACD 4.0** ([source](https://github.com/kmammou/v-hacd), BSD-3) decomposes non-convex meshes into convex hull approximations for physics (§24.5). CPU-only.

**Triangle** ([source](https://www.cs.cmu.edu/~quake/triangle.html), Shewchuk, public domain) is the reference 2D CDT and Ruppert quality-mesh generator (§3.11). Single-file C library.

**SPlisHSPlasH** ([source](https://github.com/InteractiveComputerGraphics/SPlisHSPlasH), MIT) is the most complete open-source GPU fluid simulation library. It implements SPH, PBF (Macklin 2013), DFSPH (divergence-free SPH), IISPH, PBD, and FLIP variants, all with CUDA backends. Spatial hashing for neighbour search runs on GPU; pressure and density solvers run fully on-device. The library outputs particle positions each step — couple to the SPH surface extraction pipeline from §3.8 to produce renderable meshes in real time.

**PhysX 5** ([source](https://github.com/NVIDIA-Omniverse/PhysX), BSD-3) adds GPU-native FEM soft bodies (corotational linear elasticity on tet meshes, §49.2) and GPU cloth (Position-Based Dynamics, matching §39.6 and §49.1) alongside GPU rigid body simulation and PBD fluid. The soft body solver runs entirely on CUDA; results are available as CUDA device pointers that map directly to Vulkan external memory via `VK_KHR_external_memory` + `VK_EXT_external_memory_host`. No Vulkan-native API; requires interop.

**geometry-central** ([source](https://github.com/nmwsharp/geometry-central), MIT) is the research reference implementation for discrete differential geometry: `HeatMethodDistance` (§34.2), cotangent Laplacian assembly, `StripePatterns` for seamless parameterization, vector heat method, and spectral basis computation (§35.1). CPU-only; results upload to GPU. Essential for verifying custom Vulkan compute DDG implementations against a known-correct baseline.

**gsplat** ([source](https://github.com/nerfstudio-project/gsplat), Apache 2.0) is a production-quality GPU 3DGS rasterizer supporting CUDA and ROCm backends. It implements the full pipeline from §91 — Gaussian projection, tile assignment, radix sort, and per-tile alpha compositing — with forward and backward passes for training. For Vulkan integration, render to a CUDA surface then copy via external memory, or port the tile rasterizer kernels to Vulkan compute using the same algorithm (§91.2).

**FastNoise2** ([source](https://github.com/Auburn/FastNoise2), MIT) provides a SIMD node-graph noise system supporting Perlin, OpenSimplex2, Cellular, Domain Warp, and fractal combinations (§69.1 terrain fBm). A CUDA backend is available for GPU-side evaluation. On CPU, AVX2/AVX-512 SIMD makes it fast enough for real-time heightmap generation.

**Recast/Detour** ([source](https://github.com/recastnavigation/recastnavigation), Zlib) is the canonical navmesh library used by most game engines. Recast's voxelisation and region-labelling steps (§70.1–19.2) are CPU-only; no GPU port exists. The flow-field construction in §70.3 is implemented directly in compute shaders without a library equivalent. Recast is the correct preprocessing tool for static navmesh geometry; Detour handles the runtime pathfinding query API.

**Table C — §75–§96 coverage**

| Library | Ver | GPU Backend | §75 RT Geom | §87 Point Cloud | §50 Rigid Body | §51 Crowd | §11 Remesh | §52 Fracture | §71 Terrain | §92 Geo DL | §5 2D Vector | §96 Sci Vis | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [Vulkan RT Extensions](https://www.khronos.org/registry/vulkan/) | 1.3 | Vulkan | ✓ BLAS/TLAS | — | — | — | — | — | — | — | — | — | Ray tracing AS build |
| [PCL](https://pointclouds.org) | 1.14 | None (CPU) | — | ✓ k-NN/ICP | — | — | — | — | — | — | — | ✓ vis | Point cloud CPU pipeline |
| [Open3D](https://www.open3d.org) | 0.18 | CUDA (partial) | — | ✓ k-NN/ICP/RANSAC | — | — | — | — | — | ✓ PointNet | — | ✓ vis | GPU point cloud + DL |
| [Bullet Physics](https://github.com/bulletphysics/bullet3) | 3.25 | OpenCL (btGpu) | — | — | ✓ SAP/SI/ABA | — | — | ✓ Voronoi | — | — | — | — | Physics + fracture |
| [RVO2](https://gamma.cs.unc.edu/RVO2/) | 2.0 | None (CPU MT) | — | — | — | ✓ ORCA | — | — | — | — | — | — | ORCA reference impl |
| [OpenMesh](https://www.graphics.rwth-aachen.de/OpenMesh/) | 10.0 | None (CPU) | — | — | — | — | ✓ isotropic | — | — | — | — | — | Remeshing, halfedge ops |
| [msdfgen](https://github.com/Chlumsky/msdfgen) | 1.12 | None (CPU) | — | — | — | — | — | — | — | — | ✓ MSDF atlas | — | Font atlas generation |
| [PyTorch Geometric](https://pyg.org) | 2.5 | CUDA / ROCm | — | ✓ k-NN | — | — | — | — | — | ✓ GNN/PointNet | — | — | Geometric deep learning |
| [VTK](https://vtk.org) | 9.3 | OpenGL compute | — | — | — | — | — | — | — | — | — | ✓ streams/glyphs | Scientific vis |
| [Recast/Detour](https://github.com/recastnavigation/recastnavigation) | 1.6 | None (CPU) | — | — | — | — | — | — | ✓ terrain voxel | — | — | — | Terrain navmesh preprocessing |

**Vulkan RT Extensions** (KHR, Vulkan 1.2+) are the only "library" for §75 — AS build/update APIs are in the driver. VulkanMemoryAllocator ([source](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)) simplifies the scratch/result buffer lifecycle for BLAS/TLAS operations.

**PCL** ([source](https://pointclouds.org), BSD-3) provides `pcl::KdTreeFLANN<pcl::PointXYZ>` for CPU k-NN, `pcl::IterativeClosestPoint<>` for ICP, `pcl::SACSegmentation<>` for RANSAC plane/sphere fitting, and `pcl::visualization::PCLVisualizer` for scientific vis. No GPU backend; use for preprocessing before uploading point clouds to Vulkan SSBOs.

**Open3D** ([source](https://www.open3d.org), MIT) adds a CUDA k-NN search (`open3d.core.nns.NearestNeighborSearch`), GPU ICP variants (point-to-plane, colored ICP), RANSAC registration, and a PointNet++ training pipeline via PyTorch. The `open3d.visualization.Visualizer` covers §96 scientific vis. Open3D is the most capable GPU point cloud toolkit outside CUDA-only paths.

**Bullet 3** ([source](https://github.com/bulletphysics/bullet3), Zlib) implements SAP broadphase, sequential impulse (Catto solver), and Featherstone ABA for articulated bodies (§50). A partial OpenCL backend (`btGpuBroadphaseProxy`) parallelises the broadphase; the narrow-phase and solver remain on CPU. The `HACD` component provides Voronoi-style approximate convex decomposition for pre-fracture (§52.1).

**RVO2** ([source](https://gamma.cs.unc.edu/RVO2/), Apache 2.0) is the reference CPU implementation of ORCA (van den Berg 2011) for crowd simulation (§51.1). For GPU port use the compute shader pattern from §51.1 with the §27.2 spatial hash for neighbourhood queries.

**OpenMesh** ([source](https://www.graphics.rwth-aachen.de/OpenMesh/), LGPL) provides a halfedge-based mesh data structure with `OpenMesh::Subdivider::Uniform::CatmullClarkT<>` and edge split/collapse/flip operations for isotropic remeshing (§11.1). CPU-only; use for offline remesh passes before GPU upload.

**msdfgen** ([source](https://github.com/Chlumsky/msdfgen), MIT) generates multi-channel SDF atlases from font outlines or SVG paths — the offline preprocessing step for §5.2 MSDF font rendering. Produces a `VkImage` data asset; no runtime GPU component.

**PyTorch Geometric** ([source](https://pyg.org), MIT) provides `torch_geometric.nn.PointNetConv`, graph convolution layers, and batched k-NN search (`torch_cluster.knn_graph`) all running on CUDA or ROCm. The trained model weights (§92) can be extracted and run as a Vulkan compute shader MLP inference pass.

**VTK** ([source](https://vtk.org), BSD-3) covers §96 scientific visualization: `vtkStreamTracer` for streamlines, `vtkVolume` with GPU ray casting, `vtkTensorGlyph` for superquadrics, and `vtkMarchingCubes` for isosurface extraction. The OpenGL compute backend (`vtkOpenGLGPUVolumeRayCastMapper`) uses GL compute shaders; Vulkan integration requires the VTK-m ([source](https://m.vtk.org)) parallel framework with a Kokkos backend.

**Table D — §42–§29 coverage**

| Library | Ver | GPU Backend | §42 Hair | §61 LSM | §93 DiffRender | §72 Particles | §28 Lightmap | §12 UV Pack | §62 Swept Vol | §36 Morphing | §97 Decals | §29 Occlusion | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [TressFX](https://github.com/GPUOpen-Effects/TressFX) | 4.1 | Vulkan / DX12 | ✓ sim+render | — | — | — | — | — | — | — | — | — | GPU hair simulation |
| [NvCloth](https://github.com/NVIDIAGameWorks/NvCloth) | 1.1 | CUDA | partial strand | — | — | — | — | — | — | — | — | — | Cloth+strand PBD |
| [NvDiffRast](https://github.com/NVIDIAGameWorks/nvdiffrast) | 0.3 | CUDA | — | — | ✓ diff rast | — | — | — | — | — | — | — | Differentiable rasterisation |
| [Pytorch3D](https://github.com/facebookresearch/pytorch3d) | 0.7 | CUDA | — | — | ✓ diff render | — | — | — | — | ✓ functional maps | — | — | Differentiable rendering + shape |
| [xatlas](https://github.com/jpcy/xatlas) | 2.x | None (CPU) | — | — | — | — | — | ✓ atlas pack | — | — | — | — | UV atlas generation |
| [Intel OSR](https://github.com/GameTechDev/OcclusionCulling) | ISPC | CPU SIMD | — | — | — | — | — | — | — | — | — | ✓ SW occlusion | Software occlusion rasteriser |
| [Embree](https://www.embree.org) | 4.3 | AVX-512 / CPU | — | — | ✓ ray query | — | ✓ AO bake | — | — | ✓ correspondence | — | — | CPU ray tracing for bake |
| [CGAL](https://www.cgal.org) | 6.1 | None | — | — | — | — | — | ✓ seam (CPU) | ✓ Minkowski | ✓ functional maps (CPU) | — | — | Geometry preprocessing |
| [Radeon Rays](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK) | 4.1 | Vulkan / HIP | — | — | partial | — | ✓ AO GPU | — | — | — | — | — | GPU ray casting for bake |
| [LightBaker](https://github.com/ands/lightmapper) | 1.x | OpenGL | — | — | — | — | ✓ hemicube | — | — | — | — | — | CPU/GPU lightmap bake |

**TressFX 4.1** ([source](https://github.com/GPUOpen-Effects/TressFX), MIT) is the most complete open-source GPU hair library, implementing the full §42 pipeline: position-based strand simulation (stretch, bending, torsion constraints), strand self-collision via spatial hashing, collision with scene geometry, and LOD-controlled strand-to-shell transitions. Vulkan and DX12 backends. Integrates with the engine's TLAS via `VK_KHR_ray_tracing_pipeline` for strand shadow rays.

**NvDiffRast** ([source](https://github.com/NVIDIAGameWorks/nvdiffrast), Apache 2.0) provides CUDA-accelerated differentiable rasterisation (§93.1): forward rasteriser with anti-aliased edge gradients, texture sample backward pass, and interpolation gradients. Includes a `torch.autograd.Function` wrapper. For Vulkan integration, run the forward pass in nvdiffrast (CUDA) and extract gradients into a Vulkan-mapped buffer via `VK_KHR_external_memory`.

**PyTorch3D** ([source](https://github.com/facebookresearch/pytorch3d), BSD) extends §92 (geometric deep learning) with differentiable mesh rendering (§93), functional map utilities (§36), and point cloud operations. The `pytorch3d.renderer.MeshRasterizer` implements the soft rasterisation from §93.1.

**Embree 4** ([source](https://www.embree.org), Apache 2.0) provides CPU ray tracing via AVX-512/AVX2 SIMD — the standard reference backend for offline lightmap baking (§28). `rtcOccluded1M()` batches occlusion queries for AO hemisphere sampling. Pair with a GPU denoiser (OIDN) for high-quality baked lighting.

**Radeon Rays 4** ([source](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK), MIT) is the GPU counterpart: Vulkan and HIP ray query acceleration for AO baking (§28) and differentiable rendering (§93) on AMD hardware. Complements the `VK_KHR_ray_query` inline queries in §28.2 with a higher-level batch API.

**Table E — §73–§53 coverage**

| Library | Ver | GPU Backend | §73 Ocean | §13 Voxelise | §63 TSDF | §76 Shadows | §77 IBL | §43 SecAnim | §14 Morph | §105 Sampling | §15 ProgMesh | §53 Rope | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [VkFFT](https://github.com/DTolm/VkFFT) | 1.3 | Vulkan / CUDA / HIP | ✓ IFFT core | — | — | — | — | — | — | — | — | — | GPU FFT for ocean |
| [Crest Ocean](https://github.com/wave-harmonic/crest) | 4.x | Unity (Vulkan) | ✓ full pipeline | — | — | — | — | — | — | — | — | — | Production ocean system |
| [Open3D](https://www.open3d.org) | 0.18 | CUDA | — | — | ✓ KinFu | — | — | — | — | — | — | — | TSDF fusion (KinectFusion) |
| [OpenVDB](https://www.openvdb.org) | 11.0 | None (CPU + TBB) | — | ✓ mesh→VDB | — | — | — | — | ✓ morphology | — | — | — | Mesh voxelisation + morphology |
| [CGAL](https://www.cgal.org) | 6.1 | None | — | ✓ conservative | — | — | — | — | ✓ morph (CPU) | ✓ Poisson (CPU) | — | — | Geometry preprocessing |
| [Falcor](https://github.com/NVIDIAGameWorks/Falcor) | 7.0 | Vulkan / DX12 | — | — | — | ✓ CSM/RT shadow | ✓ PMREM/LUT | — | — | ✓ Halton/Sobol | — | — | Research renderer, IBL + shadows |
| [Filament](https://github.com/google/filament) | 1.x | Vulkan / Metal | — | — | — | ✓ CSM/DPCF | ✓ IBL bake | — | — | — | — | — | Production PBR + IBL |
| [igl](https://github.com/libigl/libigl) | 2.5 | None (CPU) | — | — | — | — | — | ✓ corrective shapes | ✓ medial axis | ✓ Poisson (CPU) | ✓ progressive | — | Shape analysis, progressive mesh |
| [Discrete Elastic Rods](https://github.com/bastibl/der) | research | None (CPU) | — | — | — | — | — | — | — | — | — | ✓ Cosserat sim | Cosserat rod reference impl |
| [OIIO](https://github.com/AcademySoftwareFoundation/OpenImageIO) | 2.5 | None (CPU) | — | — | — | — | ✓ env filter | — | — | — | — | — | HDR env map processing |

**VkFFT** ([source](https://github.com/DTolm/VkFFT), MIT) is the only portable GPU FFT with a Vulkan backend. It generates optimal SPIR-V at runtime for the target device's subgroup size and supports batch 2D FFTs — the exact operation §73.2 requires for the Tessendorf ocean IFFT.

**Crest Ocean** ([source](https://github.com/wave-harmonic/crest), MIT) is a production GPU ocean system implementing the full §73 pipeline: Phillips spectrum initialisation, GPU FFT via Unity's compute, choppy wave Jacobian foam, Gerstner wave blending, and LOD-aware shoreline interaction. The Unity Vulkan backend means its shader source maps directly to the GLSL compute patterns in §73.

**Open3D** covers §63 (TSDF fusion) with its `open3d.pipelines.integration.ScalableTSDFVolume` (voxel hashing, §63.4) and `open3d.pipelines.odometry` (ICP tracking, §63.2). The CUDA backend fuses 512³ volumes at 30 fps on modern hardware.

**OpenVDB** ([source](https://www.openvdb.org), MPL-2.0) provides `openvdb::tools::meshToVolume()` for conservative mesh voxelisation (§13.1) and `openvdb::tools::morphology::dilateVoxels()` / `erodeVoxels()` for morphological operations (§14.1). No GPU execution path; NanoVDB (Table A) handles the GPU-side sparse volume traversal.

**Falcor** ([source](https://github.com/NVIDIAGameWorks/Falcor), BSD-3) is NVIDIA's research renderer with Vulkan/DX12 backends. It implements cascaded shadow maps (§76.1), ray-traced shadows via `VK_KHR_ray_query` (§76.4), PMREM prefiltering (§77.2), BRDF LUT baking (§77.3), and low-discrepancy samplers (Halton, Sobol — §105.1). Its shader source is the best available reference for §76–§77 on Vulkan.

**Filament** ([source](https://github.com/google/filament), Apache 2.0) is Google's production PBR renderer with a Vulkan backend. It implements DPCF (Distance-field Percentage Closer Filtering) soft shadows, a custom cascaded shadow system (§76.1), IBL prefiltering (§77.2–44.3), and the split-sum approximation. The Filament material compiler generates GLSL/SPIR-V from a material definition language — a useful reference for §77 IBL integration in Vulkan.

**libigl** (Table A/B) also covers §15: `igl::decimate()` generates the edge-collapse sequence, and `igl::qslim()` records vertex split data in Hoppe-compatible format. The progressive mesh implementation is CPU-side; upload the pre-computed split stream as an SSBO for GPU playback (§15.2).

**Table F — §54–§55 coverage**

| Library | Ver | GPU Backend | §54 Cloth | §78 GI | §64 SDF-Col | §98 ProcTex | §99 Acoustic | §16 Compress | §79 Caustics | §6 NURBS | §44 Retarget | §55 Fluid-Solid | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [XPBD (Matthias Müller)](https://matthias.game-tech.ch/research/small-steps-in-physics-simulation/) | research | CPU/CUDA | ✓ cloth XPBD | — | — | — | — | — | — | — | — | — | XPBD cloth reference |
| [Nvidia Flex / PhysX 5](https://developer.nvidia.com/flex) | 5.x | CUDA | ✓ cloth | — | ✓ SDF contacts | — | — | — | — | — | ✓ coupling | CUDA cloth + fluid coupling |
| [meshoptimizer](https://github.com/zeux/meshoptimizer) | 0.21 | None (CPU) | — | — | — | — | — | ✓ quantize/delta | — | — | — | — | Mesh compression pre-pass |
| [Draco](https://github.com/google/draco) | 1.5 | None (CPU) | — | — | — | — | — | ✓ edgebreaker | — | — | — | — | Connectivity + pos compression |
| [Basis Universal](https://github.com/BinomialLLC/basis_universal) | 1.16 | GPU transcode | — | — | — | ✓ geom textures | — | ✓ BC7/ETC2 | — | — | — | — | GPU texture codec |
| [Steam Audio](https://valvesoftware.github.io/steam-audio/) | 4.5 | CPU/AVX2 | — | — | — | — | ✓ acoustic RT | — | — | — | — | — | GPU occlusion + acoustic RT |
| [OpenCASCADE](https://www.opencascade.com) | 7.8 | None (CPU) | — | — | — | — | — | — | — | ✓ NURBS exact | — | — | CAD kernel, NURBS trim |
| [SVGF (Schied ref)](https://github.com/tiechui94/SVGF) | research | Vulkan | — | ✓ denoising | — | — | — | — | ✓ photon denoise | — | — | — | GI spatiotemporal denoising |
| [Cem Yuksel Yarn](https://www.cemyuksel.com/research/yarnrendering/) | research | CUDA | ✓ orthotropy | — | — | — | — | — | — | — | — | — | Woven fabric orthotropy |
| [Mixamo / Rokoko](https://www.rokoko.com) | SaaS | Cloud | — | — | — | — | — | — | — | — | ✓ retarget | — | Animation retargeting service |

**XPBD cloth** (Müller et al. 2020 "Small Steps in Physics Simulation") is the reference implementation for §54. Orthotropic constraints (§54.1), self-collision (§54.2), and tearing (§54.4) follow the patterns in the paper. No Vulkan port exists; the compute shader implementations in §54 are the primary Vulkan-native path.

**NVIDIA Flex / PhysX 5** (CUDA backend) covers §54 (cloth via particle position constraints and shape matching), §64.1 (SDF contact generation), and §55.4 (two-way fluid-solid coupling via SPH). It is CUDA-only; the Vulkan-native equivalents are the custom compute pipelines in each section.

**meshoptimizer** (§16.1, §16.4) provides `meshopt_quantizePosition()` for geometry quantization and the meshlet delta encoding scheme. The CPU decode runs before GPU upload; §16.1 shows the GPU dequantisation compute pass for streaming scenarios.

**Draco** (§16.2) implements edgebreaker connectivity compression and parallelogram position prediction. The GPU-side prediction pass (§16.2 listing) accelerates the final attribute decode for large meshes.

**Basis Universal** (§16.3) transcodes to ETC2/BC7 on CPU or GPU. The GPU transcoding compute shader pattern in §16.3 maps to `basisu_gpu.comp` in the upstream codebase.

**Steam Audio** (§99) provides GPU-accelerated acoustic ray tracing via CPU path tracing with AVX2 vectorisation and optional GPU occlusion queries. The Vulkan ray query approach in §99.3 is the Vulkan-native complement; Steam Audio is the production-quality reference.

**OpenCASCADE** (§6) is the reference CAD kernel implementing exact NURBS evaluation, trimming via boundary representation, and offset curve approximation. The CPU-side evaluation in §6.1-57.3 uses OCC geometry; the GPU Newton iteration in §6.1 replaces the OCC evaluator for display.

**Table G — §94–§56 coverage**

| Library | Ver | GPU Backend | §94 3DGS | §65 Isosurface | §66 AO | §7 CSG | §67 VolRender | §37 Geodesic | §17 UV Param | §18 Delaunay | §30 SVDAG | §56 DEM | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [gaussian-splatting](https://github.com/graphdeco-inria/gaussian-splatting) | research | CUDA | ✓ full pipeline | — | — | — | — | — | — | — | — | — | Reference 3DGS training+render |
| [gsplat](https://github.com/nerfstudio-project/gsplat) | 1.x | CUDA / Vulkan | ✓ optimized | — | — | — | — | — | — | — | — | — | Production 3DGS rasterizer |
| [OpenVDB/NanoVDB](https://www.openvdb.org) | 11.0 | CUDA/Vulkan | — | ✓ MC on VDB | — | — | ✓ DDA traversal | — | — | — | — | — | Volumetric rendering + MC |
| [XeGTAO](https://github.com/GameTechDev/XeGTAO) | main | DX12/Vulkan | — | — | ✓ GTAO | — | — | — | — | — | — | — | Ground-truth AO reference |
| [geometry-central](https://github.com/nmwsharp/geometry-central) | main | None (CPU) | — | — | — | — | — | ✓ heat method | ✓ LSCM/ARAP | — | — | — | Geodesic distance + UV |
| [xatlas](https://github.com/jpcy/xatlas) | main | None (CPU) | — | — | — | — | — | — | ✓ atlas pack | — | — | — | UV atlas generation |
| [Cork CSG](https://github.com/gilbo/cork) | main | None (CPU) | — | — | — | ✓ winding # | — | — | — | — | — | — | Boolean mesh operations |
| [Frostbite Atmosphere](https://github.com/sebh/UnrealEngineSkyAtmosphere) | research | Vulkan/DX12 | — | — | — | — | ✓ atmospheric LUT | — | — | — | — | — | Physically based sky |
| [DAGCompression](https://github.com/Phyronnaz/DAGCompression) | research | CUDA | — | — | — | — | — | — | — | — | ✓ SVDAG build | — | SVO→DAG compression |
| [LIGGGHTS/YADE](https://yade-dem.org) | 3.x | None (CPU+MPI) | — | — | — | — | — | — | — | — | — | ✓ DEM sim | Reference DEM solver |

**gaussian-splatting** (Kerbl et al. 2023, MIT) is the original reference implementation of §94 in CUDA PyTorch. The **gsplat** library ([nerfstudio-project](https://github.com/nerfstudio-project/gsplat)) is a faster, more maintainable CUDA rasterizer with an emerging Vulkan compute backend, implementing the tile-based radix-sort + compositing pipeline of §94.2 at production performance.

**NanoVDB** covers §65.4 (adaptive isosurface extraction from sparse volumes) and §67.4 (DDA traversal for volumetric ray marching). Its GLSL bindings allow direct use in Vulkan compute shaders without CUDA. OpenVDB's `tools::VolumeToMesh` implements Dual Contouring (§65.2) on the CPU; GPU marching cubes on OpenVDB nodes uses the §65.1 pattern applied to NanoVDB leaf data.

**XeGTAO** ([Intel GameTechDev](https://github.com/GameTechDev/XeGTAO), MIT) is the reference implementation of §66.3 GTAO with spatial denoising, a HLSL/DX12 implementation straightforwardly portable to GLSL/Vulkan. It includes bent-normal computation and temporal accumulation.

**geometry-central** ([nmwsharp](https://github.com/nmwsharp/geometry-central), MIT) implements the heat method (§37) and the vector heat method (§37.4) with cotangent Laplacian assembly and Cholesky factorization (via Eigen). The linear system structure is GPU-amenable via the Jacobi-CG patterns in §37.1–65.3; geometry-central provides the reference for the operator assembly step. It also implements LSCM and ARAP parameterization (§17.1–66.2).

**xatlas** (§17.4, MIT) is the most widely deployed GPU-rendering UV atlas library (used in Unity, Blender, many game studios). It generates seam cuts (§17.3 spanning tree approach) and packs charts into a rectangle atlas. CPU-only; the GPU upload of the resulting UV buffer is the standard integration path.

**Table H — §31–§57 coverage**

| Library | Ver | GPU Backend | §31 BVH Build | §32 SDF Bake | §19 Fairing | §95 Neural Geom | §8 MetaBalls | §9 Poisson | §100 VR/XR | §38 Spectral | §88 SfM/MVS | §57 Erosion | Best Use |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [RadeonRays 4](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK) | 4.x | Vulkan/HIP | ✓ LBVH+PLOC | — | — | — | — | — | — | — | — | — | GPU BVH build on AMD/Vulkan |
| [Embree 4](https://github.com/embree/embree) | 4.x | ISPC/SYCL | ✓ SAH-HLBVH | — | — | — | — | — | — | — | — | — | High-quality CPU/SYCL BVH |
| [instant-ngp](https://github.com/NVlabs/instant-ngp) | main | CUDA | — | — | — | ✓ NGP hash-grid | — | — | — | — | — | — | Real-time NeRF training+inference |
| [nerfstudio](https://github.com/nerfstudio-project/nerfstudio) | 1.x | CUDA | — | — | — | ✓ NeRF/Gaussian | — | — | — | — | — | — | NeRF/3DGS training framework |
| [PoissonRecon](https://github.com/mkazhdan/PoissonRecon) | 15.x | None (CPU+TBB) | — | — | — | — | — | ✓ screened Poisson | — | — | — | — | Watertight surface from points |
| [geometry-central](https://github.com/nmwsharp/geometry-central) | main | None (CPU) | — | — | ✓ Taubin/implicit | — | — | — | — | ✓ HKS/WKS | — | — | Mesh fairing + spectral descriptors |
| [OpenVR/OpenXR SDK](https://github.com/KhronosGroup/OpenXR-SDK) | 1.1 | Vulkan | — | — | — | — | — | — | ✓ ATW/ASW | — | — | — | VR reprojection API |
| [COLMAP](https://github.com/colmap/colmap) | 3.x | CUDA | — | — | — | — | — | — | — | — | ✓ SfM+MVS | — | Reference SfM pipeline |
| [OpenMVS](https://github.com/cdcseacave/openMVS) | 2.x | CUDA | — | — | — | — | — | — | — | — | ✓ MVS dense | — | Dense multi-view stereo |
| [Terrain erosion (GPU)](https://github.com/bshishov/UnityTerrainErosionGPU) | research | Vulkan/Unity | — | — | — | — | — | — | — | — | — | ✓ hydraulic+thermal | GPU erosion reference impl |

**RadeonRays 4** ([GPUOpen](https://github.com/GPUOpen-LibrariesAndSDKs/RadeonRays_SDK), MIT) implements LBVH and PLOC BVH construction (§31.1–70.4) with a Vulkan backend — the only portable Vulkan-native BVH builder available as a library. Its construction pipeline matches the Morton-code + radix-sort + refit pattern of §31 exactly; it exports `VkAccelerationStructure`-compatible BVH data for use with `VK_KHR_ray_tracing_pipeline`.

**Embree 4** (Intel, Apache 2.0) implements SAH-HLBVH and PLOC with an ISPC CPU backend and an emerging SYCL (oneAPI) GPU backend. On x86 CPUs it is the quality reference; the SYCL path targets Intel Arc GPUs and provides a template for §31.4 PLOC on non-AMD Vulkan hardware.

**instant-ngp** (§95.2, NVIDIA, custom license) implements the multiresolution hash grid encoding and tiny-MLP inference in CUDA. Its Vulkan interop path (for display) uses CUDA–Vulkan semaphore synchronisation; the hash grid structure in §95.2 is taken directly from its CUDA kernel. **nerfstudio** provides a training framework around gsplat (§94), instant-ngp, and NeuS (§95.4) with a Python API for inference export.

**PoissonRecon** (Kazhdan, MIT) is the reference implementation of §9 with adaptive octree refinement, screened Poisson solve, and isovalue selection. CPU+TBB; the GPU patterns in §9.1–75.4 show how to accelerate the splatting, Laplacian assembly, and isovalue computation passes.

**COLMAP** (§88, BSD-3) implements the full SfM pipeline (feature detection, matching, geometric verification, bundle adjustment) with CUDA-accelerated feature extraction and matching (§88.1). **OpenMVS** extends it with GPU-accelerated SGM depth estimation (§88.3) and depth map fusion (§88.4) on CUDA.

**What is missing.** Spanning all eight tables, the most significant gaps on Vulkan are: GPU BVH construction (covered by RadeonRays on Vulkan; Embree on SYCL), SDF baking (§32 — no Vulkan library; jump-flooding compute patterns are the only path), mesh fairing (§19 — CPU-only in geometry-central), and terrain erosion (§57 — Unity/CUDA reference only). Neural geometry (§95) is CUDA-exclusive. Spectral processing (§38) has no GPU library. The CUDA ecosystem now covers approximately thirty-five of the seventy-nine sections; the Vulkan compute shader patterns across §26–§57 remain the primary portable path for the remainder.

**Table I — §58–§22 coverage**

| Library | Ver | GPU Backend | §58 Fluid Surf | §59 CCD | §106 ConvHull | §20 Remesh | §80 SSR | §101 Silhouette | §89 Descriptors | §21 Segment | §81 Radiosity | §22 Repair | Best Use |
|---------|-----|-------------|:--------------:|:-------:|:------------:|:----------:|:-------:|:--------------:|:---------------:|:-----------:|:-------------:|:----------:|----------|
| SplishSplash | 2.13 | CUDA | ✓ (anisotropic §58.2) | — | — | — | — | — | — | — | — | — | SPH/FLIP fluid simulation |
| bullet3 | 3.25 | CUDA/OpenCL | — | ✓ (§59.1 conservative) | ✓ (GJK-based) | — | — | — | — | — | — | — | Rigid body + CCD |
| Open3D | 0.18 | CUDA/Tensor | — | — | ✓ (GPU QuickHull §106) | ✓ (CVT §20.4) | — | — | ✓ (FPFH §89.2) | — | — | ✓ (§22.1) | Point cloud processing |
| PCL | 1.14 | CUDA (partial) | — | — | — | — | — | — | ✓ (FPFH/SHOT §89) | ✓ (k-means §21.2) | — | — | Robotics point cloud |
| meshoptimizer | 0.21 | CPU | — | — | — | ✓ (edge ops §20.2) | — | — | — | — | — | ✓ (§22.4) | Mesh optimisation/repair |
| libigl | 2.5 | CPU+OpenGL | — | — | — | ✓ (ARAP §17) | — | ✓ (§101.3) | — | ✓ (rw §21.3) | — | ✓ (§22.3) | Geometry processing research |
| geometry-central | 1.0 | CPU | — | — | — | ✓ (flip §20.1) | — | ✓ (§101.4) | — | — | — | — | Differential geometry |
| Radeon Rays | 4.0 | Vulkan/HIP | — | — | — | — | — | — | — | — | — | — | Ray query (used by §80.1 HiZ) |
| drjit / Mitsuba 3 | 0.4 | CUDA/LLVM | ✓ (§58.3 depth smooth) | — | — | — | — | — | — | — | ✓ (§81.2 radiosiy) | — | Differentiable rendering |
| NVIDIA VKRay | SDK | Vulkan | — | — | — | — | ✓ (§80.1 HiZ) | — | — | — | — | — | RTX ray tracing SDK |
| Manifold | 2.4 | CPU | — | — | — | — | — | — | — | — | — | ✓ (watertight CSG §22) | Robust mesh operations |

**SplishSplash** (Bender et al., MIT) implements DFSPH and PCISPH with CUDA kernels for neighbour search, pressure solve, and anisotropic kernel computation (§58.2). Its surface reconstruction pipeline calls the GPU marching cubes of §65 on a density grid built with the Yu & Turk anisotropic splatting kernel.

**Open3D** (Intel/NVIDIA, MIT) provides a GPU tensor API covering point cloud k-NN, GPU convex hull (§106), CVT-based resampling (§20.4), FPFH descriptor computation (§89.2), and mesh repair (§22.1 non-manifold detection). Its CUDA backend is the most complete open GPU geometry library for the §58–§22 range.

**Manifold** (Knyszewski, Apache 2.0) implements exact, robust Boolean operations on manifold meshes and produces watertight output guaranteed to be manifold. It underpins the §22 repair discussion and is the recommended library for 3D printing pipeline repair.

**What is missing in §58–§22.** GPU CCD (§59) has no Vulkan library; bullet3's CCD is CPU-based with CUDA broadphase only. GPU radiosity (§81) has no modern GPU library; all production engines compute it offline on the CPU or bake it with ray tracing (§28). Screen-space reflections (§80) are universally engine-internal (Unreal TSR, Unity URP). Silhouette detection (§101) and suggestive contours (§101.4) have no GPU library. Spanning all nine tables (§1–§22), the Vulkan compute shader patterns remain the primary portable path for the majority of sections not served by CUDA-exclusive libraries.

**Table J — §33–§104 coverage**

| Library | Ver | GPU Backend | §33 VirtGeom | §82 PathTrace | §90 ICP | §102 Displace | §68 LevelSet | §103 SDF Font | §83 Atmo | §84 SSS | §74 Geo | §104 MolSurf | Best Use |
|---------|-----|-------------|:------------:|:-------------:|:-------:|:------------:|:------------:|:------------:|:--------:|:-------:|:-------:|:-----------:|----------|
| Nanite (Unreal 5) | UE5.4 | Vulkan/D3D12 | ✓ (§33 native) | — | — | ✓ (§102 mesh shader) | — | — | ✓ (§83 plugin) | — | — | — | Production virtual geometry |
| NVIDIA Falcor | 7.0 | Vulkan/D3D12 | — | ✓ (§82 full PT) | — | — | — | — | — | — | — | — | Research path tracing framework |
| Mitsuba 3 | 3.5 | CUDA/LLVM | — | ✓ (§82 MIS PT) | — | — | — | — | ✓ (§83 spectral) | ✓ (§84.3) | — | — | Differentiable physically-based |
| Open3D | 0.18 | CUDA/Tensor | — | — | ✓ (§90 GPU ICP) | — | — | — | — | — | ✓ (§74.2 LIDAR) | — | Point cloud & terrain processing |
| OpenVDB | 11.0 | CUDA/NanoVDB | — | — | — | — | ✓ (§68 sparse LS) | — | — | — | — | — | Sparse volumetric computation |
| msdfgen | 1.12 | CPU | — | — | — | — | — | ✓ (§103.3 MSDF) | — | — | — | — | Multi-channel SDF glyph baking |
| SkyAtmosphere (UE) | UE5.4 | Vulkan | — | — | — | — | — | — | ✓ (§83 LUT) | — | — | — | Production sky rendering |
| SSSS (Jiménez) | 2.0 | D3D11 (port) | — | — | — | — | — | — | — | ✓ (§84.1) | — | — | Separable screen-space skin SSS |
| PDAL | 2.7 | CPU | — | — | — | — | — | — | — | — | ✓ (§74 pipeline) | — | LIDAR/geospatial pipeline |
| APBS | 3.4 | CUDA | — | — | — | — | — | — | — | — | — | ✓ (§104.4 PB) | Molecular electrostatics solver |
| VMD | 1.9 | CUDA/OpenCL | — | — | — | — | — | — | — | — | — | ✓ (§104 SAS/SES) | Molecular visualization |

**Nanite** (Epic Games, built into Unreal Engine 5, proprietary) implements §33 end-to-end: offline cluster DAG construction (§33.1), GPU persistent-thread traversal (§33.2), software rasterisation via 64-bit atomic depth buffer (§33.3), and visibility buffer shading (§33.4). The Unreal sky atmosphere plugin implements §83 LUT-based scattering. Nanite's displacement (§102) is applied via mesh shader at cluster granularity.

**NVIDIA Falcor** (BSD-3-Clause) is a real-time rendering research framework implementing a full wavefront path tracer (§82): MIS direct lighting, shader sorting (§82.3), and SVGF denoising (§82.4). It uses Vulkan or D3D12 ray tracing pipelines and serves as the reference implementation for §82.

**OpenVDB / NanoVDB** (Academy Software Foundation, MPL-2.0) implements the VDB hierarchical sparse grid — the reference structure for §68 narrow-band level sets. NanoVDB provides a CUDA/Vulkan-compatible read-only view of the VDB tree enabling GPU advection (§68.3) and reinitialization (§68.2) on the GPU.

**What is missing in §33–§104.** GPU path tracing (§82) is dominated by CUDA/OptiX (Falcor, LuxCoreRender); Vulkan ray tracing pipeline equivalents exist (Vulkan-glTF-PBR, Kajiya) but lack the MIS+wavefront sorting of production PT. Atmospheric scattering (§83) is engine-internal in all production renderers; the Bruneton LUT approach (§83.1–96.2) is the only open, portable reference. Molecular surface (§104) GPU libraries are exclusively CUDA (VMD/APBS). Spanning all ten tables (§1–§104), the CUDA ecosystem dominates approximately forty-five sections; the Vulkan compute patterns remain the sole portable path for the remainder.

**Table K — §45–§107 coverage**

| Library | Ver | GPU Backend | §45 Soft FEM | §46 Cage Deform | §47 Lap Edit | §23 Del Refine | §60 Mink/PRM | §85 OIT | §86 Clipping | §107 Radix Sort | Best Use |
|---------|-----|-------------|:-------------:|:----------------:|:-------------:|:---------------:|:-------------:|:--------:|:-------------:|:---------------:|----------|
| projective-dynamics | research | CUDA | ✓ (§45.3–100.4) | — | — | — | — | — | — | — | PD soft body research |
| FEBio | 4.0 | CPU+CUDA | ✓ (§45.2 FEM) | — | — | — | — | — | — | — | Biomedical FEM |
| libigl | 2.5 | CPU | — | ✓ (§46 MVC) | ✓ (§47 ARAP) | — | — | — | — | — | Geometry processing |
| geometry-central | 1.0 | CPU | — | — | ✓ (§47.1 cotLap) | — | — | — | — | — | Differential geometry |
| TetGen | 1.6 | CPU | ✓ (§45.1 tets) | — | — | ✓ (§23 CDT) | — | — | — | — | Quality tet meshing |
| Triangle (Shewchuk) | 1.6 | CPU | — | — | — | ✓ (§23 Ruppert) | — | — | — | — | 2D quality meshing |
| FCL | 0.7 | CPU | — | — | — | — | ✓ (§60.2 GJK) | — | — | — | Robot collision/planning |
| OMPL | 1.6 | CPU | — | — | — | — | ✓ (§60.3 PRM/RRT) | — | — | — | Motion planning |
| OIT (McGuire) | 2013 | OpenGL/Vulkan | — | — | — | — | — | ✓ (§85.3 WBOIT) | — | — | OIT reference impl |
| Vulkan SDK samples | 1.3 | Vulkan | — | — | — | — | — | ✓ (§85.1 PPLL) | ✓ (§86.2 task) | — | Vulkan OIT + culling |
| VkRadixSort | 2023 | Vulkan | — | — | — | — | — | — | — | ✓ (§107 full sort) | Vulkan compute radix sort |
| CUB / Thrust | CUDA 12 | CUDA | — | — | — | — | — | — | — | ✓ (§107 reference) | CUDA sort primitive |

**VkRadixSort** (Wihlidal, MIT) implements the full 4-pass radix sort (§107.1–107.3) in Vulkan compute shaders, directly applicable to §31 LBVH Morton code sorting and §48 particle spatial hashing. It serves as the portable Vulkan equivalent of CUB's `DeviceRadixSort`. **CUB** (NVIDIA, BSD-3) is the authoritative CUDA implementation, used internally by Thrust, cuBLAS, and all CUDA geometry libraries in this chapter.

**TetGen** (Si, AGPL-3.0) implements constrained Delaunay tetrahedralization (§23 in 3D) and Ruppert-quality refinement. The §45.1 inside/outside GPU classify pass feeds TetGen; its output provides the tetrahedral mesh for §45.2 FEM and §45.3 projective dynamics.

**OMPL** (Rice/CMU, BSD-3) implements PRM (§60.3) and RRT (§60.4) with pluggable collision checkers. Connecting it to the GPU GJK batch checker (§60.2) via FCL's CUDA backend closes the gap between GPU broadphase and CPU planning.

**What is missing in §45–§107.** Soft body FEM (§45) has no Vulkan GPU library; the projective dynamics research code is CUDA-only. Cage deformation (§46) and Laplacian editing (§47) have CPU implementations in libigl/geometry-central but no GPU libraries. Constrained Delaunay refinement (§23) is CPU-only universally. Motion planning (§60) GPU acceleration is an active research area with no production library. OIT (§85) is widely engine-internal; the PPLL and WBOIT patterns are the only portable open references. Radix sort (§107) has VkRadixSort as the sole Vulkan open library; all other geometry libraries use CUDA/CUB internally.

**Overall library landscape summary (§1–§107).** Across all eleven tables, approximately fifty sections have production GPU library support (mostly CUDA). The remaining fifty-seven sections — spanning IK (§40), Laplacian processing (§19, §47), spectral methods (§38), constrained meshing (§23), motion planning (§60), atmospheric scattering (§83), molecular surfaces (§104), and others — rely on the Vulkan compute shader patterns presented in this chapter as the primary portable implementation path. This reflects the general state of the GPU geometry ecosystem: the CUDA library layer is mature and deep; the Vulkan/portable compute layer is the frontier, with the patterns in §1–§107 constituting a practical reference implementation guide.

### 108.12 Why No Comprehensive Vulkan/Slang Geometry Library Exists

The preceding eleven tables reveal a striking asymmetry: roughly half of the algorithm classes in this chapter have no portable (non-CUDA) library at all, and almost none of those that do have Vulkan compute as their primary backend. This is not accidental. Several reinforcing forces explain the gap, and understanding them is as useful to a systems engineer as knowing which library to reach for.

**CUDA's fifteen-year head start closed the question before Vulkan was ready.** CUDA launched in 2007; Thrust, CUB, cuBLAS, and cuSPARSE followed within the next three years. By the time Vulkan compute reached production maturity around 2019, the entire academic and production GPU geometry ecosystem had already crystallised around CUDA. Every SIGGRAPH paper with released code, every course note with GPU pseudocode, and every open-source geometry library assumed CUDA. Authors went where the momentum was, and the momentum compounded.

**No organisation has the right incentive.** NVIDIA built its CUDA library ecosystem as a deliberate platform lock-in investment — the libraries are strategic assets, not acts of altruism. No equivalent incentive exists for Vulkan. The Khronos Group standardises APIs; it does not ship algorithm implementations. AMD, Intel, and ARM each benefit from Vulkan's *existence* as a portable standard, but none benefits enough from a shared, cross-vendor compute geometry library to fund one unilaterally. The constituency that would benefit most — developers targeting hardware they do not control — is also the constituency with the least capital to invest.

**The required shader language did not exist until recently.** Writing a reusable portable geometry library in raw GLSL means maintaining one version per target (GLSL for Vulkan, HLSL for D3D12, MSL for Metal, CUDA C++ for NVIDIA). That tripling of maintenance burden alone is enough to deter most contributors. [Slang](https://shader-slang.com/) (Microsoft Research origins, open-sourced by NVIDIA in 2022–2023) is the first language capable of compiling the same source to SPIR-V, DXIL, PTX, and Metal Shading Language simultaneously. It is also the first GPU shading language with generics, interfaces, and module-level encapsulation sufficient to write library-grade code. The language prerequisite for a serious portable geometry library has existed for roughly two years — not long enough for a library ecosystem to form around it. [Source: He et al. "Slang: A System for Portable and High-Performance DSL Implementation", PLDI 2023]

**Vulkan's API surface is hostile to library authorship.** A CUDA geometry function requires approximately ten lines of setup before the first byte of algorithm code. An equivalent Vulkan compute dispatch requires pipeline objects, descriptor set layouts, descriptor pool allocation, render pass configuration (for graphics shaders), synchronisation barriers, and explicit memory allocation and layout transitions — typically 300–500 lines of boilerplate before the algorithm begins. A library must either carry that boilerplate inside every exported function (making the library enormous and hard to compose) or build an abstraction layer first (which is itself a substantial independent project). Both paths raise the barrier to contribution far above what a research group writing a paper-release codebase will accept.

**Driver fragmentation multiplies the validation burden.** A CUDA library ships against a single driver stack across homogeneous hardware. A Vulkan compute library must validate correctness and performance across AMD RDNA 2/3/4, NVIDIA Turing/Ampere/Ada/Blackwell, Intel Arc/Xe, ARM Mali-G series, Qualcomm Adreno, and MoltenVK on Apple Silicon — each with distinct driver bugs, subgroup sizes (4 to 128), memory models, cooperative matrix extension support, and occupancy characteristics. The quality-assurance surface is roughly eight times larger for the same algorithm, and most of those driver combinations generate bug reports only sporadically. A sustained engineering investment in cross-driver testing is required before a first release can be trusted — a cost that individual research groups cannot absorb.

**Dynamic GPU memory allocation is still painful in Vulkan.** Several algorithms in this chapter — BVH cavity expansion (§106), Delaunay refinement (§23.3), QuickHull horizon-edge collection (§106.3), PPLL fragment append (§85.1) — need to grow output buffers based on runtime-computed sizes. In CUDA, device-side `malloc` and `new` are supported directly inside kernels. In Vulkan compute, the idiomatic approach is to pre-allocate worst-case buffers (wasting memory), use atomic counters with a two-pass pattern (adding latency), or bounce back to the host for reallocation (adding synchronisation overhead). None of these alternatives packages cleanly as a library call with a simple size parameter, because the output size depends on data-dependent control flow that only resolves at runtime on the device.

**The users who need portability do not currently need heavy algorithms.** The applications that run sophisticated BVH construction, spectral mesh processing, or projective dynamics soft bodies are, in practice, running on NVIDIA workstation or datacenter GPUs — hardware where CUDA is already available and preferred. The users who genuinely require Vulkan portability — browser-based (WebGPU), mobile (ARM Mali, Adreno), Apple Silicon (via Metal/MoltenVK) — typically need terrain rendering, particle effects, and basic skinning, not Poisson surface reconstruction or acoustic ray tracing. The portability need and the algorithm complexity need exist in largely disjoint user populations.

**The most credible near-term path: Slang + wgpu.** Two developments have changed the calculus since 2022. First, Slang solves the language fragmentation problem: a single `.slang` source compiles portably to all major GPU targets. Second, [wgpu](https://wgpu.rs/) (Rust, used in Firefox's WebGPU implementation and adopted by several game engines) abstracts Vulkan, Metal, and D3D12 at a level that reduces the boilerplate barrier to a manageable size — comparable to writing a Vulkan application with a mature framework rather than raw API calls. A GPU geometry library written in Slang and exposing a wgpu compute interface would be the first genuine portable candidate. As of 2026 such a library does not exist, but the preconditions — language, abstraction layer, and demonstrated cross-vendor hardware base — have only recently all been satisfied simultaneously. The patterns in §1–§107 of this chapter are, in part, a pre-specification of what that library would contain.

---


## 109. Performance Reference

**Subdivision** (OpenSubdiv stencil evaluation, RTX 4080):

- 400k refined vertices (3 floats each), `GLComputeEvaluator`: ~0.3 ms, limited by SSBO scatter-gather bandwidth
- Custom Vulkan stencil kernel (same data): ~0.28 ms (comparable; pipeline setup overhead differs)
- Adaptive tessellation + hardware tessellator vs. uniform full-resolution subdivision: 3–5× faster at typical view distances

**Marching Cubes** (two-pass GPU MC, RTX 3070):

- 256³ voxel grid (density update + MC + output): ~2 ms total
- Prefix scan overhead: ~0.3 ms (eliminates atomic contention in Pass 2)
- Dynamic metaballs (moving centers, full density recompute): ~1.7 ms density + ~0.3 ms MC

**Skinning** (GPU compute pre-pass):

- 100k vertices, 4 bone influences, joint matrix fetch from SSBO: ~0.1 ms (RTX 3070)
- Bottleneck: scattered reads into `joint_mats[]`; packing as `mat3x4` (12 floats) instead of `mat4` (16 floats) reduces bandwidth by 25%

**IK** (1000 independent chains, 10 bones, 10 iterations):

- FABRIK: ~0.1 ms (RTX 4070)
- CCD: ~0.15 ms (FK recomputation overhead)
- Jacobian DLS (6-DOF, 6 joints): dominated by 3×3 solve; negligible per-chain, trivially parallelizable

**Mesh simplification** (meshoptimizer, CPU, single thread):

- 1M-triangle mesh, 50% target: ~30 ms (QEM edge collapse with heap)
- GPU vertex clustering (32³ grid, 500k input vertices): ~0.5 ms dispatch + ~1 ms radix sort (RTX 3070)

**LBVH construction** (GPU, RTX 3070):

- Morton code generation + 30-bit radix sort: ~0.8 ms for 500k triangles
- Tree construction (Karras algorithm): ~0.3 ms
- AABB propagation (bottom-up atomic): ~0.2 ms
- Total: ~1.3 ms for a fully rebuilt BVH on 500k triangles — suitable for dynamic geometry updated every frame

**NanoVDB sphere tracing** (fragment shader, 1080p, RTX 3070):

- 128 iterations per pixel, 256³ grid, ~4 ms (scene complexity dependent)
- NanoVDB SSBO accessor: ~2× slower than CUDA __ldg path due to lack of texture cache, but within acceptable budget for volume preview rendering

---


## 110. Integrations

- **Ch24 (Vulkan/EGL)** — Vulkan resource allocation patterns for SSBO-based subdivision and skinning buffers; pipeline barrier placement between compute passes.
- **Ch25 (GPU Compute)** — Compute shader dispatch setup, shared memory usage for marching cubes tile optimization, prefix-scan patterns for compaction, and radix sort for Morton-code-based LBVH (§24).
- **Ch127 (Mesh Shaders and VRS)** — Mesh shaders replace the tessellation pipeline for subdivision patches; the amplification shader's LOD selection (§10.4) uses cluster bounding spheres from the same meshlet data structures described in Ch127.
- **Ch133 (Vulkan Compute Queues)** — Skinning pre-pass, IK compute, and BVH rebuild should run on the async compute queue to overlap with graphics rendering; see Ch133 for synchronization between compute and graphics queues.
- **Ch135 (Vulkan Ray Tracing)** — The GPU-built LBVH (§24) feeds into `VkAccelerationStructureBuildGeometryInfoKHR` via `VK_ACCELERATION_STRUCTURE_BUILD_MODE_UPDATE_KHR` for dynamic geometry; Ch135 covers the full AS build and GLSL ray-tracing shader integration. The BLAS/TLAS pipeline in §75 of this chapter provides the geometry layer that Ch135's ray tracing shaders query.
- **Ch141 (Cooperative Matrices)** — QEF matrix accumulation in dual contouring (§3.3) and Jacobian construction in IK (§40.3) benefit from cooperative matrix operations when matrix sizes exceed the warp-level primitive dimensions. The MLP inference in §92 (PointNet, neural SDF) maps directly to cooperative matrix multiply-accumulate patterns.
- **Ch154 (GPU-Driven Rendering)** — GPU-driven pipelines require that geometry (subdivision patches, skinned meshes, LOD index buffers) be finalized in compute before indirect draw commands are issued; see Ch154's indirect dispatch patterns. The task-shader crowd rendering in §51.4 uses the same indirect draw compaction flow as Ch154's two-phase occlusion pass.
- **Ch204 (Shader Algorithm Catalog)** — Reference for quaternion arithmetic primitives (`quat_mul`, `quat_from_axis_angle`) used in CCD and DQS shaders; also covers RK4 integration primitives reused in §71.4 heightfield collision and §96.1 streamline tracing.
- **Ch209 (GPU Simulation Pipelines)** — The fluid (§48), deformable body (§49), rigid body (§50), fracture (§52), and crowd (§51) sections of this chapter provide the algorithm kernels; Ch209 covers how to assemble these kernels into a full simulation pipeline with shared memory layouts, double-buffering, and cross-queue synchronization. Hair strand simulation (§42) and level set evolution (§61) connect to the same pipeline architecture.
- **Ch210 (Terrain and Streaming)** — The CDLOD quadtree (§71.1), sparse clipmap (§71.3), and heightfield ray intersection (§71.4) in this chapter are the GPU geometry algorithms; Ch210 covers the full terrain system including heightmap streaming, virtual texture feedback, and integration with the Vulkan sparse binding API.
- **Ch135 (Vulkan Ray Tracing, lightmap baking)** — §28 uses `VK_KHR_ray_query` inline queries for per-texel AO baking; Ch135 covers the full RT pipeline, acceleration structure lifetime management, and shader binding table layout needed to build the bake pipeline.
- **Ch152 (Rust GPU)** — Differentiable rendering (§93) and geometric deep learning (§92) are active areas for GPU-accelerated training in Rust; the neural SDF inference kernel from §92.3 maps directly to rust-gpu compute shaders for Vulkan-native ML inference.
- **Ch154 (GPU-Driven Rendering, visibility)** — The software occlusion rasteriser (§29.2) and portal BFS (§29.3) feed into the two-phase occlusion pass (§26.4) described in Ch154; §29 provides the culling query algorithms, Ch154 provides the indirect draw compaction that acts on their results.
- **Ch133 (Vulkan Compute Queues, ocean)** — The ocean FFT pipeline (§73) dispatches four compute passes per frame (spectrum evolve, IFFT ×3 for height/Dx/Dz, Jacobian). These passes run on the async compute queue overlapping with shadow map rendering (§76); Ch133's multi-queue synchronisation patterns apply directly.
- **Ch24 (Vulkan/EGL, IBL)** — The PMREM compute dispatch (§77.2) and BRDF LUT bake (§77.3) write to cubemap array images and 2D images respectively. Ch24 covers the image layout transitions (`VK_IMAGE_LAYOUT_GENERAL` during compute write, `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` during fragment read) required for these preprocessing resources.
- **Ch210 (Terrain and Streaming, progressive mesh)** — Progressive mesh streaming (§15) and terrain clipmap streaming (§71.3) use the same pattern: a ring-buffer SSBO refilled by a background transfer queue while the graphics queue renders from already-uploaded data. Ch210 covers the transfer queue + sparse binding setup.

