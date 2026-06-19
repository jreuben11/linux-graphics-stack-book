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
10. [Integrations](#integrations)

---

## Introduction

Descriptors are Vulkan's mechanism for binding resources (buffers, images, samplers) to shaders. Understanding the descriptor system at depth — from layout to pool to update to GPU register mapping — is essential for writing high-performance Vulkan code and for avoiding the common pitfalls of descriptor set management.

This chapter covers: the descriptor set lifecycle, the performance properties of different update strategies (write-after-bind, push descriptors, descriptor buffers), the `VK_EXT_descriptor_indexing` extension that enables bindless rendering (covered briefly in Ch154 from an application perspective), and `VK_EXT_descriptor_buffer` which moves descriptor management entirely to GPU-visible memory. It concludes with how RADV and ANV implement descriptors in hardware.

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

## Integrations

- **Ch19 (Vulkan Architecture)** — descriptors are the Vulkan resource binding model; this chapter expands on the pipeline layout and descriptor set binding sections of the architecture overview
- **Ch20 (SPIR-V)** — `nonuniformEXT` maps to SPIR-V `NonUniform` decoration; descriptor types map to SPIR-V `OpTypeImage`, `OpTypeSampler`, `OpTypeAccelerationStructureKHR`
- **Ch22 (RADV)** — RADV's descriptor implementation maps directly to AMD GCN hardware resource descriptors (T# and S# in AMD documentation)
- **Ch23 (ANV)** — ANV uses the Intel surface state heap and binding tables for descriptor storage
- **Ch154 (GPU-Driven Rendering)** — bindless descriptors (`VK_EXT_descriptor_indexing`) are the prerequisite for bindless scene rendering; this chapter explains the binding model that Ch154 assumes
- **Ch135 (Vulkan Ray Tracing)** — acceleration structure descriptors (`ACCELERATION_STRUCTURE_KHR`) use the same descriptor set mechanism as buffers and images
