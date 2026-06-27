# Chapter 164: Android Runtime and Native Interop: ART, JNI, and the NDK

**Target audiences:** Systems and driver developers who need to understand how Android's managed runtime interacts with native code; graphics application developers writing Vulkan or OpenGL ES apps that cross the Java/native boundary; browser and web platform engineers understanding how Chrome, WebView, and ANGLE fit into Android's runtime model.

## Table of Contents

1. [ART and Dalvik: Where They Diverge from the JVM](#1-art-and-dalvik-where-they-diverge-from-the-jvm)
   - 1.1 [DEX Bytecode: Register-Based vs Stack-Based](#11-dex-bytecode-register-based-vs-stack-based)
   - 1.2 [dex2oat and the Compilation Tier Ladder](#12-dex2oat-and-the-compilation-tier-ladder)
   - 1.3 [Profile-Guided Compilation and Cloud Profiles](#13-profile-guided-compilation-and-cloud-profiles)
   - 1.4 [Concurrent Copying GC and Baker Read Barriers](#14-concurrent-copying-gc-and-baker-read-barriers)
   - 1.5 [userfaultfd GC (Android 13+)](#15-userfaultfd-gc-android-13)
   - 1.6 [The Zygote Fork Model and .art Boot Images](#16-the-zygote-fork-model-and-art-boot-images)
   - 1.7 [nterp: The Switch-Dispatch Interpreter (Android 12+)](#17-nterp-the-switch-dispatch-interpreter-android-12)
   - 1.8 [Non-SDK Interface Restrictions](#18-non-sdk-interface-restrictions)
   - 1.9 [ART Intrinsics](#19-art-intrinsics)
   - 1.10 [ART's Deoptimisation Model](#110-arts-deoptimisation-model)
   - 1.11 [Summary: ART vs HotSpot JVM Feature Matrix](#111-summary-art-vs-hotspot-jvm-feature-matrix)
2. [The Language-Runtime Bridge: JNI, NDK, and ART](#2-the-language-runtime-bridge-jni-ndk-and-art)
   - 2.1 [The Android GPU Call Stack](#21-the-android-gpu-call-stack)
   - 2.2 [@FastNative and @CriticalNative](#22-fastnative-and-criticalnative)
   - 2.3 [How They Work Internally: the ART JNI Trampoline](#23-how-they-work-internally-the-art-jni-trampoline)
   - 2.4 [Why Project Panama FFM Does Not Apply to Android](#24-why-project-panama-ffm-does-not-apply-to-android)
   - 2.5 [Is JNI Still Used? Current Status and Trends](#25-is-jni-still-used-current-status-and-trends)
   - 2.6 [NativeActivity, GameActivity, and ANativeWindow](#26-nativeactivity-gameactivity-and-anativewindow)
3. [The NDK's Future: Rust as Co-equal Language](#3-the-ndks-future-rust-as-co-equal-language)
   - 3.1 [The NDK Is Not Going Away](#31-the-ndk-is-not-going-away)
   - 3.2 [The Rust NDK Ecosystem](#32-the-rust-ndk-ecosystem)
   - 3.3 [APEX Delivery and NDK ABI Stability](#33-apex-delivery-and-ndk-abi-stability)
   - 3.4 [Expanding the android/ Header Namespace](#34-expanding-the-android-header-namespace)
   - 3.5 [What Will Not Replace the NDK](#35-what-will-not-replace-the-ndk)
   - 3.6 [NDK Trajectory](#36-ndk-trajectory)
4. [Integrations](#4-integrations)

---

## 1. ART and Dalvik: Where They Diverge from the JVM

Android Runtime (ART) is not a JVM. It shares Java syntax and Java's standard library API surface but diverges at every level of implementation: bytecode format, compilation model, garbage collector, bootstrap strategy, and interpreter architecture. Understanding these divergences matters directly for graphics developers: the GC model explains why `@CriticalNative` must be short, the compilation tier ladder explains warm-up jank, and the Zygote model explains the memory layout every Android process inherits.

### 1.1 DEX Bytecode: Register-Based vs Stack-Based

The JVM uses a **stack-based bytecode**: each instruction pushes or pops operands from an implicit evaluation stack. DEX — the Dalvik/ART bytecode format — uses a **register-based bytecode**: each instruction operand references a numbered virtual register directly. The same Java source compiles to significantly fewer DEX instructions because DEX eliminates the intermediate push/pop operations the JVM needs to move values between stack slots.

```
// Java source:
int z = x + y;

// JVM bytecode (stack-based, 4 instructions):
iload_1       // push x
iload_2       // push y
iadd          // pop both, push sum
istore_3      // pop into z

// DEX bytecode (register-based, 1 instruction):
add-int v2, v0, v1   // v2 = v0 + v1
```

DEX files pack multiple classes into a single `.dex` file with a shared string/type/proto pool, reducing redundant metadata compared to per-class `.class` files. The toolchain — `javac → d8/r8 → .dex` — converts JVM bytecode to DEX; since Android Studio 3.1, `d8` handles the conversion and `r8` optionally applies shrinking and obfuscation.

[Source: Dalvik bytecode format](https://source.android.com/docs/core/runtime/dalvik-bytecode), [DEX format spec](https://source.android.com/docs/core/runtime/dex-format)

### 1.2 dex2oat and the Compilation Tier Ladder

HotSpot JVM compiles "hot" methods at runtime using C1 (client) and C2 (server) tiered compilation. ART uses a separate offline AOT compiler, `dex2oat`, which runs at install time or during idle background dexopt. The tiers are:

| Compilation filter | When applied | Result |
|---|---|---|
| `verify` | First install; low-end devices | Bytecode verified but not compiled; interpreted only |
| `quicken` | Rare; low-priority installs | Verifies + quickens bytecode (replaces some opcodes with faster variants) |
| `speed-profile` | Post-install background dexopt after Play profile delivery | Compiles hot methods from a profile; the primary tier for most apps |
| `speed` | System apps; Zygote's boot image | Compiles all methods; used where profile data is not available |
| `everything` | Debug/engineering builds | Compiles every method including dead code |

The compiled output is an `.oat` file (a custom ELF extension) containing native machine code alongside the original DEX for uncompiled methods, plus an `.art` image file for pre-initialised heap objects (boot image only). At runtime, ART's interpreter is invoked for methods not yet AOT-compiled; the JIT compiler (`libart-compiler.so`, using the same Optimising backend as dex2oat) compiles hot interpreted methods in-process and records their profiles for the next dexopt cycle.

[Source: ART compilation](https://source.android.com/docs/core/runtime/configure), [dex2oat reference](https://source.android.com/docs/core/runtime/dex2oat)

### 1.3 Profile-Guided Compilation and Cloud Profiles

ART's **Profile-Guided Compilation (PGC)** is a closed feedback loop:

1. At first launch, ART interprets uncompiled code and the JIT compiles hot methods in-process.
2. The JIT records which methods and classes are hot in a per-app `.prof` file under `/data/misc/profiles/cur/`.
3. During device idle + charging, `dexopt` re-runs with the `speed-profile` filter, AOT-compiling exactly the hot set. Cold code is not compiled, saving space and improving startup.
4. **Cloud Profiles** (introduced Android 9, scaled via Play): Google Play collects anonymised, aggregated `.prof` data from the install population and delivers a pre-built "base profile" with the first APK download. This means a freshly-installed app can start with a `speed-profile` `.oat` on first launch, eliminating the cold interpreted startup of the first session. The Baseline Profiles API (`androidx.profileinstaller`) lets library and app developers ship curated profiles in the APK itself, extending the same guarantee to F-Droid and sideloaded apps.

[Source: Android performance profiles](https://developer.android.com/topic/performance/baselineprofiles/overview), [ART profile-guided compilation](https://source.android.com/docs/core/runtime/configure#how_profiles_are_used)

### 1.4 Concurrent Copying GC and Baker Read Barriers

HotSpot ships multiple GCs (G1, ZGC, Shenandoah, Serial). ART ships one production GC: **Concurrent Copying (CC)**, a compacting collector based on the Brooks/Baker read-barrier technique.

CC's central property: it can **move objects while mutator threads run**, without stopping the world for the copy phase. It achieves this via a **read barrier** inserted before every object-reference load. When an object has been moved, the old location stores a forwarding pointer; the read barrier follows it and returns the new address, updating the reference in place.

```
// Logical ART CC read barrier (inserted by dex2oat / JIT):
// Every time code loads a reference field, it becomes:
obj_ref = load(field_addr)
if (obj_ref.mark_bit == from_space) {
    obj_ref = obj_ref.forward_ptr     // follow forwarding pointer
    store(field_addr, obj_ref)        // update stale reference
}
```

ART uses **Baker read barriers** specifically: the condition check is a single branch on the mark bit, optimised for the common case (object already in to-space → branch not taken). On AArch64 the JIT generates the check inline with no function call for the fast path. The cost is one extra load + conditional branch per reference load — measurable in micro-benchmarks, but lower than the stop-the-world pauses it replaces.

Thread states (`kRunnable`, `kNative`, `kWaiting`) interact with CC:
- `kNative` threads are passively suspended: the GC does not need to wait for them during the copy phase because their managed references are pinned in the HandleScope.
- `kRunnable` threads must be stopped at safepoints before the GC can flip the from/to-space sense (the one brief STW pause CC requires).

[Source: ART GC overview](https://source.android.com/docs/core/runtime/gc-debug), [Baker read barrier paper — Tene et al. 2011](https://dl.acm.org/doi/10.1145/2258996.2259004)

### 1.5 userfaultfd GC (Android 13+)

Android 13 introduced **A-GC** (AppCompact GC), implemented via the kernel's `userfaultfd` mechanism, as an optional complement to CC. Rather than copying objects during a GC phase, A-GC compacts the heap by:

1. Remapping the heap using `mremap()` to a fresh anonymous mapping.
2. Installing a `userfaultfd` handler on the old mapping.
3. When a mutator thread faults on a remapped page, the `userfaultfd` handler copies the page content and resolves the fault — the mutator thread blocks briefly but other threads continue.

The advantage over CC's read-barrier model: **zero per-reference overhead** during normal execution. The fault overhead is paid only when an old-mapping page is first touched after a compaction, which for typical app workloads touches a small fraction of the heap. On Android 13+ devices with kernel 5.10+, ART selects between CC and userfaultfd GC based on device policy and the `ART_USE_READ_BARRIER` build flag.

[Source: Android 13 ART improvements](https://android-developers.googleblog.com/2022/08/gc-improvements-in-android-13.html), [userfaultfd kernel docs](https://www.kernel.org/doc/html/latest/admin-guide/mm/userfaultfd.html)

### 1.6 The Zygote Fork Model and .art Boot Images

HotSpot JVMs start each process independently. Android uses a **Zygote** model: a single long-lived `zygote` process pre-loads the boot class path (Android framework classes, system libraries) during device boot and then `fork()`s to create every new app process. Because `fork()` is copy-on-write, all app processes share the same physical pages for the boot class path until they diverge.

The Zygote's heap state is snapshotted as a **`.art` boot image**: a serialised heap containing pre-initialised objects for the most commonly used framework classes. At boot, the Zygote maps this image at a fixed address; forked app processes inherit it. This means:

- App startup avoids re-initialising thousands of framework objects.
- The pre-initialised heap pages are shared CoW across all processes — a significant RAM saving on devices running 50+ concurrent apps.
- The `.art` image is version-specific: an OTA update that changes the boot class path requires regenerating the image via `dex2oat`, which is part of the post-OTA `dexopt` pass.

```bash
# View the system boot image files:
adb shell ls /system/framework/*.art /data/dalvik-cache/*/*.art
# Example: /system/framework/arm64/boot.art (pre-initialised boot class path heap)
#          /data/dalvik-cache/arm64/data@app@com.example...@base.apk@classes.dex (app oat)
```

[Source: ART image format](https://source.android.com/docs/core/runtime/dex2oat#image-files), [Zygote and app startup](https://source.android.com/docs/core/arch/processes)

### 1.7 nterp: The Switch-Dispatch Interpreter (Android 12+)

Before Android 12, ART's primary interpreter was a **template interpreter** written in assembly, similar in concept to HotSpot's template interpreter. Android 12 replaced it with **nterp** (new interpreter), an assembler-written, opcode-indexed table-dispatch interpreter with two key properties:

1. **Direct DEX dispatch**: nterp operates directly on DEX bytecode without converting to an intermediate IR, reducing interpreter startup cost.
2. **JIT-aware design**: nterp methods can be on-stack-replaced (OSR) by JIT-compiled code at backedge points without unwinding the interpreter frame, because nterp's frame layout is compatible with the JIT's native frame format.

The practical effect: apps that are not AOT-compiled (new installs, debug builds, low-memory devices using the `verify` filter) run significantly faster under nterp than the old template interpreter for code that stays interpreted.

[Source: ART nterp introduction](https://android-developers.googleblog.com/2022/07/whats-new-in-android-12l-for-developers.html), [nterp source](https://android.googlesource.com/platform/art/+/refs/heads/main/runtime/interpreter/mterp/)

### 1.8 Non-SDK Interface Restrictions

HotSpot allows reflection into any private field or method of any class (with a warning since Java 9). ART enforces **non-SDK interface restrictions** (introduced Android 9, API 28): access to `@hide` and `@UnsupportedAppUsage` methods and fields in the Android framework is blocked or warned based on a greylist/blacklist classification updated with each API release.

The restrictions are enforced at multiple levels:
- **Reflection**: `Class.getDeclaredMethod()` with a hidden method name returns null or throws `NoSuchMethodException`.
- **JNI**: `FindClass` + `GetMethodID` on a hidden method returns 0 and logs a warning.
- **Bytecode access**: `invoke-virtual` to a hidden method in framework code generates a runtime check.

For native code, hidden NDK symbols are enforced at link time via `versioned` symbol visibility in the NDK's `libandroid.so` stubs.

Graphics relevance: several internal `SurfaceFlinger`, `GraphicBuffer`, and `EGL` methods that pre-Android-9 apps accessed via reflection are now hidden. Apps targeting API 28+ must use the published NDK or Java surface APIs.

[Source: Non-SDK interface restrictions](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces)

### 1.9 ART Intrinsics

ART's Optimising compiler (used by both dex2oat and the JIT) includes a table of **intrinsics** — Java methods replaced inline with hand-optimised machine code sequences rather than a function call. These cover:

| Category | Example intrinsics |
|---|---|
| `java.lang.Math` | `Math.abs`, `Math.min`, `Math.max`, `Math.sqrt`, `Math.floor`, `Math.ceil` |
| `java.lang.String` | `String.equals`, `String.charAt`, `String.indexOf`, `String.length` |
| `java.lang.System` | `System.arraycopy` (replaced with `memmove` or NEON SIMD) |
| Bit manipulation | `Integer.bitCount`, `Integer.numberOfLeadingZeros`, `Long.reverse` |
| Unsafe / VarHandle | `Unsafe.compareAndSwapInt`, `VarHandle.compareAndSet` |
| ART-specific | `Thread.currentThread()` (reads `art::Thread::Current()` TLS directly) |

For graphics-adjacent code, `System.arraycopy` on pixel buffer arrays and `Math` functions in shader-like Kotlin code are the most commonly impacted. OpenJDK HotSpot has a similar intrinsics table; the specific sets overlap but differ — HotSpot has more `String` intrinsics tied to `StringLatin1`/`StringUTF16` dual-encoding; ART's AArch64 backend has deeper NEON integration.

[Source: ART intrinsics table](https://android.googlesource.com/platform/art/+/refs/heads/main/compiler/intrinsics_list.h)

### 1.10 ART's Deoptimisation Model

ART's JIT and AOT compilers perform aggressive speculative optimisations — inline caches, class hierarchy analysis (CHA), type specialisation. When an assumption is violated at runtime (e.g., a `final` class is subclassed via a dynamically-loaded DEX, or an inline cache becomes polymorphic), ART performs **deoptimisation**: it patches the compiled code to call a deoptimisation entry point, reconstructs the interpreter state from the native frame, and falls back to interpreted execution.

Key deopt triggers in graphics-relevant code:
- **Inline cache overflow**: a virtual dispatch that was JIT-inlined for one concrete type encounters a second concrete type.
- **Class loading**: a `Class.forName()` or `DexClassLoader` load that invalidates a CHA assumption.
- **OSR exit**: on-stack replacement by nterp back to the JIT can trigger deopt at loop backedges if profiling assumptions change.

Unlike HotSpot's uncommon trap mechanism, ART deoptimisation can happen at any point — not just explicit uncommon-trap stubs — because the Optimising backend records enough metadata (the **dex register map**) to reconstruct the Dalvik register state from any native program point.

[Source: ART deoptimisation](https://android.googlesource.com/platform/art/+/refs/heads/main/runtime/deoptimize.h)

### 1.11 Summary: ART vs HotSpot JVM Feature Matrix

| Feature | OpenJDK HotSpot | Android Runtime (ART) |
|---|---|---|
| Bytecode format | JVM `.class` (stack-based) | DEX `.dex` (register-based) |
| AOT compilation | Optional (GraalVM native-image) | Standard (`dex2oat`, 5 tiers) |
| JIT compiler | C1 + C2 tiered | Optimising (single tier) + nterp |
| Garbage collector | G1 / ZGC / Shenandoah / Serial | Concurrent Copying (CC) + userfaultfd |
| GC read barriers | None (G1 uses write barriers) | Baker read barriers (CC) |
| Process start model | Fresh JVM per process | Zygote fork + CoW boot image |
| Foreign Function API | FFM / JEP 454 (Java 21+) | Not implemented — use JNI |
| JNI fast-path | None (use JNA / FFM) | `@FastNative`, `@CriticalNative` |
| Non-SDK restrictions | Java 9 module system (warnings) | Hard enforcement (API 28+) |
| Interpreter | Template interpreter (C1 threshold) | nterp (Android 12+) |
| Boot class path sharing | None | `.art` image (Zygote CoW) |
| Profile feedback | Not built-in (JFR samples) | PGC + Cloud Profiles |

---

## 2. The Language-Runtime Bridge: JNI, NDK, and ART

When an Android application issues a Vulkan command — whether written in Kotlin, Java, or native C++ — the path to the GPU hardware traverses a specific call stack that every Android graphics developer needs to understand.

### 2.1 The Android GPU Call Stack

```
Kotlin / Java (app code)
    │  JNI transition (may use @FastNative / @CriticalNative)
    ▼
Android Runtime (ART) — the Android JVM implementation
    │
    ▼
Framework native C++ (libhwui.so, libEGL.so, libGLESv2.so …)
    │  dlopen / HAL discovery
    ▼
Vendor Vulkan ICD (libvulkan_adreno.so, libvulkan_mali.so …)
    │  ioctl
    ▼
Kernel DRM driver (msm_drm, mali, pvr …)
    │
    ▼
GPU hardware
```

For performance-critical rendering, Java or Kotlin code never touches the hot path. The established pattern is the **NDK-direct** model: cross the JNI boundary once during initialisation — to create the `VkDevice`, allocate the `VkSwapchainKHR`, and record long-lived command buffers. A dedicated native thread then owns the entire GPU submission timeline (`vkQueueSubmit`, `vkAcquireNextImageKHR`, `vkQueuePresentKHR`) without ever returning to ART.

JNI transitions introduce overhead even with ART's improvements in Android 13+:
- Checking whether the calling thread holds any GC-safepoint-blocking monitors.
- Transitioning thread state from `kRunnable` to `kNative`.
- Registering the native stack frame with the GC root table.

Eliminating JNI from the render loop eliminates all of this.

### 2.2 @FastNative and @CriticalNative

ART provides two annotations for reducing JNI call overhead on cases where the boundary cannot be eliminated. Both are public API since Android 8.0:

**`@dalvik.annotation.optimization.FastNative`**: Removes the managed→unmanaged state-transition overhead. The native method must not call back into Java and must not allocate managed objects. Reduces overhead by approximately 2× on AArch64.

**`@dalvik.annotation.optimization.CriticalNative`**: Removes the JNI envelope entirely. The native method receives raw C-type arguments with no `JNIEnv*` and no `jobject`/`jclass` parameters. Must be `static`, must not interact with Java objects, must not allocate. Overhead approaches that of a direct C function call — 3–5× cheaper than standard JNI on AArch64.

```java
// File: com/example/vulkan/VulkanBridge.java
import dalvik.annotation.optimization.CriticalNative;
import dalvik.annotation.optimization.FastNative;

public class VulkanBridge {
    // Standard JNI: full JNIEnv overhead; may call back into JVM
    public static native void submitCommandBuffer(long cmdBufHandle);

    // FastNative: ~2× cheaper; no Java callbacks, no allocation
    @FastNative
    public static native int getFenceStatus(long fenceHandle);

    // CriticalNative: ~3–5× cheaper; no JNIEnv param; static only
    @CriticalNative
    public static native long getGpuTimestamp();
}
```

```c
// File: vulkan_bridge.c
#include <jni.h>
#include <vulkan/vulkan.h>

// Standard JNI — receives JNIEnv* and jclass
JNIEXPORT void JNICALL
Java_com_example_vulkan_VulkanBridge_submitCommandBuffer(
        JNIEnv* env, jclass clazz, jlong cmdBufHandle)
{
    VkCommandBuffer cmdBuf = (VkCommandBuffer)(uintptr_t)cmdBufHandle;
    // ... vkQueueSubmit etc.
}

// CriticalNative — no JNIEnv*, no jclass; raw C types only
JNIEXPORT jlong JNICALL
Java_com_example_vulkan_VulkanBridge_getGpuTimestamp()
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (jlong)ts.tv_nsec;
}
```

[Source: ART JNI Tips](https://developer.android.com/training/articles/perf-jni), [dalvik.annotation.optimization source](https://android.googlesource.com/platform/libcore/+/refs/heads/main/dalvik/src/main/java/dalvik/annotation/optimization/)

### 2.3 How They Work Internally: the ART JNI Trampoline

When ART AOT-compiles (`dex2oat`) or JIT-compiles a `native` method it generates a *JNI trampoline stub* — machine code inserted between the managed caller and the C function. On AArch64, a standard JNI stub does the following before the native function executes:

1. **Thread state transition `kRunnable → kNative`** — ART's concurrent-copying GC tracks every thread's state in `art::Thread::tls32_.state_and_flags`. Changing to `kNative` requires a `StoreRelease` memory barrier so the GC thread sees the update; without it the GC could begin compacting heap objects the native frame still references.
2. **HandleScope push** — A `HandleScope` frame is pushed onto the managed stack. Every `jobject`/`jarray`/`jstring` parameter is wrapped as an indirect handle so the GC can relocate the underlying object if it compacts the heap during the call.
3. **Exception state save** — The pending-exception slot is recorded so the stub can detect a Java exception raised indirectly by the native code.
4. **`JNIEnv*` + `jclass`/`jobject` prepend** — The native function's first two arguments are always injected: a `JNIEnv*` pointer and a `jclass` (static) or `jobject` (instance), regardless of the Java signature.
5. **Call the native function** — `blr x17` using the pointer from `ArtMethod::GetEntryPointFromJni()`.
6. **GC safepoint check on return** — The stub loads `Thread::tls32_.state_and_flags` and checks `kSuspendRequest` / `kCheckpointRequest`. If a GC is waiting, the thread calls `art::Thread::CheckSuspend()` and blocks.
7. **`kNative → kRunnable` transition** — Reversed with another barrier.
8. **HandleScope pop + local-ref cleanup** — The frame is torn down; a `jobject` return value is unwrapped from its handle.

Steps 1–2 and 6–8 — two barrier-protected state writes, the HandleScope push/pop, and the conditional GC-suspension check — dominate the call overhead. On a trivial native function these 10–15 extra instructions can be comparable to the function body itself.

#### @FastNative internals: skip the HandleScope, keep the state transition

ART records the annotation as the access flag `kAccFastNative` (`0x00080000`) in `ArtMethod`'s modifier word. The JNI trampoline generator (`art/compiler/jni/jni_compiler.cc`, `JniCompiler::Compile()`) checks this flag and emits a reduced stub:

- **Omitted**: HandleScope push/pop and all local-reference table bookkeeping. The contract guarantees no new managed objects are created, so the table never needs to exist.
- **Kept**: `kRunnable → kNative` state transition with its `StoreRelease` barriers.
- **Kept**: GC safepoint check on return.
- **Kept**: `JNIEnv*` + `jclass`/`jobject` first-argument injection — the method can still read and convert parameters through JNI helpers; it just cannot create new Java objects or call back into the JVM.

The transition barriers are the remaining dominant cost, which is why the speedup is approximately 2× rather than more.

#### @CriticalNative internals: eliminate the state transition, change the ABI

`@CriticalNative` (flag `kAccCriticalNative`, `0x00200000`) removes the thread state transition entirely. The thread stays in `kRunnable` throughout the call.

The generated stub:
- **Omitted**: `kRunnable → kNative` transition and all associated barriers.
- **Omitted**: HandleScope and all local-reference machinery.
- **Omitted**: `JNIEnv*` and `jclass`/`jobject` prepend — the C function's calling convention becomes a plain C ABI matching the Java primitive types directly (`jlong → int64_t`, `jint → int32_t`, `jfloat → float`). No JVM pointers appear in the frame.
- **Omitted**: GC safepoint check on return (the thread never left `kRunnable`).
- **Result**: the stub reduces to register setup + `blr` + `ret`, which the compiler backend treats identically to a direct C call.

The ABI change is why `@CriticalNative` is `static`-only. Instance methods require a `jobject this` as the first Java argument. Without a HandleScope there is no safe way to pin an object reference across the call — so instance methods cannot use this annotation.

#### GC interaction: why @CriticalNative methods must be short

ART's Concurrent Copying (CC) collector classifies threads by state:

- `kNative` threads are *passively suspended* from the GC's perspective: their native frames hold no live heap pointers the GC needs to scan (those are in the HandleScope), so the collector can proceed with compaction without waiting for them.
- `kRunnable` threads are *actively running*: the GC cannot compact heap objects while they might be accessing them. Before a compacting phase, the GC issues `SuspendAll()`, which spin-waits for every `kRunnable` thread to reach a safepoint or enter `kNative`.

A `@CriticalNative` call leaves the thread in `kRunnable`. If a GC `SuspendAll()` arrives mid-call, the GC thread must wait until the method returns and the thread hits its next managed safepoint. A long `@CriticalNative` call directly delays GC pause completion. Standard JNI and `@FastNative` avoid this: their `kNative` transition lets the collector proceed concurrently, paying the barrier cost in exchange for not blocking the GC. The practical rule: `@CriticalNative` is for nanosecond-to-low-microsecond operations (timestamp reads, flag polls, arithmetic on GPU handles); anything that might block the GC should use `@FastNative` instead.

#### What the three modes strip compared to standard JNI

| Step | Standard JNI | `@FastNative` | `@CriticalNative` |
|---|---|---|---|
| `kRunnable → kNative` barrier | Yes | Yes | **No** |
| HandleScope push/pop | Yes | **No** | **No** |
| `JNIEnv*` + `jclass` prepend | Yes | Yes | **No** |
| GC safepoint check on return | Yes | Yes | **No** |
| Can call Java / allocate | Yes | **No** | **No** |
| Can be instance method | Yes | Yes | **No** |
| Blocks GC during call | No | No | **Yes** |

#### Relevant ART source files

| File | Role |
|---|---|
| `art/compiler/jni/jni_compiler.cc` | Trampoline stub generator; branches on `kAccFastNative` / `kAccCriticalNative` |
| `art/runtime/entrypoints/quick/quick_jni_entrypoints.cc` | Fast/critical entry-point implementations |
| `art/runtime/thread.cc` | `TransitionFromRunnableToSuspended()`, `TransitionFromSuspendedToRunnable()`, `CheckSuspend()` |
| `art/runtime/jni/jni_env_ext.cc` | HandleScope push/pop, local reference table |
| `art/libdex/modifiers.h` | `kAccFastNative = 0x00080000`, `kAccCriticalNative = 0x00200000` |
| `art/runtime/art_method.h` | `ArtMethod::IsFastNative()`, `ArtMethod::IsCriticalNative()` |

[Source: ART JNI compiler](https://android.googlesource.com/platform/art/+/refs/heads/main/compiler/jni/jni_compiler.cc), [ART thread state machine](https://android.googlesource.com/platform/art/+/refs/heads/main/runtime/thread.cc)

#### Java licensing: do these annotations raise IP concerns?

No, for three independent reasons.

**`dalvik.*` is not Java SE IP.** `@FastNative` and `@CriticalNative` live in the `dalvik.annotation.optimization` package. The `dalvik.*` namespace is Android-proprietary and has no counterpart in the Java SE specification or OpenJDK. Oracle's IP claims centred on the `java.*` and `javax.*` API packages — their structure, sequence, and organisation (SSO). Google never asserted `dalvik.*` was Java SE; it is an ART-internal extension with no standardised equivalent.

**Oracle v. Google (SCOTUS 2021) settled the broader question.** Android reimplements 37 `java.*` / `javax.*` API packages (e.g. `java.io`, `java.util`, `java.lang`) using its own runtime (ART) rather than a licensed JVM. Oracle sued Google over this in 2010. The US Supreme Court ruled 6–2 in April 2021 that Google's reimplementation constituted **fair use**. The `dalvik.*` extensions were never part of Oracle's claim and are even further removed from any Java IP concern than the packages actually litigated.

**JNI is an open specification; Java syntax is not proprietary.** The Java Native Interface specification is published by Oracle as part of the Java SE documentation and may be freely implemented. The JNI trampoline stubs that these annotations optimise do not require a Java licence. Android compiles Java source with `javac` then converts `.class` bytecode to `.dex` via `d8`/`r8`; OpenJDK's compiler is GPL with Classpath Exception, making it freely usable. Oracle's copyright in Oracle v. Google covered the SSO of the API packages, not the Java language grammar.

### 2.4 Why Project Panama FFM Does Not Apply to Android

Java 21 standardised the **Foreign Function & Memory API** (FFM, JEP 454) as a replacement for JNI in the OpenJDK/HotSpot JVM. FFM provides `MethodHandle`-based native dispatch and `MemorySegment` for off-heap memory without `sun.misc.Unsafe`. It is widely discussed as the future of Java native interop.

**FFM is a HotSpot-internal, not a portable JVM contract.** FFM's implementation is built on HotSpot's JVM TI, MethodHandle intrinsics, and Graal-based foreign call stubs. Android Runtime (ART) is a separate, incompatible JVM implementation with its own bytecode format (Dalvik bytecode → ART IR), its own concurrent-copying garbage collector, and its own JIT/AOT pipeline (`dex2oat`, Cloud Profiles). None of the HotSpot machinery that FFM depends on exists in ART.

| Feature | OpenJDK HotSpot | Android Runtime (ART) |
|---|---|---|
| VM bytecode format | JVM `.class` files | Dalvik `.dex` bytecode |
| FFM / JEP 454 | Yes — Java 21+ | No — not implemented in ART |
| `@FastNative` / `@CriticalNative` | No | Yes — Android 8.0+ |
| `sun.misc.Unsafe` | Yes | Restricted since API 28 |
| JVM TI | Yes | Partial (debug builds only) |
| Garbage collector | G1 / ZGC / Shenandoah | Concurrent Copying (CC) |

ART's evolution path for native interop is **not FFM**: it is `@FastNative`/`@CriticalNative` for unavoidable JNI calls, the NDK-direct pattern for the hot path, and — for shared business logic — Kotlin Multiplatform (which on Android still targets ART, not Kotlin/Native).

[Source: JEP 454 — Foreign Function & Memory API](https://openjdk.org/jeps/454), [ART Dex format overview](https://source.android.com/docs/core/runtime/dex-format)

### 2.5 Is JNI Still Used? Current Status and Trends

Yes — extensively, but the architecture has evolved to confine it to the cold path.

**JNI is unavoidable at the framework boundary.** The Android API surface is Java/Kotlin by design, and every Java/Kotlin call into a system service crosses JNI. `android.media.MediaCodec` calls into the C++ `MediaCodec` in `frameworks/av`; `android.hardware.camera2` reaches the Camera HAL via `CameraService`; every `android.opengl.GLES31.*` call invokes the native GL driver through JNI. SurfaceFlinger, AudioFlinger, and the sensor stack are C++ services with JNI-wrapped Java APIs above them. This layer is not going away — it is the Android API surface.

**The hot-path pattern: JNI only at startup, zero per frame.** For graphics and game workloads the current idiom is to use JNI only at the lifecycle boundary — obtaining a surface handle, requesting permissions, reading device properties — and then drop entirely into NDK C/C++ for the render loop. Two mechanisms enable this:

- **`NativeActivity` / `GameActivity`** (Android Game Development Kit, AGDK) — `android.app.NativeActivity` delivers Android lifecycle callbacks (`onStart`, `onSurfaceCreated`, `onInputEvent`) as C function pointers via `ANativeActivity_onCreate`. The entire app loop — Vulkan frame submission, input handling, audio — runs in C/C++ with zero JNI per frame. `GameActivity` (AGDK 2021+) extends this with lower-latency input and `SurfaceView` integration.
- **`ANativeWindow`** — a C handle to the platform surface, obtained once at startup via a single `ANativeWindow_fromSurface(env, surface)` JNI call, then passed directly to `vkCreateAndroidSurfaceKHR`. All subsequent Vulkan API calls are pure C — no JNI in the render loop.

**Rust on Android bypasses JNI for new platform services.** Android's own codebase increasingly uses Rust (Keystore2, DNS-over-HTTPS, Bluetooth, `virtualizationservice`). New Rust services communicate with each other via Binder AIDL, not JNI. Where Rust must call Java APIs, the `jni` crate provides JNI bindings; but the preference is to keep Rust services below the Java layer entirely. Rust Vulkan code (`ash`, `vulkano`) calls the Vulkan C API via FFI — no JNI involved.

**Flutter, Unity, and Unreal use JNI only as a startup shim.** All three engines integrate with the Android `Activity` lifecycle and Play Store APIs via a thin JNI layer, then keep their render loops in C++. Flutter's Impeller renderer submits Vulkan commands with no JNI in the render path; Unity and Unreal follow the same pattern.

**Summary: JNI's role is narrowing.** Its domain is (a) the unchangeable `java.*` / `android.*` API surface, (b) lifecycle glue for native apps, and (c) legacy code not yet ported. `@FastNative` and `@CriticalNative` exist because even these unavoidable boundary calls matter at initialisation time. The direction of new Android platform work — Rust Binder services, `GameActivity`, AGDK — is to reduce per-frame JNI calls to zero on the hot path, not merely to make the remaining ones faster.

### 2.6 NativeActivity, GameActivity, and ANativeWindow

The zero-JNI-per-frame pattern rests on two NDK foundations: an activity hosting mechanism that delivers lifecycle events as C callbacks, and an opaque surface handle that feeds directly into Vulkan and EGL.

#### NativeActivity and android_native_app_glue

`android.app.NativeActivity` is a Java `Activity` subclass that loads a native `.so` and resolves the C entry point `ANativeActivity_onCreate` by name (overridable via `android.app.func_name` in `AndroidManifest.xml`). The NDK-provided `android_native_app_glue` library wraps this with a more ergonomic model: it spawns a **dedicated background pthread** for `android_main()`, keeping the app's C loop entirely separate from the Java main thread so that Android framework callbacks never block long enough to trigger an ANR.

The glue library's central type is `android_app`:

```c
// sources/android/native_app_glue/android_native_app_glue.h
struct android_app {
    void*             userData;
    void (*onAppCmd)(struct android_app*, int32_t cmd);   // APP_CMD_* handler
    int32_t (*onInputEvent)(struct android_app*, AInputEvent*);
    ANativeActivity*  activity;
    AConfiguration*   config;
    void*             savedState;
    size_t            savedStateSize;
    ALooper*          looper;        // LOOPER_ID_MAIN / LOOPER_ID_INPUT / LOOPER_ID_USER
    AInputQueue*      inputQueue;
    ANativeWindow*    window;        // non-NULL when surface is ready
    ARect             contentRect;
    int               activityState;
    int               destroyRequested;
};
```

The application defines `void android_main(struct android_app* app)`. The glue library's `ANativeActivity_onCreate` allocates `android_app`, opens a pipe pair for main-thread ↔ glue-thread communication, and calls `android_main()` on the new pthread. Android lifecycle callbacks on the Java main thread (`onStart`, `onResume`, `onNativeWindowCreated`, etc.) are serialised as `APP_CMD_*` integers onto the write end of the pipe; the background thread reads them via `ALooper_pollOnce()`.

`ANativeActivityCallbacks` exposes all 16 lifecycle points as C function pointers (all fire on the Java main thread; all default to NULL):

```c
struct ANativeActivityCallbacks {
    void  (*onStart)(ANativeActivity*);
    void  (*onResume)(ANativeActivity*);
    void* (*onSaveInstanceState)(ANativeActivity*, size_t* outSize);
    void  (*onPause)(ANativeActivity*);
    void  (*onStop)(ANativeActivity*);
    void  (*onDestroy)(ANativeActivity*);
    void  (*onWindowFocusChanged)(ANativeActivity*, int hasFocus);
    void  (*onNativeWindowCreated)(ANativeActivity*, ANativeWindow*);
    void  (*onNativeWindowResized)(ANativeActivity*, ANativeWindow*);
    void  (*onNativeWindowRedrawNeeded)(ANativeActivity*, ANativeWindow*);
    void  (*onNativeWindowDestroyed)(ANativeActivity*, ANativeWindow*);
    void  (*onInputQueueCreated)(ANativeActivity*, AInputQueue*);
    void  (*onInputQueueDestroyed)(ANativeActivity*, AInputQueue*);
    void  (*onContentRectChanged)(ANativeActivity*, const ARect*);
    void  (*onConfigurationChanged)(ANativeActivity*);
    void  (*onLowMemory)(ANativeActivity*);
};
```

Input events arrive via `AInputQueue` on the `LOOPER_ID_INPUT` fd:

```c
AInputEvent* event = NULL;
while (AInputQueue_getEvent(app->inputQueue, &event) >= 0) {
    if (AInputQueue_preDispatchEvent(app->inputQueue, event)) continue;
    int32_t handled = 0;
    int type = AInputEvent_getType(event);
    if (type == AINPUT_EVENT_TYPE_MOTION) {
        float x = AMotionEvent_getX(event, 0);
        float y = AMotionEvent_getY(event, 0);
        handled = 1;
    }
    AInputQueue_finishEvent(app->inputQueue, event, handled);
}
```

**NativeActivity limitations:** (1) No `SurfaceView` control — the activity owns the entire window. (2) Text input is broken — `ANativeActivity_showSoftInput()` raises the IME but there is no callback to receive the resulting text. (3) Input event coalescing — high-frequency touch events are batched and historical samples are not exposed.  [Source: android_native_app_glue.h](https://android.googlesource.com/platform/ndk/+/refs/heads/master/sources/android/native_app_glue/android_native_app_glue.h)

#### GameActivity and AGDK

The **Android Game Development Kit** (AGDK, 2021+) is a Jetpack library suite addressing NativeActivity's limitations. The core package `androidx.games:games-activity` provides `GameActivity`, a Kotlin/Java subclass (apps must subclass it: `class MainActivity : GameActivity()`) that replaces the glue layer:

```cmake
find_package(game-activity REQUIRED CONFIG)
target_link_libraries(${PROJECT_NAME} PUBLIC game-activity::game-activity_static)
# Linker entry point changes from ANativeActivity_onCreate to:
# -u Java_com_google_androidgamesdk_GameActivity_initializeNativeCode
```

The AGDK `android_app` struct replaces `ANativeActivity*` with `GameActivity*` and drops the `onInputEvent` callback in favour of a **double-buffered input array**:

```c
struct android_app {
    GameActivity* activity;     // replaces ANativeActivity*
    ANativeWindow* window;
    ALooper*       looper;
    // Input arrays — no AInputQueue callback:
    GameActivityMotionEvent motionEvents[NATIVE_APP_GLUE_MAX_NUM_MOTION_EVENTS];
    uint64_t                motionEventsCount;
    GameActivityKeyEvent    keyDownEvents[NATIVE_APP_GLUE_MAX_NUM_KEY_EVENTS];
    uint64_t                keyDownEventsCount;
    GameActivityKeyEvent    keyUpEvents[NATIVE_APP_GLUE_MAX_NUM_KEY_EVENTS];
    uint64_t                keyUpEventsCount;
    int                     textInputState;
    // ... remainder same as classic glue
};
```

Input is consumed by swapping the buffer rather than polling a queue — this exposes all historical samples and all pointer axes:

```c
GameActivityInputBuffer* ib = android_app_swap_input_buffers(app);
if (ib) {
    for (uint64_t i = 0; i < ib->motionEventsCount; i++) {
        GameActivityMotionEvent* e = &ib->motionEvents[i];
        int32_t action = e->action & AMOTION_EVENT_ACTION_MASK;
        float x = GameActivityPointerAxes_getAxisValue(
                      &e->pointers[0], AMOTION_EVENT_AXIS_X);
    }
    android_app_clear_motion_events(ib);
}
```

**Text input** is handled by `game-text-input`: `GameActivity_setImeEditorInfo()` configures the IME, and `GameActivity_getTextInputState()` retrieves pending text — the broken NativeActivity path is replaced entirely.

**Paddleboat** (`androidx.games:games-controller`) adds gamepad enumeration, mapping, vibration, and battery. It plugs into the GameActivity input stream:

```c
Paddleboat_init(env, jcontext);
// Per frame:
Paddleboat_update(env);
Paddleboat_processGameActivityMotionInputEvent(&motionEvent, sizeof(motionEvent));
// Read controller state:
Paddleboat_Controller_Data ctrl;
Paddleboat_getControllerData(0, &ctrl);
// ctrl.buttonsDown bitmask, ctrl.leftStick.stickX/Y, ctrl.triggerL2, etc.
```

[Source: GameActivity overview](https://developer.android.com/games/agdk/game-activity/overview), [android_app struct](https://developer.android.com/reference/games/game-activity/struct/android-app)

#### ANativeWindow in depth

`ANativeWindow*` is an opaque C handle to a `Surface` — the **producer end of a `BufferQueue`**. Internally, `frameworks/native`'s `Surface` class implements the `ANativeWindow` C vtable; casting a `Surface*` to `ANativeWindow*` is the bridge. Buffers pushed through it flow via Binder IPC to SurfaceFlinger as the consumer.

**Reference counting** — both Vulkan and EGL manage the window lifetime through acquire/release:

```c
ANativeWindow_acquire(window);   // increment refcount
ANativeWindow_release(window);   // decrement; may free
// vkCreateAndroidSurfaceKHR calls acquire internally;
// vkDestroySurfaceKHR calls release. A window cannot be bound
// to both a VkSurfaceKHR and an EGLSurface simultaneously.
```

**Geometry and format:**

```c
int32_t w = ANativeWindow_getWidth(window);
int32_t h = ANativeWindow_getHeight(window);
// Override: pass 0,0 to use the window's natural size
ANativeWindow_setBuffersGeometry(window, 0, 0,
    WINDOW_FORMAT_RGBA_8888);  // = AHARDWAREBUFFER_FORMAT_R8G8B8A8_UNORM
```

**Vulkan swapchain creation** — the only JNI in the entire Vulkan path:

```c
// One-time at APP_CMD_INIT_WINDOW:
ANativeWindow* anw = app->window;  // from android_app — no JNI needed here
VkAndroidSurfaceCreateInfoKHR info = {
    .sType  = VK_STRUCTURE_TYPE_ANDROID_SURFACE_CREATE_INFO_KHR,
    .window = anw,
};
vkCreateAndroidSurfaceKHR(instance, &info, NULL, &surface);
// All subsequent Vulkan calls: pure C, zero JNI
```

When obtained from `ANativeWindow_fromSurface(env, jSurface)` (e.g., from a `SurfaceHolder` in a hybrid app) that single call is the only JNI crossing for the entire render lifetime.

**CPU software rendering path** (for 2D fallback or screenshot readback):

```c
ANativeWindow_Buffer buf;
ARect dirty = {0, 0, w, h};
ANativeWindow_lock(window, &buf, &dirty);
// buf.bits → raw pixel pointer; buf.stride → row stride in pixels
memset(buf.bits, 0xFF, buf.stride * h * 4);  // fill white
ANativeWindow_unlockAndPost(window);
```

[Source: ANativeWindow NDK reference](https://developer.android.com/ndk/reference/group/a-native-window)

#### Swappy: Android Frame Pacing

**Swappy** (part of AGDK, `com.google.android.games:games-frame-pacing`) solves two jank patterns that arise when a Vulkan app calls `vkQueuePresentKHR` without vsync awareness:

- **Double-present jank**: a short frame completes early and is displayed for fewer vsync intervals than intended, causing uneven cadence.
- **Buffer stuffing**: a long frame causes the BufferQueue to fill; the CPU keeps submitting work, increasing pipeline depth and adding input latency.

Swappy injects `VkPresentTimesInfoGOOGLE` (via `VK_GOOGLE_display_timing`) into the `VkPresentInfoKHR.pNext` chain to schedule presentation at the correct future vsync, and uses Android `Choreographer` for per-device vsync calibration. On multi-refresh-rate displays it selects the closest matching rate (a 45 fps game targets 90 Hz rather than dropping to 30 Hz on 60 Hz).

```c
// Step 1 — query required extensions (adds VK_GOOGLE_display_timing if available)
SwappyVk_determineDeviceExtensions(physDevice,
    availCount, pAvail, &reqCount, ppReq);

// Step 2 — initialise after swapchain creation
uint64_t refreshNs;
SwappyVk_initAndGetRefreshCycleDuration(env, jactivity,
    physDevice, device, swapchain, &refreshNs);

// Step 3 — bind the ANativeWindow for display-timing queries
SwappyVk_setWindow(device, swapchain, anw);

// Step 4 — set target frame interval (nanoseconds)
SwappyVk_setSwapIntervalNS(device, swapchain, refreshNs); // e.g. 16_666_666 for 60 Hz

// Step 5 — replace vkQueuePresentKHR in the render loop
VkResult r = SwappyVk_queuePresent(queue, &presentInfo);

// Cleanup before vkDestroySwapchainKHR
SwappyVk_destroySwapchain(device, swapchain);
```

[Source: Android Frame Pacing library](https://developer.android.com/games/sdk/frame-pacing), [SwappyVk API reference](https://developer.android.com/games/sdk/reference/frame-pacing/group/swappy-vk)

#### Lifecycle event ordering and Vulkan swapchain

The critical ordering rule: **create the Vulkan swapchain in `APP_CMD_INIT_WINDOW`, not `APP_CMD_RESUME`**. The surface is created after `onResume` fires, and it can be destroyed while the activity remains resumed (e.g., when another window covers it). `android_app::window` is the authoritative signal:

| `APP_CMD_*` event | `window` state | Vulkan action |
|---|---|---|
| `APP_CMD_RESUME` | may be NULL | — |
| `APP_CMD_INIT_WINDOW` | non-NULL | **Create `VkSurfaceKHR` and swapchain** |
| `APP_CMD_GAINED_FOCUS` | non-NULL | Begin render loop |
| `APP_CMD_LOST_FOCUS` | non-NULL | Pause render loop |
| `APP_CMD_TERM_WINDOW` | about to become NULL | **Destroy swapchain and `VkSurfaceKHR`** |
| `APP_CMD_PAUSE` | NULL or non-NULL | — |
| `APP_CMD_DESTROY` | NULL | Destroy all Vulkan resources |

```c
void handle_cmd(struct android_app* app, int32_t cmd) {
    switch (cmd) {
    case APP_CMD_INIT_WINDOW:
        vulkan_create_surface_and_swapchain(app->window);
        break;
    case APP_CMD_TERM_WINDOW:
        vulkan_destroy_swapchain_and_surface();
        break;
    }
}
```

#### What is evolving on the managed side

For Kotlin/Java developers who want GPU access without dropping to the NDK:

**`android.graphics.vulkan` managed wrappers (Android 14+)**: A thin `AutoCloseable`-lifetime wrapper layer over the Vulkan NDK surface — `VulkanDevice`, `VulkanCommandBuffer`, `VulkanFence`. These are JNI-backed at the implementation level; the wrapper provides ergonomics and lifetime safety for Kotlin code that occasionally issues Vulkan commands, not performance for the hot render loop. [Source: Android Vulkan Overview](https://developer.android.com/games/develop/vulkan/overview)

**ANGLE as default system GLES (Android 17+)**: For apps using `android.opengl.*`, ANGLE's promotion to the default GLES implementation is transparent below the JNI boundary. The managed API surface is unchanged; the call stack below it now routes through ANGLE → Vulkan ICD instead of a vendor GLES blob.

**LiteRT Vulkan delegate**: TensorFlow Lite's successor LiteRT (Large-scale Inference with RealTime efficiency) ships a Vulkan compute delegate for on-device ML inference. As of Android 14 this is exiting experimental status. Kotlin/Java LiteRT API callers get Vulkan-accelerated inference without writing any JNI or NDK code directly. [Source: LiteRT Vulkan delegate](https://ai.google.dev/edge/litert/inference#vulkan_delegate)

---

## 3. The NDK's Future: Rust as Co-equal Language

### 3.1 The NDK Is Not Going Away

A recurring question in Android ecosystem discussions is whether the NDK is "going away" — deprecated by Kotlin, superseded by WebAssembly, or replaced by a higher-level abstraction. The answer is direct: the NDK is not going away. It is the ABI through which every native library on Android is delivered and called. SurfaceFlinger, HWComposer, the Vulkan ICD, `libaaudio.so`, codec2, the Vulkan loader itself, and `libvulkan.so` are all NDK consumers. The NDK ABI surface is stable indefinitely.

The realistic trajectory is: C and C++ remain fully supported; **Rust is being added as a co-equal native language** for new Android platform code and increasingly for app-layer native code.

### 3.2 The Rust NDK Ecosystem

Google has been shipping Rust in AOSP since 2021, and the NDK tooling has expanded accordingly.

**`ndk` and `ndk-sys` Rust crates**: Safe and unsafe Rust bindings to the NDK public API. `ndk-sys` provides raw `extern "C"` FFI bindings generated from the NDK headers; `ndk` provides idiomatic safe wrappers. Coverage includes `AHardwareBuffer`, `ANativeWindow`, `ANativeActivity`, `AInputQueue`, `AAudio`, and EGL. [Source: ndk-rs GitHub](https://github.com/rust-mobile/ndk)

```toml
# Cargo.toml
[dependencies]
ndk     = "0.9"   # idiomatic safe wrappers over NDK APIs
ndk-sys = "0.6"   # raw FFI bindings (auto-generated from NDK headers)
ash     = "0.38"  # raw Vulkan bindings — works via NDK ABI unchanged
```

**`ash` for Vulkan in Rust**: The `ash` crate provides thin Vulkan bindings generated from `vk.xml`. On Android it loads `/system/lib64/libvulkan.so` via `dlopen` at runtime — the same loader that NDK-C Vulkan apps use. No Android-specific support is required.

```rust
// File: src/vulkan_init.rs — ash Vulkan on Android
use ash::{vk, Entry, Instance};

pub fn create_vulkan_instance() -> (Entry, Instance) {
    // Loads /system/lib64/libvulkan.so via dlopen
    let entry = unsafe { Entry::load().expect("failed to load Vulkan") };

    let app_info = vk::ApplicationInfo::default()
        .application_name(c"MyApp")
        .application_version(vk::make_api_version(0, 1, 0, 0))
        .api_version(vk::API_VERSION_1_3);

    let extensions = [
        ash::khr::android_surface::NAME.as_ptr(),
        ash::khr::surface::NAME.as_ptr(),
    ];

    let create_info = vk::InstanceCreateInfo::default()
        .application_info(&app_info)
        .enabled_extension_names(&extensions);

    let instance = unsafe {
        entry.create_instance(&create_info, None)
            .expect("failed to create instance")
    };
    (entry, instance)
}
```

[Source: ash crate](https://github.com/ash-rs/ash)

**`cargo-ndk`**: Cross-compiles Rust libraries to the four Android ABI targets and sets the correct NDK toolchain environment:

```bash
# Install
cargo install cargo-ndk

# Build an AArch64 release .so targeting Android API 30
cargo ndk --target aarch64-linux-android --platform 30 build --release
# Output: target/aarch64-linux-android/release/libmygame.so

# Push to device for testing
adb push target/aarch64-linux-android/release/libmygame.so \
    /data/local/tmp/libmygame.so
```

Android Studio (via AGP 8.4+) supports Rust natively for Android library modules through a `cargo` block in `build.gradle.kts`, integrating `cargo-ndk` into the standard build pipeline without manual CI scripting. [Source: cargo-ndk](https://github.com/bbqsrc/cargo-ndk)

**`android-activity` crate**: Provides the `AndroidApp` entry-point type for fully-native Rust Android apps (the Rust equivalent of `android_native_app_glue`):

```rust
// File: src/main.rs — Rust-native Android app
#[no_mangle]
fn android_main(app: android_activity::AndroidApp) {
    android_logger::init_once(
        android_logger::Config::default()
            .with_max_level(log::LevelFilter::Debug),
    );
    // Obtain ANativeWindow via app.native_window(), init Vulkan via ash …
}
```

[Source: android-activity crate](https://github.com/rust-mobile/android-activity)

### 3.3 APEX Delivery and NDK ABI Stability

APEX packages allow individual system libraries — `libvulkan.so`, ANGLE, codec2, `libGLES*` — to be updated via Play Store without a full OTA image. **APEX delivery does not change the NDK calling convention.** The NDK public API headers (`android/`, `vulkan/`, `EGL/`) define the stable ABI that APEX-delivered libraries implement. A Rust `.so` compiled against those headers gets bug-fix updates to the underlying `libvulkan.so` automatically when Google ships an APEX update — the same guarantee C libraries have always had.

### 3.4 Expanding the android/ Header Namespace

The NDK public API surface — headers in the `android/` namespace — grows with each API level:

| Android API | Notable `android/` additions |
|---|---|
| 26 (Oreo) | `AHardwareBuffer`, `ANativeWindow` formalised |
| 29 (Q) | `AImageDecoder`, `ANeuralNetworks` stable |
| 30 (R) | `APerformanceHint`, game-mode hints |
| 33 (T) | `ASurfaceControl` stable, `AMediaCodec` extensions |
| 35 (V) | `android/thermal.h` improvements, `AChoreographer` extensions |

The `ndk-sys` crate tracks these additions via auto-generated bindings, so Rust NDK consumers gain new headers as they are formalised.

### 3.5 What Will Not Replace the NDK

**Project Panama FFM**: As established in Section 2, FFM is a HotSpot-internal not present in ART.

**WebAssembly / WebGPU on Android**: WASM modules run inside a sandbox and cannot obtain a `VkDevice` or `AHardwareBuffer` directly from the Android kernel ABI. WASM runtimes that run natively on Android (WAMR, wasmtime) are themselves NDK consumers — WASM becomes one more library loaded via the NDK, not a replacement.

**Kotlin/Native**: Kotlin Multiplatform compiles shared Kotlin code to native binaries. On iOS, Kotlin/Native runs its own runtime. On Android, KMP targets compile to Dalvik/ART bytecode — the Kotlin/Native runtime is not used on Android. KMP Android targets still run on ART and still use JNI for native interop.

**Flutter Dart FFI**: Dart FFI (`dart:ffi`) allows Dart code to call C functions. On Android, Flutter itself is a native library loaded into an Activity's thread pool via the NDK. Dart FFI calls still cross the same NDK ABI boundary. Dart FFI is an ergonomics improvement for Flutter apps, not an NDK replacement.

### 3.6 NDK Trajectory

| Horizon | NDK evolution |
|---|---|
| Near-term (2026–2027) | `ndk` Rust crate stabilises to 1.0; more `android/` headers formalised; AGP Rust plugin matures; Rust in new AOSP drivers (Nova GPU kernel driver in staging) |
| Medium-term (2027–2029) | New Google platform code defaults to Rust where applicable; C++ NDK enters "maintained but not preferred for new code" posture internally; `ndk` crate covers full public NDK surface |
| Long-term (2030+) | NDK ABI stable indefinitely; Rust and C/C++ coexist; the practical question is which language to use for new native code, not whether the NDK survives |

[Source: Android Rust overview](https://source.android.com/docs/setup/build/rust/building-rust-modules/overview), [Google security blog: Memory-safe languages in Android 13](https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html)

---

## 4. Integrations

**Ch85 — Android SurfaceFlinger**: The `ANativeWindow` described in Section 2.6 is the producer end of SurfaceFlinger's `BufferQueue`. The full consumer-side pipeline — how buffers flow from `vkQueuePresentKHR` through the Binder IPC into the compositor — is covered in Ch85.

**Ch86 — Vulkan on Android**: Ch86 covers the Vulkan driver stack, ANGLE, and Android-specific Vulkan extensions in detail. This chapter extracts the ART/JNI/NDK material that was formerly §13–14 of Ch86, so the two chapters are companions: Ch86 focuses on the GPU side of the API surface; this chapter focuses on the managed-runtime side.

**Ch87 — Android AR, ARCore, and Meta Horizon OS**: The Meta Quest platform uses an Android 14-based runtime. The ART lifecycle described here applies directly to Quest apps; OpenXR apps that call the Meta OpenXR SDK cross the same JNI boundaries at startup before dropping to native for the frame loop.

**Ch161 — Android Game Development Kit (AGDK)**: Ch161 covers the full AGDK library set — GameActivity, Paddleboat, Swappy, Oboe, Android GPU Inspector — in depth. Section 2.6 of this chapter covers the NativeActivity→GameActivity transition and ANativeWindow; Ch161 extends the same APIs with frame-pacing, audio, and performance tooling.

**Ch4 — GPU Memory Management**: ART's Concurrent Copying GC interacts with `AHardwareBuffer` and `VkDeviceMemory` only at the JNI boundary — native memory is not GC-managed. The broader GPU memory topology (unified memory, lazy allocation) is covered in Ch4.

**Ch75 — Vulkan Memory Allocator (VMA)**: `VMA_MEMORY_USAGE_GPU_LAZILY_ALLOCATED` on Android is particularly relevant on tile-based deferred renderers. VMA is described in Ch75; this chapter establishes the ABI and runtime environment in which it runs.

**Ch34 — ANGLE**: ANGLE's promotion to the default system GLES on Android 17+ (mentioned in Section 2.6) is grounded in ANGLE's Vulkan backend architecture, which Ch34 covers in full.

**Ch26 — Binder IPC**: The Zygote fork model (Section 1.6) depends on Binder for the Service Manager discovery that newly forked processes use. The non-SDK interface restrictions (Section 1.8) are also enforced partly through Binder service calls. Ch26 covers the Binder driver and protocol.

---

## Roadmap

### Near-term (6–12 months)
- **`ndk` Rust crate 1.0**: The `ndk` crate (safe Rust NDK wrappers) has been in pre-1.0 status since 2020. API stabilization tracking work in [github.com/rust-mobile/ndk](https://github.com/rust-mobile/ndk) is converging on a 1.0 milestone that pins the safe wrapper surface against NDK API 26+ and provides SemVer stability guarantees. `ndk-sys` auto-generation from NDK headers is automated via `bindgen`; the 1.0 cut will lock to NDK r27 LTS headers.
- **userfaultfd GC default expansion**: The userfaultfd-based A-GC compactor (Android 13+) is enabled by default on GKI 5.10+ devices. Android 16 extends the policy to enable A-GC by default on all devices with ≥6 GB RAM, regardless of kernel version, using the Android 16 backport of `userfaultfd` to GKI 5.4. The practical effect: more devices eliminate Baker read-barrier overhead in steady-state operation.
- **ART as fully updatable Mainline module**: ART has shipped as part of Android Mainline since Android 12, but the AOT compilation tier (`dex2oat`) still runs at OTA time. Android 16 targets fully on-demand `dex2oat` recompilation triggered by Play Store ART module updates — meaning ART GC improvements and JIT optimisations reach all devices on Android 12+ without waiting for OEM OTA schedules.
- **Non-SDK interface restrictions tightening (API 36)**: Each Android release moves more `@hide` symbols from the greylist to the blacklist. Android 16 (API 36) blacklists several `SurfaceFlinger` and `GraphicBuffer` internal methods that were still accessible in API 35 grey; games using reflection to access internal `Surface` state for screenshot or texture-sharing hacks need to migrate to `ASurfaceControl` (NDK, API 29+).
- **AGP 9 Rust native library support GA**: Android Gradle Plugin 9.0 is expected to graduate the `cargo` block for Rust native library modules from experimental to stable, integrating `cargo-ndk` into the standard `./gradlew assembleRelease` pipeline with full Play App Signing and bundle support.

### Medium-term (1–3 years)
- **ART GC pause reduction via concurrent root scanning**: ART's Concurrent Copying GC still requires a brief stop-the-world phase to flip the from/to-space sense and scan GC roots. A concurrent root-scanning prototype (similar to ZGC's colored pointer approach) is in the ART research queue; if it lands, the last STW pause in the CC GC would shrink from ~1–3 ms to sub-millisecond on large heaps, reducing jank in games with large managed object graphs (Kotlin coroutine frames, Unity managed scripting).
- **`android/` NDK header namespace expansion**: API 36–38 are expected to formalise several currently `@hide` surface control, display, and thermal APIs as stable NDK headers — `android/surface_control.h` extensions for multi-plane composition, `android/thermal.h` v2 with per-core thermal headroom, and `android/hardware_buffer_ycbcr.h` for explicit YCbCr plane layout. The `ndk-sys` crate tracks these additions automatically via bindgen.
- **APEX-delivered ART JIT profile improvements**: As ART's JIT profile recording and Cloud Profile delivery (§1.3) mature, Google is expanding the profile schema to include call-site type information (inline cache records) in the distributed `.prof` format — allowing `dex2oat speed-profile` compilation to devirtualize more call sites ahead-of-time, reducing JIT warmup time for cold-start frames in games.
- **Kotlin Multiplatform + ART alignment**: KMP on Android currently targets ART bytecode (not Kotlin/Native), meaning all KMP Android shared code still runs on ART with JNI for native interop. As KMP matures, the toolchain is expected to generate ART-optimised DEX for shared Kotlin modules — leveraging ART intrinsics for `kotlin.math.*` and `kotlin.text.*` rather than the JVM-generic bytecode sequences the current `kotlinc` JVM target emits.

### Long-term
- **Full Rust NDK surface coverage**: The `ndk` crate roadmap targets covering the entire stable `android/` header namespace by `ndk` v2.0 (estimated 2028+): `AMediaCodec`, `ANeuralNetworks`, `AImageDecoder`, `APerformanceHint`, `ASurfaceControl`, and `AChoreographer` safe wrappers. At that point, a pure-Rust Android app can use all stable NDK APIs without any `unsafe` FFI boundary except for the Vulkan and EGL surface, which remain in `ash`/`khronos-egl`.
- **ART WebAssembly (WasmGC) runtime**: The JVM ecosystem is exploring WasmGC (WebAssembly Garbage Collection, W3C standard 2023) as a compilation target for managed languages. If Google adopts WasmGC as a second execution tier in ART — running Wasm modules via ART's GC and JIT rather than a separate sandbox — it would allow WASM components (e.g., game business logic compiled from Rust or C++) to interop with ART-managed code via a shared object model, closing the current NDK/JNI boundary for intra-process Wasm↔Kotlin calls.
- **Non-JVM Android apps (Zygote-less startup)**: Google's internal experiment "Android Without Java" (referenced in AOSP internal discussions) explores whether new device categories (IoT, automotive embedded) could run Android apps that bypass the Zygote fork and ART boot image entirely, starting from a native executable linked against a minimal Android API surface. The NDK ABI would remain the stable boundary; the JVM would be optional rather than mandatory. This is long-horizon work with no public timeline, but the Rust platform services trajectory (§3) is the precondition.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
