# Chapter 161 — Android Game Development Kit (AGDK): Native Game Architecture, Input, Audio, and Frame Pacing

> **Audiences:** Graphics application developers building native Android games and high-performance apps; systems developers who want to understand how Android's C/C++ game stack maps onto the graphics, audio, and input subsystems; browser and tool engineers who need to understand the NDK lifecycle model that underpins Chrome's Android renderer and Android Studio's profiling toolchain.

---

## Table of Contents

1. [What AGDK Is](#1-what-agdk-is)
   - 1.1 [Component Overview and Distribution](#11-component-overview-and-distribution)
   - 1.2 [Historical Context: Android Game Development Before AGDK](#12-historical-context-android-game-development-before-agdk)
   - [1.3 What is the Android NDK?](#13-what-is-the-android-ndk)
   - [1.4 What is GameActivity?](#14-what-is-gameactivity)
2. [Play Asset Delivery: APK Limits, OBB, and On-Demand Asset Modules](#2-play-asset-delivery-apk-limits-obb-and-on-demand-asset-modules)
3. [NativeActivity: The Foundation and Its Limits](#3-nativeactivity-the-foundation-and-its-limits)
4. [GameActivity: The Modern Native App Model](#4-gameactivity-the-modern-native-app-model)
5. [Input Handling with GameActivity](#5-input-handling-with-gameactivity)
6. [Touch Input Prediction and End-to-End Latency Reduction](#6-touch-input-prediction-and-end-to-end-latency-reduction)
7. [Paddleboat: Game Controller Library](#7-paddleboat-game-controller-library)
8. [Android Frame Pacing: Swappy](#8-android-frame-pacing-swappy)
9. [ANativeWindow: The Surface Handle](#9-anativewindow-the-surface-handle)
10. [Vulkan on TBDR Mobile GPUs: Architecture and Optimization](#10-vulkan-on-tbdr-mobile-gpus-architecture-and-optimization)
11. [EGL Context Management: Thread Affinity, Shared Resources, and Context Loss](#11-egl-context-management-thread-affinity-shared-resources-and-context-loss)
12. [Oboe: Low-Latency Audio](#12-oboe-low-latency-audio)
13. [Android Performance Tuner](#13-android-performance-tuner)
14. [ADPF In Depth: Performance Hints, Thermal Headroom, and Adaptive Quality](#14-adpf-in-depth-performance-hints-thermal-headroom-and-adaptive-quality)
15. [Native Crash Reporting with Firebase Crashlytics and NDK Symbols](#15-native-crash-reporting-with-firebase-crashlytics-and-ndk-symbols)
16. [Memory Advice API](#16-memory-advice-api)
17. [Android GPU Inspector](#17-android-gpu-inspector)
18. [ARCore Integration: Building AR-Native Games](#18-arcore-integration-building-ar-native-games)
19. [Engine Integration: Unity, Unreal, Godot, and Bevy](#19-engine-integration-unity-unreal-godot-and-bevy)
20. [Google Play Games Services: Leaderboards, Achievements, and Cloud Save](#20-google-play-games-services-leaderboards-achievements-and-cloud-save)
21. [Foldable Devices and Multi-Window: Adaptive Game Layouts](#21-foldable-devices-and-multi-window-adaptive-game-layouts)
22. [Integrations](#22-integrations)

---

## 1. What AGDK Is

The **Android Game Development Kit** (AGDK) is a collection of C/C++ libraries, tools, and Jetpack packages that Google introduced in 2021 to address the fragmented, low-level nature of Android native game development. Before AGDK, developers writing high-performance games in C++ had to stitch together: the legacy `NativeActivity` API (broken text input, coarse input batching), the Android audio stack (OpenSL ES, complex latency tuning), frame timing (bespoke Choreographer hacks), and controller input (per-OEM fragmentation). AGDK provides production-quality solutions to each of these.

### Component overview

| Library | Gradle artifact | Purpose |
|---|---|---|
| `game-activity` | `androidx.games:games-activity` | Replaces `NativeActivity`; lifecycle + input |
| `game-text-input` | bundled with `game-activity` | Working IME text input from C/C++ |
| `games-controller` / Paddleboat | `androidx.games:games-controller` | Gamepad enumeration, mapping, vibration |
| `games-frame-pacing` / Swappy | `androidx.games:games-frame-pacing` | Vsync-aware frame presentation (GL + Vulkan) |
| `oboe` | `com.google.oboe:oboe` | Low-latency audio (OpenSL ES / AAudio abstraction) |
| `games-performance` | `androidx.games:games-performance` | Frame-timing reporting to Google Play backend |
| `games-memory-advice` | `androidx.games:games-memory-advice` | Memory pressure prediction and callbacks |

All libraries are distributed as **Prefab AAR** packages — they integrate with CMake via `find_package()` after Gradle unpacks the headers and pre-built `.a`/`.so` into the build tree. [Source: AGDK overview](https://developer.android.com/games/agdk)

### Gradle integration

```groovy
// build.gradle (app module)
android {
    defaultConfig {
        externalNativeBuild { cmake { arguments "-DANDROID_STL=c++_shared" } }
    }
}

dependencies {
    implementation 'androidx.games:games-activity:3.0.0'
    implementation 'androidx.games:games-controller:2.0.2'
    implementation 'androidx.games:games-frame-pacing:2.1.0'
    implementation 'com.google.oboe:oboe:1.9.0'
    implementation 'androidx.games:games-memory-advice:1.0.0'
}
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.22)
project(mygame)

find_package(game-activity  REQUIRED CONFIG)
find_package(games-controller REQUIRED CONFIG)
find_package(games-frame-pacing REQUIRED CONFIG)
find_package(oboe REQUIRED CONFIG)

add_library(mygame SHARED src/main.cpp)
target_link_libraries(mygame
    game-activity::game-activity_static
    games-controller::paddleboat_static
    games-frame-pacing::swappy_static
    oboe::oboe
    android log vulkan)
```

### Distribution model

AGDK libraries are **not bundled with the OS**; they ship as part of the app's APK/AAB. This lets Google update the libraries independently of Android OS releases, and lets apps always target the latest version. Pre-built binaries are provided for `arm64-v8a`, `armeabi-v7a`, `x86`, and `x86_64`. The `_static` variants link the library into the `.so`; the shared variants reduce APK size when multiple `.so` files use the same library.

### 1.2 Historical Context: Android Game Development Before AGDK

AGDK was not born in a vacuum. It is the result of fifteen years of incremental, often painful evolution of the Android native game development story.

#### Phase 1 — Java-only (Android 1.0–2.2, 2008–2010)

The original Android SDK (Android 1.0, September 2008) was Java-only. Every game ran in the Dalvik VM. The only graphics path was either `android.graphics.Canvas` (software rendering) or `GLSurfaceView`, which managed an OpenGL ES context in a Java rendering thread. There was no mechanism to run a tight C/C++ game loop, no native audio API, and no way to avoid the JNI boundary on every frame. Games like the original Angry Birds shipped as Java apps with JNI for their C++ physics engine — but the renderer and main loop were still Java.

#### Phase 2 — NDK arrives, OpenGL ES goes native (NDK r1–r4, 2009–2010)

**NDK r1** (June 2009, targeting Android 1.5 Cupcake) introduced the first Android Native Development Kit. It was narrow in scope: the available headers were `<stdlib.h>`, `<math.h>`, `<zlib.h>`, `<jni.h>`, and `<android/log.h>`. Crucially, OpenGL ES 1.1 was available via `libGLESv1_CM.so` — but OpenGL ES 2.0 was not, audio was absent, and the only way to run native code was as a `.so` called via JNI from a Java `Activity`. The game loop, lifecycle, and `GLSurfaceView` management remained in Java.

**NDK r3** (December 2009) added OpenGL ES 2.0 (`libGLESv2.so`) and EGL (`libEGL.so`), enabling programmable shaders from native C++. The Java rendering thread in `GLSurfaceView` still owned the game loop, but shader-based rendering could now run entirely in native code.

#### Phase 3 — NativeActivity: the first true native game loop (NDK r5, Android 2.3, 2010–2011)

**NDK r5** (December 2010), paired with **Android 2.3 Gingerbread** (API 9), was the pivotal release. It introduced:

- **`android.app.NativeActivity`**: A Java `Activity` subclass that loaded a native `.so` and forwarded all 16 lifecycle callbacks as C function pointers via `ANativeActivityCallbacks`. For the first time a C/C++ game loop could own the entire app lifecycle.
- **`android_native_app_glue`**: The NDK-provided threading wrapper (§3) that serialised Android callbacks to a background pthread via a pipe — keeping the game loop off the Java main thread and preventing ANR watchdog triggers.
- **OpenSL ES audio** (`libOpenSLES.so`): The Khronos OpenSL ES 1.0.1 API gave native C/C++ code direct access to the audio HAL with lower latency than Java `AudioTrack`. This was the first path to sub-50ms audio latency on Android without Java.
- **Android Asset Manager** NDK API: `AAssetManager_open()` allowed native code to read APK assets without a JNI call.

The [Android Developers blog post "Gingerbread NDK Awesomeness" (January 2011)](https://android-developers.googleblog.com/2011/01/gingerbread-ndk-awesomeness.html) framed this release explicitly as enabling game porting: "You can now write a fully-native Android application."

#### Phase 4 — Ecosystem matures: Vulkan, 64-bit, GCC removal (2014–2018)

**Android 5.0 Lollipop** (2014) made ART the default runtime (replacing Dalvik) and added 64-bit AArch64 support. NDK r10 added AArch64 and x86_64 targets.

**Android 7.0 Nougat** (2016) introduced **Vulkan 1.0** support — the most significant graphics API change since OpenGL ES 2.0. The NDK `vulkan/vulkan.h` header and `/system/lib64/libvulkan.so` loader became available. GPU vendors (Qualcomm, ARM) began shipping Vulkan ICDs in their drivers.

**NDK r17–r18** (2018) completed the GCC→Clang transition: GCC was deprecated in r17 and removed entirely in r18. All Android NDK code must use LLVM/Clang. r22 (2021) made LLD the default linker, removing the final GNU toolchain component.

#### Phase 5 — Android Game SDK and early Swappy (2019)

**Android Game SDK** (December 2019) was Google's first attempt to consolidate game-specific native libraries under a unified umbrella — but it shipped with only one component: **Android Frame Pacing** (Swappy). Frame pacing was a known pain point: without vsync-aware presentation scheduling, game engines either burned extra CPU spinning on `eglSwapBuffers`/`vkQueuePresentKHR` or jittered between vsync intervals. Swappy solved this by injecting `VK_GOOGLE_display_timing` and using Android's `Choreographer` for vsync calibration. Unity 2019.2+ integrated Swappy with a checkbox in Android Player Settings.

#### Phase 6 — Google Play Games Services (2013) and the Java social layer

**Google Play Games Services** (GPGS, launched July 2013) provided the social layer for Android games: leaderboards, achievements, cloud saves, and multiplayer matchmaking. GPGS was Java/Kotlin API-only — native games called it via JNI. It remained the only Google-provided game services SDK for eight years. In 2022, Google launched the **Google Play Games PC** client (Windows), and in 2024 GPGS APIs began supporting Kotlin coroutines. GPGS is entirely separate from the AGDK C/C++ stack; native games use it only at startup/shutdown for session initialisation and score submission.

#### Phase 7 — AGDK v1.0: the unified kit (July 2021)

**AGDK v1.0** was announced at Google for Games Developer Summit, July 12, 2021. It unified all game-specific native libraries — `GameActivity`, `game-text-input`, Paddleboat (controller), Swappy (frame pacing), Oboe (audio), Android Performance Tuner (telemetry), Memory Advice API — under the `androidx.games` Jetpack namespace distributed as Prefab AAR packages via Maven Central. The key addition was **`GameActivity`**: a modern replacement for `NativeActivity` that supported `SurfaceView`, fixed text input, exposed all historical input samples, and integrated cleanly with Kotlin app code via subclassing (`class MainActivity : GameActivity()`).

**Sceneform** (Google's high-level AR rendering library, built on Filament) had been deprecated in August 2020 and its GitHub repository was archived in December 2021 — just as AGDK launched. Sceneform had abstracted ARCore + Filament + lifecycle management into a Java API; its deprecation signalled a move toward the raw ARCore C API + native rendering that AGDK enables.

#### AGDK component version history

| Library | Stable since | Notable milestones |
|---|---|---|
| `games-frame-pacing` (Swappy) | Android Game SDK 2019 | Predates AGDK; added GL+Vulkan unified API in AGDK 1.0 |
| `games-activity` (GameActivity) | v1.0.0 July 2021 | v2: 16KB page size; v4.3: mouse support; v4.4 (May 2026): current |
| `games-controller` (Paddleboat) | v1.0 July 2021 | v2: touchpad, LED light control (June 2024) |
| `games-text-input` | v1.0 July 2021 | Bundled with games-activity; multi-line mode added v4.3 |
| `oboe` | v1.0 2018 (standalone) | Pre-dates AGDK; adopted into `androidx.games` umbrella |
| `games-performance` (Tuner) | v1.0 2021 | v2.0 (August 2024): quality prediction, Play aggregation |
| `games-memory-advice` | v1.0 2021 | v2.0 (September 2023): ML-based memory pressure model |

[Source: AGDK release notes](https://developer.android.com/games/agdk/release-notes), [NDK revision history](https://developer.android.com/ndk/downloads/revision_history), [Gingerbread NDK announcement](https://android-developers.googleblog.com/2011/01/gingerbread-ndk-awesomeness.html), [AGDK launch announcement](https://android-developers.googleblog.com/2021/07/introducing-android-game-development-kit.html)

### 1.3 What is the Android NDK?

The Android Native Development Kit (NDK) is the official toolchain and set of stable C/C++ APIs that allow application code to execute directly on the device processor without going through the Android Runtime (ART). On Android the default development path compiles Java or Kotlin to Dalvik bytecode and runs it on ART, where a garbage collector and JIT compiler manage memory and scheduling. The NDK provides a second path: compiling C or C++ to ARM, x86, or RISC-V machine code via Clang and linking against a versioned set of platform libraries — `libandroid.so`, `libEGL.so`, `libGLESv3.so`, `libvulkan.so`, `libaaudio.so`, and Bionic libc — that are guaranteed stable across Android API levels. For game development the NDK matters because game loops require deterministic, low-jitter frame timing that managed-runtime scheduling and GC pauses cannot reliably provide. A C++ game loop running on a dedicated pthread can pin to a specific CPU core, avoid garbage collection entirely, and drive the GPU driver through Vulkan or OpenGL ES with no JNI overhead on the hot path. Every AGDK library is layered entirely on the NDK: the libraries ship as CMake-consumable static or shared archives, and all runtime interaction with the Android OS — window handles, input events, audio streams, thermal signals — flows through the `<android/...>` header namespace that the NDK stabilises. Understanding the NDK's ABI stability guarantees, its Bionic libc, and its CMake integration is therefore prerequisite to understanding every AGDK component described in this chapter. [Source: NDK Guides](https://developer.android.com/ndk/guides)

### 1.4 What is GameActivity?

GameActivity is the AGDK library that provides the primary Android application lifecycle entry point for C/C++ game code, replacing the older `NativeActivity` class that shipped with NDK r5 in 2010. The fundamental problem with `NativeActivity` was architectural: it attached a native `.so` to an Android `Activity` by convention rather than by a typed API, which produced broken IME text input (the software keyboard could not deliver key events to native code), coarse input event batching that discarded intermediate touch samples between frames, no `SurfaceView` support (only `SurfaceHolder`), and difficulty integrating with Jetpack libraries that expect a Kotlin or Java `Activity` subclass. GameActivity resolves all of these: it is a Java/Kotlin `Activity` subclass (`androidx.games.activity.GameActivity`) that the application extends, forwarding lifecycle callbacks, window change events, and input events into the native layer through a well-typed C API defined in `<game-activity/GameActivity.h>`. Input events are placed into a thread-safe ring buffer rather than dispatched one at a time, preserving all historical touch coordinates between frames — a prerequisite for the touch prediction and gesture accuracy covered in §5 and §6. The native side interacts with the framework through the `android_app` struct and callback pointers, following the same threading model as `android_native_app_glue` but with correct lifecycle semantics and full input fidelity. GameActivity is the recommended entry point for all new AGDK-based titles and is referenced throughout §3–§10 of this chapter. [Source: GameActivity guide](https://developer.android.com/games/agdk/game-activity)

---

## 2. Play Asset Delivery: APK Limits, OBB, and On-Demand Asset Modules

For a game shipping gigabytes of high-resolution textures, audio, and level data, how those assets reach the device is as important as how they are rendered. Android's asset distribution has evolved through three generations: hardcoded APK limits, the legacy OBB expansion file system, and the modern Play Asset Delivery (PAD) framework. Understanding each tier is prerequisite to architecting the asset pipeline of any serious Android title.

### APK Size Limits

Google Play enforces a **150 MB compressed APK size limit** for the primary APK. This limit exists for network reliability and Play Store hosting cost reasons and has been constant since Android's early years. For simple games with procedural content, 150 MB is sufficient; for AAA-tier titles, it is far too small. Every asset byte above 150 MB requires an external delivery mechanism.

### OBB: The Legacy Expansion File System

**Opaque Binary Blob** (OBB) files are the historical workaround. Each app may have up to two OBB files — a *main expansion file* and a *patch expansion file* — each up to 2 GB. They are stored on the device at `/sdcard/Android/obb/<package_name>/main.<version_code>.<package_name>.obb` (and `patch.` prefix for the patch file).

OBB files are ZIP archives. The Android SDK provides the **APK Expansion Zip Library** (`com.android.vending.expansion.zipfile`) which mounts an OBB file as a virtual filesystem, allowing `ZipResourceFile.getInputStream()` access to individual assets without extracting them. From the NDK, OBB contents accessible via `AAssetManager` are limited to install-time assets; OBB files must be read directly via POSIX `open()`/`read()` or through a JNI bridge to the Java ZIP library.

The developer must distribute OBB files separately to Google Play (uploaded alongside the APK) and handle download orchestration manually using the **APK Expansion Downloader Library** (`com.google.android.vending.expansion.downloader`). The Downloader library handles Wi-Fi vs. cellular download prompting, background download notifications, and retry on failure. As of 2020, this approach is deprecated in favour of Play Asset Delivery for new titles, but OBB support remains in the Play Store backend indefinitely for legacy apps. [Source: APK Expansion Files guide](https://developer.android.com/google/play/expansion-files)

### Play Asset Delivery (PAD)

Google introduced **Play Asset Delivery** at Google I/O 2020 as the recommended replacement for OBB files. PAD integrates with the **Android App Bundle** (`.aab`) format and provides three delivery modes with distinct characteristics:

**Install-time packs**: Assets included in the app bundle's base module or explicit install-time asset pack modules. They are always available on-device immediately after install, require no runtime download API calls, and can be read via `AAssetManager_open()` exactly like APK assets. The size cap for install-time content (base + install-time packs combined) is approximately 1 GB compressed. This is the right mode for core game assets that must be available at first launch.

**Fast-follow packs**: Downloaded automatically immediately after install completes, before the user opens the app for the first time. By the time the user taps the game icon, fast-follow assets are typically already present. Each fast-follow pack may be up to 512 MB. Use this mode for the first level, tutorial content, and startup screen assets that cannot afford a loading screen on first run.

**On-demand packs**: Downloaded at runtime when the game explicitly requests them. Suited for later levels, DLC-equivalent content, language packs, or regional content variants. Each on-demand pack may be up to 512 MB; a game may have many on-demand packs. [Source: Play Asset Delivery overview](https://developer.android.com/guide/playcore/asset-delivery)

### Declaring Asset Packs in Gradle

In an Android App Bundle project, each asset pack is a separate Gradle module:

```groovy
// settings.gradle
include ':app', ':mainAssets', ':level2Assets', ':cinematics'
```

```groovy
// mainAssets/build.gradle
plugins { id 'com.android.asset-pack' }
assetPack {
    packName = "mainAssets"
    dynamicDelivery {
        deliveryType = "install-time"
    }
}
```

```groovy
// app/build.gradle
android {
    assetPacks = [":mainAssets", ":level2Assets", ":cinematics"]
}
```

The `deliveryType` for each pack is `"install-time"`, `"fast-follow"`, or `"on-demand"`.

### Accessing Asset Packs from C++

Install-time asset packs are accessible through the standard `AAssetManager` NDK API, since their contents are merged into the APK's asset namespace at install time:

```c
AAssetManager* assetMgr = app->activity->assetManager;
AAsset* asset = AAssetManager_open(assetMgr, "textures/ground.ktx2", AASSET_MODE_BUFFER);
const void* data = AAsset_getBuffer(asset);
size_t size = AAsset_getLength(asset);
// ... upload to GPU ...
AAsset_close(asset);
```

On-demand and fast-follow packs require the **Play Core native library** (`play/asset_packs.h`), distributed as part of the Play Core C API:

```cpp
#include "play/asset_packs.h"

// Initialise once with JVM and JNIEnv:
AssetPackManager_init(java_vm, jni_env);

// Request download of an on-demand pack:
const char* pack_names[] = {"level2Assets"};
AssetPackRequest* req = AssetPackManager_requestDownload(pack_names, 1);
// req is not synchronous — poll for completion:

// Per-frame poll (or use async callback):
AssetPackDownloadState* state;
AssetPackManager_getDownloadState("level2Assets", &state);
AssetPackDownloadStatus status = AssetPackDownloadState_getStatus(state);
if (status == ASSET_PACK_DOWNLOAD_COMPLETED) {
    AssetPackLocation* location;
    AssetPackManager_getAssetPackLocation("level2Assets", &location);
    const char* path = AssetPackLocation_getAssetsPath(location);
    // path is a filesystem directory; open files directly with POSIX open()
    AssetPackLocation_destroy(location);
}
AssetPackDownloadState_destroy(state);
AssetPackRequest_delete(req);
```

Note: `AssetPackManager_getAssetsPath()` returns a `NULL`-terminated filesystem path string pointing to the unpacked asset directory. For large packs the system may store assets in compressed form in an internal directory; `AssetPackLocation_getStorageMethod()` returns `ASSET_PACK_STORAGE_APK_ASSET` (for packs installed as part of the APK) or `ASSET_PACK_STORAGE_FILES` (for packs stored as files on the filesystem). [Source: Play Core C API](https://developer.android.com/games/playgames/asset-delivery)

### Storage and Lifecycle

On-demand and fast-follow packs are stored in the device's internal storage, separate from the APK. They do **not** count against the 150 MB APK size limit. Packs can be removed when no longer needed to free space:

```cpp
AssetPackManager_removePack("cinematics");
```

The system may also evict on-demand packs under storage pressure; the game must handle `ASSET_PACK_DOWNLOAD_NOT_INSTALLED` gracefully and re-request packs that have been removed.

### Testing with bundletool

During development, the `.aab` format cannot be directly installed to a device — only the Play Store can process it into device-specific APK sets. The **bundletool** CLI provides local equivalents:

```bash
# Build a set of APKs from the app bundle:
bundletool build-apks --bundle=app.aab --output=my_app.apks \
    --ks=debug.keystore --ks-pass=pass:android \
    --ks-key-alias=androiddebugkey --key-pass=pass:android

# Deploy to a connected device (installs the correct split APKs):
bundletool install-apks --apks=my_app.apks

# Simulate fast-follow / on-demand delivery locally:
bundletool install-apks --apks=my_app.apks \
    --modules=mainAssets,level2Assets
```

[Source: bundletool documentation](https://developer.android.com/tools/bundletool)

---

## 3. NativeActivity: The Foundation and Its Limits

Understanding `NativeActivity` is prerequisite to understanding why `GameActivity` was needed. `NativeActivity` remains supported and is still used in simpler apps, but its limitations are load-bearing motivations for everything in §4 onwards.

### The android_native_app_glue threading model

`android.app.NativeActivity` is a standard Java `Activity` subclass. On startup it loads the native `.so` specified in `AndroidManifest.xml` and resolves the C entry point `ANativeActivity_onCreate` by name (overridable via the `android.app.func_name` metadata key).

The NDK-provided `android_native_app_glue` library (`sources/android/native_app_glue/`) wraps the raw `ANativeActivityCallbacks` mechanism with a more ergonomic execution model: it spawns a **dedicated background pthread** for `android_main()`, separating the app's C loop from the Java main thread. Android lifecycle callbacks (from the Java main thread) are serialised as `APP_CMD_*` integers into a pipe; the background thread reads them via `ALooper_pollOnce()`. This ensures framework callbacks never stall long enough to trigger an ANR (Application Not Responding watchdog, typically 5 seconds).

```c
// sources/android/native_app_glue/android_native_app_glue.h (simplified)
struct android_app {
    void*             userData;
    void (*onAppCmd)(struct android_app*, int32_t cmd);   // APP_CMD_* handler
    int32_t (*onInputEvent)(struct android_app*, AInputEvent*); // 1 = consumed
    ANativeActivity*  activity;     // Java NativeActivity wrapper
    AConfiguration*   config;
    void*             savedState;
    size_t            savedStateSize;
    ALooper*          looper;       // LOOPER_ID_MAIN=1, LOOPER_ID_INPUT=2, USER=3+
    AInputQueue*      inputQueue;
    ANativeWindow*    window;       // non-NULL when surface is ready (see §9)
    ARect             contentRect;
    int               activityState;
    int               destroyRequested;
};

extern void android_main(struct android_app* app);  // application entry point
```

The `ALooper` is the multiplexed event source: `ALooper_pollOnce(timeoutMs, &fd, &events, &data)` blocks until a command or input event is ready, then calls the associated `android_poll_source::process` callback. (See §4 for how `GameActivity` replaces this model.)

### ANativeActivityCallbacks: all 16 lifecycle points

All callbacks fire on the **Java main thread** and all default to NULL. The glue library fills in its own implementations that write `APP_CMD_*` integers to the pipe:

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

### NativeActivity's three hard limitations

**1. Broken text input.** `ANativeActivity_showSoftInput()` can raise the IME keyboard, but there is no C callback to receive the resulting text — it is consumed by the Java `InputConnection` layer with no NDK bridge. Games requiring username entry or chat must invoke JNI to read Java `EditText` contents, defeating the zero-JNI goal.

**2. No `SurfaceView` control.** `NativeActivity` owns the entire window surface; it is impossible to compose native rendering alongside a standard Android `View` hierarchy (e.g., a Jetpack Compose UI overlay or an `AdView` for ads) without significant workarounds.

**3. Coarse input event batching.** Touch events arrive via `AInputQueue` one at a time per `ALooper_pollOnce` iteration. High-frequency stylus or multi-touch events are batched by the system and historical samples are not exposed through the C API — the Java `MotionEvent.getHistoricalX()` path has no NDK equivalent in `NativeActivity`. Section §5 covers how `GameActivity` resolves this with its double-buffered input model and full historical sample exposure.

---

## 4. GameActivity: The Modern Native App Model

`GameActivity` resolves all three `NativeActivity` limitations while retaining the same threading model and `ANativeWindow` lifecycle.

### Kotlin/Java subclass requirement

Unlike `NativeActivity` (which is final and usable directly), `GameActivity` must be subclassed:

```kotlin
// MainActivity.kt
import com.google.androidgamesdk.GameActivity

class MainActivity : GameActivity() {
    // Override to customise window flags, edge-to-edge, etc.
    override fun onWindowFocusChanged(hasFocus: Boolean) {
        super.onWindowFocusChanged(hasFocus)
        if (hasFocus) {
            window.decorView.systemUiVisibility =
                android.view.View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY or
                android.view.View.SYSTEM_UI_FLAG_FULLSCREEN or
                android.view.View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
        }
    }
}
```

The native entry-point symbol changes from `ANativeActivity_onCreate` to `Java_com_google_androidgamesdk_GameActivity_initializeNativeCode`. The linker flag must reflect this:

```cmake
target_link_options(mygame PRIVATE
    -u Java_com_google_androidgamesdk_GameActivity_initializeNativeCode)
```

Replace the classic glue source with the AGDK version:
```cmake
# Include AGDK's native_app_glue.c (not NDK's android_native_app_glue.c)
target_sources(mygame PRIVATE
    ${GAME_ACTIVITY_INCLUDE}/game-activity/native_app_glue/android_native_app_glue.c)
```

### The AGDK android_app struct

```c
struct android_app {
    GameActivity* activity;       // GameActivity* replaces ANativeActivity*
    ANativeWindow* window;        // same as classic — non-NULL when surface ready
    ALooper*       looper;
    AConfiguration* config;
    void*           userData;
    void (*onAppCmd)(struct android_app*, int32_t);

    // Input arrays — replaces AInputQueue callback entirely:
    GameActivityMotionEvent motionEvents[NATIVE_APP_GLUE_MAX_NUM_MOTION_EVENTS];
    uint64_t                motionEventsCount;
    GameActivityMotionEvent motionEventsFilter;   // bitmask of allowed action types
    GameActivityKeyEvent    keyDownEvents[NATIVE_APP_GLUE_MAX_NUM_KEY_EVENTS];
    uint64_t                keyDownEventsCount;
    GameActivityKeyEvent    keyUpEvents[NATIVE_APP_GLUE_MAX_NUM_KEY_EVENTS];
    uint64_t                keyUpEventsCount;
    int                     textInputState;        // 1 = pending text from IME

    void*  savedState;
    size_t savedStateSize;
    int    activityState;
    int    destroyRequested;
};
```

`GameActivity` struct replaces `ANativeActivity` and renames the mis-named `clazz` field:

```c
struct GameActivity {
    JavaVM*                       vm;
    JNIEnv*                       env;               // main-thread JNI only
    jobject                       javaGameActivity;  // NOT 'clazz' — corrected name
    const char*                   internalDataPath;
    const char*                   externalDataPath;
    const char*                   obbPath;
    int32_t                       sdkVersion;
    void*                         instance;
    struct GameActivityCallbacks* callbacks;
    AAssetManager*                assetManager;
};
```

[Source: GameActivity struct](https://developer.android.com/reference/games/game-activity/struct/game-activity), [android_app struct](https://developer.android.com/reference/games/game-activity/struct/android-app)

### Lifecycle event ordering: surface vs. resume

The critical ordering rule for Vulkan — identical to `NativeActivity` — is that **`APP_CMD_INIT_WINDOW` is the correct place to create the swapchain, not `APP_CMD_RESUME`**. The surface is created after `onResume` fires, and it can be destroyed while the activity remains in resumed state (e.g., when a system overlay covers the window). `android_app::window` is the authoritative signal:

| `APP_CMD_*` | `window` state | Vulkan action |
|---|---|---|
| `APP_CMD_START` | may be NULL | — |
| `APP_CMD_RESUME` | may be NULL | — |
| `APP_CMD_INIT_WINDOW` | **non-NULL** | Create `VkSurfaceKHR` + swapchain |
| `APP_CMD_GAINED_FOCUS` | non-NULL | Begin render loop |
| `APP_CMD_LOST_FOCUS` | non-NULL | Pause render loop |
| `APP_CMD_TERM_WINDOW` | about to be NULL | **Destroy swapchain + `VkSurfaceKHR`** |
| `APP_CMD_PAUSE` | NULL or non-NULL | — |
| `APP_CMD_STOP` | NULL | — |
| `APP_CMD_SAVE_STATE` | NULL | Serialise game state |
| `APP_CMD_DESTROY` | NULL | Destroy all Vulkan resources |

```c
void handle_cmd(struct android_app* app, int32_t cmd) {
    switch (cmd) {
    case APP_CMD_INIT_WINDOW:
        if (app->window) engine_init_vulkan(app->window);
        break;
    case APP_CMD_TERM_WINDOW:
        engine_destroy_vulkan();
        break;
    case APP_CMD_GAINED_FOCUS:
        g_rendering = true;
        break;
    case APP_CMD_LOST_FOCUS:
        g_rendering = false;
        break;
    }
}
```

---

## 5. Input Handling with GameActivity

### Double-buffered input model

`NativeActivity` processes input one event at a time via `ALooper_pollOnce`. `GameActivity` replaces this with a **double-buffered array** model: the framework fills one buffer while the app reads the other, then the app calls `android_app_swap_input_buffers()` to atomically exchange them. This exposes all historical samples and eliminates the ANR risk from a backed-up `AInputQueue`.

```c
// Per frame, before rendering:
android_app_swap_input_buffers(app);  // swap front/back

// Motion events (touch, stylus, mouse, joystick):
for (uint64_t i = 0; i < app->motionEventsCount; i++) {
    GameActivityMotionEvent* e = &app->motionEvents[i];
    int32_t action  = e->action & AMOTION_EVENT_ACTION_MASK;
    int32_t ptrIdx  = (e->action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK)
                      >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;

    float x = GameActivityPointerAxes_getAxisValue(
                  &e->pointers[ptrIdx], AMOTION_EVENT_AXIS_X);
    float y = GameActivityPointerAxes_getAxisValue(
                  &e->pointers[ptrIdx], AMOTION_EVENT_AXIS_Y);

    // Historical samples — exposed, unlike NativeActivity:
    for (int32_t h = 0; h < e->historicalCount; h++) {
        float hx = GameActivityPointerAxes_getAxisValue(
                       &e->historicalPointers[h * e->pointerCount + ptrIdx],
                       AMOTION_EVENT_AXIS_X);
    }
}
android_app_clear_motion_events(app);

// Key events:
for (uint64_t i = 0; i < app->keyDownEventsCount; i++) {
    GameActivityKeyEvent* e = &app->keyDownEvents[i];
    // e->keyCode (AKEYCODE_*), e->action, e->metaState
}
android_app_clear_key_events(app);
```

### Enabling axes and event filters

By default `GameActivity` only delivers finger touch events. To receive controller axes (joystick, trigger) or stylus data, the axis must be explicitly enabled:

```c
// Enable joystick and trigger axes for Paddleboat/controller input:
android_app_set_motion_event_filter(app, NULL);  // NULL = receive all devices

// Or for fine-grained control, check Paddleboat's active axis mask:
uint64_t axisMask = Paddleboat_getActiveAxisMask();
for (int axis = 0; axis < GAME_ACTIVITY_POINTER_INFO_AXIS_COUNT; axis++) {
    if (axisMask & (1ULL << axis))
        GameActivityPointerAxes_enableAxis(axis);
}
```

### GameTextInput: working IME integration

`game-text-input` (bundled with `game-activity`) provides the IME bridge that `NativeActivity` lacks. The app configures the editor and reads results in C:

```c
// Configure IME (call once, or when editor changes):
GameActivity_setImeEditorInfo(app->activity,
    IME_ACTION_DONE,          // action button label
    IME_FLAG_NO_FULLSCREEN,   // flags
    0);                       // inputType bitmask

GameActivity_showSoftInput(app->activity, 0);

// Check and read pending text (per-frame):
if (app->textInputState) {
    GameTextInputState state;
    GameActivity_getTextInputState(app->activity,
        [](void* ctx, const GameTextInputState* s) {
            // s->text_UTF8, s->text_length, s->selection_start/end
            memcpy(((MyApp*)ctx)->inputBuffer, s->text_UTF8, s->text_length);
        }, myApp);
}
```

[Source: GameActivity input guide](https://developer.android.com/games/agdk/game-activity/migrate-native-activity)

---

## 6. Touch Input Prediction and End-to-End Latency Reduction

Rendering at 120 frames per second means each frame has just 8.33 ms to complete. Even if the GPU hits that budget consistently, touch input still suffers from independent latency sources that stack: the touch sensor polling period, OS input batching, the rendering pipeline, and display scanout delay. For an action game or drawing application, the gap between where the user's finger is and where the game draws the cursor is the most viscerally noticeable performance issue — more so than framerate jitter. This section covers the anatomy of touch-to-display latency and the APIs available in AGDK to reduce it.

### Sources of Touch-to-Display Latency

The total latency from finger contact to visible pixel change has five components:

1. **Touch sensor sampling period**: The touchscreen controller polls the capacitive surface at a fixed rate. At 60 Hz displays this is typically 8–12 ms; at 120 Hz or 144 Hz displays, manufacturers commonly increase touch sampling to 240–360 Hz (2–4 ms poll period). This is the irreducible floor.

2. **OS input batching**: The input subsystem does not deliver every touch sample immediately to the app. It batches events within a vsync interval and delivers them as a single `MotionEvent` containing multiple historical samples. With `GameActivity`'s `android_app_swap_input_buffers()` called once per frame, the app sees all events from the previous vsync interval at once — correct, but introduces up to one vsync interval of additional delay.

3. **Rendering pipeline depth**: Most games buffer 1–2 frames ahead to keep the GPU busy. With Swappy's pipeline mode (§8), CPU frame N+1 begins while GPU processes frame N. This adds approximately one frame (8.33 ms at 120 Hz, 16.6 ms at 60 Hz) of systematic input latency.

4. **Display scanout and panel response**: After the GPU writes to the framebuffer and SurfaceFlinger performs its composition, the display controller scans out pixels row by row. On OLED panels, pixel response time is sub-millisecond; on LCD the backlight transition can add 1–5 ms.

5. **Total measured latency**: On a Pixel 7 Pro (120 Hz, 240 Hz touch) with Swappy in non-pipeline mode and 120 Hz swap interval, end-to-end latency has been measured at approximately 22–28 ms. Pipeline mode adds 8 ms. A 60 Hz display in pipeline mode can reach 80 ms total. [Source: Android input lag guide](https://developer.android.com/guide/input/input-lag)

### Historical Samples: Processing All Batched Events

`GameActivity` exposes historical touch samples within a `GameActivityMotionEvent` through the `historicalPointers` and `historicalCount` fields. Processing all historical samples — not just the current position — is essential at high frame rates to avoid "skipping" input at 120 Hz when the app only renders at 60 Hz:

```cpp
for (uint64_t i = 0; i < app->motionEventsCount; i++) {
    GameActivityMotionEvent* e = &app->motionEvents[i];
    if ((e->action & AMOTION_EVENT_ACTION_MASK) != AMOTION_EVENT_ACTION_MOVE)
        continue;

    // Process all historical samples first (oldest to newest):
    for (int32_t h = 0; h < e->historicalCount; h++) {
        int32_t base = h * e->pointerCount; // index into historicalPointers array
        float hx = GameActivityPointerAxes_getAxisValue(
                       &e->historicalPointers[base + 0],
                       AMOTION_EVENT_AXIS_X);
        float hy = GameActivityPointerAxes_getAxisValue(
                       &e->historicalPointers[base + 0],
                       AMOTION_EVENT_AXIS_Y);
        int64_t event_time_ns = e->historicalEventTimesMillis[h] * 1'000'000LL;
        game_record_touch_sample(hx, hy, event_time_ns);
    }

    // Then process the current (most recent) position:
    float x = GameActivityPointerAxes_getAxisValue(
                   &e->pointers[0], AMOTION_EVENT_AXIS_X);
    float y = GameActivityPointerAxes_getAxisValue(
                   &e->pointers[0], AMOTION_EVENT_AXIS_Y);
    game_record_touch_sample(x, y, e->eventTime);
}
```

Skipping historical processing at 120 Hz touch sampling means discarding up to 3 intermediate positions per rendered frame — visually obvious as "stair-stepping" in drawing apps and imprecise aim in action games.

### Input Prediction (Android 12+, API 34 via MotionPredictor)

Android 12 (API 31) introduced the concept of input prediction; Android 14 (API 34) formalised it through the `android.view.MotionPredictor` class, which predicts future stylus and touch positions based on recent motion history. `MotionPredictor` is a Java/Kotlin API — from a `GameActivity`-based native game, it must be called via JNI or from the Kotlin `MainActivity` subclass:

```kotlin
// In MainActivity.kt (GameActivity subclass), API 34+
import android.view.MotionPredictor

private val motionPredictor = MotionPredictor()

// Called from native via JNI to feed the predictor with each actual event:
fun feedMotionEvent(event: MotionEvent) {
    motionPredictor.record(event)
}

// Called from native to get predicted position one frame ahead:
fun getPredictedPosition(eventTime: Long): FloatArray? {
    val predicted = motionPredictor.predict(eventTime) ?: return null
    return floatArrayOf(predicted.x, predicted.y)
}
```

The predicted event's timestamp should be set to `current_frame_vsync_time + one_frame_duration_ns` — predict where the finger will be at the *next* vsync, and render the cursor there. When the next actual event arrives, correct any prediction error by animating the cursor to the true position. [Source: MotionPredictor API reference](https://developer.android.com/reference/android/view/MotionPredictor)

**Note:** As of `games-activity` 3.0, the `GameActivityPointerAxes` struct does not include a native predicted-position field. Prediction requires either the Java `MotionPredictor` path via JNI, or a custom extrapolation filter in native code (see below).

### Kalman Filter Extrapolation (Pre-API 34 / Custom)

For games targeting API levels below 34, or requiring a purely native solution, a simple velocity-extrapolation filter achieves similar results:

```cpp
struct TouchPredictor {
    float last_x = 0, last_y = 0;
    float vel_x = 0, vel_y = 0;
    int64_t last_time_ns = 0;

    void update(float x, float y, int64_t time_ns) {
        if (last_time_ns > 0) {
            float dt = (time_ns - last_time_ns) * 1e-9f;
            // Low-pass filter on velocity (alpha = 0.6)
            vel_x = 0.6f * (x - last_x) / dt + 0.4f * vel_x;
            vel_y = 0.6f * (y - last_y) / dt + 0.4f * vel_y;
        }
        last_x = x; last_y = y; last_time_ns = time_ns;
    }

    // Extrapolate position dt_ns nanoseconds into the future:
    std::pair<float,float> predict(int64_t dt_ns) const {
        float dt = dt_ns * 1e-9f;
        return {last_x + vel_x * dt, last_y + vel_y * dt};
    }
};
```

Clamp the predicted position to the screen bounds and limit the extrapolation horizon to one frame interval to avoid over-shooting on deceleration. Apply a correction animation (lerp back to actual position) on the subsequent frame to avoid visible snapping.

### High-Refresh-Rate Display Configuration

To benefit from 120 Hz touch sampling, the app must also run at 120 Hz. Swappy (§8) handles the frame-pacing side; the display refresh rate must be explicitly requested:

```kotlin
// In GameActivity subclass (Kotlin):
// Android 12+ preferred display mode:
val display = windowManager.defaultDisplay
val modes = display.supportedModes
val highRefreshMode = modes.maxByOrNull { it.refreshRate }
if (highRefreshMode != null) {
    window.attributes = window.attributes.also {
        it.preferredDisplayModeId = highRefreshMode.modeId
    }
}
```

Then in native code, configure Swappy for the selected refresh rate:

```cpp
// 120 Hz = 8,333,333 ns swap interval:
SwappyVk_setSwapIntervalNS(device, swapchain, 8'333'333LL);
```

On devices where the display operates at 120 Hz, the touch digitiser also polls at 240 Hz or higher, approximately halving the touch sampling period from the 60 Hz baseline. [Source: Frame Pacing library](https://developer.android.com/games/sdk/frame-pacing/input-buffering)

### Measuring End-to-End Latency

Use `adb` to timestamp touch events and correlate with frame delivery:

```bash
# Find the touch device input node:
adb shell getevent -lp | grep -A 20 "ABS_MT_POSITION_X"

# Record timestamped raw events:
adb shell getevent -lt /dev/input/event3 2>/dev/null | head -200
```

Each `getevent` line includes a kernel timestamp in microseconds. Compare these with `FrameInfo` data from `adb shell dumpsys gfxinfo <package> framestats` to compute total latency. Target values: casual games <50 ms, drawing/stylus apps <20 ms, action games <30 ms.

[Source: Android input lag optimisation](https://developer.android.com/games/sdk/frame-pacing/input-buffering)

---

## 7. Paddleboat: Game Controller Library

`Paddleboat` (`androidx.games:games-controller`) solves Android's game controller fragmentation problem: the NDK's `AInputQueue` delivers raw `AInputEvent` objects with keycodes and axis values, but axis mapping, button labelling, vibration support, and multi-controller enumeration vary wildly across OEM controllers, generic HID gamepads, and first-party hardware (PS5 DualSense, Xbox, Switch Pro).

### Initialization and per-frame update

```c
#include <paddleboat/paddleboat.h>

// Init once (pass JNI env and Activity jobject):
Paddleboat_ErrorCode err = Paddleboat_init(env, app->activity->javaGameActivity);

// Per frame — processes pending controller connect/disconnect events:
Paddleboat_update(env);
```

### Forwarding GameActivity events

Paddleboat acts as a filter on top of the GameActivity input arrays:

```c
for (uint64_t i = 0; i < app->motionEventsCount; i++) {
    // Returns 1 if Paddleboat consumed the event (controller input)
    if (!Paddleboat_processGameActivityMotionInputEvent(
            &app->motionEvents[i], sizeof(GameActivityMotionEvent))) {
        // Not a controller — handle as touch
        handle_touch(&app->motionEvents[i]);
    }
}
for (uint64_t i = 0; i < app->keyDownEventsCount; i++) {
    if (!Paddleboat_processGameActivityKeyInputEvent(
            &app->keyDownEvents[i], sizeof(GameActivityKeyEvent))) {
        handle_key(&app->keyDownEvents[i]);
    }
}
```

### Reading controller state

```c
typedef struct Paddleboat_Controller_Data {
    uint64_t  timestamp;         // microseconds since epoch
    uint32_t  buttonsDown;       // PADDLEBOAT_BUTTON_* bitmask
    struct {
        float stickX, stickY;   // -1.0 to 1.0
    } leftStick, rightStick;
    float     triggerL1, triggerL2, triggerR1, triggerR2;  // 0.0 to 1.0
    struct {
        float pointerX, pointerY;
    } virtualPointer;
} Paddleboat_Controller_Data;

Paddleboat_Controller_Data ctrl;
if (Paddleboat_getControllerData(0 /*index*/, &ctrl) == PADDLEBOAT_NO_ERROR) {
    if (ctrl.buttonsDown & PADDLEBOAT_BUTTON_A) jump();
    move_character(ctrl.leftStick.stickX, ctrl.leftStick.stickY);
    aim_camera(ctrl.rightStick.stickX, ctrl.rightStick.stickY);
}
```

### Vibration and lights

```c
Paddleboat_Vibration_Data vib = {
    .durationLeft  = 200,   // ms
    .durationRight = 200,
    .intensityLeft = 0.8f,
    .intensityRight = 0.4f,
};
Paddleboat_setControllerVibrationData(0, &vib, env);

// Controller info (name, vendorId, productId, capabilities):
Paddleboat_Controller_Info info;
Paddleboat_getControllerInfo(0, &info);
// info.controllerFlags & PADDLEBOAT_CONTROLLER_FLAG_VIBRATION
// info.controllerFlags & PADDLEBOAT_CONTROLLER_FLAG_TOUCHPAD
```

### Controller connect/disconnect callbacks

```c
Paddleboat_setControllerStatusCallback(
    [](int32_t idx, Paddleboat_ControllerStatus status, void* ctx) {
        if (status == PADDLEBOAT_CONTROLLER_JUST_CONNECTED)
            on_controller_connected(idx);
        else if (status == PADDLEBOAT_CONTROLLER_JUST_DISCONNECTED)
            on_controller_disconnected(idx);
    }, myApp);
```

[Source: Game Controller Library](https://developer.android.com/games/sdk/game-controller)

---

## 8. Android Frame Pacing: Swappy

**Swappy** (`games-frame-pacing`) solves two jank patterns that arise when a native app calls `vkQueuePresentKHR` (or `eglSwapBuffers`) without vsync awareness.

### The two jank problems

**Double-present jank**: A short frame (B) completes early before the next vsync boundary. The preceding frame (A) may be presented at the same vsync as B, causing A to display for only one vsync interval instead of two, then B to also display for one — creating a stutter cadence of 16 ms / 16 ms instead of the intended 33 ms / 33 ms (at 60 Hz targeting 30 fps).

**Buffer stuffing**: A long frame causes the `BufferQueue` between the app and SurfaceFlinger to fill. The app cannot detect this backpressure, so it keeps submitting work. The `BufferQueue` depth grows, and each frame displayed is older than it should be — input latency increases by one full display refresh period per queued buffer.

Swappy addresses both:
- **Short frames**: injects `VkPresentTimesInfoGOOGLE` (`VK_GOOGLE_display_timing`) into the `VkPresentInfoKHR.pNext` chain to schedule presentation at the correct future vsync, preventing early display.
- **Long frames / buffer stuffing**: inserts `VkFence` waits on the CPU when the `BufferQueue` is full, preventing buffer accumulation.
- **Multi-refresh-rate displays**: on 60/90/120 Hz panels, Swappy auto-selects the closest achievable refresh rate. A game running at 45 fps targets 90 Hz (every 2 vsyncs) rather than dropping to 30 fps on 60 Hz.

### Vsync calibration via Choreographer

Swappy uses the NDK `Choreographer` API (API 24+) or Java `Choreographer` (API 16+) to obtain per-device vsync calibration: the `AppVsyncOffsetNanos` (how far ahead of vsync the app should start work) and `PresentationDeadlineNanos` (last moment to submit a frame for the current vsync). These are queried via JNI from `WindowManager.getDefaultDisplay()` during `SwappyVk_initAndGetRefreshCycleDuration`. [Source: Frame Pacing library](https://developer.android.com/games/sdk/frame-pacing)

### SwappyVk integration sequence

```c
#include <swappy/swappyVk.h>

// Step 1 — before vkCreateDevice: query required device extensions
// Swappy adds VK_GOOGLE_display_timing if the driver supports it
uint32_t reqCount = 0;
SwappyVk_determineDeviceExtensions(physicalDevice,
    availExtCount, pAvailExts, &reqCount, NULL);
const char** ppReqExts = malloc(reqCount * sizeof(char*));
SwappyVk_determineDeviceExtensions(physicalDevice,
    availExtCount, pAvailExts, &reqCount, ppReqExts);
// Add ppReqExts to VkDeviceCreateInfo::ppEnabledExtensionNames

// Step 2 — after vkCreateSwapchainKHR:
uint64_t refreshDurationNs;
SwappyVk_initAndGetRefreshCycleDuration(
    env, app->activity->javaGameActivity,
    physicalDevice, device, swapchain, &refreshDurationNs);

// Step 3 — bind ANativeWindow for display-timing queries
SwappyVk_setWindow(device, swapchain, app->window);

// Step 4 — set target swap interval (nanoseconds)
// 16_666_666 = 60 Hz, 11_111_111 = 90 Hz, 8_333_333 = 120 Hz
SwappyVk_setSwapIntervalNS(device, swapchain, refreshDurationNs);

// Step 5 — replace vkQueuePresentKHR in the render loop
VkResult result = SwappyVk_queuePresent(queue, &presentInfo);
// Swappy injects VkPresentTimesInfoGOOGLE into presentInfo.pNext
// and calls vkQueuePresentKHR on the app's behalf

// Cleanup — before vkDestroySwapchainKHR:
SwappyVk_destroySwapchain(device, swapchain);
```

[Source: SwappyVk API reference](https://developer.android.com/games/sdk/reference/frame-pacing/group/swappy-vk)

### SwappyGL for OpenGL ES apps

For apps using GLES/EGL, the integration is identical in structure but uses `SwappyGL_*` functions:

```c
#include <swappy/swappyGL.h>

SwappyGL_init(env, app->activity->javaGameActivity);
SwappyGL_setWindow(app->window);
// Replace eglSwapBuffers:
SwappyGL_swap(display, surface);
SwappyGL_destroy();
```

### Pipeline modes

Swappy supports three pipeline depth modes:
- **Pipeline mode** (default): CPU frame N+1 overlaps GPU frame N across vsync boundaries — maximises throughput, adds ~16 ms input-to-display latency at 60 Hz.
- **Non-pipeline mode**: CPU and GPU both complete within one swap interval — minimises latency at the cost of GPU idle time.
- **Auto mode**: Swappy measures per-frame CPU and GPU durations and dynamically switches between pipeline depths and swap intervals to hit the best achievable frame rate on the device.

```c
SwappyVk_setAutoPipelineMode(true);  // enable auto-mode
// Or manually:
SwappyVk_enableStats(true);
SwappyVk_recordFrameStart(device, swapchain);
// ... render ...
SwappyVk_queuePresent(queue, &presentInfo);
```

---

## 9. ANativeWindow: The Surface Handle

`ANativeWindow*` is the C handle through which all native rendering — Vulkan, EGL, or software — accesses the Android display surface. It is the producer end of a `BufferQueue`; SurfaceFlinger is the consumer. The same handle is used by both `NativeActivity` and `GameActivity` via `android_app::window`.

### Internal architecture

Internally, `ANativeWindow*` points to a C++ `Surface` object in `frameworks/native`. `Surface` implements the `ANativeWindow` C vtable — casting a `Surface*` to `ANativeWindow*` is the bridge. The `Surface` holds a `BufferQueueProducer` that talks to SurfaceFlinger via Binder IPC (`IGraphicBufferProducer`). From the app's perspective, the buffer flow is:

```
App (NDK) → ANativeWindow → BufferQueue producer
                              ↓ Binder IPC
                           SurfaceFlinger (consumer)
                              ↓
                           HWComposer → DRM atomic commit → Display
```

### Reference counting

Both Vulkan and EGL manage surface lifetime through `acquire`/`release`:

```c
ANativeWindow_acquire(window);   // increment refcount
ANativeWindow_release(window);   // decrement; may free underlying Surface
```

`vkCreateAndroidSurfaceKHR` calls `ANativeWindow_acquire` internally. `vkDestroySurfaceKHR` calls `ANativeWindow_release`. A single `ANativeWindow` cannot be simultaneously bound to both a `VkSurfaceKHR` and an `EGLSurface` — creating the second will fail or produce undefined behaviour.

### Geometry and pixel format

```c
int32_t w = ANativeWindow_getWidth(window);
int32_t h = ANativeWindow_getHeight(window);
int32_t fmt = ANativeWindow_getFormat(window);

// Override buffer geometry (pass 0,0 to use window's natural size):
ANativeWindow_setBuffersGeometry(window, 0, 0,
    WINDOW_FORMAT_RGBA_8888);  // = AHARDWAREBUFFER_FORMAT_R8G8B8A8_UNORM (1)
                               // WINDOW_FORMAT_RGBX_8888 (2)
                               // WINDOW_FORMAT_RGB_565 (4)
```

### Vulkan integration

```c
// Obtain once at APP_CMD_INIT_WINDOW — no JNI needed when using GameActivity:
ANativeWindow* anw = app->window;

VkAndroidSurfaceCreateInfoKHR surfaceInfo = {
    .sType  = VK_STRUCTURE_TYPE_ANDROID_SURFACE_CREATE_INFO_KHR,
    .window = anw,
};
vkCreateAndroidSurfaceKHR(instance, &surfaceInfo, NULL, &vkSurface);
// All subsequent Vulkan calls: pure C, zero JNI per frame
```

When obtained from a `SurfaceHolder` or `SurfaceTexture` in a hybrid Java/C++ app, a single `ANativeWindow_fromSurface(env, jSurface)` call crosses JNI once; the handle is then valid for the render lifetime.

### EGL integration

```c
// EGLNativeWindowType is typedef'd to ANativeWindow* on Android
EGLSurface eglSurface = eglCreateWindowSurface(
    display, config, (EGLNativeWindowType)anw, NULL);
```

### CPU software rendering path

For 2D software rendering, screenshot readback, or video frame stamping:

```c
ANativeWindow_Buffer buf;
ARect dirty = {0, 0, w, h};
if (ANativeWindow_lock(window, &buf, &dirty) == 0) {
    // buf.bits: raw pixel pointer; buf.stride: row stride in pixels
    uint32_t* pixels = (uint32_t*)buf.bits;
    for (int y = 0; y < buf.height; y++)
        memset(pixels + y * buf.stride, 0xFF, buf.width * 4);
    ANativeWindow_unlockAndPost(window);
}
```

[Source: ANativeWindow NDK reference](https://developer.android.com/ndk/reference/group/a-native-window)

---

## 10. Vulkan on TBDR Mobile GPUs: Architecture and Optimization

Desktop GPUs (NVIDIA, AMD) render with an **Immediate Mode Renderer** (IMR): geometry is processed and shaded in submission order, writing directly to DRAM-backed framebuffer memory. Mobile GPUs from Qualcomm (Adreno), ARM (Mali), Imagination (PowerVR), and Apple (GPU) all use **Tile-Based Deferred Rendering** (TBDR): the screen is divided into rectangular tiles (typically 16×16 or 32×32 pixels), geometry is binned into tile lists, and each tile is rendered to completion in fast on-chip SRAM before the result is written to DRAM. This architectural difference has profound implications for Vulkan API usage and performance optimisation.

### Tile Memory: The Critical Resource

Each mobile GPU contains a block of on-chip SRAM — the *tile memory* or *local buffer* — typically 2–8 MB. This SRAM holds the colour, depth, and stencil data for the current tile being rendered. Reading and writing tile memory costs approximately one-tenth the energy and is an order of magnitude faster than reading from main DRAM (LP5 DRAM at 77 GB/s on a Snapdragon 8 Gen 3, vs. tile memory at ~2 TB/s effective bandwidth per GPU core). Keeping rendering within tile memory rather than writing intermediates to DRAM is the single most impactful mobile GPU optimisation.

The key mechanism exposing tile memory to Vulkan is the **render pass load/store operation** pair.

### Render Pass Load/Store Operations

Every `VkAttachmentDescription` specifies a `loadOp` (what happens when the tile begins rendering) and a `storeOp` (what happens when the tile finishes). The choices are:

| Operation | Meaning | DRAM cost |
|---|---|---|
| `VK_ATTACHMENT_LOAD_OP_LOAD` | Read prior framebuffer content from DRAM into tile memory | Yes — expensive |
| `VK_ATTACHMENT_LOAD_OP_CLEAR` | Fill tile memory with the specified clear value | No — free |
| `VK_ATTACHMENT_LOAD_OP_DONT_CARE` | Tile memory starts with undefined content | No — free |
| `VK_ATTACHMENT_STORE_OP_STORE` | Write tile memory back to DRAM after rendering | Yes — expensive |
| `VK_ATTACHMENT_STORE_OP_DONT_CARE` | Discard tile memory content after rendering | No — free |

**The depth buffer** should almost always use `loadOp = CLEAR` and `storeOp = DONT_CARE`. A game that renders depth to DRAM at 1080p (1920×1080, D24S8 = 4 bytes) at 60 fps writes 1920 × 1080 × 4 × 60 = ~500 MB/s of depth data that will never be read again. `DONT_CARE` eliminates this entirely.

**The colour buffer** should use `loadOp = DONT_CARE` when the first rendering pass fills every pixel (which a fullscreen skybox or clear + opaque geometry pass always does). Use `loadOp = LOAD` only when compositing on top of a previously rendered background that is not re-rendered in this pass.

**MSAA resolve**: When using 4× MSAA, the multisample colour attachment should use `storeOp = DONT_CARE` and the single-sample resolve attachment uses `storeOp = STORE`. The resolve happens in tile memory — only the resolved image is written to DRAM, not the 4× expanded MSAA buffer.

```cpp
// Optimal depth attachment: clear on load, discard on store
VkAttachmentDescription depth_attachment = {
    .format         = VK_FORMAT_D24_UNORM_S8_UINT,
    .samples        = VK_SAMPLE_COUNT_1_BIT,
    .loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR,   // initialise tile
    .storeOp        = VK_ATTACHMENT_STORE_OP_DONT_CARE, // don't write to DRAM
    .stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE,
    .stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE,
    .initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED,
    .finalLayout    = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL,
};
```

On a 2340×1080 display at 60 fps, avoiding the depth store saves 2340 × 1080 × 4 bytes × 60 fps ≈ **610 MB/s** of DRAM writes — a measurable reduction in battery draw and a 1–3 ms per-frame improvement in GPU active time on Adreno 740. [Source: ARM Mali Vulkan Best Practices](https://developer.arm.com/documentation/101897/latest/)

### Subpasses and Input Attachments

Vulkan's subpass mechanism exists precisely to enable TBDR engines to express multi-pass effects (deferred shading, screen-space effects) without DRAM round-trips. A standard deferred shading pipeline on a desktop IMR GPU:

1. G-buffer pass: render position/normal/albedo to 3–4 full-resolution attachments → write to DRAM
2. Lighting pass: read G-buffer from DRAM, compute lighting → write to DRAM

On a TBDR GPU, steps 1 and 2 can be fused into a **single render pass with two subpasses** using *input attachments*. The G-buffer attachments use `storeOp = DONT_CARE` (they never leave tile memory) and are declared as `VK_DESCRIPTOR_TYPE_INPUT_ATTACHMENT` in the lighting subpass. The GPU reads G-buffer data from tile memory, not DRAM — the round-trip is eliminated entirely.

```cpp
// Subpass 0 — G-buffer fill
VkSubpassDescription gbuffer_subpass = {
    .pipelineBindPoint    = VK_PIPELINE_BIND_POINT_GRAPHICS,
    .colorAttachmentCount = 3,
    .pColorAttachments    = gbuffer_refs, // position, normal, albedo
};

// Subpass 1 — deferred lighting (reads G-buffer from tile memory)
VkSubpassDescription lighting_subpass = {
    .pipelineBindPoint      = VK_PIPELINE_BIND_POINT_GRAPHICS,
    .inputAttachmentCount   = 3,
    .pInputAttachments      = gbuffer_input_refs, // VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL
    .colorAttachmentCount   = 1,
    .pColorAttachments      = &colour_ref,
};
```

In the lighting fragment shader, G-buffer reads use `subpassLoad()` rather than `texture()`:

```glsl
#version 450
layout (input_attachment_index = 0, set = 0, binding = 0) uniform subpassInput positionAttachment;
layout (input_attachment_index = 1, set = 0, binding = 1) uniform subpassInput normalAttachment;
layout (input_attachment_index = 2, set = 0, binding = 2) uniform subpassInput albedoAttachment;

void main() {
    vec4 position = subpassLoad(positionAttachment);
    vec4 normal   = subpassLoad(normalAttachment);
    vec4 albedo   = subpassLoad(albedoAttachment);
    // ... lighting calculation ...
}
```

Splitting this into two render passes — a common mistake when porting a desktop engine — forces G-buffer data to DRAM between passes, adding 5–15 ms of bandwidth overhead on a full-resolution deferred renderer. [Source: Vulkan subpass guide](https://www.khronos.org/blog/streamlining-render-passes)

### VK_KHR_dynamic_rendering Trade-offs on TBDR

`VK_KHR_dynamic_rendering` (core in Vulkan 1.3) simplifies render pass management: `vkCmdBeginRendering` / `vkCmdEndRendering` eliminate `VkRenderPass` and `VkFramebuffer` objects. This is ergonomically attractive and fine for single-pass rendering or IMR desktop GPUs. However, dynamic rendering fundamentally does not support **subpass input attachments** — the mechanism required for TBDR tile-memory deferred shading. 

For a game with a deferred rendering pipeline targeting Android flagship hardware, the choice is:
- Use legacy render passes with subpasses: verbose API, but 5–15 ms better than the alternative.
- Use `VK_KHR_dynamic_rendering`: simpler code, each "pass" is a separate `vkCmdBeginRendering` call, G-buffer must round-trip through DRAM.

Most mobile game engines provide a backend switch: dynamic rendering for desktop, subpass-based render passes for Android TBDR targets. [Source: Vulkan 1.3 dynamic rendering spec](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCmdBeginRendering.html)

### Vendor-Specific Extensions

Beyond the core Vulkan optimisations, GPU vendors expose hardware-specific features:

**`VK_QCOM_render_pass_transform`**: Adreno GPUs can rotate the rendered image to match the display's physical orientation in hardware during presentation, at zero cost. Without this extension, the compositor must perform a rotation blit when the app renders in landscape but the display is physically portrait (a common case in foldables). Query via `vkGetPhysicalDeviceFeatures2` with `VkPhysicalDeviceRenderPassTransformFeaturesQCOM`. [Source: Qualcomm developer documentation](https://developer.qualcomm.com/software/adreno-gpu-sdk/tools)

**`VK_QCOM_image_processing`**: Exposes Adreno's on-chip image processing units for texture filtering, sharpening, and upscaling operations within the fragment shader. Useful for implementing FidelityFX Super Resolution (FSR) or sharpening post-process passes without a separate compute dispatch.

**`VK_ARM_rasterization_order_attachment_access`** and **`VK_EXT_rasterization_order_attachment_access`** (cross-vendor): Allow fragment shaders to read from the same attachment they are writing to, enabling order-independent transparency effects that stay within tile memory on TBDR GPUs.

### Pipeline Cache Serialisation

On Android, Vulkan pipeline compilation (translating SPIR-V to the GPU's ISA) happens at runtime, typically during the first frame where the pipeline is used. Unlike OpenGL ES (where the driver caches GLSL-to-ISA translation internally), Vulkan apps are responsible for managing the pipeline cache. An uncached pipeline compile on Adreno can take 50–200 ms — a visible hitch:

```cpp
// On first launch: create an empty cache
VkPipelineCacheCreateInfo cache_info = {
    .sType           = VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO,
    .initialDataSize = 0,
    .pInitialData    = nullptr,
};
vkCreatePipelineCache(device, &cache_info, nullptr, &pipeline_cache);

// On subsequent launches: load serialised cache from disk
std::vector<uint8_t> cache_data = load_file("pipeline_cache.bin");
cache_info.initialDataSize = cache_data.size();
cache_info.pInitialData    = cache_data.data();
vkCreatePipelineCache(device, &cache_info, nullptr, &pipeline_cache);

// After creating all pipelines: serialise back to disk
size_t cache_size = 0;
vkGetPipelineCacheData(device, pipeline_cache, &cache_size, nullptr);
std::vector<uint8_t> serialised(cache_size);
vkGetPipelineCacheData(device, pipeline_cache, &cache_size, serialised.data());
save_file("pipeline_cache.bin", serialised);
```

Include the device UUID (`VkPhysicalDeviceProperties::pipelineCacheUUID`) in the cache filename to avoid loading a cache built for a different GPU or driver version — the Vulkan spec requires the implementation to reject caches with mismatched UUIDs. [Source: Android Vulkan shader precompilation guide](https://developer.android.com/games/optimize/vulkan-precompile-shaders)

### Practical TBDR Optimisation Checklist

1. Explicitly specify `loadOp`/`storeOp` for every attachment — never leave them at default `LOAD`/`STORE`.
2. Use `CLEAR` + `DONT_CARE` for the depth buffer in every render pass.
3. Implement deferred shading with subpasses and `subpassLoad()`, not two separate render passes.
4. Validate with `VK_LAYER_KHRONOS_validation` and `VkLayerSettingsCreateInfoEXT` with `VK_VALIDATION_FEATURE_ENABLE_BEST_PRACTICES_EXT` enabled — the validation layer flags suboptimal load/store ops.
5. Never `vkCmdCopyImage` between adjacent render passes on the critical render path — it forces DRAM round-trips.
6. Serialise the pipeline cache to internal storage and reload on subsequent launches.

---

## 11. EGL Context Management: Thread Affinity, Shared Resources, and Context Loss

EGL is the binding layer between OpenGL ES and the Android window system (`ANativeWindow`). While Vulkan (§10) has superseded EGL for new titles, a large installed base of Android games uses OpenGL ES 3.2 via EGL, and understanding EGL's thread semantics and context-loss recovery is essential for any game that mixes EGL and Java UI components or uses a background asset loading thread.

### Thread Affinity: eglMakeCurrent

An EGL context is explicitly bound to the calling thread with `eglMakeCurrent`. The binding is thread-local state: only one context may be current on a given thread at a time, and a context may only be current on one thread at a time.

```c
// Bind context to calling thread:
EGLBoolean ok = eglMakeCurrent(display, draw_surface, read_surface, context);
// draw_surface: surface for glDrawArrays output
// read_surface: surface for glReadPixels source (usually same as draw)

// Check which context is current:
EGLContext current = eglGetCurrentContext();
assert(current == context);  // guard at start of render functions

// Release context from thread (unbind without binding a new one):
eglMakeCurrent(display, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
```

Calling `eglMakeCurrent` on thread B while the context is still current on thread A causes `EGL_BAD_ACCESS` (error 0x3002). The correct multi-thread pattern is: **always release before migrating**. In practice, most games keep the GL context permanently on the render thread and never migrate it.

### Multi-Thread Rendering: Shared Contexts

Background asset loading in OpenGL ES (uploading textures, compiling shaders) requires an `EGLContext` that shares object names with the main render context. Sharing is established at context creation time:

```c
// Main render context (created first):
EGLContext render_ctx = eglCreateContext(display, config, EGL_NO_CONTEXT, attribs);

// Background loader context — shares objects with render_ctx:
EGLint share_attribs[] = { EGL_CONTEXT_CLIENT_VERSION, 3, EGL_NONE };
EGLContext loader_ctx = eglCreateContext(display, config,
    render_ctx,  // <-- share_context
    share_attribs);
```

**Shared objects** (visible in both contexts): textures (`glGenTextures`), buffer objects (`glGenBuffers`), renderbuffer objects, shader objects, program objects, sync objects.

**Non-shared objects**: framebuffer objects (`glGenFramebuffers`), vertex array objects (`glGenVertexArrays`), transform feedback objects. Each context has its own set of these.

The background loader thread runs in its own `pthread`, calls `eglMakeCurrent(display, pbuffer_surface, pbuffer_surface, loader_ctx)` (using a 1×1 `EGL_PBUFFER_BIT` surface since it does not render to a window), uploads textures, and then signals the render thread.

### Cross-Context Synchronisation with EGLSync

After a background upload, the render thread must not sample the texture until the upload is complete. A GL `glFinish()` on the loader thread is correct but expensive (stalls the GPU). The preferred mechanism is `EGLSyncKHR`:

```c
// On the loader thread, after glTexImage2D / glBufferData:
EGLSyncKHR fence = eglCreateSyncKHR(display, EGL_SYNC_FENCE_KHR, NULL);
// fence is inserted at the current point in the loader context's command stream
glFlush();  // ensure commands are submitted (fence does not imply flush)

// Pass 'fence' to the render thread (e.g., via an atomic pointer or a queue)

// On the render thread, before sampling the texture:
// Option A: CPU wait (blocks calling thread)
EGLint result = eglClientWaitSyncKHR(display, fence,
    EGL_SYNC_FLUSH_COMMANDS_BIT_KHR,
    EGL_FOREVER_KHR);  // or a timeout in nanoseconds

// Option B: GPU wait (preferred — render thread GPU command stream waits, CPU continues)
eglWaitSyncKHR(display, fence, 0);  // API 30+, EGL 1.5

eglDestroySyncKHR(display, fence);
```

`eglWaitSyncKHR` inserts a GPU-side wait: the render context's subsequent draw calls are delayed until the loader context's fence is signalled, without blocking the CPU thread. This is the EGL equivalent of Vulkan's `vkCmdWaitEvents`. [Source: EGL 1.5 specification §3.8](https://registry.khronos.org/EGL/specs/eglspec.1.5.pdf)

### Context Loss: Triggers and Recovery

Android can invalidate the EGL surface or context under several conditions:

**1. App backgrounding** (`APP_CMD_STOP`): When the app is fully backgrounded (not visible), Android may destroy the `EGLSurface` associated with the window. The `EGLContext` itself often survives in this scenario — objects (textures, VBOs) are retained. The app must re-create the `EGLSurface` on `APP_CMD_INIT_WINDOW`.

**2. Display connection changes**: Plugging or unplugging an external display on a device that supports display mirroring can cause the surface to be destroyed and recreated.

**3. Configuration changes**: If `android:configChanges` in `AndroidManifest.xml` does not include `orientation|screenSize`, an orientation change destroys and recreates the `Activity`, destroying the EGL context entirely. With `configChanges` set, only the surface is resized via `APP_CMD_WINDOW_RESIZED`.

**4. Low memory / hardware preemption**: In extreme low-memory cases, Android may terminate the app's GPU context entirely. All GL objects (textures, VBOs, programs) must be recreated from scratch.

Recovery pattern in `android_main`:

```c
case APP_CMD_INIT_WINDOW:
    if (egl_context == EGL_NO_CONTEXT) {
        // Full initialisation — first launch or context was destroyed
        egl_display = eglGetDisplay(EGL_DEFAULT_DISPLAY);
        eglInitialize(egl_display, &major, &minor);
        egl_context = eglCreateContext(egl_display, config, EGL_NO_CONTEXT, attribs);
        // Recreate all textures, VBOs, programs:
        gl_upload_all_resources();
    }
    // Surface re-creation always needed here:
    egl_surface = eglCreateWindowSurface(egl_display, config,
        (EGLNativeWindowType)app->window, NULL);
    eglMakeCurrent(egl_display, egl_surface, egl_surface, egl_context);
    // Recreate framebuffers that depend on surface size:
    gl_recreate_framebuffers(ANativeWindow_getWidth(app->window),
                             ANativeWindow_getHeight(app->window));
    break;

case APP_CMD_TERM_WINDOW:
    eglMakeCurrent(egl_display, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
    eglDestroySurface(egl_display, egl_surface);
    egl_surface = EGL_NO_SURFACE;
    // Do NOT destroy context here unless you know it is truly gone
    break;
```

### When to Prefer Vulkan over EGL

EGL's thread affinity model is a consistent source of subtle bugs in job-system-based engines. A common failure mode: a job-system worker uploads a texture (EGL context not current on worker thread → GL error, silent failure). Vulkan avoids this entirely — command buffers record from any thread, submission happens from one thread, and there is no per-thread current-context state. For a new Android title in 2026, Vulkan is the recommended rendering API; EGL/GLES should be retained only for compatibility with API < 24 devices or for integration with legacy middleware that requires an `EGLContext`. [Source: EGL 1.5 specification](https://registry.khronos.org/EGL/specs/eglspec.1.5.pdf)

---

## 12. Oboe: Low-Latency Audio

Audio latency is one of Android's most criticised limitations. A game requiring tight audio/visual synchronisation — footstep sounds on a frame boundary, fighting-game hit feedback — needs round-trip latency (input → process → output) under 20 ms. Android's original audio stack (OpenSL ES) had latency of 50–200 ms on most devices. AAudio (Android 8.0, API 26) introduced a low-latency MMAP path, but OpenSL ES remains necessary for API < 26 devices. **Oboe** (`com.google.oboe:oboe`) abstracts both, automatically selecting AAudio when available and falling back to OpenSL ES.

[Source: Oboe GitHub](https://github.com/google/oboe)

### Audio stream with callback

The preferred pattern is a **callback-driven** stream: Oboe calls `onAudioReady()` from a high-priority audio thread at each buffer period. The callback must not block, allocate, or call JNI.

```c
#include <oboe/Oboe.h>

class AudioEngine : public oboe::AudioStreamCallback {
    std::shared_ptr<oboe::AudioStream> stream_;
    float phase_ = 0.0f;

public:
    oboe::Result open() {
        oboe::AudioStreamBuilder builder;
        builder.setDirection(oboe::Direction::Output)
               .setPerformanceMode(oboe::PerformanceMode::LowLatency)
               .setSharingMode(oboe::SharingMode::Exclusive)
               .setFormat(oboe::AudioFormat::Float)
               .setChannelCount(oboe::ChannelCount::Stereo)
               .setSampleRate(48000)
               .setCallback(this);
        return builder.openStream(stream_);
    }

    oboe::DataCallbackResult onAudioReady(
            oboe::AudioStream* stream,
            void* audioData, int32_t numFrames) override {
        auto* out = static_cast<float*>(audioData);
        for (int i = 0; i < numFrames; i++) {
            float sample = sinf(phase_) * 0.3f;
            phase_ += 2.0f * M_PI * 440.0f / stream->getSampleRate();
            if (phase_ > 2.0f * M_PI) phase_ -= 2.0f * M_PI;
            out[i * 2 + 0] = sample;  // left
            out[i * 2 + 1] = sample;  // right
        }
        return oboe::DataCallbackResult::Continue;
    }

    void start() { stream_->requestStart(); }
    void stop()  { stream_->requestStop(); }
    void close() { stream_->close(); }
};
```

### Performance mode and sharing mode

| Setting | Options | Effect |
|---|---|---|
| `PerformanceMode` | `LowLatency`, `PowerSaving`, `None` | `LowLatency` requests MMAP/exclusive path; `PowerSaving` uses larger buffers |
| `SharingMode` | `Exclusive`, `Shared` | `Exclusive` gets the MMAP path when available (lowest latency); `Shared` shares the mixer |
| `AudioFormat` | `Float`, `I16`, `I32` | `Float` is preferred for mixing; driver may accept directly without conversion |

`PerformanceMode::LowLatency` + `SharingMode::Exclusive` activates AAudio's **MMAP path** on supported devices — the audio data is written directly to a shared memory region mapped between the app and the audio HAL, bypassing the AudioFlinger mixer. Round-trip latency on Pixel devices: ~5–10 ms.

### Input (microphone) stream

```c
oboe::AudioStreamBuilder builder;
builder.setDirection(oboe::Direction::Input)
       .setPerformanceMode(oboe::PerformanceMode::LowLatency)
       .setSharingMode(oboe::SharingMode::Exclusive)
       .setFormat(oboe::AudioFormat::Float)
       .setChannelCount(1)  // mono microphone
       .setCallback(this);
builder.openStream(stream_);
stream_->requestStart();
```

### Stream restart on disconnect

The `onErrorBeforeClose` / `onErrorAfterClose` callbacks handle headphone disconnect and other stream invalidation events. Oboe provides `AudioStreamErrorCallback` with a default restart implementation:

```c
class AudioEngine : public oboe::AudioStreamCallback,
                    public oboe::AudioStreamErrorCallback {
    void onErrorAfterClose(oboe::AudioStream*, oboe::Result) override {
        // Restart the stream on a non-audio thread
        open(); start();
    }
};
```

---

## 13. Android Performance Tuner

The **Android Performance Tuner** (APT, `androidx.games:games-performance`) reports per-frame timing data to the **Google Play backend**, enabling Google Play Console to show frame pacing histograms and recommend quality level adjustments to live apps. For the developer, it is a lightweight frame-tick instrumentation; for end users, Play can automatically suggest lower graphics settings on struggling devices.

### Architecture

APT uses **protocol buffers** to define an app's quality "fidelity parameters" (resolution, shadow quality, texture level, etc.) and "annotation" (which level / scene type is active). It sends timing data per frame and receives back "recommended fidelity parameters" from Play's device-specific performance database.

```c
#include <tuningfork/tuningfork.h>
#include <tuningfork/protobuf_nano_util.h>

// Initialise (once, early in android_main):
TuningFork_Settings settings = {};
settings.swappy_tracer_fn = &SwappyVk_injectTracer;  // integrate with Swappy
settings.swappy_version = Swappy_version();
TuningFork_init(&settings, env, app->activity->javaGameActivity);

// Per frame — tick the default PACED instrument:
TuningFork_frameTick(TFTICK_PACED_FRAME_TIME);

// Custom annotation (e.g., current level):
MyAnnotation ann = {.level = current_level, .scene = SCENE_OUTDOOR};
TuningFork_setCurrentAnnotation(&ann, sizeof(ann));

// Flush (e.g., on pause):
TuningFork_flush();
TuningFork_destroy();
```

### Instrument IDs

APT tracks separate timing histograms per instrument tag. Built-in tags:

| Tag | Measures |
|---|---|
| `TFTICK_PACED_FRAME_TIME` | Wall time per frame (includes Swappy wait) |
| `TFTICK_CPU_TIME` | CPU-only frame time |
| `TFTICK_GPU_TIME` | GPU frame time (via `VK_GOOGLE_display_timing`) |
| `TFTICK_SWAPPY_WAIT_TIME` | Time Swappy spent waiting for vsync |

Custom instruments allow per-subsystem timing (e.g., physics tick, shadow pass).

[Source: Android Performance Tuner](https://developer.android.com/games/sdk/performance-tuner)

---

## 14. ADPF In Depth: Performance Hints, Thermal Headroom, and Adaptive Quality

The **Android Dynamic Performance Framework** (ADPF) is a system-level API that lets a game communicate its thread workload expectations to the Linux kernel's CPU scheduler, and query the device's thermal state to proactively adapt rendering quality. Introduced in Android 12 (API 31) for performance hints and Android 11 (API 30) for thermal management, ADPF is one of the most impactful optimisation APIs available to mobile games — directly influencing CPU clock frequency selection through the kernel's Energy Aware Scheduler (EAS).

### Performance Hint Sessions

The core ADPF concept is the **hint session**: a group of threads registered with the system, plus a *target work duration* (how long the work should take, in nanoseconds). After each work unit, the app reports the *actual work duration*. The system uses this information to choose CPU frequency — boosting when actual duration exceeds target (the work is taking too long), and scaling back when actual duration is well below target (the work is completing early, wasting power).

All ADPF hint APIs are in `<android/performance_hint.h>`, available in NDK API 31+.

```cpp
#include <android/performance_hint.h>

// Step 1: retrieve the system hint manager (singleton, one per process):
APerformanceHintManager* manager = APerformanceHint_getManager();
```

**Creating a session**: Pass the array of thread IDs (TIDs, not PIDs) that belong to the session, and the initial target work duration:

```cpp
// Game thread TID (use gettid() from <sys/types.h>):
pid_t game_tid  = gettid();
pid_t render_tid = render_thread.native_handle(); // std::thread → OS TID via platform API

pid_t tids[] = {game_tid, render_tid};
int64_t target_duration_ns = 16'666'666LL;  // 60 fps = 16.67 ms

APerformanceHintSession* session = APerformanceHint_createSession(
    manager,
    tids,
    2,                    // num_threads
    target_duration_ns);
```

Separate sessions can be created for logically independent workloads: a game thread session and a render thread session allow the scheduler to boost each independently. The physics thread, if it has a fixed timestep budget, benefits from its own session.

**Reporting actual work duration**: Call once per frame, after the frame's work on the session's threads is complete. Measure with `std::chrono::steady_clock` (which maps to `CLOCK_MONOTONIC`):

```cpp
auto frame_start = std::chrono::steady_clock::now();

// ... game loop body: input, physics, animation, render command recording ...

auto frame_end = std::chrono::steady_clock::now();
int64_t actual_ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
    frame_end - frame_start).count();

APerformanceHint_reportActualWorkDuration(session, actual_ns);
```

The kernel's EAS processes these reports and adjusts the `schedutil` CPUfreq governor's frequency request for the affected CPU cluster. On Snapdragon 8 Gen 2/3 hardware, correct ADPF usage has been shown to reduce frame time variance by 20–40% by preventing the CPU from under-clocking during sustained load. [Source: ADPF developer guide](https://developer.android.com/games/optimize/adpf)

**Updating target duration**: When the game changes its target frame rate (e.g., switching from 60 fps to 30 fps during thermal throttling), update the session immediately:

```cpp
int64_t new_target_ns = 33'333'333LL;  // 30 fps
APerformanceHint_updateTargetWorkDuration(session, new_target_ns);
```

### Thermal Management (API 30+)

ADPF's thermal component allows the game to query device temperature status and predict imminent throttling. The `AThermal*` APIs are in `<android/thermal.h>`:

```cpp
#include <android/thermal.h>

// Acquire the thermal manager (must be released when done):
AThermalManager* thermal_mgr = AThermal_acquireManager();

// Query current thermal status:
AThermalStatus status = AThermal_getCurrentThermalStatus(thermal_mgr);
```

The `AThermalStatus` enum spans from benign to emergency:

| Status | Value | Meaning |
|---|---|---|
| `ATHERMAL_STATUS_NONE` | 0 | No thermal constraint |
| `ATHERMAL_STATUS_LIGHT` | 1 | Light throttling |
| `ATHERMAL_STATUS_MODERATE` | 2 | Moderate throttling |
| `ATHERMAL_STATUS_SEVERE` | 3 | Performance significantly impaired |
| `ATHERMAL_STATUS_CRITICAL` | 4 | Aggressive throttling; game may be unplayable |
| `ATHERMAL_STATUS_EMERGENCY` | 5 | App may be terminated to protect hardware |
| `ATHERMAL_STATUS_SHUTDOWN` | 6 | Device shutting down |

Register a callback to receive status changes asynchronously (fires on a system thread):

```cpp
AThermal_registerThermalStatusListener(thermal_mgr,
    [](void* data, AThermalStatus status) {
        static_cast<GameEngine*>(data)->on_thermal_status_changed(status);
    }, game_engine);
```

### Thermal Headroom (API 35+)

Android 15 (API 35) added **thermal headroom forecasting**: a normalised 0.0–1.0 float indicating how close the device is to a thermal throttle event in the next N seconds:

```cpp
// Query headroom forecast for the next 10 seconds:
float headroom = AThermal_getThermalHeadroom(thermal_mgr, 10 /*forecast_seconds*/);
// 0.0 = cool; 1.0 = throttle threshold; >1.0 = already throttling
```

The forecast integrates thermal inertia: a device with `headroom = 0.95` at 10-second forecast should prompt quality reductions now, before the throttle actually fires. [Source: AThermal_getThermalHeadroom](https://developer.android.com/ndk/reference/group/thermal#athermal_getthermalheadroom)

### Adaptive Quality Decision Tree

Combining thermal status and headroom, a practical adaptive quality system applies reductions in priority order from least to most impactful:

```cpp
void GameEngine::adapt_quality(AThermalStatus status, float headroom) {
    if (status == ATHERMAL_STATUS_NONE && headroom < 0.70f) {
        return;  // all quality levels at max
    }

    // Level 1: reduce shadow map resolution (least impactful)
    if (headroom > 0.70f || status >= ATHERMAL_STATUS_LIGHT) {
        shadow_map_resolution = 1024;  // was 2048
    }

    // Level 2: reduce draw distance
    if (headroom > 0.80f || status >= ATHERMAL_STATUS_MODERATE) {
        draw_distance_multiplier = 0.7f;
    }

    // Level 3: reduce particle count and post-processing quality
    if (headroom > 0.90f || status >= ATHERMAL_STATUS_SEVERE) {
        max_particle_count = 500;  // was 2000
        bloom_enabled = false;
    }

    // Level 4: reduce render resolution (most impactful — recreate swapchain)
    if (headroom > 0.95f || status >= ATHERMAL_STATUS_CRITICAL) {
        float scale = 0.75f;  // render at 75% of display resolution
        uint32_t new_width  = (uint32_t)(display_width  * scale);
        uint32_t new_height = (uint32_t)(display_height * scale);
        vulkan_recreate_swapchain(new_width, new_height);
    }

    // Level 5: reduce target frame rate via Swappy (last resort)
    if (status >= ATHERMAL_STATUS_CRITICAL) {
        // Drop to 30 fps:
        SwappyVk_setSwapIntervalNS(device, swapchain, 33'333'333LL);
        APerformanceHint_updateTargetWorkDuration(session, 33'333'333LL);
    }
}
```

Swapchain recreation (changing `VkSwapchainCreateInfoKHR.imageExtent`) is the most disruptive quality reduction because it causes a brief GPU pipeline flush. Apply it only when other measures have failed to prevent throttling.

### Kotlin Integration via PowerManager

For game code that uses a Kotlin `Activity` layer (e.g., for the main menu or social features), the thermal API is also available through `PowerManager`:

```kotlin
// API 29+ thermal status via PowerManager:
import android.os.PowerManager

val pm = getSystemService(PowerManager::class.java)
val thermalStatus = pm.currentThermalStatus
// PowerManager.THERMAL_STATUS_NONE / LIGHT / MODERATE / SEVERE / CRITICAL / EMERGENCY / SHUTDOWN

// Register a listener (API 29+):
pm.addThermalStatusListener(executor) { status ->
    // Forward to native layer via JNI
    nativeSetThermalStatus(status)
}
```

### ADPF + Swappy Integration

Swappy (§8) and ADPF are complementary: Swappy manages the *display side* of frame pacing (presentation scheduling, vsync alignment) while ADPF manages the *CPU side* (frequency scaling based on work duration). When both are active:

1. Swappy measures per-frame GPU presentation timing and adjusts the swap interval.
2. ADPF hint sessions measure CPU frame time and adjust CPU frequency via EAS.
3. Thermal management reduces quality targets, which feeds back into both: Swappy changes its swap interval, ADPF sessions get updated target durations.

A forthcoming Swappy API (`SwappyVk_setAPerformanceHintSession()`, planned but not yet released as of mid-2026 — see the Roadmap section) would allow Swappy to automatically update ADPF hint session targets whenever it changes the swap interval, completing the feedback loop without application-layer wiring.

The reference sample demonstrating the full integration is at [android/games-samples/agdk/adpf](https://github.com/android/games-samples/tree/main/agdk/adpf).

Cleanup:
```cpp
APerformanceHint_closeSession(session);
AThermal_releaseManager(thermal_mgr);
```

[Source: ADPF developer guide](https://developer.android.com/games/optimize/adpf) | [AThermal API reference](https://developer.android.com/ndk/reference/group/thermal)

---

## 15. Native Crash Reporting with Firebase Crashlytics and NDK Symbols

A game targeting 5,000 distinct Android device models — spanning 6 years of Snapdragon/Dimensity/Exynos SoCs, 4 Vulkan driver generations, and 3 major OS versions — will encounter native crashes that occur only on specific hardware and driver combinations. Without native crash symbolication, a raw SIGSEGV in `.so` code appears as an unintelligible signal/address pair in the Play Console. Firebase Crashlytics with NDK symbol upload provides automatic server-side symbolication, transforming raw crashes into function name / file / line reports.

### Setup: Gradle Plugin and Dependencies

Crashlytics NDK support requires the Firebase Crashlytics Gradle plugin (which handles symbol upload) and the NDK-specific runtime dependency:

```kotlin
// project-level build.gradle.kts
plugins {
    id("com.google.gms.google-services") version "4.4.2" apply false
    id("com.google.firebase.crashlytics") version "3.0.2" apply false
}
```

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.0.0"))
    implementation("com.google.firebase:firebase-crashlytics")
    implementation("com.google.firebase:firebase-crashlytics-ndk")
}
```

```properties
# gradle.properties
com.google.firebase.crashlytics.nativeSymbolUploadEnabled=true
com.google.firebase.crashlytics.unstrippedNativeLibsDir=app/build/intermediates/merged_native_libs
```

The `firebase-crashlytics-ndk` plugin hooks into the `assembleRelease` task: it locates unstripped `.so` files in the build tree, generates Breakpad `.sym` symbol files from them via `dump_syms`, uploads the symbols to the Firebase backend, and strips the `.so` files before packaging them into the APK. Crashes are then symbolicated server-side. [Source: Firebase Crashlytics NDK documentation](https://firebase.google.com/docs/crashlytics/ndk-reports)

### How Native Crash Capture Works

`firebase-crashlytics-ndk` installs a **signal handler** for `SIGSEGV`, `SIGABRT`, `SIGILL`, `SIGBUS`, `SIGFPE`, and `SIGTRAP`. When a crash occurs:

1. The signal handler captures the faulting address, register state, and a stack unwind (using LLVM libunwind or the platform unwinder).
2. The crash report is serialised to a file in the app's internal storage (the signal handler cannot use the heap).
3. On the *next* app launch, the Crashlytics SDK reads the crash file, symbolically decorates the stack frames using any local information available, and uploads the report to Firebase.
4. Firebase backend applies the uploaded `.sym` files to produce fully symbolicated stack traces.

### Custom Events and Keys from C++

The Firebase C++ SDK (`firebase/crashlytics.h`) allows native code to annotate crash reports:

```cpp
#include "firebase/app.h"
#include "firebase/crashlytics.h"

firebase::App* fb_app = firebase::App::Create(firebase::AppOptions(), jni_env, activity);
firebase::crashlytics::Crashlytics* crashlytics =
    firebase::crashlytics::Crashlytics::GetInstance(fb_app, &init_result);

// Custom log messages (appear in the "Logs" section of a crash report):
crashlytics->Log("world_3_loaded");
crashlytics->Log("starting_boss_encounter");

// Custom key-value pairs (appear as searchable metadata on the crash):
crashlytics->SetCustomKey("gpu_vendor",      gpu_vendor_string.c_str());
crashlytics->SetCustomKey("driver_version",  driver_version_string.c_str());
crashlytics->SetCustomKey("thermal_status",  std::to_string(current_thermal_status).c_str());
crashlytics->SetCustomKey("frame_count",     std::to_string(frame_count).c_str());
crashlytics->SetCustomKey("gpu_memory_mb",   std::to_string(gpu_memory_budget_mb).c_str());
```

The `gpu_vendor` and `driver_version` keys are particularly valuable: filtering crashes by `driver_version = "OpenGL ES 3.2 V@0615.155"` immediately isolates issues specific to a single Qualcomm driver release. Setting `thermal_status` as a key allows correlating thermal-induced overwrite bugs (stack corruption under memory pressure combined with reduced CPU clocks) with specific thermal severity levels.

### Effective Custom Keys for GPU Triage

Best practice: set these keys at game startup and update them as state changes:

| Key | Source | Value at crash |
|---|---|---|
| `gpu_vendor` | `glGetString(GL_VENDOR)` / `VkPhysicalDeviceProperties::vendorID` | "Qualcomm" / "ARM" |
| `driver_version` | `glGetString(GL_VERSION)` / `VkPhysicalDeviceProperties::driverVersion` | "OpenGL ES 3.2 V@0615.155" |
| `vulkan_api_version` | `VkPhysicalDeviceProperties::apiVersion` | "1.3.255" |
| `thermal_status` | `AThermal_getCurrentThermalStatus()` | "2" (MODERATE) |
| `frame_count` | Rolling frame counter | "1247" |
| `active_scene` | Game-specific scene identifier | "world_3_boss" |
| `gpu_memory_budget_mb` | `VK_EXT_memory_budget` heap budget | "2048" |

### Firebase Performance Monitoring vs. Android Performance Tuner

These are complementary tools targeting different data pipelines:

- **Firebase Performance Monitoring** (`firebase/performance.h`): records custom `Trace` objects (cold start duration, level load time, network round-trips) that appear in the Firebase console. Suitable for business-level performance KPIs visible to the developer team.

- **Android Performance Tuner** (§13): reports per-frame timing histograms, quality level annotations, and fidelity parameter data to the **Google Play Console**. Play can surface this data to device-specific performance recommendations and the "Android vitals" dashboard.

Both can coexist in the same app; they write to different backends.

### Monitoring Crash-Free Session Rate

The primary health KPI in the Firebase Crashlytics dashboard is the **crash-free session rate** — the percentage of app sessions that complete without a native or Java crash. Targets:

- >99.5% overall: acceptable for production
- Filter by ABI: `arm64-v8a` and `armeabi-v7a` crash rates often diverge; `armeabi-v7a` devices tend to have older drivers
- Filter by OS version: Android 12 vs. Android 15 may expose different Vulkan validation layer behaviour
- Filter by `gpu_vendor` custom key: Adreno vs. Mali crash rates for the same code can differ 10×

Set up **Crashlytics alerts** (Firebase console → Crashlytics → Alerts) to notify on any crash issue affecting >0.1% of sessions, and on any new issue with >10 occurrences. This provides faster signal than waiting for App Review ratings to decline.

[Source: Firebase Crashlytics NDK documentation](https://firebase.google.com/docs/crashlytics/ndk-reports) | [Firebase C++ SDK](https://firebase.google.com/docs/reference/cpp/crashlytics)

---

## 16. Memory Advice API

Android's `onTrimMemory(TRIM_MEMORY_RUNNING_CRITICAL)` callback fires when the OS is about to kill the app's background processes due to memory pressure — but it fires reactively, too late for the app to gracefully reduce its memory footprint. The **Memory Advice API** (`androidx.games:games-memory-advice`) provides **predictive** memory pressure warnings that arrive before the critical threshold.

### Architecture

The library samples Android's `/proc/meminfo`, `ActivityManager.MemoryInfo`, and `Debug.MemoryInfo` on a background thread, builds a simple statistical model of memory pressure trajectory, and issues callbacks when it predicts the threshold will be crossed.

```c
#include <memory_advice/memory_advice.h>

// Init once:
MemoryAdvice_init(env, app->activity->javaGameActivity);

// Register a callback (fires on the sampling thread):
MemoryAdvice_registerWatcher(500 /*poll interval ms*/,
    [](MemoryAdvice_MemoryState state, void* ctx) {
        switch (state) {
        case MEMORYADVICE_STATE_OK:
            break;
        case MEMORYADVICE_STATE_APPROACHING_LIMIT:
            reduce_texture_cache_size();    // proactive quality reduction
            break;
        case MEMORYADVICE_STATE_CRITICAL:
            drop_non_essential_resources(); // emergency release
            break;
        }
    }, myApp);

// Or poll manually per frame:
MemoryAdvice_MemoryState state = MemoryAdvice_getMemoryState();

// Cleanup:
MemoryAdvice_destroy();
```

### Percentage available estimate

```c
float pct = MemoryAdvice_getPercentageAvailableCapacity();
// 0.0 = critically low, 1.0 = plenty available
if (pct < 0.15f) drop_streaming_cache();
```

[Source: Memory Advice API](https://developer.android.com/games/sdk/memory-advice)

---

## 17. Android GPU Inspector

**Android GPU Inspector** (AGI, [github.com/google/agi](https://github.com/google/agi)) is Google's open-source GPU profiler, succeeding the deprecated Android GPU Profiler. It captures Vulkan API call traces and GPU performance counter data, integrates with **perfetto** for system-level tracing, and provides a desktop GUI for analysis.

### Two capture modes

**Frame capture**: Records all Vulkan API calls in one or more frames (analogous to RenderDoc on desktop). The capture is replayed on the device to measure per-draw-call GPU timings and validate API usage.

**System profiling**: Uses the **perfetto** tracing framework (Android 10+) to record GPU counter tracks (vertex throughput, texture cache hit rate, bandwidth, occupancy) alongside CPU scheduling, thread wakeup latency, and Choreographer vsync ticks. Output is a `.perfetto-trace` file viewable in the Perfetto UI.

### Supported GPU families

| Vendor | GPU | Counter support |
|---|---|---|
| Qualcomm | Adreno 5xx / 6xx / 7xx / 8xx | Full GPU counter support |
| ARM | Mali Bifrost / Valhall / Immortalis | Full GPU counter support |
| Imagination | PowerVR Series 9 | Partial |
| Google | Swiftshader (software) | Limited |

### Integration with Vulkan validation

AGI embeds the Khronos Vulkan Validation Layers and can highlight API misuse (missing synchronisation, suboptimal layout transitions, redundant barriers) alongside the performance data.

### Command-line capture for CI

```bash
# Capture 3 frames starting at frame 100:
agi capture --apk com.example.game --frames 100:103 --output capture.agi

# System profile for 5 seconds:
agi capture --apk com.example.game --duration 5000 --type perfetto \
    --output profile.perfetto-trace
```

### Perfetto GPU counter integration in app code

For apps that want to annotate the perfetto trace with their own events:

```c
#include <perfetto/tracing.h>

PERFETTO_DEFINE_CATEGORIES(
    perfetto::Category("game").SetDescription("Game events"));

// In render loop:
TRACE_EVENT("game", "RenderFrame");
TRACE_EVENT("game", "ShadowPass");
vkCmdDrawIndexed(cmd, indexCount, 1, 0, 0, 0);
```

[Source: Android GPU Inspector](https://gpuinspector.dev/), [perfetto GPU tracing](https://perfetto.dev/docs/data-sources/gpu)

---

## 18. ARCore Integration: Building AR-Native Games

ARCore is Google's augmented reality platform for Android. It is a separate SDK from AGDK but integrates directly with GameActivity and the NDK-direct game loop model. The combination — AGDK for lifecycle and surface management, ARCore C API for tracking and camera — is the recommended pattern for native AR games and XR apps.

### ARCore availability and CMake integration

ARCore requires Play Services AR (`com.google.ar.core`) to be installed on the device, and verification must happen at startup:

```c
// Link: add -larcore to CMakeLists.txt
// Gradle: implementation 'com.google.ar:core:1.46.0'
// CMake: find_package(arcore REQUIRED CONFIG) or link directly

#include "arcore_c_api.h"

// Check availability before ArSession_create
ArAvailability avail;
ArCoreApk_checkAvailability(env, activity, &avail);
if (avail != AR_AVAILABILITY_SUPPORTED_INSTALLED) {
    // Request install or show unsupported message
    ArCoreApk_requestInstall(env, activity, true, &install_status);
}
```

[Source: ARCore C API quickstart](https://developers.google.com/ar/develop/c/quickstart)

### ArSession lifecycle inside android_main

The `ArSession` maps cleanly onto GameActivity's `APP_CMD_*` lifecycle. The critical mapping is:

| `APP_CMD_*` event | `ANativeWindow` state | ARCore action |
|---|---|---|
| `APP_CMD_RESUME` | may be NULL | `ArSession_resume(session)` |
| `APP_CMD_INIT_WINDOW` | non-NULL | Set display geometry; create Vulkan swapchain |
| `APP_CMD_GAINED_FOCUS` | non-NULL | Begin AR frame loop |
| `APP_CMD_LOST_FOCUS` | non-NULL | Pause AR frame loop |
| `APP_CMD_PAUSE` | non-NULL | `ArSession_pause(session)` |
| `APP_CMD_TERM_WINDOW` | about to become NULL | Destroy swapchain; do not destroy `ArSession` |
| `APP_CMD_DESTROY` | NULL | `ArSession_destroy(session)` |

```c
// File: ar_game_main.cpp — combined AGDK + ARCore game loop
#include <game-activity/native_app_glue/android_native_app_glue.h>
#include "arcore_c_api.h"

static ArSession* ar_session = nullptr;
static ArFrame*   ar_frame   = nullptr;

static void handle_cmd(struct android_app* app, int32_t cmd) {
    switch (cmd) {
    case APP_CMD_CREATE:
        ArSession_create(app->activity->env, app->activity->clazz, &ar_session);
        ArFrame_create(ar_session, &ar_frame);
        break;
    case APP_CMD_RESUME:
        ArSession_resume(ar_session);
        break;
    case APP_CMD_INIT_WINDOW:
        // Set the display geometry after the surface is ready
        ArSession_setDisplayGeometry(ar_session,
            app->activity->env,          // JNIEnv*
            0,                           // ROTATION_0 (update on config change)
            ANativeWindow_getWidth(app->window),
            ANativeWindow_getHeight(app->window));
        vulkan_create_swapchain(app->window);
        break;
    case APP_CMD_CONFIG_CHANGED: {
        // Update geometry when orientation changes
        int32_t rot = AConfiguration_getOrientation(app->config);
        ArSession_setDisplayGeometry(ar_session, app->activity->env,
            rot, ANativeWindow_getWidth(app->window),
            ANativeWindow_getHeight(app->window));
        break;
    }
    case APP_CMD_PAUSE:
        ArSession_pause(ar_session);
        break;
    case APP_CMD_TERM_WINDOW:
        vulkan_destroy_swapchain();
        break;
    case APP_CMD_DESTROY:
        ArFrame_destroy(ar_frame);
        ArSession_destroy(ar_session);
        break;
    }
}

static void do_frame(struct android_app* app) {
    // 1. Update AR session — fills ar_frame with new camera pose
    ArStatus status = ArSession_update(ar_session, ar_frame);
    if (status != AR_SUCCESS) return;

    // 2. Get camera pose and projection
    ArCamera* camera;
    ArFrame_acquireCamera(ar_session, ar_frame, &camera);

    float view_mat[16], proj_mat[16];
    ArCamera_getViewMatrix(ar_session, camera, view_mat);
    ArCamera_getProjectionMatrix(ar_session, camera, 0.1f, 100.0f, proj_mat);

    ArCamera_release(camera);

    // 3. Render AR content with view_mat/proj_mat via Vulkan (see §18.2)
    vulkan_render_frame(view_mat, proj_mat);
}

void android_main(struct android_app* app) {
    app->onAppCmd = handle_cmd;
    while (!app->destroyRequested) {
        GameActivityInputBuffer* ib = android_app_swap_input_buffers(app);
        if (ib) android_app_clear_motion_events(ib);
        int events;
        android_poll_source* source;
        ALooper_pollOnce(0, nullptr, &events, (void**)&source);
        if (source) source->process(app, source);
        if (app->window) do_frame(app);
    }
}
```

### The two camera texture paths

ARCore produces the live camera image on every `ArSession_update()`. There are two paths for consuming it in rendering:

#### OpenGL ES path: ArSession_setCameraTextureName

The traditional path registers a `GL_TEXTURE_EXTERNAL_OES` texture that ARCore writes the camera YUV image into:

```c
// Call once after ArSession_create, before ArSession_resume
GLuint camera_tex_id;
glGenTextures(1, &camera_tex_id);
ArSession_setCameraTextureName(ar_session, camera_tex_id);

// After ArSession_update(), camera_tex_id contains the latest frame
// Fragment shader must use samplerExternalOES:
//   #extension GL_OES_EGL_image_external : require
//   uniform samplerExternalOES u_CameraTexture;
//   void main() { fragColor = texture(u_CameraTexture, texCoord); }
```

For apps with a multi-threaded rendering pipeline, `ArSession_setCameraTextureNames()` accepts a ring buffer of texture IDs — ARCore assigns each incoming frame to the next texture in the array, allowing the render thread to consume the previous frame while ARCore fills the next.

#### Vulkan path: AHardwareBuffer export

ARCore's Vulkan path (API 27+, requires `VK_ANDROID_external_memory_android_hardware_buffer`) exposes the camera frame as an `AHardwareBuffer`:

```c
// Configure session to expose hardware buffers:
ArConfig* config;
ArConfig_create(ar_session, &config);
ArConfig_setTextureUpdateMode(ar_session, config,
    AR_TEXTURE_UPDATE_MODE_EXPOSE_HARDWARE_BUFFER);
ArSession_configure(ar_session, config);
ArConfig_destroy(config);

// After ArSession_update(), retrieve the buffer for the current frame:
AHardwareBuffer* hw_buf = nullptr;
ArFrame_getHardwareBuffer(ar_session, ar_frame, &hw_buf);
// hw_buf is valid until the next ArSession_update() call.
// If you need to retain it across frames:
AHardwareBuffer_acquire(hw_buf);  // keep alive
// ... render the frame ...
AHardwareBuffer_release(hw_buf);  // release when done

// Import into Vulkan via VK_ANDROID_external_memory_android_hardware_buffer:
// (same pattern as AHardwareBuffer × Vulkan in Ch86 §6)
VkAndroidHardwareBufferPropertiesANDROID hw_props = { ... };
vkGetAndroidHardwareBufferPropertiesANDROID(device, hw_buf, &hw_props);
// Create VkImage backed by the AHardwareBuffer; add VkSamplerYcbcrConversion
// for the camera YUV format (VkExternalFormatANDROID).
```

The camera YUV data uses an implementation-defined external format on most devices. The Vulkan import path requires `VkSamplerYcbcrConversion` to convert Y'CBCR → RGB in the fragment shader. The ARCore SDK's `hello_ar_vulkan_c` sample shows the complete pipeline: session configuration → `ArFrame_getHardwareBuffer()` → `VkExternalFormatANDROID` → `VkSamplerYcbcrConversionCreateInfo` → per-frame image barrier → sampler usage in GLSL. [Source: hello_ar_vulkan_c](https://github.com/google-ar/arcore-android-sdk/tree/main/samples/hello_ar_vulkan_c)

### Plane detection, hit-testing, and anchors

ARCore provides world-understanding on top of the camera frame:

```c
// Plane detection — enumerate all detected planes
ArTrackableList* plane_list;
ArTrackableList_create(ar_session, &plane_list);
ArSession_getAllTrackables(ar_session, AR_TRACKABLE_PLANE, plane_list);

int32_t plane_count;
ArTrackableList_getSize(ar_session, plane_list, &plane_count);
for (int32_t i = 0; i < plane_count; i++) {
    ArTrackable* trackable;
    ArTrackableList_acquireItem(ar_session, plane_list, i, &trackable);
    ArPlane* plane = ArAsPlane(trackable);
    ArTrackingState state;
    ArTrackable_getTrackingState(ar_session, trackable, &state);
    if (state == AR_TRACKING_STATE_TRACKING) {
        // ArPlane_getPolygon() gives the boundary vertices
        // ArPlane_getCenterPose() gives the plane pose for rendering a quad
    }
    ArTrackable_release(trackable);
}
ArTrackableList_destroy(plane_list);

// Hit-test: tap screen → find surface intersection
ArHitResultList* hit_list;
ArHitResultList_create(ar_session, &hit_list);
ArFrame_hitTest(ar_session, ar_frame, tap_x, tap_y, hit_list);
int32_t hit_count;
ArHitResultList_getSize(ar_session, hit_list, &hit_count);
if (hit_count > 0) {
    ArHitResult* hit;
    ArHitResult_create(ar_session, &hit);
    ArHitResultList_getItem(ar_session, hit_list, 0, hit);
    ArAnchor* anchor;
    ArHitResult_acquireNewAnchor(ar_session, hit, &anchor);
    // Store anchor; retrieve pose each frame for object placement
    ArHitResult_destroy(hit);
}
ArHitResultList_destroy(hit_list);
```

### Game Mode API (Android 12+)

Android 12 introduced the **Game Mode API** — orthogonal to both AGDK and ARCore but relevant to any native game. It allows the system to apply preset optimisation profiles based on user preference:

- **`GAME_MODE_PERFORMANCE`**: Maximise FPS; system boosts CPU/GPU clocks. Typical result: +10–30% GPU throughput.
- **`GAME_MODE_BATTERY`**: Reduce power draw; may throttle clocks or lower default render resolution.
- **`GAME_MODE_STANDARD`**: Balanced default.

Android 13 extended the API with `setGameState()` — the app can signal `GAME_STATE_GAMEPLAY_INTERACTING`, `GAME_STATE_GAMEPLAY_UNINTERACTING`, or `GAME_STATE_LOADING`, and the system can apply per-state power policies (e.g., boost CPU during asset loading, relax during idle menus).

```java
// In Kotlin/Java layer (GameActivity subclass):
import android.app.GameManager
val gameManager = getSystemService(GameManager::class.java)
val mode = gameManager.gameMode
// mode: GAME_MODE_UNSUPPORTED / STANDARD / PERFORMANCE / BATTERY / CUSTOM

// Android 13+ game state:
import android.app.GameState
gameManager.setGameState(GameState(true, GameState.MODE_GAMEPLAY_INTERACTING))
```

The Game Mode API is a system-level hint; the actual effect is device-dependent and OEM-specific. [Source: Game Mode API](https://developer.android.com/games/optimize/adpf/gamemode/gamemode-api)

### Android XR and ARCore for Jetpack XR

Android XR (Samsung Galaxy XR / Project Moohan, 2025) introduces a new **ARCore for Jetpack XR** API (`com.google.ar:arcore-jetpack-xr`). This is Kotlin-first (not C), but the underlying session model is the same:

- `XrSession` (Jetpack XR) maps to `ArSession` (mobile ARCore): tracking, plane detection, anchor persistence.
- Hand tracking (`XrHandTrackingState`) is available as a first-class API on XR headsets.
- The rendering surface is a `SurfaceView` managed by `GameActivity` or a Jetpack Compose `AndroidView`.
- Native Vulkan rendering on XR headsets follows the same `ANativeWindow → vkCreateAndroidSurfaceKHR` path, with `VK_KHR_multiview` strongly recommended for stereo rendering.

For native games targeting both mobile ARCore and Android XR headsets, the recommended pattern is:
1. GameActivity manages the `ANativeWindow` and app lifecycle (unchanged).
2. ARCore C API handles mobile tracking; Jetpack XR SDK handles headset tracking (called from Kotlin).
3. Vulkan render loop is common code in both cases: the same command buffers, same AGDK frame pacing (Swappy), same ANativeWindow integration.

[Source: ARCore for Android XR](https://developer.android.com/develop/xr/jetpack-xr-sdk/arcore), [hello_ar_c sample](https://github.com/google-ar/arcore-android-sdk/tree/main/samples/hello_ar_c)

---

## 19. Engine Integration: Unity, Unreal, Godot, and Bevy

Most Android games are not written directly against AGDK — they go through a game engine that internally consumes AGDK components. This section traces how each major engine maps onto the libraries described in this chapter.

### 19.1 Unity

Unity's Android backend is the most deeply integrated with AGDK, largely because Google has co-developed several features with the Unity team.

**Activity type**: Unity used `NativeActivity` from the beginning. Unity 2023.1 introduced `GameActivity` as an opt-in alternative; Unity 6 (2024) made `GameActivity` the default for new projects. The switch is controlled by **Edit → Project Settings → Player → Android → Activity Type**. GameActivity provides text-input handling via `GameTextInput` (§4), eliminating the overlay workaround Unity historically used for on-screen keyboards. [Source: Unity GameActivity docs](https://developer.android.com/games/engines/unity/gameactivity)

**Swappy (Android Frame Pacing)**: Integrated since Unity 2019.2. Enable via **Player Settings → Android → Resolution and Presentation → Optimized Frame Pacing**. The integration hooks into Unity's Vulkan and OpenGL ES swapchain; Swappy's `SwappyVk_queuePresent()` / `Swappy_swap()` replaces the bare `vkQueuePresentKHR`/`eglSwapBuffers` calls. In measured cases (e.g., *Mir 2*), this reduced slow-frame percentage from 40% to 10%. [Source: Unity Optimized Frame Pacing](https://docs.unity3d.com/ScriptReference/PlayerSettings.Android-optimizedFramePacing.html)

**Android Adaptive Performance (ADPF bridge)**: The `com.unity.adaptiveperformance.google.android` package (requires Unity 2021.3+, Adaptive Performance 4.0+) integrates ADPF's `PerformanceHintManager` and `ThermalManager` into Unity's [Adaptive Performance](https://docs.unity3d.com/Packages/com.unity.adaptiveperformance@4.0/manual/) API. From C# game code:

```csharp
using UnityEngine.AdaptivePerformance;

IAdaptivePerformance ap = Holder.Instance;
// Read thermal warning level (0=none, 1=moderate, 2=imminent shutdown)
ThermalMetrics t = ap.ThermalStatus.ThermalMetrics;
if (t.WarningLevel == WarningLevel.ThrottlingImminent) {
    QualitySettings.SetQualityLevel(1);   // drop to low preset
}
// ADPF performance hints are sent automatically by the provider
```

**Oboe (audio)**: Unity 2023 added first-party Oboe integration for low-latency audio output on Android. Earlier versions used OpenSL ES through Unity's platform audio layer. The integration is transparent to C# code — configure via `AudioSettings.outputSampleRate` and `AudioSettings.dspBufferSize`; the backend selects `AAudio` (Oboe's primary path) when the device supports it.

**Memory Advice API**: Available via `GoogleMemoryAdvice` JNI bridge documented at [developer.android.com/games/engines/unity/memory-advice](https://developer.android.com/games/engines/unity/memory-advice). The Java polling interface maps `APPROACHING_LIMIT` → reduce texture resident set; `CRITICAL` → force GC.

**Android GPU Inspector**: AGI is engine-agnostic. For Vulkan traces, start AGI and attach to the Unity app. For OpenGL ES, AGI uses a custom ANGLE build to translate GLES calls to Vulkan before capture — no Unity-specific configuration needed, but Vulkan rendering mode (`Graphics API = Vulkan` in Player Settings) is recommended for highest-fidelity traces.

**AGDK support matrix for Unity:**

| AGDK Component | Unity Integration | Since |
|---|---|---|
| GameActivity | Official (opt-in then default) | 2023.1 / Unity 6 |
| Swappy frame pacing | Official checkbox | 2019.2 |
| ADPF | Official (Adaptive Performance package) | 2021.3+ |
| Oboe audio | Official (transparent) | 2023.x |
| Memory Advice API | Official (JNI helper) | 2022.x |
| Paddleboat | Via Unity Input System (indirect) | — |

### 19.2 Unreal Engine

Unreal Engine has a longer history on Android and predates AGDK, resulting in a more selective integration.

**Activity type**: UE has its own `com.epicgames.ue4.GameActivity` Java class (the name has not changed even in UE5). This is **not** `androidx.games.GameActivity` — it extends `android.app.NativeActivity` and predates AGDK by several years. It provides JNI entry points for C++ interop. UE5 has not adopted AGDK's `GameActivity` due to the scope of migration required and the existing custom solution's maturity.

**Swappy**: Integrated by default since **UE 5.2**. Controlled via `bUseSwappyForFramePacing` in `AndroidRuntimeSettings` (also settable as `UseSwappyForFramePacing=0` in the Android device profile to opt out). Swappy is injected into UE's Vulkan present path; Android's `VK_GOOGLE_display_timing` (used by Swappy for frame timing) is queried via UE's Vulkan RHI extension enumeration.

**ADPF**: The [adpf-unreal-plugin](https://github.com/android/adpf-unreal-plugin) (maintained in the Android samples org) provides `UAndroidPerformancePlugin` — a UE plugin that creates ADPF `PerformanceHintSession` objects for the game thread and render thread and reports actual vs. target frame durations each tick. Thermal headroom is forwarded to UE's `IAndroidPlatformFile` scalability system, which can call `UGameUserSettings::SetQualityLevels()` to reduce shadow quality, draw distance, or resolution. [Source: ADPF Unreal plugin](https://developer.android.com/games/engines/unreal/unreal-adpf)

```cpp
// In UnrealEngine/Plugins/AndroidPerformance/Source/...:
// Plugin calls ANativeWindow pointer to detect display refresh rate,
// then creates a hint session for the game thread:
APerformanceHintManager* hint_manager = APerformanceHint_getManager();
APerformanceHintSession* session = APerformanceHint_createSession(
    hint_manager, &game_thread_tid, 1, target_nanos);
// Each frame:
APerformanceHint_reportActualWorkDuration(session, actual_nanos);
```

**Oboe**: No first-party Oboe integration. UE uses its own Android audio backend (based on OpenSL ES with an AAudio fallback). Community plugins exist on the Marketplace.

**AGDK support matrix for Unreal Engine:**

| AGDK Component | UE Integration | Since |
|---|---|---|
| GameActivity (AGDK) | Not adopted (uses own GameActivity) | — |
| Swappy frame pacing | Official, default on | UE 5.2 |
| ADPF | Official plugin (android/adpf-unreal-plugin) | UE 5.0+ |
| Oboe | Not integrated | — |
| Memory Advice API | Not integrated | — |

### 19.3 Godot 4

Godot 4 approaches Android integration more conservatively — its plugin ecosystem is in active development and AGDK adoption has been incremental.

**Activity type**: Godot 4 currently uses `NativeActivity`. A `GameActivity` migration is proposed in [godot-proposals#7692](https://github.com/godotengine/godot-proposals/issues/7692) but had not shipped in stable releases as of mid-2026.

**Swappy**: Added in **Godot 4.4** (PR [#96439](https://github.com/godotengine/godot/pull/96439), shipped in 4.4-dev4, November 2024). Godot's Vulkan renderer queries `VK_GOOGLE_display_timing` at device initialisation; when present, Swappy's Vulkan integration (`SwappyVk_initAndGetRefreshCycleDuration`) is activated. There is no GLES3 path since Godot 4 uses Vulkan as its primary mobile renderer. Configure via `swappy_mode` in the Android export preset.

**Vulkan frame timing**: Even before Swappy was integrated, Godot 4 queried `VK_GOOGLE_display_timing` to compute `VkDisplayPresentInfoGOOGLE.desiredPresentTime` for each frame — the same mechanism Swappy uses internally. Swappy in Godot replaces this manual computation with Swappy's pipeline-depth-aware scheduling.

**Other AGDK components**: No first-party Oboe, ADPF, or Memory Advice API integration as of Godot 4.4. The GDExtension system (Godot 4's native plugin layer, replacing GDNative from 3.x) can host any Android `.aar`/`.so`, so community plugins can wrap these.

**AGDK support matrix for Godot 4:**

| AGDK Component | Godot Integration | Since |
|---|---|---|
| GameActivity (AGDK) | Proposed, not shipped | — |
| Swappy frame pacing | Official | Godot 4.4 |
| ADPF | Not integrated | — |
| Oboe | Not integrated | — |
| Memory Advice API | Not integrated | — |

### 19.4 Bevy

Bevy's Android story is uniquely Rust-native and sits closer to the AGDK primitives than any higher-level engine.

**Activity type**: Bevy uses the [`android-activity`](https://github.com/rust-mobile/android-activity) crate (maintained by the `rust-mobile` project) as its Android glue layer. The crate provides two Cargo feature flags:
- `android-game-activity`: wraps `androidx.games.GameActivity` 4.4.0+ — default since **Bevy 0.14** (requires Android API 31+)
- `android-native-activity`: wraps the legacy `android.app.NativeActivity` — use for API < 31 targets

```toml
# Cargo.toml
[target.'cfg(target_os = "android")'.dependencies]
bevy = { version = "0.15", features = ["android-game-activity"] }
```

The `android-activity` crate handles the `ANativeActivity` / `GameActivity` lifecycle callbacks, input event pumping via `GameTextInput`, and `ANativeWindow` surface creation — mapping these into `winit`'s event loop abstraction that Bevy uses on all platforms.

**Swappy, Oboe, ADPF**: Not integrated in `android-activity` or in Bevy itself. The crate scope is strictly the Activity glue and window surface. Bevy's Vulkan backend (`wgpu` via `ash`) calls `vkQueuePresentKHR` directly without Swappy wrapping. Integrating Swappy would require calling `SwappyVk_initAndGetRefreshCycleDuration` after `vkCreateDevice` and replacing `vkQueuePresentKHR` with `SwappyVk_queuePresent` in Bevy's `wgpu` Vulkan backend — achievable as a community crate but not yet shipped.

**ADPF**: No integration. Bevy's Android support does not yet include performance hint sessions. Calling `APerformanceHint_createSession()` via JNI is straightforward (the NDK function is available) and could be wrapped in a Bevy plugin, but no first-party effort exists.

**Overall**: Bevy is the most AGDK-primitive of the four engines — it uses `GameActivity` directly via `android-activity` and gets frame timing from `wgpu`'s Vulkan path. The higher AGDK components (Swappy, Oboe, ADPF) require community plugins or fork-level integration.

**AGDK support matrix for Bevy:**

| AGDK Component | Bevy Integration | Since |
|---|---|---|
| GameActivity (AGDK) | Official (android-activity crate) | Bevy 0.14 |
| Swappy frame pacing | Not integrated | — |
| ADPF | Not integrated | — |
| Oboe | Not integrated | — |
| Memory Advice API | Not integrated | — |

### 19.5 Cross-Engine Summary

| Feature | Unity | Unreal | Godot 4 | Bevy |
|---|---|---|---|---|
| **Activity type** | GameActivity (default Unity 6) | Own NativeActivity subclass | NativeActivity (GameActivity proposed) | GameActivity via android-activity |
| **Swappy frame pacing** | ✅ 2019.2+ | ✅ UE 5.2+ | ✅ Godot 4.4+ | ❌ |
| **ADPF** | ✅ Adaptive Performance pkg | ✅ adpf-unreal-plugin | ❌ | ❌ |
| **Oboe audio** | ✅ 2023+ | ❌ | ❌ | ❌ |
| **Memory Advice API** | ✅ JNI helper | ❌ | ❌ | ❌ |
| **Paddleboat** | Indirect (Input System) | Via UE gamepad API | Via Godot Input | Manual JNI |
| **ARCore** | ✅ AR Foundation + ARCore XR Plugin | ✅ ARCore UE Plugin (OpenXR trend) | ✅ godot_arcore GDExtension | ❌ |

---

## 20. Google Play Games Services: Leaderboards, Achievements, and Cloud Save

**Google Play Games Services** (GPGS) provides the social and persistence layer for Android games: leaderboards, achievements, cloud-saved game state, and multiplayer matchmaking. Although GPGS is a Java/Kotlin API (no native C header), it integrates naturally into `GameActivity`-based titles through the Kotlin `MainActivity` subclass and JNI callbacks from the C++ game loop. The 2022 **GPGS SDK v2** removed the dependency on Google Sign-In, using Play Games identity automatically — a significant simplification.

### Gradle Dependency (SDK v2)

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.google.android.gms:play-services-games-v2:20.1.2")
}
```

### Initialisation and Sign-In

With SDK v2, there is no explicit sign-in step for most use cases — the SDK uses the device's Play Games account automatically. Initialise once in `Application.onCreate()` or `MainActivity.onCreate()`:

```kotlin
import com.google.android.gms.games.PlayGamesSdk
import com.google.android.gms.games.GamesSignInClient
import com.google.android.gms.games.PlayGames

class MainActivity : GameActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        PlayGamesSdk.initialize(this)

        // Optional: verify sign-in status
        val signInClient: GamesSignInClient = PlayGames.getGamesSignInClient(this)
        signInClient.isAuthenticated.addOnCompleteListener { task ->
            val isAuthenticated = task.isSuccessful && task.result.isAuthenticated
            if (isAuthenticated) {
                // Retrieve player ID for leaderboard/achievement calls:
                PlayGames.getPlayersClient(this).currentPlayer
                    .addOnSuccessListener { player ->
                        nativeSetPlayerId(player.playerId)
                    }
            }
        }
    }
}
```

[Source: GPGS v2 sign-in guide](https://developer.android.com/games/pgs/v2)

### Leaderboards

GPGS leaderboards store integer scores associated with named leaderboard IDs (created in the Google Play Console). The most common operations are score submission and displaying the built-in leaderboard UI:

```kotlin
val leaderboardsClient = PlayGames.getLeaderboardsClient(this)

// Submit a score (non-blocking, fire-and-forget):
leaderboardsClient.submitScore(LEADERBOARD_ID_FASTEST_RUN, playerScoreMs)

// Load top 25 scores (async):
leaderboardsClient.loadTopScores(
    LEADERBOARD_ID_FASTEST_RUN,
    LeaderboardVariant.TIME_SPAN_ALL_TIME,
    LeaderboardVariant.COLLECTION_PUBLIC,
    25
).addOnSuccessListener { data ->
    val scores = data.get()?.scores
    // scores: List<LeaderboardScore> with playerName, rawScore, scoreTag
    data.get()?.release()
}

// Show built-in leaderboard UI (starts a new activity):
leaderboardsClient.getLeaderboardIntent(LEADERBOARD_ID_FASTEST_RUN)
    .addOnSuccessListener { intent -> startActivityForResult(intent, RC_LEADERBOARD) }
```

### Achievements

GPGS supports two achievement types: **standard** (either locked or unlocked) and **incremental** (a counter progressing toward a target):

```kotlin
val achievementsClient = PlayGames.getAchievementsClient(this)

// Unlock a standard achievement:
achievementsClient.unlock(ACHIEVEMENT_FIRST_BOSS_DEFEATED)

// Increment an incremental achievement (e.g., "Kill 100 enemies"):
achievementsClient.increment(ACHIEVEMENT_ENEMY_SLAYER, 1 /*steps*/)

// Reveal a hidden achievement (makes it visible before it can be unlocked):
achievementsClient.reveal(ACHIEVEMENT_SECRET_AREA_FOUND)

// Show built-in achievements UI:
achievementsClient.getAchievementsIntent()
    .addOnSuccessListener { intent -> startActivityForResult(intent, RC_ACHIEVEMENTS) }
```

### Cloud Save (Snapshots API)

The **Snapshots** API stores arbitrary byte arrays (serialised game state) in Google's cloud, enabling cross-device save sync and conflict resolution:

```kotlin
val snapshotClient = PlayGames.getSnapshotsClient(this)

// Save game state:
fun saveToCloud(slotName: String, gameStateBytes: ByteArray) {
    snapshotClient.open(slotName, true /*createIfNotFound*/,
            SnapshotsClient.RESOLUTION_POLICY_MOST_RECENTLY_MODIFIED)
        .addOnSuccessListener { dataOrConflict ->
            if (dataOrConflict.isConflict) {
                // Handle conflict by picking most recent
                val conflict = dataOrConflict.conflict
                val resolved = conflict.mostRecentSnapshot
                snapshotClient.resolveConflict(conflict.conflictId, resolved)
                return@addOnSuccessListener
            }
            val snapshot = dataOrConflict.data!!
            snapshot.snapshotContents.writeBytes(gameStateBytes)
            val metadata = SnapshotMetadataChange.Builder()
                .setDescription("Level 5, 3 stars")
                .setPlayedTimeMillis(playtimeMs)
                .build()
            snapshotClient.commitAndClose(snapshot, metadata)
        }
}

// Load game state:
fun loadFromCloud(slotName: String, callback: (ByteArray?) -> Unit) {
    snapshotClient.open(slotName, false /*createIfNotFound*/,
            SnapshotsClient.RESOLUTION_POLICY_MOST_RECENTLY_MODIFIED)
        .addOnSuccessListener { dataOrConflict ->
            val data = dataOrConflict.data ?: return@addOnSuccessListener callback(null)
            val bytes = data.snapshotContents.readFully()
            snapshotClient.discardAndClose(data)
            callback(bytes)
        }
        .addOnFailureListener { callback(null) }
}
```

The `RESOLUTION_POLICY_MOST_RECENTLY_MODIFIED` strategy resolves conflicts automatically by keeping the most recently written save. For games where progress can merge (e.g., collecting items across sessions), a custom resolution strategy can compare save state fields. [Source: Snapshots API guide](https://developer.android.com/games/pgs/snapshots)

### Multiplayer

GPGS provides two multiplayer models:

**Turn-based multiplayer** (`TurnBasedMultiplayerClient`): Asynchronous games (chess, word games) where players take turns over hours or days. Matches are persisted in the cloud; push notifications alert players when it is their turn. Up to 8 participants per match.

**Real-time multiplayer** (`RealTimeMultiplayerClient`): Synchronous games with P2P data transfer relayed through Google's TURN servers. Up to 8 players; data is sent via `sendReliableMessage` (TCP-equivalent) or `sendUnreliableMessage` (UDP-equivalent). Latency via TURN relay is typically 50–200 ms. For games requiring sub-30 ms latency, a custom WebRTC or dedicated server solution is preferable.

### Calling GPGS from C++ via JNI

All GPGS APIs are Java/Kotlin — no NDK headers exist. The standard pattern for C++ games is to cache method IDs at startup and invoke them from game event callbacks:

```cpp
// In Java/Kotlin (MainActivity.kt):
// @Keep
// fun submitScore(leaderboardId: String, score: Long) {
//     PlayGames.getLeaderboardsClient(this).submitScore(leaderboardId, score)
// }

// In C++ (game_main.cpp):
void submit_score(JNIEnv* env, jobject activity, const char* lb_id, int64_t score) {
    jclass cls = env->GetObjectClass(activity);
    jmethodID method = env->GetMethodID(cls, "submitScore",
        "(Ljava/lang/String;J)V");
    jstring jlb_id = env->NewStringUTF(lb_id);
    env->CallVoidMethod(activity, method, jlb_id, (jlong)score);
    env->DeleteLocalRef(jlb_id);
    env->DeleteLocalRef(cls);
}
```

Cache the `jclass` and `jmethodID` at startup (after `JNI_OnLoad`) since `GetMethodID` is relatively expensive. [Source: GPGS overview](https://developer.android.com/games/pgs) | [GPGS v2 migration](https://developer.android.com/games/pgs/v2)

---

## 21. Foldable Devices and Multi-Window: Adaptive Game Layouts

Foldable smartphones (Samsung Galaxy Z Fold/Flip, Motorola Razr, Google Pixel Fold) and Android tablets present game developers with a new class of layout challenge: the display dimensions, aspect ratio, and even the physical screen continuity can change at runtime without an `Activity` restart. Correct handling requires understanding Android's multi-window lifecycle and the Jetpack Window Manager's foldable device model.

### Multi-Window Mode (Android 7.0+)

Any Android 7.0+ device can place apps in split-screen (two apps side by side) or, on Android 12L+, free-form windowing on tablets. A game running in a split-screen half may receive an `ANativeWindow` that is 540×1080 rather than 1080×1920 — and the aspect ratio constraint that the game's UI was designed for no longer holds.

Detect multi-window mode from the Kotlin layer:

```kotlin
override fun onMultiWindowModeChanged(isInMultiWindowMode: Boolean, newConfig: Configuration) {
    super.onMultiWindowModeChanged(isInMultiWindowMode, newConfig)
    if (isInMultiWindowMode) {
        // Game may be in a small window; reduce quality
        nativeSetMultiWindowMode(true)
    } else {
        nativeSetMultiWindowMode(false)
    }
}
```

From native code, the simplest response to multi-window is to pause rendering or drop to a lower quality preset — the user is unlikely to play an action game in a half-screen window.

### Foldable Device States (Jetpack Window Manager)

Jetpack Window Manager (`androidx.window:window:1.3.0`) provides the `WindowInfoTracker` API for querying fold state:

```kotlin
import androidx.window.layout.WindowInfoTracker
import androidx.window.layout.FoldingFeature

class MainActivity : GameActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            WindowInfoTracker.getOrCreate(this@MainActivity)
                .windowLayoutInfo(this@MainActivity)
                .collect { layoutInfo ->
                    val fold = layoutInfo.displayFeatures
                        .filterIsInstance<FoldingFeature>()
                        .firstOrNull()

                    when {
                        fold == null ->
                            nativeSetFoldState(FOLD_STATE_FLAT)  // non-foldable
                        fold.state == FoldingFeature.State.FLAT ->
                            nativeSetFoldState(FOLD_STATE_FLAT)  // unfolded
                        fold.state == FoldingFeature.State.HALF_OPENED &&
                            fold.orientation == FoldingFeature.Orientation.HORIZONTAL ->
                            nativeSetFoldState(FOLD_STATE_TABLETOP) // screen bent horizontally
                        fold.state == FoldingFeature.State.HALF_OPENED &&
                            fold.orientation == FoldingFeature.Orientation.VERTICAL ->
                            nativeSetFoldState(FOLD_STATE_BOOK)   // screen bent vertically
                    }
                }
        }
    }
}
```

In **tabletop mode** (hinge horizontal, screen at ~90–150° angle), the bottom half of the screen is typically resting on a surface. Games can use this layout creatively: show game controls on the lower half and the game view on the upper half, using the hinge as a natural split point. The `FoldingFeature.bounds` rectangle (in window coordinates) gives the hinge's exact pixel position.

In **book mode** (hinge vertical), the two halves of the display show a split view. Games designed for a single continuous display may want to add letterboxing or re-layout their UI to avoid the hinge occluding important content.

[Source: Jetpack Window Manager foldable guide](https://developer.android.com/guide/topics/large-screens/make-apps-fold-aware)

### Swapchain Recreation on Window Resize

When the window size changes — either from multi-window resizing or from fold state change — `GameActivity` delivers `APP_CMD_WINDOW_RESIZED`. The native code **must** respond by destroying and recreating the Vulkan swapchain with the new dimensions:

```cpp
case APP_CMD_WINDOW_RESIZED: {
    int32_t new_width  = ANativeWindow_getWidth(app->window);
    int32_t new_height = ANativeWindow_getHeight(app->window);
    if (new_width != current_swap_width || new_height != current_swap_height) {
        // Wait for GPU idle before destroying swapchain:
        vkDeviceWaitIdle(device);
        // Destroy swapchain-dependent resources:
        destroy_framebuffers();
        vkDestroySwapchainKHR(device, swapchain, nullptr);
        // Recreate with new dimensions:
        create_swapchain(new_width, new_height);
        create_framebuffers(new_width, new_height);
        current_swap_width  = new_width;
        current_swap_height = new_height;
    }
    break;
}
```

Failure to recreate the swapchain after a size change causes rendering to the wrong-sized surface — Vulkan validation layers will issue a `VK_ERROR_OUT_OF_DATE_KHR` from `vkQueuePresentKHR`, and visual artefacts or black screens will result.

### Display Cutout Handling

Modern Android devices have notches (punch-hole cameras) and edge cutouts. On foldable devices, the crease itself is not a cutout but the camera notch may be present on one or both display halves. The **safe area insets** (the region guaranteed to be free of hardware cutouts) are available via:

```kotlin
// In GameActivity subclass:
val insets = ViewCompat.getRootWindowInsets(window.decorView)
val cutout = insets?.displayCutout
val safeTop    = cutout?.safeInsetTop    ?: 0  // pixels from top
val safeBottom = cutout?.safeInsetBottom ?: 0
val safeLeft   = cutout?.safeInsetLeft   ?: 0
val safeRight  = cutout?.safeInsetRight  ?: 0
nativeSetSafeInsets(safeTop, safeBottom, safeLeft, safeRight)
```

In native code, use these insets to constrain the game UI (HUD, touch controls, score display) to the safe area. Game world rendering can extend to the full display including the cutout area; UI elements must not.

### Large Screen and Tablet Optimisations

Games designed for 16:9 portrait phones often letterbox on 4:3 tablets or display in a windowed mode with black bars. Opt into full-screen resizable behaviour:

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:resizeableActivity="true"
    android:maxAspectRatio="2.4"
    android:minAspectRatio="1.0" >
    <!-- Declare supported screen sizes: -->
    <supports-screens
        android:xlargeScreens="true"
        android:largeScreens="true" />
</activity>
```

With `resizeableActivity="true"`, the game receives the full display area on tablets. The `maxAspectRatio` and `minAspectRatio` attributes prevent extreme proportions (ultra-wide monitors via DeX, etc.) from breaking layouts.

### Testing Foldable Layouts

Android Studio provides foldable emulator profiles:
- **Generic Foldable** (API 30+): 2208×1840 unfolded, 1768×1840 folded; simulates horizontal fold.
- **Pixel Fold** (API 34): 2092×2092 main display, 1080×2092 cover display.

Use **developer options → Simulated foldable** on supported physical devices (Samsung Galaxy Z Fold) to test fold/unfold without a physical device. The emulator's `adb shell wm folding-angle <degrees>` command simulates intermediate fold angles (0° = flat, 90° = tabletop). [Source: Large screen adaptations for games](https://developer.android.com/games/guides/large-screens)

---

## 22. Integrations

- **Ch85 — SurfaceFlinger and BufferQueue**: `ANativeWindow` is the producer end of a `BufferQueue`; SurfaceFlinger is the consumer. Every buffer submitted through a Vulkan swapchain or EGL surface traverses the pipeline described in Ch85. The HWComposer path (hardware overlay vs. GPU composition) applies equally to GameActivity windows.

- **Ch86 — Vulkan on Android**: The Vulkan swapchain creation path (`vkCreateAndroidSurfaceKHR`, `ANativeWindow`, TBDR-optimised render passes) is covered in depth in Ch86. `APP_CMD_INIT_WINDOW` / `APP_CMD_TERM_WINDOW` as the correct swapchain creation/destruction hooks are described there. Swappy's `SwappyVk_queuePresent` replaces `vkQueuePresentKHR` but sits above the Vulkan layer described in Ch86.

- **Ch86 — @FastNative / @CriticalNative**: The JNI transition cost that `@FastNative` reduces is the same cost that `GameActivity` eliminates from the per-frame render path. The two approaches are complementary: `@FastNative` optimises unavoidable JNI calls (framework boundaries); GameActivity eliminates JNI from the hot path entirely.

- **Ch87 — Android AR / ARCore / Android XR**: ARCore's `ArSession_update()` loop and the OpenXR `xrWaitFrame()` / `xrEndFrame()` loop are analogous to `android_main()`'s event loop. Section §18 of this chapter covers the ARCore C API + GameActivity integration pattern in depth. GameActivity and `ANativeWindow` are also used in ALVR and WiVRn to host the decoded VR stream surface on a headset.

- **Ch26 — VA-API and Video Decode**: Android's `MediaCodec` codec pipeline, accessible from C via `AMediaCodec` (NDK), produces decoded frames as `AHardwareBuffer` that can be imported into Vulkan via `VK_ANDROID_external_memory_android_hardware_buffer` — the same AHardwareBuffer model described in Ch85 and Ch86. Oboe's AAudio MMAP path and VA-API's DMA-BUF zero-copy both aim at the same goal: eliminating unnecessary copies between hardware subsystems.

- **Ch75 — Explicit GPU Sync**: Swappy's fence-based backpressure mechanism (`SwappyVk_queuePresent` inserting `VkFence` waits) uses the same `dma_fence` / `sync_file` infrastructure described in Ch75. The explicit sync model now unified across Android and Wayland (`wp_linux_drm_syncobj_v1`) has its Android origin in the `android::Fence` / `sw_sync` timeline that Swappy builds on.

- **Ch96 — libcamera**: Android's `AMediaCodec` / Camera2 NDK (`ACameraManager`, `ACameraDevice`, `AImageReader`) parallels libcamera's role on Linux. The AGDK audio path (Oboe → AAudio → audio HAL) parallels PipeWire's role for audio on Linux desktop.

---

## Roadmap

### Near-term (6–12 months)
- **`games-activity` v5 and 16KB page alignment**: Android 16 requires all native `.so` files to be aligned to 16 KB boundaries. AGP 9+ (`com.android.application` 9.x) enforces this at build time; AGDK `games-activity` 4.4+ and the Prefab AAR build system produce 16KB-aligned binaries by default. Apps still using `android_native_app_glue` from an older NDK revision need to rebuild with NDK r27b+ to meet the requirement.
- **ADPF (Android Dynamic Performance Framework) deeper AGDK integration**: The `APerformanceHint` API (NDK, Android 12+) lets apps create a hint session for their render thread and report actual vs. target work durations. AGDK is expected to expose ADPF hint sessions directly from the Swappy pipeline — `SwappyVk_setAPerformanceHintSession()` — so frame-pacing feedback automatically informs the scheduler's CPU/GPU clock headroom decisions without app-layer wiring.
- **Variable refresh rate (VRR) support in Swappy**: Displays with 1–120 Hz VRR panels (Android 13+ `DisplayManager.getSupportedModes()`) require Swappy to select a target frame interval dynamically rather than fixing to 60 Hz or 90 Hz. A `SwappyVk_setPreferredRefreshRateRange()` API is in review, targeting Swappy 2.2.
- **Oboe spatial audio**: The Android `Spatializer` API (Android 12+, `android.media.Spatializer`) provides head-tracked binaural rendering. Oboe 2.0 is expected to expose a `SpatializerOutputMode` in `AudioStreamBuilder` that routes game audio through the platform spatializer — one API call to get HRTF-based 3D audio without a third-party binaural library.
- **`games-memory-advice` ML model update**: The v2.0 ML-based memory pressure predictor (September 2023) is being retrained on Android 15+ device telemetry, including the new `/proc/meminfo` fields added for GKI. The retrained model is expected to reduce false `APPROACHING_LIMIT` signals on devices with large swap partitions (Android 15 introduces opt-in compressed swap for flagship phones).

### Medium-term (1–3 years)
- **32-bit ABI end-of-life**: Google has announced that the Play Store will stop accepting APKs targeting only 32-bit ABIs for new apps (2026) and updates to existing apps (2027). Games using `armeabi-v7a`-only native libraries must add `arm64-v8a` targets. AGDK `games-activity` static libraries for `armeabi-v7a` will be maintained but not receive new features after NDK r30.
- **Jetpack Compose + GameActivity hybrid**: As Compose for Android matures (Compose 2.x targeting Android 8+), games want Compose for settings screens and overlay UI while keeping GameActivity for the render loop. The expected solution is `ComposeGameActivity` — a `GameActivity` subclass that embeds a `ComposeView` as a transparent overlay, letting Compose UI coexist with a Vulkan swapchain without `SurfaceView` z-order conflicts.
- **Android GPU Inspector successor**: AGI is in maintenance mode (no new GPU counter schema since 2024). Google's primary GPU profiling investment is shifting to Perfetto GPU counter tracks (available since Android 12) and Android Studio's built-in CPU/GPU timeline. AGI frame capture functionality is expected to be superseded by a new frame capture tool integrated into Android Studio Electric Eel+, with RenderDoc-compatible capture format.
- **Google Play Games on PC — AGDK alignment**: The Google Play Games Windows client (2022+) runs Android apps in a custom Android Runtime on Windows. AGDK libraries already compile for the `x86_64` ABI used by the Play Games PC runtime. `SwappyVk` and `SwappyGL` targeting D3D12 (via ANGLE) are being validated; Paddleboat's controller mapping for Xbox controllers on Windows is the primary controller integration work in progress.
- **Hardware ray tracing on mobile**: Adreno 830 (Snapdragon 8 Elite) and Mali Immortalis-G920 both include hardware BVH traversal units. The Android Vulkan Profiles 2026+ tier is expected to make `VK_KHR_acceleration_structure` + `VK_KHR_ray_tracing_pipeline` required on new high-end Android devices. AGDK's Android GPU Inspector will need counter schema updates for ray traversal and BVH build timing; Swappy will need awareness of ray-tracing workload submission latency which differs from raster pipelines.

### Long-term
- **Neural rendering on mobile**: Neural Radiance Fields (NeRF) and 3D Gaussian Splatting (3DGS) are GPU-intensive at both training and inference time. As LiteRT-LM INT4 inference and dedicated NPU acceleration bring NeRF/3DGS rendering within mobile GPU budget (est. 2028–2030), a `GameActivity`-hosted Vulkan render loop consuming a LiteRT-inferred radiance field becomes the standard architecture for photorealistic environment capture in mobile games. Swappy's frame pacing handles the variable latency of neural inference per frame.
- **Unified AGDK cross-platform**: AGDK libraries (Swappy, Oboe, Paddleboat) have been evaluated for portability to ChromeOS and Android XR. A future `libgames-platform` abstraction is discussed internally that would let a single CMake target link against `SwappyVk` on Android, `SwappyVk_chromeos` on ChromeOS, and `SwappyVk_xr` on Android XR headsets — identical source, platform-selected backend.

---

*[Source: AGDK overview](https://developer.android.com/games/agdk) | [GameActivity guide](https://developer.android.com/games/agdk/game-activity/overview) | [Paddleboat](https://developer.android.com/games/sdk/game-controller) | [Swappy](https://developer.android.com/games/sdk/frame-pacing) | [Oboe](https://github.com/google/oboe) | [Performance Tuner](https://developer.android.com/games/sdk/performance-tuner) | [Memory Advice](https://developer.android.com/games/sdk/memory-advice) | [AGI](https://gpuinspector.dev/)*
