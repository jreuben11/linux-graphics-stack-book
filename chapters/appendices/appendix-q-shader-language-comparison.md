# Appendix Q: Shader Language Comparison Reference

> **Reference baseline**: GLSL 4.60 / HLSL SM 6.6 / WGSL CR June 2026 / MSL 3.1 / OpenCL C 3.0
> **Audience**: Graphics application developers and browser/web platform engineers writing cross-platform shader code.

This appendix is a side-by-side reference for the five shader languages covered in this book. See Chapter 61 (SPIR-V), Chapter 77 (Shader Toolchain), Chapter 35 (Tint/WGSL), and Appendix P (WGSL reference) for depth on individual languages.

---

## Table of Contents

1. [Scalar Types](#1-scalar-types)
2. [Vector and Matrix Types](#2-vector-and-matrix-types)
3. [Storage Qualifiers / Address Spaces](#3-storage-qualifiers--address-spaces)
4. [Entry Points](#4-entry-points)
5. [Built-in Variables](#5-built-in-variables)
6. [Resource Binding](#6-resource-binding)
7. [Texture Sampling](#7-texture-sampling)
8. [Compute: Workgroup Size and Barriers](#8-compute-workgroup-size-and-barriers)
9. [Control Flow Differences](#9-control-flow-differences)
10. [Preprocessor and Modules](#10-preprocessor-and-modules)
11. [Compilation Targets and Toolchain](#11-compilation-targets-and-toolchain)

---

## 1. Scalar Types

| Concept          | GLSL          | HLSL         | WGSL          | MSL            | OpenCL C        |
|------------------|---------------|--------------|---------------|----------------|-----------------|
| Boolean          | `bool`        | `bool`       | `bool`        | `bool`         | `int` (no bool) |
| Signed int 32    | `int`         | `int`        | `i32`         | `int`          | `int`           |
| Unsigned int 32  | `uint`        | `uint`       | `u32`         | `uint`         | `uint`          |
| Float 32         | `float`       | `float`      | `f32`         | `float`        | `float`         |
| Float 16         | `float16_t` (ext) | `half`  | `f16` (`enable f16;`) | `half` | `half`        |
| Float 64         | `double`      | `double`     | —             | `double` (rare)| `double` (ext)  |
| Integer 64       | `int64_t` (ext)| `int64_t` (SM6.6) | — (u32 pairs) | `long` | `long`         |

---

## 2. Vector and Matrix Types

### Vectors

| Components | GLSL        | HLSL        | WGSL (short) | MSL          | OpenCL C     |
|-----------|-------------|-------------|--------------|--------------|--------------|
| 2×float   | `vec2`      | `float2`    | `vec2f`      | `float2`     | `float2`     |
| 3×float   | `vec3`      | `float3`    | `vec3f`      | `float3`     | `float3`     |
| 4×float   | `vec4`      | `float4`    | `vec4f`      | `float4`     | `float4`     |
| 2×int     | `ivec2`     | `int2`      | `vec2i`      | `int2`       | `int2`       |
| 4×uint    | `uvec4`     | `uint4`     | `vec4u`      | `uint4`      | `uint4`      |
| 4×bool    | `bvec4`     | `bool4`     | `vec4<bool>` | `bool4`      | —            |

Swizzle access: `.xyzw` / `.rgba` (all five languages). HLSL additionally accepts `.abgr`.

### Matrices (column-major unless noted)

| Type         | GLSL      | HLSL         | WGSL (short) | MSL          | OpenCL C     |
|--------------|-----------|--------------|--------------|--------------|--------------|
| 4×4 float    | `mat4`    | `float4x4`   | `mat4x4f`    | `float4x4`   | —            |
| 3×3 float    | `mat3`    | `float3x3`   | `mat3x3f`    | `float3x3`   | —            |
| 2×4 float    | `mat2x4`  | `float2x4`   | `mat2x4f`    | `float2x4`   | —            |

**Storage order**: GLSL, WGSL, MSL = column-major. HLSL = **row-major** by default (`row_major`/`column_major` qualifiers available). OpenCL C matrices are not a built-in type.

**Matrix × vector**: GLSL/WGSL/MSL: `mat * vec` (matrix on left). HLSL: `mul(mat, vec)` or `mul(vec, mat)` depending on row/column convention.

---

## 3. Storage Qualifiers / Address Spaces

| Concept             | GLSL               | HLSL                   | WGSL                    | MSL                      | OpenCL C             |
|---------------------|--------------------|------------------------|-------------------------|--------------------------|----------------------|
| Vertex input        | `in`               | struct member + semantic| `@location(N)` in param | `[[stage_in]]`           | function parameter   |
| Vertex output       | `out`              | struct member + semantic| `@location(N)` in return| `[[stage_out]]`          | —                    |
| Uniform buffer      | `uniform` block    | `cbuffer`              | `var<uniform>`          | `constant T& [[buffer(N)]]` | `__constant`      |
| Storage buffer      | `buffer` (SSBO)    | `RWStructuredBuffer<T>`| `var<storage, read_write>`| `device T* [[buffer(N)]]`| `__global`         |
| Shared memory       | `shared`           | `groupshared`          | `var<workgroup>`        | `threadgroup T`          | `__local`            |
| Thread-private      | (default local)    | (default local)        | `var<function>` / `let` | (default local)          | (default local)      |
| Shader-private global| `global` (no qualifier)| `static`         | `var<private>`          | (device scope not allowed)| —                  |

---

## 4. Entry Points

### Vertex

```glsl
// GLSL
void main() { gl_Position = mvp * position; }
```
```hlsl
// HLSL
float4 VSMain(float3 pos : POSITION) : SV_Position { return mul(mvp, float4(pos,1)); }
```
```wgsl
// WGSL
@vertex fn vs_main(@location(0) pos: vec3f) -> @builtin(position) vec4f {
    return mvp * vec4f(pos, 1.0);
}
```
```metal
// MSL
vertex float4 vs_main(float3 pos [[attribute(0)]], constant Uniforms& u [[buffer(0)]]) {
    return u.mvp * float4(pos, 1.0);
}
```
```c
// OpenCL C — no vertex/fragment stages; compute only
```

### Fragment

```glsl
out vec4 frag_color;
void main() { frag_color = texture(tex, uv); }
```
```hlsl
float4 PSMain(float2 uv : TEXCOORD) : SV_Target { return tex.Sample(samp, uv); }
```
```wgsl
@fragment fn fs_main(@location(0) uv: vec2f) -> @location(0) vec4f {
    return textureSample(tex, samp, uv);
}
```
```metal
fragment float4 fs_main(float2 uv [[stage_in]], texture2d<float> tex [[texture(0)]],
                         sampler samp [[sampler(0)]]) {
    return tex.sample(samp, uv);
}
```

### Compute

```glsl
layout(local_size_x=64) in;
void main() { uint i = gl_GlobalInvocationID.x; data[i] *= 2.0; }
```
```hlsl
[numthreads(64, 1, 1)]
void CSMain(uint i : SV_DispatchThreadID) { data[i] *= 2.0; }
```
```wgsl
@compute @workgroup_size(64)
fn cs_main(@builtin(global_invocation_id) id: vec3u) { data[id.x] *= 2.0; }
```
```metal
kernel void cs_main(uint i [[thread_position_in_grid]],
                    device float* data [[buffer(0)]]) { data[i] *= 2.0; }
```
```c
// OpenCL C
__kernel void cs_main(__global float* data) {
    uint i = get_global_id(0);
    data[i] *= 2.0f;
}
```

---

## 5. Built-in Variables

### Vertex Stage

| Semantic                  | GLSL                        | HLSL (SV_*)            | WGSL `@builtin`           | MSL `[[...]]`                |
|---------------------------|-----------------------------|------------------------|---------------------------|------------------------------|
| Clip-space position out   | `gl_Position`               | `SV_Position` (out)    | `position` (out)          | `[[position]]` (out)         |
| Vertex index              | `gl_VertexID`               | `SV_VertexID`          | `vertex_index`            | `[[vertex_id]]`              |
| Instance index            | `gl_InstanceID`             | `SV_InstanceID`        | `instance_index`          | `[[instance_id]]`            |

### Fragment Stage

| Semantic                  | GLSL                        | HLSL                   | WGSL `@builtin`           | MSL `[[...]]`                |
|---------------------------|-----------------------------|------------------------|---------------------------|------------------------------|
| Fragment coordinate       | `gl_FragCoord`              | `SV_Position` (in)     | `position` (in)           | `[[position]]` (in)          |
| Front facing              | `gl_FrontFacing`            | `SV_IsFrontFace`       | `front_facing`            | `[[front_facing]]`           |
| Depth output              | `gl_FragDepth`              | `SV_Depth`             | `frag_depth`              | `[[depth(any)]]`             |
| Sample index              | `gl_SampleID`               | `SV_SampleIndex`       | `sample_index`            | `[[sample_id]]`              |
| Sample mask               | `gl_SampleMask`             | `SV_Coverage`          | `sample_mask`             | `[[sample_mask]]`            |
| Discard fragment          | `discard`                   | `discard`              | `discard`                 | `discard_fragment()`         |

### Compute Stage

| Semantic                  | GLSL                        | HLSL                   | WGSL `@builtin`           | MSL `[[...]]`                        |
|---------------------------|-----------------------------|------------------------|---------------------------|--------------------------------------|
| Global thread ID          | `gl_GlobalInvocationID`     | `SV_DispatchThreadID`  | `global_invocation_id`    | `[[thread_position_in_grid]]`        |
| Local thread ID           | `gl_LocalInvocationID`      | `SV_GroupThreadID`     | `local_invocation_id`     | `[[thread_position_in_threadgroup]]` |
| Workgroup / group ID      | `gl_WorkGroupID`            | `SV_GroupID`           | `workgroup_id`            | `[[threadgroup_position_in_grid]]`   |
| Local linear index        | `gl_LocalInvocationIndex`   | `SV_GroupIndex`        | `local_invocation_index`  | `[[thread_index_in_threadgroup]]`    |
| Workgroup count           | `gl_NumWorkGroups`          | (not a built-in)       | `num_workgroups`          | (not a built-in)                     |

---

## 6. Resource Binding

### Uniform / Constant Buffers

```glsl
layout(std140, binding=0, set=0) uniform CameraBlock { mat4 mvp; };  // Vulkan GLSL
```
```hlsl
cbuffer Camera : register(b0, space0) { float4x4 mvp; };
```
```wgsl
@group(0) @binding(0) var<uniform> camera: CameraUniforms;
```
```metal
vertex float4 vs(constant CameraUniforms& cam [[buffer(0)]]) { ... }
```

### Storage Buffers

```glsl
layout(std430, set=0, binding=1) buffer Particles { Particle p[]; };
```
```hlsl
RWStructuredBuffer<Particle> particles : register(u1, space0);
StructuredBuffer<Particle>   particles_ro : register(t1);  // read-only
```
```wgsl
@group(0) @binding(1) var<storage, read_write> particles: array<Particle>;
@group(0) @binding(2) var<storage, read>       ro_buf:    array<f32>;
```
```metal
kernel void cs(device Particle* particles [[buffer(1)]],
               const device float* ro [[buffer(2)]]) { ... }
```

### Textures and Samplers

```glsl
layout(binding=2) uniform sampler2D tex;         // combined image+sampler (Vulkan: separate)
layout(binding=2) uniform texture2D tex_only;    // Vulkan: separate texture
layout(binding=3) uniform sampler   samp;        // Vulkan: separate sampler
```
```hlsl
Texture2D<float4>  tex  : register(t2);
SamplerState       samp : register(s3);
float4 c = tex.Sample(samp, uv);
```
```wgsl
@group(0) @binding(2) var tex:  texture_2d<f32>;
@group(0) @binding(3) var samp: sampler;
let c = textureSample(tex, samp, uv);
```
```metal
fragment float4 fs(texture2d<float> tex [[texture(2)]],
                   sampler samp [[sampler(3)]]) {
    return tex.sample(samp, uv);
}
```

---

## 7. Texture Sampling

| Operation                        | GLSL                                  | HLSL                                    | WGSL                                         | MSL                                        | OpenCL C                        |
|----------------------------------|---------------------------------------|------------------------------------------|----------------------------------------------|--------------------------------------------|---------------------------------|
| Sample (implicit LOD)            | `texture(tex, uv)`                    | `tex.Sample(s, uv)`                      | `textureSample(t, s, uv)`                    | `t.sample(s, uv)`                          | `read_imagef(img, smp, coord)`  |
| Sample with bias                 | `texture(tex, uv, bias)`              | `tex.SampleBias(s, uv, bias)`            | `textureSampleBias(t, s, uv, bias)`          | `t.sample(s, uv, bias(b))`                 | —                               |
| Sample explicit LOD              | `textureLod(tex, uv, lod)`            | `tex.SampleLevel(s, uv, lod)`            | `textureSampleLevel(t, s, uv, lod)`          | `t.sample(s, uv, level(lod))`              | —                               |
| Sample with gradients            | `textureGrad(tex, uv, ddx, ddy)`      | `tex.SampleGrad(s, uv, dx, dy)`          | `textureSampleGrad(t, s, uv, ddx, ddy)`      | `t.sample(s, uv, gradient2d(dx,dy))`       | —                               |
| Shadow / depth comparison        | `texture(shadowTex, vec3(uv, depth))` | `tex.SampleCmp(s, uv, depth)`            | `textureSampleCompare(t, s, uv, depth)`      | `t.sample_compare(s, uv, depth)`           | —                               |
| Load texel (no sampler)          | `texelFetch(tex, ivec2(x,y), lod)`    | `tex.Load(int3(x, y, lod))`              | `textureLoad(t, vec2i(x,y), lod)`            | `t.read(uint2(x,y), lod)`                  | `read_imagef(img, coord)`       |
| Write storage texture            | `imageStore(img, ivec2(x,y), val)`    | `img[uint2(x,y)] = val`                  | `textureStore(t, vec2i(x,y), val)`           | `t.write(val, uint2(x,y))`                 | `write_imagef(img, coord, val)` |
| Texture dimensions               | `textureSize(tex, lod)`               | `tex.GetDimensions(...)`                 | `textureDimensions(t)` / `textureDimensions(t, lod)` | `t.get_width()`, `t.get_height()` | `get_image_width(img)`      |
| Gather 4 texels                  | `textureGather(tex, s, uv, comp)`     | `tex.Gather(s, uv)`                      | — (no gather in WGSL core yet)               | `t.gather(s, uv)`                          | —                               |

---

## 8. Compute: Workgroup Size and Barriers

### Workgroup Size Declaration

```glsl
layout(local_size_x=8, local_size_y=8, local_size_z=1) in;
```
```hlsl
[numthreads(8, 8, 1)]
void CSMain(...) { ... }
```
```wgsl
@compute @workgroup_size(8, 8, 1)
fn cs_main(...) { ... }
```
```metal
[[kernel]] [[max_total_threads_per_threadgroup(64)]]
void cs_main(uint2 tid [[thread_position_in_grid]]) { ... }
// Or use [[threads_per_threadgroup]] at dispatch
```
```c
// OpenCL C: set at host via clEnqueueNDRangeKernel local_work_size
```

### Shared / Workgroup Memory

```glsl
shared float tile[64];
```
```hlsl
groupshared float tile[64];
```
```wgsl
var<workgroup> tile: array<f32, 64>;
```
```metal
threadgroup float tile[64];
```
```c
__local float tile[64];
```

### Barriers

| Barrier type                  | GLSL                       | HLSL                   | WGSL                    | MSL                              | OpenCL C                              |
|-------------------------------|----------------------------|------------------------|-------------------------|----------------------------------|---------------------------------------|
| Workgroup exec + memory       | `barrier()`                | `GroupMemoryBarrierWithGroupSync()` | `workgroupBarrier()` | `threadgroup_barrier(mem_flags::mem_threadgroup)` | `barrier(CLK_LOCAL_MEM_FENCE)` |
| Storage buffer visibility     | `memoryBarrierBuffer()`    | `DeviceMemoryBarrier()` | `storageBarrier()`     | `threadgroup_barrier(mem_flags::mem_device)` | `barrier(CLK_GLOBAL_MEM_FENCE)` |
| Texture visibility            | `memoryBarrierImage()`     | —                      | `textureBarrier()`      | `threadgroup_barrier(mem_flags::mem_texture)` | —                             |
| Exec only (no memory)         | `controlBarrier()`         | `GroupMemoryBarrier()`  | —                       | (not separately available)       | —                                     |

---

## 9. Control Flow Differences

| Feature                | GLSL       | HLSL       | WGSL           | MSL        | OpenCL C   |
|------------------------|------------|------------|----------------|------------|------------|
| Ternary operator       | `a ? b : c`| `a ? b : c`| `select(b,c,a)`| `a ? b : c`| `a ? b : c`|
| Discard fragment       | `discard`  | `discard`  | `discard`      | `discard_fragment()` | — |
| Recursion              | Allowed (impl-def) | Disallowed | Disallowed | Disallowed | Disallowed |
| Implicit conversion    | Yes        | Yes        | **No**         | Yes        | Yes        |
| `break` in switch      | Required   | Required   | Not needed (no fallthrough) | Required | Required |
| Loop `continue` block  | —          | —          | `continuing { }` | —        | —          |
| `goto`                 | No         | No         | No             | No         | No         |
| Function pointers      | No         | No         | No             | No         | No         |

---

## 10. Preprocessor and Modules

| Feature              | GLSL                        | HLSL                       | WGSL                         | MSL                    | OpenCL C                   |
|----------------------|-----------------------------|----------------------------|------------------------------|------------------------|----------------------------|
| Preprocessor         | Yes (`#define`, `#ifdef`, `#include`) | Yes            | **No** — use `const`, `enable`, `requires` | Yes (`#include`) | Yes                |
| Include system       | `#include` (GLSL 4.60, extension) | `#include`          | **No**                       | `#include`             | `#include`                 |
| Conditional compile  | `#ifdef`                    | `#if defined(...)`         | `requires` / feature gates   | `#ifdef`               | `#ifdef`                   |
| Feature extensions   | `#extension NAME : enable`  | —                          | `enable feature_name;`       | —                      | `#pragma OPENCL EXTENSION` |
| Modules              | No (GLSL 4.60)              | No (HLSL SM6.6)            | No (WIT/WESL proposed)       | No                     | No                         |
| Specialisation constants | `layout(constant_id=N) const int X = 0;` | No  | `override X: i32 = 0;`      | `[[function_constant(N)]]` | No                  |

---

## 11. Compilation Targets and Toolchain

| Language   | Primary compiler(s)                         | Output format(s)            | Linux path to GPU          |
|------------|---------------------------------------------|-----------------------------|----------------------------|
| GLSL       | `glslang` / `glslangValidator`, `shaderc`   | SPIR-V (Vulkan), GLSL IR    | Mesa NIR via `spirv_to_nir` |
| HLSL       | `dxc` (DirectXShaderCompiler), `glslang -D` | DXIL, SPIR-V                | SPIR-V → Mesa NIR (via DXVK/VKD3D); or `dxil-spirv` |
| WGSL       | `tint` (Dawn), `naga` (wgpu)                | SPIR-V, MSL, HLSL           | SPIR-V → Mesa NIR (via Dawn/Chrome or wgpu) |
| MSL        | `xcrun metal` (macOS only), `spirv-cross`   | Metal .metallib             | Not directly on Linux; cross-compile via `spirv-cross` |
| OpenCL C   | `clang` + OpenCL frontend, vendor ICDs      | SPIR-V (OpenCL env), PTX    | Mesa Clover/Rusticl (Gallium), Intel OpenCL runtime |

**Shader format conversion paths:**

```
GLSL   ──glslang──►  SPIR-V ──spirv_to_nir──► Mesa NIR ──► GPU ISA
HLSL   ──dxc──────►  SPIR-V ─────────────────────────────────────►
WGSL   ──tint/naga►  SPIR-V ─────────────────────────────────────►
MSL    ──(macOS)──►  .metallib   [not used on Linux]
OpenCL ──clang────►  SPIR-V ──Rusticl──► Mesa NIR ──────────────►
```

**`spirv-cross` cross-compilation matrix:**

| From ↓ \ To → | GLSL | HLSL | MSL | ESSL | Reflection JSON |
|----------------|------|------|-----|------|-----------------|
| SPIR-V         | ✓    | ✓    | ✓   | ✓    | ✓               |

---

## Cross-References

- **Chapter 35** — Tint: WGSL → SPIR-V/MSL/HLSL compilation
- **Chapter 61** — SPIR-V Ecosystem: all front ends and the SPIR-V binary format
- **Chapter 77** — Shader Toolchain: glslang, DXC, SPIRV-Cross, naga in depth
- **Chapter 98** — WGSL in WebGPU/WASM contexts
- **Chapter 110** — SPIR-V tooling: spirv-val, spirv-opt, spirv-dis
- **Appendix L** — Shader Toolchain Matrix: which tools read/write which formats
- **Appendix O** — SPIR-V Binary Format Reference
- **Appendix P** — WGSL Language Reference

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
