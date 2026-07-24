# Chapter 204: Shader Algorithm Catalog — Recipes, References, and Use Cases

**Target audiences**: Graphics application developers selecting a technique before implementation; systems and driver developers understanding what the GPU is computing for performance budgeting; browser and web platform engineers mapping WebGPU/WGSL support onto algorithm requirements.

This chapter is a **no-code reference catalog**. Every entry follows the same structure: what the algorithm computes (one sentence), when to use it, key named variants worth knowing, limitations and costs, and a primary reference link. For implementation details, follow the reference. For theoretical background, see Ch135 (ray tracing and the rendering equation) and Ch154 (GPU-driven rendering).

### Why the catalog is weighted toward fragment and compute shaders

The distribution of sections in this catalog mirrors the actual distribution of algorithmic work across the GPU pipeline, not an editorial gap.

**Fragment shaders dominate because that is where visual variety lives.** Every "what colour is this pixel?" decision — lighting, shadowing, material evaluation, post-processing — executes at the per-pixel stage. The combinatorial space of visual effects is effectively unbounded: each new shading model, shadow technique, or post-process is another fragment shader recipe. No other stage has an equivalent explosion of algorithmic variety.

**Compute shaders dominate because they are the general escape hatch from the rasterisation pipeline.** Any algorithm that does not fit the vertex→primitive→rasterise→fragment model — physics, path tracing, neural inference, prefix sums, FFTs, image processing — is expressed as a compute dispatch. The breadth of compute algorithms matches the breadth of GPU computing itself.

**Vertex shaders have narrow algorithmic variety by design.** Their job is fundamentally fixed: transform a position from object space to clip space and pass along interpolated attributes. The entire recipe space reduces to matrix math variants (MVP, skinning, morph targets, instancing). All meaningful vertex shader patterns fit on roughly one page; fragment shaders would fill a book.

**Task and mesh shaders have few recipes because the problem domain is deliberately narrow.** Task shaders do one thing: cull meshlets and emit survivors, with optional LOD selection. Mesh shaders emit meshlet vertices and primitives, with some procedural geometry as a secondary use. The stage was introduced to solve a specific GPU-driven rendering bottleneck, not to express general algorithmic variety. These patterns are covered with full depth in Ch154 (GPU-Driven Rendering), where the architectural context justifies the detail.

**Tessellation (TCS/TES)** is similarly bounded: adaptive LOD, PN triangles, displacement mapping. Three patterns, well-understood, rarely evolving since the fixed-function era.

---

## Table of Contents

