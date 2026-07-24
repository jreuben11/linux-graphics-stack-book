# Chapter 174: WezTerm and Alacritty — GPU Terminal Rendering Architectures

> **Part**: Part XII — Terminal Graphics
> **Audiences**: Terminal emulator developers; TUI application developers; Rust graphics developers interested in wgpu usage on Linux.
> **Complements**: Ch44 (Kitty and Ghostty GPU rendering), Ch43 (Terminal Pixel Protocols), Ch45 (Terminal Integration with the Compositor Stack), Ch152 (Rust GPU Ecosystem).

This chapter provides a deep architectural study of two GPU-accelerated terminal emulators written in Rust: **WezTerm** and **Alacritty**. Both sit squarely on the Linux graphics stack documented throughout this book — Mesa Vulkan drivers, EGL, Wayland protocols — but they take dramatically different design philosophies. WezTerm is a feature-rich multi-backend terminal with a built-in multiplexer, support for all three major pixel graphics protocols, and an opt-in **wgpu** (WebGPU) rendering path that traverses the same Mesa Vulkan path used by game engines and Bevy. Alacritty is deliberately minimal: an **OpenGL** renderer emphasising low latency and correctness over feature breadth, with no built-in multiplexer and intentionally absent graphics protocol support. Understanding both designs illuminates complementary points in the design space of GPU terminal rendering.

Chapter 44 covers Kitty and Ghostty in depth; readers of that chapter will already understand glyph atlas fundamentals and the general shape of a GPU terminal render loop. This chapter assumes that foundation and focuses on architecture-level differences:

- **`RenderContext` abstraction** — how WezTerm's central abstraction mediates between its two renderer backends (OpenGL/glium and wgpu)
- **wgpu WebGPU path** — how WGSL shaders are handed off to Mesa via `naga` → SPIR-V → NIR
- **Alacritty's `glutin`+`winit` stack** — how it constructs an EGL context on Wayland and calls into Mesa via the `gl` crate's raw function pointers
- **Linux-specific protocol integration** — how each terminal integrates with DMA-BUF, `zwp_text_input_v3`, and fractional scale (Part VIII)

---

## Table of Contents

