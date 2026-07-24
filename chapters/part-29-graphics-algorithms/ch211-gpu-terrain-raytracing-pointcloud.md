# Chapter 211: GPU Geometry Algorithms — Terrain, Ray Tracing Geometry, and Point Clouds (Part XXIX)

*Part XXIX — Graphics Algorithms*

**Audiences:** Graphics application developers implementing large-scale terrain systems, ray tracing acceleration geometry, or point cloud processing pipelines on Linux using Vulkan.

This chapter is the fourth volume of the GPU Geometry Algorithms series. It covers three domains oriented around environment geometry and optical precision: Category VIII (Terrain, Procedural Content, and Environment) covers procedural geometry generation, navigation mesh construction and GPU pathfinding, terrain LOD and streaming, GPU particle and ribbon geometry, ocean and wave simulation, and large-scale geospatial terrain processing; Category IX (Ray Tracing and Optical Geometry) covers the ray tracing geometry pipeline (TLAS/BLAS construction, refitting, traversal), shadow geometry, IBL environment map preprocessing, GI probe geometry, geometric optics (lens / caustics / beam tracing), screen-space reflections, GPU radiosity, path tracing pipeline geometry, atmospheric scattering, subsurface scattering geometry, order-independent transparency, and polygon and frustum clipping; Category X (Point Cloud, Reconstruction, and Perception) covers point cloud processing (3D Gaussian Splatting, point splatting, SLAM), structure-from-motion and multi-view stereo, 3D feature descriptors and pose estimation, and ICP point cloud registration.

## Table of Contents

