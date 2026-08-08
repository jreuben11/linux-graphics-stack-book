# Chapter 205c: Open-Source 2D Simulation-Game Engines: Sprite Blitters, GPU Batching, and Reimplementation Modding

> **Part**: Part XI — Engines and Creative Tools
> **Audience**: Graphics and rendering engineers interested in 2D sprite-compositing architecture, and specifically in the CPU-to-GPU migration path as it plays out in real shipped codebases rather than in tutorials. Readers who work on software rasterizers, sprite batchers, or texture-atlas allocators will find directly transferable design decisions here.
> **Status**: First draft — 2026-08-08

The community reimplementation scene occupies an unusual position in graphics engineering. These projects — clean-room or decompilation-derived rewrites of commercial tycoon and strategy simulations from the 1990s — inherit a rendering contract written for a 486 with a VGA framebuffer, and must honour it while running on hardware five orders of magnitude faster. The original games composited palettized 8-bit sprites into a linear framebuffer with hand-tuned assembly. The reimplementations keep that model, because the game logic, the art assets, and in several cases the on-disk save format all depend on the exact pixel semantics of palette-index compositing. What makes them interesting is what happens next: each project has had to decide whether the GPU can be brought into a pipeline that was never designed to accommodate one.

This chapter examines that decision. It is not a survey of 2D game engines in general. An earlier research pass for this chapter considered a broader sweep of open-source simulation games — Simutrans, FreeCol, Unciv, OpenMW and others — and found most of that material too thin to justify sustained treatment: these are, from a rendering standpoint, framework consumers. They call into SDL2, Java2D, or LibGDX and inherit whatever that framework does. There is no distinctive architecture to describe, and writing full sections about them would mean padding. So this chapter narrows deliberately to the projects that contain real, distinctive GPU-architecture engineering, and relegates the framework consumers to a short comparison table in §7 where they belong.

Four codebases carry the chapter. **OpenTTD** and **OpenLoco** demonstrate what turns out to be a convergent architecture: the CPU composites the entire frame and the GPU is used only to get that finished buffer onto the screen. **OpenRCT2** is the one project in this niche that moved sprite compositing itself onto the GPU, and its OpenGL renderer — instanced draws over a texture array of palette-index atlases — is the chapter's central technical case study. **OpenRA** provides the contrast that makes OpenRCT2's design legible: it is a conventional sampler-bound sprite batcher, and its constraints are precisely the ones OpenRCT2's array-texture design sidesteps. A closing section observes that no project surveyed uses Vulkan, and treats that as a scoping observation about the problem domain rather than a gap to be filled.

---

## Table of Contents

