# Chapter 235: GPU Vector Graphics and 2D Path Rendering (Part XXIX)

**Audiences:** Graphics application developers implementing GPU-accelerated 2D UIs, SVG renderers, or
font systems; browser engineers mapping CSS/SVG rendering onto Vulkan compute; systems developers
building compositors with GPU-resident 2D rendering pipelines.

This chapter presents the algorithmic and API layer beneath GPU-native 2D rendering on Linux.
Ch208 §5 provides a compact introduction to Loop–Blinn Bézier rendering, MSDF fonts, and
stencil-then-cover; this chapter goes deep on all those techniques and adds stroke expansion,
Jump Flooding, Vello's full pipeline, COLRv1, compositing operators, gradient shaders, and
integration with the Linux compositor and browser stacks.

---

## Table of Contents

1. [The GPU 2D Rendering Problem](#1-the-gpu-2d-rendering-problem)
2. [Bézier Curve GPU Rendering](#2-bézier-curve-gpu-rendering)
3. [GPU Stroke Rendering](#3-gpu-stroke-rendering)
4. [Signed Distance Field Generation — Jump Flooding Algorithm](#4-signed-distance-field-generation--jump-flooding-algorithm)
5. [Multi-Channel SDF Fonts](#5-multi-channel-sdf-fonts)
6. [Vello — Vulkan Compute Path Rendering](#6-vello--vulkan-compute-path-rendering)
7. [Pathfinder — Tile-Based GPU Path Rendering](#7-pathfinder--tile-based-gpu-path-rendering)
8. [GPU SVG Rasterisation](#8-gpu-svg-rasterisation)
9. [Colour Font Rendering — COLRv1](#9-colour-font-rendering--colrv1)
10. [GPU Compositing and Blend Modes](#10-gpu-compositing-and-blend-modes)
11. [2D Gradient and Pattern GPU Rendering](#11-2d-gradient-and-pattern-gpu-rendering)
12. [Integration with the Linux Graphics Stack](#12-integration-with-the-linux-graphics-stack)
- [Integrations](#integrations)

---

## 1. The GPU 2D Rendering Problem

### 1.1 Why CPU Rasterisation Doesn't Scale

CPU-side rasterisers such as Cairo (pixman back-end) and FreeType work by sweeping scanlines through
a path's edge list and filling pixels. This model is single-threaded by nature: the scanline sweep
maintains global state (the active-edge table) that is updated sequentially as y increases.
Parallelising it across CPU cores requires splitting the path into horizontal bands — a coarse
decomposition that still leaves significant sequential bookkeeping. Throughput scales with path
complexity at roughly O(N·H) where N is edge count and H is image height.

Modern display surfaces are 4K–8K pixels wide, with UI toolkits rendering dozens of rounded
rectangles, gradient fills, and glyph paths every frame. A 4K surface at 120 Hz requires 4096×2160
pixels × 120 frames ≈ 1.07 Gpixels/s of throughput — a budget that a single CPU core cannot
sustain even for trivial fills.

GPU compute shaders decompose the problem differently. A GPU with 3,000 shader cores can process
thousands of path segments in parallel. The fundamental GPU 2D challenge is organising the inherently
ordered (painter's model) and topology-sensitive (winding-number fill) computation into data-parallel
GPU work.

### 1.2 Fill Rules

Vector graphics standards define two fill rules:

**Even-odd rule**: count the number of times a ray from the test point crosses the path boundary.
If odd, the point is inside. Implemented by toggling a 1-bit parity flag per crossing regardless of
direction.

**Non-zero winding number rule**: count crossings with sign (+1 for left-to-right crossings, −1 for
right-to-left). If the sum ≠ 0, the point is inside. Implemented with a signed accumulator.
SVG's default fill rule is non-zero; both are required for correct SVG rendering.
[Source: SVG 2 specification §11.3.2, https://www.w3.org/TR/SVG2/painting.html#FillRuleProperty]

### 1.3 Analytical Anti-Aliasing vs MSAA

MSAA evaluates a binary inside/outside test at N sub-pixel sample positions and averages the results,
giving N+1 possible coverage values. For 2D vector graphics this is insufficient at 8×MSAA: diagonal
edges and small radii exhibit staircase artifacts, and 8 samples costs 8× memory bandwidth.

Analytical anti-aliasing computes the exact fractional area of the pixel covered by the path, giving
a continuous coverage value between 0 and 1. For straight edges the coverage is a linear ramp; for
curved edges it requires integrating the implicit curve function. The Loop–Blinn renderer (§2.2) and
Vello's fine stage (§6.4) both use analytical coverage via the `fwidth`-based smooth-step
approximation and winding-number area integration respectively.

### 1.4 Stroke Expansion

A stroke is the Minkowski sum of a path with a disc of radius r/2. The boundary of the stroke
consists of two offset curves (parallel to the path at distance ±r/2), rounded end-caps, and
corner joins (round, miter, or bevel). Exact offset curves for cubics are degree-10 algebraic curves;
GPU rendering requires approximation. Section §3 covers the two main production techniques.

### 1.5 Painter's Model and Compositing Order

Vector graphics standards paint paths in document order, later paths occluding earlier ones via
Porter-Duff "over" compositing. GPU renderers must either process paths in order (serialised) or
resolve order via a depth-aware compositing step. Vello's coarse stage (§6.3) assigns paths to tiles
maintaining order within each tile; Pathfinder's composite pass (§7.4) assembles tile layers in order.

### 1.6 Linux GPU 2D Rendering Landscape

| System | Backend | Approach | Linux Status |
|---|---|---|---|
| Vello | wgpu/Vulkan | GPU compute prefix-scan pipeline | Active development, production-ready on Vulkan |
| Pathfinder | OpenGL/Metal | Tile-based mask atlas | Stable but less active; no first-class Vulkan |
| Skia Ganesh | OpenGL/Vulkan | Tessellation + stencil-then-cover | Shipping in Chrome/Android |
| Skia Graphite | Vulkan/Dawn | PathAtlas / compute path atlas | Shipping in Chrome (Linux) |
| WebRender | OpenGL | Brush shaders + picture caching | Shipping in Firefox; OpenGL only |
| Cairo OpenGL | OpenGL | CPU pixman + GL blit | Experimental; effectively unmaintained |
| NV_path_rendering | OpenGL (NVIDIA only) | Driver-level stencil-then-cover | NVIDIA proprietary; reference only |

[Source: linebender/vello, https://github.com/linebender/vello; servo/pathfinder, https://github.com/servo/pathfinder; Skia Graphite design, https://skia.org/docs/dev/design/graphite/; WebRender, https://github.com/servo/webrender]

---

## 2. Bézier Curve GPU Rendering

### 2.1 Quadratic Bézier: Implicit Half-Plane Test

A quadratic Bézier B(t) = (1-t)²P₀ + 2t(1-t)P₁ + t²P₂ can be rendered analytically using the
barycentric texture coordinate trick introduced by Blinn (2006). Assign texture coordinates:
P₀ → (0, 0), P₁ → (0.5, 0), P₂ → (1, 1). The GPU hardware-interpolates these linearly across the
triangle hull. In the fragment shader, the test `u·u - v < 0` is inside the curve; `u·u - v > 0`
is outside. No tessellation, no geometry shader — the fragment shader is a single comparison:

```glsl
// quadratic_bezier.frag
in vec2 uv;  // (u, v) interpolated texture coordinates
void main() {
    float f = uv.x * uv.x - uv.y;
    float a = clamp(0.5 - f / fwidth(f), 0.0, 1.0);
    fragColor = vec4(fillColor.rgb, fillColor.a * a);
}
```

[Source: Blinn "Rendering Vector Art on the GPU," GPU Gems 3, 2007,
https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-25-rendering-vector-art-gpu]

### 2.2 Loop–Blinn Cubic Bézier

Ch208 §5.1 shows the Loop–Blinn fragment shader (the `k³ − l·m` sign test). This section covers
what the fragment shader is computing and how the (k, l, m) coordinates are assigned per cubic type
on the CPU — the part that makes the technique non-trivial to implement.

**Cubic classification via discriminant.** Given control points P₀, P₁, P₂, P₃, form the
4×3 matrix M whose rows are the homogeneous coordinates of Pᵢ:

```
M = [P₀ 1]
    [P₁ 1]
    [P₂ 1]
    [P₃ 1]
```

Compute three determinants:
- d₁ = det(M₀₁₂) (rows 0,1,2)
- d₂ = −det(M₀₁₃)
- d₃ = det(M₀₂₃)

The discriminant is Δ = d₁²(3d₂² − 4d₁d₃). Cubic type:

| Condition | Type |
|---|---|
| d₁ ≠ 0, Δ > 0 | Serpentine (two inflection points) |
| d₁ ≠ 0, Δ = 0 | Cusp |
| d₁ ≠ 0, Δ < 0 | Loop (self-intersecting) |
| d₁ = 0, d₂ ≠ 0 | Quadratic degenerate |
| d₁ = d₂ = 0 | Linear or point degenerate |

**k,l,m assignment.** For each type, Loop & Blinn derive specific formulas for (k,l,m) at the four
control points. For the serpentine case (the most common):

```
l₀ = d₁·d₃ − d₂²  (= Δ/4d₁)
m₀ = d₁·d₂ − ... (computed from canonical form)
```

The exact per-vertex (k,l,m) values are derived from factoring the implicit cubic. The fragment
shader then evaluates f = k³ − l·m and uses `fwidth(f)` for analytical anti-aliasing (see Ch208 §5.1
for the shader code). The CPU-side classification and (k,l,m) computation can be done with ~50 lines
of arithmetic before uploading as vertex attributes.

[Source: Loop & Blinn "Resolution Independent Curve Rendering using Programmable Graphics Hardware,"
SIGGRAPH 2005, ACM TOG 24(3), https://dl.acm.org/doi/10.1145/1073204.1073303]

**Triangle hull coverage.** Each cubic is covered by a single triangle (or at most two for the loop
case). The triangle hull is formed by the control polygon convex hull or specific sub-triangles
determined by the cubic type. For loops, two overlapping triangles cover the curve such that the
implicit test cancels inside the self-intersection region.

**Performance.** No tessellation: one draw call per cubic with one triangle (or two for loops). The
fragment shader is 3–5 instructions. This outperforms CPU tessellation into polylines by 5–10× at
high zoom levels where tessellated curves need fine subdivision.

---

## 3. GPU Stroke Rendering

### 3.1 Naïve Tessellation: Limits

The simplest GPU stroke approach expands the path CPU-side into a triangle strip (two vertices per
path sample, mitered at joins) and uploads the strip as a vertex buffer. Limitations: the mesh is
static (changing stroke width or join style requires CPU re-expansion), caps and round joins need
many triangles for smooth curves, and large paths produce 10–100K triangles per frame.

### 3.2 Polar Stroking

Polar Stroking (Kilgard, "Polar Stroking: New Theory and Methods for Stroking Paths," ACM TOG 39:4,
SIGGRAPH 2020) provides a principled GPU-friendly approach. The key insight is parametrising the
stroke boundary by tangent angle θ rather than by arc length.
[Source: Kilgard, ACM TOG 2020, https://arxiv.org/abs/2007.00308]

**Core idea.** The stroke of a path segment sweeps a "polar wedge" as the tangent angle θ changes.
For a line segment (tangent constant) the stroke is a parallelogram. For a curve, the stroke outline
traces a curve in (x,y) space as θ varies from θ_start to θ_end. Tessellating in θ-space provides
a direct error bound: the geometric error of a chord with angular step Δθ is bounded by
r·(1 − cos(Δθ/2)) ≈ r·Δθ²/8 — so the number of steps needed for tolerance ε is
N ≈ sqrt(r/(8ε))·|Δθ|, computed analytically without recursion.

**Join geometry.** Corner joins between segments are handled as polar sweeps of the join arc:
- **Round join**: sweep the disc endpoint through [θ_out, θ_in] in Δθ increments.
- **Bevel join**: single triangle connecting the two stroke endpoints.
- **Miter join**: extend both stroke edges until they meet, clamp by the miter-limit ratio.

**Dash patterns.** Arc length parameterisation of the path allows dash-gap patterns to be applied
before polar expansion: compute cumulative arc length at each control point (analytically for line
segments, via Gauss–Legendre quadrature for cubics), then discard or subdivide path segments that
fall in gap intervals.

### 3.3 GPU-Friendly Stroke Expansion

Levien and Uguray, "GPU-Friendly Stroke Expansion," High Performance Graphics 2024 (Best Paper),
describe a fully parallel compute-shader approach to stroke expansion that avoids CPU preprocessing
entirely. This is the algorithm used in Vello's flatten stage.
[Source: Levien & Uguray, HPG 2024, https://linebender.org/gpu-stroke-expansion-paper/]

**Algorithm outline:**
1. Decompose each cubic Bézier into Euler spiral segments meeting a flatness tolerance.
2. For each Euler spiral segment, compute its parallel curve analytically (Euler spirals have
   closed-form parallel curves as Euler spirals with adjusted parameters).
3. Output line segments or circular arcs approximating the stroke boundary.

Each step is independently parallelisable across path segments. The compute shader dispatches one
workgroup per input segment; output segment count is predicted analytically (no dynamic allocation
mid-pass), enabling a single prefix-scan to allocate output buffers.

### 3.4 Round Cap, Miter, Bevel — GPU Geometry

```glsl
// join_expand.comp  —  miter join on GPU
layout(local_size_x = 64) in;
layout(set=0, binding=0) readonly  buffer PathPts  { vec2 pts[]; };
layout(set=0, binding=1) writeonly buffer StripOut { vec4 verts[]; }; // (x,y,nx,ny)
layout(push_constant) uniform PC { float halfWidth; float miterLimit; } pc;

void main() {
    uint i = gl_GlobalInvocationID.x;
    if (i + 2 >= arrayLength(pts)) return;
    vec2 a = pts[i], b = pts[i+1], c = pts[i+2];
    vec2 d0 = normalize(b - a), d1 = normalize(c - b);
    vec2 n0 = vec2(-d0.y, d0.x), n1 = vec2(-d1.y, d1.x);
    // miter direction: bisector normal
    vec2 bisect = normalize(n0 + n1);
    float miterLen = 1.0 / max(dot(bisect, n0), 0.001);
    if (miterLen > pc.miterLimit) {
        // degenerate to bevel: emit two vertices at n0 and n1
        verts[i*4+0] = vec4(b + n0 * pc.halfWidth, n0);
        verts[i*4+1] = vec4(b + n1 * pc.halfWidth, n1);
    } else {
        vec2 miterPt = b + bisect * (pc.halfWidth * miterLen);
        verts[i*4+0] = vec4(miterPt, bisect);
        verts[i*4+1] = vec4(miterPt, bisect);
    }
}
```

---

## 4. Signed Distance Field Generation — Jump Flooding Algorithm

### 4.1 JFA Overview

The Jump Flooding Algorithm (Rong & Tan, "Jump Flooding in GPU with Applications to Voronoi Diagram
and Distance Transform," I3D 2006) computes an approximate 2D Voronoi/SDF in O(log N) passes, each
amenable to GPU parallelisation.
[Source: Rong & Tan, I3D 2006, https://citeseerx.ist.psu.edu/document?doi=3049764dbcea48f64bce8e4b2de5d68ff88fee4e]

### 4.2 Algorithm

**Seeding phase.** Initialise a storage image (e.g., `R32G32_SFLOAT`) with resolution N×N:
- Boundary pixels (those on the shape edge) store their own (x, y) pixel coordinates.
- Interior and exterior pixels store (∞, ∞) — no seed assigned yet.

**JFA passes.** For k = ⌈log₂ N⌉ passes, step size s = N/2^(k−p+1) for pass p (halving each pass):
each pixel samples its 8 neighbors at offset (±s, ±s) (not just cardinal directions), and updates
its nearest seed if the neighbor holds a seed closer than the current best:

```glsl
// jfa.comp — one JFA pass
layout(set=0, binding=0) uniform sampler2D seedIn;   // (seed_x, seed_y) or (INF, INF)
layout(set=0, binding=1, rg32f) uniform image2D seedOut;
layout(push_constant) uniform PC { int step; } pc;

void main() {
    ivec2 px = ivec2(gl_GlobalInvocationID.xy);
    vec2 best = texelFetch(seedIn, px, 0).xy;
    float bestDist = (best.x > 1e9) ? 1e38 : length(vec2(px) - best);

    for (int dy = -1; dy <= 1; dy++) {
        for (int dx = -1; dx <= 1; dx++) {
            ivec2 nb = px + ivec2(dx, dy) * pc.step;
            if (any(lessThan(nb, ivec2(0))) ||
                any(greaterThanEqual(nb, imageSize(seedOut)))) continue;
            vec2 nbSeed = texelFetch(seedIn, nb, 0).xy;
            if (nbSeed.x > 1e9) continue;
            float d = length(vec2(px) - nbSeed);
            if (d < bestDist) { bestDist = d; best = nbSeed; }
        }
    }
    imageStore(seedOut, px, vec4(best, 0.0, 0.0));
}
```

After all passes, the signed distance at pixel (x,y) is `±length(vec2(x,y) - seed(x,y))`, where the
sign is determined by a separate inside/outside test (e.g., the original rasterised path).

**Vulkan pass structure.** Two `VkImage` storage images are ping-ponged:
- Bind pass-p input image as sampler + pass-p output as storage image.
- Push constant `step = N >> (p+1)`.
- Dispatch N²/64 workgroups.

Total: ⌈log₂ N⌉ × 9 texture fetches per pixel, compared to an exact BFS flood which has O(N²)
serial dependency.

### 4.3 Exact SDF vs JFA Approximation Error

JFA is an approximation: it can miss the nearest seed when the seed lies at the boundary of the jump
offset, producing errors up to ~0.5 pixels. For font SDF atlas baking (where accuracy of ±1px in a
64px glyph is acceptable) JFA is sufficient. For applications requiring sub-pixel accuracy (e.g.,
collision SDFs for physics), exact methods such as the multi-phase DFDB (Distance Field with
Dead-reckoning and Boundary) or the exact EDT (Euclidean Distance Transform via separable passes)
are preferred, at higher compute cost.

### 4.4 Applications

- **Font SDF atlas baking**: render glyphs to a high-resolution binary bitmap, run JFA offline, store
  the resulting SDF in a compact atlas. Runtime rendering samples the atlas and thresholds.
- **UI drop shadow**: run JFA on the UI layer alpha mask, produce a distance field, then evaluate
  a Gaussian-shaped falloff `exp(−d²/(2σ²))` in a fragment shader to produce a soft shadow.
- **2D collision detection**: store the SDF of static obstacles; query SDF at runtime to get
  penetration depth and gradient direction for collision response.

---

## 5. Multi-Channel SDF Fonts

### 5.1 Single-Channel SDF Limitations

A single-channel SDF assigns one distance value per pixel. At rendering time, a threshold at 0.5
gives the inside/outside boundary. This works for smooth curves but rounds sharp corners: the SDF
of a sharp corner is a circular arc in the distance field, not a true corner. At display sizes larger
than the glyph's texel resolution, this produces visibly rounded corners even on letterforms
intended to be sharp (e.g., the corner of 'L' or 'V').

### 5.2 MSDF Algorithm (Chlumský 2017)

Multi-channel SDF (MSDF) encodes three independent signed pseudo-distance fields — one per RGB
channel — such that sharp corners are encoded as a discontinuity across channels, not averaged away.
[Source: Chlumský, "Shape Decomposition for Multi-Channel Distance Fields," Master's Thesis, CTU
Prague 2017, https://github.com/Chlumsky/msdfgen]

**Pseudo-distance vs signed distance.** The signed distance at point P to a curve segment is the
perpendicular distance to the nearest point on the segment, with sign determined by which side P
lies. The pseudo-distance extends the segment beyond its endpoints: if P's projection falls outside
the segment's parameter range [0,1], the pseudo-distance is the distance to the endpoint, signed by
the endpoint's tangent normal. This makes the pseudo-distance field continuous across the segment's
end, enabling clean multi-channel blending.

**Edge coloring.** The shape's edges are assigned to colour channels R, G, B so that:
- Edges meeting at angles below a threshold (default 3.0 rad) get the same colour.
- Edges meeting at sharp corners get different colours.
- The algorithm (`edgeColoringSimple`) iterates edges and assigns colours avoiding same-colour
  adjacency at corners. This ensures that each corner's discontinuity is captured in at least one
  channel's pseudo-distance.

**msdfgen API:**

```cpp
#include <msdfgen.h>
using namespace msdfgen;

// Load glyph from font
FreetypeHandle *ft = initializeFreetype();
FontHandle *font = loadFont(ft, "NotoSans.ttf");
Shape shape;
double advance;
loadGlyph(shape, font, 'A', &advance);

// Prepare shape
shape.normalize();
edgeColoringSimple(shape, 3.0);  // 3.0 rad = ~172° corner threshold

// Generate MSDF into a 32x32 bitmap
Bitmap<float, 3> msdf(32, 32);
SDFTransformation xform(
    Projection(32.0 / advance, Vector2(0.125, 0.125)),
    Range{4.0}  // 4-pixel signed distance range
);
generateMSDF(msdf, shape, xform);

// Save as PNG for atlas packing
savePng(msdf, "glyph_A.png");

destroyFont(font);
deinitializeFreetype(ft);
```

[Source: msdfgen library, https://github.com/Chlumsky/msdfgen]

### 5.3 Atlas Generation Pipeline

Production usage: `msdf-atlas-gen` (a companion CLI tool) processes an entire font file and outputs:
- A packed atlas PNG/BMP with all glyph MSDFs.
- A JSON/CSV metrics file with each glyph's UV rectangle, advance, and bearing.

```bash
msdf-atlas-gen -font NotoSans.ttf -charset ASCII -size 48 \
    -pxrange 4 -format png -imageout atlas.png -json atlas.json
```

The atlas is uploaded as a `VkImage` with `VK_FORMAT_R8G8B8A8_UNORM`. At runtime the renderer
indexes into the atlas by UV and calls the median-of-three shader (see Ch208 §5.2 for the GLSL
listing). The `pxRange` uniform value must match the generation parameter.

### 5.4 Large Display Sizes and MTSDF

At large font sizes (>72px rendered), MSDF corner fidelity is excellent but the fixed atlas texel
count becomes the limiting factor: a 32×32 atlas glyph shown at 200px has 6× magnification, and
individual texels of the MSDF become visible as interpolation artifacts.

The MTSDF variant (Multi-channel + True SDF, alpha channel) stores a conventional SDF in the alpha
channel alongside the MSDF RGB. At large sizes the alpha SDF provides overall correct shape; the RGB
channels handle corner sharpness. The `generateMTSDF()` call in msdfgen produces this 4-channel
output with `Bitmap<float, 4>`.

---

## 6. Vello — Vulkan Compute Path Rendering

Vello (formerly piet-gpu) is a Rust GPU path renderer developed at Google and the Linebender
community. It uses a fully GPU-resident pipeline with no CPU-side rasterisation. On Linux, Vello
uses wgpu with the Vulkan backend.
[Source: linebender/vello, https://github.com/linebender/vello]

### 6.1 CPU-Side Scene Encoding

The application builds a scene using the `vello::Scene` API:

```rust
use vello::{Scene, Renderer, RendererOptions, RenderParams, AaConfig};
use vello::peniko::{Color, Fill, Stroke, Join, Cap};
use vello::kurbo::{BezPath, Rect, Circle};

let mut scene = Scene::new();

// Filled path
let mut path = BezPath::new();
path.move_to((10.0, 10.0));
path.curve_to((30.0, 0.0), (70.0, 0.0), (90.0, 10.0));
path.line_to((90.0, 90.0));
path.close_path();
scene.fill(Fill::NonZero, Affine::IDENTITY, Color::rgb8(0x40, 0x80, 0xff), None, &path);

// Stroked path
let stroke = Stroke::new(4.0).with_join(Join::Round).with_caps(Cap::Round);
scene.stroke(&stroke, Affine::IDENTITY, Color::WHITE, None, &Circle::new((50.0, 50.0), 40.0));

// Compositing layer with blend mode
scene.push_layer(Mix::Multiply, 0.8, Affine::IDENTITY, &bounds);
// ... inner content ...
scene.pop_layer();
```

Internally, `Scene` delegates to `vello_encoding::Encoding`, which packs draw commands into
compact GPU-friendly arrays:
- `DrawTag` + `DrawMonoid` per draw call (type tag, path index, colour/gradient handle).
- `PathEncoder` output: compact cubic/line segment buffer.
- `Transform` buffer: affine matrices indexed per draw call.
- `Style` buffer: fill rule, stroke parameters.

[Source: vello_encoding crate, https://github.com/linebender/vello/tree/main/vello_encoding]

### 6.2 Flatten Stage (GPU Compute)

The flatten shader converts cubic Béziers into line segments using the GPU-Friendly Stroke Expansion
algorithm (Levien & Uguray, HPG 2024, §3.3) for strokes, and an Euler spiral–based analytical
flattening for fills. Key properties:

- Segment count per input curve is computed analytically before the shader runs, enabling a
  prefix-scan to pre-allocate the output buffer (no dynamic allocation in the shader).
- Each cubic is independently processed: `gl_GlobalInvocationID.x` = segment index.
- Error tolerance is configurable (`RenderParams::tolerance`).

The flatten shader is written in WGSL (`vello_shaders/shader/flatten.wgsl`) and compiles to SPIR-V
for the Vulkan backend.
[Source: flatten.wgsl, https://github.com/linebender/vello/blob/main/vello_shaders/shader/flatten.wgsl]

### 6.3 Coarse Stage — Tile Binning

The coarse stage assigns each draw object (path + style) to the set of 16×16px tiles it intersects.
This cannot be done naïvely (direct atomic per-tile counters create contention for large paths
covering many tiles), so Vello uses a two-phase GPU prefix scan:

1. Each draw object atomically increments a coarse-bin counter for each tile it touches.
2. An exclusive prefix scan over the bin-counter array computes the output offset for each
   (draw object, tile) pair without races.
3. A scatter pass places each draw reference into its tile's command list at the computed offset.

The coarse shader (`coarse.wgsl`) reads `draw_monoids`, `bin_headers`, `paths`, and `tiles`, and
writes per-tile command lists (the `ptcl` array). Commands include `Fill`, `Clip`, `Color`,
`LinearGradient`, `Image`, and `Jump` (when a tile's command list overflows its inline buffer and
needs to continue in a heap allocation).
[Source: coarse.wgsl, https://github.com/linebender/vello/blob/main/vello_shaders/shader/coarse.wgsl]

### 6.4 Fine Stage — Per-Tile Winding Number Accumulation

The fine shader is the rasterisation kernel. One workgroup processes one 16×16 tile:

- **Shared memory**: `sh_winding[16]` (row winding accumulators), `sh_winding_y[16]`
  (column prefix sums), `sh_samples[16*16]` (per-pixel coverage).
- **Segment processing**: for each line segment assigned to this tile, the shader computes
  which pixels its scanline crossing updates and increments/decrements the appropriate winding
  accumulator using packed SWAR (SIMD Within A Register) 8-bit arithmetic.
- **Fill rule**: non-zero uses `atomicAdd` on signed 8-bit packed values; even-odd uses `XOR`.
- **Prefix sums**: two-stage prefix sum propagates winding numbers in x then y across the tile.
- **Coverage**: each pixel counts non-zero-winding samples using bit-parallel operations on
  32-bit integers packing 4 pixels worth of winding bits.
- **Output**: pixel RGBA written to the output `VkImage` after alpha-compositing with the draw
  command's colour/gradient.

[Source: fine.wgsl, https://github.com/linebender/vello/blob/main/vello_shaders/shader/fine.wgsl]

### 6.5 Renderer API and Vulkan Integration

```rust
// Create renderer from wgpu device
let renderer = Renderer::new(
    &device,
    RendererOptions {
        surface_format: Some(wgpu::TextureFormat::Bgra8UnormSrgb),
        use_cpu: false,
        antialiasing_support: AaSupport::area_only(),
        num_init_threads: NonZeroUsize::new(1),
    },
)?;

// Render scene to wgpu texture
renderer.render_to_texture(
    &device,
    &queue,
    &scene,
    &texture_view,
    &RenderParams {
        base_color: Color::WHITE,
        width: 1920,
        height: 1080,
        antialiasing_method: AaConfig::Area,
    },
)?;
```

On Linux, `wgpu` selects `wgpu::Backends::VULKAN`, issuing `vkCmdDispatch` calls for each pipeline
stage. The rendered output texture can be exported via `VK_EXT_external_memory_dma_buf` for
zero-copy Wayland display (§12.4).

---

## 7. Pathfinder — Tile-Based GPU Path Rendering

Pathfinder is a Rust GPU path renderer with a tile-based bin-and-render architecture developed at
Mozilla. It focuses on font rendering quality (hinting, subpixel AA, stem darkening) alongside
general path rendering.
[Source: servo/pathfinder, https://github.com/servo/pathfinder]

### 7.1 Path-to-Tile Decomposition

Pathfinder divides the target surface into tiles of 16×16 pixels (configurable). Each path is
decomposed into:
- **Solid tiles**: tiles entirely inside the filled path (or entirely opaque solid fills).
- **Mask tiles**: tiles that partially intersect the path boundary and require alpha coverage.

The decomposition runs on the CPU for Pathfinder's D3D9-level mode (using SIMD and Rayon for
parallelism) or on the GPU in D3D11-level mode.

### 7.2 Mask Tile Generation

For each mask tile, Pathfinder rasterises the path boundary segments that cross that tile into a
small alpha-coverage texture (the mask tile atlas). A compute shader or the CPU evaluates segment
coverage using analytic anti-aliasing (similar to the span-coverage approach in antigrain geometry).

The mask atlas is a `VkImage` (or `GLTexture`) packed with all partial-coverage tiles for the frame.
Each tile slot is 16×16 pixels; the atlas is typically 512×512 or 1024×1024 for a typical UI frame.

### 7.3 Solid Tile Fast Path

Tiles fully inside a filled solid-colour path need no alpha mask: the composite pass draws them as
opaque textured quads with a flat colour, skipping the mask atlas lookup entirely. This fast path
amortises cost for large filled shapes (e.g., a background rectangle covering most of the screen).

### 7.4 Composite Pass

The composite pass combines all tiles into the final framebuffer in document order:
1. Draw solid-colour tiles as full-coverage quads.
2. Draw mask tiles with the alpha mask applied (multiply colour by mask alpha).
3. Maintain painter's model ordering by processing layers in document order.

### 7.5 Rust API

Pathfinder's crates follow the `pathfinder_` prefix naming convention:

```rust
use pathfinder_canvas::{CanvasRenderingContext2D, Path2D, FillStyle};
use pathfinder_geometry::vector::Vector2F;
use pathfinder_renderer::gpu::options::{DestFramebuffer, RendererMode, RendererOptions};
use pathfinder_renderer::renderer::Renderer;
use pathfinder_gl::{GLDevice, GLVersion};

let canvas = CanvasRenderingContext2D::new(/* surface */ ...);
let mut path = Path2D::new();
path.move_to(Vector2F::new(10.0, 10.0));
path.line_to(Vector2F::new(90.0, 90.0));
canvas.fill_path(path, FillRule::Winding);
let scene = canvas.into_canvas().into_scene();
```

[Source: pathfinder_canvas crate, https://crates.io/crates/pathfinder_canvas]

### 7.6 Backend Support and Linux Status

Pathfinder supports OpenGL 3.0+, OpenGL ES 3.0+, Metal, and WebGL 2. There is no first-class
Vulkan backend, which limits its use in Vulkan-native Linux environments. On Linux the OpenGL
backend is used, which adds a dependency on `libGL` and indirect overhead through Mesa's driver
dispatch layer.

---

## 8. GPU SVG Rasterisation

### 8.1 Path Command Decoding on GPU

SVG path data (`d` attribute) is a sequence of commands: `M` (moveto), `L` (lineto), `H` (horizontal
line), `V` (vertical line), `C` (cubic bezier), `Q` (quadratic bezier), `A` (elliptical arc),
`Z` (closepath), each with implicit and absolute/relative variants. For GPU-side decoding, the path
is pre-parsed on the CPU into a flat tagged-union buffer:

```c
struct PathCmd {
    uint32_t tag;    // CMD_MOVE, CMD_LINE, CMD_CUBIC, CMD_QUAD, CMD_ARC, CMD_CLOSE
    float    pts[6]; // up to 3 control points (x,y pairs)
};
```

A GPU compute shader iterates over the command buffer, builds line-segment lists per subpath via
de Casteljau flattening (§8.2), and feeds the result into the scanline fill accumulator (§8.3).

### 8.2 Bézier Flattening — de Casteljau Adaptive Subdivision

A cubic Bézier is flat enough to replace with a chord when the deviation of the control polygon from
the chord satisfies the flatness criterion. The standard measure is the maximum distance of P₁ and P₂
from the chord P₀P₃:

```
flatness = max(cross(d, P₁ - P₀), cross(d, P₂ - P₀)) / |P₃ - P₀|
           where d = normalize(P₃ - P₀)
```

If `flatness · screen_scale < tolerance` (typically 0.25px), the curve is output as a single line
segment. Otherwise, split at t=0.5 via de Casteljau and recurse. GPU implementations dispatch
recursion as a BFS over a fixed-depth stack in shared memory (max depth ~8 gives 1/256px flatness).

For Vello's uses, Levien's closed-form segment-count formula (§3.3) replaces the recursive approach
with a single parallel dispatch.

### 8.3 Scanline Fill via Winding Number Accumulation

Given a set of line segments from §8.2, the GPU fill pass counts signed crossings per scanline row.
One approach uses `imageAtomicAdd` on an `R32I` image (one integer per pixel row per column): for
each segment from (x₀,y₀) to (x₁,y₁), compute the signed integer crossing count for each pixel
row it spans and atomically add ±1. A second pass reads the row counters and applies the fill rule.

The atomics-based approach has contention when many segments hit the same row. Vello's tile-based
winding accumulation (§6.4) avoids this by partitioning work into non-overlapping 16×16 tiles.

### 8.4 Clip Path GPU Evaluation

SVG `clipPath` elements define a mask that clips another element. GPU evaluation:
1. Render the clip path's fill into an offscreen `VkImage` used as a single-channel alpha mask.
2. In the composite pass, multiply the clipped element's alpha by the clip mask's alpha at each pixel.

Nested clips are resolved by compositing multiple clip masks with the 'in' Porter-Duff operator
(α_out = α_src × α_clip).

### 8.5 SVG Filter Effects on GPU

SVG filters are applied as compute passes after path rasterisation:

**feGaussianBlur**: two-pass separable Gaussian. Horizontal pass: each thread averages a row of
pixels with Gaussian weights (precomputed in a UBO). Vertical pass: same along columns. Total cost:
O(2·N·σ) per pixel for σ-wide kernel, or O(N) using recursive Deriche approximation.

**feComposite**: Porter-Duff compositing of two source images. A compute shader reads two `VkImage`
inputs and writes the output. Operator selection via push constant (over/in/out/atop/xor/arithmetic).

**feColorMatrix**: 4×5 colour transform matrix applied per pixel. GLSL:
```glsl
vec4 color = texture(src, uv);
fragColor = clamp(mat4(m) * color + vec4(m[4], m[9], m[14], m[19]), 0.0, 1.0);
```
where `m[0..19]` is the 4×5 matrix in row-major order.
[Source: SVG Filter Effects specification, https://www.w3.org/TR/filter-effects-1/]

### 8.6 SVG Gradient Shader Evaluation

**Linear gradient** — project onto the gradient vector:
```glsl
uniform vec2 p0, p1;
uniform sampler1D colorRamp;  // pre-sampled color stops
float t = dot(fragPos - p0, p1 - p0) / dot(p1 - p0, p1 - p0);
vec4 color = texture(colorRamp, clamp(t, 0.0, 1.0));
```

**Radial gradient** — two-focal-point form (SVG spec §13.4.3). Solve for parameter t:
the quadratic `a·t² + b·t + c = 0` where a, b, c are functions of the fragment position and
the two circle centers/radii. Take the largest real root with r(t) ≥ 0:
```glsl
vec2 dp = fragPos - c0;
float a = dot(dc, dc) - dr*dr;
float b = 2.0 * (dot(dp, dc) - r0*dr);
float c_coef = dot(dp, dp) - r0*r0;
float disc = b*b - 4.0*a*c_coef;
float t = (-b + sqrt(max(disc, 0.0))) / (2.0 * a);
```
[Source: SVG2 §13.4.3, https://www.w3.org/TR/SVG2/pservers.html#RadialGradients]

**Conic gradient** — angle-based:
```glsl
float t = (atan(fragPos.y - center.y, fragPos.x - center.x) - startAngle)
          / (endAngle - startAngle);
t = fract(t);  // repeat
```

---

## 9. Colour Font Rendering — COLRv1

### 9.1 COLRv1 Paint Graph

COLRv1 (OpenType 1.9+) defines colour glyphs as a directed acyclic graph (DAG) of 32 paint table
formats. The DAG has leaf nodes (fill operations) and interior nodes (transform, clip, composite):

**Leaf nodes** (generate pixel colour):
- `PaintSolid` (fmt 2/3): flat CPAL palette colour with alpha.
- `PaintLinearGradient` (fmt 4/5): three-point linear gradient (p₀, p₁, p₂ define direction and rotation).
- `PaintRadialGradient` (fmt 6/7): two-circle radial gradient.
- `PaintSweepGradient` (fmt 8/9): angular sweep around a center point.

**Clip nodes**:
- `PaintGlyph` (fmt 1): clips its child sub-graph to a glyph outline's filled region.

**Compositing**:
- `PaintComposite` (fmt 32): applies Porter-Duff or blend-mode compositing between two sub-graphs.

**Layering**:
- `PaintColrLayers` (fmt 11): references a slice of the `LayerList` table — N consecutive layers composited in z-order.
- `PaintColrGlyph` (fmt 10): re-uses another color glyph's DAG by ID (for shared sub-graphs).

**Transform nodes** (fmt 12–31): `PaintTransform` (full 2×3 affine), `PaintTranslate`,
`PaintScale`, `PaintScaleAroundCenter`, `PaintScaleUniform`, `PaintRotate`, `PaintRotateAroundCenter`,
`PaintSkew`, `PaintSkewAroundCenter` — each with a `Var` variant for variation support.

[Source: OpenType COLR table specification,
https://learn.microsoft.com/en-us/typography/opentype/spec/colr]

### 9.2 GPU COLRv1 Evaluation

For GPU-native COLRv1 rendering (as needed for animated emoji or very high-DPI displays):

1. **DAG traversal** (CPU): traverse the paint graph, building an ordered list of GPU draw calls.
2. **For each `PaintGlyph`**: render the glyph outline as an alpha mask (stencil or SDF lookup).
3. **For each gradient leaf**: invoke the appropriate gradient fragment shader (§11) into an
   intermediate FBO.
4. **For each `PaintComposite`**: blend source and backdrop FBOs using the specified compositing
   mode (§10).
5. **For each `PaintColrLayers`**: composite N layers bottom-up into a single FBO.

Intermediate FBOs use `VK_FORMAT_R8G8B8A8_UNORM` at the glyph's render resolution.
Gradient colour stops are stored in a 1D `VK_FORMAT_R8G8B8A8_UNORM` texture (the colour ramp).

### 9.3 COLRv1 Glyph Atlas

For UI rendering where glyphs are rendered repeatedly at fixed sizes, pre-render each COLRv1 glyph
into a packed atlas (similar to the MSDF atlas of §5.3). The atlas is updated when a new glyph/size
combination is requested. Runtime rendering is then a simple textured quad with an atlas UV lookup —
no per-frame gradient evaluation.

### 9.4 Comparison with Other Colour Font Formats

| Format | Scalable | GPU-Friendly | Editor Support | Typical Use |
|---|---|---|---|---|
| CBDT/CBLC | No (bitmap) | Yes (pre-rasterised) | Wide | Emoji at fixed sizes |
| SVG-in-OpenType | Yes | Limited (SVG parser needed) | Growing | Complex vector emoji |
| COLRv1 | Yes | Yes (structured DAG) | Wide (Harfbuzz, FreeType 2.13+) | Emoji, colour icons |
| MSDF (Ch235 §5) | Yes | Yes (atlas sample) | Custom | Text glyphs, not emoji |

FreeType 2.13+ includes a software COLRv1 renderer (`FT_CONFIG_OPTION_SVG`). GPU-native evaluation
(§9.2) bypasses FreeType and renders directly, eliminating the CPU rasterisation bottleneck.
[Source: FreeType COLRv1, https://freetype.org/freetype2/docs/reference/ft2-color_management.html]

---

## 10. GPU Compositing and Blend Modes

### 10.1 Porter-Duff Operators in GLSL

All GPU 2D rendering uses premultiplied alpha: store (R·α, G·α, B·α, α) rather than (R, G, B, α).
This allows addition and linear blending to work correctly with no conditional branches.

The Porter-Duff "over" operator (the default for vector graphics layering) in premultiplied form:
```glsl
// Porter-Duff over: dst = src over dst
vec4 porter_duff_over(vec4 src, vec4 dst) {
    return src + dst * (1.0 - src.a);
}
```

Other operators:
```glsl
vec4 porter_duff_in(vec4 src, vec4 dst)  { return src * dst.a; }
vec4 porter_duff_out(vec4 src, vec4 dst) { return src * (1.0 - dst.a); }
vec4 porter_duff_atop(vec4 src, vec4 dst){
    return src * dst.a + dst * (1.0 - src.a);
}
vec4 porter_duff_xor(vec4 src, vec4 dst) {
    return src * (1.0 - dst.a) + dst * (1.0 - src.a);
}
```

[Source: Porter & Duff, "Compositing Digital Images," SIGGRAPH 1984,
https://dl.acm.org/doi/10.1145/800031.808606]

### 10.2 CSS/SVG Blend Modes in Compute

CSS Compositing and Blending Level 1 defines 16 blend modes applied before Porter-Duff compositing.
Blend modes operate on non-premultiplied (straight) colour values Cs (source) and Cb (backdrop):

```glsl
// CSS blend modes — straight alpha, per-channel
float blend_multiply(float s, float b) { return s * b; }
float blend_screen(float s, float b)   { return s + b - s * b; }
float blend_overlay(float s, float b)  {
    return (b < 0.5) ? 2.0*s*b : 1.0 - 2.0*(1.0-s)*(1.0-b);
}
float blend_hard_light(float s, float b) { return blend_overlay(b, s); }
float blend_difference(float s, float b) { return abs(s - b); }
float blend_color_dodge(float s, float b) {
    return (s >= 1.0) ? 1.0 : min(b / (1.0 - s), 1.0);
}
float blend_color_burn(float s, float b)  {
    return (s <= 0.0) ? 0.0 : 1.0 - min((1.0 - b) / s, 1.0);
}
float blend_soft_light(float s, float b) {
    // W3C spec formula
    if (s <= 0.5) return b - (1.0-2.0*s)*b*(1.0-b);
    float d = (b <= 0.25) ? ((16.0*b-12.0)*b+4.0)*b : sqrt(b);
    return b + (2.0*s-1.0)*(d-b);
}
```

The compositing formula using a blend mode:
`Cr = (1 - αs)·Cb + (1 - αb)·Cs + αs·αb·B(Cs,Cb)`
where B is the blend function. Then apply Porter-Duff 'over' in premultiplied space.

[Source: CSS Compositing and Blending Level 1, W3C, https://www.w3.org/TR/compositing-1/]

### 10.3 GPU Layer Compositing Pipeline

Each CSS/SVG layer with a non-default blend mode is rendered to an offscreen `VkImage`
(`VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT`). The composite pass then:
1. Samples the layer texture as Cs (source).
2. Samples the accumulated backdrop as Cb (destination).
3. Evaluates the blend function, premultiplies, composites.
4. Writes to the output surface.

For layers with no blend mode (default 'over'), the compositor can render directly to the output
surface without an intermediate FBO — an important optimisation for flat UI hierarchies.

### 10.4 Premultiplied Alpha Throughout

Premultiplied alpha must be maintained end-to-end. Common pitfalls:
- Loading PNG images: PNG stores straight alpha; convert at upload time by multiplying RGB by alpha
  in a compute shader before storing in the `VkImage`.
- Gradient colour stops: the GLSL gradient shader (§11) must premultiply each stop's colour.
- Filter effects: feGaussianBlur must operate on premultiplied data (otherwise alpha-weighted
  averaging produces incorrect halos around semi-transparent shapes).

---

## 11. 2D Gradient and Pattern GPU Rendering

### 11.1 Linear Gradient

```glsl
// linear_gradient.frag
uniform vec2  p0, p1;          // gradient start/end in user space
uniform sampler1D colorRamp;   // pre-sampled, premultiplied, CLAMP_TO_EDGE

void main() {
    vec2  d   = p1 - p0;
    float t   = dot(fragCoord - p0, d) / dot(d, d);
    fragColor = texture(colorRamp, clamp(t, 0.0, 1.0));
}
```

For SVG `spreadMethod="repeat"`, use `fract(t)`. For `reflect`, use `1.0 - abs(fract(t/2.0)*2.0 - 1.0)`.

### 11.2 Radial Gradient

The full two-focal-point radial gradient formula from SVG §8.6 covers the case where c₀ ≠ c₁ or r₀ ≠ r₁. The standard CSS radial gradient simplifies to c₀ = c₁ = center, r₀ = 0:

```glsl
// simple radial gradient (CSS form: c0 == c1, r0 == 0)
uniform vec2  center;
uniform float radius;
uniform sampler1D colorRamp;

void main() {
    float t = length(fragCoord - center) / radius;
    fragColor = texture(colorRamp, clamp(t, 0.0, 1.0));
}
```

### 11.3 Conic / Sweep Gradient

```glsl
// conic_gradient.frag
uniform vec2  center;
uniform float startAngle;  // radians

void main() {
    float angle = atan(fragCoord.y - center.y, fragCoord.x - center.x);
    float t     = fract((angle - startAngle) / (2.0 * 3.14159265));
    fragColor   = texture(colorRamp, t);
}
```

`atan(y, x)` maps to `[-π, π]`; the `fract(…/(2π))` normalises to `[0, 1)`.

### 11.4 Mesh Gradient — Coons Patch

CSS is exploring mesh gradients (formerly "SVG mesh gradient" draft). A Coons patch is defined by
four boundary curves and a bilinear colour interpolation. For the simplest quad-mesh variant (flat
patches with corner colours c₀₀, c₁₀, c₀₁, c₁₁), the GPU evaluates bilinear interpolation per
fragment:

```glsl
// mesh_gradient.frag — single Coons patch (bilinear)
uniform vec4 c00, c10, c01, c11;  // premultiplied corner colours

void main() {
    vec2 uv = (fragCoord - patchOrigin) / patchSize;
    vec4 cx0 = mix(c00, c10, uv.x);
    vec4 cx1 = mix(c01, c11, uv.x);
    fragColor = mix(cx0, cx1, uv.y);
}
```

For bicubic Coons patches (with curved boundary Béziers), (u,v) are computed via Newton iteration on
the patch's parametrisation — feasible in a fragment shader for small patch counts.
[Source: Kovacs & Mitchell "GPU-Based Rendering of Sparse Coons Patches", GPC 2007]

### 11.5 Tiled Pattern Rendering

A repeating tile pattern is rendered using GPU texture wrap modes:

```c
VkSamplerCreateInfo sci = {
    .addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT,
    .addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT,
};
```

In the fragment shader, apply the inverse of the pattern's affine transform to fragCoord before
sampling, producing the correct pattern orientation and scale:

```glsl
vec2 patternUV = (patternInvTransform * vec3(fragCoord, 1.0)).xy / patternSize;
fragColor = texture(patternAtlas, patternUV);
```

### 11.6 Procedural Noise Patterns

2D Simplex noise in a fragment shader generates procedural organic patterns without texture memory:

```glsl
// Compact 2D Simplex noise (Gustavson 2005)
float snoise2D(vec2 v) {
    const vec4 C = vec4(0.211324865, 0.366025404, -0.577350269, 0.024390244);
    vec2  i  = floor(v + dot(v, C.yy));
    vec2  x0 = v - i + dot(i, C.xx);
    vec2  i1 = (x0.x > x0.y) ? vec2(1.0, 0.0) : vec2(0.0, 1.0);
    vec4  x12 = x0.xyxy + C.xxzz - vec4(i1, 1.0, 1.0);
    i = mod289(i);
    vec3  p  = permute(permute(i.y + vec3(0.0, i1.y, 1.0)) + i.x + vec3(0.0, i1.x, 1.0));
    vec3  m  = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy), dot(x12.zw,x12.zw)), 0.0);
    m = m*m; m = m*m;
    vec3  x  = 2.0 * fract(p * C.www) - 1.0;
    vec3  h  = abs(x) - 0.5;
    vec3  ox = floor(x + 0.5);
    vec3  a0 = x - ox;
    m *= 1.79284291 - 0.85373473 * (a0*a0 + h*h);
    vec3  g  = vec3(a0.x*x0.x+h.x*x0.y, a0.yz*x12.xz+h.yz*x12.yw);
    return 130.0 * dot(m, g);
}
```

[Source: Gustavson, "Simplex Noise Demystified," 2005,
http://staffwww.itn.liu.se/~stegu/simplexnoise/simplexnoise.pdf]

---

## 12. Integration with the Linux Graphics Stack

### 12.1 Skia GPU Backend — Ganesh and Graphite

**Ganesh (legacy GPU backend)**: wraps OpenGL and Vulkan via `GrContext`. Path rendering uses
CPU-side tessellation into triangle strips, uploaded as vertex buffers, then rendered using a stencil
pass followed by a cover pass:

```cpp
// Vulkan backend
GrVkBackendContext vkCtx = { instance, physDevice, device, queue, queueFamilyIndex };
sk_sp<GrDirectContext> grCtx = GrDirectContext::MakeVulkan(vkCtx);

// Wrap a VkImage as SkSurface
GrVkImageInfo imgInfo = { vkImage, VK_FORMAT_R8G8B8A8_UNORM, ... };
GrBackendRenderTarget rt(width, height, /* samples */ 1, imgInfo);
sk_sp<SkSurface> surface = SkSurface::MakeFromBackendRenderTarget(
    grCtx.get(), rt, kTopLeft_GrSurfaceOrigin, kRGBA_8888_SkColorType, nullptr, nullptr);
SkCanvas* canvas = surface->getCanvas();
```

**Graphite (new GPU backend)**: replaces Ganesh in Chrome (Linux Vulkan) with a more modern design:
- `PathAtlas` (`src/gpu/graphite/PathAtlas.h`): CPU rasterises small paths into an atlas, uploads
  as a `VkImage`. Avoids GPU vertex buffer overhead for small UI elements.
- `ComputePathAtlas` (`src/gpu/graphite/ComputePathAtlas.h`): GPU compute path rasterisation for
  larger or dynamic paths.
- `sparse_strips/`: sparse rendering optimisation that skips fully-transparent tiles.
- Vulkan backend: `src/gpu/graphite/vk/VulkanGraphiteContext.cpp`.

[Source: Skia Graphite source, https://skia.googlesource.com/skia/+/refs/heads/main/src/gpu/graphite/]

### 12.2 Cairo OpenGL Backend Limitations

Cairo was designed around a CPU rasterisation model (pixman). The `cairo-gl` backend
(`cairo-gl-surface-create()`) uploads completed pixman-rasterised scanlines to OpenGL textures as
a blit. True GPU-native 2D path rendering was never completed in Cairo's design — there is no shader
that evaluates path coverage on the GPU. The `cairo-gl` backend is not built by default in modern
distributions and is effectively unmaintained. Applications requiring GPU 2D rendering on Linux
should use Vello, Skia Graphite, or WebRender instead.

### 12.3 Firefox WebRender

WebRender is a GPU-accelerated 2D renderer written in Rust for Firefox. Its architecture on Linux:
[Source: WebRender source, https://github.com/servo/webrender]

**Brush shaders**: specialised GLSL shaders for each content primitive type — `image`, `solid`,
`linear_gradient`, `radial_gradient`, `border`, `text_run`. Each shader is optimised for its
primitive, avoiding general-purpose branching. Path rendering uses pre-rasterised glyph atlases
(via FreeType) for text and the brush pipeline for shapes.

**Picture caching**: the compositor divides the page into tiles (typically 512×512). Static tiles are
cached as GPU textures and reused across frames. Only tiles with changed content are re-rasterised,
reducing GPU work for static pages to near zero.

**Clip chain GPU evaluation**: nested CSS clip regions (`clip-path`, `overflow:hidden`) are resolved
via stencil operations and clip masks, combined in a clip chain evaluated in the GPU pipeline.

WebRender currently uses OpenGL (via EGL on Linux); a Vulkan migration is ongoing but not yet default
as of mid-2026.

### 12.4 Zero-Copy DMA-BUF Export for Wayland

GPU-rendered 2D surfaces can be exported from Vulkan to the Wayland compositor via DMA-BUF for
zero-copy display — no CPU readback or texture copy across processes:

```c
// 1. Create VkImage with external memory export
VkExternalMemoryImageCreateInfo externalCI = {
    .sType       = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO,
    .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
// ... create VkImage with &externalCI in pNext ...

// 2. Export the fd
VkMemoryGetFdInfoKHR fdInfo = {
    .sType      = VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR,
    .memory     = imageMemory,
    .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT,
};
int dmaBufFd;
vkGetMemoryFdKHR(device, &fdInfo, &dmaBufFd);

// 3. Import into wl_buffer via zwp_linux_dmabuf_v1
struct zwp_linux_buffer_params_v1 *params =
    zwp_linux_dmabuf_v1_create_params(dmabuf);
zwp_linux_buffer_params_v1_add(params, dmaBufFd, 0, offset, stride, modHi, modLo);
struct wl_buffer *wlBuf =
    zwp_linux_buffer_params_v1_create_immed(params, width, height, format, 0);
wl_surface_attach(surface, wlBuf, 0, 0);
wl_surface_commit(surface);
```

This pattern lets Vello's GPU-rendered `VkImage` appear directly in the Wayland compositor
(KWin, Mutter, sway) as a scanout buffer, achieving zero-copy rendering from GPU 2D path engine to
display, with the compositor performing its own GPU-side composition (Ch20).
[Source: zwp_linux_dmabuf_v1 protocol,
https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/linux-dmabuf/linux-dmabuf-unstable-v1.xml]

### 12.5 Chrome / Skia Graphite on Linux Vulkan

Chrome on Linux uses Skia Graphite as its GPU-side 2D renderer (since Chrome 128 on Vulkan-capable
devices). The GPU 2D path from a CSS `border-radius` or SVG element to the display is:
1. Blink paint layer → SkCanvas drawing commands.
2. Skia Graphite records commands into a `skgpu::graphite::Recording`.
3. Graphite submits GPU work to Vulkan via `skgpu::graphite::Context::submit()`.
4. The rendered `VkImage` is shared with the Chrome GPU process compositor (Viz).
5. Viz composites into the final swap-chain frame and presents via Vulkan WSI / EGL.

The `ComputePathAtlas` is triggered for paths that fall below a size threshold (~256px²); larger
paths use tessellation. Text glyphs use Skia's glyph atlas with MSDF rendering for display-sized
text and bitmap glyphs for small sizes.
[Source: Skia Graphite, https://skia.org/docs/dev/design/graphite/;
Chromium GPU process, https://chromium.googlesource.com/chromium/src/+/main/docs/gpu/overview.md]

---

## Integrations

- **Ch105 — Font Rendering (FreeType2, HarfBuzz, and the Text Pipeline)**: the MSDF atlas (§5) and
  COLRv1 rendering (§9) sit above the font shaping and layout layer. HarfBuzz shapes runs of
  glyphs; FreeType provides outlines; Ch235 covers how those outlines are turned into GPU distance
  fields and colour paint graphs. Ch105 covers the API layer (FT_Outline, hb_buffer_t) that feeds
  the GPU algorithms here.

- **Ch20 — Wayland Protocol Fundamentals**: the DMA-BUF zero-copy export (§12.4) feeds directly
  into the `zwp_linux_dmabuf_v1` Wayland protocol. Ch20 covers the wl_surface, wl_buffer, and
  subsurface model that the GPU 2D renderer's output plugs into.

- **Ch208 — GPU Geometry Algorithms (§5)**: provides the compact introduction to Loop–Blinn Bézier
  rendering, MSDF fonts, stroke expansion, and stencil-then-cover. Ch235 is the deep treatment of
  all those techniques plus the full Vello pipeline, COLRv1, compositing operators, and Linux stack
  integration.

- **Ch222 — Computational Geometry Algorithms on GPU**: covers 2D polygon Boolean operations,
  Voronoi/JFA for 3D (§7), convex hull, and point-location — the computational geometry primitives
  that feed GPU path rendering. Ch235 uses JFA for 2D SDF baking (§4); Ch222 covers the broader
  Voronoi and Delaunay machinery.

- **Ch207 — Shader Algorithm Catalog (VFX, Post-Processing, and GPU Compute)**: covers post-processing
  that operates on the output of GPU 2D rendering: bloom, DoF, TAA, DLSS/FSR upscaling, and
  MSDF text rendering as a texturing technique (Category X). Ch235's rendered output passes through
  the post-processing pipeline documented in Ch207.
