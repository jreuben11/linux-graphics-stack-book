# Appendix G: Synchronisation Primitives Reference

**Audience**: All readers — systems/driver developers, graphics application developers, and browser/web platform engineers.

**Scope**: The Linux graphics stack exposes synchronisation at five distinct abstraction layers. This appendix is the single place to compare all twelve named primitives side-by-side, verify the conversion path between any two of them, understand their individual lifecycles via state machine diagrams, and recall the three bugs most likely to bite you at each layer. It is a working reference: once you have absorbed the canonical chapter treatment of each primitive, keep this appendix open while writing or reviewing synchronisation-sensitive code.

**Kernel baseline**: Linux 6.10 (ntsync merged). Mesa 24.x (timeline syncobj fully supported in all Mesa Vulkan drivers). Where a primitive or feature landed in a later or earlier kernel, the specific version is called out in the relevant row or pitfall entry. Appendix C §"Kernel and Mesa Version Matrix" is the authoritative version matrix.

---

## Table of Contents

- [Section 1: Introduction — Two Mental Models](#section-1-introduction--two-mental-models)
- [Section 2: Master Reference Table](#section-2-master-reference-table)
- [Section 3: Per-Primitive State Machine Diagrams](#section-3-per-primitive-state-machine-diagrams)
  - [3.1 `dma_fence`](#31-dma_fence)
  - [3.2 `dma_resv`](#32-dma_resv)
  - [3.3 `sync_file`](#33-sync_file)
  - [3.4 `drm_syncobj` binary](#34-drm_syncobj-binary)
  - [3.5 `drm_syncobj` timeline](#35-drm_syncobj-timeline)
  - [3.6 `IN_FENCE_FD` and `OUT_FENCE_PTR`](#36-in_fence_fd-and-out_fence_ptr)
  - [3.7 `wp_linux_drm_syncobj_v1`](#37-wp_linux_drm_syncobj_v1)
  - [3.8 `VkFence`](#38-vkfence)
  - [3.9 `VkSemaphore` binary](#39-vksemaphore-binary)
  - [3.10 `VkSemaphore` timeline](#310-vksemaphore-timeline)
  - [3.11 `ntsync`](#311-ntsync)
  - [3.12 `esync` / `fsync`](#312-esync--fsync)
- [Section 4: Conversion Matrix](#section-4-conversion-matrix)
- [Section 5: Common Pitfalls](#section-5-common-pitfalls)
- [Section 6: Cross-References to Canonical Chapter Sections](#section-6-cross-references-to-canonical-chapter-sections)
- [Integrations](#integrations)
- [References](#references)

---

## Section 1: Introduction — Two Mental Models

The Linux graphics synchronisation landscape is coherent if you hold two mental models simultaneously and never conflate them.

**The kernel lingua franca model.** At the base of the stack sits `dma_fence` (`include/linux/dma-fence.h`), a one-shot, reference-counted kernel object representing a single asynchronous hardware operation that will eventually complete. Every other kernel-side primitive is either a container of `dma_fence` objects, an fd-based export of one, or a named handle that allows both kernel drivers and userspace to reference the same fence. `dma_resv` is a reservation container that holds one exclusive (write) fence and zero or more shared (read) fences for a DMA-BUF. `sync_file` is an anonymous file descriptor whose sole purpose is to carry one or more `dma_fence` objects across process boundaries or from the GPU driver to the KMS display pipeline. `drm_syncobj` is a mutable named container — unlike `dma_fence` it can be reset and reused — that exists in two variants: binary (a single fence slot, can be signalled and reset repeatedly) and timeline (a monotonically increasing 64-bit sequence number, each point mapping to a `dma_fence`).

Above the kernel, the `wp_linux_drm_syncobj_v1` Wayland protocol object is simply a Wayland protocol name for a `drm_syncobj` file descriptor that the compositor and client share for per-surface acquire and release points. On the Vulkan side, `VkSemaphore` and `VkFence` are API objects whose Linux implementations in Mesa (`src/vulkan/runtime/vk_drm_syncobj.c`) are directly backed by `drm_syncobj` binary and timeline points respectively. The `VK_KHR_external_semaphore_fd` extension is not a new object; it is the protocol — two API calls, `vkGetSemaphoreFdKHR()` and `vkImportSemaphoreFdKHR()` — for exporting and importing `VkSemaphore` payloads as POSIX file descriptors (either `drm_syncobj` fds or `sync_file` fds).

**The Win32 island model.** `ntsync` (Linux 6.10, `drivers/misc/ntsync.c`), `esync`, and `fsync` are a separate synchronisation universe. They implement Win32 event, mutex, and semaphore semantics for Wine/Proton compatibility. They use the Linux scheduler — ntsync through kernel objects obtained from `/dev/ntsync`, esync through `eventfd(2)`, fsync through `sys_futex_waitv` (Linux 5.16) — but carry no GPU fence semantics whatsoever. There is no `dma_fence` bridge from this island. You cannot use a ntsync fd as an `IN_FENCE_FD` property, import one into Vulkan, or attach one to a DMA-BUF reservation. Wine/Proton manages the boundary between Win32 synchronisation and GPU work entirely at the DXVK/VKD3D-Proton D3D-to-Vulkan translation layer using Vulkan completion callbacks; the kernel never sees a direct connection.

**Primitive enumeration.** This appendix covers twelve named primitives: `dma_fence`, `dma_resv`, `sync_file`, `drm_syncobj` (binary and timeline treated as one entry with two variants), `IN_FENCE_FD`, `OUT_FENCE_PTR`, `wp_linux_drm_syncobj_v1`, `VkFence`, `VkSemaphore` binary, `VkSemaphore` timeline, `VK_KHR_external_semaphore_fd`, and the Win32-compatibility island (`ntsync`/`esync`/`fsync`, collapsed to one row in the conversion matrix since they share the same "no GPU bridge" characteristic). The master table in Section 2 carries thirteen rows (splitting `IN_FENCE_FD` and `OUT_FENCE_PTR`), the state machine section carries twelve diagrams (the extension `VK_KHR_external_semaphore_fd` has no state machine of its own — see the note at the start of Section 3), the conversion matrix is 12×12, and the cross-reference table in Section 6 carries fourteen rows (splitting binary/timeline semaphore and binary/timeline `drm_syncobj`). These counts are consistent and intentional; the splits are explained in each section.

---

## Section 2: Master Reference Table

The table below orders primitives from lowest abstraction layer (kernel-internal) to highest (application API), with the Win32-compatibility island last. Struct names, ioctl names, and function names are in `monospace`.

| Primitive | Ownership layer | Semantic class | Representation | Primary interface | Converts to/from | Chapter reference |
|---|---|---|---|---|---|---|
| `dma_fence` | kernel-only | One-shot fence; signals exactly once; cannot be reset | `struct dma_fence` (`include/linux/dma-fence.h`) | `dma_fence_signal()`, `dma_fence_add_callback()`, `dma_fence_wait()`, `dma_fence_chain` | → `sync_file` via `sync_file_create()`; → `drm_syncobj` (kernel internal); embedded in `dma_resv` slots | Ch4 §"DMA-BUF Implicit and Explicit Fencing"; Ch8 §"GPU Scheduler and Fence Handling" |
| `dma_resv` | kernel-only | Implicit fence container; one exclusive fence (write) + zero or more shared fences (read) | `struct dma_resv` (`include/linux/dma-resv.h`); accessed via `dma_resv_lock()` / `dma_resv_unlock()` ww_mutex | `dma_resv_add_fence()`, `dma_resv_wait_timeout()`, `dma_buf_poll()` (exposes as `POLLIN`/`POLLOUT` on the DMA-BUF fd) | Contains `dma_fence` objects; poll-exposed to userspace via `dma_buf_poll()`; driver-specific export to `sync_file` | Ch4 §"DMA-BUF Implicit and Explicit Fencing" |
| `sync_file` | kernel + uAPI | fd-based export of a `dma_fence` or `dma_fence_array`/`dma_fence_chain` | Anonymous file descriptor; `struct sync_file` internally (`drivers/dma-buf/sync_file.c`) | `sync_file_create()` (kernel); `SYNC_IOC_MERGE` ioctl for merge; `SYNC_IOC_FILE_INFO` for inspection; imported/exported via `DRM_IOCTL_SYNCOBJ_IMPORT_SYNC_FILE` / `DRM_IOCTL_SYNCOBJ_EXPORT_SYNC_FILE` | Wraps `dma_fence`; ↔ `drm_syncobj` via import/export ioctls; value at KMS `IN_FENCE_FD` boundary | Ch3 §"Explicit Sync: KMS Fence Properties"; Ch4 §"DMA-BUF Implicit and Explicit Fencing" |
| `drm_syncobj` (binary + timeline) | kernel + uAPI | **Binary**: holds one `dma_fence`; resettable. **Timeline**: monotonically increasing 64-bit sequence points, each mapping to a `dma_fence` | Integer handle from `DRM_IOCTL_SYNCOBJ_CREATE`; exportable as fd via `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD` | `DRM_IOCTL_SYNCOBJ_CREATE`, `DRM_IOCTL_SYNCOBJ_DESTROY`, `DRM_IOCTL_SYNCOBJ_WAIT`, `DRM_IOCTL_SYNCOBJ_TIMELINE_WAIT`, `DRM_IOCTL_SYNCOBJ_SIGNAL`, `DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL`, `DRM_IOCTL_SYNCOBJ_QUERY` | ↔ `sync_file`; → `VkSemaphore` (binary/timeline) or `VkFence` via `VK_KHR_external_semaphore_fd`; fd passed to `wp_linux_drm_syncobj_v1` | Ch3 §"DRM Sync Objects and the Explicit Sync KMS Properties"; Ch4 §"Explicit Sync as the Alternative" |
| `IN_FENCE_FD` | kernel uAPI — DRM atomic property on a plane object | Acquire fence input at the KMS atomic commit boundary; per-plane | 64-bit signed integer DRM atomic property value; value is a `sync_file` fd or -1 (no fence) | Set via `drmModeAtomicAddProperty()` with property name `IN_FENCE_FD` on a **plane** object before `drmModeAtomicCommit()` | Consumes `sync_file` fd; pairs with `OUT_FENCE_PTR` | Ch3 §"KMS Explicit Sync Properties" |
| `OUT_FENCE_PTR` | kernel uAPI — DRM atomic property on a CRTC object | Release fence output at the KMS atomic commit boundary; per-CRTC; signals at VBLANK | 64-bit unsigned integer DRM atomic property value; value is a pointer to a `uint64_t` the kernel fills with the resulting `sync_file` fd after commit | Set via `drmModeAtomicAddProperty()` with property name `OUT_FENCE_PTR` on a **CRTC** object; pointed-to `uint64_t` receives the fd after `drmModeAtomicCommit()` returns | Produces `sync_file` fd; importable into `drm_syncobj` via `DRM_IOCTL_SYNCOBJ_IMPORT_SYNC_FILE` | Ch3 §"KMS Explicit Sync Properties" |
| `wp_linux_drm_syncobj_v1` | Wayland protocol; compositor-advertised global | Protocol face of `drm_syncobj` timeline points; per-surface acquire/release fence binding | Protocol objects: `wp_linux_drm_syncobj_manager_v1` (global factory), `wp_linux_drm_syncobj_timeline_v1` (wraps a `drm_syncobj` fd), `wp_linux_drm_syncobj_surface_v1` (per-surface acquire/release bindings) | `wp_linux_drm_syncobj_manager_v1.get_surface()`, `wp_linux_drm_syncobj_surface_v1.set_acquire_point()`, `wp_linux_drm_syncobj_surface_v1.set_release_point()` | Wraps `drm_syncobj` fd; indirectly connects to `VkSemaphore` (timeline) via `VK_KHR_external_semaphore_fd` | Ch3 §"`wp_linux_drm_syncobj_v1`"; Ch20 §"Key Extension Protocols" |
| `VkFence` | Vulkan API (CPU-side wait) | CPU-observable completion signal for a GPU queue submission; binary | `VkFence` handle; internally maps to `drm_syncobj` binary point on Linux | `vkCreateFence()`, `vkQueueSubmit(..., fence)`, `vkWaitForFences()`, `vkResetFences()`; exported via `vkGetFenceFdKHR()` with `VK_EXTERNAL_FENCE_HANDLE_TYPE_OPAQUE_FD_BIT` or `VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT` | ↔ `drm_syncobj` binary; → `sync_file` via `SYNC_FD` export | Ch24 §"Synchronisation: Fences, Binary Semaphores, Timeline Semaphores, Host Sync" |
| `VkSemaphore` binary | Vulkan API (GPU-to-GPU sync) | Binary semaphore; unsignalled/signalled; must be consumed exactly once per signal | `VkSemaphore` handle with `VK_SEMAPHORE_TYPE_BINARY`; internally maps to `drm_syncobj` binary point on Linux | `vkCreateSemaphore()`, signal via `VkSubmitInfo::pSignalSemaphores`, wait via `VkSubmitInfo::pWaitSemaphores`; exported via `vkGetSemaphoreFdKHR()` | ↔ `drm_syncobj` binary; → `sync_file` via `SYNC_FD` export | Ch24 §"Synchronisation"; Appendix A §"binary semaphore" |
| `VkSemaphore` timeline | Vulkan API (GPU-to-GPU and GPU-to-CPU) | Timeline semaphore; 64-bit monotonically increasing payload | `VkSemaphore` handle with `VK_SEMAPHORE_TYPE_TIMELINE`; internally maps to `drm_syncobj` timeline on Linux | `vkCreateSemaphore()` with `VkSemaphoreTypeCreateInfo`, signal value in `VkTimelineSemaphoreSubmitInfo`, `vkSignalSemaphore()`, `vkWaitSemaphores()`; exported via `vkGetSemaphoreFdKHR()` with `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT_KHR` | ↔ `drm_syncobj` timeline; fd → `wp_linux_drm_syncobj_timeline_v1` | Ch24 §"Synchronisation"; Ch3 §"Explicit Sync Closes the Loop" |
| `VK_KHR_external_semaphore_fd` | Vulkan API + kernel uAPI bridge | Export/import mechanism — not a stateful object; adds `vkGetSemaphoreFdKHR()` and `vkImportSemaphoreFdKHR()` to `VkDevice` | Extension interface; no distinct object type; handle type selects `drm_syncobj` fd (`OPAQUE_FD`) vs `sync_file` (`SYNC_FD`) | `vkGetSemaphoreFdKHR()`, `vkImportSemaphoreFdKHR()`; companion `VK_KHR_external_fence_fd` provides same for `VkFence` | Linchpin of cross-layer conversion: connects `VkSemaphore` ↔ `drm_syncobj` ↔ `sync_file` ↔ `IN_FENCE_FD`/`OUT_FENCE_PTR` | Ch24 §"Synchronisation"; Ch3 §"NVIDIA's Explicit Sync Support"; Ch25 §"Vulkan External Semaphores + CUDA External Semaphores" |
| `ntsync` (Linux 6.10+) | kernel + uAPI; `/dev/ntsync` character device; **Win32-compatibility island** | Win32 kernel synchronisation objects (event, mutex, semaphore) managed by the Linux kernel scheduler; no `dma_fence` semantics | fd-based objects obtained via `NTSYNC_IOC_CREATE_EVENT`, `NTSYNC_IOC_CREATE_MUTEX`, `NTSYNC_IOC_CREATE_SEM`; wait via `NTSYNC_IOC_WAIT_ANY` / `NTSYNC_IOC_WAIT_ALL`; `drivers/misc/ntsync.c` | `open("/dev/ntsync")`, then creation ioctls (`NTSYNC_IOC_CREATE_*`), operation ioctls (`NTSYNC_IOC_SET_EVENT`, `NTSYNC_IOC_RESET_EVENT`, `NTSYNC_IOC_SEM_POST`, `NTSYNC_IOC_MUTEX_UNLOCK`), wait ioctls (`NTSYNC_IOC_WAIT_ANY`, `NTSYNC_IOC_WAIT_ALL`) | **None** — no `dma_fence` bridge; cannot be used as `IN_FENCE_FD`, imported into Vulkan, or attached to DMA-BUF | Ch28 §"The Synchronisation Story: esync, fsync, and ntsync" |
| `esync` / `fsync` | Wine-internal; no kernel driver; **Win32-compatibility island** | Win32 event/mutex/semaphore emulation; pre-ntsync workarounds; no GPU semantics | esync: `eventfd(2)` fds tracked in Wine's sync object table. fsync: futex-based objects in shared memory, waited on via `sys_futex_waitv` (Linux 5.16+) | esync: `eventfd(2)`, `epoll_wait()`; `WINEESYNC=1`. fsync: `sys_futex_waitv`, shared memory; `WINEFSYNC=1`. Source: Wine `dlls/ntdll/unix/sync.c` | **None** — same Win32 island as ntsync; `eventfd`/futex fds are not `sync_file`s and cannot be used as `IN_FENCE_FD` | Ch28 §"The Synchronisation Story: esync, fsync, and ntsync" |

---

## Section 3: Per-Primitive State Machine Diagrams

The ASCII-art notation used throughout this section:

```
[STATE] --event/action--> [STATE]
```

States are drawn as labelled boxes. Terminal states are marked with an asterisk `*`. For kernel objects, states reflect both kernel-internal state and userspace-observable file-descriptor state. Diagrams are self-contained; no prior chapter context is assumed.

**Note on `VK_KHR_external_semaphore_fd`**: This primitive is an export/import mechanism — a pair of Vulkan API calls — not a stateful object in its own right. Its entire state is that of the `VkSemaphore` being exported or imported. Accordingly, Section 3 contains eleven diagrams for twelve primitives; there is no §3.x entry for `VK_KHR_external_semaphore_fd`. The diagrams for the semaphores it acts on are in §3.9 and §3.10.

### 3.1 `dma_fence`

The `dma_fence` object (`include/linux/dma-fence.h`) has the simplest lifecycle of all synchronisation primitives: it is created in a pending state, signals exactly once when the underlying hardware operation completes, then is freed when all references drop. There is no reset. Once signalled, the fence is permanently signalled until freed.

```
+---------------------+
|  CREATED / PENDING  |
+---------------------+
         |
         |  dma_fence_signal()  (called by driver interrupt handler or work queue)
         |
         v
+---------------------+
|      SIGNALLED      |
+---------------------+
         |              \
         |               \  dma_fence_add_callback() callbacks fire here
         |                \  dma_fence_wait() returns here
         |
         |  all references dropped (dma_fence_put() reaches refcount 0)
         |
         v
+---------------------+ *
|       FREED         |
+---------------------+
```

Key constraints: `dma_fence_signal()` may be called from interrupt context if `DMA_FENCE_FLAG_ENABLE_SIGNAL_BIT` is set. `dma_fence_wait()` is blocking but is permitted to be called while holding `dma_resv_lock()`. Callbacks registered via `dma_fence_add_callback()` must be removed with `dma_fence_remove_callback()` before the callback structure is freed.

### 3.2 `dma_resv`

The `dma_resv` object (`include/linux/dma-resv.h`) is a wound-wait mutex-protected container embedded in every `dma_buf`. Its lifecycle tracks both the lock state and the set of fences it holds.

```
+----------------------------+
|  UNLOCKED, fences empty    |
+----------------------------+
         |
         |  dma_resv_lock(obj, ctx)
         v
+----------------------------+
|  LOCKED                    |
+----------------------------+
         |                      |
         |  dma_resv_add_fence(  |  dma_resv_add_fence(
         |  obj, fence,         |  obj, fence,
         |  DMA_RESV_USAGE_WRITE)|  DMA_RESV_USAGE_READ)
         v                      v
+----------------------------+
|  LOCKED, exclusive fence   |
|  and/or shared fences set  |
+----------------------------+
         |
         |  dma_resv_unlock(obj)
         v
+----------------------------+
|  UNLOCKED, fences set      |
+----------------------------+
         |                      |
         |  dma_resv_wait_       |  dma_resv_lock() +
         |  timeout() or         |  dma_resv_add_fence()
         |  dma_buf_poll()       |  (replaces old fences)
         v                      |
+----------------------------+ <-+
|  UNLOCKED, fences signalled|
+----------------------------+
```

Note that `dma_resv_reserve_fences()` must be called before `dma_resv_add_fence()` to pre-allocate space in the shared fence array. Shared fences accumulate until an exclusive fence is added or until `dma_resv_prune()` is called explicitly; in long-lived buffer pool scenarios this accumulation must be managed.

### 3.3 `sync_file`

A `sync_file` (`drivers/dma-buf/sync_file.c`) is an anonymous file descriptor wrapping one or more `dma_fence` objects. Its state from a userspace perspective is entirely determined by whether the underlying fences have signalled.

```
+-------------------------------------------+
|  CREATED  (wrapping dma_fence(s))          |
+-------------------------------------------+
         |                  |                  |
         |  poll(POLLIN)     |  SYNC_IOC_MERGE  |  passed as IN_FENCE_FD
         |  or read(fd)      |  with another    |  to drmModeAtomicCommit()
         |  blocks until     |  sync_file       |
         |  fences signal    v                  |
         |              +----------+           |
         |              | NEW      |           |
         |              | sync_file|           |
         |              | (union)  |           |
         |              +----------+           |
         v                                     v
+-------------------------------------------+
|  FENCES SIGNALLED  (fd still valid)        |
+-------------------------------------------+
         |
         |  close(fd) or last reference dropped
         v
+-------------------------------------------+ *
|  FREED  (dma_fence refcounts decremented)  |
+-------------------------------------------+
```

### 3.4 `drm_syncobj` binary

The binary `drm_syncobj` (`drivers/gpu/drm/drm_syncobj.c`) differs fundamentally from `dma_fence` by being resettable. It moves between unsignalled and signalled states as it is reused across frames or submissions.

```
+----------------------------+
|  CREATED, UNSIGNALLED      |
|  (no fence attached)       |
+----------------------------+
         |                      |
         |  DRM_IOCTL_SYNCOBJ_   |  GPU submission signals it
         |  SIGNAL (CPU-side)    |  (driver calls drm_syncobj_
         |                       |   replace_fence() internally)
         v                       v
+----------------------------+
|  SIGNALLED                 |
|  (fence attached + done)   |
+----------------------------+
         |            |            |
         |  DRM_IOCTL_ |  DRM_IOCTL_ |  DRM_IOCTL_SYNCOBJ_
         |  SYNCOBJ_   |  SYNCOBJ_   |  HANDLE_TO_FD
         |  RESET      |  EXPORT_    |  --> fd exportable as
         |             |  SYNC_FILE  |  VkFence/VkSemaphore via
         v             v            |  vkImportSemaphoreFdKHR
+----------+  +----------+         |
| CREATED, |  | sync_file|         v
| UNSIG-   |  | fd       |  [Vulkan object takes ownership]
| NALLED   |  | produced |
+----------+  +----------+
         |
         |  DRM_IOCTL_SYNCOBJ_WAIT blocks until SIGNALLED
```

### 3.5 `drm_syncobj` timeline

The timeline `drm_syncobj` adds a monotonically increasing sequence number. Each integer point on the timeline corresponds to one `dma_fence` (or is pending). Points are never reset — only advanced.

```
+----------------------------------+
|  CREATED, seqno = 0 (no fences) |
+----------------------------------+
         |
         |  GPU submission signals point N
         |  (drm_syncobj_timeline_add_point() kernel-internal)
         v
+----------------------------------+
|  POINT N SIGNALLED               |
|  (points < N also signalled;     |
|   points > N pending)            |
+----------------------------------+
         |            |
         |  DRM_IOCTL_  |  DRM_IOCTL_SYNCOBJ_TIMELINE_SIGNAL(N)
         |  SYNCOBJ_    |  (CPU-side signal of point N)
         |  TIMELINE_   |
         |  WAIT(N)     |  DRM_IOCTL_SYNCOBJ_QUERY
         |  blocks      |  --> returns highest signalled seqno
         v              |
  [returns when        |
   point N signalled]  |
                        v
+----------------------------------+
|  DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD  |
|  --> fd passed to Wayland or     |
|  vkImportSemaphoreFdKHR          |
+----------------------------------+
```

### 3.6 `IN_FENCE_FD` and `OUT_FENCE_PTR`

These two DRM atomic properties work as a matched pair across the KMS commit boundary. `IN_FENCE_FD` is a plane-level acquire fence; `OUT_FENCE_PTR` is a CRTC-level release fence.

```
[Application holds sync_file fd]
  (from vkGetSemaphoreFdKHR(SYNC_FD) or from a prior OUT_FENCE_PTR)
         |
         |  drmModeAtomicAddProperty(plane_id, IN_FENCE_FD, fd)
         |  drmModeAtomicAddProperty(crtc_id, OUT_FENCE_PTR, &fence_out)
         v
+------------------------------------------+
|  drmModeAtomicCommit()                   |
|  KMS waits for IN_FENCE_FD to signal;    |
|  schedules scanout for next VBLANK;      |
|  writes new sync_file fd to *fence_out   |
+------------------------------------------+
         |
         |  drmModeAtomicCommit() returns 0
         v
+------------------------------------------+
|  fence_out now holds a valid sync_file fd|
|  that signals at VBLANK                  |
+------------------------------------------+
         |                    |
         |  poll(POLLIN) on   |  pass fence_out as next
         |  fence_out fd;     |  frame's IN_FENCE_FD
         |  blocks until      |  OR import into VkSemaphore
         |  VBLANK            |  via vkImportSemaphoreFdKHR
         v                    v
  [VBLANK event]        [next frame sync chain]
```

### 3.7 `wp_linux_drm_syncobj_v1`

The `wp_linux_drm_syncobj_v1` Wayland protocol (`wayland-protocols` staging, merged in wayland-protocols 1.34) maps `drm_syncobj` timeline sequence numbers to per-surface compositor acquire and release points.

```
[Client exports VkSemaphore (timeline) as drm_syncobj fd]
  via vkGetSemaphoreFdKHR(OPAQUE_FD)
         |
         |  wp_linux_drm_syncobj_manager_v1.import_timeline(drm_syncobj_fd)
         v
+---------------------------------------------------+
|  wp_linux_drm_syncobj_timeline_v1 created         |
|  (wraps the drm_syncobj fd in compositor)         |
+---------------------------------------------------+
         |
         |  wp_linux_drm_syncobj_surface_v1.set_acquire_point(timeline, seqno_A)
         |  wp_linux_drm_syncobj_surface_v1.set_release_point(timeline, seqno_R)
         |  wl_surface.commit()
         v
+---------------------------------------------------+
|  Compositor receives commit;                      |
|  waits for timeline point seqno_A to signal;      |
|  composites from the buffer;                      |
|  signals timeline point seqno_R when done         |
+---------------------------------------------------+
         |
         |  seqno_R signalled by compositor
         v
+---------------------------------------------------+
|  Client's VkSemaphore at value seqno_R signalled  |
|  Client may reuse buffer                          |
+---------------------------------------------------+
```

Note: release seqno_R must be strictly greater than acquire seqno_A on the same timeline.

### 3.8 `VkFence`

`VkFence` is the Vulkan CPU-side completion signal. On Linux it is internally a `drm_syncobj` binary point (`src/vulkan/runtime/vk_drm_syncobj.c`).

```
+----------------------------+
|  UNSIGNALLED               |
|  (created or after reset)  |
+----------------------------+
         |
         |  vkQueueSubmit(..., fence)
         v
+----------------------------+
|  PENDING                   |
|  (GPU work submitted)      |
+----------------------------+
         |
         |  GPU work completes
         v
+----------------------------+
|  SIGNALLED                 |
+----------------------------+
         |            |            |
         |  vkReset-  |  vkGetFence|  vkGetFenceFdKHR
         |  Fences()  |  FdKHR     |  (SYNC_FD)
         v  (reuse)   |  (OPAQUE_  |  --> sync_file fd;
                      |  FD) -->   |      fence resets to
  +-----------+       |  drm_sync- |      UNSIGNALLED
  | UNSIG-    |       |  obj fd;   |      (one-shot export)
  | NALLED    |       |  fence     |
  +-----------+       |  stays     |
                      |  SIGNALLED |
```

### 3.9 `VkSemaphore` binary

`VkSemaphore` with `VK_SEMAPHORE_TYPE_BINARY`. Backed on Linux by a `drm_syncobj` binary point. Must be consumed exactly once per signal.

```
+----------------------------+
|  UNSIGNALLED               |
+----------------------------+
         |
         |  vkQueueSubmit(pSignalSemaphores = [this semaphore])
         |  GPU work completes
         v
+----------------------------+
|  SIGNALLED                 |
+----------------------------+
         |            |            |
         |  vkQueue-  |  vkGetSem- |  vkGetSemaphoreFdKHR
         |  Submit    |  aphoreFd  |  (SYNC_FD)
         |  (pWait-   |  KHR       |  --> sync_file fd;
         |  Semaphore |  (OPAQUE_  |      semaphore resets
         |  s)        |  FD) -->   |      to UNSIGNALLED
         v            |  drm_sync- |      (one-shot export)
  +-----------+       |  obj fd    |
  | UNSIG-    |       +------------+
  | NALLED    |
  +-----------+
```

**Note**: Undefined behaviour if a binary semaphore is signalled without a prior wait consuming the previous signal, or waited on before it has been signalled.

### 3.10 `VkSemaphore` timeline

`VkSemaphore` with `VK_SEMAPHORE_TYPE_TIMELINE`. Backed on Linux by a `drm_syncobj` timeline. Multiple producers and consumers can signal and wait at different values.

```
+-----------------------------+
|  CREATED at initial value V0|
+-----------------------------+
         |
         |  vkQueueSubmit(signal value V1 > V0)
         |  GPU work completes
         v
+-----------------------------+
|  AT VALUE V1                |
|  (V0 and all prior values   |
|   are permanently signalled)|
+-----------------------------+
         |            |            |
         |  vkWait-   |  vkSignal- |  vkGetSemaphoreFdKHR
         |  Semaphore |  Semaphore |  (OPAQUE_FD)
         |  s(V1)     |  (V2>V1,   |  --> drm_syncobj
         |  blocks    |  CPU-side) |  timeline fd
         |  until V1  |           |
         v  signalled  v           v
  [returns]  AT VALUE V2    [fd passed to Wayland or
                             another VkDevice via
                             vkImportSemaphoreFdKHR]
```

**Note**: `SYNC_FD` handle type (one-shot `sync_file` export) is **not** available for timeline semaphores per the `VK_KHR_external_semaphore_fd` specification; only `OPAQUE_FD` (`drm_syncobj` fd) is supported for timeline export.

### 3.11 `ntsync`

The `ntsync` driver (`drivers/misc/ntsync.c`, Linux 6.10) exposes Win32 NT synchronisation objects as file descriptors. These are CPU scheduler objects only.

```
+----------------------------+
|  /dev/ntsync opened        |
+----------------------------+
         |
         |  NTSYNC_IOC_CREATE_EVENT /
         |  NTSYNC_IOC_CREATE_MUTEX /
         |  NTSYNC_IOC_CREATE_SEM
         v
+----------------------------+
|  OBJECT FD obtained        |
+----------------------------+
 (for event fd):
         |             |
         |  NTSYNC_IOC_ |  NTSYNC_IOC_RESET_EVENT
         |  SET_EVENT   |
         v             v
+----------+  +----------+
| SIGNALLED|  |UNSIGNALLED|
+----------+  +----------+
         |
         |  NTSYNC_IOC_WAIT_ANY([fd1, fd2, ...])
         |  blocks until any object signals;
         |  returns index of signalled object
         |
         |  close(fd)
         v
+----------------------------+ *
|  OBJECT FREED              |
+----------------------------+
```

`NTSYNC_IOC_WAIT_ALL` blocks until all listed objects are simultaneously acquirable (analogous to Win32 `WaitForMultipleObjects` with `bWaitAll=TRUE`). `NTSYNC_IOC_MUTEX_KILL` marks a mutex owner as dead, triggering abandoned-mutex semantics on next wait.

### 3.12 `esync` / `fsync`

These are Wine-internal mechanisms with no kernel module of their own.

**esync** (Linux `eventfd(2)`, no kernel version minimum beyond `eventfd` availability):

```
+--------------------------------------+
|  eventfd(0, EFD_SEMAPHORE|EFD_NONBLOCK) created  |
+--------------------------------------+
         |
         |  write(fd, &count, 8)   (signal: increment counter)
         v
+--------------------------------------+
|  SIGNALLED  (counter > 0)           |
+--------------------------------------+
         |
         |  read(fd, &count, 8)    (consume: decrement counter)
         v
+--------------------------------------+
|  UNSIGNALLED  (counter = 0)         |
+--------------------------------------+
         |
         |  epoll_wait() or poll()  (blocks until counter > 0)
         |  close(fd)
         v
+--------------------------------------+ *
|  FREED                               |
+--------------------------------------+
```

**fsync** (Linux `sys_futex_waitv`, available from Linux 5.16):

```
+----------------------------------------------+
|  Shared memory futex word allocated,         |
|  value = UNSIGNALLED (0)                     |
+----------------------------------------------+
         |
         |  atomic write to futex word (value > 0)
         v
+----------------------------------------------+
|  SIGNALLED  (sleeping threads wake via        |
|  futex_waitv notification)                    |
+----------------------------------------------+
         |
         |  futex_waitv([{futex_word, expected}])
         |  blocks until futex word != expected
         |  (re-read value to determine signal state)
```

---

## Section 4: Conversion Matrix

The table below is a 12×12 matrix of all documented conversion paths. Rows are source primitives; columns are target primitives. Cell values use the following codes:

- **Direct: `<call>`** — a single documented kernel/Vulkan API call
- **Indirect N: `step1` → `step2` [→ ...]** — N intermediate steps required
- **None** — no conversion path exists or is meaningful

The diagonal is marked `—` (identity, no conversion needed). The entire Win32-compatibility island row and column are separated by the notation `[WIN32]` because all cells in that row and column that do not involve the island itself are **None**; no kernel or Vulkan API bridges the Win32 island to the GPU fence subsystem.

Abbreviations used in cells: **SF** = `sync_file`, **DSO** = `drm_syncobj`, **dF** = `dma_fence`, **dR** = `dma_resv`, **IFF** = `IN_FENCE_FD`, **OFP** = `OUT_FENCE_PTR`, **WDS** = `wp_linux_drm_syncobj_v1`/`wp_linux_drm_syncobj_timeline_v1`, **VkF** = `VkFence`, **VkSB** = `VkSemaphore` binary, **VkST** = `VkSemaphore` timeline, **EXT** = `VK_KHR_external_semaphore_fd` (mechanism), **NTS** = `ntsync`/`esync`/`fsync`.

| From \ To | dma_fence | dma_resv | sync_file | drm_syncobj | IN_FENCE_FD | OUT_FENCE_PTR | wp_drm_syncobj | VkFence | VkSem binary | VkSem timeline | ext_sem_fd | [WIN32] ntsync |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **dma_fence** | — | Direct: `dma_resv_add_fence()` | Direct: `sync_file_create()` | Direct (kernel): `drm_syncobj_replace_fence()` | Indirect 2: dF→SF→IFF | None (OFP is output-only) | Indirect 3: dF→DSO fd→WDS | Indirect 2: dF→DSO→`vkImportFenceFdKHR` | Indirect 2: dF→DSO→`vkImportSemaphoreFdKHR` | Indirect 2: dF→DSO→`vkImportSemaphoreFdKHR(OPAQUE_FD)` | Indirect 3: dF→DSO→`vkImportSemaphoreFdKHR`→EXT mechanism | **None** |
| **dma_resv** | Direct: `dma_resv_excl_fence()` or iterate shared | — | Indirect 2: dR→dF→SF | Indirect 2: dR→dF→DSO | Indirect 3: dR→dF→SF→IFF | None | Indirect 4 | Indirect 3 | Indirect 3 | Indirect 3 | Indirect 4 | **None** |
| **sync_file** | Direct: underlying `dma_fence` via `sync_file_get_fence()` | Indirect 2: SF→dF→dR | — | Direct: `DRM_IOCTL_SYNCOBJ_IMPORT_SYNC_FILE` | Direct: pass fd value as property | None | Indirect 2: SF→DSO fd→WDS | Indirect 2: SF→DSO→`vkImportFenceFdKHR(SYNC_FD)` | Direct: `vkImportSemaphoreFdKHR(SYNC_FD, sf_fd)` | None (SYNC_FD restricted to binary; timeline requires OPAQUE_FD) | Direct (is the mechanism for SYNC_FD) | **None** |
| **drm_syncobj** | Direct: `drm_syncobj_get_fence()` (kernel) | Indirect 2: DSO→dF→dR | Direct: `DRM_IOCTL_SYNCOBJ_EXPORT_SYNC_FILE` | — | Indirect 2: DSO→SF→IFF | None | Direct: export fd via `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD`; pass to `wp_linux_drm_syncobj_manager_v1.import_timeline()` | Direct: `vkImportFenceFdKHR(OPAQUE_FD)` | Direct: `vkImportSemaphoreFdKHR(OPAQUE_FD)` | Direct: `vkImportSemaphoreFdKHR(OPAQUE_FD)` | Direct (is the mechanism for OPAQUE_FD) | **None** |
| **IN_FENCE_FD** | Indirect 2: IFF→SF→dF (kernel consumes it) | Indirect 3 | Direct: the fd value *is* a SF | Indirect 2: SF→`DRM_IOCTL_SYNCOBJ_IMPORT_SYNC_FILE` | — | None (IFF is input, OFP is output) | Indirect 3: IFF→DSO→WDS | Indirect 3 | Indirect 3 | Indirect 4 | Indirect 3 | **None** |
| **OUT_FENCE_PTR** | Indirect 2: OFP→SF→dF | Indirect 3 | Direct: the produced fd *is* a SF | Indirect 2: SF→`DRM_IOCTL_SYNCOBJ_IMPORT_SYNC_FILE` | Direct: produced fd used as next commit's IFF | — | Indirect 3: OFP→DSO→WDS | Indirect 3: OFP→SF→DSO→`vkImportFenceFdKHR` | Indirect 3: OFP→SF→`vkImportSemaphoreFdKHR(SYNC_FD)` | Indirect 4: OFP→SF→DSO→`vkImportSemaphoreFdKHR(OPAQUE_FD)` | Indirect 3 | **None** |
| **wp_drm_syncobj** | Indirect 3: WDS→DSO→dF (kernel path) | Indirect 4 | Indirect 2: WDS→DSO→SF | Direct: the underlying `drm_syncobj` fd | Indirect 3 | None | — | Indirect 3 | Indirect 3 | Direct: underlying DSO→`vkImportSemaphoreFdKHR(OPAQUE_FD)` | Direct: underlying DSO fd | **None** |
| **VkFence** | Indirect 2: `vkGetFenceFdKHR(SYNC_FD)`→SF→dF | Indirect 3 | Direct: `vkGetFenceFdKHR(SYNC_FD)` | Direct: `vkGetFenceFdKHR(OPAQUE_FD)` | Indirect 2: `vkGetFenceFdKHR(SYNC_FD)`→IFF | None | Indirect 2: VkF→DSO fd→WDS | — | None (type mismatch: VkFence is CPU-visible; VkSemaphore is GPU-GPU) | None | Direct (`VK_KHR_external_fence_fd` companion extension) | **None** |
| **VkSem binary** | Indirect 2: `vkGetSemaphoreFdKHR(SYNC_FD)`→SF→dF | Indirect 3 | Direct: `vkGetSemaphoreFdKHR(SYNC_FD)` | Direct: `vkGetSemaphoreFdKHR(OPAQUE_FD)` | Indirect 2: VkSB→SF→IFF | None | Indirect 2: VkSB→DSO fd→WDS | None (VkSemaphore GPU-GPU; VkFence CPU-wait; different purposes) | — | None (type mismatch: binary vs timeline) | Direct (is the extension) | **None** |
| **VkSem timeline** | Indirect 2: `vkGetSemaphoreFdKHR(OPAQUE_FD)`→DSO→dF | Indirect 3 | **None** (SYNC_FD restricted to binary per VK_KHR_external_semaphore_fd spec) | Direct: `vkGetSemaphoreFdKHR(OPAQUE_FD)` → `drm_syncobj` fd | Indirect 3: OPAQUE_FD→DSO→SF→IFF | None | Direct: OPAQUE_FD fd passed to `wp_linux_drm_syncobj_manager_v1.import_timeline()` | Indirect 2: VkST→DSO→`vkImportFenceFdKHR` | None (type mismatch) | — | Direct (is the extension) | **None** |
| **ext_sem_fd** | Indirect (depends on which semaphore is carried) | — | Via the semaphore being exported | Via the semaphore being exported | Via the semaphore being exported | — | Via the semaphore being exported | Via `VK_KHR_external_fence_fd` | Direct: export/import | Direct: export/import | — | **None** |
| **[WIN32] ntsync** | **None** | **None** | **None** | **None** | **None** | **None** | **None** | **None** | **None** | **None** | **None** | — |

**Prose note on the Win32 island**: The Win32-compatibility island (`ntsync`, `esync`, `fsync`) is entirely disconnected from the `dma_fence` subsystem at the kernel level. No kernel API bridges this gap. Wine/Proton manages the boundary at the D3D-to-Vulkan translation layer in DXVK/VKD3D-Proton: GPU work is submitted via Vulkan (`VkSemaphore`/`VkFence`) and Win32 synchronisation is handled separately. DXVK signals Win32 events from Vulkan completion callbacks at the D3D API boundary — not at the kernel level. This architecture means that from the kernel's perspective, a Proton game is simply a Vulkan application; the Win32 synchronisation runs in parallel in the same process but touches entirely different kernel subsystems.

**Prose note on `VkSemaphore` timeline → `sync_file`**: The `VK_KHR_external_semaphore_fd` specification explicitly restricts the `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT_KHR` handle type to binary semaphores. A timeline semaphore may only be exported as `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT_KHR` (a `drm_syncobj` fd). This means the three-step conversion timeline → `sync_file` → `IN_FENCE_FD` must pass through a `drm_syncobj` intermediate.

---

## Section 5: Common Pitfalls

Layer taxonomy for readers looking up by layer rather than by primitive:

- **Kernel DRM layer**: `dma_fence`, `dma_resv`, `sync_file`, `drm_syncobj` (binary and timeline) — see §5.1–5.4
- **KMS layer**: `IN_FENCE_FD`, `OUT_FENCE_PTR` — see §5.5
- **Wayland protocol layer**: `wp_linux_drm_syncobj_v1` — see §5.6
- **Vulkan API layer**: `VkFence`, `VkSemaphore` binary, `VkSemaphore` timeline, `VK_KHR_external_semaphore_fd` — see §5.7–5.10
- **Win32-compatibility layer**: `ntsync`, `esync`, `fsync` — see §5.11–5.12

---

#### 5.1 `dma_fence`

1. **Fence signalled from wrong context** — Calling `dma_fence_signal()` while holding a spinlock that a registered fence callback also acquires creates a deadlock: the callback fires inside `dma_fence_signal()`, tries to acquire the same lock, and spins forever. Detect with `lockdep` (`CONFIG_LOCKDEP=y`); the lock dependency chain will be reported on first occurrence. Fix: call `dma_fence_signal()` only outside the problematic lock scope, or use `dma_fence_signal_locked()` only within the correct predetermined lock scope where callbacks are known not to re-acquire it.

2. **Missing callback removal on fence destruction** — Registering a callback via `dma_fence_add_callback()` and then freeing the `dma_fence_cb` structure before the fence signals leaves a dangling pointer: the fence will fire the callback into freed memory. Detect with KASAN or KCSAN; the use-after-free will appear in the callback invocation path. Fix: always call `dma_fence_remove_callback()` before freeing the callback structure; check the return value to confirm whether the callback was already running.

3. **Assuming fence completion implies memory coherency** — Signalling a `dma_fence` does not flush CPU caches on non-coherent architectures. On ARM and RISC-V devices without hardware cache coherency between GPU and CPU, GPU-written data is not visible to the CPU even after the fence signals. Detect by observing stale data reads after GPU completion on non-coherent hardware. Fix: call the appropriate cache maintenance operation (`dma_sync_single_for_cpu()` or the driver-specific cache invalidation path) after the fence signals and before reading GPU-written memory from the CPU.

---

#### 5.2 `dma_resv`

1. **Locking order violation with `ww_mutex`** — `dma_resv` uses wound-wait mutexes (`ww_mutex`). Acquiring multiple reservation objects without using a `ww_acquire_ctx` correctly (or without using the `drm_exec` helper that wraps the wound-wait loop) produces livelock or deadlock under concurrent lock contention. Detect with `lockdep` and `CONFIG_PROVE_LOCKING`; wound-wait violations are reported as "possible deadlock" with the acquire chain. Fix: use `drm_exec` (`include/drm/drm_exec.h`) for multi-object locking in any path that must hold more than one reservation lock simultaneously.

2. **Stale shared fence accumulation** — Shared fences in a `dma_resv` are only removed when the reservation object is flushed or a new exclusive fence replaces them. In long-lived buffer pool implementations, the shared fence array grows unboundedly across many read submissions, consuming memory and slowing `dma_resv_wait_timeout()`. Detect by monitoring `dma_resv` array sizes in `dma-buf-debugfs` or by counting allocations in `dma_resv_reserve_fences()`. Fix: call `dma_resv_prune()` periodically in buffer pool reuse paths to remove signalled fences from the array before reuse.

3. **Reading exclusive fence without lock** — Calling `dma_resv_excl_fence()` without holding the reservation lock is a TOCTOU race on SMP: another CPU may be atomically installing a new fence in the exclusive slot between the read and the use. Detect with KCSAN (`CONFIG_KCSAN=y`); the tool will report the conflicting access pair. Fix: always hold `dma_resv_lock()` when reading the exclusive fence slot, or use the RCU-protected `dma_resv_excl_fence()` path with explicit `rcu_read_lock()` when a read-only snapshot is sufficient.

---

#### 5.3 `sync_file`

1. **Passing an already-closed `sync_file` fd as `IN_FENCE_FD`** — After `close(fd)`, the numeric fd value is returned to the kernel's fd table for reuse. A subsequent `drmModeAtomicAddProperty(..., IN_FENCE_FD, old_fd)` may reference a completely different file or return `EINVAL`. This is especially common in code that closes the fence fd after inspecting it with `SYNC_IOC_FILE_INFO`. Detect by auditing fd lifetimes with `strace` or by enabling `CONFIG_DEBUG_ATOMIC_SLEEP`. Fix: obtain a fresh `sync_file` fd from `vkGetSemaphoreFdKHR()` or `DRM_IOCTL_SYNCOBJ_EXPORT_SYNC_FILE` for every commit; never reuse a fd value after `close()`.

2. **Merging fences when only the most-recent matters** — `SYNC_IOC_MERGE` creates a `dma_fence_array` that waits for all constituent fences. If only the most-recent rendering fence needs to complete before scanout (and earlier frames have already been displayed), merging adds unnecessary wait dependencies and increases fence object overhead. Detect by profiling frame latency and noticing that `SYNC_IOC_MERGE` fan-in grows across frames. Fix: pass only the most-recent `sync_file` fd directly as `IN_FENCE_FD`; discard older fences rather than accumulating them.

3. **Blocking the compositor thread on `SYNC_IOC_FILE_INFO`** — On older kernel versions, `SYNC_IOC_FILE_INFO` blocks until all fences in the sync file are resolved before returning fence status information. Calling it synchronously on the frame-delivery or commit thread causes unbounded stalls when GPU work is delayed. Detect with frame timing traces (`perf record -e drm:drm_vblank_event`). Fix: use `poll()` with a short timeout to check signalled status; call `SYNC_IOC_FILE_INFO` only for debugging or offline analysis, never on the latency-critical path.

---

#### 5.4 `drm_syncobj`

1. **Timeline point regression** — Signalling a timeline `drm_syncobj` with a sequence number equal to or less than the current highest signalled value has undefined semantics; on some drivers it silently succeeds while corrupting the timeline ordering seen by waiting threads. Detect with `DRM_IOCTL_SYNCOBJ_QUERY` before signalling to verify the current payload; Vulkan Validation Layers also catch non-monotonic timeline signals. Fix: always signal strictly increasing values; maintain a monotonically increasing counter in the application or driver and never decrement it.

2. **Binary syncobj use-after-reset race** — Resetting a binary `drm_syncobj` via `DRM_IOCTL_SYNCOBJ_RESET` while another thread is waiting on the same object via `DRM_IOCTL_SYNCOBJ_WAIT` is a race: after the reset the waiting thread will see the object as unsignalled and may block indefinitely if no subsequent signal occurs. Detect with thread sanitizer or by reproducing under high concurrency stress. Fix: use timeline semantics for any multi-producer scenario, reserving binary syncobj only for strictly single-producer single-consumer patterns where reset and wait are always sequenced by higher-level application logic.

3. **Leaking syncobj handles across `fork()`** — `drm_syncobj` handles are DRM object handles (integers), not file descriptors. Unlike file descriptors, they are not duplicated into the child process on `fork()`; the child inherits the open DRM fd but the handle table lives in the parent's DRM file descriptor context. Detect by auditing any code path that calls `fork()` while holding active syncobj handles and then uses those handles in the child. Fix: if cross-process sharing is needed, export to a file descriptor with `DRM_IOCTL_SYNCOBJ_HANDLE_TO_FD` before forking; the child receives the fd via the fork and can import it with `DRM_IOCTL_SYNCOBJ_FD_TO_HANDLE`.

---

#### 5.5 `IN_FENCE_FD` / `OUT_FENCE_PTR`

1. **Setting `IN_FENCE_FD` on a non-plane object** — `IN_FENCE_FD` is a plane-level DRM atomic property. Attempting to set it on a CRTC or connector object via `drmModeAtomicAddProperty()` returns `EINVAL`. A common copy-paste error when porting code from `OUT_FENCE_PTR` (which is a CRTC-level property) to `IN_FENCE_FD`. Detect by checking the DRM object type (`drmModeGetPlane()`, `drmModeGetCrtc()`) before adding the property, or by enabling `DRM_IOCTL_MODE_ATOMIC` error logging. Fix: always attach `IN_FENCE_FD` to the plane object id and `OUT_FENCE_PTR` to the CRTC object id.

2. **Not reading `OUT_FENCE_PTR` only after `drmModeAtomicCommit()` returns** — The kernel writes the new `sync_file` fd into the userspace pointer during `drmModeAtomicCommit()`; reading the pointed-to value before the call returns (e.g., from a different thread racing with the commit call) will read an uninitialised or stale value. Detect with thread sanitizer; the race is straightforward to observe. Fix: read `OUT_FENCE_PTR` value only in the thread that called `drmModeAtomicCommit()`, after it has returned 0.

3. **Ignoring `OUT_FENCE_PTR` and spinning on `DRM_IOCTL_WAIT_VBLANK` instead** — Compositors that poll `DRM_IOCTL_WAIT_VBLANK` for frame pacing instead of using `OUT_FENCE_PTR` introduce an extra round-trip through the kernel event queue and miss the earlier explicit-sync signal. `OUT_FENCE_PTR` signals as soon as the VBLANK counter advances, allowing the compositor to begin next-frame GPU work at the earliest possible time. Detect by comparing frame latency with and without `OUT_FENCE_PTR` via `perf`/`drm_vblank_event` tracepoints. Fix: use `OUT_FENCE_PTR` as the primary release fence; schedule next-frame GPU work as soon as it signals, not at the DRM event callback.

---

#### 5.6 `wp_linux_drm_syncobj_v1`

1. **Acquire point set but compositor does not support the protocol** — If the compositor does not advertise `wp_linux_drm_syncobj_manager_v1` as a Wayland global, the client's `wl_registry_bind()` for it returns a nil proxy. Sending protocol requests on a nil proxy causes a Wayland protocol error that disconnects the client immediately. Detect by checking for `NULL` after `wl_registry_bind()` before calling any methods on the object. Fix: probe for the global's presence in the registry listener before binding; implement a fallback code path using implicit synchronisation for compositors that do not support `wp_linux_drm_syncobj_v1`.

2. **Acquire and release points on the same timeline with inverted or equal seqno** — Setting the release point to a value less than or equal to the acquire point on the same timeline defeats the synchronisation: the compositor may signal the release before it has waited for the acquire, permitting the client to reuse a buffer that is still being composited. Detect by enabling Wayland protocol tracing (`WAYLAND_DEBUG=1`) and verifying that `set_release_point` seqno is always strictly greater than `set_acquire_point` seqno on the same timeline. Fix: ensure `release_seqno > acquire_seqno` on every surface commit using the same timeline object.

3. **Not signalling the release point** — If the client sets a release point but the GPU never signals the corresponding `drm_syncobj` timeline sequence number — for example because a Vulkan submission was never issued, or because a Vulkan validation error caused an early return — the compositor will block waiting for the release point indefinitely, freezing the surface. Detect by running Vulkan Validation Layers with synchronisation validation enabled (`VkValidationFeaturesEXT` with `VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT`) during development. Fix: always pair every `set_release_point()` with a `vkQueueSubmit()` that signals the corresponding timeline semaphore value, and handle Vulkan errors before issuing the Wayland commit.

---

#### 5.7 `VkFence`

1. **CPU-blocking `vkWaitForFences()` on the render thread** — Waiting for a fence on the same thread that records and submits the next frame serialises CPU and GPU work: the CPU stalls until the GPU finishes, then begins the next frame, leaving the GPU idle during CPU recording time. Detect by observing GPU idle bubbles in `renderdoc`, `gpa`, or `intel_gpu_top`. Fix: use a separate presentation thread for fence waiting; or poll with `vkWaitForFences(..., timeout=0)` and proceed with non-blocking CPU work (input handling, physics) while the GPU completes the prior frame.

2. **Forgetting `vkResetFences()` before reuse** — Submitting to `vkQueueSubmit()` with a fence already in the `VK_FENCE_STATUS_SIGNALED` state returns `VK_ERROR_OUT_OF_HOST_MEMORY` on some implementations or triggers a validation error on others. The Vulkan specification requires fences to be in the unsignalled state when passed to `vkQueueSubmit()`. Detect with Vulkan Validation Layers; the error is `VUID-vkQueueSubmit-fence-00064`. Fix: call `vkResetFences()` before resubmitting any fence that was previously waited on successfully.

3. **Exporting a `VkFence` as `SYNC_FD` more than once** — The `VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT` export is a one-shot operation; the fence payload moves to the `sync_file` fd and the `VkFence` object resets to unsignalled. Calling `vkGetFenceFdKHR(SYNC_FD)` a second time on the same fence yields an invalid fd because the fence is now in the unsignalled state (there is nothing to export). Detect by instrumenting `vkGetFenceFdKHR()` call sites and verifying that each signalled fence exports at most once with `SYNC_FD`. Fix: use `VK_EXTERNAL_FENCE_HANDLE_TYPE_OPAQUE_FD_BIT` if multiple consumers need access to the fence's completion state.

---

#### 5.8 `VkSemaphore` binary

1. **Signal-without-consume or consume-without-signal** — Submitting a binary semaphore to `VkSubmitInfo::pSignalSemaphores` twice without an intervening wait that consumes the signal, or waiting on a binary semaphore before it has been signalled by a prior submission, are both undefined behaviour per the Vulkan specification. On Linux with RADV or ANV, this manifests as GPU hangs or `VK_ERROR_DEVICE_LOST`. Detect with Vulkan Validation Layers synchronisation validation (`VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT`). Fix: use timeline semaphores for any pattern involving more than one producer or consumer; restrict binary semaphores to the WSI acquire/release boundary where exactly-once consumption is guaranteed by the swapchain API.

2. **Reusing a binary semaphore after `VK_ERROR_OUT_OF_DATE_KHR`** — If `vkAcquireNextImageKHR()` returns `VK_ERROR_OUT_OF_DATE_KHR`, the acquire semaphore may be in an undefined state (it may or may not have been signalled, depending on implementation). Reusing the same semaphore for the next swapchain acquire risks a validation error or GPU hang. Detect by auditing swapchain recreation code paths for semaphore reuse. Fix: destroy and recreate the binary semaphore (or retire it to a pool) on any swapchain acquire error before issuing a new acquire.

3. **Exporting binary semaphore as `SYNC_FD` and expecting it to remain signalled** — The `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT_KHR` export moves the semaphore payload into the `sync_file` fd and resets the `VkSemaphore` to unsignalled. Any code holding a reference to the semaphore handle and expecting it to remain signalled after export will observe it as unsignalled. Detect by searching for `vkGetSemaphoreFdKHR(SYNC_FD)` call sites and verifying post-export semaphore usage. Fix: if the semaphore state must remain observable after export (e.g., for multi-consumer scenarios), use `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT_KHR` instead.

---

#### 5.9 `VkSemaphore` timeline

1. **Non-monotonic signal values** — Submitting a timeline semaphore signal with a value equal to or less than the current payload is a Vulkan specification error (`VUID-VkSubmitInfo2-semaphore-03868`). Some drivers silently ignore it while others corrupt the timeline ordering; in either case frame ordering breaks. Detect with Vulkan Validation Layers; the validation error is reported at `vkQueueSubmit2()` call time. Fix: maintain a monotonically increasing 64-bit counter in the application; never decrement it; use `vkGetSemaphoreCounterValue()` to query the current value if needed.

2. **Using a timeline semaphore at the WSI swapchain boundary** — `vkAcquireNextImageKHR()` and `vkQueuePresentKHR()` accept only binary semaphores or `VkFence`; passing a timeline semaphore handle returns `VK_ERROR_FEATURE_NOT_PRESENT` on conformant implementations. Detect by reading the Vulkan specification requirement (`VUID-vkAcquireNextImageKHR-semaphore-03265`) and auditing call sites. Fix: use a binary semaphore at the WSI acquire/release boundary; use timeline semaphores exclusively for all internal GPU-GPU and GPU-CPU synchronisation.

3. **CPU wait for a timeline value that is never signalled** — `vkWaitSemaphores()` with a value that no GPU submission ever signals waits indefinitely (or until the `timeout` parameter expires). This is common when a compute shader early-exits on a path that does not write the output that triggers the semaphore signal, or when the submission carrying the signal is never issued due to an earlier error. Detect by setting a finite `timeout` and checking for `VK_TIMEOUT` return; treat it as an application bug if it triggers outside of controlled teardown. Fix: ensure every value waited upon is guaranteed to be signalled by some path through the application's submission logic.

---

#### 5.10 `VK_KHR_external_semaphore_fd`

1. **fd ownership confusion after `vkImportSemaphoreFdKHR()`** — After a successful call to `vkImportSemaphoreFdKHR()`, the Vulkan implementation takes ownership of the file descriptor; the application must not call `close()` on it afterward. Calling `close()` after a successful import double-closes the fd (once by the application, once by the Vulkan implementation when the semaphore is destroyed), which may close an unrelated fd that happened to be assigned the same number. Detect with `strace` for unexpected `EBADF` errors or with fd-tracking sanitisers. Fix: treat the fd as consumed immediately upon a successful `vkImportSemaphoreFdKHR()` call; only close it on import failure.

2. **Incorrect handle type for timeline semaphore export** — Passing `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT_KHR` to `vkGetSemaphoreFdKHR()` for a timeline semaphore returns `VK_ERROR_INVALID_EXTERNAL_HANDLE`; the `SYNC_FD` handle type is only valid for binary semaphores. Detect with Vulkan Validation Layers (`VUID-VkSemaphoreGetFdInfoKHR-handleType-01136`). Fix: always use `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT_KHR` for timeline semaphore export.

3. **Not checking `VkExternalSemaphoreProperties` before export** — Not every driver supports every handle type for every semaphore. Calling `vkGetSemaphoreFdKHR()` with an unsupported handle type returns `VK_ERROR_INVALID_EXTERNAL_HANDLE`. Detect by always querying `vkGetPhysicalDeviceExternalSemaphoreProperties()` before assuming a handle type is available. Fix: query external semaphore properties at device initialisation time; gate export/import code paths on the reported `externalSemaphoreFeatures` flags.

---

#### 5.11 `ntsync`

1. **Kernel version mismatch** — Attempting to use ntsync with a kernel older than Linux 6.10 results in `/dev/ntsync` not existing; Wine/Proton silently falls back to wineserver IPC, negating the performance benefit without any explicit error to the user. Detect by checking `PROTON_LOG=1` output for `"ntsync"` vs `"esync"` or `"server"` in the sync backend selection message; also verify `ls /dev/ntsync` exists and has correct permissions. Fix: the Wine/Proton runtime selection logic handles this automatically when using the Proton release; manual Wine users should verify the kernel version and set only the `PROTON_USE_NTSYNC=1` environment variable on supported kernels.

2. **ntsync and esync/fsync double-activation** — Setting both `PROTON_USE_NTSYNC=1` and `WINEESYNC=1` (or `WINEFSYNC=1`) simultaneously causes Wine to attempt to initialise two sync backends; the interaction is undefined and can produce mixed-mode object tables where some objects are ntsync fds and others are eventfds, leading to incorrect wait behaviour. Detect by running `PROTON_LOG=1` and observing multiple sync backend initialisation messages. Fix: set exactly one sync backend variable; the recommended practice is to rely on Proton's automatic detection, which selects ntsync when `/dev/ntsync` is available and falls back to esync or fsync otherwise.

3. **Abandoned mutex deadlock** — Win32 mutexes implement "abandoned mutex" semantics: if a thread holding a mutex exits without releasing it, the next waiter acquires the mutex and receives `WAIT_ABANDONED` status to indicate the protected data may be corrupt. The ntsync driver implements this via `NTSYNC_IOC_MUTEX_KILL`, which marks a mutex owner as dead. Wine-side code that does not check for `STATUS_ABANDONED_WAIT_0` in the ioctl return value will treat the mutex as cleanly acquired and may corrupt shared state. Detect by reading `dlls/ntdll/unix/sync.c` for the return value check; test with synthetic mutex abandonment scenarios. Fix: handle `STATUS_ABANDONED_WAIT_0` per Win32 documentation in all mutex wait paths.

---

#### 5.12 `esync` / `fsync`

1. **esync fd table exhaustion** — esync allocates one `eventfd` per Win32 synchronisation object. Games that create thousands of synchronisation objects (common in D3D12 titles with per-resource fences) exhaust the per-process fd limit (`ulimit -n`, default 1024 on many distributions). The failure is silent: Wine degrades to wineserver IPC for objects beyond the fd limit. Detect by running `cat /proc/<pid>/fd | wc -l` against the Wine process and comparing to `ulimit -n`. Fix: increase `ulimit -n` to 1048576 before launching Wine (`ulimit -n 1048576`); Proton does this automatically in its launch script.

2. **fsync futex starvation under high contention** — `sys_futex_waitv` does not guarantee scheduling fairness; under pathological contention (many threads waiting on the same futex word with rapid signal/reset cycles), some threads can be consistently de-scheduled in favour of others, causing effective starvation that manifests as game stutter or hangs. Detect with `perf sched latency` and observing threads with disproportionately high scheduling latency. Fix: prefer ntsync on Linux 6.10+ which routes waits through the kernel scheduler for fairness; use fsync only as a fallback on kernels between 5.16 and 6.9.

3. **fsync not available below Linux 5.16** — `sys_futex_waitv` was added in Linux 5.16. On older kernels, setting `WINEFSYNC=1` causes Wine to attempt the syscall, fail silently (the syscall number returns `ENOSYS`), and fall through to wineserver IPC without warning. Detect by running `strace -e futex_waitv wine ...` and observing `ENOSYS`; or by profiling CPU usage and noticing it resembles wineserver IPC (high `wineserver` CPU with the sync-heavy title). Fix: on kernels older than 5.16, use `WINEESYNC=1` instead; do not set `WINEFSYNC=1` on those kernels.

---

## Section 6: Cross-References to Canonical Chapter Sections

| Primitive | Primary chapter and section | Secondary references |
|---|---|---|
| `dma_fence` | Ch4 §"DMA-BUF Implicit and Explicit Fencing" | Ch8 §"GPU Scheduler and Fence Handling" |
| `dma_resv` | Ch4 §"DMA-BUF Implicit and Explicit Fencing" | Ch4 §"Implicit vs. Explicit Sync on DMA-BUF" |
| `sync_file` | Ch4 §"Explicit Sync as the Alternative" | Ch3 §"KMS Explicit Sync Properties" |
| `drm_syncobj` (binary) | Ch3 §"DRM Sync Objects and the Explicit Sync KMS Properties" | Ch4 §"Explicit Sync as the Alternative" |
| `drm_syncobj` (timeline) | Ch3 §"DRM Sync Objects and the Explicit Sync KMS Properties" | Ch24 §"Synchronisation: Fences, Binary Semaphores, Timeline Semaphores, Host Sync" |
| `IN_FENCE_FD` | Ch3 §"KMS Explicit Sync Properties" | Ch4 §"Explicit Sync as the Alternative" |
| `OUT_FENCE_PTR` | Ch3 §"KMS Explicit Sync Properties" | Ch4 §"Explicit Sync as the Alternative" |
| `wp_linux_drm_syncobj_v1` | Ch3 §"`wp_linux_drm_syncobj_v1`: the Wayland Protocol for Explicit Sync" | Ch20 §"Key Extension Protocols" |
| `VkFence` | Ch24 §"Synchronisation: Fences, Binary Semaphores, Timeline Semaphores, Host Sync" | — |
| `VkSemaphore` binary | Ch24 §"Synchronisation" | Appendix A §"binary semaphore" |
| `VkSemaphore` timeline | Ch24 §"Synchronisation" | Ch3 §"Explicit Sync Closes the Loop" |
| `VK_KHR_external_semaphore_fd` | Ch24 §"Synchronisation" | Ch3 §"NVIDIA's Explicit Sync Support"; Ch25 §"Vulkan External Semaphores + CUDA External Semaphores" |
| `ntsync` | Ch28 §"The Synchronisation Story: esync, fsync, and ntsync" | — |
| `esync` / `fsync` | Ch28 §"The Synchronisation Story: esync, fsync, and ntsync" | — |

---

## Integrations

This appendix consolidates material that is treated in depth in the following chapters:

- **Chapter 3** (DRM/KMS and Explicit Sync): the canonical treatment of `drm_syncobj` (binary and timeline), `IN_FENCE_FD`, `OUT_FENCE_PTR`, and `wp_linux_drm_syncobj_v1`. The conversion paths from Section 4 of this appendix are the practical companion to Chapter 3's architectural explanation of how the KMS atomic commit pipeline interacts with fence objects.

- **Chapter 4** (DMA-BUF and Buffer Sharing): the canonical treatment of `dma_fence`, `dma_resv`, and `sync_file`. The pitfall entries in Section 5 of this appendix on `dma_fence` callback management and `dma_resv` ww_mutex ordering expand on the correctness requirements stated in Chapter 4.

- **Chapter 8** (GPU Scheduler and Job Submission): `dma_fence` signalling from GPU hardware interrupt context; the scheduler's role in creating and signalling per-job fences.

- **Chapter 20** (Wayland Compositor Architecture): the `wp_linux_drm_syncobj_v1` protocol in the context of compositor surface management and the overall extension ecosystem.

- **Chapter 24** (Vulkan on Linux): the Vulkan synchronisation API (`VkFence`, `VkSemaphore` binary and timeline, `VK_KHR_external_semaphore_fd`, `VK_KHR_external_fence_fd`). Section 4 of this appendix's conversion matrix shows the complete set of paths between Vulkan sync objects and kernel sync objects.

- **Chapter 25** (CUDA–Vulkan Interop): `VK_KHR_external_semaphore_fd` in the CUDA external semaphore context; external memory/semaphore import/export for GPU-compute-to-GPU-graphics pipelines.

- **Chapter 28** (Wine and Proton): the Win32-compatibility synchronisation island (`ntsync`, `esync`, `fsync`); the architecture of DXVK and VKD3D-Proton's synchronisation bridge between Win32 and Vulkan.

- **Appendix A** (Glossary): definitions of one-shot fence, binary semaphore, timeline semaphore, reservation object, sync file, and related terms referenced in this appendix.

- **Appendix C** (Kernel and Mesa Version Matrix): the authoritative table of which kernel version introduced each primitive (ntsync in 6.10, `sys_futex_waitv` in 5.16, `drm_syncobj` timeline in 5.2, etc.) and the corresponding Mesa version that added Vulkan driver support.

---

## References

1. Linux kernel DMA-BUF documentation. `Documentation/driver-api/dma-buf.rst`. [https://docs.kernel.org/driver-api/dma-buf.html](https://docs.kernel.org/driver-api/dma-buf.html)

2. Linux kernel Sync File API Guide. `Documentation/driver-api/sync_file.rst`. [https://www.kernel.org/doc/html/latest/driver-api/sync_file.html](https://www.kernel.org/doc/html/latest/driver-api/sync_file.html)

3. Linux kernel NT synchronisation primitive driver documentation. `Documentation/userspace-api/ntsync.rst`. [https://docs.kernel.org/userspace-api/ntsync.html](https://docs.kernel.org/userspace-api/ntsync.html)

4. Linux kernel source: `include/linux/dma-fence.h`. [https://github.com/torvalds/linux/blob/master/include/linux/dma-fence.h](https://github.com/torvalds/linux/blob/master/include/linux/dma-fence.h)

5. Linux kernel source: `include/linux/dma-resv.h`. [https://github.com/torvalds/linux/blob/master/include/linux/dma-resv.h](https://github.com/torvalds/linux/blob/master/include/linux/dma-resv.h)

6. Linux kernel source: `drivers/dma-buf/sync_file.c`. [https://github.com/torvalds/linux/blob/master/drivers/dma-buf/sync_file.c](https://github.com/torvalds/linux/blob/master/drivers/dma-buf/sync_file.c)

7. Linux kernel source: `drivers/gpu/drm/drm_syncobj.c`. [https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/drm_syncobj.c)

8. Linux kernel source: `drivers/misc/ntsync.c`. [https://github.com/torvalds/linux/blob/master/drivers/misc/ntsync.c](https://github.com/torvalds/linux/blob/master/drivers/misc/ntsync.c)

9. Linux kernel source: `include/uapi/drm/drm.h` (DRM_IOCTL_SYNCOBJ_* definitions). [https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm.h](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm.h)

10. Linux kernel source: `include/uapi/linux/ntsync.h`. [https://github.com/torvalds/linux/blob/master/include/uapi/linux/ntsync.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/ntsync.h)

11. Linux kernel source: `include/uapi/linux/sync_file.h` (SYNC_IOC_MERGE, SYNC_IOC_FILE_INFO). [https://github.com/torvalds/linux/blob/master/include/uapi/linux/sync_file.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/sync_file.h)

12. Wayland protocol: `linux-drm-syncobj-v1.xml`. `wayland-protocols` staging, version 1. [https://wayland.app/protocols/linux-drm-syncobj-v1](https://wayland.app/protocols/linux-drm-syncobj-v1)

13. Vulkan specification: "Synchronization and Cache Control" chapter. Khronos Group. [https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization)

14. Vulkan extension: `VK_KHR_external_semaphore_fd`. Khronos Registry. [https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_external_semaphore_fd.html](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_external_semaphore_fd.html)

15. `vkGetSemaphoreFdKHR` specification page. Khronos Registry. [https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetSemaphoreFdKHR.html](https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetSemaphoreFdKHR.html)

16. `vkImportSemaphoreFdKHR` specification page. Khronos Registry. [https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkImportSemaphoreFdKHR.html](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkImportSemaphoreFdKHR.html)

17. Mesa source: `src/vulkan/runtime/vk_drm_syncobj.c` — Vulkan drm_syncobj backend. [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/runtime/vk_drm_syncobj.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/runtime/vk_drm_syncobj.c)

18. Mesa source: `src/vulkan/wsi/wsi_common.c` — WSI synchronisation and swapchain fence handling. [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/vulkan/wsi/wsi_common.c)

19. Phoronix: "Linux 6.10 To Merge NTSYNC Driver For Emulating Windows NT Synchronization Primitives" (2024). [https://www.phoronix.com/news/Linux-6.10-Merging-NTSYNC](https://www.phoronix.com/news/Linux-6.10-Merging-NTSYNC)

20. LWN.net: "drm: add syncobj timeline support v9" — timeline syncobj patchset discussion. [https://lwn.net/Articles/768998/](https://lwn.net/Articles/768998/)

21. DRM Memory Management documentation: §"DRM Sync Objects". [https://www.kernel.org/doc/html/v5.4/gpu/drm-mm.html](https://www.kernel.org/doc/html/v5.4/gpu/drm-mm.html)

22. Linux kernel DRM KMS documentation: §"Explicit fencing properties". `Documentation/gpu/drm-kms.rst`. [https://www.kernel.org/doc/html/latest/gpu/drm-kms.html](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