1. [Introduction: Two Philosophies of GPU Terminal Rendering](#1-introduction-two-philosophies-of-gpu-terminal-rendering)
   - [1.1 What is WezTerm?](#11-what-is-wezterm)
   - [1.2 What is Alacritty?](#12-what-is-alacritty)
   - [1.3 What is wgpu?](#13-what-is-wgpu)
2. [WezTerm Architecture](#2-wezterm-architecture)
   - 2.1 [Crate Structure](#21-crate-structure)
   - 2.2 [Dual Renderer: OpenGL and wgpu](#22-dual-renderer-opengl-and-wgpu)
   - 2.3 [Initialising the wgpu Path on Linux](#23-initialising-the-wgpu-path-on-linux)
   - 2.4 [Glyph Rendering and the Texture Atlas](#24-glyph-rendering-and-the-texture-atlas)
   - 2.5 [Font Shaping: HarfBuzz and FreeType](#25-font-shaping-harfbuzz-and-freetype)
   - 2.6 [The Rendering Pipeline: Quads, Layers, and Shaders](#26-the-rendering-pipeline-quads-layers-and-shaders)
   - 2.7 [Wayland Integration and IME](#27-wayland-integration-and-ime)
   - 2.8 [Built-in Multiplexer](#28-built-in-multiplexer)
   - 2.9 [Pixel Graphics Protocols: Sixel, Kitty, and iTerm2](#29-pixel-graphics-protocols-sixel-kitty-and-iterm2)
3. [Alacritty Architecture](#3-alacritty-architecture)
   - 3.1 [Crate Structure and Design Philosophy](#31-crate-structure-and-design-philosophy)
   - 3.2 [OpenGL Context Initialisation via glutin and winit](#32-opengl-context-initialisation-via-glutin-and-winit)
   - 3.3 [The Renderer: Instanced GL Quads](#33-the-renderer-instanced-gl-quads)
   - 3.4 [The Glyph Cache and Texture Atlas](#34-the-glyph-cache-and-texture-atlas)
   - 3.5 [Font Rasterisation: crossfont and FreeType](#35-font-rasterisation-crossfont-and-freetype)
   - 3.6 [Wayland Integration](#36-wayland-integration)
   - 3.7 [Graphics Protocol Support (or Lack Thereof)](#37-graphics-protocol-support-or-lack-thereof)
4. [Comparative Analysis](#4-comparative-analysis)
5. [Linux Graphics Stack Integration](#5-linux-graphics-stack-integration)
6. [Performance Characteristics](#6-performance-characteristics)
7. [Integrations](#7-integrations)

---

## 1. Introduction: Two Philosophies of GPU Terminal Rendering

The GPU terminal landscape of 2026 can be taxonomised along two axes: **rendering API** (OpenGL vs. Vulkan/wgpu vs. custom Metal/Vulkan) and **feature scope** (minimal vs. feature-rich). Chapter 44 surveyed Kitty, which uses a mature hand-written OpenGL renderer in C with deep Kitty Graphics Protocol integration, and Ghostty, which employs a platform-optimised renderer written in Zig (Metal on macOS, OpenGL via GTK4 on Linux, with a Vulkan backend in development). This chapter covers the two prominent Rust-native entries in the same landscape.

**WezTerm** (written by Wez Furlong, repository: [`github.com/wezterm/wezterm`](https://github.com/wezterm/wezterm)) is positioned as a full-featured terminal emulator and multiplexer. Its GPU rendering story is distinctive in that it supports *two* rendering backends through a common abstraction: a legacy OpenGL path using `glium`, and a modern WebGPU path using `wgpu`. The wgpu path — enabled by `front_end = "WebGpu"` in the user configuration — selects the Vulkan backend on Linux, routing WGSL shaders through `naga` → SPIR-V → Mesa NIR → the ACO or LLVM-IR backend of the appropriate Mesa driver. WezTerm also implements Sixel, the Kitty Graphics Protocol, and the iTerm2 image protocol, and ships a built-in tmux-compatible multiplexer.

**Alacritty** (repository: [`github.com/alacritty/alacritty`](https://github.com/alacritty/alacritty)) takes the opposite design philosophy: minimalism and correctness over feature breadth. It is described on its own homepage as "a cross-platform, OpenGL terminal emulator." Its renderer uses OpenGL ES 2.0 or desktop OpenGL 3.3 with instanced rendering via `glutin` for context creation and the `gl` crate (gl_generator) for raw OpenGL function pointer dispatch, with the `winit` crate providing cross-platform windowing. Alacritty deliberately excludes a built-in multiplexer, directing users to pair it with tmux or zellij. As of version 0.17 (April 2026), it also deliberately excludes Sixel and the Kitty Graphics Protocol. This intentional constraint keeps the codebase small and the latency predictable.

The contrast between the two is instructive: WezTerm's wgpu backend and Alacritty's EGL/GL backend both traverse the same underlying Mesa drivers and kernel DRM infrastructure, but they reach those drivers via entirely different API paths — wgpu → ash Vulkan bindings → Mesa Vulkan vs. `glutin` → EGL → Mesa GL. Understanding both paths concretely anchors the abstract API layering described in Parts IV–VII.

### 1.1 What is WezTerm?

WezTerm is a GPU-accelerated terminal emulator and built-in multiplexer written in Rust, targeting Linux, macOS, Windows, and FreeBSD. Unlike most terminal emulators that commit to a single rendering API, WezTerm supports two GPU backends through a common abstraction layer: a legacy OpenGL path using the `glium` crate and a modern WebGPU path using `wgpu`. On Linux, the wgpu backend selects Vulkan as its primary graphics API, routing WGSL shader source through the `naga` compiler to SPIR-V, which Mesa ingests as NIR before handing it to the driver-specific compiler backend — ACO for AMD RADV, the ISL-based backend for Intel ANV, or the NVK backend for NVIDIA. The default frontend is OpenGL; the wgpu path is opt-in via `front_end = "WebGpu"` in the Lua configuration file. Beyond rendering, WezTerm ships a tmux-compatible built-in multiplexer with a PDU-based RPC protocol over Unix sockets, SSH tunnels, or TLS, and implements three pixel graphics protocols — Sixel, the Kitty Graphics Protocol, and the iTerm2 image protocol — exposing its configuration through an embedded Lua interpreter. The codebase is structured as a Cargo workspace of approximately 35 crates, with the `wezterm-gui` crate housing the GPU rendering layer and `mux` implementing the multiplexer domain model. In the context of the Linux graphics stack, WezTerm is significant because its wgpu path traverses the same Mesa Vulkan driver pipeline used by game engines and compute workloads, making it a concrete, inspectable example of a wgpu application on Linux.

### 1.2 What is Alacritty?

Alacritty is a cross-platform terminal emulator written in Rust, designed around the principles of minimalism, correctness, and low latency. It uses OpenGL ES 2.0 or desktop OpenGL 3.3 for rendering, initialising its graphics context through the `glutin` crate — which calls into EGL on Wayland (via the `EGL_KHR_platform_wayland` extension) and GLX on X11 — and dispatching raw OpenGL function pointers through the `gl` crate generated by `gl_generator`. The `winit` crate provides the cross-platform windowing abstraction and event loop. Alacritty deliberately excludes a built-in multiplexer (directing users to pair it with tmux or zellij), Sixel support, and the Kitty Graphics Protocol. This intentional constraint keeps the codebase compact — the main `alacritty` binary crate and the supporting `alacritty-terminal` library together are substantially smaller than WezTerm's workspace — and ensures that the render path remains predictable and auditable. On Wayland, Alacritty's EGL context is backed by a `wl_egl_window` surface; the framebuffer produced by Mesa's EGL implementation is presented to the compositor as a `wl_buffer` through the standard Wayland presentation protocol. On X11, GLX is used instead. The renderer employs instanced rendering of textured quads for glyph drawing, using a texture atlas populated by the `crossfont` crate (a FreeType wrapper). Alacritty represents the minimal end of the GPU terminal design space, and its simplicity makes it a useful reference for understanding what a compliant OpenGL terminal on Linux actually requires.

### 1.3 What is wgpu?

wgpu is a safe, portable Rust implementation of the WebGPU API — the graphics and compute API standardised by the W3C as a cross-platform successor to WebGL. In Rust native applications (as opposed to browser contexts), wgpu runs as a native library: it selects an appropriate backend API at runtime — Vulkan on Linux and Android, Metal on macOS and iOS, Direct3D 12 on Windows — and exposes a unified `Device`/`Queue`/`RenderPipeline`/`Buffer` abstraction over whichever backend is active. Shaders are written in WGSL (WebGPU Shading Language) and compiled to the backend's intermediate representation by the `naga` library: on Linux, WGSL is translated to SPIR-V, which Vulkan drivers accept via `vkCreateShaderModule`. The raw Vulkan function bindings underneath wgpu's Vulkan backend are provided by the `ash` crate; the backend implementation lives in `wgpu-hal`. For WezTerm specifically, wgpu powers the `WebGpu` frontend mode: WezTerm's WGSL vertex and fragment shaders for the terminal grid, glyph atlas sampling, and image protocol blitting are compiled at startup by `naga`, dispatched as SPIR-V to the active Mesa Vulkan driver, and executed on the GPU through a `wgpu::RenderPipeline`. Because wgpu routes through Mesa Vulkan on Linux, the same driver compilation paths exercised by Vulkan game engines are exercised by WezTerm's wgpu backend, making WezTerm an observable integration point between terminal emulation and the Vulkan ecosystem described in Part V.

---

## 2. WezTerm Architecture

### 2.1 Crate Structure

WezTerm is a Cargo workspace of approximately 35 crates. The key architectural layers are: [Source](https://github.com/wezterm/wezterm)

| Crate | Role |
|---|---|
| `wezterm` | Top-level binary; CLI parsing, daemon launch |
| `wezterm-gui` | GPU rendering (`TermWindow`, `RenderState`, `GlyphCache`), windowing |
| `window` | Cross-platform windowing abstraction; houses the `Atlas` type and `Texture2d` trait |
| `wezterm-font` | Font discovery, loading, shaping (`wezterm-font/src/shaper/harfbuzz.rs`), rasterisation |
| `wezterm-term` | Terminal emulation: VT parser, cell grid, scrollback |
| `termwiz` | Standalone terminal widget library; surfaces, escape sequence parsing, `Terminal` trait |
| `mux` | Multiplexer: `Mux` singleton, `Window`/`Tab`/`Pane` hierarchy, `Domain` abstraction |
| `wezterm-mux-server` | Headless mux server daemon; PDU-based RPC over Unix socket, SSH, or TLS |
| `wezterm-client` | Client-side mux protocol; `ClientPane` proxy type |
| `wezterm-ssh` | SSH multiplexer domain implementation |
| `vtparse` | Low-level VT escape sequence parser |
| `pty` | PTY creation and management |
| `config` | Lua-based configuration (`FrontEndSelection`, `GpuInfo`) |

The `termwiz` crate deserves particular mention: it is published independently to crates.io and can be used as a standalone library for building terminal UIs (the `Terminal` trait abstracts over Unix TTYs and Windows console), building terminal emulators (the `Surface` type models a terminal display grid), or parsing escape sequences. WezTerm consumes `termwiz` internally for its own terminal surface model, but third-party Rust terminal UI libraries also use it directly. [Source](https://crates.io/crates/termwiz)

### 2.2 Dual Renderer: OpenGL and wgpu

A key architectural fact about WezTerm that is often misunderstood: **the default rendering backend is OpenGL via `glium`, not wgpu**. The `FrontEndSelection` enum in `config/src/frontend.rs` makes this explicit: [Source](https://github.com/wezterm/wezterm/blob/main/config/src/frontend.rs)

```rust
// config/src/frontend.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, FromDynamic, ToDynamic, Default)]
pub enum FrontEndSelection {
    #[default]
    OpenGL,   // <-- default: glium (OpenGL)
    WebGpu,   // opt-in: wgpu (Vulkan on Linux)
    Software, // CPU-only software renderer
}
```

The wgpu path is opt-in: users must add `front_end = "WebGpu"` to their WezTerm Lua configuration. When wgpu is selected on Linux, wgpu selects Vulkan as its primary backend, interfacing with Mesa drivers (RADV for AMD, ANV for Intel, NVK for NVIDIA). When OpenGL is selected, WezTerm uses `glium`, which is an older safe OpenGL wrapper crate.

Both paths share the same high-level rendering abstractions in `wezterm-gui`. The central abstraction is `RenderContext`, an enum that dispatches across backends: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-gui/src/renderstate.rs)

```rust
// wezterm-gui/src/renderstate.rs
#[derive(Clone)]
pub enum RenderContext {
    Glium(Rc<GliumContext>),
    WebGpu(Rc<WebGpuState>),
}
```

Every resource allocation — vertex buffers, index buffers, texture atlases — passes through `RenderContext` and dispatches to the appropriate backend. For example, `allocate_texture_atlas` creates either a `SrgbTexture2d` (Glium path) or a `WebGpuTexture` (wgpu path):

```rust
// wezterm-gui/src/renderstate.rs
pub fn allocate_texture_atlas(&self, size: usize) -> anyhow::Result<Rc<dyn Texture2d>> {
    match self {
        Self::Glium(context) => {
            use crate::glium::texture::SrgbTexture2d;
            let surface: Rc<dyn Texture2d> = Rc::new(SrgbTexture2d::empty_with_format(
                context,
                glium::texture::SrgbFormat::U8U8U8U8,
                glium::texture::MipmapsOption::NoMipmap,
                size as u32, size as u32,
            )?);
            Ok(surface)
        }
        Self::WebGpu(state) => {
            let texture: Rc<dyn Texture2d> =
                Rc::new(WebGpuTexture::new(size as u32, size as u32, state)?);
            Ok(texture)
        }
    }
}
```

This dual-backend architecture is an important design decision: it allows WezTerm to maintain wide compatibility (OpenGL is available on virtually all Linux systems, including headless X11 and older GPUs) while offering the performance and cross-platform portability of wgpu to users who opt in.

### 2.3 Initialising the wgpu Path on Linux

When the user configures `front_end = "WebGpu"`, WezTerm initialises a `WebGpuState` asynchronously during window creation. The initialisation code in `wezterm-gui/src/termwindow/webgpu.rs` shows the standard wgpu instance-surface-adapter-device sequence: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-gui/src/termwindow/webgpu.rs)

```rust
// wezterm-gui/src/termwindow/webgpu.rs (simplified)
pub async fn new_impl(
    handle: RawHandlePair,
    dimensions: Dimensions,
    config: &ConfigHandle,
) -> anyhow::Result<Self> {
    let backends = wgpu::Backends::all();
    let instance = wgpu::Instance::new(&wgpu::InstanceDescriptor {
        backends,
        ..Default::default()
    });
    let surface = unsafe {
        instance.create_surface_unsafe(
            wgpu::SurfaceTargetUnsafe::from_window(&handle)?
        )?
    };

    // Adapter selection: honours config.webgpu_preferred_adapter if set,
    // otherwise falls back to request_adapter with power preference.
    let adapter = instance
        .request_adapter(&wgpu::RequestAdapterOptions {
            power_preference: match config.webgpu_power_preference {
                WebGpuPowerPreference::HighPerformance =>
                    wgpu::PowerPreference::HighPerformance,
                WebGpuPowerPreference::LowPower =>
                    wgpu::PowerPreference::LowPower,
            },
            compatible_surface: Some(&surface),
            force_fallback_adapter: config.webgpu_force_fallback_adapter,
        })
        .await?;

    let (device, queue) = adapter
        .request_device(&wgpu::DeviceDescriptor {
            required_limits: wgpu::Limits::downlevel_defaults()
                .using_resolution(adapter.limits()),
            ..Default::default()
        })
        .await?;
    // ...
}
```

On Linux with Mesa drivers, `wgpu::Backends::all()` causes wgpu to probe for both Vulkan and OpenGL adapters. The Vulkan adapter — backed by RADV (AMD), ANV (Intel), or NVK (NVIDIA) — is preferred when available. The `ash` crate provides the raw Vulkan bindings underneath wgpu's `wgpu-hal` Vulkan backend. Internally, wgpu-hal creates a `VkInstance`, enumerates `VkPhysicalDevice` objects, and creates a `VkDevice` — the same steps any native Vulkan application performs.

The `WebGpuState` struct holds the persistent wgpu objects: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-gui/src/termwindow/webgpu.rs)

```rust
// wezterm-gui/src/termwindow/webgpu.rs
pub struct WebGpuState {
    pub adapter_info: wgpu::AdapterInfo,
    pub downlevel_caps: wgpu::DownlevelCapabilities,
    pub surface: wgpu::Surface<'static>,
    pub device: wgpu::Device,
    pub queue: Arc<wgpu::Queue>,
    pub config: RefCell<wgpu::SurfaceConfiguration>,
    pub render_pipeline: wgpu::RenderPipeline,
    pub texture_bind_group_layout: wgpu::BindGroupLayout,
    pub texture_nearest_sampler: wgpu::Sampler,
    pub texture_linear_sampler: wgpu::Sampler,
    // ...
}
```

The `wgpu::SurfaceConfiguration` uses `PresentMode::Fifo` (equivalent to VSync) and selects the alpha compositing mode based on what the surface capabilities report — preferring `PostMultiplied` or `PreMultiplied` when available, falling back to `Auto`: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-gui/src/termwindow/webgpu.rs)

```rust
// wezterm-gui/src/termwindow/webgpu.rs
let config = wgpu::SurfaceConfiguration {
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT,
    format,
    width: dimensions.pixel_width as u32,
    height: dimensions.pixel_height as u32,
    present_mode: wgpu::PresentMode::Fifo,
    alpha_mode: if caps.alpha_modes.contains(
        &wgpu::CompositeAlphaMode::PostMultiplied) {
        wgpu::CompositeAlphaMode::PostMultiplied
    } else {
        wgpu::CompositeAlphaMode::Auto
    },
    desired_maximum_frame_latency: 2,
    ..Default::default()
};
```

On Linux/Wayland the surface is created against a `wl_surface`, using the raw Wayland window handle exposed by `winit`. The wgpu Vulkan WSI layer (`VK_KHR_wayland_surface`) creates a `VkSurfaceKHR` from this handle. From that point on, frame presentation follows the standard Vulkan swapchain path: `wgpu::SurfaceTexture` wraps a `VkImage` from the swapchain, and `surface_texture.present()` calls `vkQueuePresentKHR`.

Users can enumerate available adapters via the Lua debug overlay (`wezterm.gui.enumerate_gpus()`), which returns a list of `GpuInfo` objects containing the adapter name, backend type (`"Vulkan"`, `"Gl"`), device type (`"DiscreteGpu"`, `"IntegratedGpu"`), driver name, vendor, and device IDs. A typical Lua snippet to select a specific discrete Vulkan adapter:

```lua
-- ~/.config/wezterm/wezterm.lua
config.front_end = "WebGpu"
config.webgpu_power_preference = "HighPerformance"
-- enumerate_gpus() can be used to identify the right adapter name:
-- config.webgpu_preferred_adapter = wezterm.gui.enumerate_gpus()[1]
```

[Source: wezterm.org/config/lua/config/webgpu_preferred_adapter.html](https://wezterm.org/config/lua/config/webgpu_preferred_adapter.html)

### 2.4 Glyph Rendering and the Texture Atlas

WezTerm's glyph caching is implemented in `wezterm-gui/src/glyphcache.rs`. The central type is `GlyphCache`, which aggregates several caches: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-gui/src/glyphcache.rs)

```rust
// wezterm-gui/src/glyphcache.rs
pub struct GlyphCache {
    glyph_cache: HashMap<GlyphKey, Rc<CachedGlyph>>,
    pub atlas: Atlas,              // from ::window::bitmaps::atlas
    pub fonts: Rc<FontConfiguration>,
    pub image_cache: LfuCache<[u8; 32], DecodedImage>, // for pixel graphics
    frame_cache: HashMap<[u8; 32], Sprite>,
    line_glyphs: HashMap<LineKey, Sprite>,
    pub block_glyphs: HashMap<SizedBlockKey, Sprite>,
    pub cursor_glyphs: HashMap<(Option<CursorShape>, u8), Sprite>,
    pub color: HashMap<(RgbColor, NotNan<f32>), Sprite>,
    min_frame_duration: Duration,
}
```

The `Atlas` type (from `window::bitmaps::atlas`) uses the [`guillotiere`](https://crates.io/crates/guillotiere) crate for dynamic rectangle packing — a GPU-resident texture into which rasterised glyph bitmaps are packed. `guillotiere` implements a shelf-based guillotine cutting algorithm with rectangle coalescing on deallocation, giving better packing density than a simple shelf allocator at the cost of slightly more bookkeeping.

Each cached glyph is stored as a `CachedGlyph`:

```rust
// wezterm-gui/src/glyphcache.rs
pub struct CachedGlyph {
    pub has_color: bool,        // true for colour emoji
    pub brightness_adjust: f32,
    pub x_offset: PixelLength,
    pub y_offset: PixelLength,
    pub x_advance: PixelLength,
    pub bearing_x: PixelLength,
    pub bearing_y: PixelLength,
    pub texture: Option<Sprite>, // None for whitespace
    pub scale: f64,
}
```

The `Sprite` type encapsulates a reference into the atlas texture plus the UV rectangle of the glyph within it. When the cache misses on `GlyphKey { font_idx, glyph_pos, num_cells, style, metric, id }`, WezTerm calls into the font rasteriser, obtains a bitmap, and uploads it to the atlas via `Texture2d::write()`. For the wgpu backend this calls `WebGpuTexture::write()`, which uses `queue.write_texture()`: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-gui/src/termwindow/webgpu.rs)

```rust
// wezterm-gui/src/termwindow/webgpu.rs
impl Texture2d for WebGpuTexture {
    fn write(&self, rect: Rect, im: &dyn BitmapImage) {
        let (im_width, im_height) = im.image_dimensions();
        self.queue.write_texture(
            wgpu::TexelCopyTextureInfo {
                texture: &self.texture,
                mip_level: 0,
                origin: wgpu::Origin3d {
                    x: rect.min_x() as u32,
                    y: rect.min_y() as u32,
                    z: 0,
                },
                aspect: wgpu::TextureAspect::All,
            },
            im.pixel_data_slice(),
            wgpu::TexelCopyBufferLayout {
                offset: 0,
                bytes_per_row: Some(im_width as u32 * 4),
                rows_per_image: Some(im_height as u32),
            },
            wgpu::Extent3d {
                width: im_width as u32,
                height: im_height as u32,
                depth_or_array_layers: 1,
            },
        );
    }
}
```

When the atlas becomes full (returns `OutOfTextureSpace`), WezTerm recreates the `GlyphCache` with a fresh atlas, flushing all cached glyphs. Eviction policy is coarse: the cache is fully rebuilt rather than selectively evicted. For image data (pixel graphics protocol payloads), WezTerm uses an `LfuCache` (Least Frequently Used) keyed by a 32-byte hash of the image data.

Custom block-drawing glyphs, box-drawing characters, and progress bars are rendered procedurally using `tiny_skia` path rasterisation in `wezterm-gui/src/customglyph.rs` — vector paths defined via `PolyCommand` enums (`MoveTo`, `LineTo`, `QuadTo`, `Oval`, `Circle`, `Close`). The resulting bitmaps are then inserted into the same atlas.

### 2.5 Font Shaping: HarfBuzz and FreeType

WezTerm's font system lives in the `wezterm-font` crate. Shaping is implemented in `wezterm-font/src/shaper/harfbuzz.rs` using the HarfBuzz C library via Rust FFI bindings. The `FontShaper::shape()` method takes a Unicode text run and produces a `Vec<GlyphInfo>`: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-font/src/shaper/harfbuzz.rs)

```rust
// wezterm-font/src/shaper/harfbuzz.rs
fn shape(
    &self,
    text: &str,
    size: f64,
    dpi: u32,
    no_glyphs: &mut Vec<char>,
    presentation: Option<Presentation>,
    direction: Direction,
    range: Option<Range<usize>>,
    presentation_width: Option<&PresentationWidth>,
) -> anyhow::Result<Vec<GlyphInfo>> {
    let start = std::time::Instant::now();
    let result = self.do_shape(0, text, size, dpi, no_glyphs,
        presentation, direction, range, presentation_width);
    metrics::histogram!("shape.harfbuzz").record(start.elapsed());
    result
}
```

The internal `do_shape` method fills a `harfbuzz::Buffer` with the Unicode text, sets the shaping direction and language, then calls `harfbuzz::shape()` to produce glyph positions. HarfBuzz applies OpenType GSUB (substitution) and GPOS (positioning) lookups, enabling ligatures (`fi`, `fl`, programming ligatures like `->` and `!=` in fonts such as Fira Code), kerning pairs, and emoji cluster composition (e.g. `👨‍👩‍👧‍👦` composed from multiple codepoints via ZWJ sequences).

The `GlyphInfo` struct carries glyph positions in sub-pixel units (HarfBuzz position units are 1/64th of a pixel, stored as `hb_position_t` integers): [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-font/src/shaper/harfbuzz.rs)

```rust
// wezterm-font/src/shaper/harfbuzz.rs
fn make_glyphinfo(text: &str, num_cells: u8, font_idx: usize, info: &Info) -> GlyphInfo {
    GlyphInfo {
        num_cells,
        font_idx,
        glyph_pos: info.codepoint,
        cluster: info.cluster as u32,
        x_advance: PixelLength::new(f64::from(info.x_advance) / 64.0),
        y_advance: PixelLength::new(f64::from(info.y_advance) / 64.0),
        x_offset: PixelLength::new(f64::from(info.x_offset) / 64.0),
        y_offset: PixelLength::new(f64::from(info.y_offset) / 64.0),
        // ...
    }
}
```

Each `GlyphInfo` is then passed to `wezterm-font/src/rasterizer/harfbuzz.rs` (via `LoadedFont::rasterize_glyph()`) which calls into FreeType to render the glyph at the requested size and DPI. FreeType rendering honours fontconfig settings: `FT_LOAD_TARGET_LCD` for subpixel RGB antialiasing, `FT_LOAD_TARGET_LIGHT` for greyscale antialiasing, or `FT_LOAD_TARGET_MONO` for 1-bit output. The resulting `RasterizedGlyph` bitmap is then uploaded to the atlas as described in section 2.4.

Emoji rendering uses a font fallback mechanism: if the primary font lacks a glyph for a given codepoint, WezTerm iterates through a fallback font list (configured via `wezterm.font_with_fallback()`), finds the first font containing the glyph, and uses that font's colour bitmap (CBDT/CBLC or COLR/CPAL). Colour emoji glyphs are flagged with `CachedGlyph::has_color = true`, which causes the fragment shader to bypass the foreground-colour tinting and use the raw bitmap colours directly.

WezTerm also caches shaping results at the line level via `shapecache.rs`, storing `ShapedInfo` runs keyed by a hash of the cell sequence. This avoids re-running HarfBuzz for lines that have not changed, which is particularly valuable for static content (e.g., a prompt line that hasn't been updated).

### 2.6 The Rendering Pipeline: Quads, Layers, and Shaders

On each paint event (triggered by a Wayland frame callback or an application output notification), `TermWindow::paint_impl()` orchestrates the full rendering pass:

1. **Iterate visible panes**: for each pane in the active tab, compute the dirty cell region from the terminal's damage tracking.
2. **Text shaping**: for each visible line, check the line cache. On a miss, call `FontShaper::shape()` to produce `ShapedInfo` runs via HarfBuzz, and store the result in the shape cache.
3. **Glyph upload**: for each glyph in the shaped runs, look up `GlyphKey` in `GlyphCache`. On a miss, rasterise via FreeType and upload to the atlas.
4. **Quad allocation**: each terminal cell is represented as a quad (two triangles, four vertices). The `TripleLayerQuadAllocator` organises quads into three layers — **background fills**, **text cell quads**, and **cursor/decoration quads** — to control z-ordering without requiring depth buffer usage.
5. **Vertex data**: each `Vertex` carries position in clip space, UV coordinates into the atlas texture, foreground and background colours (as `u32` packed SRGBA), and HSV adjustment factors for colour transformations.
6. **GPU draw call**: `RenderState` submits the batched vertex and index buffers to the GPU. For the wgpu path, this is a single `RenderPass` with `RenderPass::draw_indexed()`.

The wgpu `RenderPipeline` is configured with `BlendState::ALPHA_BLENDING` and `PrimitiveTopology::TriangleList`:

```rust
// wezterm-gui/src/termwindow/webgpu.rs (render pipeline setup, abbreviated)
targets: &[Some(wgpu::ColorTargetState {
    format: config.format,
    blend: Some(wgpu::BlendState::ALPHA_BLENDING),
    write_mask: wgpu::ColorWrites::ALL,
})],
primitive: wgpu::PrimitiveState {
    topology: wgpu::PrimitiveTopology::TriangleList,
    front_face: wgpu::FrontFace::Ccw,
    cull_mode: None,
    polygon_mode: wgpu::PolygonMode::Fill,
    // ...
},
```

The shader uniform struct `ShaderUniform` carries per-frame data: HSV transform for foreground text, animation millisecond timestamp (for cursor blink computed by a cubic Bezier curve on the GPU), and the projection matrix: [Source](https://github.com/wezterm/wezterm/blob/main/wezterm-gui/src/termwindow/webgpu.rs)

```rust
// wezterm-gui/src/termwindow/webgpu.rs
#[repr(C)]
#[derive(Copy, Clone, Default, Debug, bytemuck::Pod, bytemuck::Zeroable)]
pub struct ShaderUniform {
    pub foreground_text_hsb: [f32; 3],
    pub milliseconds: u32,
    pub projection: [[f32; 4]; 4],
    // sampler2D atlas_nearest_sampler;
    // sampler2D atlas_linear_sampler;
}
```

WezTerm's shaders are written in WGSL. When wgpu compiles a `RenderPipeline`, it passes the WGSL source through `naga`, which parses WGSL, builds a typed IR, validates it, and emits SPIR-V. On the Linux Vulkan path, that SPIR-V binary is handed to the Mesa driver (RADV, ANV, or NVK) via `vkCreateShaderModule`. The driver compiles SPIR-V through Mesa NIR (the platform-independent IR) and then through ACO (RADV's AMDGPU code generator) or the LLVM-IR backend (ANV for Intel). This is structurally the same shader compilation path used by Bevy/wgpu applications (Ch40) and by Dawn-based WebGPU applications (Ch35).

### 2.7 Wayland Integration and IME

WezTerm uses `winit` as its cross-platform windowing crate. On Linux, `winit` provides both X11 (via `xcb`) and Wayland backends. The Wayland backend connects to the compositor via `wayland-client` bindings, creating a `wl_surface` and an `xdg_toplevel` shell role.

For fractional scaling, WezTerm implements the `wp_fractional_scale_v1` Wayland protocol extension (tracked as Issue #5149 and subsequently addressed). At a fractional scale factor (e.g. 1.5×), WezTerm renders at the logical pixel size and lets the compositor scale up for output, or renders at the physical pixel size with appropriate font metrics adjustments. Crashes related to fractional scaling with explicit sync (Issue #6998) were investigated as part of the broader Wayland explicit-sync adoption described in Ch45.

IME support on Wayland uses the `zwp_text_input_v3` protocol extension (tracked as Issue #1772). When IME is active, `winit`'s Wayland backend calls `zwp_text_input_v3.enable()`, signalling to the compositor's IM manager (e.g. `fcitx5`, `ibus`) to activate the input session for the focused window. WezTerm passes `set_ime_allowed(true)` through the `winit` API to trigger this. [Source](https://github.com/wezterm/wezterm/issues/1772)

Regarding DMA-BUF: WezTerm's rendering ultimately submits Wayland surface buffers through the compositor's normal frame-presentation path. On the wgpu Vulkan path, the Vulkan WSI (`VK_KHR_wayland_surface`) handles the connection between the Vulkan swapchain and the Wayland compositor. The compositor receives buffer references using `zwp_linux_dmabuf_v1` internally. WezTerm itself does not write DMA-BUF protocol code; the Mesa Vulkan driver's WSI implementation mediates this.

### 2.8 Built-in Multiplexer

WezTerm's multiplexer is a first-class architectural feature, not an add-on. The `mux` crate implements a `Mux` singleton that holds the full session graph: [Source](https://github.com/wezterm/wezterm/blob/main/mux/src/lib.rs)

```rust
// mux/src/lib.rs (simplified, showing key structures)
pub enum MuxNotification {
    PaneOutput(PaneId),
    PaneAdded(PaneId),
    PaneRemoved(PaneId),
    WindowCreated(WindowId),
    WindowRemoved(WindowId),
    WindowInvalidated(WindowId),
    TabAddedToWindow { tab_id: TabId, window_id: WindowId },
    Alert { pane_id: PaneId, alert: wezterm_term::Alert },
    // ...
}
```

The hierarchy is three-level: **Windows** contain **Tabs**, which contain **Panes** arranged in a binary tree layout. Each `Pane` is either a `LocalPane` (a native PTY process) or a `ClientPane` (a proxy for a remote pane via the mux protocol).

The **Domain** abstraction separates where panes are spawned from how they are rendered:

- `LocalDomain`: spawns native processes, WSL, or serial ports on the local machine.
- `RemoteSshDomain`: SSH sessions. Two modes: standard SSH (session ends on disconnect) or WezTerm multiplexed mode (requires `wezterm-mux-server` on the remote, survives reconnection).
- `ClientDomain`: proxy to a remote `wezterm-mux-server` reachable via SSH or TLS.
- `ExecDomain`: Lua-configurable wrapper for Docker, `systemd-run`, etc.

The mux client–server protocol uses PDUs (Protocol Data Units) encoded with LEB128-length framing: each frame carries `tagged_len` (LEB128), a `serial` number (LEB128), a PDU `ident` (LEB128), and a `bincode`-serialised payload. The `SessionHandler` on the server side performs **dirty-tracking**: it calls `pane.get_changed_since(seqno)` to find only the changed line ranges since the last acknowledged sequence number, sending incremental updates rather than full screen refreshes. This bandwidth optimisation is critical for high-latency SSH connections.

For the GUI client compositing multiple mux panes, `TermWindow::paint_impl()` iterates over all visible panes in the current tab, rendering each pane's `ClientPane` cached screen state into its allocated screen region. Pane splits are rendered as separator bars at the binary tree boundaries.

### 2.9 Pixel Graphics Protocols: Sixel, Kitty, and iTerm2

WezTerm supports all three major terminal graphics protocols: [Source](https://wezterm.org/features.html)

- **Sixel**: experimental support since release `20200620-160318-e00b076c`. Sixel byte streams are decoded by the VT parser in `wezterm-term` into pixel arrays, stored in `termwiz::image::ImageData`.
- **Kitty Graphics Protocol**: enabled by `enable_kitty_graphics = true` in the user configuration. Basic placement and animation are supported. A known conformance limitation: "The spec models image placements independently from terminal cells, but wezterm maps them to terminal cells at placement time. As a result, over-writing cells with text can poke holes in wezterm which may not appear in kitty." [Source](https://github.com/wezterm/wezterm/issues/986)
- **iTerm2**: iTerm2 inline image protocol, used by `wezterm imgcat` and compatible tools.

The image rendering pipeline is: decoded pixel data (stored as `ImageData` / `ImageDataType` in `termwiz::image`) → uploaded to a GPU texture via the `image_cache` in `GlyphCache` → sampled by the fragment shader and composited as a textured quad at the appropriate cell-grid coordinates. The `LfuCache<[u8; 32], DecodedImage>` uses a 32-byte hash of the raw image data as its key, so the same image payload appearing multiple times (e.g., in a repeated animation frame) is stored once.

For animated images, WezTerm stores the decoded frame sequence in the `GlyphCache` and uses `min_frame_duration` to pace playback without exceeding display refresh rates.

---

## 3. Alacritty Architecture

### 3.1 Crate Structure and Design Philosophy

Alacritty's design philosophy is articulated directly in its README: "Alacritty is a modern terminal emulator that comes with sensible defaults, but allows for extensive configuration. By integrating with other applications, rather than reimplementing their functionality, it manages to provide a flexible set of features with high performance." This means no built-in multiplexer (use tmux or zellij), no graphics protocols (Sixel, Kitty images), and no font ligatures by default (though ligature support for Linux was explored in PR #2677). [Source](https://github.com/alacritty/alacritty)

The Cargo workspace contains four crates: [Source](https://github.com/alacritty/alacritty)

| Crate | Role |
|---|---|
| `alacritty` | Main binary: rendering, windowing, event loop (`display/mod.rs`, `renderer/`) |
| `alacritty_terminal` | Reusable terminal emulation library: VT parser, cell grid, `Term<T>` |
| `alacritty_config` | Configuration parsing (TOML) |
| `alacritty_config_derive` | Procedural macros for config deserialization |

The separation of `alacritty_terminal` as an independently publishable library crate is analogous to WezTerm's `termwiz`: both projects factor the terminal state machine into a library usable by third parties. `alacritty_terminal` implements VT/ANSI parsing via its own `vte::ansi` module and exposes a `Term<EventListener>` type that represents the terminal grid. Third-party multiplexers (like zellij) can use `alacritty_terminal` as a VT emulation back-end.

The most recent stable release is Alacritty 0.17.0 (April 2026), which added `window.resize_increments` support on Wayland and TOML 1.1 syntax support. [Source](https://linuxiac.com/alacritty-0-17-terminal-emulator-released-with-wayland-resize-improvements/)

### 3.2 OpenGL Context Initialisation via glutin and winit

Alacritty uses `winit` for cross-platform windowing and `glutin` for OpenGL context creation. The `Display` type in `alacritty/src/display/mod.rs` owns the windowing and OpenGL state: [Source](https://github.com/alacritty/alacritty/blob/main/alacritty/src/display/mod.rs)

```rust
// alacritty/src/display/mod.rs (imports, showing key dependencies)
use glutin::context::{NotCurrentContext, PossiblyCurrentContext};
use glutin::display::GetGlDisplay;
use glutin::prelude::*;
use glutin::surface::{Surface, SwapInterval, WindowSurface};
use winit::dpi::PhysicalSize;
use winit::raw_window_handle::RawWindowHandle;
use crossfont::{Rasterize, Rasterizer, Size as FontSize};
use alacritty_terminal::term::{self, Term, TermDamage};
```

On Linux/Wayland, `glutin` creates an EGL context. The EGL path: `glutin` calls `eglGetPlatformDisplay(EGL_PLATFORM_WAYLAND_KHR, wl_display, NULL)`, which dispatches to the Mesa EGL implementation. Mesa EGL selects a DRM render node (e.g., `/dev/dri/renderD128`) and creates the EGL display. `eglCreateContext` with `EGL_CONTEXT_CLIENT_VERSION = 2` (ES 2.0) or `EGL_CONTEXT_CLIENT_VERSION = 3` (GL 3.3) creates the rendering context; Mesa maps this to a `gl_context` within the Mesa DRI driver.

Alacritty's renderer has two concrete implementations — `Gles2Renderer` (for OpenGL ES 2.0, for broader compatibility) and `Glsl3Renderer` (for desktop OpenGL 3.3 with GLSL 330 shaders). The `Renderer` struct holds the active variant: [Source](https://github.com/alacritty/alacritty/blob/main/alacritty/src/renderer/mod.rs)

```rust
// alacritty/src/renderer/mod.rs
use glutin::context::{ContextApi, GlContext, PossiblyCurrentContext};
use crate::config::debug::RendererPreference;

#[derive(Debug)]
enum TextRendererProvider {
    Gles2(Gles2Renderer),
    Glsl3(Glsl3Renderer),
}

#[derive(Debug)]
pub struct Renderer {
    text_renderer: TextRendererProvider,
    rect_renderer: RectRenderer,
    robustness: bool,
}
```

The `RendererPreference` configuration allows the user to force GLES2, GLSL3, or auto-select. Version 0.16.1 fixed crashes on GPUs with partial robustness support (`GL_KHR_robustness` / `GL_ARB_robustness`); the `robustness` field tracks whether the context supports it.

After rendering, `display.swap_buffers(&context)` calls `eglSwapBuffers`, which submits the rendered framebuffer to the Wayland compositor via `wl_egl_window` → `libwayland-egl` → `wl_buffer`. This is the standard EGL-on-Wayland presentation path: the Mesa EGL implementation handles `zwp_linux_dmabuf_v1` buffer creation and submission to the compositor transparently.

### 3.3 The Renderer: Instanced GL Quads

Alacritty's `Glsl3Renderer` uses **instanced rendering**: all terminal cell quads are submitted in a single `glDrawElementsInstanced` call. This is the architecture that made Alacritty's "500 FPS on a screen full of text" benchmark possible. [Source](https://jwilm.io/blog/announcing-alacritty/)

The renderer allocates a static index buffer for a single unit quad (indices `[0, 1, 3, 1, 2, 3]`) and a dynamic instance data buffer (`vbo_instance`) that is updated with per-cell data every frame. The OpenGL setup: [Source](https://github.com/alacritty/alacritty/blob/main/alacritty/src/renderer/text/glsl3.rs)

```rust
// alacritty/src/renderer/text/glsl3.rs
const BATCH_MAX: usize = 0x1_0000; // 65536 instances per batch

impl Glsl3Renderer {
    pub fn new() -> Result<Self, Error> {
        info!("Using OpenGL 3.3 renderer");
        let program = TextShaderProgram::new(ShaderVersion::Glsl3)?;

        unsafe {
            gl::Enable(gl::BLEND);
            gl::BlendFunc(gl::SRC1_COLOR, gl::ONE_MINUS_SRC1_COLOR);
            gl::DepthMask(gl::FALSE);

            gl::GenVertexArrays(1, &mut vao);
            gl::GenBuffers(1, &mut ebo);
            gl::GenBuffers(1, &mut vbo_instance);
            gl::BindVertexArray(vao);

            // Element buffer: indices for a single quad
            let indices: [u32; 6] = [0, 1, 3, 1, 2, 3];
            gl::BindBuffer(gl::ELEMENT_ARRAY_BUFFER, ebo);
            gl::BufferData(
                gl::ELEMENT_ARRAY_BUFFER,
                (6 * size_of::<u32>()) as isize,
                indices.as_ptr() as *const _,
                gl::STATIC_DRAW,
            );

            // Instance data buffer: per-cell attributes
            gl::BindBuffer(gl::ARRAY_BUFFER, vbo_instance);
            gl::BufferData(
                gl::ARRAY_BUFFER,
                (BATCH_MAX * size_of::<InstanceData>()) as isize,
                ptr::null(),
                gl::STREAM_DRAW,
            );
            // Vertex attribute pointers set via gl::VertexAttribDivisor(index, 1)
            // to advance per-instance rather than per-vertex
        }
    }
}
```

The blend function `gl::BlendFunc(gl::SRC1_COLOR, gl::ONE_MINUS_SRC1_COLOR)` implements **dual-source blending** (`GL_ARB_blend_func_extended`), which uses colour channels from two fragment shader outputs simultaneously. This enables subpixel antialiasing by blending the glyph's RGB channel contributions separately.

Alacritty does not perform selective redraw at the quad level; instead it re-uploads the full set of instance data every frame. This is made practical by the architecture's design: the `vbo_instance` buffer is mapped as `gl::STREAM_DRAW`, the CPU writes updated `InstanceData` structs for all visible cells, and a single `glDrawElementsInstanced` covers the whole terminal. The complete-frame-redraw strategy simplifies damage tracking while remaining efficient because the per-frame work is dominated by the PCIe upload of a few kilobytes of instance data, not by draw call overhead.

Alacritty does, however, track terminal damage at the line level (`TermDamage`) and uses `DamageTracker` to identify which grid regions changed. This damage information drives whether the display needs to be redrawn at all (if nothing changed, the frame is skipped), but when a redraw is needed, the full terminal content is re-submitted.

### 3.4 The Glyph Cache and Texture Atlas

Alacritty's glyph cache in `alacritty/src/renderer/text/glyph_cache.rs` uses a `HashMap<GlyphKey, Glyph>` with `ahash::RandomState` for fast hashing: [Source](https://github.com/alacritty/alacritty/blob/main/alacritty/src/renderer/text/glyph_cache.rs)

```rust
// alacritty/src/renderer/text/glyph_cache.rs
pub struct GlyphCache {
    cache: HashMap<GlyphKey, Glyph, RandomState>,
    rasterizer: Rasterizer,  // crossfont::Rasterizer
    pub font_key: FontKey,
    pub bold_key: FontKey,
    pub italic_key: FontKey,
    pub bold_italic_key: FontKey,
    pub font_size: crossfont::Size,
    font_offset: Delta<i8>,
    glyph_offset: Delta<i8>,
    metrics: Metrics,
    builtin_box_drawing: bool,
}

#[derive(Copy, Clone, Debug)]
pub struct Glyph {
    pub tex_id: GLuint,      // GL texture object ID
    pub multicolor: bool,    // true for colour emoji
    pub top: i16,
    pub left: i16,
    pub width: i16,
    pub height: i16,
    pub uv_bot: f32,         // UV coordinates in the atlas
    pub uv_left: f32,
    pub uv_width: f32,
    pub uv_height: f32,
}
```

The `Atlas` type in `alacritty/src/renderer/text/atlas.rs` uses a simple **shelf bin-packing** algorithm — not `guillotiere`, but a hand-written shelf allocator tuned for the monospace case where glyphs are roughly the same height: [Source](https://github.com/alacritty/alacritty/blob/main/alacritty/src/renderer/text/atlas.rs)

```rust
// alacritty/src/renderer/text/atlas.rs
pub const ATLAS_SIZE: i32 = 1024;

pub struct Atlas {
    id: GLuint,
    width: i32,
    height: i32,
    row_extent: i32,    // left-most free pixel in current row
    row_baseline: i32,  // top of current row
    row_tallest: i32,   // tallest glyph in current row
    is_gles_context: bool,
}
```

Each `Atlas` is a 1024×1024 `GL_RGBA` `GL_TEXTURE_2D`. When a glyph is rasterised, it is uploaded with `glTexSubImage2D` at the next available position in the current shelf row. When the shelf overflows, a new shelf is started at `row_baseline + row_tallest`. When the full texture fills up (`AtlasInsertError::Full`), Alacritty **appends a new `Atlas`** to the `Vec<Atlas>` in `Glsl3Renderer` rather than evicting or clearing existing glyphs. A font-or-DPI change triggers `clear()` (the `LoadGlyph` trait method), which resets the cache and starts fresh; but space exhaustion does not — existing glyphs in earlier atlases are preserved while the new atlas is appended. This contrasts with WezTerm's strategy: when WezTerm's `guillotiere`-backed atlas fills, it recreates the entire `GlyphCache` with a fresh atlas, flushing all entries.

Atlas creation uploads the texture with OpenGL calls appropriate to whether the context is GLES or desktop GL:

```rust
// alacritty/src/renderer/text/atlas.rs (Atlas::new, simplified)
unsafe {
    gl::PixelStorei(gl::UNPACK_ALIGNMENT, 1);
    gl::GenTextures(1, &mut id);
    gl::BindTexture(gl::TEXTURE_2D, id);
    gl::TexImage2D(gl::TEXTURE_2D, 0, gl::RGBA as i32,
        size, size, 0, gl::RGBA, gl::UNSIGNED_BYTE, ptr::null());
    gl::TexParameteri(gl::TEXTURE_2D, gl::TEXTURE_WRAP_S, gl::CLAMP_TO_EDGE as i32);
    gl::TexParameteri(gl::TEXTURE_2D, gl::TEXTURE_WRAP_T, gl::CLAMP_TO_EDGE as i32);
    gl::TexParameteri(gl::TEXTURE_2D, gl::TEXTURE_MIN_FILTER, gl::LINEAR as i32);
    gl::TexParameteri(gl::TEXTURE_2D, gl::TEXTURE_MAG_FILTER, gl::LINEAR as i32);
}
```

Linear filtering (`GL_LINEAR`) applies sub-pixel blending when sampling at non-integer UV coordinates, which arises at fractional scale factors.

### 3.5 Font Rasterisation: crossfont and FreeType

Alacritty uses `crossfont`, a crate it maintains itself (at [`github.com/alacritty/crossfont`](https://github.com/alacritty/crossfont)), for font loading and rasterisation. `crossfont` wraps native font engines per platform: **FreeType** on Linux and BSD, **Core Text** on macOS, **DirectWrite** on Windows.

```rust
// alacritty/src/renderer/text/glyph_cache.rs
use crossfont::{
    Error as RasterizerError, FontDesc, FontKey, GlyphKey, Metrics, Rasterize,
    RasterizedGlyph, Rasterizer, Size, Slant, Style, Weight,
};
```

On Linux, `crossfont::Rasterizer` calls into FreeType for glyph rasterisation. The rasteriser reads fontconfig settings (`antialias`, `rgba`, `lcdfilter`, `hintstyle`) and maps them to FreeType load flags, so Alacritty glyphs render consistently with system font rendering preferences. `crossfont` does not provide HarfBuzz shaping — Alacritty has no ligature support (the PR #2677 exploring HarfBuzz integration on Linux was not merged). Each Unicode codepoint is shaped to a glyph independently.

The `Rasterizer` produces `RasterizedGlyph` values:
```rust
// crossfont interface (from crates.io documentation)
pub struct RasterizedGlyph {
    pub character: char,
    pub top: i32,
    pub left: i32,
    pub width: i32,
    pub height: i32,
    pub buf: BitmapBuffer,  // RGB or RGBA pixel data
}
```

The `BitmapBuffer` is either `RGB` (for regular glyphs rendered with LCD subpixel antialiasing) or `RGBA` (for colour emoji). In the `Glyph::multicolor` field, Alacritty marks colour emoji to suppress foreground-colour tinting in the shader.

Alacritty version 0.17 added a built-in font for block element symbols in the Unicode range U+1FB82–U+1FB8B (part of the Symbols for Legacy Computing block), stored in `alacritty/src/renderer/text/builtin_font.rs`. These are vector-procedural glyphs rendered without FreeType, similar to WezTerm's `customglyph.rs` approach. [Source](https://github.com/alacritty/alacritty/releases)

### 3.6 Wayland Integration

Alacritty's Wayland path uses `winit`'s Wayland backend. The `winit` crate drives the `wayland-client` protocol bindings, creates the `wl_surface`, handles `xdg_toplevel` negotiation, and delivers input and resize events.

`glutin` integrates with Wayland via `libwayland-egl`: the `wl_egl_window` is created from the `wl_surface`, and EGL's `EGL_PLATFORM_WAYLAND_KHR` creates the EGL display. Mesa EGL then manages the DRI render node and submits buffers to the compositor using `zwp_linux_dmabuf_v1` internally. A historical Bugzilla entry (Red Hat #2025572) noted that Alacritty should depend on `libwayland-egl` explicitly to ensure this path functions correctly on Fedora-based systems.

Alacritty 0.17 added `window.resize_increments` support on Wayland, which allows compositors to constrain resize operations to cell-aligned increments — useful when floating windows are resized by dragging. [Source](https://linuxiac.com/alacritty-0-17-terminal-emulator-released-with-wayland-resize-improvements/)

IME support on Wayland is present but limited. The `winit` Wayland backend supports `zwp_text_input_v3` when `set_ime_allowed(true)` is called. A known issue (Alacritty #6644) notes that Alacritty does not set the `content_purpose` to `ContentPurpose::Terminal`, which means some IME implementations may offer non-terminal input modes (e.g. predictive text). IME is disabled in Vi mode on X11 (v0.17 change). [Source](https://github.com/alacritty/alacritty/issues/6644)

### 3.7 Graphics Protocol Support (or Lack Thereof)

As of Alacritty 0.17.0, **Alacritty does not support Sixel graphics, the Kitty Graphics Protocol, or the iTerm2 inline image protocol**. This is an intentional design decision, not an oversight. The maintainers' position is that image rendering in a terminal belongs at the multiplexer or application layer, not in the terminal emulator itself. [Source](https://www.arewesixelyet.com/)

It is important to distinguish the **Kitty keyboard protocol** (which Alacritty does support, with enhanced modifier key handling added in v0.16) from the **Kitty Graphics Protocol** (which Alacritty does not support). The Kitty keyboard protocol provides unambiguous encoding of key events (distinguishing e.g. Ctrl+[ from Escape); it has nothing to do with pixel graphics.

Community PRs and issues requesting Sixel support exist in the tracker but have remained closed. Users needing graphics in Alacritty are directed to use it alongside a multiplexer (tmux, zellij) or switch to WezTerm, Kitty, or foot.

---

## 4. Comparative Analysis

The following table compares WezTerm and Alacritty against Kitty and Ghostty (covered in Ch44). For Kitty and Ghostty entries, see Ch44 for primary sources; those columns are reproduced here for cross-reference.

| Feature | WezTerm | Alacritty | Kitty (Ch44) | Ghostty (Ch44) |
|---|---|---|---|---|
| **GPU API (default)** | OpenGL via glium | OpenGL/EGL via glutin + `gl` crate | OpenGL (C) | Metal (macOS) / OpenGL via GTK4 (Linux) |
| **GPU API (opt-in)** | wgpu (Vulkan on Linux) | — | — | Vulkan (in development) |
| **Shader language** | WGSL (wgpu path) / GLSL (GL path) | GLSL 3.3 / GLSL ES 2.0 | GLSL | GLSL (GTK4 path) |
| **Windowing** | winit | winit | hand-rolled (C, Cocoa/X11/Wayland) | GTK4 |
| **Font shaping** | HarfBuzz | None (per-codepoint) | HarfBuzz | HarfBuzz |
| **Ligatures** | Yes | No | Yes | Yes |
| **Built-in multiplexer** | Yes (tmux-compatible) | No | Limited (kitten ssh) | No |
| **Sixel** | Yes (experimental) | No | No | No |
| **Kitty Graphics Protocol** | Yes (with caveats) | No | Yes (origin) | Yes |
| **iTerm2 images** | Yes | No | No | No |
| **IME (Wayland)** | Yes (`zwp_text_input_v3`) | Partial | Yes | Yes |
| **Fractional scale** | Yes (`wp_fractional_scale_v1`) | Partial | Yes | Yes |
| **Built-in font for box drawing** | Yes (procedural via tiny_skia) | Yes (v0.17+) | Yes | Yes |
| **Written in** | Rust | Rust | C + Python | Zig |
| **Atlas allocator** | guillotiere | hand-written shelf | hand-written | hand-written |
| **Config format** | Lua | TOML | Python | UCL / HCL |

**Note**: The "Kitty Protocol" row in many community comparison tables is ambiguous. In this table it refers specifically to the **Kitty Graphics Protocol** (inline image rendering), not the **Kitty keyboard protocol** (which both Alacritty and WezTerm support).

---

## 5. Linux Graphics Stack Integration

### 5.1 WezTerm's wgpu Path through Mesa

When `front_end = "WebGpu"` is configured on Linux, WezTerm's rendering follows this stack:

```
WezTerm (Rust)
  → wgpu (Rust crate, high-level GPU API)
    → wgpu-hal Vulkan backend (Rust, uses ash for Vulkan FFI)
      → libvulkan.so / Vulkan loader
        → Mesa Vulkan driver (RADV / ANV / NVK)
          → Mesa NIR (compiler IR, platform-independent)
            → ACO backend (RADV/AMDGPU) or LLVM-IR (ANV/Intel) or NVK (NVIDIA)
              → kernel DRM / amdgpu / i915 / nouveau
                → KMS scanout → display
```

At the API boundary between wgpu and Mesa: `ash` (Rust Vulkan bindings) calls `vkCreateInstance`, `vkEnumeratePhysicalDevices`, `vkCreateDevice`, `vkCreateSwapchainKHR` (via `VK_KHR_wayland_surface`), and frame-by-frame `vkQueuePresentKHR`. Mesa's Vulkan driver (e.g., RADV) intercepts these calls at `libvulkan_radeon.so`.

WGSL shaders are compiled at pipeline creation time: `naga` parses the WGSL source, validates the type system and control flow, and emits SPIR-V binary. Mesa's Vulkan front-end (`anv_pipeline.c`, `radv_pipeline.c`) calls `vkCreateShaderModule` with this SPIR-V, which the driver passes through `spirv_to_nir()` to translate SPIR-V into Mesa NIR. From there, the per-driver backend (ACO for RADV, the Intel compiler for ANV) compiles NIR to GPU machine code. This is structurally identical to the shader compilation path described for Bevy (Ch40) and Dawn (Ch35).

This means WezTerm's wgpu path is a **concrete, real-world example** of the Mesa Vulkan stack described in Chapters 18 and 14 — not a game engine or a browser WebGPU implementation, but a terminal emulator.

### 5.2 Alacritty's EGL Path through Mesa

Alacritty's OpenGL rendering follows a different but equally concrete Mesa path:

```
Alacritty (Rust)
  → glutin (OpenGL context creation, Rust)
    → EGL (via libEGL.so, Mesa EGL implementation)
      → DRI driver interface (Mesa internal)
        → Mesa GL state tracker (gallium pipe_context)
          → Gallium3D driver (radeonsi / iris / nouveau)
            → kernel DRM / amdgpu / i915 / nouveau
              → KMS scanout → display
```

Or on Wayland via `wl_egl_window`:

```
Alacritty → glutin → EGL (EGL_PLATFORM_WAYLAND_KHR)
  → Mesa EGL → DRI render node (/dev/dri/renderD128)
  → zwp_linux_dmabuf_v1 buffer → wl_surface → compositor
```

GLSL shaders (`alacritty/res/glsl3/text.f.glsl`, `text.v.glsl`) are compiled by `glCompileShader` at startup. Mesa's GL compiler (`mesa/src/compiler/glsl/`) parses GLSL, builds Mesa IR (GLSL IR), and then converts to NIR for the Gallium3D back-end. The same NIR-to-ACO or NIR-to-LLVM-IR compilation path applies.

This makes Alacritty a concrete example of the Mesa EGL and OpenGL paths described in Chapters 19 and 4.

### 5.3 Wayland Protocol Usage Comparison

Both terminals consume the same set of Wayland protocols through `winit`'s Wayland backend:

| Protocol | WezTerm | Alacritty |
|---|---|---|
| `xdg_wm_base` / `xdg_toplevel` | Yes | Yes |
| `wp_fractional_scale_v1` | Yes | Partial |
| `zwp_text_input_v3` (IME) | Yes (Issue #1772) | Partial (Issue #5065) |
| `wl_egl_window` / EGL | Via wgpu Vulkan WSI (wgpu path) | Yes (glutin) |
| `zwp_linux_dmabuf_v1` | Via Mesa Vulkan WSI | Via Mesa EGL |
| `wp_presentation` | Via winit | Via winit |

---

## 6. Performance Characteristics

### 6.1 GPU Memory Usage

**Glyph atlas**: WezTerm uses a single dynamically-sized 2D RGBA texture atlas backed by `guillotiere` (configurable size, typically 8–16 MB of VRAM). Alacritty uses a `Vec<Atlas>` of fixed 1024×1024 `GL_RGBA` textures; each atlas consumes 4 MB of GPU VRAM, and additional atlases are appended when the current one fills. For a typical monospace terminal session the single default atlas is sufficient; heavy emoji or Unicode usage may trigger a second. WezTerm's additional `image_cache` for pixel graphics protocols can consume further GPU memory proportional to the images displayed.

**Vertex buffers**: Alacritty's `vbo_instance` for a 200-column × 50-row terminal holds at most 10,000 `InstanceData` structs. Each struct carries cell position, glyph UV coordinates, and colour data — roughly 80–120 bytes per instance, so ~1 MB or less per frame. WezTerm's quad allocator uses a similar order of magnitude.

**Framebuffer**: both terminals render into a `wl_egl_window` or Vulkan swapchain framebuffer sized to the physical window pixel dimensions. At 1920×1080 with 8-bit colour, this is 8 MB for a double-buffered configuration.

### 6.2 Power Consumption

The wgpu Vulkan path (WezTerm) and the EGL OpenGL path (Alacritty) impose fundamentally different CPU and GPU loads for a terminal workload:

- **Alacritty (OpenGL/EGL)**: the GL driver submits work to the GPU via kernel DRM using the `ioctl(DRM_IOCTL_I915_GEM_EXECBUFFER2)` / `ioctl(AMDGPU_CS)` path. For a static terminal (no output), damage tracking skips the redraw entirely, and GPU work is zero. Under output load, the full instance buffer is re-uploaded per frame. For a static terminal on an integrated GPU, GPU utilisation is near zero between frames.
- **WezTerm (wgpu/Vulkan, opt-in)**: the Vulkan command buffer submission path involves more explicit synchronisation objects (`VkSemaphore`, `VkFence`) and pipeline barriers, adding a small constant overhead per frame. However, WezTerm's triple-layer quad allocator and shape cache substantially reduce the per-frame CPU work for static content, so the GPU stays in a low-power state between frames. The default OpenGL path behaves similarly to Alacritty.

**Note: needs verification.** Precise power measurements comparing the two terminals under controlled conditions (e.g. power/perf tooling with `powertop` or `perf stat`) are not available in public literature as of 2026. Users seeking to minimise GPU power draw should consult their specific hardware's vendor tools.

### 6.3 Input-to-Visual Latency

Alacritty was designed with input latency as a primary metric. The original announcing post described the VSync-based architecture: "by capping frame rates at 60 Hz, the VT parser gains ~14.7 ms per frame to process incoming data, while the renderer uses only ~2 ms." [Source](https://jwilm.io/blog/announcing-alacritty/) Alacritty's multithreaded architecture separates the PTY read loop, VT parser, and render loop, allowing all three to proceed concurrently without the parser blocking on the display.

WezTerm also uses an asynchronous architecture with background PTY read threads (one per `LocalPane`), with the main thread handling rendering driven by Wayland frame callbacks. The built-in multiplexer adds a layer of protocol round-trips for remote panes (`ClientPane`), which can increase effective latency on high-latency links unless the mux server is colocated.

---

## Roadmap

### Near-term (6–12 months)

- **WezTerm: stabilise wgpu/Vulkan as default frontend on Linux.** The `WebGpu` front-end (wgpu 0.x → Vulkan on Linux via `ash`) is currently opt-in. Active issues (#6815, #6998, #7070) track crashes with fractional-scale Wayland compositors and explicit-sync; once resolved, the intent is to promote `WebGpu` to the default on systems where the Vulkan driver reports feature-complete support. [Source](https://github.com/wezterm/wezterm/issues/2756)
- **WezTerm: full `zwp_text_input_v3` IME coverage.** Issue #1772 tracks incomplete IME support on Wayland (pre-edit rendering, commit-string races); targeted for closure in the next stable release cycle. [Source](https://github.com/wezterm/wezterm/issues/1772)
- **WezTerm: `wp_fractional_scale_v1` crash fix.** Fractional scaling causes startup panics when the wgpu Vulkan path is active (#5149, #6998); the fix requires coordinating `wgpu-hal` swapchain resize with Wayland surface configure events. [Source](https://github.com/wezterm/wezterm/issues/5149)
- **Alacritty: Sixel tracking issue resolution.** Issue #910 on the Alacritty tracker documents ongoing pressure for Sixel and Kitty Graphics Protocol support; maintainers have not rejected the feature outright, but the deliberate minimalism philosophy means any implementation would need to preserve latency guarantees. [Source](https://github.com/alacritty/alacritty/issues/910) Note: needs verification of current open/closed status.
- **winit 0.31+ Wayland backend improvements.** Both terminals depend on `winit` for windowing; the winit 0.31 release cycle targets improved `wp_fractional_scale_v1` handling and `zwp_linux_drm_syncobj_v1` explicit-sync support on Wayland, which directly benefits both WezTerm and Alacritty. [Source](https://github.com/rust-windowing/winit)

### Medium-term (1–3 years)

- **Alacritty: Vulkan renderer investigation.** Issue #183 ("Add a Vulkan Renderer") has been open since early Alacritty history; a community fork (`w23/alacritty`, branch `new-renderer-pr`) prototyped a full-screen single-pass shader renderer. Whether this is adopted into mainline depends on whether Alacritty's maintainers accept the architectural complexity trade-off. [Source](https://github.com/alacritty/alacritty/issues/183)
- **WezTerm: `glium` OpenGL backend deprecation.** `glium` is an older safe GL wrapper no longer actively developed; as `wgpu` matures and GPU coverage widens (including wgpu's own OpenGL ES fallback for devices without Vulkan), WezTerm is expected to deprecate the `glium` path and unify on wgpu with the GL backend handling non-Vulkan hardware. Note: needs verification of official timeline.
- **`wgpu` 3.x HAL improvements benefiting both terminals.** The wgpu team is working on tighter integration of DMA-BUF import into the `wgpu-hal` Vulkan backend, which would allow WezTerm to implement zero-copy pixel graphics protocol rendering (DMA-BUF → `VkImage` → swapchain blit) without staging buffers. [Source](https://github.com/gfx-rs/wgpu)
- **Alacritty: multi-GPU and power-aware adapter selection.** On hybrid Intel+dGPU laptops, Alacritty currently relies on EGL's default device selection; a future enhancement would expose explicit GPU selection (similar to WezTerm's `webgpu_preferred_adapter`) to allow users to pin the terminal to the integrated GPU for power savings. Note: needs verification.
- **`naga` WGSL-to-SPIR-V pipeline hardening for terminal shaders.** WezTerm's WGSL shaders expose edge cases in `naga`'s validator and SPIR-V emitter; active naga issues track round-trip correctness for integer operations used in glyph index packing. Fixes here improve WezTerm shader reliability across RADV, ANV, and NVK. [Source](https://github.com/gfx-rs/naga)

### Long-term

- **Alacritty as a pure text renderer with external image overlay protocol.** The project's stated philosophy may converge on a standardised out-of-band image rendering mechanism (e.g., a Wayland sub-surface approach where the compositor or a separate process composites pixel graphics over the terminal surface) rather than in-process protocol implementation. This would keep Alacritty's renderer minimal while enabling image support via external tooling such as Überzug++. Note: speculative, based on design discussions.
- **WezTerm multiplexer protocol standardisation.** The WezTerm mux PDU protocol is currently WezTerm-proprietary; long-term there has been community interest in defining a standardised open terminal multiplexer protocol that multiple terminals and clients (including TUI frameworks using `termwiz`) could interoperate with. Note: speculative.
- **wgpu WebGPU backend portability enabling WezTerm WASM/browser targets.** As wgpu's WebGPU backend (targeting the browser via `wasm-bindgen`) matures, WezTerm's GPU rendering code — which is already abstracted behind `RenderContext` — becomes portable to WASM, enabling a fully GPU-accelerated browser-hosted WezTerm instance. This would reuse the same WGSL shaders that Mesa compiles on Linux. Note: speculative architectural direction.
- **Explicit sync (`zwp_linux_drm_syncobj_v1`) support in both terminals' Wayland paths.** As KMS explicit sync lands broadly in Mesa and compositor stacks (Ch45), both terminals will need to thread `VkSemaphore`/`VkFence` handles through their Wayland WSI paths to avoid implicit sync fallbacks that increase frame latency on modern AMD and Intel hardware. [Source](https://gitlab.freedesktop.org/wayland/wayland-protocols)

---

## 7. Integrations

- **Ch44 (Terminal GPU Rendering Architectures)** — This chapter extends Ch44's coverage to WezTerm and Alacritty. Ch44 covers Kitty and Ghostty with the same depth; together they survey the four major GPU-accelerated Rust-or-C terminals of 2026.

- **Ch43 (Terminal Pixel Protocols)** — Ch43 documents the Sixel, Kitty Graphics Protocol, and iTerm2 wire formats. WezTerm implements all three (Sixel and Kitty with caveats); Alacritty implements none. The pixel data decoded per Ch43's protocol parsing becomes a GPU texture via the `image_cache` pipeline described in section 2.9.

- **Ch45 (Terminal Integration with the Compositor Stack)** — Both terminals are Wayland clients. Ch45 documents the `wl_surface` lifecycle, DMA-BUF zero-copy buffer submission, `wp_presentation` frame timing, and explicit sync. WezTerm's wgpu Vulkan WSI path and Alacritty's EGL path are both concrete examples of the Ch45 surface submission model. The `zwp_text_input_v3` IME integration in both terminals is documented in Ch45's Wayland protocol section.

- **Ch18 (Mesa Vulkan Drivers: RADV, ANV, NVK)** — WezTerm's wgpu/WebGPU path uses RADV, ANV, or NVK depending on the GPU. The `ash` Vulkan bindings in wgpu-hal call the same `VkDevice` API entry points that any Vulkan application uses; Mesa's Vulkan driver is indifferent to whether the caller is WezTerm, Bevy, or Vulkan-CTS.

- **Ch19 (Mesa OpenGL)** — Alacritty's OpenGL renderer uses Mesa's GL state tracker (Gallium3D) via EGL. The `glCompileShader` / `glDrawElementsInstanced` calls in Alacritty's renderer traverse the same Mesa GL code paths as any desktop OpenGL application.

- **Ch14 (Mesa Shader Compiler: NIR, ACO, LLVM)** — Both WezTerm's WGSL shaders (via naga → SPIR-V → NIR) and Alacritty's GLSL shaders (via Mesa GLSL compiler → NIR) are compiled through Mesa NIR. WezTerm's shader compilation path is structurally identical to Bevy/wgpu (Ch40) and Dawn (Ch35).

- **Ch40 (Bevy and wgpu)** — WezTerm's wgpu path uses the same `wgpu` crate that Bevy's rendering engine is built on. The `wgpu::Instance`, `wgpu::Adapter`, `wgpu::Device`, `wgpu::Queue` initialisation in `WebGpuState::new_impl()` is structurally the same as Bevy's render world initialisation. WezTerm is an example of wgpu used for a non-game-engine application.

- **Ch152 (The Rust GPU Ecosystem: ash, wgpu, naga, and Bevy)** — WezTerm depends on `wgpu`, which in turn depends on `ash` for Vulkan bindings, `wgpu-hal` for the HAL layer, and `naga` for shader translation. Alacritty uses none of these; its GPU dependency graph is `glutin` (EGL context creation) + `gl` crate (gl_generator raw GL function pointers). The contrast illustrates two ends of the Rust GPU library ecosystem.

---

## References

- WezTerm repository: [github.com/wezterm/wezterm](https://github.com/wezterm/wezterm)
- WezTerm features documentation: [wezterm.org/features.html](https://wezterm.org/features.html)
- WezTerm WebGPU preferred adapter configuration: [wezterm.org/config/lua/config/webgpu_preferred_adapter.html](https://wezterm.org/config/lua/config/webgpu_preferred_adapter.html)
- WezTerm Kitty Graphics Protocol support (Issue #986): [github.com/wezterm/wezterm/issues/986](https://github.com/wezterm/wezterm/issues/986)
- WezTerm IME / zwp_text_input_v3 (Issue #1772): [github.com/wezterm/wezterm/issues/1772](https://github.com/wezterm/wezterm/issues/1772)
- WezTerm fractional scale support (Issue #5149): [github.com/wezterm/wezterm/issues/5149](https://github.com/wezterm/wezterm/issues/5149)
- WezTerm mux crate source: [github.com/wezterm/wezterm/blob/main/mux/src/lib.rs](https://github.com/wezterm/wezterm/blob/main/mux/src/lib.rs)
- Alacritty repository: [github.com/alacritty/alacritty](https://github.com/alacritty/alacritty)
- Alacritty 0.17 release notes: [linuxiac.com/alacritty-0-17-terminal-emulator-released-with-wayland-resize-improvements/](https://linuxiac.com/alacritty-0-17-terminal-emulator-released-with-wayland-resize-improvements/)
- Alacritty original announcement: [jwilm.io/blog/announcing-alacritty/](https://jwilm.io/blog/announcing-alacritty/)
- crossfont repository: [github.com/alacritty/crossfont](https://github.com/alacritty/crossfont)
- wgpu-hal README: [github.com/gfx-rs/wgpu/blob/trunk/wgpu-hal/README.md](https://github.com/gfx-rs/wgpu/blob/trunk/wgpu-hal/README.md)
- guillotiere crate: [github.com/nical/guillotiere](https://github.com/nical/guillotiere)
- Are We Sixel Yet?: [arewesixelyet.com](https://www.arewesixelyet.com/)
- termwiz crate: [crates.io/crates/termwiz](https://crates.io/crates/termwiz)
- Terminal feature comparison: [tmuxai.dev/terminal-compatibility/](https://tmuxai.dev/terminal-compatibility/)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
