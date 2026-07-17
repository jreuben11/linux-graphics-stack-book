# Chapter 120: GPU Memory Management Internals — TTM, GEM, and BAR

> **Part**: Part I — The Kernel Layer
> **Audience**: Kernel GPU driver developers and systems engineers who need to understand how GPU memory is allocated, moved between domains, evicted under pressure, and shared across drivers. Assumes familiarity with Ch4's introduction to GEM object lifecycle, DMA-BUF, and dma_resv fencing; this chapter goes deeper into the internals.
> **Status**: First draft — 2026-06-19

---

## Table of Contents

- [1. Introduction — Two Frameworks, One Problem](#1-introduction--two-frameworks-one-problem)
- [2. GPU Memory Topology](#2-gpu-memory-topology)
- [3. GEM Internals: Shmem, DMA-Contiguous, and Driver-Specific Variants](#3-gem-internals-shmem-dma-contiguous-and-driver-specific-variants)
- [4. TTM: Translation Table Manager in Depth](#4-ttm-translation-table-manager-in-depth)
- [5. The gpu_buddy / drm_buddy VRAM Allocator](#5-the-gpu_buddy--drm_buddy-vram-allocator)
- [6. drm_gpuvm and drm_exec: GPU Virtual Memory Management](#6-drm_gpuvm-and-drm_exec-gpu-virtual-memory-management)
- [7. Resizable BAR and Smart Access Memory](#7-resizable-bar-and-smart-access-memory)
- [8. VRAM Eviction Under Memory Pressure](#8-vram-eviction-under-memory-pressure)
- [9. DMA-BUF Cross-Driver Sharing: The Kernel View](#9-dma-buf-cross-driver-sharing-the-kernel-view)
- [10. Debugging GPU Memory](#10-debugging-gpu-memory)
- [Integrations](#integrations)
- [References](#references)

---

## 1. Introduction — Two Frameworks, One Problem

GPU memory management is one of the most complex subsystems in the Linux graphics stack. Unlike CPU memory — governed by the buddy allocator, slab caches, and the virtual memory subsystem — GPU memory must simultaneously manage four distinct domains:

- **VRAM**: on-card GDDR6X or HBM3 DRAM, the fastest memory for GPU access (up to ~1.5 TB/s on high-end RDNA4/Blackwell hardware), but limited in size (8–192 GB) and not directly accessible by the CPU except through a narrow PCIe BAR window.
- **System RAM (GTT / TT)**: accessible by the GPU through an IOMMU DMA mapping or a legacy GART aperture, roughly 50–100 GB/s, and usable as a spill location for buffer objects (BOs) when VRAM is full.
- **PCIe BAR**: a CPU-visible aperture into VRAM, historically limited to 256 MB by 32-bit MMIO decode constraints, expanded by Resizable BAR (ReBAR) to the full VRAM size on modern platforms.
- **Cross-process buffer sharing**: any of the above domains can hold a buffer that must be importable by a different process or a different kernel driver.

The Linux kernel provides two competing frameworks for managing these domains. **GEM** (Graphics Execution Manager, merged in Linux 2.6.28, 2009) is the simpler, driver-centric model: it defines a common object lifecycle and handle-based access control, but leaves memory placement entirely to the driver. **TTM** (Translation Table Manager), originally developed for vmwgfx and adopted by amdgpu, nouveau, and other complex drivers, adds a generalised multi-tier memory manager with automated eviction, per-placement LRU lists, and a sophisticated locking protocol.

Layered on top of TTM, two newer subsystems have emerged. **drm_buddy** (later generalised into the `gpu_buddy` allocator at `drivers/gpu/buddy.c`, Linux 5.15+) is a power-of-two buddy allocator for VRAM address space. **drm_gpuvm** (Linux 6.7) provides a generic framework for tracking GPU virtual address space mappings, replacing per-driver implementations like `amdgpu_vm_bo_map`.

Chapter 4 of this book covers:

- **GEM object lifecycle**
- **DMA-BUF**
- **dma_resv implicit fencing**
- **PRIME multi-GPU sharing**
- **GBM**
- **format modifiers**

This chapter goes deeper into internals that Ch4 only sketches:

- **TTM placement and eviction machinery**
- **buddy allocator's internals**
- **drm_gpuvm/drm_exec locking protocol**
- **Resizable BAR upgrade path**
- **debugging interfaces** — let you observe GPU memory state in production

---

## 2. GPU Memory Topology

Understanding the physical topology is prerequisite to understanding why the memory management subsystems are designed as they are.

### VRAM: GPU-Local DRAM

Modern discrete GPUs carry their own DRAM stack, directly attached to the GPU die or chiplet. GDDR6X (used in NVIDIA GeForce RTX 40/50 series and AMD RX 7000/9000 series) achieves approximately 576–800 GB/s. HBM3 and HBM3e (used in AMD Instinct MI300X, NVIDIA H100/H200) achieve 3.2–5.2 TB/s thanks to wide-bus stacked DRAM placed directly on the same package interposer as the GPU.

The GPU has its own memory management unit (MMU) which translates GPU virtual addresses to physical addresses in VRAM or system RAM. On AMD RDNA hardware this is the VMHUB; on NVIDIA Turing+ it is the GMMU. The MMU page table format is entirely vendor-specific and is programmed by the kernel driver.

From the CPU's perspective, VRAM is opaque memory on the other side of a PCIe link. The CPU accesses it exclusively through the PCIe BAR aperture (see §7).

### System RAM as a GPU Memory Domain (GTT / TT)

Every TTM-based driver exposes a `TTM_PL_TT` domain representing system RAM pages that the GPU can access via DMA. On x86 systems with an IOMMU (AMD Vi / Intel VT-d), the driver calls `dma_map_sg()` or programs the IOMMU page tables directly to create a scatter-gather-capable GPU-visible mapping. On legacy or embedded platforms without an IOMMU, drivers use a GART (Graphics Address Remapping Table) aperture to present physically discontiguous system pages as a contiguous GPU address range.

The GPU accesses GTT memory at PCIe bandwidth: Gen4 x16 delivers ~64 GB/s bidirectional. This is fast enough for streaming workloads but insufficient for GPU-local computation on textures or vertex buffers. GTT is therefore used as:

1. A spill location for VRAM overflow during eviction.
2. A staging area for CPU-side uploads before a DMA blit to VRAM.
3. A permanent home for command buffers and ring buffers (where CPU write latency matters more than GPU read bandwidth).

### PCIe BAR and the CPU Aperture

Every PCIe device exposes one or more Base Address Registers (BARs) in its PCI configuration space. The system BIOS/UEFI maps these into physical memory address space during boot, allowing the CPU to read and write them via MMIO. For GPU VRAM:

- **BAR 0** is the VRAM aperture (for amdgpu; NVIDIA uses a similar arrangement).
- **BAR 2** or **BAR 4** may expose doorbell registers, MMIO control registers, or extended apertures.

Before Resizable BAR support (see §7), PCI specifications required BARs used for 32-bit MMIO decode to fit within 32-bit address space, effectively limiting BAR0 to 256 MB even on GPUs with 8–80 GB of VRAM. The driver must therefore "window" the BAR: only 256 MB of VRAM is CPU-visible at any given time, and large CPU→VRAM uploads must go through GTT staging.

### Topology Diagram

```
┌───────────────────────────────────────────────────────────────────┐
│  CPU + System RAM                                                  │
│  ┌──────────────┐    IOMMU/GART    ┌─────────────────────────────┐│
│  │  Page Tables │ ─────────────── ▶│  DMA-mapped GTT pages       ││
│  └──────────────┘                  └─────────────────────────────┘│
│         │                                       │                  │
│         │  PCIe Gen4/5 x16                      │ PCIe DMA        │
│         ▼                                       ▼                 │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  GPU                                                         │ │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────────────┐ │ │
│  │  │ GPU MMU  │  │ COMPUTE   │  │ VRAM (GDDR6X / HBM3)    │ │ │
│  │  │ (VMHUB)  │─▶│ ENGINES   │─▶│ up to 1.5 TB/s          │ │ │
│  │  └──────────┘  └───────────┘  └──────────────────────────┘ │ │
│  │       │                                                      │ │
│  │  BAR0 aperture (CPU→VRAM, PCIe bandwidth ~32–64 GB/s)       │ │
│  └──────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

---

## 3. GEM Internals: Shmem, DMA-Contiguous, and Driver-Specific Variants

Ch4 introduced `struct drm_gem_object` and `drm_gem_object_funcs` in detail. Rather than repeating that ground, this section focuses on the specialised GEM helper types that drivers actually use and the specific internals that matter for debugging and driver development.

### drm_gem_shmem_object: The Common Heap-Backed Case

For drivers that do not use TTM (e.g., panfrost, v3d, etnaviv, lima), `drm_gem_shmem_object` is the standard way to implement GEM objects backed by anonymous shared memory (`shmem_file_setup`). The struct embeds `drm_gem_object`:

```c
/* Source: include/drm/drm_gem_shmem_helper.h */
struct drm_gem_shmem_object {
    struct drm_gem_object base;
    struct mutex pages_lock;
    struct page **pages;
    unsigned int pages_use_count;
    int pages_mark_dirty_on_put;
    int pages_mark_accessed_on_put;
    struct sg_table *sgt;          /* scatter-gather table for DMA */
    struct list_head madv_list;
    unsigned int madv;
    bool pinned;
};
```

The shmem path supports `madvise`-style memory pressure hints (`MADV_DONTNEED`), allowing the kernel to reclaim pages under memory pressure. This is the mechanism behind `drm_gem_shmem_purge()` and the `drm_gem_object_status()` callback that drivers expose to userspace via fdinfo.

[Source](https://github.com/torvalds/linux/blob/master/include/drm/drm_gem_shmem_helper.h)

### drm_gem_dma_object: DMA-Contiguous for Embedded Platforms

Display controllers and video IP on embedded SoCs often require physically contiguous memory. `drm_gem_dma_object` allocates via `dma_alloc_attrs()` which, when `CONFIG_DMA_CMA` is enabled, uses the Contiguous Memory Allocator (CMA) to satisfy the contiguity constraint:

```c
/* Source: include/drm/drm_gem_dma_helper.h */
struct drm_gem_dma_object {
    struct drm_gem_object base;
    dma_addr_t dma_addr;    /* device (bus) address of the allocation */
    void *vaddr;            /* kernel virtual address (always mapped) */
    bool map_noncoherent;   /* true for non-coherent DMA on ARM */
    struct sg_table *sgt;
};
```

The `dma_addr` is what gets programmed into hardware scan-out registers. The `vaddr` is always CPU-accessible, which distinguishes this helper from the TTM VRAM case where the CPU aperture may not cover the full buffer.

[Source](https://github.com/torvalds/linux/blob/master/include/drm/drm_gem_dma_helper.h)

### Driver-Embedded GEM Objects: amdgpu, i915, nouveau

Each major driver defines a BO type that embeds `drm_gem_object` (for GEM-only drivers) or `ttm_buffer_object` which itself embeds `drm_gem_object`:

- `struct amdgpu_bo` — embeds `ttm_buffer_object`, adds `amdgpu_bo_flags` (domain, pin count, tiling flags), AMDGPU-specific domain bitmask (`AMDGPU_GEM_DOMAIN_VRAM`, `AMDGPU_GEM_DOMAIN_GTT`, `AMDGPU_GEM_DOMAIN_GDS`, `AMDGPU_GEM_DOMAIN_DOORBELL`), and a pointer to `struct amdgpu_bo_va` for each VA mapping.
- `struct nouveau_bo` — embeds `ttm_buffer_object`, adds NVKM memory domain info and a per-channel push-buffer flag.
- `struct i915_gem_object` — does **not** embed TTM; i915 rolled its own memory manager (`i915_ttm_object` for discrete memory on DG1/DG2/Arc). The `i915_gem_object` carries `struct intel_memory_region *mm` for placement policy.

This embedding relationship means that `container_of(ttm_bo, struct amdgpu_bo, tbo)` is the standard idiom for recovering the driver-specific BO from a TTM callback parameter.

---

## 4. TTM: Translation Table Manager in Depth

TTM generalises multi-domain GPU memory management. Introduced for vmwgfx and later adopted by amdgpu, nouveau, and i915 (for discrete/lmem objects), TTM's core design decision is to treat memory domains as named **resource managers** and to handle placement, eviction, and migration uniformly.

### Core Structures

**`ttm_device`** is the per-device context. It owns an array of resource managers indexed by memory type constant:

```c
/* Source: include/drm/ttm/ttm_device.h */
struct ttm_device {
    struct ttm_device_funcs *funcs;
    struct ttm_resource_manager *man_drv[TTM_NUM_MEM_TYPES]; /* [0..63] */
    spinlock_t lru_lock;
    struct mutex io_reserve_mutex;
    struct ttm_pool pool;   /* system page pool */
    struct dentry *debugfs; /* /sys/kernel/debug/dri/N/ttm_... */
    /* ... */
};
```

Memory type constants are small integers: `TTM_PL_SYSTEM` (0), `TTM_PL_TT` (1), `TTM_PL_VRAM` (2), `TTM_PL_PRIV` (3). Drivers can also allocate additional private types up to `TTM_NUM_MEM_TYPES` (64).

**`ttm_resource_manager`** represents one memory domain:

```c
/* Source: include/drm/ttm/ttm_resource.h */
struct ttm_resource_manager {
    bool use_type;
    bool use_tt;             /* true if this type uses ttm_tt mapping */
    uint64_t size;           /* total capacity in pages */
    const struct ttm_resource_manager_func *func;
    struct ttm_lru_bulk_move *bulk_move;
    struct list_head lru[TTM_MAX_BO_PRIORITY]; /* per-priority LRU lists */
    struct dma_fence *move_fences[TTM_NUM_MOVE_FENCES];
};
```

[Source](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html)

**`ttm_buffer_object`** is the central TTM object. Every TTM-using driver's BO embeds this struct:

```c
/* Source: include/drm/ttm/ttm_bo.h */
struct ttm_buffer_object {
    struct drm_gem_object base;   /* GEM object — must be first */
    struct ttm_device *bdev;
    enum ttm_bo_type type;        /* ttm_bo_type_device, _kernel, _sg */
    struct ttm_resource *resource; /* current placement */
    struct ttm_tt *ttm;            /* system-RAM page table for TT domain */
    unsigned pin_count;
    unsigned priority;
    struct ttm_lru_bulk_move *bulk_move;
    struct work_struct delayed_delete;
    struct sg_table *sg;           /* for ttm_bo_type_sg */
};
```

**`ttm_resource`** describes where a BO currently lives:

```c
/* Source: include/drm/ttm/ttm_resource.h */
struct ttm_resource {
    unsigned long start;              /* offset within the memory type (in pages) */
    size_t size;                      /* size in bytes */
    uint32_t mem_type;                /* TTM_PL_* constant */
    uint32_t placement;               /* flags from ttm_place.flags */
    struct ttm_bus_placement bus;     /* CPU-accessible bus mapping info */
    struct ttm_buffer_object *bo;     /* weak back-reference to owning BO */
    struct ttm_lru_item lru;          /* LRU list node */
};
```

### Placement Specification

A driver expresses placement policy as a `struct ttm_placement` carrying an ordered array of `struct ttm_place` preferences. TTM tries each in order:

```c
/* Source: include/drm/ttm/ttm_placement.h */
struct ttm_place {
    uint64_t fpfn;      /* first permissible page number (0 = no constraint) */
    uint64_t lpfn;      /* last permissible page number (0 = no constraint) */
    uint32_t mem_type;  /* TTM_PL_VRAM, TTM_PL_TT, TTM_PL_SYSTEM, ... */
    uint32_t flags;     /* TTM_PL_FLAG_DESIRED, TTM_PL_FLAG_FALLBACK,
                         * TTM_PL_FLAG_CONTIGUOUS, TTM_PL_FLAG_TOPDOWN */
};

struct ttm_placement {
    unsigned num_placement;
    const struct ttm_place *placement; /* ordered preference list */
};
```

Note that **`TTM_PL_FLAG_WC` no longer exists**. Write-combining (important for BAR-mapped VRAM) is now specified separately through the `ttm_caching` enum, which resource managers apply when setting up CPU mappings:

```c
/* Source: include/drm/ttm/ttm_caching.h */
enum ttm_caching {
    ttm_uncached,         /* no write combining, most conservative */
    ttm_write_combined,   /* write-combining; used for VRAM BAR mappings */
    ttm_cached,           /* fully cached, requires CPU snoop of GPU writes */
};
```

A typical amdgpu placement for a VRAM-preferred buffer that can fall back to GTT:

```c
/* Source: drivers/gpu/drm/amd/amdgpu/amdgpu_object.c (simplified) */
static const struct ttm_place vram_placement_flags[] = {
    { .mem_type = TTM_PL_VRAM, .flags = TTM_PL_FLAG_DESIRED },
    { .mem_type = TTM_PL_TT,   .flags = TTM_PL_FLAG_FALLBACK },
};

static const struct ttm_placement vram_preferred = {
    .num_placement = ARRAY_SIZE(vram_placement_flags),
    .placement     = vram_placement_flags,
};
```

### ttm_bo_validate: Moving a BO to the Desired Placement

The key function for BO migration is `ttm_bo_validate()`. If the BO is already in an acceptable placement, it is a no-op. Otherwise it calls into the resource manager to allocate in the new domain, executes the move via a GPU blit or CPU memcpy (via the driver's `ttm_device_funcs::move` callback), and waits for or attaches a fence:

```c
/* Source: include/drm/ttm/ttm_bo.h */
int ttm_bo_validate(struct ttm_buffer_object *bo,
                    struct ttm_placement *placement,
                    struct ttm_operation_ctx *ctx);
```

`ttm_operation_ctx` carries flags for interruptible waits, no-wait (trylock) semantics, and whether the caller is in a memory reclaim context (which suppresses further eviction recursion to prevent stack overflows).

### BO Reservation: The ww_mutex Locking Protocol

Before any code can modify a TTM BO (change its placement, submit it in a GPU job, or read its resource pointer), it must **reserve** the BO. Reservation acquires a per-BO `ww_mutex` — a wound-wait mutex designed to prevent deadlock when multiple BOs must be locked simultaneously:

```c
/* Source: include/drm/ttm/ttm_bo.h */
static inline int ttm_bo_reserve(struct ttm_buffer_object *bo,
                                 bool interruptible,
                                 bool no_wait,
                                 struct ww_acquire_ctx *ticket);

static inline void ttm_bo_unreserve(struct ttm_buffer_object *bo);
```

The `ticket` parameter is the ww_acquire context. When locking multiple BOs for a batch submission, the driver acquires a single `ww_acquire_ctx` via `ww_acquire_init(&ticket, &reservation_ww_class)`, then locks each BO with that ticket. If a deadlock would occur, one loser is "wounded" — its `ttm_bo_reserve` returns `-EDEADLK` — and the driver must unlock all previously acquired BOs, back off, and retry. The `drm_exec` helper (see §6) automates this retry loop.

### TTM Pinning

BOs that must not be evicted (display framebuffers, GPU ring buffers) are pinned with `ttm_bo_pin()` / `ttm_bo_unpin()`. A pinned BO has `pin_count > 0` and is excluded from the LRU eviction scan. Drivers must ensure that the pin count is symmetric — every `ttm_bo_pin` call must be paired with an `ttm_bo_unpin` — to avoid permanently leaking VRAM.

---

## 5. The gpu_buddy / drm_buddy VRAM Allocator

TTM's resource managers need an underlying allocator to manage the address space within each memory domain. For VRAM, a buddy allocator is ideal: GPU hardware has natural power-of-two alignment requirements, and the buddy algorithm provides O(log N) allocation and coalescing.

### History: From drm_buddy to gpu_buddy

The buddy allocator was originally implemented as `drm_buddy` in `drivers/gpu/drm/drm_buddy.c` (merged in Linux 5.15). In a subsequent kernel release (the split commits appear in the 6.x series — check `git log --oneline drivers/gpu/buddy.c` for the exact version), the core allocator logic was promoted to a generic subsystem at `drivers/gpu/buddy.c` with the public API renamed to `gpu_buddy_*`, reflecting its use beyond DRM (e.g., by the Xe display engine). The header `include/drm/drm_buddy.h` still exists as a thin wrapper for DRM drivers, but the actual struct types and function signatures now live in `include/linux/gpu_buddy.h`.

[Source](https://github.com/torvalds/linux/blob/master/include/linux/gpu_buddy.h)

### Core Structures

```c
/* Source: include/linux/gpu_buddy.h (condensed — exact bit layout
 * implementation-defined; header encodes block offset, order, state,
 * and a "cleared" flag in a compact u64) */
struct gpu_buddy_block {
    u64 header;   /* encodes: block offset, buddy order, allocation state,
                   * and clear/zeroed bit — exact field positions are
                   * internal to the allocator */
    /* embedded in the per-order free list or allocated list */
};

struct gpu_buddy {
    struct mutex lock;
    u64 size;                    /* total managed size in bytes */
    struct list_head *free;      /* per-order free lists [0..MAX_ORDER] */
    struct list_head *clear_list;
    unsigned long *roots;        /* bitmask of root blocks */
    u64 avail;                   /* bytes currently available */
    /* ... */
};
```

### Allocation API

```c
/* Source: include/linux/gpu_buddy.h */
int gpu_buddy_alloc_blocks(struct gpu_buddy *mm,
                           u64 start,          /* start of allowed range (bytes) */
                           u64 end,            /* end of allowed range (bytes) */
                           u64 size,           /* requested allocation size */
                           u64 min_page_size,  /* minimum block size (alignment) */
                           struct list_head *blocks, /* output: allocated block list */
                           unsigned long flags);

void gpu_buddy_free_list(struct gpu_buddy *mm,
                         struct list_head *objects,
                         unsigned int flags);
```

**Flags** control allocation behaviour:

| Flag | Meaning |
|------|---------|
| `GPU_BUDDY_RANGE_ALLOCATION` | Restrict to `[start, end)` address range |
| `GPU_BUDDY_TOPDOWN_ALLOCATION` | Allocate from high addresses downward |
| `GPU_BUDDY_CONTIGUOUS_ALLOCATION` | Require a single contiguous block (no fragmentation) |
| `GPU_BUDDY_CLEAR_ALLOCATION` | Prefer blocks already zeroed (security-sensitive allocations) |
| `GPU_BUDDY_CLEARED` | Mark freed blocks as cleared on free |
| `GPU_BUDDY_TRIM_DISABLE` | Suppress automatic splitting/trimming |

The output is a `list_head` of `gpu_buddy_block` entries — an allocation may span multiple non-contiguous blocks unless `GPU_BUDDY_CONTIGUOUS_ALLOCATION` is set. This is important: a large GPU texture allocation might be split across two buddy blocks if VRAM is fragmented, and the driver must build a scatter-gather mapping that covers both.

### How AMDGPU Uses gpu_buddy for VRAM

AMDGPU's VRAM resource manager (`struct amdgpu_vram_mgr`) is backed by a `gpu_buddy` instance:

```c
/* Source: drivers/gpu/drm/amd/amdgpu/amdgpu_vram_mgr.c */
struct amdgpu_vram_mgr {
    struct ttm_resource_manager manager;
    struct gpu_buddy mm;          /* the buddy allocator for VRAM address space */
    spinlock_t lock;
    atomic64_t vis_usage;         /* visible (BAR-accessible) VRAM usage */
};
```

Initialisation:

```c
/* Source: amdgpu_vram_mgr.c — amdgpu_vram_mgr_init() */
gpu_buddy_init(&mgr->mm, man->size, PAGE_SIZE);
```

Allocation (`amdgpu_vram_mgr_new` callback, invoked by TTM `ttm_bo_validate`):

```c
/* Source: amdgpu_vram_mgr.c — simplified */
ret = gpu_buddy_alloc_blocks(&mgr->mm,
                             fpfn << PAGE_SHIFT,
                             lpfn << PAGE_SHIFT,
                             size,
                             min_block_size,
                             &vres->blocks,
                             vres->flags);
```

The `vres->flags` are derived from the `ttm_place.flags` and may include `GPU_BUDDY_CONTIGUOUS_ALLOCATION` if the buffer requires a physically contiguous VRAM range (needed for display scanout or certain PCIe peer-to-peer operations).

### Buddy Algorithm: Splitting and Coalescing

The buddy system manages VRAM as a binary tree of power-of-two-sized blocks. An initial root block covers the entire VRAM range. Allocation of size S:

1. Find the smallest order O such that 2^O pages ≥ S.
2. If no free block of order O exists, split a block of order O+1 into two buddies of order O.
3. Return one buddy; place the other on the order-O free list.

On free, the released block checks whether its buddy is also free. If so, the two buddies coalesce into a block of order O+1, and the process repeats upward. This O(log N) coalescing prevents fragmentation over time.

The `gpu_buddy_block.header` encodes the block's offset and order in a compact u64, so the full state of the allocator can be printed as a sequence of offset:order pairs — visible in the AMDGPU debugfs output (see §10).

---

## 6. drm_gpuvm and drm_exec: GPU Virtual Memory Management

GPU drivers must track every mapping from GPU virtual address space to a backing GEM BO. Before Linux 6.7, each driver maintained its own data structures: `amdgpu_vm_bo_map`, `xe_vm_bind_op`, `panthor_gpuva`, etc. The **`drm_gpuvm`** subsystem (merged in Linux 6.7, contributed by Danilo Krummrich) provides a generic, lockable, maple-tree-backed representation of a GPU's virtual address space.

[Source](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html)

### Core Structures

**`drm_gpuvm`**: the GPU virtual address space for one GPU context or VM. Backed by a maple tree for fast range queries.

```c
/* Source: include/drm/drm_gpuvm.h (condensed) */
struct drm_gpuvm {
    const char *name;
    struct drm_device *drm;
    struct drm_gem_object *r_obj;  /* root BO (kernel-allocated page tables) */
    struct {
        u64 addr;
        u64 range;
    } mm_start;                    /* kernel-reserved VA region */
    struct kref kref;
    struct dma_resv *resv;         /* shared reservation object */
    struct list_head evict;        /* BOs to be evicted before next exec */
    struct list_head extobj;       /* external (non-VM-local) BOs */
    const struct drm_gpuvm_ops *ops;
};
```

**`drm_gpuva`**: a single VA→BO mapping:

```c
/* Source: include/drm/drm_gpuvm.h */
struct drm_gpuva {
    struct drm_gpuvm *vm;
    struct drm_gpuvm_bo *vm_bo;   /* link to the (vm, obj) pair */
    struct {
        u64 addr;
        u64 range;
    } va;                          /* GPU virtual address range */
    struct {
        struct drm_gem_object *obj;
        u64 offset;               /* byte offset into BO */
    } gem;
    /* rb-tree node inside drm_gpuvm */
};
```

**`drm_gpuvm_bo`**: the abstraction linking one GEM BO to one VM. A BO can be mapped at multiple VA ranges within the same VM, but they all share the same `drm_gpuvm_bo` link:

```c
/* Source: include/drm/drm_gpuvm.h */
struct drm_gpuvm_bo {
    struct drm_gpuvm *vm;
    struct drm_gem_object *obj;
    struct kref kref;
    struct {
        bool evicted;       /* true: scheduled for eviction before next exec */
        struct list_head list;  /* on drm_gpuvm.evict list */
    } evict;
    struct list_head list;      /* on drm_gem_object.gpuva.list */
    struct list_head extobj;    /* on drm_gpuvm.extobj list */
    struct list_head gpuvas;    /* drm_gpuva list for this (vm, obj) pair */
};
```

### Obtaining a vm_bo Link

Before mapping a BO into a VM, the driver must obtain or create the `drm_gpuvm_bo` link:

```c
/* Source: include/drm/drm_gpuvm.h */
struct drm_gpuvm_bo *drm_gpuvm_bo_obtain(struct drm_gpuvm *gpuvm,
                                          struct drm_gem_object *obj);

struct drm_gpuvm_bo *drm_gpuvm_bo_obtain_prealloc(struct drm_gpuvm_bo *vm_bo);
```

`drm_gpuvm_bo_obtain` looks up the existing `vm_bo` for `(gpuvm, obj)` in `obj->gpuva.list`; if none exists, it allocates one. `drm_gpuvm_bo_obtain_prealloc` is used when the driver has pre-allocated a `vm_bo` (to avoid GFP_KERNEL allocations in atomic contexts) and wants to install it only if no existing link is found.

### VM_BIND and the State Machine

`drm_gpuvm` supports the **VM_BIND** execution model, where userspace explicitly binds/unbinds BO ranges to GPU VA ranges rather than relying on implicit mapping. The `drm_gpuvm_ops` vtable provides callbacks for the state machine steps:

```c
/* Source: include/drm/drm_gpuvm.h */
struct drm_gpuvm_ops {
    int (*vm_free)(struct drm_gpuvm *gpuvm);
    int (*sm_step_map)(struct drm_gpuva_op *op, void *priv);
    int (*sm_step_remap)(struct drm_gpuva_op *op, void *priv);
    int (*sm_step_unmap)(struct drm_gpuva_op *op, void *priv);
    /* ... */
    int (*vm_bind_prepare)(struct drm_gpuvm_exec *vm_exec);
    int (*vm_bind)(struct drm_gpuvm_exec *vm_exec);
};
```

`sm_step_map`, `sm_step_remap`, `sm_step_unmap` are called for each atomic step of a bind/unbind/remap operation computed against the existing VA space maple tree.

### drm_exec: The Multi-BO Locking Helper

Submitting GPU work requires locking all BOs referenced by the job (to attach fences and prevent concurrent eviction). `drm_exec` automates the ww_mutex retry protocol:

```c
/* Source: include/drm/drm_exec.h */
struct drm_exec {
    u32 flags;
    struct ww_acquire_ctx ticket;
    unsigned int num_objects;
    unsigned int max_objects;
    struct drm_gem_object **objects;   /* dynamically grown array */
    struct drm_gem_object *contended;  /* the BO that triggered -EDEADLK */
    struct drm_gem_object *prelocked;  /* the BO pre-locked before retry */
};
```

The canonical usage pattern for a GPU job submission:

```c
/* Source: pattern from drivers/gpu/drm/xe/ and drivers/gpu/drm/panthor/ */
struct drm_exec exec;
drm_exec_init(&exec, DRM_EXEC_INTERRUPTIBLE_WAIT, 0);

drm_exec_until_all_locked(&exec) {
    /* Lock the GPU VM's root object first */
    ret = drm_exec_prepare_obj(&exec, vm->r_obj, 1);
    drm_exec_retry_on_contention(&exec);
    if (ret)
        goto err;

    /* Lock each BO referenced by this job */
    drm_gpuvm_for_each_va_in_job(gpuva, job) {
        ret = drm_exec_prepare_obj(&exec, gpuva->gem.obj, 1);
        drm_exec_retry_on_contention(&exec);
        if (ret)
            goto err;
    }
}

/* Submit the job, attach out-fence to all locked BOs */
ret = submit_job_and_attach_fence(job, &exec);

err:
drm_exec_fini(&exec);  /* releases all ww_mutex locks */
```

`drm_exec_until_all_locked` is a macro that loops until the `contended` field is NULL (no outstanding deadlock wounds). `drm_exec_retry_on_contention` checks whether the current lock step generated a wound and, if so, jumps back to the top of the `drm_exec_until_all_locked` loop after unlocking all previously acquired BOs.

`drm_gpuvm_exec` is a thin wrapper that combines `drm_exec` with the `drm_gpuvm` pointer and an optional `vm_bind_prepare` callback, used specifically for VM_BIND operations:

```c
/* Source: include/drm/drm_gpuvm.h */
struct drm_gpuvm_exec {
    struct drm_exec exec;
    u32 flags;
    struct drm_gpuvm *vm;
    struct {
        unsigned int num_fences;
        struct drm_gpuvm_exec_func *fn;  /* optional extra BO locking */
    } extra;
};
```

[Source](https://github.com/torvalds/linux/blob/master/include/drm/drm_gpuvm.h)

---

## 7. Resizable BAR and Smart Access Memory

### The Historical Constraint

Under the original PCIe BAR specification, a device's BAR must fit within the 32-bit MMIO address space if the device signals 32-bit decode capability. In practice, BIOS/UEFI historically mapped GPU BAR0 (the VRAM aperture) at a 32-bit physical address, limiting it to 256 MB even on GPUs with 16–80 GB of VRAM.

The consequence for GPU memory management: the CPU could only directly read or write the lowest 256 MB of VRAM through the BAR. CPU→VRAM uploads for large textures had to go through a two-step process:

1. CPU writes to GTT (system RAM): one memcpy at system RAM bandwidth (~50 GB/s).
2. GPU DMA blit from GTT to VRAM: one GPU-initiated DMA transfer at PCIe bandwidth (~32–64 GB/s).

Both steps are serialised by fences, so the effective upload path incurs significant latency in addition to the bandwidth cost.

### Resizable BAR: The PCIe Mechanism

The PCIe Resizable BAR capability (defined in PCIe Base Specification 4.0, §7.8.6) allows a device to advertise a set of supported BAR sizes as a bitmask. The BIOS/UEFI or the OS can then select a larger size if address space is available. The kernel exposes this through `pci_resize_resource()`:

```c
/* Source: drivers/pci/pci.c */
int pci_resize_resource(struct pci_dev *dev, int resno, int size);
/* resno: BAR index (0 = BAR0); size: log2(size_in_MB) - 20 */
```

AMDGPU performs this resize during device initialisation. The current entry point is `amdgpu_device_resize_fb_bar()` in `drivers/gpu/drm/amd/amdgpu/amdgpu_device.c`, which replaced the older per-generation `amdgpu_resize_bar0()` (GFX7/8-era) function. The modern function handles all supported ASICs:

```c
/* Source: drivers/gpu/drm/amd/amdgpu/amdgpu_device.c */
int amdgpu_device_resize_fb_bar(struct amdgpu_device *adev);
/* Attempts to resize BAR0 to cover the full VRAM extent.
 * Requires a 64-bit root bus window; returns early (no-op) on 32-bit
 * architectures without high memory, to avoid integer overflow in the
 * resource size computation.
 * On success, adev->gmc.aper_size == adev->gmc.real_vram_size. */
```

The historical (2017) per-generation implementation showed the core mechanic clearly. The `pci_resize_resource` path remains the same:

```c
/* Historical reference: drivers/gpu/drm/amd/amdgpu/ — GFX7/8 pattern
 * (Source: https://lkml.iu.edu/hypermail/linux/kernel/1703.2/05968.html) */
u32 size = max(ilog2(adev->mc.real_vram_size - 1) + 1, 20) - 20;
r = pci_resize_resource(adev->pdev, 0, size);
/* Error handling: -ENOTSUPP (hardware), -ENOSPC (address space full) */
```

[Source: current function](https://lkml.rescloud.iu.edu/2307.0/06189.html) — [historical GFX7/8 implementation](https://lkml.iu.edu/hypermail/linux/kernel/1703.2/05968.html)

### AMD Smart Access Memory (SAM)

AMD's marketing term **Smart Access Memory** (SAM), introduced with the Ryzen 5000 + RDNA2 platform in 2020, refers to enabling full ReBAR on AMD Advantage and desktop platforms via a BIOS toggle. Technically, SAM is identical to PCIe ReBAR — the same `pci_resize_resource()` path in the kernel. AMD's contribution was ensuring that:

- The BIOS allocates the full VRAM aperture in 64-bit PCI address space (avoiding the 32-bit decode constraint).
- The AGESA firmware initialises the CPU memory map to include the large BAR.
- `amdgpu` detects the full BAR and marks the VRAM resource manager as CPU-visible for the full VRAM extent.

On success, `adev->gmc.aper_size` equals `adev->gmc.real_vram_size`, and the CPU can directly write to any VRAM address through `adev->mman.aper_base_kaddr + offset`.

### CPU Caching for BAR Mappings

VRAM accessed through a PCIe BAR is non-coherent with the CPU caches. The driver must map it write-combining to aggregate CPU writes into PCIe TLPs efficiently:

```c
/* Source: drivers/gpu/drm/amd/amdgpu/amdgpu_ttm.c */
/* When creating a CPU mapping of a VRAM resource: */
if (mem_type == TTM_PL_VRAM)
    caching = ttm_write_combined;  /* enum ttm_caching — no WC flag anymore */
```

Write-combining mode batches CPU stores into 64-byte write-combining buffers (WCBs) before flushing them as burst PCIe writes. Without write-combining, each store would generate a separate PCIe transaction, degrading effective write bandwidth by 10–100x.

### Performance Impact

With ReBAR enabled, the CPU→VRAM upload path collapses to a single step: the CPU memcpy directly into the BAR-mapped VRAM. This eliminates the GTT staging buffer and the GPU DMA blit. Benchmarks on RDNA2/RDNA3 hardware show:

- **Texture streaming workloads**: 5–15% frame rate improvement in bandwidth-limited games.
- **Asset streaming (open-world games)**: up to 20% reduction in load time for large texture sets.
- **VBO updates**: significant improvement for CPU-driven particle systems or morphing meshes.

The gain is bounded by PCIe bandwidth (~32–64 GB/s for Gen4/Gen5 x16), which remains well below VRAM bandwidth (~1 TB/s). Compute-bound or texture-bound workloads that are not upload-limited see negligible benefit.

---

## 8. VRAM Eviction Under Memory Pressure

When a new buffer object cannot fit in VRAM (because the buddy allocator returns `-ENOMEM`), TTM must evict existing BOs to free space. This eviction path is one of the most latency-sensitive and deadlock-prone code paths in the entire graphics stack.

### The LRU Eviction List

Every TTM resource manager maintains per-priority LRU lists (`struct list_head lru[TTM_MAX_BO_PRIORITY]`). When a BO is unreserved and not pinned, it is placed (or re-placed) at the tail of its priority's LRU list. TTM scans from the head (oldest) for eviction candidates.

Priorities (`TTM_MAX_BO_PRIORITY` = 4) let drivers hint that some BOs (e.g., kernel-owned ring buffers at priority 3) should be evicted last. Userspace BOs typically sit at priority 0 or 1.

### Eviction Mechanics

The eviction path is driven by `ttm_bo_evict_first()` (for single BO eviction) and `ttm_lru_walk_for_evict()` (for bulk eviction to satisfy a large allocation):

```c
/* Source: include/drm/ttm/ttm_bo.h */
int ttm_bo_evict_first(struct ttm_device *bdev,
                       struct ttm_resource_manager *man,
                       struct ttm_operation_ctx *ctx);

bool ttm_bo_eviction_valuable(struct ttm_buffer_object *bo,
                               const struct ttm_place *place);

s64 ttm_bo_swapout(struct ttm_device *bdev,
                   struct ttm_operation_ctx *ctx,
                   struct ttm_resource_manager *man,
                   gfp_t gfp_flags,
                   s64 target);
```

The eviction sequence for a single VRAM BO:

1. **Reserve the victim BO** (`ttm_bo_reserve` with the LRU's `ww_acquire_ctx`). If the BO is already reserved by another thread (it is in use), skip it and try the next LRU entry.
2. **Wait for GPU idle** on the BO's `dma_resv`: the BO may have pending GPU operations. TTM calls `dma_resv_wait_timeout()` with the timeout from `ttm_operation_ctx`. If the wait times out, skip this BO.
3. **Allocate in the fallback domain** (GTT or system): call `ttm_bo_validate()` with a GTT-preferred placement to find space in system RAM.
4. **Execute the move**: the driver's `ttm_device_funcs::move` callback performs a DMA blit (GPU-initiated or CPU memcpy) from VRAM to GTT pages. A fence is attached to the BO's `dma_resv` tracking the in-flight move.
5. **Release the VRAM resource** (`ttm_resource_free`): the VRAM pages are returned to the `gpu_buddy` allocator, making them available for the new allocation.

AMDGPU's eviction callback is `amdgpu_bo_move()` (in `amdgpu_ttm.c`), which uses the SDMA (System DMA) engine for GPU-accelerated copies or falls back to a CPU memcpy for small objects or when SDMA is unavailable.

### Swapout to System Memory

When both VRAM and GTT are exhausted, TTM can swap BOs to system memory (anonymous pages not backed by any GPU-accessible mapping) via `ttm_bo_swapout()`. The BO's GPU pages are released entirely and the data lives in the `ttm_tt` structure's `struct page **pages` array, subject to the Linux page reclaim / swap system. This is rarely triggered in practice because it requires both memory domains to be full, a condition that typically indicates a misconfigured system or a memory leak.

### Eviction Thrashing and OOM

If eviction cannot find a suitable victim (all BOs are pinned, or GPU fences are stuck), `ttm_bo_validate` returns `-ENOSPC` or `-ENOMEM`. The driver propagates this to userspace as an allocation failure. If the GPU itself is stuck waiting for a BO that is in the middle of eviction, the result is a GPU hang — the DRM scheduler's hangcheck timer fires, triggering a GPU reset (see Ch102 for scheduler details).

The `/proc/PID/fdinfo` interface exposes per-process VRAM usage (see §10), allowing monitoring tools to detect VRAM over-subscription before eviction thrashing begins.

### AMDGPU-Specific Eviction Notes

AMDGPU uses `amdgpu_bo_evict()` as the driver-level hook, which in addition to the TTM mechanics:

- Updates the `amdgpu_vm` page table to invalidate any GPU VA mappings pointing to the evicted BO (using the `drm_gpuvm` eviction list).
- Triggers a TLB invalidation (via the GPU's GRBM or GFX/SDMA TLB flush commands) to ensure the GPU MMU does not cache stale PTEs.
- Updates `adev->gmc.vis_vram_size` accounting (the `vis_usage` atomic in `amdgpu_vram_mgr`).

---

## 9. DMA-BUF Cross-Driver Sharing: The Kernel View

Ch4 covers DMA-BUF from the API and buffer-lifecycle perspective. This section focuses on the kernel-internal mechanics that matter for driver developers.

### The dma_buf_ops Contract

The exporting driver must implement `struct dma_buf_ops`:

```c
/* Source: include/linux/dma-buf.h */
struct dma_buf_ops {
    /* Called when a new importer attaches */
    int (*attach)(struct dma_buf *, struct dma_buf_attachment *);
    void (*detach)(struct dma_buf *, struct dma_buf_attachment *);

    /* Called by the importer to get a DMA-mappable scatter-gather table */
    struct sg_table *(*map_dma_buf)(struct dma_buf_attachment *,
                                    enum dma_data_direction);
    void (*unmap_dma_buf)(struct dma_buf_attachment *,
                          struct sg_table *, enum dma_data_direction);

    void (*release)(struct dma_buf *);

    /* CPU access — must flush/invalidate GPU caches as needed */
    int (*begin_cpu_access)(struct dma_buf *, enum dma_data_direction);
    int (*end_cpu_access)(struct dma_buf *, enum dma_data_direction);

    /* mmap support: map the buffer into user VMA */
    int (*mmap)(struct dma_buf *, struct vm_area_struct *vma);

    /* Kernel-space virtual mapping */
    int (*vmap)(struct dma_buf *, struct iosys_map *);
    void (*vunmap)(struct dma_buf *, struct iosys_map *);

    /* Explicit sync support (DMA_BUF_IOCTL_SYNC) */
    int (*begin_cpu_access_partial)(struct dma_buf *,
                                    enum dma_data_direction, unsigned int,
                                    unsigned int);
};
```

The GEM subsystem provides a generic `drm_gem_prime_export()` that implements this vtable by delegating to `drm_gem_object_funcs::get_sg_table` for the scatter-gather table and `drm_gem_object_funcs::vmap` for kernel-space mapping.

[Source](https://github.com/torvalds/linux/blob/master/include/linux/dma-buf.h)

### Implicit vs. Explicit Synchronisation

A DMA-BUF carries a `struct dma_resv *resv` (the reservation object), which maintains the fence history. **Implicit synchronisation** means the GPU driver automatically adds a write fence when submitting a GPU job that writes to a DMA-BUF, and other consumers (e.g., KMS scanout, another GPU driver) wait on that fence via `dma_resv_wait_timeout()` before accessing the buffer. This is the traditional Linux GPU sync model.

**Explicit synchronisation** bypasses `dma_resv`. The producer exports an `OUT_FENCE` as a `sync_file` fd; the consumer imports it as an `IN_FENCE`. The kernel Wayland protocol `linux-drm-syncobj-v1` enables this model between compositors and clients. When using explicit sync, it is the userspace application's responsibility to ensure correct ordering.

### Cross-Driver Example: Video Decode → GL Texture

A typical VA-API video decode → display chain:

```
1. V4L2/VA-API decoder (amdgpu VCNDE engine):
   → allocates amdgpu_bo in VRAM
   → submits decode job, attaches write fence to dma_resv

2. Export to userspace via DRM_IOCTL_PRIME_HANDLE_TO_FD:
   → drm_gem_prime_export() → dma_buf fd

3. Import in Mesa (RADV Vulkan driver):
   → DRM_IOCTL_PRIME_FD_TO_HANDLE → new GEM handle
   → drm_gem_prime_import() creates a GEM object backed by the same dma_buf
   → RADV creates VkImage with VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT

4. Vulkan texture sampling:
   → drm_exec locks both BOs (the texture and the render target)
   → dma_resv_wait_timeout() waits for the VCNDE decode fence
   → Vulkan render job attaches a new write fence

5. Wayland compositor presentation (wlroots / Mutter):
   → receives DMA-BUF fd from client via zwp_linux_dmabuf_v1
   → imports into KMS (DRM_IOCTL_MODE_ADDFB2)
   → KMS checks dma_resv for pending fences before programming CRTC
```

Each step uses the same underlying kernel objects (`dma_buf`, `dma_resv`, `dma_fence`) — the cross-driver sharing is entirely mediated by file descriptors and reference counting.

---

## 10. Debugging GPU Memory

### /proc/PID/fdinfo: Per-Process Memory Accounting

Every open DRM file descriptor in a process exposes memory usage statistics via `/proc/PID/fdinfo/N`. The kernel DRM subsystem defines standardised `drm-` prefixed keys:

```
drm-driver:       amdgpu
drm-pdev:         0000:03:00.0
drm-client-id:    12345
drm-total-vram0:  2048 MiB
drm-shared-vram0: 512 MiB
drm-active-vram0: 1800 MiB
drm-total-gtt0:   256 MiB
drm-active-gtt0:  128 MiB
drm-total-cpu0:   64 MiB
```

The `drm-total-<region>` key is the sum of all BOs in the region; `drm-shared-<region>` counts BOs imported via DMA-BUF; `drm-active-<region>` counts BOs referenced by pending GPU jobs. Key names use the region naming defined by each driver (amdgpu uses `vram0`, `gtt0`, `cpu0`). The `drm-memory-<region>` prefix is a deprecated alias for `drm-resident-<region>` (still emitted by older amdgpu kernel versions for compatibility).

[Source](https://docs.kernel.org/gpu/drm-usage-stats.html)

Tools that aggregate fdinfo: `nvtop`, `radeontop`, `intel_gpu_top`, `drm_info`, and the `MangoHud` overlay all parse this interface.

### AMDGPU Debugfs Interfaces

```bash
# VRAM buddy allocator state — shows free blocks by order
cat /sys/kernel/debug/dri/0/amdgpu_vram_mm

# TTM page pool state (system pages held in reserve)
cat /sys/kernel/debug/dri/0/ttm_page_pool

# Per-fence context queue depths (helpful for spotting fence leaks)
cat /sys/kernel/debug/dri/0/amdgpu_fence_info

# GEM object list with placement and size
cat /sys/kernel/debug/dri/0/amdgpu_gem_info

# GPU memory usage summary (visible vs. total VRAM)
cat /sys/kernel/debug/dri/0/amdgpu_gtt_mm
```

### Nouveau Debugfs Interfaces

```bash
# drm_mm dump of VRAM address space (shows allocated ranges)
cat /sys/kernel/debug/dri/1/vram

# GTT allocation state
cat /sys/kernel/debug/dri/1/gtt

# Channel (context) memory usage
cat /sys/kernel/debug/dri/1/clients
```

### Intel i915 / Xe Debugfs Interfaces

```bash
# GEM object inventory with sizes, pinned state, and domain
cat /sys/kernel/debug/dri/0/i915_gem_objects

# GTT address space dump
cat /sys/kernel/debug/dri/0/i915_gem_gtt

# Local memory (lmem) usage for DG2/Arc discrete memory
cat /sys/kernel/debug/dri/0/i915_gem_lmem_mm

# Xe: VM address space dump
cat /sys/kernel/debug/dri/0/xe_gem_objects
```

### drm_info: System-Level View

The `drm_info` tool (available via most distribution package managers) queries `/proc/PID/fdinfo` across all processes and presents a sorted table of GPU memory usage by process. It also reads `drm-driver`, `drm-pdev`, and `drm-engine-*` fields for a complete per-client GPU activity view.

```bash
$ drm_info
Name       PID    VRAM      GTT      CPU    GPU%
firefox    12345  1024 MiB  128 MiB  16 MiB  12%
steam      67890  2048 MiB  256 MiB  32 MiB  45%
kwin_wayl  456    512 MiB   64 MiB   8 MiB   3%
```

### OOM Analysis: When Eviction Fails

The sequence leading to a GPU OOM crash:

1. VRAM fills: `gpu_buddy_alloc_blocks` returns `-ENOSPC` for new allocations.
2. TTM eviction: `ttm_bo_evict_first` iterates the LRU, trying to move BOs to GTT.
3. GTT fills: `ttm_resource_manager` for `TTM_PL_TT` also returns `-ENOSPC`.
4. Swapout attempt: `ttm_bo_swapout` tries to evict to system pages — fails if system memory is also exhausted (true OOM).
5. `ttm_bo_validate` returns `-ENOMEM` to the GPU job submission path.
6. The GPU job fails; the DRM scheduler marks it failed and signals the GPU hang watchdog.
7. GPU reset clears in-flight state; all affected fences are signalled with `-ENODEV`.

To diagnose: compare `drm-total-vram0` across all PIDs in fdinfo (total must not exceed GPU VRAM); check for VRAM leaks by monitoring `drm-total-vram0` over time while the workload runs; use `amdgpu_gem_info` to identify BOs by handle that are not being freed.

`vm.swappiness` (the sysctl controlling Linux's willingness to swap process memory to disk) does not directly affect GPU BO swapout — TTM manages its own eviction policy. However, low `vm.swappiness` values can starve TTM of system pages for GTT allocation on memory-constrained systems.

### LMKD and GPU Memory Pressure

The Linux Low Memory Kill Daemon (`lmkd`) monitors `/proc/PID/oom_score_adj` and `meminfo` to kill low-priority processes under system memory pressure. It does **not** directly observe GPU VRAM pressure — a system can be GPU-VRAM-exhausted while `lmkd` sees ample free system RAM. This means GPU OOM can occur without LMKD triggering, and conversely LMKD may kill a GPU-heavy process (freeing its GEM objects and VRAM) as a side-effect of system memory pressure. On Android and embedded systems where `lmkd` is active, GPU BO lifecycle management must account for `madvise(MADV_DONTNEED)`-style pressure signals that can purge shmem-backed GEM objects (`drm_gem_shmem_purge`).

### NVIDIA Proprietary Driver Memory Inspection

The NVIDIA proprietary driver does not expose TTM or GEM internals. GPU memory usage is accessible through:

```bash
# Per-process VRAM usage via nvidia-smi
nvidia-smi --query-compute-apps=pid,used_memory --format=csv

# Detailed per-process memory breakdown (proprietary driver sysfs)
cat /proc/driver/nvidia/gpus/*/clients
cat /proc/driver/nvidia/gpus/*/information   # total VRAM, free VRAM
```

The `nvidia-smi` tool's `dmon` subcommand provides streaming per-second VRAM usage, framebuffer allocation, and BAR1 memory utilisation — analogous to what `drm_info` provides for open-source drivers.

### umdump: Userspace Memory Dump

`umdump` is a debugging tool for capturing the current state of all mapped GEM/DMA-BUF objects for a process, including their backing memory content, placement, and fence state. It is not widely distributed but exists in Mesa's development tooling (`mesa/tools/`). For driver developers debugging corruption or use-after-free issues in GPU buffer management, `umdump` combined with `amdgpu_gem_info` debugfs output provides a complete picture of BO state at a given point in time.

---

## Roadmap

### Near-term (6–12 months)

- **DMEM cgroup adoption across TTM drivers**: The `dmem` (Device Memory) cgroup controller, introduced for Intel Xe in Linux 6.14, is expected to be wired into AMDGPU and other TTM-based drivers so that VRAM budgets can be enforced per-cgroup for containerised and gaming workloads. [Source](https://www.phoronix.com/news/DMEM-cgroup-vRAM-Control)
- **drm_gpuvm robustness and adoption**: The `drm_gpuvm` + `drm_exec` framework introduced in Linux 6.7 continues to see adoption in nouveau and refinements in Xe/amdgpu — particularly around `drm_gpuvm_bo` abstraction, evicted-object tracking, and userspace VM_BIND IOCTL stabilisation. Note: specific patchset status needs verification against current DRM-next.
- **TTM eviction policy improvements for low-VRAM GPUs**: Work has been posted to fix AMDGPU VRAM management heuristics that cause stuttering on 4–8 GB RDNA2/RDNA3 GPUs under mixed workloads — tuning when the eviction LRU demotes BOs from VRAM to GTT. [Source](https://pixelcluster.github.io/VRAM-Mgmt-fixed/)
- **drm_buddy / gpu_buddy defragmentation**: The buddy allocator lacks a compaction path; in-flight proposals address fragmentation of VRAM in long-running compositor + game sessions by coalescing freed blocks during idle frames. Note: needs verification against current lore.kernel.org threads.
- **Improved `drm_info` / fdinfo standardisation**: Continued push to unify the per-client `drm-total-vram0` and `drm-resident-vram0` fdinfo keys across all drivers so monitoring tools (`drm_info`, `nvtop`, `radeontop`) have a consistent interface. [Source](https://docs.kernel.org/gpu/drm-usage-stats.html)

### Medium-term (1–3 years)

- **CXL-backed GPU memory tiering**: CXL 2.0/3.0 memory expanders can expose Type-3 memory as NUMA nodes. Work is underway in the memory-tiering subsystem to let DRM drivers register GPU VRAM and CXL nodes in a common tier hierarchy, enabling the kernel's NUMA balancing daemon to migrate GPU BOs between HBM, GDDR, CXL-attached DRAM, and system RAM transparently. [Source](https://kernel-internals.org/mm/cxl-memory-tiering/)
- **Unified GPU–CPU virtual address space (SVM / HMM integration)**: The Heterogeneous Memory Management (HMM) subsystem provides `migrate_to_ram` and `migrate_vma` hooks; deeper integration of HMM with TTM's placement engine (replacing the current GTT→VRAM manual migration model with demand-paged GPU faults) is a stated goal for discrete GPU drivers on platforms with PCIe P2P or CXL.cache coherent interconnects. [Source](https://www.kernel.org/doc/html/v5.0/vm/hmm.html)
- **Standardised GPU memory pressure notifications**: A kernel-to-userspace GPU memory pressure interface (analogous to `memory.pressure` in memory cgroups) is under design discussion, allowing compositors and runtimes to receive early warning before VRAM is exhausted and self-evict GPU caches proactively. Note: needs verification — RFC stage as of 2026.
- **Sparse/large-BAR mandatory support and 64-bit BAR normalisation**: As PCIe 6.x and CXL 3.x become mainstream, the expectation is that all discrete GPUs will ship with full-VRAM ReBAR enabled by default, removing the need for the 256 MB windowed BAR fallback and simplifying the TTM CPU mapping path.

### Long-term

- **GPU memory as a first-class kernel MM citizen**: The long-term architectural goal is to give GPU VRAM pages a `struct page` representation (the `DEV_PAGEMAP` + HMM `device_private` page infrastructure) so they participate in LRU scanning, OOM scoring, and page reclaim alongside system RAM — eliminating the current two-tier world where LMKD is blind to GPU pressure.
- **Descriptor-based GPU virtual memory (beyond VM_BIND)**: Research prototypes (e.g., proposals from Intel and Red Hat for Vulkan sparse binding on Linux) suggest replacing the current BO-centric model with a descriptor-based GPU VA allocation model where userspace allocates VA ranges independently of backing storage, enabling large sparse textures and GPU-driven LOD streaming without driver involvement per mip level.
- **Formal GPU memory bandwidth QoS / throttling**: Analogous to CPU bandwidth cgroups, future DRM cgroup extensions may expose per-client VRAM bandwidth quotas enforced by the GPU's memory controller, relevant for multi-tenant cloud GPU virtualisation (vGPU/SR-IOV environments). [Source](https://lwn.net/Articles/1000744/)

---

## Integrations

- **Ch1 — DRM Architecture and the Driver Model**: GEM and TTM are DRM subsystems; `drm_gem_object` is the fundamental DRM buffer primitive. The DRM device probe and `drm_driver` registration described in Ch1 is the context in which TTM is initialised via `ttm_device_init()`.

- **Ch2 — KMS: The Display Pipeline**: KMS framebuffers (`DRM_IOCTL_MODE_ADDFB2`) reference GEM BOs. The pinning of display BOs (`ttm_bo_pin`) to prevent eviction is a direct consequence of KMS requiring stable VRAM addresses for scanout engines.

- **Ch4 — GPU Memory Management (sibling chapter)**: Ch4 covers the GEM object lifecycle, DMA-BUF export/import, PRIME multi-GPU sharing, GBM, dma_resv fencing, and format modifiers in full. This chapter (Ch115) provides the deeper internals: TTM placement and eviction mechanics, the gpu_buddy allocator, drm_gpuvm/drm_exec locking, and Resizable BAR.

- **Ch5 — x86 GPU Drivers (amdgpu, i915, Xe)**: AMDGPU uses TTM + gpu_buddy + drm_gpuvm throughout. i915 uses a hybrid model (GEM for the legacy path, TTM for discrete lmem on Arc). Xe uses drm_gpuvm with VM_BIND as its primary GPU memory management model.

- **Ch8 — The Nouveau Kernel Driver**: Nouveau uses TTM for VRAM and GTT management. The `nouveau_mem_evict` and `nouveau_bo_move` callbacks implement the TTM move contract. Nouveau's push buffer allocation in GTT is a direct application of the `TTM_PL_TT` domain.

- **Ch89 — GPU Virtualisation**: virtio-gpu implements a simplified BO model where the guest driver allocates GEM objects and maps them to host driver resources via the virtio-gpu protocol. The host driver (amdgpu or similar) backs these with TTM BOs. VFIO passthrough bypasses all of this, giving the guest VM direct access to the physical GPU's memory domains.

- **Ch102 — The DRM GPU Scheduler**: The scheduler references BO reservations through `drm_exec` lock lists. `drm_exec_prepare_obj` and `drm_exec_until_all_locked` are called in the scheduler's job preparation path. GPU resets (triggered by eviction failure or fence timeout) clear the scheduler's job queues and signal all pending fences with error.

---

## References

- Linux Kernel GPU Documentation — DRM Memory Management: [https://www.kernel.org/doc/html/latest/gpu/drm-mm.html](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html)
- Linux Kernel GPU Documentation — DRM Client Usage Stats (fdinfo): [https://docs.kernel.org/gpu/drm-usage-stats.html](https://docs.kernel.org/gpu/drm-usage-stats.html)
- `include/drm/ttm/ttm_bo.h` — TTM buffer object API: [https://github.com/torvalds/linux/blob/master/include/drm/ttm/ttm_bo.h](https://github.com/torvalds/linux/blob/master/include/drm/ttm/ttm_bo.h)
- `include/drm/ttm/ttm_placement.h` — TTM placement structures: [https://github.com/torvalds/linux/blob/master/include/drm/ttm/ttm_placement.h](https://github.com/torvalds/linux/blob/master/include/drm/ttm/ttm_placement.h)
- `include/linux/gpu_buddy.h` — gpu_buddy allocator API: [https://github.com/torvalds/linux/blob/master/include/linux/gpu_buddy.h](https://github.com/torvalds/linux/blob/master/include/linux/gpu_buddy.h)
- `include/drm/drm_gpuvm.h` — drm_gpuvm API: [https://github.com/torvalds/linux/blob/master/include/drm/drm_gpuvm.h](https://github.com/torvalds/linux/blob/master/include/drm/drm_gpuvm.h)
- `include/drm/drm_exec.h` — drm_exec locking helper: [https://github.com/torvalds/linux/blob/master/include/drm/drm_exec.h](https://github.com/torvalds/linux/blob/master/include/drm/drm_exec.h)
- `drivers/gpu/drm/amd/amdgpu/amdgpu_vram_mgr.c` — AMDGPU VRAM buddy manager: [https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_vram_mgr.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/amdgpu/amdgpu_vram_mgr.c)
- AMDGPU BAR resize patch: [https://lkml.iu.edu/hypermail/linux/kernel/1703.2/05968.html](https://lkml.iu.edu/hypermail/linux/kernel/1703.2/05968.html)
- PCIe Resizable BAR — Phoronix AMD SAM analysis: [https://www.phoronix.com/news/AMD-Smart-Access-Memory-Initial](https://www.phoronix.com/news/AMD-Smart-Access-Memory-Initial)
- `include/linux/dma-buf.h` — DMA-BUF ops: [https://github.com/torvalds/linux/blob/master/include/linux/dma-buf.h](https://github.com/torvalds/linux/blob/master/include/linux/dma-buf.h)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
