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

## cuTile-rs: NVIDIA Tile Abstraction in Rust

[cuTile-rs](https://github.com/nvlabs/cutile-rs) is NVIDIA's official Rust library for tensor-core–oriented GPU kernel authoring. It forms part of NVIDIA's strategic investment in Rust for performance-critical GPU code — evidence that the company views Rust as viable beyond systems software and into HPC kernel authoring.

### The Tile Programming Model

The central idea is *tile programs, not threads*: a kernel is written as a single-threaded program that operates on typed data tiles, and the compiler (backed by CUDA Tile IR) maps those operations onto warps, thread-blocks, and Tensor Cores automatically. The programmer never manually manages shared memory, warp-level synchronisation, or `wmma` instruction selection.

The core abstraction is a `Tile<T, ROWS, COLS>` generic type representing a 2-D partition of an array held cooperatively by a group of threads. On NVIDIA hardware this maps to the PTX cooperative matrix family — `wmma.load`, `wmma.mma.sync`, and `wmma.store` for older architectures, and the newer `wgmma` (Warp-Group Matrix Multiply Accumulate) on Hopper/Ada — giving Tensor Core throughput with a safe Rust API. [PTX reference](https://docs.nvidia.com/cuda/parallel-thread-execution/)

### GEMM Kernel Pattern

```rust
// Illustrative cuTile-rs GEMM pattern (refer to nvlabs/cutile-rs examples for
// the exact API version targeting your CUDA toolkit).
use cutile::prelude::*;

#[kernel]
pub fn gemm_kernel(
    a: &Tensor2D<f16>,
    b: &Tensor2D<f16>,
    c: &mut Tensor2D<f32>,
) {
    // Partition the output space into tiles matching the cooperative group size.
    let output_tile = c.partition_like_output();

    // Load input tiles in shapes that match the output partition.
    let a_tile = a.load_tile_like(&output_tile);
    let b_tile = b.load_tile_like(&output_tile);

    // Tensor Core multiply-accumulate: maps to wgmma.mma_async on Hopper.
    let acc = Tile::mma(&a_tile, &b_tile);

    // Write the accumulated result back.
    output_tile.store(&acc);
}
```

The `#[kernel]` procedural macro wraps the Rust function into a CUDA kernel entry point, while `Tile::mma` is lowered by the CUDA Tile IR backend to the appropriate PTX cooperative matrix instructions for the target architecture. On B200 hardware this approach reaches 96 % of cuBLAS GEMM throughput. [Source](https://nvlabs.github.io/cutile-rs/main/)

### Why NVIDIA Is Investing

NVIDIA's rationale follows the broader industry pattern: Rust eliminates buffer overruns and data races in device code, and the ownership model can statically encode GPU memory hazards (concurrent read/write aliasing) that CUDA C++ can only catch at run time via the sanitizer. cuTile-rs targets research teams and framework authors (e.g., cuDNN successor kernels) rather than end-application developers. [Source](https://github.com/nvlabs/cutile-rs)

The tile abstraction also decouples algorithmic intent from microarchitecture. A GEMM that today targets Hopper's `wgmma` instructions can retarget Blackwell's next-generation matrix engine by recompiling; the kernel source stays unchanged. This composability is exactly what custom training loop authors need when they want to experiment with quantised data types (fp8, int4) or non-standard accumulation orders without rewriting schedule logic. The companion `TileGym` benchmark suite tests these kernels against cuBLAS baselines, measuring how close the safe Rust path gets to hand-tuned assembly across GPU generations. [Source](https://nvlabs.github.io/cutile-rs/main/)

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

    // Allocate and copy from host slice.
    let input_data: Vec<f32> = (0..1024).map(|x| x as f32).collect();
    let input = stream.memcpy_stoh(&input_data)?;

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
    let inp     = stream.memcpy_stoh(&vec![1.0f32; 1024])?;

    // Type-checked builder: args must match the PTX parameter list order.
    let mut builder = stream.launch_builder(&add_fn);
    builder.arg(&mut out);
    builder.arg(&inp);
    unsafe { builder.launch(LaunchConfig::for_num_elems(1024)) }?;

    let result = stream.memcpy_dtoh(&out)?;
    println!("result[0] = {}", result[0]);   // 1.0
    Ok(())
}
```

`LaunchConfig::for_num_elems(n)` selects a sensible block/grid split automatically. For hand-tuned configurations supply `LaunchConfig { block_dim: (256,1,1), grid_dim: (n/256, 1, 1), shared_mem_bytes: 0 }`. The PTX can also be compiled at run time via the bundled `cudarc::nvrtc::compile_ptx` wrapper, which invokes NVRTC in-process from a CUDA C source string — useful for JIT compilation of generated kernels. [Source](https://docs.rs/cudarc/latest/cudarc/driver/safe/struct.CudaContext.html)

### Pairing with cuTile-rs

The typical workflow combines both crates: cuTile-rs compiles Rust kernel code to PTX via the CUDA Tile IR backend, and cudarc loads that PTX and manages host-side data movement. cuTile-rs owns the *what* (tile computation) and cudarc owns the *how* (driver interaction, memory, streams).

From a build-system perspective, a project typically has three crates: a `kernel` crate (Rust code annotated with `#[kernel]`, compiled to PTX by the cuTile-rs procedural macro), a `host` crate (the application, which depends on `cudarc` and embeds the compiled PTX via `include_bytes!`), and a `build.rs` that invokes the cuTile-rs toolchain. This mirrors the rust-gpu / spirv-builder split exactly, but targeting CUDA rather than Vulkan. The practical benefit over raw CUDA C++ is that kernel bugs involving out-of-bounds tile accesses or race conditions on shared state are caught by rustc's borrow checker rather than discovered at run time under CUDA-MEMCHECK. For ML framework authors building custom attention kernels or quantised GEMMs, this shifts a class of subtle correctness bugs from production inference to the compile step. [Source](https://github.com/coreylowman/cudarc)

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
- **`burn-candle`** — thin wrapper over the Candle ML framework.

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

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
