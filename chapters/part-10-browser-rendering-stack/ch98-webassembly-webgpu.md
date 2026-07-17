# Chapter 98: WebAssembly and WebGPU as a Deployment Target

**Target audiences**:
- Application developers building portable GPU software
- Rust/C++ graphics developers
- Browser engineers
- wgpu users who want to understand the full stack from native Linux Vulkan through to in-browser WebGPU execution

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
12. [Native WASM+WebGPU Embedding — A DOM-Free Micro-Runtime](#12-native-wasmwebgpu-embedding--a-dom-free-micro-runtime)
    - [12.1 The Concept: Beyond the Browser Sandbox](#121-the-concept-beyond-the-browser-sandbox)
    - [12.2 Active Projects and Proposals](#122-active-projects-and-proposals)
    - [12.3 Architecture: Wasmtime + wgpu as Host](#123-architecture-wasmtime--wgpu-as-host)
    - [12.4 GPU Handle Table: The Core Abstraction](#124-gpu-handle-table-the-core-abstraction)
    - [12.5 Wiring Wasmtime Imports to wgpu](#125-wiring-wasmtime-imports-to-wgpu)
    - [12.6 The WASM Plugin Side](#126-the-wasm-plugin-side)
    - [12.7 Safe Memory Sharing](#127-safe-memory-sharing)
    - [12.8 Input Event Mapping](#128-input-event-mapping)
    - [12.9 Framework Integration](#129-framework-integration)
    - [12.10 What Is Missing: The wasi-webgpu Gap](#1210-what-is-missing-the-wasi-webgpu-gap)

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

This design has a direct implication for GPU work: data in WASM linear memory must be explicitly copied to GPU buffers via the WebGPU API (`wgpuQueueWriteBuffer` or `queue.write_buffer`). Zero-copy sharing between WASM linear memory and the GPU is not possible in the current browser WebGPU spec — the `GPUBuffer.mapAsync` path requires a separate allocation outside WASM linear memory. In a native embedding context (§12), however, zero-copy is achievable on UMA hardware by backing WASM linear memory with `mmap`-allocated pages and importing them into Vulkan via `VK_EXT_external_memory_host`; see §12 Roadmap (Long-term) for the verified approach.

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

**`winit`** is the standard Rust crate for cross-platform window creation and event loop management. It handles the OS-specific plumbing of opening a window and receiving input events without doing any rendering itself. On Linux it creates a `wl_surface` via the Wayland backend (selected when `WAYLAND_DISPLAY` is set) or an `xcb`/`xlib` window on X11, choosing at runtime based on environment variables. On WASM it creates an `HtmlCanvasElement` and integrates with the browser's `requestAnimationFrame` loop. In all cases `winit` exposes a `RawWindowHandle` and `RawDisplayHandle` (via the `raw-window-handle` crate) — opaque OS handles that GPU APIs use to create a rendering surface. `winit` carries no renderer; it is paired with wgpu, `ash` (raw Vulkan), `glutin` (OpenGL), or Skia for drawing. Bevy, egui (`eframe`), Iced, and most Rust GPU projects use it as their windowing layer.

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

**WebIDL** (Web Interface Definition Language) is the IDL used by W3C and WHATWG to specify every browser JavaScript API. Each Web Platform API — `GPUDevice`, `HTMLCanvasElement`, `fetch()`, `WebSocket` — is defined in a `.webidl` file that describes its interfaces, attributes, methods, and type coercion rules. For example, the WebGPU spec defines:

```webidl
interface GPUDevice : EventTarget {
    readonly attribute GPUSupportedFeatures features;
    GPUBuffer createBuffer(GPUBufferDescriptor descriptor);
    GPURenderPipeline createRenderPipeline(GPURenderPipelineDescriptor descriptor);
    Promise<undefined> popErrorScope();
};
```

Browser engines (Chromium's Blink, Firefox's Gecko) run a WebIDL compiler over these files to auto-generate the C++ ↔ JavaScript binding glue. `web-sys` applies the same principle to Rust: it runs `wasm-bindgen`'s WebIDL processor over the same `.webidl` files to produce Rust types and `extern "C"` stubs. This means `web_sys::GpuDevice` is a direct mirror of the WebIDL `GPUDevice` — same methods, same attribute names, same async `Promise<T>` → `JsFuture` mapping. WebIDL is also the upstream source of truth: when the WebGPU spec adds a new method, `web-sys` gains it by regenerating from the updated `.webidl`. The `[Exposed=Window,Worker]` annotation in WebIDL maps to which Rust feature flag gates the binding. WebIDL's counterpart in the WASM Component Model ecosystem is **WIT** (WebAssembly Interface Types) — same concept (typed interface definition), different syntax and target runtime (see §12 Roadmap).

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

## 12. Native WASM+WebGPU Embedding — A DOM-Free Micro-Runtime

### 12.1 The Concept: Beyond the Browser Sandbox

The chapters above describe WebGPU and WASM as deployed *inside* a browser: a `<canvas>` element is the rendering surface, JavaScript is the glue, and the browser provides the WebGPU JS API. But the same two technologies — WASM as a portable, sandboxed binary format, WebGPU as an explicit GPU API — can be combined in a completely different deployment model: embedded inside a **native GUI application**, with no HTML, no CSS, no DOM, and no JavaScript runtime anywhere in the stack.

The use case is a secure, lightweight plugin or widget system:

- A native desktop application (Rust egui app, Qt QML app, Slint app) wants to execute untrusted third-party UI components or logic
- Those components need GPU access to render custom visualisations, shader-heavy widgets, or game-like content
- The host application wants to sandbox the plugin — it cannot access the filesystem, the network, or host memory beyond what is explicitly shared
- CEF (Chromium Embedded Framework) or Qt WebEngine would work but bring 100–300 MB of Chromium runtime, DOM parsing, CSS layout, V8 JIT, and the full Web Platform for a use case that needs none of those things

The alternative: embed a **lightweight WASM runtime** (Wasmtime, ~4 MB shared library) and expose GPU access through a custom set of WASM import functions backed by the host's native GPU device (wgpu on Rust hosts, Dawn on C++ hosts). The WASM module renders into a texture; the host composites that texture into its own UI. The result is a GPU-accelerated plugin sandbox with millisecond startup, ~4 MB runtime overhead, and a well-defined security boundary.

```
CEF/QtWebEngine approach:        WASM micro-runtime approach:
──────────────────────────       ──────────────────────────
HTML + CSS + JS runtime          No DOM, no CSS, no JS
Chromium GPU command buffer      Native wgpu / Dawn directly
100–300 MB runtime               ~4 MB Wasmtime
Full Web Platform attack surface Capability-based imports only
DOM = shared mutable state       Linear memory = isolated sandbox
```

This architecture is an emerging pattern, not yet a polished product. The components exist and compose correctly; what is missing is a standardised ABI (`wasi-webgpu`) and a packaged host library that GUI frameworks can integrate.

### 12.2 Active Projects and Proposals

Six projects are at varying stages of relevance to the native WASM+GPU embedding model. Each is described below with its current status, how to use it today, and the gaps that prevent it from being a complete solution.

---

#### `wasi-webgpu` (Bytecode Alliance / W3C WASI Community Group)

**Status**: Design phase (WebAssembly CG, mid-2026). No Wasmtime implementation exists.

The most directly relevant standards effort. `wasi-webgpu` is a WASI interface proposal to expose WebGPU surface and GPU device access to WASM modules running outside a browser. A companion proposal, `wasi-graphics-context`, defines surface and swapchain acquisition. Together they would give a WASM plugin typed access to a GPU device via WIT-generated bindings rather than hand-rolled `func_wrap` closures. [Source: github.com/WebAssembly/wasi-webgpu](https://github.com/WebAssembly/wasi-webgpu)

*How to use today*: There is no usable implementation yet. The WIT files are in flux; the closest you can do is read the current proposal shape and write a private ABI that mirrors it (which is what §12.3–12.9 does). The intended future usage once the proposal stabilises:

```wit
// wasi:webgpu/webgpu.wit (proposal shape as of mid-2026 — subject to change)
package wasi:webgpu;

interface webgpu {
  resource gpu-device {
    create-buffer:   func(size: u64, usage: u32) -> gpu-buffer;
    create-shader:   func(wgsl: string) -> gpu-shader-module;
    create-pipeline: func(shader: borrow<gpu-shader-module>) -> gpu-render-pipeline;
    queue-submit:    func(encoder: gpu-command-encoder);
  }
  resource gpu-buffer {
    write: func(offset: u64, data: list<u8>);
  }
  resource gpu-command-encoder {
    begin-render-pass: func(surface: borrow<gpu-surface>) -> gpu-render-pass-encoder;
  }
  resource gpu-render-pass-encoder {
    set-pipeline: func(pipeline: borrow<gpu-render-pipeline>);
    draw:         func(vertices: u32, instances: u32);
    end:          func() -> gpu-command-encoder;
  }
}
```

A guest module would then be built with `cargo component build --target wasm32-wasip2` and import `wasi:webgpu/webgpu` as a WIT dependency. The `wit-bindgen`-generated guest bindings replace all the raw `extern "C"` declarations in §12.6.

*Current gaps*:
- **No implementation**: No Wasmtime version implements even the device-creation path. No polyfill or shim exists.
- **WIT interface not frozen**: The WIT file has had breaking revisions; binding generated today will not match the final spec.
- **No surface model**: `wasi-graphics-context` (how a plugin acquires a Wayland/X11/Win32 window surface) is a separate, equally early proposal. Without it, `wasi-webgpu` alone cannot present pixels to the screen.
- **No security model**: GPU resource quotas, per-plugin memory limits, and preemption of GPU work from a misbehaving plugin are not yet addressed in the proposal.
- **No testing infrastructure**: No reference test suite, no CTS equivalent for the WASI GPU path.

---

#### Makepad (`makepad/makepad`)

**Status**: Production. Makepad Studio IDE and Robrix Matrix client ship as production deployments on native and WASM targets.

The closest *working* project to the DOM-free GPU rendering model. Makepad is a Rust UI framework that renders entirely to GPU — no DOM, no layout engine — and compiles to either a native binary (Vulkan/Metal/OpenGL) or a WASM module targeting a bare `<canvas>` via WebGL2/WebGPU. [Source: github.com/makepad/makepad](https://github.com/makepad/makepad)

*How to use*:

```toml
# Cargo.toml
[dependencies]
makepad-widgets = "0.6"

[[bin]]
name = "my_app"
```

```rust
use makepad_widgets::*;

live_design! {
    // Makepad "Live DSL" — a CSS-like shader/layout language embedded in Rust macros.
    // Changes are hot-reloaded at runtime without recompilation.
    import makepad_draw::shader::std::*;

    MyView = {{MyView}} {
        draw_bg: {
            // Inline WGSL-like shader — compiled by Makepad's own shader compiler
            fn pixel(self) -> vec4 {
                return mix(#f00, #00f, self.pos.x);
            }
        }
        height: 200, width: Fill
    }
}

#[derive(Live, LiveHook, Widget)]
pub struct MyView { #[walk] walk: Walk, #[layout] layout: Layout }

impl Widget for MyView {
    fn draw_walk(&mut self, cx: &mut Cx2d, _scope: &mut Scope, walk: Walk) -> DrawStep {
        self.draw_bg.draw_walk(cx, walk);
        DrawStep::done()
    }
    fn handle_event(&mut self, _cx: &mut Cx, _event: &Event, _scope: &mut Scope) {}
}

app_main!(App);

struct App { ui: WidgetRef }
impl AppMain for App {
    fn handle_event(&mut self, cx: &mut Cx, event: &Event) {
        self.ui.handle_widget_event(cx, event);
    }
}
```

Build and run:
```bash
# Native (Vulkan/Metal/OpenGL — picked at runtime)
cargo run

# WASM (targets browser <canvas> via WebGL2)
cargo install makepad-studio           # includes wasm build tooling
makepad build wasm --release           # outputs web/ directory
# Serve with any HTTP server; open in browser
```

*Current gaps*:
- **Inverted architecture**: Makepad apps *are* the WASM module. The §12 model requires WASM modules to be *plugins inside* a native host that owns the GPU. Makepad has no mechanism for a Wasmtime host to load a Makepad-built component as a sandboxed plugin.
- **No inter-plugin isolation**: All Makepad widgets share the same process, the same GPU device, and the same heap. There is no sandbox boundary between components.
- **WASM target is browser-only**: Makepad's WASM path outputs a module for `wasm32-unknown-unknown` targeting the browser's WebGL2/WebGPU API via `web-sys`. It cannot be loaded by Wasmtime because it depends on browser imports (`canvas.getContext`, `requestAnimationFrame`) that Wasmtime does not provide.
- **Custom shader language**: Makepad's Live DSL uses its own shader language (GLSL-like, not WGSL) compiled by its own backend. Plugins cannot bring arbitrary WGSL shaders without going through Makepad's shader compiler.
- **No headless / server-side path**: Makepad requires a GPU surface; it cannot render offscreen to a texture inside Wasmtime for server-side compositing.

---

#### `wasi-nn` (Wasmtime production, version 0.7)

**Status**: Production in Wasmtime 19+. Backends: OpenVINO, GGML (CPU), CoreML, experimental CUDA/Torch. [Source: wasmtime/crates/wasi-nn](https://github.com/bytecodealliance/wasmtime/tree/main/crates/wasi-nn)

Proves the hardware-access-via-WASI pattern at production quality. A WASM module calls imported functions (`nn_graph_init`, `nn_set_input`, `nn_compute`, `nn_get_output`); the host dispatches to a native ML framework without the module ever touching a native pointer. The GPU render equivalent — `gpu_create_pipeline`, `gpu_draw`, `gpu_submit` — is the same pattern applied to rendering.

*How to use*:

**Guest (WASM module)**:
```toml
# Cargo.toml for the WASM plugin
[dependencies]
wasi-nn = "0.7"
```
```rust
use wasi_nn::{ExecutionTarget, GraphEncoding, TensorType};

fn infer(model: &[u8], input: &[f32]) -> Vec<f32> {
    let graph = wasi_nn::load(
        &[model],
        GraphEncoding::Onnx,
        ExecutionTarget::GPU,      // host picks the actual GPU backend
    ).expect("load graph");
    let ctx = wasi_nn::init_execution_context(graph).unwrap();
    wasi_nn::set_input(ctx, 0, wasi_nn::Tensor {
        dimensions: &[1, 3, 224, 224],
        type_:      TensorType::F32,
        data:       bytemuck::cast_slice(input),
    }).unwrap();
    wasi_nn::compute(ctx).unwrap();
    let mut out = vec![0f32; 1000];
    wasi_nn::get_output(ctx, 0, bytemuck::cast_slice_mut(&mut out)).unwrap();
    out
}
```

**Host (Wasmtime)**:
```rust
use wasmtime_wasi_nn::{WasiNnCtx, backend::onnxruntime::OnnxruntimeBackend};

let mut wasi_nn_ctx = WasiNnCtx::new();
wasi_nn_ctx.add_backend(Box::new(OnnxruntimeBackend::default()));

// Wire into the Wasmtime linker (witx-based API, pre-Component-Model):
wasmtime_wasi_nn::witx::add_to_linker(&mut linker, |s: &mut MyState| {
    (&mut s.wasi_nn_ctx, &mut s.wasi_ctx)
})?;
```

*Current gaps*:
- **Inference only**: `wasi-nn` exposes tensor graph execution. There is no path to raw GPU compute shaders, vertex/fragment pipelines, or WGSL. A WASM module cannot render pixels via `wasi-nn`.
- **No render surface**: Tensor output is a flat `Vec<f32>` — there is no way to write the result directly to a GPU texture that a compositor could scanout.
- **No wgpu interop**: `wasi-nn` tensors live in wasi-nn's own memory model. They cannot be passed directly to a `wgpu::Buffer` without a CPU-side copy, even on UMA hardware, because the `wasi-nn` ABI has no concept of DMA-BUF or external memory handles.
- **witx-based ABI (legacy)**: `wasi-nn` still uses the witx interface model predating the Component Model. Migration to a WIT-based interface is planned but not yet shipped — the current API will break when the Component Model version lands.
- **Backend gaps on Linux**: OpenVINO backend works well on Intel hardware; GGML is CPU-only by default. ROCm and pure Vulkan compute backends are not yet available, leaving AMD and discrete NVIDIA GPUs underserved.

---

#### Dawn standalone (C++ WebGPU host)

**Status**: Buildable from source; used in production by Flutter Web, Chromium, and Skia Graphite. No prebuilt packages; build via CMake + depot_tools. [Source: dawn.googlesource.com/dawn](https://dawn.googlesource.com/dawn)

Dawn exposes the full `webgpu.h` C API as a standalone native library. A C++ (or Rust-via-FFI) host can create a `WGPUDevice` and simultaneously embed Wasmtime, wiring Dawn's GPU functions as Wasmtime imports. This gives the complete §12 architecture using a production-quality WebGPU implementation as the GPU backend rather than wgpu.

*How to use*:

```bash
# Fetch and build Dawn (requires depot_tools, Clang, CMake)
git clone https://dawn.googlesource.com/dawn && cd dawn
cp scripts/standalone.gclient .gclient
gclient sync
cmake -B build -DDAWN_BUILD_SAMPLES=OFF -DDAWN_ENABLE_VULKAN=ON
cmake --build build --target webgpu_dawn -j$(nproc)
# Produces: build/src/dawn/native/libwebgpu_dawn.so
```

```cpp
// Minimal Dawn + Wasmtime host (C++)
#include <webgpu/webgpu.h>
#include <dawn/native/DawnNative.h>
#include <wasmtime.h>   // C API

// 1. Create Dawn device (Vulkan backend on Linux)
WGPUInstanceDescriptor inst_desc{};
WGPUInstance instance = wgpuCreateInstance(&inst_desc);

WGPUAdapter adapter = nullptr;
wgpuInstanceRequestAdapter(instance, nullptr,
    [](WGPURequestAdapterStatus, WGPUAdapter a, const char*, void* u) {
        *reinterpret_cast<WGPUAdapter*>(u) = a;
    }, &adapter);

WGPUDevice device = nullptr;
WGPUDeviceDescriptor dev_desc{};
wgpuAdapterRequestDevice(adapter, &dev_desc,
    [](WGPURequestDeviceStatus, WGPUDevice d, const char*, void* u) {
        *reinterpret_cast<WGPUDevice*>(u) = d;
    }, &device);

// 2. Wire to Wasmtime: expose wgpuDeviceCreateBuffer as an import
wasmtime_func_callback_t gpu_create_buffer_cb =
    [](void* env, wasmtime_caller_t*, const wasmtime_val_t* args, size_t,
       wasmtime_val_t* results, size_t) -> wasm_trap_t* {
        WGPUDevice dev = reinterpret_cast<WGPUDevice>(env);
        uint64_t size = args[0].of.i64;
        uint32_t usage = args[1].of.i32;
        WGPUBufferDescriptor desc{ .size = size, .usage = usage };
        WGPUBuffer buf = wgpuDeviceCreateBuffer(dev, &desc);
        results[0].of.i32 = register_handle(buf);   // your handle table
        return nullptr;
    };
wasmtime_linker_define_func(linker, "env", "gpu_create_buffer",
    create_buf_type, gpu_create_buffer_cb, device, nullptr);
```

*Current gaps*:
- **No prebuilt packages**: Dawn must be built from source using Google's `depot_tools` / `gclient` workflow, which is unfamiliar to most Rust/native developers. No Debian, Fedora, or crates.io package exists.
- **C++ only at the API boundary**: Dawn's primary API is C++ (`webgpu_cpp.h`); the C API (`webgpu.h`) is auto-generated but less ergonomic. Bridging to Wasmtime (Rust) requires writing C FFI boilerplate or using the unstable `dawn-rs` crate.
- **No packaged Wasmtime integration**: The wiring between `WGPUDevice` and Wasmtime import functions is entirely hand-written — the same problem §12.10 identifies for wgpu.
- **Build fragility**: Dawn vendors specific versions of SPIRV-Tools, Abseil, and Tint via `depot_tools`. These conflict with system packages and Cargo workspace dependencies; embedding in a mixed C++/Rust project requires careful CMake + Cargo integration.
- **Validation layer mismatch**: Dawn's built-in validation is separate from the Vulkan validation layers. Shader errors in standalone Dawn produce different messages than in Chrome, complicating cross-environment debugging.

---

#### `bevy_mod_scripting` (WASM game scripts)

**Status**: Active pre-release (0.10.x). WASM target supported via the `wasm` feature flag; Lua and Rhai targets are more mature. [Source: github.com/makspll/bevy_mod_scripting](https://github.com/makspll/bevy_mod_scripting)

Bevy's community scripting plugin loads WASM modules as game scripts that can query and mutate ECS components. The structural pattern — WASM imports → host dispatches to Rust subsystems — matches the §12 GPU model exactly; GPU access is the missing resource class.

*How to use*:

```toml
# Cargo.toml (host app)
[dependencies]
bevy = "0.15"
bevy_mod_scripting = { version = "0.10", features = ["wasm"] }
```

```rust
use bevy::prelude::*;
use bevy_mod_scripting::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_plugins(ScriptingPlugin)
        .add_plugins(bevy_mod_scripting::wasm::WasmScriptingPlugin)
        .add_systems(Startup, load_script)
        .run();
}

fn load_script(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(ScriptCollection::<WasmScript> {
        scripts: vec![Script::new("my_plugin.wasm",
                                   asset_server.load("scripts/my_plugin.wasm"))],
    });
}
```

WASM plugin side (the script module — compiled with `--target wasm32-unknown-unknown`):
```rust
// No std, no wgpu — only the host ABI declared as extern
extern "C" {
    fn bevy_get_component_f32(entity: u64, component_type: u32, field: u32) -> f32;
    fn bevy_set_transform(entity: u64, x: f32, y: f32, z: f32);
    // No GPU functions available here yet
}

#[no_mangle]
pub extern "C" fn on_update(delta_time: f32) {
    let entity = 1u64;
    let speed = unsafe { bevy_get_component_f32(entity, 0, 0) };
    unsafe { bevy_set_transform(entity, speed * delta_time, 0.0, 0.0) };
}
```

*Current gaps*:
- **No GPU access**: Scripts can query and mutate ECS components but cannot issue draw calls, create render pipelines, or execute shaders. Adding GPU access requires the host to expose `wgpu::Device` functions as additional imports — which `bevy_mod_scripting` does not do today.
- **No render hooks**: Scripts cannot add draw calls to a render pass, modify the render graph, or inject a custom shader variant. The render world is entirely opaque to scripts.
- **Immature isolation model**: The WASM sandbox boundary exists at the Wasmtime level, but `bevy_mod_scripting` 0.10 does not enforce per-script resource quotas, GPU memory limits, or CPU time budgets. A misbehaving script can stall the frame loop.
- **Reflection-only ECS access**: Only `Reflect`-derived components are accessible from scripts. Components that do not derive `Reflect` (including most third-party crates and all render-world components) are invisible.
- **API instability**: The scripting API will change before Bevy 1.0; `bevy_mod_scripting` is not covered by Bevy's compatibility guarantees, and the WASM feature is the least-tested backend.

---

#### Extism (`extism/extism`) — Production WASM Plugin Framework

**Status**: Production (v1.0+). Used in Zellij terminal multiplexer, GoatCounter analytics, and several CLI tools. Multi-language host SDKs (Rust, Go, Python, JS, Ruby). [Source: github.com/extism/extism](https://github.com/extism/extism)

Extism is a mature production WASM plugin framework built on Wasmtime. It is not GPU-specific, but it demonstrates the fully productised host side of the §12 architecture — including plugin lifecycle management, structured input/output via shared memory, and a PDK (Plugin Development Kit) that hides the raw `extern "C"` ABI. Its model is the closest existing analogue to what a future `wasmtime-gpu` library should look like.

*How to use*:

```toml
# Host Cargo.toml
[dependencies]
extism = "1"
```

```rust
use extism::*;

let plugin = Plugin::new(
    Wasm::file("my_plugin.wasm"),
    [],      // no extra imports; Extism provides its own host functions
    false,   // not wasi-enabled
)?;

// Call an export by name; input/output is typed via serde
let result: String = plugin.call::<&str, String>("process", "hello world")?;
```

WASM plugin side (uses the Extism PDK — Rust):
```toml
[dependencies]
extism-pdk = "1"
```
```rust
use extism_pdk::*;

#[plugin_fn]
pub fn process(input: String) -> FnResult<String> {
    Ok(format!("processed: {input}"))
}
// Extism PDK handles the memory protocol (ptr/len pairs, shared memory region)
// so plugin authors never write extern "C" declarations manually
```

For GPU access, Extism's host function registration maps cleanly onto the §12 model:
```rust
// Register custom GPU host functions alongside Extism's built-ins
let gpu_create_buffer = Function::new(
    "gpu_create_buffer",
    [ValType::I64],   // size
    [ValType::I32],   // handle
    UserData::new(device.clone()),
    move |caller, inputs, outputs| {
        let size = inputs[0].unwrap_i64() as u64;
        let handle = state.create_buffer(size);
        outputs[0] = Val::I32(handle);
        Ok(())
    },
);

let plugin = Plugin::new(Wasm::file("gpu_plugin.wasm"),
                          [gpu_create_buffer, /* ... */], true)?;
```

*Current gaps*:
- **No GPU integration**: Extism provides no built-in GPU host functions. A host using Extism for GPU plugins must register all GPU functions manually (as shown above) — the same hand-wiring problem as the raw Wasmtime approach.
- **Memory protocol mismatch**: Extism's built-in memory protocol (input/output via a managed shared heap region) is designed for serialised data, not raw pixel buffers or WGSL strings. Large GPU data payloads (vertex buffers, textures) are awkward to pass through Extism's standard memory model.
- **No surface / swapchain concept**: Extism has no notion of a rendering surface, frame loop, or vsync — concepts fundamental to interactive GPU rendering.
- **No async dispatch**: Extism's call model is synchronous. GPU submission (`queue.submit`) and shader compilation (`create_render_pipeline`) are inherently asynchronous; mapping them to synchronous Extism calls requires polling on the host side.
- **Plugin isolation too coarse**: Extism loads one WASM module per `Plugin` instance. Multi-plugin GPU scenes (several plugins rendering into the same swapchain) have no coordination model in Extism.

### 12.3 Architecture: Wasmtime + wgpu as Host

The full data flow:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Native Host Application                           │
│                                                                      │
│  ┌──────────────┐   ┌────────────────────────────────────────────┐  │
│  │ winit /      │   │           Wasmtime Engine                   │  │
│  │ egui /       │   │  ┌──────────────────────────────────────┐  │  │
│  │ Qt / Slint   │   │  │         WASM Plugin Module            │  │  │
│  │              │   │  │  • #![no_std], no web-sys             │  │  │
│  │ keyboard /   │──▶│  │  • extern "C" GPU imports             │  │  │
│  │ mouse events │   │  │  • exports: init, render, on_input    │  │  │
│  │              │   │  └───────────────┬──────────────────────┘  │  │
│  └──────┬───────┘   │                  │ calls host ABI imports   │  │
│         │           │  ┌───────────────▼──────────────────────┐  │  │
│         │           │  │  Wasmtime Linker (host import fns)    │  │  │
│         │           │  │  gpu_create_buffer(size) → handle     │  │  │
│         │           │  │  gpu_write_buffer(h, ptr, len, off)   │  │  │
│         │           │  │  gpu_create_shader(wgsl_ptr, len) → h │  │  │
│         │           │  │  gpu_create_pipeline(shader) → h      │  │  │
│         │           │  │  gpu_begin_render_pass(out_tex) → h   │  │  │
│         │           │  │  gpu_set_pipeline(enc, pipe)          │  │  │
│         │           │  │  gpu_draw(enc, vtx, inst)             │  │  │
│         │           │  │  gpu_submit(enc)                      │  │  │
│         │           │  │  gpu_get_output_texture() → h         │  │  │
│         │           │  └───────────────┬──────────────────────┘  │  │
│         │           └──────────────────│──────────────────────────┘  │
│         │                              │                              │
│         │           ┌──────────────────▼──────────────────────────┐  │
│         │           │   wgpu Device + Queue + GpuHandleTable       │  │
│         │           │   output_texture: wgpu::Texture              │  │
│         │           └──────────────────┬──────────────────────────┘  │
│         │                              │                              │
│         │           ┌──────────────────▼──────────────────────────┐  │
│         └──────────▶│   GUI Framework Compositor                   │  │
│                     │   (egui / Iced / Slint / Qt)                 │  │
│                     │   displays WASM output_texture as a widget   │  │
│                     └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

The key design invariants:

1. **WASM module never holds native pointers.** It only holds `u32` handles that index into the host's `GpuHandleTable`. Passing a corrupted handle produces a controlled error, not a memory safety violation.
2. **All GPU resource lifetimes are host-managed.** The WASM module calls `gpu_drop_buffer(handle)` to signal intent; the host frees the real `wgpu::Buffer`. The host can also apply per-plugin resource quotas.
3. **Host reads WASM memory; WASM never reads host memory.** WASM linear memory is a `&[u8]` slice from the host's perspective. The WASM module cannot reach outside its sandbox by construction.
4. **Rendering flows into a host-owned texture.** The WASM module renders into a `wgpu::Texture` that the host pre-allocated. The host composites it into the GUI without the WASM module ever controlling the compositor.

### 12.4 GPU Handle Table: The Core Abstraction

The handle table maps `u32` integer handles to real wgpu objects. This is the same pattern as Unix file descriptors, OpenGL object names, and Windows `HANDLE` — an opaque integer that the guest uses and the host resolves.

```rust
// host/src/gpu_handle_table.rs
use std::collections::HashMap;
use std::sync::atomic::{AtomicU32, Ordering};

pub struct GpuHandleTable {
    pub buffers:        HashMap<u32, wgpu::Buffer>,
    pub textures:       HashMap<u32, wgpu::Texture>,
    pub texture_views:  HashMap<u32, wgpu::TextureView>,
    pub shaders:        HashMap<u32, wgpu::ShaderModule>,
    pub pipelines:      HashMap<u32, wgpu::RenderPipeline>,
    pub encoders:       HashMap<u32, wgpu::CommandEncoder>,
    pub command_bufs:   HashMap<u32, wgpu::CommandBuffer>,
    next_handle:        AtomicU32,
    // Handle 0 is reserved for the host-managed output texture
}

impl GpuHandleTable {
    pub fn new() -> Self {
        Self {
            buffers: HashMap::new(),
            textures: HashMap::new(),
            texture_views: HashMap::new(),
            shaders: HashMap::new(),
            pipelines: HashMap::new(),
            encoders: HashMap::new(),
            command_bufs: HashMap::new(),
            next_handle: AtomicU32::new(1), // 0 reserved
        }
    }

    pub fn alloc(&self) -> u32 {
        self.next_handle.fetch_add(1, Ordering::Relaxed)
    }
}

// Host state threaded through Wasmtime Store
pub struct HostState {
    pub device:          wgpu::Device,
    pub queue:           wgpu::Queue,
    pub output_texture:  wgpu::Texture,       // handle 0 for WASM
    pub output_view:     wgpu::TextureView,
    pub handles:         GpuHandleTable,
}
```

### 12.5 Wiring Wasmtime Imports to wgpu

Each host ABI function is a Wasmtime `Linker::func_wrap` closure that receives a `Caller<HostState>`, reads the handle table, and dispatches to real wgpu operations.

```rust
// host/src/linker.rs
use wasmtime::*;

pub fn add_gpu_imports(linker: &mut Linker<HostState>) -> anyhow::Result<()> {

    // gpu_get_output_texture() -> u32
    // Returns handle 0, the pre-allocated render target
    linker.func_wrap("gpu", "get_output_texture",
        |_: Caller<HostState>| -> u32 { 0 })?;

    // gpu_create_buffer(size: u32, usage: u32) -> u32
    linker.func_wrap("gpu", "create_buffer",
        |mut caller: Caller<HostState>, size: u32, usage: u32| -> u32 {
            let usage = wgpu::BufferUsages::from_bits_truncate(usage);
            let buf = caller.data().device.create_buffer(&wgpu::BufferDescriptor {
                label: None,
                size: size as u64,
                usage,
                mapped_at_creation: false,
            });
            let h = caller.data().handles.alloc();
            caller.data_mut().handles.buffers.insert(h, buf);
            h
        })?;

    // gpu_write_buffer(buf_h: u32, wasm_ptr: u32, len: u32, offset: u64)
    linker.func_wrap("gpu", "write_buffer",
        |mut caller: Caller<HostState>, buf_h: u32, wasm_ptr: u32, len: u32, offset: u64| {
            // Read from WASM linear memory — safe because WASM execution is
            // paused at this import boundary; no concurrent mutation possible.
            let mem = caller.get_export("memory")
                .and_then(Extern::into_memory)
                .expect("WASM module must export 'memory'");
            let data: Vec<u8> = mem.data(&caller)
                [wasm_ptr as usize..(wasm_ptr + len) as usize]
                .to_vec();          // copy out before mutably borrowing caller (see Roadmap §12 Long-term for zero-copy on UMA)
            let state = caller.data_mut();
            let buf = state.handles.buffers.get(&buf_h).expect("invalid buffer handle");
            state.queue.write_buffer(buf, offset, &data);
        })?;

    // gpu_create_shader(wgsl_ptr: u32, wgsl_len: u32) -> u32
    linker.func_wrap("gpu", "create_shader",
        |mut caller: Caller<HostState>, wgsl_ptr: u32, wgsl_len: u32| -> u32 {
            let mem = caller.get_export("memory")
                .and_then(Extern::into_memory).unwrap();
            let wgsl = std::str::from_utf8(
                &mem.data(&caller)[wgsl_ptr as usize..(wgsl_ptr + wgsl_len) as usize]
            ).expect("WGSL must be valid UTF-8").to_owned();
            let shader = caller.data().device.create_shader_module(
                wgpu::ShaderModuleDescriptor {
                    label: None,
                    source: wgpu::ShaderSource::Wgsl(wgsl.into()),
                });
            let h = caller.data().handles.alloc();
            caller.data_mut().handles.shaders.insert(h, shader);
            h
        })?;

    // gpu_begin_render_pass(out_tex_h: u32) -> u32   (returns encoder handle)
    linker.func_wrap("gpu", "begin_render_pass",
        |mut caller: Caller<HostState>, _out_tex_h: u32| -> u32 {
            // _out_tex_h == 0 → use host output_view; other values could
            // reference plugin-owned render targets.
            let enc = caller.data().device.create_command_encoder(
                &wgpu::CommandEncoderDescriptor::default());
            let h = caller.data().handles.alloc();
            caller.data_mut().handles.encoders.insert(h, enc);
            h
        })?;

    // gpu_set_pipeline(enc_h: u32, pipe_h: u32)
    linker.func_wrap("gpu", "set_pipeline",
        |_caller: Caller<HostState>, _enc_h: u32, _pipe_h: u32| {
            // In a real impl: look up encoder, begin_render_pass on it,
            // set_pipeline on the render pass. Split here for brevity.
        })?;

    // gpu_draw(enc_h: u32, vertices: u32, instances: u32)
    linker.func_wrap("gpu", "draw",
        |_caller: Caller<HostState>, _enc_h: u32, _vertices: u32, _instances: u32| {})?;

    // gpu_submit(enc_h: u32)
    linker.func_wrap("gpu", "submit",
        |mut caller: Caller<HostState>, enc_h: u32| {
            if let Some(enc) = caller.data_mut().handles.encoders.remove(&enc_h) {
                let cmd_buf = enc.finish();
                caller.data().queue.submit(std::iter::once(cmd_buf));
            }
        })?;

    // gpu_drop_buffer(h: u32)  — plugin signals it is done with a resource
    linker.func_wrap("gpu", "drop_buffer",
        |mut caller: Caller<HostState>, h: u32| {
            caller.data_mut().handles.buffers.remove(&h);
            // wgpu::Buffer drops here, freeing GPU memory
        })?;

    Ok(())
}
```

Loading and driving the WASM module:

```rust
// host/src/runtime.rs
pub struct WasmRuntime {
    store:    Store<HostState>,
    instance: Instance,
}

impl WasmRuntime {
    pub fn load(wasm_bytes: &[u8], state: HostState) -> anyhow::Result<Self> {
        let engine = Engine::default();
        let module = Module::new(&engine, wasm_bytes)?;

        let mut linker = Linker::<HostState>::new(&engine);
        add_gpu_imports(&mut linker)?;

        let mut store = Store::new(&engine, state);
        let instance = linker.instantiate(&mut store, &module)?;

        // Call the plugin's init export
        instance
            .get_typed_func::<(), ()>(&mut store, "wasm_init")?
            .call(&mut store, ())?;

        Ok(Self { store, instance })
    }

    pub fn render(&mut self, time_ms: f64) -> anyhow::Result<()> {
        self.instance
            .get_typed_func::<f64, ()>(&mut self.store, "wasm_render")?
            .call(&mut self.store, time_ms)
    }

    pub fn on_mouse_move(&mut self, x: f32, y: f32) -> anyhow::Result<()> {
        self.instance
            .get_typed_func::<(f32, f32), ()>(&mut self.store, "wasm_mouse_move")?
            .call(&mut self.store, (x, y))
    }

    pub fn on_key_down(&mut self, keycode: u32) -> anyhow::Result<()> {
        self.instance
            .get_typed_func::<u32, ()>(&mut self.store, "wasm_key_down")?
            .call(&mut self.store, keycode)
    }

    pub fn on_resize(&mut self, width: u32, height: u32) -> anyhow::Result<()> {
        self.instance
            .get_typed_func::<(u32, u32), ()>(&mut self.store, "wasm_resize")?
            .call(&mut self.store, (width, height))
    }
}
```

### 12.6 The WASM Plugin Side

The plugin is a `#![no_std]` Rust crate with no browser dependencies. It declares the host GPU ABI as `extern "C"` imports under the `"gpu"` module namespace:

```rust
// plugin/src/lib.rs
#![no_std]
extern crate alloc;

// GPU imports — implemented by the Wasmtime host
#[link(wasm_import_module = "gpu")]
extern "C" {
    fn get_output_texture() -> u32;
    fn create_buffer(size: u32, usage: u32) -> u32;
    fn write_buffer(buf_h: u32, ptr: *const u8, len: u32, offset: u64);
    fn create_shader(wgsl_ptr: *const u8, wgsl_len: u32) -> u32;
    fn create_render_pipeline(shader_h: u32) -> u32;
    fn begin_render_pass(out_tex_h: u32) -> u32;
    fn set_pipeline(enc_h: u32, pipe_h: u32);
    fn draw(enc_h: u32, vertices: u32, instances: u32);
    fn submit(enc_h: u32);
    fn drop_buffer(h: u32);
}

// WGSL shader source embedded at compile time
const TRIANGLE_WGSL: &str = include_str!("triangle.wgsl");

// Plugin-private state in WASM linear memory (not visible to host)
static mut PIPELINE_H: u32 = 0;
static mut VERTEX_BUF_H: u32 = 0;

const VERTICES: &[[f32; 2]] = &[[0.0, 0.5], [-0.5, -0.5], [0.5, -0.5]];

// Exported: called once by host after instantiation
#[no_mangle]
pub extern "C" fn wasm_init() {
    unsafe {
        let bytes = core::slice::from_raw_parts(
            VERTICES.as_ptr() as *const u8,
            VERTICES.len() * 8,
        );
        // Usage bits: VERTEX (0x0020) | COPY_DST (0x0008)
        VERTEX_BUF_H = create_buffer(bytes.len() as u32, 0x0028);
        write_buffer(VERTEX_BUF_H, bytes.as_ptr(), bytes.len() as u32, 0);

        let shader_h = create_shader(
            TRIANGLE_WGSL.as_ptr(),
            TRIANGLE_WGSL.len() as u32,
        );
        PIPELINE_H = create_render_pipeline(shader_h);
    }
}

// Exported: called every frame by host
#[no_mangle]
pub extern "C" fn wasm_render(time_ms: f64) {
    let _ = time_ms;
    unsafe {
        let out_tex = get_output_texture();      // handle 0
        let enc = begin_render_pass(out_tex);
        set_pipeline(enc, PIPELINE_H);
        draw(enc, 3, 1);
        submit(enc);
    }
}

#[no_mangle]
pub extern "C" fn wasm_mouse_move(_x: f32, _y: f32) {}

#[no_mangle]
pub extern "C" fn wasm_key_down(_keycode: u32) {}

#[no_mangle]
pub extern "C" fn wasm_resize(_w: u32, _h: u32) {}
```

Build the plugin:

```bash
# No std, no web APIs, no wasm-bindgen
cargo build --target wasm32-unknown-unknown --release
# Output: target/wasm32-unknown-unknown/release/plugin.wasm (~30 KB stripped)
```

### 12.7 Safe Memory Sharing

The critical correctness property is that reading WASM linear memory from a Wasmtime import function is **always safe**, even without explicit locks. Wasmtime's execution model guarantees:

1. **WASM execution is fully paused** at an import boundary. When `gpu_write_buffer` executes in the host, the WASM module's instruction pointer is suspended. No WASM thread can mutate WASM memory concurrently with a synchronous import call.

2. **The host reads a `&[u8]` slice**, not a raw pointer. `mem.data(&caller)` returns a Rust shared reference to the WASM linear memory region. Rust's borrow checker enforces that no mutation through `data_mut()` can occur while this reference is live.

3. **Data must be copied before re-borrowing `caller` mutably.** The `to_vec()` call in `gpu_write_buffer` copies the WASM data before `caller.data_mut()` is used to access the handle table. This is the only safe ordering in the proof-of-concept — the alternative (passing the WASM slice directly to `queue.write_buffer`) would require a `'static` lifetime that the temporary borrow cannot satisfy. A production implementation on UMA hardware can eliminate this software copy entirely by backing WASM linear memory with `mmap`-allocated pages imported into Vulkan via `VK_EXT_external_memory_host` (see §12 Roadmap, Long-term); on discrete PCIe GPUs the PCIe bus transfer still occurs regardless.

```rust
// CORRECT: copy out of WASM memory first, then mutably borrow state
let data: Vec<u8> = mem.data(&caller)[ptr..ptr+len].to_vec();  // copy
let state = caller.data_mut();                                   // now safe
state.queue.write_buffer(buf, offset, &data);

// INCORRECT: would not compile — conflicting borrows
// state.queue.write_buffer(buf, offset, &mem.data(&caller)[ptr..ptr+len]);
```

**Resource quota enforcement.** Because the host owns the handle table, it can enforce per-plugin resource limits:

```rust
fn create_buffer(mut caller: Caller<HostState>, size: u32, usage: u32) -> u32 {
    let state = caller.data();
    let current_usage: u64 = state.handles.buffers.values()
        .map(|b| b.size()).sum();
    if current_usage + size as u64 > MAX_PLUGIN_BUFFER_BYTES {
        return u32::MAX; // sentinel: allocation denied
    }
    // ... proceed with allocation
}
```

### 12.8 Input Event Mapping

Input flows from the native windowing system through the host into WASM exports. The translation layer maps platform key codes to a simple u32 encoding the WASM plugin declares as its own ABI:

```rust
// host/src/input.rs
use winit::keyboard::{KeyCode, PhysicalKey};

/// Maps winit physical keys to a portable u32 keycode.
/// ASCII values (32–126) for printable characters;
/// values ≥ 0x100 for non-printable (arrows, function keys, etc.)
pub fn map_keycode(key: PhysicalKey) -> u32 {
    match key {
        PhysicalKey::Code(KeyCode::Space)        => 32,
        PhysicalKey::Code(KeyCode::Enter)        => 13,
        PhysicalKey::Code(KeyCode::Escape)       => 27,
        PhysicalKey::Code(KeyCode::Backspace)    => 8,
        PhysicalKey::Code(KeyCode::Tab)          => 9,
        PhysicalKey::Code(KeyCode::ArrowLeft)    => 0x100,
        PhysicalKey::Code(KeyCode::ArrowRight)   => 0x101,
        PhysicalKey::Code(KeyCode::ArrowUp)      => 0x102,
        PhysicalKey::Code(KeyCode::ArrowDown)    => 0x103,
        PhysicalKey::Code(KeyCode::F1)           => 0x111,
        // Letters: map to ASCII uppercase (plugin handles case via modifier state)
        PhysicalKey::Code(k) => {
            let s = format!("{:?}", k);
            if s.starts_with("Key") && s.len() == 4 {
                s.chars().nth(3).unwrap_or('\0') as u32
            } else {
                0 // unknown key → 0, plugin ignores
            }
        }
        _ => 0,
    }
}
```

The host event loop translates winit events and calls WASM exports:

```rust
// Inside winit event_loop.run() / run_on_demand()
use winit::event::{ElementState, WindowEvent};

match event {
    WindowEvent::CursorMoved { position, .. } => {
        // Normalise to 0.0..1.0 viewport-relative coordinates
        let (w, h) = (window_size.width as f64, window_size.height as f64);
        runtime.on_mouse_move((position.x / w) as f32,
                              (position.y / h) as f32)?;
    }
    WindowEvent::MouseInput { state: ElementState::Pressed, button, .. } => {
        let btn_id: u32 = match button {
            MouseButton::Left  => 0,
            MouseButton::Right => 1,
            MouseButton::Middle => 2,
            _ => 0xFF,
        };
        runtime.on_key_down(0x200 | btn_id)?; // 0x200 prefix = mouse button
    }
    WindowEvent::KeyboardInput { event, .. } if event.state == ElementState::Pressed => {
        runtime.on_key_down(map_keycode(event.physical_key))?;
    }
    WindowEvent::Resized(new_size) => {
        // Recreate output texture at new size; notify plugin
        state.recreate_output_texture(new_size.width, new_size.height);
        runtime.on_resize(new_size.width, new_size.height)?;
    }
    WindowEvent::RedrawRequested => {
        let t = start_time.elapsed().as_secs_f64() * 1000.0;
        runtime.render(t)?;
        // Composite output_texture into GUI frame — see §12.9
    }
    _ => {}
}
```

Modifier state (Shift, Ctrl, Alt) can be passed as a bitfield via a separate import function `gpu_get_modifiers() -> u32` that the host populates from the winit `ModifiersState`, avoiding a separate API call per keypress.

### 12.9 Framework Integration

The integration point in each GUI framework is the same conceptually — display a GPU texture as a widget — but the mechanism differs.

| Framework | Texture import mechanism | WASM render call site | Notes |
|---|---|---|---|
| **egui / eframe** | `egui_wgpu::Renderer::register_native_texture(view, filter)` → `TextureId` | `App::update()` before `CentralPanel::show` | `RenderState` gives direct `wgpu::Device` access; cleanest integration |
| **Iced** | `iced_wgpu` custom `Primitive::Custom` with `wgpu::TextureView` | Inside `Application::view()` → custom `canvas::Program` | Requires `iced_wgpu` backend; `iced_tiny_skia` cannot accept GPU textures |
| **gpui** | `gpui::Image` from raw GPU texture via `gpui::RenderImage` | Custom `Element::paint()` | gpui on Linux uses Vulkan; share `VkDevice` with wgpu via `wgpu::Device::from_hal` |
| **xilem** | Custom `View` rendering Vello `Image` from wgpu texture | Inside `xilem::App` render pass | Vello renders via wgpu; texture import is `vello::peniko::Image` backed by wgpu |
| **Slint** | `slint::Image::from_rgba8_premultiplied(pixel_data)` (CPU readback) or `slint::platform::opengl_context` for GPU path | `Timer`-driven repaint | GPU path experimental; CPU readback loses zero-copy advantage |
| **Qt / QML** | `QSGTexture::fromNativeObject(vkImage, ...)` in `QQuickItem::updatePaintNode()` | `QQuickWindow::beforeRendering` signal | Requires `QSGRendererInterface::Vulkan`; share `VkDevice` with wgpu |

**egui integration (most complete example):**

```rust
// host/src/egui_app.rs
use eframe::{egui, egui_wgpu};

struct App {
    runtime:    WasmRuntime,
    texture_id: egui::TextureId,
}

impl eframe::App for App {
    fn update(&mut self, ctx: &egui::Context, frame: &mut eframe::Frame) {
        // 1. Drive the WASM render export
        let t = frame.info().cpu_usage.unwrap_or(0.0) as f64 * 1000.0;
        self.runtime.render(t).expect("WASM render failed");

        // 2. Forward egui input events to WASM plugin
        ctx.input(|i| {
            for event in &i.events {
                match event {
                    egui::Event::PointerMoved(pos) => {
                        self.runtime.on_mouse_move(pos.x, pos.y).ok();
                    }
                    egui::Event::Key { key, pressed: true, .. } => {
                        self.runtime.on_key_down(egui_key_to_u32(*key)).ok();
                    }
                    _ => {}
                }
            }
        });

        // 3. Display the WASM output texture as an egui image widget
        egui::CentralPanel::default().show(ctx, |ui| {
            let size = ui.available_size();
            ui.image(egui::load::SizedTexture::new(self.texture_id, size));
        });

        ctx.request_repaint(); // continuous animation
    }
}

fn setup(cc: &eframe::CreationContext) -> Box<dyn eframe::App> {
    let render_state = cc.wgpu_render_state.as_ref().unwrap();
    let device = &render_state.device;
    let queue  = &render_state.queue;

    // Pre-allocate the WASM render target texture
    let output_texture = device.create_texture(&wgpu::TextureDescriptor {
        label: Some("wasm_output"),
        size: wgpu::Extent3d { width: 800, height: 600, depth_or_array_layers: 1 },
        mip_level_count: 1,
        sample_count: 1,
        dimension: wgpu::TextureDimension::D2,
        format: wgpu::TextureFormat::Rgba8UnormSrgb,
        usage: wgpu::TextureUsages::RENDER_ATTACHMENT | wgpu::TextureUsages::TEXTURE_BINDING,
        view_formats: &[],
    });
    let output_view = output_texture.create_view(&Default::default());

    // Register with egui's wgpu renderer so it can be shown as a TextureId
    let texture_id = render_state.renderer.write().register_native_texture(
        device,
        &output_view,
        wgpu::FilterMode::Linear,
    );

    let state = HostState {
        device: device.clone(),
        queue: queue.clone(),
        output_texture,
        output_view,
        handles: GpuHandleTable::new(),
    };

    let wasm_bytes = include_bytes!("../../plugin/target/wasm32-unknown-unknown/release/plugin.wasm");
    let runtime = WasmRuntime::load(wasm_bytes, state).unwrap();

    Box::new(App { runtime, texture_id })
}
```

### 12.10 What Is Missing: The wasi-webgpu Gap

The proof-of-concept above works, but three things prevent it from being a production ecosystem:

**1. No standard ABI.** The GPU import function names and signatures (`gpu_create_buffer`, `gpu_write_buffer`, etc.) are invented here. A WASM plugin built for one host will not run on another host with a different ABI. The `wasi-webgpu` proposal would standardise this into a portable WASI interface — plugins would target `wasi:webgpu/device` and run on any compliant host (Wasmtime, WasmEdge, or even a future browser mode). Until that proposal stabilises, any deployment of this architecture is tied to a private host ABI.

**2. No packaged host library.** There is no `wasmtime-gpu` crate analogous to `wasmtime-wasi` that GUI developers can add to their `Cargo.toml`. The wiring between Wasmtime imports and wgpu must be hand-written. This is the gap that a focused library (a few thousand lines) could close.

**3. WASM modules cannot use wgpu directly.** The wgpu crate's WebGPU backend targets the browser's JavaScript WebGPU API (via `web-sys`). There is no `wgpu` backend for "running inside Wasmtime with host-provided GPU imports". Plugin developers cannot write `use wgpu::*; let device = instance.request_device(...)` and have it work in this runtime — they must use the raw `extern "C"` host ABI. A thin crate wrapping the host ABI with a wgpu-shaped API would fix this but does not yet exist.

**The expected unlock**: when `wasi-webgpu` defines a stable WASM interface, the Bytecode Alliance will implement it in Wasmtime as a first-class WASI implementation (parallel to `wasmtime-wasi`). Plugin developers will be able to use a safe Rust wrapper crate; GUI frameworks will gain a standardised `WasmGpuView` widget; and a WASM plugin compiled once will run in any compliant host — native GUI, server-side, or browser — without recompilation.

---

## Roadmap

### Near-term (6–12 months)

- **WebGPU subgroups and subgroup matrices** reach stable status in Chrome and are proposed for the W3C Candidate Recommendation. Subgroups expose warp-level SIMD operations (`subgroupBroadcast`, `subgroupAdd`) and fixed-size matrix-multiply units (tensor cores / matrix engines) to WGSL shaders, directly accelerating browser-side ML inference. [Source: Chrome Developers — What's next for WebGPU](https://developer.chrome.google.cn/blog/next-for-webgpu)
- **WASI 0.3 finalisation and Wasmtime 37+ preview support**: WASI 0.3 adds native async I/O and structured concurrency to the component model; follow-on 0.3.x releases will add cancellation tokens, stream optimisations, and a POSIX-threads-compatible threading model. While not directly used in browser GPU code, WASI 0.3 enables GPU-compute WASM modules to run in server-side runtimes (Wasmtime, WasmEdge) with async dispatch. [Source: The State of WebAssembly 2025–2026](https://platform.uno/blog/the-state-of-webassembly-2025-2026/)
- **`shader-f16` extension for native half-precision compute** is entering origin trial in Chrome; it allows WGSL shaders to use the `f16` type for bandwidth-efficient ML inference and image processing passes, matching what fp16 path already offers in Vulkan on Mesa. [Source: GPU for the Web Working Group Charter 2025](https://www.w3.org/2025/01/gpuweb-charter.html)
- **WebGPU Candidate Recommendation stabilisation**: W3C GPU for the Web Working Group published the WebGPU and WGSL specifications as Candidate Recommendation Drafts (June 2026). The Recommendation track process is expected to conclude with a formal W3C Recommendation within 12 months, giving browser vendors and wgpu a stable normative target. [Source: W3C WebGPU specification](https://www.w3.org/TR/webgpu/)
- **Firefox WebGPU enabled by default**: as of mid-2026 WebGPU is still disabled by default in Firefox; the expectation is enablement once the wgpu-in-Firefox path passes the full WebGPU CTS on Linux Vulkan. Note: needs verification against mozilla-central planning.
- **WASM Component Model toolchain (wit-bindgen, cargo-component, wasm-tools)**: WASI Preview 2, stable in Wasmtime 18+ (2024) and production-ready in mid-2026, adds the Component Model to production Wasmtime. The Component Model's WIT (WebAssembly Interface Types) IDL and `wit-bindgen` code-generator allow hosts to define typed, versioned interfaces in `.wit` files and generate Rust host+guest bindings automatically — replacing the hand-rolled `func_wrap` closures used in §12 with generated, type-safe stubs. `cargo-component` and `wasm-tools` complete the build pipeline. These tools are available now; the missing piece is a stabilised `wasi-webgpu.wit` interface file (see medium-term). When that lands, the entire §12 host-ABI (`gpu_create_buffer`, `gpu_write_buffer`, etc.) becomes auto-generated from the WIT definition. [Source: Bytecode Alliance Component Model documentation](https://component-model.bytecodealliance.org/)
- **WebGPU Compatibility Mode**: A Chrome 130+ extension that allows WebGL-style resource management patterns (implicit synchronisation, non-zero default framebuffer) within WebGPU shaders, lowering the migration barrier from WebGL to WebGPU. On Linux the compatibility layer routes through ANGLE (OpenGL ES → Vulkan) or directly to Mesa's Vulkan drivers depending on the backend selected by Dawn. The extension is entering W3C standardisation alongside the main WebGPU spec; once stable it removes the largest porting friction for the existing corpus of WebGL content. [Source: WebGPU Compatibility Mode explainer — gpuweb/gpuweb](https://github.com/gpuweb/gpuweb)
- **WASM threads and SharedArrayBuffer GPU dispatch**: WASM threads (WebAssembly 2.0, Shared Linear Memory + Atomics) are supported in all major browsers and in Wasmtime with `--wasm-features=threads`. Near-term work is completing ergonomics: `wasm-bindgen-rayon` for parallel data preparation, `parking_lot` compatibility under WASM threads, and stable patterns for submitting GPU `CommandEncoder`s from worker threads while the main thread owns the swapchain. On Linux this maps to Vulkan multi-queue submission; Mesa's RADV and ANV both support concurrent compute and graphics queues. This enables parallel shader compilation and multi-threaded mesh staging into GPU buffers without blocking the render loop. [Source: WebAssembly Threads Proposal](https://github.com/WebAssembly/threads)
- **`wasi-nn` migration to Component Model WIT**: The current `wasi-nn` 0.7 API uses the legacy witx interface model. A WIT-based `wasi:nn/inference` interface is in design; when it ships, the `wasmtime-wasi-nn` crate will replace `witx::add_to_linker` with a Component Model binding and guest modules will use `cargo component build`. This is the template for how `wasi-webgpu` will be deployed — `wasi-nn`'s migration is the rehearsal run.
- **`wasi-nn` Vulkan compute and ROCm backends**: A Vulkan compute backend for `wasi-nn` (dispatching ONNX graph operators as Vulkan compute shaders via `ort` + `wgpu`) is under community development. This would make the `ExecutionTarget::GPU` path work on AMD and NVIDIA Linux GPUs without requiring OpenVINO or CUDA, closing the biggest backend gap for Linux deployments of WASM ML workloads.
- **Extism v2 host-function API hardening**: Extism is moving its host-function registration API toward a more ergonomic builder pattern in v2, reducing the boilerplate of the `Function::new` + `Val` approach shown in §12.2. For GPU plugin hosts built on Extism, this reduces the code required to register each GPU import function.
- **Makepad WASM performance improvements (compute shaders)**: Makepad's WASM target currently uses WebGL2 as its GPU backend; migration to WebGPU (via `web-sys` WebGPU bindings) is in progress. This would unlock compute shaders in Makepad's Live DSL, enabling GPU-accelerated layout and animation effects in the browser path.

### Medium-term (1–3 years)

- **WESL (WGSL Extended Shading Language)**: a community-driven superset of WGSL adding modules, generics, and import semantics is under active design. If adopted into the W3C standard or as a pre-processing layer, WESL would substantially improve large shader codebase maintainability in wgpu and Emscripten projects. [Source: gpuweb/gpuweb GitHub](https://github.com/gpuweb/gpuweb)
- **WASI 1.0 and GPU-compute WASM runtimes**: WASI 1.0, targeting late 2026 or early 2027, is expected to stabilise the system interface for production server deployments. A WASI GPU extension (analogous to `wasi-nn`) could allow WASM modules to dispatch WebGPU-style compute workloads in server-side runtimes without a browser, unifying the browser and edge-compute deployment paths. [Source: WebAssembly in 2026 — DEV Community](https://dev.to/mysterious_xuanwu_5a00815/webassembly-in-2026-beyond-the-browser-and-into-the-cloud-2599)
- **Memory64 mainstream adoption**: WebAssembly 3.0 (shipped late 2025) includes Memory64, which lifts the 4 GB linear memory cap. Widespread toolchain support (wasm-bindgen, wasm-pack, Emscripten) for Memory64 is expected to land over 2026–2027, enabling GPU-accelerated WASM applications that stage large buffers (4K/8K texture atlases, large ML weight tensors) entirely in WASM linear memory before upload to the GPU. [Source: The State of WebAssembly 2025–2026](https://platform.uno/blog/the-state-of-webassembly-2025-2026/)
- **wgpu v1.x async pipeline compilation**: wgpu's roadmap includes fully async pipeline and shader compilation (`create_shader_module_async`, `create_render_pipeline_async`), matching the WebGPU spec's async path and eliminating main-thread stalls during shader compilation in WASM deployments. Note: track progress at [gfx-rs/wgpu](https://github.com/gfx-rs/wgpu).
- **Portable ray-tracing via WebGPU extension**: the GPU for the Web Working Group has discussed a ray-tracing extension (analogous to `VK_KHR_ray_tracing_pipeline`) for WebGPU; if accepted it would allow wgpu-based renderers to expose BVH traversal in the browser, with Mesa's RADV and ANV RT extensions as the Linux backend. Note: needs verification against current gpuweb/gpuweb issues.
- **`wasi-webgpu` + `wasi-graphics-context` as stabilised WIT interfaces**: When `wasi-webgpu` stabilises in the Component Model (projected 2027–2028), the §12 architecture changes fundamentally. Instead of hand-reading raw WASM linear memory (`mem.data(&caller)[ptr..len]`) and maintaining integer GPU handle tables, the host and plugin communicate through typed WIT interface boundaries:

  ```wit
  // wasi-webgpu.wit (illustrative, based on current proposal shape)
  interface gpu-device {
      resource buffer-handle;
      create-buffer: func(size: u64, usage: buffer-usages) -> buffer-handle;
      write-buffer: func(handle: borrow<buffer-handle>, offset: u64, data: list<u8>);
      submit: func(cmds: list<command-buffer>);
  }
  ```

  `wit-bindgen` generates the host-side dispatch and guest-side import stubs; no `unsafe` pointer reads cross the boundary. `wasi-graphics-context` provides the companion WIT interface for surface and swapchain acquisition (analogous to `eglCreateWindowSurface`), so a plugin can obtain a Wayland or X11 rendering surface without the host exposing a raw handle. A WASM plugin compiled once against these WIT interfaces runs in any compliant host — Wasmtime on Linux, WasmEdge, or a future browser mode — without recompilation, closing the portability gap §12.10 identifies. [Source: WebAssembly/wasi-webgpu](https://github.com/WebAssembly/wasi-webgpu) | [WebAssembly/wasi-graphics-context](https://github.com/WebAssembly/wasi-graphics-context)
- **Emscripten 4.x and emdawnwebgpu for C++ WebGPU**: The `emdawnwebgpu` project (part of the Dawn repository) provides an Emscripten-compatible C++ WebGPU header that targets the browser's native `navigator.gpu` when compiled for `wasm32-unknown-emscripten`. Emscripten 4.x is hardening the integration between `emdawnwebgpu`, Emscripten's threading model, and the browser's GPU command encoder, with the goal of allowing C++ graphics libraries (Dear ImGui, Filament, BGFX) to target WebGPU via Emscripten with the same source as their native Vulkan/Metal builds. On Linux, the Emscripten toolchain compiles C++ against Dawn's WebGPU headers to produce WASM that dispatches to Chrome's Dawn via the JavaScript WebGPU API — with Mesa's ANV or RADV as the final Vulkan backend. This closes the gap between C++ graphics codebases that currently require separate Vulkan and WebGL ports. Note: verify Emscripten 4.x release timeline against [emscripten.org](https://emscripten.org/). [Source: emdawnwebgpu — Dawn repository](https://dawn.googlesource.com/dawn)
- **Dawn standalone CMake packaging and Rust bindings**: Dawn's build team is working on a CMake install target and a `dawn-rs` crate that would allow Rust applications to depend on Dawn via Cargo without invoking `depot_tools`. This would eliminate the biggest barrier to building Dawn-backed Wasmtime hosts in Rust — the current requirement to hand-write C FFI for every `webgpu.h` function used.
- **`bevy_mod_scripting` GPU render hooks**: The `bevy_mod_scripting` roadmap includes exposing Bevy's render world to scripts via a safe ABI. The planned approach mirrors `wasi-nn`: scripts call `bevy_gpu_create_render_pipeline(shader_wgsl_ptr, len)` and `bevy_gpu_draw(pipeline, vertices)`, and the host dispatches to Bevy's `RenderApp`. This would make WASM scripts first-class participants in Bevy's render graph. Note: this is on the roadmap but not yet scheduled for a specific Bevy release.
- **`wasi-webgpu` first Wasmtime prototype**: The Bytecode Alliance has indicated intent to implement `wasi-webgpu` in Wasmtime once the WIT interface freezes. Based on the pattern of `wasi-nn`, the implementation path is: WIT interface freeze → `wit-bindgen` generates host and guest stubs → Wasmtime team writes the host-side dispatch to wgpu or Dawn → first PR lands in `wasmtime/crates/wasi-webgpu`. The current estimate (needs verification) is 2027–2028, contingent on `wasi-graphics-context` co-stabilising.
- **Extism GPU plugin pattern library**: As the GPU-via-WASM-plugin pattern matures, a community library (`extism-gpu` or similar) is likely to emerge that provides pre-registered GPU host functions (buffer management, pipeline creation, texture upload) on top of Extism's host-function API, eliminating the hand-wiring step shown in §12.2. No such library exists as of mid-2026.

### Long-term

- **WASM GC and high-level language GPU bindings**: WebAssembly Garbage Collection (in WebAssembly 3.0) enables runtimes for Java, Kotlin, Dart, and Python to target WASM without a GC-in-WASM stub. Long-term, these languages may gain idiomatic WebGPU bindings, broadening the set of languages that can write GPU code deployable to both native Linux and the browser from a single source. [Source: WebAssembly in 2026 — atakinteractive](https://www.atakinteractive.com/blog/webassembly-2026)
- **Unified WASM+native GPU abstraction layer**: wgpu already abstracts over Vulkan, Metal, D3D12, and WebGPU. A longer-term architectural goal (visible in wgpu RFCs) is to erase the distinction between `wasm32-unknown-unknown` and native targets at the API level, so that a single build can produce both a native binary and a WASM module with no `#[cfg(target_arch = "wasm32")]` guards. Note: needs verification against gfx-rs roadmap documents.
- **Server-side GPU WASM via WASI-GPU**: analogous to `wasi-nn` for inference, a `wasi-gpu` interface could expose a GPU abstraction to WASM modules running in Wasmtime or WasmEdge on Linux, dispatching to Vulkan via Mesa. This would allow the same WASM binary to run GPU compute in a browser tab and on a GPU-accelerated edge server without recompilation. Note: needs verification; no formal WASI proposal exists as of mid-2026.
- **Zero-copy between WASM linear memory and the GPU**: as noted in §2.2 and §12.7, all current paths from WASM linear memory to a GPU buffer require a CPU-side copy (`queue.write_buffer`, `to_vec()`). Two distinct technical routes could close this gap, and the native one has a working proof-of-concept.

  *Browser path.* The WebGPU spec would need a mechanism to create a `GPUBuffer` backed by a `SharedArrayBuffer` — the WASM linear memory segment — so that the GPU reads directly from the same physical pages the CPU writes. The prerequisite is that `SharedArrayBuffer`-backed memory be in a form the GPU driver can map (aligned, pinned, satisfying IOMMU constraints). This is architecturally feasible on unified-memory platforms (Apple Silicon, AMD APU, Intel iGPU) where CPU and GPU share the same DRAM; on discrete PCIe GPUs the copy across the PCIe bus remains unavoidable. No WebGPU spec proposal for this exists as of mid-2026.

  *Native embedding path (§12) — verified approach.* A working proof-of-concept ([lilting.ch, 2026](https://lilting.ch/en/articles/wasm-metal-zero-copy-gpu-inference-apple-silicon)) achieves zero-copy WASM↔GPU memory on Apple Silicon using Metal and Wasmtime, with ~0.03 MB RSS overhead versus ~16.78 MB for the copy path. The technique chains three layers:

  1. **Page-aligned allocation**: `mmap(MAP_ANON | MAP_PRIVATE)` provides a 16 KB-aligned region. `malloc` cannot be used — it does not guarantee the alignment that GPU APIs require.

  2. **GPU import without copy**: On Metal, `MTLDevice.makeBuffer(bytesNoCopy:length:options:deallocator:)` wraps the `mmap` pointer as a `MTLBuffer` with pointer identity confirmed (same physical pages, no copy). On Linux/Vulkan the equivalent is `VK_EXT_external_memory_host`: `vkImportHostPointerMemoryEXT()` imports a host `void *` as `VkDeviceMemory`, which is then bound to a `VkBuffer`. This extension is widely supported on AMD, Intel, and NVIDIA Vulkan drivers.

  3. **Wasmtime `MemoryCreator` + `LinearMemory`**: Wasmtime exposes two traits — `MemoryCreator` (a factory) and `LinearMemory` (the backing) — that replace its default `mmap` allocator. Implementing these and returning the pre-allocated pointer from `LinearMemory::as_ptr()` gives the WASM module the same physical pages as the GPU buffer:

  ```rust
  // host: implement LinearMemory backed by the mmap/VK_EXT_external_memory_host region
  struct ExternalLinearMemory { ptr: *mut u8, byte_size: usize, byte_capacity: usize }

  unsafe impl LinearMemory for ExternalLinearMemory {
      fn byte_size(&self)     -> usize { self.byte_size }
      fn byte_capacity(&self) -> usize { self.byte_capacity }
      fn grow_to(&mut self, new_size: usize) -> anyhow::Result<()> {
          if new_size > self.byte_capacity { anyhow::bail!("fixed-size external memory") }
          self.byte_size = new_size;
          Ok(())
      }
      fn as_ptr(&self) -> *mut u8 { self.ptr }
  }
  ```

  **Mandatory constraints**:
  - Guard pages are required: Wasmtime's JIT omits bounds checks when guard pages are present; the tail of the allocation must be `mprotect`'d to `PROT_NONE`.
  - `MemoryCreator` parameters use WASM page units (64 KB), not OS page size (4 KB or 16 KB on ARM).
  - The host must unmap the GPU buffer (Vulkan `vkUnmapMemory` / Metal `endAccess`) before submitting GPU commands that read it, then re-acquire access after. This is coordinated around the WASM `wasm_render()` export boundary.

  **Note**: wgpu's `Buffer::map_async` + `get_mapped_range_mut` is **not** a viable alternative for this use case. `map_async` is asynchronous (requires `device.poll()` before the mapping is available), `BufferViewMut` does not implement `DerefMut` to a raw slice (it exposes only `WriteOnly<'a, [u8]>`), and wgpu's own docs state "while a buffer is mapped, you may not submit any commands to the GPU that access it" — mutually exclusive with the read-while-WASM-writes model. The correct Vulkan path is `VK_EXT_external_memory_host` + raw `ash` bindings, bypassing wgpu's buffer abstraction for this allocation only.

  **Hardware limitation**: the zero-copy property only holds on unified-memory architectures (Apple Silicon, AMD APU/iGPU, Intel Iris Xe) where CPU and GPU share physical DRAM. On discrete PCIe GPUs (NVIDIA Ada, AMD RDNA discrete) the pages imported via `VK_EXT_external_memory_host` still reside in system RAM and must traverse the PCIe bus for the GPU to read them; the technique eliminates the software copy overhead but not the PCIe transfer.

  [Source: Zero-copy GPU inference on Apple Silicon with WebAssembly and Metal](https://lilting.ch/en/articles/wasm-metal-zero-copy-gpu-inference-apple-silicon) | [Wasmtime `LinearMemory` trait](https://docs.wasmtime.dev/api/wasmtime/trait.LinearMemory.html) | [wgpu `Buffer::unmap` constraints](https://docs.rs/wgpu/latest/wgpu/struct.Buffer.html)

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

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
