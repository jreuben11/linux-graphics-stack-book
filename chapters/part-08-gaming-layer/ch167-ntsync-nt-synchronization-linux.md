# Chapter 167 — NTSYNC: NT Synchronization Primitives in the Linux Kernel

This chapter targets three audiences: **Linux gaming and Wine/Proton developers** who need to understand how ntsync affects game compatibility and performance; **kernel developers** interested in the design decisions behind a new misc character device that implements non-POSIX semantics; and **systems engineers** diagnosing Wine performance bottlenecks who want to understand the full history from wineserver round-trips through esync, fsync, and into the kernel-native solution.

---

## Table of Contents

1. [Introduction: The NT Synchronization Problem](#1-introduction-the-nt-synchronization-problem)
2. [esync and fsync: The Userspace Predecessors](#2-esync-and-fsync-the-userspace-predecessors)
3. [The ntsync Kernel Module](#3-the-ntsync-kernel-module)
4. [Wine Integration](#4-wine-integration)
5. [Proton Integration](#5-proton-integration)
6. [Performance Impact](#6-performance-impact)
7. [Kernel Configuration and Distro Support](#7-kernel-configuration-and-distro-support)
8. [Security Considerations](#8-security-considerations)
9. [Integrations](#9-integrations)

---

## 1. Introduction: The NT Synchronization Problem

Every Windows application that uses threads, mutexes, events, or semaphores ultimately calls into the Windows NT kernel through a family of `Nt*` system calls: `NtCreateSemaphore`, `NtCreateMutex`, `NtCreateEvent`, `NtWaitForSingleObject`, `NtWaitForMultipleObjects`, and their `Rtl*` wrapper equivalents. These APIs have been central to Windows programming since NT 3.1, and they carry semantics that do not map cleanly onto any single POSIX primitive.

The critical function is `NtWaitForMultipleObjects` (exposed to Win32 programmers as `WaitForMultipleObjects`). In its *wait-all* mode, this call blocks the calling thread until **all** of the specified handles are simultaneously signaled, and then **atomically acquires all of them in a single operation**. No other thread can observe a state in which some objects are acquired and others are not. This is a vectored, multi-typed, state-mutating atomic wait — a combination that has no direct analogue in POSIX.

When Wine runs a Windows application on Linux, it must implement the full NT synchronization object model. The traditional approach — still used when no better option is available — routes every synchronization call as an RPC through the *wineserver*, a dedicated single-threaded process that acts as the Windows kernel surrogate. This design is correct: the wineserver can maintain the necessary state and enforce atomicity. But as applications began using synchronization primitives more aggressively — modern D3D12 games may create tens of thousands of events and mutexes to coordinate GPU command queues — the overhead of a context switch into the wineserver and back for every `WaitForSingleObject` became a measurable bottleneck. For heavily multi-threaded games, the wineserver's single-threaded event loop became the limiting factor on frame rate. [Source](https://lwn.net/Articles/974344/)

The solution required moving synchronization into the kernel. The challenge was finding an approach that:

1. Correctly implements Windows NT object semantics (including wait-all atomicity and event types that have no POSIX equivalent).
2. Does not require every distribution to carry out-of-tree patches.
3. Is acceptable to upstream Linux kernel maintainers.

This chapter traces the journey from userspace workarounds to a merged kernel driver.

---

## 2. esync and fsync: The Userspace Predecessors

### 2.1 esync: eventfd-Based Synchronization

The first practical bypass of the wineserver bottleneck was **esync** (eventfd synchronization), developed by Zebediah Figura at CodeWeavers, inspired by earlier work by Daniel Santos on hybrid synchronization schemes. The esync patches were not upstreamed to Wine; they lived in Figura's personal `zfigura/wine` fork and were subsequently incorporated by Valve into the Proton wine fork. Esync represents each NT synchronization object as a Linux `eventfd` file descriptor, cached in the ntdll layer rather than in the wineserver. [Source](https://github.com/zfigura/wine/blob/master/README.esync)

The core insight is that `epoll` allows a single thread to wait on many file descriptors simultaneously without a round-trip to a separate process. When a Windows application calls `WaitForSingleObject`, esync can check the state and block directly on the eventfd from within the calling thread, entirely bypassing the wineserver RPC. Signaling an object is similarly a direct `write()` to the eventfd.

The mechanics for each object type:

- **Auto-reset events and semaphores**: Use `EFD_SEMAPHORE` flag. A `read()` atomically decrements the counter and blocks if it is zero.
- **Manual-reset events**: Poll for `POLLIN` without consuming the value; a separate `write(1)` sets the event, and a `write(0)` via a control path resets it.
- **Server-bound objects** (processes, threads, message queues): The wineserver creates the eventfd and signals it appropriately; ntdll retrieves the fd and can then wait on it alongside ntdll-owned objects.

Activating esync requires setting `WINEESYNC=1` in the environment. It was never merged into upstream Wine; it lived in forks and in Valve's Proton tree.

**File descriptor exhaustion.** Each synchronization object consumes one file descriptor. A modern AAA game that creates tens of thousands of mutexes and events can exhaust the default per-process limit of 4,096 file descriptors. Users had to raise this limit system-wide — typically by setting `DefaultLimitNOFILE=1048576` in systemd's configuration files — before esync could be used with such titles. [Source](https://github.com/zfigura/wine/blob/master/README.esync)

**The wait-all limitation.** Esync cannot implement true atomic wait-all semantics. The `epoll_wait`-based approach polls each object individually, verifies that all are signaled, and then attempts to acquire them one by one in a loop — if any acquisition fails between the check and the acquire, it restarts. This introduces the possibility of livelock and subtle ordering bugs. As the esync README acknowledges, "implementing wait-all ... is not exactly possible the way they'd like it to be possible." [Source](https://github.com/zfigura/wine/blob/master/README.esync)

### 2.2 fsync: Futex-Based Synchronization

**fsync** (futex synchronization) replaced the eventfd model with Linux futexes (fast user-space mutexes). Futexes store state in shared memory and call into the kernel only when a thread actually needs to block, resulting in fewer system calls for the uncontended case.

The key difficulty was implementing `WaitForMultipleObjects`. A single futex can only be waited on by one thread at a time, and there is no standard way to wait on multiple futexes simultaneously. The initial fsync implementation used out-of-tree kernel patches to add a `FUTEX_WAIT_MULTIPLE` operation — this was never accepted upstream, which meant fsync required a custom kernel, severely limiting its reach.

The situation partially improved when **`futex_waitv()`** was merged in Linux 5.16 as part of the futex2 system call family. [Source](https://www.collabora.com/news-and-blog/blog/2023/02/17/the-futex-waitv-syscall-gaming-on-linux/) This new syscall allows a process to wait on a list of up to 128 futexes simultaneously, waking when any one of them is signaled. Proton's Wine gained support for this through the `WINEFSYNC_FUTEX_WAITV` environment variable, enabling a lighter-weight multi-wait path without out-of-tree patches.

However, futex_waitv() still does not solve the wait-all atomicity problem. It provides *wait-any* semantics: wake when any futex fires. Implementing wait-all still requires the retry loop, with all the same race conditions as esync. Furthermore, fsync stores object state in shared memory that any cooperating process can read and write — the kernel enforces no invariants on that memory.

### 2.3 The Out-of-Tree Patchwork Problem

For several years, Linux gamers who wanted Wine performance beyond what the wineserver provided had to install custom kernel modules or patched Wine builds. The ecosystem fragmented into:

- **wine-staging**: Community patchset including esync support.
- **Frogging-Family/wine-tkg-git**: Build system for Wine with esync, fsync, and ntsync patches.
- **Proton**: Valve's Wine fork shipping esync since circa 2018, fsync since 2019, and ntsync patches beginning 2024.
- **GE-Proton (GloriousEggroll)**: Community Proton fork with aggressive inclusion of new patches.

Each of these required users to use specific kernel-module packages (for fsync's out-of-tree futex patches) or accept that certain synchronization operations would not be atomic. Maintainers of these trees spent significant effort rebasing the synchronization patches across kernel and Wine version updates. The upstreaming of ntsync into both the Linux kernel (6.14) and Wine (11.0) ends this fragmentation.

| Feature | wineserver RPC | esync | fsync (futex_waitv) | ntsync |
|---|---|---|---|---|
| Wait-any | Yes (slow) | Yes (fast) | Yes (fast) | Yes (fast) |
| Wait-all (atomic) | Yes (correct) | No (retry loop) | No (retry loop) | Yes (correct) |
| File descriptor per object | No | Yes | No | Yes (object fd) |
| Requires custom kernel | No | No | No (post-5.16) | No (post-6.14) |
| Upstream Wine | Yes | No | No | Yes (11.0+) |
| NtPulseEvent correctness | Yes | No | No | Yes |
| Abandoned mutex detection | Yes | No | No | Yes |

---

## 3. The ntsync Kernel Module

### 3.1 Design Goals and Development History

The ntsync driver was designed and implemented by Elizabeth Figura (Zebediah Figura) of CodeWeavers, the same engineer who built esync and fsync. The fundamental goal is stated in the patch series cover letter: "NT synchronization primitives are unique in that the wait functions both are vectored, operate on multiple types of object with different behaviour (mutex, semaphore, event), and affect the state of the objects they wait on. … Certain operations, such as `NtPulseEvent()` or the 'wait-for-all' mode in `NtWaitForMultipleObjects()`, require direct control over the underlying wait queue that cannot be implemented in user space." [Source](https://lkml.iu.edu/hypermail/linux/kernel/2412.1/02004.html)

The earliest public proposal, titled "Kernel interface for Wine synchronization primitives," appeared on the wine-devel mailing list in January 2021. The in-kernel module went through several name iterations in community discussion — "winesync," "fastsync," and finally "ntsync" — before the current name was settled on. After extensive review on the Linux kernel mailing lists spanning multiple years and patch revision series (RFC v1 through v7), the driver was presented at the Linux Plumbers Conference in 2023, submitted by Greg Kroah-Hartman and accepted by Linus Torvalds for the Linux **6.14** merge window, which opened in January 2025. [Source](https://www.gamingonlinux.com/2025/01/ntsync-for-proton-wine-now-in-linux-kernel-6-14-that-should-make-many-steamos-users-happy/)

An important distinction: out-of-tree community kernel patches from the Frogging-Family and CachyOS projects shipped a `winesync` module for kernels 6.6–6.13. This module has a different API from the upstream ntsync driver. Users who installed these patches and then upgraded to a 6.14 kernel needed to switch from the DKMS module to the built-in driver and update to Wine 11 or GE-Proton 10.9+. [Source](https://github.com/Frogging-Family/wine-tkg-git/issues/936)

**Note on Linux 6.10.** An earlier, partial ntsync implementation was merged in Linux 6.10, but it was intentionally marked `depends on BROKEN` in Kconfig because not all of the object types and wait paths had been wired up. Patch 28 of the v5 series is titled "ntsync: No longer depend on BROKEN" — this removal was part of what landed in 6.14. [Source](https://www.mail-archive.com/linux-kselftest@vger.kernel.org/msg12778.html)

### 3.2 Device Model and Instance Isolation

The driver exposes a single character device at `/dev/ntsync`. Crucially, **each `open()` call on the device returns an independent instance**. All NT synchronization objects created on one instance are invisible to any other instance; they can only be used together with other objects from the same open file description. This isolation model means each Wine prefix (each emulated Windows virtual machine) opens its own fd to the device and creates all of its objects through that fd.

The kernel represents this with the `ntsync_device` structure:

```c
/* drivers/misc/ntsync.c, Linux 6.14 */
struct ntsync_device {
    struct file *file;
    /* Protects all-wait operations across objects in this instance */
    struct mutex wait_all_lock;
};
```

Individual synchronization objects are represented by `ntsync_obj`, which contains a type field, a union for type-specific state, per-object spinlock, waiter lists, and an `all_hint` atomic for optimizing the common case where an object is not involved in any current wait-all operation.

### 3.3 Object Types

The driver implements three NT synchronization primitive types:

**Semaphore (`NTSYNC_TYPE_SEM`)**: A counting semaphore with a volatile 32-bit `count` and a static 32-bit `max`. The semaphore is signaled when `count > 0`. Acquisition decrements `count` by one. The ioctl `NTSYNC_IOC_SEM_POST` increments `count` by a specified amount (equivalent to `NtReleaseSemaphore`), returning `EOVERFLOW` if the result would exceed `max`.

**Mutex (`NTSYNC_TYPE_MUTEX`)**: Tracks a 32-bit `owner` (thread ID) and a 32-bit recursion `count`. The mutex is signaled when `owner == 0`. Acquisition sets `owner` to the calling thread's ID and increments `count`; the same thread can recursively acquire the mutex by incrementing `count` again. If the owning thread exits without releasing the mutex, `NTSYNC_IOC_KILL_OWNER` marks it as abandoned; subsequent acquisition returns `EOWNERDEAD` (analogous to Windows' `WAIT_ABANDONED`).

**Event (`NTSYNC_TYPE_EVENT`)**: Binary state with a `signaled` flag and a `manual` flag. When `manual == 0` (auto-reset), the event is de-signaled upon acquisition. When `manual == 1` (manual-reset), the event remains signaled until explicitly reset with `NTSYNC_IOC_RESET_EVENT`.

### 3.4 The UAPI Header

The complete user-space interface is defined in `include/uapi/linux/ntsync.h` in the kernel source tree. The key structures are:

```c
/* include/uapi/linux/ntsync.h, Linux 6.14 */
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
/* Copyright (C) 2021-2022 Elizabeth Figura */

struct ntsync_sem_args {
    __u32 count;
    __u32 max;
};

struct ntsync_mutex_args {
    __u32 owner;
    __u32 count;
};

struct ntsync_event_args {
    __u32 signaled;
    __u32 manual;
};

struct ntsync_wait_args {
    __u64 timeout;   /* nanoseconds; U64_MAX for infinite */
    __u64 objs;      /* pointer to array of __u32 object fds */
    __u32 count;     /* number of objects */
    __u32 owner;     /* thread ID, for mutex acquisition */
    __u32 index;     /* output: index of acquired object (WAIT_ANY) */
    __u32 alert;     /* optional alertable event object fd */
    __u32 flags;     /* NTSYNC_WAIT_REALTIME = 0x1 */
    __u32 pad;
};

#define NTSYNC_MAX_WAIT_COUNT 64

/* ioctl magic byte 'N' */
#define NTSYNC_IOC_CREATE_SEM    _IOWR('N', 0x80, struct ntsync_sem_args)
#define NTSYNC_IOC_WAIT_ANY      _IOWR('N', 0x82, struct ntsync_wait_args)
#define NTSYNC_IOC_WAIT_ALL      _IOWR('N', 0x83, struct ntsync_sem_args)
#define NTSYNC_IOC_CREATE_MUTEX  _IOWR('N', 0x84, struct ntsync_mutex_args)
#define NTSYNC_IOC_MUTEX_UNLOCK  _IOWR('N', 0x85, struct ntsync_mutex_args)
#define NTSYNC_IOC_KILL_OWNER    _IOW ('N', 0x86, __u32)
#define NTSYNC_IOC_CREATE_EVENT  _IOWR('N', 0x87, struct ntsync_event_args)
#define NTSYNC_IOC_SET_EVENT     _IOWR('N', 0x88, struct ntsync_event_args)
#define NTSYNC_IOC_RESET_EVENT   _IOWR('N', 0x89, struct ntsync_event_args)
#define NTSYNC_IOC_PULSE_EVENT   _IOWR('N', 0x8a, struct ntsync_event_args)
#define NTSYNC_IOC_READ_SEM      _IOR ('N', 0x8b, struct ntsync_sem_args)
#define NTSYNC_IOC_READ_MUTEX    _IOR ('N', 0x8c, struct ntsync_mutex_args)
#define NTSYNC_IOC_READ_EVENT    _IOR ('N', 0x8d, struct ntsync_event_args)
```

[Source](https://docs.kernel.org/userspace-api/ntsync.html)

### 3.5 Creating and Using Synchronization Objects

Each object is created via an ioctl on the device fd. The ioctl returns a new file descriptor representing the object. All subsequent operations on that object use that per-object fd:

```c
/* Example: creating and waiting on an ntsync event — userspace code */
#include <sys/ioctl.h>
#include <fcntl.h>
#include <linux/ntsync.h>

/* Open a device instance — one per Wine prefix */
int dev_fd = open("/dev/ntsync", O_RDWR);

/* Create an auto-reset event, initially unsignaled */
struct ntsync_event_args event_args = { .signaled = 0, .manual = 0 };
int event_fd = ioctl(dev_fd, NTSYNC_IOC_CREATE_EVENT, &event_args);

/* Signal the event from another thread */
ioctl(event_fd, NTSYNC_IOC_SET_EVENT, &event_args);

/* Wait on the event with a 1-second timeout */
struct ntsync_wait_args wait = {
    .timeout = 1000000000ULL,   /* 1 second in nanoseconds */
    .objs    = (uintptr_t)&event_fd,
    .count   = 1,
    .owner   = 0,               /* not waiting on a mutex */
    .index   = 0,               /* output */
    .alert   = 0,               /* no alertable event */
    .flags   = 0,
};
int ret = ioctl(dev_fd, NTSYNC_IOC_WAIT_ANY, &wait);
/* On success, wait.index is the index (0) of the acquired object */
```

Object file descriptors are reference-counted and follow normal POSIX fd lifetime rules: they are released on `close()` and automatically cleaned up when the process exits or is killed.

### 3.6 NTSYNC_IOC_WAIT_ANY and NTSYNC_IOC_WAIT_ALL

These are the two wait ioctls, and they represent the core value of the driver.

**`NTSYNC_IOC_WAIT_ANY`**: Acquires at most one object from the provided list. The kernel queues the calling thread on the waiter list of each object, then checks if any object is already signaled. If one is found, it is acquired immediately and the thread is returned its index. If none is signaled, the thread sleeps. When an object is signaled by another thread, the kernel's wakeup path scans the waiter list for `WAIT_ANY` waiters, acquires the object for the first eligible waiter, and wakes that thread. Only one thread can acquire the object per signal. The `index` field of `ntsync_wait_args` is written with the index of the acquired object on success, or `count` if the optional `alert` event fired. [Source](https://docs.kernel.org/userspace-api/ntsync.html)

**`NTSYNC_IOC_WAIT_ALL`**: This is the capability that no userspace approach could provide correctly. The kernel must acquire **all** objects in the list simultaneously. The challenge is concurrency: if the kernel were to acquire them one by one, another thread could acquire object 2 between the kernel's acquisition of objects 1 and 3, violating atomicity.

The solution uses the `wait_all_lock` device-wide mutex. When an object is involved in any ongoing wait-all operation, it is marked with an `all_hint` counter. For any operation that must acquire multiple objects atomically, the `wait_all_lock` is taken first, then all per-object spinlocks. This total order prevents deadlocks. The `try_wake_all()` function:

1. Acquires `wait_all_lock`.
2. Locks all objects in the wait list.
3. Checks that all are simultaneously signaled.
4. Uses `atomic_try_cmpxchg()` to atomically mark the wait as satisfied.
5. Decrements counters / updates state for all objects.
6. Unlocks all objects and the device mutex.
7. Wakes the waiting thread.

[Source](https://www.mail-archive.com/linux-doc@vger.kernel.org/msg38850.html)

The implementation deliberately optimizes for the common case. Wait-all is known to be rare in practice: the design pays higher overhead for wait-all (device mutex) to keep wait-any extremely fast (per-object spinlock only, if `all_hint == 0`). [Source](https://www.mail-archive.com/linux-doc@vger.kernel.org/msg38850.html)

**`NtPulseEvent` correctness.** The Windows `NtPulseEvent` call signals an event and then immediately resets it, but *only wakes threads that are already waiting at the moment of the pulse*. This requires direct queue inspection — the kernel must walk the waiter list at signal time and wake exactly the threads currently sleeping, then reset the state before any new waiter can observe the signaled state. The wineserver could implement this because it controls the queue. Userspace esync and fsync could not, because by the time a `write()` reaches an eventfd or futex, there is no way to atomically inspect-and-wake the existing waiters and reset the state. The ntsync driver's `NTSYNC_IOC_PULSE_EVENT` implements this correctly by holding the per-object spinlock across the entire wake-and-reset sequence.

### 3.7 Error Semantics

The kernel documentation defines a precise set of error codes that mirror Windows behavior:

| errno | Meaning |
|---|---|
| `ETIMEDOUT` | Timeout expired without acquiring any object |
| `EINTR` | Signal received while sleeping (Wine must restart the wait) |
| `EOWNERDEAD` | Mutex acquired but previous owner died (abandoned mutex) |
| `EOVERFLOW` | `NTSYNC_IOC_SEM_POST` would exceed the maximum count |
| `EPERM` | `NTSYNC_IOC_MUTEX_UNLOCK` called by non-owner thread |
| `EINVAL` | Invalid parameters (e.g., count > `NTSYNC_MAX_WAIT_COUNT`) |

[Source](https://docs.kernel.org/userspace-api/ntsync.html)

---

### 3.8 Kernel Selftests

The ntsync driver ships with an extensive test suite under `tools/testing/selftests/drivers/ntsync/`. These tests are intended to be more accessible to kernel developers than Wine's own synchronization test suite, and they specifically test edge cases that Wine may not exercise. The test structure mirrors the object types: `ntsync.c` contains individual test functions for semaphore state (`test_sem_state`), mutex state including abandoned-mutex detection (`test_mutex_state`), manual- and auto-reset event state, and dedicated tests for `NTSYNC_IOC_WAIT_ANY` and `NTSYNC_IOC_WAIT_ALL` including race conditions involving simultaneous signaling. [Source](https://www.mail-archive.com/linux-doc@vger.kernel.org/msg38856.html)

Running the selftests is straightforward on any kernel 6.14+ system with `/dev/ntsync` accessible:

```bash
cd tools/testing/selftests/drivers/ntsync
make
sudo ./ntsync
```

The tests are designed to run without root — provided `/dev/ntsync` has appropriate permissions — and cover both correctness properties (correct state transitions under single-threaded use) and concurrency properties (correct wakeup under simultaneous signaling from multiple threads).

---

## 4. Wine Integration

### 4.1 The Wineserver Architecture and the Bypass

**The wineserver round-trip cost.** To understand why ntsync matters, it helps to measure the cost it eliminates. A single wineserver synchronization round-trip involves:

1. The client (ntdll) serializes a request into a fixed-size request buffer.
2. The client sends the request header over a Unix domain socket to the wineserver.
3. The wineserver's `select()` loop wakes up, receives the request, and processes it.
4. The wineserver sends a reply back over the socket.
5. The client receives the reply and continues.

This is two context switches (client to kernel, kernel to wineserver, wineserver to kernel, kernel to client) and two socket operations per synchronization call. For a game that issues several hundred `WaitForSingleObject` calls per frame at 60 Hz, this overhead accumulates to tens of thousands of unnecessary context switches per second. For games like Dirt 3, whose engine creates a thread pool that hammers synchronization primitives between frames to dispatch work, the wineserver becomes the single-threaded bottleneck that limits the entire application to the wineserver's scheduling granularity.

Wine's synchronization architecture has three tiers, selected at initialization:

1. **Wineserver RPC** — always available, always correct, always slow. Every `NtWaitForSingleObject` call requires at least one round-trip.
2. **esync / fsync** — out-of-tree (not in upstream Wine before Wine 11), faster for uncontended paths, not atomically correct for wait-all.
3. **ntsync** — kernel-native, correct, fast; requires Linux 6.14+ with `CONFIG_NTSYNC`. Upstream in Wine 11.0.

The selection logic lives in `dlls/ntdll/unix/sync.c` in the Wine source tree. [Source](https://github.com/wine-mirror/wine/blob/master/dlls/ntdll/unix/sync.c) When ntsync is compiled in (guarded by `#ifdef NTSYNC_IOC_EVENT_READ`, i.e., the kernel header being present), Wine opens `/dev/ntsync` at ntdll startup. If the open succeeds, ntsync is used; otherwise, Wine falls back to the wineserver path.

Wine 10.15 (October 2025) introduced the first upstream Wine patches supporting ntsync. [Source](https://www.phoronix.com/news/Wine-10.15-With-NTSYNC) Wine 10.16 completed fast synchronization support. [Source](https://gamingonlinux.com/2025/10/wine-10-16-released-with-fast-synchronization-support-using-ntsync/) Wine 11.0, released on January 13, 2026, shipped ntsync as a supported feature for any system running Linux 6.14 or later. [Source](https://www.winehq.org/news/2026011301)

### 4.2 The inproc_sync Architecture

The ntsync integration in Wine is built around the concept of **in-process synchronization** (`inproc_sync`). An `inproc_sync` is a kernel-backed object that replaces the wineserver handle for a synchronization primitive:

```c
/* dlls/ntdll/unix/sync.c, Wine 11 */
struct inproc_sync {
    LONG refcount;
    int fd;              /* per-object fd from NTSYNC_IOC_CREATE_* */
    unsigned int access;
    unsigned short type;
    unsigned short closed;
};
```

For purely in-process objects (mutexes, semaphores, events created by the application), Wine creates the ntsync object directly and never contacts the wineserver. For server-bound objects — thread handles, process handles, message queue events — the wineserver creates the ntsync object on the kernel side and hands the file descriptor back to ntdll via a server request, so both the wineserver and ntdll can observe the same kernel object. [Source](https://gitlab.winehq.org/wine/wine/-/merge_requests/8445)

The core wait function that ties this together is `linux_wait_objs()`:

```c
/* Simplified from dlls/ntdll/unix/sync.c */
static NTSTATUS linux_wait_objs(int *fds, int count, BOOL wait_all,
                                ULONGLONG timeout, int alert_fd)
{
    struct ntsync_wait_args args = {
        .timeout = timeout,
        .objs    = (ULONG_PTR)fds,
        .count   = count,
        .owner   = GetCurrentThreadId(),
        .flags   = 0,
        .alert   = alert_fd >= 0 ? alert_fd : 0,
    };

    int ret = ioctl(ntsync_dev_fd,
                    wait_all ? NTSYNC_IOC_WAIT_ALL : NTSYNC_IOC_WAIT_ANY,
                    &args);
    if (ret == 0)   return STATUS_SUCCESS;   /* or STATUS_ABANDONED_WAIT_n */
    if (errno == ETIMEDOUT) return STATUS_TIMEOUT;
    if (errno == EINTR)     /* restart or return STATUS_USER_APC */;
    /* ... */
}
```

### 4.3 Environment Variables and Fallback Chain

Wine's synchronization backends are controlled by environment variables when using community-patched builds (esync and fsync were never upstream). With ntsync now upstream in Wine 11, ntsync is auto-detected; the legacy variables still affect older paths:

| Variable | Effect |
|---|---|
| `WINEESYNC=1` | Enable esync (eventfd) — requires Esync-patched Wine |
| `WINEFSYNC=1` | Enable fsync (futex shared memory) — requires Fsync-patched Wine |
| `WINEFSYNC_FUTEX_WAITV=1` | Use `futex_waitv()` for multi-object waits within fsync |
| *(no variable)* | Wine 11+ auto-detects `/dev/ntsync`; falls back to wineserver if absent |

For Wine 11 on Linux 6.14+, no environment variable is needed. The driver is detected at startup by attempting to open `/dev/ntsync`.

---

## 5. Proton Integration

### 5.1 GE-Proton and Community Builds

**GE-Proton** (GloriousEggroll's Proton fork) was the first widely-used Proton variant to ship ntsync support. GE-Proton 10-9 (July 2025) added ntsync alongside FSR 4 upgrade support. [Source](https://www.gamingonlinux.com/2025/07/ge-proton-10-9-released-with-ntsync-and-fsr4-upgrade-support/) **GE-Proton 10-10** made ntsync the default on supported systems — it is enabled automatically when `CONFIG_NTSYNC` is present in the running kernel. The release notes explain: "NTSYNC is now enabled by default and will be used if the kernel supports it." [Source](https://www.patreon.com/posts/ge-proton10-10-134462589)

For GE-Proton prior to 10-10, and for any Proton build that does not auto-enable ntsync, users can force the backend:

```bash
# Steam launch options for a game using ntsync
PROTON_USE_NTSYNC=1 %command%
```

This environment variable instructs the Wine layer inside Proton to prefer the ntsync backend over fsync or esync.

### 5.2 Valve's Proton Experimental

Valve's official **Proton Experimental** branch added Wine-side ntsync support aligned with the Wine 10.x development cycle. Unlike GE-Proton, Proton Experimental historically enabled fsync by default. Users wanting ntsync with official Proton should verify the `PROTON_USE_NTSYNC=1` launch option is set until upstream Proton adopts ntsync as its default. **Note: needs verification** for the exact Proton Experimental version that first shipped ntsync support — Valve's internal patch tracking is not always publicly documented.

### 5.3 The Steam Deck Kernel and ntsync

The Steam Deck ships with Valve's custom SteamOS kernel. At the time Linux 6.14 merged ntsync (January 2025), the Steam Deck was running kernel 6.5. SteamOS's kernel does not immediately track mainline releases; Valve selects kernel versions based on stability, driver compatibility, and hardware support requirements.

The GamingOnLinux reporting at the time of the 6.14 merge noted that Valve "developed ntsync as a general solution acceptable in upstream Wine, but there's no urgency in including it in the Deck / SteamOS kernel" immediately. [Source](https://www.gamingonlinux.com/2025/01/ntsync-for-proton-wine-now-in-linux-kernel-6-14-that-should-make-many-steamos-users-happy/) Steam Deck users should check whether their current SteamOS kernel includes `CONFIG_NTSYNC` by running:

```bash
ls /dev/ntsync
```

If the device node is absent, ntsync is not available and Proton will use its fsync fallback.

---

## 6. Performance Impact

### 6.1 Measured Performance Gains

Elizabeth Figura's patch series cover letter for v6 (submitted December 2024) includes benchmark data comparing upstream vanilla Wine (wineserver RPC only) against Wine with ntsync enabled on Linux 6.14: [Source](https://lkml.iu.edu/hypermail/linux/kernel/2412.1/02004.html)

| Game | Vanilla Wine (FPS) | Wine + ntsync (FPS) | Gain |
|---|---|---|---|
| Dirt 3 | 110.6 | 860.7 | +678% |
| Resident Evil 2 | 26 | 77 | +196% |
| Call of Juarez | 99.8 | 224.1 | +124% |
| Tiny Tina's Wonderlands | 130 | 360 | +177% |
| Anger Foot | — | — | +43% |

The cover letter explicitly notes: "The gain in performance varies wildly depending on the application in question and the user's hardware. For some games NT synchronization is not a bottleneck and no change can be observed." [Source](https://lkml.iu.edu/hypermail/linux/kernel/2412.1/02004.html)

**Important caveat**: These benchmarks compare vanilla Wine (wineserver only) against Wine with ntsync. Users coming from esync or fsync — the common case in Proton gaming prior to Wine 11 — will not see gains of this magnitude. The improvement over fsync is more moderate and primarily manifests as reduced frame time variance, lower input latency, and elimination of stutters rather than large average-FPS increases.

### 6.2 Why the Gains Are Title-Dependent

The performance benefit is proportional to the number of synchronization operations per frame. For a single-threaded game that never calls `WaitForMultipleObjects`, ntsync provides essentially no benefit. For a heavily multi-threaded D3D12 game that uses NT events to synchronize CPU-side work submission with GPU execution — coordinating dozens of worker threads via semaphores and events — every frame may involve hundreds or thousands of wait operations. The per-operation overhead of wineserver RPC (two context switches, a socket send, a socket recv) multiplied by thousands of operations per frame produces the catastrophic frame rates seen in the vanilla Wine column above. [Source](https://lwn.net/Articles/974344/)

### 6.2.1 Comparing Against Proton + fsync Baselines

The table above compares vanilla Wine (wineserver only) against ntsync. Users of Proton, which shipped fsync support since Proton 4.11-13 (2019), start from a higher baseline. For such users the gains are more modest:

- Proton with fsync: already eliminates most wineserver round-trips for simple single-object waits.
- Proton with ntsync over fsync: eliminates the remaining wineserver calls for server-bound objects, removes the retry-loop overhead for wait-all, and fixes `NtPulseEvent` correctness. Typical improvements over fsync are in the 10–40% range for frame rates, with greater benefits in frame time consistency.

The Debian ntsync howto observes that improvements "are most modest" for GPU-bottlenecked games and "most significant" for CPU-bound, multi-threaded titles. [Source](https://wiki.debian.org/Wine/NtsyncHowto)

### 6.3 Frame Time and Stuttering

Even where average FPS gains are modest, ntsync's correctness improvements eliminate an important category of stutter. Under esync's retry-loop implementation of wait-all, a race condition can cause a thread to spin for multiple milliseconds before all objects are successfully acquired. This appears in frame-time graphs as irregular spikes that the average-FPS metric obscures. With ntsync's atomic wait-all, these spikes disappear. For competitive games where input latency in the 1–4 ms range matters, this can be a more significant improvement than the FPS numbers suggest. The Debian Wiki notes that "performance improvements typically range from 40–200% in FPS gains depending on workload" and that the technology "is most beneficial in multi-threaded, CPU-bound scenarios." [Source](https://wiki.debian.org/Wine/NtsyncHowto)

---

## 7. Kernel Configuration and Distro Support

### 7.1 CONFIG_NTSYNC

The driver is controlled by the `CONFIG_NTSYNC` Kconfig option:

```
# drivers/misc/Kconfig
config NTSYNC
    tristate "NT synchronization primitive emulation"
    help
      This module implements the NT synchronization primitive driver,
      used to emulate Windows NT synchronization primitives in Wine
      and other NT emulators. It provides a character device at
      /dev/ntsync.
```

[Source](https://cateee.net/lkddb/web-lkddb/NTSYNC.html)

The driver can be built as a loadable module (`CONFIG_NTSYNC=m`) or compiled into the kernel (`CONFIG_NTSYNC=y`). When built as a module, it can be loaded on demand or persistently configured via `/etc/modules-load.d/`. To load it as a module automatically at boot:

```bash
# /etc/modules-load.d/ntsync.conf
ntsync
```

```bash
# Verify the module is loaded and device node exists
lsmod | grep ntsync
ls -la /dev/ntsync
```

### 7.2 Distribution Support

**Fedora**: Fedora 42 (released spring 2025, based on kernel 6.14) includes `CONFIG_NTSYNC` enabled by default. The Fedora project initially proposed automatically loading the module system-wide. [Source](https://fedoraproject.org/wiki/Changes/NTSYNC)

**Arch Linux**: Arch's `linux` and `linux-zen` kernels for versions 6.14 and newer include ntsync. The Arch wiki notes that users need kernel 6.14 or newer and should check with `ls /dev/ntsync`. [Source](https://bbs.archlinux.org/viewtopic.php?id=298994)

**Ubuntu**: Ubuntu 25.04 (Plucky Puffin, April 2025) shipped kernel 6.14 and includes ntsync support.

**Debian**: Debian's ntsync support arrived with kernel 6.14.5-1~exp1 in the experimental branch, then progressed to Forky (Debian 14) and Sid. For Debian Trixie (which uses kernel 6.12), a DKMS backport package is available. [Source](https://wiki.debian.org/Wine/NtsyncHowto)

**Linux Mint**: ntsync became available in Linux Mint 22 with the kernel 6.14 upgrade. [Source](https://forums.linuxmint.com/viewtopic.php?t=449946)

### 7.3 Checking Availability

To confirm ntsync is available on a running system:

```bash
# Check for the device node
ls /dev/ntsync && echo "ntsync available" || echo "ntsync not available"

# Check kernel config (if kernel headers are installed)
zcat /proc/config.gz 2>/dev/null | grep CONFIG_NTSYNC
# or
grep CONFIG_NTSYNC /boot/config-$(uname -r) 2>/dev/null

# Verify Wine is using ntsync (Wine 11+ on Linux 6.14+)
# Run a Wine application and check open fds
wine notepad.exe &
lsof /dev/ntsync
```

If `/dev/ntsync` exists but is not writable by the current user, the permissions issue described in Section 8 applies.

---

## 8. Security Considerations

### 8.1 Device Access Permissions

The `/dev/ntsync` character device was initially created with mode `0600` (readable and writable by root only), making it inaccessible to regular users without additional configuration. This was identified as a practical barrier immediately after the 6.14 merge.

A patch proposing `0666` permissions was submitted to the kernel list by Mike Lothian. Elizabeth Figura supported the change, arguing that "this is not a hardware device in any sense, it's a software driver that only lives in a char device" and that the only realistic risk is a bug in the ntsync driver itself, not privilege escalation via the device. Kernel developer Darrick J. Wong advocated for conservative defaults (strict kernel permissions, relaxed at runtime via udev rules) following the pattern of `/dev/loop-control`. [Source](https://www.phoronix.com/news/Linux-NTSYNC-Permissions-Issue)

The practical resolution for distributions is a udev rule:

```
# /etc/udev/rules.d/50-ntsync.rules
KERNEL=="ntsync", MODE="0666"
```

Distributions shipping Proton and Wine configurations typically include such a rule. Users can also add themselves to a group that owns the device, though as of Linux 6.14 the most common approach is the `0666` udev rule.

### 8.2 Instance Isolation and Object Lifetime

The isolation model is by design robust: objects created by one open file description cannot be passed to ioctls on a different device fd. The kernel enforces this by embedding the device pointer in each object and checking it on every ioctl. An application that maliciously or accidentally passes a cross-instance fd will receive `EINVAL` rather than accessing another process's synchronization state. [Source](https://docs.kernel.org/userspace-api/ntsync.html)

Object lifetime is governed by standard POSIX file descriptor semantics. Each object fd is reference-counted in the kernel. When the last reference to an object fd is closed — whether by explicit `close()` or by process termination or killing — the kernel decrements the reference count and frees the object. Any threads currently sleeping in `NTSYNC_IOC_WAIT_ANY` or `NTSYNC_IOC_WAIT_ALL` on a now-freed object will wake with `EINVAL`. The Wine layer handles this in its cleanup path for process and thread exit.

### 8.2.1 Flatpak and Container Access

An issue has been raised in the Flatpak project requesting explicit permission control for `/dev/ntsync` access from within Flatpak sandboxes. [Source](https://github.com/flatpak/flatpak/issues/6199) The concern is that a Flatpak-sandboxed Wine application needs access to `/dev/ntsync` to use ntsync, but the Flatpak permission model for character devices requires explicit declaration. Users running Steam via Flatpak (a common configuration) may find that ntsync is unavailable inside the sandbox even when the kernel supports it, unless the Flatpak manifest or runtime includes the necessary device access declaration. This is an active area of ecosystem integration work as of 2025–2026.

### 8.3 Comparison with Related Kernel Mechanisms

The ntsync trust model is comparable to other Linux mechanisms that grant unprivileged user access to kernel-managed state:

- **`memfd_create(2)`**: Creates anonymous files in kernel memory. Unprivileged; objects are isolated to the process (or explicitly shared). Analogous instance isolation.
- **`userfaultfd(2)`**: Allows userspace to handle page faults. Initially required `CAP_SYS_PTRACE`; this was relaxed in Linux 5.11 with `UFFD_USER_MODE_ONLY` and a sysctl knob (`/proc/sys/vm/unprivileged_userfaultfd`). The ntsync permission debate mirrors the userfaultfd history: initial caution, followed by practical relaxation.
- **`eventfd(2)`**: Fully unprivileged from its introduction; ntsync's predecessor esync used it directly. The analogy to eventfd is instructive — eventfd exposes a simple kernel counter to userspace without any security concern, and ntsync objects are conceptually more elaborate eventfds.

The consensus in the kernel community is that ntsync presents no privilege escalation risk because the objects it manages are software-only state with no connection to hardware resources, DMA, memory mappings, or privileged kernel data. The realistic threat model is solely a kernel bug in `ntsync.c` being exploitable.

---

## Roadmap

### Near-term (6–12 months)

- **Default device permissions fix in the kernel.** The `/dev/ntsync` character device shipped with mode `0600` in Linux 6.14, requiring a udev rule for non-root access. Kernel maintainer Greg Kroah-Hartman indicated openness to accepting a patch relaxing the default; a resolution — either in-kernel mode change or a standardized udev rule shipped with systemd — is expected to land in a near-term kernel release. [Source](https://www.phoronix.com/news/Linux-NTSYNC-Permissions-Issue)
- **Flatpak permission model integration.** Flatpak 1.17.4 added support for enabling ntsync unconditionally as part of development toward the Flatpak 1.18 release, addressing the GitHub issue requesting explicit `--device=ntsync` permission instead of the blunt `--device=all` workaround. [Source](https://github.com/flatpak/flatpak/issues/6199)
- **Proton 11 and Wine 11 stabilization.** Proton 11 (based on Wine 11, which merged ntsync support in its 10.15 development release) shipped in beta April–May 2026. Near-term work covers eliminating remaining regressions in frame-stutter patterns and ensuring the WoW64 architecture interoperates correctly with the ntsync dispatch path. [Source](https://www.phoronix.com/news/Wine-10.15-With-NTSYNC)
- **Broader distro enablement.** Ubuntu 26.04 LTS ships Linux kernel 6.14+ with `CONFIG_NTSYNC=m` enabled by default, bringing ntsync to the largest LTS user base; similar work is tracked in Fedora's Changes/NTSYNC page and in openSUSE. [Source](https://fedoraproject.org/wiki/Changes/NTSYNC)
- **Self-test coverage expansion.** The kernel selftests suite (`tools/testing/selftests/drivers/ntsync/`) already covers basic object operations; patches extending test coverage for edge cases in `NTSYNC_IOC_WAIT_ALL` with abandoned mutexes and alertable waits are expected as the driver matures. Note: needs verification for specific patchset status.

### Medium-term (1–3 years)

- **ARM64 / Steam Deck OLED and successor hardware.** Valve's Arm64 compatibility layer (shipping in Proton 11) brings Wine and ntsync to ARM Linux platforms. Medium-term work involves verifying that ntsync's spinlock and atomic operations behave correctly under ARM's weaker memory model and that performance characteristics hold on ARM SoCs used in handheld gaming devices. [Source](https://www.techpowerup.com/348297/steams-proton-gets-wine-11-gaming-performance-improvements-valve-launches-arm64-compatibility-layer)
- **Anti-cheat ecosystem compatibility.** As ntsync becomes ubiquitous, anti-cheat modules (BattlEye, Easy Anti-Cheat) must be updated to recognize `/dev/ntsync` fds in game process fd tables as benign and to interact correctly with ntsync-dispatched synchronization rather than wineserver RPC. This is an ongoing negotiation between Valve/Wine developers and anti-cheat vendors (see Chapter 171).
- **Container and sandbox integration.** Fully declarative per-app ntsync access in Flatpak, Toolbox, and Distrobox — analogous to how GPU device passthrough is modeled — requires coordinated work between the Flatpak runtime team, Steam's runtime container, and potentially a new `ntsync` portal in xdg-desktop-portal. Note: needs verification on specific portal proposal status.
- **Wine Staging and community fork convergence.** With upstream Wine 11 shipping ntsync, the long-maintained esync/fsync patchsets in wine-staging and wine-tkg-git become redundant. Medium-term maintenance effort focuses on deprecating and eventually removing these out-of-tree synchronization backends while preserving fallback paths for kernels older than 6.14.

### Long-term

- **Possible extension to other NT object types.** The current driver covers semaphores, mutexes, and auto/manual-reset events — the core wait-any/wait-all primitives. Future NT object types used by newer Windows APIs (e.g., keyed events, I/O completion ports semantics as used by newer runtimes) could be added if Wine developers identify the wineserver as a bottleneck for those paths. Note: needs verification; no RFC is currently public.
- **Integration with Linux futex2 and io_uring.** The kernel's synchronization primitive landscape continues to evolve: futex2 extensions and io_uring's `IORING_OP_FUTEX_WAIT` open architectural questions about whether ntsync's wait queues could eventually be expressed on top of, or composed with, these mechanisms rather than being a fully independent character device. This is speculative; current maintainers have not proposed such a unification.
- **Upstreaming ntsync semantics for non-Wine use cases.** The vectored, multi-typed, state-mutating atomic wait that ntsync implements is a genuinely useful primitive for any multi-threaded application. Long-term, the kernel community may explore whether a generalized version of wait-all semantics (beyond the Wine-specific UAPI) belongs in a broader Linux synchronization framework. This remains a speculative direction without a current proposal.

---

## 9. Integrations

**Chapter 28 (Windows Compatibility: Wine and Proton).** This chapter provides the broader context of Wine's architecture: the wineserver, the NT emulation layer, DXVK, and the overall Proton stack. Ntsync is the synchronization enabler for that entire compatibility layer — without correct and fast synchronization semantics, no amount of translation quality in DXVK or VKD3D-Proton can compensate for wineserver bottlenecks. The dispatch table in `dlls/ntdll/unix/sync.c` that selects between ntsync, fsync, esync, and wineserver RPC is the same code path that Chapter 28's wineserver description terminates at.

**Chapter 104 (DXVK and VKD3D-Proton).** D3D12's synchronization model is built around *fences* — `ID3D12Fence` objects that are signaled when GPU command lists complete. VKD3D-Proton translates D3D12 fences into Vulkan timeline semaphores on the GPU side, but the CPU-side signaling still routes through NT event objects that Wine must handle. When a D3D12 command queue calls `Signal()` on a fence, VKD3D-Proton internally fires an NT event that wakes any threads waiting on that fence via `WaitForSingleObject`. In a D3D12 game with multiple command queues submitting work in parallel — the common pattern for modern engines — every frame involves dozens of such fence signals and waits. Ntsync's low-overhead, correct event semantics directly reduce the CPU time spent in fence coordination, improving the throughput of VKD3D-Proton's command queue threads.

**Chapter 171 (Linux Anti-Cheat).** Anti-cheat systems that run inside Wine (or that inspect the Wine process from the host Linux side) interact with the same process model that ntsync improves. Some anti-cheat modules enumerate open file descriptors or inspect process memory to detect game state; the addition of `/dev/ntsync` fds and per-object fds to a Wine process's fd table may require anti-cheat modules to be updated to recognize these as benign. Furthermore, some anti-cheat systems inject code into the game process and may interact with Wine's synchronization dispatch — understanding whether the game's synchronization is going through ntsync versus wineserver RPC is relevant for diagnosing compatibility issues between specific anti-cheat modules and ntsync-enabled Wine builds.

---

## Further Reading

- [NT Synchronization Primitive Driver — The Linux Kernel Documentation](https://docs.kernel.org/userspace-api/ntsync.html) — the authoritative userspace-facing API reference.
- [linux/drivers/misc/ntsync.c — torvalds/linux on GitHub](https://github.com/torvalds/linux/blob/master/drivers/misc/ntsync.c) — kernel implementation; skim `ntsync_wait_any()`, `ntsync_wait_all()`, and `try_wake_all()` for the core algorithms.
- [linux/include/uapi/linux/ntsync.h — torvalds/linux on GitHub](https://github.com/torvalds/linux/blob/master/include/uapi/linux/ntsync.h) — the complete UAPI header.
- [NT Synchronization Primitive Driver — LWN.net coverage](https://lwn.net/Articles/974344/) — covers the design discussion and the evolution from v4 to v6.
- [wine/dlls/ntdll/unix/sync.c — wine-mirror on GitHub](https://github.com/wine-mirror/wine/blob/master/dlls/ntdll/unix/sync.c) — Wine's ntdll synchronization implementation, including the `linux_wait_objs()` path.
- [esync README — zfigura/wine on GitHub](https://github.com/zfigura/wine/blob/master/README.esync) — Zebediah Figura's original documentation of the esync design and its limitations.
- [The futex_waitv() syscall and gaming on Linux — Collabora](https://www.collabora.com/news-and-blog/blog/2023/02/17/the-futex-waitv-syscall-gaming-on-linux/) — context on fsync's kernel dependency and how futex_waitv() partially addressed it.
- [NTSYNC v6 patch series cover letter](https://lkml.iu.edu/hypermail/linux/kernel/2412.1/02004.html) — Zebediah Figura's December 2024 submission with performance benchmarks and design rationale.
- [Debian Wiki: Wine/NtsyncHowto](https://wiki.debian.org/Wine/NtsyncHowto) — practical setup guide including DKMS backport for pre-6.14 kernels.
