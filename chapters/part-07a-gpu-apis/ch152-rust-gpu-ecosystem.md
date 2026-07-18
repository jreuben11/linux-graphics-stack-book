# Chapter 152: Rust GPU Ecosystem: wgpu, ash, and naga

**Target audiences**: Rust developers building GPU-accelerated applications; developers migrating C/C++ GPU code to Rust; and engineers interested in how type-safe, memory-safe abstractions over Vulkan and WebGPU work in practice.

---

## Table of Contents

1. [Introduction](#introduction)
   - [Stack Layer Taxonomy: What Each Library Actually Does](#stack-layer-taxonomy-what-each-library-actually-does)
   - [The Two Full-Stack Paths](#the-two-full-stack-paths)
   - [Detailed Path Comparison](#detailed-path-comparison)
2. [ash: Raw Vulkan Bindings in Rust](#ash-raw-vulkan-bindings-in-rust)
3. [wgpu: The Cross-Platform GPU Abstraction](#wgpu-the-cross-platform-gpu-abstraction)
4. [wgpu-hal: The Backend Abstraction Layer](#wgpu-hal-the-backend-abstraction-layer)
5. [naga: The Shader Compiler](#naga-the-shader-compiler)
6. [WGSL: WebGPU Shading Language](#wgsl-webgpu-shading-language)
7. [GPU Compute with wgpu](#gpu-compute-with-wgpu)
8. [Rendering Pipelines with wgpu](#rendering-pipelines-with-wgpu)
9. [Ecosystem: erupt, vulkano, gpu-allocator](#ecosystem-erupt-vulkano-gpu-allocator)
10. [cuTile-rs: NVIDIA Tile Abstraction in Rust](#cutile-rs-nvidia-tile-abstraction-in-rust)
11. [cudarc: Rust CUDA Driver API](#cudarc-rust-cuda-driver-api)
12. [Vulkano: Compile-Time Safe Vulkan](#vulkano-compile-time-safe-vulkan)
13. [rust-gpu: Rust as a Shading Language](#rust-gpu-rust-as-a-shading-language)
14. [blade: Minimal GPU Library](#blade-minimal-gpu-library)
15. [vello: Compute-Based 2D Vector Rendering](#vello-compute-based-2d-vector-rendering)
16. [burn: Rust ML Framework](#burn-rust-ml-framework)
17. [encase and bytemuck: GPU Buffer Layout](#encase-and-bytemuck-gpu-buffer-layout)
18. [naga_oil: Shader Module Composition](#naga_oil-shader-module-composition)
19. [winit and raw-window-handle: Window and GPU Surface](#winit-and-raw-window-handle-window-and-gpu-surface)
20. [glam: Mathematics for Rust GPU](#glam-mathematics-for-rust-gpu)
21. [egui: Immediate-Mode GUI on wgpu](#egui-immediate-mode-gui-on-wgpu)
22. [candle: Rust ML Inference](#candle-rust-ml-inference)
23. [wgpu-profiler: GPU Timestamp Profiling](#wgpu-profiler-gpu-timestamp-profiling)
24. [Shader Hot-Reloading](#shader-hot-reloading)
25. [pixels: Simple wgpu Framebuffer](#pixels-simple-wgpu-framebuffer)
26. [rspirv: SPIR-V IR in Rust](#rspirv-spir-v-ir-in-rust)
27. [Rust Ownership vs the GPU Memory Model](#rust-ownership-vs-the-gpu-memory-model)
    - [Rust's Official Strategic Direction for GPU](#rusts-official-strategic-direction-for-gpu)
28. [Contributing to wgpu-core, wgpu-hal, and naga](#contributing-to-wgpu-core-wgpu-hal-and-naga)
29. [Roadmap](#roadmap)
30. [Integrations](#integrations)

---

## Introduction

The Rust GPU ecosystem has matured rapidly. The two main tracks are:

1. **Raw Vulkan** via `ash` or `erupt` — full control, unsafe, idiomatic Rust bindings to the Vulkan C API
2. **WebGPU-safe abstraction** via `wgpu` — safe Rust API, runs on Vulkan/Metal/DX12/WebGPU, with `naga` as the shader compiler

On Linux, `wgpu` targets Vulkan (via `wgpu-hal/vulkan`) and OpenGL (via `wgpu-hal/gles`). `ash` is the canonical raw Vulkan crate. Both are integral to the Rust game/rendering ecosystem and are used by projects from Bevy (game engine) to Firefox (WebGPU) to llama.cpp's Vulkan backend.

Sources: [wgpu](https://github.com/gfx-rs/wgpu) | [ash](https://github.com/ash-rs/ash) | [naga](https://github.com/gfx-rs/wgpu/tree/trunk/naga)

### Stack Layer Taxonomy: What Each Library Actually Does

A recurring point of confusion is treating libraries as alternatives when they operate at different layers. The chapter covers libraries at three distinct positions in the GPU stack — understanding which layer a library occupies is the prerequisite for any meaningful comparison.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     APPLICATION / ALGORITHM LAYER                    │
│   burn (ML)  ·  vello (2D)  ·  bevy (engine)  ·  candle (inference) │
└────────────────────────┬────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────────┐
│                  HOST MANAGEMENT LAYER  (CPU-side)                   │
│  wgpu ─── manages Vulkan/Metal/DX12 device, queue, buffers           │
│  ash  ─── raw Vulkan C API bindings                                  │
│  cudarc ── CUDA Driver API: contexts, streams, CudaSlice<T>          │
│  cuTile-rs ─ CUDA Tile IR host runtime + kernel launch               │
└────────────────────────┬────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────────┐
│                  SHADER / KERNEL AUTHORING LAYER (GPU-side code)     │
│  rust-gpu  ── Rust → SPIR-V (via rustc_codegen_spirv)               │
│  WGSL      ── WebGPU Shading Language (text, compiled by naga)       │
│  GLSL/HLSL ── traditional shader languages (text, compiled by glslc) │
│  PTX/CUDA C ─ NVIDIA kernel language (compiled by nvcc / NVRTC)      │
└────────────────────────┬────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────────┐
│                     RUNTIME / DRIVER LAYER                           │
│  Vulkan (RADV, ANV, NVK) ·  CUDA Driver  ·  Metal  ·  DX12          │
└─────────────────────────────────────────────────────────────────────┘
```

**cudarc vs rust-gpu** is therefore not an apples-to-apples comparison:

- `cudarc` is a **host management** library. It does for CUDA what `wgpu` does for Vulkan: manages device contexts, allocates GPU memory (`CudaSlice<T>`), and launches kernels. The kernels themselves are PTX or CUDA cubin compiled by nvcc or NVRTC — cudarc does not author GPU code.
- `rust-gpu` is a **shader authoring** compiler. It takes Rust source annotated with `#[spirv(compute)]` and compiles it to SPIR-V via a custom `rustc` backend. The resulting SPIR-V is consumed by `wgpu` or `ash` — rust-gpu does not manage GPU memory or device state.

The closer comparisons are:

| Comparison | Left | Right | Axis |
|-----------|------|-------|------|
| Host management | `cudarc` | `wgpu` / `ash` | CUDA vs Vulkan |
| Shader authoring | `rust-gpu` | WGSL / GLSL | Rust vs DSL |
| Full compute stack | `cudarc` + nvcc/NVRTC | `wgpu` + `rust-gpu` | CUDA path vs Vulkan path |
| NVIDIA ML compute | `cudarc` + cuTile-rs | `wgpu` + `burn-wgpu` | CUDA tensor cores vs Vulkan compute |

### The Two Full-Stack Paths

For GPU compute in Rust, the ecosystem splits into two end-to-end paths:

**CUDA path** — NVIDIA-exclusive, highest ML/HPC throughput:
```
Rust host code
  → cudarc (CudaContext, CudaSlice<T>, launch_builder)
  → PTX kernel (from nvcc / NVRTC / cuTile-rs)
  → CUDA Driver API → NVIDIA hardware only
```
Use when: NVIDIA hardware is guaranteed, cuBLAS/cuDNN/NCCL access is required, maximising throughput for ML training/inference.

**Vulkan path** — cross-vendor, runs on AMD (RADV), Intel (ANV), NVIDIA (NVK or proprietary), Apple (Metal via wgpu):
```
Rust host code
  → wgpu (Device, Queue, Buffer, ComputePipeline)
  → SPIR-V shader (from WGSL text or rust-gpu Rust source via naga)
  → Vulkan Driver → any GPU vendor
```
Use when: portability across GPU vendors is required, Vulkan graphics + compute are combined, or open-source driver stack (RADV/ANV) matters.

`rust-gpu` belongs exclusively to the Vulkan path. `cudarc` belongs exclusively to the CUDA path. They are not alternatives; an application that needs both NVIDIA CUDA (for cuDNN) and Vulkan compute (for cross-vendor compatibility) would use both independently.

### Detailed Path Comparison

The table below covers every axis that matters when choosing between the two paths for a new Rust GPU project:

| Property | CUDA path (cudarc + nvcc) | Vulkan path (wgpu + rust-gpu / WGSL) |
|----------|--------------------------|--------------------------------------|
| **GPU vendors** | NVIDIA only | AMD, Intel, NVIDIA, Apple Silicon |
| **Open-source drivers** | No (proprietary CUDA driver) | Yes (RADV, ANV, NVK, Asahi) |
| **Host API crate** | `cudarc` | `wgpu` (safe) or `ash` (unsafe) |
| **Kernel/shader language** | PTX / CUDA C++ (text/binary) | WGSL (text) or Rust via `rust-gpu` |
| **Kernel compiled by** | nvcc, NVRTC, cuTile-rs | naga (WGSL→SPIR-V), rustc_codegen_spirv (rust-gpu) |
| **GPU code is Rust** | No — kernel is CUDA C / PTX | Optional — rust-gpu compiles Rust to SPIR-V |
| **Borrow checker on GPU code** | No | Partial — rust-gpu runs it; WGSL does not |
| **cuBLAS / cuDNN / NCCL** | Yes (via cudarc feature flags) | No — no Vulkan equivalent for cuDNN/NCCL |
| **Tensor Core access** | Native (cuTile-rs, cuBLAS) | `VK_KHR_cooperative_matrix` (limited vendor support) |
| **ML throughput (NVIDIA A100)** | Baseline (cuBLAS reference) | ~60–80% of cuBLAS for GEMM via llama.cpp / burn-wgpu |
| **WebAssembly target** | No | Yes — wgpu targets WebGPU in WASM; WGSL shaders unchanged |
| **Graphics + compute** | Separate (CUDA interop required) | Unified — same wgpu device for rendering and compute |
| **Windows / macOS** | Windows only; macOS dropped CUDA in 10.14 | Yes — Metal backend via wgpu |
| **Compile-time safety** | Host only (cudarc types); kernel is runtime | Host: wgpu safe API; shader: WGSL type-checked by naga |
| **Ecosystem maturity** | Very high (CUDA since 2007) | Growing (wgpu stable, rust-gpu experimental) |
| **When to choose** | NVIDIA-only, ML training, cuDNN required | Portability, graphics, open-source stack, WASM |

**Combining both paths** is legitimate but uncommon. A typical scenario: a Linux desktop application uses `wgpu` + WGSL for real-time Vulkan rendering (cross-vendor) and also links `cudarc` for an NVIDIA-specific AI inference pass (cuDNN/TensorRT). The two subsystems share no code and communicate only through CPU buffers or CUDA–Vulkan interop (`VK_KHR_external_memory`).

**rust-gpu's position within the Vulkan path** is as an alternative shader authoring language — not a different runtime. If you are already using `wgpu`, you can switch from WGSL to rust-gpu shaders without changing any host code: the SPIR-V module that `wgpu::Device::create_shader_module` receives looks identical to naga's WGSL-compiled output. The choice is purely about whether you want to write shader logic in Rust (with `#[spirv]` attributes and `spirv-builder` in `build.rs`) or in WGSL (with `include_str!` or `wgpu::ShaderSource::Wgsl`).

---

## ash: Raw Vulkan Bindings in Rust

### What ash Is

`ash` (crates.io: `ash`) provides thin, zero-cost Rust wrappers around Vulkan. All Vulkan functions are loaded at runtime via `vkGetInstanceProcAddr`/`vkGetDeviceProcAddr`. ash does not add safety or lifetimes — it is `unsafe` and mirrors the Vulkan C API closely.

```toml
[dependencies]
ash = "0.38"
```

### Instance and Device Creation

```rust
use ash::{vk, Entry, Instance};

unsafe fn create_instance() -> (Entry, Instance) {
    let entry = Entry::linked();

    let app_info = vk::ApplicationInfo::default()
        .application_name(c"MyApp")
        .api_version(vk::make_api_version(0, 1, 3, 0));

    let layer_names = [c"VK_LAYER_KHRONOS_validation".as_ptr()];
    let extension_names = [
        ash::khr::surface::NAME.as_ptr(),
        ash::khr::wayland_surface::NAME.as_ptr(),
    ];

    let create_info = vk::InstanceCreateInfo::default()
        .application_info(&app_info)
        .enabled_layer_names(&layer_names)
        .enabled_extension_names(&extension_names);

    let instance = entry.create_instance(&create_info, None).unwrap();
    (entry, instance)
}
```

### Extension Loading Pattern

ash uses a strongly typed extension loader pattern:

```rust
use ash::khr;

unsafe fn setup_extensions(entry: &ash::Entry, instance: &ash::Instance,
    device: &ash::Device)
{
    let surface_loader      = khr::surface::Instance::new(entry, instance);
    let wayland_surface     = khr::wayland_surface::Instance::new(entry, instance);
    let swapchain_loader    = khr::swapchain::Device::new(instance, device);
    let sync2_loader        = khr::synchronization2::Device::new(instance, device);

    let surface  = wayland_surface.create_wayland_surface(&create_info, None).unwrap();
    let (swapchain, images) = swapchain_loader.create_swapchain(&swapchain_ci, None).unwrap();
}
```

### Memory Management with ash

ash provides no memory allocator. Combine with `gpu-allocator`:

```rust
use gpu_allocator::vulkan::{Allocator, AllocatorCreateDesc, AllocationCreateDesc};

let mut allocator = Allocator::new(&AllocatorCreateDesc {
    instance: instance.clone(),
    device: device.clone(),
    physical_device,
    debug_settings: Default::default(),
    buffer_device_address: true,
    allocation_sizes: Default::default(),
}).unwrap();

let allocation = allocator.allocate(&AllocationCreateDesc {
    name: "vertex buffer",
    requirements: vk::MemoryRequirements { size: 1024 * 1024, .. },
    location: gpu_allocator::MemoryLocation::GpuOnly,
    linear: true,
    allocation_scheme: Default::default(),
}).unwrap();

let buffer = device.create_buffer(&vk::BufferCreateInfo::default()
    .size(1024 * 1024)
    .usage(vk::BufferUsageFlags::VERTEX_BUFFER | vk::BufferUsageFlags::TRANSFER_DST),
    None).unwrap();
device.bind_buffer_memory(buffer, allocation.memory(), allocation.offset()).unwrap();
```

---

## wgpu: The Cross-Platform GPU Abstraction

### wgpu Architecture

```
Application
    │
    ▼
wgpu (safe Rust API — WebGPU standard)
    │
    ▼
wgpu-hal (unsafe backend trait)
    ├─ vulkan   ← Linux primary backend
    ├─ gles     ← OpenGL ES (Mesa/Zink)
    ├─ metal    ← macOS/iOS
    ├─ dx12     ← Windows
    └─ webgpu   ← web browsers (via WASM)
```

### Device Creation

```rust
use wgpu::{Instance, InstanceDescriptor, Backends, RequestAdapterOptions, PowerPreference};

async fn init_wgpu() -> (wgpu::Device, wgpu::Queue) {
    let instance = Instance::new(&InstanceDescriptor {
        backends: Backends::VULKAN | Backends::GL,
        ..Default::default()
    });

    let adapter = instance
        .request_adapter(&RequestAdapterOptions {
            power_preference: PowerPreference::HighPerformance,
            compatible_surface: None,
            force_fallback_adapter: false,
        })
        .await
        .unwrap();

    println!("Using: {} ({:?})", adapter.get_info().name, adapter.get_info().backend);

    let (device, queue) = adapter
        .request_device(&wgpu::DeviceDescriptor {
            required_features: wgpu::Features::TEXTURE_COMPRESSION_BC
                | wgpu::Features::TIMESTAMP_QUERY,
            required_limits: wgpu::Limits::default(),
            label: None,
            memory_hints: Default::default(),
        }, None)
        .await
        .unwrap();

    (device, queue)
}
```

### Buffer and Texture Creation

```rust
let vertex_buffer = device.create_buffer(&wgpu::BufferDescriptor {
    label: Some("vertex buffer"),
    size: 1024 * 1024,
    usage: wgpu::BufferUsages::VERTEX | wgpu::BufferUsages::COPY_DST,
    mapped_at_creation: false,
});

queue.write_buffer(&vertex_buffer, 0, bytemuck::cast_slice(&vertices));

let texture = device.create_texture(&wgpu::TextureDescriptor {
    label: Some("render target"),
    size: wgpu::Extent3d { width: 1920, height: 1080, depth_or_array_layers: 1 },
    mip_level_count: 1,
    sample_count: 1,
    dimension: wgpu::TextureDimension::D2,
    format: wgpu::TextureFormat::Rgba8UnormSrgb,
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT | wgpu::TextureUsages::TEXTURE_BINDING,
    view_formats: &[],
});
```

---

## wgpu-hal: The Backend Abstraction Layer

### The hal Trait

`wgpu-hal` defines the unsafe backend interface that all platform backends implement:

```rust
/* wgpu/wgpu-hal/src/lib.rs (simplified): */
pub trait Api: Clone + Sized {
    type Instance: Instance<A = Self>;
    type Adapter: Adapter<A = Self>;
    type Device: Device<A = Self>;
    type Queue: Queue<A = Self>;
    type CommandEncoder: CommandEncoder<A = Self>;
    type Buffer: Send + Sync;
    type Texture: Send + Sync;
}

pub trait Device<A: Api>: Send + Sync {
    unsafe fn create_buffer(&self, desc: &BufferDescriptor) -> Result<A::Buffer, DeviceError>;
    unsafe fn create_texture(&self, desc: &TextureDescriptor) -> Result<A::Texture, DeviceError>;
}
```

### Vulkan Backend Key Logic

```rust
/* wgpu/wgpu-hal/src/vulkan/device.rs */
impl crate::Device<super::Api> for super::Device {
    unsafe fn create_buffer(
        &self,
        desc: &crate::BufferDescriptor,
    ) -> Result<super::Buffer, crate::DeviceError> {
        let vk_info = vk::BufferCreateInfo::default()
            .size(desc.size)
            .usage(conv::map_buffer_usage(desc.usage))
            .sharing_mode(vk::SharingMode::EXCLUSIVE);

        let raw = self.shared.raw.create_buffer(&vk_info, None)?;
        let allocation = self.mem_allocator.lock().allocate(&AllocationCreateDesc {
            requirements: self.shared.raw.get_buffer_memory_requirements(raw),
            location: if desc.memory_flags.contains(crate::MemoryFlags::PREFER_COHERENT) {
                gpu_allocator::MemoryLocation::CpuToGpu
            } else {
                gpu_allocator::MemoryLocation::GpuOnly
            },
            ..Default::default()
        })?;
        self.shared.raw.bind_buffer_memory(raw, allocation.memory(), allocation.offset())?;
        Ok(super::Buffer { raw, allocation })
    }
}
```

---

## naga: The Shader Compiler

### naga's Role

naga parses WGSL, GLSL, or SPIR-V, builds an IR, and emits SPIR-V, GLSL, HLSL, or MSL:

```
Input:   WGSL ─┐
         GLSL ─┤→ naga IR → SPIR-V → Vulkan
       SPIR-V ─┘           → GLSL   → OpenGL/GLES
                            → HLSL   → D3D12
                            → MSL    → Metal
```

### naga IR Structure

```rust
pub struct Module {
    pub types: Arena<Type>,
    pub constants: Arena<Constant>,
    pub global_variables: Arena<GlobalVariable>,
    pub functions: Arena<Function>,
    pub entry_points: Vec<EntryPoint>,
}

pub struct Function {
    pub arguments: Vec<FunctionArgument>,
    pub result: Option<FunctionResult>,
    pub local_variables: Arena<LocalVariable>,
    pub body: Block,
}
```

### Using naga Standalone

```rust
use naga::{front::wgsl, back::spv, valid};

fn compile_wgsl_to_spirv(wgsl_source: &str) -> Vec<u32> {
    let module = wgsl::parse_str(wgsl_source).expect("WGSL parse failed");

    let info = valid::Validator::new(
        valid::ValidationFlags::all(),
        valid::Capabilities::all(),
    )
    .validate(&module)
    .expect("Validation failed");

    let options = spv::Options {
        lang_version: (1, 6),
        flags: spv::WriterFlags::ADJUST_COORDINATE_SPACE,
        ..Default::default()
    };
    spv::write_vec(&module, &info, &options, None).expect("SPIR-V emit failed")
}
```

---

## WGSL: WebGPU Shading Language

### WGSL Vertex and Fragment Shaders

```wgsl
struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) texcoord: vec2<f32>,
};

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) texcoord: vec2<f32>,
};

@group(0) @binding(0)
var<uniform> transform: mat4x4<f32>;

@vertex
fn vs_main(in: VertexInput) -> VertexOutput {
    var out: VertexOutput;
    out.clip_position = transform * vec4<f32>(in.position, 1.0);
    out.texcoord = in.texcoord;
    return out;
}

@group(0) @binding(1) var my_texture: texture_2d<f32>;
@group(0) @binding(2) var my_sampler: sampler;

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    return textureSample(my_texture, my_sampler, in.texcoord);
}
```

### WGSL Compute Shader

```wgsl
@group(0) @binding(0) var<storage, read>       a: array<f32>;
@group(0) @binding(1) var<storage, read>       b: array<f32>;
@group(0) @binding(2) var<storage, read_write> out: array<f32>;

@compute @workgroup_size(64, 1, 1)
fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
    let i = gid.x;
    if i >= arrayLength(&a) { return; }
    out[i] = a[i] + b[i];
}
```

---

## GPU Compute with wgpu

```rust
async fn run_compute(device: &wgpu::Device, queue: &wgpu::Queue) {
    let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
        label: Some("compute"),
        source: wgpu::ShaderSource::Wgsl(include_str!("compute.wgsl").into()),
    });

    let pipeline = device.create_compute_pipeline(&wgpu::ComputePipelineDescriptor {
        label: Some("add pipeline"),
        layout: None,
        module: &shader,
        entry_point: Some("main"),
        compilation_options: Default::default(),
        cache: None,
    });

    let a = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
        label: Some("a"),
        contents: bytemuck::cast_slice(&[1.0f32; 1024]),
        usage: wgpu::BufferUsages::STORAGE,
    });
    let out = device.create_buffer(&wgpu::BufferDescriptor {
        label: Some("output"),
        size: 1024 * 4,
        usage: wgpu::BufferUsages::STORAGE | wgpu::BufferUsages::COPY_SRC,
        mapped_at_creation: false,
    });

    let bind_group = device.create_bind_group(&wgpu::BindGroupDescriptor {
        label: None,
        layout: &pipeline.get_bind_group_layout(0),
        entries: &[
            wgpu::BindGroupEntry { binding: 0, resource: a.as_entire_binding() },
            wgpu::BindGroupEntry { binding: 2, resource: out.as_entire_binding() },
        ],
    });

    let mut encoder = device.create_command_encoder(&Default::default());
    {
        let mut pass = encoder.begin_compute_pass(&Default::default());
        pass.set_pipeline(&pipeline);
        pass.set_bind_group(0, &bind_group, &[]);
        pass.dispatch_workgroups(1024 / 64, 1, 1);
    }
    queue.submit([encoder.finish()]);
}
```

---

## Rendering Pipelines with wgpu

```rust
let surface = instance.create_surface(&window).unwrap();
let surface_caps = surface.get_capabilities(&adapter);
let surface_format = surface_caps.formats.iter()
    .find(|f| f.is_srgb()).copied()
    .unwrap_or(surface_caps.formats[0]);

surface.configure(&device, &wgpu::SurfaceConfiguration {
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT,
    format: surface_format,
    width: window_size.width,
    height: window_size.height,
    present_mode: wgpu::PresentMode::Fifo,
    desired_maximum_frame_latency: 2,
    alpha_mode: wgpu::CompositeAlphaMode::Auto,
    view_formats: vec![],
});

let output = surface.get_current_texture().unwrap();
let view = output.texture.create_view(&Default::default());
let mut encoder = device.create_command_encoder(&Default::default());
{
    let mut rp = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
        color_attachments: &[Some(wgpu::RenderPassColorAttachment {
            view: &view,
            resolve_target: None,
            ops: wgpu::Operations {
                load: wgpu::LoadOp::Clear(wgpu::Color { r: 0.1, g: 0.2, b: 0.3, a: 1.0 }),
                store: wgpu::StoreOp::Store,
            },
        })],
        ..Default::default()
    });
    rp.set_pipeline(&render_pipeline);
    rp.draw(0..3, 0..1);
}
queue.submit([encoder.finish()]);
output.present();
```

---

## Ecosystem: erupt, vulkano, gpu-allocator

### erupt: Alternative Raw Vulkan Bindings

> **Maintenance status:** erupt's last meaningful release was 0.23 (2022). The crate is effectively unmaintained; new Vulkan extensions added after 2022 are not reflected in it. For new projects, use `ash` instead.

`erupt` is an auto-generated alternative to `ash` with slightly different ergonomics: it exposes a flat module namespace and uses `vk1_0`, `vk1_1`, `vk1_2` feature flags to control API version. For projects already using it, the API remains functional for core Vulkan 1.2 and earlier extension set.

```toml
[dependencies]
erupt = "0.23"  # unmaintained; prefer ash for new work
```

### vulkano: Safe Vulkan Abstraction

`vulkano` aims to encode Vulkan validity rules in Rust's type system:

```rust
use vulkano::buffer::{Buffer, BufferCreateInfo, BufferUsage};
use vulkano::memory::allocator::{AllocationCreateInfo, MemoryTypeFilter};

let buffer = Buffer::new_slice::<f32>(
    &memory_allocator,
    BufferCreateInfo { usage: BufferUsage::STORAGE_BUFFER, ..Default::default() },
    AllocationCreateInfo {
        memory_type_filter: MemoryTypeFilter::HOST_SEQUENTIAL_WRITE,
        ..Default::default()
    },
    1024,
).unwrap();
```

### gpu-allocator: The Standard Memory Allocator

`gpu-allocator` implements TLSF sub-allocation, analogous to VMA in C++:

```toml
[dependencies]
gpu-allocator = { version = "0.27", features = ["vulkan"] }
```

Memory locations:
- `MemoryLocation::GpuOnly` — device-local, not CPU-mappable
- `MemoryLocation::CpuToGpu` — host-visible + device-local (upload buffer)
- `MemoryLocation::GpuToCpu` — host-visible for readback

### rust-gpu: Rust Shaders on the GPU

[rust-gpu](https://github.com/EmbarkStudios/rust-gpu) (Embark Studios) compiles Rust to SPIR-V:

```rust
/* Rust shader (compiles to SPIR-V via rust-gpu): */
#![no_std]
use spirv_std::glam::{vec4, Vec4};
use spirv_std::spirv;

#[spirv(vertex)]
pub fn main_vs(#[spirv(position)] out_pos: &mut Vec4) {
    *out_pos = vec4(0.0, 1.0, 0.0, 1.0);
}
```

### Bevy Game Engine

Bevy uses `wgpu` as its rendering backend:
- `bevy_render` — render app, command encoder abstraction
- `bevy_pbr` — PBR material system with WGSL shaders
- `bevy_core_pipeline` — shadow, bloom, SSAO, TAA passes

```bash
WGPU_BACKEND=vulkan cargo run   # explicit Vulkan
WGPU_BACKEND=gl cargo run       # OpenGL fallback
```

---

## cuTile-rs: NVIDIA Tile Abstraction in Rust

[cuTile-rs](https://github.com/nvlabs/cutile-rs) is NVIDIA's official Rust library for tensor-core–oriented GPU kernel authoring. It forms part of NVIDIA's strategic investment in Rust for performance-critical GPU code — evidence that the company views Rust as viable beyond systems software and into HPC kernel authoring.

### The Tile Programming Model

The central idea is *tile programs, not threads*: a kernel is written as a single-threaded program that operates on typed data tiles, and the compiler (backed by CUDA Tile IR) maps those operations onto warps, thread-blocks, and Tensor Cores automatically. The programmer never manually manages shared memory, warp-level synchronisation, or `wmma` instruction selection.

The core abstraction is a `Tensor<T, Shape>` generic type (with compile-time shape notation such as `Tensor<f32, { [M, N] }>`) representing a block of data held cooperatively by a group of threads. On NVIDIA hardware, CUDA Tile IR maps tile operations to the PTX cooperative matrix family — `wmma.load`, `wmma.mma.sync`, and `wmma.store` for Ampere and Ada Lovelace, and the newer `wgmma` (Warp-Group Matrix Multiply Accumulate, sm_90+) for Hopper — giving Tensor Core throughput behind a safe Rust API. [PTX reference](https://docs.nvidia.com/cuda/parallel-thread-execution/)

### Kernel Pattern

Kernels are defined inside a `#[cutile::module]` block. Each kernel function is annotated with `#[cutile::entry()]`. Shape dimensions are compile-time generics in const-expression notation:

```rust
use cutile::prelude::*;

// The #[cutile::module] macro processes this mod block at build time,
// JIT-compiling kernel functions through CUDA Tile IR to cubin.
#[cutile::module]
mod kernels {
    use cutile::core::*;

    // Element-wise addition kernel — illustrates the core API.
    // For GEMM/matmul, CUDA Tile IR selects the appropriate Tensor Core
    // instructions (wmma/wgmma) automatically based on the target GPU.
    #[cutile::entry()]
    fn tile_add<const N: i32>(
        z: &mut Tensor<f32, { [N] }>,
        x: &Tensor<f32, { [-1] }>,  // -1 = dynamic size inferred from partition
        y: &Tensor<f32, { [-1] }>,
    ) {
        // load_tile_like: free function; loads x and y shaped to match z
        let tx = load_tile_like(x, z);
        let ty = load_tile_like(y, z);
        // store: writes the tile expression result back to z
        z.store(tx + ty);
    }
}
```

Key API points:
- **`#[cutile::module]`** on the `mod` block (not `#[kernel]` on individual functions)
- **`#[cutile::entry()]`** on each kernel function
- **`Tensor<T, Shape>`** with const-expression shape (not `Tensor2D<T>`)
- **`load_tile_like(src, template)`** — free function, not a method; loads `src` partitioned to match `template`'s shape
- **`z.store(expr)`** — writes a tile expression result to the output tensor

On the host side, tiles are partitioned and launched via cuTile-rs's own `cuda-core` / `cuda-bindings` crates, which wrap the CUDA Driver API independently of cudarc. On B200 hardware, cuTile-rs GEMM kernels reach 96.4% of cuBLAS throughput. [Source](https://nvlabs.github.io/cutile-rs/main/)

### Why NVIDIA Is Investing: "Fearless Concurrency on the GPU"

NVIDIA's formal case is made in a research paper:

> **"Fearless Concurrency on the GPU"** — Melih Elibol, Jared Roesch, Isaac Gelado, Eric Buehler, Michael Garland (NVIDIA Research); arXiv:2606.15991, June 2026. [[arxiv.org](https://arxiv.org/abs/2606.15991)]

The paper's thesis, verbatim from the abstract: *"Rust has made safe systems programming practical on the CPU, but writing custom GPU kernels in Rust still forces programmers outside the language's ownership guarantees. We present cuTile Rust, a tile-based system for safe, idiomatic GPU kernel authoring in Rust... Our evaluation shows that these abstractions can preserve performance on high-end GPUs."*

The central claim: Rust ownership, extended to tiles at the GPU kernel level, statically encodes GPU memory hazards — concurrent read/write aliasing — that CUDA C++ can only catch at runtime via the compute sanitizer. cuTile-rs partitions mutable tensors into disjoint pieces before launch (ownership ≈ exclusive access) and shares immutable tensors across threads (read-only ≈ shared reference). This directly mirrors the borrow checker's aliasing XOR mutability rule, applied at the tile granularity.

Performance from the paper (NVIDIA B200 hardware):
- Element-wise operations: **7 TB/s**
- Safe Rust GEMM (persistent kernel, f16, M=N=K=8192): **2.07 PFlop/s** — 92% of B200 peak, within 0.3% of low-level Tile IR
- cuBLAS comparison: **96.4% of cuBLAS throughput** for GEMM
- Inference engine "Grout" (built on cuTile-rs): **171 tok/s** Qwen3-4B on RTX 5090; **82 tok/s** Qwen3-32B on B200 — competitive with vLLM and SGLang

The tile abstraction also decouples algorithmic intent from microarchitecture. A GEMM that today targets Hopper's `wgmma` instructions can retarget Blackwell's next-generation matrix engine by recompiling; the kernel source stays unchanged. This composability is exactly what custom training loop authors need when they want to experiment with quantised data types (fp8, int4) or non-standard accumulation orders without rewriting schedule logic. The companion `TileGym` benchmark suite tests these kernels against cuBLAS baselines. [Source](https://nvlabs.github.io/cutile-rs/main/)

cuTile-rs targets research teams and framework authors (e.g., cuDNN successor kernels) rather than end-application developers. It is an NVlabs research project — NVIDIA's position is that Rust ownership can be extended to GPU kernels at the tile level, achieving safety without sacrificing Tensor Core performance. [Source](https://github.com/nvlabs/cutile-rs)

---

## cudarc: Rust CUDA Driver API

[cudarc](https://github.com/coreylowman/cudarc) provides safe, ergonomic Rust wrappers around the CUDA Driver API, cuBLAS, cuDNN, cuRAND, NVRTC, and related libraries. It is the standard Rust crate for driving NVIDIA GPUs from host code without writing CUDA C++.

### Architecture

cudarc exposes three API tiers for each wrapped library:

- **Safe API** — ergonomic types, error as `Result`, no `unsafe` at call sites
- **Result API** — wraps raw FFI with error checking, some `unsafe`
- **Sys API** — raw generated FFI bindings, fully `unsafe`

The primary entry points are `CudaContext` (device handle, maps to `CUcontext`) and `CudaStream` (command queue, maps to `CUstream`), with `CudaSlice<T>` as the typed GPU allocation. [Source](https://docs.rs/cudarc)

### Device Initialisation and Memory Allocation

```rust
use cudarc::driver::safe::{CudaContext, CudaStream, LaunchConfig};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a context on GPU 0 (analogous to cudaSetDevice(0)).
    let ctx = CudaContext::new(0)?;
    let stream = ctx.default_stream();

    // Allocate device memory initialised to zero — returns CudaSlice<f32>.
    let mut output: cudarc::driver::CudaSlice<f32> =
        stream.alloc_zeros::<f32>(1024)?;

    // Allocate and copy from host slice (htod = host-to-device).
    let input_data: Vec<f32> = (0..1024).map(|x| x as f32).collect();
    let input = stream.clone_htod(&input_data)?;

    Ok(())
}
```

### Launching a PTX Kernel

cudarc loads PTX assembly produced offline (by `nvcc -ptx`, NVRTC, or the cuTile-rs compiler pipeline) via `CudaContext::load_module`, then retrieves individual kernel handles with `CudaModule::load_function`:

```rust
use cudarc::driver::safe::{CudaContext, CudaModule, LaunchConfig};
use cudarc::driver::Ptx;

fn launch_ptx_kernel() -> Result<(), Box<dyn std::error::Error>> {
    let ctx = CudaContext::new(0)?;
    let stream = ctx.default_stream();

    // Load PTX from a file produced by nvcc or cuTile-rs.
    // Ptx::from_file reads the .ptx text; Ptx::from_src accepts a &str.
    let ptx    = Ptx::from_file("./add_kernel.ptx");
    let module = ctx.load_module(ptx)?;              // Arc<CudaModule>

    // Retrieve the specific kernel entry point by name.
    let add_fn = module.load_function("add_kernel")?;

    let mut out = stream.alloc_zeros::<f32>(1024)?;
    let inp     = stream.clone_htod(&vec![1.0f32; 1024])?;

    // Type-checked builder: args must match the PTX parameter list order.
    let mut builder = stream.launch_builder(&add_fn);
    builder.arg(&mut out);
    builder.arg(&inp);
    unsafe { builder.launch(LaunchConfig::for_num_elems(1024)) }?;

    // clone_dtoh allocates a new Vec<T> from device memory (device-to-host)
    let result: Vec<f32> = stream.clone_dtoh(&out)?;
    println!("result[0] = {}", result[0]);   // 1.0
    Ok(())
}
```

`LaunchConfig::for_num_elems(n)` selects a sensible block/grid split automatically. For hand-tuned configurations supply `LaunchConfig { block_dim: (256,1,1), grid_dim: (n/256, 1, 1), shared_mem_bytes: 0 }`. The PTX can also be compiled at run time via the bundled `cudarc::nvrtc::compile_ptx` wrapper, which invokes NVRTC in-process from a CUDA C source string — useful for JIT compilation of generated kernels. [Source](https://docs.rs/cudarc/latest/cudarc/driver/safe/struct.CudaContext.html)

### cuTile-rs vs cudarc: Independent CUDA Stacks

cuTile-rs and cudarc are **independent** libraries with separate CUDA integration paths — they do not pair in a producer/consumer relationship.

**cuTile-rs** includes its own `cuda-core` and `cuda-bindings` crates. When a `#[cutile::module]` block is processed, cuTile-rs JIT-compiles the kernel AST through CUDA Tile IR directly to a **cubin** (binary). It uses its own CUDA Driver API wrappers to load and launch that cubin. It does not produce PTX for another crate to load.

**cudarc** is a general-purpose safe Rust wrapper around the CUDA Driver API — for loading PTX kernels, wrapping cuBLAS/cuDNN/cuRAND, and managing memory. It is independent of cuTile-rs.

A project *can* use both crates simultaneously — cuTile-rs for tiled Tensor Core kernels and cudarc for cuBLAS calls or custom PTX kernels — because both ultimately wrap the same CUDA Driver API. But they do not integrate: cuTile-rs kernels are launched through cuTile-rs APIs; cudarc kernels are launched through cudarc APIs. There is no workflow where cuTile-rs emits PTX that cudarc loads.

The build-system analogy with rust-gpu/spirv-builder is apt at a conceptual level (both separate kernel compilation from host code) but the mechanism differs: rust-gpu emits SPIR-V consumed by wgpu; cuTile-rs emits cubin consumed by its own runtime. [Source: cuTile-rs](https://github.com/NVlabs/cutile-rs) | [cudarc](https://github.com/coreylowman/cudarc)

---

## Vulkano: Compile-Time Safe Vulkan

[vulkano](https://github.com/vulkano-rs/vulkano) encodes as much of the Vulkan specification as possible into Rust's type system, turning Vulkan validity violations that would be runtime errors (or worse, silent corruption) into compile-time failures.

> **Note:** The existing Ecosystem section contains a brief `vulkano` buffer-allocation snippet. This section expands on the pipeline and shader compilation side of the API.

### Compile-Time Shader Integration

The `vulkano_shaders::shader!` macro compiles GLSL inline or from a file at Cargo build time, emitting a Rust module with typed structs for every uniform block, push constant, and vertex input:

```rust
mod vs {
    vulkano_shaders::shader! {
        ty: "vertex",
        src: r"
            #version 450
            layout(location = 0) in vec2 position;
            void main() {
                gl_Position = vec4(position, 0.0, 1.0);
            }
        ",
    }
}

mod fs {
    vulkano_shaders::shader! {
        ty: "fragment",
        src: r"
            #version 450
            layout(location = 0) out vec4 f_color;
            void main() { f_color = vec4(1.0, 0.0, 0.0, 1.0); }
        ",
    }
}
```

The macro calls `glslc` (or the bundled `shaderc`) and embeds the SPIR-V as a byte literal. The generated `load()` function returns an `Arc<ShaderModule>` tied to a specific `Device` — making it impossible to mix shaders across devices. [Source](https://github.com/vulkano-rs/vulkano/tree/master/examples)

### Graphics Pipeline Creation

```rust
use vulkano::{
    pipeline::{
        GraphicsPipeline, GraphicsPipelineCreateInfo, PipelineLayout,
        graphics::{
            vertex_input::VertexDefinition,
            viewport::{Viewport, ViewportState},
            input_assembly::InputAssemblyState,
            rasterization::RasterizationState,
            multisample::MultisampleState,
            color_blend::ColorBlendState,
        },
    },
    render_pass::Subpass,
};

let vs = unsafe { vs::load(device.clone()) }.unwrap();
let fs = unsafe { fs::load(device.clone()) }.unwrap();

let vertex_input_state = MyVertex::per_vertex()
    .definition(&vs.info().input_interface)
    .unwrap();

let stages = [
    PipelineShaderStageCreateInfo::new(vs),
    PipelineShaderStageCreateInfo::new(fs),
];
let layout = PipelineLayout::from_stages(&device, &stages).unwrap();

let subpass = Subpass::from(render_pass.clone(), 0).unwrap();
let pipeline = GraphicsPipeline::new(
    device.clone(),
    None,
    GraphicsPipelineCreateInfo {
        stages:              stages.into_iter().collect(),
        vertex_input_state:  Some(vertex_input_state),
        input_assembly_state: Some(InputAssemblyState::default()),
        viewport_state:      Some(ViewportState::default()),
        rasterization_state: Some(RasterizationState::default()),
        multisample_state:   Some(MultisampleState::default()),
        color_blend_state:   Some(ColorBlendState::with_attachment_states(
            1, Default::default(),
        )),
        subpass:             Some(subpass.into()),
        dynamic_state:       [DynamicState::Viewport].into_iter().collect(),
        ..GraphicsPipelineCreateInfo::layout(layout)
    },
).unwrap();
```

### Type-System Safety Guarantees

vulkano's key invariant is render-pass compatibility: `Subpass::from(render_pass, 0)` binds the pipeline type to a concrete render-pass object, and the Rust type system prevents using the pipeline with an incompatible render pass at no runtime overhead. `Subbuffer<T>` wraps a typed slice view over a `Buffer`, enabling the allocator to reject mismatched bindings at compile time. [Source](https://docs.rs/vulkano)

**Historical note — `AutoCommandBufferBuilder`.** In vulkano versions through 0.34, command recording used an `AutoCommandBufferBuilder<PrimaryAutoCommandBuffer>` that enforced render-pass state via Rust generics: the type parameter transitioned between `OutsideRenderPass` and `InsideRenderPass` states so that calling `draw()` outside a render pass was a compile error. The current vulkano master replaced this with a *task graph* abstraction (`RecordingCommandBuffer`) where synchronisation is derived from declared resource accesses rather than from type-state transitions. The two designs make the same guarantee — you cannot submit a mis-ordered command sequence — but via different mechanisms: the old approach is more explicit about render-pass bracketing, while the task graph approach scales better to complex multi-pass frames with automatic dependency tracking. If you encounter vulkano tutorials referencing `AutoCommandBufferBuilder::primary(...)`, they target the 0.34 era; the current pipeline-creation pattern shown above is from master as of 2025–2026. [Source](https://github.com/vulkano-rs/vulkano/tree/master/examples)

---

## rust-gpu: Rust as a Shading Language

[rust-gpu](https://github.com/rust-gpu/rust-gpu) (originally Embark Studios, now community-governed under the Rust-GPU GitHub organisation) compiles ordinary Rust code to SPIR-V, making Rust usable as a shading language for Vulkan, wgpu, and any SPIR-V consumer.

> **Note:** The Ecosystem section contains a short stub for rust-gpu. This section provides the full treatment.

### spirv-builder: Compiling Shaders in build.rs

The `spirv-builder` crate invokes `rustc` with `--target spirv-unknown-vulkan1.2` from a Cargo `build.rs` script:

```rust
// shader_crate/build.rs
use spirv_builder::{MetadataPrintout, SpirvBuilder};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    SpirvBuilder::new("../shader_crate", "spirv-unknown-vulkan1.2")
        .print_metadata(MetadataPrintout::Full)
        .build()?;
    Ok(())
}
```

The compiled `.spv` file path is written to `OUT_DIR`, where it can be embedded with `include_bytes!` in the host crate. [Source](https://github.com/rust-gpu/rust-gpu)

### Shader Entry Points

Entry points are ordinary Rust functions tagged with `#[spirv(...)]` attributes from `spirv-std`:

```rust
#![no_std]
use spirv_std::spirv;
use spirv_std::glam::{Vec2, Vec4, vec4};

/// Vertex shader: passes through clip-space position.
#[spirv(vertex)]
pub fn main_vs(
    #[spirv(vertex_index)] vert_idx: u32,
    #[spirv(position)] out_pos: &mut Vec4,
) {
    // Full-screen triangle trick — no vertex buffer needed.
    let x = (vert_idx & 1) as f32 * 4.0 - 1.0;
    let y = (vert_idx >> 1) as f32 * 4.0 - 1.0;
    *out_pos = vec4(x, y, 0.0, 1.0);
}

/// Fragment shader: samples a 2-D texture.
#[spirv(fragment)]
pub fn main_fs(
    #[spirv(frag_coord)] frag_coord: Vec4,
    #[spirv(descriptor_set = 0, binding = 0)] tex: &spirv_std::image::Image2d,
    #[spirv(descriptor_set = 0, binding = 1)] sampler: &spirv_std::Sampler,
    output: &mut Vec4,
) {
    let uv = Vec2::new(frag_coord.x / 1920.0, frag_coord.y / 1080.0);
    *output = tex.sample(*sampler, uv);
}
```

Compute shaders use `#[spirv(compute(threads(64, 1, 1)))]` and receive `#[spirv(global_invocation_id)] id: glam::UVec3` as built-in inputs.

### spirv-std: The GPU Standard Library

`spirv-std` provides the primitives that the GPU needs but the Rust standard library does not supply in a `#![no_std]` context:

- **`Image<...>` / `Image2d`** — typed texture handles with `sample()`, `fetch()`, `read()`, `write()`
- **`Sampler`** — opaque sampler handle
- **`arch::`** — architecture intrinsics (barriers, fences, atomics)
- **`glam` re-export** — `Vec2`, `Vec4`, `Mat4`, etc. as GPU-compatible types

### Limitations

rust-gpu imposes hard constraints that stem from SPIR-V's execution model:

- **No dynamic dispatch** — trait objects (`dyn Trait`) are not supported; all dispatch must be monomorphised at compile time.
- **No recursion** — SPIR-V prohibits recursive call graphs; the compiler rejects programs that contain cycles.
- **No heap allocation** — `Box`, `Vec`, `HashMap` are unavailable; all data lives in fixed-size locals or buffer bindings.
- **Limited `std`** — only `#![no_std]`-compatible crates are usable in shader crates.
- **Experimental `rustc` backend** — the `rustc_codegen_spirv` backend is not stable; it must be built from a pinned nightly toolchain specified in `rust-toolchain.toml`. [Source](https://github.com/rust-gpu/rust-gpu)

---

## blade: Minimal GPU Library

[blade](https://github.com/kvark/blade) (authored by Mykhailo Parfeniuk / kvark, the architect of wgpu's `wgpu-hal`) is a deliberately small GPU abstraction library. Where wgpu targets maximum portability through a layered architecture, blade targets minimal indirection for research renderers, game prototypes, and tools that only need to run on one or two backends.

### Design Philosophy

blade collapses wgpu's `wgpu → wgpu-hal → backend` stack into a single `blade-graphics` crate. There is no backend abstraction trait: on Linux the Vulkan path is compiled directly; on macOS it is Metal; on the web it is WebGL via GLES. The library uses `ash` for Vulkan directly (no additional wrapper) and exposes a minimal surface area.

Resource limits enforce the philosophy: a maximum of 8 resources per bind group (`RESOURCES_IN_GROUP`), 256 bytes of plain data per pipeline (`PLAIN_DATA_SIZE`), and 1 000 passes per command encoder (`PASS_COUNT`). These are deliberate constraints that prevent API patterns that scale poorly.

### Context Initialisation and Resource Creation

```rust
use blade_graphics as gpu;

fn main() -> Result<(), gpu::PlatformError> {
    // Single-call init; selects Vulkan on Linux.
    let context = unsafe {
        gpu::Context::init(gpu::ContextDesc {
            validation: cfg!(debug_assertions),
            overlay: false,
            ..Default::default()
        })?
    };

    // Allocate a GPU-only buffer (no explicit memory type selection needed).
    let vertex_buf = context.create_buffer(gpu::BufferDesc {
        name:  "vertices",
        size:  (3 * std::mem::size_of::<[f32; 4]>()) as u64,
        memory: gpu::Memory::Device,
    });

    // Create a command encoder for a single frame.
    let mut encoder = context.create_command_encoder(gpu::CommandEncoderDesc {
        name:        "frame",
        buffer_count: 2,
    });

    // ... record render pass ...

    // Submit and get a sync point for CPU–GPU synchronisation.
    let sync = context.submit(&mut encoder);
    context.wait_for(sync, !0); // wait indefinitely
    Ok(())
}
```

[Source: blade-graphics crate docs](https://docs.rs/blade-graphics)

### Comparison with wgpu

| Aspect | wgpu | blade |
|---|---|---|
| Backend selection | Runtime via `wgpu-hal` trait | Compile-time `#[cfg]` |
| Memory management | Hidden inside HAL | Explicit `Memory::Device` / `Memory::Shared` |
| Render graph | User-built or implicit | Single-pass, manual |
| Target use case | Production apps, Firefox | Research, prototypes, tools |
| `unsafe` surface | Minimal (safe API) | `Context::init` is `unsafe` |

blade's single-pass approach means there is no render graph scheduler: the developer explicitly records passes in order. This is simpler for small renderers but becomes burdensome for complex multi-pass pipelines. The codebase is also small enough to read in a day, making it an excellent learning resource for GPU API internals. kvark originally developed blade as a testbed for ideas that later informed wgpu-hal's design — the `Context` API mirrors `wgpu-hal`'s `Device` trait philosophy while exposing the Vulkan / Metal / GLES selection as a compile-time cfg rather than a runtime dispatch chain. [Source](https://github.com/kvark/blade)

---

## vello: Compute-Based 2D Vector Rendering

[vello](https://github.com/linebender/vello) is a GPU-accelerated 2-D vector renderer from the Linebender project (the same team behind the Xilem UI framework). Unlike Cairo or Skia, which rasterise on the CPU, vello encodes all geometry on the CPU into a compact data stream and executes the entire rasterisation pipeline as WGSL compute shaders on the GPU via wgpu.

### Rendering Pipeline

vello's GPU pipeline has four compute stages:

1. **Coarse rasterisation** — tiles the scene into 16×16 pixel bins, determining which draw calls touch each tile.
2. **Path encoding** — curves are flattened to line segments using the Euler-spiral flatten algorithm.
3. **Fine rasterisation** — each tile is rendered in a single-pass kernel that resolves fill/stroke coverage.
4. **Composite** — the final RGBA image is composited into the wgpu texture.

All shaders are WGSL compiled by naga, with no Vulkan-specific extensions required, making vello portable across any wgpu backend. [Source](https://github.com/linebender/vello)

### Scene Construction and Rendering

```rust
use vello::{
    Scene, Renderer, RendererOptions, RenderParams, AaConfig,
    peniko::{Color, Fill},
    kurbo::{Affine, Circle, Rect},
};

async fn render_scene(device: &wgpu::Device, queue: &wgpu::Queue)
    -> wgpu::Texture
{
    // CPU-side scene encoding — no GPU calls yet.
    let mut scene = Scene::new();
    scene.fill(
        Fill::NonZero,
        Affine::IDENTITY,
        Color::from_rgba8(255, 0, 0, 255),
        None,
        &Circle::new((200.0, 200.0), 100.0),
    );
    scene.fill(
        Fill::NonZero,
        Affine::IDENTITY,
        Color::from_rgba8(0, 128, 255, 200),
        None,
        &Rect::new(50.0, 50.0, 350.0, 350.0),
    );

    // GPU renderer — created once, reused every frame.
    let mut renderer = Renderer::new(device, RendererOptions {
        use_cpu:              false,
        antialiasing_support: vello::AaSupport::all(),
        ..Default::default()
    }).unwrap();

    // Output texture.
    let texture = device.create_texture(&wgpu::TextureDescriptor {
        label:  Some("vello output"),
        size:   wgpu::Extent3d { width: 800, height: 600, depth_or_array_layers: 1 },
        mip_level_count: 1, sample_count: 1,
        dimension: wgpu::TextureDimension::D2,
        format:    wgpu::TextureFormat::Rgba8Unorm,
        usage:     wgpu::TextureUsages::STORAGE_BINDING | wgpu::TextureUsages::COPY_SRC,
        view_formats: &[],
    });
    let view = texture.create_view(&Default::default());

    // Dispatch the compute pipeline — all GPU work happens here.
    renderer.render_to_texture(device, queue, &scene, &view, &RenderParams {
        base_color:          Color::WHITE,
        width:               800,
        height:              600,
        antialiasing_method: AaConfig::Msaa16,
    }).unwrap();

    texture
}
```

### Role in the Xilem / Linux Desktop Ecosystem

Vello is the intended rendering backend for Xilem, Linebender's reactive UI framework, replacing software Cairo on Linux/Wayland. Applications use `winit` for windowing and event dispatch, `wgpu` targeting Vulkan (via `wgpu-hal/vulkan`) for the GPU layer, and vello for all 2-D drawing. The compute-based approach avoids the GPU stall points of traditional rasterisation (no depth buffer, no triangle setup) and achieves better GPU utilisation for vector-heavy UIs than tile-based rasterisers.

One important property of vello's `Scene` model is that it is entirely CPU-side and allocation-free once the internal buffer has been warmed up. A new frame begins with `scene.reset()`, which clears the draw-call list without releasing memory; all geometry is then re-encoded into the same backing storage. This means the GPU compute pipeline always ingests a fresh scene description with zero cross-frame state, which simplifies correctness reasoning dramatically compared to an incremental scene graph where partial redraws and dirty-region tracking can cause subtle rendering artefacts. The tradeoff is that every visible element must be re-submitted every frame — which is acceptable for UI (where most of the scene changes every frame due to animations or scroll) but would be wasteful for a static document renderer. For those use cases vello can cache pre-encoded `Fragment` objects that represent sub-scenes, deferring re-encoding to only the changed portions. [Source](https://github.com/linebender/vello)

---

## burn: Rust ML Framework

[burn](https://github.com/tracel-ai/burn) is a Rust deep-learning framework designed for backend portability. The same model code runs on WGSL compute shaders (via wgpu), CUDA (via cudarc), ROCm, Metal, and WebAssembly without modification. The abstraction key is the `Backend` trait.

### The Backend Trait

Every burn operation is generic over `B: Backend`. Backends are pluggable:

- **`burn-wgpu`** — WGSL compute shaders compiled by naga, runs on any wgpu backend (Vulkan, Metal, DX12, WebGPU).
- **`burn-cuda`** — kernel generation and launch via cudarc, targets NVIDIA CUDA.
- **`burn-ndarray`** — CPU fallback using ndarray, useful for testing.
- **`burn-candle`** — thin wrapper over the Candle ML framework. **Deprecated** as of 2024; use `burn-cuda` (via CubeCL) instead for CUDA targets.

The compute language unifying `burn-wgpu` and `burn-cuda` is **CubeCL**, burn's own GPU DSL that compiles to WGSL or PTX depending on the backend. [Source](https://github.com/tracel-ai/burn)

### Module Derive and Linear Layer

```rust
use burn::{
    nn::{Linear, LinearConfig, Relu},
    module::Module,
    tensor::{backend::Backend, Tensor},
};

/// A simple MLP generic over any burn Backend.
#[derive(Module, Debug)]
pub struct Mlp<B: Backend> {
    fc1: Linear<B>,
    fc2: Linear<B>,
    activation: Relu,
}

impl<B: Backend> Mlp<B> {
    pub fn new(device: &B::Device) -> Self {
        Self {
            fc1: LinearConfig::new(784, 512).init(device),
            fc2: LinearConfig::new(512, 10).init(device),
            activation: Relu::new(),
        }
    }

    /// Forward pass: [batch, 784] → [batch, 10]
    pub fn forward(&self, x: Tensor<B, 2>) -> Tensor<B, 2> {
        let x = self.fc1.forward(x);
        let x = self.activation.forward(x);
        self.fc2.forward(x)
    }
}
```

`#[derive(Module)]` implements parameter serialisation, device movement (`model.to_device(&dev)`), gradient zeroing, and autodiff (`AutodiffModule`) without any boilerplate. The `Linear<B>` struct holds `Param<Tensor<B, 2>>` weight and optional bias, initialised from a uniform distribution `U(-1/√d_in, 1/√d_in)`. [Source](https://docs.rs/burn/latest/burn/nn/struct.Linear.html)

### Running on wgpu vs CUDA

```rust
// Select backend at the call site — model code is identical.
#[cfg(feature = "wgpu")]
type MyBackend = burn_wgpu::Wgpu;

#[cfg(feature = "cuda")]
type MyBackend = burn_cuda::Cuda;

fn main() {
    let device = Default::default();
    let model: Mlp<MyBackend> = Mlp::new(&device);
    let input = Tensor::<MyBackend, 2>::zeros([32, 784], &device);
    let output = model.forward(input);
    println!("output shape: {:?}", output.shape());
}
```

This backend-agnostic design means a model developed and tested on `burn-ndarray` (pure CPU, easy to debug) can be deployed on `burn-wgpu` for Vulkan GPUs — including AMD (via RADV) and Intel (via ANV) — or `burn-cuda` for NVIDIA, with no changes to model code. The CubeCL compiler handles kernel fusion and tiling differently for each target: for WGSL it emits `@compute` shader modules consumed by wgpu, while for CUDA it emits PTX loaded via cudarc's `load_module` / `load_function` path. This positions burn as the Rust equivalent of PyTorch's device-agnostic tensor abstraction, but with Rust's ownership guarantees preventing the reference-counting pitfalls that cause mysterious memory leaks in Python ML frameworks. [Source](https://github.com/tracel-ai/burn)

---

## encase and bytemuck: GPU Buffer Layout

Passing structured data from Rust to GPU shaders requires exact byte layout. WGSL's std430 rules (uniform buffers use std140) impose alignment constraints that differ from Rust's `#[repr(C)]`. Two crates address this complementarily.

### WGSL Alignment Rules

WGSL uniform buffers follow **std140**: each member is aligned to the larger of its natural alignment and 16 bytes. Storage buffers follow **std430**: members align to their natural alignment (4 bytes for `f32`, 16 bytes for `vec4<f32>`). A `struct { f32, vec3<f32> }` in std430 has 4 bytes of padding after the `f32` because `vec3` aligns to 16.

### encase: Derive-Based Padding

[encase](https://crates.io/crates/encase) provides a `ShaderType` derive macro that emits the correct padding for WGSL host-shareable types:

```rust
use encase::{ShaderType, UniformBuffer};
use encase::matrix::AsMat4;

#[derive(ShaderType)]
struct CameraUniform {
    view_proj: glam::Mat4,   // 64 bytes, 16-byte aligned — std140 OK
    eye_pos:   glam::Vec3,   // 12 bytes, but padded to 16 in std140
    _pad:      f32,           // explicit pad to reach 16-byte boundary
    time:      f32,           // 4 bytes
    _pad2:     [f32; 3],     // pad struct to 16-byte multiple
}

fn write_uniform(device: &wgpu::Device, queue: &wgpu::Queue, cam: &CameraUniform)
    -> wgpu::Buffer
{
    let mut byte_buf = UniformBuffer::new(Vec::<u8>::new());
    byte_buf.write(cam).unwrap();
    let bytes = byte_buf.into_inner();

    let buf = device.create_buffer(&wgpu::BufferDescriptor {
        label: Some("camera uniform"),
        size:  bytes.len() as u64,
        usage: wgpu::BufferUsages::UNIFORM | wgpu::BufferUsages::COPY_DST,
        mapped_at_creation: false,
    });
    queue.write_buffer(&buf, 0, &bytes);
    buf
}
```

`UniformBuffer::write` serialises the struct into a `Vec<u8>` respecting std140 layout, so `glam::Vec3` gets the 16-byte alignment it needs automatically. [Source](https://docs.rs/encase)

### bytemuck: Zero-Copy Casting

[bytemuck](https://crates.io/crates/bytemuck) is complementary: it enables zero-copy casts from typed slices to `&[u8]` for types that are already correctly laid out in Rust memory (`#[repr(C)]` structs where Rust layout matches GPU layout — common for simple vertex data):

```rust
use bytemuck::{Pod, Zeroable};

#[repr(C)]
#[derive(Copy, Clone, Pod, Zeroable)]
struct Vertex {
    position: [f32; 3],  // 12 bytes
    color:    [f32; 3],  // 12 bytes (no alignment surprise — both are vec3 of f32)
}

fn upload_vertices(queue: &wgpu::Queue, buf: &wgpu::Buffer, verts: &[Vertex]) {
    // bytemuck::cast_slice is zero-copy: no allocation, just pointer reinterpretation.
    queue.write_buffer(buf, 0, bytemuck::cast_slice(verts));
}
```

**Rule of thumb**: use `encase` for uniform/storage buffers containing types with non-trivial WGSL alignment (structs with `vec3`, `mat4`, mixed scalar+vector); use `bytemuck` for vertex buffers and index buffers where the Rust layout is already flat.

These two crates are not in conflict — a single frame may use both. A typical wgpu application uses `bytemuck::cast_slice` to upload vertex positions every frame (no padding, hot path) and `encase::UniformBuffer` to serialise a camera or lighting uniform once per frame (complex struct, correct padding guaranteed). The derive macros are also composable: a struct can derive both `bytemuck::Pod` and `encase::ShaderType`, though the latter will enforce padding that the former requires to be pre-applied in the Rust struct definition. Getting the padding wrong in either direction is a silent correctness bug on the GPU — the shader reads a value from the wrong byte offset and produces corrupted output with no diagnostic. Deriving `ShaderType` and then running naga's `validate` on the WGSL is the most reliable way to catch mismatches before they reach a GPU. [Source: encase](https://docs.rs/encase) | [Source: bytemuck](https://docs.rs/bytemuck)

---

## naga_oil: Shader Module Composition

[naga_oil](https://github.com/bevyengine/naga_oil) is a shader composition and preprocessing library built on top of naga. It adds `#import` / `#define_import_path` directives to WGSL (and GLSL), enabling modular shader code with scoped namespaces — the feature most conspicuously missing from plain WGSL. It is the engine behind Bevy's PBR shader library.

### The Composer API

A `Composer` instance holds the set of known shader modules. Modules declare their public name with `#define_import_path` and are added incrementally; their dependents are resolved at compose time:

```rust
use naga_oil::compose::{Composer, ComposableModuleDescriptor, NagaModuleDescriptor};

fn build_pbr_shader() -> naga::Module {
    let mut composer = Composer::default();

    // Register the PBR types module (no entry point — library only).
    composer.add_composable_module(ComposableModuleDescriptor {
        source:         include_str!("pbr_types.wgsl"),
        file_path:      "pbr_types.wgsl",
        language:       naga_oil::compose::ShaderLanguage::Wgsl,
        ..Default::default()
    }).unwrap();

    // Register mesh utility functions.
    composer.add_composable_module(ComposableModuleDescriptor {
        source:         include_str!("mesh_functions.wgsl"),
        file_path:      "mesh_functions.wgsl",
        language:       naga_oil::compose::ShaderLanguage::Wgsl,
        ..Default::default()
    }).unwrap();

    // Compose the final fragment shader that imports both modules.
    composer.make_naga_module(NagaModuleDescriptor {
        source:    include_str!("pbr_fragment.wgsl"),
        file_path: "pbr_fragment.wgsl",
        ..Default::default()
    }).unwrap()
}
```

### Import Syntax

WGSL modules use Rust-inspired import paths:

```wgsl
// pbr_types.wgsl — declares its public name:
#define_import_path bevy_pbr::pbr_types

struct StandardMaterial {
    base_color:        vec4<f32>,
    emissive:          vec4<f32>,
    perceptual_roughness: f32,
    metallic:          f32,
};
```

```wgsl
// pbr_fragment.wgsl — imports by path:
#import bevy_pbr::pbr_types::{StandardMaterial}
#import bevy_pbr::mesh_functions::{get_world_position}

@fragment
fn fragment(in: MeshVertexOutput) -> @location(0) vec4<f32> {
    let material: StandardMaterial = get_material();
    let world_pos = get_world_position(in.instance_index, in.vertex_index);
    return material.base_color;
}
```

### Virtual Function Override

naga_oil supports overriding `virtual` functions declared in library modules, enabling a plugin-like customisation pattern:

```wgsl
// Override a virtual lighting function from Bevy's PBR pipeline:
#import bevy_pbr::lighting as Lighting

override fn Lighting::point_light(world_position: vec3<f32>) -> vec3<f32> {
    let original = Lighting::point_light(world_position);
    // Quantise to cel-shading bands.
    return floor(original * 3.0) / 3.0;
}
```

All calls to `Lighting::point_light` throughout the composed shader graph are redirected to this override. Bevy uses this mechanism extensively in its render pipeline to allow user code to swap in custom lighting, shadow, and material functions without forking the engine shaders. [Source](https://github.com/bevyengine/naga_oil)

### Design Tradeoffs

naga_oil's composition model has performance implications to be aware of. Each call to `add_composable_module` builds naga IR for that module in isolation, which means compile times scale with the module count rather than the total shader size — a good property for incremental builds (modifying `mesh_functions.wgsl` only recompiles modules that import it). However, the `Composer` is an in-process object that holds state between frames, so hot-reloading a shader module during development means removing the old module, clearing its dependents, and re-adding them — a process the Bevy `hot_reload` feature gate handles automatically. The tree-shaking semantics (only imported items that are actually called end up in the final SPIR-V) mean naga_oil is also useful as a shader preprocessor for large codebases where monolithic shaders would otherwise grow unboundedly. Projects not using Bevy can use naga_oil standalone by adding `naga-oil` to `Cargo.toml` and calling `Composer::make_naga_module`, then feeding the result to naga's SPIR-V writer or directly to wgpu's `device.create_shader_module_from_naga`. This makes naga_oil a composable building block for any Rust graphics project that wants modular WGSL without coupling to Bevy's render graph. [Source](https://github.com/bevyengine/naga_oil)

---

## winit and raw-window-handle: Window and GPU Surface

Every real wgpu and ash application needs a window and an OS-presentable GPU surface. Two crates form the bridge: `winit` for cross-platform window creation and event dispatch, and `raw-window-handle` for the type-safe handle that wgpu, ash-window, and other GPU crates accept.

### winit 0.30 and the App Trait

`winit` 0.30 introduced an `ApplicationHandler` trait that replaces the older closure-based event loop:

```toml
[dependencies]
winit   = "0.30"
wgpu    = "22"
```

```rust
use winit::{
    application::ApplicationHandler,
    event::WindowEvent,
    event_loop::{ActiveEventLoop, EventLoop},
    window::{Window, WindowAttributes},
};

struct App {
    window: Option<Arc<Window>>,
    // wgpu state, surface, device, queue…
}

impl ApplicationHandler for App {
    fn resumed(&mut self, event_loop: &ActiveEventLoop) {
        // Window must be created here (not in new()) for Android/web compat
        let window = Arc::new(
            event_loop.create_window(WindowAttributes::default()
                .with_title("wgpu app")
                .with_inner_size(winit::dpi::LogicalSize::new(1280, 720)))
            .unwrap()
        );
        // Create wgpu surface from the window — see below
        self.window = Some(window);
    }

    fn window_event(&mut self, event_loop: &ActiveEventLoop,
                    _id: WindowId, event: WindowEvent) {
        match event {
            WindowEvent::CloseRequested => event_loop.exit(),
            WindowEvent::RedrawRequested => {
                // render frame here
                self.window.as_ref().unwrap().request_redraw();
            }
            WindowEvent::Resized(new_size) => {
                // rebuild swapchain / reconfigure surface
                let _ = new_size;
            }
            _ => {}
        }
    }
}

fn main() {
    let event_loop = EventLoop::new().unwrap();
    let mut app = App { window: None };
    event_loop.run_app(&mut app).unwrap();
}
```

**Platform selection.** On Linux, winit auto-detects the display server: Wayland when `WAYLAND_DISPLAY` is set, X11 otherwise. The `WINIT_UNIX_BACKEND=wayland` or `=x11` environment variable overrides detection. The `wayland` and `x11` Cargo features must be enabled (they are by default).

---

### raw-window-handle and wgpu Surface Creation

`raw-window-handle` 0.6 defines two traits — `HasWindowHandle` and `HasDisplayHandle` — that expose OS-level handles (Wayland `wl_surface` + `wl_display`, X11 `xcb_window_t`, etc.) in a type-safe enum. `wgpu::Instance::create_surface` accepts any type implementing both traits:

```rust
use wgpu::SurfaceTarget;

// Inside App::resumed(), after creating `window`:
let instance = wgpu::Instance::new(&wgpu::InstanceDescriptor {
    backends: wgpu::Backends::VULKAN,  // or PRIMARY for auto-select
    ..Default::default()
});

// wgpu accepts the Arc<Window> directly — it implements HasWindowHandle
let surface = instance.create_surface(window.clone()).unwrap();

// Request an adapter compatible with this surface
let adapter = instance.request_adapter(&wgpu::RequestAdapterOptions {
    power_preference:       wgpu::PowerPreference::HighPerformance,
    compatible_surface:     Some(&surface),
    force_fallback_adapter: false,
}).await.unwrap();

let (device, queue) = adapter.request_device(
    &wgpu::DeviceDescriptor::default(), None).await.unwrap();

// Configure the surface (replaces swapchain setup)
let caps   = surface.get_capabilities(&adapter);
let format = caps.formats[0]; // first is usually Bgra8UnormSrgb on Vulkan/Linux
surface.configure(&device, &wgpu::SurfaceConfiguration {
    usage:        wgpu::TextureUsages::RENDER_ATTACHMENT,
    format,
    width:        window.inner_size().width,
    height:       window.inner_size().height,
    present_mode: wgpu::PresentMode::Fifo,
    alpha_mode:   caps.alpha_modes[0],
    view_formats: vec![],
    desired_maximum_frame_latency: 2,
});
```

**Frame loop.** Inside `WindowEvent::RedrawRequested`:

```rust
let output  = surface.get_current_texture().unwrap();
let view    = output.texture.create_view(&Default::default());
let mut enc = device.create_command_encoder(&Default::default());
{
    let mut rpass = enc.begin_render_pass(&wgpu::RenderPassDescriptor {
        color_attachments: &[Some(wgpu::RenderPassColorAttachment {
            view: &view,
            resolve_target: None,
            ops: wgpu::Operations {
                load:  wgpu::LoadOp::Clear(wgpu::Color::BLACK),
                store: wgpu::StoreOp::Store,
            },
        })],
        ..Default::default()
    });
    rpass.set_pipeline(&render_pipeline);
    rpass.draw(0..3, 0..1);
}
queue.submit([enc.finish()]);
output.present();
```

---

### ash-window: Vulkan Surface from winit

For ash (raw Vulkan), `ash-window` converts a `raw-window-handle` handle into a `VkSurfaceKHR`:

```toml
[dependencies]
ash        = "0.38"
ash-window = "0.13"
raw-window-handle = "0.6"
```

```rust
use ash_window;
use raw_window_handle::{HasDisplayHandle, HasWindowHandle};

unsafe fn create_surface(
    entry:    &ash::Entry,
    instance: &ash::Instance,
    window:   &impl HasWindowHandle + HasDisplayHandle,
) -> vk::SurfaceKHR {
    ash_window::create_surface(
        entry,
        instance,
        window.display_handle().unwrap().as_raw(),
        window.window_handle().unwrap().as_raw(),
        None,
    ).unwrap()
}
```

`ash-window` calls the correct platform extension under the hood:
- Wayland → `vkCreateWaylandSurfaceKHR` (`VK_KHR_wayland_surface`)
- X11/XCB → `vkCreateXcbSurfaceKHR` (`VK_KHR_xcb_surface`)

The required instance extensions to enable are returned by `ash_window::enumerate_required_extensions(display_handle)`.

[Source: winit](https://github.com/rust-windowing/winit) | [ash-window](https://github.com/ash-rs/ash/tree/main/ash-window) | [raw-window-handle](https://github.com/rust-windowing/raw-window-handle)

---

## glam: Mathematics for Rust GPU

[glam](https://github.com/bitshifter/glam-rs) is the standard math library for Rust GPU development, used by Bevy, vello, wgpu tutorials, burn, and spirv-std (rust-gpu's GPU standard library). It provides SIMD-accelerated `Vec2`, `Vec3`, `Vec4`, `Mat4`, `Quat`, and related types with a clean, ergonomic API.

```toml
[dependencies]
glam = "0.29"   # SIMD on x86_64 (SSE2), aarch64 (NEON), WASM (simd128)
```

**Vulkan-correct by default.** Unlike GLM (which needs `GLM_FORCE_DEPTH_ZERO_TO_ONE`), glam's perspective functions already use the correct conventions for Vulkan and wgpu:

```rust
use glam::{Mat4, Vec3, f32::*};

let aspect = width as f32 / height as f32;

// perspective_rh: right-handed, depth in [0, 1] — correct for Vulkan/wgpu
let proj = Mat4::perspective_rh(
    45_f32.to_radians(), // vertical FoV
    aspect,
    0.1,                 // near
    1000.0,              // far
);

// look_at_rh: right-handed camera — correct for Vulkan NDC
let view = Mat4::look_at_rh(
    Vec3::new(3.0, 3.0, 3.0), // eye
    Vec3::ZERO,                // target
    Vec3::Y,                   // up
);

let model = Mat4::from_rotation_y(elapsed_s * 1.0_f32.to_radians())
          * Mat4::from_scale(Vec3::splat(1.0));

let mvp = proj * view * model;
```

**bytemuck integration.** glam types implement `bytemuck::Pod` and `bytemuck::Zeroable`, enabling zero-copy uploads to GPU buffers:

```rust
use bytemuck::cast_slice;

// Mat4 is [f32; 16] under the hood — cast_slice gives &[u8]
queue.write_buffer(&uniform_buf, 0, cast_slice(&[mvp]));
```

**Quaternion animation.** Slerp-based smooth rotation, used in glTF animation channel evaluation:

```rust
use glam::Quat;

let q_start = Quat::from_rotation_y(0.0);
let q_end   = Quat::from_rotation_y(std::f32::consts::PI);
let q       = Quat::slerp(q_start, q_end, t); // t in [0, 1]
let rot_mat = Mat4::from_quat(q);
```

**In rust-gpu shaders.** `spirv-std` re-exports glam as `spirv_std::glam`, making the same `Vec4`, `Mat4` types available in GPU shader code:

```rust
// Vertex shader with rust-gpu
#![no_std]
use spirv_std::glam::{vec4, Mat4, Vec4};
use spirv_std::spirv;

#[spirv(vertex)]
pub fn vs_main(
    position: Vec4,
    #[spirv(push_constant)] mvp: &Mat4,
    #[spirv(position)] out: &mut Vec4,
) {
    *out = *mvp * position;
}
```

`glam` is not WGSL-compatible directly — WGSL uses its own `vec4<f32>` / `mat4x4<f32>` types — but it serves as the CPU-side math layer that produces the `[f32; 16]` data uploaded to uniform buffers consumed by WGSL shaders.

[Source: glam-rs](https://github.com/bitshifter/glam-rs)

---

## egui: Immediate-Mode GUI on wgpu

[egui](https://github.com/emilk/egui) is the dominant Rust immediate-mode GUI library: stateless, re-described each frame from a closure, no retained widget tree. On wgpu it runs via the `egui-wgpu` integration crate; `egui-winit` handles input translation from winit events.

```toml
[dependencies]
egui        = "0.29"
egui-wgpu   = "0.29"
egui-winit  = "0.29"
```

**Integration pattern.** Three objects coordinate the integration:

| Object | Crate | Role |
|--------|-------|------|
| `egui::Context` | `egui` | Retains GUI state across frames (fonts, animations, memory) |
| `egui_winit::State` | `egui-winit` | Translates winit events → egui input; tracks screen scale |
| `egui_wgpu::Renderer` | `egui-wgpu` | Holds wgpu buffers/textures/pipeline; renders egui paint jobs |

```rust
use egui_wgpu::Renderer;
use egui_winit::State;

struct EguiState {
    ctx:      egui::Context,
    winit:    State,
    renderer: Renderer,
}

impl EguiState {
    fn new(device: &wgpu::Device, surface_format: wgpu::TextureFormat,
           window: &Window) -> Self {
        let ctx      = egui::Context::default();
        let winit    = State::new(ctx.clone(), ctx.viewport_id(),
                                  window, None, None, None);
        let renderer = Renderer::new(device, surface_format, None, 1, false);
        Self { ctx, winit, renderer }
    }

    // Call once per frame before rendering
    fn run(&mut self, window: &Window, queue: &wgpu::Queue,
           device: &wgpu::Device, encoder: &mut wgpu::CommandEncoder,
           view: &wgpu::TextureView,
           ui_fn: impl FnOnce(&egui::Context))
    {
        // 1. Collect input (call egui_winit::State::on_window_event for each event)
        let raw_input = self.winit.take_egui_input(window);

        // 2. Run the UI description closure
        let full_output = self.ctx.run(raw_input, ui_fn);

        // 3. Handle platform output (copy text, cursor changes)
        self.winit.handle_platform_output(window, full_output.platform_output);

        // 4. Tessellate and upload
        let tris = self.ctx.tessellate(full_output.shapes,
                                        full_output.pixels_per_point);
        for (id, delta) in &full_output.textures_delta.set {
            self.renderer.update_texture(device, queue, *id, delta);
        }
        let screen_desc = egui_wgpu::ScreenDescriptor {
            size_in_pixels:  [window.inner_size().width,
                               window.inner_size().height],
            pixels_per_point: full_output.pixels_per_point,
        };
        self.renderer.update_buffers(device, queue, encoder, &tris, &screen_desc);

        // 5. Render into the surface texture
        let mut rpass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
            color_attachments: &[Some(wgpu::RenderPassColorAttachment {
                view, resolve_target: None,
                ops: wgpu::Operations {
                    load:  wgpu::LoadOp::Load,  // draw on top of scene
                    store: wgpu::StoreOp::Store,
                },
            })],
            ..Default::default()
        });
        self.renderer.render(&mut rpass, &tris, &screen_desc);

        // 6. Free textures no longer referenced
        for id in &full_output.textures_delta.free {
            self.renderer.free_texture(id);
        }
    }
}
```

**Usage** — the `ui_fn` closure describes the UI declaratively:

```rust
egui_state.run(&window, &queue, &device, &mut encoder, &view, |ctx| {
    egui::Window::new("Debug").show(ctx, |ui| {
        ui.label(format!("FPS: {fps:.1}"));
        ui.add(egui::Slider::new(&mut roughness, 0.0..=1.0).text("Roughness"));
        ui.add(egui::Slider::new(&mut metallic,  0.0..=1.0).text("Metallic"));
        if ui.button("Reload shader").clicked() {
            // trigger hot-reload (see §24)
        }
    });
    egui::plot::Plot::new("timing").show(ctx, |plot| {
        plot.line(egui::plot::Line::new(gpu_times.clone()));
    });
});
```

egui is the standard choice for in-application debug panels, material editors, and profiler overlays in Rust wgpu applications. [Source: egui](https://github.com/emilk/egui)

---

## candle: Rust ML Inference

[candle](https://github.com/huggingface/candle) is Hugging Face's Rust ML framework, optimised for inference workloads. Unlike burn (which targets training + inference with pluggable wgpu/CUDA backends), candle is explicitly inference-focused, minimal-dependency, and has first-class Safetensors support for loading production model weights directly.

```toml
[dependencies]
candle-core         = "0.9"
candle-nn           = "0.9"
candle-transformers = "0.9"   # prebuilt LLM architectures

[features]
cuda   = ["candle-core/cuda"]   # CUDA via cudarc
metal  = ["candle-core/metal"]  # Metal for macOS/Apple Silicon
```

**Device selection and tensor operations:**

```rust
use candle_core::{Device, Tensor, DType};

// Selects CUDA:0 if compiled with --features cuda, else CPU
let device = Device::cuda_if_available(0)?;  // or Device::Cpu, Device::Metal(0)

let a = Tensor::randn(0f32, 1f32, (512, 512), &device)?;
let b = Tensor::randn(0f32, 1f32, (512, 512), &device)?;

// Matrix multiply — runs on CUDA/Metal/CPU transparently
let c = a.matmul(&b)?;

// Element-wise ops, reductions
let mean = c.mean_all()?;
let relu = c.relu()?;
```

**Loading a real LLM.** Candle ships prebuilt transformer implementations in `candle-transformers`. The pattern is load weights from Safetensors, build the model, run generation:

```rust
use candle_core::{Device, DType};
use candle_transformers::models::mistral::{Config, Model};
use candle_nn::VarBuilder;

let device = Device::cuda_if_available(0)?;

// Load weights from Safetensors shards (standard HF model format)
let weights = candle_core::safetensors::load_many(
    &["model-00001-of-00003.safetensors",
      "model-00002-of-00003.safetensors",
      "model-00003-of-00003.safetensors"],
    &device)?;
let vb = VarBuilder::from_tensors(weights, DType::BF16, &device);

let config = Config::config_7b_v0_1(false); // flash-attention=false for CPU compat
let mut model = Model::new(&config, vb)?;

// Tokenise (via tokenizers crate), run forward pass, sample
let input_ids = Tensor::new(&[1u32, 1234, 567, 89], &device)?.unsqueeze(0)?;
let logits = model.forward(&input_ids, 0)?;
```

**candle vs burn.** Both are Rust ML frameworks; the key differences:

| | candle | burn |
|--|--------|------|
| Primary focus | Inference | Training + inference |
| wgpu backend | No | Yes (`burn-wgpu`) |
| CUDA backend | Yes (cudarc) | Yes (`burn-cuda` / CubeCL) |
| Autograd | Partial (via `candle-core`) | Full (`burn` module system) |
| Model zoo | Large (HF ecosystem) | Growing |
| Dependency weight | Minimal | Moderate |
| Safetensors | First-class | Via conversion |

For production LLM inference on Linux (load a HF checkpoint, run generation), candle is the simpler path. For training custom models or using Vulkan/wgpu compute, burn is more appropriate.

[Source: candle](https://github.com/huggingface/candle)

---

## wgpu-profiler: GPU Timestamp Profiling

[wgpu-profiler](https://github.com/Wumpf/wgpu-profiler) wraps wgpu's timestamp query API into an ergonomic scope-based profiler. It requires `wgpu::Features::TIMESTAMP_QUERY` on the device.

```toml
[dependencies]
wgpu-profiler = "0.20"
```

**Setup.** Request the timestamp query feature at device creation:

```rust
let (device, queue) = adapter.request_device(&wgpu::DeviceDescriptor {
    required_features: wgpu::Features::TIMESTAMP_QUERY
                     | wgpu::Features::TIMESTAMP_QUERY_INSIDE_ENCODERS,
    ..Default::default()
}, None).await.unwrap();

use wgpu_profiler::{GpuProfiler, GpuProfilerSettings};
let mut profiler = GpuProfiler::new(GpuProfilerSettings::default()).unwrap();
```

**Per-frame profiling with RAII scopes:**

```rust
// Wraps the CommandEncoder; each scope inserts begin/end timestamp queries
let mut enc = device.create_command_encoder(&Default::default());

{
    let mut scope = profiler.scope("shadow pass", &mut enc, &device);
    // record shadow map draw calls into scope.encoder()…
    {
        let mut inner = scope.scope("depth clear", &device);
        // sub-scopes produce nested timing entries
    }
}

{
    let mut scope = profiler.scope("main pass", &mut enc, &device);
    // record G-buffer / forward pass…
}

// Must resolve timestamp queries into a readable buffer before submit
profiler.resolve_queries(&mut enc);
queue.submit([enc.finish()]);

// End frame; results appear 1-2 frames later (GPU latency)
profiler.end_frame().unwrap();

// Poll for completed results each frame
if let Some(results) = profiler.process_finished_frame(queue.get_timestamp_period()) {
    for scope in &results {
        println!("{}: {:.3} ms", scope.label,
                 (scope.time.end - scope.time.start) * 1000.0);
    }
}
```

**Tracy integration.** With the `tracy` Cargo feature, `wgpu-profiler` forwards GPU timing data to [Tracy](https://github.com/wolfpld/tracy) so GPU and CPU timelines appear in the same profiler view — the same workflow as the C++ `TracyVkCtx` integration, but from pure Rust.

`wgpu-profiler` is the standard GPU profiling layer for wgpu applications; Bevy uses it internally via `bevy_render::diagnostic::RenderDiagnosticsPlugin`.

---

## Shader Hot-Reloading

Shader hot-reloading — detecting that a `.wgsl` (or `.glsl`, `.slang`) file changed on disk and recompiling the pipeline without restarting the application — dramatically accelerates GPU shader development iteration. In wgpu, pipelines are Rust objects; hot-reloading means recreating them at runtime.

**The notify + channel pattern:**

```toml
[dependencies]
notify         = "6"    # cross-platform file system events
notify-debouncer-mini = "0.4"
```

```rust
use notify::{Watcher, RecommendedWatcher, RecursiveMode};
use notify_debouncer_mini::new_debouncer;
use std::sync::mpsc;
use std::time::Duration;

// Spawn a watcher thread; send reload events over a channel
fn watch_shaders(tx: mpsc::Sender<()>) {
    let (mut debouncer, mut rx) = new_debouncer(
        Duration::from_millis(200), None).unwrap();
    debouncer.watcher()
        .watch(Path::new("shaders/"), RecursiveMode::Recursive).unwrap();
    for result in rx.iter() {
        if result.is_ok() {
            let _ = tx.send(());  // notify main thread
        }
    }
}

// In the main render thread, poll the channel each frame:
fn try_reload(rx: &mpsc::Receiver<()>, device: &wgpu::Device,
              pipeline: &mut wgpu::RenderPipeline,
              layout: &wgpu::PipelineLayout,
              format: wgpu::TextureFormat)
{
    if rx.try_recv().is_ok() {
        let src = std::fs::read_to_string("shaders/main.wgsl").unwrap();
        let module = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label:  Some("reloaded"),
            source: wgpu::ShaderSource::Wgsl(src.into()),
        });
        // Rebuild the pipeline with the new module
        *pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
            layout:  Some(layout),
            vertex:  wgpu::VertexState { module: &module, entry_point: Some("vs_main"), .. Default::default() },
            fragment: Some(wgpu::FragmentState {
                module:  &module,
                entry_point: Some("fs_main"),
                targets: &[Some(format.into())],
            }),
            ..Default::default()
        });
        eprintln!("shader reloaded");
    }
}
```

**naga_oil hot-reloading.** When using `naga_oil`'s `Composer`, modules must be explicitly removed and re-added — the `Composer` holds per-module naga IR:

```rust
// On file change for "shaders/lighting.wgsl":
composer.remove_composable_module("bevy_pbr::lighting");
composer.add_composable_module(ComposableModuleDescriptor {
    source:          &std::fs::read_to_string("shaders/lighting.wgsl")?,
    file_path:       "shaders/lighting.wgsl",
    as_name:         Some("bevy_pbr::lighting".into()),
    ..Default::default()
})?;
// Recreate all downstream pipelines that import this module
```

Bevy's `hot_reload` feature gate (`BEVY_ASSET_PROCESSOR=1 cargo run --features bevy/file_watcher`) automates this for all registered shader assets.

**Thread safety note.** wgpu `Device` is `Send + Sync`, but pipeline creation must happen on a thread that has access to it. The common pattern is: watcher thread sends an event over a channel, main render thread polls the channel once per frame before recording commands, and recreates the pipeline if signalled.

---

## pixels: Simple wgpu Framebuffer

[pixels](https://github.com/parasyte/pixels) provides a simple CPU-writable pixel buffer that is rendered to the screen via wgpu. It is the right choice for software rasterizers, cellular automata, retro game emulators, and any application that wants direct pixel manipulation without managing wgpu pipelines manually.

```toml
[dependencies]
pixels = "0.14"
winit  = "0.30"
```

```rust
use pixels::{Pixels, SurfaceTexture};

// Create pixel buffer (width × height RGBA u8 pixels)
let surface_texture = SurfaceTexture::new(
    window_width, window_height, &window);
let mut pixels = Pixels::builder(320, 240, surface_texture)
    .build()?;

// In the render loop:
let frame = pixels.frame_mut(); // &mut [u8], length = 320 * 240 * 4
for (i, pixel) in frame.chunks_exact_mut(4).enumerate() {
    let x = (i % 320) as u8;
    let y = (i / 320) as u8;
    pixel.copy_from_slice(&[x, y, 128, 255]); // RGBA gradient
}
pixels.render()?;  // uploads frame buffer to GPU, presents to screen
```

**Resize handling:**

```rust
WindowEvent::Resized(new_size) => {
    pixels.resize_surface(new_size.width, new_size.height)?;
}
```

`pixels` uses wgpu internally with a simple scaling shader, so the pixel buffer (e.g. 320×240) can be presented at any window resolution with nearest-neighbour or bilinear scaling. Custom post-processing passes can be appended to the wgpu render pipeline via `pixels.render_with(|encoder, render_target, context| { ... })`.

---

## rspirv: SPIR-V IR in Rust

[rspirv](https://github.com/gfx-rs/rspirv) is a Rust library for reading, building, and manipulating SPIR-V binary modules. It is used internally by rust-gpu's `rustc_codegen_spirv` and is useful for any tool that needs to inspect or construct SPIR-V programmatically — custom code generators, instrumentation passes, or analysis tools.

```toml
[dependencies]
rspirv = "0.12"
spirv  = "0.3"  # SPIR-V opcode enum
```

**Reading and inspecting SPIR-V:**

```rust
use rspirv::dr::load_bytes;

let spirv_bytes: Vec<u8> = std::fs::read("shader.spv")?;
let module = load_bytes(&spirv_bytes).unwrap();

// Walk all instructions in the module
for inst in module.all_inst_iter() {
    println!("{:?}", inst.class.opname);
}

// List all OpEntryPoint declarations
for entry in &module.entry_points {
    println!("Entry: {:?}", entry.operands[1]); // name operand
}
```

**Building SPIR-V programmatically:**

```rust
use rspirv::dr::Builder;
use rspirv::binary::Assemble;
use spirv::{AddressingModel, ExecutionModel, MemoryModel};

let mut b = Builder::new();
b.set_version(1, 5);
b.capability(spirv::Capability::Shader);
b.memory_model(AddressingModel::Logical, MemoryModel::GLSL450);

let void     = b.type_void();
let voidf    = b.type_function(void, vec![]);
let entry_fn = b.begin_function(void, None,
    spirv::FunctionControl::NONE, voidf).unwrap();
b.begin_block(None).unwrap();
b.ret().unwrap();
b.end_function().unwrap();
b.entry_point(ExecutionModel::Vertex, entry_fn, "main", vec![]);

// Assemble to Vec<u32> — pass to vkCreateShaderModule or wgpu
let spirv_words: Vec<u32> = b.module().assemble();
```

**Relationship to naga.** naga (§5) is the higher-level path: it ingests WGSL/GLSL/SPIR-V and emits SPIR-V with full type-system understanding, validation, and optimisation passes. `rspirv` operates one level lower — it manipulates the raw SPIR-V instruction stream. Use naga when you want to translate or validate shaders; use rspirv when you want to instrument, analyse, or generate SPIR-V from a custom IR.

[Source: rspirv](https://github.com/gfx-rs/rspirv)

---

## Rust Ownership vs the GPU Memory Model

This is the central conceptual question every Rust developer faces when moving to GPU programming: Rust's borrow checker guarantees that at any point in time, a value is either exclusively mutably borrowed (`&mut T`) or shared-read-only (`&T`) — never both. A GPU workgroup has 64 (or 256, or 1024) threads simultaneously writing to the same block of shared memory. These two models do not reconcile cleanly. Understanding where Rust's guarantees end and where GPU discipline begins is essential for writing correct GPU code in Rust.

### The Fundamental Incompatibility

Rust's memory model is designed around the **aliasing XOR mutability** invariant: the compiler can assume that if two references (`&T`, `&mut T`) to the same location exist, they follow strict non-overlapping access rules. This enables LLVM to generate code without defensive memory fences or volatile loads.

A GPU's execution model is inherently the opposite. Consider a WGSL reduce operation:

```wgsl
var<workgroup> tile: array<f32, 64>;  // 64 threads share this

@compute @workgroup_size(64)
fn reduce(@builtin(local_invocation_index) lid: u32) {
    tile[lid] = load_element(lid);     // all 64 threads write simultaneously
    workgroupBarrier();                // <- programmer-inserted synchronisation
    
    var stride = 32u;
    while stride > 0u {
        if lid < stride {
            tile[lid] += tile[lid + stride]; // reads and writes overlap across threads
        }
        workgroupBarrier();
        stride >>= 1u;
    }
}
```

In Rust terms, `tile` would be `Arc<UnsafeCell<[f32; 64]>>` and every access would require `unsafe`. But on the GPU this is the normal, correct, high-performance pattern — not an exception. The `workgroupBarrier()` plays the role of a mutex, but it is not a mutex: it does not prevent concurrent access, it sequences it in time across threads.

---

### The Four Boundaries

The incompatibility manifests differently at four distinct boundaries in the GPU stack:

#### 1. CPU ↔ GPU Resource Handoff

When you call `queue.submit([cmd_buf])`, the encoded GPU work takes "logical ownership" of every buffer, texture, and pipeline referenced inside it. Rust's borrow checker cannot track this — a `wgpu::Buffer` is an `Arc`-counted opaque handle, not a value the compiler can reason about.

**wgpu enforces this at runtime.** Trying to map a buffer while it is in-flight on the GPU produces an error, not a compile error:

```rust
// Schedule GPU work that reads from `buf`
let cmd = device.create_command_encoder(&Default::default());
// ... record commands using buf ...
queue.submit([cmd.finish()]);

// Attempting to read buf immediately is a runtime error (not compile error):
// "Buffer is not mappable" or "Buffer is in use by the GPU"
let slice = buf.slice(..);
slice.map_async(wgpu::MapMode::Read, |r| assert!(r.is_ok()));
device.poll(wgpu::Maintain::Wait);  // must wait for GPU to finish first
let data: &[u8] = slice.get_mapped_range().deref();
```

There is no way to express "this buffer is currently owned by the submitted GPU work" in Rust's type system without affine types (linear types) — something Rust does not have. wgpu, ash, and vulkano all handle this through runtime tracking, not compile-time proof.

**ash** does not even track it: if you map a `VkBuffer` while a command buffer that reads it is executing, you get a GPU fault or undefined behaviour. The contract is entirely the programmer's responsibility.

**Vulkano's task graph** goes furthest toward compile-time enforcement: declared resource accesses in the task graph builder determine what barriers are inserted before execution, and you cannot record a draw that writes to a resource that is simultaneously declared as input-only. But this is still expressed through runtime-checked API calls, not Rust type-level proofs.

#### 2. Shader Code Written as Text (WGSL, GLSL, HLSL)

For shaders compiled by naga or glslc from text strings embedded in your Rust source, the borrow checker has zero jurisdiction over the GPU code. The shader is a string; Rust sees only:

```rust
device.create_shader_module(wgpu::ShaderModuleDescriptor {
    source: wgpu::ShaderSource::Wgsl(include_str!("shader.wgsl").into()),
    ..Default::default()
})
```

Everything inside `shader.wgsl` — workgroup shared memory, atomic operations, barrier synchronisation — is invisible to the borrow checker. Naga validates types and binding compatibility but does not enforce aliasing rules. Safety is entirely the shader author's discipline: insert `workgroupBarrier()` before reading values written by other threads, use `atomic<T>` for fine-grained concurrent mutation.

**WGSL's type system does enforce one rule**: you cannot use a plain `f32` storage buffer for concurrent writes — you must declare it as `atomic<u32>` or `atomic<i32>`, and atomic types only support the `atomicAdd`, `atomicMin`, `atomicMax`, etc. operations. This is analogous to Rust's `Atomic*` types, but it is enforced by WGSL's validator, not by Rust's borrow checker.

```wgsl
// Wrong — concurrent writes to plain f32 are UB:
@group(0) @binding(0) var<storage, read_write> counter: f32;  // naga rejects atomicAdd on this

// Correct — atomic<u32> wrapper enables safe concurrent mutation:
@group(0) @binding(0) var<storage, read_write> counter: atomic<u32>;
// shader code:
atomicAdd(&counter, 1u);  // safe: hardware atomic
```

#### 3. Rust Shaders Compiled via rust-gpu (Rust → SPIR-V)

This is the most interesting boundary: rust-gpu compiles actual Rust code to SPIR-V, so the borrow checker does run on the shader source. But GPU execution semantics require patterns that violate Rust's safety rules, so they are marked `unsafe`.

**Workgroup shared memory** cannot be expressed as a safe Rust abstraction. In rust-gpu it uses `unsafe static mut` — the same opt-out used for global mutable state on the CPU:

```rust
// rust-gpu shader — workgroup shared memory
#![no_std]
use spirv_std::spirv;
use spirv_std::arch;

// SAFETY: GPU workgroup barrier synchronises all accesses
static mut TILE: [f32; 64] = [0.0; 64];

#[spirv(compute(threads(64)))]
pub unsafe fn reduce(
    #[spirv(local_invocation_index)] lid: u32,
    #[spirv(storage_buffer, descriptor_set = 0, binding = 0)] input: &[f32],
    #[spirv(storage_buffer, descriptor_set = 0, binding = 1)] output: &mut [f32],
) {
    // unsafe: concurrent write — all threads write simultaneously
    TILE[lid as usize] = input[lid as usize];
    arch::workgroup_memory_barrier();   // unsafe: GPU-specific side effect

    let mut stride = 32u32;
    while stride > 0 {
        if lid < stride {
            // unsafe: concurrent read-modify-write across threads
            TILE[lid as usize] += TILE[(lid + stride) as usize];
        }
        arch::workgroup_memory_barrier();
        stride >>= 1;
    }

    if lid == 0 {
        output[0] = TILE[0];
    }
}
```

Every GPU-specific operation that violates Rust's memory model is `unsafe`:
- Entry point functions are `pub unsafe fn` — not because they're inherently dangerous to call (the GPU calls them, not Rust code), but because GPU calling conventions and built-in register semantics (`gl_LocalInvocationIndex`, `gl_GlobalInvocationID`) cannot be expressed as safe Rust function arguments.
- `spirv_std::arch::workgroup_memory_barrier()` is `unsafe` — it has sequenced side effects across hardware threads, invisible to Rust's aliasing model.
- `static mut` shared memory is `unsafe` — it is shared mutable state without a lock, relying on the programmer to insert barriers correctly.

The borrow checker *does* catch genuine CPU-level bugs: if you try to borrow `TILE` both mutably and immutably in the same expression, rustc rejects it (though on the GPU, separate threads would have separate register windows, so the borrow checker's diagnosis would be wrong). This is an area where rust-gpu's CPU↔GPU semantic mismatch is most visible: the borrow checker runs on a single-threaded sequential model; the shader runs on thousands of threads simultaneously.

#### 4. GPU-Side Buffer Aliasing and Descriptor Indexing

Vulkan allows two `VkBuffer` handles to point to overlapping ranges of the same `VkDeviceMemory` — memory aliasing for transient resources (see Ch. 82, VMA). From Rust's perspective, two `&[u8]` pointing to overlapping GPU memory would be immediate undefined behaviour. wgpu's safe API prevents you from creating aliasing buffer mappings on the CPU side; it does not prevent the GPU from accessing aliased regions via descriptors, because that aliasing is invisible to the host-side type system.

This is why `unsafe` is the boundary in `wgpu-hal`: the HAL's `unsafe fn create_buffer` and the descriptor-binding operations below the safe `wgpu::Device` API can legally create aliasing resource views that the borrow checker cannot reason about.

---

### How Each Library Positions on This Spectrum

| Library | Host-side GPU lifetime | Shader-side memory safety | GPU-specific ops |
|---------|----------------------|--------------------------|-----------------|
| **ash** | `unsafe`, manual | Text (GLSL/HLSL/WGSL) — none | Programmer's responsibility |
| **wgpu** | Safe, runtime-validated | WGSL type-safe, no aliasing rules | Programmer's discipline |
| **wgpu-hal** | `unsafe` trait | WGSL/SPIR-V — external | Programmer's responsibility |
| **vulkano** | Type-states + task graph | GLSL — compile-time stage errors only | Runtime-checked |
| **rust-gpu** | N/A — GPU code | Borrow checker runs; GPU ops are `unsafe` | `unsafe` + `static mut` |
| **cudarc** | Safe `CudaSlice<T>` | PTX binary — none | Programmer's responsibility |
| **cuTile-rs** | Rust kernel + borrow checker | Rust borrow checker + cooperative `unsafe` | `unsafe` Tile ops |

**Vulkano is the most aggressive** about encoding GPU invariants in Rust's type system. Its phantom-type command buffer builder (`OutsideRenderPass` / `InsideRenderPass`) makes it a compile error to issue a draw call outside a render pass. Its task graph encodes declared resource accesses so barriers can be inserted automatically. But it stops at render-pass bracketing — it cannot encode image layout transitions, per-subresource memory dependency chains, or concurrent shader access patterns in the type system, because those would require linear or dependent types.

**rust-gpu's `unsafe` is honest** about the fundamental mismatch: it runs the borrow checker on shader Rust code, catching real bugs (type errors, out-of-bounds on statically-sized arrays), while explicitly acknowledging — via `unsafe` — the patterns where CPU and GPU semantics diverge.

---

### Why True Compile-Time GPU Memory Safety Is a Research Problem

Fully encoding the GPU memory model in Rust's type system would require:

1. **Affine / linear types** — to express "this buffer is consumed by GPU submit, unavailable CPU-side until the fence signals". Rust does not have linear types; `Arc` is the pragmatic substitute.

2. **Dependent types** — to express "a `workgroupBarrier()` here ensures that all writes by threads in group `g` to addresses in range `[base, base+stride)` before this point are visible to all reads after this point." Rust's type system cannot express this; it would require a verification framework like Dafny, Lean, or Coq applied to the shader IR.

3. **Session types or effect systems** — to track what GPU pipeline stage a command is executing in, ensuring barriers are inserted between dependent operations. This is an active area of PL research.

The pragmatic consensus in the Rust GPU ecosystem is:
- **Host side**: safe wrappers (`wgpu::Device`, `wgpu::Buffer`) backed by runtime validation are sufficient for the CPU-GPU handoff. The cost of a runtime check on `map_async` is negligible compared to GPU work.
- **Shader side**: rely on the shader language's type system (WGSL's `atomic<T>`, stage-specific type rules) plus programmer discipline for barrier placement. Naga's validator catches type errors; correct barrier ordering remains the programmer's responsibility.
- **Rust shaders (rust-gpu)**: use `unsafe` to explicitly delineate the boundary. The borrow checker's coverage of GPU code is useful but partial; `unsafe` is the honest acknowledgment of what it cannot verify.

The long-term roadmap section (§29) mentions the open goal of "encoding Vulkan synchronisation and resource lifetimes as Rust types without hiding API concepts" — this remains a research-level aspiration rather than a solved problem.

---

### Rust's Official Strategic Direction for GPU

The Rust Project is pursuing GPU support through concrete, officially tracked initiatives — but the strategy is deliberately pragmatic rather than type-theoretic. Understanding what is and is not official clarifies where the ecosystem is heading.

#### Governance: The GPU Working Group Proposal

Before examining technical initiatives, there is a governance signal worth noting. In February 2025, Christian Legnitto (maintainer of rust-gpu and rust-cuda) opened [rust-lang/leadership-council#155](https://github.com/rust-lang/leadership-council/issues/155) proposing an **official Rust GPU Working Group**. The motivation: GPU and AI companies need an obvious entry point into Rust language governance; an official working group can publish on the Rust blog and convene cross-company input.

The proposal covers both AI/compute and graphics GPU use cases. It remains unresolved as of mid-2026, in part because the Rust project has been phasing out classic domain working groups. If chartered, this would be the first official Rust governance body with a GPU mandate. For now, GPU work proceeds through the Project Goals mechanism and individual contributor tracks — without a chartered team.

#### Official GPU Targets

Two GPU targets are part of the Rust standard target tier policy:

- **`nvptx64-nvidia-cuda`** — **Tier 2** (officially supported, tested in CI). As of Rust 1.97 (released July 9, 2026), the baseline was raised to SM 7.0+ / PTX 7.0+, dropping pre-Turing hardware (Maxwell, Pascal) and requiring CUDA 11+ drivers. The Rust blog frames this as maintenance curation, not commitment expansion: "Maintaining support for these architectures would require substantial effort. These removals let us focus development efforts on improving correctness and performance for currently supported hardware." [[Source: blog.rust-lang.org/2026/05/01/nvptx-baseline-update](https://blog.rust-lang.org/2026/05/01/nvptx-baseline-update/)]

- **`amdgcn-amd-amdhsa`** — **Tier 3** (builds in CI but not guaranteed to work; community-maintained by `@Flakebi`). This covers AMD ROCm / HIP kernel compilation.

Neither target includes the Rust standard library in any meaningful form — `no_std` is the only viable option. There is no official SPIR-V target; the Khronos SPIR-V path is entirely community-driven via the rust-gpu project (see §§4–6).

#### `extern "gpu-kernel"` ABI Unification

[Tracking issue #135467](https://github.com/rust-lang/rust/issues/135467) (opened January 2025) proposes a unified `extern "gpu-kernel"` calling convention that would replace the separate `extern "ptx-kernel"` (NVIDIA) and `extern "amdgpu-kernel"` (AMD) ABIs currently required to write GPU entry points. The motivation is cross-vendor portability: a single attribute declares "this function is a GPU kernel entry point" without binding to a specific vendor's ABI name. The feature is partially implemented behind a nightly gate as of mid-2026; it is not yet stable.

```rust
// Proposed syntax (nightly, feature-gated)
#[no_mangle]
pub extern "gpu-kernel" fn add_vectors(
    a: *const f32,
    b: *const f32,
    out: *mut f32,
    n: u32,
) {
    // kernel body — targets both nvptx64 and amdgcn
}
```

This is a language-level change, not a library. It does not extend the borrow checker's reach into GPU shared memory; it only standardises the function ABI for the host-to-device call boundary.

#### `std::offload` and `std::autodiff` — The Official High-Level Strategy

The closest thing to an official Rust GPU memory model strategy is the **`std::offload` project goal**, part of the 2025H1 and 2025H2 Rust Project Goal slate, sponsored by Manuel Drehwald (Lawrence Livermore National Laboratory / MIT):

- **`#[offload]`** — a macro that marks a Rust function for GPU execution. The compiler (via LLVM's GPU offloading infrastructure, which supports NVIDIA, AMD, and Intel) generates a GPU kernel and the host-side dispatch code. Ownership is used to determine which data must be transferred to the device before the call and retrieved afterward.

- **`#[autodiff]`** — automatic differentiation at the Rust IR level, enabling ML training workloads without a Python layer. This was the first piece shipped (nightly unstable as of late 2025).

- **`#[batching]`** — vectorised batch execution of the offloaded function.

[[Source: Rust Project Goals 2025H2 — "GPU offloading"](https://rust-lang.github.io/rust-project-goals/2025h2/offload.html)]

The critical design choice in `std::offload` is **not** to run the borrow checker inside the GPU kernel. The kernel body is compiled through LLVM's GPU backend (same path as Clang's GPU offloading); Rust ownership governs the host↔device data movement boundary, but the GPU-side memory model is LLVM's. This is an explicit architectural decision: solve the problem LLVM can already solve, rather than extending Rust's type system into territory that requires linear or dependent types.

```rust
// Hypothetical std::offload usage (nightly, in-progress as of 2026)
#[offload(target = "nvptx64")]
fn vector_add(a: &[f32], b: &[f32], out: &mut [f32]) {
    // Rust ownership tracks that `a`, `b` → device (read-only)
    // and `out` → device (write) + returned to host after kernel
    // The borrow checker enforces this at the *call site*, not inside
    // the kernel body
    for i in 0..a.len() {
        out[i] = a[i] + b[i];
    }
}
```

The 2026 project goal, **"High-Level ML Optimizations"** (lang champion `@TC`, compiler champion Oli Scherer), extends this with an MLIR backend — enabling whole-model ML training (matmul, convolution, attention) through Rust's ownership model while delegating the numerical backend to MLIR's GPU dialects. The targets include NVIDIA, AMD, Intel, TPU, Trainium, and Qualcomm LPU.

#### What Is Not Official Rust

Several GPU Rust projects are influential but not part of the official Rust Project:

| Project | Origin | Status |
|---------|--------|--------|
| **rust-gpu** (`rustc_codegen_spirv`) | Embark Studios → community org (Rust-GPU) | Transition announced Aug 12, 2024; EmbarkStudios fork archived Oct 31, 2025; community org now actively maintained with compute/AI focus [[blog](https://rust-gpu.github.io/blog/transition-announcement/)] |
| **cuTile-rs** | NVlabs (NVIDIA Research) | NVlabs research project; formal paper "Fearless Concurrency on the GPU" arXiv:2606.15991 (June 2026); tile-level ownership model, not a Rust compiler extension |
| **cuda-oxide** | NVlabs (NVIDIA Research) | Compiles Rust to PTX via Stable MIR / `rustc_public`; separate from cuTile-rs |
| **VectorWare Rust-std-on-GPU** | VectorWare startup | Experimental; uses CUDA hostcall to run Rust std on GPU; considering upstreaming |
| **WGSL** | W3C WebGPU WG | Rust-influenced syntax, separate language; W3C Candidate Recommendation March 2026; **not affiliated with the Rust Project** |
| **wgpu** | gfx-rs / Mozilla lineage | Cross-platform GPU abstraction; **not affiliated with Google** (Dawn/Tint are Google's C++ WebGPU implementation; wgpu is an independent community project) |

No RFC for GPU-specific linear types, affine types, or dependent types exists as of mid-2026. The Unsafe Code Guidelines (UCG) working group has no active GPU memory model work — Stacked Borrows and Tree Borrows are CPU-only aliasing models; the GPU/SPIR-V memory model is a separate specification domain. SPIR-V is not on the official target tier roadmap.

The **Rust effects system initiative** (formerly "keyword generics"; see [rust-lang/effects-initiative](https://github.com/rust-lang/rust/issues/102090)) covers `async`, `const`, `try`, `unsafe`, and generators as tracked effects. GPU execution is not proposed as an effect and is not mentioned anywhere in the initiative. Any framing of "GPU offload as a Rust effect" is speculative inference, not an upstream position.

#### The Strategic Picture

The Rust Project's GPU strategy can be summarised as:

1. **Targets**: Tier 2 (nvptx64) and Tier 3 (amdgcn) targets give Rust compilation to GPU ISAs without solving the memory safety problem.

2. **ABI**: `extern "gpu-kernel"` unifies the entry-point convention cross-vendor without extending the type system.

3. **High-level offload**: `std::offload` uses LLVM's existing GPU infrastructure and Rust ownership to manage the boundary. This avoids requiring new type theory while delivering practical GPU integration.

4. **ML optimisations**: MLIR backend for 2026 enables high-performance training in Rust without a Python runtime.

5. **Governance**: GPU Working Group proposal (leadership-council#155) signals intent to establish an official forum, though it remains unchartered.

The strategy is deliberately *not* to solve the fundamental incompatibility between Rust's aliasing XOR mutability and GPU shared memory by extending the type system. That remains a research problem. The official approach accepts runtime validation at the host boundary, LLVM/MLIR for the GPU numerical path, and `unsafe` for low-level kernel code — the same pragmatic layering that the rest of the Rust unsafe ecosystem uses.

**Industry alignment**: NVIDIA is the only major GPU vendor with a formal, published position on Rust for GPU kernels — the cuTile-rs NVlabs project and the "Fearless Concurrency on the GPU" paper (arXiv:2606.15991, 2026). Google's GPU implementation stack (Dawn, Tint) is C++; Rust appears in the WebGPU ecosystem only through the independent wgpu community project, not Google. AMD and Intel have compatible GPU backends (SPIR-V via Vulkan) but no stated Rust GPU strategy. The Rust-GPU community project — now maintained at the [Rust-GPU org](https://github.com/Rust-GPU/rust-gpu) after Embark's 2024 transition — represents the SPIR-V path independently of all four vendors.

---

## Contributing to wgpu-core, wgpu-hal, and naga

The wgpu ecosystem lives in a single monorepo at [github.com/gfx-rs/wgpu](https://github.com/gfx-rs/wgpu). The four crates most relevant to Linux Vulkan development are:

- **`wgpu`** — the public API surface (`Device`, `Queue`, `Buffer`, `Texture`, `RenderPipeline`, etc.)
- **`wgpu-core`** — the implementation: resource tracking, command encoding, WebGPU validation, and the device lifecycle
- **`wgpu-hal`** — the hardware abstraction layer: the `hal::Api` trait and its Vulkan (`src/vulkan/`), GLES (`src/gles/`), and Metal backends
- **`naga`** — the shader translation library: WGSL/SPIR-V/GLSL front-ends, the typed-SSA IR, and back-end writers for SPIR-V, GLSL, MSL, HLSL, and WGSL

Naga merged into the wgpu monorepo in 2023 (previously at `gfx-rs/naga`) and is developed in the `naga/` subdirectory but still published as an independent crate on crates.io. [Source: naga consolidation RFC](https://github.com/gfx-rs/wgpu/issues/4231)

### Repository Setup

```bash
git clone https://github.com/gfx-rs/wgpu
cd wgpu
cargo build -p wgpu-core
cargo build -p wgpu-hal
cargo build -p naga
# Run tests against the Vulkan backend on Linux
WGPU_BACKEND=vulkan cargo test -p wgpu
```

Key environment variables for Linux development:

| Variable | Effect |
|---|---|
| `WGPU_BACKEND=vulkan` | Force the Vulkan backend (disables auto-selection) |
| `WGPU_POWER_PREF=high` | Prefer discrete GPU at adapter enumeration |
| `RUST_LOG=wgpu_core=debug,wgpu_hal=trace` | Verbose per-crate logging |
| `WGPU_VALIDATION=1` | Enable wgpu-core's internal validation layer |

The repository CI runs on Linux using `lavapipe` (the llvmpipe Vulkan software renderer) so hardware GPU is not required for basic test runs. For driver-specific testing, the full Mesa stack (ANV, RADV, NVK) is used in downstream CI at Firefox and Bevy.

### wgpu-core: Resource Tracking and Validation

`wgpu-core` implements the WebGPU specification layer above the HAL. Its central type is `Device<A>`, where `A: HalApi` is a backend type parameter:

```rust
// wgpu-core/src/device/resource.rs (simplified)
pub struct Device<A: HalApi> {
    pub(crate) raw: ManuallyDrop<A::Device>,  // the wgpu-hal device
    pub(crate) adapter: Arc<Adapter<A>>,
    pub(crate) limits: wgt::Limits,
    pub(crate) features: wgt::Features,
    // resource registries keyed by typed IDs
    pub(crate) buffers: Storage<Buffer<A>>,
    pub(crate) textures: Storage<Texture<A>>,
    pub(crate) render_pipelines: Storage<RenderPipeline<A>>,
}
```

`HalApi` is the trait bound that makes `wgpu-core` backend-agnostic. Every GPU call routes through `device.raw` (the `wgpu-hal` device) after wgpu-core validates the WebGPU invariants.

**To add a new wgpu feature** (e.g., exposing a new Vulkan extension as a `wgpu::Features` flag):

1. Add a `Features::MY_FEATURE` bitflag constant in `wgpu-types/src/features.rs`.
2. Map it to the Vulkan extension in `wgpu-hal/src/vulkan/adapter.rs` inside `PhysicalDeviceFeatures::from_extensions_and_requested_features`.
3. Implement the HAL operation in `wgpu-hal/src/vulkan/device.rs` or `command.rs`.
4. Guard the wgpu-core operation with `device.features.contains(Features::MY_FEATURE)`.
5. Add a `gpu_test` in `wgpu/tests/` that exercises the new path (see Testing section).

Validation logic lives in `wgpu-core/src/validation.rs` (for resource state and API invariants) and in `naga/src/valid/` (for shader IR). PRs that touch WebGPU API behaviour are expected to reference the relevant spec section in the PR description.

### wgpu-hal: Vulkan Backend Implementation

`wgpu-hal` abstracts GPU APIs behind traits in `wgpu-hal/src/lib.rs`. The root trait is:

```rust
// wgpu-hal/src/lib.rs (abridged)
pub trait Api: Clone + Sized + 'static {
    type Instance: Instance<A = Self>;
    type Adapter: Adapter<A = Self>;
    type Device: Device<A = Self>;
    type Queue: Queue<A = Self>;
    type CommandEncoder: CommandEncoder<A = Self>;
    type Buffer: Send + Sync + fmt::Debug;
    type Texture: Send + Sync + fmt::Debug;
    type RenderPipeline: Send + Sync;
    // ...
}
```

The Vulkan backend implements `Api` in `wgpu-hal/src/vulkan/`. The key files are:

| File | Role |
|---|---|
| `instance.rs` | `vkCreateInstance`, extension/layer enumeration, debug utils |
| `adapter.rs` | Physical device selection, feature/limit mapping, extension requirements |
| `device.rs` | `vkCreateDevice`, buffer/texture/pipeline creation, descriptor sets |
| `command.rs` | `CommandEncoder`: render and compute pass recording, pipeline barriers |
| `conv.rs` | Type conversion: wgpu enums/structs ↔ Vulkan enums/structs |

The backend uses **ash** exclusively for Vulkan bindings. Extension objects are loaded at device creation and stored in the `Device` struct:

```rust
// wgpu-hal/src/vulkan/device.rs (pattern for an extension)
pub struct Device {
    shared: Arc<DeviceShared>,
    // extension function tables, loaded once at vkCreateDevice time
    ext_mesh_shader: Option<ash::khr::MeshShader>,
    ext_external_memory_fd: ash::khr::ExternalMemoryFd,
    // ...
}

// Usage in command recording:
if let Some(ext) = &self.ext_mesh_shader {
    unsafe { ext.cmd_draw_mesh_tasks(cmd_buffer, x, y, z); }
}
```

To implement a new Vulkan extension in the HAL:

1. Add the extension name to `DeviceExtensions::required_by_features` or `optional_extensions` in `adapter.rs`.
2. Load the ash extension struct in `Device::new` and store it as `Option<ash::ext::Foo>`.
3. Implement the HAL trait method, guarding with `if let Some(ext) = &self.ext_foo`.
4. Add `conv.rs` conversions for any new Vulkan enums the extension introduces.

The `wgpu-hal/src/vulkan/conv.rs` file is the most frequently touched during feature work — it contains hundreds of `match` arms mapping wgpu's format/usage/stage enums to Vulkan equivalents.

### naga: Shader IR and Compilation

Naga's IR is a typed SSA representation. The root types are:

```rust
// naga/src/lib.rs — core IR containers
pub struct Module {
    pub types: UniqueArena<Type>,       // deduplicated type table
    pub constants: Arena<Constant>,
    pub global_variables: Arena<GlobalVariable>,
    pub functions: Arena<Function>,
    pub entry_points: Vec<EntryPoint>,
}

pub struct Function {
    pub arguments: Vec<FunctionArgument>,
    pub result: Option<FunctionResult>,
    pub local_variables: Arena<LocalVariable>,
    pub expressions: Arena<Expression>,  // SSA expression nodes
    pub body: Block,                     // statement list
}
```

`Arena<T>` is naga's typed handle-indexed allocator. Every expression and statement is reached via a `Handle<T>`, which is an index into the arena — there are no raw pointers in the IR.

**Contributing to naga** falls into four categories:

**1. Fixing WGSL validation** — the validator runs in `naga/src/valid/` after parsing. It enforces shader-stage constraints (`@vertex`/`@fragment`/`@compute`), type compatibility, resource binding rules, and WGSL spec invariants. Most validation bugs are discovered via WebGPU CTS failures. The fix pattern is: identify the `ValidationError` variant, trace it to the validator pass that emits it, and add or relax the check per spec.

**2. Adding IR nodes for new language features** — new WGSL built-ins (e.g., subgroup operations, `textureSampleBias` variants) require adding `Expression::*` or `Statement::*` variants. The Rust compiler enforces exhaustive `match` coverage across all backends, so adding a variant immediately surfaces all sites that need updating.

**3. Improving back-end writers** — the SPIR-V writer (`naga/src/back/spv/`) and GLSL writer (`naga/src/back/glsl/`) are the most active on Linux. Common contributions: fixing texture sampling edge cases, improving control-flow translation (WGSL `loop`/`continuing` blocks → SPIR-V structured control flow), and handling integer type-cast corner cases.

**4. Adding snapshot tests** — naga uses the `insta` crate for snapshot testing. Each test compiles a shader input and compares the output against a stored snapshot:

```bash
# Run tests and accept new or changed snapshots
INSTA_UPDATE=new cargo test -p naga
# Review changed snapshots interactively
cargo insta review
```

Snapshot files live in `naga/tests/snapshots/`. The CI enforces that all snapshots are committed and reviewed — un-reviewed snapshot changes fail CI. This makes naga regressions detectable without a physical GPU.

### Testing Infrastructure

`wgpu/tests/` contains the hardware integration tests. The `gpu_test` attribute macro declares a test with device requirements:

```rust
// wgpu/tests/render_pass.rs (representative pattern)
#[gpu_test]
static DEPTH_CLIP_CONTROL: GpuTestConfiguration = GpuTestConfiguration::new()
    .parameters(
        TestParameters::default()
            .features(Features::DEPTH_CLIP_CONTROL)
            .expect_fail(FailureCase::backend(Backends::GLES)),
    )
    .run_sync(|ctx| {
        // ctx.device: wgpu::Device, ctx.queue: wgpu::Queue
        let pipeline = ctx.device.create_render_pipeline(&RenderPipelineDescriptor {
            // ...
        });
        // submit and readback
    });
```

Tests that require features absent on the test device are automatically skipped. `expect_fail` marks known backend-specific failures without breaking CI.

**WebGPU CTS integration**: wgpu maintains a harness that runs the W3C WebGPU conformance test suite against `wgpu-core`. CTS failures in `webgpu:shader/execution/*` are usually naga issues; failures in `webgpu:api/*` are usually wgpu-core validation or resource management issues.

```bash
# Run the CTS against the Vulkan backend (requires cloning the CTS separately)
WGPU_BACKEND=vulkan cargo run -p wgpu-cts-runner -- --filter webgpu:shader/execution
```

### Mozilla's Upstream Relationship

Firefox does not maintain a wgpu fork. Mozilla engineers contribute upstream to `gfx-rs/wgpu` and pin a specific commit in `mozilla-central/third_party/rust/` managed via `mach vendor rust`. The update cadence is approximately quarterly, gated by Firefox's release train and CTS pass rate on the pinned commit.

Mozilla's primary upstream contributions from the Firefox integration have been: WGSL validation correctness fixes (reducing spec divergence from Tint that caused Firefox–Chrome WebGPU behaviour differences), Linux Wayland DMABuf surface integration in `wgpu-hal::vulkan::Surface`, GLES backend robustness for the Android WebGPU path, and `wgpu-core` resource lifecycle edge cases surfaced by the WebGPU CTS.

The most impactful areas for contributors who want their work to benefit Firefox are: naga WGSL validation correctness (reducing CTS failures in `webgpu:shader/`), the Vulkan HAL's Linux Wayland surface path, and `wgpu-core` resource tracking for `GPUBuffer.mapAsync` and timestamp query edge cases.

[Source: wgpu CONTRIBUTING.md](https://github.com/gfx-rs/wgpu/blob/trunk/CONTRIBUTING.md)
[Source: Mozilla central wgpu-core vendored source](https://searchfox.org/mozilla-central/source/third_party/rust/wgpu-core)

---

## Roadmap

### Near-term (6–12 months)

- **wgpu stable API consolidation**: The gfx-rs project is working toward a stable 1.0 release of wgpu, with focus on API stability guarantees and reduced breaking changes across minor versions. [Source](https://wgpu.rs/)
- **naga repository consolidation**: Following the RFC to merge naga into the wgpu monorepo ([Source](https://github.com/gfx-rs/wgpu/issues/4231)), naga continues development as an integrated but separately publishable crate within `gfx-rs/wgpu`, with improved WGSL validation and SPIR-V round-trip fidelity.
- **rust-gpu community stewardship**: After Embark Studios handed off the project to community governance under the `Rust-GPU` GitHub organization ([Source](https://rust-gpu.github.io/blog/transition-announcement/)), the near-term priority is stabilising the rustc SPIR-V backend and expanding test coverage for Vulkan compute shaders.
- **wgpu WebGPU spec compliance**: As the WebGPU specification stabilises in browsers, wgpu's `wgpu-core` (used as Firefox's WebGPU implementation) is tracking spec changes to keep browser and native backends in sync. Note: needs verification of specific milestone.
- **gpu-allocator TLSF improvements**: Ongoing refinements to the TLSF sub-allocator to reduce fragmentation and add D3D12 backend support alongside the existing Vulkan/Metal paths. [Source](https://github.com/Traverse-Research/gpu-allocator)

### Medium-term (1–3 years)

- **wgpu bindless and ray-tracing extensions**: There is active discussion in the gfx-rs community about exposing `VK_KHR_ray_tracing_pipeline` and bindless resource models via opt-in `wgpu::Features` flags, analogous to how mesh-shader support (`POLYGON_MODE_LINE`, `MULTI_DRAW_INDIRECT`) was progressively added. Note: no merged implementation confirmed at time of writing.
- **rust-gpu DXIL and WGSL targets**: The community rust-gpu roadmap lists potential support for DXIL (Direct3D shader bytecode) and WGSL as additional compiler output targets beyond the current SPIR-V backend, enabling Rust shaders on non-Vulkan platforms. [Source](https://github.com/EmbarkStudios/rust-gpu/issues/47)
- **naga GLSL front-end stability**: naga's GLSL ingestion path is considered less mature than WGSL/SPIR-V; medium-term work targets full GLSL 4.5/4.6 coverage so existing shader assets can be translated without hand-editing.
- **vulkano 1.0 type-system hardening**: The vulkano project aims to encode additional Vulkan validity constraints (pipeline compatibility, render-pass compatibility) in Rust's type system to eliminate a remaining class of runtime panics. Note: needs verification of current milestone status.
- **Bevy renderer rewrite (Bevy 0.16+)**: Bevy's render architecture is undergoing modularisation (retained render world, async pipeline compilation) that feeds back into wgpu feature requests around pipeline caching and async GPU resource uploads. [Source](https://bevyengine.org/)

### Long-term

- **Safe Rust Vulkan abstractions at zero cost**: The broader ecosystem goal is to close the ergonomics and performance gap between `ash` (fully unsafe, zero-overhead) and `wgpu` (safe but abstracted), potentially via a mid-level crate that encodes Vulkan synchronisation and resource lifetimes as Rust types without hiding API concepts.
- **Rust shaders in production engines**: As rust-gpu matures, a long-term goal is first-class shader authoring in Rust for production game engines (Bevy being the leading candidate), removing the impedance mismatch of writing game logic in Rust and shaders in WGSL/GLSL.
- **WebGPU on Linux without a browser**: wgpu's native Vulkan backend could serve as the WebGPU implementation layer for Linux desktop applications, allowing the same WGSL shaders to target both the browser (via `wasm32`) and a native Linux Vulkan driver without any code change.
- **Cooperative matrix and ML inference in wgpu**: Long-term, extensions such as `VK_KHR_cooperative_matrix` and `VK_NV_cooperative_matrix2` may be exposed through wgpu's feature flag system, enabling GEMM-optimised shaders for ML inference workloads written in safe Rust. Note: no timeline confirmed.

---

## Integrations

- **Ch19 (Vulkan Architecture)** — `ash` exposes identical Vulkan concepts to the C API; instance, device, command buffers, pipelines map 1:1
- **Ch20 (SPIR-V)** — naga produces SPIR-V for wgpu's Vulkan backend; naga's SPIR-V front-end consumes glslc output; rust-gpu also targets SPIR-V via `rustc_codegen_spirv`
- **Ch22 (RADV)** — wgpu selects RADV on AMD via Vulkan enumeration; `adapter.get_info()` returns the Mesa driver name
- **Ch23 (ANV)** — ANV is the Vulkan driver used by wgpu on Intel
- **Ch57 (WebGPU in Chromium)** — Firefox's WebGPU implementation (`wgpu-core`) is the production wgpu implementation
- **Ch141 (Cooperative Matrices)** — wgpu does not yet expose `VK_KHR_cooperative_matrix`; that requires `ash` or raw Vulkan; blade's Vulkan backend uses `khr::cooperative_matrix` directly
- **Ch134 (Asahi/Apple GPU)** — wgpu's Metal backend is used by the Asahi Linux WebGPU stack
- **Ch25 (GPU Compute)** — cudarc and cuTile-rs cover CUDA-path GPU compute; burn's `burn-wgpu` backend uses wgpu storage buffers for the same compute patterns on open drivers
- **Ch35 (Dawn/WebGPU)** — Dawn is Chrome's WebGPU implementation; wgpu (Firefox) and Dawn share the WGSL/naga shader layer and both consume SPIR-V from rust-gpu or `vulkano_shaders`
- **Ch40 (Bevy/wgpu)** — Bevy's entire renderer is built on wgpu; naga_oil's `Composer` powers Bevy's modular PBR shader system; vello is the planned 2-D rendering layer for Bevy UI
- **Ch177 (NVK — NVIDIA Vulkan)** — cuTile-rs and cudarc target the CUDA path; on Linux with NVK (open-source NVIDIA Vulkan), wgpu and ash speak to the same hardware through the Vulkan layer instead
- **Ch15 (ACO — AMD Shader Compiler)** — naga-compiled WGSL from wgpu, vello, and burn-wgpu all enter the Mesa shader pipeline (ACO on RADV) as SPIR-V; ACO's instruction scheduling is the final stage before the GPU executes shaders that began as Rust or WGSL source
- **Ch24 (Vulkan WSI / EGL)** — winit + ash-window create `VkSurfaceKHR` via `VK_KHR_wayland_surface` or `VK_KHR_xcb_surface`; wgpu's surface creation calls the same WSI extensions under the hood
- **Ch82 (VMA / Vulkan Helpers)** — glam's `Mat4` / `Vec4` types fill the same role in Rust that GLM fills in C++; both upload via aligned uniform buffers to the same GPU memory managed by VMA / gpu-allocator
- **Ch84 (bgfx / Filament / IGL)** — egui provides a comparable immediate-mode overlay to Dear ImGui; the egui-wgpu backend is architecturally parallel to imgui_impl_vulkan
- **Ch25 (GPU Compute)** — candle's CUDA backend uses cudarc (§11); its inference throughput on NVIDIA hardware is comparable to burn-cuda; both are Rust alternatives to Python/PyTorch for on-device LLM inference
- **Ch30 (Debugging and Profiling)** — wgpu-profiler timestamps feed the same Tracy timeline as CPU profiling; the GPU scope names appear as GPU zones in the Tracy UI

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
