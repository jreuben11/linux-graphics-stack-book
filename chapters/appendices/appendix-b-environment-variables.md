# Appendix B: Environment Variables Reference

**Scope**: This appendix targets all three book audiences — systems and driver developers, graphics application developers, and browser and web platform engineers. It serves as a single authoritative lookup table for every environment variable, kernel module parameter, and sysfs debug knob referenced throughout the book. Readers should treat this appendix as a companion to the main text: the chapters provide the architectural context explaining *why* a variable exists; this appendix tells you *exactly* what to type and what to expect. Production-safety annotations are included throughout so that tuning variables useful in development environments are never accidentally shipped in production builds.

---

## Table of Contents

- [B.1 Mesa OpenGL / libGL Variables](#b1-mesa-opengl--libgl-variables)
- [B.2 Mesa Vulkan / Vulkan Loader Variables](#b2-mesa-vulkan--vulkan-loader-variables)
- [B.3 Mesa Shader Cache Variables](#b3-mesa-shader-cache-variables)
- [B.4 NIR / GLSL Compiler Debug Variables](#b4-nir--glsl-compiler-debug-variables)
- [B.5 RADV-Specific Variables](#b5-radv-specific-variables)
  - [B.5a RADV_DEBUG Flags](#b5a-radv_debug-flags)
  - [B.5b RADV_PERFTEST Flags](#b5b-radv_perftest-flags)
  - [B.5c ACO_DEBUG Flags](#b5c-aco_debug-flags)
  - [B.5d AMDGPU Userspace Debug](#b5d-amdgpu-userspace-debug)
- [B.6 Intel ANV / i915 Variables](#b6-intel-anv--i915-variables)
- [B.7 Vulkan Validation Layer Variables](#b7-vulkan-validation-layer-variables)
- [B.8 DXVK Variables](#b8-dxvk-variables)
- [B.9 VKD3D-Proton Variables](#b9-vkd3d-proton-variables)
- [B.10 Wine / Proton Variables](#b10-wine--proton-variables)
- [B.11 GStreamer / VA-API Variables](#b11-gstreamer--va-api-variables)
- [B.12 EGL Variables](#b12-egl-variables)
- [B.13 DRM / Kernel Module Parameters (sysfs)](#b13-drm--kernel-module-parameters-sysfs)
  - [B.13a DRM Core Module Parameters](#b13a-drm-core-module-parameters)
  - [B.13b AMDGPU Module Parameters](#b13b-amdgpu-module-parameters)
  - [B.13c i915 Module Parameters](#b13c-i915-module-parameters)
  - [B.13d DRM Memory Manager Debug](#b13d-drm-memory-manager-debug)
- [B.14 Variable Stacking and Interaction](#b14-variable-stacking-and-interaction)
- [B.15 Alphabetical Index](#b15-alphabetical-index)
- [References](#references)

---

## How to Use This Appendix

Each section covers one subsystem layer, ordered roughly from kernel to userspace to match the book's chapter flow. Tables use the following columns:

- **Variable / Parameter**: the exact string as used in a shell export or sysfs path.
- **Default**: the value when unset; `(unset)` means the feature is off unless the variable is set.
- **Effect**: what setting the variable does, in one to three sentences. Flag-valued variables have an expanded sub-table immediately below.
- **Introduced**: the Mesa, kernel, loader, or tool version where the variable first appeared. Entries predating reliable tracking are marked `(pre-Mesa 7.x)` or similar.
- **Debug-only?**: `Yes` means do not set in production builds — performance regressions or visual corruption may result. `No` means safe in production (e.g., a path override). `Conditional` means it depends on the specific flag value.
- **Chapter**: cross-reference to the primary chapter where the subsystem is documented in depth.

> **Warning**: Variables marked **Debug-only: Yes** can cause severe performance regressions, shader miscompiles that appear as rendering corruption, or GPU hangs when left set across system restarts. Never export them in shell startup files (`~/.profile`, `/etc/environment`) without understanding the consequence. The most dangerous offenders are `NIR_DEBUG=print` (disk I/O per shader per pass, easily gigabytes of output), `RADV_DEBUG=shaders` (unbounded stderr output), and `DXVK_LOG_LEVEL=debug` (per-draw-call logging at tens of thousands of lines per frame).

---

## B.1 Mesa OpenGL / libGL Variables

These variables control Mesa's OpenGL and libGL loader behaviour. They are parsed in `src/loader/loader.c` and `src/mesa/main/debug.c` in the Mesa source tree. [Source: Mesa GitLab, `src/loader/loader.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/loader/loader.c)

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `MESA_DEBUG` | (unset) | Enables Mesa internal error and debug output to stderr. Accepts a comma-separated list of flags: `silent` (suppress error messages), `flush` (flush after every draw call), `incomplete_tex` (warn on incomplete textures), `incomplete_fbo` (warn on incomplete framebuffer objects), `context` (warn on invalid context operations). Without any flags but with the variable set, all categories are printed. | Mesa 7.x | Yes | Ch. 14 |
| `LIBGL_DEBUG` | (unset) | Controls libGL loader verbosity. Set to `verbose` to print driver loading decisions including which `*_dri.so` module was selected and why. Set to `quiet` to suppress all loader warnings. Useful when diagnosing "no hardware acceleration" failures on first boot. | Mesa 3.x | Yes | Ch. 13 |
| `LIBGL_DRIVERS_PATH` | `/usr/lib/dri` (distro-dependent) | Colon-separated list of directories searched for `*_dri.so` DRI driver modules. Overrides the compiled-in default. Useful in multi-installation development environments where an in-tree Mesa build needs to take precedence over the system Mesa. | Mesa 6.x | No | Ch. 13 |
| `MESA_LOADER_DRIVER_OVERRIDE` | (unset) | Forces loading of a specific DRI driver by name (e.g., `radeonsi`, `iris`, `zink`) regardless of PCI ID detection. Most commonly used to force the Zink OpenGL-over-Vulkan driver: `MESA_LOADER_DRIVER_OVERRIDE=zink`. | Mesa 17.x | No (production use for Zink override) | Ch. 13 |
| `DRI_PRIME` | (unset) | Selects the GPU for DRI-based OpenGL rendering on multi-GPU systems. `DRI_PRIME=1` selects the non-default (typically discrete) GPU. Can also accept a PCI address in the form `pci-0000_01_00_0` for explicit selection. Applies only to DRI-based Mesa OpenGL; Vulkan uses separate ICD selection (see B.2). | Mesa 9.1 | No | Ch. 13 |
| `MESA_EXTENSION_OVERRIDE` | (unset) | Space-separated list of OpenGL extension names to force-enable (prefix `+`) or force-disable (prefix `-`). Example: `MESA_EXTENSION_OVERRIDE="+GL_EXT_texture_compression_s3tc -GL_ARB_compute_shader"`. Overrides `drirc` configuration. | Mesa 9.x | Yes | Ch. 14 |
| `DRIRC_DISABLE` | (unset) | Set to any non-empty value to disable all `~/.drirc` and `/etc/drirc` XML per-application workaround processing. Useful when testing whether a drirc entry is causing a regression. | Mesa 6.x | Yes | Ch. 14 |
| `MESA_GL_VERSION_OVERRIDE` | (unset) | Override the OpenGL version string reported to applications. Format: `MAJOR.MINOR` optionally followed by `COMPAT` for compatibility profile, e.g., `4.6COMPAT`. Mesa does not guarantee full implementation of the overridden version; use only to test application code paths. | Mesa 9.x | Yes | Ch. 14 |
| `MESA_GLSL_VERSION_OVERRIDE` | (unset) | Override the GLSL version string reported to applications, e.g., `460`. Can enable extension code paths the driver has not fully validated. | Mesa 9.x | Yes (may cause driver crashes) | Ch. 14 |
| `MESA_GLES_VERSION_OVERRIDE` | (unset) | Override the OpenGL ES version string reported to applications. Same caveats as `MESA_GL_VERSION_OVERRIDE`. | Mesa 9.x | Yes | Ch. 14 |
| `MESA_VK_VERSION_OVERRIDE` | (unset) | Override the Vulkan instance version reported by the driver. Format: `MAJOR.MINOR.PATCH`. Cannot exceed the driver's actual supported instance version. | Mesa 21.x | Yes | Ch. 18 |

> **Note**: `LIBGL_DRIVERS_PATH` and `MESA_LOADER_DRIVER_OVERRIDE` interact. If both are set, `MESA_LOADER_DRIVER_OVERRIDE` specifies the driver *name* to look for, and `LIBGL_DRIVERS_PATH` specifies *where* to search for it. Setting only `MESA_LOADER_DRIVER_OVERRIDE=zink` with the default search path requires that `zink_dri.so` is installed in `/usr/lib/dri`.

---

## B.2 Mesa Vulkan / Vulkan Loader Variables

These variables are processed by the Khronos Vulkan loader (`Vulkan-Loader` repository) before Mesa or any ICD sees the application call. The canonical documentation lives in [`docs/LoaderEnvironmentVariables.md`](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderEnvironmentVariables.md) in the Khronos loader repository. Mesa Vulkan driver-specific variables (RADV, ANV, NVK) appear in B.5 and B.6.

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `VK_ICD_FILENAMES` | (unset) | Colon-separated list of explicit ICD JSON manifest file paths to load, completely overriding default discovery from `/usr/share/vulkan/icd.d/` and `/etc/vulkan/icd.d/`. Use absolute paths; relative paths are resolved from the loader's working directory in ways that may surprise. **Ignored when running with elevated privileges** for security reasons. Deprecated in favour of `VK_DRIVER_FILES` (loader 1.3.207+) but still functional. | Vulkan 1.0 loader | No (useful in dev installs) | Ch. 18 |
| `VK_DRIVER_FILES` | (unset) | Alias for `VK_ICD_FILENAMES` introduced in loader 1.3.207 to match naming conventions. If both `VK_DRIVER_FILES` and `VK_ICD_FILENAMES` are set, `VK_DRIVER_FILES` takes precedence. Colon-separated list of full paths to ICD JSON manifest files or directories containing them. | Loader 1.3.207 | No | Ch. 18 |
| `VK_ADD_DRIVER_FILES` | (unset) | Colon-separated list of ICD JSON paths to *add* to the default discovery set, without replacing it. Useful for testing a new ICD alongside system-installed ones. Also ignored with elevated privileges. | Loader 1.3.207 | No | Ch. 18 |
| `VK_LAYER_PATH` | (unset) | Colon-separated directories searched for implicit and explicit layer JSON manifests. Supplements (does not replace) the default search path. | Vulkan 1.0 loader | No | Ch. 18 |
| `VK_INSTANCE_LAYERS` | (unset) | Colon-separated list of instance layer names to activate. For example, `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` enables the Khronos validation layer for all Vulkan applications in the environment. Activates layers as if they were requested by the application in `vkCreateInstance`. | Vulkan 1.0 loader | Yes | Ch. 18 |
| `VK_DEVICE_LAYERS` | (unset) | Deprecated device-level layer activation. Retained for compatibility with Vulkan 1.0-era tooling; device layers were removed from the Vulkan specification. Prefer `VK_INSTANCE_LAYERS`. | Vulkan 1.0 loader | Yes | Ch. 18 |
| `VK_LOADER_DEBUG` | (unset) | Vulkan loader debug verbosity. Comma-separated flags: `error`, `warn`, `info`, `debug`, `perf`, `all`. Logs ICD/layer discovery, manifest parsing, and function interception decisions to stderr. Using `warn` alongside `LD_BIND_NOW=1` exposes dynamic linker issues at startup. | Vulkan 1.0 loader | Yes | Ch. 18 |
| `VK_LOADER_DRIVERS_DISABLE` | (unset) | Comma-separated ICD driver names (as they appear in JSON manifest `"library_path"` basenames) to suppress at load time. Useful for forcing a specific GPU when multiple ICDs are installed. | Loader 1.3.x | Yes | Ch. 18 |
| `VK_LOADER_LAYERS_DISABLE` | (unset) | Comma-separated layer names to prevent from loading even if listed in implicit layer manifest files. Use to suppress unwanted implicit layers (e.g., Steam overlay) during testing. | Loader 1.3.x | Yes | Ch. 18 |
| `DISABLE_LAYER_AMD_SWITCHABLE_GRAPHICS_1` | (unset) | Set to `1` to disable AMD's implicit switchable graphics layer, which is sometimes installed by AMD driver packages and causes issues on unsupported configurations. | AMD driver | Yes | Ch. 18 |

> **Note**: All `VK_ICD_FILENAMES`, `VK_DRIVER_FILES`, and `VK_ADD_DRIVER_FILES` are silently ignored when the application is launched with elevated privileges (`setuid`, `sudo`). This is a deliberate security boundary to prevent privilege escalation through malicious ICD injection. If you need Vulkan debugging as root, place ICD manifests in the system directories instead.

---

## B.3 Mesa Shader Cache Variables

Mesa's on-disk shader cache stores compiled GPU binaries keyed by a hash of the shader source plus driver version plus hardware identifier. The implementation lives in `src/util/disk_cache.c` in the Mesa tree. [Source: Mesa `src/util/disk_cache.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/util/disk_cache.c)

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `MESA_SHADER_CACHE_DISABLE` | `false` | Set to `true` or `1` to disable the on-disk shader cache entirely for all Mesa drivers. Causes every shader program to be compiled from scratch at application startup. Useful for isolating cache corruption bugs or testing cache-miss compile times. Note: even when set, the EGL Android blob cache API (`EGL_ANDROID_blob_cache`) remains active if the application uses it. | Mesa 17.x | Yes | Ch. 16 |
| `MESA_SHADER_CACHE_DIR` | `$XDG_CACHE_HOME/mesa_shader_cache` or `~/.cache/mesa_shader_cache` | Override the root directory for Mesa's on-disk shader cache. Mesa creates per-architecture subdirectories within this root. Useful for placing the cache on a faster storage device or a shared NFS mount. | Mesa 17.x | No | Ch. 16 |
| `MESA_SHADER_CACHE_MAX_SIZE` | `1G` | Maximum total size of the shader cache directory. Accepts integer suffixes: `K` (kilobytes), `M` (megabytes), `G` (gigabytes). Eviction is LRU. Set to a larger value for workstations compiling many distinct shader permutations. | Mesa 17.x | No | Ch. 16 |
| `MESA_SHADER_CACHE_SHOW_STATS` | (unset) | Set to `true` to track and print hit/miss statistics for the shader cache when the application terminates. Useful for measuring how effective the cache is for a given workload. | Mesa 21.x | Yes | Ch. 16 |
| `MESA_GLSL_CACHE_DISABLE` | `false` | Older alias for `MESA_SHADER_CACHE_DISABLE`, deprecated since Mesa 20.2. Retained for backward compatibility. New deployments should use `MESA_SHADER_CACHE_DISABLE`. | Mesa 13.x | Yes | Ch. 16 |

> **Note**: Separate cache subdirectories are created per hardware architecture (e.g., one for GFX10, another for GFX11 AMD GPUs, another for Intel Xe), so `MESA_SHADER_CACHE_DIR` pointing to a shared location across multiple machines with different GPU generations is safe — there will be no cross-contamination of incompatible binaries.

---

## B.4 NIR / GLSL Compiler Debug Variables

These variables control Mesa's compiler pipeline. NIR (the NVIDIA IR, now the common Mesa IR) passes are logged through `src/compiler/nir/nir.c`. [Source: Mesa `src/compiler/nir/nir.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/compiler/nir/nir.c)

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `NIR_DEBUG` | (unset) | Comma-separated NIR pass debug flags. Set to `help` to print all available options. Key flags: `print` (dump full NIR after every pass to stderr — extremely verbose), `print_vs`/`print_fs`/`print_cs`/`print_gs`/`print_tcs`/`print_tes` (restrict `print` to one shader stage), `validate` (run NIR validator after every pass — catches IR invariant violations at the pass boundary), `tgsi` (print TGSI during TGSI-to-NIR translation), `print_internal` (include internal helper shaders in `print` output). | Mesa 13.x | Yes | Ch. 15 |
| `MESA_SPIRV_DUMP_PATH` | (unset) | Directory path into which Mesa will dump raw SPIR-V modules before compilation. Each SPIR-V blob is written as a `.spv` file named by a hash. Inspect dumps with `spirv-dis` (from SPIRV-Tools). Useful for isolating Vulkan shader compilation bugs without modifying the application. | Mesa 20.x | Yes | Ch. 15 |

> **Warning**: `NIR_DEBUG=print` generates one full shader dump to stderr after each NIR optimisation pass. A single complex Vulkan application at startup may execute hundreds of shaders through dozens of passes each, producing many gigabytes of output. Use stage-specific flags (`NIR_DEBUG=print_fs`) and redirect to a file: `NIR_DEBUG=print_fs some-app 2>nir_dump.txt`.

### GLSL Version Override Variables

The following variables override the version strings Mesa reports to applications at the GLSL/OpenGL level. They are parsed in `src/mesa/main/version.c`.

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `MESA_GL_VERSION_OVERRIDE` | (unset) | Override the OpenGL version string (e.g., `4.6COMPAT`). Suffix `COMPAT` enables compatibility profile. Mesa makes no guarantee that all features of the overridden version are correctly implemented; may cause crashes in applications that exercise unimplemented paths. | Mesa 9.x | Yes | Ch. 14 |
| `MESA_GLSL_VERSION_OVERRIDE` | (unset) | Override the GLSL version string (e.g., `460`). Can enable shader code paths the driver has not validated. | Mesa 9.x | Yes | Ch. 14 |
| `MESA_GLES_VERSION_OVERRIDE` | (unset) | Override the OpenGL ES version string reported to applications. | Mesa 9.x | Yes | Ch. 14 |

---

## B.5 RADV-Specific Variables

RADV is Mesa's AMD Radeon Vulkan driver, first shipped in Mesa 13.0. Its debug infrastructure is concentrated in `src/amd/vulkan/radv_debug.c`; the dedicated debug subsystem (`RADV_DEBUG`, `RADV_PERFTEST`) was substantially expanded around Mesa 17.x. [Source: Mesa `src/amd/vulkan/radv_debug.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_debug.c)

### B.5a RADV_DEBUG Flags

`RADV_DEBUG` accepts a comma-separated list of flag tokens. Parsing occurs at device creation time; changes take effect only on the next application launch.

| Variable | Default | Effect | Debug-only? | Chapter |
|---|---|---|---|---|
| `RADV_DEBUG` | (unset) | Activates one or more of the flags below. | Conditional | Ch. 20 |

**RADV_DEBUG flag sub-table:**

| Flag | Effect | Debug-only? |
|---|---|---|
| `info` | Print GPU/driver information (ASIC name, VRAM size, compute unit count) at device creation. | Yes |
| `startup` | Extra startup logging including queue creation and WSI surface details. | Yes |
| `shaders` | Dump compiled shaders (final ISA assembly plus NIR) to stderr at pipeline creation. Can produce hundreds of megabytes of output for shader-heavy titles. | Yes |
| `metashaders` | Dump RADV's internal meta-operation shaders (blits, clears, resolves). | Yes |
| `nomemorycache` | Disable the in-memory pipeline cache. Forces every pipeline to be recompiled from NIR. | Yes |
| `nofc` | Disable DCC fast-clear optimization. | Yes |
| `nodcc` | Disable Delta Color Compression. Production workaround for DCC-related corruption bugs. | No (workaround) |
| `nocmask` | Disable color mask compression (CMASK). | No |
| `nofmask` | Disable FMASK MSAA compression. | No |
| `nohtile` | Disable HiZ/HiS (hierarchical depth/stencil tile) acceleration. | No |
| `noibvbo` | Disable indirect buffer usage for vertex buffer objects. | No |
| `nobatchchain` | Disable command buffer chaining optimization. | No |
| `shaderstats` | Print shader SGPR/VGPR register usage statistics per pipeline. | Yes |
| `pipeline` | Log pipeline creation and cache lookup events. | Yes |
| `gpl` | Log Graphics Pipeline Library (GPL / `VK_EXT_graphics_pipeline_library`) activity including fast-link decisions. | Yes |
| `hang` | Enable GPU hang detection. RADV waits on submission fences with a timeout and reports the offending command buffer on hang. | Yes |
| `img` | Log image creation, format selection, and layout transition events. | Yes |
| `syncshaders` | Force synchronous shader compilation — disables background compile threads. Eliminates shader compile stutter for profiling at the cost of longer pipeline creation stalls. | No (causes stutter) |
| `preoptir` | Dump NIR before the optimisation pipeline runs (pre-ACO). | Yes |
| `nodynamicds` | Disable dynamic depth-stencil state support. Workaround for driver bugs with dynamic state paths. | No |
| `nggc` | Disable NGG (Next-Generation Geometry) culling on GFX10+. | No (workaround) |
| `skipcleanup` | Skip end-of-frame cleanup operations; useful for measuring raw GPU throughput without cleanup overhead. | Yes |
| `allentrypoints` | Expose all Vulkan entry points regardless of extension enablement; useful for loader testing. | Yes |
| `video` | Log video decode and encode operation submissions. | Yes |
| `nort` | Disable ray tracing support even on hardware with RT cores. | No |
| `checkir` | Run the ACO IR validator after compilation. More thorough than `ACO_DEBUG=validateir` at the cost of compile time. | Yes |
| `spirv` | Dump SPIR-V modules to the `MESA_SPIRV_DUMP_PATH` directory at pipeline creation. Requires `MESA_SPIRV_DUMP_PATH` to be set to an existing writable directory. | Yes |
| `nir` | Dump NIR for RADV's pipeline shaders (similar to `NIR_DEBUG=print` but scoped to RADV). | Yes |

### B.5b RADV_PERFTEST Flags

`RADV_PERFTEST` enables experimental performance features that are either not yet stable enough for unconditional enablement or are being tested before becoming the default. These are production-safe when noted but may be removed as features graduate to default.

| Variable | Default | Effect | Production-safe? | Chapter |
|---|---|---|---|---|
| `RADV_PERFTEST` | (unset) | Activates one or more of the flags below. | Conditional | Ch. 20 |

**RADV_PERFTEST flag sub-table:**

| Flag | Effect | Production-safe? |
|---|---|---|
| `gpl` | Enable Graphics Pipeline Library (`VK_EXT_graphics_pipeline_library`) fast-link path. Reduces pipeline stutter in games using DXVK or VKD3D-Proton. Has graduated to default on most GFX10+ hardware in Mesa 24.x. | Yes (when stable for your hardware) |
| `nggc` | Enable NGG primitive culling on GFX10+ hardware. Reduces GPU work for occluded geometry. May be default on newer hardware in recent Mesa versions. | Yes |
| `emulate_rt` | Emulate ray tracing on hardware without RT acceleration units (pre-RDNA2). Uses shader-based BVH traversal. Very slow compared to hardware RT; intended for compatibility testing. | No |
| `rt` | Enable Vulkan ray tracing extensions on RDNA2+ (older toggle before RT was unconditionally exposed). Mostly superseded on current Mesa. | Yes |
| `video_decode` | Enable experimental hardware video decode. | Conditional |
| `dmashaders` | Use the DMA queue for shader upload operations. Experimental throughput optimisation. | Experimental |
| `transfer_queue` | Enable async transfer queue for DMA operations (GFX11+ / RDNA3+). Improves upload bandwidth for streaming workloads. | Yes (GFX11+) |
| `ext_ms_image` | Enable extended multisampled image support (`VK_EXT_multisampled_render_to_single_sampled`). | Experimental |
| `pswave32` | Force pixel shader compilation in Wave32 mode (32 invocations per wavefront) instead of the default Wave64. Can improve occupancy for certain workloads. | Yes (GFX10+) |
| `gewave32` | Force geometry shader compilation in Wave32 mode. | Yes (GFX10+) |
| `cswave32` | Force compute shader compilation in Wave32 mode. | Yes (GFX10+) |

### B.5c ACO_DEBUG Flags

ACO is RADV's native shader compiler backend, replacing LLVM for most shader stages. Its debug infrastructure is in `src/amd/compiler/aco_debug.cpp`. [Source: Mesa `src/amd/compiler/aco_debug.cpp`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/aco_debug.cpp)

| Variable | Default | Effect | Debug-only? | Chapter |
|---|---|---|---|---|
| `ACO_DEBUG` | (unset) | Comma-separated ACO compiler debug flags. Key flags below. | Yes | Ch. 15 |

**ACO_DEBUG flag sub-table:**

| Flag | Effect | Debug-only? |
|---|---|---|
| `validateir` | Run the ACO IR validator after every compiler pass. Catches IR invariant violations early, making bisection of miscompiles easier. Significant compile-time overhead. | Yes |
| `validatera` | Run the register allocator validator to detect invalid register assignments. Complements `validateir` by checking the output of register allocation specifically. | Yes |
| `perfwarnings` | Emit performance warning annotations when the compiler detects suboptimal patterns (e.g., unnecessary wait states, poor instruction scheduling). | Yes |
| `force-waitcnt` | Insert `s_waitcnt vmcnt(0) lgkmcnt(0)` after every instruction. Makes memory hazards immediately visible at the cost of rendering the output completely unoptimized. | Yes |
| `novn` | Disable value numbering (CSE at the ACO IR level). Useful for isolating value numbering bugs. | Yes |
| `noopt` | Disable all ACO optimizations. Produces correct but unoptimized code. | Yes |
| `nosched` | Disable ACO's instruction scheduler. Useful for isolating scheduling bugs vs. correctness bugs. | Yes |

### B.5d AMDGPU Userspace Debug

`AMDGPU_DEBUG` controls debug logging in the `libdrm_amdgpu` userspace library layer, which sits between Mesa/RADV and the kernel DRM driver. Source: `amdgpu/amdgpu_device.c` in the libdrm repository. [Source: libdrm `amdgpu/amdgpu_device.c`](https://gitlab.freedesktop.org/mesa/drm/-/blob/main/amdgpu/amdgpu_device.c)

| Variable | Default | Effect | Debug-only? | Chapter |
|---|---|---|---|---|
| `AMDGPU_DEBUG` | (unset) | Comma-separated libdrm amdgpu debug flags: `info` (ASIC info at open), `bo` (buffer object allocation and import/export), `cs` (command submission packets), `vm` (VM bind/unbind and page table operations), `all` (all of the above). | Yes | Ch. 7 |

---

## B.6 Intel ANV / i915 Variables

ANV is Mesa's Intel Vulkan driver. It shares debug infrastructure with the Intel OpenGL drivers (i965, iris) through `src/intel/dev/intel_debug.c`. [Source: Mesa `src/intel/dev/intel_debug.c`](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/intel/dev/intel_debug.c)

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `INTEL_DEBUG` | (unset) | Comma-separated flags controlling debug output for all Intel Mesa drivers (iris, crocus, i965, ANV). Set to `help` to list all available flags. Key flags: `bat` (batch buffer dumps to `/tmp/gen*.batch`), `buf` (buffer object allocations), `color` (ANSI color in output), `nohiz` (disable hierarchical depth), `noccs` (disable CCS color compression), `noaux` (disable all auxiliary surfaces), `perf` (GPU performance counter queries), `spill_fs` (force fragment shader register spilling to test spill code paths), `cs`/`wm`/`vs`/`gs`/`tes`/`tcs` (per-stage shader dumps), `blorp` (blit/render/other-operations shader dumps), `l3` (L3 cache configuration decisions), `do32` (force 32-wide SIMD dispatch for fragment shaders). | Mesa 7.x | Yes | Ch. 21 |
| `ANV_DEBUG` | (unset) | ANV-specific Vulkan driver debug flags. Comma-separated: `batch` (extra batch buffer logging), `color` (ANSI color output), `nohiz` (disable HiZ), `noccs` (disable CCS), `noaux` (disable aux surfaces). Most flags overlap with `INTEL_DEBUG`; ANV checks both variables. | Mesa 17.x | Yes | Ch. 21 |
| `ANV_ENABLE_PIPELINE_CACHE` | `true` | Set to `false` to disable ANV's in-memory pipeline cache. Forces every pipeline to be rebuilt from SPIR-V on every application launch. Useful for cache invalidation testing. | Mesa 17.x | Yes | Ch. 21 |
| `ANV_QUEUE_THREAD_DISABLE` | (unset) | Set to any non-empty value to disable ANV's asynchronous submission queue thread, serializing all GPU work to the calling thread. Simplifies single-threaded GPU debugging with tools like GDB. | Mesa 20.x | Yes | Ch. 21 |
| `INTEL_MEASURE` | (unset) | Enable Intel's built-in GPU performance measurement infrastructure. Accepts an interval specification controlling how frequently measurements are collected. Outputs CSV-compatible timing data to stdout or a file. | Mesa 21.x | Yes | Ch. 28 |
| `INTEL_MEASURE_BATCH` | `1` | Number of frames to batch before flushing Intel Measure output. Higher values reduce output overhead. | Mesa 21.x | Yes | Ch. 28 |

---

## B.7 Vulkan Validation Layer Variables

The Khronos Vulkan Validation Layers (VVL) are loaded via the layer mechanism (see B.2) and controlled by these additional variables. VVL source is at [github.com/KhronosGroup/Vulkan-ValidationLayers](https://github.com/KhronosGroup/Vulkan-ValidationLayers).

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `VK_LAYER_SETTINGS_PATH` | (unset) | Path to a `vk_layer_settings.txt` file that overrides validation layer options. Controls report flags (error/warn/info/debug/perf), message severity filtering, best practices sub-checks, GPU-Assisted Validation (GPU-AV) settings, synchronization validation toggles, and more. See VVL documentation for the full settings schema. | VVL 1.2.x | Yes | Ch. 18 |
| `VK_LAYER_ENABLES` | (unset) | Comma-separated list to enable specific VVL feature subsets without a settings file. Values include `VK_VALIDATION_FEATURE_ENABLE_GPU_ASSISTED_EXT` (GPU-AV), `VK_VALIDATION_FEATURE_ENABLE_BEST_PRACTICES_EXT` (best practices), `VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT` (synchronization validation), and `VK_VALIDATION_FEATURE_ENABLE_DEBUG_PRINTF_EXT` (shader debug printf). | VVL 1.2.x | Yes | Ch. 18 |
| `VALIDATION_ERROR_LIMIT` | (unset) | Stop emitting validation errors after this many have been reported. Prevents log flooding when a single root cause generates thousands of errors. VVL-specific; not part of the Vulkan specification. | VVL 1.3.x | Yes | Ch. 18 |
| `VK_LOADER_DRIVERS_DISABLE` | (unset) | (Also in B.2.) Suppress specific ICD drivers during validation testing to ensure only the target driver is exercised. | Loader 1.3.x | Yes | Ch. 18 |

> **Warning**: Enabling GPU-AV (`VK_LAYER_ENABLES=VK_VALIDATION_FEATURE_ENABLE_GPU_ASSISTED_EXT`) and synchronization validation simultaneously typically reduces rendering performance by 10–50×. Use them in separate passes: GPU-AV for memory safety, sync validation for hazard checking. Both together on a complex title may make the application unusable for interactive testing.

---

## B.8 DXVK Variables

DXVK translates Direct3D 8/9/10/11 to Vulkan. Its environment variable handling is in `src/util/config.cpp` and `src/util/log/log.cpp`. [Source: DXVK GitHub `src/util/config.cpp`](https://github.com/doitsujin/dxvk/blob/master/src/util/config.cpp)

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `DXVK_LOG_LEVEL` | `info` | Log verbosity: `none`, `error`, `warn`, `info`, `debug`. At `debug` level, DXVK logs every D3D API call with arguments, making it extremely verbose (millions of lines per minute in heavy 3D scenes). | DXVK 0.50 | `debug` yes; others no | Ch. 25 |
| `DXVK_LOG_PATH` | (unset) | Directory path for DXVK log files. If unset, logs are written to stderr or the game's working directory as `d3d11.log` / `d3d9.log`. | DXVK 0.50 | No | Ch. 25 |
| `DXVK_STATE_CACHE` | `1` | Set to `0` to disable DXVK's pipeline state cache entirely. Disabling eliminates cache file I/O but causes full shader recompilation every launch. | DXVK 0.60 | Yes | Ch. 25 |
| `DXVK_STATE_CACHE_PATH` | game directory | Override directory for DXVK's `.dxvk-cache` pipeline state cache files. Useful for placing shared caches on faster storage. | DXVK 0.60 | No | Ch. 25 |
| `DXVK_CONFIG_FILE` | `dxvk.conf` (in game directory) | Path to the DXVK configuration file. DXVK first searches the game directory for `dxvk.conf`; this variable overrides that path. The config file supports per-key/value settings corresponding to most DXVK behaviours. | DXVK 0.70 | No | Ch. 25 |
| `DXVK_HUD` | (unset) | Enable DXVK's built-in on-screen HUD. Accepts comma-separated tokens or `1` (equivalent to `devinfo,fps`) or `full` (all elements). Individual tokens: `devinfo` (GPU name and driver), `fps` (frame rate), `frametimes` (frame time graph), `submissions` (Vulkan queue submissions per frame), `drawcalls` (draw calls per frame), `pipelines` (pipeline count / compile events), `descriptors` (descriptor set usage), `memory` (VRAM/RAM usage), `gpuload` (GPU utilization estimate), `version` (DXVK version), `api` (D3D API version detected), `compiler` (background compile thread activity), `samplers` (active sampler count), `scale=N` (HUD scale factor). | DXVK 0.60 | No (dev-friendly) | Ch. 25 |
| `DXVK_FRAME_RATE` | `0` | Frame rate limiter in frames per second. `0` means unlimited. Useful for power-constrained or thermal-limited scenarios. | DXVK 1.9 | No | Ch. 25 |
| `DXVK_ASYNC` | `0` | Enable asynchronous pipeline compilation (compile shaders on background threads; may cause brief visual corruption as placeholder pipelines are used). Present in async-patched community forks; removed from upstream DXVK by the maintainer due to correctness concerns. Not available in mainline DXVK or Valve's Proton tree. | Fork-specific | Yes | Ch. 25 |

> **Note**: `DXVK_HUD` is one of the few debug-adjacent variables that is broadly safe in production use. Game streamers and benchmark operators routinely set `DXVK_HUD=fps,frametimes` for on-screen overlays without any performance impact. The `gpuload` token relies on driver-reported occupancy estimates which may not be accurate on all hardware.

---

## B.9 VKD3D-Proton Variables

VKD3D-Proton is Valve's fork of VKD3D, implementing Direct3D 12 over Vulkan for use with Proton. Its environment variable documentation is maintained in the project's [README.md on GitHub](https://github.com/HansKristian-Work/vkd3d-proton/blob/master/README.md). Variable parsing is in `libs/vkd3d/vkd3d_debug.c` and `libs/vkd3d/command.c`.

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `VKD3D_DEBUG` | `warn` | Log level for VKD3D-Proton runtime messages: `none`, `err`, `warn`, `info`, `fixme`, `debug`, `trace`. At `trace` level, every D3D12 API call is logged with arguments — one to ten million lines per second in a typical game. | VKD3D-Proton 2.0 | `debug`/`trace` yes; others no | Ch. 25 |
| `VKD3D_SHADER_DEBUG` | `none` | Debug level for messages from the DXBC/DXIL-to-SPIR-V shader compiler: `none`, `err`, `warn`, `info`, `debug`, `trace`. Use `info` to see which shader stages are being compiled and from which format. | VKD3D-Proton 2.0 | Yes | Ch. 25 |
| `VKD3D_LOG_FILE` | (unset) | Redirect `VKD3D_DEBUG` log output to a file instead of stderr. Critical for capturing logs from games that redirect their own stderr. | VKD3D-Proton 2.x | No | Ch. 25 |
| `VKD3D_CONFIG` | (unset) | Comma-separated feature and behaviour flags. See sub-table below. | VKD3D-Proton 2.0 | Conditional | Ch. 25 |
| `VKD3D_VULKAN_DEVICE` | (unset) | Zero-based Vulkan device index to force selection of a specific GPU. The index corresponds to the order devices are returned by `vkEnumeratePhysicalDevices`. | VKD3D-Proton 2.x | No | Ch. 25 |
| `VKD3D_FILTER_DEVICE_NAME` | (unset) | Select a GPU by substring of its `VkPhysicalDeviceProperties.deviceName`. More stable than device index across driver updates. | VKD3D-Proton 2.x | No | Ch. 25 |
| `VKD3D_DISABLE_EXTENSIONS` | (unset) | Semicolon-separated list of Vulkan extension names that VKD3D-Proton should not use even if advertised by the driver. Use to work around extension-specific bugs. | VKD3D-Proton 2.x | Yes | Ch. 25 |
| `VKD3D_FEATURE_LEVEL` | (auto) | Override the D3D feature level reported to the application, e.g., `12_0`, `12_1`, `12_2`. Use to test applications that behave differently under lower feature levels. | VKD3D-Proton 2.3 | Yes | Ch. 25 |
| `VKD3D_SHADER_CACHE_PATH` | `$XDG_CACHE_HOME` | Directory for VKD3D-Proton's compiled shader cache. Set to `0` to disable the cache entirely. | VKD3D-Proton 2.4 | No | Ch. 25 |
| `VKD3D_SHADER_DUMP_PATH` | (unset) | Directory into which VKD3D will dump raw shader bytecode (`$hash.spv`, `$hash.dxbc`, `$hash.dxil`) for all compiled shaders. | VKD3D-Proton 2.x | Yes | Ch. 25 |
| `VKD3D_SHADER_OVERRIDE` | (unset) | Directory containing replacement SPIR-V shader binaries named by hash. If a shader's hash matches a file in this directory, VKD3D substitutes the provided SPIR-V. | VKD3D-Proton 2.x | Yes | Ch. 25 |
| `VKD3D_PROFILE_PATH` | (unset) | Output path for profiling data (requires a build with `-Denable_profiling=true`). | VKD3D-Proton 2.x | Yes | Ch. 28 |
| `VKD3D_SWAPCHAIN_PRESENT_MODE` | (auto) | Force a specific Vulkan present mode: `IMMEDIATE`, `MAILBOX`, `FIFO`, `FIFO_RELAXED`, `FIFO_LATEST_READY` (VK_EXT_present_mode_fifo). | VKD3D-Proton 2.x | No | Ch. 25 |
| `VKD3D_GRAPHICS_PIPELINE_LIBRARY` | `1` | Set to `0` to disable VKD3D's use of `VK_EXT_graphics_pipeline_library`. Disabling removes GPL-based stutter reduction but may work around GPL implementation bugs on certain drivers. | VKD3D-Proton 2.7 | No | Ch. 25 |
| `VKD3D_QUEUE_PROFILE` | (unset) | Enable GPU queue timeline profiling output for work submission analysis. Outputs timing data useful for identifying CPU/GPU synchronization bottlenecks. | VKD3D-Proton 2.8 | Yes | Ch. 28 |

**VKD3D_CONFIG flag sub-table:**

| Flag | Effect | Production-safe? |
|---|---|---|
| `vk_debug` | Enable Vulkan debug extensions and load validation layers. Forces VKD3D to act as if validation is present. | No |
| `skip_application_workarounds` | Bypass all per-application workarounds for debugging; see actual Vulkan errors without workaround masking. | No |
| `nodxr` | Disable DirectX Raytracing support entirely. | Conditional |
| `dxr` | Force-enable DXR even when the driver check would otherwise deem it unsafe. | No |
| `dxr12` | Enable experimental DXR 1.2 support when `VK_EXT_opacity_micromap` is available. | Experimental |
| `force_static_cbv` | Disable dynamic CBV descriptor management on NVIDIA; speed hack that may cause correctness issues. | No |
| `single_queue` | Disable asynchronous compute and transfer queues; all work goes through a single queue. | No |
| `no_upload_hvv` | Disable using host-visible VRAM for the UPLOAD heap; forces system RAM. Conserves VRAM on 8 GB GPUs. | Conditional |
| `force_host_cached` | Map all host-visible allocations with cached memory type. Improves RenderDoc capture performance. | No |
| `no_invariant_position` | Disable invariant position workarounds used for multi-pass rendering. | No |
| `breadcrumbs` | Enable GPU crash breadcrumb instrumentation. Instruments every draw call with a marker; surviving markers indicate how far execution reached before hang. | No (debug only) |
| `pipeline_library_app_cache` | Enable application-level pipeline cache backing for the shader library. | Conditional |
| `descriptor_qa_checks` | Enable descriptor QA validation (verifies descriptor set correctness). | No |
| `no_pipeline_cache` | Disable VKD3D's shader pipeline cache entirely. | Yes |

---

## B.10 Wine / Proton Variables

Wine is the Windows compatibility layer. Proton is Valve's Wine fork with additional features for Steam Play. Synchronisation variable parsing lives in `dlls/ntdll/unix/sync.c`. [Source: Wine GitLab `dlls/ntdll/unix/sync.c`](https://gitlab.winehq.org/wine/wine/-/blob/master/dlls/ntdll/unix/sync.c)

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `WINEESYNC` | `0` | Enable eventfd-based synchronization (esync) to replace Wine's wineserver RPC for NT synchronization object operations. Reduces CPU overhead dramatically for synchronization-heavy Windows applications. Requires no special kernel support beyond standard eventfd. | Wine-esync patch | No | Ch. 25 |
| `WINEFSYNC` | `0` | Enable futex-based synchronization (fsync) using the `futex_waitv()` syscall (merged in Linux 5.16). Preferred over esync when available: lower latency and lower CPU usage. When both `WINEFSYNC=1` and `WINEESYNC=1` are set, Wine prefers fsync if the kernel supports it, otherwise falls back to esync. If only `WINEFSYNC=1` is set and the kernel lacks `futex_waitv`, Wine may fall back to slow wineserver RPC. | Wine-fsync patch | No | Ch. 25 |
| `WINENETSYNC` | `0` | Enable NTSYNC, a kernel driver (`drivers/misc/ntsync.c`) merged in Linux 6.14 that implements Windows synchronization semantics natively in the kernel. Supersedes both esync and fsync when kernel 6.14+ is available. Wine 11 detects kernel 6.14+ automatically and prefers NTSYNC. | Wine 10.16 / Linux 6.14 | No | Ch. 25 |
| `WINEDEBUG` | (unset) | Channel-based Wine debug output. Format: `+channel` (enable), `-channel` (disable), `warn+channel`, `err+channel`. Common channels: `+d3d` (D3D state machine), `+vulkan` (Vulkan calls via winevulkan), `+opengl` (OpenGL calls), `+wined3d` (wined3d renderer), `+heap` (heap allocations), `fixme-all` (suppress all FIXME messages). Run `wine --debugmsg help` for the full channel list. Each enabled channel writes to stderr. | Wine 1.x | Yes | Ch. 25 |
| `WINEDLLOVERRIDES` | (unset) | Override which DLL implementation (native or builtin) is used for each DLL. Format: `dll=type[,type]` where types are `n` (native Windows DLL), `b` (builtin Wine DLL), or comma-separated fallback list. Example: `WINEDLLOVERRIDES="d3d11=n,b"` loads the native (DXVK) `d3d11.dll` first, falling back to Wine's builtin if it fails. | Wine 1.x | No | Ch. 25 |
| `PROTON_LOG` | `0` | Valve Proton: set to `1` to enable verbose Proton startup and runtime logging. Log written to `~/$STEAM_APP_ID.log`. Includes environment setup, Wine configuration steps, and DLL loading decisions. | Proton 3.7 | Yes | Ch. 25 |
| `PROTON_DUMP_DEBUG_COMMANDS` | `0` | Dump all Proton wrapper commands to `/tmp/proton_$USER/run`. Useful for understanding exactly what command line Proton constructs to launch a game. | Proton 4.x | Yes | Ch. 25 |
| `PROTON_NO_ESYNC` | `0` | Force-disable esync even if available. Useful when diagnosing sync-related bugs to isolate whether esync is the cause. | Proton 4.x | No | Ch. 25 |
| `PROTON_NO_FSYNC` | `0` | Force-disable fsync even if available, falling back to esync or wineserver. | Proton 5.13 | No | Ch. 25 |
| `STEAM_COMPAT_DATA_PATH` | (set by Steam launcher) | Path to the Proton compatibility data directory (the Wine prefix). Set automatically by Steam; override for non-Steam use of Proton. | Proton 3.7 | No | Ch. 25 |
| `STEAM_COMPAT_INSTALL_PATH` | (set by Steam launcher) | Path to the game's installation directory. Set automatically by Steam. | Proton 4.x | No | Ch. 25 |

> **Note**: As of Wine 11 and Linux 6.14+, `WINENETSYNC`/NTSYNC is the preferred synchronization mechanism. The NTSYNC driver implements Windows synchronization semantics (mutexes, semaphores, keyed events) natively in kernel space, providing correctness improvements beyond what esync and fsync achieved through userspace approximation. On distributions shipping Linux 6.14+ with Wine 11+, neither `WINEESYNC` nor `WINEFSYNC` need to be explicitly set — Wine detects and uses NTSYNC automatically.

---

## B.11 GStreamer / VA-API Variables

GStreamer uses a category/level debug system. VA-API variables control the hardware video acceleration driver selection used by GStreamer, FFmpeg, and standalone VA-API applications.

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `GST_DEBUG` | (unset) | Colon-separated GStreamer debug category/level pairs, or a global integer level (0–9). Format: `category:level`. Example: `GST_DEBUG=vaapi*:6,GST_ELEMENT:4`. Level meanings: 0=none, 1=ERROR, 2=WARNING, 3=FIXME, 4=INFO, 5=DEBUG, 6=LOG, 7=TRACE, 9=MEMDUMP. Wildcard (`*`) matches category substrings. | GStreamer 0.10 | Yes | Ch. 26 |
| `GST_DEBUG_FILE` | (unset) | Redirect all GStreamer debug output to a file instead of stderr. Essential for capturing logs from media players that suppress stderr. | GStreamer 0.10 | Yes | Ch. 26 |
| `GST_DEBUG_NO_COLOR` | (unset) | Set to `1` to disable ANSI colour codes in `GST_DEBUG` output. Useful when redirecting to files or non-terminal outputs. | GStreamer 0.10 | Yes | Ch. 26 |
| `GST_VA_DRM_DEVICE` | (unset) | Override the DRM device node (e.g., `/dev/dri/renderD128`) used by the GStreamer VA-API plugin (`gst-plugins-bad` / `gstreamer-vaapi`). Required on multi-GPU systems to direct video decode to the correct GPU. Source: `subprojects/gst-plugins-bad/sys/va/gstvautils.c`. | gst-plugins-bad 1.20 | No | Ch. 26 |
| `LIBVA_DRIVER_NAME` | (auto-detected via PCI ID) | Override the VA-API driver to load by name. Recognized values include `radeonsi` (Mesa's GalliumSt VA-API driver for AMD), `iHD` (Intel Media Driver for Gen9+), `i965` (legacy Intel VA-API driver for Gen6–8), `nvidia` (NVIDIA VDPAU bridge), and any driver named `$NAME_drv_video.so` in the search path. | libva 1.x | No | Ch. 26 |
| `LIBVA_DRIVERS_PATH` | `/usr/lib/dri` (distro-dependent) | Colon-separated paths searched for VA-API driver `.so` files. Mirrors `LIBGL_DRIVERS_PATH`'s role for VA-API. Override when using a non-system Mesa installation that includes `radeonsi_drv_video.so`. | libva 1.x | No | Ch. 26 |
| `LIBVA_MESSAGING_LEVEL` | `1` | VA-API message verbosity: `0` = errors only, `1` = errors and warnings, `2` = all messages including informational. | libva 2.x | Yes | Ch. 26 |

> **Note**: `LIBVA_DRIVER_NAME` and `GST_VA_DRM_DEVICE` must both target the same physical GPU on multi-GPU systems. Setting `LIBVA_DRIVER_NAME=radeonsi` while `GST_VA_DRM_DEVICE=/dev/dri/renderD129` (which may be the Intel iGPU) causes VA-API context creation to fail with a cryptic `vaInitialize failed` error. See B.14 for the correct combination pattern.

---

## B.12 EGL Variables

These variables control Mesa's EGL loader and platform selection. Parsed in `src/egl/main/eglapi.c` and `src/egl/drivers/dri2/platform_*.c`.

| Variable | Default | Effect | Introduced | Debug-only? | Chapter |
|---|---|---|---|---|---|
| `EGL_PLATFORM` | (auto-detected) | Force the EGL platform backend. Valid values: `x11` (use `EGL_EXT_platform_x11`), `wayland` (use `EGL_EXT_platform_wayland`), `drm` (use `EGL_EXT_platform_device` with a DRM fd), `surfaceless` (EGL_MESA_platform_surfaceless for headless rendering), `device` (EGL_EXT_platform_device for compute-only use). Auto-detection inspects the current display server. | Mesa EGL | No | Ch. 12 |
| `EGL_LOG_LEVEL` | `warning` | EGL loader log verbosity: `fatal` (only fatal errors), `warning` (default), `info` (include informational messages), `debug` (include all debug output including platform selection). | Mesa EGL | Yes | Ch. 12 |
| `MESA_EGL_NO_X11` | (unset) | Set to `1` to instruct Mesa's EGL loader to skip X11 platform support entirely, even if an X display is available. Forces DRM or surfaceless mode. Useful in container environments where X11 headers were available at build time but an X server is not present at runtime. | Mesa 20.x | No | Ch. 12 |

---

## B.13 DRM / Kernel Module Parameters (sysfs)

> **Important**: The following entries are **not** environment variables. They are Linux kernel module parameters exposed via `/sys/module/` and controlled via `modprobe` arguments or runtime writes to sysfs. They are included here because they are routinely used alongside user-space environment variables in debugging workflows, and because the official references are scattered across driver-specific documentation.

All module parameters can be set permanently in `/etc/modprobe.d/*.conf` as `options <module> <param>=<value>`, set transiently by writing to sysfs (where writeable), or passed on initial module load.

### B.13a DRM Core Module Parameters

Path: `/sys/module/drm/parameters/`. Source: `drivers/gpu/drm/drm_drv.c`. [Source: kernel.org `drivers/gpu/drm/drm_drv.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/drm_drv.c)

| Parameter | Default | Effect | Debug-only? | Chapter |
|---|---|---|---|---|
| `debug` | `0` | Bitmask enabling DRM core debug output to `dmesg`. Bit meanings: `0x01` = DRM core, `0x02` = driver (per-driver debug), `0x04` = KMS (modesetting), `0x08` = PRIME (dma-buf), `0x10` = atomic (atomic modesetting), `0x20` = VBL (vblank), `0x40` = state (full state dumps on atomic commits), `0x80` = lease, `0x100` = DP AUX channel. Set via `/sys/module/drm/parameters/debug` or `drm.debug=0x46` on the kernel command line. | Yes | Ch. 5 |
| `vblank_offdelay` | `5000` | Milliseconds of inactivity before disabling vblank interrupts. `0` means never disable; `-1` means always disable immediately after each vblank. Tuning this affects power consumption on idle displays. | No | Ch. 5 |
| `timestamp_precision_usec` | `20` | Target precision for DRM hardware timestamps, in microseconds. Tighter values increase CPU overhead for timestamp reconciliation. | No | Ch. 5 |

### B.13b AMDGPU Module Parameters

Path: `/sys/module/amdgpu/parameters/`. Source: `drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c`. [Source: kernel `drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c)

| Parameter | Default | Effect | Debug-only? | Chapter |
|---|---|---|---|---|
| `debug` | `0` | AMDGPU driver debug bitmask. Bit definitions in `amdgpu_drv.c`. Common bits: `0x1` = AMDGPU_DEBUG_INFO, `0x100` = AMDGPU_DEBUG_REG. | Yes | Ch. 7 |
| `gpu_recovery` | `-1` (auto) | Enable (`1`) or disable (`0`) GPU reset on hang. Auto (`-1`) enables recovery on supported hardware. Disabling is useful when debugging hangs to preserve state for post-mortem analysis with umr or amdgpu_top. | No | Ch. 7 |
| `vm_debug` | `0` | Set to `1` to log all VM page table bind and unbind operations to dmesg. Very verbose on workloads with frequent buffer evictions. | Yes | Ch. 7 |
| `sched_jobs` | `32` | Number of jobs in the GPU DRM scheduler ring buffer. Increase for workloads with many simultaneous command submissions. Requires module reload. | No | Ch. 7 |
| `ppfeaturemask` | (auto from VBIOS) | Bitmask controlling power and thermal management feature enablement. Common use: `ppfeaturemask=0xffffffff` to enable all power features including manual GPU frequency/voltage control. Hex value; see `drivers/gpu/drm/amd/pm/powerplay/amd_powerplay.c` for bit definitions. | No | Ch. 7 |
| `gttsize` | (auto) | GTT (Graphics Translation Table) size in megabytes. Controls the aperture available for CPU-mapped GPU buffers. | No | Ch. 7 |
| `aspm` | `-1` (auto) | PCIe Active State Power Management: `0` = disable, `1` = enable, `-1` = use firmware default. Enable on laptops for battery savings; disable on workstations if experiencing PCIe link instability. | No | Ch. 7 |

### B.13c i915 Module Parameters

Path: `/sys/module/i915/parameters/`. Source: `drivers/gpu/drm/i915/i915_params.c`. [Source: kernel `drivers/gpu/drm/i915/i915_params.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/i915/i915_params.c)

> **Note**: Intel Xe-class hardware (Meteor Lake and newer) uses the `xe` kernel driver rather than `i915`. The `xe` module has its own parameters at `/sys/module/xe/parameters/`.

| Parameter | Default | Effect | Debug-only? | Chapter |
|---|---|---|---|---|
| `enable_guc` | `3` (Gen11+) | Enable GuC (`0x1`), HuC (`0x2`), or both (`0x3`) firmware-based command submission. GuC provides per-context priority scheduling; HuC accelerates H.264/H.265 decode. On older hardware (Gen9 and prior) defaults to `0`. | No | Ch. 8 |
| `enable_fbc` | `1` | Enable Frame Buffer Compression (FBC) power saving for static desktop frames. May cause display glitches on some panels; set to `0` as a workaround. | No | Ch. 8 |
| `enable_psr` | `1` | Enable Panel Self Refresh (PSR). Allows the display to refresh from its local memory without GPU involvement. PSR2 (selective region update) is controlled by `enable_psr2_sel_fetch`. | No | Ch. 8 |
| `modeset` | `-1` (auto) | Enable (`1`), disable (`0`), or auto-detect (`-1`) kernel modesetting. On modern kernels this is always auto-detected; explicit override is needed only for debugging. | No | Ch. 8 |
| `force_probe` | (unset) | Comma-separated list of PCI device IDs (hex, no prefix) to force i915 to probe even if the device is not in the supported list. Used for pre-production hardware bring-up. | Yes | Ch. 8 |

### B.13d DRM Memory Manager Debug

The DRM memory manager debug (`DRM_DEBUG_MM`) is not a module parameter but a kernel build-time option (`CONFIG_DRM_DEBUG_MM=y`). When enabled, it instruments TTM and GEM memory managers with additional allocation tracking. At runtime, memory allocation state can be dumped from `/sys/kernel/debug/dri/0/mm_dump` (requires `CONFIG_DEBUG_FS=y` and debugfs mounted). To enable driver-channel dmesg output alongside, combine with `drm.debug=0x02`. [Source: kernel documentation](https://www.kernel.org/doc/html/latest/gpu/drm-internals.html)

---

## B.14 Variable Stacking and Interaction

Most environment variables operate independently, but several key diagnostic workflows require setting multiple variables in combination. This section documents the most useful combinations, ordered by use case.

### GPU Hang Debugging (AMDGPU / RADV)

```bash
# User-space
RADV_DEBUG=hang AMDGPU_DEBUG=vm

# Kernel-side (write to sysfs)
echo 0x02 | sudo tee /sys/module/drm/parameters/debug
```

`RADV_DEBUG=hang` enables RADV's internal GPU hang detection: it waits on submission fences with a 10-second timeout and, on timeout, prints which Vulkan command buffer was in flight and the pipeline state at the point of hang. `AMDGPU_DEBUG=vm` enables the libdrm layer's VM fault logging, recording the faulting virtual address. `drm.debug=0x02` (driver channel) prints kernel-side GPU scheduler state and TTM eviction events to dmesg. Together the three provide: the command buffer that hung (RADV), the faulting VA (libdrm amdgpu), and the kernel memory manager state at the time (DRM core).

### Shader Miscompile Bisection (RADV + ACO)

```bash
RADV_DEBUG=shaders,shaderstats ACO_DEBUG=validateir,validatera NIR_DEBUG=validate
```

`RADV_DEBUG=shaders` dumps the final ISA assembly for every compiled pipeline stage to stderr. `RADV_DEBUG=shaderstats` appends SGPR/VGPR register usage statistics after each dump — essential for identifying register pressure issues. `ACO_DEBUG=validateir` runs the ACO IR validator after each compiler pass, catching IR invariant violations at the pass boundary rather than as downstream miscompiles. `ACO_DEBUG=validatera` runs the register allocation validator. `NIR_DEBUG=validate` catches NIR invariant violations before they even reach ACO. Redirect stderr to a file and use `spirv-dis` alongside if the miscompile suspect is in SPIR-V input: `RADV_DEBUG=spirv,shaders`.

### GPL Fast-Link: RADV + DXVK

```bash
RADV_PERFTEST=gpl RADV_DEBUG=gpl DXVK_LOG_LEVEL=info
```

`RADV_PERFTEST=gpl` enables RADV's Graphics Pipeline Library fast-link path on hardware or Mesa versions where it is not yet the default. `RADV_DEBUG=gpl` logs every pipeline creation event, showing whether the fast-link or slow-link code path was taken. DXVK must also request `VK_EXT_graphics_pipeline_library` — set `dxvk.enableGraphicsPipelineLibrary = True` in `dxvk.conf`. A common mistake is enabling GPL only on one side; `RADV_DEBUG=gpl` output distinguishes the two paths.

> **Tip**: As of Mesa 24.x and DXVK 2.x, GPL fast-link is enabled by default on GFX10+ hardware in most configurations. Use `RADV_DEBUG=gpl` to confirm your specific game and DXVK version are using the fast path before spending time tuning.

### VKD3D-Proton DXR on Pre-RDNA2 Hardware

```bash
VKD3D_CONFIG=dxr,emulate_rt RADV_PERFTEST=emulate_rt
```

`VKD3D_CONFIG=dxr` forces VKD3D to expose D3D12 raytracing to the application regardless of the hardware safety assessment. `RADV_PERFTEST=emulate_rt` enables RADV's shader-based BVH traversal fallback for hardware without dedicated RT acceleration (pre-RDNA2). On RDNA2+ hardware, `RADV_PERFTEST=emulate_rt` is not needed; RADV exposes RT extensions unconditionally. Performance with software-emulated RT on pre-RDNA2 hardware is typically 5–20× slower than native; this combination is for compatibility testing, not production use.

### Wine Synchronization Fallback Chain

```bash
WINEFSYNC=1 WINEESYNC=1
```

When both are set, Wine's synchronization initialization code in `dlls/ntdll/unix/sync.c` probes for `futex_waitv` kernel support at runtime. If the probe succeeds (Linux 5.16+), fsync is used. If the probe fails, Wine falls back to esync (eventfd-based). Setting only `WINEFSYNC=1` without `WINEESYNC=1` risks a silent fallback to wineserver RPC synchronization if fsync initialization fails, which can cause significant CPU overhead for synchronization-heavy games. Setting both provides graceful degradation across kernel versions.

On Linux 6.14+ with Wine 11+, set `WINENETSYNC=1` instead — NTSYNC supersedes both and is unconditionally superior when available.

### Multi-GPU OpenGL and Vulkan Selection

```bash
# OpenGL (Mesa DRI)
DRI_PRIME=1

# Vulkan (explicit ICD)
VK_DRIVER_FILES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json
```

`DRI_PRIME=1` applies only to Mesa's DRI-based OpenGL. It uses the DRM render node device selection to pick the non-default GPU. Vulkan applications use the ICD loader mechanism and must be directed via `VK_DRIVER_FILES` or `VK_ICD_FILENAMES`. Both may be needed simultaneously when a single application uses OpenGL and Vulkan interop (e.g., an OpenXR runtime that composites via OpenGL but renders via Vulkan). Ensure both variables point to the same physical GPU.

### VA-API Device Selection on Multi-GPU Systems

```bash
LIBVA_DRIVER_NAME=radeonsi LIBVA_DRIVERS_PATH=/usr/lib/dri GST_VA_DRM_DEVICE=/dev/dri/renderD128
```

`LIBVA_DRIVER_NAME` selects the software driver module; `LIBVA_DRIVERS_PATH` provides the directory to find it; `GST_VA_DRM_DEVICE` specifies the DRM render node. All three must target the same physical GPU. On a system with an AMD dGPU at `/dev/dri/renderD128` and an Intel iGPU at `/dev/dri/renderD129`, mixing `LIBVA_DRIVER_NAME=iHD` with `GST_VA_DRM_DEVICE=/dev/dri/renderD128` will cause `vaInitialize` to fail with an unhelpful error message.

### Vulkan Validation + DXVK Without Log Flooding

```bash
VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation \
VK_LAYER_SETTINGS_PATH=/path/to/vk_layer_settings.txt \
VALIDATION_ERROR_LIMIT=50 \
DXVK_LOG_LEVEL=warn
```

Running DXVK under Vulkan validation layers is extremely useful for catching driver and application bugs, but naively combining `DXVK_LOG_LEVEL=debug` (per-draw-call) with validation at default settings produces so many log lines the system becomes unusable. Use `VALIDATION_ERROR_LIMIT=50` to stop after 50 errors. Set `DXVK_LOG_LEVEL=warn` to see only DXVK warnings, not per-call trace. Configure `vk_layer_settings.txt` to enable only the validation category relevant to the current investigation (e.g., `VK_VALIDATION_FEATURE_ENABLE_SYNCHRONIZATION_VALIDATION_EXT` for hazard checking, not simultaneously with GPU-AV).

### Full Proton Diagnostic Stack

```bash
PROTON_LOG=1 \
WINEDEBUG=+d3d,+wined3d \
DXVK_LOG_LEVEL=info \
VKD3D_DEBUG=warn \
RADV_DEBUG=hang
```

A practical starting point for diagnosing Proton game launch failures or crashes: `PROTON_LOG=1` provides the Proton startup/environment log; `WINEDEBUG=+d3d,+wined3d` adds Wine D3D state machine events; `DXVK_LOG_LEVEL=info` provides D3D-to-Vulkan API translation events without per-draw spam; `VKD3D_DEBUG=warn` surfaces VKD3D warnings (only relevant when the game uses D3D12); `RADV_DEBUG=hang` catches GPU hangs with command buffer identification. Add `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` and `VALIDATION_ERROR_LIMIT=20` only when investigating a specific Vulkan validation error, due to the 10–50× performance overhead of GPU-AV.

---

## B.15 Alphabetical Index

The following flat index maps each variable or parameter name to its appendix section and production-safety classification. Section numbers use the form `B.N` matching the heading structure above.

| Variable / Parameter | Section | Debug-only? |
|---|---|---|
| `ACO_DEBUG` | B.5c | Yes |
| `AMDGPU_DEBUG` | B.5d | Yes |
| `ANV_DEBUG` | B.6 | Yes |
| `ANV_ENABLE_PIPELINE_CACHE` | B.6 | Yes |
| `ANV_QUEUE_THREAD_DISABLE` | B.6 | Yes |
| `DISABLE_LAYER_AMD_SWITCHABLE_GRAPHICS_1` | B.2 | Yes |
| `DRI_PRIME` | B.1 | No |
| `DRIRC_DISABLE` | B.1 | Yes |
| `DXVK_ASYNC` | B.8 | Yes |
| `DXVK_CONFIG_FILE` | B.8 | No |
| `DXVK_FRAME_RATE` | B.8 | No |
| `DXVK_HUD` | B.8 | No |
| `DXVK_LOG_LEVEL` | B.8 | Conditional |
| `DXVK_LOG_PATH` | B.8 | No |
| `DXVK_STATE_CACHE` | B.8 | Yes |
| `DXVK_STATE_CACHE_PATH` | B.8 | No |
| `EGL_LOG_LEVEL` | B.12 | Yes |
| `EGL_PLATFORM` | B.12 | No |
| `GST_DEBUG` | B.11 | Yes |
| `GST_DEBUG_FILE` | B.11 | Yes |
| `GST_DEBUG_NO_COLOR` | B.11 | Yes |
| `GST_VA_DRM_DEVICE` | B.11 | No |
| `INTEL_DEBUG` | B.6 | Yes |
| `INTEL_MEASURE` | B.6 | Yes |
| `INTEL_MEASURE_BATCH` | B.6 | Yes |
| `LIBGL_DEBUG` | B.1 | Yes |
| `LIBGL_DRIVERS_PATH` | B.1 | No |
| `LIBVA_DRIVER_NAME` | B.11 | No |
| `LIBVA_DRIVERS_PATH` | B.11 | No |
| `LIBVA_MESSAGING_LEVEL` | B.11 | Yes |
| `MESA_DEBUG` | B.1 | Yes |
| `MESA_EGL_NO_X11` | B.12 | No |
| `MESA_EXTENSION_OVERRIDE` | B.1 | Yes |
| `MESA_GLES_VERSION_OVERRIDE` | B.4 | Yes |
| `MESA_GL_VERSION_OVERRIDE` | B.4 | Yes |
| `MESA_GLSL_CACHE_DISABLE` | B.3 | Yes |
| `MESA_GLSL_VERSION_OVERRIDE` | B.4 | Yes |
| `MESA_LOADER_DRIVER_OVERRIDE` | B.1 | No |
| `MESA_SHADER_CACHE_DIR` | B.3 | No |
| `MESA_SHADER_CACHE_DISABLE` | B.3 | Yes |
| `MESA_SHADER_CACHE_MAX_SIZE` | B.3 | No |
| `MESA_SHADER_CACHE_SHOW_STATS` | B.3 | Yes |
| `MESA_SPIRV_DUMP_PATH` | B.4 | Yes |
| `MESA_VK_VERSION_OVERRIDE` | B.1 | Yes |
| `NIR_DEBUG` | B.4 | Yes |
| `NVK_DEBUG` | B.2 | Yes |
| `PROTON_DUMP_DEBUG_COMMANDS` | B.10 | Yes |
| `PROTON_LOG` | B.10 | Yes |
| `PROTON_NO_ESYNC` | B.10 | No |
| `PROTON_NO_FSYNC` | B.10 | No |
| `RADV_DEBUG` | B.5a | Conditional |
| `RADV_PERFTEST` | B.5b | Conditional |
| `STEAM_COMPAT_DATA_PATH` | B.10 | No |
| `STEAM_COMPAT_INSTALL_PATH` | B.10 | No |
| `VALIDATION_ERROR_LIMIT` | B.7 | Yes |
| `VK_ADD_DRIVER_FILES` | B.2 | No |
| `VK_DEVICE_LAYERS` | B.2 | Yes |
| `VK_DRIVER_FILES` | B.2 | No |
| `VK_ICD_FILENAMES` | B.2 | No |
| `VK_INSTANCE_LAYERS` | B.2 | Yes |
| `VK_LAYER_ENABLES` | B.7 | Yes |
| `VK_LAYER_PATH` | B.2 | No |
| `VK_LAYER_SETTINGS_PATH` | B.7 | Yes |
| `VK_LOADER_DEBUG` | B.2 | Yes |
| `VK_LOADER_DRIVERS_DISABLE` | B.2 | Yes |
| `VK_LOADER_LAYERS_DISABLE` | B.2 | Yes |
| `VKD3D_CONFIG` | B.9 | Conditional |
| `VKD3D_DEBUG` | B.9 | Conditional |
| `VKD3D_DISABLE_EXTENSIONS` | B.9 | Yes |
| `VKD3D_FEATURE_LEVEL` | B.9 | Yes |
| `VKD3D_FILTER_DEVICE_NAME` | B.9 | No |
| `VKD3D_FRAME_RATE` | B.9 | No |
| `VKD3D_GRAPHICS_PIPELINE_LIBRARY` | B.9 | No |
| `VKD3D_LOG_FILE` | B.9 | No |
| `VKD3D_PROFILE_PATH` | B.9 | Yes |
| `VKD3D_QUEUE_PROFILE` | B.9 | Yes |
| `VKD3D_SHADER_CACHE_PATH` | B.9 | No |
| `VKD3D_SHADER_DEBUG` | B.9 | Yes |
| `VKD3D_SHADER_DUMP_PATH` | B.9 | Yes |
| `VKD3D_SHADER_OVERRIDE` | B.9 | Yes |
| `VKD3D_SWAPCHAIN_PRESENT_MODE` | B.9 | No |
| `VKD3D_VULKAN_DEVICE` | B.9 | No |
| `WINEESYNC` | B.10 | No |
| `WINEDEBUG` | B.10 | Yes |
| `WINEDLLOVERRIDES` | B.10 | No |
| `WINEFSYNC` | B.10 | No |
| `WINENETSYNC` | B.10 | No |
| `amdgpu.aspm` | B.13b | No |
| `amdgpu.debug` | B.13b | Yes |
| `amdgpu.gpu_recovery` | B.13b | No |
| `amdgpu.gttsize` | B.13b | No |
| `amdgpu.ppfeaturemask` | B.13b | No |
| `amdgpu.sched_jobs` | B.13b | No |
| `amdgpu.vm_debug` | B.13b | Yes |
| `drm.debug` | B.13a | Yes |
| `drm.timestamp_precision_usec` | B.13a | No |
| `drm.vblank_offdelay` | B.13a | No |
| `i915.enable_fbc` | B.13c | No |
| `i915.enable_guc` | B.13c | No |
| `i915.enable_psr` | B.13c | No |
| `i915.force_probe` | B.13c | Yes |
| `i915.modeset` | B.13c | No |

---

## Integrations

This appendix consolidates variables that appear individually throughout the book. Key cross-references:

- **Chapter 5** (DRM Architecture): `drm.debug` bitmask usage during atomic modesetting debugging.
- **Chapter 7** (AMDGPU Driver): `AMDGPU_DEBUG`, `amdgpu.*` module parameters, and GPU scheduler tuning.
- **Chapter 8** (i915 Driver): `i915.*` module parameters for GuC, FBC, PSR, and Xe migration.
- **Chapter 12** (EGL): `EGL_PLATFORM`, `MESA_EGL_NO_X11` for platform backend selection.
- **Chapter 13** (Mesa Loader): `LIBGL_DRIVERS_PATH`, `MESA_LOADER_DRIVER_OVERRIDE`, `DRI_PRIME`.
- **Chapter 14** (Mesa OpenGL): `MESA_DEBUG`, `MESA_GL_VERSION_OVERRIDE`, `MESA_EXTENSION_OVERRIDE`.
- **Chapter 15** (Mesa Compilers): `NIR_DEBUG`, `ACO_DEBUG`, `MESA_SPIRV_DUMP_PATH`.
- **Chapter 16** (Shader Cache): `MESA_SHADER_CACHE_*` variables in the disk cache workflow.
- **Chapter 18** (Vulkan Loader): All `VK_*` variables, layer activation, and ICD selection.
- **Chapter 20** (RADV): Full `RADV_DEBUG` and `RADV_PERFTEST` flag coverage.
- **Chapter 21** (ANV): `INTEL_DEBUG`, `ANV_*` variables.
- **Chapter 25** (Gaming Layer): `DXVK_*`, `VKD3D_*`, `WINE*`, `PROTON_*` variables for the full translation stack.
- **Chapter 26** (Video Decode): `GST_DEBUG`, `LIBVA_*`, `GST_VA_DRM_DEVICE` for VA-API debugging.
- **Chapter 28** (Profiling and Tracing): `INTEL_MEASURE`, `VKD3D_QUEUE_PROFILE`, `VKD3D_PROFILE_PATH`.

---

## References

1. Mesa 3D Graphics Library — Environment Variables documentation. [https://docs.mesa3d.org/envvars.html](https://docs.mesa3d.org/envvars.html)
2. Mesa RADV driver documentation. [https://docs.mesa3d.org/drivers/radv.html](https://docs.mesa3d.org/drivers/radv.html)
3. Mesa source — `src/amd/vulkan/radv_debug.c`. [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_debug.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/vulkan/radv_debug.c)
4. Mesa source — `src/amd/compiler/aco_debug.cpp`. [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/aco_debug.cpp](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/compiler/aco_debug.cpp)
5. Mesa source — `src/compiler/nir/nir.c`. [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/compiler/nir/nir.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/compiler/nir/nir.c)
6. Mesa source — `src/loader/loader.c`. [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/loader/loader.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/loader/loader.c)
7. Mesa source — `src/util/disk_cache.c`. [https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/util/disk_cache.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/util/disk_cache.c)
8. Khronos Vulkan-Loader — Loader Environment Variables documentation. [https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderEnvironmentVariables.md](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderEnvironmentVariables.md)
9. Khronos Vulkan-Loader — Driver Interface documentation. [https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDriverInterface.md](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDriverInterface.md)
10. DXVK GitHub repository — README and configuration. [https://github.com/doitsujin/dxvk/blob/master/README.md](https://github.com/doitsujin/dxvk/blob/master/README.md)
11. VKD3D-Proton GitHub repository — README. [https://github.com/HansKristian-Work/vkd3d-proton/blob/master/README.md](https://github.com/HansKristian-Work/vkd3d-proton/blob/master/README.md)
12. VKD3D-Proton — Debugging and Diagnostics. [https://deepwiki.com/HansKristian-Work/vkd3d-proton/6-debugging-and-diagnostics](https://deepwiki.com/HansKristian-Work/vkd3d-proton/6-debugging-and-diagnostics)
13. Collabora — "The futex_waitv() syscall and gaming on Linux". [https://www.collabora.com/news-and-blog/blog/2023/02/17/the-futex-waitv-syscall-gaming-on-linux/](https://www.collabora.com/news-and-blog/blog/2023/02/17/the-futex-waitv-syscall-gaming-on-linux/)
14. GamingOnLinux — "Linux Kernel 5.16 brings futex2 for Linux Gaming". [https://www.gamingonlinux.com/2022/01/linux-kernel-516-is-out-now-bringing-the-futex2-work-to-help-linux-gaming/](https://www.gamingonlinux.com/2022/01/linux-kernel-516-is-out-now-bringing-the-futex2-work-to-help-linux-gaming/)
15. Wine 11 — NTSync support announcement. [https://9to5linux.com/wine-11-officially-released-with-ntsync-support-vulkan-h-264-decoding-and-more](https://9to5linux.com/wine-11-officially-released-with-ntsync-support-vulkan-h-264-decoding-and-more)
16. GStreamer debugging tools documentation. [https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html](https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html)
17. libva source — `va/va.c` (`LIBVA_DRIVER_NAME`, `LIBVA_DRIVERS_PATH`). [https://github.com/intel/libva](https://github.com/intel/libva)
18. Linux kernel DRM documentation. [https://www.kernel.org/doc/html/latest/gpu/drm-internals.html](https://www.kernel.org/doc/html/latest/gpu/drm-internals.html)
19. Linux kernel — `drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c`. [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c)
20. Linux kernel — `drivers/gpu/drm/i915/i915_params.c`. [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/i915/i915_params.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/gpu/drm/i915/i915_params.c)
21. Arch Wiki — AMDGPU. [https://wiki.archlinux.org/title/AMDGPU](https://wiki.archlinux.org/title/AMDGPU)
22. Arch Wiki — Vulkan. [https://wiki.archlinux.org/title/Vulkan](https://wiki.archlinux.org/title/Vulkan)
23. Khronos Vulkan Validation Layers — GitHub. [https://github.com/KhronosGroup/Vulkan-ValidationLayers](https://github.com/KhronosGroup/Vulkan-ValidationLayers)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
