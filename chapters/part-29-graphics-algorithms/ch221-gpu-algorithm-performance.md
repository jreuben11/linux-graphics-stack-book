# Chapter 221: GPU Algorithm Performance and Optimization

**Part**: XXIX — Graphics Algorithms

**Audiences**: Systems and driver developers optimising GPU workload throughput; graphics application developers diagnosing GPU bottlenecks; anyone who has written a compute shader and wants to know why it is slower than expected.

This chapter is about the gap between what a GPU is capable of and what your shader actually achieves. Every GPU has a theoretical peak compute throughput (TFLOPS) and a theoretical peak memory bandwidth (GB/s). Most shaders hit neither. They stall waiting for memory, underutilise shader cores through divergence, serialize through barriers, or waste CPU time submitting work. The sections below give you the mental models, concrete measurements, and code patterns needed to close that gap — across AMD RDNA, Intel Xe, and NVIDIA Ampere/Ada on Linux via Vulkan.

Cross-references: Ch 25 (Vulkan compute basics), Ch 133 (multi-queue architecture), Ch 141 (cooperative matrices), Ch 148 (synchronisation), Ch 154 (GPU-driven rendering), Ch 157 (descriptor binding).

---

## Table of Contents

1. [The GPU Performance Model — Roofline Analysis](#1-the-gpu-performance-model--roofline-analysis)
2. [Occupancy Analysis](#2-occupancy-analysis)
3. [Memory Access Optimisation](#3-memory-access-optimisation)
4. [Shared Memory and LDS](#4-shared-memory-and-lds)
5. [Warp and Wave Divergence](#5-warp-and-wave-divergence)
6. [Barriers and Pipeline Stalls](#6-barriers-and-pipeline-stalls)
7. [Persistent Kernels and Work Stealing](#7-persistent-kernels-and-work-stealing)
8. [Subgroup and Cooperative Primitives](#8-subgroup-and-cooperative-primitives)
9. [Multi-Queue and Async Compute](#9-multi-queue-and-async-compute)
10. [Profiling Tools on Linux](#10-profiling-tools-on-linux)
11. [Driver Overhead Reduction](#11-driver-overhead-reduction)
12. [Vendor-Specific Tuning Notes](#12-vendor-specific-tuning-notes)
13. [Integrations](#13-integrations)

---

## 1. The GPU Performance Model — Roofline Analysis

### 1.1 The Two Ceilings

A GPU computation has two hard limits: how fast it can perform arithmetic and how fast it can move data. The **roofline model**, a performance-analysis framework formalised around 2009, maps any algorithm onto a single plane whose axes are:

- **Arithmetic intensity** (AI): floating-point operations per byte of DRAM traffic, measured in FLOPs/byte (also written I = W/Q, work over traffic).
- **Attainable performance**: min(Peak FLOPS, AI × Peak Bandwidth).

At low arithmetic intensity the algorithm is **memory-bound**: performance scales with bandwidth, not compute. At high arithmetic intensity it is **compute-bound**: it needs more FLOPS than the memory system can feed. The crossover — where `Peak FLOPS = AI × Peak BW` — is the **ridge point**. An algorithm's roofline position is permanent for a given hardware platform; you move it only by restructuring the algorithm (e.g. fusing passes to re-use data in cache) or by increasing per-element compute (e.g. fusing a normalisation step into a shader that was already memory-bound).

### 1.2 Hardware Numbers (Representative 2024 Values)

| GPU | Peak FP32 (TFLOPS) | Peak Bandwidth (GB/s) | Ridge Point (FLOPs/byte) |
|---|---|---|---|
| AMD RX 7900 XTX (RDNA 3) | ~61 | ~960 | ~64 |
| NVIDIA RTX 4090 (Ada) | ~83 | ~1,008 | ~82 |
| Intel Arc A770 (Xe-HPG) | ~17 | ~560 | ~30 |
| AMD Radeon 780M (RDNA 3 iGPU) | ~8 | ~51 (shared LPDDR5) | ~157 |

*Note: TDP-limited runtimes vary. Verify current peak values from vendor datasheets.*

The **bandwidth wall** refers to the fact that DRAM bandwidth grows far more slowly than peak compute: GPU compute roughly doubles every two years; DRAM bandwidth grows perhaps 25–30% per generation. The ridge point moves right. Algorithms that were compute-bound in 2018 may be memory-bound on 2024 GPUs at the same code.

### 1.3 Worked Example: Element-Wise vs. Matrix Multiply

**Element-wise multiply** (`C[i] = A[i] * B[i]` for N floats): read 2N floats, write N floats, perform N FLOPs. Arithmetic intensity = 1 / 12 FLOPs/byte (3 × 4-byte loads/stores, one multiply). This sits far left of every GPU's ridge point. The algorithm will saturate DRAM bandwidth and leave the shader cores mostly idle. On the RX 7900 XTX this reaches ~80 GB/s for large N (well below the 960 GB/s peak) because most time is spent issuing memory requests, not computing.

**Naïve matrix multiply** (`C = A × B`, N×N, column-major): each output element C[i,j] requires loading N elements from row i of A and N from column j of B — 2N reads per output, N² outputs, N³ multiply-adds. Arithmetic intensity = N³ FLOPs / (3N² × 4 bytes) = N/12 FLOPs/byte. For N = 1024 this is ~85 FLOPs/byte, putting it above the RX 7900 XTX's ridge point: compute-bound. A tiled implementation that keeps tiles in shared memory / LDS performs the same arithmetic but reduces DRAM traffic toward the shared-memory bandwidth limit (~20 TB/s peak for AMD RDNA), achieving high utilisation of FLOP units.

### 1.4 Cache as the Middle Ground

The roofline model conventionally plots DRAM bandwidth. Real GPUs have cache hierarchies that act as internal bandwidth amplifiers:

- **L1 texture cache** (per-CU / per-SM): ~1–2 TB/s effective bandwidth on current hardware. Spatially-local access patterns fit here.
- **L2 cache**: ~4–10 TB/s on RDNA 3 / Ada. Shared across CUs. High L2 hit rates shift an algorithm from the DRAM roof to a flatter, higher L2 roof.
- **Shared memory / LDS** (explicitly managed): comparable bandwidth to L1 but programmer-controlled; covered in §4.

Plotting on the roofline is therefore a family of ceilings, not just one. An element-wise op with high spatial reuse (e.g. processing adjacent pixels in a post-process pass with a 3×3 kernel) may achieve L1 bandwidth instead of DRAM bandwidth, lifting effective performance 10–20×.

---

## 2. Occupancy Analysis

### 2.1 What Occupancy Means

A GPU compute unit (AMD: CU; NVIDIA: SM) is designed for **latency hiding through parallelism**. When one wave/warp stalls waiting for a memory load to complete, the hardware switches to another wave to keep shader ALU pipelines busy. The more waves that are simultaneously resident on a CU, the better the hardware can hide individual-wave latency.

**Occupancy** is the ratio of active wavefronts to the maximum the hardware supports. On AMD RDNA 2 the hardware documentation describes each SIMD unit as holding 16 wavefront slots; a Compute Unit on RDNA contains 2 SIMD32 units, giving 32 wavefront slots per CU (note: the exact SIMD-per-CU count and slot depth differ between GCN and RDNA generations — verify against current AMD ISA documentation before computing occupancy budgets for a specific GPU). Occupancy = (occupied slots) / (maximum slots for that GPU).

[Source: AMD occupancy guide, https://gpuopen.com/learn/occupancy-explained/]

On NVIDIA, each SM supports a maximum number of resident warps dependent on GPU generation (e.g. Ampere SM supports up to 64 warps simultaneously per SM).

### 2.2 The Three Limiters

Three resource budgets limit how many waves can co-reside on a CU:

1. **VGPRs (vector general-purpose registers)**: AMD GCN architecture documented 65,536 VGPRs per CU. As an illustrative example with those figures: a shader requiring 64 VGPRs per thread × 32 threads per wave = 2,048 VGPRs per wave; 65,536 / 2,048 = 32 simultaneous waves. At 120 VGPRs/thread: 65,536 / (120 × 32) ≈ 17 waves. (Note: RDNA-specific VGPR bank sizes and the exact maximum wavefront count per CU differ from GCN — verify against vendor documentation or the RGP profiler's occupancy panel for the target GPU before calculating budgets.) [Source: AMD occupancy guide, https://gpuopen.com/learn/occupancy-explained/]

2. **LDS (Local Data Share / shared memory)**: On RDNA, 64 KB of LDS per CU. A workgroup allocating 32 KB uses half the CU's LDS budget, allowing at most two concurrent workgroups regardless of register pressure.

3. **Thread block (workgroup) size**: A workgroup must fit on the CU without exceeding the SIMD slot or register/LDS limits. `maxComputeWorkGroupInvocations` in Vulkan is 1,024 on current hardware, meaning a single 1,024-thread workgroup occupies all SIMD slots on one CU with no room for another workgroup.

In Vulkan, query occupancy constraints at startup:

```c
// Query compute limits — Vulkan 1.0 core
VkPhysicalDeviceLimits limits = physical_device_properties.limits;
// limits.maxComputeWorkGroupInvocations  — max threads per workgroup (typically 1024)
// limits.maxComputeSharedMemorySize      — max shared memory per workgroup (typically 32-64 KB)

// For subgroup size control (AMD, NVIDIA, Intel all support this on Linux):
VkPhysicalDeviceSubgroupSizeControlProperties sc_props = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_SUBGROUP_SIZE_CONTROL_PROPERTIES,
};
VkPhysicalDeviceProperties2 props2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2,
    .pNext = &sc_props,
};
vkGetPhysicalDeviceProperties2(physical_device, &props2);
// sc_props.minSubgroupSize, sc_props.maxSubgroupSize
```

[Source: Vulkan spec `VkPhysicalDeviceSubgroupSizeControlProperties`, https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html]

### 2.3 When Low Occupancy Is Acceptable

Low occupancy is not always a problem. If a shader is **compute-bound** with very few memory stalls, even 25% occupancy may be sufficient — the shader units stay busy executing arithmetic between the rare cache misses. The critical question is: *does the GPU stall waiting for memory while all resident waves are also stalled?* If so, raising occupancy by reducing register pressure will help. If not, the register-pressure/performance trade-off runs the other direction: more registers often means fewer spills and fewer instructions, which is faster even at lower occupancy.

To reduce VGPR pressure on AMD:
- Use `f16` / `u16` types where precision allows (packs 2 values per VGPR).
- Move wave-invariant data to scalar registers (SGPRs): data that is the same across all 32 threads in a wave — e.g. buffer addresses, dispatch constants — belongs in SGPRs. `ByteAddressBuffer` loads indexed by a constant offset go into SGPRs, not VGPRs. [Source: AMD RDNA optimization, https://gpuopen.com/learn/optimizing-gpu-occupancy-resource-usage-large-thread-groups/]
- The Mesa `AMD_DEBUG=w32cs` flag forces wave32 mode on compute shaders, and `w64cs` forces wave64. Narrower waves halve the register file committed to a wave, which can improve occupancy at the cost of reduced SIMD lane utilisation.

### 2.4 Reading Occupancy from Profilers

**AMD RGP** (Radeon GPU Profiler): the Wavefront Occupancy view shows a timeline of wavefronts resident on each SIMD, colour-coded by shader type. The x-axis is GPU time; the y-axis is SIMD slot fill. Sparse vertical bands indicate low occupancy intervals. The Event Details panel shows the average wavefront count and the limiting resource (VGPRs, LDS, or workgroup count). [Source: AMD RGP documentation, https://gpuopen.com/rgp/]

**NVIDIA Nsight Graphics**: the SM Warp State chart shows per-SM warp counts over time, broken down by stall reason (L1 miss, L2 miss, execution dependency, memory bar). An SM running at 80 warps of a possible 64 per partition indicates full occupancy with no spare capacity. [Source: NVIDIA Nsight Graphics, https://developer.nvidia.com/nsight-graphics]

---

## 3. Memory Access Optimisation

### 3.1 Coalesced Access

A GPU memory transaction fetches a contiguous block from DRAM. When multiple threads in a wave request addresses that fall within the same block, those requests merge into a single transaction — this is **coalescing**. When requests scatter across non-contiguous addresses, the hardware issues multiple transactions: throughput falls proportionally.

Transaction sizes are vendor- and generation-specific. The key principle is that **stride-1 access** (adjacent threads read adjacent 4-byte elements) fully coalesces into the minimum number of cache-line transactions; **stride-N access** for N > cache line width / 4 issues one transaction per thread. (Note: verify exact transaction sizes per GPU generation against vendor documentation and profiler counter feedback before optimising.)

**Example — AoS vs. SoA:**

```glsl
// Array-of-Structures layout: poor coalescing for component-wise ops
struct Particle { vec3 pos; float mass; vec3 vel; float life; };  // 32 bytes
layout(binding=0) buffer Particles { Particle p[]; };
// Thread i accesses p[i].pos.x at offset i*32+0 — stride 32 bytes
// Thread i+1 accesses p[i+1].pos.x at offset (i+1)*32+0
// 32-thread wave reads 32 * 32 = 1024 bytes touching many cache lines

// Structure-of-Arrays layout: fully coalesced
layout(binding=0) buffer PosX   { float pos_x[]; };
layout(binding=1) buffer PosY   { float pos_y[]; };
layout(binding=2) buffer PosZ   { float pos_z[]; };
// Thread i accesses pos_x[i] — adjacent, coalesced into one or two transactions
```

For position update kernels (read pos, update, write pos), SoA is consistently faster when all threads process the same component. AoS can be preferable when a single thread processes all components of one particle and the total working set fits in L1.

### 3.2 Alignment Requirements

Vulkan enforces minimum buffer alignment for storage buffer descriptors. Query at startup:

```c
VkPhysicalDeviceLimits limits = props.limits;
// limits.minStorageBufferOffsetAlignment — typically 16–64 bytes
// limits.minUniformBufferOffsetAlignment — typically 64–256 bytes
```

Misaligned bindings either fail validation or trigger driver fix-up paths. Sub-element alignment (e.g. binding a `vec3` array with stride 12 bytes, not 16) often triggers slow general-purpose memory paths rather than optimised vector loads. Pad `vec3` arrays to `vec4` (16 bytes) in GPU buffers for correct alignment.

### 3.3 Texture Cache vs. Buffer for Read-Only Data

Vulkan exposes two read paths for spatially-local read-only data:

- **`VK_DESCRIPTOR_TYPE_STORAGE_BUFFER`** (`coherent` in GLSL): routes through the L2 data cache. Optimal for linear, strided access.
- **`VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE` / texture fetch**: routes through the L1 texture cache (TMU), which is optimised for 2D spatial locality. For data with 2D access patterns (grid cells, image tiles, lookup tables indexed by screen-space coordinates) the texture cache delivers higher hit rates than the L2 data cache.

When porting a compute kernel that reads from a 2D grid, binding the grid as a `sampled2D` with nearest-neighbour filtering and `VK_BORDER_COLOR_FLOAT_TRANSPARENT_BLACK` often outperforms a `storage_buffer` binding at the same access pattern, because the TMU hardware pre-fetches adjacent texels speculatively.

### 3.4 Thread-Group ID Swizzling for L2 Locality

By default, Vulkan dispatches workgroups in row-major order: group (0,0), (1,0), (2,0), … Groups launched at the same time may access distant memory regions, resulting in L2 thrashing. Remapping the linear dispatch index to a tiled layout increases L2 hit rates:

```glsl
// Compute shader — thread-group ID swizzling for 2D dispatch
// Use with vkCmdDispatch(ceil(W/8), ceil(H/8), 1) where local size = (8,8,1)
layout(local_size_x = 8, local_size_y = 8, local_size_z = 1) in;

layout(push_constant) uniform PC {
    uvec2 dispatch_dims;  // (W_groups, H_groups)
    uint  tile_width;     // N groups per horizontal tile, e.g. 16
};

uvec2 swizzle_workgroup_id() {
    uint linear   = gl_WorkGroupID.y * dispatch_dims.x + gl_WorkGroupID.x;
    uint tile     = linear / (tile_width * dispatch_dims.y);
    uint within   = linear % (tile_width * dispatch_dims.y);
    uint tile_row = within / tile_width;
    uint tile_col = within % tile_width;
    return uvec2(tile * tile_width + tile_col, tile_row);
}
```

NVIDIA testing on Battlefield V DXR showed that horizontal tiling (N=16) increased L2 read hit rate from 63% to 86%, yielding 47% performance improvement on a VRAM-limited workload. [Source: NVIDIA developer blog, https://developer.nvidia.com/blog/optimizing-compute-shaders-for-l2-locality-using-thread-group-id-swizzling/]

---

## 4. Shared Memory and LDS

### 4.1 Bank Conflicts

Both NVIDIA and AMD implement shared memory / LDS as a banked SRAM. AMD RDNA documents 32 banks of 32-bit (4-byte) storage each in LDS. [Source: AMD RDNA performance guide, https://gpuopen.com/rdna-performance-guide/] A 32-thread wave accesses LDS simultaneously; if all 32 threads address the same bank, the hardware serialises the accesses into 32 sequential operations — a **32-way bank conflict**.

Bank index = (byte address / 4) % 32.

```glsl
// Bank conflict example — stride-1 access (ideal, no conflicts):
// Thread i reads shared_mem[i] — bank = i % 32, all different
shared float data[1024];
float val = data[gl_SubgroupInvocationID]; // no conflict in a 32-wide wave

// Bank conflict example — stride-16 float access (worst case):
// Thread i reads shared_mem[i * 16] — bank = (i*16) % 32 = (i*16 mod 32)
// i=0 → bank 0, i=1 → bank 16, i=2 → bank 0 (conflict with i=0!)
float strided_val = data[gl_SubgroupInvocationID * 16]; // 16-way conflict for 32 threads
```

**Padding strategy**: add one extra column element to each row of a 2D tile stored in shared memory. If a row has `TILE_W` elements, allocate `TILE_W + 1` floats per row. The extra element shifts each row's base by one bank, so column accesses that previously all landed on bank 0 now land on banks 0, 1, 2, … This adds ~3% storage overhead for a tile and eliminates column-stride conflicts in matrix transpose and convolution kernels.

```glsl
// Tiled matrix transpose without and with padding:
const int TILE = 32;
shared float tile_nop[TILE][TILE];       // column access: 32-way conflict
shared float tile_pad[TILE][TILE + 1];   // column access: no conflict (each row offset by 1 bank)
```

### 4.2 Double-Buffering for Async Prefetch

A **double-buffer** pattern uses two LDS tiles: one being consumed by the current computation while the next tile is loaded from global memory simultaneously. This overlaps the global memory load latency with ALU computation:

```glsl
layout(local_size_x = 64, local_size_y = 1) in;
shared float buf[2][64];  // two-slot LDS buffer

void main() {
    int tid = int(gl_LocalInvocationID.x);
    int ping = 0, pong = 1;

    // Load first tile synchronously
    buf[ping][tid] = input_data[gl_WorkGroupID.x * 128 + tid];
    barrier();

    for (int block = 1; block < num_blocks; ++block) {
        // Start loading next tile while computing on current
        buf[pong][tid] = input_data[block * 64 + tid + 64];
        // Compute on ping buffer — latency of pong load hides here
        float result = compute(buf[ping][tid]);
        output_data[(block - 1) * 64 + tid] = result;
        barrier();
        // Swap
        int tmp = ping; ping = pong; pong = tmp;
    }
    // Drain last tile
    output_data[(num_blocks - 1) * 64 + tid] = compute(buf[ping][tid]);
}
```

The `barrier()` call is a `groupMemoryBarrier()` + execution barrier. On AMD RDNA this compiles to `s_barrier` in the ISA, which synchronises all waves in the workgroup at that point. The double-buffer pattern is most effective when the global memory load latency (typically 400–700 GPU cycles for an L2 miss on current hardware) exceeds the compute time for one tile.

---

## 5. Warp and Wave Divergence

### 5.1 What Divergence Costs

A GPU wave (AMD: wavefront, 32 threads; NVIDIA: warp, 32 threads) executes one instruction at a time across all lanes. When threads in a wave take different branches, the hardware must execute both paths, **disabling** the threads that did not take each path (predicated off). The cost is proportional to the number of distinct paths: two-way divergence doubles execution time for the divergent region; N-way divergence multiplies it by N.

Divergence is worst in loops with thread-varying trip counts:

```glsl
// High divergence — each thread loops a different number of times
int count = item_count[gl_GlobalInvocationID.x];  // varies per thread
float acc = 0.0;
for (int i = 0; i < count; ++i) {
    acc += data[gl_GlobalInvocationID.x * MAX + i];
}
```

Threads finishing early must wait (predicated off) until the last thread completes its loop. If `count` varies widely across threads, the effective throughput is dominated by the maximum, not the mean.

### 5.2 Ballot and Vote Intrinsics

GLSL subgroup operations (from `GL_KHR_shader_subgroup`) allow inspecting and acting on the divergence state of a wave without full barrier overhead:

```glsl
#extension GL_KHR_shader_subgroup_ballot : enable
#extension GL_KHR_shader_subgroup_vote   : enable

// subgroupBallot: which lanes pass a predicate?
uvec4 active = subgroupBallot(some_condition);
uint  count  = subgroupBallotBitCount(active);  // how many lanes are active

// subgroupAny / subgroupAll: fast wave-level reductions
if (subgroupAny(needs_expensive_op)) {
    // Only enter if at least one lane needs this
    result = expensive_fallback();
} else {
    result = fast_path();
}
```

[Source: GL_KHR_shader_subgroup extension spec, https://github.com/KhronosGroup/GLSL/blob/main/extensions/khr/GL_KHR_shader_subgroup.txt]

`subgroupBallot` is free on hardware that exposes the active mask as a register (NVIDIA: `__ballot_sync`; AMD: `s_mov_b64` from the exec mask). The ballot can drive work-item compaction: threads with `active` bits set in the ballot can be re-indexed to pack them contiguously, converting a sparse loop into a dense one.

### 5.3 Predication vs. Branching

For short divergent regions (a few instructions), **predication** (executing both paths, masking the unwanted one) is often faster than **branching** (inserting a conditional jump, flushing pipeline state). The crossover depends on:

- Branch probability: if the branch is taken by nearly all threads nearly always, a real branch skips the other path entirely.
- Code size of each path: very long paths make predication expensive.

A useful heuristic: prefer predication for paths shorter than ~8 instructions and probability < 90%. Use real branches for longer divergent regions. Mesa's ACO compiler performs this analysis automatically on SPIR-V → ISA lowering.

### 5.4 RDNA Wave32 vs. Wave64 and Divergence Cost

On AMD RDNA, the default wave width is 32 (wave32). The preceding GCN architecture used wave64. Narrower waves mean:

- **Lower divergence overhead**: a two-way branch in a 32-thread wave idles at most 16 threads, vs. 32 in wave64. The absolute waste is halved.
- **Lower occupancy per workgroup slot**: wave32 uses half the register file of wave64 for the same code, allowing more concurrent waves.

The Mesa `AMD_DEBUG=w64cs` flag forces wave64 on compute shaders for comparison. For algorithms with high divergence (e.g. ray traversal with variable path depth), wave32 consistently outperforms wave64. For throughput-limited VALU (vector ALU) workloads with no divergence, wave64 can be marginally better due to higher SIMD lane utilisation. [Source: AMD RDNA performance guide, https://gpuopen.com/rdna-performance-guide/]

---

## 6. Barriers and Pipeline Stalls

### 6.1 What a Pipeline Barrier Serialises

`vkCmdPipelineBarrier2` (Vulkan 1.3 / `VK_KHR_synchronization2`) inserts a **dependency scope** into the command buffer. It specifies:

- **`srcStageMask`**: stages that must complete before the barrier.
- **`dstStageMask`**: stages that must not start until the barrier completes.
- **`srcAccessMask`**: cache writeback (make-available) from the completing stages.
- **`dstAccessMask`**: cache invalidation (make-visible) for the waiting stages.

A barrier with `srcStageMask = ALL_COMMANDS` and `dstStageMask = ALL_COMMANDS` is a full GPU pipeline drain: nothing from before the barrier overlaps with anything after it. On hardware this typically means waiting for all in-flight commands to retire, flushing L1/L2 caches, then restarting. This is expensive (hundreds of microseconds on a busy GPU) and should be reserved for frame boundaries or CPU readback. [Source: Ch148 Vulkan synchronisation]

### 6.2 Tightening Stage Masks

The primary optimisation for barriers is to use the **narrowest correct stage mask**:

```c
// Overly broad — serialises everything:
VkMemoryBarrier2 mb_bad = {
    .srcStageMask  = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT,
    .srcAccessMask = VK_ACCESS_2_MEMORY_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT,
    .dstAccessMask = VK_ACCESS_2_MEMORY_READ_BIT,
};

// Tight — only blocks what needs blocking:
// Scenario: compute shader writes SSBO, next compute shader reads it.
VkMemoryBarrier2 mb_good = {
    .sType         = VK_STRUCTURE_TYPE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_2_SHADER_STORAGE_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_STORAGE_READ_BIT,
};
VkDependencyInfo dep = {
    .sType              = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .memoryBarrierCount = 1,
    .pMemoryBarriers    = &mb_good,
};
vkCmdPipelineBarrier2(cmd, &dep);
```

[Source: Vulkan synchronisation2 spec, https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization-pipeline-barriers]

### 6.3 Split Barriers with VkEvent

`VkEvent` implements a **split barrier**: the producer-side synchronisation scope (`vkCmdSetEvent2`) can be placed immediately after the write, and the consumer-side scope (`vkCmdWaitEvents2`) can be placed as late as needed before the read. Commands between the two can execute freely on different shader units, enabling work to overlap the barrier:

```c
// Producer side — place immediately after compute shader dispatch that writes buf:
vkCmdSetEvent2(cmd, event, &dep_write_done);

// … record other draws / dispatches that don't touch buf …

// Consumer side — just before the dispatch that reads buf:
vkCmdWaitEvents2(cmd, 1, &event, &dep_wait_read);
```

Use split barriers for shadow map generation → lighting pass dependencies: the shadow map write can be signalled early, the lighting pass scheduled later, and the GPU can run other work (particle update, TAA) in the gap.

### 6.4 Image Layout Transitions

Every `VkImage` barrier that includes an image layout transition (e.g. `UNDEFINED → SHADER_READ_ONLY_OPTIMAL`) may trigger a decompress or metadata flush operation on hardware. On AMD RDNA, the hardware DCC (Delta Color Compression) metadata must be resolved when transitioning a compressed render target to a sampled image. This is an internal compute shader dispatch inserted by the driver — an extra cost invisible in the Vulkan command buffer but visible in RGP as a "LayoutTransition" internal event.

Minimise unnecessary transitions: keep images in the same layout across multiple render passes if possible, and avoid `UNDEFINED → SHADER_READ_ONLY_OPTIMAL` for images that have never been written (prefer transitioning to `GENERAL` once at upload time and leaving them there for read-only resources).

### 6.5 Submit Overhead

`vkQueueSubmit2` itself carries CPU overhead: the driver must validate the submission, pack kernel ring-buffer entries, and potentially ring an interrupt. On Linux with AMDGPU, a single `vkQueueSubmit2` carries measurable CPU overhead in the driver's queue-submit path (Note: exact timing varies by driver version and submission complexity — measure with a CPU profiler such as `perf` or Nsight Systems rather than assuming a fixed value). [Source: Mesa RADV queue implementation, `src/amd/vulkan/radv_queue.c`, https://gitlab.freedesktop.org/mesa/mesa]

Batch multiple command buffer submissions into a single `vkQueueSubmit2` call using an array of `VkCommandBufferSubmitInfo`. Pre-record command buffers for static work and reuse them across frames to avoid re-recording overhead.

---

## 7. Persistent Kernels and Work Stealing

### 7.1 The Pattern

A **persistent kernel** is a compute shader that does not exit after processing a fixed grid of work. Instead it loops, atomically dequeuing work items from a GPU-side queue until the queue is empty, then exits. This eliminates the round-trip through the CPU to re-dispatch when a workload has variable or dynamically generated amounts of work.

```glsl
// Persistent kernel — Vulkan compute, GLSL
#extension GL_ARB_gpu_shader_int64 : enable

layout(local_size_x = 64) in;

layout(set=0, binding=0) buffer WorkQueue {
    uint head;                     // next item to dequeue (atomic counter)
    uint count;                    // total items
    uint items[];                  // work item indices
};

layout(set=0, binding=1) buffer Results {
    float results[];
};

void main() {
    while (true) {
        // Atomically claim one work item
        uint idx = atomicAdd(head, 1u);
        if (idx >= count) break;
        results[items[idx]] = do_work(items[idx]);
    }
}
```

The `atomicAdd` on a GPU-visible `VkBuffer` (device address atomics, exposed via `VK_KHR_buffer_device_address` and `GL_EXT_buffer_reference`) serialises one 32-bit word per dequeue. The cost is one atomic operation per work item; for coarse-grained items this overhead is negligible.

### 7.2 Use Cases

- **Variable-depth ray tracing**: path length per pixel varies widely (shadow rays terminate quickly; diffuse bounces can go 4–8 levels deep). A persistent kernel over a ray queue ensures that fast-finishing rays immediately pick up the next queued ray rather than leaving SIMD lanes idle.
- **Dynamic particle simulation**: particle death and spawn rates vary per frame. A persistent emitter kernel dequeues free slots from an atomic stack and initialises them.
- **Stream compaction without readback**: a persistent kernel can compact a sparse list without returning the count to the CPU — the kernel signals completion via a VkSemaphore or writes a `vkCmdSetEvent2` trigger when the queue empties.

### 7.3 Vulkan Implementation Constraints

True cooperative launch (where all SMs / CUs rendezvous and communicate) requires that all workgroups are launched simultaneously and can communicate via device-scope atomics. Vulkan does not expose a direct equivalent of CUDA's `cudaLaunchCooperativeKernel`. The nearest equivalent is:

- Launch a dispatch sized to exactly the number of CUs/SMs (query `VkPhysicalDeviceShaderSMBuiltinsPropertiesNV::shaderSMCount` on NVIDIA, or `VkPhysicalDeviceShaderCorePropertiesAMD::computeUnitsPerShaderArray` on AMD via `VK_AMD_shader_core_properties`).
- Use `VK_KHR_workgroup_memory_explicit_layout` for cross-workgroup shared memory when needed.
- Use device-scope atomics (`gl_DeviceCoherent` in GLSL, or `Device` scope in SPIR-V) for the work queue rather than workgroup-scope atomics.

The persistent kernel pattern works without cooperative launch — each workgroup simply loops on the atomic counter. True device-scope synchronisation (a global barrier across all workgroups) is not achievable without cooperative launch and is therefore avoided in production persistent kernels.

---

## 8. Subgroup and Cooperative Primitives

### 8.1 Subgroup Operations in GLSL

The `GL_KHR_shader_subgroup` family of extensions exposes hardware-level wave operations. All current NVIDIA, AMD, and Intel Vulkan drivers on Linux support the arithmetic subgroup extension: [Source: GL_KHR_shader_subgroup spec, https://github.com/KhronosGroup/GLSL/blob/main/extensions/khr/GL_KHR_shader_subgroup.txt]

```glsl
#extension GL_KHR_shader_subgroup_arithmetic  : enable
#extension GL_KHR_shader_subgroup_ballot      : enable
#extension GL_KHR_shader_subgroup_shuffle     : enable

// Reduction across the wave — no shared memory needed:
float lane_val = some_per_thread_computation();
float wave_sum = subgroupAdd(lane_val);
float wave_min = subgroupMin(lane_val);

// Inclusive prefix scan (each lane gets sum of itself and all lower lanes):
float inclusive_sum = subgroupInclusiveAdd(lane_val);
// Exclusive prefix scan (each lane gets sum of all lower lanes, not itself):
float exclusive_sum = subgroupExclusiveAdd(lane_val);

// Shuffle: lane i reads from lane j without shared memory:
float neighbor = subgroupShuffle(lane_val, (gl_SubgroupInvocationID + 1) % gl_SubgroupSize);

// Broadcast from one lane to all:
float val_0 = subgroupBroadcast(lane_val, 0u);
```

Subgroup operations compile to single ISA instructions on all three vendors (AMD: `v_add_f32` with DPP modifier or DS instructions; NVIDIA: warp shuffle; Intel: SIMD lane operations). They are faster than shared-memory reductions for wave-size problems because they avoid the LDS write-barrier-read cycle (≈3–10 clock overhead on RDNA vs. ≈1 clock for a DPP instruction).

### 8.2 Subgroup Size Control

On AMD, NVIDIA, and Intel, the hardware subgroup size can differ from the default. `VK_EXT_subgroup_size_control` allows the application to request a specific subgroup size for a pipeline:

```c
VkPipelineShaderStageRequiredSubgroupSizeCreateInfo req = {
    .sType              = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_REQUIRED_SUBGROUP_SIZE_CREATE_INFO,
    .requiredSubgroupSize = 32,  // request wave32 on AMD (or warp32 on NVIDIA)
};
VkPipelineShaderStageCreateInfo stage = {
    .sType  = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO,
    .flags  = VK_PIPELINE_SHADER_STAGE_CREATE_REQUIRE_FULL_SUBGROUPS_BIT,
    .pNext  = &req,
    // ...
};
```

[Source: Vulkan spec `VK_EXT_subgroup_size_control`, https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html]

On AMD, requesting subgroup size 32 forces wave32 for that pipeline regardless of the Mesa default. On Intel Xe, the subgroup size affects EU (Execution Unit) lane width. Profiling is needed to determine which is faster per workload.

### 8.3 Cooperative Matrices for GEMM

`VK_KHR_cooperative_matrix` (promoted to KHR in 2023, core in Vulkan 1.4) exposes hardware MMA (matrix multiply-accumulate) units. A group of subgroup lanes cooperates to compute a tile of a matrix product using dedicated hardware. See Chapter 141 for a full treatment. The key occupancy concern here: a shader using cooperative matrix operations typically requires more VGPRs per thread (to hold matrix fragments), which lowers wave occupancy. The trade-off is acceptable because MMA instructions achieve very high arithmetic throughput per clock — the reduced occupancy does not create latency exposure the way it would for a memory-bound shader.

---

## 9. Multi-Queue and Async Compute

### 9.1 Queue Families on Linux

Vulkan exposes the GPU's hardware command processors as **queue families**. On Linux, the major drivers expose:

| Vendor | Queue Families | Typical Count |
|---|---|---|
| AMD RADV (RDNA 3) | Graphics+Compute+Transfer, Compute-only (ACE), Transfer-only (SDMA) | 1 + 2 + 2 |
| Intel ANV (Xe2) | Graphics+Compute, Compute-only, Copy, Video | 1 + 1 + 1 + 1 |
| NVIDIA NVK / proprietary | Graphics+Compute, Transfer | 1 + (1–5 copy engines) |

[Source: Ch133 Vulkan Compute Queues and Task Graphs]

Enumerate families with `vkGetPhysicalDeviceQueueFamilyProperties2`. Select a dedicated compute family (one without `VK_QUEUE_GRAPHICS_BIT`) for async workloads.

### 9.2 Async Compute Overlap

The benefit of a dedicated compute queue is that the GPU hardware can execute compute work simultaneously with graphics work on different engine blocks. On AMD RDNA, the Asynchronous Compute Engines (ACEs) dispatch compute work independently of the Graphics Command Processor (GFX ring). The Micro Engine Scheduler (MES) firmware arbitrates CU access between them: if the GFX ring is running a vertex shader that uses 40% of CUs, the ACE can schedule compute waves on the remaining 60%.

A typical async overlap pattern for shadow rendering:

```text
Frame N:
  GFX queue:   [G-buffer pass]──────────────────[Lighting pass]
  ACE queue:              [Shadow map compute]
  Timeline semaphore chain: shadow map completes → lighting reads it
```

Timeline semaphores (Vulkan 1.2 core) chain cross-queue dependencies:

```c
// Signal from ACE queue when shadow map is ready:
VkSemaphoreSubmitInfo sig = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = timeline_sem,
    .value     = frame_count * 2 + 1,  // monotonically increasing
    .stageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
};
// Wait on GFX queue before lighting pass:
VkSemaphoreSubmitInfo wait = {
    .sType     = VK_STRUCTURE_TYPE_SEMAPHORE_SUBMIT_INFO,
    .semaphore = timeline_sem,
    .value     = frame_count * 2 + 1,
    .stageMask = VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT,
};
```

[Source: Ch133 Vulkan Compute Queues and Task Graphs; Ch148 Vulkan Synchronisation]

On AMD RDNA the ACE queues run concurrently with the GFX ring when their resource footprints allow it. The benefit is maximised when the async workload is memory-bandwidth bound (and thus CU-sparse) while the graphics workload is triangle-heavy. Submitting a shadow-map depth pass — which is vertex-only and occupies few fragment shader units — on the async queue is less useful than a post-process or particle compute.

For NVIDIA GPUs, hardware async compute overlap availability depends on the GPU generation and driver. On consumer GPUs (GTX 10xx through RTX 40xx), the degree of actual hardware overlap for graphics+compute concurrent execution is driver- and workload-dependent. Note: needs verification per GPU generation — measure with Nsight before relying on overlap for performance.

### 9.3 DMA Engine for Async Uploads

The transfer queue (backed by SDMA on AMD, copy engines on NVIDIA) can stream texture data from host to device memory simultaneously with rendering. Use it for streaming open-world asset loading:

```c
// Allocate staging buffer (HOST_VISIBLE | HOST_COHERENT)
// memcpy into staging buffer on CPU
// Submit copy (vkCmdCopyBufferToImage) to transfer queue
// Signal transfer semaphore when done
// Wait on graphics queue before first sample of the texture
```

RDNA 3 exposes two SDMA engines; submitting to both can double upload bandwidth. The transfer queue is separate from compute, so uploads occur simultaneously with both rendering and compute without contention.

---

## 10. Profiling Tools on Linux

### 10.1 AMD Radeon GPU Profiler (RGP)

RGP is the primary low-level profiling tool for AMD GPUs on Linux (Ubuntu 24.04 LTS with Vulkan workloads is officially supported). [Source: AMD RGP documentation, https://gpuopen.com/rgp/]

**Capturing a profile**: Install the Radeon Developer Panel (RDP) and Radeon GPU Profiler. From the terminal:

```bash
# Install from AMD GPU Tools PPA or build from source
# https://github.com/GPUOpen-Tools/radeon_gpu_profiler
RadeonDeveloperPanel &   # launches background capture daemon
# Run Vulkan application — trigger capture from RDP UI or via API:
# VK_AMD_RASTERIZATION_ORDER must not be disabled; RGP hooks driver capture APIs
```

RGP captures via AMD's SQTT (Shader Queue Thread Trace) hardware mechanism, which records per-instruction timing for every shader stage.

**Key views**:

- **Frame Summary**: command buffer submission timeline, queue utilisation bars for GFX, ACE, and SDMA queues.
- **Wavefront Occupancy** (SQTT Occupancy Track): per-SIMD wavefront fill over time. Gaps indicate low occupancy intervals; their cause (VGPR, LDS, or workgroup limit) appears in the side panel.
- **Barrier Analysis**: lists every barrier in the frame, coloured by whether it caused a pipeline stall, flushed caches, or triggered an internal driver shader (DCC decompress, HTILE decompress). Unnecessary barriers appear with zero-cost stalls — they can be removed. High-cost barriers flag optimisation opportunities.
- **Instruction Timing**: for shaders instrumented via SQTT, displays average latency per ISA instruction. Identifies which instructions are the pipeline bottleneck (e.g. `IMAGE_SAMPLE` with long cache-miss latency vs. `V_MUL_F32` completing in 1 cycle).

RGP v2.7 (latest as of mid-2024) adds support for RX 9000 series hardware and thread divergence metrics. [Source: AMD RGP GitHub repository, https://github.com/GPUOpen-Tools/radeon_gpu_profiler]

### 10.2 NVIDIA Nsight Graphics on Linux

Nsight Graphics is available as a Linux x86-64 download from developer.nvidia.com. [Source: https://developer.nvidia.com/nsight-graphics]

On Linux, Nsight Graphics attaches via a capture layer to Vulkan applications:

```bash
# Launch application under Nsight:
/path/to/nsight-graphics/host/linux-desktop-nomad-x64/ngfx \
    --activity=FrameDebugger \
    -- ./my_vulkan_app
```

**Key profiling views**:

- **GPU Trace**: SM warp state over time, broken into busy / stalled (memory, sync, texture) / idle.
- **Shader Profiler**: instruction-level throughput, register count, theoretical vs. achieved occupancy per shader.
- **Memory chart**: L1/L2 hit rates, DRAM bandwidth utilisation.
- **Range Profiler**: hardware counter collection per draw/dispatch; exposes throughput for each GPU unit (L2, VRAM, SM).

Note: Nsight Systems (separate from Nsight Graphics) is better suited for whole-frame CPU+GPU timeline analysis and is the tool to use for identifying `vkQueueSubmit` overhead and CPU-side bottlenecks.

### 10.3 Intel GPU Top and GPA

```bash
# intel_gpu_top from intel-gpu-tools package:
sudo intel_gpu_top         # live terminal display of EU utilisation, memory bw, ring engines
sudo intel_gpu_top -J      # JSON output for scripting
```

For deeper per-frame analysis, Intel Graphics Performance Analyzers (GPA) provides a Frame Analyzer equivalent to Nsight / RGP for Intel Xe GPUs.

The `VK_INTEL_performance_query` Vulkan extension exposes Intel hardware counters directly from application code without external tooling:

```c
// Enumerate available metrics:
uint32_t count;
vkEnumeratePerformanceQueryCountersKHR(device, queue_family, &count, NULL, NULL);
// ... select and create VkQueryPool with type VK_QUERY_TYPE_PERFORMANCE_QUERY_INTEL
```

### 10.4 Application-Level GPU Timestamps and VK_EXT_performance_query

For portable vendor-counter access from Vulkan, `VK_KHR_performance_query` (not to be confused with the Intel-specific extension) lets applications collect hardware performance counters:

```c
// Scope GPU time with Vulkan timestamp queries (universally supported):
VkQueryPoolCreateInfo qp_info = {
    .sType      = VK_STRUCTURE_TYPE_QUERY_POOL_CREATE_INFO,
    .queryType  = VK_QUERY_TYPE_TIMESTAMP,
    .queryCount = 2,
};
vkCreateQueryPool(device, &qp_info, NULL, &query_pool);

// In command buffer:
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT, query_pool, 0);
// ... dispatches ...
vkCmdWriteTimestamp2(cmd, VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT, query_pool, 1);

// After submission, read results:
uint64_t timestamps[2];
vkGetQueryPoolResults(device, query_pool, 0, 2,
    sizeof(timestamps), timestamps, sizeof(uint64_t),
    VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT);
double gpu_time_ms = (timestamps[1] - timestamps[0]) *
    props.limits.timestampPeriod * 1e-6;
```

### 10.5 Perfetto and u_trace

Mesa integrates with Perfetto via its `u_trace` framework. Enabling GPU tracing:

```bash
export MESA_GPU_TRACEFILE=/tmp/trace.perfetto
./my_vulkan_app
# Open trace in https://ui.perfetto.dev
```

The trace includes GPU command submission events, queue events, and shader dispatch timestamps for all Mesa-based drivers (RADV, ANV, turnip). This is the primary tool for cross-vendor frame analysis on Linux when RGP and Nsight are not available. [Source: Mesa GPU performance tracing docs, https://docs.mesa3d.org/gpu-perf-tracing.html]

### 10.6 AMD umr for Low-Level State

`umr` (userspace Mesa register dumper) is an AMD-specific tool for reading GPU MMIO registers and hardware ring state directly:

```bash
# Install from: https://gitlab.freedesktop.org/tomstdenis/umr
sudo umr -s gfx_init       # dump GFX ring init state
sudo umr --waves            # show currently running wavefronts (ISA + register state)
sudo umr -c 0,0 --disasm    # disassemble shader from compute ring
```

`umr --waves` is invaluable for diagnosing GPU hangs: it captures the ISA instruction pointer and register file of every active wavefront at hang time, allowing identification of which shader and instruction caused the hang.

---

## 11. Driver Overhead Reduction

### 11.1 Command Buffer Pre-Recording

Vulkan command buffers are re-recorded every frame in most tutorial-style code. For static geometry (terrain, skybox, static props), record the command buffer once and reuse it:

```c
// Record static command buffer once at scene load:
VkCommandBufferBeginInfo begin = {
    .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO,
    // No ONE_TIME_SUBMIT_BIT — buffer will be submitted multiple times
};
vkBeginCommandBuffer(static_cmd, &begin);
// ... record static draws ...
vkEndCommandBuffer(static_cmd);

// Each frame: submit static + dynamic command buffers together in one vkQueueSubmit2:
VkCommandBufferSubmitInfo cmds[] = {
    { .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_SUBMIT_INFO, .commandBuffer = static_cmd },
    { .sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_SUBMIT_INFO, .commandBuffer = dynamic_cmd },
};
```

### 11.2 Secondary Command Buffers for Render-Pass Parallelism

Secondary command buffers can be recorded in parallel on multiple CPU threads and then executed within a primary command buffer:

```c
// Primary command buffer:
vkCmdBeginRenderPass(primary, &rp_begin_info,
    VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);
vkCmdExecuteCommands(primary, num_secondary, secondary_cmds);
vkCmdEndRenderPass(primary);
```

Secondary command buffers must be created with the `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT` flag and inherit the render pass from the primary. This pattern amortises command recording CPU time across multiple threads.

### 11.3 Push Constants vs. UBO vs. SSBO Access Latency

| Data Path | Access Latency | Max Size | Notes |
|---|---|---|---|
| Push constants | Fastest — stored in GPU command packet header, zero memory access | 128–256 bytes (driver-dependent) | Best for per-draw matrices, indices |
| Uniform buffer (UBO) | L1/L2 cached — typically 1–4 cycles if hot | Spec minimum 16 KB; drivers allow much larger | Read-only, constant across a draw |
| Storage buffer (SSBO) | L2 cached, coherent — 4–40 cycles depending on cache hit | Effectively unlimited | Read/write, incoherent without barriers |

Use push constants for the MVP matrix, per-draw material index, and other small per-draw constants. Use UBOs for frame-global data (camera, lights). Use SSBOs for large per-object data arrays, writable outputs, and bindless resource tables.

### 11.4 VK_EXT_descriptor_buffer

`VK_EXT_descriptor_buffer` eliminates the CPU-side descriptor set binding path (`vkCmdBindDescriptorSets`). Descriptors are written directly to a GPU-visible `VkBuffer` as plain bytes, and the shader accesses them via device addresses. [Source: Ch157 Vulkan Descriptor Binding in Depth]

```c
// Get descriptor size for each type:
VkDeviceSize sampled_image_size;
VkDeviceSize storage_buffer_size;
vkGetDescriptorSetLayoutSizeEXT(device, layout, &sampled_image_size);

// Write descriptor directly into GPU buffer:
VkDescriptorImageInfo img_info = { .imageView = my_view, .imageLayout = SHADER_READ_ONLY };
VkDescriptorGetInfoEXT get_info = {
    .sType = VK_STRUCTURE_TYPE_DESCRIPTOR_GET_INFO_EXT,
    .type  = VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE,
    .data.pSampledImage = &img_info,
};
vkGetDescriptorEXT(device, &get_info, sampled_image_size,
    (char*)descriptor_buffer_ptr + binding_offset);

// Bind by device address — no vkCmdBindDescriptorSets:
VkDescriptorBufferBindingInfoEXT bind = {
    .sType   = VK_STRUCTURE_TYPE_DESCRIPTOR_BUFFER_BINDING_INFO_EXT,
    .address = descriptor_buffer_device_addr,
    .usage   = VK_BUFFER_USAGE_SAMPLER_DESCRIPTOR_BUFFER_BIT_EXT,
};
vkCmdBindDescriptorBuffersEXT(cmd, 1, &bind);
```

The main benefit is reducing the number of kernel-mode state updates: `vkCmdBindDescriptorSets` may require the driver to flush shader state or update hardware descriptor tables; `VK_EXT_descriptor_buffer` paths go through a simpler device-address update.

### 11.5 Indirect Draw and Dispatch for GPU-Driven Rendering

`vkCmdDrawIndexedIndirect` and `vkCmdDispatchIndirect` read their parameters from a `VkBuffer` rather than from CPU-supplied arguments. This enables the GPU to generate its own draw lists (via a culling compute shader) without CPU readback. See Chapter 154 for the full GPU-driven rendering pattern. The key overhead benefit: eliminating the GPU→CPU→GPU round-trip removes 2–3 frame latencies of `vkQueueWaitIdle` + readback + re-submit.

---

## 12. Vendor-Specific Tuning Notes

### 12.1 AMD RDNA

**Wave32 vs. Wave64**: RDNA defaults to wave32. Use `VK_AMD_shader_core_properties2` (`shaderNumWavesSupportedPerSIMD`) to query wave slot counts. Force wave64 with `AMD_DEBUG=w64cs` for comparison; most compute-heavy shaders benefit from wave32's lower divergence cost. [Source: Mesa envvars.html, https://docs.mesa3d.org/envvars.html]

**SGPR pressure**: Load wave-invariant data (buffer device addresses, dispatch-wide uniforms) via `ByteAddressBuffer` to keep them in SGPRs. Accessing a uniform through a VGPR costs one VGPR per thread per value; SGPRs cost one register for the entire 32-thread wave.

**NSA addressing** (Non-Sequential Addressing in texture instructions): RDNA 3 added NSA for `IMAGE_SAMPLE` instructions with non-contiguous register sources, reducing the register pressure for multi-component texture coordinate loads. Texture-heavy shaders on RDNA 3 (RX 7000) automatically benefit when Mesa's ACO compiler uses NSA. No application-level action needed; check RGP instruction timing to verify NSA instructions appear.

**VK_AMD_gcn_shader dot product ops**: `GL_AMD_gcn_shader` exposes `dot2Add`, `dot4Add`, and similar intrinsics that compile to a single GCN/RDNA ISA dot-product instruction (V_DOT4_U32_U8, etc.). Useful for summed-area tables, feature descriptors, and low-precision convolutions.

**LDS bank conflict debugging**: RGP's instruction timing view shows elevated latency for LDS instructions. If `DS_READ` or `DS_WRITE` instructions have 4× or more the expected latency, apply the padding strategy from §4.1.

### 12.2 Intel Xe

Intel Xe-HPG and Xe2 use an EU (Execution Unit) pair model: two EUs share an instruction dispatch slot, effectively a 16-wide SIMD executing two 8-wide operations in parallel. Subgroup size 16 maps cleanly to one EU pair; subgroup size 8 uses one EU; subgroup size 32 spans two EU pairs. Use `VK_EXT_subgroup_size_control` to request size 16 or 32 depending on register pressure.

**L3 cache partitioning**: Intel Xe2 L3 cache is shared between render and compute workloads. Competing render and compute dispatches can thrash L3 when both have high working-set sizes. Prefer serialising render+compute when the compute working set is large, rather than overlapping on the compute queue; measure with `intel_gpu_top` to confirm.

**`VK_INTEL_performance_query`**: Provides raw EU utilisation counters, L3 cache hit rates, and sampler throughput directly from the application, enabling in-production monitoring without external tools. Not to be confused with the KHR variant; the Intel extension has a different enumeration model. [Source: Vulkan spec, https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html]

### 12.3 NVIDIA Ampere / Ada

**L2 sector size**: NVIDIA Ada (RTX 4000 series) increased L2 cache to 72 MB on the RTX 4090. The L2 is sector-addressed (128-byte cache lines subdivided into 32-byte sectors for partial access). Accesses to 32-byte aligned, 32-byte regions avoid fetching unused sectors. For 4-component float (`vec4`) arrays, natural 16-byte alignment is sufficient; for 8-byte half-float pairs, alignment to 8 bytes avoids sector splits.

**Async copy engines**: Ampere exposes 5 CE (Copy Engine) units, Ada maintains this count. On Linux with the open-source NVK driver or proprietary driver, these appear as multiple Vulkan transfer queues. Streaming uploads from host to device can use multiple queues simultaneously for higher DMA throughput. (Note: NVK driver support for all 5 CEs is under active development as of mid-2024 — verify current support level.)

**Tensor core layout** (`sm_80`+, Ampere and later): the `m16n8k16` PTX MMA layout requires A in row-major, B in column-major fragment order. Via `VK_KHR_cooperative_matrix` on NVK or the proprietary driver, the layout requirements are expressed in the `VkCooperativeMatrixPropertiesKHR::AType` and `BType` fields. See Chapter 141 for details.

---

## 13. Integrations

This chapter is the performance counterpart to the API-focused chapters elsewhere in the book. The relationships are:

- **Chapter 25 — GPU Compute**: fundamentals of Vulkan compute pipeline creation, ICD discovery, and OpenCL/ROCm. Chapter 221 assumes those foundations and addresses what to do when that compute shader is slower than expected.
- **Chapter 133 — Vulkan Compute Queues and Task Graphs**: the multi-queue hardware model (AMD ACEs, Intel engines, NVIDIA CEs), timeline semaphore chaining, and frame graph organisation. Section 9 of this chapter is a performance-focused summary; Ch133 provides full implementation detail.
- **Chapter 141 — Vulkan Cooperative Matrices**: tensor core / matrix core programming via `VK_KHR_cooperative_matrix`. Section 8 of this chapter covers the occupancy trade-off; Ch141 covers the full API and hardware mapping.
- **Chapter 148 — Vulkan Synchronisation**: the complete barrier, event, semaphore, and fence model. Section 6 of this chapter covers barrier performance; Ch148 covers correctness and the full synchronisation model.
- **Chapter 154 — GPU-Driven Rendering**: indirect draw, GPU culling, and draw-count elimination. Section 11 of this chapter references the indirect dispatch pattern; Ch154 provides the full multi-pass culling pipeline.
- **Chapter 157 — Vulkan Descriptor Binding**: `VK_EXT_descriptor_buffer`, bindless descriptors, and push constant vs. UBO trade-offs. Section 11 of this chapter summarises the latency hierarchy; Ch157 covers driver-level implementation in RADV and ANV.
- **Chapter 30 — Debugging and Profiling**: the broader tool ecosystem including Validation Layers, RenderDoc, and Mesa's environment-variable debug flags. The profiling tool summary in Section 10 of this chapter focuses narrowly on performance counters and occupancy; Ch30 covers correctness-oriented debugging.
- **Chapter 15 — ACO: AMD's Optimising Compiler**: Mesa's register allocator, instruction scheduler, and RDNA ISA output. Understanding ACO's output (viewable in RGP's instruction timing view) bridges the gap between GLSL source and the ISA bottlenecks identified by profiling.
