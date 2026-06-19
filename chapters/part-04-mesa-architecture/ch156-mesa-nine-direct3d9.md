# Chapter 156: Mesa Nine: Direct3D 9 State Tracker

**Target audiences**: Wine and Proton developers who need to understand why Nine outperforms the OpenGL translation path; Mesa contributors working on the Gallium interface; and systems engineers optimising D3D9 game compatibility on Linux.

---

## Table of Contents

1. [Introduction](#introduction)
2. [The D3D9 on Linux Problem](#the-d3d9-on-linux-problem)
3. [Mesa Nine Architecture](#mesa-nine-architecture)
4. [Gallium State Tracker Interface](#gallium-state-tracker-interface)
5. [The Nine State Machine](#the-nine-state-machine)
6. [Wine and Proton Integration](#wine-and-proton-integration)
7. [Performance Comparison with WineD3D](#performance-comparison-with-wined3d)
8. [Nine Shader Translation](#nine-shader-translation)
9. [Building and Enabling Nine](#building-and-enabling-nine)
10. [Integrations](#integrations)

---

## Introduction

**Mesa Nine** (also called "Gallium Nine") is a Direct3D 9 state tracker built directly into Mesa. Instead of translating D3D9 calls to OpenGL (as Wine's WineD3D does), Nine translates D3D9 API calls directly to Gallium3D pipe driver calls. This eliminates the entire OpenGL layer, dramatically improving performance for D3D9 games running under Wine or Proton.

Nine is available in Mesa's `src/gallium/frontends/nine/` and is enabled by default in most Linux distributions. It requires a Gallium driver (RadeonSI, Iris, Nouveau, LLVMpipe) — not Zink, not ANV/RADV.

The concept was pioneered by Axel Davy and others; it became a viable Wine integration via the `d3dadapter9` native module loaded by Wine's D3D9 stub.

Sources: [Mesa Nine source](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/gallium/frontends/nine) | [Nine FAQ](https://github.com/iXit/wine-nine-standalone)

---

## The D3D9 on Linux Problem

### WineD3D Translation Chain

Standard Wine D3D9 game path:

```
D3D9 game → WineD3D (Wine's D3D implementation)
  → translate every D3D9 call to OpenGL
    → Mesa OpenGL frontend (GLX/EGL)
      → Gallium3D state tracker for OpenGL
        → pipe driver (RadeonSI/Iris/etc.)
```

The OpenGL translation adds:
- **State impedance**: D3D9 and OpenGL have different state machines; mapping is lossy and requires workarounds
- **Shader translation**: D3D SM 2.0/3.0 bytecode → GLSL → NIR → GPU ISA (three translations)
- **Flush overhead**: OpenGL state flushes at awkward points

### Nine's Shortcut

```
D3D9 game → Nine state tracker (Mesa)
  → Gallium3D pipe calls
    → pipe driver (RadeonSI/Iris/etc.)
```

Nine eliminates WineD3D entirely. It speaks Gallium3D natively, the same language RadeonSI and Iris use.

---

## Mesa Nine Architecture

### Source Tree

```
mesa/src/gallium/frontends/nine/
├── nine_state.c          # D3D9 state machine → pipe_context calls
├── nine_shader.c         # D3D9 SM2/SM3 bytecode → TGSI/NIR
├── nine_pipe.c           # D3D9 → Gallium type conversions
├── device9.c             # IDirect3DDevice9 implementation
├── swapchain9.c          # IDirect3DSwapChain9 implementation
├── texture9.c            # IDirect3DTexture9
├── surface9.c            # IDirect3DSurface9
├── vertexbuffer9.c       # IDirect3DVertexBuffer9
├── indexbuffer9.c        # IDirect3DIndexBuffer9
├── vertexshader9.c       # IDirect3DVertexShader9
├── pixelshader9.c        # IDirect3DPixelShader9
└── nine_helpers.h        # Common macros and helpers
```

### Key Objects

Nine implements the full `IDirect3DDevice9` COM interface:

```c
/* nine/device9.c */
struct NineDevice9 {
    struct NineUnknown base;        /* COM reference counting */
    struct pipe_context *pipe;      /* Gallium pipe context */
    struct pipe_screen *screen;     /* Gallium screen */

    /* Current D3D9 state: */
    struct nine_state state;

    /* Caches for compiled shaders, vertex declarations, etc.: */
    struct util_hash_table *vs_cache;
    struct util_hash_table *ps_cache;

    /* Swap chain: */
    struct NineSwapChain9 *swapchain;
};
```

---

## Gallium State Tracker Interface

### How Nine Calls Gallium

Nine is one of Mesa's Gallium frontends alongside the OpenGL state tracker and Clover (OpenCL). It calls `pipe_context` methods directly:

```c
/* nine/nine_state.c: translating D3D9 draw call to Gallium */
static void
nine_update_state_and_commit(struct NineDevice9 *device)
{
    struct pipe_context *pipe = device->pipe;

    /* Set vertex shader: */
    pipe->bind_vs_state(pipe, device->state.vs->cso);

    /* Set pixel shader: */
    pipe->bind_fs_state(pipe, device->state.ps->cso);

    /* Set vertex buffers: */
    pipe->set_vertex_buffers(pipe, 0,
        device->state.num_vstreams, 0, false,
        device->state.vtxbuf);

    /* Set rasterizer state: */
    pipe->bind_rasterizer_state(pipe, device->state.cso_rasterizer);

    /* Set blend state: */
    pipe->bind_blend_state(pipe, device->state.cso_blend);

    /* Set depth/stencil: */
    pipe->bind_depth_stencil_alpha_state(pipe, device->state.cso_zsa);

    /* Draw: */
    pipe->draw_vbo(pipe, &device->state.draw, 0, NULL, &info, 1);
}
```

### D3D9 → Gallium Format Mapping

```c
/* nine/nine_pipe.c */
enum pipe_format
d3d9_to_pipe_format_checked(struct pipe_screen *screen,
    D3DFORMAT format, unsigned usage)
{
    switch (format) {
    case D3DFMT_A8R8G8B8: return PIPE_FORMAT_B8G8R8A8_UNORM;
    case D3DFMT_X8R8G8B8: return PIPE_FORMAT_B8G8R8X8_UNORM;
    case D3DFMT_R5G6B5:   return PIPE_FORMAT_B5G6R5_UNORM;
    case D3DFMT_A16B16G16R16F: return PIPE_FORMAT_R16G16B16A16_FLOAT;
    case D3DFMT_DXT1:     return PIPE_FORMAT_DXT1_RGB;
    case D3DFMT_DXT3:     return PIPE_FORMAT_DXT3_RGBA;
    case D3DFMT_DXT5:     return PIPE_FORMAT_DXT5_RGBA;
    case D3DFMT_D24S8:    return PIPE_FORMAT_Z24_UNORM_S8_UINT;
    /* ... */
    default:
        return PIPE_FORMAT_NONE;
    }
}
```

---

## The Nine State Machine

### D3D9 Render States

D3D9 has ~200 render states (blending, fog, alpha test, stencil, etc.). Nine maps these to Gallium CSOs (Constant State Objects) which are compiled on first use:

```c
/* nine/nine_state.c: building rasterizer CSO from D3D9 state */
static void *
nine_create_rasterizer_state(struct NineDevice9 *device)
{
    struct pipe_rasterizer_state rs;
    memset(&rs, 0, sizeof(rs));

    rs.fill_front = rs.fill_back =
        (device->state.rs[D3DRS_FILLMODE] == D3DFILL_WIREFRAME)
            ? PIPE_POLYGON_MODE_LINE : PIPE_POLYGON_MODE_FILL;

    rs.cull_face =
        (device->state.rs[D3DRS_CULLMODE] == D3DCULL_CW)  ? PIPE_FACE_FRONT :
        (device->state.rs[D3DRS_CULLMODE] == D3DCULL_CCW) ? PIPE_FACE_BACK  :
                                                              PIPE_FACE_NONE;

    rs.scissor = !!(device->state.rs[D3DRS_SCISSORTESTENABLE]);
    rs.depth_clip_near = rs.depth_clip_far = 1;

    /* D3D9 uses [0,1] depth range; Gallium too (unlike OpenGL's [-1,1]): */
    rs.clip_halfz = 1;

    return device->pipe->create_rasterizer_state(device->pipe, &rs);
}
```

### State Dirty Tracking

Nine tracks which parts of D3D9 state changed:

```c
/* nine/nine_state.h */
#define NINE_STATE_FB         (1 << 0)   /* render target / depth buffer */
#define NINE_STATE_VIEWPORT   (1 << 1)
#define NINE_STATE_SCISSOR    (1 << 2)
#define NINE_STATE_RASTERIZER (1 << 3)
#define NINE_STATE_BLEND      (1 << 4)
#define NINE_STATE_DSA        (1 << 5)   /* depth/stencil/alpha */
#define NINE_STATE_VS         (1 << 6)
#define NINE_STATE_PS         (1 << 7)
#define NINE_STATE_TEXTURE    (1 << 8)
#define NINE_STATE_VDECL      (1 << 9)   /* vertex declaration */
/* ... */

/* Only update dirty state before draw: */
void nine_update_state(struct NineDevice9 *device) {
    uint32_t dirty = device->state.dirty;
    if (dirty & NINE_STATE_RASTERIZER) nine_update_rasterizer(device);
    if (dirty & NINE_STATE_BLEND)      nine_update_blend(device);
    if (dirty & NINE_STATE_VS)         nine_update_vs(device);
    /* ... */
    device->state.dirty = 0;
}
```

---

## Wine and Proton Integration

### d3d9-nine: The Wine Native DLL

Nine is loaded into Wine via the `d3d9.dll` native override. Two approaches:

**1. wine-nine-standalone** ([github.com/iXit/wine-nine-standalone](https://github.com/iXit/wine-nine-standalone)):

```bash
# Install:
sudo pacman -S wine-nine    # Arch
# or build from source

# Enable per-game in Wine prefix:
WINEPREFIX=~/.wine ninewinecfg  # GUI tool
# or manually:
WINEPREFIX=~/.wine wine regsvr32 d3d9.dll
```

**2. Direct Mesa integration** (the `mesa-nine-standalone` package, distribution-specific):

```bash
# Fedora:
sudo dnf install mesa-nine-dri
# Ubuntu/Debian (unofficial PPA):
sudo add-apt-repository ppa:kisak/kisak-mesa
sudo apt install mesa-nine
```

### How Wine Loads Nine

```c
/* Wine's d3d9.dll (native override) calls Mesa via libd3d9.so: */
HRESULT WINAPI Direct3DCreate9Ex(UINT sdk_version, IDirect3D9Ex **out)
{
    /* Load Mesa Nine via GPA: */
    void *nine_handle = dlopen("d3d9-nine.so", RTLD_NOW);
    D3DAdapter9CreateProc create = dlsym(nine_handle, "D3DAdapter9Create");
    return create(sdk_version, out);
}
```

Nine uses a private DRI extension (`DRI_IMAGE_DRIVER`) to connect to the Mesa Gallium driver's pipe context from userspace.

### Proton Nine

Proton (Valve's Steam Play compatibility layer) includes Nine support but typically prefers DXVK for D3D9 (since DXVK targets Vulkan and RADV/ANV). Nine is available as an alternative when DXVK has issues or on hardware that DXVK doesn't support well.

```bash
# Force Nine in Proton (not recommended for most games — DXVK is usually better):
PROTON_USE_NINE=1 %command%
```

---

## Performance Comparison with WineD3D

### Benchmark Context

Nine's advantage is most pronounced in:
- **Draw call-heavy scenes**: Nine skips the OpenGL frontend overhead per draw
- **Shader-heavy workloads**: D3D9 SM3 → TGSI is one hop; SM3 → GLSL → NIR is two
- **Old titles** from 2004–2012 that rely heavily on D3D9 fixed-function or SM2/3 features

Typical improvement over WineD3D in CPU-bound D3D9 games:
- 30–50% improvement in draw-call-heavy games (GTA San Andreas, World of Warcraft era)
- 10–20% in shader-heavy games
- May be slower than DXVK for games using SM2.0 extensively (DXVK's Vulkan path is highly optimised)

### Why DXVK Sometimes Wins

DXVK translates D3D9→D3D11-like→Vulkan. On RADV/ANV with good pipeline caching, DXVK can beat Nine because:
- DXVK's Vulkan path avoids OpenGL driver overhead
- DXVK has mature shader translation (SPIRV-Cross → NIR → ISA)
- DXVK handles async pipeline compilation better

Nine wins when the OpenGL Gallium path has driver-specific optimisations (e.g. RadeonSI has fast paths for old D3D9 fixed-function state).

---

## Nine Shader Translation

### D3D9 Shader Model to NIR

Nine compiles D3D9 vertex and pixel shader bytecode (SM 1.1–3.0) via its own translator to TGSI (legacy) and increasingly to NIR:

```c
/* nine/nine_shader.c: D3D9 bytecode → TGSI/NIR */
struct nine_shader_info {
    unsigned type;          /* PIPE_SHADER_VERTEX or PIPE_SHADER_FRAGMENT */
    const DWORD *byte_code; /* D3D9 shader bytecode */

    struct {
        uint8_t ndecl;           /* number of declarations */
        struct nine_vs_output vs_output[PIPE_MAX_ATTRIBS];
    };
};

/* Translate: */
nine_translate_shader(device, &info, &variant);
/* Returns: info.cso — a compiled Gallium CSO (pipe_shader_state) */
```

### Shader Model 3.0 Features

SM3.0 (D3D9c, DX9.0c) adds:
- Dynamic flow control (loops, subroutines)
- 32 instruction slots for vertex shaders (vs SM2: 256)
- 512 instruction slots for pixel shaders
- `texldd`, `texldl`, `texldb` texture sampling with derivatives

Nine handles these by mapping SM3.0 constructs to NIR equivalents:

```c
/* SM3 flow control → NIR control flow: */
case D3DSIO_IF:
    nir_push_if(b, nine_src(b, &insn->src[0]));
    break;
case D3DSIO_LOOP:
    nir_push_loop(b);
    break;
case D3DSIO_ENDLOOP:
    nir_pop_loop(b, NULL);
    break;
```

---

## Building and Enabling Nine

### Mesa Build with Nine

```bash
# Nine requires: -Dgallium-nine=true (enabled by default if Gallium drivers are built)
meson setup builddir \
    -Dgallium-drivers=radeonsi,iris,nouveau,softpipe \
    -Dgallium-nine=true \
    -Dvulkan-drivers=amd,intel
ninja -C builddir
```

### Checking Nine Availability

```bash
# Is Nine present?
ls /usr/lib/d3d/d3dadapter9.so.1
# or:
ls /usr/lib32/d3d/d3dadapter9.so.1  # 32-bit (needed for 32-bit games)

# Test Nine directly:
NINE_DEBUG=1 wine some_d3d9_game.exe 2>&1 | grep -i nine
```

### Environment Variables

```bash
# Nine debug output:
NINE_DEBUG=1        # basic debug
NINE_DEBUG=stat     # state machine dumps
NINE_DEBUG=shader   # shader translation output
NINE_DUMMY_ADAPTER=0  # use real GPU (default)

# Override Nine to use software rasterizer (for testing):
GALLIUM_DRIVER=softpipe wine game.exe
```

---

## Integrations

- **Ch14 (Gallium3D)** — Nine is a Gallium frontend: it calls `pipe_context` and `pipe_screen` directly, the same interface used by the OpenGL, OpenCL, and Vulkan (Zink) state trackers
- **Ch15 (TGSI and NIR)** — Nine translates D3D9 SM1–SM3 bytecode to NIR, which then goes through the standard Gallium NIR→ISA pipeline (RadeonSI, Iris, etc.)
- **Ch16 (RadeonSI)** — RadeonSI is the primary Gallium driver for Nine on AMD GPUs; Nine directly calls RadeonSI's `pipe_context` without any Vulkan or OpenGL layer
- **Ch17 (Iris)** — Iris is the Gallium driver Nine uses on Intel; Nine works on Iris but not on ANV (which is a Vulkan driver, not Gallium)
- **Ch66 (Wine/Proton D3D)** — Wine's D3D9 → WineD3D path is what Nine replaces; DXVK is the alternative that targets Vulkan instead of Gallium
- **Ch152 (Rust GPU)** — wgpu targets Vulkan/GLES, not Gallium; Nine's Gallium-native approach has no Rust equivalent
