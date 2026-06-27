# Chapter 161 — Android Game Development Kit (AGDK): Native Game Architecture, Input, Audio, and Frame Pacing

> **Audiences:** Graphics application developers building native Android games and high-performance apps; systems developers who want to understand how Android's C/C++ game stack maps onto the graphics, audio, and input subsystems; browser and tool engineers who need to understand the NDK lifecycle model that underpins Chrome's Android renderer and Android Studio's profiling toolchain.

---

## Table of Contents

1. [What AGDK Is](#1-what-agdk-is)
   - 1.1 [Component Overview and Distribution](#11-component-overview-and-distribution)
   - 1.2 [Historical Context: Android Game Development Before AGDK](#12-historical-context-android-game-development-before-agdk)
2. [NativeActivity: The Foundation and Its Limits](#2-nativeactivity-the-foundation-and-its-limits)
3. [GameActivity: The Modern Native App Model](#3-gameactivity-the-modern-native-app-model)
4. [Input Handling with GameActivity](#4-input-handling-with-gameactivity)
5. [Paddleboat: Game Controller Library](#5-paddleboat-game-controller-library)
6. [Android Frame Pacing: Swappy](#6-android-frame-pacing-swappy)
7. [ANativeWindow: The Surface Handle](#7-anativewindow-the-surface-handle)
8. [Oboe: Low-Latency Audio](#8-oboe-low-latency-audio)
9. [Android Performance Tuner](#9-android-performance-tuner)
10. [Memory Advice API](#10-memory-advice-api)
11. [Android GPU Inspector](#11-android-gpu-inspector)
12. [ARCore Integration: Building AR-Native Games](#12-arcore-integration-building-ar-native-games)
13. [Engine Integration: Unity, Unreal, Godot, and Bevy](#13-engine-integration-unity-unreal-godot-and-bevy)
14. [Integrations](#14-integrations)

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
- **`android_native_app_glue`**: The NDK-provided threading wrapper (§2) that serialised Android callbacks to a background pthread via a pipe — keeping the game loop off the Java main thread and preventing ANR watchdog triggers.
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

---

## 2. NativeActivity: The Foundation and Its Limits

Understanding `NativeActivity` is prerequisite to understanding why `GameActivity` was needed. `NativeActivity` remains supported and is still used in simpler apps, but its limitations are load-bearing motivations for everything in §3 onwards.

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
    ANativeWindow*    window;       // non-NULL when surface is ready (see §7)
    ARect             contentRect;
    int               activityState;
    int               destroyRequested;
};

extern void android_main(struct android_app* app);  // application entry point
```

The `ALooper` is the multiplexed event source: `ALooper_pollOnce(timeoutMs, &fd, &events, &data)` blocks until a command or input event is ready, then calls the associated `android_poll_source::process` callback.

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

**3. Coarse input event batching.** Touch events arrive via `AInputQueue` one at a time per `ALooper_pollOnce` iteration. High-frequency stylus or multi-touch events are batched by the system and historical samples are not exposed through the C API — the Java `MotionEvent.getHistoricalX()` path has no NDK equivalent in `NativeActivity`.

---

## 3. GameActivity: The Modern Native App Model

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

## 4. Input Handling with GameActivity

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

## 5. Paddleboat: Game Controller Library

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

## 6. Android Frame Pacing: Swappy

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

## 7. ANativeWindow: The Surface Handle

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

## 8. Oboe: Low-Latency Audio

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

## 9. Android Performance Tuner

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

## 10. Memory Advice API

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

## 11. Android GPU Inspector

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

## 12. ARCore Integration: Building AR-Native Games

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

    // 3. Render AR content with view_mat/proj_mat via Vulkan (see §12.2)
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

## 13. Engine Integration: Unity, Unreal, Godot, and Bevy

Most Android games are not written directly against AGDK — they go through a game engine that internally consumes AGDK components. This section traces how each major engine maps onto the libraries described in this chapter.

### 13.1 Unity

Unity's Android backend is the most deeply integrated with AGDK, largely because Google has co-developed several features with the Unity team.

**Activity type**: Unity used `NativeActivity` from the beginning. Unity 2023.1 introduced `GameActivity` as an opt-in alternative; Unity 6 (2024) made `GameActivity` the default for new projects. The switch is controlled by **Edit → Project Settings → Player → Android → Activity Type**. GameActivity provides text-input handling via `GameTextInput` (§3), eliminating the overlay workaround Unity historically used for on-screen keyboards. [Source: Unity GameActivity docs](https://developer.android.com/games/engines/unity/gameactivity)

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

### 13.2 Unreal Engine

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

### 13.3 Godot 4

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

### 13.4 Bevy

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

### 13.5 Cross-Engine Summary

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

## 14. Integrations

- **Ch85 — SurfaceFlinger and BufferQueue**: `ANativeWindow` is the producer end of a `BufferQueue`; SurfaceFlinger is the consumer. Every buffer submitted through a Vulkan swapchain or EGL surface traverses the pipeline described in Ch85. The HWComposer path (hardware overlay vs. GPU composition) applies equally to GameActivity windows.

- **Ch86 — Vulkan on Android**: The Vulkan swapchain creation path (`vkCreateAndroidSurfaceKHR`, `ANativeWindow`, TBDR-optimised render passes) is covered in depth in Ch86. `APP_CMD_INIT_WINDOW` / `APP_CMD_TERM_WINDOW` as the correct swapchain creation/destruction hooks are described there. Swappy's `SwappyVk_queuePresent` replaces `vkQueuePresentKHR` but sits above the Vulkan layer described in Ch86.

- **Ch86 — @FastNative / @CriticalNative**: The JNI transition cost that `@FastNative` reduces is the same cost that `GameActivity` eliminates from the per-frame render path. The two approaches are complementary: `@FastNative` optimises unavoidable JNI calls (framework boundaries); GameActivity eliminates JNI from the hot path entirely.

- **Ch87 — Android AR / ARCore / Android XR**: ARCore's `ArSession_update()` loop and the OpenXR `xrWaitFrame()` / `xrEndFrame()` loop are analogous to `android_main()`'s event loop. Section 12 of this chapter covers the ARCore C API + GameActivity integration pattern in depth. GameActivity and `ANativeWindow` are also used in ALVR and WiVRn to host the decoded VR stream surface on a headset.

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
