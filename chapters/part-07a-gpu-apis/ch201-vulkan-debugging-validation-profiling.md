# Chapter 201: Vulkan Debugging, Validation, and Profiling

> **Part**: Part VII-A — GPU APIs
> **Audience**: Primarily graphics application developers debugging correctness and performance problems in Vulkan renderers, and systems/driver developers diagnosing GPU-side hangs, corruption, and stalls that surface through these tools.
> **Status**: First draft — 2026-08-21

Vulkan's explicit design trades driver-side error checking for application-side correctness obligations, and moves the burden of catching mistakes onto opt-in, user-space tooling. This chapter is a working reference for that tooling on Linux: the `VK_LAYER_KHRONOS_validation` meta-layer and its sub-features (core validation, GPU-Assisted Validation, Synchronization Validation, Debug Printf), the debug-instrumentation extensions applications wire into their own code (`VK_EXT_debug_utils`, `VK_EXT_device_fault`), and the frame- and hardware-level profilers — RenderDoc, AMD's Radeon GPU Profiler, Intel's Graphics Performance Analyzers, and NVIDIA Nsight Graphics — that turn a captured frame or a GPU crash into an actionable diagnosis. Chapter 148 is the authoritative reference on Vulkan synchronization semantics; this chapter's synchronization-validation section covers only the tooling that detects violations of that model, not the model itself.

---

## Table of Contents

