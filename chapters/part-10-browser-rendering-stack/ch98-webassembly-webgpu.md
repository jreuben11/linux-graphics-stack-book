# Chapter 98: WebAssembly and WebGPU as a Deployment Target

**Target audiences**: Application developers building portable GPU software, Rust/C++ graphics developers, browser engineers, and wgpu users who want to understand the full stack from native Linux Vulkan through to in-browser WebGPU execution.

---

## Table of Contents

1. [Introduction: The Portability Promise](#1-introduction-the-portability-promise)
2. [WebAssembly Fundamentals](#2-webassembly-fundamentals)
3. [Emscripten: C++ to WebGPU via emdawnwebgpu](#3-emscripten-c-to-webgpu-via-emdawnwebgpu)
4. [wgpu: Rust's Portable GPU Abstraction](#4-wgpu-rusts-portable-gpu-abstraction)
5. [wasm-bindgen and Browser Integration](#5-wasm-bindgen-and-browser-integration)
6. [WGSL: The Shader Language](#6-wgsl-the-shader-language)
7. [How Browser WebGPU Maps to Linux Vulkan](#7-how-browser-webgpu-maps-to-linux-vulkan)
8. [WASM SIMD for CPU-Side Computation](#8-wasm-simd-for-cpu-side-computation)
9. [Real-World Use Cases](#9-real-world-use-cases)
10. [Build Tooling and Distribution](#10-build-tooling-and-distribution)
11. [Integrations](#11-integrations)

---

## 1. Introduction: The Portability Promise

The GPU programming landscape has long fragmented along platform boundaries: Vulkan on Linux, Metal on macOS, Direct3D 12 on Windows. WebGPU and WebAssembly together offer a credible escape from this fragmentation. A single codebase written in Rust with [wgpu](https://github.com/gfx-rs/wgpu) or in C++ with [Emscripten](https://emscripten.org/) and [emdawnwebgpu](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/emdawnwebgpu/pkg/README.md) can compile to:

- **Native Linux binary** — dispatching to Vulkan through Mesa's Vulkan drivers (RADV, ANV, NVK, etc.)
- **Native macOS binary** — dispatching to Metal
- **Browser WebAssembly module** — dispatching to the browser's WebGPU implementation (Dawn in Chrome, WebGPU-RS in Firefox), which itself translates to Vulkan/Metal/D3D12 underneath

This matters for several reasons that go beyond developer convenience. GPU-accelerated tools can ship to users who will not install native software: WebGPU-backed shader editors, image processors, and simulation tools run inside any modern browser tab. Machine-learning inference that previously required a GPU server can execute locally in the browser via WebGPU compute shaders — eliminating network round-trips and keeping user data on-device. Portable game engines like Bevy and Godot 4 can target web and desktop from a single render graph implementation.

The Linux angle is especially interesting: on Linux, the same Mesa Vulkan drivers that power a native Vulkan game also sit underneath Chrome's and Firefox's WebGPU implementations. A RADV-based workstation runs the exact same physical GPU silicon regardless of whether a Rust wgpu app compiles to `x86_64-unknown-linux-gnu` or `wasm32-unknown-unknown` — only the API translation path differs.

### 1.1 Why Not Just Ship Native Binaries?

The obvious question is why portable GPU deployment matters when native Linux packaging tools (Flatpak, AppImage, Snap) already provide good distribution. The answer is distribution friction. A Flatpak still requires user action: finding, downloading, and installing a package. A browser tab requires nothing beyond a URL. For tools used occasionally — a one-time mesh converter, a shader prototyping environment, a machine learning demo — the zero-install browser path reaches orders of magnitude more users.

Beyond convenience, the browser sandbox provides meaningful security properties. A WebGPU/WASM application cannot read arbitrary files or establish arbitrary network connections; the browser's permission model contains it. For professional tools handling sensitive user data (images, 3D scans, medical imagery), processing in-browser avoids transmitting that data to a server at all.

### 1.2 The Execution Model at a Glance

Understanding how these pieces assemble is helpful before diving into each layer:

```
┌─────────────────── Browser Tab ──────────────────────────┐
│  HTML + JavaScript                                        │
│       ↓                                                   │
│  WASM module (compiled from Rust or C++)                  │
│       ↓ wasm-bindgen / JS interop                         │
│  navigator.gpu  (WebGPU JS API)                           │
│       ↓                                                   │
│  Dawn (Chrome) / WebGPU-RS (Firefox)                     │
│       ↓                                                   │
│  Vulkan (Mesa RADV/ANV/NVK on Linux)                      │
│       ↓                                                   │
│  Physical GPU (AMD, Intel, NVIDIA)                        │
└──────────────────────────────────────────────────────────┘
```

Each arrow in this stack represents a translation or abstraction boundary. The developer writes WGSL shaders and wgpu/WebGPU API calls once; the runtime selects the appropriate path based on whether the code runs natively or in the browser.

---

## 2. WebAssembly Fundamentals

WebAssembly (WASM) is a binary instruction format designed as a portable compilation target for high-level languages. It is specified by the W3C and is supported natively by all major browsers and by standalone runtimes such as Wasmtime and WasmEdge. [Source: WebAssembly specification](https://webassembly.github.io/spec/core/)

### 2.1 Binary Format

A `.wasm` file begins with a four-byte magic header followed by a four-byte version:

```
00 61 73 6D   ; magic: \0asm
01 00 00 00   ; version: 1
```

The body consists of a sequence of **sections**, each identified by a one-byte section id and a LEB128-encoded byte length. The standard section types are: [Source: WebAssembly 3.0 spec §Binary/Modules](https://webassembly.github.io/spec/core/binary/modules.html)

| Id | Section    | Purpose                                            |
|----|------------|----------------------------------------------------|
|  1 | Type       | Function signature declarations                    |
|  2 | Import     | Imports from the host environment                  |
|  3 | Function   | Maps function indices to type indices              |
|  4 | Table      | Indirect function call tables                      |
|  5 | Memory     | Linear memory declarations                        |
|  6 | Global     | Mutable and immutable global variables            |
|  7 | Export     | Names exported to the host                        |
|  8 | Start      | Optional module entry point                       |
|  9 | Element    | Table initialization data                         |
| 10 | Code       | Function bodies                                   |
| 11 | Data       | Memory initialization data                        |
|  0 | Custom     | Debug info, DWARF, name sections (ignored by VM)  |

### 2.2 Linear Memory Model

WASM modules see a single flat address space called **linear memory**, exposed to the host as an `ArrayBuffer` or `SharedArrayBuffer`. Memory is allocated in pages of 64 KB each; modules declare an initial page count and an optional maximum. All pointer arithmetic happens within this linear address space — there is no virtual memory, no kernel page tables, and no `mmap`. The host (JavaScript or Wasmtime) grants the module access to a contiguous region; all WASM load/store instructions validate against that region's bounds.

This design has a direct implication for GPU work: data in WASM linear memory must be explicitly copied to GPU buffers via the WebGPU API (`wgpuQueueWriteBuffer` or `queue.write_buffer`). Zero-copy sharing between WASM linear memory and the GPU is not possible in the current WebGPU spec.

### 2.3 WASI: WebAssembly System Interface

WASI provides a capability-based POSIX-like API for WASM modules running outside browsers: filesystem access (`wasi_snapshot_preview1::fd_read`, `fd_write`), networking, clocks, and environment variables. WASI is not used for browser-deployed GPU applications (the browser provides its own WebGPU and DOM APIs), but it is the foundation for server-side WASM runtimes that might eventually host GPU compute offload. [Source: WASI GitHub](https://github.com/WebAssembly/WASI)

### 2.4 Threading: SharedArrayBuffer and Atomics

WASM threading maps directly onto the browser's Web Workers model. When a WASM module's linear memory is backed by a `SharedArrayBuffer`, multiple workers can instantiate the same module and share that memory. The `memory.atomic.wait` and `memory.atomic.notify` instructions correspond to `Atomics.wait` / `Atomics.notify` in JavaScript. Crucially, `SharedArrayBuffer` requires the page to be **cross-origin isolated**, enforced by HTTP response headers:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Without these COOP/COEP headers, `SharedArrayBuffer` is unavailable and threaded WASM will fail to initialize. [Source: web.dev COOP/COEP guide](https://web.dev/articles/coop-coep)

### 2.5 SIMD Extension (v128)

The WASM SIMD proposal adds a single 128-bit vector type `v128` and a set of lane-typed operations: `f32x4.add`, `i32x4.mul`, `f32x4.sqrt`, `i8x16.shuffle`, and so on. The type is opaque at the WASM level — interpretation as `f32x4`, `i32x4`, `i64x2`, etc. is determined by the operation, not the type. SIMD is enabled at compile time and is distinct from threading; it does not require cross-origin isolation. [Source: WebAssembly SIMD proposal](https://github.com/WebAssembly/simd)

### 2.6 Execution Model

Browsers JIT-compile WASM. V8 (Chrome) uses a two-tier strategy: Liftoff for fast baseline compilation (single-pass, generates slower code) followed by TurboFan for optimized re-compilation of hot functions. SpiderMonkey (Firefox) similarly uses a baseline compiler followed by IonMonkey for hot paths. Standalone runtimes can use AOT compilation via `wasm2c` (compiles WASM to C) or Cranelift (used by Wasmtime). The JIT layer is the primary source of WebGPU's overhead compared to native Vulkan — not the API translation itself, but the V8 JavaScript glue between the WASM module and the browser's WebGPU implementation.

### 2.7 Upcoming Proposals

The **garbage collection** (WasmGC) proposal adds structured heap types and GC'd references, enabling managed languages (Kotlin, Java, Dart) to compile to WASM without bundling a GC runtime. The **reference types** proposal adds `externref` and `funcref` for holding opaque host values (such as GPU objects) inside WASM tables. Neither is required for GPU work — wgpu and emdawnwebgpu handle GPU object lifetimes through linear memory handles — but they improve the ecosystem for WASM applications that mix GPU work with managed-language logic.

The **memory64** proposal extends the linear memory model to 64-bit addressing, allowing modules to address more than 4 GB of memory. This is particularly relevant for ML inference workloads where model weights may exceed 4 GB. The proposal is implemented behind flags in V8 and SpiderMonkey as of 2026.

The **component model** (part of WASI Preview 2) defines composable interfaces and types for linking WASM modules, opening the door to sandboxed GPU compute modules that can be linked with untrusted content while sharing the same GPU device — though this use case is still at the research stage.

### 2.8 wasm32-unknown-unknown vs wasm32v1-none

The standard Rust WASM target is `wasm32-unknown-unknown`. This target assumes the "unknown" environment, which historically allowed Rust to enable WASM proposals that the host environment might not support. A newer target, `wasm32v1-none`, was stabilised in Rust 1.84 and explicitly targets the WebAssembly 1.0 MVP with no extensions enabled by default. Extensions like SIMD128 or threading must be opted in explicitly via target features. For GPU applications that have no special WASM feature requirements beyond what the browser provides, `wasm32-unknown-unknown` remains the practical choice as it is better supported by `wasm-pack` and the broader Rust WASM ecosystem. [Source: wgpu issue #6826](https://github.com/gfx-rs/wgpu/issues/6826)

---

## 3. Emscripten: C++ to WebGPU via emdawnwebgpu

### 3.1 The Emscripten Toolchain

Emscripten is an LLVM-based compiler toolchain that translates C and C++ to WebAssembly. Its core tools are:

- `emcc` / `em++` — drop-in replacements for `gcc`/`g++` that target WASM
- `emar` — WebAssembly-targeting `ar`
- `emcmake` / `emmake` — configure CMake or Make to use the Emscripten toolchain

[Source: Emscripten documentation](https://emscripten.org/)

### 3.2 emdawnwebgpu: The Current WebGPU Path

Historically, Emscripten bundled its own WebGPU bindings enabled via `-sUSE_WEBGPU=1`. Those bindings are now **deprecated** — they tracked an older `webgpu.h` revision and were unmaintained. A linker warning directs users to the replacement. [Source: Emscripten deprecation PR #24220](https://github.com/emscripten-core/emscripten/pull/24220)

The current path is **emdawnwebgpu**, maintained by the Dawn team. It is Dawn's implementation of the `webgpu.h` C API, layered on top of the browser's JavaScript WebGPU API. [Source: emdawnwebgpu README](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/emdawnwebgpu/pkg/README.md)

Three ways to include it in a build:

```bash
# Simplest: use the Emscripten built-in port (requires Emscripten 4.0.10+)
emcc main.cpp --use-port=emdawnwebgpu -o out.html

# CMake cross-compilation (recommended for larger projects)
emcmake cmake -B build-web -DDAWN_FETCH_DEPENDENCIES=ON
cmake --build build-web
```

For CMake projects, link against `emdawnwebgpu_cpp`:

```cmake
# CMakeLists.txt (web target)
target_link_libraries(myapp PRIVATE emdawnwebgpu_cpp)
target_link_options(myapp PRIVATE
    "-sASYNCIFY=1"        # Required for async adapter/device requests
    "-sUSE_GLFW=3"        # If using GLFW for windowing
    "--closure=1"         # Minimize JS glue code in release builds
)
```

### 3.3 The Async Adapter/Device Request Pattern

Browser WebGPU is fundamentally asynchronous: `navigator.gpu.requestAdapter()` returns a JavaScript Promise. Emscripten's ASYNCIFY transforms C/C++ code that calls `emscripten_sleep()` (or async WebGPU functions) into code that can be suspended and resumed, allowing synchronous-looking C++ to interoperate with Promise-based APIs.

The call chain from C++ to the browser:

```cpp
// C++ (compiled with ASYNCIFY=1 via emdawnwebgpu)
#include <webgpu/webgpu_cpp.h>

wgpu::Instance instance = wgpu::CreateInstance();

// Calls navigator.gpu.requestAdapter() in JS, suspends via ASYNCIFY,
// resumes when the Promise resolves
wgpu::Adapter adapter;
instance.RequestAdapter(
    nullptr,
    [](WGPURequestAdapterStatus status, WGPUAdapter raw, char const*, void* userdata) {
        *reinterpret_cast<wgpu::Adapter*>(userdata) =
            wgpu::Adapter::Acquire(raw);
    },
    &adapter);

// Similarly for device
wgpu::Device device;
adapter.RequestDevice(
    nullptr,
    [](WGPURequestDeviceStatus status, WGPUDevice raw, char const*, void* userdata) {
        *reinterpret_cast<wgpu::Device*>(userdata) =
            wgpu::Device::Acquire(raw);
    },
    &device);
```

For the render loop, use Emscripten's main loop API instead of a blocking `while` — the browser requires cooperative yielding to handle events:

```cpp
#include <emscripten.h>

void RenderFrame() {
    // ... encode and submit GPU commands
}

int main() {
    // ... setup adapter, device, swapchain
    emscripten_set_main_loop(RenderFrame, 0, false);
    return 0;
}
```

ASYNCIFY comes at a cost: it can increase binary size significantly (roughly 1.5–2× depending on the code) due to the coroutine transformation it applies to functions that might suspend. A compile-time `ASYNCIFY_ONLY` list can scope the transformation to reduce overhead.

### 3.4 WGSL Shaders Are Shared

One of the most practical payoffs of the WebGPU model is shader portability. WGSL shader source files are consumed identically by the native Dawn library and the browser's WebGPU implementation:

```cpp
// Same shader string compiles natively (via Dawn→Vulkan/Metal)
// and in WASM (via emdawnwebgpu→browser WebGPU)
const char* kShaderSource = R"(
@vertex fn vs_main(@builtin(vertex_index) idx: u32) -> @builtin(position) vec4f {
    var pos = array<vec2f, 3>(
        vec2f(0.0,  0.5),
        vec2f(-0.5, -0.5),
        vec2f(0.5, -0.5)
    );
    return vec4f(pos[idx], 0.0, 1.0);
}

@fragment fn fs_main() -> @location(0) vec4f {
    return vec4f(0.4, 0.6, 0.9, 1.0);
}
)";
```

The cross-platform demo at [github.com/kainino0x/webgpu-cross-platform-demo](https://github.com/kainino0x/webgpu-cross-platform-demo) demonstrates this pattern with CMake, Dawn, and emdawnwebgpu.

---

## 4. wgpu: Rust's Portable GPU Abstraction

### 4.1 Architecture

wgpu is a safe-Rust GPU library implementing the WebGPU API surface across all major graphics backends. As of v29.0.3 (May 2026), it supports: [Source: wgpu GitHub](https://github.com/gfx-rs/wgpu)

| Backend                | Platform          | WASM? |
|------------------------|-------------------|-------|
| Vulkan (via `ash`)     | Linux, Android    | No    |
| Metal (via `objc2`)    | macOS, iOS        | No    |
| D3D12 (via `windows-rs`) | Windows         | No    |
| WebGPU (via `web-sys`) | Browser           | Yes   |
| WebGL2 (`webgl` feature) | Browser (fallback) | Yes |
| OpenGL ES              | Linux embedded    | No    |

The layered architecture:

```
┌─────────────────────────────────────────────────────────┐
│                    wgpu (public API)                     │
│   wgpu::Instance, Device, Queue, RenderPipeline, ...     │
├─────────────────────────────────────────────────────────┤
│                   wgpu-core                              │
│   Resource tracking, validation, command encoding        │
├─────────────────────────────────────────────────────────┤
│                    wgpu-hal                              │
│   Hardware abstraction: vulkan.rs, metal.rs, dx12.rs,   │
│   gles.rs, webgpu.rs                                     │
├────────────┬────────────┬───────────────┬───────────────┤
│   Vulkan   │   Metal    │     D3D12     │   WebGPU JS   │
│   (Mesa,   │  (macOS)   │   (Windows)   │ (web-sys/wasm)│
│  NVIDIA)   │            │               │               │
└────────────┴────────────┴───────────────┴───────────────┘
```

The minimum supported Rust version is **1.87**. [Source: wgpu README](https://github.com/gfx-rs/wgpu)

### 4.2 Core Initialization Pattern

The same init code works on native Linux (Vulkan) and in the browser (WebGPU). wgpu selects the appropriate backend automatically:

```rust
use wgpu::*;

async fn init() -> (Device, Queue) {
    // On Linux native: creates a Vulkan instance
    // In WASM: wraps the browser's navigator.gpu
    let instance = Instance::new(&InstanceDescriptor {
        backends: Backends::all(),
        ..Default::default()
    });

    let adapter = instance
        .request_adapter(&RequestAdapterOptions {
            power_preference: PowerPreference::HighPerformance,
            compatible_surface: None,
            force_fallback_adapter: false,
        })
        .await
        .expect("no adapter found");

    let (device, queue) = adapter
        .request_device(
            &DeviceDescriptor {
                required_features: Features::empty(),
                required_limits: Limits::default(),
                label: None,
                memory_hints: MemoryHints::default(),
            },
            None,
        )
        .await
        .expect("device request failed");

    (device, queue)
}
```

### 4.3 Building for WASM

```bash
# Add the target
rustup target add wasm32-unknown-unknown

# Build with the WebGPU feature
cargo build --target wasm32-unknown-unknown --features webgpu

# Or with wasm-pack (which also runs wasm-bindgen and wasm-opt)
wasm-pack build --target web --features webgpu
```

The `webgpu` Cargo feature enables the `web-sys` WebGPU bindings. Without it, wgpu falls back to WebGL2 via the `webgl` feature (wider browser support but no compute shaders). [Source: wgpu documentation](https://docs.rs/wgpu/latest/wgpu/)

### 4.4 Surface and Window Integration

On native Linux, wgpu surfaces are created from a `RawWindowHandle` (provided by `winit`). On WASM, winit's `EventLoop` target creates a `SurfaceTarget` from an `HtmlCanvasElement`. The same application loop drives both paths:

```rust
use winit::{event_loop::EventLoop, window::WindowBuilder};

fn main() {
    let event_loop = EventLoop::new().unwrap();
    let window = WindowBuilder::new()
        .with_title("wgpu portable demo")
        .build(&event_loop)
        .unwrap();

    // On WASM, winit appends the canvas to document.body
    #[cfg(target_arch = "wasm32")]
    {
        use winit::platform::web::WindowExtWebSys;
        web_sys::window()
            .and_then(|w| w.document())
            .and_then(|d| d.body())
            .and_then(|b| {
                b.append_child(&web_sys::Element::from(window.canvas().unwrap()))
                    .ok()
            });
    }

    // event_loop.run(...) — same API on both platforms
}
```

---

## 5. wasm-bindgen and Browser Integration

### 5.1 The wasm-bindgen Bridge

[wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) is the Rust-to-JavaScript interoperability layer. The `#[wasm_bindgen]` attribute generates the JS glue code that lets JavaScript call Rust functions and Rust call JavaScript APIs:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

At `wasm-pack build` time, wasm-bindgen processes the compiled WASM file and emits:
- `pkg/my_lib_bg.wasm` — the optimised WASM binary
- `pkg/my_lib.js` — JS glue (imports, type coercions, memory management)
- `pkg/my_lib.d.ts` — TypeScript declarations

### 5.2 web-sys: DOM and WebGPU Bindings

`web-sys` provides Rust bindings to virtually every browser Web API, generated directly from the WebIDL specification. Features are gated by Cargo features to avoid bloating the binary:

```toml
# Cargo.toml
[dependencies]
web-sys = { version = "0.3", features = [
    "Window",
    "Document",
    "HtmlCanvasElement",
    "Navigator",
    "Gpu",                  # navigator.gpu
    "GpuAdapter",
    "GpuDevice",
    "GpuCanvasContext",
] }
```

Accessing the canvas and setting up a WebGPU surface from Rust:

```rust
use web_sys::{window, HtmlCanvasElement};
use wasm_bindgen::JsCast;

fn get_canvas() -> HtmlCanvasElement {
    window()
        .unwrap()
        .document()
        .unwrap()
        .get_element_by_id("gpu-canvas")
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()
        .unwrap()
}
```

### 5.3 Async/Await in WASM

Browser APIs are Promise-based; Rust's async/await integrates via `wasm-bindgen-futures`:

```rust
use wasm_bindgen_futures::spawn_local;

// Called from JS or #[wasm_bindgen(start)]
#[wasm_bindgen(start)]
pub fn start() {
    // Panic hook: redirects Rust panics to browser console.error
    console_error_panic_hook::set_once();

    spawn_local(async {
        run().await;
    });
}

async fn run() {
    let (device, queue) = init().await;
    // ... render loop via requestAnimationFrame
}
```

`spawn_local` schedules the future on the browser's microtask queue — this is the WASM equivalent of `tokio::spawn` for single-threaded browser contexts.

### 5.4 requestAnimationFrame from Rust

The standard browser render loop hooks into `requestAnimationFrame`. wasm-bindgen-futures and `Closure` make this idiomatic in Rust:

```rust
use wasm_bindgen::closure::Closure;
use web_sys::window;

fn request_animation_frame(f: &Closure<dyn FnMut()>) {
    window()
        .unwrap()
        .request_animation_frame(f.as_ref().unchecked_ref())
        .unwrap();
}
```

Alternatively, winit's WASM backend handles the `requestAnimationFrame` loop automatically when `event_loop.run()` is called in WASM context, making the application loop completely platform-agnostic. [Source: winit documentation](https://docs.rs/winit/latest/winit/)

### 5.5 Debugging WASM in the Browser

- **console_error_panic_hook** crate: `console_error_panic_hook::set_once()` redirects Rust `panic!` messages to `console.error`, making panics visible in DevTools
- **Chrome DevTools WASM debugging**: With DWARF debug info embedded (via `wasm-pack build --dev`) and the C/C++ DevTools Extension installed, DevTools can step through Rust/C++ source mapped to WASM instructions
- **Source maps**: `wasm-pack` generates `.wasm.map` files for source-level debugging
- `console_log` crate: provides a `log` facade backend that writes to `console.log`

---

## 6. WGSL: The Shader Language

WGSL (WebGPU Shading Language) is a strongly-typed shader language designed specifically for WebGPU. It is the sole shader language accepted by the browser WebGPU API — unlike native Vulkan (which accepts SPIR-V) or OpenGL (which accepts GLSL). [Source: W3C WGSL specification](https://www.w3.org/TR/WGSL/)

### 6.1 Basic Syntax

```wgsl
// Uniform buffer binding
struct Camera {
    view_proj: mat4x4f,
    eye_pos: vec3f,
}

@group(0) @binding(0) var<uniform> camera: Camera;

// Texture and sampler
@group(1) @binding(0) var t_diffuse: texture_2d<f32>;
@group(1) @binding(1) var s_diffuse: sampler;

// Vertex stage
struct VertexOutput {
    @builtin(position) clip_position: vec4f,
    @location(0) tex_coords: vec2f,
    @location(1) world_normal: vec3f,
}

@vertex
fn vs_main(
    @location(0) position: vec3f,
    @location(1) tex_coords: vec2f,
    @location(2) normal: vec3f,
) -> VertexOutput {
    var out: VertexOutput;
    out.clip_position = camera.view_proj * vec4f(position, 1.0);
    out.tex_coords = tex_coords;
    out.world_normal = normalize(normal);
    return out;
}

// Fragment stage
@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
    let color = textureSample(t_diffuse, s_diffuse, in.tex_coords);
    let light = max(dot(in.world_normal, vec3f(0.0, 1.0, 0.0)), 0.0);
    return vec4f(color.rgb * light, color.a);
}
```

### 6.2 Compute Shaders

```wgsl
// Parallel reduction: sum an array of f32
struct Data {
    values: array<f32>,
}

@group(0) @binding(0) var<storage, read>       input:  Data;
@group(0) @binding(1) var<storage, read_write> output: Data;

// Workgroup shared memory
var<workgroup> partial_sum: array<f32, 64>;

@compute @workgroup_size(64)
fn main(
    @builtin(global_invocation_id) global_id: vec3u,
    @builtin(local_invocation_id)  local_id:  vec3u,
) {
    let idx = global_id.x;
    partial_sum[local_id.x] = input.values[idx];
    workgroupBarrier();

    // Tree reduction within workgroup
    for (var stride = 32u; stride > 0u; stride >>= 1u) {
        if (local_id.x < stride) {
            partial_sum[local_id.x] += partial_sum[local_id.x + stride];
        }
        workgroupBarrier();
    }

    if (local_id.x == 0u) {
        output.values[global_id.x / 64u] = partial_sum[0];
    }
}
```

Key WGSL characteristics:
- **No implicit type coercions**: `vec4f(1)` must be `vec4f(1.0, 1.0, 1.0, 1.0)`
- **Address spaces**: `<uniform>`, `<storage>`, `<workgroup>`, `<private>`, `<function>`
- **`workgroupBarrier()`**: synchronises shared memory within a workgroup
- **Built-in functions**: `dot`, `cross`, `normalize`, `reflect`, `clamp`, `mix`, `textureSample`, `textureLoad`, `textureDimensions`

### 6.3 Tint: Dawn's WGSL Compiler

[Tint](https://dawn.googlesource.com/tint.git) is the WGSL compiler embedded in Dawn. It performs:

1. **Parsing and validation**: enforces WGSL grammar, type-checking, and WebGPU-specific constraints (no recursion, no unbounded loops without `@diagnostic` suppression)
2. **IR construction**: builds Tint's internal representation
3. **Backend code generation**:
   - → SPIR-V (for Vulkan on Linux, Android)
   - → MSL (for Metal on macOS, iOS)
   - → HLSL (for D3D12 on Windows)

This means the same WGSL source runs on all platforms without the developer managing multiple shader dialects. The trade-off is that WGSL is slightly more constrained than GLSL or HLSL: no global constructors, no unsized arrays in uniforms, explicit address spaces. [Source: DarthShader WGSL paper](https://arxiv.org/html/2409.01824v1)

wgpu's shader pipeline on native targets uses **Naga** (Rust-native WGSL → SPIR-V/MSL/GLSL compiler) rather than Tint. Both Naga and Tint accept WGSL; Naga is used when wgpu translates shaders natively, while Tint runs inside the browser when shaders are submitted to the browser's WebGPU API.

---

## 7. How Browser WebGPU Maps to Linux Vulkan

When Chrome or Firefox runs a WebGPU application on Linux, every GPU operation travels through several translation layers before reaching the physical GPU. Understanding this path explains both the performance characteristics and the API constraints.

### 7.1 Dawn's Vulkan Backend

Dawn is Google's C++ implementation of WebGPU, embedded in Chromium (and used as a native library by wgpu's native backend on some configurations). On Linux, Dawn's Vulkan backend is the primary path. The key source files live in `src/dawn/native/vulkan/`: [Source: Dawn repository](https://dawn.googlesource.com/dawn)

- `BackendVk.cpp` — initialises `VkInstance`, enumerates `VkPhysicalDevice`
- `PhysicalDeviceVk.cpp` — maps `WGPUAdapter` properties from `VkPhysicalDeviceProperties`
- `DeviceVk.cpp` — wraps `VkDevice` and `VkQueue`; manages `VkCommandPool` per thread
- `CommandBufferVk.cpp` — translates `WGPUCommandBuffer` → `VkCommandBuffer` recording
- `RenderPassCache.cpp` — caches `VkRenderPass` objects keyed on attachment formats and load/store ops
- `BufferVk.cpp` — manages `VkBuffer` backed by `ResourceMemoryAllocatorVk` (Dawn's allocator, analogous to VMA)
- `TextureVk.cpp` — manages `VkImage` with explicit layout transitions inserted by Dawn

### 7.2 The Translation Stack in Chrome on Linux

```
WASM JS API call (navigator.gpu.submitCommandBuffer)
       ↓
  V8 JavaScript engine (JIT)
       ↓
  Dawn WebGPU validation layer (catches API misuse)
       ↓
  Dawn Vulkan backend (CommandBufferVk.cpp)
       ↓
  VkSubmitInfo → VkQueue (Mesa RADV / ANV / NVK)
       ↓
  Physical GPU
```

The V8 JIT and Dawn's validation layer are the two principal sources of overhead relative to a direct native Vulkan application.

### 7.3 WebGPU vs Vulkan: Synchronisation Model

Vulkan's synchronisation model is explicit: pipeline barriers (`vkCmdPipelineBarrier`), semaphores, fences, and events must be inserted manually by the developer. Forgetting a barrier produces undefined behaviour (not an error in release mode). This explicitness enables very tight GPU scheduling but is notoriously difficult to use correctly.

WebGPU's model is implicit: the API tracks hazards automatically within a `GPUCommandEncoder`. If a storage buffer is written by a compute pass and read by a subsequent render pass within the same `GPUCommandBuffer`, the implementation inserts the necessary barrier automatically. Developers do not write pipeline barriers in WebGPU code. Dawn implements this by maintaining a resource usage tracking table during command encoding and emitting `vkCmdPipelineBarrier` calls at render/compute pass boundaries.

The cost: Dawn may insert more conservative barriers than a hand-tuned Vulkan application would, sacrificing some pipeline overlap. For most workloads this is a modest overhead, but it can matter in complex multi-pass pipelines.

### 7.4 WebGPU Limits vs Vulkan Device Limits

WebGPU defines a set of **default limits** that are intentionally lower than typical Vulkan device limits, to ensure portability across all WebGPU implementations (including mobile and lower-end hardware): [Source: MDN GPUSupportedLimits](https://developer.mozilla.org/en-US/docs/Web/API/GPUSupportedLimits)

| WebGPU limit                      | Default value | Notes                              |
|-----------------------------------|---------------|------------------------------------|
| `maxTextureDimension2D`           | 8192          | Vulkan min: 4096; typical GPU: 16384–32768 |
| `maxBufferSize`                   | 256 MB        | Vulkan has no `maxBufferSize` equivalent |
| `maxUniformBufferBindingSize`     | 65536         | 64 KB per binding                 |
| `maxStorageBufferBindingSize`     | 128 MB        | Device can raise with `requiredLimits` |
| `maxBindGroups`                   | 4             | Vulkan supports up to device max  |
| `maxComputeWorkgroupSizeX/Y/Z`    | 256           | Vulkan: device-dependent          |
| `maxComputeInvocationsPerWorkgroup` | 256         |                                    |

Applications can request higher limits at device creation:

```rust
// wgpu: request higher buffer size limit
let (device, queue) = adapter.request_device(
    &DeviceDescriptor {
        required_limits: Limits {
            max_buffer_size: 1 << 30, // 1 GB
            max_texture_dimension_2d: 16384,
            ..Limits::default()
        },
        ..Default::default()
    },
    None,
).await?;
```

The adapter will only grant limits up to the hardware maximum; the request does not error if the hardware supports it.

### 7.5 Performance Overhead: A Qualitative Assessment

Published comparative benchmarks between native Vulkan and browser WebGPU on the same hardware are limited and workload-dependent. The overhead comes from multiple sources:

- **V8 JIT bridge**: each call from WASM into the browser's WebGPU JS API crosses the WASM/JS boundary, which incurs a small but measurable cost. Modern V8 inlines many of these calls.
- **Dawn validation layer**: in debug builds significant; in release Chrome, validation is reduced but not eliminated.
- **Conservative barriers**: as described above, Dawn inserts barriers at pass boundaries unconditionally.
- **Swapchain present path**: the browser compositor (Viz/SurfaceFlinger) may add a frame of latency relative to a native Vulkan application that calls `vkQueuePresentKHR` directly.

*Note: Precise percentage overhead figures depend heavily on workload, hardware, and browser version and should be measured on the target hardware and browser. The overhead is generally small for throughput-bound compute workloads but more noticeable for latency-sensitive interactive applications.*

---

## 8. WASM SIMD for CPU-Side Computation

WebGPU handles GPU computation, but many pipelines require CPU-side pre/post-processing: mesh deformation, physics, collision detection, or ML data preprocessing. WASM SIMD provides vectorised CPU computation that runs alongside WebGPU without the overhead of GPU<→CPU round-trips.

### 8.1 SIMD Intrinsics in C++ (Emscripten)

Compile with `-msimd128` to enable SIMD128. Then include `<wasm_simd128.h>`:

```cpp
#include <wasm_simd128.h>
#include <cstddef>

// Vector multiply-accumulate: dst[i] = a[i] * b[i] + c[i]
void vec_mac(float* dst, const float* a, const float* b,
             const float* c, size_t n) {
    size_t i = 0;
    for (; i + 4 <= n; i += 4) {
        v128_t va = wasm_v128_load(a + i);
        v128_t vb = wasm_v128_load(b + i);
        v128_t vc = wasm_v128_load(c + i);
        v128_t vr = wasm_f32x4_add(wasm_f32x4_mul(va, vb), vc);
        wasm_v128_store(dst + i, vr);
    }
    // scalar tail
    for (; i < n; ++i) dst[i] = a[i] * b[i] + c[i];
}
```

Key `wasm_simd128.h` functions: `wasm_f32x4_splat`, `wasm_f32x4_mul`, `wasm_f32x4_add`, `wasm_f32x4_sqrt`, `wasm_i32x4_mul`, `wasm_v128_load`, `wasm_v128_store`. [Source: Emscripten SIMD PR](https://github.com/emscripten-core/emscripten/pull/8559)

### 8.2 SIMD Intrinsics in Rust

Rust's `std::arch::wasm32` module (re-exported from `core::arch::wasm32`) provides SIMD intrinsics behind the `simd128` target feature: [Source: Rust core::arch::wasm32](https://doc.rust-lang.org/beta/core/arch/wasm32/index.html)

```rust
#![cfg(target_arch = "wasm32")]
use std::arch::wasm32::*;

#[target_feature(enable = "simd128")]
unsafe fn dot_product_simd(a: &[f32], b: &[f32]) -> f32 {
    assert_eq!(a.len(), b.len());
    let mut acc = f32x4_splat(0.0);
    let n = a.len();
    let mut i = 0;
    while i + 4 <= n {
        let va = v128_load(a[i..].as_ptr() as *const v128);
        let vb = v128_load(b[i..].as_ptr() as *const v128);
        acc = f32x4_add(acc, f32x4_mul(va, vb));
        i += 4;
    }
    // Horizontal sum
    let sum = f32x4_extract_lane::<0>(acc)
            + f32x4_extract_lane::<1>(acc)
            + f32x4_extract_lane::<2>(acc)
            + f32x4_extract_lane::<3>(acc);
    // Scalar tail
    let tail: f32 = a[i..].iter().zip(b[i..].iter()).map(|(x, y)| x * y).sum();
    sum + tail
}
```

Enable SIMD at build time:

```bash
RUSTFLAGS='-C target-feature=+simd128' \
  cargo build --target wasm32-unknown-unknown --features webgpu
```

Auto-vectorisation: LLVM's vectoriser, when targeting WASM with `+simd128`, will often auto-vectorise loops without manual intrinsics — the same way it targets SSE/AVX on x86_64. Check the output WASM with `wasm-objdump -d out.wasm | grep f32x4` to verify vectorisation occurred.

### 8.3 Mixed CPU+GPU Workloads

A common pattern for GPU-accelerated ML preprocessing or physics:

```
WASM SIMD (CPU)                  WebGPU (GPU)
─────────────────                ─────────────────
Parse/decode input data          Vertex shader
Normalize/preprocess (SIMD)  →   wgpuQueueWriteBuffer →  GPU Buffer
Sort spatial index (SIMD)        Compute shader
Collision broadphase (SIMD)      Render pipeline
```

The bridge is `queue.write_buffer` (wgpu) or `wgpuQueueWriteBuffer` (C WebGPU API). This performs a CPU→GPU copy from WASM linear memory into a `VkBuffer` (via Vulkan staging buffers on the native path, or via the browser's internal GPU memory management on the browser path). The copy cost is the only "impedance mismatch" between the two computational domains.

---

## 9. Real-World Use Cases

### 9.1 Bevy on WASM with WebGPU

Bevy is a Rust game engine built directly on wgpu. Its WASM/WebGPU path is straightforward because the render graph abstraction is identical on all targets. [Source: Bevy WebGPU announcement](https://bevy.org/news/bevy-webgpu/)

```bash
# Build a Bevy app for the web with WebGPU backend
cargo build --target wasm32-unknown-unknown --features bevy/webgpu

# Or use Bevy's own build helper for examples
cargo run -p build-wasm-example -- --api webgpu my_example
```

Key Bevy-specific WASM considerations:
- **Asset loading**: Bevy's `AssetServer` uses `fetch` (via `web-sys`) rather than filesystem I/O on WASM; assets must be served by an HTTP server
- **winit canvas integration**: Bevy configures winit to attach the canvas to a specified DOM element via `App::add_plugins(DefaultPlugins.set(WindowPlugin { ... }))`
- **WebGL2 still default**: for custom Bevy apps targeting wide browser compatibility, `bevy/webgl2` is the safer choice until WebGPU ships in all major browsers; `bevy/webgpu` opts into the newer path

### 9.2 Godot 4 Web Export with WebGPU

Godot 4.2+ introduced a WebGPU render backend for web export. The export process generates a `.wasm` module plus JavaScript loader; the Godot editor provides an "Export → Web" template. [Source: Bevy WebGPU issue #8315 (context for comparison)](https://github.com/bevyengine/bevy/issues/8315)

The render backend uses Godot's `RenderingDevice` abstraction (analogous to wgpu's `Device`) which maps to WebGPU in the browser build and to Vulkan/Metal/D3D12 on desktop. WGSL shaders are authored in Godot's shader language and compiled to the appropriate target at export time.

### 9.3 GPU-Accelerated Tools in the Browser

**Shader editors**: Tools equivalent to Shadertoy can be built with wgpu+WASM: the user edits WGSL source in a `<textarea>`, the Rust code recompiles the `ShaderModule` via `device.create_shader_module()`, and the new pipeline is hot-swapped on the next frame. No server round-trip required.

**GPU image editors**: WebGPU compute shaders can implement convolution filters, tone mapping, and colour space conversion on full-resolution images in the browser. The image data loads from `<input type="file">` into WASM linear memory, is uploaded to a GPU buffer via `queue.write_buffer`, processed by a compute shader, and read back via `device.create_buffer` with `BufferUsages::MAP_READ | BufferUsages::COPY_DST`.

**Figma** is a notable production example: Figma's web renderer uses WebGPU (via Dawn in Chrome) for its 2D compositing pipeline, replacing an earlier WebGL path with significant rendering performance improvements. [Source: Figma WebGPU blog post](https://www.figma.com/blog/figma-rendering-powered-by-webgpu/)

### 9.3a GPU-Accelerated Shader Editors

A compelling use case that demonstrates the tight WASM/WebGPU loop is live WGSL editing. The user types WGSL source into a textarea element; a Rust/WASM event handler listens for `input` events, creates a new `ShaderModule`, rebuilds the `RenderPipeline` (which is fast — typically under a millisecond for a simple fragment shader), and the next `requestAnimationFrame` tick renders with the new shader. The entire pipeline — edit → compile → GPU upload → render — completes within a single frame budget on modern hardware, enabling interactive shader development without any server round-trip. This is architecturally simpler than the classic Shadertoy model (which used GLSL compiled on the server) because WebGPU shader compilation is synchronous from the application's perspective (the browser may defer optimisation but always provides a usable pipeline immediately via `createShaderModule`).

### 9.4 ML Inference in the Browser

**WebLLM**: The [mlc-ai/web-llm](https://github.com/mlc-ai/web-llm) project runs large language models in the browser via WebGPU. The workflow: model weights (quantised to 4-bit or 8-bit) are downloaded once and cached in the browser's cache storage; inference runs entirely on the user's GPU via WebGPU compute shaders. WebLLM uses MLC-LLM (Apache TVM's machine learning compiler) to generate optimised WebGPU kernels. [Source: WebLLM arXiv paper](https://arxiv.org/html/2412.15803v1)

**Transformers.js**: Hugging Face's [Transformers.js](https://huggingface.co/spaces/Xenova/webgpu-embedding-benchmark/) library runs Transformers models in JavaScript using ONNX Runtime Web. The WebGPU Execution Provider (`deviceType: "webgpu"`) delegates matrix multiplications to WebGPU compute shaders:

```javascript
import { pipeline } from '@xenova/transformers';
// WebGPU EP used automatically when available
const embedder = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2',
    { device: 'webgpu' });
const output = await embedder('Hello, world!', { pooling: 'mean', normalize: true });
```

**WebNN**: The Web Neural Network API (`navigator.ml`) is an emerging browser standard that provides a higher-level ML inference API than raw WebGPU compute. It is designed to delegate to hardware ML accelerators (NPUs, DSPs) when available, falling back to WebGPU compute. WebNN and WebGPU are complementary: WebNN handles model inference; WebGPU handles rendering and custom compute.

### 9.5 Stable Diffusion and Generative AI

Several projects demonstrate stable diffusion inference running entirely in the browser via WebGPU:

- **Transformers.js** (Hugging Face): runs quantised SDXL-Turbo and Stable Diffusion 2.1 via ONNX Runtime Web with WebGPU EP. Image generation at 512×512 takes approximately 5–10 seconds on a mid-range GPU (RTX 3060-class), reaching the boundary of interactive usability.
- **MLC Web LLM**: in addition to LLMs, MLC's compiler generates optimised WebGPU compute kernels for diffusion model UNet blocks. The key technique is tiling and vectorising matrix multiplications in WGSL compute shaders with explicit workgroup memory reuse — the same optimisation applied to GPU matrix multiplication on native Vulkan, but expressed in WGSL and constrained by WebGPU's smaller workgroup memory limits.

The practical ceiling on browser-based generative AI is the `maxBufferSize` and `maxStorageBufferBindingSize` WebGPU limits (256 MB and 128 MB defaults). Models larger than these limits cannot be held in a single WebGPU buffer and must be split across multiple bindings, adding management overhead. Applications can request raised limits at device creation, but the browser grants only what the hardware supports — on integrated graphics with shared system memory, the limit may be lower than on discrete GPUs.

---

## 10. Build Tooling and Distribution

### 10.1 wasm-pack

`wasm-pack` is the standard Rust→WebAssembly packaging tool. It wraps `cargo build`, `wasm-bindgen`, and `wasm-opt` into a single command: [Source: wasm-pack docs](https://rustwasm.github.io/docs/wasm-pack/commands/build.html)

```bash
# Development build (no optimisation, DWARF debug info embedded)
wasm-pack build --dev --target web

# Release build (optimised, stripped)
wasm-pack build --release --target web --features webgpu

# Outputs:
#   pkg/my_lib_bg.wasm       — the WASM binary
#   pkg/my_lib.js            — JS bindings
#   pkg/my_lib.d.ts          — TypeScript types
#   pkg/package.json         — npm package metadata
```

The `--target` flag controls the JS module format:
- `web` — ES module with `import` statements (for direct browser use or bundlers)
- `bundler` — CommonJS-compatible (for webpack/rollup)
- `nodejs` — Node.js module

### 10.2 wasm-opt: Binary Size Optimisation

`wasm-opt` (part of [Binaryen](https://github.com/WebAssembly/binaryen)) applies WASM-level optimisation passes: dead code elimination, inlining, constant folding, and register allocation. `wasm-pack` invokes it automatically in release mode.

Manual invocation:

```bash
wasm-opt -O3 --strip-debug -o out-opt.wasm out.wasm
# -Os: optimise for size
# -Oz: maximum size reduction (may increase compile time)
```

Typical size reductions: 15–20% in code size; 9% in compressed size, on top of LLVM's own optimisation. [Source: wasm-opt effectiveness](https://rustwasm.github.io/book/reference/code-size.html)

### 10.3 Compression and Serving

WASM compresses extremely well under gzip and Brotli — typical ratios of 2–3× — because the binary format contains repeated patterns (instruction opcodes, LEB128 sizes, type signatures). Serving pre-compressed `.wasm.gz` or `.wasm.br` files avoids on-the-fly compression overhead. Nginx and Cloudflare Pages serve pre-compressed assets automatically when the `.gz` or `.br` file exists alongside the original.

**Critical**: browsers require `Content-Type: application/wasm` for WASM files to be treated as WebAssembly (for streaming compilation via `WebAssembly.instantiateStreaming`). Without this header, the file is downloaded but not streamed-compiled.

```nginx
# nginx.conf
types {
    application/wasm wasm;
}
```

### 10.4 COOP/COEP Headers for Threads

SharedArrayBuffer (required for WASM threads) requires cross-origin isolation. Add these headers to every response in the deployment:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

On platforms without header control (GitHub Pages), the `coi-serviceworker` npm package installs a service worker that patches these headers client-side. [Source: Wasmer COOP/COEP docs](https://docs.wasmer.io/sdk/wasmer-js/how-to/coop-coep-headers/)

### 10.5 Local Development Server

WASM cannot load from `file://` URLs due to CORS restrictions on `fetch`. Use a local HTTP server during development:

```bash
# Python (simplest)
python3 -m http.server 8080 --bind 127.0.0.1

# With COOP/COEP headers (required for SharedArrayBuffer/threads)
# Use a server that supports custom headers, e.g. devserver (Rust)
cargo install devserver
devserver --address 127.0.0.1:8080 --header "Cross-Origin-Opener-Policy: same-origin" \
          --header "Cross-Origin-Embedder-Policy: require-corp"
```

### 10.6 Hosting and CDN

- **GitHub Pages**: free static hosting; no header control (use `coi-serviceworker` for threading)
- **Cloudflare Pages**: free tier with `_headers` file for custom response headers
- **Vercel**: `vercel.json` `headers` configuration
- **Netlify**: `netlify.toml` `[[headers]]` section

All modern CDNs will serve pre-compressed WASM if the compressed variants are uploaded alongside originals. Cloudflare automatically compresses assets at the edge.

### 10.6a Firefox WebGPU on Linux: WebGPU-RS

Firefox implements WebGPU using **wgpu** itself — the same Rust library described in Section 4. Firefox's GPU process links against a fork of wgpu (the `wgpu` crate maintained in `mozilla-central`) and maps the `GPUDevice` JavaScript object to a `wgpu::Device`. When Firefox runs on Linux, wgpu selects the Vulkan backend and calls into Mesa. This means that Firefox's WebGPU implementation, unlike Chrome's (which uses Dawn), shares code with the Rust WASM WebGPU story: a Rust wgpu application deployed to a browser gets the same wgpu logic in Firefox (via Firefox's bundled wgpu) and on native (via the wgpu crate linked directly). The practical implication is that subtle wgpu bugs fixed upstream flow into both Firefox's WebGPU implementation and native Rust applications simultaneously. Firefox's MSRV constraint for wgpu is tracked at [https://github.com/gfx-rs/wgpu](https://github.com/gfx-rs/wgpu).

### 10.7 Chrome DevTools WASM Debugging

Chrome DevTools (v88+) supports source-level WASM debugging when DWARF debug information is embedded. Install the **C/C++ DevTools Support (DWARF)** extension, then:

```bash
# Build with debug info
wasm-pack build --dev --target web
# The .wasm file contains DWARF; DevTools extension parses it
```

For Rust, `wasm-pack --dev` passes `-C debuginfo=2` to `rustc`, embedding Rust source locations in the WASM DWARF sections. Chrome's DevTools can then show Rust source in the debugger, set breakpoints, and inspect variables — treating the WASM binary like a native binary with debug symbols.

---

## 11. Integrations

**Chapter 35 (Dawn and WebGPU)**: Dawn is the WebGPU C++ implementation whose Vulkan backend on Linux translates every WebGPU API call into Vulkan. When a browser-deployed wgpu or emdawnwebgpu application runs on Linux, it is Dawn—via Chromium or as a standalone native library—that interacts with Mesa's Vulkan drivers. Chapter 35 covers Dawn's architecture, the adapter enumeration logic, and the pipeline to `VkDevice` in detail.

**Chapter 40 (Bevy and wgpu)**: Bevy's render graph abstracts over wgpu's `RenderPipeline`, `ComputePipeline`, and `CommandEncoder`. Chapter 40 covers Bevy's rendering architecture; the current chapter explains the WASM deployment mechanism that takes Bevy's wgpu calls into the browser WebGPU JS API.

**Chapter 41 (Godot 4)**: Godot's `RenderingDevice` abstraction maps to Vulkan on desktop and WebGPU in the web export template. The web export path described in this chapter is the deployment mechanism for Godot Web games.

**Chapter 61 (SPIR-V)**: WGSL is the source language; SPIR-V is the binary shader IR that Dawn/Tint generates for the Vulkan backend. Chapter 61 covers the SPIR-V format, its reflection capabilities, and how Mesa's NIR compiler consumes SPIR-V from Dawn when a browser WebGPU workload reaches the Linux GPU driver.

**Chapter 77 (Shader Toolchain)**: Tint (WGSL→SPIR-V/MSL/HLSL) and Naga (wgpu's WGSL→SPIR-V compiler) are part of the broader shader compilation pipeline. Chapter 77 covers the full spectrum from GLSL through SPIRV-Cross to Tint, placing WGSL in context alongside the other shader languages in the Linux graphics ecosystem.

**Chapter 87 (LLM Inference on Linux GPU)**: WebLLM and Transformers.js represent the browser-deployed face of the GPU-accelerated LLM inference pipeline. Chapter 87 covers the native Linux side — llama.cpp with Vulkan, vLLM, and GPU kernel optimisation for LLM inference — while this chapter describes the WebGPU path that brings the same models to browser users without a native install.

**Chapter 91 (MLIR and IREE)**: IREE (Intermediate Representation Execution Environment) compiles ML models not only to native Vulkan compute shaders but also to WebAssembly as a deployment target, using IREE's WASM backend. Chapter 91 covers IREE's MLIR-based compilation pipeline; the WASM output of that pipeline is deployed using the infrastructure described in this chapter.

---

*Sources consulted for this chapter:*

- [WebAssembly 3.0 specification — Binary/Modules](https://webassembly.github.io/spec/core/binary/modules.html)
- [emdawnwebgpu README — Dawn's WebGPU for Emscripten](https://dawn.googlesource.com/dawn/+/refs/heads/main/src/emdawnwebgpu/pkg/README.md)
- [Emscripten — Using WebGPU](https://emscripten.org/docs/porting/multimedia_and_graphics/WebGPU-support.html)
- [Emscripten PR #24220 — Deprecate -sUSE_WEBGPU in favour of emdawnwebgpu](https://github.com/emscripten-core/emscripten/pull/24220)
- [Chrome for Developers — Build an app with WebGPU](https://developer.chrome.com/docs/web-platform/webgpu/build-app)
- [kainino0x/webgpu-cross-platform-demo](https://github.com/kainino0x/webgpu-cross-platform-demo)
- [wgpu GitHub repository (v29.0.3)](https://github.com/gfx-rs/wgpu)
- [wasm-bindgen guide](https://rustwasm.github.io/docs/wasm-bindgen/)
- [web-sys Canvas example](https://rustwasm.github.io/docs/wasm-bindgen/examples/2d-canvas.html)
- [wasm-pack build documentation](https://rustwasm.github.io/docs/wasm-pack/commands/build.html)
- [W3C WGSL specification](https://www.w3.org/TR/WGSL/)
- [DarthShader: Fuzzing WebGPU Shader Translators & Compilers](https://arxiv.org/html/2409.01824v1)
- [Dawn repository — src/dawn/native/vulkan/](https://dawn.googlesource.com/dawn)
- [MDN — GPUSupportedLimits](https://developer.mozilla.org/en-US/docs/Web/API/GPUSupportedLimits)
- [web.dev — COOP and COEP](https://web.dev/articles/coop-coep)
- [web.dev — WebAssembly threads](https://web.dev/articles/webassembly-threads)
- [Rust core::arch::wasm32 SIMD intrinsics](https://doc.rust-lang.org/beta/core/arch/wasm32/index.html)
- [Binaryen / wasm-opt Optimizer Cookbook](https://github.com/WebAssembly/binaryen/wiki/Optimizer-Cookbook)
- [Rust and WebAssembly — code size reference](https://rustwasm.github.io/book/reference/code-size.html)
- [Bevy WebGPU announcement](https://bevy.org/news/bevy-webgpu/)
- [mlc-ai/web-llm — In-browser LLM inference](https://github.com/mlc-ai/web-llm)
- [WebLLM arXiv paper](https://arxiv.org/html/2412.15803v1)
- [Figma rendering with WebGPU](https://www.figma.com/blog/figma-rendering-powered-by-webgpu/)
- [Wasmer COOP/COEP header patching](https://docs.wasmer.io/sdk/wasmer-js/how-to/coop-coep-headers/)
