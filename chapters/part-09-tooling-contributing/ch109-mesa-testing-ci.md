# Chapter 109: Mesa Testing — piglit, dEQP, and Continuous Integration

**Part IX — Tooling and Contributing**

**Audiences**: Mesa contributors, GPU driver developers, CI/CD engineers, and graphics conformance engineers. This chapter is the operational companion to [Chapter 31: Conformance and Regression Testing](ch31-conformance-regression-testing.md), which covers the theory of conformance certification and the test suite architectures. Chapter 109 focuses on how Mesa's testing infrastructure actually works day-to-day: the tools contributors run locally, the CI pipeline that gates every merge request, the hardware farms that execute tests on real GPUs, and the workflows for writing, submitting, and maintaining tests.

---

## Table of Contents

1. [Why GPU Driver Testing Is Hard](#1-why-gpu-driver-testing-is-hard)
2. [piglit — Mesa's OpenGL Regression Suite](#2-piglit--mesas-opengl-regression-suite)
   - [Test Categories and Directory Layout](#21-test-categories-and-directory-layout)
   - [Running piglit](#22-running-piglit)
   - [The `.shader_test` Format](#23-the-shader_test-format)
   - [Contributing a piglit Test](#24-contributing-a-piglit-test)
3. [dEQP and VK-GL-CTS](#3-deqp-and-vk-gl-cts)
   - [Test Modules](#31-test-modules)
   - [Running dEQP Locally](#32-running-deqp-locally)
   - [QPA Log Files and Result States](#33-qpa-log-files-and-result-states)
   - [Caselist Files and Mustpass](#34-caselist-files-and-mustpass)
4. [deqp-runner — Mesa's Parallel CTS Executor](#4-deqp-runner--mesas-parallel-cts-executor)
   - [Core Features](#41-core-features)
   - [Basic Usage](#42-basic-usage)
   - [Baselines and Expected Failures](#43-baselines-and-expected-failures)
   - [Flake Handling](#44-flake-handling)
   - [Sharding for CI](#45-sharding-for-ci)
5. [Mesa's GitLab CI Pipeline](#5-mesas-gitlab-ci-pipeline)
   - [Pipeline Stages](#51-pipeline-stages)
   - [Build Jobs](#52-build-jobs)
   - [Driver-Specific CI Includes](#53-driver-specific-ci-includes)
   - [Test Jobs and Hardware Runners](#54-test-jobs-and-hardware-runners)
   - [Merge Bot and Queue Management](#55-merge-bot-and-queue-management)
6. [LAVA — Hardware-in-the-Loop CI for ARM GPUs](#6-lava--hardware-in-the-loop-ci-for-arm-gpus)
   - [LAVA Architecture](#61-lava-architecture)
   - [How Mesa Uses LAVA](#62-how-mesa-uses-lava)
   - [CI-tron: The Next-Generation Hardware CI Gateway](#63-ci-tron-the-next-generation-hardware-ci-gateway)
7. [Shader Compile Testing with shader-db](#7-shader-compile-testing-with-shader-db)
   - [What shader-db Measures](#71-what-shader-db-measures)
   - [Capturing Shaders and Running the Suite](#72-capturing-shaders-and-running-the-suite)
   - [Before/After Comparisons](#73-beforeafter-comparisons)
   - [Shader Reduction](#74-shader-reduction)
8. [Trace-Based Render Comparison Testing](#8-trace-based-render-comparison-testing)
   - [apitrace and Piglit's Replayer](#81-apitrace-and-piglits-replayer)
   - [Driver Trace Configurations](#82-driver-trace-configurations)
   - [Running Traces Locally](#83-running-traces-locally)
9. [Writing Tests for Mesa](#9-writing-tests-for-mesa)
   - [When to Add a Test](#91-when-to-add-a-test)
   - [Writing a C piglit Test](#92-writing-a-c-piglit-test)
   - [Writing a `.shader_test` File](#93-writing-a-shader_test-file)
   - [Submitting Tests with Driver Fixes](#94-submitting-tests-with-driver-fixes)
10. [Performance and Microbenchmarks](#10-performance-and-microbenchmarks)
    - [vkmark and glmark2](#101-vkmark-and-glmark2)
    - [Phoronix Test Suite Integration](#102-phoronix-test-suite-integration)
    - [Performance Regression Tracking](#103-performance-regression-tracking)
11. [Integrations](#11-integrations)

---

## 1. Why GPU Driver Testing Is Hard

Testing a GPU driver is qualitatively harder than testing most other software. The difficulty comes from several compounding factors.

**Silent failures.** A GPU driver that produces wrong pixels does not crash — it silently misrenders. A rasterization bug that shifts a triangle edge by one pixel, a shader compiler bug that miscalculates a specular reflection, or a texture sampling bug that reads from the wrong mip level can all pass undetected without pixel-level comparison against a reference. Unlike a server application where incorrect output typically produces a non-200 HTTP status code or a thrown exception, a GPU driver can be profoundly wrong while appearing to "work."

**Non-determinism.** GPU hardware is parallel and pipelined. Floating-point operations may be performed in different order depending on workgroup scheduling. Shader compilers optimise based on live register analysis and heuristics that can change between compiler versions. Two runs of the same test on the same hardware can produce slightly different pixel values, making strict equality comparison unreliable for floating-point rendering. Mesa's test infrastructure uses threshold-based comparison (typically a tolerance of 1/256 per channel) for floating-point cases, while restricting exact-match expectations to integer or fixed-point results.

**Hardware specificity.** A bug in the driver's handling of an AMD GCN-generation texture cache affects only hardware with that cache architecture. A bug in the NIR lowering pass for Qualcomm Adreno's A7xx tile-based rendering affects only that GPU family. Hardware coverage must span all supported GPUs, which is expensive and time-consuming. Mesa's CI consequently maintains physical GPU hardware farms across x86_64 (Intel and AMD), ARM64 (Qualcomm, Arm Mali), and RISC-V SBCs.

**Unrelated-change regressions.** Because shader compilation is triggered at draw time and is sensitive to pipeline state combinations, a change to an unrelated pass in the NIR compiler can cause a previously-passing test to fail not because of a logic error in the patch, but because the pass ordering changed in a way that altered register allocation decisions. These "transitive" regressions are particularly tricky because `git bisect` points at a neutral-looking commit.

**Conformance cost.** The Vulkan CTS (`dEQP-VK`) comprises over 700,000 test cases as of 2026. A full run against real hardware takes eight or more hours. No contributor can afford to run the full CTS before posting a merge request. Mesa's layered testing strategy solves this with fast subsets for MR-time CI, nightly full runs for post-merge validation, and shader-db for compiler-only metrics that run without a GPU.

**Mesa's testing philosophy** is therefore layered:

1. **Compile-time**: shader-db statistics catch compiler regressions without hardware.
2. **Software renderer**: Lavapipe (software Vulkan) and llvmpipe can run the full CTS suite and piglit without a GPU, catching API-correctness issues.
3. **Fast hardware subset**: a curated subset of dEQP and piglit tests runs on real GPUs during the merge request pipeline, targeting tests known to catch regressions in the changed driver code.
4. **Full nightly CTS**: complete VK-GL-CTS and piglit runs scheduled nightly against all supported hardware configurations.
5. **Trace replay**: application traces captured from real games and applications replay headlessly and compare rendered frames against stored checksums.

---

## 2. piglit — Mesa's OpenGL Regression Suite

piglit ([http://piglit.freedesktop.org](http://piglit.freedesktop.org)) is Mesa's primary OpenGL regression test suite. It predates dEQP and covers OpenGL and OpenGL ES functionality, extension compliance, and compiler behaviour through thousands of small, focused tests. Unlike dEQP's C++ test framework, piglit tests are either standalone C programs or text-format `.shader_test` scripts interpreted by the `shader_runner` tool. [Source](https://gitlab.freedesktop.org/mesa/piglit)

### 2.1 Test Categories and Directory Layout

The piglit repository is organised under `tests/`:

```
tests/
  spec/           OpenGL specification compliance
    arb_texture_buffer_object/
    ext_texture_norm16/
    glsl-1.30/
    ...
  bugs/           Regression tests for fixed driver bugs
    fdo12345.shader_test
    ...
  shaders/        General GLSL shader tests
  texturing/      Texture format, sampling, filtering tests
  fbo/            Framebuffer object tests
  perf/           Performance microbenchmarks (timing-based)
  sanity/         Minimal smoke tests run first
```

The `spec/` subdirectory mirrors the GL extension registry: each extension or GL version has its own subdirectory whose name matches the Khronos registry string (e.g., `spec/arb_derivative_control/`). This makes it easy to locate tests when debugging an extension-specific regression.

Test profiles are Python files in `tests/`:
- `sanity.py` — minimal smoke tests, useful to verify the test runner itself works.
- `quick.py` — reduced variants of heavy tests; suitable for fast runs during development.
- `gpu.py` — GPU-intensive tests that need real hardware.
- `all.py` — the full suite.

[Source: piglit tests/](https://gitlab.freedesktop.org/mesa/piglit/-/tree/main/tests)

### 2.2 Running piglit

piglit requires a Python 3 environment and a working OpenGL implementation. The typical development workflow:

```bash
# Build piglit
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j$(nproc)

# Run the quick profile on llvmpipe (no GPU needed)
LIBGL_ALWAYS_SOFTWARE=1 \
  ./piglit run quick results/quick-llvmpipe -j4

# Run only tests matching an extension name
LIBGL_ALWAYS_SOFTWARE=1 \
  ./piglit run -t ext_texture_norm16 all results/norm16

# Generate an HTML summary
./piglit summary html --overwrite summary/ results/quick-llvmpipe

# Console diff between two runs (regression detection)
./piglit summary console results/before results/after
```

The `-j4` flag parallelises test execution across four worker processes. piglit stores results as JSON in the output directory; the summary tools parse these files.

For comparing a patch against a baseline:

```bash
# Record baseline
./piglit run quick results/baseline

# Apply patch, rebuild Mesa, re-run
./piglit run quick results/after-patch

# Show regressions and fixes
./piglit summary console results/baseline results/after-patch
```

piglit's HTML summary generates per-test-status pages: `regressions.html`, `fixes.html`, `changes.html`, and `problems.html`, each filtered to the relevant subset of result changes. [Source](https://docs.mesa3d.org/submittingpatches.html)

### 2.3 The `.shader_test` Format

The `.shader_test` format is piglit's text-based test description language. A single file specifies requirements, shader source code, draw commands, and pixel probes. It is parsed and executed by the `shader_runner` binary built as part of piglit.

A minimal `.shader_test` that verifies a fragment shader outputs green:

```glsl
# tests/spec/glsl-1.20/execution/green-output.shader_test

[require]
GLSL >= 1.20

[vertex shader]
void main()
{
    gl_Position = gl_Vertex;
}

[fragment shader]
void main()
{
    gl_FragColor = vec4(0.0, 1.0, 0.0, 1.0);
}

[test]
draw rect -1 -1 2 2
probe all rgba 0.0 1.0 0.0 1.0
```

A test for a specific extension that may not be present:

```glsl
# tests/spec/arb_derivative_control/execution/dfdx-coarse.shader_test

[require]
GL >= 3.0
GLSL >= 1.30
GL_ARB_derivative_control

[vertex shader passthrough]

[fragment shader]
#extension GL_ARB_derivative_control : require
uniform float x;
out vec4 color;
void main()
{
    float d = dFdxCoarse(x);
    color = (abs(d - 1.0) < 0.01) ? vec4(0,1,0,1) : vec4(1,0,0,1);
}

[test]
uniform float x gl_FragCoord.x
draw rect -1 -1 2 2
probe all rgba 0.0 1.0 0.0 1.0
```

Key sections:

| Section | Purpose |
|---------|---------|
| `[require]` | Minimum GL version, GLSL version, and required extensions. If requirements are not met, the test is reported as `skip`. |
| `[vertex shader]` / `[fragment shader]` | GLSL source for the named pipeline stage. |
| `[vertex shader passthrough]` | Synthetic built-in that emits `gl_Position = gl_Vertex` without custom source. |
| `[geometry shader]`, `[tessellation control shader]`, `[tessellation evaluation shader]` | Additional pipeline stages. |
| `[test]` | Draw and probe commands executed sequentially. |

Common `[test]` commands:

```
# Draw a full-viewport rectangle
draw rect -1 -1 2 2

# Set a uniform
uniform float threshold 0.5
uniform vec3 color 0.0 1.0 0.0

# Probe a specific pixel (x, y)
probe rgb 64 64 0.0 1.0 0.0

# Probe all pixels
probe all rgba 0.0 1.0 0.0 1.0

# Probe relative position (0.0-1.0 fraction of framebuffer)
relative probe rgba (0.5, 0.5) (0.0, 1.0, 0.0, 1.0)

# Test that the shader fails to link
link error
```

[Source: piglit tests/spec/arb_derivative_control/](https://github.com/linyaa-kiwi/piglit/blob/master/tests/spec/arb_derivative_control/execution/dfdx-coarse.shader_test)

### 2.4 Contributing a piglit Test

The convention for bug regression tests is to name the file after the bug report: `tests/bugs/fdo12345.shader_test` for freedesktop bug #12345 or `tests/bugs/gitlab-mesa-1234.shader_test` for Mesa GitLab issue #1234. The file should include a comment at the top identifying the bug and what it tests.

Test submission goes alongside the driver fix in the same Mesa merge request — or as a standalone MR to the `mesa/piglit` repository on freedesktop GitLab. Mesa's CI automatically includes piglit in its test matrix, so a new test immediately runs in CI and demonstrates the fix.

---

## 3. dEQP and VK-GL-CTS

dEQP (drawElements Quality Program) is the Khronos-maintained conformance test suite for OpenGL ES, OpenGL, Vulkan, and EGL. It lives in the **VK-GL-CTS** repository ([https://github.com/KhronosGroup/VK-GL-CTS](https://github.com/KhronosGroup/VK-GL-CTS)) and is the definitive test suite for conformance certification. Chapter 31 covers the theory of Khronos conformance certification; this chapter focuses on how dEQP is used day-to-day in Mesa's CI.

### 3.1 Test Modules

dEQP organises tests into modules, each compiled to a separate binary:

| Binary | Module prefix | Coverage |
|--------|--------------|---------|
| `deqp-vk` | `dEQP-VK` | Vulkan |
| `glcts` | `dEQP-GL45` | OpenGL 4.5 |
| `deqp-gles2` | `dEQP-GLES2` | OpenGL ES 2.0 |
| `deqp-gles3` | `dEQP-GLES3` | OpenGL ES 3.0 |
| `deqp-gles31` | `dEQP-GLES31` | OpenGL ES 3.1 |
| `deqp-egl` | `dEQP-EGL` | EGL |

Test names use a hierarchical dot-separated scheme that encodes the specification area:

```
dEQP-VK.api.smoke.create_sampler
dEQP-VK.pipeline.multisample.explicit_reconstructed_sample_locations_dynamic_set
dEQP-VK.spirv_assembly.instruction.compute.saturated_conversion.sf32_to_ui16
```

Tests are implemented as C++ subclasses of `tcu::TestCase`. The framework calls `init()` once per test, then iterates `iterate()` until it returns `tcu::TestNode::STOP`. This design allows multi-frame tests while keeping the runner in control of execution flow.

[Source: VK-GL-CTS framework/common/tcuTestCase.hpp](https://github.com/KhronosGroup/VK-GL-CTS/blob/main/framework/common/tcuTestCase.hpp)

### 3.2 Running dEQP Locally

Build VK-GL-CTS with CMake:

```bash
git clone https://github.com/KhronosGroup/VK-GL-CTS.git
cd VK-GL-CTS
python3 external/fetch_sources.py
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DDEQP_TARGET=x11_vulkan
cmake --build build -j$(nproc)
```

Run Vulkan smoke tests (fast sanity check):

```bash
./build/external/vulkancts/modules/vulkan/deqp-vk \
  --deqp-case="dEQP-VK.api.smoke.*" \
  --deqp-log-filename=smoke-results.qpa
```

Run a full Vulkan API group:

```bash
./build/external/vulkancts/modules/vulkan/deqp-vk \
  --deqp-caselist-file=external/vulkancts/mustpass/main/vk-default.txt \
  --deqp-log-images=disable \
  --deqp-log-shader-sources=disable \
  --deqp-log-filename=TestResults.qpa
```

Run only tests matching a pattern using wildcard expansion in the case name:

```bash
./build/external/vulkancts/modules/vulkan/deqp-vk \
  --deqp-case="dEQP-VK.memory.*"
```

Run a specific fraction for parallelism (split into 4 parts, run part 1):

```bash
./build/external/vulkancts/modules/vulkan/deqp-vk \
  --deqp-caselist-file=vk-default.txt \
  --deqp-fraction=0,4
```

[Source: VK-GL-CTS external/vulkancts/README.md](https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/vulkancts/README.md)

### 3.3 QPA Log Files and Result States

dEQP writes results to `.qpa` XML log files. Each test case entry looks like:

```xml
<TestCaseResult Version="0.3.3" CasePath="dEQP-VK.api.smoke.create_sampler" 
                StartTime="1234567890">
  <Text>Sampler created successfully</Text>
  <Result StatusCode="Pass">Pass</Result>
</TestCaseResult>
```

The allowed result status codes are:

| Status | Meaning |
|--------|---------|
| `Pass` | Test passed. |
| `Fail` | Test failed — output did not match expected result. |
| `NotSupported` | Feature required by test not supported by this implementation; not a failure. |
| `QualityWarning` | Implementation produced acceptable but sub-optimal results (e.g., precision lower than recommended but within spec). Does not fail conformance. |
| `CompatibilityWarning` | Non-blocking deviation noted for record. |
| `InternalError` | Test framework error, not a driver failure per se. |
| `Crash` | Process crashed during the test. |

Conformance submissions allow `NotSupported`, `QualityWarning`, and `CompatibilityWarning` in addition to `Pass`. Any `Fail`, `Crash`, or `InternalError` in the mustpass list is a conformance failure.

Parse qpa logs with the included `testlog-to-csv` tool:

```bash
python3 executor/tools/testlog-to-csv.py TestResults.qpa > results.csv
```

Or use the **cherry** web viewer (from `external/deqp-coverage-tool/`) for an interactive browser-based result explorer.

### 3.4 Caselist Files and Mustpass

The mustpass list for official conformance submissions is at `external/vulkancts/mustpass/main/vk-default.txt`. This file lists all test cases that must pass for a Vulkan conformance submission. Mesa's CI does not run the full mustpass in every MR; instead, it runs driver-specific subsets maintained in `src/<driver>/ci/`:

```
src/amd/ci/deqp-radv-navi10-fails.txt    # known failures on Navi10
src/amd/ci/deqp-radv-vega10-flakes.txt   # intermittent tests on Vega10
src/freedreno/ci/deqp-a630-fails.txt     # expected failures on Adreno 630
```

[Source: Mesa src/amd/ci/](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/ci)

---

## 4. deqp-runner — Mesa's Parallel CTS Executor

deqp-runner ([https://gitlab.freedesktop.org/mesa/parallel-deqp-runner](https://gitlab.freedesktop.org/mesa/parallel-deqp-runner)) is a Rust tool that wraps dEQP, piglit, IGT GPU Tools, skqp, fluster, vkd3d-proton, and gtest to parallelise execution, handle known failures, detect intermittent tests, and emit CI-friendly output. As of 2026, the current release is 0.23.2 (February 2026). [Source: crates.io/crates/deqp-runner](https://crates.io/crates/deqp-runner)

The original motivation for rewriting the parallel runner in Rust was that the C++ predecessor had stability issues with crash handling and lacked per-test timeout enforcement. The Rust version tracks crashes separately from fails and uses per-thread Vulkan shader caches to speed up Vulkan CI runs.

### 4.1 Core Features

- **CPU-parallel test execution**: runs multiple dEQP or piglit instances simultaneously, one per allocated CPU core.
- **Baseline (expected failures)**: a CSV file documents which tests are expected to fail; regressions against the baseline fail the CI job.
- **Flake detection**: tests that unexpectedly fail are automatically re-run; persistent failures are counted as real failures, intermittent ones are counted as flakes and do not block CI.
- **Sharding**: the test list is split across N parallel CI jobs so the total wall time scales with hardware availability.
- **JUnit XML output**: emits results in JUnit format that GitLab CI parses to display per-test pass/fail in the pipeline UI.
- **Crash isolation**: each test runs in a subprocess; a GPU hang that kills the test process is reported as a crash rather than silently dropping the test.
- **Multiple backends**: the same tool wraps dEQP, piglit, IGT, gtest, and fluster; CI jobs for different test suites use the same runner infrastructure.

Install via Cargo:

```bash
cargo install deqp-runner
```

Or use the pre-built binary distributed in Mesa's CI Docker images.

### 4.2 Basic Usage

Run a dEQP caselist in parallel across all CPUs:

```bash
deqp-runner run \
  --deqp ./deqp-vk \
  --caselist vk-smoke.txt \
  --output results/ \
  --jobs $(nproc) \
  --timeout 60
```

Run a piglit suite:

```bash
piglit-runner run \
  --piglit-folder /path/to/piglit \
  --profile quick \
  --output results/ \
  --jobs 8
```

Run a suite defined in a TOML file (used in Mesa CI for complex multi-caselist configurations):

```bash
deqp-runner suite \
  --suite src/amd/ci/deqp-radv-navi10.toml \
  --output results/ \
  --jobs 16
```

The TOML suite file specifies which dEQP binary, caselist, baseline, and skip files to use for a given hardware configuration.

### 4.3 Baselines and Expected Failures

The `--baseline` flag accepts a CSV file documenting tests that are expected to fail on the current hardware:

```
# src/amd/ci/deqp-radv-navi10-fails.txt
dEQP-VK.pipeline.monolithic.blend.dual_source.format.b10g11r11_ufloat_pack32.states.color_write_mask_none,Fail
dEQP-VK.memory.requirements.core.buffer.usage_flags.8.tiling_optimal,NotSupported
```

If a test listed as `Fail` in the baseline starts passing, deqp-runner marks that as an "unexpectedly passing" result — a regression in reverse that must be reviewed. This catches situations where a test was silently being skipped but should now run.

If a test not listed in the baseline starts failing, deqp-runner fails the CI job and reports the new failure, prompting the MR author to either fix the driver or document the new expected failure.

### 4.4 Flake Handling

The `--flakes` flag accepts a file of regex patterns matching test names known to be intermittently unreliable:

```
# src/amd/ci/deqp-radv-vega10-flakes.txt
dEQP-VK.synchronization.*
dEQP-VK.robustness.robustness2.*
```

Tests matching a flake pattern that fail are automatically re-run once. If the re-run passes, the result is counted as a flake and the CI job continues. If it fails again, it counts as a real failure. This prevents intermittent GPU scheduling variations from blocking unrelated MRs.

Driver maintainers must keep their flake lists minimal. Mesa's CI documentation requires that each driver's test farm produce spurious failures no more than once per week, to avoid blocking other contributors' work. [Source](https://docs.mesa3d.org/ci/index.html)

### 4.5 Sharding for CI

A full dEQP-VK run takes many hours even with parallelism on a single machine. Mesa's CI shards the test list across multiple CI jobs using the `--fraction` flag concept — each CI job processes a different slice of the caselist.

In Mesa's `.gitlab-ci.yml`, the `parallel:` keyword spawns multiple identical jobs, each receiving a different `CI_NODE_INDEX` / `CI_NODE_TOTAL` pair. The runner script uses these values to select the appropriate fraction:

```yaml
radv-navi10-deqp-vk:
  extends: .deqp-test
  parallel: 8
  variables:
    DEQP_VK_CASELIST: vk-default.txt
    GPU_VERSION: radv-navi10
```

Each of the 8 parallel jobs processes 1/8 of the caselist independently and uploads its results to MinIO object storage. A final aggregation job collects all fragments and produces the combined JUnit report.

[Source: Mesa .gitlab-ci/deqp-runner.sh](https://cgit.freedesktop.org/mesa/mesa/tree/.gitlab-ci/deqp-runner.sh)

---

## 5. Mesa's GitLab CI Pipeline

Mesa's primary CI lives at [https://gitlab.freedesktop.org/mesa/mesa](https://gitlab.freedesktop.org/mesa/mesa) and runs on every merge request and every post-merge push to `main`. The `.gitlab-ci.yml` at the repository root orchestrates the full pipeline; it includes driver-specific sub-files via `include: local:` directives.

### 5.1 Pipeline Stages

The pipeline defines stages that execute sequentially:

```yaml
stages:
  - sanity
  - container
  - container-2
  - git-archive
  - meson-x86_64
  - build-misc
  - amd
  - intel
  - arm
  - broadcom
  - freedreno
  - software-renderer
  - layered-backends
  - deploy
  - success
```

Stages within a group run in parallel where jobs have no inter-dependencies. The driver-specific stages (`amd`, `intel`, `arm`, etc.) contain test jobs that require the `meson-x86_64` build artifacts from the build stage. [Source: Igalia/mesa .gitlab-ci.yml](https://github.com/Igalia/mesa/blob/main/.gitlab-ci.yml)

### 5.2 Build Jobs

The build stage compiles Mesa for multiple configurations:

```yaml
.meson-build:
  stage: meson-x86_64
  image: $MESA_DEBIAN_IMAGE
  script:
    - meson setup build/
        -Dbuildtype=debugoptimized
        -Dgallium-drivers=softpipe,llvmpipe,radeonsi
        -Dvulkan-drivers=amd,intel,swrast
        -Dglx=dri
    - ninja -C build -j$(($(nproc) + 2))
  artifacts:
    paths:
      - build/src/
    expire_in: 1 week
```

The built Mesa `.so` files are uploaded as GitLab CI artifacts and downloaded by the subsequent test jobs. Test machines do not need a compiler — they only need the pre-built Mesa artifacts plus the driver under test.

Build configurations include:
- `x86_64` with GCC and Clang (to catch warnings that differ between compilers)
- `arm64` cross-compiled with `aarch64-linux-gnu-gcc`
- `armhf` cross-compiled for 32-bit ARM embedded devices
- Windows builds (for Mesa's D3D12/Zink backend on WSL)

A "structured tagging" scheme (`MESA_IMAGE_TAG` variables) ensures Docker images are rebuilt only when their build scripts change, avoiding unnecessary container rebuilds on every pipeline run. [Source](https://docs.mesa3d.org/ci/docker.html)

### 5.3 Driver-Specific CI Includes

Each driver maintains its own CI configuration in `src/<driver>/ci/gitlab-ci.yml`, included from the main `.gitlab-ci.yml`:

```yaml
include:
  - local: 'src/amd/ci/gitlab-ci.yml'
  - local: 'src/freedreno/ci/gitlab-ci.yml'
  - local: 'src/gallium/drivers/iris/ci/gitlab-ci.yml'
  - local: 'src/panfrost/ci/gitlab-ci.yml'
  # ... one include per driver
```

Driver CI files define rules that scope test jobs to relevant changes. A test job for RADV only triggers when files under `src/amd/`, `src/vulkan/`, or shared NIR paths change:

```yaml
radv-navi10-vk-deqp:
  extends: .deqp-test-radv
  rules:
    - changes:
        - src/amd/**/*
        - src/compiler/nir/**/*
        - src/vulkan/**/*
      when: on_success
    - when: never
```

This scoping prevents, say, a change to the Panfrost driver from triggering expensive RADV tests that cannot possibly be affected. It is the primary mechanism that keeps MR pipeline times under 20 minutes despite Mesa supporting dozens of drivers. [Source: Mesa src/amd/ci/](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/ci)

### 5.4 Test Jobs and Hardware Runners

Test jobs use GitLab runner tags to target specific hardware. Machines running GPU test jobs are tagged with identifiers like:

```
mesa-ci-x86-64-lava-amdgpu-raven
mesa-ci-aarch64-lava-panfrost-t860
mesa-ci-x86-64-radeonsi-polaris10
```

Each tagged runner is a physical machine (or a LAVA device) with the named GPU. Job tagging routes test execution to the appropriate hardware:

```yaml
.radv-rules:
  tags:
    - mesa-ci-x86-64-radeonsi-navi10
```

Mesa CI farms include:
- **x86_64 Intel** (Kaby Lake, Ice Lake, Tiger Lake, Arc Alchemist) via Collabora's lab
- **x86_64 AMD** (Vega10, Raven/Picasso, Navi10, Navi31) via Collabora and AMD's CI lab
- **ARM64 Qualcomm** (Adreno 630 on SM845, A7xx on SM8350) via LAVA
- **ARM64 Arm Mali** (T860 on RK3399, G57 on RK3588) via LAVA (Panfrost/PanVK)
- **RISC-V** (experimental; coverage varies by driver and available hardware)

Results are uploaded to MinIO object storage (formerly at `minio-packet.freedesktop.org`; the Equinix infrastructure that hosted this endpoint shut down in April 2025 and has been migrated to successor hosting) where they are accessible to downstream aggregation jobs and artifact browsers.

### 5.5 Merge Bot and Queue Management

Mesa uses the **Marge** merge bot ([https://gitlab.freedesktop.org/marge-bot/marge-bot](https://gitlab.freedesktop.org/marge-bot/marge-bot)) to serialise merge request integration. When a maintainer approves an MR and assigns it to Marge, the bot:

1. Rebases the MR onto current `main`.
2. Runs the full CI pipeline on the rebased branch.
3. If all required jobs pass, merges to `main` automatically.
4. If any required job fails, reports the failure and leaves the MR unmerged.

Contributors can monitor the Marge queue via `bin/ci/marge_queue.sh` and run targeted CI jobs locally using `bin/ci/ci_run_n_monitor.py --regex "radv"` to filter by job name pattern. [Source](https://docs.mesa3d.org/ci/index.html)

---

## 6. LAVA — Hardware-in-the-Loop CI for ARM GPUs

LAVA (Linaro Automation and Validation Architecture) ([https://lava.readthedocs.io/](https://lava.readthedocs.io/)) provides remote-controlled, power-cycled hardware testing for embedded and mobile GPUs. Mesa uses LAVA primarily for ARM GPU CI — Panfrost (Arm Mali), PanVK (Mali Vulkan), Turnip (Qualcomm Adreno), etnaviv (Vivante), Lima (Arm Mali older cores), and V3D (Broadcom VideoCore).

### 6.1 LAVA Architecture

LAVA consists of:

- **LAVA server**: a Django web application that schedules jobs, tracks device state, and exposes a REST/XML-RPC API.
- **LAVA dispatcher**: a daemon running on each worker host that communicates with the LAVA server, controls DUT power via a PDU (power distribution unit), and monitors the device console via UART.
- **DUT (Device Under Test)**: the physical ARM board with the GPU to be tested. The dispatcher power-cycles the board, deploys a kernel and ramdisk over TFTP or fastboot, and reads test output from the UART console.

A LAVA job is a YAML document submitted to the server:

```yaml
# Simplified LAVA job definition for Mesa CI
job_name: mesa-panfrost-rk3399-deqp
device_type: rk3399

timeouts:
  job:
    minutes: 30
  action:
    minutes: 5

actions:
  - deploy:
      to: tftp
      kernel:
        url: https://storage.freedesktop.org/lava-kernel/kernel-rk3399.tar.gz
        type: image
      ramdisk:
        url: https://storage.freedesktop.org/lava-rootfs/mesa-arm64.tar.gz
        compression: gz

  - boot:
      method: u-boot
      commands: ramdisk
      auto_login:
        login_prompt: "login:"
        username: root
      prompts:
        - "root@lava:"

  - test:
      definitions:
        - repository: inline
          from: inline
          name: mesa-deqp
          path: inline/test.yaml
          definitions:
            - run:
                steps:
                  - export DISPLAY=:0
                  - /mesa-ci/deqp-runner run
                      --deqp /deqp-vk
                      --caselist /ci/panfrost-t860.txt
                      --output /results/
                      --jobs 4
```

After the job completes, the dispatcher uploads results artifacts. The LAVA server tracks pass/fail and board health history. [Source: LAVA documentation](https://lava.readthedocs.io/)

### 6.2 How Mesa Uses LAVA

Mesa's LAVA integration is coordinated through `lavacli` (the LAVA command-line client) running inside Docker containers on GitLab CI worker machines. The workflow:

1. A GitLab CI job tagged `mesa-ci-x86-64-lava-$DEVICE_TYPE` starts on the LAVA gateway machine.
2. The job spawns a Docker container with `lavacli` and authenticates using a token stored in `lavacli.yaml`.
3. A LAVA job YAML is generated and submitted via `lavacli jobs submit`.
4. The CI job polls the LAVA server for job status.
5. When the LAVA job completes, test result artifacts are retrieved from MinIO.
6. The GitLab CI job parses results and reports pass/fail to the pipeline.

Authentication tokens are stored in `/etc/gitlab-runner/config.toml` volume mounts, keeping credentials out of the Mesa branch history.

Mesa tests two LAVA labs:
- **Collabora's LAVA lab**: the primary ARM GPU test farm, hosting Panfrost (RK3399, RK3588), Turnip (SM845 Adreno 630), and V3D (Raspberry Pi 4).
- **Lima lab**: for Lima driver testing on older Mali hardware.

LAVA includes automatic **health checks**: before accepting a test job, the dispatcher runs a minimal boot-and-smoke-test sequence. If a board fails its health check, LAVA marks it `offline` and stops dispatching jobs to it, preventing test results from being corrupted by broken hardware. When a board hangs during a test, the dispatcher power-cycles it via the PDU, recovers it to a clean state, and marks the LAVA job as failed due to infrastructure error rather than a test failure.

An nginx pass-through HTTP cache runs on the LAVA gateway to reduce redundant downloads of the Mesa kernel and rootfs images, which are large and change infrequently. [Source: Mesa LAVA CI docs](https://docs.mesa3d.org/ci/LAVA.html)

### 6.3 CI-tron: The Next-Generation Hardware CI Gateway

**CI-tron** ([https://gitlab.freedesktop.org/gfx-ci/ci-tron](https://gitlab.freedesktop.org/gfx-ci/ci-tron)) is a bare-metal automated CI system for Linux driver development that serves as an alternative to LAVA for certain Mesa hardware farms. It acts as a CI gateway service with components responsible for DUT orchestration and management.

CI-tron's design principles prioritise flexibility (no lock-in to specific CI software), simplicity (auto-discovery of DUTs), and maintainability (minimal administrative burden). Physically, a CI-tron deployment consists of a gateway server connected to a private network switch, a switchable power delivery unit (PDU) for DUT power control, a USB hub for serial port connections, and one or more DUT machines.

In 2025, Equinix shut down its operations supporting Mesa's x86_64 bare-metal CI cluster (which had operated for roughly five years). This closure accelerated the migration of x86_64 hardware CI from Mesa's previous bare-metal CI scripts toward CI-tron and LAVA-based solutions. As of 2026, Mesa's bare-metal CI scripts have been formally deprecated — the documentation notes "Bare-metal support is being removed from Mesa, and adding new devices is no longer supported," with new hardware farms directed to CI-tron or LAVA instead. [Source](https://docs.mesa3d.org/ci/bare-metal.html)

---

## 7. Shader Compile Testing with shader-db

**shader-db** ([https://gitlab.freedesktop.org/mesa/shader-db](https://gitlab.freedesktop.org/mesa/shader-db)) is a corpus of real-world GLSL shaders captured from games and applications, used to measure Mesa's shader compiler throughput and code quality. Unlike dEQP or piglit, shader-db never renders a single pixel — it tests only the compiler, making it runnable without a GPU on any machine with Mesa built.

### 7.1 What shader-db Measures

For each shader in the corpus, shader-db records compiler statistics per driver backend:

| Metric | Description |
|--------|------------|
| Instructions | Total native instructions emitted |
| Sends | Memory send instructions (expensive on some architectures) |
| Spills | Register spill count (spills to memory degrade performance) |
| Loops | Number of loop constructs (affects branch prediction) |
| Cycles | Estimated execution cycles (where the backend models this) |
| Register pressure | Peak live registers during compilation |

The statistics are driver-specific. `i965` (Intel) reports `instructions/sends/loops/cycles`; `radeonsi` reports `sgprs/vgprs/instructions`; `etnaviv` reports `code_size`. The stats are printed via the `MESA_SHADER_STATS` mechanism or the driver's own statistics infrastructure.

### 7.2 Capturing Shaders and Running the Suite

To capture shaders from a running game or application, set `MESA_SHADER_CAPTURE_PATH`:

```bash
mkdir captured-shaders/
MESA_SHADER_CAPTURE_PATH=$(pwd)/captured-shaders/ \
  LIBGL_DRIVERS_PATH=/path/to/mesa/lib \
  ./my-game
```

Mesa writes one `.shader_test` file per linked shader program into the capture directory. These files can be contributed to shader-db to expand the corpus.

To run shader-db against a Mesa build:

```bash
git clone https://gitlab.freedesktop.org/mesa/shader-db.git
cd shader-db

# Run for i965 (Intel GL driver)
LIBGL_ALWAYS_SOFTWARE=0 \
  LIBGL_DRIVERS_PATH=/path/to/mesa/build/src/mesa/drivers/dri/i965 \
  ./run -j$(nproc) shaders/ > stats-after.txt
```

The `run` script is a parallel wrapper that invokes the Mesa GL state tracker to compile each shader and collect statistics. [Source: mesa/shader-db README](https://cgit.freedesktop.org/mesa/shader-db/tree/README.md)

### 7.3 Before/After Comparisons

The workflow for evaluating a compiler patch:

```bash
# Build Mesa on the base branch
git stash
meson setup build-before/ -Dbuildtype=release
ninja -C build-before/ -j$(nproc)

# Run shader-db on base
LIBGL_DRIVERS_PATH=build-before/... ./run shaders/ > stats-before.txt

# Apply the patch
git stash pop
meson setup build-after/ -Dbuildtype=release
ninja -C build-after/ -j$(nproc)

# Run shader-db on the patched build
LIBGL_DRIVERS_PATH=build-after/... ./run shaders/ > stats-after.txt

# Diff
./compare stats-before.txt stats-after.txt
```

Example output from the `compare` script:

```
total instructions in shared programs: 1,781,593 -> 1,734,957 (-2.62%)
instructions in affected programs:     1,238,390 -> 1,191,754 (-3.77%)
helped: 12,782
HURT:   0
```

Mesa commit messages for compiler improvements conventionally include this stats block as a "shader-db before/after" result to document the improvement. Reviewers use it as a quick sanity-check that the optimisation actually helps and does not regress other shaders (HURT > 0 warrants investigation). [Source: Igalia shader-db blog](https://blogs.igalia.com/apinheiro/2015/09/optimizing-shader-assembly-instruction-on-mesa-using-shader-db/)

Mesa has also introduced a unified **shader statistics framework** (landed circa Mesa 24.x) that exposes per-backend statistics through a common XML-described interface, enabling cross-driver comparison and future CI integration of shader-db runs into the GitLab pipeline. [Source: Phoronix shader stats framework](https://www.phoronix.com/news/Mesa-Shader-Stats-Framework)

### 7.4 Shader Reduction

When shader-db (or a CI run) finds a shader that triggers a compiler bug, the next step is reduction — minimising the shader to the smallest form that still reproduces the bug. Tools:

- **glsl-reduce**: Mesa's own GLSL reduction tool, using creduce-style passes.
- **spirv-reduce**: for SPIR-V shaders, reduces a SPIR-V binary while preserving the bug (`spirv-reduce --target-env vulkan1.3 bad.spv --interestingness-test ./check.sh`). Part of the SPIRV-Tools suite.
- **spirv-fuzz**: generates semantics-preserving transformations of valid SPIR-V to stress-test shader compiler correctness.

A reduced shader is far easier to debug and attach to a bug report than the original 10,000-line game shader.

---

## 8. Trace-Based Render Comparison Testing

Shader-compile testing and dEQP conformance tests cover specification compliance, but they do not test whether Mesa correctly renders real application workloads. Trace-based testing fills this gap by replaying captured API traces and comparing rendered frames against known-good reference checksums.

### 8.1 apitrace and Piglit's Replayer

Mesa uses two capture-and-replay tools:

- **apitrace** ([https://apitrace.github.io/](https://apitrace.github.io/)): captures OpenGL, OpenGL ES, and EGL API call sequences. Replay with: `apitrace replay -w <trace.trace>` or headlessly for CI with piglit's replayer module.
- **renderdoc** ([https://renderdoc.org/](https://renderdoc.org/)): captures Vulkan and D3D API call sequences including resource data.

Piglit's `replayer` module (in `replayer/`) wraps both tools. Given a trace file and a reference checksum, it replays the trace, dumps the rendered frame as a PNG, and computes a checksum:

```bash
PIGLIT_REPLAY_DEVICE_NAME=radv-navi10 \
PIGLIT_REPLAY_DESCRIPTION_FILE=src/amd/ci/traces-radv.yml \
  ./piglit run -l verbose --timeout 300 -j4 replay ~/results/
```

If the rendered frame's checksum does not match the stored reference, the test fails and the piglit summary shows a visual diff. [Source: Mesa local-traces docs](https://docs.mesa3d.org/ci/local-traces.html)

### 8.2 Driver Trace Configurations

Each driver maintains a YAML file listing which traces to run and what the expected checksums are:

```yaml
# src/amd/ci/traces-radv.yml
traces:
  - path: games/gtavc/frame1234.trace
    expectations:
      - device: radv-navi10
        checksum: a3f1b2c9d4e5f6a7b8c9d0e1f2a3b4c5
  - path: games/portal2/frame100.rdc
    expectations:
      - device: radv-navi10
        checksum: 1234567890abcdef1234567890abcdef
    restricted: true   # non-redistributable; skip if not available
```

Traces marked `restricted: true` are stored in a private object store accessible only to CI runners with appropriate credentials. Contributors without access have these tests skipped rather than failed, preventing them from being blocked by traces they cannot obtain. [Source: Mesa CI docs — application traces](https://docs.mesa3d.org/ci/index.html)

### 8.3 Running Traces Locally

To simulate the CI environment for traces:

```bash
# Download the public traces database
git clone https://gitlab.freedesktop.org/gfx-ci/tracie/traces-db.git

# Set the device name to match your hardware
export PIGLIT_REPLAY_DEVICE_NAME=radv-navi10
export PIGLIT_REPLAY_DESCRIPTION_FILE=src/amd/ci/traces-radv.yml

# Override Mesa driver path if needed
export LIBGL_DRIVERS_PATH=/path/to/mesa/build/src/gallium/targets/dri

# Run traces
./piglit run -l verbose --timeout 300 -j4 replay ~/traces-results/
./piglit summary html --overwrite traces-summary/ ~/traces-results/
```

For traces that produce slightly different pixel values due to floating-point scheduling differences, the `PIGLIT_REPLAY_EXTRA_ARGS=--tolerance=1` flag loosens the comparison threshold.

Large game traces may be trimmed using `gltrim` (part of apitrace) to extract only the frames of interest, reducing storage requirements and replay time in CI. [Source: Collabora apitrace trimming blog](https://www.collabora.com/news-and-blog/blog/2021/02/01/trimming-apitrace-workload-captures-for-better-mesa-testing/)

---

## 9. Writing Tests for Mesa

### 9.1 When to Add a Test

Mesa's contribution guidelines require that every bug fix be accompanied by a regression test. The rationale is simple: without a test, the bug will silently re-emerge whenever someone modifies the relevant code path. The test serves as executable documentation of the correct behaviour.

Tests should be added to the appropriate suite:
- **piglit `.shader_test`**: for GLSL compiler bugs, OpenGL specification compliance, extension behaviour — anything expressible as a shader plus a pixel probe.
- **piglit C test**: for bugs requiring complex GL state setup (e.g., FBO configurations, multi-draw sequences, synchronisation scenarios) that cannot be expressed in the text format.
- **dEQP**: for Vulkan bugs, since dEQP-VK is the primary Vulkan test suite. If the test exercises a Vulkan feature that dEQP already covers with existing infrastructure, extend dEQP. If it is a Mesa-specific regression, a simpler piglit test (using `VK_` extensions) may be more practical.
- **IGT GPU Tools**: for kernel DRM driver bugs (KMS, GEM, syncobj, DMA-BUF). See Chapter 31 for IGT test writing.

### 9.2 Writing a C piglit Test

A C piglit test links against `libpiglitutil` and uses the piglit API:

```c
/* tests/spec/arb_texture_buffer_object/texture-buffer-size-clamping.c */
#include "piglit-util-gl.h"

PIGLIT_GL_TEST_CONFIG_BEGIN
    config.supports_gl_core_version = 31;
    config.window_visual = PIGLIT_GL_VISUAL_RGBA | PIGLIT_GL_VISUAL_DOUBLE;
PIGLIT_GL_TEST_CONFIG_END

void piglit_init(int argc, char **argv)
{
    piglit_require_extension("GL_ARB_texture_buffer_object");
}

enum piglit_result piglit_display(void)
{
    GLuint buf, tex;
    GLint max_size;
    float green[] = {0.0, 1.0, 0.0, 1.0};
    bool pass = true;

    glGetIntegerv(GL_MAX_TEXTURE_BUFFER_SIZE, &max_size);

    /* Create a buffer larger than the implementation maximum */
    glGenBuffers(1, &buf);
    glBindBuffer(GL_TEXTURE_BUFFER, buf);
    glBufferData(GL_TEXTURE_BUFFER, (GLsizeiptr)max_size * 2 * sizeof(float),
                 NULL, GL_STATIC_DRAW);

    glGenTextures(1, &tex);
    glBindTexture(GL_TEXTURE_BUFFER, tex);
    glTexBufferARB(GL_TEXTURE_BUFFER, GL_R32F, buf);

    /* The implementation must clamp the accessible size to max_size */
    GLint accessible;
    glGetTexLevelParameteriv(GL_TEXTURE_BUFFER, 0,
                             GL_TEXTURE_BUFFER_SIZE, &accessible);
    pass = pass && (accessible <= max_size);

    glDeleteTextures(1, &tex);
    glDeleteBuffers(1, &buf);

    return pass ? PIGLIT_PASS : PIGLIT_FAIL;
}
```

Key API functions:

| Function | Purpose |
|----------|---------|
| `piglit_require_extension(name)` | Skip the test if the named extension is absent. |
| `piglit_draw_rect(x, y, w, h)` | Draw a rectangle in normalised device coordinates. |
| `piglit_probe_pixel_rgba(x, y, expected)` | Sample pixel at (x,y) and compare to expected RGBA. |
| `piglit_probe_rect_rgba(x, y, w, h, expected)` | Sample all pixels in a rectangle. |
| `piglit_check_gl_error(expected)` | Verify that glGetError() returns the expected code. |

[Source: piglit tests/util/piglit-util-gl.h](https://cgit.freedesktop.org/piglit/tree/tests/util/piglit-util-gl.h)

### 9.3 Writing a `.shader_test` File

For most GLSL compiler bugs, the `.shader_test` format is simpler and sufficient. A template for a new bug regression test:

```glsl
# Regression test for Mesa GitLab issue #1234:
# NIR constant folding incorrectly folds a mod(x, 0) to 0 instead of NaN/undefined.
# Fixed by commit abc123def456.
#
# Tests that mod(x, 0.0) does not produce the GL_INVALID_VALUE error
# (GLSL spec does not define the result, but Mesa was producing a wrong value
# that diverged from the reference implementation on this specific pattern).

[require]
GLSL >= 1.30

[vertex shader passthrough]

[fragment shader]
#version 130
uniform float divisor;
out vec4 color;

void main()
{
    float x = 3.5;
    /* Before the fix, NIR folded this to 0.0 incorrectly */
    float result = mod(x, divisor);
    /* With divisor=1.0, result should be 0.5 */
    color = (abs(result - 0.5) < 0.01)
          ? vec4(0.0, 1.0, 0.0, 1.0)
          : vec4(1.0, 0.0, 0.0, 1.0);
}

[test]
uniform float divisor 1.0
draw rect -1 -1 2 2
probe all rgba 0.0 1.0 0.0 1.0
```

Name the file according to the specification area: `tests/spec/glsl-1.30/execution/built-in-functions/fs-mod-by-one.shader_test`.

### 9.4 Submitting Tests with Driver Fixes

The review process expects tests to accompany fixes:

1. **Mesa MR**: include both the driver fix and the piglit `.shader_test` (or dEQP extension) in the same MR. Reviewers confirm that the test fails on the unfixed code and passes after the fix.
2. **Standalone piglit MR**: for complex tests that deserve independent review, submit to the `mesa/piglit` GitLab repository separately and link the MR in the Mesa bug.
3. **CI validation**: Mesa's CI runs piglit automatically. The new test will appear in the CI output for the MR and in nightly runs going forward.
4. **dEQP contributions**: if the bug affects Vulkan conformance, consider contributing a test to VK-GL-CTS as well. The Khronos CTS contributor guide describes the submission process, including Python test generators for generating large systematic test groups.

Both the code fix and the test are reviewed by Mesa maintainers. The reviewer checks that the test is correctly named, placed in the right directory, and actually exercises the fixed code path (not just a test that happens to pass even without the fix).

---

## 10. Performance and Microbenchmarks

### 10.1 vkmark and glmark2

**vkmark** ([https://github.com/vkmark/vkmark](https://github.com/vkmark/vkmark)) is a Vulkan benchmark modelled on glmark2. It renders a series of scenes — triangle throughput, texture fillrate, vertex buffer performance, instanced rendering — and reports a composite score and per-scene frame times. It has proven useful as a quick Mesa WSI implementation regression check.

```bash
# Run all scenes with default window size
vkmark

# Run a specific scene
vkmark --scene vertex

# Headless (off-screen) mode
vkmark --winsys-dir /usr/lib/vkmark/ --winsys offscreen
```

**glmark2** ([https://github.com/glmark2/glmark2](https://github.com/glmark2/glmark2)) covers OpenGL ES 2.0 workloads: texture sampling, buffer streaming, shadow mapping, bump mapping, and compute shaders. It is useful for radeonsi and iris regression testing.

```bash
glmark2 --off-screen   # headless, for CI
glmark2 --benchmark texture:texture-filter=nearest
```

### 10.2 Phoronix Test Suite Integration

The **Phoronix Test Suite** (PTS) ([https://www.phoronix-test-suite.com/](https://www.phoronix-test-suite.com/)) provides a framework for running a large collection of GPU and CPU benchmarks in a reproducible manner. For Mesa development:

```bash
# Install a benchmark suite
phoronix-test-suite install opengl-demos

# Run it and compare against a baseline
phoronix-test-suite benchmark opengl-demos

# Compare system results
phoronix-test-suite compare result-before result-after
```

PTS automates downloading benchmark binaries, running them with standardised settings, and recording results with system information (kernel version, Mesa version, GPU model) for reproducible comparisons. OpenBenchmarking.org hosts a public result database where Mesa performance across driver versions can be tracked. [Source](https://www.phoronix-test-suite.com/)

### 10.3 Performance Regression Tracking

Mesa's CI includes performance-tracking jobs that run vkmark and other lightweight benchmarks and compare frame times against a stored baseline. If frame time exceeds the baseline by more than a configured threshold (typically 5%), the CI job reports a performance regression warning.

The Collabora and Intel teams have deployed Grafana-based performance dashboards that track shader-db statistics and benchmark results over time, plotting instruction counts and frame times per Mesa commit. These dashboards allow maintainers to identify the specific commit that introduced a performance regression and bisect accordingly.

For compiler performance specifically, the shader-db workflow (Section 7) is the primary CI-integrated tool. For runtime performance, trace-based benchmarks that record frame times rather than just correctness checksums complement the functional trace tests described in Section 8.

**skQP** (Skia's cross-platform GPU correctness test suite, run via `skqp-runner` in deqp-runner's multi-backend support) is a separate tool used in some Mesa CI configurations to catch rendering-correctness regressions in the GPU paint pipeline. It focuses on pixel-accuracy of 2D operations rather than performance measurement.

---

## 11. Integrations

**[Chapter 17: Software Renderers](../part-04-mesa-architecture/ch17-software-renderers.md)** — Lavapipe (software Vulkan) and llvmpipe are first-class CI targets. They can run the full dEQP-VK and piglit suites without GPU hardware, making them the first line of defence for API-correctness regressions on any developer machine.

**[Chapter 14: NIR — Mesa's Shader Intermediate Representation](../part-04-mesa-architecture/ch14-nir-shader-ir.md)** — shader-db tests the NIR compiler pipeline. Compiler patches to NIR lowering passes, dead code elimination, and algebraic simplification are validated using shader-db before/after statistics. The `MESA_SHADER_CAPTURE_PATH` mechanism that feeds shader-db works by intercepting NIR at the point where shaders are linked.

**[Chapter 16: Mesa's Vulkan Common Infrastructure](../part-04-mesa-architecture/ch16-mesa-vulkan-common.md)** — VK-GL-CTS (`dEQP-VK`) tests Mesa's Vulkan common layer (`src/vulkan/`) in addition to driver-specific backends. Tests for synchronisation primitives, pipeline caches, descriptor set layouts, and render passes exercise the common Vulkan infrastructure shared by RADV, ANV, Turnip, NVK, and PanVK.

**[Chapter 30: Debugging and Profiling](ch30-debugging-profiling.md)** — apitrace, used in trace-based CI testing (Section 8), is also the primary debugging tool for rendering errors. When a CI trace test fails with a pixel mismatch, the developer loads the trace in apitrace's `qapitrace` GUI to inspect individual draw calls. RenderDoc captures used in Vulkan trace tests integrate directly with RenderDoc's replay API.

**[Chapter 31: Conformance and Regression Testing](ch31-conformance-regression-testing.md)** — the theory chapter for this operational guide. Chapter 31 covers the Khronos Adopter Program, conformance submission requirements, IGT GPU Tools for kernel driver testing, and fuzzing with spirv-fuzz and sanitisers. This chapter covers how those tools run inside Mesa's day-to-day CI pipeline.

**[Chapter 32: Contributing to the Linux Graphics Stack](ch32-contributing.md)** — the contribution workflow chapter. The requirement to include regression tests with every bug fix (Section 9) is enforced during code review as described in Chapter 32. The piglit and dEQP test writing guides in this chapter complement the patch submission and commit message standards in Chapter 32.

**[Chapter 6: ARM and Embedded GPU Drivers](../part-02-gpu-drivers/ch06-arm-embedded-gpu-drivers.md)** and **[Chapter 90: Panfrost, Panthor, and Lima](../part-02-gpu-drivers/ch90-panfrost-panthor-lima.md)** — the ARM GPU drivers (Panfrost, Lima, Turnip, etnaviv, V3D) all rely on LAVA-based CI (Section 6) because their target hardware is embedded ARM SBCs that cannot run a standard GitLab runner. Changes to Panfrost or Turnip trigger LAVA-dispatched CI jobs on physical ARM hardware. Chapter 90 covers the Panfrost driver architecture in depth.

**[Chapter 93: GPU Performance Analysis](ch93-gpu-performance-analysis.md)** — the benchmarking methodology described in Section 10 is the application-level companion to Chapter 93's low-level GPU performance counter and hardware profiler coverage. vkmark and glmark2 provide the workload; Chapter 93's tools (AMD's RGP, Intel's GPA, Qualcomm's Snapdragon Profiler) provide the internal GPU timing visibility needed to diagnose what a performance regression looks like at the hardware level.

---

### References

- piglit repository: [https://gitlab.freedesktop.org/mesa/piglit](https://gitlab.freedesktop.org/mesa/piglit)
- VK-GL-CTS (Khronos Conformance Test Suite): [https://github.com/KhronosGroup/VK-GL-CTS](https://github.com/KhronosGroup/VK-GL-CTS)
- deqp-runner on crates.io: [https://crates.io/crates/deqp-runner](https://crates.io/crates/deqp-runner)
- Mesa CI documentation: [https://docs.mesa3d.org/ci/index.html](https://docs.mesa3d.org/ci/index.html)
- Mesa LAVA CI: [https://docs.mesa3d.org/ci/LAVA.html](https://docs.mesa3d.org/ci/LAVA.html)
- LAVA documentation: [https://lava.readthedocs.io/](https://lava.readthedocs.io/)
- CI-tron: [https://gitlab.freedesktop.org/gfx-ci/ci-tron](https://gitlab.freedesktop.org/gfx-ci/ci-tron)
- shader-db (mirrored): [https://cgit.freedesktop.org/mesa/shader-db/tree/README.md](https://cgit.freedesktop.org/mesa/shader-db/tree/README.md)
- Mesa shader statistics framework: [https://www.phoronix.com/news/Mesa-Shader-Stats-Framework](https://www.phoronix.com/news/Mesa-Shader-Stats-Framework)
- apitrace trimming for CI: [https://www.collabora.com/news-and-blog/blog/2021/02/01/trimming-apitrace-workload-captures-for-better-mesa-testing/](https://www.collabora.com/news-and-blog/blog/2021/02/01/trimming-apitrace-workload-captures-for-better-mesa-testing/)
- Mesa local traces: [https://docs.mesa3d.org/ci/local-traces.html](https://docs.mesa3d.org/ci/local-traces.html)
- piglit `.shader_test` format: [https://blogs.igalia.com/siglesias/2015/05/29/piglit-iii-how-to-write-glsl-shader-tests/](https://blogs.igalia.com/siglesias/2015/05/29/piglit-iii-how-to-write-glsl-shader-tests/)
- shader-db optimisation walkthrough: [https://blogs.igalia.com/apinheiro/2015/09/optimizing-shader-assembly-instruction-on-mesa-using-shader-db/](https://blogs.igalia.com/apinheiro/2015/09/optimizing-shader-assembly-instruction-on-mesa-using-shader-db/)
- vkmark: [https://github.com/vkmark/vkmark](https://github.com/vkmark/vkmark)
- Phoronix Test Suite: [https://www.phoronix-test-suite.com/](https://www.phoronix-test-suite.com/)
- Mesa src/amd/ci/: [https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/ci](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/ci)
- VK-GL-CTS Vulkan README: [https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/vulkancts/README.md](https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/vulkancts/README.md)
