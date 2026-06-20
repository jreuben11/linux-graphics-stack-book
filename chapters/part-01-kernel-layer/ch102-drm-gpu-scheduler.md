# Chapter 102: The DRM GPU Scheduler and Multi-Process Fairness

**Target audiences:** Kernel driver developers implementing GPU command submission; systems engineers diagnosing GPU starvation, priority inversion, or unexplained hangs; Wayland compositor developers optimising render latency under load.

---

## Table of Contents

1. [Introduction — The Multi-Process GPU Problem](#1-introduction--the-multi-process-gpu-problem)
2. [Core Data Structures](#2-core-data-structures)
3. [Job Lifecycle](#3-job-lifecycle)
4. [The CFS-Inspired Fair Scheduling Algorithm](#4-the-cfs-inspired-fair-scheduling-algorithm)
5. [Timeline Fence Model](#5-timeline-fence-model)
6. [Multi-Queue Scheduling Across Engine Types](#6-multi-queue-scheduling-across-engine-types)
7. [Timeout and Hang Detection](#7-timeout-and-hang-detection)
8. [Per-Process GPU Time Accounting](#8-per-process-gpu-time-accounting)
9. [Connection to Wayland Compositing](#9-connection-to-wayland-compositing)
10. [Priority Inversion and Debugging](#10-priority-inversion-and-debugging)
11. [Integrations](#11-integrations)

---

## 1. Introduction — The Multi-Process GPU Problem

Modern Linux desktops run multiple processes that need the GPU simultaneously. A Wayland compositor renders window decorations, a Vulkan game submits thousands of draw calls, a VA-API pipeline decodes 4K video on the compute engine, and a background machine-learning job saturates the tensor cores. On most consumer GPUs — which lack hardware preemption between arbitrary command buffers — this creates a fundamental problem: without an arbiter, one process can monopolise the GPU indefinitely, starving interactive clients and causing visible frame drops or hard hangs that the kernel cannot recover from automatically.

The **DRM GPU scheduler** (`drivers/gpu/drm/scheduler/`) is that arbiter. It sits between driver submission paths and the hardware ring buffers, providing:

- **Fairness** — a CFS-inspired virtual-runtime algorithm that gives each client a proportional share of GPU time according to its priority.
- **Priority classes** — four levels from `KERNEL` (highest, used for internal driver work) down to `LOW`, enabling the Wayland compositor to preempt application rendering.
- **Dependency tracking** — integration with `dma_fence` and `drm_syncobj` so jobs wait for the right prerequisites before executing.
- **Hang detection** — a timeout watchdog that calls the driver back when a job exceeds its deadline, enabling per-ring reset rather than a full GPU reset whenever possible.

The scheduler is a shared library used by virtually every modern DRM driver: AMDGPU, Intel i915 and Xe, Nouveau, the ARM Mali Panfrost/Panthor drivers, and others all depend on it. Understanding it is prerequisite knowledge for any work touching GPU command submission in the Linux kernel.

All source paths in this chapter are relative to the kernel tree at commit [`8c13415c8a43`](https://github.com/torvalds/linux/tree/8c13415c8a4383447c21ec832b20b3b283f0e01a) (Linux `master`, post-v6.19, targeting v6.20, June 2026). Where behaviour differs between v6.19 and v6.20-development is noted explicitly. [Source: `drivers/gpu/drm/scheduler/`](https://github.com/torvalds/linux/tree/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/scheduler)

---

## 2. Core Data Structures

### 2.1 `drm_gpu_scheduler` — the per-engine scheduler

Every hardware engine (GFX ring, compute ring, DMA engine, video decode) has its own `drm_gpu_scheduler` instance. [Source: `include/drm/gpu_scheduler.h`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/include/drm/gpu_scheduler.h)

```c
struct drm_gpu_scheduler {
    const struct drm_sched_backend_ops  *ops;         /* driver callbacks */
    u32                                  credit_limit; /* max in-flight job credits */
    atomic_t                             credit_count; /* currently consumed credits */
    long                                 timeout;      /* hang-detection deadline (jiffies) */
    const char                          *name;
    struct drm_sched_rq                  rq;           /* single run queue (CFS-ordered) */
    struct workqueue_struct             *submit_wq;    /* job dispatch workqueue */
    struct workqueue_struct             *timeout_wq;   /* TDR workqueue */
    struct work_struct                   work_run_job; /* dispatches next ready job */
    struct work_struct                   work_free_job;/* frees completed jobs */
    struct delayed_work                  work_tdr;     /* timeout-detection-reset work */
    struct list_head                     pending_list; /* jobs sent to hardware */
    spinlock_t                           job_list_lock;
    struct ewma_drm_sched_avgtime        avg_job_us;  /* EWMA of job duration */
    int                                  hang_limit;
    atomic_t                            *score;        /* entity count (load metric) */
    bool                                 ready;
    bool                                 free_guilty;
    bool                                 pause_submit;
    struct device                       *dev;
};
```

Key points:

- **`credit_limit` / `credit_count`** replace the old `hw_submission_limit` seen in older kernels. Each job carries a `credits` value (typically 1, but bandwidth-heavy jobs can claim more). The scheduler only dispatches a job if `credit_count + job->credits <= credit_limit`, providing backpressure without blocking the caller.
- **Single `rq`**: the old design had multiple run queues indexed by priority. The current fair scheduler uses *one* run queue per engine, ordered by virtual runtime. Priority is encoded in the vruntime growth rate (see §4).
- **Workqueue-based**: there is no `drm_sched_main` kthread. Two `struct work_struct` items — `work_run_job` and `work_free_job` — drive the submit/free pipeline. `submit_wq` can be shared across schedulers or privately owned (`own_submit_wq = true`).

The driver callbacks are declared in:

```c
struct drm_sched_backend_ops {
    /* Optional: return a fence the scheduler must wait on before run_job */
    struct dma_fence *(*prepare_job)(struct drm_sched_job *,
                                     struct drm_sched_entity *);
    /* Submit the job to hardware; return a hw completion fence */
    struct dma_fence *(*run_job)(struct drm_sched_job *);
    /* Called when a job exceeds sched->timeout */
    enum drm_gpu_sched_stat (*timedout_job)(struct drm_sched_job *);
    /* Free job resources after completion or error */
    void (*free_job)(struct drm_sched_job *);
    /* Optional: cancel a pending job */
    void (*cancel_job)(struct drm_sched_job *);
};
```

### 2.2 `drm_sched_rq` — the run queue

```c
struct drm_sched_rq {
    spinlock_t              lock;
    struct list_head        entities;        /* all entities on this rq */
    struct rb_root_cached   rb_tree_root;    /* CFS-ordered by vruntime */
    enum drm_sched_priority head_prio;       /* priority of leftmost entity */
};
```

The run queue is a **red-black tree** ordered by each entity's virtual runtime (`vruntime`), plus a linked list for O(n) enumeration. `drm_sched_select_entity()` picks the leftmost (minimum-vruntime) ready entity, exactly as Linux's process CFS picks the leftmost runnable task. [Source: `sched_rq.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/scheduler/sched_rq.c)

### 2.3 `drm_sched_entity` — the per-client submission queue

An entity represents one client's connection to one scheduler. A DRM file (`drm_file`) talking to AMDGPU may own several entities — one per engine per context.

```c
struct drm_sched_entity {
    spinlock_t               lock;
    struct drm_sched_rq     *rq;              /* run queue this entity sits in */
    struct drm_sched_entity_stats *stats;     /* vruntime, runtime (ref-counted) */
    struct drm_gpu_scheduler **sched_list;    /* schedulers to load-balance across */
    unsigned int             num_sched_list;
    enum drm_sched_priority  priority;
    struct spsc_queue        job_queue;       /* single-producer, single-consumer */
    uint64_t                 fence_context;   /* base dma_fence context */
    struct dma_fence __rcu  *last_scheduled;  /* last submitted fence (RCU) */
    struct task_struct      *last_user;       /* process that last pushed a job */
    bool                     stopped;
    struct completion        entity_idle;     /* signaled when queue is empty */
    ktime_t                  oldest_job_waiting; /* vruntime tree key */
    struct rb_node           rb_tree_node;    /* position in rq->rb_tree_root */
    atomic_t                *guilty;
};
```

The `stats` object (`drm_sched_entity_stats`) is **reference-counted separately** from the entity — because jobs outlive the entity that created them once they reach the hardware, the accounting object must survive until the last job completes.

When an entity has multiple schedulers in `sched_list` (e.g., an AMDGPU context can run on any of several compute rings), `drm_sched_entity_select_rq()` calls `drm_sched_pick_best()` to choose the least-loaded engine once the previous job finishes and the queue drains. This provides **transparent load balancing across identical engines** without driver involvement.

### 2.4 `drm_sched_job` — one GPU submission

```c
struct drm_sched_job {
    struct drm_gpu_scheduler *sched;
    struct drm_sched_fence   *s_fence;      /* scheduled + finished fence pair */
    struct drm_sched_entity  *entity;
    struct drm_sched_entity_stats *entity_stats; /* snapshot for accounting */
    enum drm_sched_priority   s_priority;
    u32                       credits;       /* flow-control weight */
    struct xarray             dependencies; /* dma_fences to wait on */
    struct spsc_node          queue_node;   /* position in entity->job_queue */
    struct list_head          list;         /* position in pending_list */
    atomic_t                  karma;        /* hang-repeat counter */
};
```

### 2.5 `drm_sched_fence` — the dual-fence progress object

```c
struct drm_sched_fence {
    struct dma_fence  scheduled;   /* signals when job *starts* on GPU */
    struct dma_fence  finished;    /* signals when job *completes* on GPU */
    struct dma_fence *parent;      /* hw fence returned by run_job() */
    ktime_t           deadline;    /* propagated from downstream waiters */
    struct drm_gpu_scheduler *sched;
    spinlock_t        lock;
    void             *owner;
    uint64_t          drm_client_id;
};
```

The `scheduled` fence is useful for tools and profilers that need to know when the GPU started executing a submission (not just when it completed). The `finished` fence is what `drm_syncobj` signals to unblock the next job in the dependency chain. The `parent` fence is the hardware fence returned by `run_job()`; when the driver signals it (GPU interrupt → IRQ handler → `dma_fence_signal`), the scheduler's completion callbacks chain through to `finished`. [Source: `sched_fence.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/scheduler/sched_fence.c)

---

## 3. Job Lifecycle

A GPU submission follows a fixed four-phase lifecycle through the scheduler.

### Phase 1 — Initialisation

```c
int drm_sched_job_init(struct drm_sched_job *job,
                       struct drm_sched_entity *entity,
                       u32 credits,
                       void *owner,
                       uint64_t drm_client_id);
```

This zeroes `job`, allocates `job->s_fence` from the kmem cache, sets `job->credits`, and initialises `job->dependencies` as an xarray. It does **not** initialise the fences' sequence numbers yet — that happens at arm time. [Source: `sched_main.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/scheduler/sched_main.c)

### Phase 2 — Dependency Addition

Before arming, the driver populates the dependency xarray with every fence this job must wait for:

```c
/* Explicit: add a specific dma_fence */
int drm_sched_job_add_dependency(struct drm_sched_job *job,
                                 struct dma_fence *fence);

/* Implicit: extract fences from a GEM object's reservation object */
int drm_sched_job_add_implicit_dependencies(struct drm_sched_job *job,
                                            struct drm_gem_object *obj,
                                            bool write);
```

`drm_sched_job_add_dependency` deduplicates by `dma_fence` context: if a fence from the same context already exists in the xarray, it keeps whichever has the later sequence number. This collapses dependency chains from the same command stream into a single fence, bounding the xarray size.

`drm_sched_job_add_implicit_dependencies` is a thin wrapper around `drm_sched_job_add_resv_dependencies`, which walks a GEM object's `dma_resv` and adds either the shared (read) or exclusive (write) fences. It must be called while the `dma_resv` lock is held and before the caller updates the reservation with the new fence.

### Phase 3 — Arm

```c
void drm_sched_job_arm(struct drm_sched_job *job);
```

This is the point of no return. `job_arm`:
1. Calls `drm_sched_entity_select_rq()` to perform load balancing if the entity spans multiple schedulers (only when the queue is empty and the last job has signalled).
2. Derives `job->sched` from `entity->rq` via `container_of`.
3. Captures `job->s_priority` from `entity->priority`.
4. Takes a reference on `entity->stats` into `job->entity_stats`.
5. Calls `drm_sched_fence_init()`, which allocates sequence numbers and initialises both `scheduled` and `finished` as proper `dma_fence` objects with unique (context, seqno) pairs.

After `job_arm`, the `s_fence->scheduled` and `s_fence->finished` fences can be shared with other code (e.g., stored in a `drm_syncobj` timeline point) before the job is submitted.

### Phase 4 — Submission

```c
void drm_sched_entity_push_job(struct drm_sched_job *sched_job);
```

The job is pushed onto the entity's `spsc_queue` (a lock-free single-producer single-consumer ring). The SPSC design is intentional: the producing CPU (ioctl thread) writes without locks; the consuming side (workqueue) reads without locks. Atomicity at the queue boundary is achieved by the `spsc_queue_push()` return value: `true` means the queue was empty before this push. When `first == true`:

1. `drm_sched_rq_add_entity()` inserts the entity into `rq->entities` and places it in the rb-tree at the appropriate vruntime position (recomputed from the saved delta, see §4).
2. `drm_sched_wakeup(sched)` queues `work_run_job` on `submit_wq`.

### The Run and Free Work Items

`drm_sched_run_job_work()` runs on `submit_wq`:
1. Calls `drm_sched_select_entity()` to pick the minimum-vruntime ready entity.
2. Pops the job from the entity's queue via `drm_sched_entity_pop_job()`.
3. Checks credit availability: if `credit_count + job->credits > credit_limit`, returns `ERR_PTR(-ENOSPC)` and the entity stays in the tree without advancing.
4. Optionally calls `ops->prepare_job()` if defined, waiting on the returned fence.
5. Calls `ops->run_job()`, which submits to the hardware ring and returns a hw completion fence.
6. Attaches a completion callback on the hw fence; when it fires, `drm_sched_job_done()` is called.
7. Queues `work_run_job` again to process the next entity.

`drm_sched_free_job_work()` runs after `job_done`: it accumulates GPU time into `entity_stats->runtime`, calls `ops->free_job()`, and re-queues `work_run_job`.

### Cleanup

```c
void drm_sched_job_cleanup(struct drm_sched_job *job);
```

Called from `ops->free_job()` to release `job->dependencies` (xa_destroy), drop the fence reference, and release `job->entity_stats`.

---

## 4. The CFS-Inspired Fair Scheduling Algorithm

The DRM scheduler in the v6.20 development tree (commit `8c13415c8a43`, post-v6.19), authored by Tvrtko Ursulin of Igalia, replaces the previous FIFO/round-robin/priority-array design with a CFS-inspired algorithm using **virtual runtime (vruntime)**. Kernels through v6.19 used multiple per-priority run queues with `drm_sched_rq_select_entity_rr()` or `drm_sched_rq_select_entity_fifo()` selection; this section describes the new algorithm present in the v6.20 development tree. [Source: `sched_rq.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/scheduler/sched_rq.c) [Fair DRM scheduler blog](https://blogs.igalia.com/tursulin/fair-er-drm-gpu-scheduler/)

### Priority Enum

```c
enum drm_sched_priority {
    DRM_SCHED_PRIORITY_INVALID = -1,
    DRM_SCHED_PRIORITY_KERNEL  = 0,   /* internal driver use */
    DRM_SCHED_PRIORITY_HIGH    = 1,   /* Wayland compositor render queue */
    DRM_SCHED_PRIORITY_NORMAL  = 2,   /* default for userspace */
    DRM_SCHED_PRIORITY_LOW     = 3,   /* background tasks */
    DRM_SCHED_PRIORITY_COUNT
};
```

Note the ordering: numerically, `KERNEL < HIGH < NORMAL < LOW`, i.e., *lower* numeric value means *higher* priority. This is reflected in the vruntime multipliers below.

### Virtual Runtime Multipliers

```c
/* From drivers/gpu/drm/scheduler/sched_rq.c */
static const unsigned int vruntime_shift[] = {
    [DRM_SCHED_PRIORITY_KERNEL] = 1,
    [DRM_SCHED_PRIORITY_HIGH]   = 2,
    [DRM_SCHED_PRIORITY_NORMAL] = 4,
    [DRM_SCHED_PRIORITY_LOW]    = 7,
};
```

When a job finishes, the entity's vruntime is updated:

```
vruntime += real_gpu_ns << vruntime_shift[priority]
```

Because vruntime grows as `real_gpu_ns << shift`, an entity with a larger shift accumulates virtual time faster for the same real GPU time, and therefore the scheduler will deprioritise it relative to lower-shift entities. The ratio of real GPU time between two priority levels A (higher) and B (lower) is `2^(shift_B − shift_A)`. This produces the following approximate GPU time ratios under sustained competing load:

| Priority | Shift | Real GPU time relative to baseline |
|---|---|---|
| KERNEL | 1 | ~64× LOW, ~8× NORMAL, ~2× HIGH |
| HIGH | 2 | ~32× LOW, ~4× NORMAL |
| NORMAL | 4 | ~8× LOW |
| LOW | 7 | baseline |

In practice the ratios are approximate — job durations, credit consumption, and the scheduler's interactivity heuristic (below) all affect real distribution.

### Entity Selection

`drm_sched_select_entity()` holds `rq->lock` and walks the rb-tree leftmost-first:

```c
struct drm_sched_entity *
drm_sched_select_entity(struct drm_gpu_scheduler *sched)
{
    struct drm_sched_rq *rq = &sched->rq;
    struct rb_node *rb;

    spin_lock(&rq->lock);
    for (rb = rb_first_cached(&rq->rb_tree_root); rb; rb = rb_next(rb)) {
        struct drm_sched_entity *entity =
            rb_entry(rb, struct drm_sched_entity, rb_tree_node);

        if (drm_sched_entity_is_ready(entity)) {
            if (!drm_sched_can_queue(sched, entity)) {
                spin_unlock(&rq->lock);
                return ERR_PTR(-ENOSPC);  /* credit limit hit */
            }
            reinit_completion(&entity->entity_idle);
            break;
        }
    }
    spin_unlock(&rq->lock);
    return rb ? rb_entry(rb, ...) : NULL;
}
```

An entity is "ready" if its `job_queue` is non-empty and its `dependency` fence is NULL (all input fences have signalled). Entities waiting on a fence are invisible to the scheduler — they do not occupy a slot and do not starve other entities.

### Re-entry After Inactivity

When an entity goes idle (queue drains), its `vruntime` is saved as a *delta* relative to the queue's `min_vruntime`:

```c
static void drm_sched_entity_save_vruntime(struct drm_sched_entity *entity,
                                           ktime_t min_vruntime)
{
    ktime_t vruntime = stats->vruntime;
    if (min_vruntime && vruntime > min_vruntime)
        vruntime = ktime_sub(vruntime, min_vruntime);
    else
        vruntime = 0;
    stats->vruntime = vruntime;
}
```

When it becomes active again, `drm_sched_entity_restore_vruntime()` re-anchors it to the current `min_vruntime` plus the saved delta, with special handling:

- If the entity has *higher* priority than the current queue head, it is inserted with a negative vruntime offset (`-ns_to_ktime(rq_prio - prio)`), letting it run before existing entities.
- If it has *lower* priority and the queue is non-empty, it is pushed behind the "middle" by a duration proportional to its own and the scheduler's average job durations.
- For same-priority re-entry, the entity with shorter average jobs (more interactive) gets a -1 ns head-start.

This means **priority acts as a persistent bias** on scheduling order without causing complete starvation of lower-priority entities.

### Initialisation

```c
int drm_sched_init(struct drm_gpu_scheduler *sched,
                   const struct drm_sched_init_args *args);
```

Drivers pass all parameters via `drm_sched_init_args`:

```c
struct drm_sched_init_args {
    const struct drm_sched_backend_ops *ops;
    struct workqueue_struct            *submit_wq;   /* NULL → create private */
    struct workqueue_struct            *timeout_wq;  /* NULL → system_wq */
    u32                                 credit_limit;
    unsigned int                        hang_limit;
    long                                timeout;     /* jiffies; MAX_SCHEDULE_TIMEOUT = disabled */
    atomic_t                           *score;
    const char                         *name;
    struct device                      *dev;
};
```

An AMDGPU ring initialising its scheduler passes the ring's `timeout` (driver-tuned, typically 10 seconds for GFX, 60 seconds for compute) and a `credit_limit` matching the ring buffer's maximum in-flight depth.

---

## 5. Timeline Fence Model

The scheduler's fence model connects userspace explicit sync (Vulkan semaphores, `drm_syncobj`) to the kernel's `dma_fence` infrastructure.

### The Fence Pair

Every job gets a `drm_sched_fence` containing two `dma_fence` objects:

- `scheduled`: signals when `drm_sched_fence_scheduled()` is called in `drm_sched_run_job_work`, just before `ops->run_job()`. Useful for profiling ("when did the GPU start this work?").
- `finished`: signals via `drm_sched_fence_finished()` when the hardware fence (`parent`) signals. This is what unblocks the next job in the pipeline.

The `parent` fence is set with `drm_sched_fence_set_parent()`, which uses `smp_store_release()` to guarantee visibility before the fence signals — important for drivers that delegate waits to firmware (GuC, MEC).

### drm_syncobj Integration

A `drm_syncobj` is a kernel object that holds a `dma_fence` reference. Timeline syncobjs (`drm_syncobj_timeline`) map sequence numbers to fences, implementing Vulkan timeline semaphores at the kernel level.

The dependency chain for a Vulkan `vkQueueSubmit` with timeline semaphores:

1. `vkQueueSubmit` calls the driver's `submit` ioctl.
2. The driver calls `drm_syncobj_find_fence(syncobj, seqno, ...)` to obtain the input `dma_fence`.
3. `drm_sched_job_add_dependency(job, input_fence)` stores it in `job->dependencies`.
4. `drm_sched_job_arm(job)` initialises `job->s_fence->finished` with a new (context, seqno).
5. The driver stores `job->s_fence->finished` into the output syncobj timeline point at the requested signal seqno.
6. `drm_sched_entity_push_job(job)` enqueues the job.
7. The scheduler checks `drm_sched_entity_is_ready()` — false while `entity->dependency` (the input fence) is unsignalled; true once it fires.
8. `ops->run_job()` submits to hardware; the hw fence is stored in `s_fence->parent`.
9. When the GPU completes and signals the hw fence, `drm_sched_fence_finished()` signals `s_fence->finished`.
10. The syncobj timeline point at the signal seqno is now signalled, unblocking `vkWaitSemaphores` in userspace or another job's `drm_sched_job_add_dependency`.

This chain is entirely non-blocking: the scheduler kworker never sleeps waiting for a fence. Instead, `drm_sched_entity_wakeup()` — a `dma_fence_cb` registered on `entity->dependency` — fires when the fence signals and calls `drm_sched_wakeup()` to re-queue `work_run_job`.

---

## 6. Multi-Queue Scheduling Across Engine Types

Modern GPUs have multiple independent engine classes, each with its own `drm_gpu_scheduler`:

### AMDGPU Engine Topology

On an RDNA3+ GPU, AMDGPU creates schedulers for (exact counts are hardware and firmware dependent):

| Engine | Ring prefix | Representative count |
|---|---|---|
| Graphics (GFX) | `gfx` | 1 (or 2 with async GFX on RDNA3+) |
| Async compute | `comp` | varies (2–8 depending on SKU) |
| DMA / blit | `sdma` | 2–4 |
| Video decode (VCN) | `vcn_dec` | 1–2 |
| Video encode | `vcn_enc` | 1–2 |

Each ring maps to one `drm_gpu_scheduler`, embedded as `ring->sched`. The ring registers itself in `adev->gpu_sched[hw_ip][hw_prio]` during `amdgpu_ring_init()`, which lets the context layer (`amdgpu_ctx.c`) find the right scheduler for each IP block and priority. [Source: `drivers/gpu/drm/amd/amdgpu/amdgpu_ring.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/amd/amdgpu/amdgpu_ring.c)

### Context Entities and Priority

An `amdgpu_ctx` (created by `DRM_IOCTL_AMDGPU_CTX`) holds a matrix of entities: one per (engine class × priority). When userspace submits via `DRM_IOCTL_AMDGPU_CS`, it selects an IP block and the context's entity for that block is used. Priority is set at context creation time, mapping userspace priorities through `amdgpu_ctx_to_drm_sched_prio()` to `DRM_SCHED_PRIORITY_*`:

```c
/* From drivers/gpu/drm/amd/amdgpu/amdgpu_ctx.c */
static enum drm_sched_priority
amdgpu_ctx_to_drm_sched_prio(int32_t ctx_prio)
{
    switch (ctx_prio) {
    case AMDGPU_CTX_PRIORITY_VERY_LOW:
    case AMDGPU_CTX_PRIORITY_LOW:       return DRM_SCHED_PRIORITY_LOW;
    case AMDGPU_CTX_PRIORITY_NORMAL:    return DRM_SCHED_PRIORITY_NORMAL;
    case AMDGPU_CTX_PRIORITY_HIGH:
    case AMDGPU_CTX_PRIORITY_VERY_HIGH: return DRM_SCHED_PRIORITY_HIGH;
    default: return DRM_SCHED_PRIORITY_NORMAL;
    }
}
```

Raising a context above `AMDGPU_CTX_PRIORITY_NORMAL` requires `CAP_SYS_NICE`, enforced in `amdgpu_ctx_priority_permit()`.

A Wayland compositor requests `DRM_SCHED_PRIORITY_HIGH` for its rendering context. Under the CFS algorithm, this means its entities' vruntime grows at shift-2 rate versus normal applications' shift-4 rate, giving it roughly 4× more GPU time per real second when competing with a `NORMAL`-priority client.

### Intel Xe Architecture

Intel Xe (introduced in Linux 6.8) uses `xe_exec_queue` as the abstraction per (VM, engine). Each exec queue has its own `drm_sched_entity`, and priority is stored in `q->sched_props.priority`. The Xe GuC submission backend submits jobs to GuC firmware, which implements its own priority-aware scheduling on the hardware engine. The DRM scheduler layer handles dependency tracking, timeout detection, and process accounting, while the GuC handles the final hardware schedule. [Source: `drivers/gpu/drm/xe/xe_exec_queue.c`]

### Cross-Engine Ordering

The DRM scheduler does **not** enforce ordering across schedulers. A GFX job and a compute job on separate schedulers can run in any order relative to each other. Cross-engine ordering is the responsibility of userspace, expressed as:

- **Explicit sync**: `drm_sched_job_add_dependency()` with a `drm_syncobj` fence produced by the GFX engine, consumed by the compute engine.
- **Implicit sync** (legacy): `drm_sched_job_add_implicit_dependencies()` from the GEM object's `dma_resv`, which tracks the last writer across all engines.

This design is correct: the scheduler's role is intra-engine fairness and per-engine flow control. Cross-engine serialisation is semantically userspace's responsibility and must be expressed as explicit data dependencies.

---

## 7. Timeout and Hang Detection

### The Timeout Watchdog

Every job that starts execution arms a delayed work item:

```c
mod_delayed_work(sched->timeout_wq, &sched->work_tdr, sched->timeout);
```

`sched->timeout` is set at `drm_sched_init()` time in jiffies. AMDGPU uses ~10 000 ms for GFX rings and ~60 000 ms for compute rings. Setting `timeout = MAX_SCHEDULE_TIMEOUT` disables hang detection for that scheduler (used during suspend/resume).

### The TDR Sequence

When `work_tdr` fires (job took longer than `sched->timeout`):

1. `drm_sched_job_timedout()` removes the oldest job from `pending_list`.
2. Calls `ops->timedout_job(job)`, which returns `enum drm_gpu_sched_stat`:
   - `DRM_GPU_SCHED_STAT_RESET`: driver performed a reset and scheduler should restart.
   - `DRM_GPU_SCHED_STAT_ENODEV`: device lost; no restart.
   - `DRM_GPU_SCHED_STAT_NO_HANG`: timeout was spurious (e.g., the job actually completed); reinsert and continue.
   - `DRM_GPU_SCHED_STAT_NONE`: no meaningful result.
3. If `free_guilty = true`, the guilty job is freed via `ops->free_job()`.
4. If the device is still alive, `drm_sched_start()` is called.

### AMDGPU Timeout Recovery

In AMDGPU's `timedout_job` callback ([Source: `drivers/gpu/drm/amd/amdgpu/amdgpu_job.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/amd/amdgpu/amdgpu_job.c)), the recovery sequence is:

1. If the device has been unplugged, return `DRM_GPU_SCHED_STAT_ENODEV` immediately.
2. Dump IP state for debugging (`amdgpu_job_core_dump`), unless running under SR-IOV (where the host driver handles recovery).
3. Attempt `amdgpu_ring_soft_recovery()` — a lightweight per-ring reset via `ring->funcs->reset`.
4. If soft recovery fails, escalate to `amdgpu_device_gpu_recover()` for a full GPU reset.
5. Return `DRM_GPU_SCHED_STAT_NO_HANG`, which tells the scheduler infrastructure to reinsert the job into the pending list.

The key AMDGPU observation: when soft recovery succeeds (the ring is back), the function calls `drm_sched_wqueue_stop()` before the reset and `drm_sched_wqueue_start()` after, which stops and restarts the submit workqueue around the ring reset window.

```c
/* From drivers/gpu/drm/amd/amdgpu/amdgpu_job.c (condensed) */
static enum drm_gpu_sched_stat amdgpu_job_timedout(struct drm_sched_job *s_job)
{
    struct amdgpu_ring *ring = to_amdgpu_ring(s_job->sched);
    struct amdgpu_job  *job  = to_amdgpu_job(s_job);
    struct amdgpu_device *adev = ring->adev;

    if (!drm_dev_enter(adev_to_drm(adev), &idx))
        return DRM_GPU_SCHED_STAT_ENODEV;

    if (!amdgpu_sriov_vf(adev))
        amdgpu_job_core_dump(adev, job);

    /* Attempt soft per-ring recovery first */
    if (amdgpu_gpu_recovery &&
        amdgpu_ring_is_reset_type_supported(ring, AMDGPU_RESET_TYPE_SOFT_RESET) &&
        amdgpu_ring_soft_recovery(ring, job->vmid, s_job->s_fence->parent))
        goto exit;  /* recovered */

    /* Per-ring hard reset */
    if (amdgpu_gpu_recovery &&
        amdgpu_ring_is_reset_type_supported(ring, AMDGPU_RESET_TYPE_PER_QUEUE) &&
        ring->funcs->reset) {
        drm_sched_wqueue_stop(&ring->sched);
        if (!amdgpu_ring_reset(ring, job->vmid, job->hw_fence)) {
            drm_sched_wqueue_start(&ring->sched);
            goto exit;
        }
    }

    /* Full GPU reset */
    if (amdgpu_device_should_recover_gpu(adev))
        amdgpu_device_gpu_recover(adev, job, &reset_context);

exit:
    drm_dev_exit(idx);
    return DRM_GPU_SCHED_STAT_NO_HANG;
}
```

AMDGPU timeout defaults, configured via the `amdgpu_lockup_timeout` module parameter: 10,000 ms for GFX, SDMA, and video rings; 60,000 ms for compute rings.

From userspace, a GPU reset surfaces as a Vulkan `VK_ERROR_DEVICE_LOST` on the next `vkQueueSubmit` or `vkWaitForFences` call.

### `drm_sched_stop` and `drm_sched_start`

`drm_sched_stop(sched, bad_job)`:
- Sets `pause_submit = true`, preventing new dispatch.
- Cancels `work_run_job` and `work_free_job`.
- For all jobs in `pending_list`, removes hw fence callbacks; credits are refunded if a job hadn't been acknowledged.
- Moves the guilty job to the front of the pending list for TDR examination.

`drm_sched_start(sched, errno)`:
- Iterates `pending_list` in submission order.
- For each surviving job: if `s_fence->parent` is still unsignalled, reattaches a hw fence callback; if already signalled, immediately calls `drm_sched_job_done()`.
- If `errno != 0`, signals all pending jobs' `finished` fences with the error, causing `VK_ERROR_DEVICE_LOST` propagation.
- Clears `pause_submit`, rearms `work_tdr`, and queues `work_run_job`.

---

## 8. Per-Process GPU Time Accounting

### `drm_sched_entity_stats`

The ref-counted `drm_sched_entity_stats` object tracks:

```c
struct drm_sched_entity_stats {
    struct kref     kref;
    spinlock_t      lock;
    ktime_t         runtime;       /* total GPU ns consumed by this entity */
    ktime_t         prev_runtime;  /* snapshot for delta computation */
    ktime_t         vruntime;      /* CFS virtual runtime */
    struct ewma_drm_sched_avgtime avg_job_us;  /* EWMA of job duration */
};
```

GPU time is accumulated in `drm_sched_free_job_work()` by calling `drm_sched_entity_stats_job_add_gpu_time()`, which computes `hw_fence_signal_time - job_run_time` and adds it to `stats->runtime`.

### `/proc/pid/fdinfo` — DRM Usage Statistics

DRM exports per-process, per-engine statistics through the standard Linux `fdinfo` mechanism. Reading `/proc/self/fdinfo/<drm_fd>` produces output like:

```
drm-driver:     amdgpu
drm-pdev:       0000:03:00.0
drm-client-id:  42
drm-engine-gfx: 12345678 ns
drm-cycles-gfx: 8765432
drm-engine-compute: 3000000 ns
drm-cycles-compute: 2100000
drm-memory-vram: 512 MiB
drm-memory-gtt:  128 MiB
```

Field semantics ([Source: kernel.org DRM usage stats](https://www.kernel.org/doc/html/latest/gpu/drm-usage-stats.html)):

- `drm-engine-<name>: <uint> ns` — time the named engine spent executing workloads for this file descriptor, in nanoseconds.
- `drm-cycles-<name>: <uint>` — raw GPU cycle count for the same engine.
- `drm-client-id` — unique ID distinguishing file descriptor instances; use this to avoid double-counting when the same fd is duplicated.
- `drm-memory-*` — buffer residency in each memory region. **Note:** this key is deprecated (an alias for `drm-resident-*`) and is only printed by AMDGPU; new code should use `drm-resident-*`.

The data is populated by `dev->driver->show_fdinfo()`, a driver-specific callback invoked by the core DRM infrastructure in `drm_show_fdinfo()`. AMDGPU implements this in `drivers/gpu/drm/amd/amdgpu/amdgpu_fdinfo.c`. The per-engine `drm-engine-*` values are sourced from the scheduler entity's GPU time tracking (`drm_sched_entity_stats::runtime`), accumulated in `drm_sched_free_job_work()` and exposed per engine. [Source: `drivers/gpu/drm/drm_file.c`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/drm_file.c)

### Consumer: MangoHUD

MangoHUD reads `/proc/self/fdinfo` at regular intervals, diffs the `drm-engine-*` counters between samples, and divides by the sampling period to produce "GPU utilisation %" displayed in the in-game overlay. Because the counters are per-fd and per-engine, MangoHUD can report separate utilisation for GFX, compute, and video decode — provided the driver exports them.

### Credit-Based Flow Control and Time Slicing

Most consumer GPUs lack hardware preemption between arbitrary command buffers: once a draw call starts executing, it runs to completion. The scheduler approximates time-slicing by:

1. Setting `credit_limit` to match the ring's in-flight capacity (e.g., 64 credits for AMDGPU GFX).
2. Assigning each job a `credits` value (typically 1).
3. Processing entities in vruntime order: as each job completes and frees a credit, the *next job* scheduled may come from a *different entity* (because `drm_sched_select_entity` re-selects from the rb-tree).

The result: short-running jobs achieve near-fair interleaving; long-running compute jobs accumulate vruntime rapidly and are deprioritised against interactive entities between each ring-buffer segment.

---

## 9. Connection to Wayland Compositing

### The Compositor's Render Pipeline

The Wayland compositor (e.g., wlroots or KWin) runs a per-frame pipeline:

1. **Receive client buffers** — wl_surface.commit delivers a dmabuf with an implicit (or explicit) fence.
2. **Composite** — the compositor renders all surfaces into the back-buffer using its Vulkan or GL renderer.
3. **Present** — commit to DRM KMS with `drmModeAtomicCommit`.

The DRM GPU scheduler participates in steps 2 and 3.

### Step 2: Render Job Scheduling

The compositor's render entity is initialised at `DRM_SCHED_PRIORITY_HIGH`. When `vkQueueSubmit` (or `eglSwapBuffers` → GL driver submit) pushes the composite job:

1. The entity is inserted into the GFX scheduler's rb-tree with a vruntime adjusted for HIGH priority.
2. `drm_sched_select_entity()` picks it over any `NORMAL`-priority application jobs with equal or greater virtual runtime.
3. `ops->run_job()` submits the composite command buffer; `s_fence->finished` is stored in the output `drm_syncobj`.

### Step 3: KMS Atomic Commit

wlroots' DRM backend calls `gbm_surface_lock_front_buffer()` to get the rendered buffer, then:

```c
drmModeAtomicAddProperty(req, crtc_id, OUT_FENCE_PTR, &out_fence_fd);
/* IN_FENCE_FD: the renderer's drm_syncobj, exported as a sync_file */
drmModeAtomicAddProperty(req, plane_id, IN_FENCE_FD, render_fence_fd);
drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_NONBLOCK, NULL);
```

The `IN_FENCE_FD` wraps the `dma_fence` from `s_fence->finished`. KMS will not scan out the new buffer until this fence signals — ensuring the GPU has finished rendering before the display controller reads from the buffer.

The complete dependency chain:

```
compositor vkQueueSubmit
  → drm_sched_entity_push_job (priority HIGH)
  → drm_sched_select_entity (beats NORMAL apps)
  → ops->run_job → GPU executes composite
  → hw fence signals
  → drm_sched_fence_finished fires
  → s_fence->finished signals
  → sync_file (IN_FENCE_FD) signals
  → KMS atomic commit unblocks
  → VBLANK → display scanout
```

### Priority Inversion at the KMS Level

If the compositor's `s_fence->finished` is not yet signalled when the VBLANK deadline arrives, KMS holds the current buffer and the compositor misses a frame. This is the GPU equivalent of a missed deadline. By assigning `DRM_SCHED_PRIORITY_HIGH` to the compositor entity, the scheduler ensures its jobs run first, minimising the risk of missing VBLANK.

The `linux-drm-syncobj-v1` Wayland protocol (discussed in Ch75) provides the userspace mechanism for clients to export and import `drm_syncobj`-backed fences, enabling explicit frame-paced composition without implicit sync overhead.

---

## 10. Priority Inversion and Debugging

### Priority Inversion Scenario

Consider:
- A `HIGH`-priority compositor job has an input dependency on an app job via a shared buffer's implicit fence.
- The app job is queued at `NORMAL` priority but is held behind three other `NORMAL`-priority jobs from different processes.
- Under light load the vruntime algorithm handles this gracefully. Under heavy load (many `NORMAL` entities), the app job's vruntime grows slowly and it may wait several frame periods before scheduling.
- The compositor job, though `HIGH` priority, cannot start until its input fence (the app's output) signals.

This is a classic **priority inversion via dependency**: the `HIGH`-priority waiter is blocked on a `NORMAL`-priority holder.

**Mitigations:**

1. **Avoid implicit sync dependencies from compositor to application outputs.** The compositor should use explicit sync (`linux-drm-syncobj-v1`): the application signals its own explicit fence, the compositor takes a dependency on that fence. The compositor's entity itself has no direct dependency on the app's scheduler entity — only on the fence. This is fine, because fence signalling is not priority-dependent; the app's job will run when scheduled, and the compositor's entity remains at HIGH priority for its own job.
2. **Future work: deadline propagation.** The `drm_sched_fence` structure already has a `deadline` field used to propagate deadlines from `dma_fence` waiters. A full deadline-boosting mechanism (similar to priority inheritance in mutex locks) would temporarily elevate the app entity's vruntime priority when a HIGH-priority entity is waiting on its fence.

### Debugging Tools

**Tracepoints:**

The scheduler exports stable-uAPI tracepoints ([Source: `gpu_scheduler_trace.h`](https://github.com/torvalds/linux/blob/8c13415c8a4383447c21ec832b20b3b283f0e01a/drivers/gpu/drm/scheduler/gpu_scheduler_trace.h)):

```bash
# Enable scheduler tracing
echo 1 > /sys/kernel/debug/tracing/events/gpu_scheduler/enable

# Key tracepoints:
# drm_sched_job_queue  - job enqueued by userspace
# drm_sched_job_run    - job dispatched to hardware
# drm_sched_job_done   - job completed on GPU
# drm_sched_job_add_dep - dependency added to a job
```

Each event records `sched_name`, `device`, fence `context:seqno`, job count, and `drm_client_id`. Timeline reconstruction from these events reveals scheduling gaps, starvation, and timeout sequences.

**AMDGPU debugfs:**

```bash
# Per-ring scheduler state
cat /sys/kernel/debug/dri/0/amdgpu_fence_info
cat /sys/kernel/debug/dri/0/amdgpu_ring_gfx_0

# Check for hung jobs
cat /sys/kernel/debug/dri/0/amdgpu_gpu_recover
```

**`drm_info` tool:**

The `drm_info` userspace tool reads `drm-engine-*` from fdinfo for all processes and presents a per-engine utilisation view, useful for identifying which process is consuming GPU time and from which engine.

**Credit / queue depth tuning:**

Some drivers expose `hw_submission_limit` (the predecessor to `credit_limit`) or ring-size knobs via sysfs. Reducing these increases scheduler responsiveness at the cost of throughput, as the scheduler gets to re-select entities more frequently.

**Hang simulation and testing:**

The kernel scheduler has a KUnit test suite in `drivers/gpu/drm/scheduler/tests/` covering basic job submission, dependency resolution, and timeout behaviour. Driver developers can add mock ring backends via `mock_scheduler.c` to test TDR sequences without real hardware.

---

## Roadmap

### Near-term (6–12 months)

- **Fair scheduler stabilisation and merge**: The CFS-inspired fair DRM scheduler (replacing per-priority FIFO queues with a single vruntime-ordered red-black tree) is progressing through patch review after reaching v5; the expectation is mainline inclusion in a v6.20 or v6.21 cycle. [Source: Phoronix — Fair DRM Scheduler v5](https://www.phoronix.com/news/Fair-DRM-Scheduler-v5)
- **Deadline propagation / priority inheritance**: The `drm_sched_fence::deadline` field exists in the current tree but the boosting mechanism — which would temporarily elevate a blocked entity's effective vruntime when a `HIGH`-priority waiter holds a fence dependency on it — is described as future work in the chapter and in mailing-list discussion. Full implementation is anticipated on a short horizon. [Source: RFC v3 Deadline DRM Scheduler](https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg537779.html)
- **KUnit test coverage expansion**: The `drivers/gpu/drm/scheduler/tests/` suite currently covers basic submission, dependency resolution, and TDR. Near-term patch series aim to add stress tests for concurrent entity fairness and synthetic hang/recovery scenarios to gate the fair-scheduler changes. Note: needs verification for specific patch landing timeline.
- **`drm-engine-*` fdinfo standardisation**: Ongoing work to make per-engine GPU time accounting (§8) consistent across all DRM drivers — AMDGPU, Intel Xe, Nouveau, Panfrost — so that tools like `drm_info` and `nvtop` can display comparable per-process utilisation figures without driver-specific workarounds. Note: needs verification for exact kernel version target.

### Medium-term (1–3 years)

- **Hardware preemption integration**: Most consumer GPUs today lack mid-command-buffer preemption, making the software scheduler the only arbitration layer. AMD RDNA 3/4 and Intel Xe HPC expose finer-grained preemption hooks; integrating these with the DRM scheduler's entity model would allow true time-slicing between clients rather than waiting for a job to complete before switching. [Source: Towards Fully-fledged GPU Multitasking via Proactive Memory Scheduling (arXiv 2512.24637)](https://arxiv.org/pdf/2512.24637)
- **SR-IOV and GuC multi-VM scheduling**: Intel Xe has landed SR-IOV scheduler groups; AMD is pursuing similar VF-per-VM submission paths. Coordinating the host DRM scheduler's vruntime fairness with in-hardware GuC/MES firmware schedulers — so that per-VM GPU time guarantees are enforced at both the software and firmware layers — is an active design area. [Source: Intel Xe Multi-Device SVM / SR-IOV, December 2025](https://www.linux.org/threads/phoronix-intels-xe-linux-driver-ready-with-multi-device-svm-to-end-out-2025.60682/)
- **Heterogeneous-engine credit model**: Jobs submitted to compute, video, or display engines carry driver-defined credit costs. A richer credit model — where the total credit budget is dynamically shared across engine types based on observed throughput rather than a static per-engine `credit_limit` — would improve resource utilisation for mixed workloads such as simultaneous game rendering, video transcoding, and ML inference. Note: needs verification for specific proposal or RFC.
- **Real-time / `SCHED_DEADLINE` integration for GPU**: Research (GCAPS, FIKIT) and kernel mailing-list discussion explore coupling Linux `SCHED_DEADLINE` task scheduling with GPU job scheduling so that a real-time camera pipeline's GPU work inherits the task's deadline and is boosted accordingly. [Source: GCAPS: GPU Context-Aware Preemptive Priority-based Scheduling (arXiv 2406.05221)](https://arxiv.org/pdf/2406.05221)
- **Cross-driver load balancing for compute**: In systems with multiple GPU devices (multi-GPU desktops, discrete + integrated), the scheduler's `sched_list` load-balancing currently operates within one driver domain. A unified cross-driver scheduler that can route compute jobs across AMD and Intel devices transparently is under discussion in the compute/HSA community. Note: needs verification for specific upstream proposal.

### Long-term

- **Unified GPU scheduling framework**: As AI/ML workloads drive heterogeneous compute adoption, there is architectural pressure to converge DRM GPU scheduling, DMA engine scheduling (`dmaengine`), and NPU/accelerator scheduling under a single framework with consistent fairness semantics, deadline propagation, and cgroup-based resource limits. Note: speculative; no concrete RFC as of mid-2026.
- **cgroup GPU accounting and throttling**: Linux has CPU and memory cgroup controllers; a `gpu` cgroup controller that enforces GPU time quotas per cgroup — building on `drm-engine-*` fdinfo accounting — has been discussed at kernel summits. Full implementation would require the scheduler to track entity vruntime at the cgroup level and gate submission when a cgroup exceeds its quota. Note: speculative direction.
- **Formal verification of TDR state machine**: The timeout-detection-reset (TDR) escalation path (soft reset → per-ring reset → full GPU reset) involves complex interactions with the scheduler's pending list, fence signalling, and driver callbacks. Applying formal methods (e.g., TLA+ or Dafny) to verify the absence of races and livelock in this path has been suggested but not pursued upstream. Note: speculative; no concrete proposal.

---

## 11. Integrations

- **Ch1 — DRM Architecture**: The scheduler is a core DRM subsystem component, built as a separate `drm_sched` module linked by all DRM drivers. The `drm_gpu_scheduler` struct is embedded or referenced by every driver's ring/engine object.

- **Ch3 — DRM Syncobj and Timeline Fences**: `drm_syncobj` timeline points hold `drm_sched_fence::finished` fences. `drm_syncobj_find_fence()` provides input fences for `drm_sched_job_add_dependency()`. The scheduler's output (`s_fence->finished`) is what advances a timeline semaphore point, unblocking `vkWaitSemaphores`.

- **Ch4 — GPU Memory Management**: `drm_sched_job_add_implicit_dependencies()` reads `dma_resv` fences from GEM BOs, coupling the scheduler to the memory reservation system. Jobs cannot start executing until all memory hazards for their BOs are resolved.

- **Ch5 — AMDGPU**: AMDGPU embeds one `drm_gpu_scheduler` per ring (`ring->sched`) under `drivers/gpu/drm/amd/amdgpu/`. `amdgpu_ctx` manages per-context entities across all ring types via `amdgpu_ctx_to_drm_sched_prio()`. The `amdgpu_job_timedout` callback drives TDR recovery with soft → per-ring → full-GPU escalation. `amdgpu_fdinfo.c` exports per-entity GPU time to `/proc/pid/fdinfo`.

- **Ch5 — Intel Xe**: Xe uses `xe_exec_queue` wrapping a `drm_sched_entity` per VM-per-engine. GuC submission provides hardware-level scheduling; the DRM scheduler handles dependency tracking, TDR, and accounting above it.

- **Ch8 — Nouveau**: Nouveau wraps `drm_gpu_scheduler` in its channel/ring abstraction. The NVK Vulkan driver's submission path uses the same scheduler for scheduling fairness on NVIDIA hardware under the open-source stack.

- **Ch21 — wlroots**: The wlroots DRM backend uses GBM + EGL for rendering and relies on the scheduler's HIGH-priority entity path for its render submissions. The `IN_FENCE_FD` passed to `drmModeAtomicCommit` carries the scheduler's output fence.

- **Ch75 — Explicit GPU Sync**: The `linux-drm-syncobj-v1` Wayland protocol enables compositor and client to exchange `drm_syncobj`-backed fences, replacing implicit synchronisation. The scheduler's `drm_sched_fence::finished` is the underlying signal that propagates through syncobj timeline points to Wayland protocol events.

- **Ch89 — GPU Virtualisation**: In VM pass-through and SR-IOV scenarios, each VM's workloads are submitted through a `drm_sched_entity` inside the host kernel. Fairness between VMs depends on the scheduler's vruntime algorithm; misbehaving VMs that submit excessive work accumulate vruntime and are naturally deprioritised.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
