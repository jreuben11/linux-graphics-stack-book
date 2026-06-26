# Appendix P: WGSL Language Reference

> **Reference baseline**: WGSL W3C Candidate Recommendation Draft, June 2026.
> **Primary source**: [W3C WGSL specification](https://www.w3.org/TR/WGSL/)

**Audience**: Application developers and browser/web platform engineers writing WebGPU shaders for native Rust (wgpu/Bevy/Godot) or browser deployments. This appendix is a compact reference card — not a tutorial. See Chapter 35 (Dawn/Tint), Chapter 40 (Bevy/wgpu), and Chapter 98 (WebAssembly/WebGPU) for architectural context.

---

## Table of Contents

1. [Scalar and Composite Types](#1-scalar-and-composite-types)
2. [Texture and Sampler Types](#2-texture-and-sampler-types)
3. [Address Spaces](#3-address-spaces)
4. [Attributes](#4-attributes)
5. [Built-in Values (`@builtin`)](#5-built-in-values-builtin)
6. [Built-in Functions](#6-built-in-functions)
7. [Control Flow](#7-control-flow)
8. [Entry Point Structure](#8-entry-point-structure)
9. [Key Differences from GLSL](#9-key-differences-from-glsl)

---

## 1. Scalar and Composite Types

### Scalar Types

| Type   | Description                              | Notes                                     |
|--------|------------------------------------------|-------------------------------------------|
| `bool` | Boolean                                  | Not directly storable in buffers          |
| `i32`  | Signed 32-bit integer                    |                                           |
| `u32`  | Unsigned 32-bit integer                  |                                           |
| `f32`  | 32-bit float (IEEE 754)                  |                                           |
| `f16`  | 16-bit float                             | Requires `enable f16;` + `shader-f16` ext |

### Vector Types

Short-form aliases are preferred in modern WGSL:

| Full form       | Short alias | Description               |
|-----------------|-------------|---------------------------|
| `vec2<f32>`     | `vec2f`     | 2-component float vector  |
| `vec3<f32>`     | `vec3f`     | 3-component float vector  |
| `vec4<f32>`     | `vec4f`     | 4-component float vector  |
| `vec2<i32>`     | `vec2i`     | 2-component int vector    |
| `vec3<u32>`     | `vec3u`     | 3-component uint vector   |
| `vec4<bool>`    | `vec4<bool>`| 4-component bool vector   |

Component access via swizzle: `.x .y .z .w` or `.r .g .b .a`. Swizzle reordering is allowed (`v.yzx`).

### Matrix Types

Column-major storage. `matCxR<f32>` has C columns and R rows.

| Type         | Short alias | Dimensions    |
|--------------|-------------|---------------|
| `mat2x2<f32>`| `mat2x2f`   | 2 cols, 2 rows|
| `mat3x3<f32>`| `mat3x3f`   | 3 cols, 3 rows|
| `mat4x4<f32>`| `mat4x4f`   | 4 cols, 4 rows|
| `mat3x4<f32>`| `mat3x4f`   | 3 cols, 4 rows|

Matrix-vector multiply: `m * v` (matrix left, column vector right).

### Array Types

```wgsl
// Fixed-size array
var data: array<f32, 4> = array<f32, 4>(1.0, 2.0, 3.0, 4.0);

// Runtime-sized array (only valid in storage address space, last member of struct)
struct DynamicBuffer { elements: array<f32> }
```

### Struct Types

```wgsl
struct Vertex {
    @location(0) position: vec4f,
    @location(1) uv:       vec2f,
}

struct Uniforms {
    @align(16) model:      mat4x4f,
    @align(16) view_proj:  mat4x4f,
    time:                  f32,
    @size(12)  _pad:       vec3f,    // explicit padding
}
```

### Atomic Types

Only valid in `storage` or `workgroup` address space.

```wgsl
var<workgroup> counter: atomic<u32>;
var<storage, read_write> shared: atomic<i32>;
```

### Pointer Types

```wgsl
fn add_one(p: ptr<function, f32>) {
    *p = *p + 1.0;
}
```

---

## 2. Texture and Sampler Types

### Sampled Textures

| Type                         | GLSL equivalent         |
|------------------------------|-------------------------|
| `texture_1d<f32>`            | `sampler1D`             |
| `texture_2d<f32>`            | `sampler2D`             |
| `texture_2d_array<f32>`      | `sampler2DArray`        |
| `texture_3d<f32>`            | `sampler3D`             |
| `texture_cube<f32>`          | `samplerCube`           |
| `texture_cube_array<f32>`    | `samplerCubeArray`      |
| `texture_multisampled_2d<f32>`| `sampler2DMS`          |

### Depth Textures

```wgsl
texture_depth_2d
texture_depth_2d_array
texture_depth_cube
texture_depth_multisampled_2d
```

### Storage Textures (compute/write)

```wgsl
@group(0) @binding(2)
var output: texture_storage_2d<rgba8unorm, write>;
```

Access qualifiers: `read`, `write`, `read_write` (requires `bgra8unorm_storage` or format-specific caps).

### Sampler Types

```wgsl
var samp:      sampler;             // regular sampling
var samp_cmp:  sampler_comparison;  // for shadow maps
```

---

## 3. Address Spaces

| Address Space | `var<...>` syntax         | Scope            | R/W                |
|---------------|---------------------------|------------------|--------------------|
| `function`    | `var<function>` (default) | Function local   | read_write         |
| `private`     | `var<private>`            | Shader invocation| read_write         |
| `workgroup`   | `var<workgroup>`          | Workgroup        | read_write         |
| `uniform`     | `var<uniform>`            | Global / binding | read               |
| `storage`     | `var<storage, read>`      | Global / binding | read               |
| `storage`     | `var<storage, read_write>`| Global / binding | read_write         |
| `handle`      | (textures/samplers only)  | Global / binding | read               |

`uniform` buffers have a 65536-byte size limit per binding. Use `storage` for larger data.

---

## 4. Attributes

### Resource Binding

```wgsl
@group(0) @binding(0) var<uniform> camera: CameraUniforms;
@group(0) @binding(1) var          tex:    texture_2d<f32>;
@group(0) @binding(2) var          samp:   sampler;
@group(1) @binding(0) var<storage, read_write> particles: array<Particle>;
```

### Vertex Input / Output

```wgsl
struct VertIn  { @location(0) pos: vec3f, @location(1) uv: vec2f }
struct VertOut { @builtin(position) clip_pos: vec4f, @location(0) uv: vec2f }
```

### Fragment Output

```wgsl
struct FragOut {
    @location(0) color:  vec4f,   // render target 0
    @location(1) normal: vec4f,   // render target 1 (MRT)
}
```

### Interpolation

```wgsl
@interpolate(perspective, center)   // default for float
@interpolate(linear)                // no perspective correction
@interpolate(flat)                  // no interpolation (integer / provoking vertex)
@interpolate(flat, first)           // provoking vertex = first
```

### Struct Layout

```wgsl
@align(N)   // force member alignment to N bytes (must be power of 2)
@size(N)    // force member to occupy N bytes (for explicit padding)
```

### Entry-Point Tags

```wgsl
@vertex
@fragment
@compute @workgroup_size(64, 1, 1)
@compute @workgroup_size(8, 8)      // 2D workgroup
```

### Miscellaneous

```wgsl
@invariant   // on @builtin(position): guarantee same value across draw calls (avoids z-fighting)
@must_use    // return value must not be discarded
@diagnostic(off, derivative_uniformity)  // suppress a specific diagnostic
```

---

## 5. Built-in Values (`@builtin`)

### Vertex Shader Inputs

| Built-in             | Type  | Description                          |
|----------------------|-------|--------------------------------------|
| `vertex_index`       | `u32` | Index of this vertex (gl_VertexID)   |
| `instance_index`     | `u32` | Instance index (gl_InstanceID)       |

### Vertex Shader Outputs

| Built-in             | Type    | Description                             |
|----------------------|---------|-----------------------------------------|
| `position`           | `vec4f` | Clip-space position (gl_Position)       |

### Fragment Shader Inputs

| Built-in             | Type    | Description                             |
|----------------------|---------|-----------------------------------------|
| `position`           | `vec4f` | Framebuffer position (gl_FragCoord)     |
| `front_facing`       | `bool`  | True for front-facing primitive         |
| `frag_depth`         | `f32`   | Override depth output                   |
| `sample_index`       | `u32`   | MSAA sample index                       |
| `sample_mask`        | `u32`   | MSAA coverage mask (input)              |

### Fragment Shader Outputs

| Built-in             | Type  | Description                              |
|----------------------|-------|------------------------------------------|
| `frag_depth`         | `f32` | Write to depth buffer                    |
| `sample_mask`        | `u32` | Write MSAA coverage mask                 |

### Compute Shader Inputs

| Built-in                  | Type     | Description                              |
|---------------------------|----------|------------------------------------------|
| `global_invocation_id`    | `vec3u`  | Global thread ID (gl_GlobalInvocationID) |
| `local_invocation_id`     | `vec3u`  | Local thread ID within workgroup         |
| `local_invocation_index`  | `u32`    | Linearised local ID                      |
| `workgroup_id`            | `vec3u`  | Workgroup ID (gl_WorkGroupID)            |
| `num_workgroups`          | `vec3u`  | Total workgroup count                    |

---

## 6. Built-in Functions

### Math

| Function                      | Description                          |
|-------------------------------|--------------------------------------|
| `abs(e)`                      | Absolute value                       |
| `ceil(e)` / `floor(e)`        | Ceiling / floor                      |
| `clamp(e, lo, hi)`            | Clamp to range                       |
| `cos(e)` / `sin(e)` / `tan(e)`| Trigonometry (radians)               |
| `acos` / `asin` / `atan`      | Inverse trig                         |
| `atan2(y, x)`                 | 4-quadrant arctangent                |
| `cross(a, b)`                 | Cross product (`vec3f` only)         |
| `degrees(e)` / `radians(e)`   | Unit conversion                      |
| `distance(a, b)`              | Euclidean distance                   |
| `dot(a, b)`                   | Dot product                          |
| `exp(e)` / `exp2(e)`          | Exponential                          |
| `fract(e)`                    | Fractional part                      |
| `inverseSqrt(e)`              | 1 / sqrt(e)                          |
| `length(v)`                   | Vector magnitude                     |
| `log(e)` / `log2(e)`          | Logarithm                            |
| `max(a, b)` / `min(a, b)`     | Component-wise max / min             |
| `mix(a, b, t)`                | Linear interpolation (lerp)          |
| `modf(e)` → `{fract, whole}`  | Decompose into fract + integer parts |
| `normalize(v)`                | Unit vector                          |
| `pow(base, exp)`              | Power                                |
| `reflect(e, n)`               | Reflection vector                    |
| `refract(e, n, eta)`          | Refraction vector                    |
| `round(e)`                    | Round to nearest integer             |
| `sign(e)`                     | Sign (-1, 0, +1)                     |
| `smoothstep(lo, hi, x)`       | Hermite smooth interpolation         |
| `sqrt(e)`                     | Square root                          |
| `step(edge, x)`               | 0 if x < edge, else 1               |
| `trunc(e)`                    | Truncate to integer (towards zero)   |
| `saturate(e)`                 | clamp(e, 0.0, 1.0)                  |
| `select(f, t, cond)`          | Ternary: cond ? t : f               |

### Bit Manipulation

| Function                  | Description                                       |
|---------------------------|---------------------------------------------------|
| `countLeadingZeros(e)`    | Number of leading zero bits                       |
| `countOneBits(e)`         | Population count                                  |
| `countTrailingZeros(e)`   | Number of trailing zero bits                      |
| `extractBits(e, off, cnt)`| Extract bit field                                 |
| `insertBits(e, n, off, cnt)`| Insert bit field                               |
| `reverseBits(e)`          | Bit reversal                                      |
| `firstLeadingBit(e)`      | Index of highest set bit                          |
| `firstTrailingBit(e)`     | Index of lowest set bit                           |

### Type Construction and Conversion

```wgsl
f32(myInt)          // explicit cast — no implicit conversions in WGSL
vec3f(1.0)          // splat constructor
vec4f(pos, 1.0)     // vec3 + scalar
mat4x4f(col0, col1, col2, col3)  // from column vectors
```

### Texture Sampling

| Function                                    | Description                              |
|---------------------------------------------|------------------------------------------|
| `textureSample(t, s, coords)`               | Sample at coords                         |
| `textureSampleBias(t, s, coords, bias)`     | Sample with LOD bias                     |
| `textureSampleLevel(t, s, coords, lod)`     | Sample at explicit LOD                   |
| `textureSampleGrad(t, s, coords, ddx, ddy)` | Sample with explicit derivatives         |
| `textureSampleCompare(t, s, coords, depth)` | Shadow map comparison                    |
| `textureSampleBaseClampToEdge(t, s, coords)`| Sample base level, clamp to edge         |
| `textureLoad(t, coords, level)`             | Load texel without sampler               |
| `textureStore(t, coords, value)`            | Write storage texture                    |
| `textureDimensions(t)` / `textureDimensions(t, level)` | Texture size in texels      |
| `textureNumLayers(t)`                       | Array layer count                        |
| `textureNumLevels(t)`                       | Mip level count                          |
| `textureNumSamples(t)`                      | MSAA sample count                        |

### Pack / Unpack

```wgsl
pack4x8snorm(v: vec4f)   → u32   // pack 4 snorm8 values
pack4x8unorm(v: vec4f)   → u32   // pack 4 unorm8 values
pack2x16float(v: vec2f)  → u32   // pack 2 fp16 values
pack2x16snorm(v: vec2f)  → u32
pack2x16unorm(v: vec2f)  → u32
unpack4x8snorm(e: u32)   → vec4f
unpack4x8unorm(e: u32)   → vec4f
unpack2x16float(e: u32)  → vec2f
unpack2x16snorm(e: u32)  → vec2f
unpack2x16unorm(e: u32)  → vec2f
```

### Atomic Operations

All return the previous value except `atomicStore`.

| Function                                        | Description                   |
|-------------------------------------------------|-------------------------------|
| `atomicLoad(&a)`                                | Atomic read                   |
| `atomicStore(&a, v)`                            | Atomic write (returns void)   |
| `atomicAdd(&a, v)`                              | Fetch-and-add                 |
| `atomicSub(&a, v)`                              | Fetch-and-subtract            |
| `atomicMax(&a, v)` / `atomicMin(&a, v)`         | Fetch-and-max / min           |
| `atomicAnd(&a, v)` / `atomicOr(&a, v)` / `atomicXor(&a, v)` | Bitwise fetch-and-op |
| `atomicExchange(&a, v)`                         | Swap                          |
| `atomicCompareExchangeWeak(&a, cmp, v)` → `{old, exchanged}` | CAS           |

### Synchronisation (Compute Only)

```wgsl
workgroupBarrier()   // sync workgroup memory + execution
storageBarrier()     // sync storage buffer visibility across invocations
textureBarrier()     // sync texture writes across invocations
```

### Derivatives (Fragment Only)

```wgsl
dpdx(e)    // partial derivative with respect to screen x
dpdy(e)    // partial derivative with respect to screen y
fwidth(e)  // abs(dpdx(e)) + abs(dpdy(e)) — useful for anti-aliasing
```

### Subgroup Operations (extension: `enable subgroups;`)

```wgsl
subgroupBroadcast(value, sourceLaneIndex)
subgroupBroadcastFirst(value)
subgroupAdd(value)   subgroupMul(value)
subgroupMin(value)   subgroupMax(value)
subgroupAnd(value)   subgroupOr(value)   subgroupXor(value)
subgroupAll(cond)    subgroupAny(cond)
subgroupBallot(cond)          → vec4u  // bitmask of invocations where cond is true
subgroupShuffle(value, id)
subgroupShuffleXor(value, mask)
```

---

## 7. Control Flow

```wgsl
// Conditional
if condition { ... } else if other { ... } else { ... }

// Switch (value must be i32 or u32; no fallthrough)
switch x {
    case 0u { ... }
    case 1u, 2u { ... }   // multiple values per case
    default { ... }
}

// Loops
for (var i = 0u; i < 10u; i++) { ... }
while condition { ... }
loop {
    if done { break; }
    continuing { i++; }   // runs at the end of each iteration (even after continue)
}

// Early exit
break;           // exit loop or switch
continue;        // next iteration
return value;    // exit function
discard;         // discard fragment (fragment shaders only, terminates invocation)
```

---

## 8. Entry Point Structure

### Vertex Shader

```wgsl
@group(0) @binding(0) var<uniform> mvp: mat4x4f;

struct VIn  { @location(0) pos: vec3f, @location(1) uv: vec2f }
struct VOut { @builtin(position) clip: vec4f, @location(0) uv: vec2f }

@vertex
fn vs_main(in: VIn) -> VOut {
    return VOut(mvp * vec4f(in.pos, 1.0), in.uv);
}
```

### Fragment Shader

```wgsl
@group(0) @binding(1) var tex:  texture_2d<f32>;
@group(0) @binding(2) var samp: sampler;

@fragment
fn fs_main(@location(0) uv: vec2f) -> @location(0) vec4f {
    return textureSample(tex, samp, uv);
}
```

### Compute Shader

```wgsl
@group(0) @binding(0) var<storage, read_write> data: array<f32>;

@compute @workgroup_size(64)
fn cs_main(@builtin(global_invocation_id) id: vec3u) {
    let i = id.x;
    if i >= arrayLength(&data) { return; }
    data[i] = data[i] * 2.0;
}
```

---

## 9. Key Differences from GLSL

| Topic                    | GLSL                                   | WGSL                                              |
|--------------------------|----------------------------------------|---------------------------------------------------|
| **Implicit conversions** | `float f = 1;` OK                      | No — `let f = f32(1);` required                   |
| **Mutability**           | All variables mutable by default       | `let` = immutable; `var` = mutable                |
| **Entry points**         | `void main()` (one per stage)          | `@vertex`/`@fragment`/`@compute` on any function  |
| **Recursion**            | Implementation-defined                 | Disallowed by spec                                |
| **Preprocessor**         | `#ifdef`, `#include`, `#define`        | None — use `enable`, `requires`, `const`          |
| **Global mutable state** | `uniform`, `in`, `out` globals         | Only via `var<private>` or `var<workgroup>`; inputs/outputs are function parameters |
| **Texture + sampler**    | Combined `sampler2D`                   | Separate `texture_2d<f32>` and `sampler`          |
| **Precision qualifiers** | `lowp`, `mediump`, `highp`             | Not present; use `f16` for reduced precision      |
| **Opaque types in buffers** | Not allowed                         | Not allowed                                       |
| **GLSL `gl_*` globals**  | `gl_Position`, `gl_FragCoord`, etc.    | `@builtin(position)`, `@builtin(frag_coord)`, etc.|
| **Ternary operator**     | `cond ? a : b`                         | `select(a, b, cond)` (note: argument order!)      |
| **Modulo**               | `%` (undefined for negative)           | `%` (truncation semantics) or `modf`              |
| **Integer division**     | `int a / int b`                        | `i32 / i32` — same; result truncates towards zero |
| **Matrix constructor**   | Column-major, values by column         | Column-major, constructor takes column vectors    |

### Common Pitfalls

```wgsl
// select() argument order is (false_val, true_val, condition) — opposite of ternary
let y = select(0.0, 1.0, x > 0.5);   // returns 1.0 when x > 0.5

// No implicit i32 → f32
let n: f32 = f32(array_index);         // must cast explicitly

// Texture coordinates: WGSL uses normalised [0,1] for textureSample, texel for textureLoad
let colour = textureSample(tex, samp, uv);      // uv in [0,1]
let texel  = textureLoad(tex, vec2i(px, py), 0); // px, py in texel space
```

---

## Cross-References

- **Chapter 35** — Dawn/Tint: WGSL parsing, Tint IR, and SPIR-V/MSL/HLSL code generation from WGSL
- **Chapter 40** — Bevy/wgpu: naga WGSL compiler (Rust alternative to Tint), naga IR → SPIR-V → Mesa NIR
- **Chapter 61** — SPIR-V Ecosystem: front ends including WGSL → SPIR-V via Tint and naga
- **Chapter 98** — WebAssembly/WebGPU: WGSL in native wgpu embeddings and browser deployments
- **Appendix L** — Shader Toolchain Matrix: Tint and naga in the broader compiler landscape
- **Appendix O** — SPIR-V Binary Format Reference: the binary output that Tint/naga produce from WGSL

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