- [1. `VK_LAYER_KHRONOS_validation` Activation and Sub-Layers](#1-vk_layer_khronos_validation-activation-and-sub-layers)
  - [1.1 Activation Paths](#11-activation-paths)
  - [1.2 Configuring the Layer with `VkLayerSettingsCreateInfoEXT`](#12-configuring-the-layer-with-vklayersettingscreateinfoext)
  - [1.3 The Sub-Layers](#13-the-sub-layers)
- [2. Best Practices Sub-Layer](#2-best-practices-sub-layer)
- [3. Synchronization Validation](#3-synchronization-validation)
- [4. GPU-Assisted Validation](#4-gpu-assisted-validation)
- [5. Debug Printf](#5-debug-printf)
- [6. `VK_EXT_debug_utils`](#6-vk_ext_debug_utils)
- [7. `VK_EXT_device_fault`](#7-vk_ext_device_fault)
- [8. RenderDoc Deep Dive](#8-renderdoc-deep-dive)
- [9. AMD Radeon GPU Profiler](#9-amd-radeon-gpu-profiler)
- [10. Intel Graphics Performance Analyzers](#10-intel-graphics-performance-analyzers)
- [11. NVIDIA Nsight Graphics](#11-nvidia-nsight-graphics)
- [12. `VK_KHR_performance_query`](#12-vk_khr_performance_query)
- [13. `vkconfig` — the Vulkan Configurator](#13-vkconfig--the-vulkan-configurator)
- [14. A Debugging Workflow](#14-a-debugging-workflow)
- [Integrations](#integrations)
- [References](#references)

---

## 1. `VK_LAYER_KHRONOS_validation` Activation and Sub-Layers

### 1.1 Activation Paths

`VK_LAYER_KHRONOS_validation` is the single, unified validation layer shipped by Khronos, distributed by every major Linux distribution as `vulkan-validationlayers` and by the LunarG Vulkan SDK tarball, with source at `github.com/KhronosGroup/Vulkan-ValidationLayers`. It supersedes the older per-feature layer chain (`VK_LAYER_LUNARG_core_validation`, `VK_LAYER_LUNARG_object_tracker`, and similar) that shipped before Vulkan 1.1; those names are gone from current SDKs, and code or scripts still enabling them individually will silently do nothing.

Four independent mechanisms enable and configure the layer, in the order LunarG's own documentation recommends them [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/khronos_validation_layer.md):

1. **`vkconfig`**, the Vulkan Configurator GUI shipped with the SDK (§13) — the recommended path for interactive development.
2. **`VK_EXT_layer_settings`**, an instance extension letting the application configure the layer programmatically at `vkCreateInstance` time (§1.2) — the recommended path for CI and reproducible builds.
3. **A `vk_layer_settings.txt` configuration file**, read from the current working directory or from `$HOME/.local/share/vulkan/settings.d/` on Linux.
4. **Environment variables** — `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` to enable the layer itself, and `VK_LOADER_LAYERS_ENABLE`/`VK_LAYER_PATH` to point the loader at a locally built layer instead of the distro package. `VK_LAYER_PATH` searches its `explicit_layer.d` manifest directory ahead of the system directories, which lets a developer test a freshly built validation layer against a distribution-packaged Mesa driver without root access.

Layers are also enabled programmatically by name in `VkInstanceCreateInfo::ppEnabledLayerNames`, which is the only mechanism guaranteed to work identically across development and shipped builds — environment variables are a development convenience that a packaged application should not rely on for its own debug builds.

### 1.2 Configuring the Layer with `VkLayerSettingsCreateInfoEXT`

`VK_EXT_layer_settings` replaces the older, deprecated `VK_EXT_validation_features`/`VkValidationFeaturesEXT` mechanism with a generic, layer-agnostic settings channel: any layer can expose named settings, and the application supplies values for the ones it cares about via a `VkLayerSettingsCreateInfoEXT` chained onto `VkInstanceCreateInfo::pNext`. Each entry is a `VkLayerSettingEXT` naming the target layer, a setting key, a `VkLayerSettingTypeEXT` (bool32, int32, string, and so on), and a pointer to the value array:

```c
// vk_layer_settings_example.c — enabling Debug Printf verbose output programmatically
VkBool32 printf_verbose = VK_TRUE;
VkLayerSettingEXT layer_setting = {
    .pLayerName   = "VK_LAYER_KHRONOS_validation",
    .pSettingName = "printf_verbose",
    .type         = VK_LAYER_SETTING_TYPE_BOOL32_EXT,
    .valueCount   = 1,
    .pValues      = &printf_verbose,
};
VkLayerSettingsCreateInfoEXT layer_settings_ci = {
    .sType        = VK_STRUCTURE_TYPE_LAYER_SETTINGS_CREATE_INFO_EXT,
    .settingCount = 1,
    .pSettings    = &layer_setting,
};
VkInstanceCreateInfo instance_ci = {
    .sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO,
    .pNext = &layer_settings_ci,
    /* ... */
};
```
[Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/khronos_validation_layer.md)

The full settings vocabulary — hundreds of keys covering every sub-layer below — is generated from the layer's JSON manifest and published as `khronos_validation_layer.html` alongside the layer sources; `vkconfig`'s GUI is a front-end over the same setting keys and can export the equivalent `vk_layer_settings.txt`, environment-variable script, or `VK_EXT_layer_settings` C++ snippet for a given configuration, which is a fast way to discover exact key names without hand-reading the manifest.

### 1.3 The Sub-Layers

`VK_LAYER_KHRONOS_validation` is not one monolithic checker; it is several independently toggleable validation objects compiled into one layer binary, each targeting a different class of bug:

- **Core validation** — object lifetime tracking (using a destroyed or never-created handle), parameter and handle validity against the specification's Valid Usage (VU) statements, and descriptor/render-pass/pipeline state consistency checks. This is the default-on baseline and the layer's original purpose.
- **Thread safety validation** — flags concurrent access to Vulkan objects from multiple threads without external synchronization, in scenarios the specification defines as externally-synchronized.
- **Stateless parameter validation** — the subset of VU checks that require no other Vulkan state (enum range checks, null-pointer checks, struct `sType` validation), run first as a cheap filter before more expensive state-dependent checks.
- **Best Practices** — non-error, performance-oriented warnings (§2).
- **Synchronization Validation** — hazard detection across barriers and command buffers (§3).
- **GPU-Assisted Validation** — SPIR-V-instrumented, GPU-side runtime checks (§4).
- **Debug Printf** — GPU-side `printf` diagnostics that reuse the GPU-Assisted Validation instrumentation path (§5).

Each is individually enabled or disabled through the `VkLayerSettingsCreateInfoEXT`/`vk_layer_settings.txt` mechanism in §1.2; core validation, thread safety, and stateless parameter checking are on by default, while Best Practices, Synchronization Validation, GPU-Assisted Validation, and Debug Printf are opt-in because of their cost or noise profile.

---

## 2. Best Practices Sub-Layer

The Best Practices sub-layer is enabled via `VK_VALIDATION_FEATURE_ENABLE_BEST_PRACTICES_EXT` (through `VkValidationFeaturesEXT` or the layer-settings equivalent) and reports messages tagged with `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT` in the debug messenger callback. Unlike core validation, none of these are specification violations — the API usage is legal — but each flags a pattern that is measurably wasteful on real hardware: allocating more descriptor sets than a driver can efficiently manage (a documented example is exceeding 4096 descriptor sets), using a non-dedicated allocation for a resource large enough to benefit from a dedicated one, redundant `loadOp` clears layered on top of an application-issued `vkCmdClearAttachments`, and unnecessary image layout transitions inserted where the same layout was already current [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/best_practices.md).

A second layer of Best Practices checks is vendor-specific and off by default even when the sub-layer itself is enabled, because the guidance is only valid for one IHV's hardware. AMD contributed a set of AMD-specific checks — GPUOpen's announcement frames them as encoding RDNA performance-guide advice directly into layer output [Source](https://gpuopen.com/learn/vulkan-best-practice-layer/) — enabled with:

```bash
VK_LAYER_ENABLES=VK_VALIDATION_FEATURE_ENABLE_BEST_PRACTICES_EXT;VALIDATION_CHECK_ENABLE_VENDOR_SPECIFIC_AMD \
VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation ./app
```

The same `VALIDATION_CHECK_ENABLE_VENDOR_SPECIFIC_*` mechanism is the extension point for other vendors' guidance as it is contributed upstream; readers should check the current `best_practices.md` in the Vulkan-ValidationLayers repository for which vendor filters exist at a given SDK version rather than assume AMD is the only one, since this is an area that has grown incrementally.

Practically, Best Practices is the sub-layer worth running continuously in a development build rather than only when chasing a specific bug — it is the only checker in the stack that catches *legal but wasteful* API usage before a profiler run is even needed.

---

## 3. Synchronization Validation

Synchronization Validation is enabled via `VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT` and performs command-buffer-level tracking of every resource access — reads and writes, by pipeline stage and access type — to detect the hazard classes Chapter 148 defines from first principles: read-after-write, write-after-write, and write-after-read races that are missing the pipeline barrier or subpass dependency required to order them. Where core validation checks that a barrier's *fields* are individually well-formed (valid stage/access mask combinations, a real image handle), Synchronization Validation checks that the barrier *set as a whole* is sufficient to make every access safe — the semantic layer above the syntactic one. Ch148 §"Synchronisation Validation" documents the availability/visibility model this tooling is checking against; this section covers only the checker's own behavior and failure modes.

Two properties of the checker matter in practice. First, it is a *replay-time* analysis performed on recorded command buffers, not a static analysis of shader code or a runtime GPU trace — it sees exactly the barriers, layout transitions, and access patterns the application recorded, and cannot see hazards introduced by GPU-side indirection it has no visibility into (indirect draw parameters, for instance). Second, it operates per-submission by default and does not track hazards across multiple `vkQueueSubmit` calls unless the relevant synchronization (semaphores, fences) is itself present in the traced state — a missing *cross-submission* hazard is a common source of intermittent, hard-to-reproduce corruption that Synchronization Validation alone will not surface if the two submissions are separated by no shared synchronization object at all.

Common false-positive patterns that a working knowledge of the checker's limits helps triage quickly:

- **Presentation-engine interactions.** The swapchain image's transition out of `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` and into a renderable layout on `vkAcquireNextImageKHR` is synchronized by the acquire semaphore, which the checker cannot always fully resolve when `VK_EXT_swapchain_maintenance1` or platform-specific present timing extensions are layered on top; spurious "image layout undefined at first access" warnings on the very first frame after acquire are a known noise source.
- **External APIs and interop.** Resources shared with EGL, CUDA, or another API via external memory/semaphore extensions have producer/consumer relationships the layer cannot observe on the other side of the interop boundary; Synchronization Validation will report a hazard on a resource that is, in fact, correctly synchronized by an external fence the layer never sees.
- **`VK_ACCESS_2_NONE` execution-only barriers.** As Ch148 notes, a pure write-after-read (WAR) barrier legitimately carries `VK_ACCESS_2_NONE` in both `srcAccessMask` and `dstAccessMask` since no cache flush is required — code unfamiliar with this pattern sometimes "fixes" the reported warning by adding unnecessary access masks, which silences nothing because there was no error, only informational tracking noise from an overly broad message severity filter.

Synchronization Validation should be run on every render-graph or pass-ordering change, not held in reserve for hang-hunting; it is cheap enough on CPU (it does no GPU-side instrumentation) to run in an application's own debug configuration continuously, unlike GPU-Assisted Validation.

---

## 4. GPU-Assisted Validation

GPU-Assisted Validation (GPU-AV) answers a question core validation structurally cannot: is the *data* a shader reads through a bound resource actually valid, given everything that happened on the GPU timeline before this shader ran? Core validation and Synchronization Validation both operate on the CPU, working from the command buffer's recorded structure; neither can see what index a shader computes at runtime to index into a descriptor array, nor whether a `VK_KHR_buffer_device_address` pointer dereferenced inside a shader actually points at live, in-bounds device memory.

GPU-AV closes that gap by rewriting the application's SPIR-V at `vkCreateShaderModule` time. The layer passes the shader's bytecode through the SPIR-V optimizer with an instrumentation pass that inserts additional instructions performing the runtime check, then hands the modified module to the driver as if it were the application's own [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/gpu_validation.md). Checks added this way include out-of-bounds descriptor array indexing and reads of an unwritten (null) descriptor when `VK_EXT_descriptor_indexing`/bindless layouts are in use, and — where the driver exposes `VK_KHR_buffer_device_address` — out-of-bounds pointer arithmetic on buffer device addresses obtained from `vkGetBufferDeviceAddress`. Buffer device address support specifically makes GPU-AV's *bounds checking of BDA pointer dereferences* possible; on a device without it, that specific check class is unavailable and the layer silently disables it while the descriptor-indexing checks continue to function.

The requirement list GPU-AV itself enforces, beyond Vulkan 1.1 as a baseline, includes one free descriptor set slot (GPU-AV reduces the value the application observes from `maxBoundDescriptorSets` by one, because it needs its own descriptor set to carry instrumentation state), the `fragmentStoresAndAtomics` and `vertexPipelineStoresAndAtomics` device features (so the instrumented shaders can write validation results back), and `timelineSemaphore` support to avoid forcing a `vkQueueWaitIdle` between every submission, which would otherwise be catastrophic for interactive frame rates. Other feature requirements exist for individual check classes; where a device lacks one, GPU-AV disables only the checks that depend on it rather than failing outright [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/gpu_validation.md).

Overhead is workload-dependent and not a fixed percentage: LunarG's own GPU-Assisted Validation white paper reported roughly a 10% frame-rate reduction on its reference test scene at the time of publication [Source](https://www.lunarg.com/news-insights/white-papers/vulkan-gpu-assisted-validation/), but the current developer documentation is explicit that a naive, fully-instrumented pass "can be painfully slow — like multiple seconds a frame slow" on shader-heavy content before selective instrumentation and other optimizations are applied [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/gpu_validation.md). In practice, GPU-AV is a tool for a targeted debugging session against a reduced repro case — a single frame or a handful of draws — rather than something left continuously enabled the way Synchronization Validation can be. Because it consumes a descriptor set slot and rewrites every shader module, it should be the *last* validation layer enabled in a debugging session, after core validation and Synchronization Validation have ruled out cheaper explanations.

> **Note: needs verification.** Whether current SDK releases document an updated overhead figure superseding the 2019-era ~10% white-paper number was not confirmed from a primary source at time of writing; treat the figure above as historical context rather than a current-release guarantee, and expect actual overhead to vary widely by shader complexity and descriptor-indexing density.

---

## 5. Debug Printf

Debug Printf reuses GPU-AV's SPIR-V instrumentation machinery for a different purpose: instead of inserting a bounds check, it lowers a shader-side `printf`-style call into instructions that write formatted text back to the host through the same channel GPU-AV uses to report validation failures, built on `VK_KHR_shader_non_semantic_info` (core in Vulkan 1.3) [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/debug_printf.md). In GLSL:

```glsl
// debug_printf_example.frag — shader-side diagnostic output
#extension GL_EXT_debug_printf : enable

void main() {
    float depth = gl_FragCoord.z;
    if (depth > 0.99) {
        debugPrintfEXT("Near-far clip: fragCoord=(%f,%f) depth=%f\n",
                        gl_FragCoord.x, gl_FragCoord.y, depth);
    }
}
```

`glslang` recognizes `GL_EXT_debug_printf` and emits the appropriate non-semantic instructions automatically; HLSL and Slang shaders use ordinary `printf(...)` calls that `dxc` and `slangc` lower the same way. Because it shares instrumentation infrastructure with GPU-AV, Debug Printf is subject to the same category of requirements (Vulkan 1.1+, a free descriptor slot, GPU memory for the output buffer) and the two are configured through the same layer-settings namespace, with dedicated switches — the layer setting `printf_verbose` controls whether output includes shader stage/invocation metadata alongside the formatted string, and `VK_LAYER_PRINTF_ONLY_PRESET=1` is a convenience environment variable that turns on Debug Printf while turning off every other validation category, which is the fastest way to isolate printf noise from unrelated validation warnings during a debugging session [Source](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/debug_printf.md). `VK_LAYER_PRINTF_ENABLE=1` instead layers Debug Printf on top of the layer's other active checks. Output messages carry `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT` severity and a fixed magic value (`0x4fe1fef9`) in `messageIdNumber`, which lets a debug messenger callback (§6) filter printf output from genuine validation errors programmatically rather than by string matching. Debug Printf requires validation layer version 1.2.135.0 or newer.

Because both GPU-AV's own checks and Debug Printf instrument SPIR-V through the same mechanism, running them simultaneously multiplies instrumentation overhead and is rarely useful — a targeted debugging session should typically enable one or the other rather than both, isolating a suspected out-of-bounds access with GPU-AV first and only reaching for Debug Printf once the failing invocation is narrowed down enough that manual print statements add value over the automated bounds-check report.

---

## 6. `VK_EXT_debug_utils`

`VK_EXT_debug_utils` is the extension that makes every other tool in this chapter legible. It is not, itself, a validation mechanism — it is an object-naming and event-labeling protocol that validation messages, RenderDoc's event list, and every vendor profiler in this chapter consume as their primary source of human-readable identifiers.

`vkSetDebugUtilsObjectNameEXT` attaches a UTF-8 name to any Vulkan handle — a `VkImage`, `VkBuffer`, `VkPipeline`, `VkCommandBuffer`, or any other dispatchable or non-dispatchable object — after creation. Without it, every validation message and every profiler resource list reports raw 64-bit handles, and correlating "descriptor set `0x7f3a2c001120` is missing a binding" back to a specific `VkDescriptorSet` in application code is a matter of print-debugging the handle value. With consistent naming discipline, the same message instead reads with the application's own resource name.

`vkCmdBeginDebugUtilsLabelEXT`/`vkCmdEndDebugUtilsLabelEXT` bracket a region of a command buffer with a named, colored label — a GPU-side breadcrumb visible in RenderDoc's event list as a collapsible group, in RGP's and Nsight's timelines as a named region, and, critically, in `VK_EXT_device_fault`'s post-hang diagnostic output (§7) as the last-known label active when the device stopped responding.

The messenger — created with `vkCreateDebugUtilsMessengerEXT` — is the delivery mechanism for every layer's output, filtered along two independent axes that together form a filter matrix:

| | `GENERAL` | `VALIDATION` | `PERFORMANCE` | `DEVICE_ADDRESS_BINDING`* |
|---|---|---|---|---|
| `VERBOSE` | loader/layer info | — | — | — |
| `INFO` | layer load/unload | Debug Printf output | — | binding lifecycle events |
| `WARNING` | — | spec-adjacent misuse | Best Practices warnings | — |
| `ERROR` | — | VU violations, sync hazards, GPU-AV failures | — | — |

*`DEVICE_ADDRESS_BINDING` is contributed by `VK_EXT_device_address_binding_report` (§7) rather than core `VK_EXT_debug_utils`, but shares the same messenger callback plumbing.

```c
// vk_debug_messenger_matrix.c — filtering by both axes independently
VkDebugUtilsMessengerCreateInfoEXT messenger_ci = {
    .sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT,
    .messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT
                      | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT,
    .messageType     = VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT
                      | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT,
    .pfnUserCallback = debug_callback,
};
```

A callback that only registers `ERROR`-severity `VALIDATION`-type messages will never see Debug Printf output (which is `INFO`-severity) or Best Practices warnings (which are `PERFORMANCE`-type, not `VALIDATION`-type) — a frequent source of "I enabled the validation layer and it's not printing anything" reports that trace back to an overly narrow filter rather than a broken layer. Chapter 24 §9 covers the basic messenger setup pattern for application bootstrap; this matrix is the reference for choosing filter bits deliberately once the basic wiring is in place.

---

## 7. `VK_EXT_device_fault`

Validation layers catch what the specification forbids; `VK_EXT_device_fault` is for the remaining case where the GPU stops responding anyway, and the application needs a starting point for postmortem analysis rather than a repro. It gives an application queryable diagnostic state after a `VK_ERROR_DEVICE_LOST` return, via `vkGetDeviceFaultInfoEXT` [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetDeviceFaultInfoEXT.html):

```c
// vk_device_fault_query.c — the two-call query pattern
VkDeviceFaultCountsEXT fault_counts = { .sType = VK_STRUCTURE_TYPE_DEVICE_FAULT_COUNTS_EXT };
vkGetDeviceFaultInfoEXT(device, &fault_counts, NULL);  // pFaultInfo == NULL: query counts only

std::vector<VkDeviceFaultAddressInfoEXT> address_infos(fault_counts.addressInfoCount);
std::vector<VkDeviceFaultVendorInfoEXT>  vendor_infos(fault_counts.vendorInfoCount);
std::vector<uint8_t> vendor_binary(fault_counts.vendorBinarySize);

VkDeviceFaultInfoEXT fault_info = {
    .sType          = VK_STRUCTURE_TYPE_DEVICE_FAULT_INFO_EXT,
    .pAddressInfos  = address_infos.data(),
    .pVendorInfos   = vendor_infos.data(),
    .pVendorBinaryData = vendor_binary.data(),
};
vkGetDeviceFaultInfoEXT(device, &fault_counts, &fault_info);
```

The two-call convention mirrors every other Vulkan enumeration API: an initial call with `pFaultInfo == NULL` returns `addressInfoCount`, `vendorInfoCount`, and `vendorBinarySize` in `VkDeviceFaultCountsEXT` so the application can allocate storage, and the second call fills it. `VkDeviceFaultInfoEXT::description` is a null-terminated, human-readable UTF-8 summary of the fault; `pAddressInfos` is an array of `VkDeviceFaultAddressInfoEXT` giving faulting GPU virtual addresses and the type of addressing error (out-of-bounds, unmapped, and so on); `pVendorInfos` is an array of `VkDeviceFaultVendorInfoEXT` giving a driver-defined fault code, description string, and vendor-specific fault data per reported fault; and `pVendorBinaryData` is an opaque, vendor-defined binary crash dump that requires that vendor's own tooling to interpret — the extension standardizes only the query mechanism and the common header of the binary blob, not its contents [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VkDeviceFaultInfoEXT.html).

`VkDeviceFaultVendorInfoEXT` is the layer worth reading closely on AMD hardware in particular: AMD's driver populates it with decoded UVM (Unified Virtual Memory) page-fault register state — the faulting virtual address's fault type (read/write/execute, page-not-present versus permission violation) as reported by the GPU's memory management unit — giving a concrete address and access-type pair to correlate against application-side allocation tracking, rather than only the generic description string.

`VK_EXT_device_fault` answers "what kind of fault, at what address" but not "which draw call, in which frame." Two complementary mechanisms close that gap:

- **Debug-label breadcrumbs.** The last `vkCmdBeginDebugUtilsLabelEXT` region active in each queue at the moment of the hang — recoverable because well-behaved applications write a GPU-visible marker on every label boundary, typically into a small host-visible buffer read back after `VK_ERROR_DEVICE_LOST` — narrows the fault from "somewhere in this frame" to "inside this named pass." This convention is not part of the `VK_EXT_debug_utils` specification itself; it is an application-level pattern layered on top of the labeling mechanism §6 describes, and vendor tools such as Nsight Aftermath (§11) formalize it further.
- **`VK_EXT_device_address_binding_report`.** This extension reports the binding and unbinding of GPU virtual address ranges to Vulkan objects as they occur, letting an application (or a tool observing the same debug messenger) maintain its own live map from VA range to object identity. Cross-referencing a `VkDeviceFaultAddressInfoEXT` address against that map after a fault distinguishes a use-after-free (the address falls inside a range that was reported bound and then unbound before the fault) from an out-of-bounds access (the address falls just outside the limits of a still-bound range) [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_device_address_binding_report.html).

Together, `VK_EXT_device_fault` plus debug-label breadcrumbs plus `VK_EXT_device_address_binding_report` form the minimum viable in-application GPU-crash diagnostic stack for a shipping title that cannot assume RenderDoc or a vendor profiler is attached — the pairing the outline for this chapter calls out explicitly, and the pattern every vendor's own crash-dump tooling (Nsight Aftermath, RGP's post-mortem mode) builds on rather than replaces.

---

## 8. RenderDoc Deep Dive

RenderDoc is the open-source, cross-vendor frame debugger for Vulkan, OpenGL, and Direct3D, with its Vulkan layer implementation rooted at `renderdoc/driver/vulkan/vk_core.cpp` in the upstream tree [Source](https://github.com/baldurk/renderdoc). Chapter 125 covers RenderDoc's Linux installation, packaging, and general UI in depth; this section focuses on the capture-triggering and inspection workflow specific to a Vulkan debugging session.

**Triggering a capture.** RenderDoc offers two independent capture-triggering mechanisms that solve different problems. The **overlay/UI path** — launching an application through `qrenderdoc` or injecting via `renderdoccmd inject` — is the default for interactive exploration: a screen overlay shows frame timing and a capture button, requiring no source changes. The **in-application API path** links the target directly against `renderdoc_app.h`'s function-pointer table, obtained via `dlsym` against a `RENDERDOC_GetAPI` symbol exposed by RenderDoc's injected library, and gives the application programmatic control over exactly when a capture starts and ends [Source](https://github.com/baldurk/renderdoc/blob/v1.x/renderdoc/api/app/renderdoc_app.h):

```c
// renderdoc_explicit_capture.c — explicit frame-boundary capture
#include "renderdoc_app.h"
#include <dlfcn.h>

RENDERDOC_API_1_6_0 *rdoc_api = NULL;

void init_renderdoc(void) {
    if (void *mod = dlopen("librenderdoc.so", RTLD_NOW | RTLD_NOLOAD)) {
        pRENDERDOC_GetAPI RENDERDOC_GetAPI =
            (pRENDERDOC_GetAPI)dlsym(mod, "RENDERDOC_GetAPI");
        RENDERDOC_GetAPI(eRENDERDOC_API_Version_1_6_0, (void **)&rdoc_api);
    }
}

void capture_one_frame(void) {
    if (rdoc_api) rdoc_api->StartFrameCapture(NULL, NULL);  // NULL,NULL = current device/window
    render_frame();
    if (rdoc_api) rdoc_api->EndFrameCapture(NULL, NULL);
}
```

`StartFrameCapture`/`EndFrameCapture` accept device and window handles, with `NULL` acting as a wildcard matching whichever device/window is currently active — the pattern to reach for when an application has exactly one swapchain, which covers most cases. `TriggerCapture` is a coarser alternative that captures whichever frame completes next without the application bracketing start/end itself. `SetCaptureFilePathTemplate` controls where `.rdc` files land and how they are named; `GetAPIVersion` lets the application confirm which API version RenderDoc actually granted, since RenderDoc may return a newer, backward-compatible version than the one requested [Source](https://github.com/baldurk/renderdoc/blob/v1.x/renderdoc/api/app/renderdoc_app.h). The explicit API is the right choice for automated capture-on-failure hooks — pairing a capture trigger with the validation-layer error callback from §6 so a captured `.rdc` is produced automatically the moment a validation error fires.

**Inspecting a capture.** Once loaded, the **event browser** lists every draw, dispatch, copy, and barrier in submission order, nested under any `VK_EXT_debug_utils` labels (§6) active at the time — which is the single biggest legibility win from consistent object naming and label discipline. Selecting an event exposes:

- The **resource inspector** — a live preview of any bound texture at its state *at that point in the frame*, a hex/structured view of buffer contents, and the full contents of every bound descriptor set, resolved back to the resource each descriptor points at.
- The **pipeline state inspector** — the complete fixed-function and shader-stage state for the selected draw: vertex input bindings, the compiled VS/FS (or other stage) SPIR-V disassembly and, where debug info was preserved, the original GLSL/HLSL source, and blend/depth/stencil state as actually bound, not as the application believes it bound.
- **GPU counter integration** — RenderDoc can pull hardware performance counters per-event where the underlying driver exposes `VK_KHR_performance_query` (§12) or a vendor-specific counter path; on AMD hardware this bridges into RGP-compatible counter sets, and on NVIDIA hardware into counters surfaced through the Nsight SDK integration, letting a single capture correlate a specific draw call against occupancy or bandwidth data without a second, separate profiling pass.

**Remote capture.** `renderdoccmd remoteserver` runs a headless capture/replay server on a machine that has no local display attached — a render-farm node or an SSH-only GPU box — and the desktop `qrenderdoc` UI on a developer's workstation connects to it over TCP/UDP ports 38920–38927, driving capture and replay on the remote GPU while showing the UI locally [Source](https://renderdoc.org/docs/how/how_network_capture_replay.html). This is the standard pattern for debugging a Vulkan application running inside a headless CI runner or a cloud GPU instance from a local machine with a display.

---

## 9. AMD Radeon GPU Profiler

The Radeon GPU Profiler (RGP), part of AMD's GPUOpen tools suite (`github.com/GPUOpen-Tools/radeon_gpu_profiler`), is a low-level, hardware-thread-trace-based profiler distinct from RenderDoc's API-level frame capture — RGP shows what the GPU's shader engines actually executed, cycle by cycle, rather than what commands the application recorded [Source](https://github.com/GPUOpen-Tools/radeon_gpu_profiler).

RGP's data source is **SQTT** (SQ Thread Trace — "SQ" being the shader processor's sequencer), a hardware trace facility built into RDNA and GCN silicon that records a timestamped instruction-level trace of wavefront execution directly from the shader engines during a captured frame. From an SQTT capture, RGP reconstructs:

- **A GPU timeline** of every pipeline stage's execution window, showing overlap (or its absence) between, for instance, an async compute queue's work and the graphics queue's work in the same frame.
- **Wavefront occupancy**, visualized per shader engine and colorable by originating draw/dispatch event, showing how many wavefronts were resident on each SIMD unit over time — the primary signal for whether a shader is register- or LDS-limited (low occupancy despite available work) versus genuinely compute-bound.
- **Barrier stall visualization**, which isolates time the GPU spent idle waiting on a pipeline barrier or queue synchronization point rather than executing shader work — the RGP-side complement to the Synchronization Validation checks in §3, which catch *missing* barriers on the CPU side but cannot show the *cost* of the correct ones that are present.

`VK_AMD_shader_info` supplies static, compile-time shader information — register usage (VGPR/SGPR counts), LDS usage, and ISA instruction counts per shader stage — queryable through `vkGetShaderInfoAMD` independent of any capture, useful for a fast register-pressure check before reaching for a full SQTT trace. Captures are saved in AMD's `.rgp` format, an offline, self-contained binary that the RGP desktop application loads for analysis without requiring the original GPU or driver session to still be running, which makes it practical to capture on a render-farm node and analyze on a workstation.

---

## 10. Intel Graphics Performance Analyzers

Intel Graphics Performance Analyzers (GPA) is Intel's frame- and hardware-level profiler for its Xe-family GPUs, covering the same Frame Analyzer / event-level GPU metrics role as RGP and Nsight Graphics on their respective vendors' hardware [Source](https://www.intel.com/content/www/us/en/developer/tools/graphics-performance-analyzers/overview.html). Its hardware-counter view centers on Xe's execution-unit (EU) architecture: **EU thread occupancy** measures how fully the available hardware threads across an Xe tile's EUs are populated with active work, and is read alongside stall metrics rather than in isolation — high occupancy (Intel's own guidance cites figures approaching 90% as "good") combined with elevated stall percentages (roughly 5–10% or higher) points at an L3 cache bottleneck rather than a genuinely compute-bound kernel, since the threads are resident but blocked on memory rather than retiring instructions [Source](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/gpu-compute-media-hotspots-analysis.html). **L3 cache hit rate** and **sampler throughput/busy-vs-bottleneck** metrics are surfaced the same way, per shader stage per event, in GPA's Frame Analyzer.

> **Note: needs verification.** Publicly available documentation for GPA's newest releases weights Windows workflows more heavily than Linux ones in the material available at time of writing, and a specific statement of Xe2 PMU counter access from Linux userspace (as opposed to the Windows driver stack) could not be confirmed from a primary Intel source. Readers targeting Xe2 hardware on Linux should confirm current GPA Linux support and PMU counter availability against Intel's release notes for the SDK version in use before relying on this section for that platform specifically.

GPA's Vulkan support tracks current Vulkan SDK releases (version 1.4.304.0 support is documented as of the 2024.4 release cycle) [Source](https://www.intel.com/content/www/us/en/developer/articles/release-notes/intel-gpa-current-release-notes.html), and its counter-capture path integrates with the same `VK_KHR_performance_query` mechanism (§12) that RenderDoc and RGP draw on, rather than a wholly separate proprietary counter API — the practical consequence being that a driver exposing accurate `VK_KHR_performance_query` counters benefits every tool in this chapter simultaneously, not just Intel's own.

---

## 11. NVIDIA Nsight Graphics

NVIDIA Nsight Graphics is NVIDIA's frame debugger and profiler for Vulkan, OpenGL, and Direct3D, documented at `docs.nvidia.com/nsight-graphics` [Source](https://docs.nvidia.com/nsight-graphics/UserGuide/index.html). Its Vulkan-specific capabilities most relevant to a debugging-and-profiling workflow are:

- **The shader debugger**, which supports source-line stepping through GLSL/SPIR-V shaders directly on RTX-capable hardware, using the same debug-info-preserving compilation path (`-g` with `glslc`/`dxc`, preserving `OpDebugLine`/`NonSemantic.Shader.DebugInfo.100` instructions) that RenderDoc's pipeline inspector relies on for source correlation (§8).
- **The range profiler**, which captures NVIDIA-specific hardware counter sets over an application-defined range of frames or draws rather than a single instant, useful for characterizing steady-state performance rather than one anomalous frame.
- **Memory access pattern heat maps**, visualizing spatial locality of memory traffic per shader invocation — a diagnostic for cache-unfriendly access patterns that counter aggregates alone do not make visible.
- **The Shader Profiler**, reporting per-ISA-instruction throughput and stall-reason breakdowns at the SASS instruction level, the NVIDIA-hardware analogue of RGP's wavefront occupancy view.

`VK_NV_device_diagnostics_config` is the extension applications enable to get GPU crash-dump data with NVIDIA-specific depth, configured through `VkDeviceDiagnosticsConfigFlagBitsNV` at device creation: `VK_DEVICE_DIAGNOSTICS_CONFIG_ENABLE_SHADER_ERROR_REPORTING_BIT_NV` puts the GPU into a mode that surfaces more runtime shader errors (useful for debugging corruption caused by in-shader faults such as invalid memory access from a shader), and `VK_DEVICE_DIAGNOSTICS_CONFIG_ENABLE_RESOURCE_TRACKING_BIT_NV` enables driver-level tracking of live and recently destroyed resources so a GPU page-fault address can be mapped back to the resource that produced it — the NVIDIA-specific analogue of `VK_EXT_device_address_binding_report` from §7 [Source](https://docs.nvidia.com/nsight-aftermath/SDK/index.html). With R615 and newer NVIDIA drivers, crash-relevant shader debug information is embedded directly into the resulting GPU crash dump, which Nsight Graphics and the standalone Nsight Aftermath decoder consume before falling back to external shader-debug-info lookups [Source: NVIDIA Nsight Aftermath SDK Guide](https://docs.nvidia.com/nsight-aftermath/SDK/index.html). On Linux, this diagnostic path is documented as always enabled without requiring a special build configuration, supported for D3D12 and Vulkan with driver R535 and newer, though visualizing the captured data still requires Nsight Graphics Pro [Source](https://docs.nvidia.com/nsight-aftermath/UserGuide/gpu-crash-dump-setting-up.html).

---

## 12. `VK_KHR_performance_query`

`VK_KHR_performance_query` is the vendor-neutral core mechanism that lets an application — or any of the profilers above — enumerate and capture hardware performance-monitoring-unit (PMU) counters through standard Vulkan calls rather than a vendor-proprietary API [Source](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_performance_query.html).

Counter discovery follows Vulkan's standard two-call enumeration pattern per queue family, since counter availability can differ by queue: `vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR` called with `pCounters == NULL` and `pCounterDescriptions == NULL` returns the counter count, and a second call with both arrays sized to that count fills in `VkPerformanceCounterKHR` (unit, scope, storage type, a UUID identifying the counter across driver versions) and `VkPerformanceCounterDescriptionKHR` (human-readable name and description) pairs. An application selects a subset of those counters by index and chains a `VkQueryPoolPerformanceCreateInfoKHR` onto `VkQueryPoolCreateInfo` when creating a query pool of the new `VK_QUERY_TYPE_PERFORMANCE_QUERY_KHR` type, then records the query around the command-buffer region of interest exactly as with any other Vulkan query type.

Counter *semantics* are irreducibly vendor-specific even though the enumeration mechanism is not: an AMD driver's counter set exposes SQTT-adjacent shader-engine metrics, an Intel driver's exposes Xe EU-occupancy and L3-hit-rate metrics matching what GPA's Frame Analyzer surfaces (§10), and an ARM Mali driver's counter set reflects tile-based deferred-rendering-specific metrics with no direct desktop-GPU analogue — there is no portable mapping from "counter index 3" on one vendor to any counter on another, and application code that wants to build a cross-vendor performance overlay must resolve counters by their descriptive metadata (name, unit, category) rather than assuming index stability. This is also the mechanism RenderDoc's GPU counter integration (§8) and both RGP's and GPA's own tooling build on where they draw counter data through the Vulkan API rather than through a lower-level vendor driver interface, making `VK_KHR_performance_query` support quality a direct lever on how good every downstream tool's counter data can be on a given driver.

---

## 13. `vkconfig` — the Vulkan Configurator

`vkconfig` is the GUI configuration tool shipped with the Vulkan SDK (source in `github.com/LunarG/VulkanTools`) for managing layers and their settings across every Vulkan application on a system without touching each application's own code. It exposes the same layer-enable/disable, layer-ordering, and per-layer settings surface described in §1 through a UI, and can apply a chosen configuration either to a single launched executable or globally, system-wide, to every Vulkan application until turned off [Source](https://vulkan.lunarg.com/doc/view/latest/linux/vkconfig.html).

On Linux, `vkconfig`'s persisted state lives under two directories in `$HOME/.local/share/vulkan/`: `settings.d/vk_layer_settings.txt` holds the per-layer settings values (the same file format an application or CI script can hand-author or generate directly, bypassing the GUI entirely), and `loader_settings.d/vk_loader_settings.json` tells the Vulkan loader itself which layers are enabled and in what order, independent of any per-application `VK_INSTANCE_LAYERS` environment variable [Source](https://github.com/KhronosGroup/Vulkan-Utility-Libraries/blob/main/docs/layer_configuration.md). A `vk_layer_settings.txt` placed next to an application's own executable takes precedence for that application specifically over the globally applied `settings.d` configuration, which is the mechanism for shipping a fixed, reproducible validation configuration alongside an automated test binary while leaving a developer's interactive `vkconfig` global settings untouched. `vkconfig`'s context menu can also export a chosen configuration as any of these formats — `vk_layer_settings.txt`, a shell script setting the equivalent environment variables, or a `VK_EXT_layer_settings` C++ snippet matching §1.2's pattern — which is the fastest path from "I found the right settings interactively in the GUI" to "I have the exact code to reproduce them."

---

## 14. A Debugging Workflow

The tools above compose into a workflow that should, in most cases, be worked through roughly in order of cost — cheapest and least invasive first — rather than reaching for the heaviest tool immediately:

1. **Layer activation.** Enable `VK_LAYER_KHRONOS_validation` with core validation, thread safety, stateless parameter checking (all default-on), and Best Practices (§2) for every development build. This layer of checking is cheap enough to run continuously and catches the largest fraction of bugs for the least effort.
2. **Messenger callback.** Wire a `VK_EXT_debug_utils` messenger (§6) with a filter matrix wide enough to see `PERFORMANCE`-type Best Practices output alongside `VALIDATION`-type errors, and name every long-lived resource with `vkSetDebugUtilsObjectNameEXT` from the start of a project rather than retrofitting it during a debugging session — the cost of doing so later is disproportionately higher because it means re-reading validation logs that are currently full of bare hex handles.
3. **Synchronization Validation on any pass-ordering change** (§3) — enabled for the duration of testing a new render-graph edge or barrier, given its low CPU-only cost relative to GPU-AV.
4. **RenderDoc frame capture** (§8) once a symptom is visually reproducible — corrupt geometry, a wrong texture, missing geometry — to inspect the actual bound state at the offending draw, which is very often sufficient to diagnose a logic bug (wrong descriptor bound, stale pipeline state, incorrect blend mode) without needing GPU-side instrumentation at all.
5. **The vendor GPU profiler matching the target hardware** (§§9–11) once the symptom is a performance regression rather than a correctness bug, or once RenderDoc's capture confirms the *state* is correct and the problem must therefore be in execution efficiency — occupancy, stalls, cache behavior — which only hardware-level tracing exposes.
6. **GPU-Assisted Validation and Debug Printf** (§§4–5), reserved for the specific case of suspected descriptor-indexing or buffer-device-address corruption that RenderDoc's static resource inspection cannot catch because the bad index or pointer is itself computed at runtime — this is the most expensive tool in the stack and should be scoped to a minimal repro before enabling.
7. **`VK_EXT_device_fault` on an actual hang** (§7), read against debug-label breadcrumbs and `VK_EXT_device_address_binding_report` state, when the device is lost outright and no capture tool was attached to catch the moment of failure — the fallback path for production or automated-test crashes rather than an interactive-session-first tool.

The recurring theme across every tool in this chapter is that they read the same substrate: `VK_EXT_debug_utils` names and labels make validation output, RenderDoc's event list, and every vendor profiler's timeline simultaneously legible, and `VK_KHR_performance_query` is the one counter-collection mechanism nearly every tool above eventually draws from. Investing early in consistent object naming and label discipline is not optional polish — it is the single change that makes every other tool in this chapter more useful at once.

---

## Integrations

- **Chapter 24 (Vulkan and EGL for Application Developers)** covers the baseline instance/device setup and the introductory validation-layer activation pattern (its §9) that this chapter assumes as a starting point and builds on with the full sub-layer, `VkLayerSettingsCreateInfoEXT`, and GPU-side-instrumentation detail Ch24 does not attempt.
- **Chapter 148 (Vulkan Synchronisation)** is the canonical reference for the availability/visibility memory model and every synchronization primitive; §3 of this chapter covers only how Synchronization Validation detects violations of that model, and should be read alongside Ch148 rather than as a substitute for it.
- **Chapter 157 (Vulkan Descriptor Binding in Depth)** covers bindless/`VK_EXT_descriptor_indexing` architecture in full; §4's description of GPU-Assisted Validation's descriptor-indexing bounds checks is the debugging-side counterpart to the binding architecture Ch157 documents.
- **Chapter 173 (`VK_EXT_shader_object`)** documents pipeline-free shader binding; its dynamic-state-heavy workflow changes what a captured RenderDoc frame's pipeline state inspector shows compared to classic `VkPipeline` objects, and the debugging workflow in §14 applies unchanged to shader-object-based renderers.
- **Chapter 125 (RenderDoc on Linux)** is the full reference for RenderDoc installation, packaging, UI tour, and the programmatic Python API; §8 here assumes that grounding and focuses specifically on the Vulkan capture-triggering and inspection workflow relevant to a debugging session.
- **Chapter 137 (GPU Performance Profiling Ecosystem on Linux)** surveys the broader Linux GPU profiling landscape — MangoHud, Mesa shader-db, Perfetto's GPU timeline, and the `perf` counter path — alongside the same vendor profilers §§9–11 cover in Vulkan-specific depth; readers wanting the tool landscape rather than the Vulkan debugging workflow should start there.
- **Chapter 30 (Debugging and Profiling)** covers the broader development-time toolchain — `gdb` integration, Mesa debug environment variables, and kernel-level GPU debugging via `dyndbg` — that complements the Vulkan-API-level tooling this chapter documents when a bug turns out to live below the API boundary, in the driver or kernel itself.

---

## References

- [`VK_LAYER_KHRONOS_validation` layer documentation](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/khronos_validation_layer.md) — activation methods, `VkLayerSettingsCreateInfoEXT` usage (§1)
- [Vulkan-ValidationLayers `best_practices.md`](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/best_practices.md) — Best Practices sub-layer scope and vendor-specific check filters (§2)
- ["Vulkan's Best Practice layer now has AMD-specific checks" — AMD GPUOpen](https://gpuopen.com/learn/vulkan-best-practice-layer/) — `VALIDATION_CHECK_ENABLE_VENDOR_SPECIFIC_AMD` activation (§2)
- [Vulkan-ValidationLayers `gpu_validation.md`](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/gpu_validation.md) — GPU-Assisted Validation instrumentation mechanism, requirements, overhead caveats (§4)
- [LunarG Vulkan GPU-Assisted Validation white paper](https://www.lunarg.com/news-insights/white-papers/vulkan-gpu-assisted-validation/) — historical ~10% overhead measurement (§4)
- [Vulkan-ValidationLayers `debug_printf.md`](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/debug_printf.md) — Debug Printf mechanism, GLSL/HLSL usage, activation variables (§5)
- [Vulkan Specification — Synchronization](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#synchronization) — availability/visibility model referenced from Ch148 (§3)
- [`vkGetDeviceFaultInfoEXT` man page](https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetDeviceFaultInfoEXT.html) — two-call query pattern (§7)
- [`VkDeviceFaultInfoEXT` man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VkDeviceFaultInfoEXT.html) — structure fields, vendor binary blob semantics (§7)
- [`VK_EXT_device_address_binding_report` man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_device_address_binding_report.html) — VA binding lifecycle reporting for crash postmortem (§7)
- [RenderDoc source repository, `renderdoc/driver/vulkan/vk_core.cpp`](https://github.com/baldurk/renderdoc) — Vulkan layer implementation (§8)
- [RenderDoc in-application API header, `renderdoc_app.h`](https://github.com/baldurk/renderdoc/blob/v1.x/renderdoc/api/app/renderdoc_app.h) — `RENDERDOC_GetAPI`, `StartFrameCapture`/`EndFrameCapture`, `TriggerCapture` (§8)
- [RenderDoc documentation — network capture and replay](https://renderdoc.org/docs/how/how_network_capture_replay.html) — `renderdoccmd remoteserver` usage and port range (§8)
- [GPUOpen-Tools `radeon_gpu_profiler` repository](https://github.com/GPUOpen-Tools/radeon_gpu_profiler) — RGP overview and release notes (§9)
- ["New Work Graphs sample and RGP support for GPU Work Graphs" — AMD GPUOpen](https://gpuopen.com/learn/rgp-work-graphs/) — wavefront occupancy UI, VK_AMDX_shader_enqueue event naming (§9)
- [AMD Radeon GPU Profiler manual — GPUOpen](https://gpuopen.com/manuals/rgp_manual/) — SQTT-based capture and analysis views (§9)
- [Intel Graphics Performance Analyzers overview](https://www.intel.com/content/www/us/en/developer/tools/graphics-performance-analyzers/overview.html) — product scope (§10)
- [Intel GPA release notes (2024.4)](https://www.intel.com/content/www/us/en/developer/articles/release-notes/intel-gpa-current-release-notes.html) — Vulkan SDK version support (§10)
- [Intel VTune Profiler GPU Compute/Media Hotspots Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/gpu-compute-media-hotspots-analysis.html) — EU occupancy vs. stall interpretation guidance (§10)
- [NVIDIA Nsight Graphics User Guide](https://docs.nvidia.com/nsight-graphics/UserGuide/index.html) — shader debugger, range profiler, memory heat maps, Shader Profiler (§11)
- [NVIDIA Nsight Aftermath SDK Guide](https://docs.nvidia.com/nsight-aftermath/SDK/index.html) — `VK_NV_device_diagnostics_config` flags, embedded shader debug info in crash dumps (§11)
- [NVIDIA Nsight Aftermath — GPU Crash Dump Setup](https://docs.nvidia.com/nsight-aftermath/UserGuide/gpu-crash-dump-setting-up.html) — Linux support status, driver version requirement (§11)
- [`VK_KHR_performance_query` man page](https://registry.khronos.org/vulkan/specs/latest/man/html/VK_KHR_performance_query.html) — extension overview (§12)
- [`vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR` man page](https://registry.khronos.org/vulkan/specs/latest/man/html/vkEnumeratePhysicalDeviceQueueFamilyPerformanceQueryCountersKHR.html) — two-call counter enumeration (§12)
- [Vulkan Configurator (`vkconfig`) documentation — LunarG](https://vulkan.lunarg.com/doc/view/latest/linux/vkconfig.html) — GUI scope, global vs. per-application configuration (§13)
- [Vulkan-Utility-Libraries `layer_configuration.md`](https://github.com/KhronosGroup/Vulkan-Utility-Libraries/blob/main/docs/layer_configuration.md) — `settings.d`/`loader_settings.d` file layout on Linux (§13)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