- **[VIII. Terrain, Procedural Content, and Environment](#viii-terrain-procedural-content-and-environment)**
  - [69. Procedural Geometry Generation](#69-procedural-geometry-generation)
  - [70. Navigation Meshes and GPU Pathfinding](#70-navigation-meshes-and-gpu-pathfinding)
  - [71. Terrain LOD and Streaming](#71-terrain-lod-and-streaming)
  - [72. GPU Particle Systems and Ribbon Geometry](#72-gpu-particle-systems-and-ribbon-geometry)
  - [73. GPU Ocean and Wave Simulation](#73-gpu-ocean-and-wave-simulation)
  - [74. GPU Geospatial Processing: Large-Scale Terrain](#74-gpu-geospatial-processing-large-scale-terrain)
- **[IX. Ray Tracing and Optical Geometry](#ix-ray-tracing-and-optical-geometry)**
  - [75. Ray Tracing Geometry Pipeline](#75-ray-tracing-geometry-pipeline)
  - [76. Shadow Geometry Algorithms](#76-shadow-geometry-algorithms)
  - [77. IBL and Environment Map Preprocessing](#77-ibl-and-environment-map-preprocessing)
  - [78. Global Illumination Geometry](#78-global-illumination-geometry)
  - [79. Geometric Optics: Lens, Caustics, and Beam Tracing](#79-geometric-optics-lens-caustics-and-beam-tracing)
  - [80. Screen-Space Reflections (SSR/SSSR)](#80-screen-space-reflections-ssrsssr)
  - [81. GPU Radiosity: Form Factor Computation](#81-gpu-radiosity-form-factor-computation)
  - [82. GPU Path Tracing Pipeline](#82-gpu-path-tracing-pipeline)
  - [83. Atmospheric Scattering and Sky Rendering](#83-atmospheric-scattering-and-sky-rendering)
  - [84. Subsurface Scattering Geometry](#84-subsurface-scattering-geometry)
  - [85. Order-Independent Transparency](#85-order-independent-transparency)
  - [86. GPU Polygon and Frustum Clipping](#86-gpu-polygon-and-frustum-clipping)
- **[X. Point Cloud, Reconstruction, and Perception](#x-point-cloud-reconstruction-and-perception)**
  - [87. Point Cloud Processing](#87-point-cloud-processing)
  - [88. GPU Structure from Motion and Multi-View Stereo](#88-gpu-structure-from-motion-and-multi-view-stereo)
  - [89. 3D Feature Descriptors and Pose Estimation](#89-3d-feature-descriptors-and-pose-estimation)
  - [90. Iterative Closest Point (ICP) Registration](#90-iterative-closest-point-icp-registration)


---

## VIII. Terrain, Procedural Content, and Environment

Large-scale environment geometry requires specialized LOD, streaming, and simulation approaches. This category covers GPU terrain rendering with quadtree or clipmap LOD and streaming from disk, hydraulic erosion simulation that produces naturalistic height fields, ocean wave synthesis via inverse FFT of a Philips/JONSWAP spectrum, GPU geospatial processing for satellite-scale terrain meshes, procedural geometry generation via compute shaders (L-systems, instanced geometry), particle and ribbon geometry systems for VFX, and navigation mesh construction for AI pathfinding on dynamically generated terrain. All of these share a common challenge: representing continuous, kilometre-scale natural phenomena efficiently on GPU.

### 69. Procedural Geometry Generation

*Audience: graphics application developers.*

Procedural generation creates geometry analytically rather than from stored mesh data, enabling effectively infinite variation at GPU-native speed. The common pattern: a compute shader writes vertex/index data into a pre-allocated SSBO that is then consumed directly by the rasterizer — no CPU roundtrip.

### 69.1 GPU Terrain: Fractional Brownian Motion and Erosion

A terrain heightmap can be generated entirely in a compute shader using layered noise (fBm):

```glsl
// terrain_gen.comp
layout(set=0,binding=0, r32f) uniform writeonly image2D heightmap;
layout(local_size_x=16, local_size_y=16) in;

float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1,311.7))) * 43758.5453); }
float noise(vec2 p) {
    vec2 i = floor(p), f = fract(p);
    f = f*f*(3.0-2.0*f);
    return mix(mix(hash(i), hash(i+vec2(1,0)), f.x),
               mix(hash(i+vec2(0,1)), hash(i+vec2(1,1)), f.x), f.y);
}

float fbm(vec2 p) {
    float v=0.0, a=0.5;
    for (int i=0; i<8; i++) { v += a*noise(p); p*=2.1; a*=0.5; }
    return v;
}

void main() {
    ivec2 coord = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv    = vec2(coord) / vec2(imageSize(heightmap));
    imageStore(heightmap, coord, vec4(fbm(uv * 4.0)));
}
```

**Hydraulic erosion** (Mei et al. 2007) iterates across the heightmap: each cell receives rain, water flows to lower neighbors driven by height differences, sediment is eroded proportional to flow velocity, and deposited where flow slows:

```glsl
// erosion_step.comp — simplified single-pass thermal + hydraulic
layout(set=0,binding=0) buffer Height   { float h[]; };
layout(set=0,binding=1) buffer Water    { float w[]; };
layout(set=0,binding=2) buffer Sediment { float s[]; };
layout(local_size_x=64) in;

void main() {
    uint c = gl_GlobalInvocationID.x;
    ivec2 xy = ivec2(c % W, c / W);
    // compute outflow to 4 neighbors by height difference
    float totalH = h[c] + w[c];
    float d[4]; float totalFlow = 0.0;
    for (int i = 0; i < 4; i++) {
        ivec2 n = xy + OFFSETS[i];
        float hn = h[n.y*W+n.x] + w[n.y*W+n.x];
        d[i] = max(0.0, totalH - hn);
        totalFlow += d[i];
    }
    float scale = min(w[c], totalFlow) / max(totalFlow, 1e-6);
    for (int i = 0; i < 4; i++) {
        uint nc = uint((xy + OFFSETS[i]).y*W + (xy + OFFSETS[i]).x);
        float flow = scale * d[i];
        atomicAdd_f(w, nc, +flow);
        atomicAdd_f(w, c,  -flow);
        atomicAdd_f(s, nc, +flow * EROSION_K * h[c]);
        atomicAdd_f(h, c,  -flow * EROSION_K);
    }
}
```

100–200 iterations produce visually plausible erosion channels and alluvial fans. [Source: Mei et al. "Fast Hydraulic Erosion Simulation", 2007; Olsen "Realtime Procedural Terrain Generation", 2004]

### 69.2 L-Systems in Compute Shaders

L-systems expand a string of symbols by applying production rules iteratively. GPU parallelisation uses a **parallel prefix scan** to determine output positions before writing:

1. Each symbol in the current string is one thread; it reads its character and looks up the production rule length.
2. Prefix scan over rule lengths gives each symbol its output offset.
3. Each thread writes its expanded string into the output buffer at its computed offset.
4. Repeat for N generations.

```glsl
// lsystem_expand.comp
layout(set=0,binding=0) readonly  buffer InStr  { uint8_t in_str[]; };
layout(set=0,binding=1) writeonly buffer OutStr { uint8_t out_str[]; };
layout(set=0,binding=2) readonly  buffer Offsets{ uint offsets[]; };  // prefix scan result
layout(set=0,binding=3) readonly  buffer Rules  { uint8_t rules[512]; uint ruleLen[256]; };
layout(local_size_x=64) in;

void main() {
    uint i    = gl_GlobalInvocationID.x;
    uint sym  = in_str[i];
    uint base = offsets[i];
    uint len  = ruleLen[sym];
    for (uint j = 0; j < len; j++)
        out_str[base + j] = rules[sym * MAX_RULE_LEN + j];
}
```

A turtle interpretation pass then converts the symbol string to vertex data: `F` → emit cylinder segment, `+`/`−` → rotate, `[`/`]` → push/pop transform stack. Branch stacks require serialization but can be pre-computed in a first pass. [Source: Prusinkiewicz & Lindenmayer "The Algorithmic Beauty of Plants" §1; Lienhard et al. "Fast and Simple Creation of GPU Tree Models", TPCG 2010]

### 69.3 Wave Function Collapse for Tile Geometry

Wave Function Collapse (WFC) generates tile-map geometry consistent with local adjacency constraints extracted from an example. The GPU implementation adapts the AC-3 arc-consistency propagation as a BFS wavefront:

```glsl
// wfc_propagate.comp — one pass of constraint propagation
layout(set=0,binding=0) buffer CellMask { uint mask[]; }; // bitfield of allowed tiles per cell
layout(set=0,binding=1) buffer Changed  { uint changed; };
layout(local_size_x=64) in;

layout(push_constant) uniform PC { uint W, H; } pc;
// adjacency_table[tile * 4 + direction] = allowed-neighbor bitfield
layout(set=0,binding=2) readonly buffer AdjTable { uint adj[]; };

void main() {
    uint c   = gl_GlobalInvocationID.x;
    ivec2 xy = ivec2(c % pc.W, c / pc.W);
    uint m   = mask[c];
    uint newm = 0xFFFFFFFF;
    for (int dir = 0; dir < 4; dir++) {
        ivec2 n = xy + OFFSETS[dir];
        if (n.x < 0 || n.x >= int(pc.W) || n.y < 0 || n.y >= int(pc.H)) continue;
        uint nm = mask[n.y*pc.W + n.x];
        // For each allowed tile in neighbor, OR its allowed-neighbor bits for direction
        uint allowed = 0u;
        uint bits = nm;
        while (bits != 0) {
            int t = findLSB(bits); bits &= bits-1;
            allowed |= adj[t * 4 + (dir ^ 1)];
        }
        newm &= allowed;
    }
    newm &= m;
    if (newm != m) { mask[c] = newm; atomicOr(changed, 1u); }
}
```

Iterate until `changed == 0`. A CPU-side collapse step picks the lowest-entropy cell, collapses its mask to one tile, and re-runs the propagation kernel. [Source: Gumin "Wave Function Collapse" (2016); Merrell "Example-Based Model Synthesis", I3D 2007]

### 69.4 Task Shader Procedural Instancing

For buildings, rocks, and foliage, the amplification/task shader is an ideal procedural generator: given a sparse set of placement points in a SSBO, the task shader evaluates placement rules (slope, altitude, biome mask), generates per-instance transforms, and emits mesh shader workgroups only for visible, rule-passing instances:

```glsl
// proc_instance.task.glsl
layout(set=0,binding=0) readonly buffer PlacementPoints { vec4 pts[]; };
layout(set=0,binding=1) readonly buffer BiomeMask       { float biome[]; };
layout(local_size_x=32) in;
taskPayloadSharedEXT InstancePayload { mat4 transforms[32]; uint count; } payload;

void main() {
    uint id = gl_GlobalInvocationID.x;
    if (gl_LocalInvocationID.x == 0) payload.count = 0;
    barrier();

    vec4  pt    = pts[id];
    float slope = computeSlope(pt.xy);
    float b     = biome[id];

    bool emit = slope < MAX_SLOPE && b > MIN_BIOME_WEIGHT;
    // frustum cull
    if (emit) emit = sphereInFrustum(pt.xyz, INST_RADIUS);

    if (emit) {
        uint slot = atomicAdd(payload.count, 1);
        payload.transforms[slot] = buildTransform(pt.xyz, slope);
    }
    barrier();
    if (gl_LocalInvocationID.x == 0)
        EmitMeshTasksEXT(payload.count, 1, 1);
}
```

[Source: Wihlidal "GPU-Driven Rendering Pipelines", SIGGRAPH 2015; Graham "Procedural Generation in Game Development", GDC 2019]

---

### 70. Navigation Meshes and GPU Pathfinding

*Audience: graphics application developers.*

Navigation mesh (navmesh) generation and pathfinding are geometry algorithms in disguise: voxelisation is a rasterization problem, region labelling is a BFS problem on a graph, and flow field computation is a parallel shortest-path problem on a grid — all map to Vulkan compute naturally.

### 70.1 Voxelisation for NavMesh Generation (Recast-style)

Recast's CPU navmesh generation begins with a **solid heightfield**: rasterize each triangle onto a voxel column grid, producing spans of solid voxels. The GPU equivalent uses depth-peeling compute:

```glsl
// voxelise.comp — rasterize triangles into column heightfield
layout(set=0,binding=0) readonly buffer Tris   { TriangleData tris[]; };
layout(set=0,binding=1) buffer Heightfield { Span spans[MAX_CELLS * MAX_SPANS]; uint spanCount[]; };
layout(local_size_x=64) in;

void main() {
    uint t   = gl_GlobalInvocationID.x;
    // Project triangle to XZ grid, find overlapping columns
    vec3 v0 = tris[t].v0, v1 = tris[t].v1, v2 = tris[t].v2;
    ivec2 minCell = ivec2(floor(min(min(v0.xz, v1.xz), v2.xz) / CELL_SIZE));
    ivec2 maxCell = ivec2(ceil(max(max(v0.xz, v1.xz), v2.xz) / CELL_SIZE));
    for (int cz = minCell.y; cz <= maxCell.y; cz++)
    for (int cx = minCell.x; cx <= maxCell.x; cx++) {
        // compute min/max Y of triangle within this column via ray-tri intersection
        float yMin, yMax;
        if (columnTriangleClip(v0, v1, v2, cx, cz, yMin, yMax)) {
            uint sc = atomicAdd(spanCount[cz*GRID_W+cx], 1u);
            spans[(cz*GRID_W+cx)*MAX_SPANS + sc] = Span(yMin, yMax, walkable(v0,v1,v2));
        }
    }
}
```

After voxelisation, a walkability filter checks each span's clearance (vertical space above), slope (triangle normal vs. up), and step height — fully parallel per span.

### 70.2 Parallel Region Labelling via GPU BFS

Recast clusters walkable spans into **regions** for polygon extraction. The GPU equivalent is a parallel iterative BFS flood-fill: each thread checks its 4-connected neighbours and updates its label to the minimum neighbour label if smaller:

```glsl
// region_label.comp
layout(set=0,binding=0) buffer Labels  { uint label[]; };   // init: label[i] = i
layout(set=0,binding=1) buffer Changed { uint changed; };
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    uint myL = label[i];
    uint minL = myL;
    for each walkable neighbour j of i:
        minL = min(minL, label[j]);
    if (minL < myL) {
        label[i] = minL;
        atomicOr(changed, 1u);
    }
}
```

Iterate until `changed == 0`. Convergence requires O(diameter) iterations (typically 20–50 for game-scale meshes). A path-compression step (like union-find) can accelerate convergence. [Source: Recast navigation: https://github.com/recastnavigation/recastnavigation; parallel label propagation: Soman et al. "Fast Transitive Closure using Graph Partitioning", SC 2011]

### 70.3 GPU Flow Fields for Navigation

A **flow field** precomputes for every cell a steering direction towards the goal, avoiding obstacles. Construction is a parallel BFS from the goal cell, computing a distance field, then differencing adjacent cells to obtain the gradient (steering direction):

```glsl
// flow_field_step.comp — one BFS wavefront expansion
layout(set=0,binding=0) buffer Dist   { uint dist[]; };   // init: goal=0, others=UINT_MAX
layout(set=0,binding=1) buffer Changed{ uint changed; };
layout(push_constant) uniform PC { uint W, H; } pc;
layout(local_size_x=64) in;

void main() {
    uint c = gl_GlobalInvocationID.x;
    if (!walkable[c]) return;
    uint myD = dist[c];
    for each cardinal neighbour n of c:
        if (walkable[n] && dist[n] < myD - 1) {
            atomicMin(dist[c], dist[n] + 1);
            atomicOr(changed, 1u);
        }
}
```

After convergence, a separate pass computes the steering direction at each cell as the gradient of `dist` (direction towards the lowest-distance neighbour). Agents sample their current cell's direction vector to steer without per-agent A* queries.

Flow fields scale to thousands of agents at constant cost per frame (one texture read per agent). Recompute only when the goal changes or obstacles move. [Source: Elijah Emerson "Crowd Pathfinding and Steering Using Flow Field Tiles", Game AI Pro 2013; https://github.com/recastnavigation/recastnavigation detour for reference path finding]

---

### 71. Terrain LOD and Streaming

*Audience: graphics application developers.*

§69 covered procedural terrain *generation*. This section covers runtime terrain *rendering*: quadtree LOD selection, geomorphing to prevent popping, virtual texture streaming, and GPU heightfield collision.

### 71.1 CDLOD: Continuous Distance-Dependent Level of Detail

CDLOD (Strugar 2010) traverses a heightmap quadtree in compute, selecting per-node LOD based on projected screen-space error:

```glsl
// cdlod_traverse.comp
struct QuadNode { vec2 min, max; int level; };
layout(set=0,binding=0) readonly buffer QuadTree { QuadNode nodes[]; };
layout(set=0,binding=1) writeonly buffer DrawList { uint ids[]; };
layout(set=0,binding=2) buffer DrawCount { uint count; };
layout(local_size_x=64) in;

float screenError(QuadNode n, vec3 cam) {
    float dist=max(0.0,length(cam.xz-(n.min+n.max)*0.5)-length(n.max-n.min)*0.5);
    return ERROR_SCALE*float(1<<n.level)/max(dist,0.001);
}
void main() {
    uint id=gl_GlobalInvocationID.x;
    QuadNode n=nodes[id];
    if(!nodeInFrustum(n)) return;
    if(n.level==0 || screenError(n,camPos)<SCREEN_ERROR_THRESHOLD) {
        uint slot=atomicAdd(count,1u); ids[slot]=id;
    }
}
```

Selected nodes drive `vkCmdDrawIndirect`. [Source: Strugar "Continuous Distance-Dependent Level of Detail for Rendering Heightmaps", JGT 2010]

### 71.2 Geomorphing to Prevent LOD Popping

Vertices blend between fine and coarse grid positions as a node enters/leaves its LOD range:

```glsl
// terrain.vert
float morphFactor(float dist, float mStart, float mEnd) {
    return clamp((dist-mStart)/(mEnd-mStart), 0.0, 1.0);
}
void main() {
    vec2 uv      = (aPos.xz - nodeMin) / nodeSize;
    float hFine  = texture(heightmap, uv).r * HEIGHT_SCALE;
    float hCoarse= sampleSnappedHeight(uv);  // nearest even-grid sample
    float mf     = morphFactor(length(aPos-camPos), morphStart, morphEnd);
    float h      = mix(hFine, hCoarse, mf);
    gl_Position  = vp * vec4(aPos.x, h, aPos.z, 1.0);
}
```

### 71.3 Virtual Terrain Texturing: Sparse Clipmaps

A clipmap stores terrain texture as a stack of annular rings centred on the camera, each ring at a progressively coarser resolution. When the camera moves, GPU compute fills new tiles into the ring boundary:

```glsl
// clipmap_update.comp
layout(set=0,binding=0, rgba8) uniform writeonly image2DArray clipmap;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec3 coord=ivec3(gl_GlobalInvocationID.xy, CURRENT_LEVEL);
    vec2  uv=(vec2(coord.xy)+clipmapOffset[CURRENT_LEVEL])*texelScale[CURRENT_LEVEL];
    imageStore(clipmap, coord, sampleTerrainTex(uv));
}
```

[Source: Tanner et al. "The Clipmap: A Virtual Mipmap", SIGGRAPH 1998; GPU Gems 2 §2]

### 71.4 GPU Heightfield Ray Intersection

Binary search along the ray's XZ projection finds the terrain hit point for physics queries (vehicle wheels, character stepping):

```glsl
// heightfield_ray.comp
layout(local_size_x=64) in;
void main() {
    uint r=gl_GlobalInvocationID.x;
    vec3 o=rayOrigin[r], d=rayDir[r];
    float tLo=0.0, tHi=MAX_T;
    for(int i=0; i<20; i++) {
        float tMid=(tLo+tHi)*0.5;
        vec3 p=o+tMid*d;
        float h=texture(heightmap, p.xz/TERRAIN_SIZE).r*HEIGHT_SCALE;
        if(p.y<h) tHi=tMid; else tLo=tMid;
    }
    hitT[r]=(tLo+tHi)*0.5;
    hitNorm[r]=terrainNormal(o+hitT[r]*d);
}
```

---

### 72. GPU Particle Systems and Ribbon Geometry

*Audience: graphics application developers.*

Particle systems are the standard GPU geometry for effects (fire, smoke, sparks, rain, magic). Unlike §48 physics-driven SPH, this section covers the *rendering geometry pipeline*: how particles are born, evolved, compacted, and converted into drawable primitives each frame.

### 72.1 Particle Lifecycle: Emit, Age, Kill

Particles are stored in a flat SSBO pool. Each frame: (1) increment age and kill dead particles; (2) emit new particles into freed slots. Stream compaction (prefix scan) packs live particles to a dense range for rendering:

```glsl
// particle_update.comp
layout(set=0,binding=0) buffer Particles { ParticleData p[]; };
layout(set=0,binding=1) buffer AliveOut  { uint alive[]; };
layout(set=0,binding=2) buffer AliveCount{ uint count; };
layout(push_constant) uniform PC { float dt; } pc;
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    p[i].age  += pc.dt;
    p[i].pos  += p[i].vel * pc.dt + vec3(0,-9.8,0)*pc.dt*pc.dt*0.5;
    p[i].vel  += vec3(0,-9.8,0) * pc.dt;
    p[i].vel  *= 0.99;  // drag
    if (p[i].age < p[i].lifetime) {
        uint slot = atomicAdd(count, 1u);
        alive[slot] = i;
    }
}
```

Emission fills slots after compaction using a free-list counter. [Source: Harris "GPU Gems 3 §39: Parallel Prefix Sum"; Fatahalian et al. "Sequentially Consistent Atomic Operations in GLSL"]

### 72.2 Billboard Expansion in Mesh Shaders

Each live particle becomes a screen-aligned quad. The mesh shader generates 4 vertices and 2 triangles per particle, avoiding geometry shader overhead:

```glsl
// particle_mesh.glsl
layout(triangles, max_vertices=128, max_primitives=64) out;
layout(local_size_x=64) in;
taskPayloadSharedEXT struct { uint idx[64]; uint count; } payload;
out vec2 vUV[];
out vec4 vColor[];
void main() {
    SetMeshOutputsEXT(payload.count * 4, payload.count * 2);
    for (uint t = gl_LocalInvocationID.x; t < payload.count; t += 64) {
        ParticleData pa = particles[payload.idx[t]];
        float sz = pa.size * (1.0 - pa.age / pa.lifetime);  // shrink on death
        vec3 right = cameraRight * sz;
        vec3 up    = cameraUp   * sz;
        gl_MeshVerticesEXT[t*4+0].gl_Position = vp * vec4(pa.pos - right - up, 1);
        gl_MeshVerticesEXT[t*4+1].gl_Position = vp * vec4(pa.pos + right - up, 1);
        gl_MeshVerticesEXT[t*4+2].gl_Position = vp * vec4(pa.pos + right + up, 1);
        gl_MeshVerticesEXT[t*4+3].gl_Position = vp * vec4(pa.pos - right + up, 1);
        vUV[t*4+0]=vec2(0,0); vUV[t*4+1]=vec2(1,0);
        vUV[t*4+2]=vec2(1,1); vUV[t*4+3]=vec2(0,1);
        gl_PrimitiveTriangleIndicesEXT[t*2+0] = uvec3(t*4+0,t*4+1,t*4+2);
        gl_PrimitiveTriangleIndicesEXT[t*2+1] = uvec3(t*4+0,t*4+2,t*4+3);
    }
}
```

### 72.3 Ribbon / Trail Geometry via Rolling Ring Buffer

A ribbon is the surface swept by a particle along its path — N recent positions stored in a ring buffer, converted to a quad strip each frame:

```glsl
// ribbon_expand.comp — ring buffer → quad strip
layout(set=0,binding=0) readonly buffer RingBuf { vec4 history[N_PARTICLES * TRAIL_LEN]; };
layout(set=0,binding=1) readonly buffer Head    { uint head[N_PARTICLES]; };
layout(set=0,binding=2) writeonly buffer Strip  { vec4 verts[]; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    uint h = head[p];
    for (uint s = 0; s < TRAIL_LEN - 1; s++) {
        uint cur  = (h + s)          % TRAIL_LEN;
        uint next = (h + s + 1)      % TRAIL_LEN;
        vec3 a    = history[p*TRAIL_LEN + cur ].xyz;
        vec3 b    = history[p*TRAIL_LEN + next].xyz;
        vec3 dir  = normalize(b - a);
        vec3 perp = normalize(cross(dir, cameraPos - a));
        float w   = mix(WIDTH_TIP, WIDTH_BASE, float(s) / float(TRAIL_LEN));
        verts[(p*(TRAIL_LEN-1) + s)*2 + 0] = vec4(a - perp*w, 1);
        verts[(p*(TRAIL_LEN-1) + s)*2 + 1] = vec4(a + perp*w, 1);
    }
}
```

### 72.4 Soft Particles via Depth Comparison

Soft particles fade out where they intersect scene geometry, avoiding hard silhouettes. The fragment shader reads the depth buffer and blends by proximity:

```glsl
// soft_particle.frag
uniform sampler2D depthBuffer;
in vec2 screenUV;
in float particleDepth;
void main() {
    float sceneZ = texture(depthBuffer, screenUV).r;
    float sceneLinear = linearize(sceneZ);
    float partLinear  = linearize(particleDepth);
    float softness    = clamp((sceneLinear - partLinear) / SOFT_DISTANCE, 0.0, 1.0);
    fragColor.a *= softness;
}
```

Requires the depth buffer to be readable as a texture (`VK_IMAGE_LAYOUT_DEPTH_READ_ONLY_STENCIL_ATTACHMENT_OPTIMAL` or a depth copy pre-pass). [Source: Vlachos "Soft Particles", GDC 2007]

---

### 73. GPU Ocean and Wave Simulation

*Audience: graphics application developers.*

Ocean rendering combines two complementary techniques: an FFT-based statistical wave model (Tessendorf 2001) for large-scale height-field waves, and Gerstner (trochoidal) waves for deterministic art-directed swells. Both produce vertex-displacement maps consumed by a tessellated grid each frame.

### 73.1 Phillips Spectrum and Initial Conditions

The Phillips spectrum P(k) models the energy distribution of ocean waves as a function of wave vector **k**:

```
P(k) = A · exp(-1/(k·L)²) / k⁴ · |k̂ · ŵ|²
```

where L = V²/g (Pierson-Moskowitz limit speed), V = wind speed, ŵ = wind direction. GPU initialisation generates the complex amplitude h₀(**k**) by sampling Gaussian random numbers and weighting by √P(k):

```glsl
// ocean_init.comp — initialise complex spectrum h₀(k)
layout(set=0,binding=0, rg32f) uniform writeonly image2D h0;   // complex amplitudes
layout(set=0,binding=1) uniform sampler2D gaussianNoise;       // pre-generated Gaussian pairs
layout(push_constant) uniform PC {
    float A; vec2 windDir; float windSpeed; float g;
    int N; float L;  // grid size N×N, patch size L metres
} pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  k   = (vec2(id) - float(pc.N)*0.5) * (2.0*3.14159265/pc.L);
    float km  = max(length(k), 1e-6);
    float kL  = pc.windSpeed * pc.windSpeed / pc.g;
    float ph  = pc.A * exp(-1.0/(km*kL)*(km*kL)) / (km*km*km*km);
    float kw  = dot(normalize(k), pc.windDir);
    ph *= kw * kw;
    if (kw < 0.0) ph *= 0.07;   // damp waves against wind
    vec2 xi = texture(gaussianNoise, vec2(id)/float(pc.N)).rg;
    imageStore(h0, id, vec4(xi * sqrt(ph * 0.5), 0, 0));
}
```

[Source: Tessendorf "Simulating Ocean Water", SIGGRAPH Course Notes 2001]

### 73.2 Time Evolution and IFFT

Each frame, propagate h₀ to h(**k**,t) using the dispersion relation ω(k) = √(g·|**k**|):

```glsl
// ocean_update.comp — time-evolve spectrum, prepare IFFT inputs
layout(set=0,binding=0) readonly buffer H0    { vec2 h0[];    };
layout(set=0,binding=1) writeonly buffer Hkt  { vec2 hkt[];   };  // height
layout(set=0,binding=2) writeonly buffer Dxt  { vec2 dxt[];   };  // choppy X
layout(set=0,binding=3) writeonly buffer Dzt  { vec2 dzt[];   };  // choppy Z
layout(push_constant) uniform PC { float t; float g; float L; int N; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    int   idx = id.y * pc.N + id.x;
    vec2  k   = (vec2(id) - float(pc.N)*0.5) * (2.0*3.14159265/pc.L);
    float km  = max(length(k), 1e-6);
    float w   = sqrt(pc.g * km);
    // Euler formula: e^{iwt} = cos(wt) + i·sin(wt)
    float c   = cos(w * pc.t), s = sin(w * pc.t);
    vec2  h   = h0[idx];
    vec2  hc  = vec2(h.x, -h.y);   // conjugate of h0(-k)
    vec2  ht  = vec2(h.x*c - h.y*s, h.x*s + h.y*c)
              + vec2(hc.x*c + hc.y*s, -hc.x*s + hc.y*c);
    hkt[idx]  = ht;
    vec2 kn   = k / km;
    dxt[idx]  = vec2(-ht.y * kn.x,  ht.x * kn.x);  // ik_x · h(k,t)
    dzt[idx]  = vec2(-ht.y * kn.y,  ht.x * kn.y);  // ik_z · h(k,t)
}
```

Apply a 2D IFFT to `hkt`, `dxt`, `dzt` using a GPU FFT library (VkFFT, §73.5) to obtain height field H(x,z,t) and choppy displacement (Dx, Dz).

### 73.3 Choppy Waves via Jacobian

Horizontal displacement (choppiness) moves water particles toward wave crests, sharpening peaks. The Jacobian J of the displacement field detects wave breaking (J < 0 → foam):

```glsl
// ocean_jacobian.comp — compute Jacobian for foam mask
layout(set=0,binding=0) readonly buffer Dx { float dx[]; };
layout(set=0,binding=1) readonly buffer Dz { float dz[]; };
layout(set=0,binding=2, r8) uniform writeonly image2D foamMask;
layout(push_constant) uniform PC { int N; float L; float choppiness; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id = ivec2(gl_GlobalInvocationID.xy);
    int   w  = pc.N;
    ivec2 nx = ivec2((id.x+1)%w, id.y), px = ivec2((id.x-1+w)%w, id.y);
    ivec2 nz = ivec2(id.x, (id.y+1)%w), pz = ivec2(id.x, (id.y-1+w)%w);
    float dDx_dx = (dx[nz.y*w+nx.x] - dx[pz.y*w+px.x]) / (2.0 * pc.L / float(w));
    float dDz_dz = (dz[nz.y*w+nz.x] - dz[pz.y*w+pz.x]) / (2.0 * pc.L / float(w));
    float J = (1.0 + pc.choppiness*dDx_dx) * (1.0 + pc.choppiness*dDz_dz);
    float foam = clamp(1.0 - J, 0.0, 1.0);
    imageStore(foamMask, id, vec4(foam));
}
```

[Source: Tessendorf 2001; Bruneton & Neyret "Real-time Realistic Ocean Lighting", EuroGraphics 2010]

### 73.4 Gerstner Wave Summation

For art-directed waves, sum N Gerstner (trochoidal) waves analytically — no FFT required, suitable for 4–8 waves with precise control:

```glsl
// ocean.vert — Gerstner wave displacement in vertex shader
struct GerstnerWave { vec2 dir; float amp, freq, phase, steep; };
layout(set=0,binding=0) readonly buffer Waves { GerstnerWave waves[MAX_WAVES]; };
layout(push_constant) uniform PC { int N; float time; } pc;
void main() {
    vec3 pos = inPos;
    vec3 nor = vec3(0,1,0);
    for (int i=0; i<pc.N; i++) {
        GerstnerWave w = waves[i];
        float theta = dot(w.dir, pos.xz)*w.freq + pc.time*w.phase;
        float s = sin(theta), c = cos(theta);
        pos.xz += w.steep * w.amp * w.dir * c;
        pos.y  += w.amp * s;
        nor.xz -= w.dir * w.freq * w.amp * c;
        nor.y  -= w.steep * w.freq * w.amp * s;
    }
    gl_Position = vp * vec4(pos, 1.0);
    vNormal     = normalize(nor);
}
```

[Source: Finch "Effective Water Simulation from Physical Models", GPU Gems 1, Ch. 1]

### 73.5 VkFFT Integration

VkFFT ([source](https://github.com/DTolm/VkFFT), MIT) provides Vulkan-native FFT on the GPU — the compute backbone for §73.2. It generates SPIR-V at runtime for the target device:

```cpp
VkFFTApplication app{};
VkFFTConfiguration cfg{};
cfg.FFTdim        = 2;
cfg.size[0]       = N; cfg.size[1] = N;
cfg.device        = &device;
cfg.queue         = &computeQueue;
cfg.isInputFormatted = 1;
cfg.bufferSize    = &bufferSize;
cfg.buffer        = &hktBuffer;
VkFFTResult res   = initializeVkFFT(&app, cfg);
// Each frame:
VkFFTLaunchParams lp{}; lp.commandBuffer = cmd; lp.inputBuffer = &hktBuffer;
VkFFTAppend(&app, -1, &lp);  // inverse FFT
```

[Source: Tolmachev "VkFFT — A Performant, Cross-Platform and Open-Source GPU FFT Library", 2021]

---

### 74. GPU Geospatial Processing: Large-Scale Terrain

*Audience: systems developers, graphics application developers.*

Real-world terrain from LIDAR point clouds or satellite DEMs requires GPU processing pipelines distinct from procedural terrain (§71): reprojection from geographic coordinates (WGS84/UTM), sparse tiling, massive point cloud decimation, and multi-level terrain mesh streaming. Applications include mapping, simulation, autonomous vehicle testing, and digital twins.

### 74.1 WGS84 to ECEF and Local Frame Projection

Reproject geographic coordinates (longitude, latitude, altitude) to Earth-Centred Earth-Fixed (ECEF) Cartesian coordinates, then to a local ENU (East-North-Up) frame for rendering:

```glsl
// geo_project.comp — WGS84 → ECEF → local ENU frame
layout(set=0,binding=0) readonly buffer LLA  { vec3 lla[];  };  // lon,lat,alt in degrees/metres
layout(set=0,binding=1) writeonly buffer ECEF { vec3 ecef[]; };
layout(push_constant) uniform PC { vec3 origin_LLA; uint N; } pc;  // local origin for ENU
layout(local_size_x=64) in;
const float a = 6378137.0;           // WGS84 semi-major
const float e2 = 0.00669437999014;   // eccentricity²
void main() {
    uint i   = gl_GlobalInvocationID.x;
    if (i >= pc.N) return;
    float lat = radians(lla[i].y), lon = radians(lla[i].x), alt = lla[i].z;
    float N_  = a / sqrt(1.0 - e2*sin(lat)*sin(lat));
    ecef[i]   = vec3((N_+alt)*cos(lat)*cos(lon),
                     (N_+alt)*cos(lat)*sin(lon),
                     (N_*(1.0-e2)+alt)*sin(lat));
}
// Second pass: rotate ECEF to local ENU using rotation matrix from origin LLA
```

[Source: Bowring "Transformation from Spatial to Geographic Coordinates", Survey Review 1976; EPSG:4978 WGS84 specification]

### 74.2 GPU LIDAR Point Cloud Decimation

Reduce massive LIDAR datasets (billions of points) to a renderable density using a GPU voxel-grid filter: keep one point per grid cell, selected by highest return intensity:

```glsl
// lidar_decimate.comp — voxel-grid decimation of LIDAR point cloud
layout(set=0,binding=0) readonly buffer Points { vec4 pts[]; };  // xyz + intensity
layout(set=0,binding=1) buffer VoxelBest { vec4 best[]; };       // best point per voxel
layout(set=0,binding=2) buffer VoxelInt  { float bestInt[]; };   // best intensity per voxel
layout(push_constant) uniform PC { vec3 gridMin; float cellSize; ivec3 dim; uint N; } pc;
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    if (p >= pc.N) return;
    ivec3 c = ivec3((pts[p].xyz - pc.gridMin) / pc.cellSize);
    if (any(lessThan(c,ivec3(0)))||any(greaterThanEqual(c,pc.dim))) return;
    uint idx = c.z*pc.dim.y*pc.dim.x + c.y*pc.dim.x + c.x;
    // Atomic max on intensity to select best return per cell
    atomicMax_float_idx(bestInt[idx], /* index field in best[] */ p, pts[p].w, p);
    // After: compact best[] to output cloud (see §72.2 prefix-scan pattern)
}
```

[Source: Rusu & Cousins "3D is Here: Point Cloud Library", ICRA 2011; Hornung et al. "OctoMap: An Efficient Probabilistic 3D Mapping Framework", Autonomous Robots 2013]

### 74.3 GPU Terrain Mesh Streaming with CDLOD

Continuous Distance-Dependent Level of Detail (CDLOD, Strugar 2010) generates terrain meshes from a height map hierarchy on-the-fly with no precomputed meshes. The GPU produces one quad-grid per visible tile at the appropriate LOD:

```glsl
// cdlod_terrain.vert — CDLOD terrain vertex: sample heightmap with LOD blend
layout(push_constant) uniform PC { vec2 tileOffset; float tileScale; int lod;
    vec3 cameraPos; mat4 mvp; } pc;
uniform sampler2D heightmapMip;   // mip chain of heightmap
in vec2 vertexUV;                 // grid vertex [0,1]²
void main() {
    vec2 worldXZ  = pc.tileOffset + vertexUV * pc.tileScale;
    // Sample two LOD levels and blend based on camera distance
    float h0 = textureLod(heightmapMip, worldXZ/TERRAIN_SIZE, float(pc.lod)).r;
    float h1 = textureLod(heightmapMip, worldXZ/TERRAIN_SIZE, float(pc.lod+1)).r;
    float dist   = length(vec2(worldXZ.x,worldXZ.y) - pc.cameraPos.xz);
    float blend  = clamp((dist - pc.tileScale*MORPH_START) / (pc.tileScale*MORPH_RANGE), 0.0, 1.0);
    float height = mix(h0, h1, blend) * MAX_HEIGHT;
    gl_Position  = pc.mvp * vec4(worldXZ.x, height, worldXZ.y, 1.0);
}
```

[Source: Strugar "Continuous Distance-Dependent Level of Detail for Rendering Heightmaps", JCGT 2010; Ulrich "Rendering Massive Terrains Using Chunked LOD", SIGGRAPH 2002]

### 74.4 GPU Terrain Normal and Slope Computation

Compute normals and slope maps from height map data for lighting, road gradient analysis, and erosion simulation (§57):

```glsl
// terrain_normals.comp — compute normals from heightmap via finite differences
layout(set=0,binding=0) uniform sampler2D heightmap;
layout(set=0,binding=1, rgba16_snorm) writeonly image2D normalMap;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    float hL  = texelFetchOffset(heightmap, pix, 0, ivec2(-1,0)).r;
    float hR  = texelFetchOffset(heightmap, pix, 0, ivec2( 1,0)).r;
    float hD  = texelFetchOffset(heightmap, pix, 0, ivec2(0,-1)).r;
    float hU  = texelFetchOffset(heightmap, pix, 0, ivec2(0, 1)).r;
    vec3  n   = normalize(vec3(hL-hR, 2.0*CELL_SIZE/MAX_HEIGHT, hD-hU));
    imageStore(normalMap, pix, vec4(n*0.5+0.5, 0.0));
}
```

[Source: Strugar 2010; Turkowski "Filters for Common Resampling Tasks", Graphics Gems 1990]

---


---

## IX. Ray Tracing and Optical Geometry

Ray tracing unifies shadowing, reflection, ambient occlusion, global illumination, and caustics under a single BVH traversal abstraction. This category covers the Vulkan ray tracing pipeline (ray generation, closest-hit, any-hit, miss shaders), full path-tracer construction, shadow geometry algorithms viewed from a geometry perspective (shadow volumes, shadow map ray casting), IBL preprocessing (importance sampling, SH projection from environment maps), radiosity form-factor computation via hemicube or GPU ray casting, atmospheric scattering geometry (Rayleigh/Mie phase functions in world space), screen-space reflection ray generation and reprojection, subsurface scattering geometry (dipole and BSSRDF volumetric model), and order-independent transparency as an optical compositing problem. GPU polygon clipping is included here as a core primitive for software rasterizers and ray-geometry intersection pipelines.

### 75. Ray Tracing Geometry Pipeline

*Audience: graphics application developers, systems developers.*

Vulkan ray tracing (VK_KHR_ray_tracing_pipeline, core in Vulkan 1.2+ with KHR extensions) organises geometry into a two-level hierarchy. §24 covered LBVH *construction* as a compute algorithm; this section covers the Vulkan RT *pipeline* as a geometry-consumption interface — acceleration structure management, procedural geometry via intersection shaders, and animated geometry update strategies.

### 75.1 Acceleration Structure Fundamentals: BLAS and TLAS

- **BLAS (Bottom-Level Acceleration Structure):** Contains actual geometry — triangle meshes or AABBs for procedural geometry. Built once per unique mesh shape.
- **TLAS (Top-Level Acceleration Structure):** Contains instances of BLASes with per-instance 3×4 transforms, visibility masks, and shader binding table offsets. Rebuilt each frame for animated scenes.

Queries traverse TLAS → BLAS → geometry test. RT cores (NVIDIA) and BVH hardware (AMD RDNA3+) handle traversal; application provides shader callbacks at each intersection event.

### 75.2 Building BLAS from Vertex/Index Buffers

```cpp
VkAccelerationStructureGeometryKHR geom{};
geom.sType        = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_KHR;
geom.geometryType = VK_GEOMETRY_TYPE_TRIANGLES_KHR;
geom.flags        = VK_GEOMETRY_OPAQUE_BIT_KHR;
auto& tri = geom.geometry.triangles;
tri.sType         = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_TRIANGLES_DATA_KHR;
tri.vertexFormat  = VK_FORMAT_R32G32B32_SFLOAT;
tri.vertexData.deviceAddress = vertexBufferDeviceAddress;
tri.vertexStride  = sizeof(Vertex);
tri.maxVertex     = vertexCount - 1;
tri.indexType     = VK_INDEX_TYPE_UINT32;
tri.indexData.deviceAddress  = indexBufferDeviceAddress;

VkAccelerationStructureBuildGeometryInfoKHR buildInfo{};
buildInfo.sType         = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_GEOMETRY_INFO_KHR;
buildInfo.type          = VK_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL_KHR;
buildInfo.flags         = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_TRACE_BIT_KHR;
buildInfo.geometryCount = 1;
buildInfo.pGeometries   = &geom;

VkAccelerationStructureBuildSizesInfoKHR sizeInfo{};
uint32_t primCount = triangleCount;
vkGetAccelerationStructureBuildSizesKHR(device,
    VK_ACCELERATION_STRUCTURE_BUILD_TYPE_DEVICE_KHR,
    &buildInfo, &primCount, &sizeInfo);
// Allocate result and scratch buffers, create AS handle, then:
buildInfo.mode = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR;
buildInfo.dstAccelerationStructure  = blas;
buildInfo.scratchData.deviceAddress = scratchAddress;
VkAccelerationStructureBuildRangeInfoKHR range{triangleCount, 0, 0, 0};
const auto* pRange = &range;
vkCmdBuildAccelerationStructuresKHR(cmd, 1, &buildInfo, &pRange);
```

For multiple submeshes in one BLAS, pass an array of `VkAccelerationStructureGeometryKHR` and corresponding `VkAccelerationStructureBuildRangeInfoKHR` — one per submesh, different hit group offsets for different materials. [Source: Vulkan Spec §37.9; NVIDIA Vulkan Ray Tracing Tutorial https://nvpro-samples.github.io/vk_raytracing_tutorial_KHR/]

### 75.3 TLAS and Instance Transforms

```cpp
VkAccelerationStructureInstanceKHR inst{};
memcpy(inst.transform.matrix, glm::value_ptr(glm::transpose(modelMatrix)),
       sizeof(inst.transform.matrix));          // row-major 3×4
inst.instanceCustomIndex                    = objectID;          // gl_InstanceCustomIndexEXT
inst.mask                                   = 0xFF;
inst.instanceShaderBindingTableRecordOffset = hitGroupIndex;
inst.flags = VK_GEOMETRY_INSTANCE_TRIANGLE_FACING_CULL_DISABLE_BIT_KHR;
inst.accelerationStructureReference         = blasDeviceAddress;
```

For dynamic scenes, rebuild the TLAS each frame (instances are just a flat buffer of 64-byte structs — fast) while BLASes stay fixed unless mesh topology changes. For skinned meshes use BLAS refit (§75.5) then TLAS rebuild.

### 75.4 Procedural Geometry via Intersection Shaders

Geometry that cannot be represented as triangle soups (analytic spheres, SDF primitives, Bézier patches) uses AABB-based BLASes and a custom intersection shader. The AABB defines the bounding volume; the intersection shader computes the exact hit point:

```glsl
// sphere.rint — analytic sphere intersection
#version 460
#extension GL_EXT_ray_tracing : require

struct SphereData { vec3 center; float radius; };
layout(set=0,binding=2) readonly buffer Spheres { SphereData spheres[]; };

void main() {
    SphereData s = spheres[gl_PrimitiveID];
    vec3  oc = gl_WorldRayOriginEXT - s.center;
    float a  = dot(gl_WorldRayDirectionEXT, gl_WorldRayDirectionEXT);
    float b  = 2.0 * dot(oc, gl_WorldRayDirectionEXT);
    float c  = dot(oc, oc) - s.radius * s.radius;
    float disc = b*b - 4.0*a*c;
    if (disc < 0.0) return;
    float t = (-b - sqrt(disc)) / (2.0 * a);
    if (t < gl_RayTminEXT || t > gl_RayTmaxEXT) {
        t = (-b + sqrt(disc)) / (2.0 * a);
        if (t < gl_RayTminEXT || t > gl_RayTmaxEXT) return;
    }
    reportIntersectionEXT(t, 0u);
}
```

SDF sphere tracing (§3.7) runs as a hybrid: coarse BVH traversal from RT hardware, fine sphere-march inside the custom intersection shader — combining hardware AS traversal efficiency with analytical SDF accuracy. [Source: Vulkan Spec §14.7 Ray Tracing Shaders; Quilez "Raymarching Signed Distance Fields"]

### 75.5 BLAS Refit for Animated Geometry

When vertex positions change (skinning, cloth, fluid surface) but triangle topology stays constant, **refit** updates the BLAS tree in O(N) rather than rebuilding in O(N log N):

```cpp
// First frame: build with ALLOW_UPDATE flag
buildInfo.flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_BUILD_BIT_KHR |
                  VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_UPDATE_BIT_KHR;
buildInfo.mode  = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR;
// ...build...

// Each subsequent frame after skinning pre-pass (§39.4):
buildInfo.mode    = VK_BUILD_ACCELERATION_STRUCTURE_MODE_UPDATE_KHR;
buildInfo.srcAccelerationStructure = blas;   // refit in place
buildInfo.dstAccelerationStructure = blas;
vkCmdBuildAccelerationStructuresKHR(cmd, 1, &buildInfo, &pRange);
```

Refit traverses the existing BLAS bottom-up recomputing AABBs from updated vertex positions without re-sorting primitives. Quality degrades over many frames as the tree drifts from optimal. A common strategy: refit for K frames, then full rebuild every K+1th frame. [Source: Wyman et al. "Introduction to DirectX Raytracing" §3; NVIDIA Vulkan RT best practices]

---

### 76. Shadow Geometry Algorithms

*Audience: graphics application developers.*

Shadows are the most geometry-intensive secondary effect in real-time rendering. Each technique represents a different geometry computation on the GPU: CSM partitions the view frustum, shadow volumes extrude silhouette edges, VSM filters depth maps, and RT shadows cast rays against the scene TLAS.

### 76.1 Cascaded Shadow Maps: Frustum Partitioning

CSM splits the view frustum into N subfrusta, each rendered to its own shadow map at a resolution appropriate for that depth range. GPU frustum split computation:

```glsl
// csm_splits.comp — compute N cascade split depths
layout(set=0,binding=0) writeonly buffer Splits { float splitDepths[MAX_CASCADES+1]; };
layout(push_constant) uniform PC {
    float zNear; float zFar; int N; float lambda;  // lambda blends log/uniform
} pc;
layout(local_size_x=1) in;
void main() {
    splitDepths[0] = pc.zNear;
    splitDepths[pc.N] = pc.zFar;
    for (int i=1; i<pc.N; i++) {
        float fi  = float(i) / float(pc.N);
        float log = pc.zNear * pow(pc.zFar/pc.zNear, fi);
        float uni = pc.zNear + (pc.zFar - pc.zNear) * fi;
        splitDepths[i] = mix(uni, log, pc.lambda);
    }
}
```

Each cascade renders the scene with a tight ortho projection enclosing the cascade subfustum intersected with the scene AABB. [Source: Engel "Cascaded Shadow Maps", ShaderX 5, 2006]

### 76.2 Shadow Volume Stencil: Silhouette Detection

GPU shadow volumes (Everitt & Kilgard "Robust Shadow Volumes" 2002) require detecting silhouette edges — edges where one adjacent face is front-lit and the other is back-lit. Compute silhouette in a pre-pass:

```glsl
// silhouette.comp — detect silhouette edges
layout(set=0,binding=0) readonly buffer Edges { EdgeAdj edges[]; };  // each edge: v0,v1,f0,f1
layout(set=0,binding=1) readonly buffer FaceNormals { vec3 fNorm[]; };
layout(set=0,binding=2) writeonly buffer Silhouette { SilEdge sil[]; };
layout(set=0,binding=3) buffer SilCount { uint count; };
layout(push_constant) uniform PC { vec3 lightPos; } pc;
layout(local_size_x=64) in;
void main() {
    uint e = gl_GlobalInvocationID.x;
    bool f0lit = dot(fNorm[edges[e].f0], pc.lightPos - edgeMidpoint(edges[e])) > 0.0;
    bool f1lit = dot(fNorm[edges[e].f1], pc.lightPos - edgeMidpoint(edges[e])) > 0.0;
    if (f0lit != f1lit) {
        uint slot = atomicAdd(count, 1u);
        sil[slot] = SilEdge(edges[e].v0, edges[e].v1, f0lit ? edges[e].f0 : edges[e].f1);
    }
}
```

Then extrude silhouette edges to infinity in a mesh shader or vertex shader, render front/back faces into stencil with zfail. [Source: Everitt & Kilgard "Practical and Robust Stenciled Shadow Volumes", 2002]

### 76.3 VSM / EVSM: Moment Shadow Maps

Variance Shadow Maps (Donnelly & Lauritzen 2006) store E[d] and E[d²] in a two-channel shadow map, enabling hardware PCF filtering and Gaussian blur for soft shadows:

```glsl
// shadow_depth.frag — write moments to VSM
layout(location=0) out vec2 moments;
void main() {
    float d  = gl_FragCoord.z;
    moments  = vec2(d, d*d);
    // Bias: moments.y += 0.25*(dFdx(d)*dFdx(d) + dFdy(d)*dFdy(d))
}
// shadow_sample.glsl — Chebyshev upper bound
float chebyshev(vec2 moments, float d) {
    if (d <= moments.x) return 1.0;
    float var = moments.y - moments.x*moments.x;
    var       = max(var, 1e-5);
    float delta = d - moments.x;
    return var / (var + delta*delta);
}
```

EVSM (Annen et al. 2008) extends to 4 moments (positive/negative exponential warp) to eliminate light bleeding. [Source: Donnelly & Lauritzen "Variance Shadow Maps", I3D 2006]

### 76.4 Ray-Traced Shadows via VK_KHR_ray_query

Inline ray queries against the scene TLAS (§75.3) produce ground-truth shadow masks in a deferred lighting pass:

```glsl
// rt_shadow.comp
#extension GL_EXT_ray_query : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) readonly buffer GBufPos { vec4 gPos[]; };
layout(set=0,binding=2, r8) uniform writeonly image2D shadowMask;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  p    = gPos[pix.y*WIDTH+pix.x].xyz;
    vec3  ldir = normalize(lightPos - p);
    float llen = length(lightPos - p);
    rayQueryEXT rq;
    rayQueryInitializeEXT(rq, tlas,
        gl_RayFlagsTerminateOnFirstHitEXT | gl_RayFlagsSkipClosestHitShaderEXT,
        0xFF, p + ldir*0.002, 0.001, ldir, llen - 0.01);
    rayQueryProceedEXT(rq);
    float shadow = (rayQueryGetIntersectionTypeEXT(rq,true)
                    == gl_RayQueryCommittedIntersectionNoneEXT) ? 1.0 : 0.0;
    imageStore(shadowMask, pix, vec4(shadow));
}
```

[Source: Vulkan Spec VK_KHR_ray_query; Wyman "A Gentle Introduction to DirectX Raytracing", SIGGRAPH 2018]

---

### 77. IBL and Environment Map Preprocessing

*Audience: graphics application developers.*

Image-Based Lighting (IBL) precomputes the integral of incoming radiance over a hemisphere, storing the result in two GPU textures: a prefiltered radiance map (PMREM) and a BRDF integration LUT. A third preprocessing step projects the environment into L2 spherical harmonics for diffuse irradiance.

### 77.1 Spherical Harmonics Projection

Project an HDR cubemap into 9 L2 SH coefficients (one per colour channel × 9 basis functions). Each pixel contributes to all 9 bands:

```glsl
// sh_project.comp — SH projection from equirectangular HDR
layout(set=0,binding=0) uniform sampler2D hdrEnv;
layout(set=0,binding=1) buffer SHCoeffs { vec3 sh[9]; };  // initialise to 0 before dispatch
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 id  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv  = (vec2(id)+0.5) / vec2(ENV_W, ENV_H);
    float phi = uv.x * 2.0*3.14159265;
    float theta = uv.y * 3.14159265;
    vec3  dir = vec3(sin(theta)*cos(phi), cos(theta), sin(theta)*sin(phi));
    float sinT = sin(theta);
    vec3  col = texture(hdrEnv, uv).rgb * sinT;   // solid angle weight
    // L2 SH basis functions
    float Y[9];
    Y[0] = 0.282095;
    Y[1] = 0.488603*dir.y; Y[2] = 0.488603*dir.z; Y[3] = 0.488603*dir.x;
    Y[4] = 1.092548*dir.x*dir.y; Y[5] = 1.092548*dir.y*dir.z;
    Y[6] = 0.315392*(3.0*dir.z*dir.z-1.0);
    Y[7] = 1.092548*dir.x*dir.z; Y[8] = 0.546274*(dir.x*dir.x-dir.y*dir.y);
    for (int k=0; k<9; k++) atomicAdd_vec3(sh[k], col*Y[k]);
}
// Normalise by (4π / N_pixels) after dispatch
```

[Source: Ramamoorthi & Hanrahan "An Efficient Representation for Irradiance Environment Maps", SIGGRAPH 2001]

### 77.2 PMREM: Prefiltered Radiance Mip Chain

Each mip level stores the GGX-filtered radiance for a different roughness value. Importance-sample the GGX NDF to generate sample directions, then accumulate:

```glsl
// pmrem.comp — one mip level, roughness = u_roughness
layout(set=0,binding=0) uniform samplerCube envCube;
layout(set=0,binding=1, rgba16f) uniform writeonly imageCube pmrem;
layout(push_constant) uniform PC { float roughness; uint mipSize; uint numSamples; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec3 id  = ivec3(gl_GlobalInvocationID);
    vec3  N   = cubeFaceDir(id.z, (vec2(id.xy)+0.5)/float(pc.mipSize));
    vec3  col = vec3(0); float totalW = 0.0;
    for (uint i=0; i<pc.numSamples; i++) {
        vec2  xi  = hammersley(i, pc.numSamples);
        vec3  H   = importanceSampleGGX(xi, N, pc.roughness);
        vec3  L   = normalize(2.0*dot(N,H)*H - N);
        float NdL = max(dot(N,L), 0.0);
        if (NdL > 0.0) {
            col    += texture(envCube, L).rgb * NdL;
            totalW += NdL;
        }
    }
    imageStore(pmrem, id, vec4(col / max(totalW, 1e-5), 1.0));
}
```

[Source: Karis "Real Shading in Unreal Engine 4", SIGGRAPH 2013]

### 77.3 BRDF Integration LUT

Precompute the split-sum BRDF integral for all (NdotV, roughness) pairs into a 2D R16G16 LUT:

```glsl
// brdf_lut.comp — split-sum BRDF integration
layout(set=0,binding=0, rg16f) uniform writeonly image2D brdfLUT;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    vec2  uv        = (vec2(gl_GlobalInvocationID.xy)+0.5) / vec2(LUT_SIZE);
    float NdotV     = uv.x;
    float roughness = uv.y;
    vec3  V = vec3(sqrt(1.0-NdotV*NdotV), 0.0, NdotV);
    float A = 0.0, B = 0.0;
    for (uint i=0; i<NUM_SAMPLES; i++) {
        vec2  xi  = hammersley(i, NUM_SAMPLES);
        vec3  H   = importanceSampleGGX(xi, vec3(0,0,1), roughness);
        vec3  L   = normalize(2.0*dot(V,H)*H - V);
        float NdL = max(L.z, 0.0);
        float NdH = max(H.z, 0.0);
        float VdH = max(dot(V,H), 0.0);
        if (NdL > 0.0) {
            float G   = G_SmithGGX(NdotV, NdL, roughness);
            float Gv  = G * VdH / max(NdH * NdotV, 1e-5);
            float Fc  = pow(1.0-VdH, 5.0);
            A += (1.0-Fc)*Gv; B += Fc*Gv;
        }
    }
    imageStore(brdfLUT, ivec2(gl_GlobalInvocationID.xy), vec4(A,B,0,0)/float(NUM_SAMPLES));
}
```

At runtime: `Lr = pmrem.sample(R, roughness) * (F0*A + B)` where A,B are from the LUT. [Source: Karis 2013; Lagarde & de Rousiers "Moving Frostbite to PBR", SIGGRAPH 2014]

---

### 78. Global Illumination Geometry

*Audience: graphics application developers.*

Global illumination algorithms compute indirect lighting — light that bounces one or more times before reaching the eye. Three GPU geometry methods dominate real-time GI: Reflective Shadow Maps (RSM) sample the first indirect bounce from shadow map geometry; Light Propagation Volumes (LPV) propagate SH-encoded radiance through a 3D grid; Voxel Cone Tracing (VXGI) traces cones through a sparse voxel scene representation.

### 78.1 Reflective Shadow Maps

RSM (Dachsbacher & Stamminger 2005) treats each shadow map texel as a virtual point light (VPL) for the first indirect bounce. Each G-buffer pixel gathers from a random subset of RSM VPLs:

```glsl
// rsm_gather.comp — indirect irradiance from RSM VPLs
layout(set=0,binding=0) uniform sampler2D rsmPos;    // world-space position in RSM
layout(set=0,binding=1) uniform sampler2D rsmNorm;   // surface normal in RSM
layout(set=0,binding=2) uniform sampler2D rsmFlux;   // reflected flux in RSM
layout(set=0,binding=3) readonly buffer GBufPos  { vec4 gPos[]; };
layout(set=0,binding=4) readonly buffer GBufNorm { vec4 gNorm[]; };
layout(set=0,binding=5, rgba16f) uniform writeonly image2D indirectOut;
layout(set=0,binding=6) readonly buffer RSMSamples { vec2 samples[N_RSM_SAMPLES]; };
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  x   = gPos[pix.y*WIDTH+pix.x].xyz;
    vec3  n   = gNorm[pix.y*WIDTH+pix.x].xyz;
    vec3  E   = vec3(0.0);
    for (uint s=0; s<N_RSM_SAMPLES; s++) {
        vec2  uv   = samples[s];
        vec3  xp   = texture(rsmPos,  uv).xyz;
        vec3  np   = texture(rsmNorm, uv).xyz;
        vec3  flux = texture(rsmFlux, uv).rgb;
        vec3  d    = x - xp;
        float dist2= max(dot(d,d), 1e-4);
        float form = max(0.0, dot(np, d)) * max(0.0, dot(n, -d));
        E += flux * form / (dist2 * dist2);
    }
    imageStore(indirectOut, pix, vec4(E / float(N_RSM_SAMPLES), 1.0));
}
```

[Source: Dachsbacher & Stamminger "Reflective Shadow Maps", I3D 2005]

### 78.2 Light Propagation Volumes

LPV (Kaplanyan & Dachsbacher 2010) propagates SH-encoded radiance through a 3D grid of 32×32×32 cells in 8 GPU passes (one per direction of the 6-faced propagation kernel plus diagonals):

```glsl
// lpv_propagate.comp — one LPV propagation step
layout(set=0,binding=0) readonly buffer LPVIn  { vec4 shR_in[];  vec4 shG_in[];  vec4 shB_in[];  };
layout(set=0,binding=1) writeonly buffer LPVOut { vec4 shR_out[]; vec4 shG_out[]; vec4 shB_out[]; };
layout(push_constant) uniform PC { ivec3 gridDim; } pc;
layout(local_size_x=8,local_size_y=8,local_size_z=8) in;
const ivec3 FACES[6] = {ivec3(1,0,0),ivec3(-1,0,0),ivec3(0,1,0),
                         ivec3(0,-1,0),ivec3(0,0,1),ivec3(0,0,-1)};
void main() {
    ivec3 v   = ivec3(gl_GlobalInvocationID);
    uint  out = flatIdx(v);
    vec4  accR = vec4(0), accG = vec4(0), accB = vec4(0);
    for (int f=0; f<6; f++) {
        ivec3 nb = v - FACES[f];
        if (any(lessThan(nb,ivec3(0))) || any(greaterThanEqual(nb,pc.gridDim))) continue;
        uint src = flatIdx(nb);
        // Rotate SH coefficients to align FACES[f] direction, scale by solid angle
        mat4 R = shRotation(vec3(FACES[f]));
        accR += R * shR_in[src] * SH_SOLID_ANGLE;
        accG += R * shG_in[src] * SH_SOLID_ANGLE;
        accB += R * shB_in[src] * SH_SOLID_ANGLE;
    }
    shR_out[out] = accR; shG_out[out] = accG; shB_out[out] = accB;
}
```

[Source: Kaplanyan & Dachsbacher "Cascaded Light Propagation Volumes for Real-Time Indirect Illumination", I3D 2010]

### 78.3 Voxel Cone Tracing (VXGI)

VXGI (Crassin et al. 2011) stores a sparse voxel octree (§27.3) of scene radiance, then traces cones through it using anisotropic voxel mip-maps. The cone traces N levels of the SVO hierarchy to compute indirect diffuse and specular:

```glsl
// vxgi_cone.glsl — diffuse cone trace for one fragment
vec3 traceConeDiffuse(vec3 origin, vec3 normal, float aperture) {
    vec3 accum    = vec3(0.0);
    float occlusion = 0.0;
    float dist    = VOXEL_SIZE;
    while (dist < MAX_DIST && occlusion < 1.0) {
        float diameter = max(VOXEL_SIZE, 2.0 * aperture * dist);
        float mip      = log2(diameter / VOXEL_SIZE);
        vec3  pos      = origin + normal * dist;
        vec4  voxel    = textureLod(voxelSVO, worldToSVO(pos), mip);
        accum      += (1.0 - occlusion) * voxel.a * voxel.rgb;
        occlusion  += (1.0 - occlusion) * voxel.a;
        dist       += diameter * CONE_STEP;
    }
    return accum;
}
void main() {
    vec3 indirect = vec3(0.0);
    // Hemispherical diffuse: 5 cones at aperture 60°
    for (int c=0; c<5; c++)
        indirect += traceConeDiffuse(worldPos+normal*VOXEL_SIZE, coneDir[c], radians(60.0));
    indirect /= 5.0;
}
```

[Source: Crassin et al. "Interactive Indirect Illumination Using Voxel Cone Tracing", CGF 2011; NVIDIA VXGI SDK]

### 78.4 GI Denoising Geometry Buffer

All three GI methods produce noisy output. Spatiotemporal denoising (SVGF, Schied et al. 2017) uses a geometry-guided bilateral filter with 3×3 À-trous wavelet passes, rejecting samples across depth and normal discontinuities:

```glsl
// svgf_atrous.comp — one À-trous wavelet pass
layout(set=0,binding=0) uniform sampler2D colorIn;
layout(set=0,binding=1) uniform sampler2D gDepth;
layout(set=0,binding=2) uniform sampler2D gNormal;
layout(set=0,binding=3, rgba16f) uniform writeonly image2D colorOut;
layout(push_constant) uniform PC { int stepWidth; float phiColor; float phiNormal; float phiDepth; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec4  cen = texelFetch(colorIn, pix, 0);
    vec3  nc  = texelFetch(gNormal, pix, 0).xyz;
    float dc  = texelFetch(gDepth,  pix, 0).r;
    vec4  sum = vec4(0.0); float wSum = 0.0;
    const int r = 2;
    for (int dy=-r; dy<=r; dy++) for (int dx=-r; dx<=r; dx++) {
        ivec2 nb  = pix + ivec2(dx,dy)*pc.stepWidth;
        vec4  cn  = texelFetch(colorIn, nb, 0);
        float wC  = exp(-dot(cen.rgb-cn.rgb,cen.rgb-cn.rgb)/pc.phiColor);
        float wN  = pow(max(0.0,dot(nc,texelFetch(gNormal,nb,0).xyz)),pc.phiNormal);
        float wD  = exp(-abs(dc-texelFetch(gDepth,nb,0).r)/pc.phiDepth);
        float w   = wC*wN*wD*kernel[dy+r][dx+r];
        sum += cn*w; wSum += w;
    }
    imageStore(colorOut, pix, sum/max(wSum,1e-5));
}
```

[Source: Schied et al. "Spatiotemporal Variance-Guided Filtering", HPG 2017]

---

### 79. Geometric Optics: Lens, Caustics, and Beam Tracing

*Audience: graphics application developers.*

Geometric optics models light as rays that refract at dielectric surfaces and focus into caustics. GPU implementations use the scene TLAS for refraction ray tracing, photon mapping for caustic density estimation, and beam tracing for specular-specular transport.

### 79.1 Lens Flare Geometry

GPU lens flares place starburst and halo sprites along the screen-space line from light source to screen centre. A depth-aware occlusion test determines flare intensity:

```glsl
// lensflare.comp — compute flare element positions and intensities
layout(set=0,binding=0) readonly buffer LightSrc { vec4 lightScreenPos; };
layout(set=0,binding=1) uniform sampler2D depthBuf;
layout(set=0,binding=2) writeonly buffer Flares { FlareElement flares[MAX_ELEMENTS]; };
layout(local_size_x=1) in;
void main() {
    vec2  lsp   = lightScreenPos.xy;           // NDC light position
    float vis   = 0.0;
    const int   S = 16;
    for (int i=0; i<S; i++) {
        vec2 jit = disk16[i] * FLARE_OCCLUSION_RADIUS;
        vec2 uv  = lsp*0.5+0.5 + jit;
        float sd = texture(depthBuf, uv).r;
        float ld = lightScreenPos.w;           // light depth in NDC
        if (sd >= ld - 1e-3) vis += 1.0/float(S);
    }
    vec2  axis = vec2(0.5) - lsp*0.5+0.5;
    for (uint e=0; e<N_FLARE_ELEMENTS; e++) {
        flares[e].pos    = (lsp*0.5+0.5) + axis * flareOffset[e];
        flares[e].scale  = flareScale[e] * vis;
        flares[e].tileID = flareTile[e];
    }
}
```

[Source: Mittring "Finding Next Gen — CryEngine 2", SIGGRAPH 2007; Jimenez et al. "Practical Real-Time Lens-Flare Rendering", CGF 2012]

### 79.2 Caustic Photon Mapping

Caustics (focused light through glass/water) are computed via bidirectional photon tracing. Photons are emitted from the light source, refracted through dielectrics using Snell's law, and collected on diffuse surfaces:

```glsl
// caustic_photon.raygen.glsl
layout(location=0) rayPayloadEXT struct { vec3 power; vec3 dir; uint depth; } pload;
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1) buffer Photons { Photon photons[]; };
layout(set=0,binding=2) buffer PhotonCount { uint count; };
void main() {
    uint idx = gl_LaunchIDEXT.x;
    pload.power = lightColor * PHOTON_POWER;
    pload.dir   = uniformSampleCone(lightDir, lightAngle, halton2D(idx));
    pload.depth = 0u;
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF,
                0, 1, 0, lightPos, 0.001, pload.dir, MAX_DIST, 0);
}
// caustic_chit.glsl — refract at dielectric, deposit photon on diffuse surface
// At dielectric: compute Fresnel ratio, refract via Snell's law, recurse
// At diffuse surface: store photon position + power in buffer
```

[Source: Jensen "Realistic Image Synthesis Using Photon Mapping", 2001; GPU photon mapping: Hachisuka et al. "Progressive Photon Mapping", SIGGRAPH Asia 2008]

### 79.3 Caustic Density Estimation

After tracing N photons, estimate caustic irradiance at each surface point via GPU k-NN in the photon buffer (§87.1 pattern) and kernel density estimation:

```glsl
// caustic_kde.comp — k-NN density estimation at G-buffer pixels
layout(set=0,binding=0) readonly buffer Photons { Photon photons[]; };
layout(set=0,binding=1) readonly buffer GBufPos { vec4 gPos[]; };
layout(set=0,binding=2, rgba16f) uniform writeonly image2D causticOut;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    vec3  p   = gPos[pix.y*WIDTH+pix.x].xyz;
    // Find K nearest photons (brute force for small N, §87.1 k-d tree for large)
    float r2  = KDE_RADIUS*KDE_RADIUS;
    vec3  E   = vec3(0.0);
    for (uint i=0; i<N_PHOTONS; i++) {
        float d2 = dot(photons[i].pos-p, photons[i].pos-p);
        if (d2 < r2) E += photons[i].power * (1.0 - sqrt(d2/r2));  // cone kernel
    }
    imageStore(causticOut, pix, vec4(E / (3.14159*r2), 1.0));
}
```

[Source: Jensen 2001; Zhu et al. "Real-time Caustics Using Photon Mapping on GPU", 2007]

---

### 80. Screen-Space Reflections (SSR/SSSR)

*Audience: graphics application developers.*

Screen-space reflections (SSR) trace reflection rays against the depth buffer using hierarchical Z-buffer ray marching, providing real-time specular GI for surfaces visible on screen. Stochastic SSR (SSSR) adds per-pixel random ray selection and temporal accumulation to handle rough surfaces with variable BRDF lobe widths.

### 80.1 Hierarchical Z-Buffer Ray Marching

Build a mip chain of the depth buffer where each level stores the maximum depth in each 2×2 tile. March the reflection ray through increasing mip levels, stepping coarsely over empty space:

```glsl
// ssr_hiz_march.comp — hierarchical Z ray march for reflection
layout(set=0,binding=0) uniform sampler2D hiZPyramid;  // max-depth mip chain
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2) uniform sampler2D gDepth;
layout(set=0,binding=3, rgba16f) uniform writeonly image2D ssrOut;
layout(push_constant) uniform PC { mat4 proj; mat4 invProj; mat4 invView; float maxDist; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  N    = texelFetch(gNormal, pix, 0).xyz;
    float d    = texelFetch(gDepth,  pix, 0).r;
    vec3  P    = viewSpacePos(pix, d, pc.invProj);
    vec3  V    = normalize(-P);
    vec3  R    = reflect(-V, N);
    // March R through HiZ pyramid
    vec3  Q    = P; int mip = 0;
    vec4  hit  = vec4(0);
    for (int i=0; i<64; i++) {
        Q += R * (0.01 * float(1<<mip));
        vec4  clip = pc.proj * vec4(Q,1); clip /= clip.w;
        vec2  uv   = clip.xy*0.5+0.5;
        float zHiZ = textureLod(hiZPyramid, uv, float(mip)).r;
        if (Q.z < zHiZ - 0.01) { if (mip==0) { hit=vec4(uv,Q.z,1); break; } mip=max(0,mip-1); }
        else mip = min(mip+1, MAX_MIP);
        if (uv.x<0||uv.x>1||uv.y<0||uv.y>1||Q.z>0) break;
    }
    imageStore(ssrOut, pix, hit);
}
```

[Source: McGuire & Mara "Efficient GPU Screen-Space Ray Tracing", JCGT 2014; Stachowiak "Stochastic All The Things" 2018]

### 80.2 Stochastic SSR with GGX Importance Sampling

SSSR (Stachowiak 2018) samples one ray per pixel per frame from the GGX BRDF lobe, giving rough-surface reflections at the cost of one sample per pixel plus temporal accumulation:

```glsl
// sssr_sample.comp — sample one GGX reflection ray per pixel
layout(set=0,binding=0) uniform sampler2D gRoughness;
layout(set=0,binding=1) readonly buffer BlueNoise { vec2 bn[]; };  // §105 blue noise
layout(set=0,binding=2, rgba16f) uniform writeonly image2D rayDirOut;
layout(push_constant) uniform PC { uint frameIdx; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float rough= texelFetch(gRoughness, pix, 0).r;
    uint  seed = (pix.y*WIDTH+pix.x + pc.frameIdx*WIDTH*HEIGHT) % BN_SIZE;
    vec2  xi   = bn[seed];
    // GGX importance sample: half-vector H from roughness
    float a    = rough*rough;
    float phi  = 2.0*3.14159*xi.x;
    float cosT = sqrt((1.0-xi.y)/(1.0+(a*a-1.0)*xi.y));
    float sinT = sqrt(1.0-cosT*cosT);
    vec3  H    = tangentToWorld(vec3(sinT*cos(phi), sinT*sin(phi), cosT), gNormal(pix));
    vec3  R    = reflect(-viewDir(pix), H);
    imageStore(rayDirOut, pix, vec4(R,rough));
}
```

[Source: Stachowiak 2018; Walter et al. "Microfacet Models for Refraction through Rough Surfaces", EGSR 2007]

### 80.3 Temporal Accumulation and Reprojection

Accumulate SSR samples over multiple frames by reprojecting the previous frame's result:

```glsl
// ssr_temporal.comp — temporal accumulation for SSR
layout(set=0,binding=0) uniform sampler2D ssrCurrent;   // this frame's SSR sample
layout(set=0,binding=1) uniform sampler2D ssrHistory;   // previous accumulated result
layout(set=0,binding=2) uniform sampler2D motionVec;    // §100.2 motion vectors
layout(set=0,binding=3, rgba16f) uniform writeonly image2D ssrAccum;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec2  uv   = (vec2(pix)+0.5)/vec2(imageSize(ssrAccum));
    vec2  mv   = texture(motionVec, uv).rg;
    vec4  cur  = texelFetch(ssrCurrent, pix, 0);
    vec4  hist = texture(ssrHistory, uv-mv);
    // Neighbourhood clamp to prevent ghosting
    vec4  mn   = vec4(1e30), mx = vec4(-1e30);
    for (int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        vec4 s = texelFetch(ssrCurrent,pix+ivec2(dx,dy),0);
        mn=min(mn,s); mx=max(mx,s);
    }
    hist       = clamp(hist, mn, mx);
    imageStore(ssrAccum, pix, mix(hist, cur, 0.1));
}
```

[Source: Karis "High Quality Temporal Supersampling", SIGGRAPH 2014; Stachowiak 2018]

### 80.4 SSR Fade and Composite

Fade reflections at screen edges (where the ray exits the screen) and blend with the specular IBL (§77) fallback:

```glsl
// ssr_composite.frag — blend SSR with IBL fallback
uniform sampler2D ssrAccum;    // §80.3 accumulated SSR
uniform sampler2D iblSpecular; // §77 IBL specular
uniform sampler2D gRoughness;
uniform sampler2D gMetallic;
in vec2 texUV;
void main() {
    vec4  ssr  = texture(ssrAccum, texUV);
    float conf = ssr.a;  // hit confidence (0=miss, 1=hit)
    // Fade at screen edges
    float edgeFade = 1.0;
    edgeFade *= smoothstep(0.0, 0.05, texUV.x) * smoothstep(1.0, 0.95, texUV.x);
    edgeFade *= smoothstep(0.0, 0.05, texUV.y) * smoothstep(1.0, 0.95, texUV.y);
    conf *= edgeFade;
    // Fade with roughness (SSR valid only for smooth surfaces)
    conf *= 1.0 - smoothstep(0.3, 0.6, texture(gRoughness,texUV).r);
    vec3  ibl  = texture(iblSpecular, texUV).rgb;
    fragColor  = vec4(mix(ibl, ssr.rgb, conf), 1.0);
}
```

[Source: McGuire & Mara 2014; Yasin "Screen Space Reflections in The Surge", GDC 2018]

---

### 81. GPU Radiosity: Form Factor Computation

*Audience: graphics application developers.*

Radiosity (Goral et al. 1984) computes diffuse inter-reflections by solving a linear system where the coefficient matrix contains *form factors* F_{ij} — the fraction of energy leaving patch i that arrives at patch j. GPU hemicube rendering computes form factors for all patches in a scene in parallel.

### 81.1 Hemicube Rendering for Form Factors

For each patch i, render the scene from i's centroid onto a hemicube (five 90° faces). The delta form factor ΔF_{ij} for each hemicube texel is a precomputed weight; the form factor F_{ij} is the sum over all texels covered by patch j:

```glsl
// hemicube_project.comp — project patches onto hemicube face, accumulate form factors
layout(set=0,binding=0) readonly buffer PatchCentroid { vec3 cen[];  };
layout(set=0,binding=1) readonly buffer PatchNormal   { vec3 nor[];  };
layout(set=0,binding=2) readonly buffer PatchArea     { float area[]; };
layout(set=0,binding=3) buffer FormFactor { float F[MAX_PATCHES][MAX_PATCHES]; };
layout(push_constant) uniform PC { uint srcPatch; uint hemiFace; uvec2 hemiRes; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    // Ray from patch centroid through hemicube texel
    vec3  rayDir = hemicubeRayDir(pix, pc.hemiFace, pc.hemiRes);
    float dff    = deltaFormFactor(pix, pc.hemiFace, pc.hemiRes);  // precomputed weight
    // Find which patch this ray hits (via BVH traverse §31)
    uint  hitPatch = bvhRayQuery(cen[pc.srcPatch] + nor[pc.srcPatch]*1e-3, rayDir);
    if (hitPatch == INVALID) return;
    atomicAdd_float(F[pc.srcPatch][hitPatch], dff);
}
```

[Source: Cohen & Greenberg "The Hemi-Cube: A Radiosity Solution for Complex Environments", SIGGRAPH 1985; GPU radiosity: Stamminger et al. "Hierarchical Solution for Radiosity on GPUs", EG 2002]

### 81.2 Progressive Radiosity Solve

The radiosity system B = E + ρ·F·B (B=radiosity, E=emission, ρ=reflectance, F=form factor matrix) is solved by iterative shooting: in each step, the patch with the most unshot energy broadcasts to all others:

```glsl
// radiosity_shoot.comp — shoot energy from max-energy patch to all others
layout(set=0,binding=0) buffer Radiosity { vec3 B[];       };  // per-patch radiosity
layout(set=0,binding=1) buffer Unshot    { vec3 unshot[];  };  // energy waiting to be shot
layout(set=0,binding=2) readonly buffer FormFactor { float F[MAX_PATCHES][MAX_PATCHES]; };
layout(set=0,binding=3) readonly buffer Reflectance { vec3 rho[]; };
layout(push_constant) uniform PC { uint srcPatch; float srcArea; } pc;
layout(local_size_x=64) in;
void main() {
    uint j  = gl_GlobalInvocationID.x;
    vec3 dB = rho[j] * F[pc.srcPatch][j] * unshot[pc.srcPatch] * pc.srcArea / area[j];
    B[j]       += dB;
    unshot[j]  += dB;
}
// CPU: after each pass, zero unshot[srcPatch], find next max-energy patch, repeat
```

[Source: Cohen et al. "A Progressive Refinement Approach for Fast Radiosity Image Generation", SIGGRAPH 1988; Neumann "Monte Carlo Radiosity", Computing 1995]

### 81.3 Hierarchical Radiosity

For large scenes, hierarchical radiosity (Hanrahan et al. 1991) subdivides patches and only computes form factors at the appropriate refinement level. GPU kernel: for each patch pair, test whether their form factor contribution exceeds a threshold; if so, subdivide:

```glsl
// hrad_refine.comp — decide whether patch pair (i,j) needs subdivision
layout(set=0,binding=0) readonly buffer PatchArea { float area[]; };
layout(set=0,binding=1) readonly buffer FormFactor { float F[][]; };
layout(set=0,binding=2) buffer RefineFlags { uint subdivide[]; };
layout(push_constant) uniform PC { float eps; } pc;
layout(local_size_x=64) in;
void main() {
    uint pair = gl_GlobalInvocationID.x;
    uint i = patchPairs[pair].x, j = patchPairs[pair].y;
    // BF oracle: if F[i][j] * B[i] * area[i] > eps → need finer form factor
    float BF = length(unshot[i]) * F[i][j] * area[i];
    if (BF > pc.eps) {
        atomicOr(subdivide[i], 1u);
        atomicOr(subdivide[j], 1u);
    }
}
```

[Source: Hanrahan et al. "A Rapid Hierarchical Radiosity Algorithm", SIGGRAPH 1991]

### 81.4 Irradiance Caching for Radiosity Rendering

Store per-vertex irradiance from the radiosity solve; interpolate at shade points using the irradiance cache (Ward & Heckbert 1992):

```glsl
// irradiance_cache.frag — lookup interpolated irradiance from radiosity patches
uniform sampler2D gWorldPos;
uniform sampler2D gNormal;
layout(set=0,binding=0) readonly buffer IrradCache { IrradSample samples[]; };
layout(set=0,binding=1) readonly buffer KDTree { uint kd[]; };
in vec2 texUV;
void main() {
    vec3  P  = texture(gWorldPos, texUV).xyz;
    vec3  N  = texture(gNormal,   texUV).xyz;
    vec3  E  = vec3(0.0); float wSum = 0.0;
    // k-NN lookup in irradiance cache (§37.1 pattern → BVH §31)
    for (int k=0; k<CACHE_K; k++) {
        IrradSample s = findKthNearest(kd, samples, P, k);
        float w = 1.0 / (length(P-s.pos) + s.Ri * (1.0-dot(N,s.nor)));
        E    += w * s.irrad;
        wSum += w;
    }
    fragColor = vec4(E/wSum * albedo, 1.0);
}
```

[Source: Ward & Heckbert "Irradiance Gradients", EGWR 1992; Krivánek et al. "Radiance Caching for Efficient Global Illumination Computation", IEEE TVCG 2005]

---

### 82. GPU Path Tracing Pipeline

*Audience: graphics application developers, systems developers.*

Unbiased Monte Carlo path tracing produces reference-quality global illumination by simulating light transport directly. GPU path tracing uses the TLAS/BLAS hierarchy of `VK_KHR_ray_tracing_pipeline` or ray queries, with multiple importance sampling (MIS) weighting direct-light and BRDF samples. The challenge is per-ray divergence: shader sorting (wavefront path tracing) groups coherent rays together before dispatch.

### 82.1 Primary Ray Generation and TLAS Traversal

```glsl
// pt_raygen.rgen — generate primary rays from a thin-lens camera model
#version 460
#extension GL_EXT_ray_tracing : require
layout(set=0,binding=0) uniform accelerationStructureEXT tlas;
layout(set=0,binding=1, rgba32f) uniform image2D accumBuffer;
layout(set=0,binding=2) readonly buffer BlueNoise { vec2 bn[]; };
layout(push_constant) uniform PC { mat4 invViewProj; vec3 camPos; float lensR; uint frame; } pc;
layout(location=0) rayPayloadEXT PathPayload payload;
void main() {
    ivec2 pix  = ivec2(gl_LaunchIDEXT.xy);
    uint  seed = (pix.y*gl_LaunchSizeEXT.x + pix.x + pc.frame*gl_LaunchSizeEXT.x*gl_LaunchSizeEXT.y) % BN_SIZE;
    vec2  jitter = bn[seed];
    vec2  uv   = (vec2(pix)+jitter) / vec2(gl_LaunchSizeEXT.xy) * 2.0 - 1.0;
    vec4  target = pc.invViewProj * vec4(uv, 1.0, 1.0); target /= target.w;
    // Thin-lens DoF
    vec2  lens  = concentricDiskSample(bn[(seed+1)%BN_SIZE]) * pc.lensR;
    vec3  fPoint = target.xyz;
    vec3  origin = pc.camPos + vec3(lens,0);
    vec3  dir    = normalize(fPoint - origin);
    payload.radiance = vec3(0); payload.throughput = vec3(1); payload.depth = 0;
    traceRayEXT(tlas, gl_RayFlagsNoneEXT, 0xFF, 0, 0, 0, origin, 1e-3, dir, 1e4, 0);
    vec4 prev  = imageLoad(accumBuffer, pix);
    imageStore(accumBuffer, pix, prev + vec4(payload.radiance, 1.0));
}
```

[Source: Pharr et al. "Physically Based Rendering", 4th ed. 2023, §16; Shirley et al. "Ray Tracing in One Weekend" series]

### 82.2 Closest-Hit Shader: MIS Direct Lighting

Multiple importance sampling (MIS) combines a direct-light sample and a BRDF sample using power heuristic weights to reduce variance:

```glsl
// pt_closesthit.rchit — shade surface point with MIS direct lighting + BRDF sample
#extension GL_EXT_ray_tracing : require
#extension GL_EXT_ray_query   : require
layout(location=0) rayPayloadInEXT PathPayload payload;
layout(location=1) rayPayloadEXT   ShadowPayload shadow;
hitAttributeEXT vec2 bary;
void main() {
    SurfacePoint sp = reconstructSurface(gl_PrimitiveID, bary);
    if (sp.emission != vec3(0)) { payload.radiance += payload.throughput * sp.emission; return; }
    // --- Direct light sample (NEE) ---
    LightSample ls = sampleLight(sp.pos, sp.nor, payload.seed);
    float pdfBRDF  = evalBRDFpdf(sp, ls.dir);
    float wLight   = misPowerHeuristic(ls.pdf, pdfBRDF);
    bool visible   = shadowRay(sp.pos, ls.dir, ls.dist);
    if (visible)
        payload.radiance += payload.throughput * evalBRDF(sp, ls.dir) * ls.Le * wLight / ls.pdf;
    // --- BRDF sample (continue path) ---
    vec3 wi; float pdfW;
    vec3 f   = sampleBRDF(sp, payload.seed, wi, pdfW);
    float pdfLight = lightPdf(sp.pos, wi);
    float wBRDF    = misPowerHeuristic(pdfW, pdfLight);
    payload.throughput *= f * abs(dot(wi,sp.nor)) * wBRDF / max(pdfW, 1e-8);
    // Russian roulette
    float q = max(payload.throughput.r, max(payload.throughput.g, payload.throughput.b));
    if (rand(payload.seed) > q) { payload.throughput=vec3(0); return; }
    payload.throughput /= q;
    // Spawn next ray
    payload.origin = sp.pos + sp.nor*1e-3; payload.dir = wi; payload.depth++;
}
```

[Source: Pharr et al. 2023 §12.2 MIS; Veach "Robust Monte Carlo Methods for Light Transport Simulation", PhD 1997]

### 82.3 Wavefront Path Tracing: Shader Sorting

GPU warp divergence kills throughput when adjacent paths hit different materials. Wavefront path tracing (Laine et al. 2013) uses separate queues per shader type. After intersection, paths are sorted by material ID and dispatched in homogeneous waves:

```glsl
// pt_sort_queues.comp — classify intersection records by material type
layout(set=0,binding=0) readonly buffer HitRecords { HitRecord hits[]; };
layout(set=0,binding=1) buffer Queue_Lambert  { uint q[]; uint cnt; };
layout(set=0,binding=2) buffer Queue_GGX      { uint q[]; uint cnt; };
layout(set=0,binding=3) buffer Queue_Glass    { uint q[]; uint cnt; };
layout(set=0,binding=4) buffer Queue_Miss     { uint q[]; uint cnt; };
layout(local_size_x=64) in;
void main() {
    uint p = gl_GlobalInvocationID.x;
    if (hits[p].miss) {
        uint s = atomicAdd(Queue_Miss.cnt, 1u); Queue_Miss.q[s]=p; return;
    }
    switch (hits[p].materialType) {
        case MAT_LAMBERT: { uint s=atomicAdd(Queue_Lambert.cnt,1u); Queue_Lambert.q[s]=p; break; }
        case MAT_GGX:     { uint s=atomicAdd(Queue_GGX.cnt,1u);     Queue_GGX.q[s]=p;     break; }
        case MAT_GLASS:   { uint s=atomicAdd(Queue_Glass.cnt,1u);    Queue_Glass.q[s]=p;   break; }
    }
}
// Each queue is then dispatched as a separate compute shader (material kernel)
```

[Source: Laine et al. "Megakernels Considered Harmful: Wavefront Path Tracing on GPUs", HPG 2013]

### 82.4 Denoising: SVGF À-trous Wavelet Filter

After accumulating N samples, SVGF (§78.4 pattern applied to path tracing) denoises via spatially-varying À-trous wavelets guided by geometry buffers:

```glsl
// svgf_atrous.comp — SVGF À-trous wavelet denoise pass k
layout(set=0,binding=0) uniform sampler2D colorIn;    // noisy accumulated radiance
layout(set=0,binding=1) uniform sampler2D gNormal;
layout(set=0,binding=2) uniform sampler2D gDepth;
layout(set=0,binding=3) uniform sampler2D gAlbedo;
layout(set=0,binding=4, rgba32f) writeonly image2D colorOut;
layout(push_constant) uniform PC { int stepWidth; float phiColor; float phiNorm; float phiDepth; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    vec3  cC   = texelFetch(colorIn,pix,0).rgb / texelFetch(gAlbedo,pix,0).rgb;  // modulate albedo
    vec3  nC   = texelFetch(gNormal,pix,0).xyz;
    float dC   = texelFetch(gDepth, pix,0).r;
    vec3  sum  = vec3(0); float wSum=0.0;
    const float kernel[3] = {3.0/8.0, 1.0/4.0, 1.0/16.0};
    for (int dy=-2;dy<=2;dy++) for(int dx=-2;dx<=2;dx++) {
        ivec2 nb   = pix + ivec2(dx,dy)*pc.stepWidth;
        vec3  cN   = texelFetch(colorIn,nb,0).rgb / texelFetch(gAlbedo,nb,0).rgb;
        float wC   = exp(-dot(cN-cC,cN-cC)/pc.phiColor);
        float wN   = pow(max(dot(nC,texelFetch(gNormal,nb,0).xyz),0.0),pc.phiNorm);
        float wD   = exp(-abs(dC-texelFetch(gDepth,nb,0).r)/pc.phiDepth);
        float h    = kernel[abs(dx)]*kernel[abs(dy)];
        float w    = h*wC*wN*wD; sum+=w*cN; wSum+=w;
    }
    imageStore(colorOut,pix,vec4(sum/wSum * texelFetch(gAlbedo,pix,0).rgb,1.0));
}
```

[Source: Schied et al. "Spatiotemporal Variance-Guided Filtering", HPG 2017; §51.4]

---

### 83. Atmospheric Scattering and Sky Rendering

*Audience: graphics application developers.*

Atmospheric scattering (Mie + Rayleigh) produces realistic sky colours, horizon glow, and aerial perspective. GPU implementations precompute transmittance and in-scatter into 2D/4D lookup tables (LUTs), then evaluate them at runtime with two texture samples — following Bruneton & Neyret (2008) and the Unreal/Unity production extensions.

### 83.1 Transmittance LUT Precomputation

For each (view zenith, altitude) pair, integrate extinction along the ray to space using the Chapman function approximation:

```glsl
// atmo_transmittance.comp — precompute transmittance LUT T(h, cosθ)
layout(set=0,binding=0, rg32f) writeonly image2D transmittanceLUT;  // (Rₑ, Rₘ) transmittance
layout(push_constant) uniform PC { float Rg; float Rt; float HR; float HM;
    float betaR; float betaM; int numSteps; } pc;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    vec2  uv  = (vec2(gl_GlobalInvocationID.xy)+0.5)/vec2(imageSize(transmittanceLUT));
    float h   = pc.Rg + uv.x*(pc.Rt-pc.Rg);          // altitude
    float cosT= uv.y * 2.0 - 1.0;                     // cos of view-zenith
    float dt  = atmosphericRayLength(h, cosT, pc.Rt) / float(pc.numSteps);
    vec2  tau = vec2(0.0);  // optical depth (Rayleigh, Mie)
    for (int i=0; i<pc.numSteps; i++) {
        float ri = h + (float(i)+0.5)*dt*cosT;
        tau.x   += exp(-(ri-pc.Rg)/pc.HR) * dt;
        tau.y   += exp(-(ri-pc.Rg)/pc.HM) * dt;
    }
    vec2 T = exp(-vec2(pc.betaR,pc.betaM) * tau);
    imageStore(transmittanceLUT, ivec2(gl_GlobalInvocationID.xy), vec4(T,0,0));
}
```

[Source: Bruneton & Neyret "Precomputed Atmospheric Scattering", EGSR 2008; Hillaire "A Scalable and Production Ready Sky and Atmosphere Rendering Technique", EGSR 2020]

### 83.2 Single-Scatter In-Scatter LUT

For each (altitude, view zenith, sun zenith) triple, integrate the single-scatter contribution (light reaching a point, scattering toward the viewer) using the transmittance LUT:

```glsl
// atmo_inscatter.comp — single in-scatter LUT (3D texture: altitude × viewZen × sunZen)
layout(set=0,binding=0) uniform sampler2D transmittanceLUT;
layout(set=0,binding=1, rgba32f) writeonly image3D inScatterLUT;
layout(local_size_x=4,local_size_y=4,local_size_z=4) in;
void main() {
    vec3 uvw  = (vec3(gl_GlobalInvocationID)+0.5)/vec3(imageSize(inScatterLUT));
    float h   = Rg + uvw.x*(Rt-Rg);
    float cosV= uvw.y*2.0-1.0;
    float cosS= uvw.z*2.0-1.0;
    float dt  = atmosphericRayLength(h,cosV,Rt)/float(STEPS);
    vec3  inR=vec3(0), inM=vec3(0);
    for (int i=0; i<STEPS; i++) {
        float t  = (float(i)+0.5)*dt;
        float ri = sqrt(h*h + t*t + 2.0*h*t*cosV);
        vec2  Ti = textureLod(transmittanceLUT, txUV(ri,cosS), 0.0).rg;  // sun→point
        vec2  Tv = textureLod(transmittanceLUT, txUV(h, cosV), 0.0).rg;  // point→cam
        float rhoR = exp(-(ri-Rg)/HR), rhoM = exp(-(ri-Rg)/HM);
        inR += rhoR * Ti * Tv * dt;
        inM += rhoM * Ti * Tv * dt;
    }
    imageStore(inScatterLUT, ivec3(gl_GlobalInvocationID),
               vec4(betaR*inR + betaM/(4.0*PI)*inM, 1.0));
}
```

[Source: Bruneton & Neyret 2008; Elek "Rendering Parametrizable Planetary Atmospheres with Multiple Scattering in Real-Time", CESCG 2009]

### 83.3 Runtime Sky Evaluation

At runtime, the sky colour in direction d from camera altitude h is the in-scatter LUT lookup, attenuated by the sun's phase function:

```glsl
// sky.frag — evaluate sky colour from precomputed LUTs
uniform sampler2D transmittanceLUT;
uniform sampler3D inScatterLUT;
uniform vec3 sunDir;
in vec3 viewDir;
void main() {
    vec3  V    = normalize(viewDir);
    float cosV = V.y;                       // view zenith angle
    float cosS = sunDir.y;                  // sun zenith
    float h    = cameraAltitude + Rg;       // metres from planet center
    vec3  inS  = texture(inScatterLUT, vec3(altToUV(h), cosToUV(cosV), cosToUV(cosS))).rgb;
    // Phase functions
    float mu   = dot(V, sunDir);
    float phR  = 3.0/(16.0*PI) * (1.0+mu*mu);
    float phM  = 3.0/(8.0*PI) * (1.0-g*g)*(1.0+mu*mu) / ((2.0+g*g)*pow(1.0+g*g-2.0*g*mu, 1.5));
    fragColor  = vec4(inS * (phR + phM) * SOLAR_IRRADIANCE, 1.0);
}
```

[Source: Bruneton & Neyret 2008; Hillaire 2020]

### 83.4 Aerial Perspective (Depth-Based Fog)

Apply aerial perspective to scene geometry by integrating the in-scatter and transmittance along the view ray to each depth sample:

```glsl
// aerial_perspective.comp — compute aerial perspective for each depth sample
layout(set=0,binding=0) uniform sampler2D depthBuffer;
layout(set=0,binding=1) uniform sampler3D inScatterLUT;
layout(set=0,binding=2) uniform sampler2D transmittanceLUT;
layout(set=0,binding=3, rgba16f) writeonly image2D aerialFog;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float depth= texelFetch(depthBuffer, pix, 0).r;
    vec3  worldPos = worldSpacePos(pix, depth);
    float dist     = length(worldPos - cameraPos);
    // Transmittance and inscatter from camera to world position
    vec2  T    = texture(transmittanceLUT, txUV(cameraAlt, dot(normalize(worldPos-cameraPos), UP))).rg;
    vec3  inS  = texture(inScatterLUT, scatterUV(cameraAlt, viewAngle, sunAngle)).rgb;
    vec3  fog  = (1.0-T.x) * inS * SOLAR_IRRADIANCE;
    imageStore(aerialFog, pix, vec4(fog, T.x));  // apply in composite pass
}
```

[Source: Bruneton & Neyret 2008; Lagarde & de Rousiers "Moving Frostbite to Physically Based Rendering", SIGGRAPH 2014]

---

### 84. Subsurface Scattering Geometry

*Audience: graphics application developers.*

Subsurface scattering (SSS) models light penetrating translucent materials (skin, marble, wax) and exiting at a different surface point. GPU implementations use the separable screen-space SSS blur (Jiménez et al. 2009), the pre-integrated skin BRDF (d'Eon & Luebke 2007), and ray-traced diffusion profiles via `VK_KHR_ray_query` for high-quality offline SSS.

### 84.1 Screen-Space SSS Blur (Jiménez et al. 2009)

Blur the irradiance buffer in screen space along each of three directions using a Gaussian kernel shaped by the diffusion profile of the material:

```glsl
// sss_blur.comp — separable screen-space SSS blur along one axis
layout(set=0,binding=0) uniform sampler2D irradianceTex;
layout(set=0,binding=1) uniform sampler2D depthTex;
layout(set=0,binding=2, rgba16f) writeonly image2D ssssOut;
layout(push_constant) uniform PC { vec2 dir; float sssStrength; float correction; } pc;
// Skin diffusion kernel: 6 Gaussians (d'Eon & Luebke 2007), precomputed
const vec4 KERNEL[6] = {
    vec4(0.233, 0.455, 0.649, 0.0),   // variance 0.0064
    vec4(0.1,   0.336, 0.344, 0.0064),
    vec4(0.118, 0.198, 0.0,  0.0484),
    vec4(0.113, 0.007, 0.007, 0.187),
    vec4(0.358, 0.004, 0.0,  0.567),
    vec4(0.078, 0.0,   0.0,  1.99)
};
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix   = ivec2(gl_GlobalInvocationID.xy);
    float depth = texelFetch(depthTex, pix, 0).r;
    vec4  color = vec4(0.0);
    for (int i=-KERNEL_SIZE; i<=KERNEL_SIZE; i++) {
        float s   = KERNEL[abs(i)%6].w;
        ivec2 nb  = pix + ivec2(round(pc.dir * float(i) / sqrt(s)));
        float dnb = texelFetch(depthTex, nb, 0).r;
        float wD  = exp(-abs(depth-dnb)*pc.correction);
        color    += texelFetch(irradianceTex, nb, 0) * KERNEL[abs(i)%6].xyzw * wD;
    }
    imageStore(ssssOut, pix, color);
}
```

[Source: Jiménez et al. "Screen-Space Perceptual Rendering of Human Skin", ACM TAP 2009; d'Eon & Luebke "Advanced Techniques for Realistic Real-Time Skin Rendering", GPU Gems 3 2007]

### 84.2 Pre-Integrated Skin BRDF

The pre-integrated skin shading model (d'Eon & Luebke 2007) precomputes the integral of the diffusion profile over a sphere of radius 1/κ (curvature) and stores it as a 2D LUT indexed by (N·L, curvature):

```glsl
// skin_brdf.frag — evaluate pre-integrated skin BRDF from LUT
uniform sampler2D skinBRDF_LUT;   // (NdotL, curvature) → pre-integrated diffusion
uniform sampler2D curvatureTex;   // per-pixel mean curvature (1/radius)
uniform vec3 lightDir;
in vec3 worldNor;
in vec2 texUV;
void main() {
    float NdotL  = dot(worldNor, lightDir) * 0.5 + 0.5;  // remap [−1,1]→[0,1]
    float kappa  = texture(curvatureTex, texUV).r;         // curvature in 1/m
    // LUT stores RGBE irradiance for each (NdotL, curvature) pair
    vec3  irrad  = texture(skinBRDF_LUT, vec2(NdotL, kappa * 0.1)).rgb;
    vec3  albedo = texture(skinAlbedo, texUV).rgb;
    fragColor    = vec4(irrad * albedo, 1.0);
}
```

[Source: d'Eon & Luebke 2007; Penner "Pre-Integrated Skin Shading", ShaderX 9 2011]

### 84.3 Ray-Traced SSS via Diffusion Dipole Ray Queries

For high-quality SSS, shoot diffusion probe rays below the surface and accumulate the dipole contribution from each hit:

```glsl
// sss_raytrace.comp — ray-traced SSS via dipole model and ray query
layout(set=0,binding=0) readonly buffer SurfacePoints { vec3 pos[]; vec3 nor[]; };
layout(set=0,binding=1) uniform accelerationStructureEXT tlas;
layout(set=0,binding=2) writeonly buffer SSSIrrad { vec3 irrad[]; };
layout(push_constant) uniform PC { int nRays; float sigma_a; float sigma_s; float eta; } pc;
layout(local_size_x=64) in;
void main() {
    uint v    = gl_GlobalInvocationID.x;
    vec3 p    = pos[v], n = nor[v];
    float mfp = 1.0 / (pc.sigma_a + pc.sigma_s);  // mean free path
    vec3  E   = vec3(0.0);
    for (int r=0; r<pc.nRays; r++) {
        // Sample a point on the surface within ~3 MFP using disc sampling
        vec3 probe = p + sampleDisk(n, mfp*3.0, halton2D(r));
        float dist = length(probe - p);
        // Dipole BSSRDF: R_d(dist)
        float Rd   = dipoleRd(dist, pc.sigma_a, pc.sigma_s, pc.eta);
        // Test visibility from probe to light
        rayQueryEXT rq;
        rayQueryInitializeEXT(rq, tlas, gl_RayFlagsTerminateOnFirstHitEXT, 0xFF,
                              probe+n*1e-3, 0.0, lightDir, 1e4);
        rayQueryProceedEXT(rq);
        if (rayQueryGetIntersectionTypeEXT(rq,true)==gl_RayQueryCommittedIntersectionNoneEXT)
            E += Rd * lightColor * max(dot(n,lightDir),0.0);
    }
    irrad[v] = E * 2.0 * PI * mfp / float(pc.nRays);
}
```

[Source: Jensen et al. "A Practical Model for Subsurface Light Transport", SIGGRAPH 2001; d'Eon & Luebke 2007]

### 84.4 Translucency: Thin-Slab Transmission

For thin translucent objects (ears, leaves), approximate SSS by transmitting direct light through the slab using Beer's law with a transmitted thickness map:

```glsl
// translucency.frag — thin-slab SSS via transmitted thickness (Beer's law)
uniform sampler2D thicknessMap;   // pre-baked or real-time ray-traced thickness
uniform vec3 lightDir;
uniform vec3 lightColor;
uniform vec3 subsurfaceColor;     // material absorption spectrum
in vec3 worldNor;
in vec2 texUV;
void main() {
    float thickness = texture(thicknessMap, texUV).r;  // metres
    vec3  V         = normalize(viewDir);
    // Wrap lighting for back-face transmittance
    float NdotL_wrap = dot(-worldNor, lightDir) * 0.5 + 0.5;
    // Beer–Lambert attenuation: T = exp(-sigma_a * thickness)
    vec3  T         = exp(-subsurfaceColor * thickness);
    vec3  backLight = lightColor * T * NdotL_wrap;
    // Add to front-face diffuse
    vec3  diffuse   = max(dot(worldNor, lightDir), 0.0) * lightColor;
    fragColor       = vec4((diffuse + backLight) * texture(albedo, texUV).rgb, 1.0);
}
```

[Source: Green "Real-Time Approximations to Subsurface Scattering", GPU Gems 1 2004; Jiménez et al. 2010 "Separable Subsurface Scattering"]

---

### 85. Order-Independent Transparency

*Audience: graphics application developers.*

Transparent geometry must be composited in back-to-front depth order, but sorting draw calls per-frame is expensive and breaks GPU parallelism for complex scenes (particle systems, hair, foliage). Order-independent transparency (OIT) algorithms resolve alpha blending on the GPU without sorting: per-pixel linked lists (PPLL), k-buffer, and weighted blended OIT (WBOIT) cover the quality/performance trade-off.

### 85.1 Per-Pixel Linked List Construction

Append each transparent fragment to a per-pixel linked list stored in a flat buffer. A head-pointer image stores the list head per pixel; atomic exchange chains fragments together:

```glsl
// oit_ppll_build.frag — append fragment to per-pixel linked list
layout(set=0,binding=0, r32ui) uniform uimage2D headImg;   // per-pixel head pointer
layout(set=0,binding=1) buffer FragList { OITFragment frags[]; uint count; };
layout(location=0) in vec4 fragColor;
void main() {
    uint slot = atomicAdd(count, 1u);
    if (slot >= MAX_FRAGS) return;
    frags[slot].color = packUnorm4x8(fragColor);
    frags[slot].depth = gl_FragCoord.z;
    // Atomically prepend to per-pixel list
    uint prev = imageAtomicExchange(headImg, ivec2(gl_FragCoord.xy), slot);
    frags[slot].next = prev;
}
```

[Source: Yang et al. "Real-Time Concurrent Linked List Construction on the GPU", EG 2010; Crassin "OIT and GBuffer Compositing Using the New D3D11 Atomic Counting and UAVs", NVIDIA 2010]

### 85.2 Per-Pixel Linked List Resolve (Sort and Blend)

In a fullscreen pass, traverse the linked list for each pixel, sort fragments by depth, and composite back-to-front:

```glsl
// oit_ppll_resolve.comp — sort and blend per-pixel fragment list
layout(set=0,binding=0, r32ui) uniform readonly uimage2D headImg;
layout(set=0,binding=1) readonly buffer FragList { OITFragment frags[]; };
layout(set=0,binding=2, rgba16f) uniform writeonly image2D outColor;
layout(local_size_x=8,local_size_y=8) in;
void main() {
    ivec2 pix   = ivec2(gl_GlobalInvocationID.xy);
    uint  head  = imageLoad(headImg, pix).r;
    // Collect up to MAX_LAYERS fragments into a local array
    uint  idx[MAX_LAYERS]; int n=0;
    for (uint cur=head; cur!=0xFFFFFFFF && n<MAX_LAYERS; cur=frags[cur].next) idx[n++]=cur;
    // Insertion sort by depth (small n: 4-16 typical)
    for(int i=1;i<n;i++) { uint k=idx[i]; int j=i-1;
        while(j>=0 && frags[idx[j]].depth < frags[k].depth) { idx[j+1]=idx[j]; j--; } idx[j+1]=k; }
    // Back-to-front alpha blend
    vec4 color = vec4(0.0);
    for(int i=0;i<n;i++) {
        vec4 fc = unpackUnorm4x8(frags[idx[i]].color);
        color   = fc.a*fc + (1.0-fc.a)*color;
    }
    imageStore(outColor, pix, color);
}
```

### 85.3 Weighted Blended OIT (McGuire & Bavoil 2013)

WBOIT avoids sorting entirely by accumulating a weighted sum and weight sum in two render targets. The final composite divides out the weights. Approximate but hardware-friendly and suitable for particles:

```glsl
// wboit_accum.frag — accumulate weighted OIT into two render targets
layout(location=0) out vec4 accumRT;   // sum: w(z) * premultAlpha(color)
layout(location=1) out float revealRT; // product: (1 - alpha) per pixel
in vec4 fragColor;
void main() {
    float z = gl_FragCoord.z;
    // McGuire & Bavoil weight function: balances near/far coverage
    float w = clamp(pow(min(1.0, fragColor.a*10.0)+0.01, 3.0)*1e8
              * pow(1.0-z*0.9, 3.0), 1e-2, 3e3);
    accumRT  = vec4(fragColor.rgb*fragColor.a, fragColor.a) * w;
    revealRT = fragColor.a;  // blended with glBlendFunc(ZERO, ONE_MINUS_SRC_ALPHA)
}
```

```glsl
// wboit_composite.frag — reconstruct final colour from WBOIT accum buffers
uniform sampler2D accumTex;
uniform sampler2D revealTex;
in vec2 texUV;
void main() {
    vec4  accum  = texture(accumTex,  texUV);
    float reveal = texture(revealTex, texUV).r;
    if (reveal >= 1.0 - 1e-4) discard;  // fully opaque: skip
    fragColor = vec4(accum.rgb / max(accum.a, 1e-5), 1.0-reveal);
}
```

[Source: McGuire & Bavoil "Weighted Blended Order-Independent Transparency", JCGT 2013]

### 85.4 K-Buffer Stochastic Transparency

K-buffer (Sintorn & Assarsson 2009) keeps only the k nearest fragments per pixel. For k=4, use a 4-entry insertion sort in the fragment shader with atomic compare-and-swap on the sorted buffer:

```glsl
// kbuffer_build.frag — maintain sorted k-nearest fragments per pixel
layout(set=0,binding=0) buffer KBuffer { uint64_t kbuf[][K]; };  // packed depth+color, K per pixel
in vec4 fragColor;
void main() {
    ivec2 pix  = ivec2(gl_FragCoord.xy);
    uint  flat = pix.y*WIDTH+pix.x;
    uint64_t entry = (uint64_t(floatBitsToUint(gl_FragCoord.z))<<32)|uint64_t(packUnorm4x8(fragColor));
    // Insert entry into sorted k-buffer at this pixel (max depth evicted)
    for (int k=0; k<K; k++) {
        uint64_t old = atomicMax(kbuf[flat][k], entry);
        if (old == 0UL || entry < old) break;  // inserted in sorted position
        entry = max(entry, old);  // propagate evicted entry to next slot
    }
}
```

[Source: Sintorn & Assarsson "A Real-Time Shadow Algorithm for Unstructured Meshes", SIGGRAPH 2009; Myers & Bavoil "Stochastic Transparency", I3D 2007]

---

### 86. GPU Polygon and Frustum Clipping

*Audience: systems developers, graphics application developers.*

Clipping geometry against a frustum or arbitrary half-spaces is required for: shadow map rendering (clip to light frustum), portal rendering, decal projection (§97), constructive solid geometry (§7), and conservative rasterisation. GPU Sutherland-Hodgman clips polygons against each plane independently; hardware clipping handles triangle-level frustum culling automatically, but compute-shader clipping is needed for non-standard clip volumes.

### 86.1 Sutherland-Hodgman on GPU (Per-Polygon Thread)

Run the Sutherland-Hodgman algorithm independently for each polygon. Each thread processes one input polygon against all clip planes, outputting a clipped polygon:

```glsl
// sh_clip.comp — Sutherland-Hodgman polygon clipping against N half-spaces
layout(set=0,binding=0) readonly buffer InPolys   { Polygon inP[];  };   // variable-length vertex lists
layout(set=0,binding=1) readonly buffer ClipPlanes { vec4 planes[]; };   // ax+by+cz+d≥0 = inside
layout(set=0,binding=2) writeonly buffer OutPolys  { Polygon outP[]; };
layout(push_constant) uniform PC { uint N_POLYS; uint N_PLANES; } pc;
layout(local_size_x=64) in;
void main() {
    uint poly = gl_GlobalInvocationID.x;
    if (poly >= pc.N_POLYS) return;
    // Copy input polygon to local registers (≤16 vertices typical)
    vec3  buf[32]; int nv = inP[poly].n;
    for(int i=0;i<nv;i++) buf[i]=inP[poly].v[i];
    for (uint pl=0; pl<pc.N_PLANES; pl++) {
        vec4  P = planes[pl];
        vec3  tmp[32]; int nt=0;
        for (int i=0; i<nv; i++) {
            vec3 a=buf[i], b=buf[(i+1)%nv];
            bool aIn=(dot(P.xyz,a)+P.w)>=0.0, bIn=(dot(P.xyz,b)+P.w)>=0.0;
            if (aIn) tmp[nt++]=a;
            if (aIn!=bIn) {
                float t=(dot(P.xyz,a)+P.w)/(dot(P.xyz,a)-dot(P.xyz,b));
                tmp[nt++]=mix(a,b,t);
            }
        }
        nv=nt; for(int i=0;i<nv;i++) buf[i]=tmp[i];
        if (nv<3) break;
    }
    outP[poly].n=nv; for(int i=0;i<nv;i++) outP[poly].v[i]=buf[i];
}
```

[Source: Sutherland & Hodgman "Reentrant Polygon Clipping", CACM 1974; Blinn & Newell "Clipping Using Homogeneous Coordinates", SIGGRAPH 1978]

### 86.2 Triangle Frustum Culling via Mesh Shader Amplification

The task/amplification shader phase (§101.2 mesh shader pattern) naturally implements per-cluster frustum culling before spawning mesh shader workgroups:

```glsl
// frustum_cull.task.glsl — amplification shader: cull clusters against view frustum
layout(local_size_x=32) in;
taskPayloadSharedEXT uint visibleClusters[32];
void main() {
    uint c   = gl_GlobalInvocationID.x;
    bool vis = sphereInFrustum(clusterBounds[c].sphere, frustumPlanes);
    // Stream compaction via ballot: emit only visible clusters
    uvec4 ballot = subgroupBallot(vis);
    uint  nVis   = subgroupBallotBitCount(ballot);
    if (gl_LocalInvocationID.x==0) EmitMeshTasksEXT(nVis,1,1);
    if (vis) {
        uint slot = subgroupBallotExclusiveBitCount(ballot);
        visibleClusters[slot] = c;
    }
}
```

[Source: Wihlidal 2016 §90; Kubisch 2018; Harris "GPU-Driven Rendering Pipelines", SIGGRAPH 2015]

### 86.3 Conservative Rasterisation for Voxelisation

Conservative rasterisation expands each triangle outward so every voxel column touched by the triangle is rasterised — required for §13 solid voxelisation. Implement via geometry shader triangle expansion or the `VK_EXT_conservative_rasterization` extension:

```glsl
// conservative_rast.geom — expand triangle for conservative rasterisation
layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;
uniform vec2 pixelSize;  // 1/viewport
void main() {
    vec2 p[3]; for(int i=0;i<3;i++) p[i]=gl_in[i].gl_Position.xy/gl_in[i].gl_Position.w;
    // Edge half-planes and their outward normals (scaled by half-pixel diagonal)
    float diag = length(pixelSize)*0.5*sqrt(2.0);
    for (int i=0;i<3;i++) {
        vec2 e   = p[(i+1)%3]-p[i];
        vec2 n   = normalize(vec2(-e.y,e.x));
        p[i]    += n*diag;
    }
    for (int i=0;i<3;i++) {
        gl_Position=vec4(p[i]*gl_in[i].gl_Position.w, gl_in[i].gl_Position.zw);
        EmitVertex();
    }
    EndPrimitive();
}
```

[Source: Hasselgren et al. "Conservative Rasterization", GPU Gems 2 2005; §41 voxelisation]

### 86.4 Clip-Space W-Buffer and Homogeneous Clipping

Clipping in homogeneous coordinates before the perspective divide handles the near-plane clipping case exactly, preventing division-by-zero artefacts for triangles crossing the camera:

```glsl
// homogeneous_clip.comp — clip triangle in homogeneous space against w=ε near plane
layout(set=0,binding=0) readonly buffer InTris  { vec4 hVerts[][3]; };  // homogeneous clip coords
layout(set=0,binding=1) buffer OutTris { vec4 oVerts[][3]; uint count; };
layout(local_size_x=64) in;
void main() {
    uint t  = gl_GlobalInvocationID.x;
    vec4 v[3]; for(int i=0;i<3;i++) v[i]=hVerts[t][i];
    float eps = 1e-4;
    // Sutherland-Hodgman against w≥ε (near plane in homogeneous space)
    vec4 out_[4]; int no=0;
    for(int i=0;i<3;i++) {
        vec4 a=v[i], b=v[(i+1)%3];
        bool aIn=(a.w>=eps), bIn=(b.w>=eps);
        if(aIn) out_[no++]=a;
        if(aIn!=bIn) { float t2=(a.w-eps)/(a.w-b.w); out_[no++]=mix(a,b,t2); }
    }
    // Fan-triangulate out_ into output
    for(int i=1;i<no-1;i++) {
        uint s=atomicAdd(count,1u);
        oVerts[s][0]=out_[0]; oVerts[s][1]=out_[i]; oVerts[s][2]=out_[i+1];
    }
}
```

[Source: Blinn & Newell 1978; Heckbert & Moreton "Interpolation for Polygon Texture Mapping and Shading", State of the Art in Computer Graphics 1991]

---


---

## X. Point Cloud, Reconstruction, and Perception

Point clouds are the native output of depth sensors, LiDAR scanners, and multi-view reconstruction pipelines. This category covers GPU-accelerated point cloud processing (normal estimation via PCA on local neighbourhoods, outlier removal, voxel downsampling), Structure from Motion and multi-view stereo for recovering 3D geometry from unstructured image collections, Iterative Closest Point (ICP) registration for aligning successive point cloud frames, and 3D feature descriptor computation (FPFH, SHOT, learned descriptors) for recognition and pose estimation. These algorithms form the geometry perception pipeline for robotics, autonomous vehicles, and AR/VR applications.

### 87. Point Cloud Processing

*Audience: systems developers, graphics application developers.*

Point clouds are the direct output of depth cameras (structured light, LiDAR, stereo), photogrammetry pipelines, and 3DGS/NeRF density fields. GPU point cloud processing bridges raw sensor data and the mesh/SDF representations used throughout this chapter.

### 87.1 GPU k-NN via Tiled Brute-Force

For N ≤ 100k points and k ≤ 32, a **tiled brute-force** approach exploiting shared memory beats tree traversal by eliminating branch divergence:

```glsl
// knn_brute.comp — k=16 nearest neighbours, TILE=64
layout(set=0,binding=0) readonly buffer Points { vec4 pts[]; };
layout(set=0,binding=1) writeonly buffer KNN   { uint knn[16 * N_POINTS]; };
layout(local_size_x=64) in;
shared vec4 tile[64];

void main() {
    uint i = gl_GlobalInvocationID.x;
    vec3 pi = pts[i].xyz;
    float dist[16]; uint idx[16];
    for (int j=0; j<16; j++) { dist[j]=1e30; idx[j]=0; }

    for (uint base=0; base<N_POINTS; base+=64) {
        tile[gl_LocalInvocationID.x] = pts[base + gl_LocalInvocationID.x];
        barrier();
        for (uint t=0; t<64; t++) {
            float d = length(pi - tile[t].xyz);
            if (d < dist[15]) {
                dist[15]=d; idx[15]=base+t;
                for (int s=14; s>=0 && dist[s]>dist[s+1]; s--)
                    { float td=dist[s]; dist[s]=dist[s+1]; dist[s+1]=td;
                      uint ti=idx[s];  idx[s]=idx[s+1];   idx[s+1]=ti; }
            }
        }
        barrier();
    }
    for (int j=0; j<16; j++) knn[i*16+j]=idx[j];
}
```

For larger clouds use the GPU k-d tree (§27.1) or uniform grid radius search (§27.2). [Source: Garcia et al. "Fast k Nearest Neighbor Search Using GPU", CVPR Workshop 2008]

### 87.2 Normal Estimation from k-NN

Per-point normals are the smallest eigenvector of the 3×3 neighbourhood covariance matrix. Each thread computes its local covariance and solves the 3×3 symmetric eigenproblem via cyclic Jacobi iteration:

```glsl
// normal_est.comp
layout(local_size_x=64) in;
void main() {
    uint i = gl_GlobalInvocationID.x;
    vec3 pi = pts[i].xyz;
    vec3 mean = vec3(0);
    for (int j=0; j<K; j++) mean += pts[knn[i*K+j]].xyz;
    mean /= float(K);
    mat3 C = mat3(0.0);
    for (int j=0; j<K; j++) {
        vec3 d = pts[knn[i*K+j]].xyz - mean;
        C += outerProduct(d,d);
    }
    // 3x3 symmetric Jacobi eigensolver (5-6 sweeps)
    mat3 V; vec3 lambda;
    jacobiEigen3x3(C, V, lambda);  // lambda[0] ≤ lambda[1] ≤ lambda[2]
    normals[i] = vec4(V[0], 0.0);  // normal = eigenvector of smallest eigenvalue
}
```

Orientation consistency (pointing toward the sensor) requires a minimum spanning tree propagation — a CPU post-pass for static clouds. [Source: Rusu "Semantic 3D Object Maps for Everyday Manipulation", PhD 2009]

### 87.3 GPU ICP Registration

**Iterative Closest Point** (Besl & McKay 1992) aligns source cloud P to target Q. Each iteration: (1) GPU k-NN finds nearest neighbours; (2) GPU reduction accumulates the 3×3 cross-covariance H = Σ(pᵢ−μₚ)(qᵢ−μ_q)ᵀ; (3) CPU SVD of H gives R = V Uᵀ; (4) apply R,t and repeat.

```glsl
// icp_reduce.comp — accumulate cross-covariance and means per workgroup
layout(set=0,binding=0) readonly buffer Src { vec4 src[]; };
layout(set=0,binding=1) readonly buffer NN  { uint nn[]; };
layout(set=0,binding=2) readonly buffer Tgt { vec4 tgt[]; };
layout(set=0,binding=3) buffer Reduction { float H[9]; float meanP[3]; float meanQ[3]; };
layout(local_size_x=64) in;
shared mat3 sH[64]; shared vec3 sMp[64], sMq[64];
void main() {
    uint i=gl_GlobalInvocationID.x;
    vec3 p=src[i].xyz, q=tgt[nn[i]].xyz;
    sH[gl_LocalInvocationID.x]=outerProduct(p,q);
    sMp[gl_LocalInvocationID.x]=p; sMq[gl_LocalInvocationID.x]=q;
    barrier();
    for (uint s=32; s>0; s>>=1) {
        if (gl_LocalInvocationID.x<s) {
            sH[gl_LocalInvocationID.x]  += sH[gl_LocalInvocationID.x+s];
            sMp[gl_LocalInvocationID.x] += sMp[gl_LocalInvocationID.x+s];
            sMq[gl_LocalInvocationID.x] += sMq[gl_LocalInvocationID.x+s];
        }
        barrier();
    }
    if (gl_LocalInvocationID.x==0) {
        for (int r=0;r<3;r++) for (int c=0;c<3;c++)
            atomicAdd(Reduction.H[r*3+c], sH[0][r][c]);
        for (int r=0;r<3;r++) {
            atomicAdd(Reduction.meanP[r], sMp[0][r]);
            atomicAdd(Reduction.meanQ[r], sMq[0][r]);
        }
    }
}
```

Point-to-plane ICP (project residual onto surface normal) converges 3× faster and has a closed-form 6×6 linear system solve per iteration. [Source: Besl & McKay "A Method for Registration of 3-D Shapes", PAMI 1992; Chen & Medioni "Object Modeling by Registration of Multiple Range Images", IVC 1992]

### 87.4 GPU RANSAC Primitive Fitting

RANSAC scores each candidate model against all points in parallel. Generate N candidates (plane, sphere, cylinder) from random minimal subsets; dispatch one kernel to count inliers for all N simultaneously:

```glsl
// ransac_score.comp — score N candidate planes against M points
layout(set=0,binding=0) readonly buffer Points     { vec4 pts[]; };     // M points
layout(set=0,binding=1) readonly buffer Candidates { vec4 planes[]; };  // N (n,d) planes
layout(set=0,binding=2) buffer InlierCounts { uint counts[]; };          // N counters
layout(push_constant) uniform PC { float threshold; } pc;
layout(local_size_x=64) in;
void main() {
    uint p=gl_GlobalInvocationID.x % M_POINTS;
    uint c=gl_GlobalInvocationID.x / M_POINTS;
    float dist = abs(dot(pts[p].xyz, Candidates[c].xyz) + Candidates[c].w);
    if (dist < pc.threshold) atomicAdd(counts[c], 1u);
}
```

Dispatch with `(N*M/64, 1, 1)` workgroups — all N×M scores evaluate in one pass. Read back the winning candidate index. [Source: Fischler & Bolles "Random Sample Consensus", CACM 1981; Schnabel et al. "Efficient RANSAC for Point-Cloud Shape Detection", CGF 2007]

### 87.5 Euclidean Segmentation via Label Propagation

Cluster points within distance δ using the same iterative label-propagation pattern from §70.2 and §50.2, but in 3D over a spatial hash grid:

```glsl
// seg_label.comp
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint lm = label[i];
    ivec3 ci = gridCell(pts[i].xyz);
    for (int dz=-1;dz<=1;dz++) for(int dy=-1;dy<=1;dy++) for(int dx=-1;dx<=1;dx++) {
        uint cell = hashCell(ci+ivec3(dx,dy,dz));
        for (uint k=gridStart[cell]; k<gridEnd[cell]; k++) {
            uint j = gridPts[k];
            if (length(pts[i].xyz-pts[j].xyz) < DELTA)
                lm = min(lm, label[j]);
        }
    }
    if (lm < label[i]) { label[i]=lm; atomicOr(changed,1u); }
}
```

Iterate until `changed==0`. Compact to dense cluster IDs via prefix scan. [Source: Rusu et al. "Towards 3D Point Cloud Based Object Maps", Robotics and Autonomous Systems 2008]

---

### 88. GPU Structure from Motion and Multi-View Stereo

*Audience: systems developers, graphics application developers.*

Structure from Motion (SfM) and Multi-View Stereo (MVS) reconstruct 3D geometry from multiple camera images. GPU acceleration targets: feature detection and matching (ORB, AKAZE), fundamental matrix estimation via GPU-RANSAC, depth map computation via semi-global matching (SGM), and depth map fusion into a dense point cloud (§9 Poisson surface).

### 88.1 GPU-Accelerated ORB Feature Detection

ORB (Rublee et al. 2011) combines FAST corner detection with binary BRIEF descriptors. The FAST response at each pixel is computed in parallel:

```glsl
// fast_detect.comp — FAST-9 corner response at each pixel
layout(set=0,binding=0) uniform sampler2D grayImg;
layout(set=0,binding=1, r8ui) uniform writeonly uimage2D responseOut;
layout(local_size_x=16,local_size_y=16) in;
const ivec2 CIRCLE9[16] = {ivec2(0,3),ivec2(1,3),ivec2(2,2),ivec2(3,1),
                            ivec2(3,0),ivec2(3,-1),ivec2(2,-2),ivec2(1,-3),
                            ivec2(0,-3),ivec2(-1,-3),ivec2(-2,-2),ivec2(-3,-1),
                            ivec2(-3,0),ivec2(-3,1),ivec2(-2,2),ivec2(-1,3)};
void main() {
    ivec2 pix = ivec2(gl_GlobalInvocationID.xy);
    float cen = texelFetch(grayImg, pix, 0).r;
    float thresh = 0.1;
    uint bright=0u, dark=0u;
    for (int i=0;i<16;i++) {
        float nb = texelFetch(grayImg, pix+CIRCLE9[i], 0).r;
        if (nb > cen+thresh) bright |= (1u<<i);
        if (nb < cen-thresh) dark   |= (1u<<i);
    }
    // FAST-9: 9 consecutive bits set in 16-bit circle
    bool corner = false;
    for (int i=0;i<16&&!corner;i++) corner = (popcount((bright|(bright<<16))>>(i)&0x1FFu)==9)
                                           ||(popcount((dark  |(dark  <<16))>>(i)&0x1FFu)==9);
    imageStore(responseOut, pix, uvec4(corner?255u:0u));
}
```

[Source: Rublee et al. "ORB: An Efficient Alternative to SIFT or SURF", ICCV 2011; Sinha et al. "GPU-Based Video Feature Tracking And Matching", EDGE 2006]

### 88.2 RANSAC Fundamental Matrix Estimation

GPU RANSAC samples N hypothesis sets in parallel, each computing the 7/8-point fundamental matrix and counting inliers:

```glsl
// ransac_fundamental.comp — parallel RANSAC for fundamental matrix F
layout(set=0,binding=0) readonly buffer Matches { vec4 matches[]; };  // x,y in img A; x,y in img B
layout(set=0,binding=1) readonly buffer Seeds   { uint seeds[][8]; };  // random 8-point sets
layout(set=0,binding=2) buffer InlierCount { uint cnt[]; };
layout(set=0,binding=3) buffer BestF { mat3 F[]; };
layout(push_constant) uniform PC { uint N_HYPOTHESES; uint N_MATCHES; float thresh; } pc;
layout(local_size_x=64) in;
void main() {
    uint h   = gl_GlobalInvocationID.x;
    if (h >= pc.N_HYPOTHESES) return;
    // Compute F from 8 sampled correspondences via DLT (8-point algorithm)
    mat3 F   = compute8PointF(matches, seeds[h]);
    uint inl = 0u;
    for (uint m=0; m<pc.N_MATCHES; m++) {
        vec3 x1 = vec3(matches[m].xy,1), x2 = vec3(matches[m].zw,1);
        float err = abs(dot(x2, F*x1));  // Sampson distance
        if (err < pc.thresh) inl++;
    }
    cnt[h]   = inl;
    F[h]     = F;
}
// CPU: find argmax(cnt), polish with all inliers → final F
```

[Source: Hartley & Zisserman "Multiple View Geometry in Computer Vision", 2004; GPU RANSAC: Ni et al. "Out-of-core Bundle Adjustment for Large-scale 3D Reconstruction", ICCV 2007]

### 88.3 Semi-Global Matching (SGM) Depth Estimation

SGM (Hirschmüller 2008) computes dense stereo depth maps by aggregating matching costs along multiple scan-line paths. GPU parallelism runs all rows simultaneously:

```glsl
// sgm_cost.comp — compute matching cost volume (SAD-based)
layout(set=0,binding=0) uniform sampler2D imgL, imgR;
layout(set=0,binding=1) writeonly buffer CostVol { float cost[]; };  // [H][W][DISP_RANGE]
layout(push_constant) uniform PC { int dispRange; int winSz; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    for (int d=0; d<pc.dispRange; d++) {
        float sad = 0.0;
        for (int dy=-pc.winSz;dy<=pc.winSz;dy++)
            for (int dx=-pc.winSz;dx<=pc.winSz;dx++) {
                float vL = texelFetch(imgL, pix+ivec2(dx,dy), 0).r;
                float vR = texelFetch(imgR, pix+ivec2(dx-d,dy), 0).r;
                sad += abs(vL-vR);
            }
        cost[(pix.y*WIDTH+pix.x)*pc.dispRange+d] = sad;
    }
}
// SGM aggregation: for each direction (8 paths), run dynamic programming along path
```

[Source: Hirschmüller "Stereo Processing by Semiglobal Matching and Mutual Information", IEEE TPAMI 2008; GPU SGM: Hernandez-Juarez et al. "Embedded Real-Time Stereo Estimation via Semi-Global Matching on the GPU", CVPRW 2016]

### 88.4 Depth Map Fusion into Point Cloud

Fuse per-image depth maps into a consistent dense point cloud using confidence-weighted averaging, then optionally feed into §9 Poisson surface reconstruction:

```glsl
// depth_fuse.comp — back-project depth map pixels into world-space point cloud
layout(set=0,binding=0) uniform sampler2D depthMap;
layout(set=0,binding=1) readonly buffer Confidence { float conf[]; };
layout(set=0,binding=2) buffer PointCloud { vec3 pts[]; vec3 nrm[]; float w[]; };
layout(set=0,binding=3) buffer PointCount { uint count; };
layout(push_constant) uniform PC { mat4 invViewProj; mat3 KInv; float confThresh; } pc;
layout(local_size_x=16,local_size_y=16) in;
void main() {
    ivec2 pix  = ivec2(gl_GlobalInvocationID.xy);
    float d    = texelFetch(depthMap, pix, 0).r;
    float c    = conf[pix.y*WIDTH+pix.x];
    if (d <= 0.0 || c < pc.confThresh) return;
    vec4  ws   = pc.invViewProj * vec4(vec2(pix)/vec2(WIDTH,HEIGHT)*2-1, d*2-1, 1);
    ws /= ws.w;
    uint slot  = atomicAdd(count, 1u);
    pts[slot]  = ws.xyz;
    nrm[slot]  = computeNormalFromDepth(depthMap, pix, pc.KInv);
    w[slot]    = c;
}
```

[Source: Merrell et al. "Real-Time Visibility-Based Fusion of Depth Maps", ICCV 2007; Schönberger & Frahm "Structure-from-Motion Revisited", CVPR 2016]

---

### 89. 3D Feature Descriptors and Pose Estimation

*Audience: systems developers, graphics application developers.*

3D feature descriptors encode local geometry around a keypoint into a compact vector used for point cloud matching and pose estimation. GPU algorithms compute Fast Point Feature Histograms (FPFH) and SHOT descriptors in parallel across all keypoints, then use GPU-RANSAC (§88.2 pattern) to estimate the 6-DOF rigid transform aligning two point clouds.

### 89.1 Normal Estimation via PCA

Before computing descriptors, estimate the surface normal at each point from its k-NN via PCA. The smallest eigenvector of the 3×3 covariance matrix of the neighbour positions is the normal:

```glsl
// normal_pca.comp — PCA normal estimation from k-NN
layout(set=0,binding=0) readonly buffer Pts { vec3 pts[]; };
layout(set=0,binding=1) readonly buffer KNN { uint knn[K_NN]; };  // k-NN indices per point
layout(set=0,binding=2) writeonly buffer Normals { vec3 nor[]; };
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    vec3 c  = pts[p];
    for (uint k=0;k<K_NN;k++) c += pts[knn[p*K_NN+k]];
    c /= float(K_NN+1);
    mat3 Cov = mat3(0.0);
    for (uint k=0;k<K_NN;k++) {
        vec3 d = pts[knn[p*K_NN+k]] - c;
        Cov   += outerProduct(d,d);
    }
    // Smallest eigenvalue of Cov → normal (power iteration on inverse)
    vec3 n = smallestEigenvec3x3(Cov);
    // Orient toward view point (flip if dot < 0)
    nor[p] = dot(n, VIEW_ORIGIN - pts[p]) > 0.0 ? n : -n;
}
```

[Source: Rusu et al. "Towards 3D Point Cloud Based Object Maps for Household Environments", Robotics and Autonomous Systems 2008]

### 89.2 FPFH Descriptor Computation

FPFH (Rusu et al. 2009) encodes the pairwise angular relationships between a point and its k-neighbours into a 33-dimensional histogram:

```glsl
// fpfh.comp — compute FPFH descriptor for each keypoint
layout(set=0,binding=0) readonly buffer Pts { vec3 pts[]; };
layout(set=0,binding=1) readonly buffer Nor { vec3 nor[]; };
layout(set=0,binding=2) readonly buffer KNN { uint knn[]; };
layout(set=0,binding=3) writeonly buffer FPFH { float fpfh[N_PTS][33]; };
layout(local_size_x=64) in;
void main() {
    uint p  = gl_GlobalInvocationID.x;
    float hist[33] = {0.0};
    for (uint k=0; k<K_NN; k++) {
        uint q   = knn[p*K_NN+k];
        vec3 d   = normalize(pts[q]-pts[p]);
        // Darboux frame angles: α, φ, θ
        vec3 u   = nor[p];
        vec3 v   = cross(d, u);
        vec3 w   = cross(u, v);
        float alpha = dot(v, nor[q]);
        float phi   = dot(u, d);
        float theta = atan(dot(w,nor[q]), dot(u,nor[q]));
        // Bin into 11-bin histograms for each angle
        hist[int((alpha+1.0)*5.0)]     += 1.0/float(K_NN);
        hist[11+int((phi+1.0)*5.0)]    += 1.0/float(K_NN);
        hist[22+int((theta+3.14159)*11.0/6.28318)] += 1.0/float(K_NN);
    }
    for (int i=0;i<33;i++) fpfh[p][i] = hist[i];
}
```

[Source: Rusu et al. "Fast Point Feature Histograms (FPFH) for 3D Registration", ICRA 2009]

### 89.3 Descriptor Matching via GPU k-NN

Match FPFH descriptors between two point clouds by finding the nearest-neighbour in descriptor space. Brute-force is O(N·M·33); approximate GPU k-NN via product quantization for large clouds:

```glsl
// fpfh_match.comp — brute-force nearest-neighbour matching in FPFH space
layout(set=0,binding=0) readonly buffer FPFH_A { float fA[N_A][33]; };
layout(set=0,binding=1) readonly buffer FPFH_B { float fB[N_B][33]; };
layout(set=0,binding=2) writeonly buffer Matches { uint match[N_A]; float matchDist[N_A]; };
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    float bestD = 1e30; uint bestJ = 0u;
    for (uint j=0; j<N_B; j++) {
        float d = 0.0;
        for (int k=0; k<33; k++) { float diff=fA[i][k]-fB[j][k]; d+=diff*diff; }
        if (d < bestD) { bestD=d; bestJ=j; }
    }
    match[i]=bestJ; matchDist[i]=bestD;
}
```

[Source: Rusu et al. 2009; GPU ANN: Johnson & Hebert "Using Spin Images for Efficient Object Recognition in Cluttered 3D Scenes", IEEE TPAMI 1999]

### 89.4 6-DOF Pose Estimation via GPU-RANSAC

Use the §88.2 GPU-RANSAC pattern with 3-point samples (minimum for a rigid transform) to estimate the transformation matrix aligning the two point clouds:

```glsl
// pose_ransac.comp — GPU-RANSAC for 6-DOF point cloud alignment
layout(set=0,binding=0) readonly buffer Pts_A { vec3 pA[]; };
layout(set=0,binding=1) readonly buffer Pts_B { vec3 pB[]; };
layout(set=0,binding=2) readonly buffer Matches { uint match[]; };
layout(set=0,binding=3) readonly buffer Seeds   { uint seeds[][3]; };
layout(set=0,binding=4) buffer InlierCount { uint cnt[]; };
layout(set=0,binding=5) buffer BestTransform { mat4 T[]; };
layout(push_constant) uniform PC { uint N_HYPO; uint N_MATCHES; float thresh; } pc;
layout(local_size_x=64) in;
void main() {
    uint h  = gl_GlobalInvocationID.x;
    if (h >= pc.N_HYPO) return;
    // Compute rigid transform from 3 point correspondences (SVD-based)
    mat4 Th = computeRigidTransform3(pA, pB, match, seeds[h]);
    uint inl = 0u;
    for (uint m=0; m<pc.N_MATCHES; m++) {
        vec3 ta = (Th * vec4(pA[m],1.0)).xyz;
        if (length(ta - pB[match[m]]) < pc.thresh) inl++;
    }
    cnt[h]=inl; T[h]=Th;
}
// CPU: pick argmax(cnt), refine with all inliers via ICP (§87)
```

[Source: Rusu et al. "Fast Global Registration", ECCV 2016; Fischler & Bolles "Random Sample Consensus: A Paradigm for Model Fitting", CACM 1981]

---

### 90. Iterative Closest Point (ICP) Registration

*Audience: systems developers, graphics application developers.*

ICP (Besl & McKay 1992) aligns two point clouds by alternating: (1) find nearest-neighbour correspondences; (2) compute the rigid transform minimising their sum-squared distance. GPU parallelism covers both steps. §89 covers FPFH-based initialisation; ICP refines the result.

### 90.1 GPU k-NN Correspondence Search

For each point in the source cloud, find the nearest point in the target cloud. Brute-force on GPU: O(N·M) dot products, but with SIMD parallelism across 10s of millions of pairs:

```glsl
// icp_nn.comp — nearest-neighbour correspondences between source and target
layout(set=0,binding=0) readonly buffer Src { vec3 src[]; };
layout(set=0,binding=1) readonly buffer Tgt { vec3 tgt[]; };
layout(set=0,binding=2) writeonly buffer Corr { uint corrIdx[]; float corrDist[]; };
layout(local_size_x=64) in;
void main() {
    uint i   = gl_GlobalInvocationID.x;
    if (i >= N_SRC) return;
    float bestD=1e30; uint bestJ=0u;
    for (uint j=0; j<N_TGT; j++) {
        float d = dot(src[i]-tgt[j],src[i]-tgt[j]);
        if (d<bestD) { bestD=d; bestJ=j; }
    }
    corrIdx[i]=bestJ; corrDist[i]=sqrt(bestD);
}
```

[Source: Besl & McKay "A Method for Registration of 3-D Shapes", IEEE TPAMI 1992; GPU ICP: Tamaki et al. "Softassign and EM-ICP on GPU", IVCNZ 2010]

### 90.2 Cross-Covariance Matrix via Parallel Reduction

From N correspondence pairs (pᵢ, qᵢ), compute the cross-covariance H = Σ (pᵢ − p̄)(qᵢ − q̄)ᵀ needed for SVD. Use a parallel reduction:

```glsl
// icp_crosscov.comp — parallel reduction of cross-covariance matrix
layout(set=0,binding=0) readonly buffer Src  { vec3 src[];  };
layout(set=0,binding=1) readonly buffer Corr { uint corrIdx[]; };
layout(set=0,binding=2) readonly buffer Tgt  { vec3 tgt[];  };
layout(set=0,binding=3) readonly buffer Means { vec3 srcMean; vec3 tgtMean; };
layout(set=0,binding=4) buffer H { float H[9]; };  // 3x3 cross-covariance
layout(local_size_x=64) in;
shared float sH[64][9];
void main() {
    uint i  = gl_GlobalInvocationID.x;
    uint li = gl_LocalInvocationID.x;
    mat3 local_H = mat3(0.0);
    if (i < N_SRC) {
        vec3 p = src[i] - srcMean;
        vec3 q = tgt[corrIdx[i]] - tgtMean;
        local_H = outerProduct(p,q);
    }
    // Store to shared, reduce
    for (int k=0;k<9;k++) sH[li][k] = local_H[k/3][k%3];
    barrier();
    for (uint s=32u;s>0u;s>>=1) {
        if (li<s) for(int k=0;k<9;k++) sH[li][k]+=sH[li+s][k];
        barrier();
    }
    if (li==0) for(int k=0;k<9;k++) atomicAdd_float(H[k], sH[0][k]);
}
```

[Source: Besl & McKay 1992; Segal & Akeley "The OpenGL Graphics System", 2004 §ICP appendix]

### 90.3 SVD-Based Transform Estimation

The optimal rotation is R = V·Uᵀ where H = U·Σ·Vᵀ. Compute on CPU after the GPU reduction for a 3×3 matrix (negligible cost); translation is t = q̄ − R·p̄:

```glsl
// icp_transform_apply.comp — apply current R,t to source point cloud
layout(set=0,binding=0) readonly buffer SrcIn  { vec3 srcIn[];  };
layout(set=0,binding=1) writeonly buffer SrcOut { vec3 srcOut[]; };
layout(set=0,binding=2) readonly buffer RT { mat3 R; vec3 t; };
layout(local_size_x=64) in;
void main() {
    uint i    = gl_GlobalInvocationID.x;
    if (i >= N_SRC) return;
    srcOut[i] = R * srcIn[i] + t;
}
// Iterate: dispatch icp_nn → icp_crosscov → CPU SVD → icp_transform_apply
// Converges in 20-50 iterations for ≤45° initial misalignment
```

### 90.4 Point-to-Plane ICP for Faster Convergence

Point-to-plane ICP (Chen & Medioni 1992) minimises the sum of squared distances along the target normal, converging roughly 10× faster than point-to-point. The optimal transform is solved by a 6×6 linear system:

```glsl
// icp_point2plane.comp — accumulate 6x6 linear system for point-to-plane ICP
layout(set=0,binding=0) readonly buffer Src     { vec3 src[];  };
layout(set=0,binding=1) readonly buffer TgtPts  { vec3 tgt[];  };
layout(set=0,binding=2) readonly buffer TgtNors { vec3 tnor[]; };
layout(set=0,binding=3) readonly buffer Corr    { uint corrIdx[]; };
layout(set=0,binding=4) buffer AtA { float ata[36]; float atb[6]; };  // 6x6 system
layout(local_size_x=64) in;
void main() {
    uint i  = gl_GlobalInvocationID.x;
    if (i>=N_SRC) return;
    vec3 p  = src[i], q = tgt[corrIdx[i]], n = tnor[corrIdx[i]];
    // Row of A: [n × p, n]
    vec3 cn = cross(p, n);
    float a[6] = {cn.x,cn.y,cn.z,n.x,n.y,n.z};
    float b    = dot(n, q-p);
    for (int r=0;r<6;r++) {
        atomicAdd_float(atb[r], a[r]*b);
        for(int c=r;c<6;c++) atomicAdd_float(ata[r*6+c], a[r]*a[c]);
    }
}
// Solve 6x6 AtA·x = Atb on CPU → x = [ω₁,ω₂,ω₃,t₁,t₂,t₃] → compose transform
```

[Source: Chen & Medioni "Object Modeling by Registration of Multiple Range Images", IVC 1992; Low "Linear Least-Squares Optimization for Point-to-Plane ICP Surface Registration", UNC TR 2004]

---


---


## Integrations

**Ch209** (this series) covers the BVH spatial data structures and GPU-driven virtual geometry that Cat IX acceleration structure construction wraps. **Ch206** (Shader Algorithm Catalog — Ray Tracing and Procedural) covers the ray tracing shader patterns (hit shaders, miss shaders, ReSTIR, denoising) that consume the acceleration geometry catalogued in Cat IX. **Ch135** (Vulkan Ray Tracing) covers the Vulkan API for acceleration structure creation, compaction, and update — the implementation layer for the algorithms in Cat IX. **Ch212** (this series) covers the neural geometry techniques (NeRF, Gaussian Splatting) that extend the point cloud representations in Cat X with learned radiance fields. **Ch27** (VR/AR and OpenXR) covers the reprojection geometry (Cat X §100) for XR applications.
