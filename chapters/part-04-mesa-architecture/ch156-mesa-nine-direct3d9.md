# Chapter 156: Mesa Nine: Direct3D 9 State Tracker

**Target audiences**: Wine and Proton developers who need to understand why Nine outperformed the OpenGL translation path; Mesa contributors working on the Gallium interface; and systems engineers studying D3D9 game compatibility on Linux. As of Mesa 25.2 (August 2025) the Nine frontend has been removed from Mesa mainline — this chapter documents the architecture at removal, which remains a valuable reference for the ~25 KLOC design.

---

## Table of Contents

1. [Introduction](#introduction)
   - [1.1 What is Direct3D 9?](#11-what-is-direct3d-9)
   - [1.2 What is Gallium3D?](#12-what-is-gallium3d)
   - [1.3 What is Wine?](#13-what-is-wine)
2. [The D3D9 on Linux Problem](#the-d3d9-on-linux-problem)
3. [Mesa Nine Architecture](#mesa-nine-architecture)
4. [Gallium State Tracker Interface](#gallium-state-tracker-interface)
5. [nine_state.c Internals: Render-State Dirty Tracking](#nine_statec-internals-render-state-dirty-tracking)
6. [Stateblocks: IDirect3DStateBlock9](#stateblocks-idirect3dstateblock9)
7. [nine_context.c: The Command Queue](#nine_contextc-the-command-queue)
8. [Shader Translation Pipeline](#shader-translation-pipeline)
9. [Constant Buffer Handling and Performance](#constant-buffer-handling-and-performance)
10. [Fixed-Function Pipeline Emulation](#fixed-function-pipeline-emulation)
11. [SwapChain9 and Present()](#swapchain9-and-present)
12. [DRI3 and DMA-BUF Texture Path](#dri3-and-dma-buf-texture-path)
13. [Enabling Nine in Wine](#enabling-nine-in-wine)
14. [Performance Comparison: Nine vs WineD3D vs DXVK](#performance-comparison-nine-vs-wined3d-vs-dxvk)
15. [Deprecation and Removal (Mesa 25.1–25.2)](#deprecation-and-removal-mesa-251252)
16. [Integrations](#integrations)

---

## Introduction

**Mesa Nine** (also called "Gallium Nine") is a Direct3D 9 state tracker that was built directly into Mesa from approximately 2013 through Mesa 25.1 (early 2025). Instead of translating D3D9 API calls to OpenGL (as Wine's WineD3D does), Nine translated D3D9 calls directly to Gallium3D `pipe_context` calls, eliminating the entire OpenGL layer and dramatically improving performance for D3D9 games running under Wine.

Nine was available in Mesa's `src/gallium/frontends/nine/` and required a Gallium driver:

- **RadeonSI** — AMD
- **Iris** — Intel
- **Nouveau** — Nvidia
- **LLVMpipe** — software rendering

It did not work with Zink (which sits above a Vulkan driver) or with native Vulkan drivers such as RADV or ANV.

The concept was pioneered by Axel Davy and others; it became a viable Wine integration via the `d3dadapter9` native module loaded by Wine's D3D9 stub. The project is now archived following removal in Mesa 25.2, but the design remains instructive as a near-minimal Gallium frontend: approximately 25 KLOC that cleanly maps a COM-based API to a pipe-level state machine.

Sources: [Mesa Nine source (last tag with Nine: mesa-25.0)](https://gitlab.freedesktop.org/mesa/mesa/-/tree/mesa-25.0/src/gallium/frontends/nine) | [wine-nine-standalone](https://github.com/iXit/wine-nine-standalone) | [Mesa Gallium Nine docs](https://idr.pages.freedesktop.org/mesa/gallium-nine.html)

### 1.1 What is Direct3D 9?

Direct3D 9 (D3D9) is the graphics API introduced with Microsoft DirectX 9, released in 2002. It defines a COM-based interface through which Windows applications submit draw calls, manage GPU resources (textures, vertex buffers, index buffers, render targets), and control fixed-function and programmable pipeline state. The API's state model is built around integer-keyed render-state tokens (the `D3DRS_*` enum), texture-stage states, and sampler states, all set on a central `IDirect3DDevice9` object. Shader programs are expressed as Direct3D Shader Model 1 through 3 (SM1–SM3) bytecode, a binary format that predates GLSL text compilation and targets the vertex and pixel shader units of GPU hardware from that era.

D3D9 titles remained commercially dominant through the mid-2010s: large portions of the PC gaming catalogue were built on it and continued shipping D3D9 executables long after DirectX 11 and 12 superseded the API. On Linux, running this catalogue requires either translating the D3D9 surface to an available native graphics API or implementing D3D9 directly atop the native Linux driver stack — the problem that Mesa Nine addressed. The D3D9 specification is documented in the Windows SDK; the SM1–SM3 bytecode format is also described in the Wine project source and in Valve's DXVK shader-compiler documentation.

### 1.2 What is Gallium3D?

Gallium3D is Mesa's internal hardware-abstraction layer. It sits between frontend state trackers — which implement APIs such as OpenGL, OpenCL, or, in Nine's case, Direct3D 9 — and backend pipe drivers such as RadeonSI for AMD GCN/RDNA hardware, Iris for Intel Gen9+, or Nouveau for NVIDIA. The contract between a frontend and a driver is the `pipe_context` interface, a C vtable of function pointers covering resource allocation, draw commands, state binding, and synchronisation.

Key Gallium abstractions relevant to Nine include `pipe_screen`, which represents a GPU device and its capabilities and is used for resource allocation and format-support queries; `pipe_context`, the command-submission context whose methods Nine calls directly (`set_constant_buffer`, `bind_vs_state`, `draw_vbo`); CSO (Compiled State Objects), pre-compiled rasterizer, depth-stencil, and blend state objects that avoid redundant validation on every draw call; and `pipe_resource`, the generic buffer and texture type backed by driver-managed GPU memory. Gallium3D was introduced into Mesa around 2008 and had become the standard path for all hardware-accelerated Mesa drivers by the time Nine was developed, making it the natural target for a high-performance D3D9 implementation. [Source: Mesa Gallium documentation](https://docs.mesa3d.org/gallium/index.html)

### 1.3 What is Wine?

Wine is an open-source compatibility layer that enables Windows application binaries to run on Linux and other POSIX systems without a Windows kernel or virtual machine. It intercepts Windows API calls and translates them to equivalent POSIX or Linux-specific system calls, re-implementing the Win32, COM, and DirectX API surfaces in user space.

In the context of graphics, Wine includes WineD3D, its own implementation of the Direct3D API family (D3D8 through D3D11). WineD3D historically translated all D3D calls to OpenGL, routing through Mesa's OpenGL state tracker and then through Gallium3D to the hardware driver — an inherently multi-layer path with significant overhead. Wine also includes a minimal D3D9 stub DLL (`d3d9.dll`) whose behaviour can be overridden by a native implementation; Mesa Nine exploited this mechanism by providing `d3d9.so` via the `d3dadapter9` Wine native module as a drop-in replacement that called Gallium directly. Wine is the primary deployment vehicle for Mesa Nine. Proton, Valve's fork of Wine used in Steam Play, later adopted DXVK — a Vulkan-based D3D9/D3D11 translator — as its default path, which contributed to the decision to deprecate Mesa Nine. [Source: wine-nine-standalone](https://github.com/iXit/wine-nine-standalone)

---

## The D3D9 on Linux Problem

### WineD3D Translation Chain

Standard Wine D3D9 game path before Nine:

```
D3D9 game → WineD3D (Wine's D3D implementation)
  → translate every D3D9 call to OpenGL
    → Mesa OpenGL frontend (GLX/EGL)
      → Gallium3D state tracker for OpenGL
        → pipe driver (RadeonSI/Iris/etc.)
```

The OpenGL translation imposes several layers of overhead:

- **State impedance mismatch**: D3D9 and OpenGL have fundamentally different state machines. D3D9 uses integer render-state tokens (e.g. `D3DRS_ZENABLE`) whereas OpenGL uses boolean capability bits and immediate-mode state calls. WineD3D must keep both state machines in sync, doing double bookkeeping.
- **Double validation**: As Axel Davy explained in a 2015 interview, Wine must validate D3D9 API calls, then the OpenGL state tracker must validate the resulting OpenGL calls. Gallium is a low-level trusted API, so Nine needed only the D3D9-level checks. [Source](https://www.gamingonlinux.com/2015/02/an-interview-with-gallium-nine-project-developer-axel-davy/)
- **Shader translation chain**: D3D9 SM2/SM3 bytecode → GLSL → NIR → GPU ISA requires three separate compilation passes; Nine shortened this to SM bytecode → TGSI/NIR → ISA.
- **Shader compilation timing**: Wine/WineD3D deferred shader compilation to draw time, causing in-game stuttering. Nine compiled shaders upon `CreateVertexShader`/`CreatePixelShader` calls, eliminating the stutter. [Source](https://www.gamingonlinux.com/2015/02/an-interview-with-gallium-nine-project-developer-axel-davy/)

### Nine's Shortcut

```
D3D9 game → Nine state tracker (Mesa)
  → Gallium3D pipe calls
    → pipe driver (RadeonSI/Iris/etc.)
```

Nine speaks Gallium3D natively — the same internal language that RadeonSI, Iris, and other Mesa drivers expose. The OpenGL frontend is bypassed entirely.

---

## Mesa Nine Architecture

### Source Tree

Nine's source lived in `mesa/src/gallium/frontends/nine/` and consisted of approximately 40 C source files:

```
mesa/src/gallium/frontends/nine/
├── nine_state.c          # D3D9 state machine → pipe_context calls + dirty tracking
├── nine_state.h          # NINE_STATE_* bitmask defines, struct nine_state
├── nine_context.c        # Command queue (worker thread, ring buffer dispatch)
├── nine_queue.c          # Ring buffer: nine_cmdbuf / nine_queue_pool structures
├── nine_shader.c         # D3D9 SM1–SM3 bytecode → TGSI/NIR translation
├── nine_pipe.c           # D3D9 → Gallium type conversions (formats, compares, etc.)
├── nine_ff.c             # Fixed-function pipeline emulation (T&L, fog, texture stages)
├── nine_buffer_upload.c  # Sub-allocated vertex/index buffer upload pool
├── device9.c             # IDirect3DDevice9 COM implementation
├── swapchain9.c          # IDirect3DSwapChain9 / Present() logic
├── stateblock9.c         # IDirect3DStateBlock9 Capture/Apply
├── texture9.c            # IDirect3DTexture9
├── surface9.c            # IDirect3DSurface9
├── vertexbuffer9.c       # IDirect3DVertexBuffer9
├── indexbuffer9.c        # IDirect3DIndexBuffer9
├── vertexshader9.c       # IDirect3DVertexShader9 (holds compiled CSO)
├── pixelshader9.c        # IDirect3DPixelShader9
├── vertexdeclaration9.c  # IDirect3DVertexDeclaration9 → pipe_vertex_element[]
├── query9.c              # IDirect3DQuery9 (occlusion, timestamp queries)
├── nine_helpers.c/h      # Reference counting, COM macros, range pool
└── nine_debug.c          # NINE_DEBUG env-var driven trace output
```

[Source: Mesa Nine directory listing, mesa-25.0](https://gitlab.freedesktop.org/mesa/mesa/-/tree/mesa-25.0/src/gallium/frontends/nine)

### Key Objects

Nine implements the full `IDirect3DDevice9` COM interface. The central object is `NineDevice9`:

```c
/* nine/device9.c (mesa-25.0) */
struct NineDevice9 {
    struct NineUnknown base;        /* COM reference counting (AddRef/Release) */
    struct pipe_context *pipe;      /* Gallium pipe context — main thread */
    struct pipe_context *pipe_secondary; /* worker thread's pipe context */
    struct pipe_screen *screen;     /* Gallium screen for resource allocation */

    /* Current D3D9 state visible to the application thread: */
    struct nine_state state;

    /* Caches: */
    struct cso_context *cso;        /* Gallium CSO cache (compiled state objects) */
    struct util_hash_table *vs_cache;
    struct util_hash_table *ps_cache;

    /* Swap chain (owns the DRI present context): */
    struct NineSwapChain9 **swapchains;
    unsigned nswapchains;

    /* Command queue to worker thread: */
    struct nine_queue_pool *ctx_queue;
};
```

The `nine_state` struct embedded in `NineDevice9` holds the entire current D3D9 device state: render state tokens, textures, vertex streams, index buffer, shaders, viewport, scissor, and per-constant dirty-range lists.

---

## Gallium State Tracker Interface

### How Nine Calls Gallium

Nine is one of Mesa's Gallium frontends alongside the OpenGL state tracker and Clover (OpenCL). It calls `pipe_context` methods directly:

```c
/* nine/nine_state.c: translating D3D9 draw state to Gallium */
static void
nine_update_rasterizer(struct NineDevice9 *device)
{
    struct pipe_rasterizer_state rs;
    memset(&rs, 0, sizeof(rs));

    /* D3DRS_FILLMODE → fill_front / fill_back */
    rs.fill_front = rs.fill_back =
        (device->state.rs[D3DRS_FILLMODE] == D3DFILL_WIREFRAME)
            ? PIPE_POLYGON_MODE_LINE : PIPE_POLYGON_MODE_FILL;

    /* D3DRS_CULLMODE → cull_face */
    rs.cull_face =
        (device->state.rs[D3DRS_CULLMODE] == D3DCULL_CW)  ? PIPE_FACE_FRONT :
        (device->state.rs[D3DRS_CULLMODE] == D3DCULL_CCW) ? PIPE_FACE_BACK  :
                                                              PIPE_FACE_NONE;

    rs.scissor = !!(device->state.rs[D3DRS_SCISSORTESTENABLE]);
    rs.multisample = !!(device->state.rs[D3DRS_MULTISAMPLEANTIALIAS]);

    /* D3D9 uses [0,1] depth range; Gallium matches (unlike OpenGL's [-1,1]): */
    rs.clip_halfz = 1;
    rs.depth_clip_near = rs.depth_clip_far = 1;

    /* Upload the new CSO: */
    cso_set_rasterizer(device->cso, &rs);
}
```

### D3D9 → Gallium Format Mapping

Nine's `nine_pipe.c` contains the complete D3D9 surface format to `pipe_format` mapping. A representative excerpt:

```c
/* nine/nine_pipe.c (mesa-25.0) */
enum pipe_format
d3d9_to_pipe_format_checked(struct pipe_screen *screen,
    D3DFORMAT format, enum pipe_texture_target target,
    unsigned sample_count, unsigned bindings, boolean srgb)
{
    enum pipe_format fmt;
    switch (format) {
    case D3DFMT_A8R8G8B8: fmt = PIPE_FORMAT_B8G8R8A8_UNORM; break;
    case D3DFMT_X8R8G8B8: fmt = PIPE_FORMAT_B8G8R8X8_UNORM; break;
    case D3DFMT_R5G6B5:   fmt = PIPE_FORMAT_B5G6R5_UNORM;   break;
    case D3DFMT_A16B16G16R16F: fmt = PIPE_FORMAT_R16G16B16A16_FLOAT; break;
    case D3DFMT_R32F:     fmt = PIPE_FORMAT_R32_FLOAT;       break;
    case D3DFMT_DXT1:     fmt = PIPE_FORMAT_DXT1_RGB;        break;
    case D3DFMT_DXT3:     fmt = PIPE_FORMAT_DXT3_RGBA;       break;
    case D3DFMT_DXT5:     fmt = PIPE_FORMAT_DXT5_RGBA;       break;
    case D3DFMT_D24S8:    fmt = PIPE_FORMAT_Z24_UNORM_S8_UINT; break;
    case D3DFMT_D16:      fmt = PIPE_FORMAT_Z16_UNORM;       break;
    default:
        return PIPE_FORMAT_NONE;
    }
    /* Check that the driver supports this format for the requested bindings: */
    if (!screen->is_format_supported(screen, fmt, target,
                                     sample_count, sample_count, bindings))
        return PIPE_FORMAT_NONE;
    return fmt;
}
```

Note that D3D9 uses BGRA component ordering (inherited from GDI), whereas Vulkan and most modern APIs default to RGBA. The mapping table in `nine_pipe.c` preserves the BGRA component ordering whenever a native Gallium format is available.

---

## nine_state.c Internals: Render-State Dirty Tracking

### The NINE_STATE_* Bitmask

Nine avoids redundant Gallium calls by tracking which categories of D3D9 state have changed since the last draw. The dirty flags are grouped into coarse categories:

```c
/* nine/nine_state.h (mesa-25.0) — representative subset */
#define NINE_STATE_FB           (1 << 0)   /* render target / depth buffer */
#define NINE_STATE_VIEWPORT     (1 << 1)
#define NINE_STATE_SCISSOR      (1 << 2)
#define NINE_STATE_RASTERIZER   (1 << 3)   /* D3DRS_FILLMODE, CULLMODE, etc. */
#define NINE_STATE_BLEND        (1 << 4)   /* D3DRS_ALPHABLENDENABLE, blendop */
#define NINE_STATE_DSA          (1 << 5)   /* depth/stencil/alpha test */
#define NINE_STATE_VS           (1 << 6)   /* vertex shader object */
#define NINE_STATE_PS           (1 << 7)   /* pixel shader object */
#define NINE_STATE_TEXTURE      (1 << 8)   /* texture bindings */
#define NINE_STATE_SAMPLER      (1 << 9)   /* sampler state (filter, wrap) */
#define NINE_STATE_VDECL        (1 << 10)  /* vertex declaration */
#define NINE_STATE_IDXBUF       (1 << 11)  /* index buffer */
#define NINE_STATE_STREAMFREQ   (1 << 12)  /* stream source frequency (instancing) */
#define NINE_STATE_VS_CONST     (1 << 13)  /* VS float/int/bool constants dirty */
#define NINE_STATE_PS_CONST     (1 << 14)  /* PS float/int/bool constants dirty */
#define NINE_STATE_BLEND_COLOR  (1 << 15)
#define NINE_STATE_STENCIL_REF  (1 << 16)
#define NINE_STATE_SAMPLE_MASK  (1 << 17)
#define NINE_STATE_MULTISAMPLE  (1 << 18)
#define NINE_STATE_SWVP         (1 << 19)  /* software vertex processing mode */

/* Composite groups for common update patterns: */
#define NINE_STATE_FREQUENT \
   (NINE_STATE_RASTERIZER | NINE_STATE_TEXTURE | NINE_STATE_SAMPLER | \
    NINE_STATE_VS_CONST    | NINE_STATE_PS_CONST | NINE_STATE_MULTISAMPLE)

#define NINE_STATE_COMMON \
   (NINE_STATE_FB | NINE_STATE_BLEND | NINE_STATE_DSA | NINE_STATE_VIEWPORT | \
    NINE_STATE_VDECL | NINE_STATE_IDXBUF | NINE_STATE_STREAMFREQ)
```

The `struct nine_state` embeds a `changed` sub-structure that tracks dirty bits at both the group level and the fine-grained render-state and constant-register level:

```c
/* nine/nine_state.h (mesa-25.0) */
struct nine_state {
    /* Current render state token array (D3DRS_* indexed): */
    DWORD rs[NINED3DRS_COUNT];

    /* Sampler state for up to NINE_MAX_SAMPLERS samplers: */
    DWORD ss[NINE_MAX_SAMPLERS][D3DSAMP_LAST + 1];

    /* Shader constant registers: */
    float   *vs_const_f;   /* up to 256 float4 registers */
    int      vs_const_i[NINE_MAX_CONST_I][4];
    BOOL     vs_const_b[NINE_MAX_CONST_B];
    float   *ps_const_f;   /* up to 224 float4 registers */
    int      ps_const_i[NINE_MAX_CONST_I][4];
    BOOL     ps_const_b[NINE_MAX_CONST_B];

    /* Dirty-tracking: */
    struct {
        uint32_t group;   /* NINE_STATE_* bitmask for this frame */

        /* Per-render-state dirty bits (one bit per D3DRS_* token): */
        uint32_t rs[(NINED3DRS_COUNT + 31) / 32];

        /* Per-sampler dirty bits: */
        uint16_t sampler[NINE_MAX_SAMPLERS];

        /* Dirty ranges for shader float constants
         * (linked list of nine_range: [bgn, end) register spans): */
        struct nine_range *vs_const_f;
        struct nine_range *ps_const_f;
        struct nine_range *vs_const_i;
        uint16_t ps_const_i;
        struct nine_range *vs_const_b;
        uint16_t ps_const_b;

        /* User clip planes: */
        uint8_t ucp;
    } changed;

    /* Compiled Gallium CSO handles (rebuilt when dirty): */
    void *cso_rasterizer;
    void *cso_blend;
    void *cso_zsa;   /* depth/stencil/alpha */
};
```

### Dispatch on Dirty Flags

Before every draw call Nine calls `nine_update_state()`, which uses the group bitmask to skip unchanged categories entirely:

```c
/* nine/nine_state.c (conceptual reconstruction from upstream) */
void
nine_update_state(struct NineDevice9 *device)
{
    struct nine_state *state = &device->state;
    uint32_t dirty = state->changed.group;

    if (dirty & NINE_STATE_FB)
        nine_update_framebuffer(device);

    if (dirty & NINE_STATE_VIEWPORT)
        nine_context_set_viewport(device, &state->viewport);

    if (dirty & NINE_STATE_SCISSOR)
        nine_context_set_scissor(device, &state->scissor);

    if (dirty & NINE_STATE_RASTERIZER)
        nine_update_rasterizer(device);

    if (dirty & NINE_STATE_BLEND)
        nine_update_blend(device);

    if (dirty & NINE_STATE_DSA)
        nine_update_dsa(device);

    if (dirty & NINE_STATE_VS_CONST)
        nine_update_vs_constants(device);

    if (dirty & NINE_STATE_PS_CONST)
        nine_update_ps_constants(device);

    if (dirty & NINE_STATE_TEXTURE)
        nine_update_textures_and_samplers(device);

    /* Clear dirty flags after successful update: */
    state->changed.group = 0;
}
```

### Blend and Depth/Stencil/Alpha Mapping

The blend state translation maps D3D9's flat render-state tokens onto `pipe_blend_state`'s per-render-target structure:

```c
/* nine/nine_state.c: nine_update_blend() */
static void
nine_update_blend(struct NineDevice9 *device)
{
    const DWORD *rs = device->state.rs;
    struct pipe_blend_state blend;
    memset(&blend, 0, sizeof(blend));

    blend.rt[0].blend_enable = !!rs[D3DRS_ALPHABLENDENABLE];
    if (blend.rt[0].blend_enable) {
        blend.rt[0].rgb_src_factor  = d3dblend_to_pipe(rs[D3DRS_SRCBLEND]);
        blend.rt[0].rgb_dst_factor  = d3dblend_to_pipe(rs[D3DRS_DESTBLEND]);
        blend.rt[0].rgb_func        = d3dblendop_to_pipe(rs[D3DRS_BLENDOP]);
        blend.rt[0].alpha_src_factor = d3dblend_alpha_to_pipe(rs[D3DRS_SRCBLENDALPHA]);
        blend.rt[0].alpha_dst_factor = d3dblend_alpha_to_pipe(rs[D3DRS_DESTBLENDALPHA]);
        blend.rt[0].alpha_func       = d3dblendop_to_pipe(rs[D3DRS_BLENDOPALPHA]);
    }
    blend.rt[0].colormask = rs[D3DRS_COLORWRITEENABLE] & 0xF;

    /* Independent blend (MRT) for render targets 1–3: */
    if (rs[D3DRS_SEPARATEALPHABLENDENABLE]) {
        blend.independent_blend_enable = TRUE;
        /* ... populate blend.rt[1..3] from D3DRS_COLORWRITEENABLE1..3 */
    }

    cso_set_blend(device->cso, &blend);
}
```

The depth/stencil/alpha mapping follows the same pattern. `pipe_depth_stencil_alpha_state` covers both the D3D9 `D3DRS_ZENABLE`/`ZFUNC`/`STENCILENABLE` group and the alpha-test path (`D3DRS_ALPHATESTENABLE`, `D3DRS_ALPHAFUNC`, `D3DRS_ALPHAREF`), which is a fixed-function feature absent from modern APIs.

```c
/* nine/nine_state.c: nine_update_dsa() — abridged */
static void
nine_update_dsa(struct NineDevice9 *device)
{
    const DWORD *rs = device->state.rs;
    struct pipe_depth_stencil_alpha_state dsa;
    memset(&dsa, 0, sizeof(dsa));

    dsa.depth_enabled   = !!rs[D3DRS_ZENABLE];
    dsa.depth_writemask = !!rs[D3DRS_ZWRITEENABLE];
    dsa.depth_func      = d3dcmp_to_pipe(rs[D3DRS_ZFUNC]);

    dsa.stencil[0].enabled    = !!rs[D3DRS_STENCILENABLE];
    dsa.stencil[0].func       = d3dcmp_to_pipe(rs[D3DRS_STENCILFUNC]);
    dsa.stencil[0].fail_op    = d3dstencilop_to_pipe(rs[D3DRS_STENCILFAIL]);
    dsa.stencil[0].zfail_op   = d3dstencilop_to_pipe(rs[D3DRS_STENCILZFAIL]);
    dsa.stencil[0].zpass_op   = d3dstencilop_to_pipe(rs[D3DRS_STENCILPASS]);
    dsa.stencil[0].valuemask  = rs[D3DRS_STENCILMASK];
    dsa.stencil[0].writemask  = rs[D3DRS_STENCILWRITEMASK];

    /* Two-sided stencil (SM2+): */
    if (rs[D3DRS_TWOSIDEDSTENCILMODE]) {
        dsa.stencil[1].enabled  = TRUE;
        /* ... CCW face operations from D3DRS_CCW_* tokens ... */
    }

    /* Alpha test — D3D9 fixed function, emulated via shader in hardware: */
    dsa.alpha_enabled = !!rs[D3DRS_ALPHATESTENABLE];
    dsa.alpha_func    = d3dcmp_to_pipe(rs[D3DRS_ALPHAFUNC]);
    dsa.alpha_ref_value = (float)rs[D3DRS_ALPHAREF] / 255.0f;

    cso_set_depth_stencil_alpha(device->cso, &dsa);
}
```

---

## Stateblocks: IDirect3DStateBlock9

### What a Stateblock Does

A D3D9 stateblock (`IDirect3DStateBlock9`) is a snapshot mechanism: the application can capture a slice of device state and replay it later. This is widely used by games and middleware to save/restore render state across draw calls (e.g. saving state before a UI pass). D3D9 defines three stateblock types:

- `D3DSBT_ALL` — captures all device state
- `D3DSBT_PIXELSTATE` — captures only pixel shader and pixel-pipeline render states
- `D3DSBT_VERTEXSTATE` — captures only vertex shader and vertex-pipeline render states

Applications can also call `BeginStateBlock()` / `EndStateBlock()` to capture only the subset of state that was touched between those calls.

### NineStateBlock9 Structure

```c
/* nine/stateblock9.h (mesa-25.0) */
struct NineStateBlock9 {
    struct NineUnknown base;             /* COM ref counting */
    enum nine_stateblock_type type;      /* NINESBT_ALL, NINESBT_PIXEL, etc. */
    struct nine_state state;             /* Captured state snapshot */
};
```

The embedded `nine_state` is a full copy of the device state structure, including the `changed` dirty-tracking bitmask. The bitmask serves a dual purpose: during `Capture()` it records which state categories were ever touched by the snapshot; during `Apply()` it gates which categories are written back to the device.

### Capture() Implementation

```c
/* nine/stateblock9.c (mesa-25.0) */
HRESULT NINE_WINAPI
NineStateBlock9_Capture(struct NineStateBlock9 *This)
{
    struct NineDevice9 *device = This->base.device;
    struct nine_state *dst = &This->state;   /* snapshot */
    struct nine_state *src = &device->state; /* live device state */

    /*
     * nine_state_copy_common() copies only the state categories that are
     * marked dirty in the *mask* argument. By passing dst as the mask,
     * Capture() refreshes only those categories the snapshot already
     * knew about — i.e. the regions the stateblock was created to track.
     */
    if (This->type == NINESBT_ALL)
        nine_state_copy_common_all(device, dst, src,
                                   /*mask=*/dst, /*apply=*/FALSE,
                                   NULL, device->caps.MaxStreams);
    else
        nine_state_copy_common(device, dst, src,
                               /*mask=*/dst, /*apply=*/FALSE, NULL);

    /* Vertex declaration is a COM object; use nine_bind() to AddRef/Release: */
    if (dst->changed.group & NINE_STATE_VDECL)
        nine_bind(&dst->vdecl, src->vdecl);

    return D3D_OK;
}
```

The key insight is the `mask` parameter: passing the snapshot's own `changed` field as the mask means `Capture()` only refreshes state that the stateblock was tracking. If the stateblock was created with `D3DSBT_PIXELSTATE`, only pixel-shader and related render states are refreshed.

### Apply() Implementation

```c
/* nine/stateblock9.c (mesa-25.0) */
HRESULT NINE_WINAPI
NineStateBlock9_Apply(struct NineStateBlock9 *This)
{
    struct NineDevice9 *device = This->base.device;
    struct nine_state *dst = &device->state;  /* live device state */
    struct nine_state *src = &This->state;    /* snapshot to restore */
    struct nine_range_pool *pool = &device->range_pool;

    if (This->type == NINESBT_ALL)
        nine_state_copy_common_all(device, dst, src,
                                   /*mask=*/src, /*apply=*/TRUE,
                                   pool, device->caps.MaxStreams);
    else
        nine_state_copy_common(device, dst, src,
                               /*mask=*/src, /*apply=*/TRUE, pool);

    /* Forward the restored state to the worker thread's context: */
    nine_context_apply_stateblock(device, src);

    if ((src->changed.group & NINE_STATE_VDECL) && src->vdecl)
        nine_bind(&dst->vdecl, src->vdecl);

    return D3D_OK;
}
```

During `Apply()` the mask is now `src` (the snapshot), so only state categories the snapshot contains are written to the device. The `apply=TRUE` flag causes `nine_state_copy_common()` to propagate dirty bits through the `nine_range_pool` allocator for constants, ensuring later draws pick up the restored constant values.

### Constant-Range Lists

Shader constant registers are handled specially. Rather than treating the entire 256-element float4 constant array as one dirty unit, Nine maintains linked lists of `nine_range` structs that record which contiguous register spans were modified:

```c
/* nine/nine_helpers.h */
struct nine_range {
    struct nine_range *next;
    short bgn, end;   /* half-open range [bgn, end) of float4 registers */
};
```

During stateblock copy, the constant merge logic:

```c
/* nine/nine_state.c: copying VS float constants with range tracking */
if (mask->changed.group & NINE_STATE_VS_CONST) {
    struct nine_range *r;
    for (r = mask->changed.vs_const_f; r; r = r->next) {
        memcpy(&dst->vs_const_f[r->bgn * 4],
               &src->vs_const_f[r->bgn * 4],
               (r->end - r->bgn) * 4 * sizeof(float));
    }
}
```

This means that if a stateblock only touched registers 0–3 and 200–203, the copy skips the remaining 248 float4 slots entirely — a significant win for games that use large constant arrays but only modify a few each frame.

---

## nine_context.c: The Command Queue

### Motivation: CPU/GPU Overlap on Multi-core Systems

Nine's original design called `pipe_context` from the application thread directly. From approximately Mesa 19.0 onward, Nine adopted a **worker thread** and a **ring buffer** to decouple the application thread from the Gallium driver thread, enabling CPU/GPU overlap. This is conceptually similar to Vulkan's secondary command buffer model.

The implementation uses two C files:
- `nine_queue.c` — the lock-free ring buffer
- `nine_context.c` — the wrapper that encodes commands into the ring and has the worker thread decode and dispatch them to `pipe_context`

### Ring Buffer: nine_queue_pool

```c
/* nine/nine_queue.c (mesa-25.0) */

#define NINE_CMD_BUFS       32        /* power-of-two ring depth */
#define NINE_CMD_BUFS_MASK  31
#define NINE_QUEUE_SIZE     (8192 * 16 + 128)  /* ~131 KB per command buffer */
#define NINE_CMD_BUF_INSTR  256       /* max instructions per buffer */

struct nine_cmdbuf {
    unsigned instr_size[NINE_CMD_BUF_INSTR]; /* byte size of each instruction */
    unsigned num_instr;                       /* number of instructions in buf */
    unsigned offset;                          /* current write offset into mem_pool */
    void    *mem_pool;                        /* raw memory for instruction payloads */
    BOOL     full;
};

struct nine_queue_pool {
    struct nine_cmdbuf pool[NINE_CMD_BUFS];
    unsigned head;          /* producer writes here */
    unsigned tail;          /* consumer reads here */
    unsigned cur_instr;

    /* Condition variables for producer/consumer synchronisation: */
    cnd_t    event_pop;
    cnd_t    event_push;
    mtx_t    mutex_pop;
    mtx_t    mutex_push;
    BOOL     worker_wait;
};
```

The ring operates on 32 command buffers. The producer (application thread) calls `nine_queue_alloc()` to get memory for an instruction payload, then `nine_queue_flush()` when the buffer is full or on `Present()`. The consumer (worker thread) calls `nine_queue_wait_flush()` and then iterates with `nine_queue_get()` until the buffer is exhausted.

### Encoding and Decoding Draw Calls

Each command in the ring is a small C struct containing a function pointer (the handler on the worker side) and a copy of all arguments. For example, a `DrawPrimitive` command:

```c
/* nine/nine_context.c — producer side (application thread) */
struct csmt_cmd_draw_primitive {
    D3DPRIMITIVETYPE PrimitiveType;
    UINT StartVertex;
    UINT PrimitiveCount;
};

void
nine_context_draw_primitive(struct NineDevice9 *device,
                             D3DPRIMITIVETYPE PrimitiveType,
                             UINT StartVertex,
                             UINT PrimitiveCount)
{
    struct nine_state *state = &device->state;
    /* First flush any pending dirty state to the worker's copy: */
    nine_update_state(device);

    /* Allocate space in the ring buffer: */
    struct csmt_cmd_draw_primitive *cmd =
        nine_queue_alloc(device->ctx_queue,
                         sizeof(struct csmt_cmd_draw_primitive));
    if (!cmd) {
        /* Ring full — flush and retry: */
        nine_queue_flush(device->ctx_queue);
        cmd = nine_queue_alloc(device->ctx_queue,
                               sizeof(struct csmt_cmd_draw_primitive));
    }
    cmd->PrimitiveType = PrimitiveType;
    cmd->StartVertex   = StartVertex;
    cmd->PrimitiveCount = PrimitiveCount;
}

/* nine/nine_context.c — consumer side (worker thread) */
static void
exec_draw_primitive(struct NineDevice9 *device,
                    struct csmt_cmd_draw_primitive *cmd)
{
    struct pipe_context *pipe = device->pipe;
    struct pipe_draw_info info;
    /* ... fill pipe_draw_info from cmd and device->worker_state ... */
    pipe->draw_vbo(pipe, &info, 0, NULL, &draw, 1);
}
```

The worker thread loops calling `nine_queue_wait_flush()`, then dispatches each command via a function pointer table. On `Present()`, the application thread calls `nine_queue_flush()` which signals the worker and blocks until the swap is complete. [Source: Mesa nine: use threaded context MR !11866](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/11866)

---

## Shader Translation Pipeline

### D3D9 Shader Models

Nine supported the full range of Direct3D 9 shader models:

| Shader Model | Vertex Instructions | Pixel Instructions | Key Features |
|---|---|---|---|
| SM 1.1 | 128 | 12 | Basic register-based ops, no flow control |
| SM 2.0 | 256 | 96 | Static flow control (`if`/`loop` with const predicates) |
| SM 2.x | 256 | 512 | Arbitrary swizzle, dynamic flow control on some hw |
| SM 3.0 | 512 | 512 | Full dynamic flow control, vertex texture fetch, `vFace`/`vPos` |

### Translation Entry Point

The top-level translator in `nine_shader.c` parses D3D9 bytecode and emits TGSI via `ureg_program`. The entry point:

```c
/* nine/nine_shader.c (mesa-25.0) */
struct nine_shader_info {
    unsigned type;               /* PIPE_SHADER_VERTEX or PIPE_SHADER_FRAGMENT */
    const DWORD *byte_code;      /* D3D9 shader bytecode (DWORD-aligned) */
    unsigned byte_size;

    /* Output from the translator: */
    struct {
        uint8_t  num_inputs;
        uint8_t  input_map[PIPE_MAX_ATTRIBS];  /* D3DDECLUSAGE → TGSI semantic */
        uint32_t sampler_mask;
        uint32_t sampler_mask_shadow;
        uint16_t rt_mask;         /* which colour outputs are written */
        BOOL     position_t;      /* pre-transformed position (FVF path) */
        BOOL     point_size;
        unsigned const_i_base;   /* integer constant register base */
        unsigned const_b_base;   /* boolean constant base */
        unsigned bumpenvmat_needed;
    };

    void *cso;                   /* compiled Gallium CSO, set on success */
};

HRESULT
nine_translate_shader(struct NineDevice9 *device,
                      struct nine_shader_info *info,
                      struct nine_shader_variant *variant);
```

### Opcode Dispatch Table

The translator uses a static table mapping each D3D9 opcode to a TGSI opcode and an optional special-case handler:

```c
/* nine/nine_shader.c: opcode dispatch (representative excerpt) */
#define _OPI(o, t, vv0, vv1, pv0, pv1, ndst, nsrc, h) \
    { D3DSIO_##o, TGSI_OPCODE_##t, {vv0, vv1}, {pv0, pv1}, ndst, nsrc, h }

static const struct sm1_op_info inst_table[] = {
    _OPI(MOV,    MOV,    V(0,0), V(3,0), V(0,0), V(3,0), 1, 1, NULL),
    _OPI(ADD,    ADD,    V(0,0), V(3,0), V(0,0), V(3,0), 1, 2, NULL),
    _OPI(MAD,    MAD,    V(0,0), V(3,0), V(0,0), V(3,0), 1, 3, NULL),
    _OPI(MUL,    MUL,    V(0,0), V(3,0), V(0,0), V(3,0), 1, 2, NULL),
    _OPI(RCP,    RCP,    V(0,0), V(3,0), V(0,0), V(3,0), 1, 1, SPECIAL(RCP)),
    _OPI(POW,    POW,    V(0,0), V(3,0), V(0,0), V(3,0), 1, 2, SPECIAL(POW)),
    _OPI(TEX,    TEX,    V(0,0), V(0,0), V(0,0), V(1,3), 1, 0, SPECIAL(TEX)),
    _OPI(TEXKILL,KILL_IF,V(0,0), V(0,0), V(0,0), V(3,0), 1, 0, SPECIAL(TEXKILL)),
    _OPI(TEXLDD, TXD,    V(0,0), V(0,0), V(2,1), V(3,0), 1, 4, SPECIAL(TEXLDD)),
    /* SM2 flow control: */
    _OPI(IF,     IF,     V(2,0), V(3,0), V(2,1), V(3,0), 0, 1, SPECIAL(IF)),
    _OPI(IFC,    IF,     V(2,1), V(3,0), V(2,1), V(3,0), 0, 2, SPECIAL(IFC)),
    _OPI(LOOP,   BGNLOOP,V(2,0), V(3,0), V(3,0), V(3,0), 0, 2, SPECIAL(LOOP)),
    _OPI(ENDLOOP,ENDLOOP,V(2,0), V(3,0), V(3,0), V(3,0), 0, 0, NULL),
    _OPI(BREAK,  BRK,    V(2,1), V(3,0), V(2,1), V(3,0), 0, 0, NULL),
    _OPI(BREAKC, BREAKC, V(2,1), V(3,0), V(2,1), V(3,0), 0, 2, SPECIAL(BREAKC)),
    _OPI(RET,    RET,    V(2,1), V(3,0), V(2,1), V(3,0), 0, 0, NULL),
};
```

`V(major, minor)` encodes the minimum SM version that supports the opcode. The `SPECIAL()` macro attaches a C handler function for opcodes that don't translate 1:1 to a TGSI instruction.

### Vertex Declaration Mapping

A D3D9 `IDirect3DVertexDeclaration9` is an array of `D3DVERTEXELEMENT9` structs, each specifying a semantic (POSITION, TEXCOORD0, NORMAL, COLOR0, etc.) and stream/offset. Nine's translator maps these to TGSI input registers:

```c
/* nine/nine_shader.c: DCL handler for vertex inputs */
if (is_input && tx->version.major == 3) {
    /* SM3: inputs declared with explicit semantic usage and index */
    ureg_DECL_vs_input(tx->ureg, sem.reg.idx);
    tx->info->input_map[sem.reg.idx] = sm1_to_nine_declusage(&sem);
    tx->info->num_inputs = MAX2(tx->info->num_inputs,
                                sem.reg.idx + 1);
}
```

The `sm1_to_nine_declusage()` function converts `D3DDECLUSAGE_POSITION`, `D3DDECLUSAGE_TEXCOORD`, etc. to `TGSI_SEMANTIC_POSITION`, `TGSI_SEMANTIC_TEXCOORD`, etc., allowing the vertex declaration and vertex shader to be matched up at draw time.

---

## Constant Buffer Handling and Performance

### The D3D9 Constant Register File

D3D9 exposes a flat array of constant registers to shaders: up to 256 float4 registers for vertex shaders and 224 for pixel shaders. Applications update these with `SetVertexShaderConstantF(start, data, count)` and `SetPixelShaderConstantF()`. Each call touches only a contiguous subrange.

Nine tracks dirty ranges as linked lists of `nine_range` structs:

```c
/* Application calls SetVertexShaderConstantF(64, data, 8): */
void
NineDevice9_SetVertexShaderConstantF(struct NineDevice9 *device,
                                     UINT StartRegister,
                                     const float *pConstantData,
                                     UINT Vector4fCount)
{
    struct nine_state *state = &device->state;
    memcpy(&state->vs_const_f[StartRegister * 4],
           pConstantData,
           Vector4fCount * 4 * sizeof(float));

    /* Record the dirty range [64, 72) for later upload: */
    nine_ranges_insert(&state->changed.vs_const_f,
                       StartRegister,
                       StartRegister + Vector4fCount,
                       &device->range_pool);
    state->changed.group |= NINE_STATE_VS_CONST;
}
```

At draw time, `nine_update_vs_constants()` iterates only the dirty ranges and uploads them:

```c
static void
nine_update_vs_constants(struct NineDevice9 *device)
{
    struct nine_state *state = &device->state;
    struct nine_range *r;

    for (r = state->changed.vs_const_f; r; r = r->next) {
        /* Upload only [r->bgn, r->end) float4 registers: */
        pipe->set_constant_buffer(pipe, PIPE_SHADER_VERTEX, 0, FALSE,
            &(struct pipe_constant_buffer){
                .user_buffer = &state->vs_const_f[r->bgn * 4],
                .buffer_size = (r->end - r->bgn) * 4 * sizeof(float),
                .buffer_offset = r->bgn * 4 * sizeof(float),
            });
    }

    /* Return dirty range structs to the pool: */
    nine_ranges_release(&state->changed.vs_const_f, &device->range_pool);
}
```

### Comparison with DXVK's D3D9 Path

DXVK translates D3D9 to a Vulkan-like command model. In DXVK's D3D9 backend (`src/d3d9/d3d9_device.cpp`), shader constants are maintained in a staging buffer that is uploaded to a GPU buffer at draw time. Early DXVK versions uploaded the entire 256-register array as a single `VkCmdUpdateBuffer` / staging transfer, regardless of how many registers changed. [Note: DXVK has since added its own constant-dirty-range tracking in later versions, partially closing this gap.] [Source: DXVK GitHub](https://github.com/doitsujin/dxvk)

Nine's advantage is that the range-based dirty tracking is baked into the core data structure and has been present since the project's origin (circa Mesa 10.3). For CPU-bound D3D9 games where `SetVertexShaderConstantF` is called thousands of times per frame with small updates, Nine avoids copying multiple kilobytes of unchanged constant data per draw.

The sub-allocated buffer upload pool in `nine_buffer_upload.c` extends this to vertex and index buffers. It pre-allocates large 4096-byte-aligned slabs with persistent coherent CPU mapping (`PIPE_MAP_PERSISTENT | PIPE_MAP_COHERENT`), allowing the application thread to write buffer data without any explicit `pipe_context` map/unmap calls.

---

## Fixed-Function Pipeline Emulation

### The D3D9 Fixed-Function Problem

Direct3D 9 carries full backward compatibility with D3D6/D3D7's fixed-function transform-and-lighting (T&L) pipeline. Many pre-2005 titles used no vertex or pixel shaders at all, relying instead on the hardware T&L engine and the texture-stage cascade rather than programmable shaders. Nine had to implement this pipeline in software because Gallium (like all modern GPU drivers) does not expose a fixed-function pipeline — only vertex and pixel shaders.

The `nine_ff.c` file (approximately 2,000 lines) handles this by generating TGSI vertex shaders and pixel shaders on the fly that implement the fixed-function semantics. This is conceptually the same as what Mesa's OpenGL state tracker does via `st_program.c` for the OpenGL fixed-function pipeline.

### Vertex Transform and Lighting

When no vertex shader is bound (`device->state.vs == NULL`), Nine calls its internal fixed-function VS builder (`nine_ff_get_vs()`) which synthesises a vertex shader from the current matrix stack and lighting configuration. The builder is invoked lazily from `nine_update_state()` when `NINE_STATE_FF` is dirty:

```c
/* nine/nine_ff.c (mesa-25.0) — abridged */
static HRESULT
nine_ff_build_vs(struct NineDevice9 *device, struct nine_ff_vs_key *key)
{
    struct ureg_program *ureg = ureg_create(PIPE_SHADER_VERTEX);
    /* Declare input positions, normals, texture coordinates
     * based on the FVF (Flexible Vertex Format) descriptor: */
    struct ureg_src pos    = ureg_DECL_vs_input(ureg, NINE_DECLUSAGE_POSITION);
    struct ureg_src normal = ureg_DECL_vs_input(ureg, NINE_DECLUSAGE_NORMAL);

    /* World × View × Projection transform: */
    struct ureg_dst oPos = ureg_DECL_output(ureg, TGSI_SEMANTIC_POSITION, 0);
    ureg_DP4(ureg, ureg_writemask(oPos, TGSI_WRITEMASK_X),
             pos, ureg_DECL_constant(ureg, NINE_FF_VS_CONST_MVP + 0));
    ureg_DP4(ureg, ureg_writemask(oPos, TGSI_WRITEMASK_Y),
             pos, ureg_DECL_constant(ureg, NINE_FF_VS_CONST_MVP + 1));
    ureg_DP4(ureg, ureg_writemask(oPos, TGSI_WRITEMASK_Z),
             pos, ureg_DECL_constant(ureg, NINE_FF_VS_CONST_MVP + 2));
    ureg_DP4(ureg, ureg_writemask(oPos, TGSI_WRITEMASK_W),
             pos, ureg_DECL_constant(ureg, NINE_FF_VS_CONST_MVP + 3));

    /* Lighting loop (up to 8 D3D9 lights): */
    if (key->lighting) {
        for (unsigned i = 0; i < key->num_lights; i++)
            nine_ff_emit_lighting(ureg, key, i, pos, normal);
    }

    /* Texture coordinate generation (TEXGEN modes): */
    for (unsigned stage = 0; stage < key->num_tex_stages; stage++)
        nine_ff_emit_texgen(ureg, key, stage, pos, normal);

    ureg_END(ureg);
    return nine_ureg_to_vs_cso(device, ureg, key);
}
```

The synthesised shader is keyed on a `nine_ff_vs_key` struct that encodes the current FVF flags, light count, enabled lights, texgen modes, and fog type. The key is hashed and the resulting CSO cached so subsequent frames reuse the compiled shader if the fixed-function configuration is unchanged.

### Texture Stage Cascade

D3D9's pixel pipeline is described by a cascade of `D3DTSS_*` texture stage states (up to 8 stages). Each stage performs a colour and alpha combine operation on the output of the previous stage and a sampled texture. Nine's fixed-function pixel shader emulator in `nine_ff.c` reads the `D3DTSS_COLOROP`, `D3DTSS_COLORARG1/2`, `D3DTSS_ALPHAOP`, etc. tokens and emits a TGSI fragment shader that implements the cascade:

```c
/* nine/nine_ff.c — pixel shader key for texture stages */
struct nine_ff_ps_key {
    uint32_t texcoord_gen;     /* bitfield: which stages use texgen */
    uint8_t  colorop[8];       /* D3DTOP_* for each stage */
    uint8_t  alphaop[8];
    uint8_t  colorarg1[8];     /* D3DTA_* arg selectors */
    uint8_t  colorarg2[8];
    uint8_t  alphaarg1[8];
    uint8_t  alphaarg2[8];
    uint8_t  sampler_type[8];  /* 2D, CUBE, VOLUME */
    uint8_t  fog_mode;         /* D3DFOG_NONE / LINEAR / EXP / EXP2 */
    BOOL     alpha_test;
};
```

The combiner operations include `D3DTOP_MODULATE` (multiply), `D3DTOP_ADD`, `D3DTOP_BLENDDIFFUSEALPHA`, `D3DTOP_DOTPRODUCT3`, and around 20 others. Nine emits a TGSI instruction sequence for each operation, reading from `TGSI_SEMANTIC_TEXCOORD` registers and `TGSI_SEMANTIC_COLOR` interpolated values. [Source: nine/nine_ff.c in Mesa](https://gitlab.freedesktop.org/mesa/mesa/-/tree/mesa-25.0/src/gallium/frontends/nine)

### Quirk Handling

`nine_quirk.c` contains a database of per-game workarounds. D3D9's specification allows some behaviours (such as reading back render target contents immediately after a draw, or using `D3DPOOL_SCRATCH` for texture staging) that Gallium drivers implement differently from Windows drivers. Nine detected known games by executable name or D3D9 application-GUID and applied patches:

```c
/* nine/nine_quirk.c (mesa-25.0) — Note: exact entries are illustrative;
 * the actual quirk table content needs verification against upstream source */
static const struct nine_quirk_entry nine_quirk_table[] = {
    /* GTA San Andreas: does not set D3DRS_SRGBWRITEENABLE but expects
     * sRGB writes on the backbuffer. */
    { "gta_sa.exe",   NINE_QUIRK_FORCE_SRGB_BACKBUFFER },
    /* Max Payne 2: sends draw calls before Clear() on a fresh device. */
    { "maxpayne2.exe", NINE_QUIRK_ALLOW_DRAW_BEFORE_CLEAR },
    /* Dark Messiah: expects D3DFMT_X8R8G8B8 render targets to be
     * SRGB-readable without the SRGB flag. */
    { "mm.exe",        NINE_QUIRK_DARK_MESSIAH_MSAA },
};
```

The Mesa Nine bug tracker ([Mesa-3D issue tracker on GitHub](https://github.com/iXit/Mesa-3D/issues)) and the Gallium Nine issue list on freedesktop.org Bugzilla record the games for which quirks were added. The pattern — detect by executable name, set a flag, adjust behaviour in the state machine — is consistent across the file; the exact quirk flags for specific titles require source verification.

---

## SwapChain9 and Present()

### The Present() Pipeline

`IDirect3DSwapChain9::Present()` is the most time-critical call in Nine: it must flush all pending draw commands, synchronise with the X server, and schedule a buffer swap — all with minimal latency.

Nine's `swapchain9.c` owns the DRI present context and manages the swap chain's backbuffers (typically 2 or 3 for triple-buffering). The Present() path:

1. **Flush the command queue**: `nine_queue_flush(device->ctx_queue)` signals the worker thread to drain and execute all pending commands.
2. **Resolve MSAA**: if the backbuffer was created with a multisampled format (`D3DMULTISAMPLE_4_SAMPLES`, etc.), Nine calls `pipe_context::blit()` to resolve the MSAA surface into the presentable single-sampled buffer.
3. **Copy to presentable surface** (if dirty rectangle or stretching): Nine supports `pSourceRect`/`pDestRect` in Present(), which requires a Gallium blit. Full-screen, full-surface presents skip the blit.
4. **X Present**: Nine calls the X Present extension (`XPresentPixmap`) with the DRI3 pixmap handle for the resolved backbuffer. The X server schedules the pixel delivery to the display during the next vertical blank.
5. **Rotate backbuffers**: after signalling the present, Nine advances the backbuffer index and the just-presented buffer becomes the new render target for the next frame.

```c
/* nine/swapchain9.c (mesa-25.0) — abridged Present() logic */
HRESULT NINE_WINAPI
NineSwapChain9_Present(struct NineSwapChain9 *This,
                       const RECT *pSourceRect,
                       const RECT *pDestRect,
                       HWND hDestWindowOverride,
                       const RGNDATA *pDirtyRegion,
                       DWORD dwFlags)
{
    struct NineDevice9 *device = This->base.device;

    /* (1) Flush all pending draw commands to the pipe driver: */
    nine_context_pipe_flush(device);

    /* (2) MSAA resolve if needed: */
    if (This->params.MultiSampleType != D3DMULTISAMPLE_NONE)
        nine_context_resolve_target(device, This->backbuffer[This->cur_back]);

    /* (3) DRI3 present — hands the DMA-BUF fd to the X server: */
    PRESENTPixmap(This->present, This->present_buffers[This->cur_back],
                 This->present_output, This->swap_interval,
                 pSourceRect, pDestRect);

    /* (4) Advance backbuffer index: */
    This->cur_back = (This->cur_back + 1) % This->num_back;

    return D3D_OK;
}
```

### Triple Buffering and Frame Pacing

Nine supported triple-buffering via `D3DPRESENT_INTERVAL_ONE` (vsync) and `D3DPRESENT_INTERVAL_IMMEDIATE` (no vsync). With DRI3+Present, triple buffering works by keeping three DRI3 pixmaps: one on-screen, one submitted to the X server (in flight), and one being rendered into by the GPU. The Present extension returns a serial number for each submitted frame, and Nine uses the `PresentWaitMSC` call to block until the in-flight frame is displayed before submitting a new one, preventing the application from getting arbitrarily far ahead of the display. [Source: X Present extension spec](https://cgit.freedesktop.org/xorg/proto/presentproto)

---

## DRI3 and DMA-BUF Texture Path

### The X11 Rendering Problem

Nine's output (the rendered backbuffer) needs to reach the X server for display. Two DRI backends handle this:

- **DRI2** (legacy): the Gallium driver renders into a GEM buffer, then the kernel copies it into an X pixmap via the DRM driver. This copy is expensive.
- **DRI3** (preferred): the Gallium driver renders into a DMA-BUF exported from a GEM buffer, and DRI3 passes the file descriptor directly to the X server with `DRI3PixmapFromBuffers`. No copy occurs — the X compositor and the Gallium driver share the same GEM buffer. [Source: Mesa Gallium Nine docs](https://idr.pages.freedesktop.org/mesa/gallium-nine.html)

The `d3dadapter9.c` entry point in `wine-nine-standalone` initialises the backend:

```c
/* wine-nine-standalone/d3d9-nine/d3dadapter9.c */
HRESULT
d3dadapter9_new(Display *gdi_display, boolean ex, IDirect3D9Ex **ppOut)
{
    struct D3DAdapter9DRI *This = CALLOC_STRUCT(D3DAdapter9DRI);
    /* Try DRI3 backend first (lower overhead, no copy): */
    This->dri_backend = backend_create(gdi_display,
                                       DefaultScreen(gdi_display));
    if (!This->dri_backend) {
        /* Fallback to DRI2: */
        This->dri_backend = backend_create_dri2(gdi_display,
                                                DefaultScreen(gdi_display));
    }
    /* ... */
}
```

[Source: wine-nine-standalone d3dadapter9.c](https://github.com/iXit/wine-nine-standalone/blob/main/d3d9-nine/d3dadapter9.c)

### DRI3 Open: Render Node Access

DRI3's `DRI3Open` request returns a file descriptor for the render node (`/dev/dri/renderD128`). Nine uses this to open the Gallium driver's `pipe_screen` without needing root or GBM. The `ID3DAdapter9` interface (Mesa's custom extension) takes the render-node FD and returns a `pipe_screen *`:

```c
/* include/d3dadapter/d3dadapter9.h */
typedef HRESULT (WINAPI *PFN_D3DADAPTER9CREATE)(
    struct pipe_screen *screen,
    int minorVersion,
    struct D3DAdapter9DRM **ppOut
);
```

The `d3dadapter9.so` library is located by `wine-nine-standalone` via the `D3D_MODULE_PATH` environment variable or the Mesa installation prefix. Mesa builds it as a DRI driver alongside `radeonsi_dri.so`.

### XWayland Considerations

Under XWayland, X11 clients believe they are talking to a native X11 server, but the X server itself is a Wayland client. The DRI3 protocol still works in this configuration because XWayland implements `DRI3` and passes DMA-BUF file descriptors through to the underlying Wayland compositor. However, there are two caveats:

1. **Present extension**: Nine's `SwapChain9::Present()` uses the X Present extension (`XPresentPixmap`) to flip buffers atomically. Under XWayland, `XPresent` works but goes through an additional compositing step because XWayland doesn't have direct flip access to the display hardware — it must re-present the buffer to the Wayland compositor. This adds one frame of latency compared to running on native X11.

2. **Tearing**: On native X11 with DRI3+Present, Nine achieves tear-free vsync by letting the X server schedule flips. Under XWayland the Wayland compositor controls vsync, and Nine's vsync requests are advisory only.

> **Note: needs verification** — the exact XWayland DRI3 codepath and latency figures depend on the compositor. The behaviour described above applies to wlroots-based compositors (Sway, Hyprland) and Mutter (GNOME). KWin (KDE) may behave differently.

---

## Enabling Nine in Wine

### Prerequisites

Nine requires:
1. Mesa built with `-Dgallium-nine=true` (default when Gallium drivers are enabled)
2. The `d3dadapter9.so` library installed (usually in `/usr/lib/d3d/` and `/usr/lib32/d3d/` for 32-bit)
3. A Wine variant patched with the Nine loader, or the `wine-nine-standalone` module

Most D3D9 games are 32-bit and require both the 64-bit and 32-bit `d3dadapter9.so` libraries.

### wine-nine-standalone

The preferred method since at least Mesa 19.x was the `wine-nine-standalone` project:

```bash
# Arch Linux:
sudo pacman -S wine-nine

# Fedora (from Copr):
sudo dnf copr enable siro/wine-nine
sudo dnf install wine-nine

# Ubuntu (unofficial PPA):
sudo add-apt-repository ppa:kisak/kisak-mesa
sudo apt install mesa-nine
```

Enable Nine for a given Wine prefix using the bundled GUI tool:

```bash
WINEPREFIX=~/.wine ninewinecfg
```

Or manually via Wine registry:

```bash
WINEPREFIX=~/.wine wine regsvr32 d3d9nine.dll
```

### WINEDLLOVERRIDES

Nine is loaded by overriding Wine's built-in `d3d9.dll`:

```bash
WINEDLLOVERRIDES="d3d9=n,b" wine game.exe
# "n" = native (use the installed d3d9nine.dll)
# "b" = builtin fallback if native load fails
```

Nine can also be pointed at a non-default `d3dadapter9.so`:

```bash
D3D_MODULE_PATH=/opt/mesa-25.0/lib/d3d/d3dadapter9.so \
WINEDLLOVERRIDES="d3d9=n,b" wine game.exe
```

### Mesa Build with Nine

```bash
# Nine is enabled by default alongside Gallium drivers.
# Explicit flag for clarity:
meson setup builddir \
    -Dgallium-drivers=radeonsi,iris,nouveau,softpipe \
    -Dgallium-nine=true \
    -Dvulkan-drivers=amd,intel
ninja -C builddir

# Install:
sudo ninja -C builddir install
```

Mesa builds `d3dadapter9.so` and places it in `${prefix}/lib/d3d/`. For 32-bit support, a second Mesa build with `--cross-file=meson/cross/i686-linux-gnu.txt` is needed.

### Checking Nine Availability

```bash
# Check for the installed libraries:
ls /usr/lib/d3d/d3dadapter9.so.1
ls /usr/lib32/d3d/d3dadapter9.so.1   # needed for 32-bit games

# Verify Nine is being used:
NINE_DEBUG=1 wine game.exe 2>&1 | head -20
# Expected: "D3D9 state tracker: loading d3dadapter9.so"

# Force software fallback for testing:
GALLIUM_DRIVER=softpipe WINEDLLOVERRIDES="d3d9=n,b" wine game.exe
```

### Environment Variables

```bash
NINE_DEBUG=1          # basic debug trace
NINE_DEBUG=stat       # state machine dump at exit
NINE_DEBUG=shader     # shader bytecode and TGSI/NIR output
NINE_DEBUG=ff         # fixed-function pipeline trace
NINE_THROTTLE=0       # disable frame throttle (may cause tearing)
D3D_BACKEND=dri3      # force DRI3 backend (default)
D3D_BACKEND=dri2      # force DRI2 backend (legacy fallback)
```

---

## Performance Comparison: Nine vs WineD3D vs DXVK

### Why Nine Was Faster than WineD3D

The performance improvement over WineD3D came from several architectural factors:

1. **Eliminated OpenGL translation layer**: Nine bypassed the Mesa OpenGL frontend entirely. WineD3D had to maintain an OpenGL state machine in parallel with D3D9 semantics.

2. **Single-hop shader compilation**: D3D9 bytecode → TGSI/NIR → ISA is two passes. WineD3D's path was D3D9 bytecode → GLSL text → NIR → ISA, with GLSL parsing adding significant CPU time.

3. **Eager shader compilation**: Nine compiled shaders at `CreateVertexShader` / `CreatePixelShader` time. WineD3D originally deferred compilation to the first draw, causing stutter. [Source](https://www.gamingonlinux.com/2015/02/an-interview-with-gallium-nine-project-developer-axel-davy/)

4. **Range-based constant upload**: Nine's `nine_range` lists avoided re-uploading unchanged constant registers. WineD3D re-set all touched state categories each draw.

Typical improvement over WineD3D was reported at 30–80% FPS uplift in CPU-bound D3D9 titles (GTA: San Andreas, Portal, World of Warcraft). An Arch Linux user benchmark measured Portal at 89 FPS with Nine versus a lower baseline with WineD3D, on an AMD HD 7730M laptop. [Source](https://wiki.ixit.cz/d3d9)

### Nine vs DXVK

DXVK translates D3D9 → DXVK's D3D11-like IR → Vulkan SPIR-V → NIR → ISA. On RADV/ANV with good pipeline caching, DXVK can match or exceed Nine because:

- DXVK's Vulkan path benefits from explicit pipeline objects, reducing driver overhead per draw
- DXVK handles async pipeline compilation, eliminating compile-time stalls on first use of a shader variant
- DXVK runs on any Vulkan driver (RADV, ANV, NVK, proprietary NVIDIA) and on Windows via native D3D11; Nine required a Gallium driver and only worked on Linux/X11

LinuxReviews benchmarks placed Nine at 10–20 FPS faster than DXVK in some titles and roughly equal in others, with the difference depending heavily on how CPU-bound the game was and how mature DXVK's D3D9 support was at the time. [Source](https://linuxreviews.org/Gallium_Nine)

> **Note: needs verification** — direct Nine-vs-DXVK benchmarks are sparse for post-2022 DXVK versions, and the gap narrowed as DXVK's D3D9 backend matured. The DXVK 1.9.1 release claimed a 15% average uplift over the native D3D9 path on CPU-bound workloads, suggesting DXVK's performance was competitive with Nine at that time. [Source](https://linuxiac.com/dxvk-2-7-1-fixes-msaa-boosts-d3d9-game-performance/)

### Proton Nine

Proton (Valve's Steam Play layer) defaulted to DXVK for D3D9 titles. Nine was available as a manual override:

```bash
# Force Nine in Proton (manual override — not recommended for most games):
PROTON_USE_NINE=1 %command%
```

Valve's preference for DXVK was primarily portability: DXVK runs on both RADV and proprietary NVIDIA drivers, whereas Nine only worked on Gallium drivers.

---

## Deprecation and Removal (Mesa 25.1–25.2)

### Timeline

- **Mesa 25.1 (April 2025)**: The `src/gallium/frontends/nine/` state tracker was deprecated by Mike Blumenkrantz, flagged for removal in 25.2. Gallium-XA was deprecated in the same commit. [Source](https://www.phoronix.com/news/Gallium-Nine-Deprecated)
- **Mesa 25.2 (August 2025)**: Nine was removed from Mesa mainline. The `d3dadapter9.so` library stopped being built. [Source](https://docs.mesa3d.org/relnotes/25.2.0.html)

### Reasons for Removal

The deprecation notice cited:

1. **DXVK dominance**: DXVK (and vkd3d-proton for D3D12) became the canonical D3D compatibility layers. DXVK runs on any Vulkan driver, not just Gallium, making Nine's Gallium-only restriction a liability as RADV, ANV, and NVK matured.
2. **Maintenance burden**: Nine was a 25 KLOC codebase implementing a full COM API hierarchy. As Gallium's `pipe_context` interface evolved, Nine required constant updates. With fewer maintainers and fewer users (most had migrated to DXVK), the benefit-to-maintenance ratio was unfavourable.
3. **X11-only restriction**: Nine's DRI3/Present path was fundamentally tied to X11. Wayland-native gaming was increasingly common, and Nine had no clean path to Wayland without a complete Present implementation rewrite.

### Ecosystem Impact

- **wine-nine-standalone**: The [iXit/wine-nine-standalone](https://github.com/iXit/wine-nine-standalone) project, which decoupled Nine from Wine, lost its upstream source. The repository was effectively archived once Mesa mainline removed the code.
- **Distribution packages**: Downstream distributions (Arch, Fedora, Ubuntu) dropped the `d3dadapter9.so` package during their Mesa 25.2 packaging cycles.
- **Proton**: Proton already defaulted to DXVK for D3D9, so the impact on Valve's compatibility layer was minimal.
- **Historical reference**: The Nine source (tag `mesa-25.0`) remains the definitive example of a minimal Gallium frontend implementing a complete COM-style graphics API over `pipe_context`. [Source: mesa-25.0 tag](https://gitlab.freedesktop.org/mesa/mesa/-/tree/mesa-25.0/src/gallium/frontends/nine)

### Long-term Trajectory

The removal of Nine (and Clover for OpenCL before it) reflects a broader Mesa strategy of consolidating around Vulkan-backed paths. RADV, ANV, and NVK are the primary hardware drivers; Zink (OpenGL on Vulkan) bridges legacy OpenGL applications on top of those. The Gallium frontend model — which Nine exemplified — remains in use for RadeonSI, Iris, and Nouveau, but the era of new COM-style API state trackers built on Gallium appears to have ended.

---

## Integrations

- **Ch14 (Gallium3D)**: Nine is a Gallium frontend — it calls `pipe_context` and `pipe_screen` directly, the same interface used by the OpenGL, OpenCL, and Zink (Vulkan) state trackers. The `pipe_context` draw, bind, and set_* methods are Nine's primary interface to hardware.
- **Ch15 (TGSI and NIR)**: Nine translates D3D9 SM1–SM3 bytecode to TGSI (and subsequently NIR via Mesa's TGSI-to-NIR lowering pass), which then goes through the standard Gallium NIR → ISA pipeline (RadeonSI, Iris, etc.).
- **Ch16 (RadeonSI)**: RadeonSI was the primary Gallium driver for Nine on AMD GPUs. Nine directly called RadeonSI's `pipe_context` without any Vulkan or OpenGL layer, which is why the Nine + RadeonSI combination was significantly faster than DXVK + RADV on older hardware.
- **Ch17 (Iris)**: Iris is the Gallium driver Nine used on Intel hardware (Gen 8+). Nine worked on Iris but not on ANV (Intel's Vulkan driver), because ANV does not expose a `pipe_context` interface.
- **Ch66 (Wine/Proton D3D)**: Wine's D3D9 → WineD3D path is what Nine replaced. DXVK is the successor that targets Vulkan instead of Gallium, providing wider hardware and OS compatibility. DXVK's D3D9 mode is documented in that chapter.
- **Ch152 (Rust GPU)**: wgpu targets Vulkan/Metal/GL, not Gallium; there is no Rust equivalent of Nine's native Gallium approach.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
