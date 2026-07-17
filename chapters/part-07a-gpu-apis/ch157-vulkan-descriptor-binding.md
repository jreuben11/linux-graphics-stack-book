# Chapter 157: Vulkan Descriptor Binding in Depth

**Target audiences**: Vulkan application developers who need to go beyond tutorial-level descriptor set usage; engine developers designing bindless resource systems; and driver developers studying how descriptor binding maps to GPU hardware registers.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Descriptor Fundamentals](#descriptor-fundamentals)
3. [Descriptor Set Layout and Pool](#descriptor-set-layout-and-pool)
4. [Descriptor Updates: Write and Copy](#descriptor-updates-write-and-copy)
5. [Push Descriptors](#push-descriptors)
6. [VK_EXT_descriptor_indexing: Bindless Descriptors](#vk_ext_descriptor_indexing-bindless-descriptors)
7. [VK_EXT_descriptor_buffer: GPU-Side Descriptor Management](#vk_ext_descriptor_buffer-gpu-side-descriptor-management)
8. [Push Constants vs Descriptor Sets](#push-constants-vs-descriptor-sets)
9. [Driver Implementation: RADV and ANV](#driver-implementation-radv-and-anv)
10. [Descriptor Update Templates](#descriptor-update-templates)
11. [VK_EXT_mutable_descriptor_type](#vk_ext_mutable_descriptor_type)
12. [VK_EXT_inline_uniform_block](#vk_ext_inline_uniform_block)
13. [Frame-in-Flight Descriptor Management](#frame-in-flight-descriptor-management)
14. [Null Descriptors (VK_EXT_robustness2)](#null-descriptors-vk_ext_robustness2)
15. [NVK Descriptor Implementation](#nvk-descriptor-implementation)
16. [Acceleration Structure Descriptors](#acceleration-structure-descriptors)
17. [Roadmap](#roadmap)
18. [Integrations](#integrations)

---

## Introduction

Descriptors are Vulkan's mechanism for binding resources (buffers, images, samplers) to shaders. Understanding the descriptor system at depth — from layout to pool to update to GPU register mapping — is essential for writing high-performance Vulkan code and for avoiding the common pitfalls of descriptor set management.

This chapter covers:

- **Descriptor set lifecycle** — layout creation, pool allocation, set updates, and command-buffer binding
- **Update strategies** — performance properties of write-after-bind, push descriptors, and descriptor buffers
- **`VK_EXT_descriptor_indexing`** — enables bindless rendering (covered briefly in Ch154 from an application perspective)
- **`VK_EXT_descriptor_buffer`** — moves descriptor management entirely to GPU-visible memory
- **RADV and ANV implementation** — how these drivers implement descriptors in hardware

---

## Descriptor Fundamentals

### What a Descriptor Describes

A **descriptor** is a compact hardware-readable record that tells the GPU where to find a resource:

| Descriptor Type | Hardware Content |
|---|---|
| `UNIFORM_BUFFER` | GPU VA + size |
| `STORAGE_BUFFER` | GPU VA + size |
| `SAMPLED_IMAGE` | GPU image view handle |
| `STORAGE_IMAGE` | GPU image view handle + format |
| `SAMPLER` | GPU sampler state object |
| `COMBINED_IMAGE_SAMPLER` | Image view handle + sampler state |
| `ACCELERATION_STRUCTURE_KHR` | GPU VA of TLAS |

On AMD hardware (RADV), a `COMBINED_IMAGE_SAMPLER` is a 32-byte descriptor (16 bytes for the image + 16 bytes for the sampler). On Intel (ANV), a `SAMPLED_IMAGE` is 32 bytes (surface state).

### Descriptor Set Hierarchy

```
VkDescriptorSetLayout
  → describes binding slots (type, count, stage flags)

VkDescriptorPool
  → pre-allocates memory for N sets with M descriptors of each type

VkDescriptorSet
  → allocated from pool, backed by pool memory
  → populated by vkUpdateDescriptorSets
  → bound to pipeline via vkCmdBindDescriptorSets
```

### GLSL to Descriptor Mapping

```glsl
/* Shader: */
layout(set = 0, binding = 0) uniform CameraUBO {
    mat4 view_proj;
};
layout(set = 0, binding = 1) uniform sampler2D albedo_texture;
layout(set = 1, binding = 0) uniform ObjectUBO {
    mat4 model;
    vec4 color;
};
```

This maps to two descriptor set layouts:
- Set 0: binding 0 = UBO, binding 1 = combined image sampler
- Set 1: binding 0 = UBO

---

## Descriptor Set Layout and Pool

### VkDescriptorSetLayout Creation

```c
VkDescriptorSetLayoutBinding bindings[] = {
    {
        .binding         = 0,
        .descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        .descriptorCount = 1,
        .stageFlags      = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
    },
    {
        .binding         = 1,
        .descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        .descriptorCount = 1,
        .stageFlags      = VK_SHADER_STAGE_FRAGMENT_BIT,
    },
};

VkDescriptorSetLayoutCreateInfo layout_ci = {
    .sType        = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
    .bindingCount = ARRAY_SIZE(bindings),
    .pBindings    = bindings,
};
vkCreateDescriptorSetLayout(device, &layout_ci, NULL, &set_layout);
```

### Descriptor Pool Sizing

The pool must be pre-sized correctly — underestimating causes `VK_ERROR_OUT_OF_POOL_MEMORY`:

```c
VkDescriptorPoolSize pool_sizes[] = {
    { VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,          100 },
    { VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,  200 },
    { VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,           50 },
};

VkDescriptorPoolCreateInfo pool_ci = {
    .sType         = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO,
    .maxSets       = 50,    /* maximum simultaneously allocated sets */
    .poolSizeCount = ARRAY_SIZE(pool_sizes),
    .pPoolSizes    = pool_sizes,
    /* VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT: allow individual free */
};
vkCreateDescriptorPool(device, &pool_ci, NULL, &pool);
```

Without `FREE_DESCRIPTOR_SET_BIT`, the only way to reclaim pool memory is `vkResetDescriptorPool` (frees all sets).

### Allocation

```c
VkDescriptorSetAllocateInfo alloc_info = {
    .sType              = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
    .descriptorPool     = pool,
    .descriptorSetCount = 1,
    .pSetLayouts        = &set_layout,
};
VkDescriptorSet set;
VkResult r = vkAllocateDescriptorSets(device, &alloc_info, &set);
/* r may be VK_ERROR_OUT_OF_POOL_MEMORY if pool is exhausted */
```

---

## Descriptor Updates: Write and Copy

### vkUpdateDescriptorSets

```c
VkDescriptorBufferInfo buf_info = {
    .buffer = uniform_buffer,
    .offset = 0,
    .range  = sizeof(CameraUBO),
};

VkDescriptorImageInfo img_info = {
    .sampler     = sampler,
    .imageView   = albedo_view,
    .imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
};

VkWriteDescriptorSet writes[] = {
    {
        .sType           = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET,
        .dstSet          = set,
        .dstBinding      = 0,
        .dstArrayElement = 0,
        .descriptorCount = 1,
        .descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        .pBufferInfo     = &buf_info,
    },
    {
        .sType           = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET,
        .dstSet          = set,
        .dstBinding      = 1,
        .dstArrayElement = 0,
        .descriptorCount = 1,
        .descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        .pImageInfo      = &img_info,
    },
};
vkUpdateDescriptorSets(device, ARRAY_SIZE(writes), writes, 0, NULL);
```

### Update-After-Bind

Standard descriptor sets cannot be updated while in use on the GPU. The `VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT` flag (from `VK_EXT_descriptor_indexing`) removes this restriction:

```c
VkDescriptorBindingFlags flags =
    VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT |
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT;

VkDescriptorSetLayoutBindingFlagsCreateInfo flags_ci = {
    .sType         = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_BINDING_FLAGS_CREATE_INFO,
    .bindingCount  = 1,
    .pBindingFlags = &flags,
};

/* Attach to layout create info via pNext: */
layout_ci.pNext = &flags_ci;
/* Pool must also have UPDATE_AFTER_BIND_BIT: */
pool_ci.flags |= VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT;
```

With update-after-bind, descriptors in a set that is bound but not yet consumed by the GPU can be updated. Essential for streaming texture atlases and bindless systems.

---

## Push Descriptors

### VK_KHR_push_descriptor

Push descriptors eliminate the pool/set allocation lifecycle. Descriptors are written directly into the command buffer:

```c
/* Requires pipeline layout created with push descriptor flag: */
VkDescriptorSetLayoutCreateInfo layout_ci = {
    .flags = VK_DESCRIPTOR_SET_LAYOUT_CREATE_PUSH_DESCRIPTOR_BIT_KHR,
    /* ... */
};

/* Per-draw update — no vkAllocateDescriptorSets needed: */
vkCmdPushDescriptorSetKHR(cmd,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipeline_layout,
    0,               /* set index */
    1,               /* write count */
    &write);         /* same VkWriteDescriptorSet as above */
```

Push descriptors are ideal for per-draw data that changes every draw call. The Vulkan spec guarantees the data is embedded in the command buffer, so no pool management is needed.

**Performance note**: Push descriptors have higher per-draw CPU cost than bindless (one CPU write per descriptor per draw) but lower than managing pools for small counts. RADV and ANV implement push descriptors by allocating a small GPU buffer from an internal ring and writing descriptors there at `vkCmdPushDescriptorSetKHR` time.

---

## VK_EXT_descriptor_indexing: Bindless Descriptors

### Enabling Variable-Count Descriptor Arrays

```c
VkPhysicalDeviceDescriptorIndexingFeatures di = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DESCRIPTOR_INDEXING_FEATURES,
    .shaderSampledImageArrayNonUniformIndexing = VK_TRUE,
    .runtimeDescriptorArray                    = VK_TRUE,
    .descriptorBindingVariableDescriptorCount  = VK_TRUE,
    .descriptorBindingPartiallyBound           = VK_TRUE,
    .descriptorBindingUpdateAfterBind          = VK_TRUE,
};
```

### The Bindless Texture Array

```c
VkDescriptorSetLayoutBinding texture_binding = {
    .binding         = 0,
    .descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
    .descriptorCount = 65536,  /* max textures; array is partially bound */
    .stageFlags      = VK_SHADER_STAGE_ALL_GRAPHICS,
};

VkDescriptorBindingFlags texture_flags =
    VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT |
    VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT |
    VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT;
```

### Shader Access

```glsl
#version 460
#extension GL_EXT_nonuniform_qualifier       : require
#extension GL_EXT_descriptor_indexing        : require

layout(set = 0, binding = 0)
uniform sampler2D textures[];   /* runtime-sized array */

layout(location = 0) in flat uint tex_id;
layout(location = 1) in vec2 uv;

void main() {
    /* Non-uniform index: different invocations in same wave may use different indices */
    vec4 color = texture(textures[nonuniformEXT(tex_id)], uv);
    fragColor = color;
}
```

**Slang equivalent** — `[[vk::binding(N,M)]]` makes the set/binding assignment explicit and compiler-verified; `NonUniformResourceIndex()` replaces the `nonuniformEXT` qualifier without requiring any GLSL extension declaration.

```slang
// File: bindless_texture_array.slang
// Slang improvements:
// - [[vk::binding(0,0)]] pins set and binding at the declaration site; no implicit numbering
// - NonUniformResourceIndex() is a built-in; no GL_EXT_nonuniform_qualifier extension needed
// - nointerpolation on the uint attribute explicitly marks flat interpolation, matching GLSL flat

[[vk::binding(0, 0)]]
Sampler2D textures[];                          // runtime-sized bindless array (set 0, binding 0)

[shader("fragment")]
float4 main(
    [[vk::location(0)]] nointerpolation uint  tex_id : TEXCOORD0,
    [[vk::location(1)]]              float2   uv     : TEXCOORD1
) : SV_Target
{
    // NonUniformResourceIndex signals wave-divergent access to the compiler;
    // equivalent to nonuniformEXT but is a language built-in, not an extension toggle
    return textures[NonUniformResourceIndex(tex_id)].Sample(uv);
}
```

`nonuniformEXT` instructs the compiler to emit a scalar or SIMD-divergent texture fetch (e.g. AMD's `image_sample_nsa` or Intel's per-channel send), which avoids incorrect results when lane divergence causes different lanes to access different descriptors.

---

## VK_EXT_descriptor_buffer: GPU-Side Descriptor Management

### Motivation

Traditional descriptors live in driver-managed memory. `VK_EXT_descriptor_buffer` (Vulkan 1.3 extension) exposes descriptors as raw bytes in application-managed buffers:

- Descriptors are written directly to a `VkBuffer` in application code
- The GPU reads descriptors from that buffer (base address + offset)
- No pool, no set allocation, no `vkUpdateDescriptorSets`

### Usage Pattern

```c
/* Get descriptor size for this device: */
VkPhysicalDeviceDescriptorBufferPropertiesEXT props = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DESCRIPTOR_BUFFER_PROPERTIES_EXT };
vkGetPhysicalDeviceProperties2(phys_dev, &(VkPhysicalDeviceProperties2){
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
    .pNext = &props });

size_t sampler_size = props.samplerDescriptorSize;           /* e.g. 16 bytes on AMD */
size_t sampled_image_size = props.sampledImageDescriptorSize; /* e.g. 16 bytes on AMD */

/* Allocate descriptor buffer (GPU-visible): */
VkBuffer desc_buf;
VkBufferCreateInfo desc_buf_ci = {
    .size  = 65536 * (sampler_size + sampled_image_size),
    .usage = VK_BUFFER_USAGE_RESOURCE_DESCRIPTOR_BUFFER_BIT_EXT
           | VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT,
};
vkCreateBuffer(device, &desc_buf_ci, NULL, &desc_buf);

/* Map the descriptor buffer: */
void *desc_ptr;
vkMapMemory(device, desc_mem, 0, VK_WHOLE_SIZE, 0, &desc_ptr);

/* Write a combined image sampler descriptor: */
VkDescriptorImageInfo img_info = { .sampler = samp, .imageView = view,
    .imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL };
VkDescriptorGetInfoEXT get_info = {
    .sType = VK_STRUCTURE_TYPE_DESCRIPTOR_GET_INFO_EXT,
    .type  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
    .data.pCombinedImageSampler = &img_info,
};
vkGetDescriptorEXT(device, &get_info,
    sampler_size + sampled_image_size,   /* descriptor size */
    (uint8_t*)desc_ptr + tex_index * stride);  /* write at offset */
```

### Binding Descriptor Buffer

```c
VkDescriptorBufferBindingInfoEXT binding = {
    .sType   = VK_STRUCTURE_TYPE_DESCRIPTOR_BUFFER_BINDING_INFO_EXT,
    .address = desc_buf_gpu_va,  /* vkGetBufferDeviceAddress result */
    .usage   = VK_BUFFER_USAGE_RESOURCE_DESCRIPTOR_BUFFER_BIT_EXT,
};
vkCmdBindDescriptorBuffersEXT(cmd, 1, &binding);

uint32_t buf_index = 0;
VkDeviceSize offset = 0;
vkCmdSetDescriptorBufferOffsetsEXT(cmd,
    VK_PIPELINE_BIND_POINT_GRAPHICS,
    pipeline_layout,
    0,            /* set = 0 */
    1,            /* 1 set */
    &buf_index,   /* which descriptor buffer */
    &offset);     /* byte offset within descriptor buffer */
```

`VK_EXT_descriptor_buffer` removes pool fragmentation and enables CPU-side descriptor prefetching. On RADV, it maps directly to the hardware's 32-byte SGPRs that point to descriptor arrays.

---

## Push Constants vs Descriptor Sets

### Push Constants

Push constants are the fastest path for small, per-draw data:

```c
/* Max size: vkPhysicalDeviceLimits.maxPushConstantsSize (min 128 bytes, typically 256) */
struct PushConst {
    mat4  mvp;         /* 64 bytes */
    vec4  color;       /* 16 bytes */
    uint  tex_id;      /* 4 bytes */
    uint  _pad[3];     /* 12 bytes */
};  /* 96 bytes total */

vkCmdPushConstants(cmd, pipeline_layout,
    VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_FRAGMENT_BIT,
    0, sizeof(PushConst), &push_data);
```

In hardware, push constants live in SGPRs (AMD) or bound constant registers (Intel). On RADV, push constants are stored in a small buffer and loaded via `s_load_dwordx4` at shader start. They are the fastest CPU→GPU data path and should be preferred for all data ≤ 128 bytes.

### Choosing the Right Binding Strategy

| Approach | Cost | Best For |
|---|---|---|
| Push constants | Zero setup; per-draw writes | Per-draw scalars (MVP, material index) |
| Push descriptors | One CPU write per descriptor | Per-draw resources |
| Descriptor sets (preallocated) | Set allocation overhead | Per-material or per-frame data |
| Bindless (descriptor indexing) | Layout setup only | All textures in one set |
| Descriptor buffers | Raw memory writes | High-frequency streaming updates |

---

## Driver Implementation: RADV and ANV

### RADV Descriptor Sets

RADV stores descriptor sets in `BO` (buffer objects) allocated from the device heap. Each descriptor set layout has a fixed size computed from binding types and counts:

```c
/* radv/radv_descriptor_set.c */
void radv_descriptor_set_create(struct radv_device *device,
    struct radv_descriptor_pool *pool,
    const struct radv_descriptor_set_layout *layout,
    struct radv_descriptor_set **out_set)
{
    /* Allocate from pool's BO: */
    set->mapped_ptr = pool->mapped_ptr + pool->current_offset;
    pool->current_offset += align(layout->size, 64);

    /* For each binding, write zeroes initially: */
    memset(set->mapped_ptr, 0, layout->size);
}
```

RADV descriptors are 32 bytes (image/sampler) or 16 bytes (buffer), matching AMD GCN hardware's resource descriptor format (4 dwords / 8 dwords).

```c
/* AMD image descriptor (32 bytes, T# in GCN nomenclature): */
struct radv_image_descriptor {
    uint64_t base_va;       /* bits [39:8] shifted */
    uint32_t word1;         /* width, height */
    uint32_t word2;         /* depth, array */
    uint32_t word3;         /* format, type */
    uint32_t word4;         /* mip levels, filtering */
    uint32_t word5;         /* base_level, last_level */
    uint32_t word6;         /* min_lod, max_lod */
    uint32_t word7;         /* metadata */
};
```

### ANV Descriptor Sets

ANV (Intel) stores descriptors in a surface state heap:

```c
/* anv/anv_descriptor_set.c */
struct anv_descriptor_set {
    struct anv_bo *bo;          /* descriptor BO */
    void         *mapped_ptr;
    /* Surface state entries reference anv_state_pool (surface state heap): */
    struct anv_state surface_states;
};
```

On Intel Gen12+, descriptors are bindless surface states stored in the shader's binding table. ANV's bindless path uses `EXT_descriptor_indexing` to put all surface states in one binding table slot 0, indexed by the shader.

### Non-Uniform Index: Driver Handling

When GLSL uses `nonuniformEXT(idx)`, the compiler emits Vulkan SPIR-V with `NonUniform` decoration:

```spirv
%tex = OpLoad %sampler2D %textures_arr
; NonUniform means different invocations may have different idx values:
%idx_nonuniform = OpCopyObject %uint %idx
OpDecorate %idx_nonuniform NonUniform

%sampled = OpImageSampleImplicitLod %v4f32 %tex %uv
```

RADV's ACO backend handles NonUniform by emitting a waterfall loop: it executes the texture fetch once per unique descriptor index value across the wave, masking inactive lanes. Intel's compiler uses a scalar send instruction when the hardware supports it.

---

## Descriptor Update Templates

### Motivation

`vkUpdateDescriptorSets` accepts an array of `VkWriteDescriptorSet` structs, each with its own `sType`, `pNext`, and metadata fields. For hot paths — materials updated per draw, bindless streaming slots refreshed every frame — the overhead of parsing these structs in the driver is measurable. `VkDescriptorUpdateTemplate` (promoted to Vulkan 1.1 core) solves this by pre-baking the mapping from a tightly-packed application struct to a set of descriptor writes. At update time, the driver performs a single dispatch through the pre-compiled template rather than iterating a heterogeneous write array. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCreateDescriptorUpdateTemplate.html)

### The Template Lifecycle

```c
/* Describe how entries map from the tightly-packed source struct: */
VkDescriptorUpdateTemplateEntry entries[] = {
    {
        .dstBinding      = 0,
        .dstArrayElement = 0,
        .descriptorCount = 1,
        .descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
        .offset          = offsetof(MyDescData, buf_info),   /* offset in pData */
        .stride          = sizeof(MyDescData),               /* stride between array elements */
    },
    {
        .dstBinding      = 1,
        .dstArrayElement = 0,
        .descriptorCount = 1,
        .descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER,
        .offset          = offsetof(MyDescData, img_info),
        .stride          = sizeof(MyDescData),
    },
};

VkDescriptorUpdateTemplateCreateInfo tmpl_ci = {
    .sType                      = VK_STRUCTURE_TYPE_DESCRIPTOR_UPDATE_TEMPLATE_CREATE_INFO,
    .pNext                      = NULL,
    .flags                      = 0,
    .descriptorUpdateEntryCount = ARRAY_SIZE(entries),
    .pDescriptorUpdateEntries   = entries,
    .templateType               = VK_DESCRIPTOR_UPDATE_TEMPLATE_TYPE_DESCRIPTOR_SET,
    .descriptorSetLayout        = set_layout,
    .pipelineBindPoint          = 0,      /* unused for DESCRIPTOR_SET type */
    .pipelineLayout             = VK_NULL_HANDLE,
    .set                        = 0,
};

VkDescriptorUpdateTemplate tmpl;
vkCreateDescriptorUpdateTemplate(device, &tmpl_ci, NULL, &tmpl);
```

### Using the Template

The application packs its per-frame resource handles into one contiguous struct and passes a raw pointer:

```c
/* Tightly-packed source struct — no VkStructureType overhead: */
typedef struct {
    VkDescriptorBufferInfo buf_info;
    VkDescriptorImageInfo  img_info;
} MyDescData;

MyDescData data = {
    .buf_info = { .buffer = ubo, .offset = 0, .range = sizeof(CameraUBO) },
    .img_info = { .sampler = samp, .imageView = view,
                  .imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL },
};

/* Single call — driver reads from data using the pre-baked template: */
vkUpdateDescriptorSetWithTemplate(device, descriptor_set, tmpl, &data);
```

The `offset` and `stride` fields in each `VkDescriptorUpdateTemplateEntry` index into the raw `pData` pointer using byte arithmetic — no Vulkan struct headers needed. The driver (RADV, ANV) compiles the template entries at `vkCreateDescriptorUpdateTemplate` time into a compact internal representation, reducing per-update dispatch overhead.

**DXVK and RADV use**: DXVK uses `vkUpdateDescriptorSetWithTemplate` for all its per-draw descriptor updates since it tracks D3D11 state changes as tightly-packed arrays, matching the template model exactly. RADV implements the template path in `radv_descriptor_set.c` using `vk_descriptor_update_template` from Mesa's common Vulkan runtime layer (`src/vulkan/runtime/vk_descriptor_update_template.c`).

---

## VK_EXT_mutable_descriptor_type

### Motivation: D3D12 Volatile Descriptor Heaps

D3D12 uses descriptor heaps where a single slot can hold an SRV, UAV, or CBV depending on the root signature. When DXVK and VKD3D-Proton translate D3D12 to Vulkan, they need descriptor slots whose type can change without rebuilding the pool. `VK_EXT_mutable_descriptor_type` (initially `VK_VALVE_mutable_descriptor_type`, now EXT and promoted toward core) provides exactly this. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_mutable_descriptor_type.html)

### API Structure

Each binding declared as `VK_DESCRIPTOR_TYPE_MUTABLE_EXT` is accompanied by a `VkMutableDescriptorTypeListEXT` that enumerates the concrete types it may hold at any given time:

```c
/* Enumerate which concrete types each mutable binding may hold: */
VkDescriptorType mutable_types[] = {
    VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE,
    VK_DESCRIPTOR_TYPE_STORAGE_IMAGE,
    VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,
    VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,
    VK_DESCRIPTOR_TYPE_UNIFORM_TEXEL_BUFFER,
    VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER,
};

VkMutableDescriptorTypeListEXT mutable_list = {
    .descriptorTypeCount = ARRAY_SIZE(mutable_types),
    .pDescriptorTypes    = mutable_types,
};

VkMutableDescriptorTypeCreateInfoEXT mutable_ci = {
    .sType                         = VK_STRUCTURE_TYPE_MUTABLE_DESCRIPTOR_TYPE_CREATE_INFO_EXT,
    .pNext                         = NULL,
    .mutableDescriptorTypeListCount = 1,
    .pMutableDescriptorTypeLists    = &mutable_list,
};

/* Pool declares mutable capacity using MUTABLE as the pool type: */
VkDescriptorPoolSize pool_sizes[] = {
    { VK_DESCRIPTOR_TYPE_MUTABLE_EXT, 65536 },
};

VkDescriptorPoolCreateInfo pool_ci = {
    .sType         = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO,
    .pNext         = &mutable_ci,    /* type list chained here */
    .flags         = VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT,
    .maxSets       = 1,
    .poolSizeCount = ARRAY_SIZE(pool_sizes),
    .pPoolSizes    = pool_sizes,
};

VkDescriptorPool pool;
vkCreateDescriptorPool(device, &pool_ci, NULL, &pool);
```

The layout binding similarly declares the mutable type with the same `VkMutableDescriptorTypeCreateInfoEXT` pNext chain on `VkDescriptorSetLayoutCreateInfo`.

### Type Switch at Update Time

At update time the caller specifies the actual concrete type in the `VkWriteDescriptorSet.descriptorType` field. The driver sizes each mutable slot to the maximum hardware descriptor size across all listed types, so no reallocation occurs on type switch:

```c
/* Update slot 42 as a sampled image: */
VkWriteDescriptorSet write = {
    .sType           = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET,
    .dstSet          = descriptor_set,
    .dstBinding      = 0,
    .dstArrayElement = 42,
    .descriptorCount = 1,
    .descriptorType  = VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE,  /* concrete type */
    .pImageInfo      = &img_info,
};
vkUpdateDescriptorSets(device, 1, &write, 0, NULL);

/* Later, reuse same slot as a uniform buffer — same pool, no realloc: */
write.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
write.pBufferInfo    = &buf_info;
write.pImageInfo     = NULL;
vkUpdateDescriptorSets(device, 1, &write, 0, NULL);
```

**DXVK and VKD3D-Proton**: Both translation layers use `VK_EXT_mutable_descriptor_type` to emulate D3D12's descriptor heap model. VKD3D-Proton's bindless heap implementation allocates one large mutable descriptor pool per heap type and maps D3D12 `SetGraphicsRootDescriptorTable` calls to offset arithmetic within it, eliminating the per-type pool management that would otherwise be required. [Source](https://github.com/doitsujin/dxvk)

RADV has supported this extension since Mesa 22.2 and sizes mutable slots at 32 bytes (the AMD image descriptor T# size, which is the largest concrete type on GCN/RDNA hardware).

---

## VK_EXT_inline_uniform_block

### Motivation

Uniform buffers require a `VkBuffer` allocation, memory binding, and a descriptor that points to it. For small per-draw constants (a material index, a push-like struct, a small parameter block) this round-trip through the allocator is wasteful. `VK_EXT_inline_uniform_block` (promoted to Vulkan 1.3 core) embeds the uniform data directly inside the descriptor set, eliminating the backing buffer. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_inline_uniform_block.html)

### Capacity Limits

```c
VkPhysicalDeviceInlineUniformBlockProperties iub_props = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_INLINE_UNIFORM_BLOCK_PROPERTIES,
};
/* Query via VkPhysicalDeviceProperties2 pNext chain: */
vkGetPhysicalDeviceProperties2(phys_dev, &(VkPhysicalDeviceProperties2){
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
    .pNext = &iub_props,
});
/* iub_props.maxInlineUniformBlockSize: minimum 256 bytes guaranteed by spec */
/* Typical values: 256 bytes (RADV, ANV), 4096 bytes (some mobile GPUs) */
```

### Descriptor Set Layout and Pool

For inline uniform blocks, `descriptorCount` in the binding is the **byte size** of the block (must be a multiple of 4), not the number of descriptors:

```c
VkDescriptorSetLayoutBinding iub_binding = {
    .binding         = 0,
    .descriptorType  = VK_DESCRIPTOR_TYPE_INLINE_UNIFORM_BLOCK,
    .descriptorCount = sizeof(MaterialParams),  /* byte size, multiple of 4 */
    .stageFlags      = VK_SHADER_STAGE_FRAGMENT_BIT,
};

/* Pool also needs inline uniform block capacity declared in bytes: */
VkDescriptorPoolInlineUniformBlockCreateInfo iub_pool_ci = {
    .sType                         = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_INLINE_UNIFORM_BLOCK_CREATE_INFO,
    .maxInlineUniformBlockBindings = 16,
};
/* Chain via pNext on VkDescriptorPoolCreateInfo */
```

### Writing Inline Data

```c
MaterialParams params = { .roughness = 0.4f, .metallic = 1.0f, .tex_id = 7 };

VkWriteDescriptorSetInlineUniformBlock iub_write = {
    .sType    = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET_INLINE_UNIFORM_BLOCK,
    .pNext    = NULL,
    .dataSize = sizeof(params),
    .pData    = &params,
};

VkWriteDescriptorSet write = {
    .sType           = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET,
    .pNext           = &iub_write,        /* chain the inline data here */
    .dstSet          = descriptor_set,
    .dstBinding      = 0,
    .dstArrayElement = 0,
    .descriptorCount = sizeof(params),    /* byte count, not descriptor count */
    .descriptorType  = VK_DESCRIPTOR_TYPE_INLINE_UNIFORM_BLOCK,
    .pBufferInfo     = NULL,
};
vkUpdateDescriptorSets(device, 1, &write, 0, NULL);
```

### GLSL Binding

```glsl
/* layout binding matches the C-side byte-count binding: */
layout(set = 0, binding = 0) uniform MaterialBlock {
    float roughness;
    float metallic;
    uint  tex_id;
} material;

/* textures[] and uv declared elsewhere (set 1, bindless array + vertex interp): */
layout(set = 1, binding = 0) uniform sampler2D textures[];
layout(location = 0) in vec2 uv;

void main() {
    vec4 col = texture(textures[nonuniformEXT(material.tex_id)], uv);
    /* apply PBR with material.roughness, material.metallic ... */
}
```

**Slang equivalent** — `ParameterBlock<T>` groups the material constants and the bindless texture array into a single typed descriptor set without any manual `layout(set=N, binding=M)` annotations; the inline-uniform-block distinction disappears because ordinary struct fields auto-pack into a generated constant buffer at binding 0.

```slang
// File: material_bindless_paramblock.slang
// Slang improvements:
// - ParameterBlock<MaterialSet> owns its own descriptor set; Slang assigns set and binding
//   indices automatically — no hand-written layout() qualifiers required
// - Ordinary data fields (roughness, metallic, tex_id) collapse into an auto-generated
//   ConstantBuffer at binding 0, replacing the inline uniform block concept entirely
// - Resource fields (textures[]) follow at binding 1; unbounded array must be the last field

struct MaterialParams {
    float roughness;
    float metallic;
    uint  tex_id;
};

struct MaterialSet {
    MaterialParams material;   // auto-packed into ConstantBuffer at binding 0
    Sampler2D      textures[]; // bindless array at binding 1; must be last field in set
};

// Slang assigns a descriptor set number; no explicit set index needed
ParameterBlock<MaterialSet> gMaterial;

[shader("fragment")]
float4 main([[vk::location(0)]] float2 uv : TEXCOORD0) : SV_Target
{
    MaterialParams mat = gMaterial.material;
    // NonUniformResourceIndex handles divergent lane access without extension declarations
    float4 col = gMaterial.textures[NonUniformResourceIndex(mat.tex_id)].Sample(uv);
    // Apply PBR using mat.roughness and mat.metallic …
    return col;
}
```

On RADV, inline uniform block data is stored directly in the pool's BO at the set's memory offset, adjacent to other descriptor data. The driver maps the binding's byte range directly into GPU-accessible memory, meaning no buffer object, no VMA allocation, and no device-address resolution at bind time.

---

## Frame-in-Flight Descriptor Management

### The Hazard

A descriptor set updated on the CPU while the GPU is still reading it in a previous frame's command buffer causes undefined behaviour. Standard Vulkan synchronisation (fences, semaphores) only signals when a frame completes; it does not prevent the next frame from overwriting descriptors before the GPU is done reading them. The correct pattern is to maintain N independent copies of mutable descriptor state — one per frame-in-flight slot. [Source: Vulkan-Samples descriptor_management](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/descriptor_management)

### N-Sets-Per-N-Frames Pattern

```c
#define FRAMES_IN_FLIGHT 3

typedef struct {
    VkDescriptorPool pool;
    VkDescriptorSet  set;
    VkFence          fence;      /* signals when this frame's cmd buf completes */
} FrameResources;

FrameResources frames[FRAMES_IN_FLIGHT];
uint32_t       frame_index = 0;

void init_frame_resources(VkDevice device, VkDescriptorSetLayout layout) {
    for (uint32_t i = 0; i < FRAMES_IN_FLIGHT; i++) {
        /* One pool per frame — reset the entire pool on reuse: */
        VkDescriptorPoolSize sizes[] = {
            { VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER,         4 },
            { VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER, 8 },
        };
        VkDescriptorPoolCreateInfo pool_ci = {
            .sType         = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO,
            /* NOTE: no FREE_DESCRIPTOR_SET_BIT — pool-reset is cheaper */
            .maxSets       = 4,
            .poolSizeCount = ARRAY_SIZE(sizes),
            .pPoolSizes    = sizes,
        };
        vkCreateDescriptorPool(device, &pool_ci, NULL, &frames[i].pool);

        VkDescriptorSetAllocateInfo alloc = {
            .sType              = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO,
            .descriptorPool     = frames[i].pool,
            .descriptorSetCount = 1,
            .pSetLayouts        = &layout,
        };
        vkAllocateDescriptorSets(device, &alloc, &frames[i].set);

        VkFenceCreateInfo fence_ci = {
            .sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO,
            .flags = VK_FENCE_CREATE_SIGNALED_BIT,   /* pre-signaled so first wait succeeds */
        };
        vkCreateFence(device, &fence_ci, NULL, &frames[i].fence);
    }
}

void begin_frame(VkDevice device) {
    FrameResources *fr = &frames[frame_index % FRAMES_IN_FLIGHT];

    /* Wait for this slot's previous submission to finish: */
    vkWaitForFences(device, 1, &fr->fence, VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &fr->fence);

    /* Safe to reset the pool now — GPU is done reading this slot's descriptors: */
    vkResetDescriptorPool(device, fr->pool, 0);

    /* Re-allocate and update descriptors for this frame: */
    /* ... vkAllocateDescriptorSets, vkUpdateDescriptorSets ... */
}
```

### Pool-Reset vs FREE_DESCRIPTOR_SET_BIT

`VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT` allows `vkFreeDescriptorSets` on individual sets, but the driver must then maintain a free-list inside the pool, adding per-allocation overhead. For the frame-in-flight pattern, `vkResetDescriptorPool` is strictly faster: it frees all sets atomically in O(1) by resetting the pool's offset pointer, with no free-list management.

**Timeline semaphore alternative**: With `VK_KHR_timeline_semaphore` (Vulkan 1.2 core), applications can gate descriptor writes on a semaphore value rather than a fence, enabling finer-grained overlap between CPU descriptor preparation and GPU execution in long pipelines.

---

## Null Descriptors (VK_EXT_robustness2)

### Motivation

By default, Vulkan defines access through a null (zeroed) descriptor or an out-of-bounds descriptor as undefined behaviour — the GPU may read garbage, fault, or hang. In sparse and streaming scenes it is convenient to leave descriptor slots empty (backed by `VK_NULL_HANDLE`) and have unbound slots return zero safely. `VK_EXT_robustness2` adds `nullDescriptor` support: when enabled, reading a null descriptor returns zero values, and out-of-bounds buffer reads return zero or clamp. [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_robustness2.html)

### Enabling the Feature

```c
VkPhysicalDeviceRobustness2FeaturesEXT robustness2 = {
    .sType               = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_ROBUSTNESS_2_FEATURES_EXT,
    .pNext               = NULL,
    .robustBufferAccess2 = VK_FALSE,   /* tighter bounds checking for buffers */
    .robustImageAccess2  = VK_FALSE,   /* tighter bounds checking for images */
    .nullDescriptor      = VK_TRUE,    /* enable null descriptor safe reads */
};

VkDeviceCreateInfo device_ci = {
    /* ... */
    .pNext = &robustness2,
};
vkCreateDevice(phys_dev, &device_ci, NULL, &device);
```

### Null Descriptor Behaviour

With `nullDescriptor` enabled:

- A `VkBuffer` or `VkImageView` of `VK_NULL_HANDLE` can be written into a descriptor (previously validation error).
- Reads from a null `SAMPLED_IMAGE` or `STORAGE_IMAGE` descriptor return `(0, 0, 0, 0)`.
- Reads from a null `UNIFORM_BUFFER` or `STORAGE_BUFFER` descriptor return `0` for all components.
- Reads from a null `UNIFORM_TEXEL_BUFFER` or `STORAGE_TEXEL_BUFFER` return `(0, 0, 0, 0)`.
- Writes to a null storage descriptor are discarded safely.
- Buffer device address (BDA) binds of address `0` are also well-defined: reads return `0`.

### Use Cases

**Sparse scene streaming**: A texture streaming system pre-allocates a bindless array of 65536 combined image sampler slots. Streaming logic fills slots asynchronously as textures are decoded. Until a slot is populated, it remains null; shaders read `(0, 0, 0, 1)` (black, opaque), producing a visible but non-crashing fallback.

**Conditional binding**: A material system that optionally binds a normal map can leave the normal-map slot null when not present and branch in the shader (`if (normal_tex_id == 0xFFFF)`), with the null read providing a safe fallback.

DXVK and VKD3D-Proton both require `nullDescriptor` because D3D11/D3D12 explicitly allow null resource binds with defined zero-return semantics. RADV exposes `nullDescriptor` on all supported hardware by zeroing the relevant hardware descriptor words (image base VA = 0 and size = 0 on AMD GCN/RDNA produces a hardware zero-return for out-of-range addresses).

---

## NVK Descriptor Implementation

### GPU-VA-Based Descriptor Model

NVK (the open-source NVIDIA Vulkan driver in Mesa) uses a descriptor model where descriptor sets are backed directly by `nvkmd_mem` — GPU-visible BO memory managed through the `nvkmd` abstraction layer. Rather than a driver-side allocation scheme separate from GPU memory, each descriptor pool allocates one `struct nvkmd_mem *` slab and sub-allocates sets from it via a `util_vma_heap`. Descriptors are written as raw bytes into a CPU-mapped pointer (`set->map`), and `nvkmd_mem_sync_map_to_gpu()` flushes dirty ranges to GPU-accessible memory. [Source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/nouveau/vulkan/nvk_descriptor_set.c)

### Key Structs

```c
/* From src/nouveau/vulkan/nvk_descriptor_set.h */
struct nvk_descriptor_pool {
   struct vk_object_base   base;
   struct list_head        sets;
   uint64_t                mem_size_B;
   struct nvkmd_mem       *mem;        /* single GPU-visible slab */
   void                   *host_mem;   /* CPU-mapped pointer into slab */
   struct util_vma_heap    heap;       /* sub-allocator within slab */
};

struct nvk_descriptor_set {
   struct vk_object_base           base;
   struct nvk_descriptor_pool     *pool;
   struct list_head                link;
   struct nvk_descriptor_set_layout *layout;
   void                           *map;          /* CPU pointer into pool's slab */
   uint64_t                        mem_offset_B; /* offset within pool's slab */
   uint32_t                        size_B;
   union nvk_buffer_descriptor     dynamic_buffers[]; /* flexible array */
};
```

The GPU address of a descriptor set is computed as:

```c
static inline struct nvk_buffer_address
nvk_descriptor_set_addr(const struct nvk_descriptor_set *set)
{
   return (struct nvk_buffer_address) {
      .base_addr = set->pool->mem->va->addr + set->mem_offset_B,
      .size      = set->size_B,
   };
}
```

This exposes the TLAS-style VA directly to the pipeline, matching NVIDIA hardware's native descriptor model where the GPU accesses descriptors through a 64-bit base address in constant memory.

### Descriptor Write Path

Writes use a `nvk_descriptor_writer` helper that tracks a dirty byte range and calls `nvkmd_mem_sync_map_to_gpu()` only for changed regions:

```c
/* Simplified from nvk_descriptor_set.c::nvk_descriptor_writer_finish() */
static void nvk_descriptor_writer_finish(struct nvk_descriptor_writer *w) {
   if (w->set != NULL && w->set->pool->mem != NULL &&
       w->dirty_start < w->dirty_end) {
      nvkmd_mem_sync_map_to_gpu(w->set->pool->mem,
                                w->set->mem_offset_B + w->dirty_start,
                                w->dirty_end - w->dirty_start);
   }
}
```

### Contrast with RADV

RADV also uses a GPU-visible BO per pool, but its descriptor infrastructure predates `VK_EXT_descriptor_buffer` and uses Mesa's common Vulkan `vk_object_base` layer without an equivalent to NVK's nvkmd abstraction. RADV writes descriptors directly into the mapped BO pointer using AMD GCN hardware resource descriptor format (32-byte image T# / 16-byte buffer V# — GCN buffer resource descriptors are 4 dwords), while NVK writes NVIDIA-format texture/sampler headers (indexed through a global bindless descriptor table on Turing and later GPUs).

---

## Acceleration Structure Descriptors

### Binding an Acceleration Structure

Ray tracing shaders access the top-level acceleration structure (TLAS) through a dedicated descriptor type, `VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR`. The layout binding is created like any other:

```c
VkDescriptorSetLayoutBinding as_binding = {
    .binding         = 0,
    .descriptorType  = VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR,
    .descriptorCount = 1,
    .stageFlags      = VK_SHADER_STAGE_RAYGEN_BIT_KHR
                     | VK_SHADER_STAGE_CLOSEST_HIT_BIT_KHR
                     | VK_SHADER_STAGE_MISS_BIT_KHR,
};
```

### Writing the TLAS Descriptor

`VkWriteDescriptorSetAccelerationStructureKHR` is chained onto the `pNext` of `VkWriteDescriptorSet`, replacing the usual `pBufferInfo` / `pImageInfo`:

```c
VkAccelerationStructureKHR tlas;  /* built via vkBuildAccelerationStructuresKHR */

VkWriteDescriptorSetAccelerationStructureKHR as_write = {
    .sType                      = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET_ACCELERATION_STRUCTURE_KHR,
    .pNext                      = NULL,
    .accelerationStructureCount = 1,
    .pAccelerationStructures    = &tlas,
};

VkWriteDescriptorSet write = {
    .sType           = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET,
    .pNext           = &as_write,        /* chain acceleration structure write */
    .dstSet          = descriptor_set,
    .dstBinding      = 0,
    .dstArrayElement = 0,
    .descriptorCount = 1,
    .descriptorType  = VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR,
};
vkUpdateDescriptorSets(device, 1, &write, 0, NULL);
```

[Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_acceleration_structure.html)

### GLSL Ray-Gen Shader Usage

```glsl
#version 460
#extension GL_EXT_ray_tracing : require

/* Acceleration structure binding: */
layout(set = 0, binding = 0) uniform accelerationStructureEXT topLevelAS;

/* Output image for ray tracing result: */
layout(set = 0, binding = 1, rgba8) uniform image2D result_image;

/* Camera matrices UBO (set 0, binding 2): */
layout(set = 0, binding = 2) uniform CameraUBO {
    mat4 inv_view;
    mat4 inv_proj;
} camera;

layout(location = 0) rayPayloadEXT vec3 hit_value;

void main() {
    vec2 pixel_center = vec2(gl_LaunchIDEXT.xy) + vec2(0.5);
    vec2 uv = pixel_center / vec2(gl_LaunchSizeEXT.xy);
    vec2 ndc = uv * 2.0 - 1.0;

    vec4 origin    = camera.inv_view * vec4(0, 0, 0, 1);
    vec4 target    = camera.inv_proj * vec4(ndc.x, ndc.y, 1, 1);
    vec4 direction = camera.inv_view * vec4(normalize(target.xyz), 0);

    traceRayEXT(
        topLevelAS,            /* the AS descriptor */
        gl_RayFlagsOpaqueEXT,
        0xFF,                  /* cull mask */
        0, 1, 0,               /* SBT offsets */
        origin.xyz, 0.001,     /* ray origin + tmin */
        direction.xyz, 1000.0, /* ray direction + tmax */
        0                      /* payload location */
    );

    imageStore(result_image, ivec2(gl_LaunchIDEXT.xy), vec4(hit_value, 1.0));
}
```

**Slang equivalent** — `[shader("raygen")]` replaces the `#extension GL_EXT_ray_tracing` entry-point convention; the ray payload becomes a typed local struct passed directly to `TraceRay<>`, eliminating the fragile `layout(location=N) rayPayloadEXT` numbering scheme.

```slang
// File: raygen.slang
// Slang improvements:
// - [shader("raygen")] is a first-class entry-point annotation; no extension pragmas needed
// - RaytracingAccelerationStructure is a built-in Slang type bound with [[vk::binding]]
// - Payload is a typed struct local to TraceRay<>; no rayPayloadEXT location aliasing risk
[require(spirv_1_4)]  // ray tracing requires SPIR-V 1.4 minimum

struct CameraUBO {
    float4x4 inv_view;
    float4x4 inv_proj;
};

struct Payload {
    float3 hit_value;
};

[[vk::binding(0, 0)]] RaytracingAccelerationStructure topLevelAS;
[[vk::binding(1, 0)]] RWTexture2D<float4>             result_image;
[[vk::binding(2, 0)]] ConstantBuffer<CameraUBO>        camera;

[shader("raygen")]
void main()
{
    uint2 pixel = DispatchRaysIndex().xy;           // gl_LaunchIDEXT equivalent
    uint2 size  = DispatchRaysDimensions().xy;      // gl_LaunchSizeEXT equivalent

    float2 uv  = (float2(pixel) + 0.5f) / float2(size);
    float2 ndc = uv * 2.0f - 1.0f;

    // Note: mul(M, v) in Slang row-major convention matches GLSL column-major M*v
    // when the application fills the UBO with a transposed matrix, as is standard practice
    float4 origin    = mul(camera.inv_view, float4(0, 0, 0, 1));
    float4 target    = mul(camera.inv_proj, float4(ndc.x, ndc.y, 1, 1));
    float4 direction = mul(camera.inv_view, float4(normalize(target.xyz), 0));

    RayDesc ray;
    ray.Origin    = origin.xyz;
    ray.TMin      = 0.001f;
    ray.Direction = direction.xyz;
    ray.TMax      = 1000.0f;

    Payload payload = {};
    TraceRay(
        topLevelAS,
        RAY_FLAG_FORCE_OPAQUE,  // gl_RayFlagsOpaqueEXT equivalent
        0xFF,                   // cull mask
        0, 1, 0,                // hit-group offset, geometry-contribution multiplier, miss index
        ray,
        payload                 // typed inout struct; no rayPayloadEXT location integer
    );

    result_image[pixel] = float4(payload.hit_value, 1.0f);
}
```

### Driver Storage: TLAS as 64-bit VA

On RADV, an acceleration structure descriptor is stored as the 64-bit device address returned by `vkGetAccelerationStructureDeviceAddressKHR`. The descriptor slot holds exactly 8 bytes (one `uint64_t`) — the GPU VA of the TLAS buffer. The AMD ray tracing hardware (RDNA 2+) reads the TLAS through this VA directly in the BVH traversal unit. The descriptor update path in `radv_descriptor_set.c` writes the VA into the set's mapped BO pointer, consistent with how buffer descriptors are written. On NVIDIA hardware (NVK), the same VA-based model applies: the TLAS descriptor is the 64-bit device address of the acceleration structure BO, fetched by the hardware's RTX traversal pipeline.

---

## Roadmap

### Near-term (6–12 months)

- **`VK_EXT_descriptor_heap` shipping in Vulkan SDK**: Ratified in Vulkan 1.4.340 and developed by NVIDIA, AMD, Arm, Nintendo, Valve, and Google, this extension completely replaces the descriptor-set subsystem with explicit heap memory management — one sampler heap and one resource heap — eliminating `VkDescriptorPool` and `VkPipelineLayout` entirely. SDK support was targeted for Q1 2026. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)
- **Mesa driver support for `VK_EXT_descriptor_heap`**: RADV and ANV are expected to land initial implementations tracking the new extension. RADV already implements `VK_EXT_descriptor_buffer` (since Mesa 23.0) and `VK_EXT_mutable_descriptor_type` (since Mesa 22.2), providing the foundation for the heap model. [Source](https://www.phoronix.com/news/RADV-VK_EXT_descriptor_buffer)
- **VKD3D-Proton adoption of descriptor heap**: Valve's D3D12-over-Vulkan translation layer, which depends on efficient descriptor management for Windows game compatibility, is positioned as an early adopter of `VK_EXT_descriptor_heap` for bindless D3D12 resource binding. Note: needs verification on merge timeline.
- **Vulkan Roadmap 2026 profile inclusion**: The Khronos Vulkan Working Group is targeting `VK_EXT_descriptor_heap` for promotion to a `KHR` extension and potential inclusion in the Vulkan Roadmap 2026 hardware baseline, which would mandate support on conformant drivers. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)

### Medium-term (1–3 years)

- **Promotion of `VK_EXT_descriptor_heap` to `VK_KHR_descriptor_heap` or core**: The working group is actively soliciting developer feedback before finalising the API as a KHR extension. If widely adopted, it could be mandated in a future Vulkan core version analogous to how descriptor indexing moved to Vulkan 1.2 core. [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_descriptor_heap.html)
- **GPU-side descriptor update via compute**: As `VK_EXT_descriptor_heap` exposes descriptor memory directly (CPU and GPU accessible), future tooling and engine middleware is expected to handle descriptor patching entirely in compute shaders, eliminating CPU-side `vkUpdateDescriptorSets` round-trips for dynamic scenes. Note: needs verification of specific driver timelines.
- **Shader compiler integration (`glslang`, `DXC`, `Tint`)**: WGSL (WebGPU Shading Language) and HLSL compilers will need new backends or decorations to target the heap model's offset-based lookup. Early compiler experiments are expected as `VK_EXT_descriptor_heap` stabilises. Note: needs verification.
- **`VK_EXT_mutable_descriptor_type` convergence**: As the heap model makes descriptor types fluid by design, the rationale for the separate mutable-descriptor-type extension diminishes; it may be deprecated in favour of the heap model's inherent type agnosticism. [Source](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_mutable_descriptor_type.html)

### Long-term

- **Unified descriptor model across Vulkan, D3D12, and Metal**: `VK_EXT_descriptor_heap` intentionally mirrors D3D12's descriptor-heap design and is closer to Metal's argument buffers. Long-term convergence at the abstraction-layer level (wgpu, Vulkan portability) may allow a single descriptor-heap code path to target all three APIs without per-backend descriptor translation. Note: speculative.
- **Hardware-native bindless on all GPU tiers**: Today bindless requires `VK_EXT_descriptor_indexing` plus driver workarounds on older IP. Future GPU hardware generations (post-RDNA 4, post-Xe2) may expose native heap addressing with no software emulation layer, reducing driver complexity. Note: speculative, no public hardware commitment.
- **Integration with ray tracing resource binding**: Acceleration structures (`VK_KHR_acceleration_structure`) currently use a separate descriptor type. Future extensions may unify TLAS/BLAS handles into the descriptor heap model, simplifying bindless ray tracing scene management. Note: speculative direction based on API evolution trends.

---

## Integrations

- **Ch19 (Vulkan Architecture)** — descriptors are the Vulkan resource binding model; this chapter expands on the pipeline layout and descriptor set binding sections of the architecture overview
- **Ch20 (SPIR-V)** — `nonuniformEXT` maps to SPIR-V `NonUniform` decoration; descriptor types map to SPIR-V `OpTypeImage`, `OpTypeSampler`, `OpTypeAccelerationStructureKHR`
- **Ch22 (RADV)** — RADV's descriptor implementation maps directly to AMD GCN hardware resource descriptors (T# and S# in AMD documentation)
- **Ch23 (ANV)** — ANV uses the Intel surface state heap and binding tables for descriptor storage
- **Ch104 (DXVK/VKD3D-Proton)** — both translation layers depend on `VK_EXT_mutable_descriptor_type` to emulate D3D12 volatile descriptor heaps and `nullDescriptor` to emulate null resource binds; `vkUpdateDescriptorSetWithTemplate` is used throughout for per-draw descriptor updates
- **Ch16 (Mesa Vulkan Common)** — NVK's `nvk_descriptor_set` and the descriptor update template path both build on Mesa's common `vk_object_base`, `vk_descriptor_update_template`, and `util_vma_heap` infrastructure shared across RADV, ANV, and NVK
- **Ch154 (GPU-Driven Rendering)** — bindless descriptors (`VK_EXT_descriptor_indexing`) are the prerequisite for bindless scene rendering; this chapter explains the binding model that Ch154 assumes; null descriptors (`VK_EXT_robustness2`) enable safe sparse binding in streaming scenes
- **Ch135 (Vulkan Ray Tracing)** — acceleration structure descriptors (`VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR`) use the same descriptor set mechanism as buffers and images; RADV and NVK store TLAS handles as 64-bit GPU VAs in the descriptor slot

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