- [1. The Shared Starting Point: Palettized CPU Compositing](#1-the-shared-starting-point-palettized-cpu-compositing)
  - [1.1 Why These Codebases Are Still 8-Bit Palettized](#11-why-these-codebases-are-still-8-bit-palettized)
  - [1.2 OpenTTD's Blitter Matrix](#12-openttds-blitter-matrix)
  - [1.3 Inside a Blitter Inner Loop](#13-inside-a-blitter-inner-loop)
- [2. OpenTTD: The Presentation-Only OpenGL Backend](#2-openttd-the-presentation-only-opengl-backend)
  - [2.1 What "Presentation-Only" Means Precisely](#21-what-presentation-only-means-precisely)
  - [2.2 The Video Buffer Handoff: PBOs, Persistent Mapping, Fences](#22-the-video-buffer-handoff-pbos-persistent-mapping-fences)
  - [2.3 Paint(): The Entire Game World as One Quad](#23-paint-the-entire-game-world-as-one-quad)
  - [2.4 Three Shader Programs, One Palette Problem](#24-three-shader-programs-one-palette-problem)
  - [2.5 The Single Exception: The Hardware Mouse Cursor](#25-the-single-exception-the-hardware-mouse-cursor)
- [3. OpenRCT2: Instanced Sprite Compositing on the GPU](#3-openrct2-instanced-sprite-compositing-on-the-gpu)
  - [3.1 Three Renderer Modes](#31-three-renderer-modes)
  - [3.2 The Per-Sprite Command Struct](#32-the-per-sprite-command-struct)
  - [3.3 Instancing: Four Vertices, Thousands of Sprites, One Draw Call](#33-instancing-four-vertices-thousands-of-sprites-one-draw-call)
  - [3.4 The Texture Array of Atlases](#34-the-texture-array-of-atlases)
  - [3.5 GL_R8UI: Palette Indices Live on the GPU](#35-gl_r8ui-palette-indices-live-on-the-gpu)
  - [3.6 The Real Ceiling: Array Layers, Not Sampler Units](#36-the-real-ceiling-array-layers-not-sampler-units)
  - [3.7 Transparency by Depth Peeling](#37-transparency-by-depth-peeling)
  - [3.8 Why It Is Still Labelled Experimental](#38-why-it-is-still-labelled-experimental)
- [4. OpenLoco: Hardware Presentation Without a GPU Renderer](#4-openloco-hardware-presentation-without-a-gpu-renderer)
- [5. OpenRA: Sampler-Bounded Sprite Batching](#5-openra-sampler-bounded-sprite-batching)
- [6. Modding APIs: Three Embedded-Language Choices](#6-modding-apis-three-embedded-language-choices)
  - [6.1 OpenTTD: NewGRF Bytecode Plus Squirrel](#61-openttd-newgrf-bytecode-plus-squirrel)
  - [6.2 OpenRCT2: Duktape, JavaScript, and Shipped TypeScript Types](#62-openrct2-duktape-javascript-and-shipped-typescript-types)
  - [6.3 OpenRA: Lua Missions and the Mod SDK](#63-openra-lua-missions-and-the-mod-sdk)
- [7. Brief Survey: The Framework Consumers](#7-brief-survey-the-framework-consumers)
- [8. The Absence of Vulkan](#8-the-absence-of-vulkan)
- [Integrations](#integrations)
- [References](#references)

---

## 1. The Shared Starting Point: Palettized CPU Compositing

### 1.1 Why These Codebases Are Still 8-Bit Palettized

Every project in this chapter descends from a game whose art was authored as indices into a 256-entry palette. That is not merely a storage format; in these games the palette index is semantically load-bearing:

- **Company and player recolouring** is implemented as a palette remap. A vehicle owned by one company and the same vehicle owned by another are the same sprite data, drawn through different remap tables. Converting the art to RGBA up front would require baking every recolouring variant, multiplying the asset set by the number of players.
- **Index 0 is transparency**, not an alpha channel. The blitter tests `if (colour != 0)` and skips the write.
- **Animated palette entries** produce the shimmering water, flashing lights, and rotating beacons in the original games. These are not animated sprites; they are palette entries whose RGB values are rewritten each frame while the sprite data on screen never changes.
- **Reserved index ranges** encode effects such as darkening for tunnels or shadow, again applied by table lookup rather than by arithmetic blending.

A renderer that wants to move this work to the GPU therefore cannot simply upload RGBA textures. It must either resolve the palette on the CPU before upload — which reintroduces per-frame CPU work proportional to pixels touched — or carry palette indices all the way into the fragment shader and resolve them there. OpenRCT2 takes the second route, and §3.5 examines the consequences.

### 1.2 OpenTTD's Blitter Matrix

OpenTTD's answer to the palette question is to make the pixel format a runtime-selectable strategy. The `src/blitter/` directory contains a factory-dispatched family of blitters:

| Blitter family | Members | Notes |
| --- | --- | --- |
| 8 bpp | `8bpp_simple`, `8bpp_optimized`, `8bpp_base` | Palette indices written directly to an 8-bit framebuffer |
| 32 bpp | `32bpp_simple`, `32bpp_optimized`, `32bpp_base` | Palette resolved to RGBA during blit |
| 32 bpp SIMD | `32bpp_sse2`, `32bpp_ssse3`, `32bpp_sse4` | Hand-vectorized specializations sharing `32bpp_sse_func.hpp` |
| 32 bpp animated | `32bpp_anim`, `32bpp_anim_sse2`, `32bpp_anim_sse4` | Maintain a parallel animation buffer for palette-animated pixels |
| Other | `40bpp_anim`, `null` | Extended-depth and headless variants |

[Source: OpenTTD `src/blitter/`](https://github.com/OpenTTD/OpenTTD/tree/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/blitter)

The `32bpp_anim` family is the architecturally interesting one. Because palette animation requires knowing which screen pixels came from animated palette entries, these blitters write a second buffer alongside the colour buffer recording the palette index of each pixel. The renderer can then re-resolve only the animated entries without recompositing the scene. This "animation buffer" concept reappears in §2.4 as a distinct GPU shader path, because the OpenGL backend must know whether the active blitter needs it.

The SIMD variants are pure throughput optimizations of the same semantics — they exist because sprite blitting in a busy view is memory-bandwidth-bound and the inner loop is trivially vectorizable. Their presence is a good indicator of where the real cost sits in a CPU sprite pipeline: not in per-sprite setup, but in the per-pixel inner loop.

### 1.3 Inside a Blitter Inner Loop

The simplest blitter shows the contract every other variant implements. Note the per-pixel `switch` on blitter mode, the zoom handled by striding the source pointer, and index 0 as transparency:

```cpp
// OpenTTD src/blitter/8bpp_simple.cpp
void Blitter_8bppSimple::Draw(Blitter::BlitterParams *bp, BlitterMode mode, ZoomLevel zoom)
{
	const uint8_t *src, *src_line;
	uint8_t *dst, *dst_line;

	/* Find where to start reading in the source sprite */
	src_line = (const uint8_t *)bp->sprite + (bp->skip_top * bp->sprite_width + bp->skip_left) * ScaleByZoom(1, zoom);
	dst_line = (uint8_t *)bp->dst + bp->top * bp->pitch + bp->left;

	for (int y = 0; y < bp->height; y++) {
		dst = dst_line;
		dst_line += bp->pitch;

		src = src_line;
		src_line += bp->sprite_width * ScaleByZoom(1, zoom);

		for (int x = 0; x < bp->width; x++) {
			uint colour = 0;

			switch (mode) {
				case BlitterMode::ColourRemap:
				case BlitterMode::CrashRemap:
					colour = bp->remap[*src];
					break;

				case BlitterMode::Transparent:
				case BlitterMode::TransparentRemap:
					if (*src != 0) colour = bp->remap[*dst];
					break;

				case BlitterMode::BlackRemap:
					if (*src != 0) *dst = 0;
					break;

				default:
					colour = *src;
					break;
			}
			if (colour != 0) *dst = colour;
			dst++;
			src += ScaleByZoom(1, zoom);
		}
	}
}
```

[Source: OpenTTD `src/blitter/8bpp_simple.cpp`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/blitter/8bpp_simple.cpp)

Three details are worth extracting, because they explain why porting this to a GPU is not mechanical.

First, `BlitterMode::Transparent` reads the *destination* pixel (`bp->remap[*dst]`) and remaps it. This is a read-modify-write against the framebuffer with a table lookup in the middle — a dependent read that a naive GPU port cannot express with fixed-function blending. OpenRCT2 handles the equivalent case with the depth-peeling machinery in §3.7.

Second, zoom is implemented by striding the source pointer with `ScaleByZoom(1, zoom)` rather than by filtering. These games nearest-sample by design; the pixel-art aesthetic depends on it. Every GPU path in this chapter accordingly sets `GL_NEAREST` filtering, and the presentation paths in §2 and §4 do the same.

Third, `bp->remap` is a per-draw-call table. On the CPU that is a pointer swap. On the GPU it becomes per-instance state, which is why OpenRCT2's command struct in §3.2 carries an `ivec3 palettes` field.

---

## 2. OpenTTD: The Presentation-Only OpenGL Backend

OpenTTD ships an OpenGL video driver (`sdl2_opengl_v`) alongside the plain SDL2 driver. It is easy to assume from the name that this driver renders the game with the GPU. It does not, and understanding exactly what it does instead is the clearest possible illustration of the architecture that §4 shows OpenLoco converging on independently.

### 2.1 What "Presentation-Only" Means Precisely

The OpenGL backend does not participate in sprite compositing at all. The sequence each frame is:

1. The video driver asks the backend for a pointer to a writable pixel buffer.
2. The active CPU blitter — whichever of the §1.2 family is configured — composites the entire game world, all windows, all text, and all sprites into that buffer, exactly as it would with no GPU present.
3. The backend uploads the buffer to a single 2D texture.
4. The backend draws one fullscreen quad sampling that texture.

The GPU's contribution is steps 3 and 4: getting a finished CPU-composited image onto the screen, with scaling, and in some configurations resolving the palette. It performs zero per-sprite work. The class declaration reflects this directly — the members are a pixel-buffer object, a video texture, shader programs for the buffer formats, and a single fullscreen quad:

```c
// OpenTTD src/video/opengl.h (abridged — members relevant to presentation)
class OpenGLBackend : public SpriteEncoder {
private:
	GLuint vid_pbo;      ///< Pixel buffer object storing the memory used for the video driver to draw to.
	GLuint vid_texture;  ///< Texture handle for the video buffer texture.
	GLuint vid_program;  ///< Shader program for rendering a RGBA video buffer.
	GLuint pal_program;  ///< Shader program for rendering a paletted video buffer.
	GLuint vao_quad;     ///< Vertex array object storing the rendering state for the fullscreen quad.
	GLuint vbo_quad;     ///< Vertex buffer with a fullscreen quad.
	GLuint pal_texture;  ///< Palette lookup texture.

	GLuint anim_pbo;     ///< Pixel buffer object for the animation buffer.
	GLuint anim_texture; ///< Texture handle for the animation buffer texture.
	...
};
```

[Source: OpenTTD `src/video/opengl.h`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/video/opengl.h)

Why stop here? Because the alternative is the OpenRCT2 project described in §3 — a multi-year effort that is still labelled experimental a decade after it began. OpenTTD's game view is a tile-based isometric scene where a busy screen composites a large number of small sprites, but the game is not frame-rate-bound on a modern CPU. The gain from GPU sprite compositing is speculative; the gain from GPU presentation is concrete and small in scope: hardware scaling, avoidance of a CPU blit-to-window, and cooperation with the compositor's buffer flow. The project took the concrete win.

### 2.2 The Video Buffer Handoff: PBOs, Persistent Mapping, Fences

The one genuinely GPU-technical part of the OpenTTD backend is how the CPU-written pixels reach the GPU without a stall. The backend exposes a buffer-acquire/release pair rather than a blit call:

```c
// OpenTTD src/video/opengl.h — the video driver's interface to the backend
void *GetVideoBuffer();
void *GetAnimBuffer();
void ReleaseVideoBuffer(const Rect &update_rect);
void ReleaseAnimBuffer(const Rect &update_rect);
void Paint();
```

[Source: OpenTTD `src/video/opengl.h`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/video/opengl.h)

The memory the blitter writes into is the mapping of a pixel buffer object. Where the driver supports it, the mapping is *persistent* — the buffer stays mapped across frames instead of being mapped and unmapped each time — and correctness is maintained with sync objects rather than with implicit driver synchronization:

```c
// OpenTTD src/video/opengl.h
bool persistent_mapping_supported; ///< Persistent pixel buffer mapping supported.
GLsync sync_vid_mapping;           ///< Sync object for the persistently mapped video buffer.
GLsync sync_anim_mapping;          ///< Sync object for the persistently mapped animation buffer.
```

[Source: OpenTTD `src/video/opengl.h`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/video/opengl.h)

This is a textbook CPU-producer/GPU-consumer pattern, and it is the same pattern a software rasterizer needs when presenting through a GPU-backed window system. The `update_rect` passed to `ReleaseVideoBuffer` lets the backend upload only the damaged region — the CPU compositor already tracks dirty rectangles for its own benefit, and that information is reused to bound the texture upload.

### 2.3 Paint(): The Entire Game World as One Quad

The decisive evidence for the presentation-only characterization is the paint function itself. The entire frame — every tile, vehicle, window, and glyph — is delivered by a four-vertex triangle strip:

```c
// OpenTTD src/video/opengl.cpp
void OpenGLBackend::Paint()
{
	_glClear(GL_COLOR_BUFFER_BIT);
	_glDisable(GL_BLEND);

	/* Blit video buffer to screen. */
	_glActiveTexture(GL_TEXTURE0);
	_glBindTexture(GL_TEXTURE_2D, this->vid_texture);
	_glActiveTexture(GL_TEXTURE1);
	_glBindTexture(GL_TEXTURE_1D, this->pal_texture);
	/* Is the blitter relying on a separate animation buffer? */
	if (BlitterFactory::GetCurrentBlitter()->NeedsAnimationBuffer()) {
		_glActiveTexture(GL_TEXTURE2);
		_glBindTexture(GL_TEXTURE_2D, this->anim_texture);
		_glUseProgram(this->remap_program);
		_glUniform4f(this->remap_sprite_loc, 0.0f, 0.0f, 1.0f, 1.0f);
		_glUniform2f(this->remap_screen_loc, 1.0f, 1.0f);
		_glUniform1f(this->remap_zoom_loc, 0);
		_glUniform1i(this->remap_rgb_loc, 1);
	} else {
		_glUseProgram(BlitterFactory::GetCurrentBlitter()->GetScreenDepth() == 8 ? this->pal_program : this->vid_program);
	}
	_glBindVertexArray(this->vao_quad);
	_glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

	_glEnable(GL_BLEND);
}
```

[Source: OpenTTD `src/video/opengl.cpp`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/video/opengl.cpp)

One `glDrawArrays`, four vertices, per frame. There is no sprite geometry, no batching, no atlas, and no instancing — because there is nothing to batch. The scene was already flattened to pixels before this function was entered.

### 2.4 Three Shader Programs, One Palette Problem

`Paint()` chooses among three programs, and the choice is driven entirely by what the CPU blitter produced. This is where the palette semantics of §1.1 surface in the GPU path:

- **`vid_program`** — the active blitter wrote 32-bit RGBA. The shader is a straight textured blit.
- **`pal_program`** — the active blitter wrote 8-bit palette indices (`GetScreenDepth() == 8`). The video texture holds indices; the shader resolves them through `pal_texture`, a `GL_TEXTURE_1D` palette lookup bound to unit 1. The palette resolution that the 32 bpp blitters do per pixel on the CPU is here done per fragment on the GPU.
- **`remap_program`** — the active blitter needs an animation buffer (`NeedsAnimationBuffer()`, true for the `32bpp_anim` family). Both the colour texture and the animation texture are bound, and the shader combines them, applying the current palette to animated pixels while passing through static RGBA.

The third path is the payoff of the animation-buffer design from §1.2. Palette animation on the CPU otherwise means rewriting affected pixels every frame; here the CPU writes the animation buffer once when the scene changes, and per-frame palette animation becomes a uniform update plus a fragment-shader lookup. This is a genuine GPU offload — just of palette resolution, not of compositing.

### 2.5 The Single Exception: The Hardware Mouse Cursor

One qualification is necessary, because the backend's header contains machinery that looks like a GPU sprite renderer. `OpenGLBackend` derives from `SpriteEncoder`, overrides `Is32BppSupported()` and `Encode()`, and owns a `sprite_program`, a `remap_program`, an `OpenGLSprite` type, and an LRU cursor cache:

```c
// OpenTTD src/video/opengl.h (abridged)
class OpenGLBackend : public SpriteEncoder {
	LRUCache<SpriteID, std::unique_ptr<OpenGLSprite>> cursor_cache; ///< Cache of encoded cursor sprites.
	GLuint sprite_program;  ///< Shader program for blitting sprites.
	...
	void DrawMouseCursor();
	void PopulateCursorCache();
	void RenderOglSprite(OpenGLSprite *gl_sprite, PaletteID pal, int x, int y, ZoomLevel zoom);

	bool Is32BppSupported() override { return true; }
	Sprite *Encode(SpriteType sprite_type, const SpriteLoader::SpriteCollection &sprite, SpriteAllocator &allocator) override;
};
```

[Source: OpenTTD `src/video/opengl.h`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/video/opengl.h)

So a GPU sprite path does exist. Tracing its call sites resolves the apparent contradiction: in `src/video/opengl.cpp`, `RenderOglSprite` is defined once and called from exactly one place — inside `DrawMouseCursor()`. The mouse cursor is the sole sprite that OpenTTD's OpenGL backend composites on the GPU, and it is drawn that way for latency reasons: a hardware cursor can be updated without recompositing or re-uploading the frame beneath it.

The precise statement, then, is that OpenTTD's OpenGL driver is presentation-only *for the game world*, with the hardware mouse cursor as the single GPU-composited sprite. That nuance matters for anyone reading the header and expecting a sprite renderer.

---

## 3. OpenRCT2: Instanced Sprite Compositing on the GPU

OpenRCT2 is the one project in this niche that moved sprite compositing itself onto the GPU. A theme park at full zoom-out is a dense isometric scene — terrain tiles, path tiles, ride track segments, scenery, and a large population of individually-animated guests, each of which is a separate sprite with its own palette remap. Compositing this on the CPU is exactly the workload §1.3's inner loop is bad at, and it is the workload that motivated a GPU renderer.

The engineering problem is stated compactly: **thousands of sprites per frame, each drawn from a different source image, each with its own palette remap and clipping rectangle, and all of them palette-indexed rather than RGBA.** A naive port issues one draw call per sprite with one texture bind per sprite, which trades CPU pixel work for CPU driver overhead and generally loses. The rest of this section is how OpenRCT2 avoids that.

### 3.1 Three Renderer Modes

OpenRCT2 exposes three drawing configurations, and the middle one is the OpenTTD architecture of §2:

| Mode | Sprite compositing | GPU role |
| --- | --- | --- |
| Software | CPU | None (CPU blit to window) |
| Software with hardware display | CPU | Upload finished buffer, present with scaling |
| OpenGL (experimental, opt-in) | **GPU** | Full sprite compositing |

Only the third mode is a GPU sprite renderer. The second exists for the same reasons OpenTTD's does, and its presence in the same codebase as a full GPU renderer is telling: even where the GPU path exists, the presentation-only path is retained as the safer default.

### 3.2 The Per-Sprite Command Struct

The GPU renderer's core data structure is a per-instance command. Every sprite the game asks to draw becomes one of these structs appended to a CPU-side batch:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/DrawCommands.h
// Per-instance data for images
struct DrawRectCommand
{
    ivec4 clip;
    GLint texColourAtlas;
    vec4 texColourBounds;
    GLint texMaskAtlas;
    vec4 texMaskBounds;
    ivec3 palettes;
    GLint flags;
    GLuint colour;
    ivec4 bounds;
    GLint depth;
    GLfloat zoom;

    enum
    {
        FLAG_NO_TEXTURE = (1u << 2u),
        FLAG_MASK = (1u << 3u),
        FLAG_CROSS_HATCH = (1u << 4u),
        FLAG_TTF_TEXT = (1u << 5u),
        // bits 8 to 16 used to store hinting threshold.
        FLAG_TTF_HINTING_THRESHOLD_MASK = 0xff00
    };
};
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/DrawCommands.h`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/DrawCommands.h)

Read this struct as a translation table from the CPU blitter's arguments to per-instance vertex attributes. `bounds` replaces the destination pointer arithmetic. `clip` replaces the caller-imposed clipping that the CPU path handled by adjusting `skip_left`/`skip_top`. `palettes` carries the three remap tables that were `bp->remap`. `zoom` replaces `ScaleByZoom`. `flags` encodes the blitter-mode `switch`. And critically, **`texColourAtlas` and `texMaskAtlas` are integers identifying which atlas the sprite lives in** — the field that makes the whole design work, for reasons §3.4 develops.

The batch container is a small but instructive optimization. It is a grow-only vector with a separate logical size, so that clearing between frames is O(1) and never releases capacity:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/DrawCommands.h
template<typename T>
class CommandBatch
{
private:
    std::vector<T> _instances;
    size_t _numInstances;

public:
    void clear() { _numInstances = 0; }

    T& allocate()
    {
        if (_numInstances + 1 > _instances.size())
        {
            _instances.resize((_numInstances + 1) << 1);
        }
        return _instances[_numInstances++];
    }

    [[nodiscard]] size_t size() const { return _numInstances; }
    const T* data() const { return _instances.data(); }
};
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/DrawCommands.h`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/DrawCommands.h)

The doubling growth (`(_numInstances + 1) << 1`) means the batch reaches steady-state capacity within a few frames and then performs no allocation at all. `allocate()` returns a reference for in-place construction, so the drawing context fills fields directly in the batch rather than building a temporary and copying it — which matters when the call site executes thousands of times per frame.

The drawing context maintains three such batches, and the split is significant:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp
struct
{
    LineCommandBatch lines;
    RectCommandBatch rects;
    RectCommandBatch transparent;
} _commandBuffers;
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp)

Opaque rectangles and transparent rectangles go into separate batches because they require fundamentally different draw strategies — §3.7.

### 3.3 Instancing: Four Vertices, Thousands of Sprites, One Draw Call

The mechanism that turns a batch into a single draw call is instanced rendering. Two vertex buffers are bound to one vertex array object: a static four-vertex buffer describing a unit quad, and a streaming buffer holding the `DrawRectCommand` array.

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp
constexpr size_t kInitialInstancesBufferSize = 32768;

DrawRectShader::DrawRectShader()
    : OpenGLShaderProgram("drawrect")
    , _maxInstancesBufferSize(kInitialInstancesBufferSize)
{
    GetLocations();

    glCall(glGenBuffers, 1, &_vbo);
    glCall(glGenBuffers, 1, &_vboInstances);
    glCall(glGenVertexArrays, 1, &_vao);

    glCall(glBindBuffer, GL_ARRAY_BUFFER, _vbo);
    glCall(glBufferData, GL_ARRAY_BUFFER, sizeof(kVertexData), kVertexData, GL_STATIC_DRAW);

    glCall(glBindVertexArray, _vao);
    /* ... per-vertex attributes vVertMat, vVertVec from _vbo ... */

    glCall(glBindBuffer, GL_ARRAY_BUFFER, _vboInstances);
    glCall(glBufferData, GL_ARRAY_BUFFER, sizeof(DrawRectCommand) * kInitialInstancesBufferSize, nullptr, GL_STREAM_DRAW);

    glCall(glVertexAttribIPointer, vClip, 4, GL_INT, glSizeOf<DrawRectCommand>(),
        reinterpret_cast<void*>(offsetof(DrawRectCommand, clip)));
    glCall(glVertexAttribIPointer, vTexColourAtlas, 1, GL_INT, glSizeOf<DrawRectCommand>(),
        reinterpret_cast<void*>(offsetof(DrawRectCommand, texColourAtlas)));
    /* ... one glVertexAttrib*Pointer per DrawRectCommand field ... */

    glCall(glVertexAttribDivisor, vClip, 1);
    glCall(glVertexAttribDivisor, vTexColourAtlas, 1);
    glCall(glVertexAttribDivisor, vTexColourCoords, 1);
    glCall(glVertexAttribDivisor, vTexMaskAtlas, 1);
    glCall(glVertexAttribDivisor, vTexMaskCoords, 1);
    glCall(glVertexAttribDivisor, vPalettes, 1);
    glCall(glVertexAttribDivisor, vFlags, 1);
    glCall(glVertexAttribDivisor, vColour, 1);
    glCall(glVertexAttribDivisor, vBounds, 1);
    glCall(glVertexAttribDivisor, vDepth, 1);
    glCall(glVertexAttribDivisor, vZoom, 1);

    Use();
    glCall(glUniform1i, uTexture, 0);
    glCall(glUniform1i, uPaletteTex, 1);
    glCall(glUniform1i, uPeelingTex, 2);
    glCall(glUniform1i, uPeeling, 0);
}
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp)

`glVertexAttribDivisor(attr, 1)` is the pivotal call. It declares that the attribute advances once per *instance* rather than once per vertex. Every one of the eleven `DrawRectCommand` fields gets a divisor of 1; the two quad attributes (`vVertMat`, `vVertVec`) keep the default divisor of 0 and advance per vertex. The GPU therefore reads four quad vertices repeatedly while stepping through the command array once per sprite.

Upload and draw are correspondingly minimal:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp
void DrawRectShader::SetInstances(const RectCommandBatch& instances)
{
    glCall(glBindVertexArray, _vao);
    glCall(glBindBuffer, GL_ARRAY_BUFFER, _vboInstances);

    if (instances.size() > _maxInstancesBufferSize)
    {
        glCall(glBufferData, GL_ARRAY_BUFFER, sizeof(DrawRectCommand) * instances.size(), instances.data(), GL_STREAM_DRAW);
        _maxInstancesBufferSize = instances.size();
    }
    else
    {
        glCall(glBufferSubData, GL_ARRAY_BUFFER, 0, sizeof(DrawRectCommand) * instances.size(), instances.data());
    }

    _instanceCount = static_cast<GLsizei>(instances.size());
}

void DrawRectShader::DrawInstances()
{
    glCall(glBindVertexArray, _vao);
    glCall(glDrawArraysInstanced, GL_TRIANGLE_STRIP, 0, 4, _instanceCount);
}
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp)

The buffer-sizing branch reallocates with `glBufferData` only when the batch exceeds the current capacity, and otherwise respecifies with `glBufferSubData`. Combined with `GL_STREAM_DRAW` and the 32768-instance initial allocation, steady-state frames perform one `glBufferSubData` and one `glDrawArraysInstanced`.

The vertex shader consumes these attributes and — importantly — forwards the per-instance values with the `flat` qualifier, since they are constant across the instance's four vertices and must not be interpolated:

```glsl
// OpenRCT2 data/shaders/drawrect.vert
#version 330 core

// Allows for about 8 million draws per frame
const float DEPTH_INCREMENT = 1.0 / float(1u << 22u);

uniform ivec2 uScreenSize;

in ivec4 vClip;
in int   vTexColourAtlas;
in vec4  vTexColourCoords;
in int   vTexMaskAtlas;
in vec4  vTexMaskCoords;
in ivec3 vPalettes;
in int   vFlags;
in uint  vColour;
in ivec4 vBounds;
in int   vDepth;
in float vZoom;

in mat4x2 vVertMat;
in vec2   vVertVec;

flat out int   fTexColourAtlas;
flat out int   fTexMaskAtlas;
flat out vec3  fPalettes;
out vec3       fPeelPos;
/* ... */

void main()
{
    // Clamp position by vClip, correcting interpolated values for the clipping
    vec2 m = clamp(
        ((vVertMat * vec4(vClip)) - (vVertMat * vec4(vBounds))) / vec2(vBounds.zw - vBounds.xy) + vVertVec, 0.0, 1.0);
    vec2 pos = mix(vec2(vBounds.xy), vec2(vBounds.zw), m);

    fTexColourAtlas = vTexColourAtlas;
    fTexMaskAtlas = vTexMaskAtlas;

    // Transform screen coordinates to texture coordinates
    float depth = 1.0 - (float(vDepth) + 1.0) * DEPTH_INCREMENT;
    pos = pos / vec2(uScreenSize);
    pos.y = pos.y * -1.0 + 1.0;
    fPeelPos = vec3(pos, depth * 0.5 + 0.5);

    // Transform texture coordinates to viewport coordinates
    pos = pos * 2.0 - 1.0;
    gl_Position = vec4(pos, depth, 1.0);
}
```

[Source: OpenRCT2 `data/shaders/drawrect.vert`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/data/shaders/drawrect.vert)

The `vVertMat`/`vVertVec` construction deserves note. Rather than passing quad corner offsets and computing a rectangle, the static vertex data encodes a small matrix and vector per corner such that a single `mix` between `vBounds.xy` and `vBounds.zw` yields the correct clipped corner position. Clipping is thus folded into vertex position computation with no branching — the `clamp` handles the case where the clip rectangle cuts into the sprite, and the same expression adjusts texture coordinates consistently. This replaces the CPU blitter's `skip_left`/`skip_top` pointer arithmetic.

The depth encoding is also worth extracting. `DEPTH_INCREMENT = 1.0 / 2^22` allocates a distinct depth value per draw order index, giving roughly 8 million orderable draws per frame in a 2D scene that has no natural depth. This converts the CPU renderer's implicit "draw order is submission order" into an explicit depth value, which is what allows the entire batch to be submitted in one unordered instanced draw and still resolve correctly through the depth test.

### 3.4 The Texture Array of Atlases

Batching sprites into one draw call is only useful if the shader can reach every sprite's pixels during that draw. Since each sprite is a different source image, the naive approach — one texture per sprite, bound before its draw — is exactly what instancing must eliminate. Atlasing is the classical answer: pack many sprites into one large texture and index them with texture coordinates. But OpenRCT2's sprite set is far too large for a single atlas, so a single atlas merely converts a per-sprite bind into a per-atlas bind.

OpenRCT2's answer is to make all atlases layers of one `GL_TEXTURE_2D_ARRAY`. The design intent is stated directly in the header:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/TextureCache.h

// This is the maximum width and height of each atlas, basically the
// granularity at which new atlases are allocated (2048 -> 4 MB of VRAM)
constexpr int32_t kTextureCacheMaxAtlasSize = 2048;

// Pixel dimensions of smallest supported slots in texture atlases
// Must be a power of 2!
constexpr int32_t kTextureCacheSmallestSlot = 32;

// Location of an image (texture atlas index, slot and normalized coordinates)
struct AtlasTextureInfo : public BasicTextureInfo
{
    GLuint slot;
    ivec4 bounds;
    ImageIndex image;
};

// Represents a texture atlas that images of a given maximum size can be allocated from
// Atlases are all stored in the same 2D texture array, occupying the specified index
// Slots in atlases are always squares.
class Atlas final
{
private:
    GLuint _index = 0;
    int32_t _imageSize = 0;
    int32_t _atlasWidth = 0;
    int32_t _atlasHeight = 0;
    std::vector<GLuint> _freeSlots;

    int32_t _cols = 0;
    int32_t _rows = 0;
    ...
};
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/TextureCache.h`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.h)

Each `Atlas` is one layer of the array texture and serves exactly one power-of-two size class. Allocation within a layer is a free-slot list rather than a rectangle packer:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/TextureCache.h
void Initialise(int32_t atlasWidth, int32_t atlasHeight)
{
    _atlasWidth = atlasWidth;
    _atlasHeight = atlasHeight;

    _cols = std::max(1, _atlasWidth / _imageSize);
    _rows = std::max(1, _atlasHeight / _imageSize);

    _freeSlots.resize(_cols * _rows);
    for (size_t i = 0; i < _freeSlots.size(); i++)
    {
        _freeSlots[i] = static_cast<GLuint>(i);
    }
}

AtlasTextureInfo Allocate(int32_t actualWidth, int32_t actualHeight)
{
    assert(!_freeSlots.empty());

    GLuint slot = _freeSlots.back();
    _freeSlots.pop_back();
    /* ... compute bounds and normalized coords ... */
}

void Free(const AtlasTextureInfo& info)
{
    assert(_index == info.index);
    _freeSlots.push_back(info.slot);
}

static int32_t CalculateImageSizeOrder(int32_t actualWidth, int32_t actualHeight)
{
    int32_t actualSize = std::max(actualWidth, actualHeight);

    if (actualSize < kTextureCacheSmallestSlot)
    {
        actualSize = kTextureCacheSmallestSlot;
    }

    return static_cast<int32_t>(ceil(log2f(static_cast<float>(actualSize))));
}
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/TextureCache.h`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.h)

This is a size-class slab allocator rather than a bin packer, and the tradeoff is deliberate. A general rectangle packer achieves better VRAM utilization but has no O(1) free operation and fragments as sprites are evicted and reloaded. Rounding each image up to the next power of two via `ceil(log2f(...))` and grouping equal size classes into dedicated layers makes both allocate and free a vector push/pop, at the cost of internal fragmentation — a 33×33 sprite occupies a 64×64 slot. For a renderer that streams sprites in and out as the player scrolls and zooms, predictable O(1) churn is worth more than packing density.

The payoff is in the flush path. Compositing the entire opaque scene binds exactly two textures:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp
void OpenGLDrawingContext::FlushRectangles()
{
    if (_commandBuffers.rects.empty())
        return;

    OpenGLAPI::SetTexture(0, GL_TEXTURE_2D_ARRAY, _textureCache->GetAtlasesTexture());
    OpenGLAPI::SetTexture(1, GL_TEXTURE_2D, _textureCache->GetPaletteTexture());

    _drawRectShader->Use();
    _drawRectShader->SetInstances(_commandBuffers.rects);
    _drawRectShader->DrawInstances();

    _commandBuffers.rects.clear();
}
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp)

Two texture binds, one buffer upload, one draw call — for every opaque sprite on screen, drawn from arbitrarily many atlases. This is the architectural claim of the whole renderer, and it holds because the per-instance `texColourAtlas` integer selects the array layer inside the shader rather than requiring a bind outside it.

### 3.5 GL_R8UI: Palette Indices Live on the GPU

The most distinctive property of this renderer is easy to miss: the atlas array does not store colours. It stores palette indices, in a single-channel unsigned-integer format.

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp (in EnlargeAtlasesTexture)
glCall(
    glTexImage3D, GL_TEXTURE_2D_ARRAY, 0, GL_R8UI, _atlasesTextureDimensions, _atlasesTextureDimensions,
    _atlasesTextureCapacity, 0, GL_RED_INTEGER, GL_UNSIGNED_BYTE, nullptr);
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp)

`GL_R8UI` with `GL_RED_INTEGER`: one byte per pixel, delivered to the shader as an unsigned integer with no normalization and no filtering. The fragment shader accordingly declares integer samplers:

```glsl
// OpenRCT2 data/shaders/drawrect.frag (abridged)
uniform usampler2DArray uTexture;
uniform usampler2D      uPaletteTex;
uniform sampler2D       uPeelingTex;
```

[Source: OpenRCT2 `data/shaders/drawrect.frag`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/data/shaders/drawrect.frag)

and resolves colour in two steps — sample the index from the atlas layer, then remap it through the palette texture:

```glsl
// OpenRCT2 data/shaders/drawrect.frag (abridged)
texel = texture(uTexture, vec3(colourU, colourV, fTexColourAtlas)).r;
/* ... */
texel = texture(uPaletteTex, vec2(texel + 0xC5u, fPalettes.z) / 256.0f).r;
/* ... */
uint mask = texture(uTexture, vec3(maskU, maskV, fTexMaskAtlas)).r;
```

[Source: OpenRCT2 `data/shaders/drawrect.frag`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/data/shaders/drawrect.frag)

Note the third argument to the atlas sample: `fTexColourAtlas`, the flat-interpolated per-instance atlas index, used directly as the array layer coordinate. This is the single line that makes cross-atlas batching work.

The consequences of the `GL_R8UI` choice are worth spelling out, because this is 8-bit palettized compositing executed on a modern GPU:

- **VRAM is one quarter of the RGBA equivalent.** The header's comment budgets a 2048×2048 layer at 4 MB, which is exactly 2048² × 1 byte.
- **Palette remapping is per-fragment and free-ish.** The `bp->remap[*src]` table lookup from §1.3 becomes a dependent texture fetch. Because the palette lives in a texture rather than in uniforms, an arbitrary number of distinct remaps can be in flight within a single batch — the per-instance `palettes` field selects which row of the palette texture to use. This is what allows thousands of differently-recoloured guests to composite in one draw call.
- **Filtering must be nearest, and is.** Integer textures cannot be linearly filtered, and interpolating palette *indices* would be meaningless anyway — the average of index 5 and index 9 is not a colour between them. The cache sets `GL_NEAREST` for both minification and magnification, which happens to be exactly the pixel-art requirement noted in §1.3. The format constraint and the aesthetic constraint agree.
- **The CPU never resolves the palette.** Unlike OpenTTD's 32 bpp blitters, sprite upload is a direct copy of the source index data.

### 3.6 The Real Ceiling: Array Layers, Not Sampler Units

It is often said that the motivation for a texture-array design is the GPU's limit on simultaneously bound sampler units — a bind-one-texture-per-atlas renderer cannot scale past that limit. As a general statement about OpenGL that is sound: a shader can only reference a bounded number of texture image units, and OpenGL 3.3 core guarantees at least 16 per stage via `GL_MAX_TEXTURE_IMAGE_UNITS`, with desktop drivers commonly reporting more.

It is worth being precise about what that limit does and does not explain, because the distinction is easy to blur. A shader author does not usually discover the hardware maximum at runtime and adapt to it; they pick a fixed number of sampler uniforms at authoring time and declare them in the shader source. That authored number is a *budget*, and it is normally set well below any plausible hardware ceiling. §5 shows OpenRA doing exactly this — its budget is eight, which is below even the GL 3.3 guaranteed minimum of sixteen, and therefore cannot be a hardware limit that OpenRA ran into. What OpenRA demonstrates is the *consequence* of committing to a fixed sampler budget of any size: batches must be flushed whenever they would span more atlases than the budget holds. The array-texture design does not merely raise that budget, it removes the question.

However, this rationale should not be attributed to OpenRCT2's own engineering. Searching the OpenRCT2 source, its shader files, and the pull request that introduced batched instanced drawing turns up no discussion of sampler-unit limits. [PR #4147 "OpenGL sprite batch drawing"](https://github.com/OpenRCT2/OpenRCT2/pull/4147) (merged 2016-07-27) describes the change only as an effort to combine draw operations into batches executed with a single draw call. What the source *does* show is a different and more specific ceiling.

> **Note: needs verification.** A specific figure of roughly 32 sampler units as the motivating hardware constraint for OpenRCT2's texture-array design could not be substantiated in the project's source, shader files, or pull-request history. The general sampler-unit argument above is derived from the OpenGL specification's `GL_MAX_TEXTURE_IMAGE_UNITS` minimum, not from OpenRCT2's stated rationale. Treat the "32" figure as unverified; the limits the code actually queries are given below.

The texture cache queries two limits at initialization and derives its capacity from them:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp (in CreateTextures)
glCall(glGetIntegerv, GL_MAX_TEXTURE_SIZE, &_atlasesTextureDimensions);
/* ... */
glCall(glGetIntegerv, GL_MAX_ARRAY_TEXTURE_LAYERS, &_atlasesTextureIndicesLimit);
if (_atlasesTextureDimensions < _atlasesTextureIndicesLimit)
    _atlasesTextureIndicesLimit = _atlasesTextureDimensions;

glCall(glBindTexture, GL_TEXTURE_2D_ARRAY, _atlasesTexture);
glCall(glTexParameteri, GL_TEXTURE_2D_ARRAY, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glCall(glTexParameteri, GL_TEXTURE_2D_ARRAY, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp)

So the binding constraint on how many atlases can exist is `GL_MAX_ARRAY_TEXTURE_LAYERS`, clamped down to `GL_MAX_TEXTURE_SIZE`. New atlas creation checks against it directly:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp (in AllocateImage)
if (static_cast<int32_t>(_atlases.size()) >= _atlasesTextureIndicesLimit)
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp)

Growing the array reallocates the whole 3D texture with geometric growth, reading back and restoring existing layers:

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp (in EnlargeAtlasesTexture, abridged)
glCall(glGetTexImage, GL_TEXTURE_2D_ARRAY, 0, GL_RED_INTEGER, GL_UNSIGNED_BYTE, oldPixels.data());
/* ... */
_atlasesTextureCapacity = (_atlasesTextureCapacity + 6) << 1uL;
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp)

The `glGetTexImage` readback is the expensive part — a full round trip of the entire atlas array through client memory — and the `(capacity + 6) << 1` growth exists to make it rare. This resize path has been the subject of maintenance work in its own right: [PR #7977 "Fix OpenGL renderer stuttering when loading new textures into atlas."](https://github.com/OpenRCT2/OpenRCT2/pull/7977) (merged 2018-09-07) and [PR #15616 "Fix #15006: Prevent allocating empty texture atlases"](https://github.com/OpenRCT2/OpenRCT2/pull/15616) (merged 2021-10-20) both address costs arising directly from it.

The accurate summary of the design is therefore:

- The fragment shader references **one** array sampler for all sprite pixels, no matter how many atlases exist. Sampler units are a non-issue *by construction* — the flush path in §3.4 binds two textures total.
- The number of atlases is bounded by `GL_MAX_ARRAY_TEXTURE_LAYERS` (clamped to `GL_MAX_TEXTURE_SIZE`), typically a far larger budget than any sampler-unit count.
- Instancing and the array texture are **mutually dependent**, and this is the crux. Instancing alone cannot batch sprites drawn from different atlases, because switching atlas would require a bind between draws. The array texture alone would still leave one draw call per sprite. Together, the per-instance `texColourAtlas` attribute lets a single `glDrawArraysInstanced` cover sprites resident in different atlases. Neither technique achieves the goal without the other.

### 3.7 Transparency by Depth Peeling

The `BlitterMode::Transparent` case in §1.3 read the destination pixel and remapped it. That is order-dependent read-modify-write against the framebuffer, and it cannot be expressed by submitting an unordered instanced batch with a depth test. OpenRCT2 resolves it with depth peeling — repeated passes that each extract the next-nearest transparent layer.

```cpp
// OpenRCT2 src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp
void OpenGLDrawingContext::FlushCommandBuffers()
{
    Guard::Assert(_inDraw == true);

    glCall(glEnable, GL_DEPTH_TEST);
    glCall(glDepthFunc, GL_LESS);

    _swapFramebuffer->BindOpaque();
    _drawRectShader->Use();
    _drawRectShader->DisablePeeling();

    FlushLines();
    FlushRectangles();

    HandleTransparency();
}

void OpenGLDrawingContext::HandleTransparency()
{
    if (_commandBuffers.transparent.empty())
    {
        return;
    }

    _drawRectShader->Use();
    _drawRectShader->SetInstances(_commandBuffers.transparent);

    int32_t max_depth = MaxTransparencyDepth(_commandBuffers.transparent);
    for (int32_t i = 0; i < max_depth; ++i)
    {
        _swapFramebuffer->BindTransparent();

        glCall(glEnable, GL_DEPTH_TEST);
        glCall(glDepthFunc, GL_GREATER);
        _drawRectShader->Use();

        if (i > 0)
        {
            _drawRectShader->EnablePeeling(_swapFramebuffer->GetBackDepthTexture());
        }

        OpenGLAPI::SetTexture(0, GL_TEXTURE_2D_ARRAY, _textureCache->GetAtlasesTexture());
        OpenGLAPI::SetTexture(1, GL_TEXTURE_2D, _textureCache->GetPaletteTexture());

        _drawRectShader->Use();
        _drawRectShader->DrawInstances();
        _swapFramebuffer->ApplyTransparency(
            *_applyTransparencyShader, _textureCache->GetPaletteTexture(), _textureCache->GetBlendPaletteTexture());
    }

    _commandBuffers.transparent.clear();
}
```

[Source: OpenRCT2 `src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp)

The structure repays close reading:

- **Opaque geometry draws once**, with `glDepthFunc(GL_LESS)` into the opaque framebuffer. This is the common case and it costs one instanced draw.
- **Transparent geometry is re-drawn `max_depth` times**, where `max_depth` is computed from the batch by `MaxTransparencyDepth`. Crucially the instance buffer is uploaded *once* via `SetInstances` before the loop, and each iteration re-issues `DrawInstances` against the same buffer. The cost of an extra peel is a draw call, not a re-upload.
- **`glDepthFunc(GL_GREATER)` inverts the depth test** so each pass captures the layer just behind the previous one, and from the second iteration onward `EnablePeeling` binds the previous pass's depth texture as `uPeelingTex` so the shader can reject fragments already resolved. The fragment shader's `float peel = texture(uPeelingTex, fPeelPos.xy).r;` is that test, and `fPeelPos` is the `depth * 0.5 + 0.5` value the vertex shader in §3.3 computed for exactly this purpose.
- **`ApplyTransparency` composites each peeled layer** using both `GetPaletteTexture()` and `GetBlendPaletteTexture()`. The second palette is the GPU equivalent of the CPU blitter's destination remap: rather than reading and remapping the framebuffer pixel inline, the blend palette encodes the result of combining a source index with a destination index.

The cost model is therefore linear in the *depth* of transparent overlap rather than in the count of transparent sprites, which for a theme park — where transparency is mostly used for effects and for the see-through modes applied to scenery and track — is usually a small number. This is a considered answer to a genuinely hard problem: the game's transparency semantics are defined by palette table lookups against the destination, not by alpha compositing, so standard back-to-front alpha blending would produce wrong colours rather than merely wrong ordering.

### 3.8 Why It Is Still Labelled Experimental

The OpenGL renderer remains opt-in and marked experimental, and the issue tracker explains why. The open OpenGL-specific reports span correctness, stability, and platform integration:

| Issue | Title | Category |
| --- | --- | --- |
| [#13387](https://github.com/OpenRCT2/OpenRCT2/issues/13387) | CTD when OpenGL render is selected on Manjaro 20.2 (Linux) | Driver/platform crash |
| [#22755](https://github.com/OpenRCT2/OpenRCT2/issues/22755) | OpenGL Error 0x0501 during startup | GL state / `GL_INVALID_VALUE` |
| [#26859](https://github.com/OpenRCT2/OpenRCT2/issues/26859) | OpenGL renderer randomly causes Windows OS to run at <1 FPS on VRR screens | Display timing |
| [#18841](https://github.com/OpenRCT2/OpenRCT2/issues/18841) | Fullscreen Borderless Windows with OpenGL rendering does not work correctly | Window management |
| [#16885](https://github.com/OpenRCT2/OpenRCT2/issues/16885) | Font Scaling issue in OpenGL | Scaling correctness |
| [#25700](https://github.com/OpenRCT2/OpenRCT2/issues/25700) | The mouse pointer disappears when the OpenGL engine is enabled | Cursor integration |
| [#13652](https://github.com/OpenRCT2/OpenRCT2/issues/13652) | Mouse Disappears when in Fullscreen when using Drawing Engine OpenGL | Cursor integration |
| [#15387](https://github.com/OpenRCT2/OpenRCT2/issues/15387) | Windows' volume overlay isn't visible in OpenGL Drawing Engine | Compositor/overlay interaction |
| [#16981](https://github.com/OpenRCT2/OpenRCT2/issues/16981) | Window moving generates "ghost pixels" in windowed mode on furthest right pixel row | Pixel-exactness |
| [#10657](https://github.com/OpenRCT2/OpenRCT2/issues/10657) | OpenGL erroneous pixels on top of window | Pixel-exactness |
| [#11531](https://github.com/OpenRCT2/OpenRCT2/issues/11531) | Bottom dot glitches when moving around in opengl mode | Pixel-exactness |
| [#9442](https://github.com/OpenRCT2/OpenRCT2/issues/9442) | OpenGL FPS drop with "4" invisible scenery | Performance regression |
| [#16504](https://github.com/OpenRCT2/OpenRCT2/issues/16504) | Assertion failed when hovering over RCT1 path when using OpenGL | Renderer assertion |
| [#17552](https://github.com/OpenRCT2/OpenRCT2/issues/17552) | "Show bounding boxes" not visible on OpenGL's extra zoom options | Feature parity |
| [#24466](https://github.com/OpenRCT2/OpenRCT2/issues/24466) | OpenGL crashes do not provide a screenshot | Diagnostics |

All fifteen were open as of 2026-08-08.

The pattern is instructive rather than merely unfortunate. The software renderer's output is *definitionally* correct — it is the reference — and it depends on nothing but the CPU. The OpenGL renderer must reproduce that output bit-for-bit across the entire driver matrix while also interacting correctly with window management, display timing, and scaling. Note what the category column contains and, more tellingly, what it does not: driver crashes, window-management defects, cursor and overlay integration failures, and off-by-one pixel differences. Almost nothing is a defect in the batching architecture of §3.3–3.5 — the instancing, atlas allocation, and palette resolution are not what is failing. A GPU 2D renderer for a game with exact pixel semantics inherits the whole compatibility burden of the graphics stack in exchange for performance the software path mostly already delivers. That tradeoff, more than any technical obstacle, is why the other projects in this chapter stopped at presentation.

---

## 4. OpenLoco: Hardware Presentation Without a GPU Renderer

OpenLoco (MIT) reimplements *Chris Sawyer's Locomotion*, and shares direct lineage with OpenRCT2's rendering model — the two original games used closely related engines. Its renderer is software-only in the sense that matters: sprite compositing happens entirely on the CPU, in `SoftwareDrawingEngine` and `SoftwareDrawingContext`, using routines inherited from the original game's painting code. There is no GPU sprite path and no OpenGL renderer.

An earlier survey of this project recorded that it had an *open, unimplemented* issue tracking a hardware-passthrough display path. That is now incorrect on both counts, and the correction is more interesting than the original claim. The issue requesting a hardware display option was closed as completed in mid-2023, and the implementation is present in the codebase — not in a new GPU renderer, but inside the software drawing engine itself.

What was asked for was explicitly a hardware *pass-through* display option for the software renderer, in the manner OpenRCT2 already provided — that is, OpenRCT2's middle mode from §3.1, not its OpenGL mode. The request states both the constraint and the motivation directly:

> Currently, OpenLoco uses a software renderer. This will most likely continue to be the case for a long time, as we still use the original Locomotion routines for painting in OpenLoco.
>
> A hardware back-end is not just a performance advantage. Some software relies on it for to ease image capture, for instance (e.g. streaming purposes). It's also a requirement to do proper interpolated linear scaling (cf. #376). I therefore propose we add a hardware (pass-through) display option for the software renderer, in the same way OpenRCT2 provides.

[Source: OpenLoco issue #418, "Option for hardware display"](https://github.com/OpenLoco/OpenLoco/issues/418)

Two things are worth extracting. First, the reason the software renderer is expected to persist indefinitely is *asset and code lineage*, not performance — the painting routines are inherited from the original game, and replacing them means rewriting the part of the engine that defines the game's visual identity. Second, neither stated motivation is frame rate: a GPU-backed presentation path eases screen capture for streaming, and it is a prerequisite for properly interpolated linear scaling. Both are presentation concerns, which is exactly what shipped.

The implementation initializes an SDL renderer, preferring the driver's default — hardware-accelerated where available — and falling back to the explicit software renderer only if that fails:

```cpp
// OpenLoco src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp
void SoftwareDrawingEngine::initialize(SDL_Window* window)
{
    _renderer = SDL_CreateRenderer(window, nullptr);
    if (_renderer == nullptr)
    {
        // Try to fallback to software renderer.
        Logging::warn("Hardware acceleration not available, falling back to software renderer.");

        _renderer = SDL_CreateRenderer(window, "software");
        if (_renderer == nullptr)
        {
            Logging::error("Unable to create software renderer: {}", SDL_GetError());
            std::abort();
        }
    }

    _window = window;
    createPalette();
}
```

[Source: OpenLoco `src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp`](https://github.com/OpenLoco/OpenLoco/blob/fd6db4114e720647fe2d90f26cd28979140971a8/src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp)

The present path then walks the palette-index buffer to the screen in four steps, and it is worth following because it is the OpenTTD architecture of §2 expressed through SDL's renderer abstraction instead of raw OpenGL:

```cpp
// OpenLoco src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp (abridged)
void SoftwareDrawingEngine::present()
{
    /* ... lock _screenSurface ... */

    // Copy pixels from the virtual screen buffer to the surface
    auto& rt = getScreenRT();
    if (rt.bits != nullptr)
    {
        std::memcpy(_screenSurface->pixels, rt.bits, _screenSurface->pitch * _screenSurface->h);
    }

    /* ... unlock ... */

    // Convert colours via palette mapping onto the RGBA surface.
    if (!SDL_BlitSurface(_screenSurface, nullptr, _screenRGBASurface, nullptr)) { /* ... */ }

    // Copy the RGBA pixels into screen texture.
    if (!SDL_UpdateTexture(_screenTexture, nullptr, _screenRGBASurface->pixels, _screenRGBASurface->pitch)) { /* ... */ }

    const auto scaleFactor = Config::get().scaleFactor;
    if (scaleFactor > 1.0f)
    {
        // Copy screen texture to the scaled texture.
        if (!SDL_SetRenderTarget(_renderer, _scaledScreenTexture)) { /* ... */ }
        if (!SDL_RenderTexture(_renderer, _screenTexture, nullptr, nullptr)) { /* ... */ }

        // Copy scaled texture to primary render target.
        if (!SDL_SetRenderTarget(_renderer, nullptr)) { /* ... */ }
        if (!SDL_RenderTexture(_renderer, _scaledScreenTexture, nullptr, nullptr)) { /* ... */ }
    }
    else
    {
        if (!SDL_RenderTexture(_renderer, _screenTexture, nullptr, nullptr)) { /* ... */ }
    }

    // Display buffers.
    if (!SDL_RenderPresent(_renderer)) { /* ... */ }
}
```

[Source: OpenLoco `src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp`](https://github.com/OpenLoco/OpenLoco/blob/fd6db4114e720647fe2d90f26cd28979140971a8/src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp)

Note `_screenSurface` is a palettized surface with an SDL palette attached, `_screenRGBASurface` is its RGBA target, and the palette resolution happens in `SDL_BlitSurface`. Where OpenTTD's `pal_program` does that conversion in a fragment shader (§2.4), OpenLoco does it on the CPU during the blit and hands RGBA to the texture. Both then present via one textured draw. The intermediate `_scaledScreenTexture` at scale factors above 1 is a two-stage upscale — integer-scale into a render-target texture, then scale that to the window — which gives sharp integer scaling followed by a single filtered step, and the texture scale mode is set to `SDL_SCALEMODE_NEAREST` to preserve pixel-art edges.

The convergence is the point of this section. Three independent projects, three different codebases, three different API surfaces — OpenTTD's hand-written OpenGL backend, OpenRCT2's "software with hardware display" mode, and OpenLoco's SDL renderer path — arrived at the identical architecture: **CPU composites the frame; GPU uploads it and presents it with scaling.** Only OpenRCT2 went further, and §3.8 describes what that has cost. The class in OpenLoco is still named `SoftwareDrawingEngine` after the hardware option shipped, which is an accurate name: the drawing is software; only the display is hardware.

---

## 5. OpenRA: Sampler-Bounded Sprite Batching

OpenRA (GPL-3.0) reimplements the *Command & Conquer* / *Red Alert* / *Dune 2000* lineage in C# on SDL2 and OpenGL. Architecturally it is the odd one out here, and the useful one: unlike the three isometric tycoon reimplementations, OpenRA composites sprites on the GPU as a matter of course. It is a conventional sprite batcher, and its constraints are exactly the ones OpenRCT2's array-texture design sidesteps — which makes it the clean contrast that renders §3 legible.

OpenRA's platform layer is SDL2 plus OpenGL, with no other backend: the default platform assembly contains `OpenGL.cs`, `Sdl2GraphicsContext.cs`, `Sdl2PlatformWindow.cs`, `Shader.cs`, `Texture.cs`, `VertexBuffer.cs` and a `ThreadedGraphicsContext.cs` that marshals GL calls onto a dedicated render thread.

The sprite batcher's central constraint is declared in its first line of state:

```csharp
// OpenRA OpenRA.Game/Graphics/SpriteRenderer.cs
public class SpriteRenderer : Renderer.IBatchRenderer
{
    public const int SheetCount = 8;
    static readonly string[] SheetIndexToTextureName = Exts.MakeArray(SheetCount, i => $"Texture{i}");

    readonly Renderer renderer;
    readonly IShader shader;

    Vertex[] vertices;
    readonly Sheet[] sheets = new Sheet[SheetCount];

    BlendMode currentBlend = BlendMode.Alpha;
    int vertexCount = 0;
    int sheetCount = 0;
```

[Source: OpenRA `OpenRA.Game/Graphics/SpriteRenderer.cs`](https://github.com/OpenRA/OpenRA/blob/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Game/Graphics/SpriteRenderer.cs)

Eight sheets, bound to eight shader uniforms named `Texture0` through `Texture7`. A "sheet" is OpenRA's atlas. Each sheet occupies its own sampler, and the shader can reference eight of them. This is one-texture-per-atlas — the design OpenRCT2 does not use.

Read that number carefully, because it is the crux of the comparison and it is easy to misread. Eight is *below* the sixteen texture image units OpenGL 3.3 core guarantees (§3.6). It is therefore not a hardware ceiling OpenRA collided with; it is a budget the renderer chose and declared in its shader, generating the uniform names from the constant itself. The interesting property of the design is not the specific value but the fact that a fixed value exists at all, and that the batcher must reason about it on every single sprite.

The consequence is that batching is bounded not by vertex-buffer capacity but by *how many distinct atlases the sprites in the batch come from*. When a sprite needs a ninth sheet, the batch must be flushed:

```csharp
// OpenRA OpenRA.Game/Graphics/SpriteRenderer.cs
int2 SetRenderStateForSprite(Sprite s)
{
    renderer.CurrentBatchRenderer = this;

    if (s.BlendMode != currentBlend || vertexCount + 4 > renderer.TempVertexBufferSize)
        Flush();

    currentBlend = s.BlendMode;

    // Check if the sheet (or secondary data sheet) have already been mapped
    var sheet = s.Sheet;
    var sheetIndex = 0;
    for (; sheetIndex < sheetCount; sheetIndex++)
        if (sheets[sheetIndex] == sheet)
            break;

    var secondarySheetIndex = 0;
    var ss = s as SpriteWithSecondaryData;
    if (ss != null)
    {
        var secondarySheet = ss.SecondarySheet;
        for (; secondarySheetIndex < sheetCount; secondarySheetIndex++)
            if (sheets[secondarySheetIndex] == secondarySheet)
                break;

        // If neither sheet has been mapped both index values will be set to ns.
        // This is fine if they both reference the same texture, but if they don't
        // we must increment the secondary sheet index to the next free sampler.
        if (secondarySheetIndex == sheetIndex && secondarySheet != sheet)
            secondarySheetIndex++;
    }

    // Make sure that we have enough free samplers to map both if needed, otherwise flush
    if (Math.Max(sheetIndex, secondarySheetIndex) >= sheets.Length)
    {
        Flush();
        sheetIndex = 0;
        secondarySheetIndex = ss != null && ss.SecondarySheet != sheet ? 1 : 0;
    }

    if (sheetIndex >= sheetCount)
    {
        sheets[sheetIndex] = sheet;
        sheetCount++;
    }
    /* ... */
    return new int2(sheetIndex, secondarySheetIndex);
}
```

[Source: OpenRA `OpenRA.Game/Graphics/SpriteRenderer.cs`](https://github.com/OpenRA/OpenRA/blob/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Game/Graphics/SpriteRenderer.cs)

The comments name the resource in question outright — "we must increment the secondary sheet index to the next free **sampler**", "make sure that we have enough free **samplers** to map both if needed, otherwise flush". Note what "free samplers" means here: free slots within OpenRA's own eight, not units remaining on the device. The error path that guards the public buffer-drawing entry point states the budget as the renderer's own property rather than the hardware's:

```csharp
// OpenRA OpenRA.Game/Graphics/SpriteRenderer.cs
// PERF: methods that throw won't be inlined by the JIT, so extract a static helper for use on hot paths
static void ThrowSheetOverflow(string paramName)
{
    throw new ArgumentException($"SpriteRenderer only supports {SheetCount} simultaneous textures", paramName);
}
```

[Source: OpenRA `OpenRA.Game/Graphics/SpriteRenderer.cs`](https://github.com/OpenRA/OpenRA/blob/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Game/Graphics/SpriteRenderer.cs)

"`SpriteRenderer` only supports 8 simultaneous textures" — the subject of the sentence is the renderer, and the message interpolates `SheetCount` rather than any queried device limit. This is the accurate form of the sampler argument: a self-imposed batching budget with a flush as its cost, not a hardware wall.

Also note the linear scan: for every sprite drawn, the code walks up to eight already-mapped sheets looking for a match. That is cheap at eight and would not be at eight hundred — which is itself a reason the budget stays small, and an argument for eliminating the search entirely as OpenRCT2 does.

Flushing binds the mapped sheets to their samplers and issues an indexed quad batch:

```csharp
// OpenRA OpenRA.Game/Graphics/SpriteRenderer.cs
public void Flush()
{
    if (vertexCount > 0)
    {
        for (var i = 0; i < sheetCount; i++)
        {
            shader.SetTexture(SheetIndexToTextureName[i], sheets[i].GetTexture());
            sheets[i] = null;
        }

        renderer.Context.SetBlendMode(currentBlend);
        shader.PrepareRender();

        renderer.DrawQuadBatch(ref vertices, shader, vertexCount);
        renderer.Context.SetBlendMode(BlendMode.None);

        vertexCount = 0;
        sheetCount = 0;
    }
}
```

[Source: OpenRA `OpenRA.Game/Graphics/SpriteRenderer.cs`](https://github.com/OpenRA/OpenRA/blob/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Game/Graphics/SpriteRenderer.cs)

Two further contrasts with OpenRCT2 are worth drawing:

**Four vertices per sprite versus one instance per sprite.** OpenRA writes real geometry — `Util.FastCreateQuad` appends four `Vertex` structures per sprite into a CPU array, and `DrawQuadBatch` draws `numVertices / 4 * 6` indices through a shared quad index buffer. The batch size ceiling `TempVertexBufferSize` is rounded down to a multiple of four (`vertexBatchSize - vertexBatchSize % 4`). OpenRCT2 instead uploads one ~96-byte command per sprite and lets the GPU replay four static vertices, which moves less data per sprite and avoids constructing geometry on the CPU.

**Palette handling.** OpenRA also keeps palettes on the GPU — sprites carry a `paletteTextureIndex` resolved by `ResolveTextureIndex`, and `TextureChannel.RGBA` sprites bypass palette assignment entirely. So both engines resolve palettes in shaders. The difference is that OpenRA carries palette selection in *per-vertex* data (replicated four times per sprite) rather than as a per-instance attribute.

**Flush triggers.** OpenRA flushes on blend-mode change, on vertex-buffer exhaustion, and on sampler exhaustion. OpenRCT2 flushes on opaque/transparent split (§3.2) and re-draws for transparency depth (§3.7), but never on atlas count. A single frame in OpenRA may therefore contain many draw calls determined by sprite-to-sheet locality; a frame in OpenRCT2 contains one opaque draw plus a small number of peel passes regardless of atlas distribution.

None of this makes OpenRA's design wrong, and the eight-sheet budget is not a deficiency. Eight is ample when sprite sheets are organized per-faction and per-tileset so that a frame's sprites cluster into few sheets, and it keeps the per-sprite linear scan short. The point of the comparison is narrower and more useful than "OpenRA ran out of samplers": OpenRA must *reason about atlas identity on every sprite* and flush when a batch outgrows its budget, whereas OpenRCT2's per-instance `texColourAtlas` attribute turns atlas identity into ordinary vertex data that costs nothing to vary. That is what the `usampler2DArray` buys — not a bigger budget, but the removal of a per-sprite decision and its associated flush.

OpenRA's release cadence also deserves a precise statement, since it is easy to misreport. The project's stable releases lag its commit activity substantially: development proceeds continuously on the main branch, with periodic playtest builds published between comparatively infrequent stable releases. Anyone citing "the current version" of OpenRA should distinguish the latest stable release from the latest playtest.

---

## 6. Modding APIs: Three Embedded-Language Choices

All four projects support deep user modification, and each solved the embedded-language problem differently. The comparison is unusually clean because the underlying requirement is nearly identical — let untrusted third-party content drive game logic without recompiling the engine — and yet the three answers share no technology.

### 6.1 OpenTTD: NewGRF Bytecode Plus Squirrel

OpenTTD splits modding into two entirely separate systems by concern.

**NewGRF** handles content: vehicles, buildings, industries, terrain, and their graphics and behaviour. It is not a scripting language but a binary data format descended from the original game's graphics-replacement mechanism, structured as numbered "actions" that define sprite sets, property overrides, and callback-driven variational logic. Authors generally write in a higher-level language and compile to the binary form. The format is declarative-with-callbacks rather than imperative: a NewGRF describes how a vehicle's properties and appearance vary as a function of game state, and the engine evaluates that description. This choice keeps content mods cheap to evaluate — a callback resolution is a table walk, not a script invocation — which matters when the engine may evaluate them for every vehicle every tick. Notably the NewGRF specification is not maintained in the OpenTTD repository; the `docs/` directory contains no NewGRF specification files, and the format is documented externally on the project wiki. [Source: OpenTTD wiki — NewGRF](https://wiki.openttd.org/en/Manual/NewGRF)

**Squirrel** handles behaviour. OpenTTD embeds the Squirrel scripting language (vendored under `src/3rdparty/squirrel`) and exposes it through two distinct APIs sharing one implementation:

- **NoAI** — computer-player scripts. An AI script plays the game through the same API surface a human would use: it builds infrastructure, buys vehicles, and manages finances, subject to the same rules.
- **GameScript** — a scenario/rules layer. Rather than playing, a GameScript imposes goals, restrictions, and events on the players.

The API surface is substantial: `src/script/api` contains on the order of 158 headers, following a consistent `script_*.hpp` naming scheme — `script_company.hpp`, `script_vehicle.hpp`, `script_road.hpp`, `script_town.hpp`, `script_controller.hpp`, `script_game.hpp`, and so on. Each wraps one game domain and is exposed to Squirrel as a class.

[Source: OpenTTD `src/script/api`](https://github.com/OpenTTD/OpenTTD/tree/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/script/api)

Two properties of this design are worth noting for anyone embedding a scripting language in a simulation. First, the API is a *deliberate* wrapper layer rather than direct binding to engine internals, which lets the project maintain API compatibility for scripts across engine refactors. Second, because both NoAI and GameScript run inside a deterministic multiplayer simulation, script execution must be deterministic and bounded — the surrounding `script_instance` machinery suspends and resumes scripts across ticks rather than letting them run to completion synchronously.

### 6.2 OpenRCT2: Duktape, JavaScript, and Shipped TypeScript Types

OpenRCT2 chose JavaScript, executed by the embedded duktape engine — a compact interpreter targeting ES5. The developer-facing story is the notable part: the project ships a TypeScript declaration file describing the entire plugin API, so that authors write TypeScript against real types and compile to the ES5 the engine executes.

The declaration file is large — roughly 5,900 lines — and lives at `distribution/scripting/openrct2.d.ts`. Its entry point is a single global registration function:

```typescript
// OpenRCT2 distribution/scripting/openrct2.d.ts (abridged)
export type PluginType = "local" | "remote" | "intransient";

declare global {
    function registerPlugin(metadata: PluginMetadata): void;

    interface PluginMetadata {
        name: string;
        version: string;
        authors: string | string[];
        type: PluginType;
        licence: string;
        minApiVersion?: number;
        /**
         * The Plug-in API version the current plug-in is designed for. This is used for backwards compatibility.
         * E.g.: 66
         */
        targetApiVersion: number;
        main: () => void;
    }
}
```

[Source: OpenRCT2 `distribution/scripting/openrct2.d.ts`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/distribution/scripting/openrct2.d.ts)

A minimal plugin is therefore:

```typescript
// Minimal OpenRCT2 plugin, written against openrct2.d.ts
registerPlugin({
    name: "example-plugin",
    version: "1.0",
    authors: ["example"],
    type: "local",
    licence: "MIT",
    targetApiVersion: 66,
    main: () => {
        console.log("plugin loaded");
    },
});
```

Three design decisions stand out.

**`PluginType` encodes the multiplayer trust model in the type system.** A `"local"` plugin runs only for the client that installed it. A `"remote"` plugin is one the server distributes and that participates in game state — meaning it must run identically on every client. `"intransient"` marks a plugin that survives across game loads rather than being torn down. Because OpenRCT2 is a deterministic lockstep simulation, a plugin that mutates game state on one client but not others desynchronizes the game; making the distinction a declared, checkable property rather than a convention is what allows the engine to enforce it.

**`targetApiVersion` plus optional `minApiVersion` is explicit API versioning.** The comment in the declaration file names the mechanism's purpose directly — backwards compatibility. The engine can present older semantics to a plugin declaring an older target, which lets the API evolve without invalidating the plugin ecosystem. Contrast this with the NewGRF approach, where compatibility is managed by the versioned action format itself.

**Shipping `.d.ts` in the distribution is a deliberate developer-experience investment.** The engine executes ES5; nothing in the runtime requires TypeScript. The declaration file exists purely so that plugin authors get completion and type checking, and it must be maintained in lockstep with the C++ API bindings. That is real ongoing cost accepted for ecosystem health.

### 6.3 OpenRA: Lua Missions and the Mod SDK

OpenRA embeds Lua, and scopes it primarily at *mission* logic — the scripted campaign and single-player scenarios that the C&C lineage is built around. Scripts drive objectives, reinforcements, triggers, and camera and interface behaviour.

The API is organized as a set of script globals, one C# class per domain, under `OpenRA.Mods.Common/Scripting/Global/`. The listing reads as a map of what mission scripting needs to touch: `ActorGlobal`, `PlayerGlobal`, `MapGlobal`, `TriggerGlobal`, `ReinforcementsGlobal`, `CameraGlobal`, `MediaGlobal`, `BeaconGlobal`, `RadarGlobal`, `UserInterfaceGlobal`, `LightingGlobal`, `DateTimeGlobal`, `AngleGlobal`, `ColorGlobal`, `CoordinateGlobals`, and `UtilsGlobal`.

[Source: OpenRA `OpenRA.Mods.Common/Scripting/Global`](https://github.com/OpenRA/OpenRA/tree/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Mods.Common/Scripting/Global)

`TriggerGlobal` is the architecturally central one. Mission scripts are overwhelmingly event-driven — when this actor dies, when this area is entered, when this many units are produced — and the trigger global is the binding between Lua callbacks and engine events. This is a different shape from OpenTTD's suspend/resume AI scripts: an OpenRA mission script is mostly callback registration, not a running agent.

Beyond per-mission scripting, OpenRA's larger modding story is **total conversion**, supported by a separate **Mod SDK** repository (GPL-3.0) described as a software development kit for building your own games using the OpenRA engine. This is a materially different ambition from the other projects surveyed: OpenTTD's NewGRF and OpenRCT2's plugins extend a specific game, whereas the OpenRA Mod SDK treats the engine as a platform on which unrelated games are built, with the shipped C&C-lineage mods as reference implementations. Correspondingly, most of what a total conversion changes lives in declarative YAML rule definitions rather than in Lua, with Lua reserved for scripted scenarios.

**Summary of the three approaches:**

| Project | Content/data layer | Behaviour layer | Engine | Type support |
| --- | --- | --- | --- | --- |
| OpenTTD | NewGRF (binary action format) | Squirrel — NoAI (agents) + GameScript (rules) | Vendored Squirrel | None (API docs) |
| OpenRCT2 | Object/scenario files | JavaScript plugins | duktape (ES5) | Shipped `openrct2.d.ts` |
| OpenRA | YAML rule definitions | Lua mission scripts | Lua via script globals | None (API docs) |

The recurring lesson is that each project separated *declarative content* from *imperative behaviour* and used different technology for each. None chose a single language for both. Chapter 205d takes up the general design space these three points sit within.

---

## 7. Brief Survey: The Framework Consumers

The following projects are simulation or strategy games of comparable ambition and community size, but from a rendering-architecture standpoint they delegate to a framework and add nothing distinctive. They are included for completeness and to substantiate the scoping decision announced in the introduction — there is no CPU-to-GPU migration story to tell about them, because they never wrote a sprite pipeline of their own.

| Project | Licence | Language | Rendering | Mod format |
| --- | --- | --- | --- | --- |
| Simutrans | Artistic Licence | C++ | SDL2, software rendering | Data files (`.dat`/`.pak`) |
| Unciv | MPL-2.0 | Kotlin | LibGDX; LWJGL3 OpenGL backend on desktop | JSON data mods |
| FreeCol | GPL-2.0 | Java | Java2D / Swing | XML rule and spec files |

[Sources: [Simutrans](https://github.com/aburch/simutrans) (licence text: `simutrans/license.txt`; SDL backend selection in `CMakeLists.txt` and `configure.ac`), [Unciv](https://github.com/yairm210/Unciv) (LWJGL3 declared in `gradle/libs.versions.toml` and used throughout `desktop/src/com/unciv/app/desktop/`), [FreeCol](https://github.com/FreeCol/freecol) (`Graphics2D` and `javax.swing` used pervasively across `src/net/sf/freecol/client/gui/`)]

Three observations, and then this section is done:

- **Simutrans** is the closest analogue to OpenTTD in genre and to §1's model in technique — palettized software sprite compositing presented through SDL2. It simply has not developed a distinct GPU path worth contrasting with OpenTTD's.
- **Unciv** is the only project in this chapter, surveyed or otherwise, whose rendering is GPU-first by default, because LibGDX is a GPU-first framework and its desktop backend runs on LWJGL3 over OpenGL. Unciv does not write a sprite batcher; LibGDX's `SpriteBatch` does that work, and Unciv's own developer documentation describes the division of labour explicitly: *"Images in LibGDX are displayed on screen by a SpriteBatch, which uses GL to bind textures to load them in-memory, and can then very quickly display them on-screen. The actual rendering is then very fast, but the binding process is slow."* [Source: Unciv `docs/Developers/Map-rendering.md`](https://github.com/yairm210/Unciv/blob/master/docs/Developers/Map-rendering.md)

  This is worth one extra sentence, because it shows the problem of §3 and §5 recurring one abstraction layer up. Unciv's response to expensive texture binds is to pack images into atlases capped at 2048×2048 — the same figure OpenRCT2 chose for `kTextureCacheMaxAtlasSize` (§3.4), and for the same stated reason of chipset limits — and then to reorder drawing so that all tiles' terrain is rendered before all tiles' improvements, so that sheet swaps scale with the number of layers rather than the number of tiles. That is OpenRA's sheet-locality problem (§5) and its mitigation by asset organization, arrived at independently and solved by draw-order scheduling instead of by a sampler budget. The mechanism, however, lives in LibGDX, which is outside this chapter's scope.
- **FreeCol** targets Java2D/Swing, a CPU 2D API with optional and platform-dependent hardware acceleration underneath. Its rendering concerns are Swing painting concerns.

Two further exclusions, stated so their absence is not read as oversight. **OpenDUNE** (GPL-2.0) reimplements the original *Dune II*; it is effectively dormant, with no release since 2018, and its renderer is a straightforward software framebuffer — it serves here only as a contrast to OpenRA's actively-developed treatment of the same lineage. **OpenMW** is an RPG engine with a fully 3D renderer and a substantial scripting story of its own, and is treated in Chapter 205d rather than here.

---

## 8. The Absence of Vulkan

No project examined in this chapter uses Vulkan. A source search across OpenRCT2, OpenTTD, OpenLoco, and OpenRA returns no Vulkan references in any of the four codebases. OpenRA's platform layer offers exactly one graphics backend — SDL2 with OpenGL. OpenTTD's GPU path is OpenGL, used as described in §2. OpenLoco reaches the GPU only through SDL's renderer abstraction. OpenRCT2's experimental renderer is OpenGL.

The generalization across the whole niche is therefore narrow and specific: **OpenGL appears only as an optional layer over a CPU sprite pipeline, or is absent entirely.** Not one project made a GPU API its primary or sole rendering path, and none adopted a modern explicit API.

This should be read as a scoping observation about the problem domain rather than as a deficiency. The reasons are visible throughout the preceding sections:

- **The workload does not demand it.** These are 2D sprite compositors for games designed to run on 1990s hardware. Even OpenRCT2's dense theme-park scene is within reach of a well-optimized CPU blitter with SIMD specializations, as §1.2's blitter matrix attests. Vulkan's advantages — explicit multithreaded command submission, fine-grained synchronization, reduced driver overhead — address bottlenecks these renderers do not have.
- **The cost is concentrated exactly where Vulkan is most expensive.** §3.8's defect list is dominated by driver, platform, and window-system integration problems. Vulkan demands substantially more of that same integration work — explicit swapchain management, memory allocation, synchronization primitives, and per-platform surface handling — for renderers whose correctness bar is bit-exact reproduction of a software reference.
- **The existing GPU use is trivially expressible in OpenGL.** Uploading a texture and drawing a fullscreen quad (§2.3) is a handful of OpenGL calls. Expressing the same operation in Vulkan requires a swapchain, a render pass or dynamic rendering setup, command buffers, and synchronization objects, to accomplish something the older API does in a dozen lines.
- **Contributor availability matters for volunteer projects.** OpenGL 3.3-era code is far more widely understood than Vulkan, and a renderer that few contributors can maintain is a liability in a project sustained by volunteers.

The honest conclusion is that the sprite-compositing problem these projects face was solved adequately on the CPU decades ago, and the GPU's marginal contribution — scaling, presentation, and in OpenRCT2's case palette-resolved batched compositing — is fully served by OpenGL. This chapter does not propose Vulkan ports for any of them, and readers should be sceptical of any account of this niche that presents one as an obvious improvement.

---

## Integrations

**Chapter 17 — Software Renderers (llvmpipe and Lavapipe)** covers the general-purpose CPU rasterization path in Mesa, and is the direct architectural analogue of §1's blitter stacks: both are software renderers that must present their finished pixel buffers through a GPU-backed window system, and both solve the handoff with the same PBO-and-fence machinery seen in OpenTTD's `GetVideoBuffer()`/`ReleaseVideoBuffer()` pair (§2.2). The instructive difference is specialization — llvmpipe JIT-compiles a general shading pipeline, while OpenTTD's blitters are hand-written specializations of a single fixed operation (palette-index blit) with SIMD variants per instruction set, which is why they remain competitive on a workload llvmpipe would handle far more slowly.

**Chapter 205d — Modding Architectures: Scripting, Sandboxing, and Hot-Reload** generalizes §6's three data points. The Squirrel, duktape-JavaScript, and Lua choices documented here are three resolutions of the same embedding problem under different constraints — determinism inside a lockstep multiplayer simulation for OpenTTD's NoAI/GameScript, an explicitly versioned API plus a declared local/remote trust model for OpenRCT2's plugins, and callback-driven scenario scripting for OpenRA — and Chapter 205d contrasts them against WASM sandboxing, Harmony IL patching, and Godot's GDExtension, where isolation and hot-reload are primary design goals rather than consequences. OpenMW, excluded from this chapter, is treated there.

**Chapter 84 — bgfx and Cross-Platform Rendering Abstraction** presents the modern alternative to §3's hand-built solution. OpenRCT2 solved instanced sprite batching from first principles: its own atlas allocator (§3.4), its own per-instance command encoding (§3.2–3.3), its own depth-peeling transparency (§3.7), and its own texture-array growth strategy — all written directly against OpenGL 3.3 and carrying the driver-compatibility burden of §3.8. bgfx supplies transient vertex buffers, instance buffers, and a backend-agnostic view/sort model that would express the same batch across OpenGL, Vulkan, Direct3D, and Metal without the renderer knowing which. The tradeoff visible by comparison is control over exact pixel semantics: OpenRCT2's `GL_R8UI` palette-index atlases and blend-palette transparency (§3.5, §3.7) are unusual enough that an abstraction layer's conveniences would have to be bypassed to reproduce them.

---

## References

**OpenRCT2**

- [GitHub — OpenRCT2/OpenRCT2](https://github.com/OpenRCT2/OpenRCT2) — Reimplementation of *RollerCoaster Tycoon 2* (GPL-3.0, C++). All file citations pinned to commit `8839956`.
- [`src/openrct2-ui/drawing/engines/opengl/DrawCommands.h`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/DrawCommands.h) — `DrawRectCommand` per-instance struct and `CommandBatch` container.
- [`src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/DrawRectShader.cpp) — `glVertexAttribDivisor` setup, `SetInstances`, `DrawInstances`.
- [`src/openrct2-ui/drawing/engines/opengl/TextureCache.h`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.h) — `Atlas` slot allocator, `kTextureCacheMaxAtlasSize`, `CalculateImageSizeOrder`.
- [`src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/TextureCache.cpp) — `GL_MAX_ARRAY_TEXTURE_LAYERS` / `GL_MAX_TEXTURE_SIZE` queries, `GL_R8UI` `glTexImage3D`, atlas array growth.
- [`src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/src/openrct2-ui/drawing/engines/opengl/OpenGLDrawingEngine.cpp) — command buffers, `FlushCommandBuffers`, `FlushRectangles`, `HandleTransparency`.
- [`data/shaders/drawrect.vert`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/data/shaders/drawrect.vert) — per-instance attribute inputs, `DEPTH_INCREMENT`, clip-folded position computation.
- [`data/shaders/drawrect.frag`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/data/shaders/drawrect.frag) — `usampler2DArray` atlas sampling, palette remap, depth-peel rejection.
- [`distribution/scripting/openrct2.d.ts`](https://github.com/OpenRCT2/OpenRCT2/blob/8839956721913ef48e6981477339a01f6f39e4f5/distribution/scripting/openrct2.d.ts) — shipped TypeScript declarations, `registerPlugin`, `PluginMetadata`, `PluginType`.
- [PR #4147 — "OpenGL sprite batch drawing"](https://github.com/OpenRCT2/OpenRCT2/pull/4147) — merged 2016-07-27; introduced batched instanced drawing. Its description states the goal as combining draw operations into a single draw call, and does not cite sampler-unit limits.
- [PR #7977 — "Fix OpenGL renderer stuttering when loading new textures into atlas."](https://github.com/OpenRCT2/OpenRCT2/pull/7977) — merged 2018-09-07; addresses cost of the atlas-array resize path.
- [PR #15616 — "Fix #15006: Prevent allocating empty texture atlases"](https://github.com/OpenRCT2/OpenRCT2/pull/15616) — merged 2021-10-20.
- Open OpenGL-renderer defect reports as of 2026-08-08 — issues [#9442](https://github.com/OpenRCT2/OpenRCT2/issues/9442), [#10657](https://github.com/OpenRCT2/OpenRCT2/issues/10657), [#11531](https://github.com/OpenRCT2/OpenRCT2/issues/11531), [#13387](https://github.com/OpenRCT2/OpenRCT2/issues/13387), [#13652](https://github.com/OpenRCT2/OpenRCT2/issues/13652), [#15387](https://github.com/OpenRCT2/OpenRCT2/issues/15387), [#16504](https://github.com/OpenRCT2/OpenRCT2/issues/16504), [#16885](https://github.com/OpenRCT2/OpenRCT2/issues/16885), [#16981](https://github.com/OpenRCT2/OpenRCT2/issues/16981), [#17552](https://github.com/OpenRCT2/OpenRCT2/issues/17552), [#18841](https://github.com/OpenRCT2/OpenRCT2/issues/18841), [#22755](https://github.com/OpenRCT2/OpenRCT2/issues/22755), [#24466](https://github.com/OpenRCT2/OpenRCT2/issues/24466), [#25700](https://github.com/OpenRCT2/OpenRCT2/issues/25700), [#26859](https://github.com/OpenRCT2/OpenRCT2/issues/26859).

**OpenTTD**

- [GitHub — OpenTTD/OpenTTD](https://github.com/OpenTTD/OpenTTD) — Reimplementation of *Transport Tycoon Deluxe* (C++). Licence confirmed as GNU GPL version 2 from the repository's [`COPYING.md`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/COPYING.md) (the repository's SPDX auto-detection reports no assertion). Citations pinned to commit `8ef6fa5`.
- [`src/blitter/`](https://github.com/OpenTTD/OpenTTD/tree/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/blitter) — the 8 bpp / 32 bpp / SIMD / animated blitter family.
- [`src/blitter/8bpp_simple.cpp`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/blitter/8bpp_simple.cpp) — reference blitter inner loop.
- [`src/video/opengl.h`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/video/opengl.h) — `OpenGLBackend` members: PBOs, fullscreen quad, palette texture, persistent mapping and `GLsync` fences, cursor cache.
- [`src/video/opengl.cpp`](https://github.com/OpenTTD/OpenTTD/blob/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/video/opengl.cpp) — `OpenGLBackend::Paint()` single-quad presentation; `RenderOglSprite` called only from `DrawMouseCursor()`.
- [`src/script/api`](https://github.com/OpenTTD/OpenTTD/tree/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/script/api) — the Squirrel-exposed NoAI / GameScript API surface (~158 `script_*.hpp` headers).
- [`src/3rdparty/`](https://github.com/OpenTTD/OpenTTD/tree/8ef6fa58a83f197c2dca78d032eb0f4e19a45f32/src/3rdparty) — vendored dependencies including `squirrel` and `opengl`.
- [OpenTTD wiki — NewGRF](https://wiki.openttd.org/en/Manual/NewGRF) — the NewGRF content-modding format, documented outside the source repository.
- [OpenTTD wiki — Script (NoAI / GameScript)](https://wiki.openttd.org/en/Development/Script/Main%20Page) — Squirrel scripting documentation for AI and GameScript authors.

**OpenLoco**

- [GitHub — OpenLoco/OpenLoco](https://github.com/OpenLoco/OpenLoco) — Reimplementation of *Chris Sawyer's Locomotion* (MIT, C++). Citations pinned to commit `fd6db41`.
- [`src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp`](https://github.com/OpenLoco/OpenLoco/blob/fd6db4114e720647fe2d90f26cd28979140971a8/src/OpenLoco/src/Graphics/SoftwareDrawingEngine.cpp) — SDL renderer initialization with software fallback; `present()` palette-blit-upload-present path; two-stage scaled presentation.
- [`src/OpenLoco/src/Graphics/`](https://github.com/OpenLoco/OpenLoco/tree/fd6db4114e720647fe2d90f26cd28979140971a8/src/OpenLoco/src/Graphics) — software drawing engine and context; no GPU sprite renderer.
- [OpenLoco issue #418 — "Option for hardware display"](https://github.com/OpenLoco/OpenLoco/issues/418) — request for a hardware pass-through display option for the software renderer; **closed as completed 2023-05-30**. Corrects an earlier characterization of this issue as open and unimplemented.

**OpenRA**

- [GitHub — OpenRA/OpenRA](https://github.com/OpenRA/OpenRA) — Reimplementation of the *Command & Conquer* / *Red Alert* / *Dune 2000* engines (GPL-3.0, C#). Citations pinned to commit `a520984`.
- [`OpenRA.Game/Graphics/SpriteRenderer.cs`](https://github.com/OpenRA/OpenRA/blob/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Game/Graphics/SpriteRenderer.cs) — `SheetCount = 8`, sampler-bounded sheet mapping, flush-on-sampler-exhaustion.
- [`OpenRA.Game/Renderer.cs`](https://github.com/OpenRA/OpenRA/blob/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Game/Renderer.cs) — `DrawQuadBatch`, `TempVertexBufferSize`, `IBatchRenderer` flush coordination.
- [`OpenRA.Platforms.Default/`](https://github.com/OpenRA/OpenRA/tree/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Platforms.Default) — the sole graphics backend: SDL2 plus OpenGL, with a threaded graphics context.
- [`OpenRA.Mods.Common/Scripting/Global/`](https://github.com/OpenRA/OpenRA/tree/a520984d91eda9de48a62b1d15c1e3bad0d4fb1a/OpenRA.Mods.Common/Scripting/Global) — Lua script globals for mission scripting.
- [GitHub — OpenRA/OpenRAModSDK](https://github.com/OpenRA/OpenRAModSDK) — Mod SDK for building standalone games on the OpenRA engine (GPL-3.0).

**Surveyed framework-consumer projects**

- [GitHub — aburch/simutrans](https://github.com/aburch/simutrans) — Transport simulation (Artistic Licence per `simutrans/license.txt`, C++, SDL2 software rendering).
- [GitHub — yairm210/Unciv](https://github.com/yairm210/Unciv) — Civilization-alike (MPL-2.0, Kotlin, LibGDX with LWJGL3 OpenGL desktop backend).
- [Unciv `docs/Developers/Map-rendering.md`](https://github.com/yairm210/Unciv/blob/master/docs/Developers/Map-rendering.md) — project documentation on LibGDX `SpriteBatch`, texture-bind cost, the 2048×2048 atlas cap, and layer-ordered drawing to minimise sheet swaps.
- [GitHub — FreeCol/freecol](https://github.com/FreeCol/freecol) — Colonization-alike (GPL-2.0, Java, Java2D/Swing).

**Specification references**

- [OpenGL 3.3 Core Profile Specification](https://registry.khronos.org/OpenGL/specs/gl/glspec33.core.pdf) — `GL_MAX_TEXTURE_IMAGE_UNITS`, `GL_MAX_ARRAY_TEXTURE_LAYERS`, `GL_MAX_TEXTURE_SIZE` minimum required values; instanced array semantics for `glVertexAttribDivisor` and `glDrawArraysInstanced`.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
