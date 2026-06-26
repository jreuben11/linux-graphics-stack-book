# Appendix U: WebGPU API Quick Reference

> **Reference baseline**: WebGPU W3C Candidate Recommendation Draft, June 2026
> **Primary sources**: [W3C WebGPU spec](https://www.w3.org/TR/webgpu/) | [wgpu docs](https://docs.rs/wgpu/latest/wgpu/) | [Dawn C++ API](https://dawn.googlesource.com/dawn)

**Audience**: Browser/web platform engineers and application developers using the WebGPU JavaScript API, the Rust `wgpu` crate, or Dawn's C++ `webgpu.h`. This appendix covers the JavaScript/WebIDL API surface; the `wgpu` Rust equivalents are noted in brackets. See Chapter 35 (Dawn), Chapter 40 (Bevy/wgpu), and Chapter 98 (WebAssembly/WebGPU) for architectural context and usage patterns. For the WGSL shader language, see Appendix P.

---

## Table of Contents

1. [Initialisation — Adapter and Device](#1-initialisation--adapter-and-device)
2. [GPUBuffer](#2-gpubuffer)
3. [GPUTexture and GPUTextureView](#3-gputexture-and-gputextureview)
4. [GPUSampler](#4-gpusampler)
5. [GPUShaderModule](#5-gpushadermodule)
6. [Bind Groups and Layouts](#6-bind-groups-and-layouts)
7. [GPURenderPipeline](#7-gpurenderpipeline)
8. [GPUComputePipeline](#8-gpucomputepipeline)
9. [GPUCommandEncoder and Passes](#9-gpucommandencoder-and-passes)
10. [GPUQueue](#10-gpuqueue)
11. [Canvas and Swap Chain](#11-canvas-and-swap-chain)
12. [Key Enums and Constants](#12-key-enums-and-constants)
13. [Error Handling](#13-error-handling)

---

## 1. Initialisation — Adapter and Device

```js
// JS / WebIDL
const adapter = await navigator.gpu.requestAdapter({
    powerPreference: 'high-performance',  // 'low-power' | 'high-performance' | undefined
    forceFallbackAdapter: false,
});
const device = await adapter.requestDevice({
    requiredFeatures: ['shader-f16', 'texture-compression-bc'],
    requiredLimits:   { maxBufferSize: 2147483648 },
    defaultQueue:     { label: 'main queue' },
});
device.lost.then(info => console.error('Device lost:', info.reason, info.message));
```

```rust
// wgpu Rust equivalent
let instance = wgpu::Instance::new(&wgpu::InstanceDescriptor {
    backends: wgpu::Backends::all(), ..Default::default()
});
let adapter = instance.request_adapter(&wgpu::RequestAdapterOptions {
    power_preference: wgpu::PowerPreference::HighPerformance,
    compatible_surface: Some(&surface),
    ..Default::default()
}).await.unwrap();
let (device, queue) = adapter.request_device(&wgpu::DeviceDescriptor {
    required_features: wgpu::Features::SHADER_F16,
    required_limits:   wgpu::Limits::default(),
    ..Default::default()
}, None).await.unwrap();
```

**`GPUAdapter` methods:**

| Method / Property                        | Description                                           |
|------------------------------------------|-------------------------------------------------------|
| `adapter.requestDevice(desc)`            | → `Promise<GPUDevice>`                                |
| `adapter.features`                       | `GPUSupportedFeatures` — set of feature names         |
| `adapter.limits`                         | `GPUSupportedLimits` — max values                     |
| `adapter.info` (formerly `requestAdapterInfo`) | Vendor, architecture, device, driver string    |
| `adapter.isFallbackAdapter`              | True if software/CPU fallback                         |

---

## 2. GPUBuffer

```js
// Create
const buf = device.createBuffer({
    label:            'vertex buffer',
    size:             byteLength,
    usage:            GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST,
    mappedAtCreation: true,   // map immediately for initialisation
});
// Write initial data (only valid when mappedAtCreation: true)
new Float32Array(buf.getMappedRange()).set(vertexData);
buf.unmap();

// Upload data after creation
device.queue.writeBuffer(buf, byteOffset, typedArray);

// Read back (map for CPU read)
await buf.mapAsync(GPUMapMode.READ, offset, size);
const data = new Float32Array(buf.getMappedRange(offset, size));
// ... read data ...
buf.unmap();

// Destroy
buf.destroy();
```

**`GPUBufferUsage` flags (bitmask):**

| Flag                        | Hex      | Purpose                                          |
|-----------------------------|----------|--------------------------------------------------|
| `MAP_READ`                  | `0x0001` | `mapAsync(READ)` — read results back to CPU      |
| `MAP_WRITE`                 | `0x0002` | `mapAsync(WRITE)` — write from CPU               |
| `COPY_SRC`                  | `0x0004` | Source of `copyBufferToBuffer` / `copyBufferToTexture` |
| `COPY_DST`                  | `0x0008` | Destination; required for `writeBuffer`          |
| `INDEX`                     | `0x0010` | Index buffer in render pass                      |
| `VERTEX`                    | `0x0020` | Vertex buffer in render pass                     |
| `UNIFORM`                   | `0x0040` | Uniform / constant buffer in bind group          |
| `STORAGE`                   | `0x0080` | Storage buffer in bind group (compute)           |
| `INDIRECT`                  | `0x0100` | Indirect draw/dispatch arguments                 |
| `QUERY_RESOLVE`             | `0x0200` | Resolve `GPUQuerySet` results into this buffer   |

**`GPUBuffer` properties / methods:**

| Member                                   | Description                                     |
|------------------------------------------|-------------------------------------------------|
| `buf.size`                               | Buffer size in bytes                            |
| `buf.usage`                              | Usage bitmask                                   |
| `buf.mapState`                           | `'unmapped'` \| `'pending'` \| `'mapped'`       |
| `buf.mapAsync(mode, offset?, size?)`     | → `Promise<void>`; `mode`: `GPUMapMode.READ/WRITE` |
| `buf.getMappedRange(offset?, size?)`     | → `ArrayBuffer` (only while mapped)             |
| `buf.unmap()`                            | Release CPU mapping; buffer usable by GPU again |
| `buf.destroy()`                          | Free GPU memory immediately                     |

---

## 3. GPUTexture and GPUTextureView

```js
// Create texture
const tex = device.createTexture({
    label:         'colour target',
    size:          { width: 1920, height: 1080, depthOrArrayLayers: 1 },
    mipLevelCount: 1,
    sampleCount:   1,                  // 1 = no MSAA; 4 = 4× MSAA
    dimension:     '2d',               // '1d' | '2d' | '3d'
    format:        'rgba8unorm',
    usage:         GPUTextureUsage.RENDER_ATTACHMENT | GPUTextureUsage.TEXTURE_BINDING,
});

// Upload texture data
device.queue.writeTexture(
    { texture: tex, mipLevel: 0, origin: { x: 0, y: 0, z: 0 } },
    pixelData,                         // ArrayBuffer | TypedArray
    { bytesPerRow: 1920 * 4, rowsPerImage: 1080 },
    { width: 1920, height: 1080 }
);

// Create view (default: full texture, all mips and layers)
const view = tex.createView();
// Custom view:
const mip_view = tex.createView({
    format:          'rgba8unorm',
    dimension:       '2d',
    aspect:          'all',            // 'all' | 'depth-only' | 'stencil-only'
    baseMipLevel:    0,
    mipLevelCount:   1,
    baseArrayLayer:  0,
    arrayLayerCount: 1,
});
tex.destroy();
```

**`GPUTextureUsage` flags:**

| Flag                    | Hex      | Purpose                                              |
|-------------------------|----------|------------------------------------------------------|
| `COPY_SRC`              | `0x01`   | Source for copy operations                           |
| `COPY_DST`              | `0x02`   | Destination for copy / `writeTexture`                |
| `TEXTURE_BINDING`       | `0x04`   | Sample in shader via `texture_2d<f32>` binding       |
| `STORAGE_BINDING`       | `0x08`   | Read/write in shader via `texture_storage_2d`        |
| `RENDER_ATTACHMENT`     | `0x10`   | Use as colour/depth/stencil attachment               |

**Common `GPUTextureFormat` values:**

| Format              | Channels | Bits/ch | sRGB? | Notes                          |
|---------------------|----------|---------|-------|--------------------------------|
| `rgba8unorm`        | RGBA     | 8       | No    | Most common colour format      |
| `rgba8unorm-srgb`   | RGBA     | 8       | Yes   | Linear decode on sample        |
| `bgra8unorm`        | BGRA     | 8       | No    | Preferred swap chain on some platforms |
| `rgba16float`       | RGBA     | 16      | No    | HDR render target              |
| `rgba32float`       | RGBA     | 32      | No    | High-precision compute         |
| `r32float`          | R        | 32      | No    | Depth copy, single-channel     |
| `depth24plus`       | D        | 24+     | —     | Depth-only; stencil optional   |
| `depth24plus-stencil8` | D+S   | 24+8    | —     | Combined depth-stencil         |
| `depth32float`      | D        | 32      | —     | High-precision depth           |
| `bc7-rgba-unorm`    | RGBA     | 8 avg   | No    | BC7 compressed (desktop)       |
| `etc2-rgb8unorm`    | RGB      | 8 avg   | No    | ETC2 compressed (mobile/WebGL) |
| `rg11b10ufloat`     | RGB      | 11/11/10| No    | Compact HDR render target      |
| `stencil8`          | S        | 8       | —     | Stencil-only                   |

---

## 4. GPUSampler

```js
const sampler = device.createSampler({
    label:         'linear clamp',
    addressModeU:  'clamp-to-edge',    // 'clamp-to-edge' | 'repeat' | 'mirror-repeat'
    addressModeV:  'clamp-to-edge',
    addressModeW:  'clamp-to-edge',
    magFilter:     'linear',           // 'nearest' | 'linear'
    minFilter:     'linear',
    mipmapFilter:  'linear',           // 'nearest' | 'linear'
    lodMinClamp:   0,
    lodMaxClamp:   32,
    compare:       undefined,          // 'never'|'less'|'equal'|'less-equal'|'greater'|'not-equal'|'greater-equal'|'always' (for shadow samplers)
    maxAnisotropy: 16,                 // 1–16; requires 'texture-anisotropic-filtering' feature
});
```

---

## 5. GPUShaderModule

```js
const shader = device.createShaderModule({
    label: 'my shader',
    code:  wgslSourceString,
    // compilationHints: [{ entryPoint: 'vs_main', layout: pipelineLayout }]
});
// Async compilation info (optional, non-blocking check):
const info = await shader.getCompilationInfo();
for (const msg of info.messages) {
    console.log(msg.type, msg.lineNum, msg.linePos, msg.message);
}
```

---

## 6. Bind Groups and Layouts

```js
// Explicit bind group layout
const bgl = device.createBindGroupLayout({
    entries: [
        { binding: 0, visibility: GPUShaderStage.VERTEX | GPUShaderStage.FRAGMENT,
          buffer: { type: 'uniform' } },
        { binding: 1, visibility: GPUShaderStage.FRAGMENT,
          texture: { sampleType: 'float', viewDimension: '2d', multisampled: false } },
        { binding: 2, visibility: GPUShaderStage.FRAGMENT,
          sampler: { type: 'filtering' } },
        { binding: 3, visibility: GPUShaderStage.COMPUTE,
          storageTexture: { access: 'write-only', format: 'rgba8unorm', viewDimension: '2d' } },
        { binding: 4, visibility: GPUShaderStage.COMPUTE,
          buffer: { type: 'storage' } },        // 'uniform' | 'storage' | 'read-only-storage'
    ]
});

const pipeline_layout = device.createPipelineLayout({ bindGroupLayouts: [bgl] });

// Bind group (concrete resources bound to layout slots)
const bg = device.createBindGroup({
    layout: bgl,
    entries: [
        { binding: 0, resource: { buffer: uniformBuf, offset: 0, size: 256 } },
        { binding: 1, resource: textureView },
        { binding: 2, resource: sampler },
        { binding: 3, resource: storageTexView },
        { binding: 4, resource: { buffer: storageBuf } },
    ]
});
```

**`GPUShaderStage` visibility flags:**

| Flag        | Value | Stage     |
|-------------|-------|-----------|
| `VERTEX`    | `0x1` | Vertex    |
| `FRAGMENT`  | `0x2` | Fragment  |
| `COMPUTE`   | `0x4` | Compute   |

**Buffer binding types:** `'uniform'` | `'storage'` | `'read-only-storage'`  
**Texture sample types:** `'float'` | `'unfilterable-float'` | `'depth'` | `'sint'` | `'uint'`  
**Sampler types:** `'filtering'` | `'non-filtering'` | `'comparison'`

---

## 7. GPURenderPipeline

```js
const pipeline = device.createRenderPipeline({
    label:  'main pipeline',
    layout: pipeline_layout,   // or 'auto' for implicit layout
    vertex: {
        module:     shader,
        entryPoint: 'vs_main',
        buffers: [{
            arrayStride:    5 * 4,  // bytes per vertex
            stepMode:       'vertex',   // 'vertex' | 'instance'
            attributes: [
                { format: 'float32x3', offset: 0,      shaderLocation: 0 },  // position
                { format: 'float32x2', offset: 3 * 4,  shaderLocation: 1 },  // uv
            ]
        }]
    },
    fragment: {
        module:     shader,
        entryPoint: 'fs_main',
        targets: [{
            format: 'rgba8unorm',
            blend: {
                color: { operation: 'add', srcFactor: 'src-alpha', dstFactor: 'one-minus-src-alpha' },
                alpha: { operation: 'add', srcFactor: 'one',       dstFactor: 'zero' },
            },
            writeMask: GPUColorWrite.ALL,
        }]
    },
    primitive: {
        topology:         'triangle-list',  // 'point-list'|'line-list'|'line-strip'|'triangle-list'|'triangle-strip'
        stripIndexFormat: undefined,        // 'uint16'|'uint32' (strip topologies only)
        frontFace:        'ccw',            // 'cw' | 'ccw'
        cullMode:         'back',           // 'none' | 'front' | 'back'
        unclippedDepth:   false,
    },
    depthStencil: {
        format:              'depth24plus',
        depthWriteEnabled:   true,
        depthCompare:        'less',        // compare function (see enum below)
        stencilFront:        { compare: 'always', failOp: 'keep', depthFailOp: 'keep', passOp: 'keep' },
        stencilBack:         { compare: 'always', failOp: 'keep', depthFailOp: 'keep', passOp: 'keep' },
        stencilReadMask:     0xFFFFFFFF,
        stencilWriteMask:    0xFFFFFFFF,
        depthBias:           0,
        depthBiasSlopeScale: 0,
        depthBiasClamp:      0,
    },
    multisample: {
        count:                  4,          // MSAA samples (1 = no MSAA)
        mask:                   0xFFFFFFFF,
        alphaToCoverageEnabled: false,
    },
});
```

**Vertex formats (`GPUVertexFormat`):**

| Format         | WGSL type   | Bytes | Notes                        |
|----------------|-------------|-------|------------------------------|
| `float32`      | `f32`       | 4     |                              |
| `float32x2`    | `vec2f`     | 8     |                              |
| `float32x3`    | `vec3f`     | 12    |                              |
| `float32x4`    | `vec4f`     | 16    |                              |
| `uint32`       | `u32`       | 4     |                              |
| `uint8x4`      | `vec4u` (unorm)| 4  | Normalised 0–1 range         |
| `unorm8x4`     | `vec4f`     | 4     | 4 × normalised uint8         |
| `snorm8x4`     | `vec4f`     | 4     | 4 × normalised int8          |
| `unorm16x2`    | `vec2f`     | 4     | 2 × normalised uint16        |
| `float16x2`    | `vec2<f16>` | 4     | Requires `shader-f16`        |
| `float16x4`    | `vec4<f16>` | 8     | Requires `shader-f16`        |

**Compare functions:** `'never'` | `'less'` | `'equal'` | `'less-equal'` | `'greater'` | `'not-equal'` | `'greater-equal'` | `'always'`

**`GPUColorWrite` flags:** `RED=1` | `GREEN=2` | `BLUE=4` | `ALPHA=8` | `ALL=15`

---

## 8. GPUComputePipeline

```js
const compute_pipeline = device.createComputePipeline({
    label:  'particle update',
    layout: 'auto',
    compute: {
        module:     compute_shader,
        entryPoint: 'cs_main',
        constants:  { 0: 64.0 },    // pipeline-overridable constants (WGSL `override`)
    },
});
// Async variant (non-blocking compilation):
const pipeline_p = device.createComputePipelineAsync({ ... });
```

---

## 9. GPUCommandEncoder and Passes

### Render Pass

```js
const enc = device.createCommandEncoder({ label: 'frame encoder' });

const pass = enc.beginRenderPass({
    label: 'main pass',
    colorAttachments: [{
        view:          swapchainView,
        resolveTarget: msaaResolveView,   // for MSAA resolve (omit if sampleCount=1)
        loadOp:        'clear',           // 'load' | 'clear'
        storeOp:       'store',           // 'store' | 'discard'
        clearValue:    { r: 0, g: 0, b: 0, a: 1 },
    }],
    depthStencilAttachment: {
        view:              depthView,
        depthLoadOp:       'clear',
        depthStoreOp:      'discard',
        depthClearValue:   1.0,
        stencilLoadOp:     'clear',
        stencilStoreOp:    'discard',
        stencilClearValue: 0,
    },
    occlusionQuerySet: querySet,    // optional
    timestampWrites:   { querySet, beginningOfPassWriteIndex: 0, endOfPassWriteIndex: 1 },
});

pass.setPipeline(renderPipeline);
pass.setBindGroup(0, bindGroup);
pass.setVertexBuffer(0, vertexBuf, offset, size);
pass.setIndexBuffer(indexBuf, 'uint32', offset, size);  // 'uint16' | 'uint32'
pass.setViewport(x, y, width, height, minDepth, maxDepth);
pass.setScissorRect(x, y, width, height);
pass.setBlendConstant({ r:0, g:0, b:0, a:0 });
pass.setStencilReference(0);

// Draw calls
pass.draw(vertexCount, instanceCount, firstVertex, firstInstance);
pass.drawIndexed(indexCount, instanceCount, firstIndex, baseVertex, firstInstance);
pass.drawIndirect(indirectBuf, indirectOffset);
pass.drawIndexedIndirect(indirectBuf, indirectOffset);

// Occlusion queries
pass.beginOcclusionQuery(queryIndex);
pass.endOcclusionQuery();

pass.end();
```

### Compute Pass

```js
const compute = enc.beginComputePass({
    label: 'particle pass',
    timestampWrites: { querySet, beginningOfPassWriteIndex: 2, endOfPassWriteIndex: 3 },
});
compute.setPipeline(computePipeline);
compute.setBindGroup(0, computeBindGroup);
compute.dispatchWorkgroups(Math.ceil(N / 64));              // x, y, z (y,z default to 1)
compute.dispatchWorkgroupsIndirect(indirectBuf, offset);
compute.end();
```

### Copy Commands (on encoder, outside passes)

```js
enc.copyBufferToBuffer(src, srcOffset, dst, dstOffset, size);
enc.copyBufferToTexture(
    { buffer: src, offset: 0, bytesPerRow: 1920*4, rowsPerImage: 1080 },
    { texture: dst, mipLevel: 0, origin: {x:0,y:0,z:0} },
    { width: 1920, height: 1080, depthOrArrayLayers: 1 }
);
enc.copyTextureToBuffer(
    { texture: src, mipLevel: 0 },
    { buffer: dst, offset: 0, bytesPerRow: alignTo256(width*4), rowsPerImage: height },
    { width, height, depthOrArrayLayers: 1 }
);
enc.copyTextureToTexture(srcImageCopyTexture, dstImageCopyTexture, extent);
enc.clearBuffer(buffer, offset, size);

// Resolve timestamp/occlusion queries into buffer
enc.resolveQuerySet(querySet, firstQuery, queryCount, destinationBuffer, destinationOffset);

// Debug groups (visible in GPU frame captures)
enc.pushDebugGroup('draw sky');
// ...
enc.popDebugGroup();
enc.insertDebugMarker('frame start');

// Submit
const cmdBuf = enc.finish();
device.queue.submit([cmdBuf]);
```

---

## 10. GPUQueue

```js
device.queue.submit([commandBuffer]);
device.queue.writeBuffer(buffer, bufferOffset, data, dataOffset?, size?);
device.queue.writeTexture(destination, data, dataLayout, size);
device.queue.copyExternalImageToTexture(source, destination, copySize);

// Signal CPU that submitted work completed
await device.queue.onSubmittedWorkDone();
```

---

## 11. Canvas and Swap Chain

```js
// Browser / WebGPU canvas path
const canvas  = document.getElementById('canvas');
const context = canvas.getContext('webgpu');
const format  = navigator.gpu.getPreferredCanvasFormat();  // 'rgba8unorm' | 'bgra8unorm'

context.configure({
    device,
    format,
    alphaMode:   'opaque',    // 'opaque' | 'premultiplied'
    colorSpace:  'srgb',      // 'srgb' | 'display-p3'
    usage:       GPUTextureUsage.RENDER_ATTACHMENT,
});

// Per-frame:
const swapView = context.getCurrentTexture().createView();
// ... render into swapView ...
// No explicit present — browser presents at rAF callback return

// wgpu equivalent (native):
let surface_config = wgpu::SurfaceConfiguration {
    usage:  wgpu::TextureUsages::RENDER_ATTACHMENT,
    format: surface.get_capabilities(&adapter).formats[0],
    width, height,
    present_mode: wgpu::PresentMode::Fifo,   // Fifo | Immediate | Mailbox | AutoVsync
    alpha_mode:   wgpu::CompositeAlphaMode::Auto,
    ..Default::default()
};
surface.configure(&device, &surface_config);
let frame = surface.get_current_texture().unwrap();
let view  = frame.texture.create_view(&Default::default());
// ... render ...
frame.present();
```

---

## 12. Key Enums and Constants

### `GPUPrimitiveTopology`
`'point-list'` | `'line-list'` | `'line-strip'` | `'triangle-list'` | `'triangle-strip'`

### `GPUCullMode`
`'none'` | `'front'` | `'back'`

### `GPUFrontFace`
`'ccw'` (counter-clockwise, default) | `'cw'`

### `GPUBlendFactor`
`'zero'` | `'one'` | `'src'` | `'one-minus-src'` | `'src-alpha'` | `'one-minus-src-alpha'` | `'dst'` | `'one-minus-dst'` | `'dst-alpha'` | `'one-minus-dst-alpha'` | `'src-alpha-saturated'` | `'constant'` | `'one-minus-constant'`

### `GPUBlendOperation`
`'add'` | `'subtract'` | `'reverse-subtract'` | `'min'` | `'max'`

### `GPUStencilOperation`
`'keep'` | `'zero'` | `'replace'` | `'invert'` | `'increment-clamp'` | `'decrement-clamp'` | `'increment-wrap'` | `'decrement-wrap'`

### `GPULoadOp` / `GPUStoreOp`
Load: `'load'` | `'clear'`  
Store: `'store'` | `'discard'`

### `GPUMapMode`
`GPUMapMode.READ` (`0x1`) | `GPUMapMode.WRITE` (`0x2`)

### `GPUQueryType`
`'occlusion'` | `'timestamp'`

### `GPUPowerPreference`
`'low-power'` | `'high-performance'`

### `GPUTextureAspect`
`'all'` | `'stencil-only'` | `'depth-only'`

### `GPUTextureDimension`
`'1d'` | `'2d'` | `'3d'`

### `GPUTextureViewDimension`
`'1d'` | `'2d'` | `'2d-array'` | `'cube'` | `'cube-array'` | `'3d'`

### `GPUAddressMode`
`'clamp-to-edge'` | `'repeat'` | `'mirror-repeat'`

### `GPUFilterMode` / `GPUMipmapFilterMode`
`'nearest'` | `'linear'`

### `GPUFeatureName` (selected)

| Feature                             | Description                                   |
|-------------------------------------|-----------------------------------------------|
| `'depth-clip-control'`              | Disable depth clipping (`unclippedDepth`)     |
| `'depth32float-stencil8'`           | Combined format                               |
| `'texture-compression-bc'`          | BC1–7 formats (desktop)                       |
| `'texture-compression-etc2'`        | ETC2 formats (mobile/WebGL compat)            |
| `'texture-compression-astc'`        | ASTC formats                                  |
| `'timestamp-query'`                 | GPU timing via `GPUQuerySet`                  |
| `'indirect-first-instance'`         | `firstInstance` in indirect draws             |
| `'shader-f16'`                      | `f16` type in WGSL (`enable f16;`)            |
| `'rg11b10ufloat-renderable'`        | Render to `rg11b10ufloat`                     |
| `'float32-filterable'`              | Linear filter on `r32float`/`rg32float`       |
| `'dual-source-blending'`            | Two blend sources per fragment                |
| `'subgroups'`                       | WGSL subgroup operations (`enable subgroups;`)|
| `'bgra8unorm-storage'`              | Storage binding for `bgra8unorm`              |
| `'clip-distances'`                  | WGSL `clip_distances` built-in               |

---

## 13. Error Handling

```js
// Validation errors are surfaced via the error scope stack
device.pushErrorScope('validation');
// ... operations to validate ...
const err = await device.popErrorScope();
if (err) console.error('Validation:', err.message);

// Out-of-memory errors
device.pushErrorScope('out-of-memory');
const buf = device.createBuffer({ size: 1e12, usage: GPUBufferUsage.STORAGE });
const oom = await device.popErrorScope();
if (oom) console.error('OOM');

// Internal errors (driver bugs, etc.)
device.pushErrorScope('internal');
// ...
const internal = await device.popErrorScope();

// Uncaptured error handler (for errors outside any scope)
device.addEventListener('uncapturederror', e => console.error(e.error.message));
```

**`GPUError` subtypes:** `GPUValidationError` | `GPUOutOfMemoryError` | `GPUInternalError`

---

## Cross-References

- **Chapter 35** — Dawn: WebGPU C++ API (`webgpu.h`), the Dawn/Chrome implementation
- **Chapter 40** — Bevy/wgpu: `wgpu` Rust API, naga shader compiler, render graph abstraction
- **Chapter 98** — WebAssembly/WebGPU: `web-sys` WebGPU bindings generated from WebIDL, wgpu in WASM
- **Appendix P** — WGSL Language Reference: the shader language used with all WebGPU pipelines
- **Appendix Q** — Shader Language Comparison: WGSL vs GLSL/HLSL/MSL/OpenCL C

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
