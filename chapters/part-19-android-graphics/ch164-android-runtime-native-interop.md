# Chapter 164: Android Runtime and Native Interop: ART, JNI, and the NDK

**Target audiences:**
- **Systems and driver developers** — need to understand how Android's managed runtime interacts with native code
- **Graphics application developers** — writing Vulkan or OpenGL ES apps that cross the Java/native boundary
- **Browser and web platform engineers** — understanding how Chrome, WebView, and ANGLE fit into Android's runtime model

## Table of Contents

0. [Jetpack and AndroidX: The Modern Android Library Ecosystem](#0-jetpack-and-androidx-the-modern-android-library-ecosystem)
1. [ART and Dalvik: Where They Diverge from the JVM](#1-art-and-dalvik-where-they-diverge-from-the-jvm)
   - 1.0 [Historical Background: Dalvik and the Road to ART](#10-historical-background-dalvik-and-the-road-to-art)
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
   - 2.7 [JNI Reference Management: Local, Global, and Weak References](#27-jni-reference-management-local-global-and-weak-references)
   - 2.8 [CMake Toolchain for Android NDK: Multi-ABI Builds and Prefab](#28-cmake-toolchain-for-android-ndk-multi-abi-builds-and-prefab)
   - 2.9 [dlopen, Linker Namespaces, and Android's Library Isolation Model](#29-dlopen-linker-namespaces-and-androids-library-isolation-model)
   - 2.10 [C++ Standard Library ABI: libc++_shared vs libc++_static](#210-c-standard-library-abi-libc_shared-vs-libc_static)
   - 2.11 [Signal Handling in Native Code: Async Safety and ANR Avoidance](#211-signal-handling-in-native-code-async-safety-and-anr-avoidance)
   - 2.12 [C++ Exceptions Across the JNI Boundary](#212-c-exceptions-across-the-jni-boundary)
   - 2.13 [Memory Sanitizers on Android: ASan, HWASan, UBSan, and MTE](#213-memory-sanitizers-on-android-asan-hwasan-ubsan-and-mte)
   - 2.14 [Native Crash Analysis: Tombstones, Unwinding, and Symbolication](#214-native-crash-analysis-tombstones-unwinding-and-symbolication)
3. [The NDK's Future: Rust as Co-equal Language](#3-the-ndks-future-rust-as-co-equal-language)
   - 3.1 [The NDK Is Not Going Away](#31-the-ndk-is-not-going-away)
   - 3.2 [The Rust NDK Ecosystem](#32-the-rust-ndk-ecosystem)
   - 3.3 [APEX Delivery and NDK ABI Stability](#33-apex-delivery-and-ndk-abi-stability)
   - 3.4 [Expanding the android/ Header Namespace](#34-expanding-the-android-header-namespace)
   - 3.5 [What Will Not Replace the NDK](#35-what-will-not-replace-the-ndk)
   - 3.6 [NDK Trajectory](#36-ndk-trajectory)
   - 3.7 [The Security Motivation: Memory Safety and Android Vulnerabilities](#37-the-security-motivation-memory-safety-and-android-vulnerabilities)
   - 3.8 [JNI from Rust: The `jni` Crate](#38-jni-from-rust-the-jni-crate)
   - 3.9 [Rust in AOSP: Platform Components and the Soong Build System](#39-rust-in-aosp-platform-components-and-the-soong-build-system)
   - 3.10 [wgpu: Cross-Platform Graphics in Rust on Android](#310-wgpu-cross-platform-graphics-in-rust-on-android)
4. [Integrations](#4-integrations)

---

## 0. Jetpack and AndroidX: The Modern Android Library Ecosystem

Before examining the runtime internals, it is worth orienting the reader on the library layer that application developers interact with most directly: **Jetpack** and **AndroidX**. Understanding this layer clarifies why the chapter references libraries like `androidx.games.activity` (GameActivity), `androidx.camera.core` (CameraX), and AGDK Prefab AARs — and how they relate to the platform APIs and runtime described in §§1–3.

### 0.1 The Support Library Problem

Android's early extensibility mechanism was the **Android Support Library** — a collection of backport and utility classes that Google shipped outside the OS update cycle. By 2017 the Support Library had accumulated version confusion (`v4`, `v7`, `v13`, `v14`... `28.0.x`), overlapping APIs, and a naming scheme that no longer reflected its scope. Apps routinely depended on incompatible revisions of the same library pulled in by different dependencies.

### 0.2 AndroidX and the Jetpack Relaunch

At Google I/O 2018, Google announced **Jetpack** — an umbrella brand for all first-party Android libraries going forward — alongside **AndroidX**, a new Maven coordinate and package namespace (`androidx.*`) that replaced the entire `android.support.*` hierarchy. The migration was a hard cut: Support Library 28.x was the final release; all subsequent development moved to AndroidX. The `androidx.*` namespace is semantically versioned independently of the Android OS, allowing libraries to ship updates to devices running Android 5.0+ without waiting for OS releases. [Source: Android developers blog — Hello World, AndroidX](https://android-developers.googleblog.com/2018/05/hello-world-androidx.html)

The migration tool (`./gradlew migrateToAndroidX`) rewrites import statements and manifest entries in existing projects. Third-party libraries that depend on Support Library are mapped to their AndroidX equivalents via the `jetifier` Gradle plugin, which rewrites bytecode at build time — a temporary bridge that Google has signalled will eventually be retired as the ecosystem fully migrates.

### 0.3 Jetpack Libraries Relevant to Graphics and NDK Development

Jetpack covers a broad surface. The libraries most directly relevant to the topics in this chapter are:

| Library | AndroidX coordinates | Relevance |
|---|---|---|
| **GameActivity** | `androidx.games:games-activity` | Replaces NativeActivity for NDK apps (§2.6); ships as Prefab AAR |
| **Games Performance** (Swappy) | `androidx.games:games-frame-pacing` | Vulkan/GLES frame pacing (ch161 §8) |
| **Games Controller** (Paddleboat) | `androidx.games:games-controller` | Gamepad input in NDK (ch161 §4) |
| **Games Memory Advice** | `androidx.games:games-memory-advice` | On-device memory pressure signals (ch161 §10) |
| **CameraX** | `androidx.camera:camera-core` | Lifecycle-aware Camera2 wrapper; used with MediaPipe (ch191 §10) |
| **Lifecycle** | `androidx.lifecycle:lifecycle-runtime-ktx` | ViewModel + LiveData; governs `ArSession` pause/resume (ch87 §3) |
| **Compose** | `androidx.compose.ui:ui` | Declarative UI compiled to standard DEX, runs on ART |
| **Core KTX** | `androidx.core:core-ktx` | Kotlin extension functions over framework APIs |
| **DataStore** | `androidx.datastore:datastore-preferences` | Async preferences replacement for SharedPreferences |
| **WorkManager** | `androidx.work:work-runtime-ktx` | Background tasks; used for deferred ML model updates (ch191 §15) |

**AGDK libraries as Jetpack AARs.** The Android Game Development Kit libraries (GameActivity, Swappy, Paddleboat, Oboe, Memory Advice) are distributed as Jetpack AARs containing Prefab packages — a mechanism that packages pre-built native `.so` files and headers inside an `.aar` for consumption by `find_package()` in CMake (§2.8). This makes them first-class Jetpack citizens even though their primary interface is C/C++, not Kotlin.

### 0.4 Jetpack Compose and ART

Jetpack Compose compiles Kotlin `@Composable` functions to ordinary DEX bytecode — there is no separate runtime or interpreter. The Compose compiler plugin (a Kotlin compiler plugin, not a separate tool) transforms `@Composable` annotated functions at compile time, inserting `Composer` parameter threading and group tracking calls. At runtime, Compose's diffing and recomposition logic runs entirely inside ART, subject to the same GC, JIT, and compilation tier mechanics described in §1. The `remember {}` primitive maps directly to a slot table maintained in the composition's node tree — not to any platform-specific memory mechanism.

### 0.5 The Release Cadence Advantage

Because AndroidX libraries ship on Maven independent of Android OS releases, a library like GameActivity can add a new API for Android 14 features and ship it to apps running on Android 8.0+ the same week. Apps consuming Jetpack libraries via `implementation("androidx.games:games-activity:3.0.0")` get bugfixes and features on the Google Maven cadence (~monthly), not the Android OS release cadence (~annual). This decoupling is Jetpack's primary engineering value proposition over directly calling `android.*` platform APIs.

[Source: Jetpack release notes](https://developer.android.com/jetpack/androidx/releases)

---

## 1. ART and Dalvik: Where They Diverge from the JVM

Android Runtime (ART) is not a JVM. It shares Java syntax and Java's standard library API surface but diverges at every level of implementation: bytecode format, compilation model, garbage collector, bootstrap strategy, and interpreter architecture. Understanding these divergences matters directly for graphics developers: the GC model explains why `@CriticalNative` must be short, the compilation tier ladder explains warm-up jank, and the Zygote model explains the memory layout every Android process inherits.

### 1.0 Historical Background: Dalvik and the Road to ART

**Dalvik** was Android's original managed runtime, created by Dan Bornstein at Google and shipping with Android 1.0 in September 2008. The name comes from the fishing village of Dalvík in Iceland — Bornstein's family heritage. [Source: Wikipedia — Dalvik]

Dalvik was designed around a hard constraint that no longer applies to modern Android: devices with 256 MB of RAM running many apps simultaneously, with no swap space. Three design decisions followed directly from this constraint:

1. **DEX consolidation**: where the JVM compiles each `.java` file to a separate `.class` file, the Android build toolchain (`dx`, later `d8`) compiles all of an app's classes into a *single* `.dex` file. A single string pool and constant pool across the entire app reduces storage compared to the JVM's per-class constant pools — significant when flash storage was measured in hundreds of megabytes.
2. **Register-based bytecode** (detailed in §1.1): fewer bytecode instructions per method means smaller `.dex` files and faster interpreter dispatch.
3. **Zygote process**: a pre-warmed Android process (`/system/bin/app_process`) that `fork()`s new app processes, sharing read-only pages of the framework class libraries across all apps via copy-on-write. Described in §1.6.

**Dalvik JIT (Android 2.2 Froyo, May 2010).** Dalvik's original interpreter was purely interpretive — every bytecode instruction dispatched through a switch table. Android 2.2 added a **trace-based JIT**: rather than compiling whole methods (as HotSpot does), Dalvik's JIT identified frequently-executed *traces* — linear sequences of bytecodes across method boundaries, including branches that turned out to be always-taken — and compiled those traces to native ARM code at runtime. This was faster than interpretation for hot paths but left cold code and complex control flow uncompiled. [Source: Dalvik JIT paper — Google I/O 2010]

**Dalvik's GC: stop-the-world copying.** Dalvik's garbage collector was a concurrent mark-sweep with stop-the-world collection pauses of 50–200 ms. These pauses were the primary cause of the "Android is janky" perception during the 2.x/3.x era — a 16 ms frame budget at 60 fps is violated by a single GC pause. The 4.x era brought incremental GC improvements but the fundamental architecture remained.

**ART preview (Android 4.4 KitKat, October 2013).** Google introduced ART as a developer-accessible option in KitKat, off by default. ART's defining architectural shift was **AOT (ahead-of-time) compilation**: `dex2oat` compiled the entire `.dex` file to an `.oat` file (an ELF shared object containing native ARM64 code) at **install time**. On first run, ART executed the pre-compiled native code directly with no JIT warm-up. The cost was a longer install time (~2–5× for large apps) and additional storage for the `.oat` file.

**ART as default (Android 5.0 Lollipop, November 2014).** Dalvik was removed entirely. Every app ran under ART. The install-time AOT compilation made apps feel faster on first launch but increased install times and consumed storage — on devices with 8–16 GB flash, `.oat` files for all installed apps could consume 1–2 GB.

**Hybrid JIT+AOT (Android 7.0 Nougat, August 2016).** Google reconsidered full AOT in response to storage and install-time complaints. Nougat introduced a **tiered hybrid system**: apps are installed without upfront `dex2oat`; a new JIT compiler (method-based, not trace-based) runs during execution and records which methods are hot into a **profile file** (`.prof`). A background `dex2oat` daemon runs when the device is idle/charging and AOT-compiles only the hot methods indicated by the profile. This model — profile-guided selective AOT — became the foundation of all subsequent ART versions and is described in depth in §§1.2 and 1.3.

**Key milestones from Android 8.0 onward:**

| Android version | Year | ART improvement |
|---|---|---|
| 8.0 Oreo | 2017 | Cloud profiles (§1.3); `dex2oat` speed improvements; Vdex (verified DEX) |
| 9.0 Pie | 2018 | App startup improvements; generational GC experiments |
| 10 | 2019 | Concurrent copying GC (concurrent moving phases); reduced GC pause time |
| 11 | 2020 | ART APEX (ART updateable via Mainline without OS update) |
| 12 | 2021 | `nterp` switch-dispatch interpreter (§1.7) replacing the old `mterp` |
| 13 | 2022 | `userfaultfd`-based GC (§1.5); further GC pause reduction |
| 14 | 2023 | ART fully Mainline-updateable; profile sharing improvements |
| 15 | 2024 | Improved deoptimisation pipeline; Kotlin-specific JIT optimisations |

**Dalvik's legacy.** The DEX bytecode format, the `.dex` consolidation, and the Zygote fork model were all Dalvik inventions that ART inherited intact. ART replaced Dalvik's runtime engine (interpreter, JIT, GC) while preserving its bytecode and process architecture. Developers writing pure Kotlin or Java today have no direct awareness of Dalvik — but the `.dex` file their app ships is a direct descendant of the format Bornstein designed in 2007.

[Source: AOSP — ART and Dalvik](https://source.android.com/docs/core/runtime)

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

### 2.7 JNI Reference Management: Local, Global, and Weak References

Every `jobject`, `jclass`, `jstring`, `jarray`, or derived type returned from a JNI call is a **reference** — a handle into the ART heap, not a raw C pointer. ART manages three reference lifetimes, and confusing them is the most common source of memory corruption, native crashes, and subtle memory leaks in JNI code.

#### Local References

The default type produced by most JNI calls is a **local reference**. It is valid for exactly the duration of the native method that created it — once the JNI frame returns to Java, ART frees all local references automatically. Most JNI functions that return object types produce local refs: `FindClass`, `NewObject`, `GetObjectField`, `NewStringUTF`, `NewByteArray`, and so on. Applications can also create them explicitly with `NewLocalRef(env, obj)` and release them early with `DeleteLocalRef(env, ref)` to free the handle slot.

Local references are stored in a per-frame **local reference table** whose default capacity is **512 entries** per JNI call frame. If a native method creates more than 512 local references without deleting any, ART terminates the process with `JNI ERROR (app bug): local reference table overflow (max=512)` visible in logcat. The fix is to either call `DeleteLocalRef` inside loops that create many objects, or to call `EnsureLocalCapacity(env, n)` or `PushLocalFrame(env, n)` / `PopLocalFrame(env, result)` to pre-allocate additional capacity. The frame-based pair is useful when a native helper function needs to create a temporary burst of objects and guarantees cleanup even on early-return paths.

#### Global References

A **global reference** persists until the native code explicitly deletes it with `DeleteGlobalRef(env, ref)`. It is created from any existing reference via `NewGlobalRef(env, obj)` and is valid across JNI calls, across threads, and after the creating JNI frame has returned. ART's global reference table holds up to **51200 entries** by default. Global refs are the correct choice whenever a native object (a C++ class member, a singleton, a renderer) needs to hold a reference to a Java object between frames — for example, caching a `jobject` callback handle that the render loop will invoke.

The critical discipline: **every `NewGlobalRef` must have a matching `DeleteGlobalRef` when the cached reference is no longer needed**. The most common leak pattern is storing a global ref for a listener or callback object in `JNI_OnLoad` and forgetting to delete it in a teardown path. While ART does not unload individual classes at runtime in typical app usage, plugin-style architectures that load DEX at runtime via custom class loaders can encounter class unloading, at which point an uncollected global ref keeps the class — and its entire ClassLoader — alive indefinitely.

#### Weak Global References

**Weak global references** (`NewWeakGlobalRef` / `DeleteWeakGlobalRef`) are similar to global refs in lifetime scope, but ART's GC is allowed to clear them when no strong references remain. Before dereferencing a weak ref, always verify it is still alive:

```cpp
jweak weak_callback = env->NewWeakGlobalRef(callback_obj);
// ... later, when invoking:
if (env->IsSameObject(weak_callback, NULL)) {
    // GC has cleared the referent — the Java object was collected
    return;
}
jobject local = env->NewLocalRef(weak_callback);
env->CallVoidMethod(local, on_event_method_id, event_data);
env->DeleteLocalRef(local);
```

The `IsSameObject(env, ref, NULL)` call is the canonical liveness check. Skipping it and calling methods on a cleared weak ref produces undefined behaviour. Weak refs are the correct choice for listener and observer patterns where the native code should not prevent the Java side from being garbage collected — for example, a native audio callback that calls back into an `Activity` only if the Activity is still alive.

#### String Handling

`GetStringUTFChars(env, jstr, &isCopy)` returns a C `const char*` in **modified UTF-8** — not standard UTF-8. The difference: null bytes (`\0`) are encoded as the two-byte sequence `0xC0 0x80` so that the string remains null-terminated in the C sense, and Unicode supplementary characters (code points above U+FFFF) are encoded as CESU-8 six-byte pairs rather than standard UTF-8's four-byte sequence. Code that blindly treats this output as standard UTF-8 and passes it to a UTF-8 validator or a protocol encoder will produce malformed data for strings containing emoji or other supplementary characters.

For round-tripping arbitrary Unicode without encoding concerns, prefer `GetStringChars(env, jstr, &isCopy)`, which returns raw UTF-16 code units. The `isCopy` out-parameter signals whether ART returned a direct pointer to the internal heap string data (`JNI_FALSE`) or allocated a copy (`JNI_TRUE`). Either way, the corresponding release call — `ReleaseStringUTFChars` or `ReleaseStringChars` — is mandatory. When `isCopy == JNI_TRUE`, omitting the release leaks the copy. When `isCopy == JNI_FALSE`, omitting the release causes ART to keep the heap string pinned (unpinned only on release), interfering with the GC's ability to compact it.

`GetStringLength` returns the length in UTF-16 code units; `GetStringUTFLength` returns the byte count of the modified UTF-8 representation. Neither includes the null terminator.

#### Array Pinning and Zero-Copy Access

`GetIntArrayElements(env, arr, &isCopy)` either pins the ART heap array and returns a direct pointer (`isCopy == JNI_FALSE`) or copies the array to a native heap buffer (`isCopy == JNI_TRUE`). `ReleaseIntArrayElements(env, arr, ptr, mode)` takes one of three mode constants: `0` copies the data back and unpins, `JNI_COMMIT` copies back but keeps the array pinned (used to flush intermediate updates without ending access), and `JNI_ABORT` discards any modifications and unpins — the right choice for read-only access.

`GetPrimitiveArrayCritical(env, arr, &isCopy)` is the zero-copy API. It is the most likely call to return a direct pointer to the ART heap array without copying. The contract is severe: while holding a critical pointer, the **GC is suspended** and no other JNI calls are permitted. The critical region must be as short as possible — the ART guidelines say under 100 µs. Any heap allocation, any `pthread_mutex_lock` that another thread might hold during GC, or any blocking system call inside the critical section risks a deadlock or a severe GC pause spike. `GetByteArrayRegion` / `SetByteArrayRegion` copy a specified range without pinning at all; they are safer for small arrays or when the GC-stop risk of the critical API is not acceptable.

#### ExceptionCheck After Every Call

Any JNI function that involves Java execution — `CallVoidMethod`, `NewObject`, `FindClass`, `GetFieldID` — can leave a pending Java exception in the `JNIEnv`. ART does not propagate these exceptions automatically; native code continues executing. Failing to check `ExceptionCheck(env)` after such calls and then making another JNI call with a pending exception produces undefined behaviour. The minimum safe pattern:

```cpp
env->CallVoidMethod(obj, method_id, arg);
if (env->ExceptionCheck()) {
    env->ExceptionDescribe();  // logs the stack trace to logcat
    env->ExceptionClear();
    return;  // or ThrowNew a different exception
}
```

Detailed exception-to-C++ interop — including `ThrowNew`, conversion of C++ exceptions to Java exceptions, and the semantics of C++ exceptions propagating through JNI frames — is covered in Section 2.12.

[Source: Android JNI Tips](https://developer.android.com/training/articles/perf-jni), [JNI reference types](https://docs.oracle.com/en/java/docs/books/jni/html/refs.html)

---

### 2.8 CMake Toolchain for Android NDK: Multi-ABI Builds and Prefab

The Android NDK ships a CMake toolchain file that configures the cross-compiler, sysroot, and linker flags required to build native code for Android targets. Understanding it is essential for any NDK project that goes beyond the default Android Studio defaults.

#### The Toolchain File

The toolchain file lives at `${ANDROID_NDK}/build/cmake/android.toolchain.cmake`. It sets the cross-compiler triple (e.g. `aarch64-linux-android26-clang++`), the sysroot to the NDK's per-API-level system headers, and the link script for the target ABI. Three CMake variables control the build:

```cmake
set(ANDROID_ABI arm64-v8a)       # also: armeabi-v7a, x86_64, x86
set(ANDROID_PLATFORM android-26) # minimum API level for this .so
set(ANDROID_STL c++_shared)      # STL linkage: c++_shared, c++_static, or none
```

`ANDROID_ABI` selects the target ISA. `ANDROID_PLATFORM` sets both the minimum API level that `dlopen` requires and the sysroot generation — the NDK will only expose headers and stubs for APIs available at that level or below. `ANDROID_STL` controls which C++ Standard Library variant is linked (see Section 2.10 for the trade-offs).

#### AGP Integration

Android Gradle Plugin (AGP) integrates CMake when `externalNativeBuild.cmake.path` is set in `build.gradle.kts`. AGP drives CMake automatically: it selects the NDK version from `ndkVersion`, runs CMake configure for each ABI, and `make` (or ninja) to produce the `.so` files, which AGP then packages into the APK under `lib/<abi>/`.

```kotlin
// build.gradle.kts
android {
    ndkVersion = "27.2.12479018"  // r27 LTS
    defaultConfig {
        externalNativeBuild { cmake { path = "CMakeLists.txt" } }
        ndk { abiFilters += listOf("arm64-v8a", "armeabi-v7a") }
    }
    externalNativeBuild { cmake { path = "CMakeLists.txt" } }
}
```

The `abiFilters` list controls which ABIs are compiled. Shipping both `arm64-v8a` and `armeabi-v7a` covers essentially all active Android devices. Adding `x86_64` is recommended for emulator testing and Chrome OS; `x86` can be omitted for new projects as 32-bit x86 Android device share is negligible.

#### Per-ABI Compile Options

When a single build needs different code paths per ABI, `CMakeLists.txt` can branch on `ANDROID_ABI`:

```cmake
if(ANDROID_ABI STREQUAL "arm64-v8a")
    target_compile_options(mygame PRIVATE
        -march=armv8.2-a+fp16     # fp16 arithmetic
        -DNEON_ENABLED=1
    )
elseif(ANDROID_ABI STREQUAL "armeabi-v7a")
    target_compile_options(mygame PRIVATE
        -mfpu=neon-vfpv4
        -DNEON_ENABLED=1
    )
endif()
```

Care is needed with ARMv8.2 extensions: `-march=armv8.2-a+fp16` enables dot-product and half-precision float intrinsics that are absent on early ARMv8.0 devices (Cortex-A53, Snapdragon 625). At runtime, query `android_getCpuFamily()` + `android_getCpuFeatures()` (from `<cpu-features.h>`, NDK API 9+) or read `/proc/cpuinfo` before dispatching to the optimised path.

#### Stripping and Symbol Files

AGP automatically strips `.so` files before packaging into the release APK. Unstripped copies are retained in `build/intermediates/merged_native_libs/<variant>/<abi>/` for upload to crash reporting services. The CMake variable `CMAKE_BUILD_TYPE=RelWithDebInfo` retains DWARF debug info in the build-tree `.so` without affecting the stripped APK copy. Setting `android.defaultConfig.packaging.jniLibs.keepDebugSymbols.add("**/*.so")` in `build.gradle.kts` includes unstripped `.so` in a debug APK — useful for on-device symbolication during development but dramatically increases APK size.

#### Prefab: Pre-built Libraries in AAR Files

**Prefab** is the Android packaging mechanism for distributing pre-built native libraries and their C/C++ headers inside `.aar` files, letting Gradle-based consumers link against them without managing include paths or linker flags manually. All major AGDK components — GameActivity, Swappy (Frame Pacing), Oboe (audio), Paddleboat (gamepad) — ship as Prefab AAR packages on Maven Central.

A Prefab AAR contains a `prefab/` directory with per-module `include/` headers and per-ABI `.so` or `.a` files. AGP 7.0+ processes Prefab AARs automatically: when a CMake project calls `find_package`, AGP generates a CMake package config file (`<module>Config.cmake`) that exposes the headers and library as an IMPORTED target.

```cmake
# CMakeLists.txt — consume AGDK GameActivity via Prefab
find_package(game-activity REQUIRED CONFIG)
find_package(games-frame-pacing REQUIRED CONFIG)

target_link_libraries(mygame
    game-activity::game-activity
    games-frame-pacing::swappy_static
    android  log  vulkan)
# CMake automatically adds -I<prefab_headers> and links the correct ABI .a/.so
```

```kotlin
// build.gradle.kts — declare Prefab AAR dependencies
dependencies {
    implementation("androidx.games:games-activity:3.0.5")
    implementation("androidx.games:games-frame-pacing:2.1.4")
}
android { buildFeatures { prefab = true } }
```

The `buildFeatures.prefab = true` flag enables AGP's Prefab processing. The `find_package` call then resolves to the correct ABI variant at configure time. Without Prefab, distributing C++ libraries required either source distribution or a manual `jniLibs`-path arrangement that broke with NDK updates — Prefab makes `find_package` as ergonomic for Android as it is on desktop CMake.

#### C++ Standard Version Support

The NDK has used **Clang exclusively** since NDK r18 (2018) — GCC support was removed entirely. The Clang version tracks LLVM releases: NDK r23 shipped Clang 12, r25 shipped Clang 14, r26 shipped Clang 17, r27 shipped Clang 18, r28 shipped Clang 19. This matters because C++ standard support is determined by the Clang version bundled with the NDK, not by the Android API level of the target device.

**Setting the standard in CMake:**

```cmake
# Global (all targets in the project):
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)   # use -std=c++17, not -std=gnu++17

# Per-target (preferred for mixed-standard projects):
set_target_properties(mygame PROPERTIES
    CXX_STANDARD 20
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
)
```

`CMAKE_CXX_EXTENSIONS OFF` is important: without it CMake passes `-std=gnu++17` (GNU extensions enabled) rather than `-std=c++17`. GNU extensions include VLAs, `__int128`, and statement expressions — technically non-standard. Most Android NDK code works correctly with either, but setting `OFF` enforces portable standard C++ and avoids surprises when porting to other platforms.

**Standard version support matrix:**

| C++ standard | NDK support status | Notes |
|---|---|---|
| C++14 | Fully supported (NDK r13+) | Default before r21; all features present |
| C++17 | Fully supported (NDK r21+, Clang 9+) | **Recommended default**; `if constexpr`, `std::optional`, `std::variant`, structured bindings, fold expressions, `std::string_view` |
| C++20 | Broadly supported (NDK r23+, Clang 12+) | Language features: concepts, ranges, coroutines, designated initialisers, `std::span`, `std::bit_cast`; **modules not supported** (see below) |
| C++23 | Partially supported (NDK r27+, Clang 18+) | `std::expected`, `std::print`, `std::flat_map`, `std::flat_set`; library completeness lags language features |

**C++20 features in graphics code.** Several C++20 additions are directly useful for Android graphics and NDK work:

- **Concepts**: constrain template parameters for shader-interop types, delegate types, and SIMD wrappers without enable_if boilerplate:
  ```cpp
  template<typename T>
  concept GpuBuffer = requires(T b) {
      { b.data() } -> std::convertible_to<void*>;
      { b.size() } -> std::convertible_to<size_t>;
  };
  template<GpuBuffer B> void upload_vertices(const B& buf) { ... }
  ```
- **`std::span`**: non-owning view over contiguous data; ideal for passing vertex/index data to Vulkan `vkCmdUpdateBuffer` or LiteRT tensor buffers without copying:
  ```cpp
  void fill_vertex_buffer(std::span<const Vertex> vertices) {
      memcpy(mapped_ptr, vertices.data(), vertices.size_bytes());
  }
  ```
- **Designated initialisers**: Vulkan struct initialisation without the `{}` chain:
  ```cpp
  auto info = VkCommandPoolCreateInfo{
      .sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO,
      .flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT,
      .queueFamilyIndex = graphics_family,
  };
  ```
- **`std::bit_cast`**: type-punning without undefined behaviour; replaces `memcpy`-into-float tricks for shader constant packing.
- **Coroutines**: `co_await` / `co_yield` supported; useful for async asset loading pipelines. Requires `<coroutine>` header, present in libc++ from Clang 12+.

**C++20 modules: not supported.** Despite Clang supporting modules (`import std;`, named module units) since Clang 16, the Android NDK's CMake integration does not support C++20 modules as of NDK r28. The NDK's build system relies on traditional header inclusion, and AGP's CMake integration does not pass the module-scanning flags Clang requires. Do not use `import` syntax in NDK code targeting production — fall back to `#include`.

**`__cplusplus` macro values** for compile-time version detection:

```cpp
#if __cplusplus >= 202302L
    // C++23 path
#elif __cplusplus >= 202002L
    // C++20 path
#elif __cplusplus >= 201703L
    // C++17 path
#else
    #error "C++17 or later required"
#endif
```

**Practical recommendation.** Set `CMAKE_CXX_STANDARD 17` as the project default and opt specific targets into C++20 where concepts or `std::span` provide clear value. Avoid C++23 in production until NDK r28+ is the minimum supported NDK version in your project — library completeness across vendor toolchains that consume your `.so` (e.g., Unity native plugins, third-party SDK integrations) may lag the NDK's own Clang version.

[Source: NDK C++ support](https://developer.android.com/ndk/guides/cpp-support), [Android NDK release notes](https://github.com/android/ndk/releases)

[Source: Android NDK CMake guide](https://developer.android.com/ndk/guides/cmake), [Native dependencies with Prefab](https://developer.android.com/build/native-dependencies)

---

### 2.9 dlopen, Linker Namespaces, and Android's Library Isolation Model

Android's dynamic linker (`/system/bin/linker64`) enforces a **namespace isolation model** that restricts which shared libraries an app can open. Understanding this model is necessary for any native code that loads libraries at runtime via `dlopen`, wraps optional hardware-specific libraries, or ships plugin architectures.

#### Linker Namespace Isolation (Android 7.0+, API 24)

Before Android 7.0 (Nougat), app native code could `dlopen` arbitrary system libraries by path. This created fragility: an app relying on `/system/lib/libbinder.so` or `/system/lib/libGLES_mali.so` would break silently when the OEM changed those libraries' internal ABI between device models or Android versions. Android 7.0 introduced **linker namespace isolation** to prevent this.

Each process runs in a **default namespace** with a restricted allowed-library list. Attempting to open a private platform library by path returns NULL, and `dlerror()` reports: `dlopen failed: library "libbinder.so" not found`. The namespace configuration is defined in `/system/etc/ld.config.txt` (and vendor-partition variants `/vendor/etc/ld.config.txt`). It specifies allowed namespaces — `default`, `vndk`, `sphal` (SoC HAL), `rs` (RenderScript legacy) — and which libraries each can access. Apps run in `default`, which is limited to the NDK public library list.

#### The NDK Public Library List

The set of libraries guaranteed available to apps at any API level is the **NDK public library list**, defined in `ndk/meta/public_api.json` in the NDK distribution. It includes `libandroid.so`, `libvulkan.so`, `libEGL.so`, `libGLESv2.so`, `liblog.so`, `libz.so`, `libm.so`, `libc.so`, and a handful of others. Only these are accessible to the `default` namespace. Libraries outside this list — even if visible in `adb shell ls /system/lib64/` — cannot be opened by app code; the linker namespace blocks them.

#### dlopen in App Code

When native code needs to open a library at runtime, the correct approach is to open libraries from paths within the APK's native library directory:

```c
#include <dlfcn.h>
#include <jni.h>

// Get the native library dir from Java side and pass it to native:
// Java: getApplicationInfo().nativeLibraryDir -> String -> jstring
const char* lib_path = /* path assembled from nativeLibraryDir */;
void* handle = dlopen(lib_path, RTLD_NOW | RTLD_LOCAL);
if (!handle) {
    __android_log_print(ANDROID_LOG_ERROR, "dlopen", "%s", dlerror());
    return;
}
typedef void (*InitFn)(void);
InitFn init = (InitFn)dlsym(handle, "plugin_init");
if (init) init();
```

Play Asset Delivery can deliver `.so` files as on-demand assets to a per-app writable directory; those paths are also `dlopen`-able since they live under the app's data directory.

`RTLD_GLOBAL` vs `RTLD_LOCAL`: `RTLD_GLOBAL` exports the loaded library's symbols into the process-wide symbol table, making them visible to subsequently loaded libraries. This is occasionally required for plugin architectures where the host provides symbols that plugins depend on. However, `RTLD_GLOBAL` can cause symbol conflicts when two libraries export the same name — for example, two plugins each statically linking different versions of a codec utility, both exporting `compress()`. Prefer `RTLD_LOCAL` unless symbol visibility across `dlopen` boundaries is explicitly required.

`RTLD_NOW` vs `RTLD_LAZY`: `RTLD_NOW` resolves all undefined symbols in the library immediately at `dlopen` time; `RTLD_LAZY` (the default) defers resolution until first call. `RTLD_NOW` is useful in development to surface missing-symbol errors at load time rather than at the first call to the missing function — a crash deep in a render loop. In production, `RTLD_LAZY` is generally preferred since it distributes the resolution cost across the library's usage rather than concentrating it at startup.

#### dlopen_ext and APK Loading

`dlopen_ext()` is an Android-specific extension declared in `<android/dlext.h>` (API 21+). It accepts an `android_dlextinfo` struct that enables capabilities unavailable in POSIX `dlopen`:

```c
#include <android/dlext.h>

// Load a .so from within an APK/zip file without extraction:
int fd = open("/data/app/com.example.game/base.apk", O_RDONLY);
android_dlextinfo info = {
    .flags = ANDROID_DLEXT_USE_LIBRARY_FD
           | ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET,
    .library_fd        = fd,
    .library_fd_offset = offset_of_so_within_apk,
};
void* handle = android_dlopen_ext("libgame.so", RTLD_NOW, &info);
```

The `ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET` flag lets the dynamic linker mmap the `.so` directly from the APK's zip container, skipping extraction to disk entirely. ART's `ClassLoader` implementation uses `dlopen_ext` internally when loading `.so` files from APKs configured for uncompressed native libraries. The `ANDROID_DLEXT_USE_NAMESPACE` flag allows native code to specify which linker namespace the loaded library should be placed in — used by HAL loaders and the Vulkan loader to open vendor libraries in the `sphal` namespace.

#### Diagnosing Namespace Violations

When `dlopen` fails due to namespace isolation, `dlerror()` returns a message that names the namespace violation. Enable verbose linker logging for development diagnosis:

```bash
adb shell setprop debug.ld.all dlopenclose
adb logcat -s linker
```

This logs every `dlopen`/`dlclose` with namespace transitions, useful for identifying whether a transitive dependency is pulling in a non-public platform library.

[Source: Android linker namespaces](https://developer.android.com/ndk/guides/using-native-api-and-linker-namespaces), [android/dlext.h](https://developer.android.com/ndk/reference/group/libdl)

---

### 2.10 C++ Standard Library ABI: libc++\_shared vs libc++\_static

The choice of C++ STL linkage is one of the most consequential build-time decisions in an NDK project, and getting it wrong produces bugs that are subtle and hard to reproduce.

#### One STL, Two Linkage Modes

Since NDK r18 (Android 9), the Android NDK ships exactly one C++ Standard Library implementation: **LLVM's libc++**. The legacy STLs (gnustl, libstlport, STLport) have been removed. libc++ is available in two linkage modes:

**`c++_shared`**: the application links against `/data/app/<package>/lib/<abi>/libc++_shared.so`, which must be bundled in the APK's `lib/<abi>/` directory. All `.so` files in the APK that use C++ STL types share a single libc++ instance. This is the correct choice whenever multiple `.so` files need to exchange STL objects — `std::string`, `std::vector`, `std::function`, `std::exception` — across their boundaries.

**`c++_static`**: libc++ is statically linked into each `.so`. There is no `libc++_shared.so` to bundle. This seems simpler, but has a critical failure mode: if more than one `.so` in the same process uses `c++_static`, each has its own private copy of the STL's global state. This causes several classes of subtle bugs:
- **`dynamic_cast` failures across boundaries**: RTTI type information is stored as global symbols; with two copies, a `Base*` cast to `Derived*` through a boundary returns null even when the types match, because the two copies have different `typeinfo` addresses.
- **Double-destruction of globals**: `std::locale::global()`, `std::cout`, and other STL globals are constructed and destructed once per copy — if their lifetime crosses a `dlclose`, the destructor fires twice.
- **`std::exception` not caught across boundaries**: the exception unwinding tables from the two copies do not interoperate; a `std::runtime_error` thrown in one `.so` is not caught by `catch (std::exception&)` in another.

The rule is clear: **if the APK loads exactly one `.so` that uses C++** (plus statically-linked dependencies that do not cross STL objects), `c++_static` is safe. If the APK loads multiple `.so` files that pass STL objects to each other — including via Prefab-distributed library ABIs — use `c++_shared`.

#### Exceptions and RTTI

Both are enabled by default in NDK CMake builds (`-fexceptions -frtti`). RTTI is required for `dynamic_cast` and `typeid`; exceptions are required for standard C++ exception handling and for `std::terminate` to fire correctly. Disabling either requires care:

```cmake
# Disable exceptions and RTTI for a specific target (code size reduction):
set_target_properties(my_hot_target PROPERTIES
    COMPILE_FLAGS "-fno-exceptions -fno-rtti"
)
```

Disabling exceptions is safe only for code that genuinely never throws and never calls into standard library functions that throw (e.g., `std::vector::at()`). Mixing `-fno-exceptions` and `-fexceptions` code in the same `.so` is supported — unwinding records are emitted for `noexcept` functions so the linker can route around them — but mixing them across a `dlopen` boundary introduces the same multiple-EH-table problem as mixing `c++_static` copies.

#### The RTTI Pointer-Identity Problem in Depth

The `dynamic_cast` failure mode with `c++_static` is worth understanding precisely, because it produces incorrect behaviour rather than a crash — making it particularly hard to diagnose. When C++ compiles a class hierarchy, every class with virtual functions gets a `typeinfo` object: a `std::type_info` instance whose address serves as the type's identity token. `dynamic_cast` works by comparing `typeinfo*` pointers, not the type name strings they contain.

With `c++_shared`, there is one copy of each `typeinfo` symbol in the process, resolved by the dynamic linker. All `.so` files that reference `typeinfo for Base` get the same pointer. With `c++_static`, each `.so` has its own copy of `typeinfo for Base` at a different address. A `dynamic_cast<Derived*>(base_ptr)` executed in `.so A` against an object whose `Derived` class was compiled into `.so B` compares `A`'s `typeinfo` pointer with `B`'s `typeinfo` pointer — they point to different addresses even though the string name is identical, so the cast returns `nullptr`.

The same pointer-identity issue affects `typeid` comparisons: `typeid(*obj) == typeid(Derived)` returns `false` across a `c++_static` boundary even when `*obj` genuinely is a `Derived`. This silently breaks visitor patterns, discriminated unions, and any code that uses `typeid` for type discrimination across module boundaries.

The linker normally deduplicates weak symbols — `typeinfo` objects are emitted as `STB_WEAK` symbols so that the linker merges multiple definitions into one. With `c++_shared`, the dynamic linker performs this deduplication at runtime across all loaded `.so` files. With `c++_static`, the `typeinfo` symbols are incorporated into each `.so`'s private data segment, not exported globally, so the dynamic linker has no opportunity to merge them. This is the mechanism behind the failure; the fix is always to use `c++_shared` when types are shared across module boundaries.

#### Vtable Layout and Cross-Boundary Virtual Dispatch

Virtual function tables (vtables) are subject to a related problem. A vtable is a per-class array of function pointers, emitted as a `STB_WEAK` ELF symbol (`vtable for Foo`). In `c++_shared` builds, the dynamic linker resolves all references to `vtable for Foo` to a single definition. In `c++_static` builds, each `.so` embeds its own vtable, and objects created in one `.so` whose vtable pointers are dereferenced in another `.so` can invoke stale or mismatched function pointers if the vtable layout differs — for instance, if one `.so` was compiled against a different version of a base class header that added or reordered virtual functions. This is the C++ One Definition Rule (ODR) violation problem in practice, and `c++_static` with shared types makes ODR violations far more likely.

#### C++ Exceptions Across the JNI Boundary

A C++ exception that unwinds through a JNI call frame reaches ART's JNI trampoline, which has no catch handler for C++ exceptions. ART calls `std::terminate()`, terminating the process with a tombstone. The fix is to catch all C++ exceptions at the JNI boundary and convert them to pending Java exceptions before returning. Section 2.12 covers this pattern in full.

The STL linkage choice interacts with exception handling: if the throwing code was compiled with `c++_static` and the catching code lives in a different `.so` with its own `c++_static`, the exception class hierarchy (`std::exception`, `std::runtime_error`, etc.) has the same pointer-identity problem as `typeinfo`. A `catch (const std::exception&)` in `.so B` does not catch a `std::runtime_error` thrown in `.so A` because the two `typeinfo for std::runtime_error` pointers differ. This is another argument for `c++_shared` whenever exception types cross module boundaries.

#### ABI Stability Across NDK Versions

libc++ itself does not guarantee ABI stability across NDK versions. However, within a single APK where all `.so` files are compiled with the same NDK version, the ABI is consistent. APEX-delivered library updates (e.g., the `libvulkan.so` Mainline module) are compiled by Google with the same NDK LTS release used for the NDK-facing ABI, so app `.so` files compiled against NDK r27 LTS headers remain compatible with APEX updates that implement those same headers.

The practical guidance for teams managing multiple NDK versions across libraries: use NDK r27 LTS (or the latest LTS at the time of project start) and pin it in `ndkVersion` in `build.gradle.kts`. Mixing `.so` files compiled with NDK r21 (which shipped an older libc++ with a slightly different `std::optional` and `std::variant` layout) with files compiled against NDK r27 in the same process can cause ABI mismatches — silent data corruption, not immediate crashes — for types whose layout changed between NDK versions.

[Source: Android NDK C++ library support](https://developer.android.com/ndk/guides/cpp-support)

---

### 2.11 Signal Handling in Native Code: Async Safety and ANR Avoidance

POSIX signals are the mechanism by which the kernel notifies a process of hardware faults, inter-process signals, and timers. On Android, signals are central to crash reporting, ART GC coordination, and ANR detection. Native code that installs custom signal handlers must understand all three uses to avoid breaking the platform.

#### Signals Relevant to Android Native Code

`SIGSEGV` fires on null or invalid pointer dereferences; on ARMv8, accessing a misaligned address for a data type that requires alignment (e.g., `uint64_t` at an odd address) on some microarchitectures raises `SIGBUS` instead. `SIGABRT` is sent by `abort()`, `__android_log_assert`, and ART itself when it detects a fatal internal error. `SIGILL` fires on execution of an illegal instruction — most commonly seen when code compiled with ARMv8.2 SIMD intrinsics (`sse4.2`, `dotprod`, `fp16`) runs on an ARMv8.0 device that lacks the extension. `SIGPIPE` is generated when writing to a closed socket or pipe; the NDK default-ignores it (via `sigaction(SIGPIPE, SIG_IGN, NULL)` during `JNI_OnLoad`), preventing unexpected process termination on broken connections. `SIGUSR1` and `SIGUSR2` are used internally by ART for GC coordination — native code must not mask or reuse them.

#### Async-Signal-Safe Functions

Signal handlers execute asynchronously — the signal can arrive while the interrupted thread holds any lock, including `malloc`'s internal lock. Calling any function that acquires a lock from inside a signal handler risks deadlock. POSIX defines a set of **async-signal-safe** functions guaranteed not to hold internal locks: `write(2)`, `_exit(2)`, `getpid(2)`, `sigaction(2)`, `kill(2)`, `send(2)`, `clock_gettime(2)`. Standard library functions — `printf`, `malloc`, `pthread_mutex_lock`, `std::string` operations — are not async-signal-safe. A signal handler that calls `printf` can deadlock when the signal arrives while the main thread holds `stdout`'s lock.

The practical pattern for crash signal handlers: write a small fixed-size buffer to a pipe or `STDERR_FILENO` using `write()`, then chain to the original handler. Do not attempt to format, allocate, or log.

#### ART GC and Signals

ART's Concurrent Copying GC uses two mechanisms that involve signals:
- **`SIGUSR1`** (on some ART configurations): triggers explicit GC from external tooling.
- **Safepoint polling**: ART does not use signals for safepoints in the modern GC; instead, it polls a thread-local flag at backward branches. However, the ART runtime reserves `SIGUSR1` and `SIGUSR2` — native code should not `sigaction` these without understanding the interaction.

More critically, `pthread_sigmask` should only mask signals for threads that specifically handle them. Masking `SIGABRT` in a worker thread prevents ART's abort mechanism from reaching that thread, making crash dumps incomplete.

#### ANR Detection and SIGQUIT

An **Application Not Responding** (ANR) condition is detected by the system when the app's main thread does not process an input event within 5 seconds, or a broadcast receiver does not complete within 10 seconds. The ANR watchdog sends `SIGQUIT` to the process; Android's `debuggerd` catches `SIGQUIT` and dumps the thread stacks of all threads to logcat and a tombstone file. The dump is the primary diagnostic for ANR investigations.

For native apps using `NativeActivity` or `GameActivity`, the `android_main` function runs on the glue library's background pthread, not the Java main thread. The Java main thread is the `ANativeActivity`'s Java thread — it must continue processing Android lifecycle callbacks without blocking. The `android_native_app_glue` library handles this separation; but if the native loop blocks on a mutex that the Java thread is waiting on (for example, during surface teardown), the Java thread can stall long enough to trigger ANR.

Thread names visible in ANR traces and `debuggerd` tombstones are set via `prctl`:

```c
#include <sys/prctl.h>
prctl(PR_SET_NAME, "game_render_thread");
// Thread now appears as "game_render_thread" in ANR traces and debuggerd output
```

#### Installing a Signal Handler with Correct Chaining and an Alternate Stack

`SIGSEGV` caused by a stack overflow is particularly dangerous: the handler attempts to run on the same overflowed stack and immediately faults again, producing an infinite loop. The solution is an **alternate signal stack** installed via `sigaltstack`, combined with `SA_ONSTACK` in the `sigaction` flags:

```c
#include <signal.h>
#include <sys/mman.h>

static uint8_t alt_stack_buf[SIGSTKSZ * 4];

static void install_crash_handler(void) {
    stack_t alt = {
        .ss_sp    = alt_stack_buf,
        .ss_size  = sizeof(alt_stack_buf),
        .ss_flags = 0,
    };
    sigaltstack(&alt, NULL);

    struct sigaction sa = {}, old_sa = {};
    sa.sa_sigaction = my_crash_handler;
    sa.sa_flags     = SA_SIGINFO | SA_ONSTACK;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGSEGV, &sa, &old_sa);
    // Store old_sa to chain in handler
}

static void my_crash_handler(int signo, siginfo_t* info, void* ctx) {
    // Write crash info async-signal-safely, then chain:
    old_sa.sa_sigaction(signo, info, ctx);
}
```

Not chaining to `old_sa` (the handler registered by `debuggerd_init`) prevents tombstone generation. The rule: custom crash signal handlers must always chain to the previous handler to preserve `debuggerd` behaviour.

[Source: Android signal handling guide](https://developer.android.com/ndk/guides/signal-handling)

---

### 2.12 C++ Exceptions Across the JNI Boundary

The JNI boundary between ART-managed Java code and native C++ is not a transparent exception conduit in either direction. Understanding the rules is essential for writing JNI code that behaves correctly under error conditions.

#### The JNI Boundary as a Hard Exception Barrier

A C++ exception thrown in native code that unwinds into a JNI trampoline frame reaches code generated by ART that has no `catch` clause for C++ exceptions. ART calls `std::terminate()`, which (if `std::terminate_handler` is the default) calls `abort()`, producing a tombstone. The JNI trampoline frame is not a C++ frame in the DWARF/EHABI sense — it is generated by ART's stub emitter and does not participate in C++ exception unwinding. **Never allow a C++ exception to propagate across a `JNIEXPORT` function boundary.**

In the other direction: a Java exception that was thrown by a JNI call — or was already pending before the JNI call — does not stop native execution. The exception sits pending in the `JNIEnv` until the JNI method returns to Java, at which point ART throws it. This means a native method can inadvertently continue executing with a pending exception if it does not check `ExceptionCheck` after every call that can throw (see Section 2.7 for the `ExceptionCheck` pattern).

#### Detecting Pending Java Exceptions

```cpp
env->CallVoidMethod(obj, method_id, arg);
if (env->ExceptionCheck()) {
    env->ExceptionDescribe();   // prints stack trace to logcat
    env->ExceptionClear();      // clears the pending exception; required before further JNI calls
    // Handle the error: return early, or throw a different exception
    env->ThrowNew(
        env->FindClass("java/lang/RuntimeException"),
        "unexpected failure in native method"
    );
    return;
}
```

After `ExceptionClear()`, the pending exception is gone and further JNI calls are safe. Do not make additional JNI calls between detecting an exception and clearing it — those calls have undefined behaviour with a pending exception.

#### Throwing a Java Exception from Native Code

```cpp
jclass exc_cls = env->FindClass("java/lang/IllegalArgumentException");
if (exc_cls) {
    env->ThrowNew(exc_cls, "index out of bounds in native layer");
}
env->DeleteLocalRef(exc_cls);
return;  // return immediately after ThrowNew; the exception is thrown when the JNI frame exits
```

`ThrowNew` sets a pending exception in the `JNIEnv`. The native method must return promptly after calling `ThrowNew`; continuing to call JNI functions with a pending exception is undefined behaviour. The exception is actually thrown to the Java caller only when the JNI frame returns.

#### Converting C++ Exceptions to Java Exceptions

The safe pattern for JNI methods that call into C++ code that may throw:

```cpp
JNIEXPORT jint JNICALL Java_com_example_Renderer_initVulkan(JNIEnv* env, jobject) {
    try {
        return do_vulkan_init();  // may throw std::bad_alloc, std::runtime_error, etc.
    } catch (const std::bad_alloc&) {
        env->ThrowNew(env->FindClass("java/lang/OutOfMemoryError"),
                      "native Vulkan allocation failed");
    } catch (const std::runtime_error& e) {
        env->ThrowNew(env->FindClass("java/lang/RuntimeException"), e.what());
    } catch (...) {
        env->ThrowNew(env->FindClass("java/lang/RuntimeException"),
                      "unknown native exception");
    }
    return -1;
}
```

This pattern catches all C++ exceptions at the JNI boundary, converts them to the appropriate Java exception type, and returns a sentinel value. The Java caller sees the Java exception rather than a native crash.

#### noexcept at the JNI Boundary

Marking a JNI entry point `noexcept` is a valid optimisation for code that is proven to never throw:

```cpp
JNIEXPORT jlong JNICALL Java_com_example_GPU_getTimestamp(JNIEnv*, jclass) noexcept {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return static_cast<jlong>(ts.tv_nsec);
}
```

If a C++ exception escapes a `noexcept` function, the C++ runtime calls `std::terminate()` immediately — without unwinding — rather than letting the exception propagate toward the JNI trampoline. This is correct behaviour (it prevents the undefined-behaviour crash at the trampoline), but it bypasses the exception-to-Java-exception conversion pattern above. Use `noexcept` only on functions genuinely incapable of throwing.

#### Stack Unwinding Through the JNI Frame

C++ stack unwinding via DWARF CFI or ARM EHABI proceeds through native frames normally. The ART JNI trampoline frame is not a DWARF frame, so unwinding terminates when the C++ runtime's unwinder reaches it and fails to find a matching `catch`. This is by design — ART invokes `std::terminate()` at this point rather than letting the exception escape into managed code in an undefined state.

`JNI_OnUnload` (called when the library's class loader is GC'd, rare in practice) is safe to call JNI from, but must not throw — there is no exception handler above it in the call stack. Always wrap `JNI_OnUnload` body in a broad try-catch if it calls C++ that might throw.

[Source: Android JNI Tips — exceptions](https://developer.android.com/training/articles/perf-jni#exceptions)

---

### 2.13 Memory Sanitizers on Android: ASan, HWASan, UBSan, and MTE

Memory safety bugs — use-after-free, buffer overflow, null dereference, integer overflow — are the most common sources of security vulnerabilities and stability crashes in native Android code. The NDK ships four complementary sanitizer tools that detect these bugs at different cost points. A mature native Android codebase runs all four at appropriate stages of the development cycle.

#### AddressSanitizer (ASan)

ASan (`-fsanitize=address`) instruments every heap allocation, stack frame, and global variable with **red zones** — poisoned memory regions surrounding the allocation. A shadow memory byte map stores the poisoning state of every aligned 8-byte block. On every memory access, ASan inserts an inline check against the shadow map; if the accessed address lies in a poisoned region, ASan reports the error with a full stack trace. ASan catches heap buffer overflow, stack buffer overflow, use-after-free, use-after-return, and global buffer overflow.

The cost is significant: approximately **2× memory usage** (from the shadow map) and **1.5–2× CPU overhead** (from the inline checks). This makes ASan unsuitable for production use but ideal for CI runs and targeted bug investigation.

```cmake
# CMakeLists.txt — enable ASan for a specific target
target_compile_options(mygame PRIVATE
    -fsanitize=address -fno-omit-frame-pointer)
target_link_options(mygame PRIVATE
    -fsanitize=address)
```

ASan requires the `libclang_rt.asan-<arch>-android.so` runtime library, which must be bundled in the APK or preloaded via `LD_PRELOAD`. The NDK's `wrap.sh` mechanism (place a `wrap.sh` script in the APK's `lib/<abi>/` directory) is the recommended way to inject the ASan runtime without modifying the app's Java code.

#### HWAddressSanitizer (HWASan)

HWASan (`-fsanitize=hwaddress`) takes a different approach: it stores a **tag** in the top 8 bits of every pointer (bits 56–63, exploiting ARM's **Top Byte Ignore** — TBI — feature, which silently masks those bits on normal memory accesses). Every allocation and stack frame is tagged; every pointer derived from the allocation carries the same tag. On access, a brief inline check compares the pointer's top byte against the tag stored in a shadow map; mismatches (indicating out-of-bounds or use-after-free access) are reported.

HWASan's key advantage over ASan is lower overhead: approximately **15% memory overhead** (versus ASan's 2×) and lower CPU overhead, making it suitable for extended dogfooding sessions. It runs on Armv8+ devices (API 27+, the TBI feature is mandatory on AArch64). Android 11+ supports HWASan via `ndk-build APP_SANITIZE := hwaddress` or CMake `ANDROID_SANITIZE` variable. On Pixel 6 and later (Cortex-X1, ARMv8.5+), HWASan hardware acceleration is available through Memory Tagging Extension (see below).

#### Memory Tagging Extension (MTE)

MTE is an ARMv8.5+ hardware feature that elevates HWASan's tagging concept to hardware enforcement. Physical memory tags are stored alongside each 16-byte granule; pointer tags are stored in the top byte. On every memory access, the hardware compares the pointer tag against the granule tag in a single pipeline stage — zero CPU overhead when no violation occurs. Detection is reported via a `SIGSEGV`-like signal.

Android 13 exposes MTE control via the app manifest:

```xml
<!-- AndroidManifest.xml -->
<application android:memtagMode="async">
    <!-- async: violations reported asynchronously (non-fatal by default); best for production -->
    <!-- sync:  violations immediately terminate the process with a tombstone; best for testing -->
```

`async` mode is suitable for production deployment on Pixel 8+ devices (Cortex-X3, ARMv8.5+): it catches a meaningful fraction of memory bugs at zero throughput cost, reporting them via crash reporting SDKs without killing the process for every detection. `sync` mode is for testing — every violation produces a tombstone with a full stack trace.

On devices without ARMv8.5+ hardware (which is most devices in 2026), `android:memtagMode` has no effect. HWASan fills the same role in software.

#### UBSan (Undefined Behaviour Sanitizer)

UBSan (`-fsanitize=undefined` or specific subchecks) inserts lightweight checks for C++ undefined behaviour: signed integer overflow, null pointer dereference, out-of-bounds array access, misaligned access, and others. Unlike ASan/HWASan, UBSan has **zero memory overhead** and typically only **5–10% CPU overhead**, making it viable in CI builds that run test suites under time constraints.

```cmake
target_compile_options(mygame PRIVATE
    -fsanitize=integer-divide-by-zero,null,signed-integer-overflow,bounds
    -fno-sanitize-recover=all)  # terminate on first violation instead of continuing
target_link_options(mygame PRIVATE -fsanitize=undefined)
```

On Android, UBSan reports route through `__ubsan_on_report`, which calls `__android_log_write` to logcat. The `-fno-sanitize-recover=all` flag causes the process to abort on the first violation, producing a tombstone — useful in CI where you want a hard failure, not a log message that might be missed.

#### Symbolication of Sanitizer Output

Sanitizer output in logcat contains raw addresses. Symbolicate with `llvm-symbolizer` from the NDK:

```bash
# Symbolicate an ASan/HWASan error from logcat:
adb logcat | grep -A 20 "AddressSanitizer\|HWAddressSanitizer" | \
    ${ANDROID_NDK}/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-symbolizer \
    --obj=build/intermediates/merged_native_libs/debug/arm64-v8a/lib/arm64-v8a/libgame.so
```

The unstripped `.so` must be available locally; the stripped APK copy cannot be symbolicated. This is why retaining unstripped copies in `build/intermediates/` or uploading them to a crash reporting service is essential.

`__attribute__((no_sanitize("address")))` can suppress sanitizer checks on specific functions — hot paths where the overhead is prohibitive or where the sanitizer produces known false positives (e.g., deliberately aliasing memory in a custom allocator). Suppress sparingly; the goal is to fix the underlying issue.

#### Recommended Sanitizer Workflow

| Stage | Sanitizer | Rationale |
|---|---|---|
| CI unit/integration tests | UBSan | Near-zero overhead; catches integer and pointer UB on every test run |
| Dogfooding on Pixel 6+ | HWASan | ~15% overhead; catches memory bugs without stopping session |
| Targeted bug investigation | ASan | Full precision; 2× memory overhead acceptable for short sessions |
| Production (Pixel 8+) | MTE async mode | Zero overhead hardware tagging; catches subset of bugs in production |

[Source: Android NDK sanitizers guide](https://developer.android.com/ndk/guides/sanitizers), [HWASan design](https://clang.llvm.org/docs/HardwareAssistedAddressSanitizerDesign.html)

---

### 2.14 Native Crash Analysis: Tombstones, Unwinding, and Symbolication

When an Android process dies due to a fatal signal — `SIGSEGV`, `SIGABRT`, `SIGBUS`, `SIGILL` — the kernel delivers the signal to `debuggerd`, Android's crash reporting daemon. `debuggerd` generates a **tombstone** file containing everything needed to diagnose the crash post-hoc. Understanding the tombstone format and the toolchain for symbolication is essential for debugging crashes that only manifest on device.

#### The Tombstone File

Tombstones are written to `/data/tombstones/tombstone_NN` with NN cycling through 00–99 (oldest is overwritten when the ring fills). They are accessible via `adb bugreport` (which packages all tombstones in the bug report zip) or directly via `adb shell cat /data/tombstones/tombstone_00` (requires root or the `android.permission.READ_LOGS` on debug builds).

A tombstone contains:
- Build fingerprint and API level of the device
- Process and thread IDs, thread name, and package name
- Signal name, signal code, and fault address
- CPU register dump (all 31 general-purpose registers on AArch64, plus `sp`, `pc`, `pstate`)
- Full backtrace of the crashing thread
- Backtraces of all other threads in the process
- Open file descriptor table
- Memory maps (`/proc/<pid>/maps`)
- Logcat output (last ~1000 lines, if logd is running)

#### Annotated Tombstone Excerpt

```text
*** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'google/redfin/redfin:14/UP1A.231005.007/...'
ABI: 'arm64'
Timestamp: 2026-06-27 10:42:18.132000000+0000

pid: 12345, tid: 12347, name: game_render_thread  >>> com.example.game <<<
uid: 10188
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0000000000000010
Cause: null pointer dereference

    x0  0000000000000000  x1  00000072b1234abc  x2  0000000000000001
    x3  00000072b1234ab8  x4  0000000000000000  x5  0000000000000000
    ...
    sp  0000007ff8a12340  lr  000000720012abcc  pc  000000720012abc0

backtrace:
  #00 pc 000000000012abc0  /data/app/~~xyz/com.example.game/lib/arm64/libgame.so
  #01 pc 00000000000f1234  /data/app/~~xyz/com.example.game/lib/arm64/libgame.so
  #02 pc 00000000001a5678  /system/lib64/libandroid.so
```

The `pc` values are offsets within the loaded library (address minus load base). Symbolication maps these to source file and line numbers.

#### Symbolication with ndk-stack

`ndk-stack` is the simplest path for symbolication: it reads a tombstone or logcat stream and annotates backtrace `pc` entries with function names and source locations using the unstripped `.so` files from the build directory.

```bash
# Stream logcat through ndk-stack in real time:
adb logcat | ndk-stack \
    -sym ./app/build/intermediates/merged_native_libs/debug/arm64-v8a/

# Or symbolicate a saved tombstone:
ndk-stack -sym ./app/build/intermediates/merged_native_libs/debug/arm64-v8a/ \
    < /tmp/tombstone_00
```

`ndk-stack` matches each `pc <offset>  <library>` backtrace line and queries the unstripped copy of `<library>` using DWARF debug info. It requires the unstripped `.so` for the matching ABI to be in the directory passed to `-sym`.

#### Symbolication with llvm-symbolizer

For more control — DWARF5 support, demangling, inline expansion — use `llvm-symbolizer` directly from the NDK toolchain:

```bash
NDK_LLVM="${ANDROID_NDK}/toolchains/llvm/prebuilt/linux-x86_64/bin"
# Symbolicate a single address (0x12abc0 within libgame.so):
${NDK_LLVM}/llvm-symbolizer \
    --obj=./app/build/intermediates/merged_native_libs/debug/arm64-v8a/lib/arm64-v8a/libgame.so \
    0x12abc0
# Output:
# GameRenderer::render()
# /home/user/game/src/renderer.cpp:142:12
```

`llvm-symbolizer` expands inlined frames: if `render()` was inlined into `GameLoop::tick()` which was inlined into `android_main()`, all three frames appear in the output, each with their source location. This is valuable for release builds with `RelWithDebInfo` that inline aggressively.

#### Annotating Crashes with Application Context

Android API 23+ provides `ACrashDetail_register` (in `<android/crash_detail.h>`) to embed application-specific context in the tombstone:

```c
#include <android/crash_detail.h>

// Register a detail buffer before the frame that might crash:
static char gpu_ctx[256];
snprintf(gpu_ctx, sizeof(gpu_ctx), "frame=%u pipeline=%p", frame_count, pipeline);
crash_detail_t* detail = ACrashDetail_register(
    "gpu_state", gpu_ctx, strlen(gpu_ctx));

// ... do work that might crash ...

// Unregister when the context is no longer valid:
ACrashDetail_unregister(detail);
```

Registered details appear in the tombstone under an `Extra crash detail` section. They survive the crash because `debuggerd` reads them from the process's memory map before the process exits, using `ptrace`. This technique is valuable for embedding game state — current level, GPU pipeline handle, active shader — that disambiguates which code path triggered the crash.

#### Common Causes of Missing Symbols

Three patterns produce `?? ??:0` entries in symbolicated backtraces:
1. **Stripped `.so` without unstripped copy**: the APK's stripped library is used for symbolication instead of the build tree's unstripped copy. Always pass the `merged_native_libs` path, not the APK-extracted path.
2. **Inlined frames with `-O2` without debug info**: add `-fno-inline` or use `RelWithDebInfo` to retain DWARF. Production crash reporting services (Firebase Crashlytics, Play Vitals) require pre-uploaded unstripped `.so` files to perform server-side symbolication.
3. **Wrong ABI path**: passing `arm64-v8a` unstripped libraries to symbolicate an `armeabi-v7a` crash. Verify the ABI from the tombstone's `ABI:` header line before selecting the symbol path.

#### DWARF and ARM EHABI Unwinding Prerequisites

Reliable stack unwinding in the tombstone requires either `.debug_frame` (DWARF CFI) or `.ARM.exidx` / `.ARM.extab` (ARM Exception Handling ABI) sections in the `.so`. Building with `-fno-omit-frame-pointer` provides a fallback frame pointer chain that `debuggerd` can follow even when CFI is incomplete — at the cost of one extra register per function. For release builds, keep `-fno-omit-frame-pointer` on the `.so` used for crash reporting, even if the APK's copy is stripped.

[Source: Android native crash guide](https://source.android.com/docs/core/tests/debug/native-crash), [ndk-stack reference](https://developer.android.com/ndk/guides/ndk-stack)

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

### 3.7 The Security Motivation: Memory Safety and Android Vulnerabilities

The adoption of Rust in Android is not a stylistic preference — it is a direct response to a documented vulnerability pattern. Google's Android Security team has consistently tracked the root cause of critical and high-severity Android vulnerabilities, and the finding has been stable across years: approximately **70% of Android's security vulnerabilities are caused by memory-safety bugs in C and C++**. [Source: Google Security Blog — Memory safe languages in Android 13](https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html)

The specific vulnerability classes involved are: buffer overflows (spatial memory safety), use-after-free (temporal memory safety), double-free, use of uninitialised memory, and integer overflow feeding into pointer arithmetic. These are not rare edge cases — they are the dominant attack surface because Android's most security-sensitive components (Bluetooth, Wi-Fi, NFC, codec parsers, kernel drivers, the TrustZone TEE) are implemented in C or C++, and the combination of network-reachable code with memory-unsafe languages reliably produces high-severity RCE vulnerabilities.

**The data across the industry is consistent.** Microsoft reported the same ~70% figure for their security bugs. Chromium's security team found that ~70% of severe Chromium bugs were memory safety issues. The NSA issued a 2022 advisory recommending a shift away from C/C++ to memory-safe languages. [Source: NSA guidance on memory-safe languages](https://media.defense.gov/2022/Nov/10/2003112742/-1/-1/0/CSI_SOFTWARE_MEMORY_SAFETY.PDF)

**Why Rust and not a GC language?** Java, Kotlin, and Go eliminate most of these bugs via garbage collection, but they are unsuitable for the layers where the vulnerabilities occur: kernel drivers cannot use a GC runtime; codec parsers must have deterministic latency; Bluetooth and Wi-Fi stacks run in constrained environments where GC pauses are unacceptable. Rust's ownership and borrow-checking system eliminates the same vulnerability classes as a GC — use-after-free, double-free, data races — at compile time with zero runtime overhead and no GC pauses. This makes it the only practical memory-safe option for system-level Android code.

**The adoption timeline and measurable impact.** Google began introducing Rust into AOSP in 2021. In Android 13 (2022), new code written in Rust appeared in the Bluetooth stack, DNS resolver, and the keystore service. The Android Security team subsequently reported that **zero memory-safety vulnerabilities were found in new Rust code** shipped in Android 13 and 14, while C/C++ code continued to generate the majority of memory-safety bugs. This is the expected outcome — it is structurally impossible to write certain classes of memory bugs in safe Rust — but the empirical validation across production Android code is significant. [Source: Google Security Blog — Eliminating Memory Safety Vulnerabilities at the Source](https://security.googleblog.com/2024/09/eliminating-memory-safety-vulnerabilities-Android.html)

**Implication for graphics developers.** The `nova` DRM kernel driver (Rust, in Linux/Android kernel staging), future Vulkan ICD components, and NPU kernel drivers are the areas where Rust's safety properties are most compelling. A memory-safety bug in a GPU kernel driver can yield a full kernel compromise — the attack surface is wide (attacker-controlled shader compilation, attacker-controlled memory descriptors) and the consequence of a bug is severe.

### 3.8 JNI from Rust: The `jni` Crate

The Rust NDK ecosystem (§3.2) covers the case of calling native C/NDK APIs from Rust. The reverse direction — calling into the Java/Kotlin layer from Rust, or exposing Rust functions as JNI methods callable from Java — requires the `jni` crate. [Source: jni crate on crates.io](https://crates.io/crates/jni)

```toml
# Cargo.toml
[dependencies]
jni = "0.21"
```

**Exposing a Rust function as a JNI method.** The naming convention is identical to C JNI: `Java_<package>_<class>_<method>` with `_` replacing `.`:

```rust
// File: src/lib.rs
use jni::JNIEnv;
use jni::objects::{JClass, JString};
use jni::sys::jstring;

/// Called from Java as:
///   com.example.myapp.RustBridge.greet(String name)
#[no_mangle]
pub extern "system" fn Java_com_example_myapp_RustBridge_greet<'local>(
    mut env: JNIEnv<'local>,
    _class: JClass<'local>,
    name: JString<'local>,
) -> jstring {
    // Convert Java String to Rust &str:
    let name: String = env
        .get_string(&name)
        .expect("invalid Java string")
        .into();

    let greeting = format!("Hello from Rust, {}!", name);

    // Convert Rust String back to Java String:
    env.new_string(greeting)
        .expect("failed to create Java string")
        .into_raw()
}
```

The Java declaration needs only:
```java
// RustBridge.java
package com.example.myapp;
public class RustBridge {
    static { System.loadLibrary("myapp"); }
    public static native String greet(String name);
}
```

**Calling Java methods from Rust.** The `jni` crate also provides `JNIEnv::call_method` and related helpers to call back into the Java layer from Rust — the equivalent of the C `env->CallVoidMethod(...)` pattern described in §2.12:

```rust
use jni::objects::JObject;
use jni::signature::{Primitive, ReturnType};

fn notify_java(env: &mut JNIEnv, callback: &JObject) {
    env.call_method(
        callback,
        "onFrameReady",           // method name
        "(I)V",                   // signature: (int) → void
        &[jni::objects::JValue::Int(42)],
    ).expect("JNI call failed");

    // Always check for pending exceptions after Java calls:
    if env.exception_check().unwrap_or(false) {
        env.exception_describe().ok();
        env.exception_clear().ok();
    }
}
```

**Threading and `JavaVM`.** The `JNIEnv` is not `Send` — it is bound to the thread on which it was obtained. To call Java from a Rust background thread, obtain a `JavaVM` reference at startup (via `JNI_OnLoad`) and call `java_vm.attach_current_thread()` on the background thread:

```rust
use jni::JavaVM;
use std::sync::Arc;

static JAVA_VM: std::sync::OnceLock<Arc<JavaVM>> = std::sync::OnceLock::new();

#[no_mangle]
pub extern "system" fn JNI_OnLoad(vm: JavaVM, _: *mut std::ffi::c_void) -> jni::sys::jint {
    JAVA_VM.set(Arc::new(vm)).ok();
    jni::sys::JNI_VERSION_1_6
}

fn call_java_from_background_thread() {
    let vm = JAVA_VM.get().expect("JavaVM not initialised");
    let mut env = vm.attach_current_thread().expect("attach failed");
    // env is now valid for JNI calls on this thread
    // Automatically detaches when the AttachGuard is dropped
}
```

**`JNIEnv` vs raw `*mut JNIEnv`.** The `jni` crate wraps the raw `*mut sys::JNIEnv` pointer (which C JNI uses) in a safe `JNIEnv<'_>` struct with lifetime tracking. This prevents the most common C JNI bugs: calling `env` from the wrong thread, using a `JNIEnv` after the frame it belongs to has been popped, and forgetting to check `ExceptionCheck`. The crate's `JObject`, `JString`, `JClass`, `JByteArray` etc. are newtype wrappers over `jobject` with proper drop semantics for local references.

[Source: jni-rs documentation](https://docs.rs/jni/latest/jni/)

### 3.9 Rust in AOSP: Platform Components and the Soong Build System

Since 2021, Google has been writing new Android platform components in Rust and migrating security-critical existing components away from C/C++. This section describes what has been rewritten, what is in progress, and how Rust modules integrate into Android's `Soong` build system — relevant for engineers working on AOSP forks, vendor BSPs, or Android kernel drivers.

**Production Rust components in AOSP (as of Android 14/15):**

| Component | Previous language | Rust module | Notes |
|---|---|---|---|
| Bluetooth stack (Gabeldorsche) | C++ | `packages/modules/Bluetooth/system/gd/` | Stack-level BT host code; HCI, SDP, GATT |
| DNS resolver (`doh` module) | C | `packages/modules/DnsResolver/doh/` | DNS-over-HTTPS implementation |
| `keystore2` | C++ | `system/security/keystore2/` | Android Keystore service (API 31+) |
| `virtualizationservice` | C++ | `packages/modules/Virtualization/` | pKVM guest VM management |
| `crosvm` | Rust (upstream) | `external/crosvm/` | Virtual machine monitor for pKVM |
| Networking stack components | C | Various in `system/netd/` | Incremental migration |
| `rdroid` | New | `frameworks/native/libs/renderscript/` | RenderScript deprecation path |
| `nova` DRM kernel driver | New | Linux kernel staging | Rust Vulkan GPU kernel driver (§3.6) |

**The Soong build system and `Android.bp`.** Android's `Soong` build system (which replaced `make`-based builds in AOSP) uses `Android.bp` Blueprint files to declare build modules. Rust modules use the following module types:

```
// Android.bp — AOSP Rust module declarations
rust_library_shared {
    name: "libmy_android_lib",
    crate_name: "my_android_lib",
    srcs: ["src/lib.rs"],
    rustlibs: [
        "liblog_rust",        // Android log crate (wraps __android_log_write)
        "libnix",             // Unix syscall bindings
        "libanyhow",          // Error handling
    ],
    apex_available: [":anyapex"],  // Available for APEX delivery
    min_sdk_version: "30",
}

rust_binary {
    name: "my_rust_daemon",
    srcs: ["src/main.rs"],
    rustlibs: ["libmy_android_lib"],
    init_rc: ["my_rust_daemon.rc"],   // systemd-equivalent init script
}

rust_test {
    name: "my_android_lib_tests",
    srcs: ["src/lib.rs"],
    rustlibs: ["libmy_android_lib"],
    test_suites: ["general-tests"],
}
```

Soong resolves Rust dependencies from the AOSP tree (vendored crates in `external/rust/crates/`) rather than from crates.io. Every third-party crate used in AOSP must be reviewed and checked into `external/rust/crates/<crate-name>/`. The `cargo2android.py` tool automates the conversion of a `Cargo.toml` into an `Android.bp` and copies the crate into the correct location. [Source: AOSP — Using Rust](https://source.android.com/docs/setup/build/rust/building-rust-modules/overview)

**APEX delivery of Rust libraries.** Rust libraries compiled as `rust_library_shared` can be included in APEX packages (§3.3) the same way as C shared libraries. The `apex_available` field in `Android.bp` controls which APEXes can include the library. Rust's `libc++`-equivalent (`librustc_demangle`, `libstd`) is linked statically by default in AOSP Rust builds, avoiding the `libc++_shared.so` ABI concerns described in §2.10 — Rust's standard library is statically linked into each APEX that uses it.

**Rust in the Android kernel.** The Linux kernel has supported Rust since 6.1 (December 2022), and Android's kernel tree tracks this support. The `nova` DRM driver (for NVIDIA's open-source GPU kernel interface) is the most visible Rust GPU driver in the kernel tree — it is the successor to the `nouveau` driver, written in Rust from scratch rather than porting the existing C code. Android's GKI (Generic Kernel Image) does not yet enable Rust kernel modules by default as of 2025, but vendor kernels can enable `CONFIG_RUST` and include Rust modules. [Source: Linux kernel Rust documentation](https://docs.kernel.org/rust/index.html)

### 3.10 wgpu: Cross-Platform Graphics in Rust on Android

`wgpu` is the dominant cross-platform graphics API for Rust, implementing the WebGPU specification across multiple backends. On Android it uses the Vulkan backend via `ash`, making it the highest-level Rust graphics option that still targets the native GPU with full performance. It is used as Mozilla Firefox's WebGPU implementation on Android, giving it production-level validation on a major shipping app. [Source: wgpu on GitHub](https://github.com/gfx-rs/wgpu)

**Backend selection on Android.** `wgpu` on Android uses the Vulkan backend by default when `VK_VERSION_1_1` or higher is available (virtually all Android 8.0+ devices). A GLES3 backend (`wgpu-hal` → `wgpu-hal/gles`) is available as a fallback for devices without Vulkan:

```toml
# Cargo.toml
[dependencies]
wgpu  = { version = "22", features = ["vulkan"] }
winit = { version = "0.30", features = ["android-native-activity"] }
android-activity = { version = "0.6", features = ["game-activity"] }
```

**ANativeWindow surface creation.** `wgpu` creates an Android surface from an `ANativeWindow*` pointer, obtained from `AndroidApp::native_window()` in the `android-activity` crate:

```rust
use wgpu::SurfaceTargetUnsafe;

async fn init_wgpu(app: &android_activity::AndroidApp) -> (wgpu::Device, wgpu::Queue, wgpu::Surface) {
    let instance = wgpu::Instance::new(wgpu::InstanceDescriptor {
        backends: wgpu::Backends::VULKAN,
        ..Default::default()
    });

    // Obtain ANativeWindow* from android-activity:
    let native_window = app.native_window()
        .expect("no native window");

    // Create wgpu surface from ANativeWindow:
    let surface = unsafe {
        instance.create_surface_unsafe(
            SurfaceTargetUnsafe::from_window(&*native_window)
                .expect("failed to create surface target")
        ).expect("failed to create surface")
    };

    let adapter = instance
        .request_adapter(&wgpu::RequestAdapterOptions {
            power_preference: wgpu::PowerPreference::HighPerformance,
            compatible_surface: Some(&surface),
            ..Default::default()
        })
        .await
        .expect("no suitable Vulkan adapter");

    let (device, queue) = adapter
        .request_device(&wgpu::DeviceDescriptor::default(), None)
        .await
        .expect("failed to create device");

    (device, queue, surface)
}
```

**Swapchain configuration.** `wgpu`'s `SurfaceConfiguration` maps directly to `VkSwapchainCreateInfoKHR` on the Vulkan backend. The `present_mode` field should be set to `wgpu::PresentMode::Mailbox` or `Fifo` — `Fifo` corresponds to `VK_PRESENT_MODE_FIFO_KHR` (vsync), which is the only mode guaranteed on all Android Vulkan implementations:

```rust
let surface_caps = surface.get_capabilities(&adapter);
let config = wgpu::SurfaceConfiguration {
    usage: wgpu::TextureUsages::RENDER_ATTACHMENT,
    format: surface_caps.formats[0],   // typically Bgra8UnormSrgb on Android
    width: window_width,
    height: window_height,
    present_mode: wgpu::PresentMode::Fifo,
    alpha_mode: surface_caps.alpha_modes[0],
    view_formats: vec![],
    desired_maximum_frame_latency: 2,
};
surface.configure(&device, &config);
```

**WGSL shaders.** `wgpu` uses WGSL (WebGPU Shading Language) as its portable shader language. The `naga` compiler (included in `wgpu`) translates WGSL to SPIR-V for the Vulkan backend at runtime:

```rust
let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
    label: Some("my_shader"),
    source: wgpu::ShaderSource::Wgsl(include_str!("shader.wgsl").into()),
});
// naga compiles WGSL → SPIR-V → Vulkan VkShaderModule on Android
```

This compilation happens at `create_shader_module` time — the first call bears a compilation cost (typically 1–5 ms for simple shaders on Adreno). For production, pre-compile and cache SPIR-V via `wgpu`'s pipeline cache support, or ship pre-compiled WGSL `.spv` via `ShaderSource::SpirV`.

**Relation to Firefox on Android.** Mozilla uses `wgpu` as the WebGPU implementation in Firefox for Android (`fenix`), with `wgpu` targeting the Vulkan backend on devices that support it and the GLES3 backend as a fallback. This makes `wgpu` one of the few Rust graphics libraries with a large-scale production Android deployment, and Firefox's conformance test runs against the WebGPU CTS provide ongoing validation of `wgpu`'s Android Vulkan correctness. [Source: wgpu WebGPU conformance](https://github.com/gfx-rs/wgpu/wiki/Conformance-Testing)

**`wgpu` vs `ash` directly.** `ash` (§3.2) gives full access to every Vulkan extension and requires managing lifetimes, synchronisation, and validation manually — appropriate when Vulkan-specific extensions (`VK_ANDROID_external_memory_android_hardware_buffer`, `VK_QCOM_render_pass_transform`) are needed. `wgpu` abstracts these away behind the WebGPU API surface, trading extension access for portability and safety. For a cross-platform Rust game that also targets web (via WebGPU) and desktop, `wgpu` is the correct choice. For an Android-specific engine that needs `AHardwareBuffer` import or explicit tile-memory control, `ash` gives the necessary access.

[Source: wgpu on Android guide](https://github.com/gfx-rs/wgpu/wiki/Running-on-Android)

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
