# Chapter 106: The Vulkan Memory Model — Formal Execution and Memory Ordering

This chapter targets three audiences:

- **Vulkan application developers** — writing compute shaders and multi-queue applications who need to understand what correctness guarantees they are actually getting from barriers and atomics
- **shader compiler engineers** — implementing memory model lowering in NIR and ACO who must translate abstract semantics into hardware wait states
- **graphics researchers** — studying GPU memory consistency who need the formal definitions underlying the model

Readers are expected to be familiar with basic Vulkan command buffer recording and the SPIR-V binary format, but the memory model is treated from first principles because it differs substantially from C++ and from most concurrent-programming models developers encounter elsewhere.

---

## Table of Contents

- [Introduction: The Problem of GPU Parallelism](#introduction-the-problem-of-gpu-parallelism)
- [The Vulkan Memory Model Specification](#the-vulkan-memory-model-specification)
- [Execution Scopes](#execution-scopes)
- [Storage Classes and Visibility](#storage-classes-and-visibility)
- [Availability and Visibility Chains](#availability-and-visibility-chains)
- [Memory Operations in SPIR-V](#memory-operations-in-spir-v)
- [Subgroup Operations](#subgroup-operations)
- [Atomic Operations](#atomic-operations)
- [Barriers in Practice](#barriers-in-practice)
- [Shader Compiler Treatment: NIR and ACO](#shader-compiler-treatment-nir-and-aco)
- [Validation Tools](#validation-tools)
- [Common Bugs](#common-bugs)
- [Real-World Patterns](#real-world-patterns)
- [Integrations](#integrations)

---

## Introduction: The Problem of GPU Parallelism

A modern GPU runs tens of thousands of shader invocations in parallel, distributed across hundreds of compute units, each with its own register file, L0 instruction cache, L1 data cache, and execution pipelines. Unlike a CPU core executing a single thread, no GPU invocation can assume that memory written by another invocation is immediately visible to it — not because the hardware is broken, but because the hardware is aggressively optimised for throughput, not for sequential consistency.

The classical programmer's mental model — "I wrote to memory, then issued a barrier, so the other thread will see my write" — is **insufficient and formally undefined** on GPU hardware without understanding what a barrier actually guarantees. On x86 CPUs, the Total Store Order (TSO) model provides a strong baseline: all stores appear to all processors in a consistent order, and an `mfence` instruction serialises all outstanding loads and stores. C++11 `std::atomic<T>` operations with `memory_order_seq_cst` impose a total order on all sequentially consistent operations visible to all threads. These guarantees are expensive on CPUs but manageable because there are at most a few dozen threads competing for shared state.

GPUs offer none of these guarantees by default. The reasons are architectural:

1. **Non-coherent L1 caches.** Each compute unit has its own L1 vector cache. A store by invocation A sits in A's compute unit's L1 until that cache line is evicted or explicitly flushed. Invocation B, running on a different compute unit, reads from its own L1 — and will not see A's store until the data has been written back through the L2 and B's L1 has been invalidated.

2. **Out-of-order memory pipelines.** Vector memory instructions (VMEM) are issued into a deep in-flight pipeline. On AMD RDNA architectures there may be 64 or more VMEM operations in flight simultaneously. A later instruction that reads the same memory location may be issued before an earlier store has reached the L2.

3. **Scale makes global consistency prohibitively expensive.** A global sequentially consistent fence on a GPU with 256 compute units would require serialising all 256 L1 caches simultaneously, stalling the entire device for potentially thousands of cycles. Such a primitive would eliminate the parallelism that makes the GPU useful.

The Vulkan Memory Model is the formal specification that defines precisely what guarantees *are* provided, under what conditions, and through what mechanisms. It is not a performance pessimisation — it is the specification of the minimum contract that applications must satisfy to get correct results, without requiring pessimistic global serialisation.

### Contrast with C++ `memory_order`

The C++ memory model defines `memory_order_relaxed`, `memory_order_acquire`, `memory_order_release`, `memory_order_acq_rel`, and `memory_order_seq_cst`. These are operations on individual atomic objects and create happens-before edges between threads. The Vulkan model draws on the same theoretical framework but extends it in three critical ways:

- **Scopes:** C++ happens-before is global — if thread A releases and thread B acquires on the same atomic, the release is visible to every other thread too. Vulkan's model allows scoping synchronisation to a subset of invocations (e.g., only those within the same workgroup), reducing the cost of synchronisation when global visibility is not needed.
- **Storage classes:** C++ has a single flat memory space. Vulkan explicitly distinguishes storage buffers, uniform buffers, images, and shared (workgroup) memory, and requires that barriers and memory operations specify which storage classes they affect.
- **Availability/visibility operations:** C++ assume cache-coherent hardware. Vulkan explicitly models non-coherent caches through availability (cache writeback / flush) and visibility (cache invalidation) operations that must be paired correctly for cross-invocation communication to be defined.

Crucially, **`SequentiallyConsistent` memory semantics are not supported in Vulkan**. The Vulkan specification states this explicitly: *"SequentiallyConsistent memory semantics is not supported and must not be used."* [Source](https://docs.vulkan.org/spec/latest/appendices/memorymodel.html) The maximum ordering available is `AcquireRelease`, which the spec clarifies is treated as the effective replacement for sequential consistency in the Vulkan context.

---

## The Vulkan Memory Model Specification

The Vulkan Memory Model was introduced as the extension `VK_KHR_vulkan_memory_model` and promoted to core in **Vulkan 1.2**. The formal specification occupies its own appendix in the Vulkan specification titled *"Memory Model"* [Source](https://docs.vulkan.org/spec/latest/appendices/memorymodel.html). The companion SPIR-V extension is `SPV_KHR_vulkan_memory_model` [Source](https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_vulkan_memory_model.html).

The model defines four foundational concepts:

| Concept | Definition |
|---------|-----------|
| **Execution scope** | The set of invocations over which a synchronisation operation applies |
| **Storage class** | The category of memory (StorageBuffer, Workgroup, Image, etc.) that a memory operation acts on |
| **Memory domain** | A level in the hardware memory hierarchy (subgroup instance, workgroup instance, shader, device, host) |
| **Availability/visibility chain** | The sequence of operations required to make a write by one agent observable to a read by another agent |

### Enabling the Model in SPIR-V

A shader module that uses Vulkan memory model semantics must declare the capability at the top of the SPIR-V binary:

```spirv
; Enable the Vulkan memory model
OpCapability VulkanMemoryModel

; If Device scope synchronisation is needed:
OpCapability VulkanMemoryModelDeviceScope

; Required extension declaration
OpExtension "SPV_KHR_vulkan_memory_model"

; Memory model declaration — replace GLSL450 with Vulkan
OpMemoryModel Logical Vulkan
```

Without `OpCapability VulkanMemoryModel`, the shader is using the GLSL450 memory model, which has weaker and less formally defined guarantees. The `OpMemoryModel` instruction must specify `Vulkan` (not `GLSL450`) for the formal memory model semantics to apply.

### Feature Struct

Applications must query and enable the model through `VkPhysicalDeviceVulkanMemoryModelFeatures`:

```c
// Checking for Vulkan memory model support
VkPhysicalDeviceVulkanMemoryModelFeatures vmm_features = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_MEMORY_MODEL_FEATURES,
};
VkPhysicalDeviceFeatures2 features2 = {
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
    .pNext = &vmm_features,
};
vkGetPhysicalDeviceFeatures2(physicalDevice, &features2);

// Three independent feature bits:
// vmm_features.vulkanMemoryModel
//   - shader modules may declare OpCapability VulkanMemoryModel
// vmm_features.vulkanMemoryModelDeviceScope
//   - Device scope may be used in memory synchronisation operations
// vmm_features.vulkanMemoryModelAvailabilityVisibilityChains
//   - multi-element availability/visibility chains are permitted
```

All three feature bits are **optional** even in Vulkan 1.2 — the specification does not require implementations to expose them [Source](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPhysicalDeviceVulkan12Features.html). In practice, all desktop GPU drivers (RADV, ANV, NVK, NVIDIA proprietary) report all three as `VK_TRUE`. Older or lower-end mobile implementations may report `vulkanMemoryModel = VK_TRUE` but `vulkanMemoryModelDeviceScope = VK_FALSE` if the hardware lacks a globally coherent path for Device-scope atomics. Always query before use.

---

## Execution Scopes

An execution scope specifies *which set of invocations* are involved in a synchronisation operation. Narrower scopes require less hardware coordination and are therefore cheaper. The five scopes defined in the Vulkan spec [Source](https://docs.vulkan.org/spec/latest/appendices/memorymodel.html) are:

| SPIR-V Scope | Vulkan Concept | Hardware Concept (AMD RDNA) | Hardware Concept (NVIDIA) |
|---|---|---|---|
| `Invocation` | Single shader invocation | Single lane | Single thread |
| `Subgroup` | Invocations in the same subgroup | Single wavefront (32 or 64 lanes) | Single warp (32 threads) |
| `Workgroup` | Invocations in the same compute workgroup | Invocations across multiple wavefronts on one CU | Thread block across multiple warps on one SM |
| `QueueFamily` | All invocations submitted within one queue family | All compute engines sharing the same GDS access domain | All SMs sharing the same L2 partition |
| `Device` | All invocations on the device | Entire GPU, all CUs and engines | Entire GPU, all SMs |

Choosing the wrong scope is a correctness error, not merely a performance concern. Using `Workgroup` scope when you need `QueueFamily` means that writes are only guaranteed to propagate within the same workgroup — invocations in other workgroups dispatched concurrently have no obligation to observe them.

### SPIR-V Encoding

The `Scope` operand appears on `OpMemoryBarrier` and `OpControlBarrier`:

```spirv
; OpControlBarrier — execution + memory barrier
;   Execution Scope: Workgroup (2)
;   Memory Scope:    Workgroup (2)
;   Semantics:       AcquireRelease (0x8) | WorkgroupMemory (0x100) = 0x108 = 264
OpControlBarrier %uint_2 %uint_2 %uint_264

; OpMemoryBarrier — memory-only barrier for SSBO/UBO access at Device scope
;   Memory Scope:    Device (1)
;   Semantics:       AcquireRelease (0x8) | UniformMemory (0x40) = 0x48 = 72
;   (StorageBuffer is a storage class, not a MemorySemanticsMask bit;
;    UniformMemory covers both UBO and SSBO accesses for barrier purposes)
OpMemoryBarrier %uint_1 %uint_72
```

The SPIR-V `Scope` enumeration values are: `CrossDevice` = 0, `Device` = 1, `Workgroup` = 2, `Subgroup` = 3, `Invocation` = 4, `QueueFamily` = 5, `ShaderCallKHR` = 6. `QueueFamily` requires `OpCapability VulkanMemoryModel`.

---

## Storage Classes and Visibility

The Vulkan memory model distinguishes memory operations not only by scope but by the *storage class* of the pointer being accessed. Different storage classes correspond to different physical memory subsystems with different coherency properties.

| SPIR-V Storage Class | Physical Memory | Coherency |
|---|---|---|
| `StorageBuffer` | GPU VRAM or host-visible heap; accessed as SSBO | Non-coherent across CUs; requires explicit flush/invalidate |
| `Uniform` | Constant/uniform buffer memory | Read-only from shader; treated as cached descriptor |
| `Image` | Texture memory / L1 texture cache path | Coherency managed by image layout transitions |
| `PhysicalStorageBuffer` | Raw GPU VA pointer (BDA) | Same as StorageBuffer; requires `VK_KHR_buffer_device_address` |
| `Workgroup` | LDS (Local Data Share) / shared memory | Coherent within workgroup; coherency is hardware-enforced between lanes on same CU |

### The `NonPrivate` Requirement

By default, non-atomic memory operations are treated as **private**: the compiler may assume they are not intended for communication with other invocations and may hoist, sink, or reorder them freely. For cross-invocation communication to be defined behaviour, operations must be marked as **non-private** using:

- `NonPrivatePointer` (bit 0x20) on `Load`/`Store` memory operands — for buffer/pointer accesses
- `NonPrivateTexel` (bit 0x400) on `OpImageRead`/`OpImageWrite` image operands — for image accesses

Without these flags, there is no guarantee that a store by invocation A is observable to invocation B even after an `AcquireRelease` barrier, because the compiler is permitted to treat the store as local state that need not be flushed.

### GLSL/SPIR-V Example: SSBO Read with Correct Decoration

```glsl
// GLSL compute shader — correct cross-invocation SSBO read
#version 460
#extension GL_KHR_memory_scope_semantics : enable

layout(local_size_x = 64) in;
layout(set = 0, binding = 0, std430) coherent buffer DataBlock {
    uint data[];
} ssbo;

void main() {
    uint id = gl_GlobalInvocationID.x;
    // The 'coherent' qualifier on the buffer causes glslang/shaderc to emit
    // NonPrivatePointer on all accesses to this binding.
    // Without 'coherent', stores are private by default.
    uint val = ssbo.data[id ^ 1u];  // read neighbour's slot
    memoryBarrierBuffer();           // AcquireRelease | StorageBuffer
    ssbo.data[id] = val + 1u;
}
```

In the corresponding SPIR-V, each `OpLoad` and `OpStore` touching the coherent buffer will carry the `NonPrivatePointer` memory operand. Without the `coherent` qualifier (or the explicit `NonPrivatePointer` flag in hand-written SPIR-V), the memory model does not guarantee that the load observes the value written by the neighbouring invocation.

---

## Availability and Visibility Chains

The formal core of the Vulkan memory model is the concept of **availability and visibility operations** [Source](https://docs.vulkan.org/spec/latest/appendices/memorymodel.html#memory-model-availability-visibility). A write by one invocation becomes observable to a read by another invocation only when the write has been made *available* to a shared memory domain and then *visible* to the reading invocation.

### Formal Definitions

**Availability operation:** An operation that makes a write `(W, L)` — write `W` to location `L` — available to a set of memory domains. An availability operation that *happens-after* a write, and whose source scope includes the writer and the memory location, makes the write available in its destination domains.

**Visibility operation:** An operation that makes an available write `(W, L)` visible to a set of `(agent, reference, L)` tuples — i.e., to specific invocations that can observe it through specific references. A visibility operation that *happens-after* the corresponding availability operation makes the write visible to the reading invocations in its destination scope.

**Availability chain:** A sequence of availability operations advancing from a narrow domain (e.g., a specific invocation's private store) to progressively broader domains: subgroup instance → workgroup instance → shader domain → device domain → host domain.

**Visibility chain:** The corresponding sequence in reverse: from a broad domain toward the specific reading invocation.

### Step-by-Step Example: Producer → Consumer Across Workgroups

Consider two workgroups A and B dispatched in a single `vkCmdDispatch`. Workgroup A's invocation 0 writes a result to an SSBO; workgroup B's invocation 0 must read it. Without correct memory ordering, this is undefined behaviour even if B is dispatched after A finishes:

```
Time →
[Invocation A0]  store(ssbo[0] = 42)  →  MakeAvailable(QueueFamily) → barrier
                                                                         ↓ happens-before
[Invocation B0]                             MakeVisible(QueueFamily) →  load(ssbo[0])
```

The complete chain requires:

1. Invocation A0 performs `store` with `NonPrivatePointer` (marks the store as intended for cross-invocation communication).
2. The store must be followed by a **Release** operation with `MakePointerAvailableKHR` at scope `QueueFamily` (or `Device`). This is an availability operation: it flushes the value from A's compute unit's L1 through L2 to the shared memory domain.
3. A **happens-before** edge must exist between the Release and the corresponding Acquire. In a single-dispatch scenario this requires either (a) the two workgroups not running concurrently — guaranteed only if the implementation serialises them, which it need not — or (b) an inter-workgroup synchronisation mechanism such as global atomics or a second dispatch with a pipeline barrier between them.
4. Invocation B0 performs an **Acquire** operation with `MakePointerVisibleKHR` at scope `QueueFamily`. This is a visibility operation: it invalidates B's L1 cache entry for the SSBO address, so the subsequent load fetches from L2 (which now holds the value made available by A).
5. Invocation B0 performs the `load` with `NonPrivatePointer`.

If **either** step 2 or step 4 is missing, the chain is broken: A's write may remain in its compute unit's L1 (missing MakeAvailable), or B may read a stale value from its own L1 (missing MakeVisible). A barrier that establishes only an execution dependency — ordering — without memory semantics does not by itself flush or invalidate caches.

### Why a Barrier Alone Is Insufficient

This is the most common misconception. An `OpControlBarrier` with empty memory semantics (`None = 0x0`) guarantees only that all invocations in the execution scope reach the barrier before any of them proceeds. It says nothing about cache coherency. A store made before such a barrier by one invocation may still be invisibly cached in that invocation's compute unit's L1 after the barrier. The consumer invocation, having invalidated nothing, reads from its own stale L1 entry.

---

## Memory Operations in SPIR-V

The SPIR-V memory operand bitmask is the mechanism for expressing availability and visibility requirements at the instruction level [Source](https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_vulkan_memory_model.html). The relevant bits introduced by `SPV_KHR_vulkan_memory_model` are:

| Token | Bit | Meaning |
|-------|-----|---------|
| `MakePointerAvailableKHR` | 0x08 | Perform an availability operation on the pointed-to locations (use on `Store`) |
| `MakePointerVisibleKHR` | 0x10 | Perform a visibility operation on the pointed-to locations (use on `Load`) |
| `NonPrivatePointerKHR` | 0x20 | This access participates in inter-invocation ordering |
| `MakeTexelAvailableKHR` | 0x100 | Availability for `OpImageWrite` |
| `MakeTexelVisibleKHR` | 0x200 | Visibility for `OpImageRead` |
| `NonPrivateTexelKHR` | 0x400 | Non-private flag for image accesses |
| `VolatileTexelKHR` | 0x800 | Prevent compiler optimisation of image reads |

Additionally, the standard `Volatile` bit (0x01) on `Load`/`Store` prevents the compiler from caching or reordering the operation.

### Cross-Invocation Store with MakeAvailable

A complete SPIR-V snippet for a producer store that makes its write available at `QueueFamily` scope:

```spirv
; %ptr_ssbo is a pointer into StorageBuffer class
; %value is the uint to store
; Scope constant for QueueFamily = 5
%uint_5 = OpConstant %uint 5

; Store with:
;   NonPrivatePointerKHR (0x20) | MakePointerAvailableKHR (0x08) = 0x28
;   followed by the scope operand (QueueFamily = 5)
OpStore %ptr_ssbo %value MakePointerAvailableKHR|NonPrivatePointerKHR %uint_5
```

And the consumer load that makes the write visible:

```spirv
; Load with:
;   NonPrivatePointerKHR (0x20) | MakePointerVisibleKHR (0x10) = 0x30
;   followed by the scope operand (QueueFamily = 5)
%loaded_value = OpLoad %uint %ptr_ssbo MakePointerVisibleKHR|NonPrivatePointerKHR %uint_5
```

When `MakePointerAvailableKHR` is used on a `Store`, the semantics operand accompanying it must include `Release` or `AcquireRelease`. When `MakePointerVisibleKHR` is used on a `Load`, the semantics must include `Acquire` or `AcquireRelease`. These constraints are checked by `spirv-val`.

### Volatile

The `Volatile` memory operand on a `Load` or `Store` prevents the compiler from treating the operation as eligible for any optimisation — no hoisting, sinking, elimination, or reordering relative to other volatile operations. It is the SPIR-V equivalent of C `volatile`. Useful for spinlocks and polling loops but not a substitute for `NonPrivatePointer` + `MakePointerVisible` for cross-invocation visibility:

```spirv
; Volatile load — reads fresh from memory, but does NOT perform a visibility operation
; Still needs NonPrivatePointerKHR + MakePointerVisibleKHR for cross-invocation correctness
%val = OpLoad %uint %ptr_ssbo Volatile
```

---

## Subgroup Operations

Subgroup operations are the GPU equivalent of warp/wavefront-level SIMT intrinsics. They exploit the fact that invocations in the same subgroup execute in lockstep (on NVIDIA, always 32 lanes per warp; on AMD RDNA2+, 32 or 64 lanes per wavefront) and share registers visible to hardware-level SIMD shuffle instructions.

### `subgroupBarrier()` vs. `subgroupMemoryBarrierBuffer()`

In GLSL (with `GL_KHR_shader_subgroup_basic`), subgroup barrier functions map to `OpControlBarrier` and `OpMemoryBarrier` with `Subgroup` scope:

```glsl
// subgroupBarrier(): execution + memory barrier at Subgroup scope
// All lanes reach this point before any proceed.
// Also makes all memory classes available/visible within the subgroup.
subgroupBarrier();

// subgroupMemoryBarrierBuffer(): memory-only barrier for StorageBuffer at Subgroup scope
// Does NOT stall execution — only orders memory operations.
subgroupMemoryBarrierBuffer();

// subgroupMemoryBarrierShared(): barrier for Workgroup (shared) memory at Subgroup scope
subgroupMemoryBarrierShared();
```

Because all invocations in a subgroup execute in lockstep on current hardware, a `subgroupBarrier()` is almost always redundant in terms of execution ordering — they are already synchronised. Its utility is the **memory** component: ensuring that stores are flushed to a domain accessible to all lanes. For LDS/shared memory, hardware coherency within the wavefront often makes even this redundant in practice, though it is required by the spec for portability.

### Implicit Synchronisation in Subgroup Arithmetic

Subgroup reduction operations (`subgroupAdd`, `subgroupMax`, `subgroupAll`, `subgroupBallot`, etc.) have **implicit synchronisation semantics** at `Subgroup` scope. After `subgroupAdd(x)` returns, all lanes have contributed to and received the result — no explicit barrier is needed:

```glsl
#extension GL_KHR_shader_subgroup_arithmetic : enable

layout(local_size_x = 64) in;
layout(set = 0, binding = 0, std430) buffer Result {
    uint total;
};

void main() {
    uint my_val = gl_GlobalInvocationID.x & 0xFFu;
    // subgroupAdd: all lanes contribute, all receive the subgroup sum.
    // Implicit Subgroup-scope synchronisation; no barrier needed.
    uint subgroup_sum = subgroupAdd(my_val);
    if (subgroupElect()) {
        // Only one lane per subgroup reaches here.
        atomicAdd(total, subgroup_sum);
    }
}
```

### Divergence and Non-Uniform Control Flow

Subgroup operations require **convergent** execution — all lanes must reach the operation. When control flow diverges (e.g., an `if` branch that not all lanes take), the invocations that skip the branch are inactive. SPIR-V tracks this through the `Uniform` property: code reachable from a path where any lane may have diverged is *non-uniform*, and calling subgroup operations in non-uniform control flow is undefined behaviour:

```glsl
// BUG: non-uniform control flow breaks subgroupBallot
if (gl_LocalInvocationID.x % 2 == 0) {
    // Only even-indexed lanes reach here — the ballot is undefined
    // because not all subgroup lanes are active.
    uvec4 mask = subgroupBallot(true);  // UB: non-uniform CF
}

// CORRECT: compute the ballot before divergence
uvec4 even_mask = subgroupBallot(gl_LocalInvocationID.x % 2 == 0);
```

The `GL_EXT_shader_subgroup_extended_types` extension and the `reconvergence` model (`GL_EXT_maximal_reconvergence`) address reconvergence guarantees for structured control flow.

---

## Atomic Operations

Atomic operations are always **non-private** — the Vulkan spec states: *"Atomic operations are always considered non-private."* [Source](https://docs.vulkan.org/spec/latest/appendices/memorymodel.html) This means they always participate in inter-invocation ordering without needing an explicit `NonPrivatePointer` flag. However, an atomic's memory semantics — its acquire/release ordering — still controls whether it establishes a happens-before edge with other operations.

### GLSL Atomics and SPIR-V Encoding

```glsl
// GLSL atomic — operates on shared memory (Workgroup storage class)
layout(local_size_x = 64) in;
shared uint counter;

void main() {
    // atomicAdd: reads, adds, writes atomically.
    // In GLSL, atomics on 'shared' have Workgroup scope.
    uint old = atomicAdd(counter, 1u);
    barrier();  // execution + memory barrier to make counter value visible
    // ...
}
```

The corresponding SPIR-V for an atomic add on a StorageBuffer with `AcquireRelease` semantics:

```spirv
; OpAtomicIAdd on a StorageBuffer pointer with AcquireRelease semantics
; %ptr_counter: pointer into StorageBuffer storage class
; Scope: Device (1)
; Semantics: AcquireRelease (0x8) | UniformMemory (0x40) = 0x48 = 72
; Note: StorageBuffer is a storage class (not a MemorySemanticsMask bit).
;       UniformMemory (0x40) is the semantics bit that covers SSBO/UBO accesses.
%old_val = OpAtomicIAdd %uint %ptr_counter %uint_1 %uint_72 %uint_1
;                                          ^scope  ^semantics  ^value
```

A `None` (0x0) semantics atomic — a **relaxed atomic** — performs the read-modify-write atomically but establishes no happens-before edge. It is correct for counters where only the final count matters, not the ordering of operations between the counter modifications and other memory:

```spirv
; Relaxed atomic add — atomic but no ordering
%old_val = OpAtomicIAdd %uint %ptr_counter %uint_1 %uint_0 %uint_1
;                                          ^scope  ^None semantics
```

### Hardware Implementation

**AMD RDNA/GCN:** Buffer atomics (BUFFER_ATOMIC_ADD, BUFFER_ATOMIC_CMPSWAP, etc.) operate on L2 cache banks. On GCN and early RDNA, atomics return their old value after a round-trip through L2. AMD GDS (Global Data Share) provided a device-wide scratchpad supporting high-throughput global atomics, but it is not directly exposed to shader code through any public Vulkan API; it is used internally by the driver for certain global counter operations [Source: AMD GPU ISA documentation, `https://gpuopen.com/`].

**NVIDIA:** On Volta and later, shared memory (L1/SMEM) has hardware atomic support. Global memory atomics (to VRAM) transit through the L2 cache and are serialised there across all SMs. Floating-point atomics like `atomicAdd(float)` were not natively supported in hardware before sm_70; earlier hardware emulated them with a compare-exchange loop in the instruction scheduler.

### `atomicCompareExchange` — CAS

The compare-and-swap (CAS) operation is the foundation of lock-free algorithms. In GLSL it maps to `atomicCompSwap`; in SPIR-V it is `OpAtomicCompareExchange`:

```glsl
// Attempt to claim a "work item" atomically
uint expected = 0u;
uint desired  = 1u;
// If *ptr == expected, write desired; return old value in either case
uint prev = atomicCompSwap(ssbo.slots[id], expected, desired);
bool claimed = (prev == expected);
```

In SPIR-V, `OpAtomicCompareExchange` carries two separate memory semantic operands: one for the successful case (when the comparison succeeds and the swap occurs) and one for the failure case. This matches the C++ `std::atomic::compare_exchange_weak` with separate success/failure orderings.

---

## Barriers in Practice

GLSL provides a family of memory barrier functions that map to `OpMemoryBarrier` with specific storage class masks. Understanding the mapping is essential for writing correct compute shaders.

### `memoryBarrierBuffer()`

```glsl
// Makes all buffer (SSBO / UBO) writes available to the Device scope.
// Maps to: OpMemoryBarrier %uint_Device AcquireRelease|StorageBuffer|UniformMemory
memoryBarrierBuffer();
```

Use this when one invocation has written to an SSBO and another invocation (potentially on a different workgroup or compute unit) must read the result. On its own it only establishes the Release/availability side; the consumer needs a corresponding acquire (the same `memoryBarrierBuffer()` on the load side makes writes visible).

### `barrier()` — Execution + Memory Barrier for Workgroups

```glsl
// Combined execution + workgroup memory barrier.
// ALL invocations in the local workgroup must reach this point.
// Makes Workgroup (shared) memory available/visible to all invocations in the workgroup.
// Maps to: OpControlBarrier %uint_Workgroup %uint_Workgroup AcquireRelease|WorkgroupMemory
barrier();
```

`barrier()` in a compute shader is the standard mechanism for synchronising shared memory access between invocations in the same workgroup. It establishes both execution ordering (all reach it before any proceed) and workgroup-scope memory ordering (LDS writes are visible across wavefronts on the same CU after the barrier).

**Critical limitation:** `barrier()` only synchronises invocations **within the same workgroup**. It does nothing for cross-workgroup communication. Using `barrier()` to synchronise SSBO access between workgroups is a common bug.

### `memoryBarrierImage()`

```glsl
// Makes image store (imageStore) writes available/visible.
// Maps to: OpMemoryBarrier %uint_Device AcquireRelease|ImageMemory
memoryBarrierImage();
```

Required when using `imageStore` in one pass and `imageLoad` or texture sampling in a subsequent pass, **within the same shader stage**. Between draw calls or compute dispatches, image layout transitions (`VkImageMemoryBarrier2`) handle cache management at the command buffer level.

### `memoryBarrierShared()`

```glsl
// Memory-only barrier for Workgroup shared memory at Workgroup scope.
// Does NOT stall execution — invocations may continue after different amounts of work.
// Use when you need memory ordering but NOT execution synchronisation.
memoryBarrierShared();
```

### Common Mistake: `barrier()` Without `memoryBarrierBuffer()`

```glsl
// BUG: barrier() synchronises Workgroup memory, not StorageBuffer memory.
// After this barrier, ssbo writes from other workgroups are NOT guaranteed visible.
layout(set = 0, binding = 0, std430) buffer Data { uint data[]; };
void main() {
    data[gl_GlobalInvocationID.x] = compute_result();
    barrier();  // Only affects Workgroup (LDS) memory!
    // Reading data[other_workgroup_id] here is undefined behaviour —
    // the write may still be in the other workgroup's CU L1 cache.
    uint result = data[some_other_index];  // UB
}
```

The correct pattern for cross-workgroup SSBO communication requires either (a) separate compute dispatches with a `VkMemoryBarrier2` between them in the command buffer, or (b) global atomic coordination with matching `memoryBarrierBuffer()` calls:

```glsl
// CORRECT: separate dispatch pattern
// In the command buffer (C code):
//   vkCmdDispatch(cmd, ...) — workgroup A writes
//   vkCmdPipelineBarrier2(cmd, ...,
//       .srcStageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
//       .dstStageMask = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
//       .srcAccessMask = VK_ACCESS_2_SHADER_STORAGE_WRITE_BIT,
//       .dstAccessMask = VK_ACCESS_2_SHADER_STORAGE_READ_BIT)
//   vkCmdDispatch(cmd, ...) — workgroup B reads
```

---

## Shader Compiler Treatment: NIR and ACO

The path from SPIR-V memory semantics to hardware wait states involves two Mesa compiler stages: the NIR intermediate representation and (for AMD hardware) the ACO backend.

### NIR Barrier Representation

Mesa's NIR unified the multiple historical barrier intrinsics (the legacy per-type `nir_intrinsic_memory_barrier_buffer`, `nir_intrinsic_memory_barrier_shared`, etc.) into a single `nir_intrinsic_barrier` in Mesa 23.2. That intrinsic carries four index fields [Source: `https://android.googlesource.com/platform/external/mesa3d/+/59b4f010915c1e3748018f646a6649b541aa1b8c/src/compiler/nir/nir.h`]:

- `execution_scope`: a `nir_scope` enum value for the execution barrier (`NIR_SCOPE_NONE` if there is no execution stall component)
- `memory_scope`: a `nir_scope` enum value specifying the memory ordering scope
- `memory_semantics`: a `nir_memory_semantics` bitmask (Acquire/Release ordering + MakeAvailable/MakeVisible)
- `memory_modes`: a `nir_variable_mode` bitmask indicating which storage classes the barrier affects (e.g., `nir_var_mem_ssbo`, `nir_var_mem_shared`, `nir_var_mem_image`)

The `nir_scope` enum is Mesa's own, **distinct** from the SPIR-V scope numbering [Source: `https://android.googlesource.com/platform/external/mesa3d/+/59b4f010915c1e3748018f646a6649b541aa1b8c/src/compiler/nir/nir.h`]:

```c
// src/compiler/nir/nir.h — nir_scope enum
typedef enum {
    NIR_SCOPE_NONE,          // 0 — no execution scope
    NIR_SCOPE_INVOCATION,    // 1
    NIR_SCOPE_SUBGROUP,      // 2
    NIR_SCOPE_SHADER_CALL,   // 3 (ray tracing)
    NIR_SCOPE_WORKGROUP,     // 4
    NIR_SCOPE_QUEUE_FAMILY,  // 5
    NIR_SCOPE_DEVICE,        // 6
} nir_scope;

// nir_memory_semantics bitmask
typedef enum {
    NIR_MEMORY_ACQUIRE       = 1 << 0,
    NIR_MEMORY_RELEASE       = 1 << 1,
    NIR_MEMORY_ACQ_REL       = NIR_MEMORY_ACQUIRE | NIR_MEMORY_RELEASE,  // 0x3
    NIR_MEMORY_MAKE_AVAILABLE = 1 << 2,
    NIR_MEMORY_MAKE_VISIBLE  = 1 << 3,
} nir_memory_semantics;
```

Note: NIR scope values are **not** identical to SPIR-V scope values (where Device=1, Subgroup=3, Workgroup=2, QueueFamily=5). The `spirv_to_nir.c` front end translates between them. [Source: `https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/compiler/spirv/spirv_to_nir.c`]

A GLSL `barrier()` (compute workgroup barrier) lowers to:

```c
// NIR representation of barrier() in a compute shader
// (pseudo-code showing index field values)
nir_intrinsic_instr *bar = nir_intrinsic_instr_create(b->shader,
    nir_intrinsic_barrier);
nir_intrinsic_set_execution_scope(bar, NIR_SCOPE_WORKGROUP);
nir_intrinsic_set_memory_scope(bar, NIR_SCOPE_WORKGROUP);
nir_intrinsic_set_memory_semantics(bar, NIR_MEMORY_ACQ_REL);
nir_intrinsic_set_memory_modes(bar, nir_var_mem_shared);
```

A `memoryBarrierBuffer()` with Device scope lowers to:

```c
nir_intrinsic_set_execution_scope(bar, NIR_SCOPE_NONE);   // no execution barrier
nir_intrinsic_set_memory_scope(bar, NIR_SCOPE_DEVICE);
nir_intrinsic_set_memory_semantics(bar, NIR_MEMORY_ACQ_REL |
                                        NIR_MEMORY_MAKE_AVAILABLE |
                                        NIR_MEMORY_MAKE_VISIBLE);
nir_intrinsic_set_memory_modes(bar, nir_var_mem_ssbo | nir_var_mem_ubo);
```

NIR optimisation passes such as `nir_opt_acquire_release_barriers` (introduced in Mesa 25.2) [Source: `https://docs.mesa3d.org/relnotes/25.2.0.html`] can combine or eliminate redundant barriers — for example, two adjacent Release barriers with the same scope and modes can be merged into one — before the NIR is handed to the backend.

### ACO Wait-State Emission

ACO (AMD Compiler Optimizer) is the shader compiler backend used by RADV (the Mesa AMD Vulkan driver). It takes NIR as input and produces GCN/RDNA machine code [Source: `https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/compiler`].

AMD GPU memory pipelines have distinct counters tracking in-flight operations:

| Counter | Tracks | s_waitcnt operand |
|---------|--------|-------------------|
| `vmcnt` | In-flight vector memory loads (VMEM reads) | `vmcnt(N)` |
| `vscnt` | In-flight vector memory stores (VMEM writes) — GFX10+ only | `vscnt(N)` |
| `lgkmcnt` | In-flight LDS, GDS, constant buffer, and message operations | `lgkmcnt(N)` |
| `expcnt` | In-flight export operations (for graphics) | `expcnt(N)` |

A wait of `s_waitcnt vmcnt(0)` stalls until all outstanding VMEM loads have completed. A wait of `s_waitcnt lgkmcnt(0)` stalls until all LDS operations have completed. The ACO pass `aco_insert_waitcnt.cpp` performs a dataflow analysis that propagates outstanding operation counts through the control flow graph and inserts the minimum set of `s_waitcnt` instructions required to satisfy the memory ordering constraints expressed in the NIR barrier [Source: `https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/aco_insert_waitcnt.cpp`].

For a `memoryBarrierBuffer()` (Device scope, AcquireRelease, SSBO):

```
; AMD RDNA2 lowering of memoryBarrierBuffer()
; Wait for all outstanding VMEM loads to complete (release side: make writes available)
s_waitcnt vmcnt(0)
; On GFX10+: also wait for vector stores
s_waitcnt_vscnt null, 0
; L1 cache invalidation for the visibility side
buffer_gl1_inv       ; invalidate GL1 (L1 vector cache)
buffer_gl0_inv       ; invalidate GL0 (scalar cache) if needed
```

For a `barrier()` (Workgroup scope, AcquireRelease, LDS):

```
; AMD RDNA lowering of barrier()
; Wait for all outstanding LDS operations
s_waitcnt lgkmcnt(0)
; Hardware barrier across all wavefronts in the workgroup
s_barrier
```

The key distinction on GFX10+ (RDNA2) is the separation of VMEM loads (`vmcnt`) and VMEM stores (`vscnt`) into two independent counters, enabling the hardware to issue stores without blocking loads. A barrier that needs only to order loads need not wait for outstanding stores, improving throughput.

### Intel EU ISA Comparison

Intel's GPU architectures (Gen9 through Xe2/Battlemage) use Execution Units (EUs) that are organised into sub-slices sharing an L1 cache and slices sharing an L2. Memory synchronisation in Intel's compiler (IGC — Intel Graphics Compiler) uses `send` instructions carrying a message descriptor to the shared function (sampler, data port, etc.) with appropriate fence modes. A memory fence instruction targeting L3 flushes the data port and invalidates the L3 cache entry for the EU cluster. The Intel `fence` instruction (`send GatewayFence`) serialises outstanding memory operations similarly to `s_waitcnt vmcnt(0)` on AMD, but targets the Intel UMA-style cache hierarchy where GPU and CPU may share last-level cache.

---

## Validation Tools

### `VkPhysicalDeviceVulkanMemoryModelFeatures`

Query this struct (as shown in the [Specification section](#the-vulkan-memory-model-specification)) to determine what the implementation supports before enabling memory model features. All three fields (`vulkanMemoryModel`, `vulkanMemoryModelDeviceScope`, `vulkanMemoryModelAvailabilityVisibilityChains`) must be `VK_TRUE` before using the corresponding features.

### Synchronisation Validation Layer

The Vulkan Validation Layers (`VK_LAYER_KHRONOS_validation`) include a GPU-assisted synchronisation validation mode that intercepts all command buffer submissions and checks for synchronisation hazards at the API level:

```bash
# Enable synchronisation validation via environment variable
VK_LAYER_ENABLES=VALIDATION_CHECK_ENABLE_SYNCHRONIZATION_VALIDATION_BIT \
    ./my_vulkan_app

# Or via the layer settings file (vk_layer_settings.txt):
# khronos_validation.enables = VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT
```

This layer can detect missing barriers between render passes, incorrect image layouts, and some buffer hazard patterns. It operates at the `VkBuffer`/`VkImage` object level, not at the intra-shader invocation level — it cannot detect missing `NonPrivatePointer` decorations inside a compute shader.

### `spirv-val`

The `spirv-val` tool from the SPIRV-Tools suite validates SPIR-V modules against the specification. It checks memory model constraints including:

- `MakePointerAvailableKHR` must only appear on `Store` instructions, not `Load`
- `MakePointerVisibleKHR` must only appear on `Load` instructions, not `Store`
- When `MakePointerAvailableKHR` is set, the corresponding memory semantics must include `Release` or `AcquireRelease`
- `NonPrivatePointerKHR` must accompany `MakePointerAvailableKHR` or `MakePointerVisibleKHR`
- Scope operands must be valid for the declared execution model

```bash
# Validate a SPIR-V module for memory model correctness
spirv-val --target-env vulkan1.2 shader.spv

# Disassemble and inspect memory operands
spirv-dis shader.spv | grep -E "OpStore|OpLoad|MakePointer|NonPrivate"
```

---

## Common Bugs

### Bug 1: Missing `NonPrivatePointer` on Cross-Invocation Loads

```glsl
// BUG: coherent qualifier omitted
layout(set = 0, binding = 0, std430) buffer Data { uint values[]; };
// Without 'coherent', the glslang/shaderc compiler emits Load/Store WITHOUT NonPrivatePointerKHR.
// The memory model treats these as private — the compiler may cache the value in a register
// and never re-read from memory, even after a barrier.
uint v = values[neighbour_id];  // may return stale value
```

**Fix:** Add `coherent` to the buffer declaration, or in hand-written SPIR-V, always add `NonPrivatePointerKHR` to `Load` and `Store` on SSBO locations accessed from multiple invocations.

### Bug 2: Release Without a Matching Acquire in the Observer

```glsl
// Thread A — producer (correct Release)
ssbo.data[id] = result;
memoryBarrierBuffer();  // Release: makes write available

// Thread B — consumer (MISSING Acquire/visibility operation)
uint v = ssbo.data[id];  // No memoryBarrierBuffer() before the read!
// B's L1 may contain a stale value. The Release in A is useless without
// a corresponding Acquire (visibility operation) in B.
```

**Fix:** Thread B must also call `memoryBarrierBuffer()` before the read (or the load must carry `MakePointerVisibleKHR`) to perform the visibility operation that invalidates B's L1.

### Bug 3: Wrong Scope — Workgroup When Device Scope Is Needed

```glsl
// BUG: using workgroup-scope barrier for cross-workgroup SSBO visibility
groupshared uint partial_sum[64];
void main() {
    partial_sum[gl_LocalInvocationID.x] = compute();
    barrier();  // Workgroup scope — correct for partial_sum (LDS)

    if (gl_LocalInvocationID.x == 0) {
        // Write to SSBO for another workgroup to read
        ssbo.global_results[gl_WorkGroupID.x] = partial_sum[0];
        // BUG: no Device-scope memoryBarrierBuffer() here.
        // The write to ssbo may remain in this CU's L1 cache.
    }
}
// Another workgroup dispatched in the same vkCmdDispatch may not see the write.
```

**Fix:** After the SSBO write, call `memoryBarrierBuffer()` (Device scope), **and** ensure proper inter-dispatch synchronisation at the command buffer level.

### Bug 4: Subgroup Ballot Under Divergence

```glsl
// BUG: conditional subgroup operation with divergent control flow
void main() {
    bool do_work = (gl_GlobalInvocationID.x % 3 == 0);
    if (do_work) {
        // Only 1/3 of lanes are active here.
        // subgroupBallot assumes all lanes are active — result is undefined.
        uvec4 active = subgroupBallot(true);  // UB: non-uniform CF
    }
}
```

**Fix:** Compute the ballot before any divergence, or use `subgroupBallot(do_work)` outside the `if` block.

### Bug 5: Relaxed Atomics Used for Synchronisation

```glsl
// BUG: using a relaxed atomic as a ready flag
layout(std430) buffer Flags { uint ready; };
layout(std430) buffer Data  { uint value; };

// Producer invocation:
atomicStore(data.value, 42u);   // no GLSL atomicStore — using atomicExchange
atomicExchange(flags.ready, 1u);  // GLSL: relaxed atomic by default in GL_EXT_shader_atomic_int64
// The atomicExchange has None semantics — NO Release, NO MakeAvailable.
// There is no happens-before edge between the store to data.value
// and the exchange on flags.ready.

// Consumer invocation:
while (atomicLoad(flags.ready) == 0u) {}  // spin
// Even when the loop exits, data.value may be stale in this invocation's L1.
uint v = data.value;  // UB: data.value may not be visible
```

**Fix:** The producer atomic must use `Release` semantics (and `MakePointerAvailableKHR` with `NonPrivatePointerKHR` on the store to `data.value`). The consumer atomic must use `Acquire` semantics (and `MakePointerVisibleKHR` on the load of `data.value`). In GLSL, use the `GL_KHR_memory_scope_semantics` extension functions:

```glsl
#extension GL_KHR_memory_scope_semantics : enable
// Producer:
atomicStore(data.value, 42u, gl_ScopeDevice, gl_StorageSemanticsBuffer,
            gl_SemanticsRelease | gl_SemanticsMakeAvailable);
atomicStore(flags.ready, 1u, gl_ScopeDevice, gl_StorageSemanticsBuffer,
            gl_SemanticsRelease);
// Consumer:
uint r;
do {
    r = atomicLoad(flags.ready, gl_ScopeDevice, gl_StorageSemanticsBuffer,
                   gl_SemanticsAcquire);
} while (r == 0u);
uint v = atomicLoad(data.value, gl_ScopeDevice, gl_StorageSemanticsBuffer,
                    gl_SemanticsAcquire | gl_SemanticsMakeVisible);
```

---

## Real-World Patterns

### Pattern 1: Producer-Consumer Between Workgroups Using SSBO + Atomics

This pattern is typical of multi-pass GPU pipelines where one dispatch writes results and another reads them. The correct approach uses two separate dispatches with a `vkCmdPipelineBarrier2` in between:

```c
// Command buffer recording (C):
// --- Dispatch 1: producers write to ssbo_results ---
vkCmdDispatch(cmd, producer_groups_x, 1, 1);

// Pipeline barrier: wait for compute writes before compute reads
VkMemoryBarrier2 mem_barrier = {
    .sType         = VK_STRUCTURE_TYPE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_2_SHADER_STORAGE_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .dstAccessMask = VK_ACCESS_2_SHADER_STORAGE_READ_BIT,
};
VkDependencyInfo dep = {
    .sType              = VK_STRUCTURE_TYPE_DEPENDENCY_INFO,
    .memoryBarrierCount = 1,
    .pMemoryBarriers    = &mem_barrier,
};
vkCmdPipelineBarrier2(cmd, &dep);

// --- Dispatch 2: consumers read from ssbo_results ---
vkCmdDispatch(cmd, consumer_groups_x, 1, 1);
```

### Pattern 2: Work Queue Drain with `atomicAdd` Counter

A common GPU-driven pattern uses a single atomic counter as a work queue head:

```glsl
// work_queue_drain.comp — single dispatch with atomic work stealing
#version 460
#extension GL_KHR_memory_scope_semantics : enable

layout(local_size_x = 64) in;

layout(set = 0, binding = 0, std430) buffer WorkQueue {
    uint head;          // atomic counter: next work item index
    uint total_items;   // total number of work items
    uint items[];       // work item data
} wq;

layout(set = 0, binding = 1, std430) buffer Results {
    uint results[];
};

void main() {
    while (true) {
        // Atomically claim the next work item.
        // Acquire semantics: after this load, we can safely read items[idx].
        uint idx = atomicAdd(wq.head, 1u,
                             gl_ScopeDevice,
                             gl_StorageSemanticsBuffer,
                             gl_SemanticsAcquire);
        if (idx >= wq.total_items)
            break;

        // Process the work item. No barrier needed — Acquire on the
        // atomicAdd ensures items[idx] is visible.
        uint item_data = wq.items[idx];
        results[idx] = item_data * item_data;
    }
}
```

### Pattern 3: GPU-Driven Draw Count Accumulation for `vkCmdDrawIndexedIndirectCount`

GPU-driven rendering uses compute shaders to perform frustum/occlusion culling and write visible draw counts into a buffer consumed by `vkCmdDrawIndexedIndirectCount`. Correct memory ordering is essential [Source: `https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#drawing-mesh-shaders`]:

```glsl
// culling.comp — accumulate visible draw count
#version 460
layout(local_size_x = 64) in;

layout(set = 0, binding = 0, std430) readonly buffer Draws {
    DrawCommand cmds[];  // input draw commands
};
layout(set = 0, binding = 1, std430) buffer OutDraws {
    uint draw_count;
    DrawCommand out_cmds[];  // packed visible commands
};
layout(set = 0, binding = 2) uniform CullingParams { mat4 view_proj; };

bool is_visible(DrawCommand cmd) {
    // ... frustum test against cmd.bounds ...
    return true;
}

void main() {
    uint id = gl_GlobalInvocationID.x;
    if (id >= cmds.length()) return;

    DrawCommand cmd = cmds[id];
    if (is_visible(cmd)) {
        // Atomically claim a slot in the output array.
        // Release semantics: the write to out_cmds[slot] must be visible
        // before the draw_count is observed by the indirect draw call.
        uint slot = atomicAdd(draw_count, 1u,
                              gl_ScopeDevice,
                              gl_StorageSemanticsBuffer,
                              gl_SemanticsAcquireRelease);
        out_cmds[slot] = cmd;
    }
}
```

After this dispatch completes, a `vkCmdPipelineBarrier2` with `srcAccessMask = VK_ACCESS_2_SHADER_STORAGE_WRITE_BIT` and `dstAccessMask = VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT` (and `dstStageMask = VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT`) ensures that the draw count and command data are visible to the indirect draw stage:

```c
VkMemoryBarrier2 indirect_barrier = {
    .sType         = VK_STRUCTURE_TYPE_MEMORY_BARRIER_2,
    .srcStageMask  = VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_2_SHADER_STORAGE_WRITE_BIT,
    .dstStageMask  = VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT,
    .dstAccessMask = VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT,
};
```

Without this barrier, the GPU may begin reading the indirect command buffer for the draw call while the compute shader's writes are still in L1 cache, producing incorrect draw counts or uninitialized draw parameters — exactly the class of bug that produces intermittent corruption that passes on some frames and drivers.

---

## Roadmap

### Near-term (6–12 months)

- **Vulkan Roadmap 2026 milestone baseline consolidation:** The Vulkan Roadmap 2026 milestone (announced by Khronos in 2025) mandates Vulkan 1.4 as a baseline requirement, which includes the Vulkan Memory Model as a core feature. All Roadmap 2026-conformant drivers on desktop and mid-to-high-end mobile must report `vulkanMemoryModel = VK_TRUE`. This effectively ends optional-feature fragmentation for the model on targeted device classes. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)

- **Mesa 25.x driver conformance hardening:** RADV, ANV, and NVK all shipped Vulkan 1.4 conformance in Mesa 25.0, which subsumes `VK_KHR_vulkan_memory_model` as a promoted core feature. Ongoing work in Mesa 25.1 and beyond focuses on Vulkan CTS (Conformance Test Suite) coverage for memory model edge cases, particularly Device-scope atomics and multi-queue availability/visibility chains. [Source](https://docs.mesa3d.org/relnotes/25.1.0.html)

- **`VK_EXT_descriptor_heap` interaction with memory visibility:** The new descriptor heap extension, actively seeking developer feedback before finalisation as a KHR extension, bypasses traditional descriptor set indirection. The memory ordering implications — particularly whether descriptor reads require explicit visibility operations — are under active discussion in the Vulkan Working Group. [Source](https://www.khronos.org/blog/vulkan-introduces-roadmap-2026-and-new-descriptor-heap-extension)

- **Improved Vulkan validation layer coverage for memory model bugs:** The Khronos Validation Layer (`VK_LAYER_KHRONOS_validation`) continues expanding detection of incorrect memory scope usage and missing availability/visibility operations. Note: specific patch tracking needs verification against the Vulkan-ValidationLayers GitHub issue tracker.

### Medium-term (1–3 years)

- **Formal memory model testing tooling:** Academic and industry research (e.g., the TriCheck and GPU memory consistency verification work at SIGARCH) is being applied to GPU shader memory models. Integration of automated litmus-test generators into Vulkan CTS to cover `OpMemoryBarrier` scope combinations is a stated goal in the GPU memory consistency research community. [Source](https://www.sigarch.org/gpu-memory-consistency-specifications-testing-and-opportunities-for-performance-tooling/)

- **SPIR-V memory model extensions for heterogeneous compute:** SPIR-V serves as the portable IR for OpenCL, Vulkan, and emerging SYCL/HIP front-ends. Khronos discussions around unified memory ordering semantics across compute APIs (reducing divergence between `SPV_KHR_vulkan_memory_model` and OpenCL memory model semantics) are ongoing. Note: specific RFC numbers need verification against the KhronosGroup/SPIRV-Registry GitHub.

- **Subgroup scope tightening for wave/warp portability:** AMD RDNA, NVIDIA, and Intel Xe hardware differ in their subgroup sizes and the stability of subgroup membership. Proposals to formalise `gl_SubgroupSize` guarantees and their interaction with `Subgroup`-scoped atomics are expected to produce a follow-on SPIR-V extension. Note: needs verification against current Khronos working group drafts.

- **ACO and NIR barrier optimisation advances in Mesa:** As GPU compute workloads grow more complex (task/mesh shaders, ray tracing, work graphs), the NIR pass `nir_opt_acquire_release_barriers` and ACO's `aco_insert_waitcnt.cpp` are expected to gain dataflow-aware barrier elimination to reduce unnecessary wait states while remaining formally correct under the memory model. [Source](https://deepwiki.com/bminor/mesa-mesa/2.1-radv-(amd-vulkan-driver))

### Long-term

- **Convergence with C++ `std::atomic` on GPU:** Long-term research directions aim to close the semantic gap between C++ `memory_order` operations executed on CPUs (via OpenMP/SYCL offload) and Vulkan/SPIR-V memory model operations on GPUs within the same heterogeneous program, enabling unified happens-before reasoning across CPU and GPU threads. The SIGARCH GPU memory consistency survey identifies this as an open research challenge. [Source](https://www.sigarch.org/gpu-memory-consistency-specifications-testing-and-opportunities-for-performance-tooling/)

- **Hardware-enforced memory model validation:** As GPU hardware grows more sophisticated, future architectures may expose optional hardware support for memory model assertion checking (analogous to Intel's TSX transaction debugging features), allowing drivers to trap memory ordering violations at the hardware level rather than relying solely on software validation layers. Note: speculative direction, no public roadmap commitment confirmed.

- **Vulkan memory model extension to mesh/task shader inter-stage communication:** Mesh and task shaders (part of `VK_EXT_mesh_shader`, promoted into Vulkan 1.4 ecosystem) introduce a new inter-stage payload memory space. The formal memory ordering guarantees for this payload — which invocations can observe it, and when — are currently defined informally in prose rather than through the rigorous availability/visibility chain model. A future specification revision or extension is expected to formalise this. [Source](https://docs.vulkan.org/spec/latest/appendices/memorymodel.html)

---

## Integrations

**Ch14 — NIR (Mesa Intermediate Representation):** NIR is the intermediate representation through which all SPIR-V memory model semantics flow on their way to hardware-specific code generation. The `nir_intrinsic_barrier` with `execution_scope`, `memory_scope`, `memory_semantics`, and `memory_modes` fields is the direct encoding of the Vulkan memory model within the compiler. SPIR-V front-end translation in `spirv_to_nir.c` maps `OpControlBarrier`/`OpMemoryBarrier` operands to NIR index fields. NIR optimisation passes such as `nir_opt_acquire_release_barriers` can legally eliminate or coalesce barriers only when they can prove the memory model contract remains satisfied.

**Ch15 — ACO (AMD Compiler Optimizer):** ACO's `aco_insert_waitcnt.cpp` pass is the final translation step that converts NIR barrier semantics into AMD ISA wait-state instructions (`s_waitcnt vmcnt(0)`, `s_waitcnt_vscnt null, 0`, `s_waitcnt lgkmcnt(0)`, `s_barrier`). The correctness of ACO's barrier lowering is a direct requirement of the Vulkan memory model — any missed `s_waitcnt` produces undefined behaviour visible as rendering corruption or compute result errors.

**Ch133 — Vulkan Compute Queues and Task Graphs:** Task graph frameworks that dispatch compute work across multiple queues rely on the Vulkan memory model primitives covered in this chapter. The task graph scheduler inserts `vkCmdPipelineBarrier2` nodes at data-flow edges between task nodes, and the correctness of those barriers — the right `srcStageMask`, `dstStageMask`, `srcAccessMask`, `dstAccessMask` — is an application of the availability/visibility chain model. Compute queues from separate queue families introduce additional considerations: cross-queue memory dependencies require `VkSemaphore` signals with appropriate memory scopes.

**Ch148 — Vulkan Synchronisation Reference:** Ch148 covers the full Vulkan synchronisation API — fences, semaphores, pipeline barriers, render pass subpass dependencies, and events — which provides the **execution ordering** layer that the memory model operates within. The memory model defines what correctness means for shared memory access; the synchronisation API (especially `vkCmdPipelineBarrier2` with `VkMemoryBarrier2`) is the mechanism by which command-buffer-level availability and visibility operations are expressed. Ch148 and this chapter are complementary: Ch148 answers "which Vulkan call do I use?", this chapter answers "what does it formally guarantee?"

**Ch154 — GPU-Driven Rendering:** GPU-driven culling shaders (the canonical example of which is the draw count accumulation pattern in the [Real-World Patterns](#real-world-patterns) section above) rely critically on atomic memory ordering for `vkCmdDrawIndexedIndirectCount` to produce correct results. The `AcquireRelease` semantics on the `atomicAdd` that writes the draw count, combined with the `VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT` barrier before the draw, form the complete availability/visibility chain from compute write to indirect draw read. Ch154 examines these patterns from the rendering architecture perspective; this chapter provides the formal foundation that explains why each component of that synchronisation is necessary.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
