# Chapter 152: Rust GPU Ecosystem: wgpu, ash, and naga

**Target audiences**: Rust developers building GPU-accelerated applications; developers migrating C/C++ GPU code to Rust; and engineers interested in how type-safe, memory-safe abstractions over Vulkan and WebGPU work in practice.

---

## Table of Contents

1. [Introduction](#introduction)
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
19. [Roadmap](#roadmap)
20. [Integrations](#integrations)

---

## Introduction

The Rust GPU ecosystem has matured rapidly. The two main tracks are:

1. **Raw Vulkan** via `ash` or `erupt` — full control, unsafe, idiomatic Rust bindings to the Vulkan C API
2. **WebGPU-safe abstraction** via `wgpu` — safe Rust API, runs on Vulkan/Metal/DX12/WebGPU, with `naga` as the shader compiler

On Linux, `wgpu` targets Vulkan (via `wgpu-hal/vulkan`) and OpenGL (via `wgpu-hal/gles`). `ash` is the canonical raw Vulkan crate. Both are integral to the Rust game/rendering ecosystem and are used by projects from Bevy (game engine) to Firefox (WebGPU) to llama.cpp's Vulkan backend.

Sources: [wgpu](https://github.com/gfx-rs/wgpu) | [ash](https://github.com/ash-rs/ash) | [naga](https://github.com/gfx-rs/wgpu/tree/trunk/naga)

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

`erupt` is an alternative to `ash`, auto-generated from Vulkan XML with slightly different ergonomics:

```toml
[dependencies]
erupt = "0.23"
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
- **Ch20 (SPIR-V)** — naga produces SPIR-V for wgpu's Vulkan backend; naga's SPIR-V front-end consumes glslc output
- **Ch22 (RADV)** — wgpu selects RADV on AMD via Vulkan enumeration; `adapter.get_info()` returns the Mesa driver name
- **Ch23 (ANV)** — ANV is the Vulkan driver used by wgpu on Intel
- **Ch57 (WebGPU in Chromium)** — Firefox's WebGPU implementation (`wgpu-core`) is the production wgpu implementation
- **Ch141 (Cooperative Matrices)** — wgpu does not yet expose `VK_KHR_cooperative_matrix`; that requires `ash` or raw Vulkan
- **Ch134 (Asahi/Apple GPU)** — wgpu's Metal backend is used by the Asahi Linux WebGPU stack