- **[I. Rendering Architecture and Pipeline](#i-rendering-architecture-and-pipeline)**
  - [GPU-Driven Rendering](#gpu-driven-rendering)
  - [Clustered Forward+ Shading](#clustered-forward-shading)
  - [Deferred Shading and G-Buffer Rendering](#deferred-shading-and-g-buffer-rendering)
  - [Visibility Buffer and Deferred Texturing](#visibility-buffer-and-deferred-texturing)
  - [Variable Rate Shading](#variable-rate-shading)
  - [Shader Execution Reordering](#shader-execution-reordering)
  - [Foveated and Multiview Rendering](#foveated-and-multiview-rendering)
  - [Object-Space and Texture-Space Shading](#object-space-and-texture-space-shading)
  - [Meshlet Generation](#meshlet-generation)
- **[II. Direct and Area Lighting](#ii-direct-and-area-lighting)**
  - [Punctual Light Attenuation and IES Profiles](#punctual-light-attenuation-and-ies-profiles)
  - [LTC Area Lights](#ltc-area-lights)
  - [Capsule and Tube Area Lights](#capsule-and-tube-area-lights)
  - [Volumetric Light Shafts and God Rays](#volumetric-light-shafts-and-god-rays)
  - [Caustics](#caustics)
- **[III. Shadows and Occlusion](#iii-shadows-and-occlusion)**
  - [Shadow Rendering Techniques](#shadow-rendering-techniques)
  - [Cascaded Shadow Maps](#cascaded-shadow-maps)
  - [Omnidirectional Shadow Maps](#omnidirectional-shadow-maps)
  - [PCSS and Contact-Hardening Shadows](#pcss-and-contact-hardening-shadows)
  - [Exponential Shadow Maps](#exponential-shadow-maps)
  - [Shadow Atlas and Cached Shadows](#shadow-atlas-and-cached-shadows)
  - [Ambient Occlusion](#ambient-occlusion)
  - [Bent Normal and AO Baking](#bent-normal-and-ao-baking)


---

## I. Rendering Architecture and Pipeline

Covers the high-level decisions that determine how the GPU processes a frame: when shading happens (forward vs. deferred), how many pixels receive full shading evaluation (VRS, foveated), and how draw calls are issued (GPU-driven indirect). These techniques are prerequisites for understanding every shading choice that follows — the material and lighting categories below assume one of these pipeline configurations is already in place. GPU-driven rendering and clustered shading in particular set the upper bound on how many lights and draw calls a scene can afford, which in turn constrains every technique that builds on top.

### GPU-Driven Rendering

Traditional rendering submits one draw call per mesh per frame from the CPU, which bottlenecks at CPU-GPU command throughput and CPU-side visibility testing. **GPU-driven rendering** moves all per-draw decisions (culling, LOD selection, draw count) into compute shaders that write `VkDrawIndexedIndirectCommand` structs into a buffer, then submits a single `vkCmdDrawIndexedIndirectCount` that executes only the surviving draws.

**Why this matters**: at 10,000 draw calls/frame, CPU submission overhead is significant. GPU-driven rendering makes draw call count a GPU-compute cost (cheap) rather than a CPU cost (expensive), enabling scenes with 100k+ draw calls at constant CPU overhead.

**Shader stage**: culling and LOD selection run in a **compute shader**. Drawing uses standard **vertex + fragment** shaders reading from per-draw SSBO arrays instead of per-draw uniform updates.

#### Per-Instance Data SSBO
Replace per-draw uniform buffers with a single large SSBO of per-instance data (transform, material index, bounds). All instances are always resident on the GPU; the cull pass selects which ones draw.

```glsl
struct InstanceData {
    mat4  model;
    uint  material_id;
    uint  mesh_id;
    vec3  aabb_min;
    float _pad0;
    vec3  aabb_max;
    float _pad1;
};
layout(set=0, binding=0) readonly buffer InstanceSSBO { InstanceData instances[]; };
layout(set=0, binding=1) writeonly buffer DrawCmdSSBO { VkDrawIndexedIndirectCommand cmds[]; };
layout(set=0, binding=2) buffer DrawCount { uint draw_count; };
```

#### GPU Frustum Culling Compute
One thread per instance tests the instance AABB against the six frustum planes (in view space). Surviving instances atomically increment the draw count and write their draw command.

```glsl
layout(local_size_x=64) in;
void main() {
    uint id = gl_GlobalInvocationID.x;
    if (id >= instance_count) return;

    InstanceData inst = instances[id];
    // Test all 8 AABB corners against 6 frustum planes
    bool visible = frustum_cull(inst.aabb_min, inst.aabb_max, frustum_planes);
    if (!visible) return;

    uint cmd_idx = atomicAdd(draw_count, 1u);
    cmds[cmd_idx].indexCount    = mesh_index_count[inst.mesh_id];
    cmds[cmd_idx].instanceCount = 1u;
    cmds[cmd_idx].firstIndex    = mesh_first_index[inst.mesh_id];
    cmds[cmd_idx].vertexOffset  = mesh_vertex_offset[inst.mesh_id];
    cmds[cmd_idx].firstInstance = id;   // instance ID → indexes into InstanceSSBO
}
```

#### Two-Pass Hi-Z Occlusion Culling
A more powerful variant uses the previous frame's Hi-Z (hierarchical Z) mip pyramid to also cull occluded instances. Pass 1 culls against the previous frame's Hi-Z (fast, approximate). Pass 2 re-renders occluders to update Hi-Z, then a second cull pass tests instances that failed pass 1 against the updated Hi-Z.

```
Pass 1: cull with prev-frame Hi-Z → indirect draw surviving instances
Build Hi-Z mip from current depth buffer (compute)
Pass 2: test previously-rejected instances against new Hi-Z → draw late survivors
```

**Use cases** — standard in UE5 Nanite front-end, Frostbite, id Tech 7. Reduces GPU vertex work by 30–80% in dense urban scenes.  
**Reference** — [Wihlidal: Optimising the Graphics Pipeline with Compute (GDC 2016)](https://frostbite-wp-prd.s3.amazonaws.com/wp-content/uploads/2016/03/29204330/GDC_2016_Compute.pdf); [Aaltonen: GPU-Driven Rendering Pipelines (SIGGRAPH 2015)](https://advances.realtimerendering.com/s2015/)

#### Meshlet / Mesh Shader Cluster Pipeline
With `VK_EXT_mesh_shader`, the vertex-processing stage is replaced by a **task shader** (one threadgroup per meshlet cluster, does frustum and cone culling, emits surviving meshlets) and a **mesh shader** (one threadgroup per meshlet, reads ~64 vertices and ~126 triangles, writes `gl_MeshPrimitivesEXT`). No index buffer rasterization; all vertex amplification and culling happen in the shader.

```glsl
// Task shader — one workgroup per meshlet cluster
layout(local_size_x=32) in;
taskPayloadSharedEXT MeshPayload payload;

void main() {
    uint meshlet_id = gl_WorkGroupID.x * 32 + gl_LocalInvocationID.x;
    bool visible = cone_cull(meshlets[meshlet_id]) && frustum_cull(meshlets[meshlet_id]);
    uvec4 vote   = subgroupBallot(visible);
    uint count   = subgroupBallotBitCount(vote);
    uint idx     = subgroupBallotExclusiveBitCount(vote);
    if (visible) payload.meshlet_ids[idx] = meshlet_id;
    if (gl_LocalInvocationID.x == 0) EmitMeshTasksEXT(count, 1, 1);
}
```

**Use cases** — Nanite meshlet rasterization, GPU-driven LOD without CPU involvement, efficient backface cluster culling.  
**Reference** — [Kubisch: Turing Mesh Shaders (NVIDIA 2018)](https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/); [Vulkan Mesh Shader Spec](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_mesh_shader.html)

---

### Clustered Forward+ Shading

In a scene with hundreds or thousands of dynamic lights, naïve forward shading evaluates every light per fragment. **Clustered Forward+** (Olsson et al. 2012) divides the view frustum into a 3D grid of clusters (typically 16×9×24 = 3456 cells), assigns lights to the clusters they overlap in a compute pre-pass, then each fragment only evaluates the lights in its cluster — typically 0–20 lights even in a 1000-light scene. This is the dominant lighting architecture in modern game engines (Frostbite, UE4/5 forward path, Godot 4).

**Shader stage**: cluster AABB generation and light assignment run in **compute**. Fragment shading reads the cluster light list from a storage buffer.

#### Cluster AABB Generation
Precompute the view-space AABB (or frustum slice) for each cluster tile. This only needs recomputing when the projection matrix changes.

```glsl
// Cluster AABB generation — compute, one thread per cluster
layout(local_size_x=1, local_size_y=1, local_size_z=1) in;

struct AABB { vec3 mn; vec3 mx; };
layout(set=0, binding=0) writeonly buffer ClusterAABBs { AABB aabbs[]; };

uniform uvec3 cluster_dims;   // e.g. (16, 9, 24)
uniform mat4  inv_proj;
uniform float z_near, z_far;

vec3 screen_to_view(vec2 screen_uv, float view_z) {
    vec4 ndc  = vec4(screen_uv * 2.0 - 1.0, -1.0, 1.0);
    vec4 view = inv_proj * ndc;
    return view.xyz / view.w * (view_z / view.z);  // scale to given depth
}

void main() {
    uvec3 c  = gl_GlobalInvocationID;
    uint  idx = c.x + c.y * cluster_dims.x + c.z * cluster_dims.x * cluster_dims.y;

    // Cluster depth slice boundaries (exponential partition matches log-Z distribution)
    float z_near_slice = z_near * pow(z_far / z_near, float(c.z)     / float(cluster_dims.z));
    float z_far_slice  = z_near * pow(z_far / z_near, float(c.z + 1) / float(cluster_dims.z));

    // Screen-space tile corners
    vec2 uv_min = vec2(c.xy)           / vec2(cluster_dims.xy);
    vec2 uv_max = vec2(c.xy + uvec2(1)) / vec2(cluster_dims.xy);

    // Four corners of frustum slice at near and far depths
    vec3 corners[8];
    corners[0] = screen_to_view(vec2(uv_min.x, uv_min.y), -z_near_slice);
    corners[1] = screen_to_view(vec2(uv_max.x, uv_min.y), -z_near_slice);
    corners[2] = screen_to_view(vec2(uv_min.x, uv_max.y), -z_near_slice);
    corners[3] = screen_to_view(vec2(uv_max.x, uv_max.y), -z_near_slice);
    corners[4] = screen_to_view(vec2(uv_min.x, uv_min.y), -z_far_slice);
    corners[5] = screen_to_view(vec2(uv_max.x, uv_min.y), -z_far_slice);
    corners[6] = screen_to_view(vec2(uv_min.x, uv_max.y), -z_far_slice);
    corners[7] = screen_to_view(vec2(uv_max.x, uv_max.y), -z_far_slice);

    AABB ab;
    ab.mn = corners[0]; ab.mx = corners[0];
    for (int i = 1; i < 8; ++i) { ab.mn = min(ab.mn, corners[i]); ab.mx = max(ab.mx, corners[i]); }
    aabbs[idx] = ab;
}
```

#### Light Assignment (Culling Compute Pass)
For each light, test against cluster AABBs and append to the cluster's light list using atomic counters.

```glsl
// Light assignment — one thread per light
layout(local_size_x=64) in;

struct PointLight { vec3 pos_vs; float radius; vec3 color; float intensity; };
layout(set=0, binding=0) readonly buffer Lights      { PointLight lights[]; };
layout(set=0, binding=1) readonly buffer ClusterAABBs{ AABB aabbs[]; };
layout(set=0, binding=2) writeonly buffer LightGrid  { uint light_offset_count[]; }; // [offset, count] per cluster
layout(set=0, binding=3) writeonly buffer LightList  { uint light_indices[]; };
layout(set=0, binding=4) buffer    GlobalCounter     { uint global_light_count; };

uniform uint  num_lights;
uniform uvec3 cluster_dims;
uniform uint  max_lights_per_cluster;  // e.g. 128

void main() {
    uint light_idx = gl_GlobalInvocationID.x;
    if (light_idx >= num_lights) return;

    PointLight L = lights[light_idx];
    // Sphere-AABB overlap test for each cluster
    for (uint z = 0u; z < cluster_dims.z; ++z)
    for (uint y = 0u; y < cluster_dims.y; ++y)
    for (uint x = 0u; x < cluster_dims.x; ++x) {
        uint  cid  = x + y * cluster_dims.x + z * cluster_dims.x * cluster_dims.y;
        AABB  ab   = aabbs[cid];
        // Closest point on AABB to sphere centre
        vec3  cp   = clamp(L.pos_vs, ab.mn, ab.mx);
        float dist2 = dot(cp - L.pos_vs, cp - L.pos_vs);
        if (dist2 <= L.radius * L.radius) {
            uint slot = atomicAdd(light_offset_count[cid * 2u + 1u], 1u);
            if (slot < max_lights_per_cluster) {
                uint list_idx = atomicAdd(global_light_count, 1u);
                light_indices[list_idx] = light_idx;
                // Store list_idx as offset (simplified; production uses two-pass for contiguous offsets)
            }
        }
    }
}
```

#### Fragment Light Evaluation
```glsl
// Fragment shader — clustered light evaluation
uniform uvec3 cluster_dims;
uniform float z_near, z_far;

uint cluster_index(vec3 pos_vs) {
    float z_ratio  = log(-pos_vs.z / z_near) / log(z_far / z_near);
    uint  z_tile   = uint(z_ratio * float(cluster_dims.z));
    vec2  screen_uv = gl_FragCoord.xy / resolution;
    uvec2 xy_tile  = uvec2(screen_uv * vec2(cluster_dims.xy));
    return xy_tile.x + xy_tile.y * cluster_dims.x + z_tile * cluster_dims.x * cluster_dims.y;
}

void main() {
    uint  cid    = cluster_index(v_pos_vs);
    uint  offset = light_offset_count[cid * 2u];
    uint  count  = light_offset_count[cid * 2u + 1u];

    vec3  radiance = vec3(0.0);
    for (uint i = 0u; i < count; ++i) {
        uint      lid = light_indices[offset + i];
        PointLight L  = lights[lid];
        radiance     += evaluate_point_light(L, world_pos, N, V, albedo, roughness, metallic);
    }
    frag_color = vec4(radiance, 1.0);
}
```

**Reference** — [Olsson, Billeter & Assarsson: Clustered Deferred and Forward Shading (HPG 2012)](https://www.cse.chalmers.se/~uffe/clustered_shading_preprint.pdf)

---

### Deferred Shading and G-Buffer Rendering

Deferred shading is a rendering pipeline architecture that decouples geometry rasterization from lighting evaluation. Instead of computing the full lighting equation for every fragment as it is drawn (forward rendering), a geometry pass writes per-pixel surface attributes — the **G-Buffer** — into multiple render targets, and a subsequent lighting pass reads those attributes and evaluates all light contributions once per pixel. See §7.0 of ch84 for the architectural overview; this section contains the shader-level recipes.

**Shader stage**: The geometry pass runs in a standard **vertex + fragment shader** pair with multiple render target outputs. The lighting pass runs as a full-screen **fragment shader** (one triangle covering the screen) or a **compute shader** dispatched over the framebuffer grid. Compute is preferred when per-tile light culling is implemented (tiled deferred / clustered deferred), since compute threads naturally map to screen tiles and can share a tile-local light list via shared memory.

#### G-Buffer Write (Geometry Pass)

Writes per-pixel surface attributes into multiple render targets in a single draw call. The fragment shader encodes material properties rather than computing lighting.

```glsl
// Geometry pass fragment shader — writes G-Buffer MRTs
// Vulkan GLSL, layout locations match VkRenderPass attachment indices

layout(location = 0) out vec4 g_albedo_rough;   // RGB = albedo, A = roughness
layout(location = 1) out vec4 g_normal_metal;   // RGB = world normal (oct-encoded), A = metallic
layout(location = 2) out vec4 g_emissive;       // RGB = emissive, A = unused

layout(set = 1, binding = 0) uniform sampler2D albedo_tex;
layout(set = 1, binding = 1) uniform sampler2D normal_map;
layout(set = 1, binding = 2) uniform sampler2D arm_tex; // R=AO, G=roughness, B=metallic

layout(location = 0) in vec3 v_world_normal;
layout(location = 1) in vec3 v_world_tangent;
layout(location = 2) in vec2 v_uv;

void main() {
    vec4  albedo_smp = texture(albedo_tex, v_uv);
    vec3  arm        = texture(arm_tex, v_uv).rgb;
    vec3  ts_normal  = texture(normal_map, v_uv).rgb * 2.0 - 1.0;

    // TBN transform: tangent-space normal → world space
    vec3 T  = normalize(v_world_tangent);
    vec3 N  = normalize(v_world_normal);
    vec3 B  = cross(N, T);
    vec3 wN = normalize(mat3(T, B, N) * ts_normal);

    g_albedo_rough = vec4(albedo_smp.rgb, arm.g);      // roughness in alpha
    g_normal_metal = vec4(wN * 0.5 + 0.5, arm.b);     // pack normal to [0,1]
    g_emissive     = vec4(albedo_smp.rgb * arm.r, 0.0); // emissive driven by AO channel
}
```

**Use cases** — any renderer with >~8 dynamic lights per scene; standard architecture for Unreal Engine, Unity (URP deferred), Godot, Blender EEVEE Next.  
**Key variants** — packed G-Buffer (fit all attributes into 2 render targets using oct-encoding for normals); thin G-Buffer (albedo + packed normal only, reconstruct roughness from SSAO).  
**Reference** — [LearnOpenGL: Deferred Shading](https://learnopengl.com/Advanced-Lighting/Deferred-Shading) | [Filament deferred shading design](https://google.github.io/filament/Filament.html)

#### Deferred Lighting Pass (Full-Screen Fragment or Compute)

Reads the G-Buffer and evaluates all analytical lights, one thread per pixel. The lighting accumulation loop iterates over visible lights — either all lights globally or a tile-culled subset.

```glsl
// Deferred lighting pass — full-screen fragment shader
// Reconstruct position from depth; evaluate PBR per pixel.
// Vulkan GLSL

layout(set = 0, binding = 0) uniform sampler2D g_albedo_rough;
layout(set = 0, binding = 1) uniform sampler2D g_normal_metal;
layout(set = 0, binding = 2) uniform sampler2D g_depth;

layout(set = 0, binding = 3) uniform CameraUB {
    mat4 inv_proj;
    mat4 inv_view;
    vec3 cam_pos;
};

struct PointLight { vec3 pos; float radius; vec3 color; float intensity; };
layout(set = 0, binding = 4) readonly buffer LightBuf { PointLight lights[]; };
layout(set = 0, binding = 5) uniform LightCount { uint num_lights; };

layout(location = 0) in  vec2 v_uv;
layout(location = 0) out vec4 frag_color;

// Reconstruct world-space position from depth and inverse projection
vec3 reconstruct_position(vec2 uv, float depth) {
    vec4 clip = vec4(uv * 2.0 - 1.0, depth, 1.0);
    vec4 view = inv_proj * clip;
    view /= view.w;
    return (inv_view * vec4(view.xyz, 1.0)).xyz;
}

void main() {
    float depth  = texture(g_depth,        v_uv).r;
    vec4  alb_r  = texture(g_albedo_rough, v_uv);
    vec4  nor_m  = texture(g_normal_metal, v_uv);

    vec3 albedo    = alb_r.rgb;
    float roughness = alb_r.a;
    vec3 N          = normalize(nor_m.rgb * 2.0 - 1.0);
    float metallic  = nor_m.a;
    vec3 P          = reconstruct_position(v_uv, depth);
    vec3 V          = normalize(cam_pos - P);

    vec3 Lo = vec3(0.0);
    for (uint i = 0u; i < num_lights; ++i) {
        PointLight l   = lights[i];
        vec3  L        = normalize(l.pos - P);
        float dist     = length(l.pos - P);
        float atten    = max(0.0, 1.0 - dist / l.radius);
        vec3  radiance = l.color * l.intensity * atten * atten;
        Lo += brdf_cook_torrance(albedo, metallic, roughness, N, V, L) * radiance;
    }
    frag_color = vec4(Lo, 1.0);
}
```

**Use cases** — standard deferred renderer lighting accumulation; combined with shadow map sampling and IBL contribution added after the loop.  
**Key variants** — tiled deferred: dispatch compute with one workgroup per screen tile, load a tile-local light list into shared memory, iterate only those lights; clustered deferred: extends tiling into depth slices for correct lighting at varying depths.  
**Reference** — [Olsson & Assarsson: Tiled Shading (JCGT 2011)](https://jcgt.org/published/0001/01/03/) | [Harada et al.: A 2.5D Culling for Forward+ (SIGGRAPH Asia 2012)](https://dl.acm.org/doi/10.1145/2407746.2407780)

#### Tiled/Clustered Light Culling (Compute Prepass)

Before the lighting pass, a compute shader divides the screen into tiles (typically 16×16 pixels) and builds a per-tile list of lights whose bounding sphere overlaps the tile's frustum slice. The lighting pass reads the per-tile list, avoiding global light iteration.

**Use cases** — scenes with >32 analytical lights; required for Forward+ (applies the same tile list to a forward renderer) and tiled deferred.  
**Key variants** — clustered: extends tiles into *z* slices (froxels) for correct behaviour with wide depth ranges (outdoor scenes).  
**Limitations** — the prepass itself costs ~0.1–0.3 ms; breakeven versus full-loop deferred is around 50 lights depending on scene geometry complexity.  
**Reference** — [Dufresne: Adventures in Clustered Shading (GDC 2018)](https://advances.realtimerendering.com/s2018/) | [Filament clustered light implementation](https://github.com/google/filament/blob/main/shaders/src/light_indirect.fs)

#### Subpass-Merged G-Buffer on TBDR (Mobile)

On TBDR mobile GPUs (Adreno, Mali, PowerVR) the geometry pass and lighting pass can be expressed as Vulkan subpasses within a single render pass. The driver merges them into one tile pass: the G-Buffer attachments live in on-chip tile SRAM throughout, never written to DRAM. The lighting subpass reads G-Buffer data via `inputAttachment` samplers (or `VK_EXT_shader_tile_image` framebuffer fetch), which the hardware routes from tile SRAM rather than issuing DRAM reads.

**Use cases** — mobile deferred renderers (Unity mobile deferred, Unreal Mobile deferred); the only way to make deferred rendering bandwidth-competitive with forward on TBDR hardware.  
**Limitations** — subpass input attachments can only read the current fragment's texel (no neighbouring pixels); screen-space effects that need neighbour access (SSAO, SSR) must still break the tile pass.  
**Reference** — [ARM Mali GPU Best Practices — Deferred shading](https://developer.arm.com/documentation/101897/latest/) | ch86 §10 and §10b of this book

---

### Visibility Buffer and Deferred Texturing

The G-Buffer's geometry pass writes 4–8 full-resolution render targets. The visibility buffer replaces this with a single 64-bit render target: high bits = draw call / material ID, low bits = primitive ID. A subsequent compute pass reconstructs every material attribute on demand by looking up the mesh and sampling textures. Unreal Engine 5's Nanite uses this approach.

**Shader stage**: The triangle ID pass uses a standard **vertex + fragment** pair with a single integer MRT. The material evaluation pass uses a **compute shader** dispatched over the framebuffer.

#### Triangle ID Pass
Writes a packed `uvec2` per pixel: `x` = draw ID (encodes material and mesh), `y` = primitive ID within that draw. No texture sampling, no interpolated attributes beyond position.

```glsl
// Visibility buffer write — fragment shader
layout(location = 0) out uvec2 vis_buf;  // single R32G32_UINT render target

layout(location = 0) flat in uint v_draw_id;

void main() {
    vis_buf = uvec2(v_draw_id, gl_PrimitiveID);
}
```

**Use cases** — first pass of a visibility buffer renderer; ~4–8× lower geometry pass bandwidth than a full G-Buffer; enables heterogeneous material sets without a fixed G-Buffer schema.  
**Reference** — [Burns & Hunt: The Visibility Buffer: A Cache-Friendly Approach to Deferred Shading (JCGT 2013)](http://jcgt.org/published/0002/02/04/)

#### Barycentric Attribute Reconstruction
Given a primitive ID, fetches the three vertex positions from the index buffer and vertex buffer, computes screen-space partial derivatives, and derives correct perspective-correct barycentric coordinates — enabling any per-vertex attribute (UV, normal, colour) to be reconstructed in compute.

**Use cases** — material evaluation pass in a visibility buffer renderer; correct UV derivatives (`ddx`/`ddy` equivalents) from compute without rasterizer interpolation.  
**Reference** — [Wihlidal: Optimising the Graphics Pipeline with Compute (GDC 2016)](https://frostbite-wp-prd.s3.amazonaws.com/wp-content/uploads/2016/03/29204330/GDC_2016_Compute.pdf)

#### Deferred Texturing Compute Pass
One compute thread per pixel reads the visibility buffer, looks up the draw and primitive records from GPU-side mesh and material data, reconstructs UVs via barycentric interpolation, samples material textures, and writes the fully shaded result — or writes a surrogate G-Buffer for a standard lighting pass.

**Use cases** — complete visibility buffer pipeline; enables arbitrary material diversity at constant geometry pass cost; required for Nanite-style micro-polygon rendering.  
**Reference** — [UE5 Nanite: A Deep Dive (SIGGRAPH 2021)](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)

---

### Variable Rate Shading

Variable Rate Shading (VRS) decouples the shading rate from the pixel rate: rather than running one fragment shader invocation per pixel, the GPU can shade a 2×2, 4×2, or 4×4 tile of pixels with a single invocation and fill the whole tile with that result. VRS reduces fragment shader cost in image regions where full shading rate is wasteful — low-frequency regions, peripheral VR field of view, motion-blurred areas, or any region that will be upscaled. In Vulkan, VRS is provided by `VK_KHR_fragment_shading_rate` (core in Vulkan 1.3 on supporting hardware).

**Shader stage**: the shading rate can be set **per-draw** (uniform), **per-primitive** (vertex/geometry shader output), or **per-tile** (a shading-rate image read before rasterisation). Evaluation of the actual shading happens in the **fragment shader**, which sees the tile as a single invocation.

#### Per-Draw Uniform Rate
Set a single shading rate for an entire draw call — useful for secondary geometry (decals, particles, foliage) where full rate is unnecessary.

```c
// Vulkan: set per-draw fragment shading rate
VkFragmentShadingRateAttachmentInfoKHR sri = {
    .fragmentShadingRateAttachment       = NULL,
    .shadingRateAttachmentTexelSize      = { 1, 1 },
};
vkCmdSetFragmentShadingRateKHR(cmd,
    &(VkExtent2D){ 2, 2 },   // 2×2 pixel tile per shading invocation
    (VkFragmentShadingRateCombinerOpKHR[]){
        VK_FRAGMENT_SHADING_RATE_COMBINER_OP_KEEP_KHR,
        VK_FRAGMENT_SHADING_RATE_COMBINER_OP_KEEP_KHR,
    });
```

#### Shading Rate Image (Per-Tile)
Build a 2D image where each texel covers an N×N tile (typically 16×16 pixels) of the render target and contains the packed shading rate for that tile. Content-adaptive VRS reads luminance variance, velocity magnitude, or foveation radius to determine per-tile rates.

```glsl
// Build shading rate image — compute shader, one thread per tile
layout(set=0, binding=0) uniform sampler2D luma_buffer;   // previous-frame luminance
layout(set=0, binding=1) uniform sampler2D velocity_buf;
layout(set=0, binding=2, r8ui) uniform uimage2D shading_rate_img;

uniform float variance_threshold;  // e.g. 0.01 — below this: use 4×4
uniform float velocity_threshold;  // e.g. 0.003 — above this: use 1×1

void main() {
    ivec2 tile  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv    = (vec2(tile) + 0.5) / vec2(imageSize(shading_rate_img));

    // Sample 2×2 region of luminance and compute variance
    float l00   = texture(luma_buffer, uv + vec2(-0.5, -0.5) / resolution).r;
    float l10   = texture(luma_buffer, uv + vec2( 0.5, -0.5) / resolution).r;
    float l01   = texture(luma_buffer, uv + vec2(-0.5,  0.5) / resolution).r;
    float l11   = texture(luma_buffer, uv + vec2( 0.5,  0.5) / resolution).r;
    float mean  = (l00 + l10 + l01 + l11) * 0.25;
    float var   = dot(vec4(l00,l10,l01,l11)-mean, vec4(l00,l10,l01,l11)-mean) * 0.25;

    float vel   = length(texture(velocity_buf, uv).rg);

    // VK_KHR_fragment_shading_rate packed rates:
    // 0 = 1×1, 1 = 1×2, 4 = 2×1, 5 = 2×2, 10 = 4×2, etc.
    uint rate;
    if (vel > velocity_threshold || var > variance_threshold) {
        rate = 0u;   // 1×1 full rate
    } else if (var > variance_threshold * 0.1) {
        rate = 5u;   // 2×2
    } else {
        rate = 10u;  // 4×2 (check hardware support)
    }
    imageStore(shading_rate_img, tile, uvec4(rate));
}
```

#### Per-Primitive Rate (VRS from Vertex Shader)
The vertex shader can output a primitive shading rate as a built-in:

```glsl
// Vertex shader — set per-primitive shading rate for this triangle
layout(location=5) out gl_PerVertex {
    vec4 gl_Position;
};
// VK_KHR_fragment_shading_rate built-in (SPIR-V PrimitiveShadingRateKHR)
layout(location=1) out int gl_PrimitiveShadingRateEXT;

void main() {
    // Dynamic objects always shaded at 1×1; distant static at 2×2
    gl_PrimitiveShadingRateEXT = is_dynamic ? 0 : 5;
}
```

**Use cases** — VR foveated rendering where 4×4 is used at the periphery (§27); TAA passes where full rate is wasted; particle/foliage secondary passes; always combined with TAA or DLSS to restore temporal detail at reduced rates.  
**Reference** — [Vulkan: VK_KHR_fragment_shading_rate](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_fragment_shading_rate.html); [Turner: Variable Rate Shading (NVIDIA DX12/Vulkan)](https://developer.nvidia.com/variable-rate-shading)

---

### Shader Execution Reordering

In ray tracing, the GPU executes shaders in the order that rays hit geometry — determined by the BVH traversal hardware, not the programmer. This causes **wavefront divergence**: rays in the same wave hit different materials, execute different shader code, and access different textures, resulting in poor SIMD utilisation and cache thrash. **Shader Execution Reordering (SER)**, introduced by NVIDIA on Ada Lovelace (RTX 4000), defers the closest-hit shader invocation until similar shaders are grouped together, improving SIMD coherence by 2–3× for complex scenes.

SER is exposed through `VK_NV_ray_tracing_invocation_reorder` (Vulkan) and GLSL `GL_NV_shader_invocation_reorder`.

**Shader stage**: SER is applied in the **closest-hit shader** (and optionally miss shader) by wrapping the shader body with `hitObjectNV` construction and a `reorderThreadNV` call before the actual shading.

#### HitObject Construction and Reorder
```glsl
#extension GL_NV_shader_invocation_reorder : require

layout(set=0, binding=0) uniform accelerationStructureEXT tlas;

// Payload
layout(location=0) rayPayloadNV vec3 payload_radiance;

void main() {
    hitObjectNV hit;

    // Trace ray without invoking any shaders — capture the hit in a hitObject
    hitObjectTraceRayNV(hit,
        tlas,
        gl_RayFlagsNoneNV,
        0xFF,           // cull mask
        0, 1, 0,        // SBT offsets
        gl_WorldRayOriginNV,
        0.001,          // tmin
        gl_WorldRayDirectionNV,
        1e27,           // tmax
        0               // payload location
    );

    // Provide a hint to the hardware scheduler: group threads by material/shader
    // The 5-bit coherence hint encodes material type (affects SIMD grouping)
    uint material_hint = hitObjectGetInstanceIdNV(hit) & 0x1Fu;
    reorderThreadNV(hit, material_hint, 5 /*hint bits*/);

    // After reordering, invoke the closest-hit shader
    hitObjectExecuteShaderNV(hit, 0);
}
```

#### Coherence Hint Design
The quality of SER depends on the coherence hint accurately predicting shader divergence. A good hint encodes: material shader index, whether the surface uses alpha masking (potentially invokes any-hit), and BRDF type. A bad hint (all-zeros) still produces valid output but provides no reordering benefit.

```glsl
// Build a coherence hint from instance data
uint build_coherence_hint(hitObjectNV hit) {
    int inst_id   = hitObjectGetInstanceIdNV(hit);
    int geom_id   = hitObjectGetGeometryIndexNV(hit);
    // Encode shader type in high bits, geometry variation in low bits
    uint shader_type = (inst_id >> 8) & 0x0Fu;  // from instance custom index
    uint geom_var    = geom_id & 0x0Fu;
    return (shader_type << 1u) | (geom_var & 0x1u);
}
```

**Performance impact**: for scenes with many material types (e.g., an urban environment with glass, metal, painted surfaces, vegetation), SER can improve closest-hit throughput by 2–3×. For scenes with a single dominant material (e.g., a terrain with one BRDF), the benefit is minimal. SER is transparent to correctness — the output is identical whether or not reordering occurs.

**Reference** — [NVIDIA: Shader Execution Reordering (GTC 2022)](https://developer.nvidia.com/blog/improve-shader-performance-and-in-game-frame-rates-with-shader-execution-reordering/); [Vulkan: VK_NV_ray_tracing_invocation_reorder](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_NV_ray_tracing_invocation_reorder.html)

---

### Foveated and Multiview Rendering

VR/XR rendering must produce two views per frame at high resolution and high frame rate. Two Vulkan techniques reduce this cost: multiview renders geometry once for both eyes simultaneously; variable-rate shading coarsens shading in the peripheral visual field where the eye has low resolution.

**Shader stage**: Multiview uses a standard **vertex shader** that reads `gl_ViewIndex` to select the correct transform. FSR writes a shading rate image from **compute** before the main render pass. ATW reprojection runs as a **compute** or **fragment** shader.

#### VK_KHR_multiview
Renders a single draw call into a 2D array render target with two layers. The vertex shader uses `gl_ViewIndex` to index into an array of view matrices, writing `gl_Position` for both eyes from one geometry submission.

```glsl
#extension GL_EXT_multiview : require
layout(set=0, binding=0) uniform ViewUB { mat4 view_proj[2]; };

layout(location=0) in vec3 in_pos;

void main() {
    gl_Position = view_proj[gl_ViewIndex] * vec4(in_pos, 1.0);
}
```

**Use cases** — stereo VR rendering; reduces geometry processing and draw call overhead by up to 50%; mandatory for high-performance VR on mobile XR hardware (Quest).  
**Reference** — [Khronos: VK_KHR_multiview specification](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_multiview.html)

#### Variable Rate Shading — Fixed Foveated (VK_KHR_fragment_shading_rate)
Writes a shading rate image (one tile per N×N pixels) before the main render pass. Tiles in the peripheral screen region are marked 2×2 or 4×4 (one shader invocation per 4 or 16 pixels); the central fovea region is marked 1×1 (full rate). The rasterizer applies the coarser rate automatically during fragment dispatch.

**Use cases** — fixed foveated rendering on Quest, PSVR2, and PC VR headsets; 30–50% fragment shader cost reduction with minimal perceptual quality loss.  
**Reference** — [Qualcomm: Fixed Foveated Rendering Best Practices](https://developer.qualcomm.com/software/adreno-gpu-sdk/tools)

#### Eye-Tracked Foveated Rendering
Updates the shading rate image every frame from eye tracker coordinates: full-rate 1×1 region is centred on the gaze point rather than the screen centre. Requires per-frame compute dispatch to write the rate image before the main render pass begins.

**Use cases** — high-end VR headsets with eye tracking (Apple Vision Pro, Varjo, Pico); 40–60% fragment shader reduction; requires latency-hiding to avoid smearing artefacts.  
**Reference** — [Foveated Rendering Overview — Khronos Blog](https://www.khronos.org/blog/vulkan-variable-rate-shading)

#### Asynchronous TimeWarp / SpaceWarp (ATW/ASW)
Reprojects the last rendered frame to the current head pose using depth and motion vectors when a new frame is not ready in time, synthesising a valid display frame with minimal latency. The reprojection runs as a high-priority compute or graphics pass on a separate GPU queue.

**Use cases** — VR compositor reprojection fallback; prevents judder when the application misses its frame deadline; implemented in Oculus, OpenVR, and Monado runtimes.  
**Reference** — [Oculus: Asynchronous TimeWarp](https://developer.oculus.com/documentation/native/android/mobile-timewarp-overview/)

---

### Object-Space and Texture-Space Shading

Standard deferred shading (§8) and forward+ (§98) shade fragments in screen space — the same surface point may be shaded multiple times as the camera moves. **Object-space shading** (also called **texture-space shading** or **surface cache shading**) instead renders shading into a world-space texture atlas (the object's unwrapped UV layout), storing the result per surface texel. Subsequent frames read from this cache rather than re-shading, amortising expensive operations (GI evaluation, SSS blur, lightmapping) across many frames. Unreal Engine 5's Lumen Surface Cache, which stores indirect irradiance per texel for use in reflections and GI, is the primary production example.

**Shader stage**: shading runs in a **compute or rasterisation** pass writing into an atlas-space texture. Cache reading runs in the **fragment or compute** lighting pass, sampling the atlas by UV.

#### Surface Cache Rasterisation
Render the object using its lightmap/second UV channel. Each fragment writes shading into the atlas rather than the framebuffer.

```glsl
// Vertex shader — rasterise into texture-space (lightmap UV as position)
layout(location=0) in vec3 position;
layout(location=1) in vec2 uv_surface;   // primary UV (for materials)
layout(location=2) in vec2 uv_lightmap;  // lightmap UV (becomes clip-space position)

layout(location=0) out vec3 v_world_pos;
layout(location=1) out vec3 v_world_normal;
layout(location=2) out vec2 v_surface_uv;

uniform mat4 model;

void main() {
    v_world_pos    = (model * vec4(position, 1.0)).xyz;
    v_world_normal = normalize(mat3(model) * normal);
    v_surface_uv   = uv_surface;
    // Map lightmap UV [0,1] to NDC [-1,1] — renders into the atlas texture
    gl_Position    = vec4(uv_lightmap * 2.0 - 1.0, 0.5, 1.0);
}

// Fragment shader — write shading into surface cache atlas
layout(location=0) in  vec3 v_world_pos;
layout(location=1) in  vec3 v_world_normal;
layout(location=2) in  vec2 v_surface_uv;
layout(location=0) out vec4 surface_cache_irradiance;   // indirect diffuse
layout(location=1) out vec4 surface_cache_albedo;

void main() {
    // Evaluate world-space radiance cache at this surface point (§84)
    vec3 indirect = wsrc_irradiance(v_world_pos, v_world_normal, vec3(0.0));
    surface_cache_irradiance = vec4(indirect, 1.0);
    surface_cache_albedo     = texture(albedo_map, v_surface_uv);
}
```

#### Reading the Surface Cache in Screen-Space Lighting
```glsl
// Fragment shader — read surface cache during screen-space lighting
layout(set=1, binding=0) uniform sampler2D surface_cache_irr;   // atlas
layout(set=1, binding=1) uniform sampler2D surface_cache_alb;

layout(location=2) in vec2 v_lightmap_uv;

void main() {
    vec3 cached_irr = texture(surface_cache_irr, v_lightmap_uv).rgb;
    vec3 cached_alb = texture(surface_cache_alb, v_lightmap_uv).rgb;

    // Combine cached indirect with direct
    vec3 direct   = evaluate_direct_lights(N, V, albedo, roughness, metallic);
    vec3 indirect = cached_irr * cached_alb / 3.14159;   // Lambertian indirect
    frag_color    = vec4(direct + indirect, 1.0);
}
```

**Temporal update**: update only a fraction of atlas texels per frame (e.g., 1/16th of texels per frame in a fixed pattern), spreading the shading cost over 16 frames. Track which texels are stale using a generation counter.

**Reference** — [UE5 Lumen: Surface Cache Technical Details](https://docs.unrealengine.com/5.0/en-US/lumen-technical-details-in-unreal-engine/); [Wihlidal: Decoupled Deferred Shading on the GPU (HPG 2017)](https://dl.acm.org/doi/10.1145/3105762.3105763)

---

### Meshlet Generation

The mesh shader pipeline (§32) dispatches one task shader workgroup per **meshlet** — a small cluster of ≤64 vertices and ≤126 primitives with a precomputed bounding sphere and normal cone for efficient culling. Meshlets are not a GPU runtime algorithm; they are generated offline (or at asset-load time) by a preprocessing step that partitions the mesh. This section explains the algorithm so that the mesh shader cluster data format is fully understood.

**Tool**: the reference implementation is [meshoptimizer](https://github.com/zeux/meshoptimizer) (`meshopt_buildMeshlets`). The algorithm described here matches its output format.

#### Meshlet Cluster Data Layout
Each meshlet stores:
- An index into a shared **vertex buffer** (64 entries max): `uint8_t vertices[64]` — local vertex indices into the mesh's global vertex array.
- A packed **triangle list** (126 entries max): `uint8_t indices[378]` — triplets of indices into the local `vertices[]` array (3 bytes per triangle).
- A **bounding sphere** (centre + radius) for frustum culling.
- A **normal cone** (apex, axis direction, cutoff angle) for backface cluster culling.

```c
// meshoptimizer meshlet structure (CPU-side, matches GPU SSBO layout)
struct Meshlet {
    uint32_t vertex_offset;    // offset into global meshlet_vertices[] array
    uint32_t triangle_offset;  // offset into global meshlet_triangles[] array
    uint32_t vertex_count;     // ≤ 64
    uint32_t triangle_count;   // ≤ 126

    // Bounding sphere (for frustum cull in task shader)
    float    center[3];
    float    radius;

    // Normal cone (for backface cull in task shader)
    int8_t   cone_axis[3];     // quantised normalised axis
    int8_t   cone_cutoff;      // cos(half_angle) × 127
    float    cone_apex[3];
};
```

#### Greedy Flood-Fill Partitioning
The core algorithm grows meshlet clusters by a greedy flood-fill over the mesh adjacency graph:

1. Start with an unused triangle as the seed.
2. For each candidate adjacent triangle: if adding it keeps vertex count ≤ 64 and triangle count ≤ 126, add it to the current meshlet.
3. When no adjacent triangle fits, close the meshlet, record it, start a new one.
4. Score candidates by the number of vertices they share with the current meshlet (maximising vertex cache reuse and minimising meshlet count).

```glsl
// This runs CPU-side or in a compute preprocessing shader
// Pseudocode for the greedy partitioner:
//   for each unused_triangle t:
//     if current_meshlet.try_add(t): continue
//     else: finalise(current_meshlet); current_meshlet = new Meshlet(); current_meshlet.add(t)
//   finalise(current_meshlet)
// try_add: check if all 3 vertices of t are already in vertex_map OR
//   vertex_count + new_vertices ≤ MAX_VERTICES AND triangle_count < MAX_TRIANGLES
```

#### Normal Cone Computation
For each meshlet, compute the tightest cone enclosing all triangle normals. In the task shader, if the cone's apex direction (negated) has a dot product with the view direction greater than `cos(cone_angle + 90°)`, the entire meshlet is backfacing and can be discarded without invoking the mesh shader.

```glsl
// Task shader backface cone culling — from §32, explained here
// cone_axis: average normal direction of meshlet's triangles
// cone_cutoff: cos(max deviation of any triangle normal from cone_axis)
bool meshlet_is_backface(Meshlet m, vec3 cam_pos) {
    vec3 apex      = vec3(m.cone_apex);
    vec3 axis      = vec3(m.cone_axis) / 127.0;  // dequantise
    float cutoff   = float(m.cone_cutoff) / 127.0;
    vec3  view_dir = normalize(apex - cam_pos);
    // If all normals face away from camera: dot(view, axis) > cutoff
    return dot(view_dir, axis) >= cutoff;
}
```

**Performance impact**: meshlet generation enables the task shader to cull 40–70% of clusters on typical scene geometry before any triangle rasterization occurs, as measured in Nanite and id Tech 7 presentations.  
**Reference** — [Wihlidal: GPU-Driven Rendering (GDC 2016)](https://advances.realtimerendering.com/s2016/); [meshoptimizer: zeux/meshoptimizer](https://github.com/zeux/meshoptimizer); [Kubisch: Turing Mesh Shaders (NVIDIA 2018)](https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/)

---


---

## II. Direct and Area Lighting

Covers the mathematical models for evaluating individual light-source contributions at a surface point: punctual lights (point, spot, directional) with physically accurate distance attenuation and IES photometric profiles, analytically integrated area lights via Linearly Transformed Cosines (LTC), and extended geometric shapes (capsule, tube). Volumetric light shafts and caustics are included here because both are direct-lighting phenomena — they arise from directed light interacting with participating media or refractive surfaces, not from indirect bounces.

### Punctual Light Attenuation and IES Profiles

Punctual lights (point, spot, directional) are the simplest lighting primitives, but physically correct attenuation and realistic beam profiles require a few non-obvious details.

**Shader stage**: evaluated in the **fragment shader** (deferred lighting pass or forward pass) once per light per fragment.

#### Inverse-Square Attenuation
Physical light intensity falls off as `1/distance²`. A bare `1/d²` diverges at zero distance; the standard fix is a small epsilon bias or a windowed falloff that smoothly goes to zero at the light's influence radius.

```glsl
// UE4 smooth windowed inverse-square attenuation
float point_attenuation(float dist, float inv_radius) {
    float d_norm = dist * inv_radius;
    float falloff = 1.0 / (dist * dist + 1.0);           // avoid singularity at d=0
    float window  = pow(max(0.0, 1.0 - d_norm * d_norm * d_norm * d_norm), 2.0);
    return falloff * window;
}
```

**Reference** — [Karis: Real Shading in Unreal Engine 4 (SIGGRAPH 2013)](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)

#### Spot Light Angular Attenuation
A spotlight's cone is characterised by inner and outer half-angles. Attenuation falls from 1 inside the inner cone to 0 at the outer cone, typically with a smooth Hermite curve.

```glsl
float spot_attenuation(vec3 L, vec3 spot_dir, float cos_inner, float cos_outer) {
    float cos_angle = dot(-L, spot_dir);
    float t = clamp((cos_angle - cos_outer) / (cos_inner - cos_outer), 0.0, 1.0);
    return t * t;   // smoothstep-equivalent; use t*t*(3-2t) for C1 continuity
}
```

#### IES Photometric Profiles
Real luminaires (streetlights, theatre spots, LEDs) have complex asymmetric emission distributions measured and distributed as IES files. The 2D distribution (vertical angle, horizontal angle → candela) is baked into a 2D `LUMINANCE` texture at content-load time and sampled in the fragment shader using the light-to-fragment direction.

```glsl
layout(set=2, binding=0) uniform sampler2D ies_profile;  // (θ, φ) → luminous intensity

// direction L = normalize(fragment_pos - light_pos), in light local space
float theta = acos(clamp(-L.z, -1.0, 1.0));          // vertical: 0=up, π=down
float phi   = atan(L.y, L.x) / (2.0 * PI) + 0.5;    // horizontal: [0,1)
float ies_scale = texture(ies_profile, vec2(theta / PI, phi)).r;
// Multiply point_attenuation() by ies_scale before BRDF integration
```

**Use cases** — architectural visualisation, cinematic lighting, any scene using real fixture data; standard in Unreal, Blender Cycles, Filament.  
**Reference** — [IES LM-63 Standard](https://www.iesna.org/); [Lagarde: Moving Frostbite to PBR (SIGGRAPH 2014)](https://seblagarde.wordpress.com/2015/07/14/siggraph-2014-moving-frostbite-to-physically-based-rendering/)

---

### LTC Area Lights

Area lights — spheres, disks, rectangles, tubes — emit radiance from a surface rather than a point. Evaluating their contribution analytically (without Monte Carlo sampling) requires integrating the BRDF over the solid angle subtended by the light, which has no closed form for rough GGX specular.

**Linearly Transformed Cosines** (Heitz & de Dreu 2016) solve this by finding a 3×3 linear transformation `M` that maps the clamped cosine distribution into the GGX BRDF lobe for the current roughness and view angle. The area light integral in the transformed space reduces to the integral of a clamped cosine over a polygon — which has an exact analytic formula. `M` and its normalisation are stored in two RGBA16F lookup textures indexed by `(NdotV, roughness)` and fetched once per fragment.

**Shader stage**: evaluated in the **fragment shader** (deferred lighting pass or forward shading) after the BRDF LUT lookup. No ray tracing required.

```glsl
// LTC area light evaluation — rectangle light example
// Heitz et al. 2016: Real-Time Polygonal-Light Shading with LTC
// Two precomputed textures: ltc_mat (inverse transform matrix) and ltc_amp (scale factor)
layout(set=1, binding=0) uniform sampler2D ltc_mat;   // RGBA → columns of M⁻¹
layout(set=1, binding=1) uniform sampler2D ltc_amp;   // RG  → amplitude, fresnel scale

const float LUT_SIZE  = 64.0;
const float LUT_SCALE = (LUT_SIZE - 1.0) / LUT_SIZE;
const float LUT_BIAS  = 0.5 / LUT_SIZE;

// Look up LTC matrix for this (roughness, NdotV)
vec2 uv      = vec2(roughness, sqrt(1.0 - NdotV));
uv           = uv * LUT_SCALE + LUT_BIAS;
vec4 ltc     = texture(ltc_mat, uv);
mat3 Minv    = mat3(
    vec3(ltc.x, 0.0, ltc.y),
    vec3(  0.0, 1.0,   0.0),
    vec3(ltc.z, 0.0, ltc.w)
);

// Transform rectangle light corners into LTC space and integrate
vec3 L[4];
L[0] = normalize(Minv * (rect[0] - P));
L[1] = normalize(Minv * (rect[1] - P));
L[2] = normalize(Minv * (rect[2] - P));
L[3] = normalize(Minv * (rect[3] - P));

// Polygon irradiance — exact analytic integral of clamped cosine over polygon
float irr = 0.0;
for (int i = 0; i < 4; ++i) {
    vec3 a = L[i], b = L[(i+1)%4];
    irr += acos(clamp(dot(a, b), -1.0, 1.0)) * normalize(cross(a, b)).z;
}
irr = abs(irr) / (2.0 * PI);

// Scale by BRDF amplitude and Fresnel term from second LUT
vec2 amp     = texture(ltc_amp, uv).rg;
vec3 specular = light_color * irr * (amp.x + (1.0 - F0) * amp.y);
```

**Diffuse term**: use the same polygon irradiance formula with `Minv = identity` (clamped cosine is already the correct diffuse kernel).

**Disk and sphere lights**: approximate the shape by a quad or use the sphere solid-angle formula; the LTC matrix encodes roughness response regardless of light shape.

**Use cases** — rectangle lights (windows, monitors, LEDs), disk lights (ceiling panels, headlights), tube lights (fluorescent strips); standard in Frostbite, Filament, Three.js, and Unity HDRP.  
**Reference** — [Heitz et al.: Real-Time Polygonal-Light Shading with Linearly Transformed Cosines (SIGGRAPH 2016)](https://eheitzresearch.wordpress.com/415-2/); [Eric Heitz's LTC demo and LUT generator](https://github.com/selfshadow/ltc_code)

---

### Capsule and Tube Area Lights

Analytical area light shading (§30 LTC for polygonal area lights) has a cheaper analytic solution for **line/tube** light sources: fluorescent tubes, neon signs, lightsabers, streetlights viewed close-up. The key observation is that the closest point on a line segment to the reflection ray can be computed analytically, enabling a specular highlight that correctly stretches into a streak along the tube. For diffuse, the irradiance from an infinite line has a closed-form solution; for a finite segment a simple integral approximation suffices.

**Shader stage**: evaluated in the **fragment shader** per light per lit pixel, or in a deferred lighting **compute** pass.

#### Specular — Representative Point Method (Karis 2013)
Find the point on the tube line segment closest to the reflected ray direction, clamp to the segment endpoints, use it as the light direction for specular evaluation. Correct the normalisation so energy is preserved as the tube radius decreases to a point.

```glsl
// Tube area light specular — representative point (UE4 method)
vec3 tube_light_specular(vec3 P, vec3 N, vec3 V, vec3 tube_start, vec3 tube_end,
                          float tube_radius, float roughness, vec3 light_color) {
    vec3  R         = reflect(-V, N);
    vec3  L0        = tube_start - P;
    vec3  L1        = tube_end   - P;
    vec3  Ld        = L1 - L0;
    float len2      = dot(Ld, Ld);

    // Closest point on segment to reflection ray
    float t = dot(R, L0) * dot(R, Ld) - dot(L0, Ld);
    t = t / (len2 - dot(R, Ld) * dot(R, Ld) + 1e-6);
    t = clamp(t, 0.0, 1.0);

    vec3  closest   = L0 + t * Ld;
    vec3  centre_to_ray = closest - dot(closest, R) * R;
    // Move to tube surface if within radius
    vec3  L_spec    = normalize(closest + centre_to_ray *
                        clamp(tube_radius / length(centre_to_ray + vec3(1e-6)), 0.0, 1.0));

    // Effective roughness: GGX roughness blown up by tube solid angle
    float alpha     = roughness * roughness;
    float sphere_angle = clamp(tube_radius / length(closest), 0.0, 1.0);
    float alpha_tube = clamp(alpha + sphere_angle / (2.0 * clamp(dot(N, L_spec), 0.01, 1.0)), 0.0, 1.0);

    float NdotL     = max(dot(N, L_spec), 0.0);
    return GGX_specular(N, V, L_spec, alpha_tube) * NdotL * light_color;
}
```

#### Diffuse — Segment Irradiance Integral
Integrate the Lambertian irradiance from a finite line segment. The closed-form integral of `max(0, dot(N, L(t)))` over a segment of uniform radiance reduces to an arctan formula.

```glsl
// Tube area light diffuse — closed-form line integral (Picott 1992)
vec3 tube_light_diffuse(vec3 P, vec3 N, vec3 tube_start, vec3 tube_end, vec3 light_color) {
    vec3  L0   = tube_start - P;
    vec3  L1   = tube_end   - P;
    float d0   = length(L0); L0 /= d0;
    float d1   = length(L1); L1 /= d1;
    float NdotL0 = dot(N, L0);
    float NdotL1 = dot(N, L1);
    float LdotL  = dot(L0, L1);

    // Irradiance from a line segment using the solid-angle subtended formula
    vec3  Ld   = normalize(L1 - L0);
    float sin_theta0 = sqrt(max(0.0, 1.0 - LdotL * LdotL));
    float irr  = (NdotL0 + NdotL1) / (d0 + d1 + length(tube_start - tube_end))
               * max(0.0, sin_theta0);

    return max(irr, 0.0) * light_color;
}
```

**Reference** — [Karis: Real Shading in Unreal Engine 4 (SIGGRAPH 2013)](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)

---

### Volumetric Light Shafts and God Rays

Volumetric light shafts — visible beams of sunlight through foliage, dust motes in a cathedral, fog illuminated by headlights — require computing how much light from a directional source reaches each point in the atmosphere before it reaches the camera. Two implementations exist at very different cost points.

**Shader stage**: screen-space god rays run as a full-screen **fragment or compute** post-process. Ray-marched volumetric shafts run as a **compute** pass producing a half-resolution volume texture that is upsampled and composited.

#### Screen-Space Radial Blur (Shaft Approximation)
Projects the sun position into screen space, then radially blurs the scene colour (or an occlusion mask) from each pixel toward the sun position. Fast and plausible for outdoor daylight; breaks for occluders not at screen centre and does not capture correct in-scattering physics.

```glsl
// Screen-space god ray — radial blur from sun screen position
layout(set=0, binding=0) uniform sampler2D scene_color;
layout(set=0, binding=1) uniform sampler2D occlusion_mask;  // 1 = sky, 0 = occluded

uniform vec2  sun_screen_pos;   // sun position in [0,1] UV
uniform float decay;            // attenuation per step (e.g. 0.97)
uniform float density;          // step scale (e.g. 0.9)
uniform float weight;           // light weight (e.g. 0.5)
uniform int   num_samples;      // typically 64–100

layout(location=0) out vec3 god_ray_out;

void main() {
    vec2  uv      = gl_FragCoord.xy / resolution;
    vec2  delta   = (uv - sun_screen_pos) * (density / float(num_samples));
    vec2  sample_uv = uv;
    float illum_decay = 1.0;
    vec3  accum   = vec3(0.0);

    for (int i = 0; i < num_samples; ++i) {
        sample_uv   -= delta;
        vec3  s      = texture(scene_color, sample_uv).rgb
                     * texture(occlusion_mask, sample_uv).r;
        accum       += s * illum_decay * weight;
        illum_decay *= decay;
    }
    god_ray_out = accum;
}
```

**Composite pass**: add the god ray buffer additively to the HDR scene before tone mapping.

#### Ray-Marched Volumetric Shafts
For each screen pixel, march a ray from the camera through the scene, sampling the light visibility at each step (shadow map lookup for the directional light) and accumulating in-scattered light using the Henyey-Greenstein phase function (§9). This is the physically correct approach used in UE4/UE5's volumetric atmosphere.

```glsl
// Volumetric shaft compute — one thread per pixel, half resolution
layout(set=0, binding=0) uniform sampler2DShadow shadow_csm;
layout(set=0, binding=1, rgba16f) uniform image2D shaft_out;

uniform mat4  light_proj_view;
uniform vec3  light_dir;
uniform vec3  light_color;
uniform float g;              // HG asymmetry (e.g. 0.7 for forward scatter)
uniform float extinction;     // σ_t (density × extinction coeff)
uniform int   num_steps;      // 16–32 for half-res

float hg_phase(float cos_theta, float g_val) {
    float g2 = g_val * g_val;
    return (1.0 - g2) / (4.0 * 3.14159 * pow(1.0 + g2 - 2.0*g_val*cos_theta, 1.5));
}

void main() {
    ivec2 px      = ivec2(gl_GlobalInvocationID.xy);
    vec3  ray_dir = reconstruct_ray_dir(px);
    float cos_theta = dot(ray_dir, -light_dir);
    float phase   = hg_phase(cos_theta, g);

    vec3  accum   = vec3(0.0);
    float transmit = 1.0;

    for (int i = 0; i < num_steps; ++i) {
        float t   = scene_depth * float(i) / float(num_steps);
        vec3  pos = cam_pos + ray_dir * t;

        // Shadow map visibility at this position
        vec4  lclip  = light_proj_view * vec4(pos, 1.0);
        vec3  lndc   = lclip.xyz / lclip.w;
        float shadow = texture(shadow_csm, vec3(lndc.xy*0.5+0.5, lndc.z));

        float step_ext = extinction * (scene_depth / float(num_steps));
        accum    += transmit * shadow * phase * light_color * step_ext;
        transmit *= exp(-step_ext);
    }
    imageStore(shaft_out, px, vec4(accum, 1.0 - transmit));
}
```

**Use cases** — outdoor sunlight shafts through trees, interior cathedral beams, vehicle headlights in fog, spot-lit stage haze.  
**Reference** — [Hoffman & Preetham: Rendering Outdoor Light Scattering in Real Time (GDC 2002)](https://www.terathink.com/lengyel/GPUGems2/GPUGems2_ch16.pdf); [Crassin: Cascaded Light Propagation Volumes (I3D 2011)](https://research.nvidia.com/publication/cascaded-light-propagation-volumes-real-time-indirect-illumination)

---

### Caustics

Caustics are the bright concentrated patterns of light formed when light refracts through a transparent medium (glass, water) or reflects off a curved specular surface and focuses onto a diffuse receiver. They are absent from standard rasterization pipelines (which cannot trace refracted paths) and require dedicated techniques.

**Shader stage**: photon splatting uses a **vertex shader** to project photon hit positions into screen space and a **fragment shader** to blend the photon contribution. Screen-space caustics use a **compute or fragment shader** ray-marching from the surface. RT caustics use a `rchit` shader accumulating onto a caustic buffer.

#### Photon Splatting (GPU Caustic Map)
Trace photons from the light through the refracting surface (CPU or compute), recording where each photon hits the receiver surface. Splat each photon hit as an additive point sprite or small Gaussian into a caustic irradiance map aligned to the receiver, then apply the caustic map as an additive light contribution during shading.

```glsl
// Photon splat vertex shader — one vertex per photon hit point
layout(location=0) in vec3 photon_world_pos;   // where photon hit the diffuse receiver
layout(location=1) in vec3 photon_flux;         // RGB power of this photon

layout(location=0) out vec3 v_flux;
layout(location=1) out vec2 v_offset;           // offset within point sprite

uniform mat4  proj_view;          // caustic map projection (light POV or world-aligned)
uniform float splat_radius;       // Gaussian radius in caustic map texels

void main() {
    gl_Position  = proj_view * vec4(photon_world_pos, 1.0);
    gl_PointSize = splat_radius * 2.0;
    v_flux       = photon_flux;
    v_offset     = vec2(0.0);  // filled by rasterizer for point sprite
}
```

```glsl
// Photon splat fragment shader — additive Gaussian splat
layout(location=0) in vec3 v_flux;
layout(location=0) out vec4 caustic_out;  // additive blend enabled

void main() {
    vec2  d = gl_PointCoord - 0.5;
    float r = dot(d, d) * 4.0;
    float w = exp(-r * 2.0);               // Gaussian kernel
    caustic_out = vec4(v_flux * w, 1.0);  // additive — accumulate photon contributions
}
```

**Apply caustic map in deferred lighting**: read the caustic irradiance map at the surface UV and add to the diffuse component.

#### Screen-Space Caustics (Wyman 2005)
For underwater scenes: each water surface fragment traces a refracted ray to find where it would hit the receiver. Splat the estimated flux concentration (inversely proportional to the refracted beam divergence) into a screen-space caustic buffer.

```glsl
// Screen-space caustic accumulation — fragment shader on the water surface
uniform vec3  light_dir;
uniform float ior;   // index of refraction (water ≈ 1.333)
layout(location=0) in vec3 world_pos;
layout(location=1) in vec3 world_normal;

layout(location=0) out vec4 caustic_contribution;  // additive

void main() {
    vec3 refracted = refract(-light_dir, world_normal, 1.0 / ior);
    // Trace to receiver plane (y = receiver_y)
    float t   = (receiver_y - world_pos.y) / refracted.y;
    vec3  hit = world_pos + t * refracted;

    // Estimate flux concentration from Jacobian of refraction map
    vec3 N_dx  = dFdx(world_normal);
    vec3 N_dy  = dFdy(world_normal);
    float jacobian = abs(1.0 / (1.0 + dot(N_dx, N_dx) + dot(N_dy, N_dy)));

    // Project hit to screen and write additive contribution
    vec4 hit_clip = proj_view * vec4(hit, 1.0);
    gl_FragDepth  = hit_clip.z / hit_clip.w;
    caustic_contribution = vec4(light_color * jacobian * 0.1, 1.0);
}
```

**RT Caustics**: with `VK_KHR_ray_tracing_pipeline`, trace refracted/reflected paths from light sources; accumulate hit radiance into a caustic buffer; denoise with SVGF. Correct but expensive.

**Use cases** — underwater swimming pool caustics, wine glass shadows, gemstone sparkle, aquarium walls.  
**Reference** — [Wyman: Interactive Image-Space Caustics (Eurographics 2005)](https://www.cs.uiowa.edu/~cwyman/publications/files/caustics2005.pdf); [Jensen: A Practical Guide to Global Illumination using Photon Mapping (SIGGRAPH 2002 Course)](https://graphics.ucsd.edu/~henrik/papers/book/)

---


---

## III. Shadows and Occlusion

Shadows and ambient occlusion share a common computational problem: determining how much of the upper hemisphere above a surface point is blocked by other geometry. This category covers the major shadow map variants — basic shadow maps, cascaded shadow maps for sun lights, omnidirectional cube maps for point lights, the atlas and caching strategies needed for many shadow-casting lights, and the Percentage-Closer Soft Shadows (PCSS) technique for contact-hardening penumbrae — together with the screen-space (SSAO) and baked (bent normal, AO maps) approaches to ambient occlusion.

### Shadow Rendering Techniques

Shadows are the primary visual cue for spatial relationships between objects. The field divides into two concerns: correctness (does a fragment lie in shadow?) and softness (how wide is the penumbra?). Shadow mapping answers the correctness question by depth-testing from the light's viewpoint; every technique that follows is an elaboration of this idea — PCF and PCSS add softness, VSM/MSM add filterability, CSM adds resolution scalability for directional lights, and Virtual Shadow Maps add on-demand paging for scenes with many local lights. Ray-traced shadows are the ground truth but require a denoiser for acceptable sample counts.

**Shader stage**: Shadow map generation is a **vertex shader** + depth-only pass (no fragment shader needed for depth-only). Shadow lookup and PCF filtering happen in the **fragment shader** of the main lighting pass. VSM/MSM blur passes run as **compute shaders** or full-screen **fragment shaders**. PCSS blockers search also runs in the **fragment shader**. Ray-traced shadows use the **ray generation** and **miss shaders**.

#### Shadow Mapping
Renders the scene depth from the light's perspective into a depth texture; in the main pass, compares each fragment's light-space depth against the stored value.

**Use cases** — the baseline shadow algorithm for all real-time rendering.  
**Key variants** — omnidirectional (cube shadow map); cascaded (CSM); perspective warping (PSM, LiSPSM).  
**Limitations** — aliasing (self-shadowing acne, Peter Panning); resolution limits maximum shadow quality; perspective aliasing requires cascades.  
**Reference** — [GPU Gems Ch11: Shadow Map Antialiasing](https://developer.nvidia.com/gpugems/gpugems/part-ii-lighting-and-shadows/chapter-11-shadow-map-antialiasing)

#### PCF (Percentage-Closer Filtering)
Filters shadow visibility (not depth values) across a neighbourhood of shadow map texels to produce soft shadow edges.

**Use cases** — the standard soft shadow technique for directional and spot lights in all real-time renderers.  
**Key variants** — fixed-width kernel; Poisson disk kernel; random rotated kernel; PCSS-driven variable width.  
**Limitations** — does not produce distance-correct penumbrae (light size is not modelled); wider kernels are expensive.  
**Reference** — [Reeves et al. (1987) SIGGRAPH](https://doi.org/10.1145/37402.37429)

#### PCSS (Percentage-Closer Soft Shadows)
Estimates the correct penumbra width at each pixel by first finding the average blocker depth (blocker search pass), then using that to drive a variable-width PCF kernel.

**Use cases** — physically plausible soft shadows that harden near the caster and soften with distance; games, film, archviz.  
**Limitations** — 2× the shadow map sample cost of PCF; blocker search requires a larger kernel than the final PCF; noisy at low sample counts.  
**Reference** — [Fernando: Percentage-Closer Soft Shadows (GDC 2005)](https://developer.download.nvidia.com/shaderlibrary/docs/shadow_PCSS.pdf)

#### VSM (Variance Shadow Maps)
Stores the mean and mean-squared depth in the shadow map, then applies Chebyshev's inequality to compute an upper bound on the shadow probability without per-sample depth tests.

**Use cases** — soft shadows that can be filtered/blurred (unlike PCF); real-time blur via separable Gaussian; hardware bilinear.  
**Limitations** — light bleeding (incorrect bright regions in shadow) when multiple occluders are at different depths; not suitable for transparent shadow casters.  
**Reference** — [Donnelly & Lauritzen (2006): VSM](https://jankautz.com/publications/vsm_gi06.pdf)

#### MSM (Moment Shadow Maps)
Extends VSM to four statistical moments of the depth distribution (using the Hamburger recurrence), reducing light bleeding while maintaining filterability.

**Use cases** — high-quality soft shadows where VSM light bleeding is unacceptable; production rendering.  
**Key variants** — 4-moment (balanced quality/cost); 6-moment (higher quality, more storage).  
**Reference** — [Hamburger et al.: Moment Shadow Maps (JCGT 2015)](https://jcgt.org/published/0004/05/01/)

#### Cascaded Shadow Maps (CSM)
Partitions the view frustum into depth slices, rendering a separate shadow map for each slice at a resolution matched to the slice's angular subtended area.

**Use cases** — directional lights (sun/moon) in outdoor scenes; the universal solution to shadow map perspective aliasing.  
**Key variants** — logarithmic split (more cascades near camera); practical split (blend of log and uniform); PSSM (Parallel-Split Shadow Maps).  
**Limitations** — cascade count × shadow map render cost; blend bands at cascade boundaries; moving sun requires per-frame re-render.  
**Reference** — [GPU Gems 3 Ch10: Parallel-Split Shadow Maps](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-10-parallel-split-shadow-maps-programmable-gpus)

#### Virtual Shadow Maps (VSMs)
Implements a massive-resolution clipmap (e.g. 16k×16k) backed by on-demand physical pages, rendering only shadow map pages that overlap the camera's visible geometry.

**Use cases** — Unreal Engine 5's shadow solution; eliminates cascade count choice; handles large scenes with many local lights.  
**Limitations** — requires a page table and physical page pool management; first-frame miss cost when camera moves.  
**Reference** — [Unreal Engine 5: Virtual Shadow Maps Documentation](https://docs.unrealengine.com/5.0/en-US/virtual-shadow-maps-in-unreal-engine/)

---

### Cascaded Shadow Maps

A single shadow map for a directional light must cover the entire view frustum, leading to coarse depth resolution at close range and wasted resolution far away. **Cascaded Shadow Maps (CSM)** partition the view frustum into N depth slices and render a separate shadow map per slice, each with its own light-space projection tuned to cover only that slice. The result is high shadow resolution near the camera (sub-pixel accuracy on nearby geometry) that degrades gracefully at distance.

**Shader stage**: shadow map rendering uses the **vertex/fragment shader** (N draw calls or geometry shader with `gl_Layer`). Shadow lookup runs in the **fragment shader** of the lighting pass, selecting the correct cascade by the fragment's view-space depth.

#### Cascade Split Calculation
The standard split scheme is a blend of logarithmic and uniform splits (Parallel-Split Shadow Maps, Zhang 2006):

```glsl
// CPU-side cascade split calculation
void compute_cascade_splits(float near, float far, int n, float lambda,
                             float out_splits[]) {
    // lambda = 0: uniform, lambda = 1: logarithmic
    for (int i = 0; i < n; ++i) {
        float uniform_split = near + (far - near) * float(i + 1) / float(n);
        float log_split     = near * pow(far / near, float(i + 1) / float(n));
        out_splits[i]       = mix(uniform_split, log_split, lambda);
    }
}
// Typical: n=4, lambda=0.75, near=0.1, far=500.0
```

#### Per-Cascade Light-Space Matrix
For each cascade, compute a tight orthographic projection that fits the cascade's frustum slice viewed from the light direction. Snap to texel boundaries to prevent shimmering when the camera moves.

```c
// Per-cascade tight orthographic projection (CPU, repeated n times)
// cascade_frustum_corners[8]: world-space corners of the cascade frustum slice
vec3 light_space_corners[8];
for (int i = 0; i < 8; ++i)
    light_space_corners[i] = vec3(light_view * vec4(cascade_frustum_corners[i], 1.0));

// AABB of corners in light space
vec3 mn = light_space_corners[0], mx = light_space_corners[0];
for (int i = 1; i < 8; ++i) {
    mn = min(mn, light_space_corners[i]);
    mx = max(mx, light_space_corners[i]);
}

// Snap min/max to shadow-map texel grid (prevents shimmer)
float texel_size = (mx.x - mn.x) / shadow_map_resolution;
mn.x = floor(mn.x / texel_size) * texel_size;
mn.y = floor(mn.y / texel_size) * texel_size;
mx.x =  ceil(mx.x / texel_size) * texel_size;
mx.y =  ceil(mx.y / texel_size) * texel_size;

mat4 cascade_ortho = ortho(mn.x, mx.x, mn.y, mx.y, mn.z - pull_back, mx.z);
mat4 cascade_pv    = cascade_ortho * light_view;
```

#### Cascade Selection and Blend in Fragment Shader
```glsl
// CSM lookup — fragment shader (4 cascades, stored in a texture array)
layout(set=1, binding=0) uniform sampler2DArrayShadow csm_shadow;

uniform mat4  cascade_pv[4];
uniform float cascade_splits[4];   // view-space depth at each split
uniform float blend_range;         // blend band width (e.g. 0.05 * split distance)

float csm_shadow_factor(vec3 world_pos, float view_depth) {
    // Select cascade index
    int  cascade = 3;
    for (int i = 0; i < 4; ++i) {
        if (view_depth < cascade_splits[i]) { cascade = i; break; }
    }

    // Shadow lookup in selected cascade
    vec4  lclip = cascade_pv[cascade] * vec4(world_pos, 1.0);
    vec3  lndc  = lclip.xyz / lclip.w;
    vec3  suv   = vec3(lndc.xy * 0.5 + 0.5, float(cascade));
    float shadow = texture(csm_shadow, vec4(suv, lndc.z - 0.001));

    // Blend at cascade boundary into next cascade
    if (cascade < 3) {
        float blend_t = smoothstep(cascade_splits[cascade] - blend_range,
                                   cascade_splits[cascade], view_depth);
        if (blend_t > 0.0) {
            vec4  lclip2 = cascade_pv[cascade + 1] * vec4(world_pos, 1.0);
            vec3  lndc2  = lclip2.xyz / lclip2.w;
            vec3  suv2   = vec3(lndc2.xy * 0.5 + 0.5, float(cascade + 1));
            float shadow2 = texture(csm_shadow, vec4(suv2, lndc2.z - 0.001));
            shadow = mix(shadow, shadow2, blend_t);
        }
    }
    return shadow;
}
```

**Use cases** — every outdoor/open-world game; all real-time directional-light shadow systems; combined with PCSS (§85) per-cascade for contact-hardening outdoor shadows.  
**Reference** — [Zhang et al.: Parallel-Split Shadow Maps for Large-Scale Virtual Environments (VRCIA 2006)](https://dl.acm.org/doi/10.1145/1128923.1128975); [Engel: ShaderX6: Cascaded Shadow Maps (2007)](https://www.shaderx6.com/)

---

### Omnidirectional Shadow Maps

Point lights cast shadows in all directions — a spotlight's cone-bounded shadow map does not apply. The standard solution is a **cube shadow map**: render the scene six times from the light's position into the six faces of a cube texture, then during shading sample the cube map with the light-to-fragment vector to retrieve the stored depth and compare. A cheaper alternative is **dual paraboloid shadow mapping** (DPSM), which projects the scene onto two paraboloid surfaces (front/back hemisphere) in a single geometry pass using a mesh shader.

**Shader stage**: shadow map generation uses the standard **vertex + fragment** pipeline (or geometry shader for single-pass cube rendering). Shadow sampling runs in the **fragment shader** of the lighting pass.

#### Cube Shadow Map
Renders the scene six times with projection matrices pointing along ±X, ±Y, ±Z from the light position. Stores linear depth `length(frag_pos - light_pos)` (not projected NDC depth) to avoid distortion artefacts at cube face edges.

```glsl
// Shadow cube map generation — fragment shader (one face per render pass)
layout(location=0) in vec3 frag_world_pos;
layout(location=0) out float out_depth;

uniform vec3  light_pos;
uniform float far_plane;

void main() {
    out_depth = length(frag_world_pos - light_pos) / far_plane;  // [0, 1]
}
```

```glsl
// Shadow cube map sampling — deferred lighting fragment shader
layout(set=1, binding=0) uniform samplerCube shadow_cube;

float point_shadow(vec3 frag_pos, vec3 light_pos, float far_plane) {
    vec3  L    = frag_pos - light_pos;
    float dist = length(L) / far_plane;
    float closest = texture(shadow_cube, L).r;  // samplerCube auto-selects face
    return (dist - 0.005 > closest) ? 0.0 : 1.0;  // 0.005 = bias
}
```

**PCF for cube maps**: sample `textureCubeShadow` with a `samplerCubeShadow` for hardware PCF support, or manually sample neighbouring cube texels and average.

```glsl
layout(set=1, binding=0) uniform samplerCubeShadow shadow_cube_pcf;

float point_shadow_pcf(vec3 L, float compare_depth) {
    // Hardware PCF: returns interpolated result
    return texture(shadow_cube_pcf, vec4(L, compare_depth)).r;
}
```

**Use cases** — point lights (street lights, torches, explosions); any omni-directional shadow caster. Single-face rendering per frame (rotating through faces over multiple frames) can amortize the 6× cost for static scenes.  
**Reference** — [de Vries: LearnOpenGL — Point Shadows](https://learnopengl.com/Advanced-Lighting/Shadows/Point-Shadows)

#### Dual Paraboloid Shadow Mapping (DPSM)
Projects the scene onto two paraboloid surfaces (forward and backward hemispheres) using a non-linear vertex transform, producing two 2D shadow maps instead of six. A geometry shader or mesh shader routes each triangle to the correct hemisphere. Shadow lookup transforms the light-to-fragment vector to paraboloid UV space.

```glsl
// DPSM vertex transform — forward hemisphere (back hemisphere uses -pos.z)
layout(location=0) in vec3 in_pos;
uniform mat4 model;
uniform vec3 light_pos;
uniform float near, far;

void main() {
    vec3 P = (model * vec4(in_pos, 1.0)).xyz - light_pos;
    float L = length(P);
    P /= L;  // normalize
    // Paraboloid projection: map hemisphere to [-1,1]² disk
    float k = P.z + 1.0;
    gl_Position = vec4(P.x / k, P.y / k, (L - near) / (far - near), 1.0);
}
```

**Use cases** — point lights with tight shadow budget; mobile/VR where six render passes are too expensive.  
**Limitations** — triangles crossing the paraboloid boundary must be clipped (handled by the geometry shader); lower quality than cube maps at face boundaries.  
**Reference** — [Brabec et al.: Single Sample Soft Shadows Using Depth Maps (VMV 2002)](https://cg.cs.uni-bonn.de/publication/brabec-2002-single/); [GPU Gems Ch12: Omnidirectional Shadow Mapping](https://developer.nvidia.com/gpugems/gpugems/part-ii-lighting-and-shadows/chapter-12-omnidirectional-shadow-mapping)

---

### PCSS and Contact-Hardening Shadows

Percentage Closer Soft Shadows (PCSS) produces shadows with physically correct **contact hardening**: penumbrae widen as the receiver moves farther from the occluder, and shadows are sharp at contact points. Standard PCF uses a fixed filter kernel; PCSS adapts the kernel width per-pixel by first searching for the average blocker depth in a region around the receiver, then using the blocker-to-receiver ratio to compute the penumbra width. This two-phase approach (blocker search then PCF) enables plausible area light shadows at moderate cost.

**Shader stage**: runs in the **fragment shader** (or deferred lighting compute) once per shadow-casting light.

#### Phase 1 — Blocker Search
Sample a disc region around the projected receiver position in the shadow map and compute the average depth of texels that are in front of the receiver (i.e., are potential occluders).

```glsl
// PCSS blocker search — average occluder depth in search radius
layout(set=1, binding=0) uniform sampler2D shadow_map;

uniform float light_size_uv;    // light radius in shadow-map UV space (e.g. 0.05)
uniform int   blocker_samples;  // e.g. 16

float find_blocker_distance(vec2 uv, float z_recv, float search_radius) {
    float blocker_sum   = 0.0;
    int   blocker_count = 0;

    for (int i = 0; i < blocker_samples; ++i) {
        vec2  offset    = poisson_disk_16[i] * search_radius;
        float z_occ     = texture(shadow_map, uv + offset).r;
        if (z_occ < z_recv) {
            blocker_sum += z_occ;
            ++blocker_count;
        }
    }
    return (blocker_count > 0) ? blocker_sum / float(blocker_count) : -1.0;
}
```

#### Phase 2 — Variable-Width PCF
Use the penumbra width derived from the blocker distance to scale the PCF filter kernel.

```glsl
// PCSS full evaluation
uniform int   pcf_samples;      // e.g. 32
uniform float near_plane;       // light camera near plane (for depth linearisation)

float pcss_shadow(vec3 world_pos, mat4 light_pv) {
    vec4  lclip    = light_pv * vec4(world_pos, 1.0);
    vec3  lndc     = lclip.xyz / lclip.w;
    vec2  uv       = lndc.xy * 0.5 + 0.5;
    float z_recv   = lndc.z;

    // --- Phase 1: blocker search ---
    float search_r = light_size_uv * (z_recv - near_plane) / z_recv;
    float z_avg    = find_blocker_distance(uv, z_recv, search_r);
    if (z_avg < 0.0) return 1.0;   // no blockers → fully lit

    // --- Penumbra width (thin lens formula) ---
    float penumbra = (z_recv - z_avg) / z_avg * light_size_uv;

    // --- Phase 2: PCF with variable kernel ---
    float shadow = 0.0;
    for (int i = 0; i < pcf_samples; ++i) {
        vec2  offset = poisson_disk_32[i] * penumbra;
        shadow += texture(shadow_map, vec3(uv + offset, z_recv - 0.001)).r;
    }
    return shadow / float(pcf_samples);
}
```

#### PCSS with Blue-Noise Rotation
Rotate the Poisson disc pattern per-pixel using a blue-noise angle to remove structured banding artifacts, then denoise with TAA.

```glsl
float angle   = texture(blue_noise_tex, gl_FragCoord.xy / 128.0).r * 6.2832;
float c = cos(angle), s = sin(angle);
mat2  rot = mat2(c, s, -s, c);
// Rotate each sample offset: offset = rot * poisson_disk_32[i] * penumbra
```

**Reference** — [Fernando: Percentage-Closer Soft Shadows (GDC 2005)](https://developer.download.nvidia.com/shaderlibrary/docs/shadow_PCSS.pdf); [Jimenez: Practical Real-Time Strategies for Accurate Indirect Occlusion (SIGGRAPH 2016)](https://iryoku.com/groundtruth-based-mr-ao/)

---

### Exponential Shadow Maps

Variance Shadow Maps (VSM, §4) filter the shadow map with a Gaussian blur and reconstruct a probability-of-being-lit using Chebyshev's inequality. A known failure mode of VSM is **light bleeding**: when a fully-lit receiver near a shadowed one receives incorrect bright light due to the variance inequality's looseness. **Exponential Shadow Maps (ESM)** avoid this by storing `exp(c · depth)` in the shadow map instead of raw depth, enabling exact multiplicative compositing without the Chebyshev approximation. **EVSM** (Exponential Variance Shadow Maps) combines ESM and VSM for both filtering quality and reduced light bleeding.

**Shader stage**: ESM generation uses the **fragment shader** during the shadow pass (writes `exp(c · depth)`). ESM lookup uses the **fragment shader** of the lighting pass (compares via division/clamping rather than depth compare).

#### ESM Generation and Lookup
Store `exp(c · z_linear)` where `c` is a positive constant (typically 40–100; larger = harder shadow edge, less light bleeding but more precision issues).

```glsl
// ESM shadow map generation — fragment shader in shadow pass
uniform float esm_c;       // exponent constant, typically 80.0

layout(location=0) out float esm_depth;

void main() {
    float z_linear = gl_FragCoord.z;   // or linearise from NDC if needed
    esm_depth = exp(esm_c * z_linear);
}
```

```glsl
// ESM shadow map lookup — lighting fragment shader
layout(set=1, binding=0) uniform sampler2D esm_map;   // pre-blurred with Gaussian

float esm_shadow(vec3 world_pos, mat4 light_proj_view, float esm_c) {
    vec4  lclip  = light_proj_view * vec4(world_pos, 1.0);
    vec3  lndc   = lclip.xyz / lclip.w;
    vec2  uv     = lndc.xy * 0.5 + 0.5;
    float z_recv = lndc.z;   // depth of receiver in light space

    float esm_occluder = texture(esm_map, uv).r;  // exp(c * z_occluder), blurred
    // Receiver is lit if exp(c * z_recv) ≤ exp(c * z_occluder)
    // Visibility = clamp(exp(c * (z_occluder - z_recv)), 0, 1)
    float visibility = clamp(esm_occluder * exp(-esm_c * z_recv), 0.0, 1.0);
    return visibility;
}
```

**Key property**: because `exp(a) · exp(b) = exp(a+b)`, ESM shadow maps can be convolved with a Gaussian using standard texture filtering hardware — no PCF needed, full hardware bilinear works correctly.

#### EVSM — Exponential Variance Shadow Maps
Stores four moments: `(exp(c·z), exp(2c·z), -exp(-c·z), exp(-2c·z))` in an RGBA16F shadow map. The positive exponent handles standard receiver depths; the negative exponent handles receivers behind the occluder (reducing light bleeding). Chebyshev's inequality is applied separately to the positive and negative exponent distributions, and the minimum is taken.

```glsl
// EVSM moment generation — fragment shader
layout(location=0) out vec4 evsm_moments;
uniform float pos_c;   // positive exponent, e.g. 40.0
uniform float neg_c;   // negative exponent, e.g. 40.0

void main() {
    float z  = gl_FragCoord.z;
    float ep = exp( pos_c * z);
    float en = exp(-neg_c * z);
    evsm_moments = vec4(ep, ep*ep, en, en*en);
}
```

**Reference** — [Lauritzen: Rendering Antialiased Shadows with Variance Shadow Maps (GDC 2008)](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-8-summed-area-variance-shadow-maps); [Salvi: Rendering Filtered Shadows with Exponential Shadow Maps (ShaderX6, 2008)](https://dl.acm.org/doi/10.5555/1407073.1407086)

---

### Shadow Atlas and Cached Shadows

Individual shadow maps for point, spot, and directional lights are described in §4 and §47. In a scene with dozens of shadow-casting lights, allocating a separate framebuffer per light is impractical. A **shadow atlas** packs all lights' shadow maps into a single large texture, with a region allocated per light proportional to its screen-space influence. **Shadow caching** avoids re-rendering shadow faces whose content (static geometry only) has not changed, dramatically reducing shadow rendering cost.

**Shader stage**: the shadow atlas sampler lookup uses standard `texture(shadow_atlas, uv)` with per-light UV transform. Atlas management and cache invalidation run on the **CPU** or in **compute** (dirty-flag evaluation, region assignment).

#### Shadow Atlas Layout
A `4096×4096` or `8192×8192` depth texture subdivided into tiles. Each light is assigned a tile whose size is proportional to the light's projected screen area (larger tiles for nearby lights, smaller for distant). The assignment is recalculated each frame as lights move or appear.

```glsl
// Per-light atlas metadata (in a SSBO)
struct ShadowAtlasEntry {
    vec4  uv_rect;       // (x, y, width, height) in [0,1] atlas UV space
    mat4  proj_view;     // light's shadow projection-view matrix
    float near, far;     // for linear depth reconstruction
    uint  is_cached;     // 1 if static-only content is still valid
};
layout(set=0, binding=0) readonly buffer AtlasSSBO { ShadowAtlasEntry entries[]; };
layout(set=0, binding=1) uniform sampler2DShadow shadow_atlas;

// Shadow lookup for light i
float atlas_shadow(uint light_idx, vec3 world_pos) {
    ShadowAtlasEntry e = entries[light_idx];
    vec4  light_clip   = e.proj_view * vec4(world_pos, 1.0);
    vec3  ndc          = light_clip.xyz / light_clip.w;
    vec2  atlas_uv     = e.uv_rect.xy + (ndc.xy * 0.5 + 0.5) * e.uv_rect.zw;
    float compare      = ndc.z;
    return texture(shadow_atlas, vec3(atlas_uv, compare));  // hardware PCF
}
```

#### Shadow Map Caching (Static Geometry)
Shadow maps containing only static geometry can be rendered once and cached indefinitely, re-rendered only when:
- A dynamic object enters the light's view frustum (invalidates the affected region).
- The light parameters (position, angle, range) change.
- Static geometry is modified (level streaming, destruction).

The cache maintains a dirty flag per atlas tile. Each frame, only dirty tiles are rendered. Static tiles are rendered in a separate, persistent shadow pass; dynamic objects are overlaid in a second pass.

```glsl
// Two-pass cached shadow rendering:
// Pass 1 (conditional): render static geometry into cached tile if dirty
//   → clear dirty flag, retain tile content across frames
// Pass 2 (always):      render dynamic objects into a temporary tile
//   → composited with the cached static tile during shadow lookup
//   result: shadow = static_shadow * dynamic_shadow

float cached_shadow_lookup(uint light_idx, vec3 world_pos) {
    float s_static  = atlas_shadow_layer(light_idx, world_pos, LAYER_STATIC);
    float s_dynamic = atlas_shadow_layer(light_idx, world_pos, LAYER_DYNAMIC);
    return min(s_static, s_dynamic);  // both must be unshadowed to be lit
}
```

#### Atlas Region Assignment Heuristics
- **Screen-space footprint**: compute the projected area of the light's influence sphere in screen pixels; assign a tile of proportional size (e.g., 512² for lights covering > 30% of screen, 128² for < 5%).
- **LRU eviction**: when the atlas is full and a new light requests a tile, evict the least-recently-used tile.
- **Stable assignment**: avoid reassigning tiles across frames (causes one-frame shadow pop); prefer keeping the same tile between frames unless the light has moved significantly.

**Use cases** — Frostbite many-light shadow atlas, id Tech 7 cached shadows, Unity HDRP shadow atlas; used whenever more than 4 lights cast shadows simultaneously.  
**Reference** — [Valient: Rendering of COD:AW (SIGGRAPH 2014)](https://advances.realtimerendering.com/s2014/); [Ubisoft: Cascaded Shadow Map Caching in For Honor (GDC 2018)](https://advances.realtimerendering.com/s2018/)

---

### Ambient Occlusion

Ambient occlusion approximates how much of the sky hemisphere is blocked by surrounding geometry at each surface point, darkening crevices and contact zones where indirect light cannot reach. It is not physically exact — true AO integrates over all light directions — but it is perceptually powerful: the human visual system strongly associates darkened concavities with spatial depth. The progression in this section runs from the cheapest screen-space approximations (SSAO, HBAO) through physically motivated formulations (GTAO, which also produces bent normals for correct IBL lookup) to the reference-quality ray-traced path (RTAO). Bent normals, produced as a byproduct of GTAO, feed back into §7 (Image-Based Lighting) to improve diffuse IBL accuracy.

**Shader stage**: All screen-space AO techniques (SSAO, HBAO, GTAO) run as full-screen **compute shaders** or full-screen **fragment shaders** reading from the depth buffer and G-buffer normals. RTAO uses the **ray generation shader** to cast hemisphere rays and the **miss shader** (no hit = unoccluded). A separate **compute shader** blur/denoise pass is always required after the raw AO signal.

#### SSAO (Screen-Space Ambient Occlusion)
Estimates local occlusion from nearby geometry by sampling the depth buffer in a hemisphere around each fragment and counting how many samples fall inside geometry.

**Use cases** — the baseline real-time AO technique; available in virtually every modern engine.  
**Key variants** — Alchemy AO / SAO (McGuire 2012): spatially-aware sample count scaling; SSAO+ (NVIDIA): higher quality sampling pattern.  
**Limitations** — only sees on-screen geometry; radius parameter causes halos; misses large-scale occlusion.  
**Reference** — [McGuire et al.: Scalable Ambient Occlusion (HPG 2012)](https://jcgt.org/published/0001/01/01/)

#### HBAO / HBAO+
Horizon-Based Ambient Occlusion samples depth along multiple directions in screen space and integrates the horizon angle to estimate the visible solid angle, producing a physically motivated AO approximation.

**Use cases** — higher-quality AO than SSAO at similar cost; used in NVIDIA GameWorks.  
**Limitations** — requires depth + normal buffers; tangent-plane approximation breaks on thin surfaces; proprietary implementation in HBAO+.  
**Reference** — [Bavoil & Sainz: Screen Space Ambient Occlusion (NVIDIA 2008)](https://developer.download.nvidia.com/SDK/10.5/direct3d/Source/ScreenSpaceAO/doc/ScreenSpaceAO.pdf)

#### GTAO (Ground Truth Ambient Occlusion)
Integrates the ambient occlusion analytically over the hemisphere using the full bent-normal and visibility function, producing significantly more accurate results than SSAO or HBAO.

**Use cases** — modern engines (Unreal 5, Unity HDRP); also produces bent normals as a byproduct for directional IBL lookup.  
**Key variants** — Jimenez (2016) original; optimised GTAO in XeGTAO (Intel, open source).  
**Reference** — [Jimenez et al.: GTAO (SIGGRAPH 2016)](https://blog.selfshadow.com/publications/s2016-shading-course/) | [XeGTAO](https://github.com/GameTechDev/XeGTAO)

#### RTAO (Ray Traced Ambient Occlusion)
Casts one or more shadow rays per pixel against the full scene geometry using hardware ray tracing, accumulates over time using temporal filtering.

**Use cases** — highest-quality AO for ray-tracing–capable hardware; reference for comparing screen-space methods.  
**Limitations** — even one ray per pixel requires a denoiser (SVGF, ReLAX) for smooth results; latency of denoising pipeline.  
**Reference** — [NVIDIA: Ray Traced Ambient Occlusion](https://developer.nvidia.com/rtx/raytracing)

#### Bent Normals
The average direction of unoccluded sky hemisphere, computed per fragment as a byproduct of AO; used to fetch the correct irradiance direction from an IBL environment map rather than the geometric normal.

**Use cases** — improves IBL diffuse quality in crevices; reduces over-darkening artefact from combining AO scalar with IBL.  
**Reference** — [Jimenez et al.: GTAO (SIGGRAPH 2016)](https://blog.selfshadow.com/publications/s2016-shading-course/)

---

### Bent Normal and AO Baking

Real-time RTAO (§5) and SSAO compute per-frame approximate AO; for static geometry the same result can be baked offline into UV-mapped textures, giving exact occlusion at zero runtime cost. **Bent normals** extend this: instead of just storing a scalar AO factor, they store the average unoccluded hemisphere direction — the direction from which the most light arrives unobstructed. The bent normal replaces the geometric normal for diffuse IBL lookups, giving correct ambient shadowing including directional bias toward open sky.

**Shader stage**: baking runs as a **compute shader** dispatched over UV-space texels of the target texture, sampling rays from the mesh surface reconstructed from the UV layout.

#### AO Baking via Hemisphere Ray Sampling
For each texel, reconstruct the world-space surface position and normal from the UV parameterisation (using the same chart-to-world transform used during lightmapping). Then trace N cosine-weighted rays and count the fraction that do not intersect the mesh within a maximum distance.

```glsl
// AO bake compute shader — one thread per UV texel
layout(local_size_x=8, local_size_y=8) in;
layout(set=0, binding=0) uniform accelerationStructureEXT tlas;
layout(set=0, binding=1, r16f) uniform image2D ao_out;
layout(set=0, binding=2, rgba16f) uniform image2D bent_normal_out;

layout(set=0, binding=3) uniform sampler2D position_atlas;   // world-space XYZ
layout(set=0, binding=4) uniform sampler2D normal_atlas;     // world-space normal
layout(set=0, binding=5) uniform sampler2D blue_noise;

uniform int   num_samples;    // typically 64–256
uniform float max_distance;   // AO ray max length (e.g. 0.5 m)

layout(location=0) rayPayloadEXT bool ray_hit;

void main() {
    ivec2 px   = ivec2(gl_GlobalInvocationID.xy);
    vec3  P    = texelFetch(position_atlas, px, 0).xyz;
    vec3  N    = normalize(texelFetch(normal_atlas, px, 0).xyz);
    if (length(P) < 1e-6) return;  // empty texel

    float ao       = 0.0;
    vec3  bent_sum = vec3(0.0);
    uint  rng      = pcg_init(uvec2(px), 0u);

    for (int i = 0; i < num_samples; ++i) {
        vec3 L = cosine_hemisphere_sample(N, pcg_float(rng), pcg_float(rng));
        ray_hit = false;
        traceRayEXT(tlas,
            gl_RayFlagsTerminateOnFirstHitEXT | gl_RayFlagsSkipClosestHitShaderEXT,
            0xFF, 0, 0, 0,
            P + N * 0.001,   // offset to avoid self-intersection
            0.0,
            L,
            max_distance,
            0);
        if (!ray_hit) {
            ao       += 1.0;
            bent_sum += L;
        }
    }
    float ao_factor   = ao / float(num_samples);
    vec3  bent_normal = length(bent_sum) > 0.0 ? normalize(bent_sum) : N;

    imageStore(ao_out,          px, vec4(ao_factor, 0.0, 0.0, 0.0));
    imageStore(bent_normal_out, px, vec4(bent_normal * 0.5 + 0.5, 1.0));
}
```

#### Using Baked Bent Normals at Runtime
Replace the geometric normal with the baked bent normal when sampling diffuse IBL or SH irradiance. The bent normal points toward the most unoccluded sky direction, giving correct shadowing for recessed surfaces (cavities, under overhangs) without any runtime ray tracing.

```glsl
// Runtime bent normal application — fragment shader
layout(set=2, binding=0) uniform sampler2D ao_map;
layout(set=2, binding=1) uniform sampler2D bent_normal_map;

vec2 lm_uv = v_lightmap_uv;
float ao          = texture(ao_map, lm_uv).r;
vec3  bent_normal = normalize(texture(bent_normal_map, lm_uv).rgb * 2.0 - 1.0);

// Use bent normal for diffuse IBL — captures directional occlusion
vec3 diffuse_irr  = sh_irradiance(bent_normal) * ao;   // §56 SH evaluation
vec3 diffuse      = albedo * diffuse_irr;
```

**Use cases** — arch-viz, static game environments, character skin pre-baked occlusion; combined with lightmapping for complete static surface lighting.  
**Reference** — [Landis: Production-Ready Global Illumination (SIGGRAPH 2002 Course)](https://graphics.pixar.com/library/ProductionReadyGI/); [UE4: Baked Bent Normals](https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/LightingAndShadows/BentNormalMaps/)

---


---


## Integrations

**Ch205** (this series) covers the GI and material algorithms that build on the pipeline configurations established here. **Ch154** (GPU-Driven Rendering) is the implementation companion for Cat I — GPU-driven indirect draw, meshlet culling, and VRS. **Ch127** (Mesh Shaders and VRS) provides the hardware architecture underlying variable-rate shading and meshlet generation. **Ch135** (Vulkan Ray Tracing) provides the API context for shadow ray techniques in Cat III. **Ch24** (Vulkan and EGL for Application Developers) covers the Vulkan pipeline state objects that the pipeline configurations in Cat I compile into.
