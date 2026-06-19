# Chapter 110: SPIR-V Tooling — spirv-tools, SPIRV-Cross, and the Shader Ecosystem

**Target audiences**: Shader authors, graphics engine developers, compiler engineers, anyone working with the SPIR-V binary format at the tool level — reading or writing modules programmatically, validating correctness, cross-compiling to other shading languages, or integrating SPIR-V ingestion into a Vulkan or OpenCL driver.

---

## Table of Contents

1. [Introduction — SPIR-V as the Lingua Franca](#1-introduction--spir-v-as-the-lingua-franca)
2. [SPIR-V Binary Format](#2-spir-v-binary-format)
3. [Front-End Compilers: glslang and DXC](#3-front-end-compilers-glslang-and-dxc)
4. [spirv-tools: Assembler, Disassembler, Validator](#4-spirv-tools-assembler-disassembler-validator)
5. [spirv-opt: The SPIR-V Optimizer](#5-spirv-opt-the-spir-v-optimizer)
6. [SPIRV-Cross: Transpilation](#6-spirv-cross-transpilation)
7. [spirv-reflect: Pipeline Layout Introspection](#7-spirv-reflect-pipeline-layout-introspection)
8. [SPIR-V in Mesa: spirv_to_nir](#8-spir-v-in-mesa-spirv_to_nir)
9. [Extended SPIR-V: Ray Tracing, Mesh Shaders, Cooperative Matrices](#9-extended-spir-v-ray-tracing-mesh-shaders-cooperative-matrices)
10. [Shader Debugging with SPIR-V](#10-shader-debugging-with-spir-v)
11. [Integrations](#11-integrations)

---

## 1. Introduction — SPIR-V as the Lingua Franca

The Linux graphics stack converges on a single binary intermediate representation for GPU shaders: SPIR-V. Originally designed as an OpenCL 2.1 compute IR in 2015, SPIR-V rapidly became the mandatory shading language for Vulkan 1.0 (2016) and has since colonised every major GPU programming model on the platform:

- **Vulkan** mandates SPIR-V for all shader stages. No GLSL or HLSL source reaches the driver; [VkShaderModule](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#VkShaderModuleCreateInfo) takes a `uint32_t` array.
- **OpenCL** adopted SPIR-V as a portable binary format in OpenCL 2.1, replacing the older SPIR 1.2 format. Mesa's Rusticl consumes SPIR-V directly.
- **WebGPU** (via Dawn/Tint and wgpu/naga) compiles WGSL to SPIR-V on the Vulkan backend before handing it to the driver.
- **CUDA / NVVM** can emit SPIR-V via the PTX IR bridge for interoperability with OpenCL runtimes.
- **OpenGL with ARB_gl_spirv** allows SPIR-V shader modules for OpenGL 4.6 and OpenGL ES 3.2 contexts.

[Source](https://registry.khronos.org/SPIR-V/)

Around this binary format has grown a rich tooling ecosystem. The major components and their roles:

| Tool | Repository | Role |
|------|-----------|------|
| glslang | KhronosGroup/glslang | GLSL/HLSL → SPIR-V front end |
| DXC | microsoft/DirectXShaderCompiler | HLSL → DXIL or SPIR-V |
| Tint | dawn.googlesource.com/dawn | WGSL → SPIR-V / MSL / HLSL |
| clang (LLVM-SPIRV) | llvm/llvm-project | OpenCL C → SPIR-V |
| spirv-as | KhronosGroup/SPIRV-Tools | Text assembly → binary |
| spirv-dis | KhronosGroup/SPIRV-Tools | Binary → human-readable text |
| spirv-val | KhronosGroup/SPIRV-Tools | Specification validator |
| spirv-opt | KhronosGroup/SPIRV-Tools | SPIR-V → optimised SPIR-V |
| spirv-link | KhronosGroup/SPIRV-Tools | Multi-module linker |
| spirv-reduce | KhronosGroup/SPIRV-Tools | Bug reproducer minimiser |
| SPIRV-Cross | KhronosGroup/SPIRV-Cross | SPIR-V → GLSL / HLSL / MSL |
| spirv-reflect | KhronosGroup/SPIRV-Reflect | Lightweight pipeline layout reflection |

This chapter maps the ecosystem in depth: binary format first, then each major tool, then Mesa's ingestion layer, then the extended instruction sets that push SPIR-V into ray tracing, mesh shading, and tensor compute.

---

## 2. SPIR-V Binary Format

Every SPIR-V file or in-memory module begins with the same five 32-bit words (little-endian by convention):

```
Word 0: 0x07230203  — magic number
Word 1: 0x00010600  — version (major.minor: byte 3.2; e.g., 0x00010600 = 1.6)
Word 2: generator magic number (tool-specific; e.g., 0x00080001 = Khronos glslang reference front end, version 1)
Word 3: bound         — one greater than the largest <id> used in the module
Word 4: 0x00000000  — reserved schema word (always zero)
```

[Source: SPIR-V Specification §2.3](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html)

### 2.1 Instruction Encoding

After the five-word header, the rest of the module is a flat stream of instructions. Each instruction occupies one or more 32-bit words. The first word is a packed field:

```
Bits 31–16: word count (total words in this instruction, including this header word)
Bits 15–0:  opcode
```

Subsequent words are operands: type IDs, result IDs, literal integers, literal strings (null-terminated, padded to 32-bit alignment), and enum values. There is no padding between instructions.

For example, a simple floating-point addition:

```spirv
; %float %a %b already declared
%result = OpFAdd %float %a %b
; Encodes as:
;   Word 0: [word_count=5 | OpFAdd=129]  → 0x00050081
;   Word 1: %float (result type ID)
;   Word 2: %result (result ID)
;   Word 3: %a
;   Word 4: %b
```

Result type comes before result ID — the operand order is fixed by the specification for each opcode.

### 2.2 Module Logical Layout

The SPIR-V specification mandates that instructions appear in a fixed order. A compliant module must follow this sequence:

1. **Capabilities** — `OpCapability Shader`, `OpCapability Float64`, etc.
2. **Extensions** — `OpExtension "SPV_KHR_ray_tracing"`, etc.
3. **Extended instruction set imports** — `%GLSL = OpExtInstImport "GLSL.std.450"`
4. **Memory model** — `OpMemoryModel Logical GLSL450`
5. **Entry points** — `OpEntryPoint Fragment %main "main" %inColor %outColor`
6. **Execution modes** — `OpExecutionMode %main OriginUpperLeft`
7. **Debug information** — `OpSource GLSL 460`, `OpName %main "main"`, `OpMemberName`
8. **Annotations / decorations** — `OpDecorate %ubo DescriptorSet 0`, `OpDecorate %ubo Binding 0`
9. **Type declarations** — `OpTypeVoid`, `OpTypeFloat`, `OpTypeVector`, `OpTypePointer`, `OpTypeStruct`
10. **Constants and global variables** — `OpConstant`, `OpVariable` (in `Uniform`/`Input`/`Output` storage classes)
11. **Functions** — `OpFunction` … `OpFunctionEnd` blocks

[Source: SPIR-V Spec §2.4 Logical Layout of a Module](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html)

### 2.3 Key Opcodes Reference

| Opcode | Purpose |
|--------|---------|
| `OpCapability` | Declare a required capability (e.g., `Shader`, `StorageBuffer16BitAccess`) |
| `OpExtInstImport` | Import a named extended instruction set |
| `OpMemoryModel` | Declare addressing model and memory model |
| `OpEntryPoint` | Declare a shader entry point with its execution model and interface variables |
| `OpTypeVoid` | The void type |
| `OpTypeFloat` | Floating-point scalar type (width: 16, 32, or 64) |
| `OpTypeVector` | Vector type (`OpTypeVector %float 4` → vec4) |
| `OpTypeMatrix` | Matrix of column vectors |
| `OpTypeArray` | Fixed-length array |
| `OpTypePointer` | Typed pointer with storage class |
| `OpTypeStruct` | Structure type with ordered member types |
| `OpTypeImage` | Image type (sampled type, dimensionality, depth, array, MS, sampled, format) |
| `OpTypeSampledImage` | Combined image+sampler type |
| `OpVariable` | Declare a variable in a given storage class |
| `OpLoad` | Load from a pointer |
| `OpStore` | Store to a pointer |
| `OpFAdd` | Floating-point addition |
| `OpFMul` | Floating-point multiplication |
| `OpDot` | Floating-point dot product |
| `OpImageSampleImplicitLod` | Sample a texture with implicit LOD (needs `DerivativeGroupQuadsNV` or fragment stage) |
| `OpReturn` | Return from a void function |
| `OpReturnValue` | Return a value from a function |
| `OpDecorate` | Apply a decoration to an ID (Binding, DescriptorSet, Location, etc.) |
| `OpMemberDecorate` | Apply a decoration to a struct member |

### 2.4 SPIR-V Versions and Vulkan Correspondence

| SPIR-V Version | Release | Vulkan Minimum | Key Additions |
|---------------|---------|---------------|---------------|
| 1.0 | 2016-02 | Vulkan 1.0 | Core shading model |
| 1.1 | 2016-04 | — | SubgroupMask builtins |
| 1.2 | 2017-05 | — | OpModuleProcessed |
| 1.3 | 2018-03 | Vulkan 1.1 | Subgroup operations, variable-pointer extensions promoted |
| 1.4 | 2019-05 | Vulkan 1.2 | `OpCopyLogical`, function parameter storage classes, `OpEntryPoint` interface variables |
| 1.5 | 2020-09 | Vulkan 1.2 | Promoted several KHR extensions (8-bit storage, `VkMemoryModel`) |
| 1.6 | 2021-12 | Vulkan 1.3 | `OpTerminateInvocation`, promoted `SPV_EXT_descriptor_indexing`, removes some deprecated features |

Vulkan 1.4 implementations must support SPIR-V 1.0 through 1.6.
[Source: Vulkan Versions & Porting Guide](https://docs.vulkan.org/guide/latest/versions.html)

---

## 3. Front-End Compilers: glslang and DXC

### 3.1 glslang — The Khronos Reference Front End

glslang is the reference compiler front end for GLSL and ESSL. It accepts GLSL 1.10 through 4.60 and OpenGL ES 1.00 through 3.20, optionally HLSL (now deprecated), and emits SPIR-V.
[Source](https://github.com/KhronosGroup/glslang)

**Command-line usage:**

```bash
# Compile a fragment shader to SPIR-V (Vulkan target)
glslangValidator -V shader.frag -o shader.frag.spv

# Specify the Vulkan API version (target-env controls which SPIR-V capabilities are legal)
glslangValidator -V --target-env vulkan1.3 shader.frag -o shader.frag.spv

# Enable HLSL input (glslang HLSL support is deprecated as of April 2026)
glslangValidator -V -D shader.hlsl -S frag -o shader.hlsl.spv

# Emit human-readable SPIR-V alongside binary
glslangValidator -V -H shader.frag -o shader.frag.spv

# Embed NonSemantic debug info (DWARF-style, for RenderDoc source debugging)
glslangValidator -V -gV shader.frag -o shader.frag.spv

# Inline external includes and define macros
glslangValidator -V -DDEBUG=1 -I./include shader.frag -o shader.frag.spv
```

Shader stage is inferred from the file extension (`.vert`, `.frag`, `.comp`, `.geom`, `.tesc`, `.tese`, `.rgen`, `.rchit`, `.rmiss`, `.rahit`, `.rint`, `.rcall`, `.mesh`, `.task`).

**C++ API:**

The library API uses two primary classes: `glslang::TShader` for per-stage compilation and `glslang::TProgram` for linking and reflection.

```cpp
#include <glslang/Public/ShaderLang.h>
#include <SPIRV/GlslangToSpv.h>

glslang::InitializeProcess();

glslang::TShader shader(EShLangFragment);
const char* src = glslSource.c_str();
shader.setStrings(&src, 1);
shader.setEnvInput(glslang::EShSourceGlsl, EShLangFragment,
                   glslang::EShClientVulkan, 460);
shader.setEnvClient(glslang::EShClientVulkan,
                    glslang::EShTargetVulkan_1_3);
shader.setEnvTarget(glslang::EShTargetSpv,
                    glslang::EShTargetSpv_1_6);

EShMessages messages = (EShMessages)(EShMsgSpvRules | EShMsgVulkanRules);
if (!shader.parse(GetDefaultResources(), 460, false, messages)) {
    std::cerr << shader.getInfoLog();
}

glslang::TProgram program;
program.addShader(&shader);
if (!program.link(messages)) {
    std::cerr << program.getInfoLog();
}

std::vector<uint32_t> spirv;
glslang::SpvOptions opts;
opts.generateDebugInfo = true;
opts.optimizeSize = false;
glslang::GlslangToSpv(*program.getIntermediate(EShLangFragment),
                      spirv, &opts);

glslang::FinalizeProcess();
```

`GlslangToSpv()` walks the AST produced by `TProgram` and emits SPIR-V words directly, applying SPIR-V capabilities matching the requested target environment.

> **Note on HLSL deprecation**: glslang's HLSL front end is officially deprecated as of April 2026 and will be removed at the next major version. The recommended path for HLSL → SPIR-V is DXC.

### 3.2 DXC — DirectXShaderCompiler

DXC is Microsoft's LLVM-based HLSL compiler, supporting shader models up to SM 6.8+.
[Source](https://github.com/microsoft/DirectXShaderCompiler)

Its primary output is DXIL (DirectX Intermediate Language), but it supports SPIR-V as a cross-compilation target via a community-maintained code path that converts the frontend AST directly to SPIR-V — bypassing DXIL to preserve structured control flow that Vulkan's SPIR-V requires.

```bash
# Compile HLSL pixel shader SM6.6 to SPIR-V
dxc -spirv -T ps_6_6 -E main shader.hlsl -Fo shader.spv

# Vertex shader with debug info
dxc -spirv -T vs_6_6 -E VSMain -Zi vertex.hlsl -Fo vertex.spv

# Compute shader with SM6 wave intrinsics
dxc -spirv -T cs_6_6 -E CSMain compute.hlsl -Fo compute.spv

# Specify Vulkan version and enable specific Vulkan extensions
dxc -spirv -T ps_6_6 -E main \
    -fspv-target-env=vulkan1.3 \
    -fspv-extension=SPV_EXT_descriptor_indexing \
    shader.hlsl -Fo shader.spv
```

SM6 features that DXC emits to SPIR-V:

- **Wave intrinsics** (`WaveActiveSum`, `WavePrefixSum`, `WaveGetLaneIndex`): mapped to SPIR-V subgroup operations (`OpGroupNonUniformFAdd`, `OpGroupNonUniformBallot`, `SubgroupLocalInvocationId` builtin).
- **Mesh shaders** (SM6.5): `[numthreads]` mesh entry points → `SPV_EXT_mesh_shader` (covered in §9).
- **Ray tracing** (SM6.5): `TraceRay`, `ReportHit` → `OpTraceRayKHR`, `OpReportIntersectionKHR`.
- **Bindless / Descriptor Arrays**: unbounded HLSL descriptor arrays → `RuntimeDescriptorArray` capability.

**DXC Library API** (for integration into engines and build systems):

```cpp
#include <dxcapi.h>

CComPtr<IDxcUtils> utils;
CComPtr<IDxcCompiler3> compiler;
DxcCreateInstance(CLSID_DxcUtils, IID_PPV_ARGS(&utils));
DxcCreateInstance(CLSID_DxcCompiler3, IID_PPV_ARGS(&compiler));

// Load source
CComPtr<IDxcBlobEncoding> source;
utils->LoadFile(L"shader.hlsl", nullptr, &source);

DxcBuffer buf{ source->GetBufferPointer(),
               source->GetBufferSize(), CP_UTF8 };

// Arguments: -spirv enables SPIR-V output
std::vector<LPCWSTR> args = {
    L"-spirv", L"-T", L"ps_6_6", L"-E", L"main",
    L"-fspv-target-env=vulkan1.3"
};

CComPtr<IDxcResult> result;
compiler->Compile(&buf, args.data(), (UINT32)args.size(),
                  nullptr, IID_PPV_ARGS(&result));

CComPtr<IDxcBlob> spirvBlob;
result->GetOutput(DXC_OUT_OBJECT, IID_PPV_ARGS(&spirvBlob), nullptr);
// spirvBlob->GetBufferPointer() now holds the SPIR-V binary
```

[Source: DXC SPIR-V CodeGen Wiki](https://github.com/microsoft/DirectXShaderCompiler/wiki/SPIR%E2%80%90V-CodeGen)

### 3.3 Other Front Ends

**Tint (WGSL → SPIR-V):** Dawn's built-in WGSL compiler. See Ch35 for full coverage. Tint emits SPIR-V 1.3 targeting Vulkan 1.1. It calls spirv-val internally before returning the module to the caller.

**clang + LLVM-SPIRV (OpenCL C → SPIR-V):**

```bash
# Compile OpenCL C kernel to SPIR-V via LLVM IR
clang -cl-std=CL3.0 --target=spir64 -emit-llvm -c kernel.cl -o kernel.bc
llvm-spirv kernel.bc -o kernel.spv
```

Or with the newer direct path in LLVM ≥ 16:

```bash
clang -cl-std=CL3.0 -target spirv64 kernel.cl -o kernel.spv
```

Mesa's Rusticl OpenCL implementation ingests SPIR-V produced this way.

---

## 4. spirv-tools: Assembler, Disassembler, Validator

SPIRV-Tools is the Khronos-maintained toolkit for binary-level operations on SPIR-V modules. It is a mandatory dependency of Mesa, glslang, Dawn, and virtually every Vulkan implementation.
[Source](https://github.com/KhronosGroup/SPIRV-Tools)

### 4.1 spirv-as — Text Assembler

`spirv-as` converts human-readable SPIR-V text assembly into a binary `.spv` file:

```bash
spirv-as shader.spvasm -o shader.spv
# Specify target environment for validation during assembly
spirv-as --target-env vulkan1.3 shader.spvasm -o shader.spv
```

The text format uses `%name` for IDs (which can be numeric `%1` or symbolic `%float`):

```spirv
; Text format example — simple vec4 pass-through fragment shader
               OpCapability Shader
          %1 = OpExtInstImport "GLSL.std.450"
               OpMemoryModel Logical GLSL450
               OpEntryPoint Fragment %main "main" %inColor %outColor
               OpExecutionMode %main OriginUpperLeft
               OpDecorate %inColor Location 0
               OpDecorate %outColor Location 0
       %void = OpTypeVoid
      %voidf = OpTypeFunction %void
      %float = OpTypeFloat 32
       %vec4 = OpTypeVector %float 4
   %ptr_in4  = OpTypePointer Input %vec4
   %ptr_out4 = OpTypePointer Output %vec4
   %inColor  = OpVariable %ptr_in4 Input
  %outColor  = OpVariable %ptr_out4 Output
       %main = OpFunction %void None %voidf
      %entry = OpLabel
        %val = OpLoad %vec4 %inColor
               OpStore %outColor %val
               OpReturn
               OpFunctionEnd
```

### 4.2 spirv-dis — Disassembler

`spirv-dis` converts a binary SPIR-V module back to the text format above:

```bash
spirv-dis shader.spv
spirv-dis shader.spv -o shader.spvasm          # write to file
spirv-dis --no-color shader.spv                # disable ANSI colour codes
spirv-dis --comment shader.spv                 # add opcode comments for readability
```

This is invaluable for debugging: when a driver rejects a shader or produces wrong output, disassembling the SPIR-V binary lets you read exactly what the front end generated. Combined with `MESA_SPIRV_DUMP_PATH`, you can capture the SPIR-V Mesa actually consumes and disassemble it.

### 4.3 spirv-val — The Validator

`spirv-val` checks a SPIR-V binary against the full specification:

```bash
spirv-val shader.spv
spirv-val --target-env vulkan1.3 shader.spv
# Exit code 0 = valid, non-zero = errors printed to stderr
```

The validator checks:

- **Type consistency**: every `OpFAdd` must have operands and result of the same floating-point type.
- **Capability completeness**: if you use `OpImageSampleImplicitLod`, you need `OpCapability Shader` and `ImplicitLod`.
- **Execution model correctness**: `OpImageSampleImplicitLod` is illegal in a compute or vertex shader (no implicit derivatives).
- **Storage class rules**: `OpVariable` of a pointer type must use a storage class consistent with the pointer's pointee.
- **Layout conformance**: UBO members must satisfy `std140` offset and alignment rules; SSBO members must satisfy `std430`.
- **Structural control flow**: SPIR-V requires reducible control flow; irreducible graphs are rejected.
- **ID bounds**: no reference to an ID ≥ the bound declared in the header.

Running `spirv-val` before submitting to a driver is the first diagnostic step in any shader debugging workflow — driver error messages often point at SPIR-V locations by instruction index, which is far less useful than a spec-cited validator error.

### 4.4 spirv-cfg, spirv-stats, spirv-link, spirv-reduce

**spirv-cfg** exports the control flow graph in Graphviz `.dot` format for visualisation:

```bash
spirv-cfg shader.spv -o shader.dot
dot -Tpng shader.dot -o shader.png
```

**spirv-stats** prints statistics about opcode distribution — useful for understanding what a compiler is generating:

```bash
spirv-stats shader.spv
```

**spirv-link** merges multiple SPIR-V modules into one, resolving `Import` and `Export` linkage decorations. This supports shader library workflows where utility functions compile separately:

```bash
spirv-link lib.spv main.spv -o linked.spv
```

For linking to work, the library module must declare functions with `OpDecorate %func LinkageAttributes "func_name" Export` and the consuming module with `Import`. Both modules need `OpCapability Linkage`.

**spirv-reduce** simplifies a SPIR-V binary to the smallest version that still triggers a specified behaviour (usually a driver bug or compiler crash). The user provides an *interestingness test* — a shell script that exits 0 when the bug is present:

```bash
spirv-reduce --interestingness-test ./test.sh buggy.spv -o reduced.spv
```

This is the standard workflow for submitting minimal reproducer bugs to Mesa, RADV, ANV, or NVK bug trackers.

---

## 5. spirv-opt: The SPIR-V Optimizer

`spirv-opt` applies a sequence of passes to a SPIR-V binary, producing an optimised or reduced-size output.
[Source: SPIRV-Tools optimizer.hpp](https://github.com/KhronosGroup/SPIRV-Tools/blob/main/include/spirv-tools/optimizer.hpp)

### 5.1 Basic Usage

```bash
# Apply default performance optimisations
spirv-opt -O shader.spv -o shader_opt.spv

# Optimise for size (shader cache footprint)
spirv-opt -Os shader.spv -o shader_small.spv

# Apply specific passes in order
spirv-opt \
  --inline-entry-points-exhaustive \
  --ssa-rewrite \
  --eliminate-dead-code-aggressive \
  --merge-blocks \
  shader.spv -o shader_opt.spv

# Print IR between passes (for debugging the optimizer)
spirv-opt -O --print-all shader.spv -o /dev/null 2>&1 | less

# Load pass sequence from a config file
spirv-opt -Oconfig=passes.cfg shader.spv -o shader_opt.spv
```

### 5.2 Important Passes

| Pass flag | API function | Description |
|-----------|-------------|-------------|
| `--inline-entry-points-exhaustive` | `CreateInlineExhaustivePass()` | Inline all function calls in the entry-point call tree. Required before most other passes work well. |
| `--ssa-rewrite` | `CreateSSARewritePass()` | Convert `OpLoad`/`OpStore` on function-local variables to SSA IDs (equivalent to mem2reg in LLVM). Enables nearly all scalar passes. |
| `--eliminate-dead-code-aggressive` | `CreateAggressiveDCEPass()` | Remove instructions whose results never contribute to shader outputs. Removes temporaries left by inlining. |
| `--eliminate-dead-branches` | `CreateDeadBranchElimPass()` | Replace conditional branches on constant conditions with unconditional jumps. |
| `--merge-blocks` | `CreateBlockMergePass()` | Merge a block that has a single predecessor (single outgoing branch) with its successor. |
| `--scalar-replacement` | `CreateScalarReplacementPass()` | Decompose struct/array function-local variables accessed field-by-field into separate scalar variables. |
| `--strength-reduction` | `CreateStrengthReductionPass()` | Replace expensive instructions with cheaper equivalents (e.g., multiply-by-power-of-two → shift). |
| `--loop-unroll` | `CreateLoopUnrollPass()` | Fully unroll loops marked with `LoopControl Unroll`. |
| `--loop-invariant-code-motion` | `CreateLoopInvariantCodeMotionPass()` | Hoist loop-invariant computations out of loops. |
| `--loop-fusion` | `CreateLoopFusionPass()` | Merge adjacent loops with identical bounds and induction variables. |
| `--if-conversion` | `CreateIfConversionPass()` | Convert simple if-then-else patterns to `OpSelect`, enabling the driver's instruction scheduler to eliminate branches. |
| `--redundancy-elimination` | `CreateRedundancyEliminationPass()` | Remove duplicate computations via GVN. |
| `--simplify-instructions` | `CreateSimplifyInstructionsPass()` | Constant-fold and identity-simplify individual instructions. |
| `--eliminate-dead-functions` | `CreateEliminateDeadFunctionsPass()` | Remove unreachable functions from the module (after inlining). |

### 5.3 When spirv-opt Helps vs. Hurts

**Helpful cases:**
- Shaders compiled from GLSL via glslang, which emits one `OpVariable` per local variable and one `OpLoad`/`OpStore` per access. Running `--ssa-rewrite --eliminate-dead-code-aggressive --merge-blocks` dramatically reduces module size.
- Shaders with heavy templated or macro-expanded code, where inlining + DCE eliminates large dead paths.
- Offline shader compilation pipelines where you want a minimised binary for the driver's SPIR-V front end.

**Harmful cases:**
- DXC output targeting specific hardware optimisations. DXC already reasons about the HLSL AST and emits tight SPIR-V; re-running the optimizer can undo deliberate loop unrolling or vectorization hints.
- Shaders using explicit `NoUnroll` or `DontFlatten` hints — the optimizer may ignore them.
- Debug builds: `spirv-opt` strips `OpLine` and `OpSource` annotations under `-O`, removing source correlation.

**Mesa integration:** Mesa's SPIR-V front end (`spirv_to_nir`) does not call `spirv-opt` internally — it relies on its own NIR optimization passes after ingestion. However, Dawn and ANGLE run `spirv-opt` before submission. Some vendor drivers call it internally as part of pipeline compilation.

### 5.4 C++ API

For integration into a build system or engine:

```cpp
#include "spirv-tools/optimizer.hpp"

spvtools::Optimizer opt(SPV_ENV_VULKAN_1_3);
opt.SetMessageConsumer([](spv_message_level_t level,
                          const char* source,
                          const spv_position_t& pos,
                          const char* message) {
    std::cerr << message << '\n';
});

opt.RegisterPerformancePasses();   // equivalent to -O
// or: opt.RegisterSizePasses();   // equivalent to -Os
// or individual: opt.RegisterPass(spvtools::CreateAggressiveDCEPass());

std::vector<uint32_t> optimised;
opt.Run(spirv.data(), spirv.size(), &optimised);
```

---

## 6. SPIRV-Cross: Transpilation

SPIRV-Cross is a practical tool and library for converting SPIR-V back to high-level shading languages. It is primarily used by:

- **MoltenVK** (Apple Metal backend): SPIR-V → Metal Shading Language (MSL)
- **DXVK** and **VKD3D-Proton**: SPIR-V → HLSL for D3D11/D3D12 fallback paths (historically)
- **ANGLE**: SPIR-V → GLSL ES for WebGL2 on platforms without native Vulkan
- **WebGL polyfills and shader inspection tools**: SPIR-V → GLSL for human readability

[Source](https://github.com/KhronosGroup/SPIRV-Cross)

### 6.1 Command-Line Usage

```bash
# Transpile to GLSL (default, targets desktop OpenGL)
spirv-cross --glsl shader.spv

# Target OpenGL ES 3.10
spirv-cross --version 310 --es shader.spv

# Transpile to Metal Shading Language
spirv-cross --msl shader.spv

# Transpile to HLSL SM 5.1
spirv-cross --hlsl --shader-model 51 shader.spv

# Output JSON reflection data
spirv-cross --reflect shader.spv

# Write to file
spirv-cross --glsl shader.spv --output shader.glsl

# Fix clip-space depth from Vulkan [0,1] to OpenGL [-1,1]
spirv-cross --glsl --fixup-clipspace shader.spv
```

### 6.2 C++ API

SPIRV-Cross exposes a hierarchy of compiler classes:

```cpp
#include "spirv_cross/spirv_glsl.hpp"
#include "spirv_cross/spirv_msl.hpp"
#include "spirv_cross/spirv_hlsl.hpp"

// Load SPIR-V binary
std::vector<uint32_t> spirv = load_spv("shader.spv");

// GLSL compilation
spirv_cross::CompilerGLSL compiler(spirv);

// Resource reflection
spirv_cross::ShaderResources resources =
    compiler.get_shader_resources();

// Remap bindings for legacy GLSL (no descriptor sets)
for (auto& ubo : resources.uniform_buffers) {
    uint32_t set =
        compiler.get_decoration(ubo.id, spv::DecorationDescriptorSet);
    uint32_t binding =
        compiler.get_decoration(ubo.id, spv::DecorationBinding);
    // Flatten set/binding to a single binding slot for OpenGL
    compiler.unset_decoration(ubo.id, spv::DecorationDescriptorSet);
    compiler.set_decoration(ubo.id, spv::DecorationBinding,
                            set * 8 + binding);
}

// GLSL requires combined image+samplers;
// SPIR-V keeps them separate (as in Vulkan)
compiler.build_combined_image_samplers();
for (auto& remap : compiler.get_combined_image_samplers()) {
    compiler.set_name(remap.combined_id,
        "SPIRV_Cross_Combined" +
        compiler.get_name(remap.image_id) + "_" +
        compiler.get_name(remap.sampler_id));
}

// Set GLSL version
spirv_cross::CompilerGLSL::Options opts;
opts.version = 450;
opts.es = false;
compiler.set_common_options(opts);

// Emit GLSL source
std::string glsl = compiler.compile();
```

For Metal (used in MoltenVK):

```cpp
spirv_cross::CompilerMSL msl_compiler(spirv);
spirv_cross::CompilerMSL::Options msl_opts;
msl_opts.platform = spirv_cross::CompilerMSL::Options::macOS;
msl_opts.msl_version =
    spirv_cross::CompilerMSL::Options::make_msl_version(3, 0);
msl_compiler.set_msl_options(msl_opts);
std::string msl_src = msl_compiler.compile();
```

### 6.3 Resource Reflection via ShaderResources

`get_shader_resources()` returns a `ShaderResources` struct containing vectors of `Resource`:

```cpp
struct ShaderResources {
    SmallVector<Resource> uniform_buffers;       // UBOs (set/binding decorations)
    SmallVector<Resource> storage_buffers;       // SSBOs
    SmallVector<Resource> stage_inputs;          // vertex inputs / varying inputs
    SmallVector<Resource> stage_outputs;         // vertex outputs / fragment outputs
    SmallVector<Resource> sampled_images;        // combined image+sampler
    SmallVector<Resource> separate_images;       // OpTypeImage (no sampler)
    SmallVector<Resource> separate_samplers;     // OpTypeSampler
    SmallVector<Resource> storage_images;        // image2D etc (storage images)
    SmallVector<Resource> push_constant_buffers; // push constants
    SmallVector<Resource> acceleration_structures; // KHR ray tracing AS
    // ...
};
```

Each `Resource` has `id` (SPIR-V result ID), `base_type_id`, `type_id`, and `name`. Decorations are queried with `get_decoration(id, spv::DecorationBinding)` etc.

### 6.4 Limitations

SPIRV-Cross cannot transpile:

- **Ray tracing instructions** (`OpTraceRayKHR`, `OpReportIntersectionKHR`) to GLSL or HLSL in any meaningful way — these stages have no equivalent in legacy GLSL/HLSL.
- **Mesh/task shaders** to GLSL (no equivalent before GL_NV_mesh_shader or GL_EXT_mesh_shader).
- **Cooperative matrix instructions** — no GLSL equivalent.
- SPIR-V 1.4+ features that rely on storage class changes or new execution models invisible to GLSL.

For these cases, MoltenVK has its own hand-rolled MSL translation paths for ray tracing via Metal's `[[intersection]]` functions.

---

## 7. spirv-reflect: Pipeline Layout Introspection

spirv-reflect is a lightweight, single-header C/C++ library for extracting Vulkan pipeline layout information from a SPIR-V binary — without instantiating a full SPIRV-Cross compiler or a Vulkan device.
[Source](https://github.com/KhronosGroup/SPIRV-Reflect)

### 7.1 Core API

Integration requires only two files: `spirv_reflect.h` and `spirv_reflect.c` (or the compiled static library). No external dependencies beyond the SPIR-V binary itself.

```c
#include "spirv_reflect.h"

SpvReflectShaderModule module;
SpvReflectResult result =
    spvReflectCreateShaderModule(spirv_size, spirv_data, &module);
assert(result == SPV_REFLECT_RESULT_SUCCESS);

// --- Enumerate descriptor sets ---
uint32_t set_count = 0;
spvReflectEnumerateDescriptorSets(&module, &set_count, nullptr);

SpvReflectDescriptorSet** sets =
    (SpvReflectDescriptorSet**)malloc(set_count * sizeof(void*));
spvReflectEnumerateDescriptorSets(&module, &set_count, sets);

for (uint32_t s = 0; s < set_count; s++) {
    SpvReflectDescriptorSet* set = sets[s];
    printf("Set %u: %u bindings\n", set->set, set->binding_count);
    for (uint32_t b = 0; b < set->binding_count; b++) {
        SpvReflectDescriptorBinding* binding = set->bindings[b];
        printf("  Binding %u: %s (type %u, count %u)\n",
               binding->binding,
               binding->name,
               binding->descriptor_type,
               binding->count);
    }
}

// --- Enumerate vertex input variables ---
uint32_t input_count = 0;
spvReflectEnumerateInputVariables(&module, &input_count, nullptr);
SpvReflectInterfaceVariable** inputs =
    (SpvReflectInterfaceVariable**)malloc(input_count * sizeof(void*));
spvReflectEnumerateInputVariables(&module, &input_count, inputs);

for (uint32_t i = 0; i < input_count; i++) {
    printf("Input: location=%u name=%s format=%u\n",
           inputs[i]->location,
           inputs[i]->name,
           inputs[i]->format);
}

// --- Push constants ---
uint32_t pc_count = 0;
spvReflectEnumeratePushConstantBlocks(&module, &pc_count, nullptr);
SpvReflectBlockVariable** pcs =
    (SpvReflectBlockVariable**)malloc(pc_count * sizeof(void*));
spvReflectEnumeratePushConstantBlocks(&module, &pc_count, pcs);
for (uint32_t i = 0; i < pc_count; i++) {
    printf("PushConstant: size=%u\n", pcs[i]->size);
}

spvReflectDestroyShaderModule(&module);
```

### 7.2 Automatic VkDescriptorSetLayout Generation

The canonical use case is engine-side automatic pipeline layout creation:

```cpp
// Engine pattern: reflect SPIR-V → create VkDescriptorSetLayout automatically
void create_pipeline_layout_from_spirv(
    VkDevice device,
    const std::vector<uint32_t>& spirv,
    VkPipelineLayout* out_layout)
{
    // C++ wrapper: spv_reflect::ShaderModule (defined in spirv_reflect.h)
    spv_reflect::ShaderModule mod(spirv.size() * 4, spirv.data());

    uint32_t set_count = 0;
    mod.EnumerateDescriptorSets(&set_count, nullptr);
    std::vector<SpvReflectDescriptorSet*> sets(set_count);
    mod.EnumerateDescriptorSets(&set_count, sets.data());

    std::vector<VkDescriptorSetLayout> dsl(set_count);
    for (uint32_t s = 0; s < set_count; s++) {
        std::vector<VkDescriptorSetLayoutBinding> bindings;
        for (uint32_t b = 0; b < sets[s]->binding_count; b++) {
            auto* rb = sets[s]->bindings[b];
            bindings.push_back({
                .binding         = rb->binding,
                .descriptorType  = (VkDescriptorType)rb->descriptor_type,
                .descriptorCount = rb->count,
                .stageFlags      = VK_SHADER_STAGE_ALL_GRAPHICS,
            });
        }
        VkDescriptorSetLayoutCreateInfo ci{
            .sType        = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO,
            .bindingCount = (uint32_t)bindings.size(),
            .pBindings    = bindings.data(),
        };
        vkCreateDescriptorSetLayout(device, &ci, nullptr, &dsl[s]);
    }

    VkPipelineLayoutCreateInfo plci{
        .sType          = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO,
        .setLayoutCount = set_count,
        .pSetLayouts    = dsl.data(),
    };
    vkCreatePipelineLayout(device, &plci, nullptr, out_layout);
}
```

This pattern means shaders can add or remove descriptor bindings without requiring engine-side header changes — the engine discovers the layout at runtime or at shader-loading time.

### 7.3 CLI Usage

```bash
# YAML-format reflection dump (human-readable)
spirv-reflect -y shader.spv

# Verbose output
spirv-reflect -v shader.spv
```

Output includes entry point name, shader stage, all descriptor sets with binding types and counts, push constant ranges, and vertex inputs with format and location.

---

## 8. SPIR-V in Mesa: spirv_to_nir

Every Vulkan shader submitted to any Mesa driver — RADV, ANV, NVK, Turnip, or any of the others — passes through a single SPIR-V ingestion layer before it reaches driver-specific compilation: `spirv_to_nir`.
[Source: Mesa src/compiler/spirv/](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/compiler/spirv)

### 8.1 Entry Point and Builder

The public entry point is:

```c
nir_shader *spirv_to_nir(const uint32_t *words, size_t word_count,
                          struct nir_spirv_specialization *spec,
                          unsigned num_spec,
                          gl_shader_stage stage,
                          const char *entry_point_name,
                          const struct spirv_to_nir_options *options,
                          const nir_shader_compiler_options *nir_options);
```

Internally, the parser creates a `vtn_builder` — a large context struct that tracks:

- The current position in the instruction stream
- A `vtn_value` array indexed by SPIR-V ID (one entry per `%id` in the module), where each entry holds the ID's type category and value (type pointer, constant, variable reference, function, or ssa def).
- The current `nir_function_impl` being built.
- Decoration maps accumulated from `OpDecorate`/`OpMemberDecorate` before types and variables are processed.

### 8.2 spirv_to_nir_options

The `spirv_to_nir_options` struct (defined in `src/compiler/spirv/nir_spirv.h`) lets each Mesa driver customise ingestion behaviour:
[Source: Mesa nir_spirv.h](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/compiler/spirv/nir_spirv.h)

```c
struct spirv_to_nir_options {
   enum nir_spirv_execution_environment environment;

   /* Whether to promote FragCoord to a system value */
   bool frag_coord_is_sysval;

   /* Whether to keep ViewIndex as an input instead of rewriting to sysval */
   bool view_index_is_input;

   /* Create a nir library (for pre-compiled shader libraries) */
   bool create_library;

   /* Use deref_buffer_array_length instead of get_ssbo_size for OpArrayLength */
   bool use_deref_buffer_array_length;

   /* Initial shader_info::float_controls_execution_mode value */
   uint16_t float_controls_execution_mode;

   /* Per-driver capability declarations */
   struct spirv_supported_capabilities caps;

   /* Address format for various pointer types — drivers choose the format
    * that matches their virtual address model */
   nir_address_format ubo_addr_format;
   nir_address_format ssbo_addr_format;
   nir_address_format phys_ssbo_addr_format;
   nir_address_format push_const_addr_format;
   nir_address_format shared_addr_format;
   nir_address_format global_addr_format;
   nir_address_format temp_addr_format;
   nir_address_format constant_addr_format;

   /* Optional OpenCL built-ins shader */
   const nir_shader *clc_shader;

   /* Debug callback (logs errors with SPIR-V byte offset) */
   struct {
      void (*func)(void *private_data,
                   enum nir_spirv_debug_level level,
                   size_t spirv_offset,
                   const char *message);
      void *private_data;
   } debug;
};
```

For example, RADV sets `ubo_addr_format = nir_address_format_32bit_index_offset` while NVK uses `nir_address_format_64bit_global` to reflect the two drivers' different virtual address models.

### 8.3 Module Processing Flow

The SPIR-V compiler is split across several source files in `src/compiler/spirv/`:

- `spirv_to_nir.c` — main entry point, type handling (`vtn_handle_type`), constant handling, extension dispatch
- `vtn_variables.c` — variable and pointer handling (`vtn_handle_variable`)
- `vtn_cfg.c` — control flow graph construction
- `vtn_alu.c` — arithmetic and logical instruction translation
- `vtn_glsl450.c` — GLSL.std.450 extended instruction dispatch (`vtn_handle_glsl450_instruction`)

Processing proceeds as follows:

1. **Pre-scan phase**: The parser makes two passes. The first pass collects `OpDecorate`/`OpMemberDecorate` into decoration tables indexed by ID. The second pass processes the rest of the module in logical-layout order.

2. **Type handling** (`vtn_handle_type` in `spirv_to_nir.c`): SPIR-V types are translated to `glsl_type*` or `nir_type`. For example:
   - `SpvOpTypeFloat 32` → `glsl_floatType()`
   - `SpvOpTypeVector %float 4` → `glsl_vec4_type()`
   - `SpvOpTypeStruct` → `glsl_struct_type()` with member names from `OpMemberName`

3. **Variable handling** (`vtn_handle_variable` in `vtn_variables.c`): `OpVariable` in `Uniform`/`StorageBuffer` storage creates a `nir_variable` with `data.binding` set from `SpvDecorationBinding` and `data.descriptor_set` from `SpvDecorationDescriptorSet`.

4. **Function handling** (`vtn_handle_function`): Each `OpFunction`/`OpFunctionEnd` pair creates a `nir_function`. Inside, each basic block in SPIR-V's CFG becomes a `nir_block`. Instructions are converted to NIR intrinsics or ALU ops.

5. **Extended instruction sets**: When the parser encounters `SpvOpExtInstImport "GLSL.std.450"`, it registers `vtn_handle_glsl450_instruction` as the dispatch handler for that import ID. Subsequent `SpvOpExtInst` instructions invoke this handler with the extended opcode. Examples of the mapping: `GLSLstd450Sin` → `nir_op_fsin`, `GLSLstd450Pow` → `nir_op_fpow`, `GLSLstd450Sqrt` → `nir_op_fsqrt`.

6. **Capability validation**: If the shader uses a capability the driver did not mark as supported in `spirv_to_nir_options->caps`, `vtn_fail()` triggers a non-local exit via `vtn_longjmp` (not a crash), logging the SPIR-V byte offset and capability name. `MESA_SPIRV_FAIL_DUMP_PATH` can be set to capture the offending binary.

### 8.4 Output: nir_shader*

The output is a fully formed `nir_shader*`, with all SPIR-V IDs resolved, interface variables bound to `nir_variable` nodes with driver-facing indices, and function bodies containing NIR instructions. This `nir_shader` then enters the driver-specific lowering and optimization pipeline:

- **RADV/ACO**: `nir_shader` → `radv_nir_lower_*` → ACO backend (Ch15)
- **ANV (Intel)**: `nir_shader` → `anv_nir_lower_*` → LLVM/BRW backend
- **NVK (Nouveau)**: `nir_shader` → `nvk_nir_lower_*` → NIR → Nouveau shader assembly

### 8.5 Mesa SPIR-V Debugging

Mesa provides environment variables for capturing the SPIR-V that drivers actually consume:

```bash
# Dump all SPIR-V modules to a directory (filenames are BLAKE3 hashes)
MESA_SPIRV_DUMP_PATH=/tmp/shaders vkcube

# Disable shader cache to ensure fresh captures
MESA_SHADER_CACHE_DISABLE=1 MESA_SPIRV_DUMP_PATH=/tmp/shaders vkcube

# Replace a captured shader (for testing modifications)
MESA_SPIRV_READ_PATH=/tmp/modified vkcube
```

The captured `.spv` files can then be validated, disassembled, or modified:

```bash
spirv-val /tmp/shaders/deadbeef.frag.spv
spirv-dis /tmp/shaders/deadbeef.frag.spv | less
# Edit, recompile with glslang, then redirect via MESA_SPIRV_READ_PATH
```

[Source: Mesa SPIR-V Debugging docs](https://docs.mesa3d.org/spirv/index.html)

---

## 9. Extended SPIR-V: Ray Tracing, Mesh Shaders, Cooperative Matrices

### 9.1 Ray Tracing: SPV_KHR_ray_tracing

Ray tracing adds six new execution models and a set of shader-call instructions to SPIR-V. The extension requires SPIR-V 1.4 minimum.
[Source: SPV_KHR_ray_tracing specification](https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_ray_tracing.html)

**New execution models** (all require `OpCapability RayTracingKHR`):

| Execution Model | Role |
|----------------|------|
| `RayGenerationKHR` | Launches rays; one invocation per work item in `vkCmdTraceRaysKHR` |
| `IntersectionKHR` | Custom ray-primitive intersection test |
| `AnyHitKHR` | Called for each candidate intersection |
| `ClosestHitKHR` | Called for the final closest intersection |
| `MissKHR` | Called when no geometry is hit |
| `CallableKHR` | General callable shader invoked by `OpExecuteCallableKHR` |

**New type:**

```spirv
%as_type = OpTypeAccelerationStructureKHR
%as_var  = OpVariable %ptr_as UniformConstant
```

**Key instructions:**

```spirv
; Trace a ray into the acceleration structure
; Parameters: accel_struct, ray_flags, cull_mask, sbt_offset, sbt_stride,
;             miss_index, ray_origin(vec3), t_min, ray_direction(vec3), t_max,
;             payload_variable_id
OpTraceRayKHR %accel_struct %ray_flags %cull_mask \
              %sbt_offset %sbt_stride %miss_index \
              %origin %t_min %direction %t_max %payload

; Report an intersection (in IntersectionKHR stage)
%hit = OpReportIntersectionKHR %float_t_hit %uint_hit_kind

; Execute a callable shader
OpExecuteCallableKHR %uint_sbt_index %callable_data_id
```

**New storage classes** for data passing between shader stages:

- `RayPayloadKHR` — per-ray data passed to/from ClosestHit/Miss/AnyHit
- `HitAttributeKHR` — intersection attributes (e.g., barycentrics) from IntersectionKHR
- `CallableDataKHR` — arguments passed to callable shaders
- `ShaderRecordBufferKHR` — read-only per-shader record from the Shader Binding Table

**Mesa support**: RADV, ANV, and NVK all support `SPV_KHR_ray_tracing`. The `vtn_builder` handles these storage classes and instruction opcodes via dedicated handlers in `src/compiler/spirv/vtn_ray_query.c`.

### 9.2 Mesh Shaders: SPV_EXT_mesh_shader

Mesh shaders replace the fixed vertex+geometry pipeline with two programmable stages: task and mesh.
[Source: SPV_EXT_mesh_shader](https://github.khronos.org/SPIRV-Registry/extensions/EXT/SPV_EXT_mesh_shader.html)

**New execution models:**

- `TaskEXT` — amplification stage; dispatches variable numbers of mesh shader workgroups
- `MeshEXT` — emits vertex and primitive data for rasterisation

```spirv
OpCapability MeshShadingEXT
OpExtension "SPV_EXT_mesh_shader"

; In TaskEXT entry point:
; OpEmitMeshTasksEXT must be the last instruction in the block.
; It dispatches groupCountX × groupCountY × groupCountZ mesh shader workgroups.
OpEmitMeshTasksEXT %group_count_x %group_count_y %group_count_z %payload

; In MeshEXT entry point:
; Set actual output sizes (must be called before writing Output variables).
; Also acts as an OpControlBarrier(Workgroup, Workgroup, AcquireRelease).
OpSetMeshOutputsEXT %vertex_count %primitive_count
```

The shared payload between task and mesh uses the `TaskPayloadWorkgroupEXT` storage class:

```spirv
%TaskPayload = OpTypeStruct %uint %uint   ; custom payload structure
%payload_ptr = OpTypePointer TaskPayloadWorkgroupEXT %TaskPayload
%payload_var = OpVariable %payload_ptr TaskPayloadWorkgroupEXT
```

Each entry point may have at most one variable with `TaskPayloadWorkgroupEXT`.

**Mesa support**: RADV (RDNA2+) and ANV (DG2/Alchemist+) support `SPV_EXT_mesh_shader`. The `vtn_builder` dispatches these opcodes through handlers in `src/compiler/spirv/vtn_mesh.c` (added in Mesa 23.1).

### 9.3 Cooperative Matrices: SPV_KHR_cooperative_matrix

Cooperative matrices expose hardware tensor-core operations (NVIDIA Tensor Cores, AMD Matrix Core, Intel XMX) through SPIR-V.
[Source: SPV_KHR_cooperative_matrix](https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_cooperative_matrix.html)

**Type declaration:**

```spirv
OpCapability CooperativeMatrixKHR
OpCapability VulkanMemoryModel       ; required when Shader capability present
OpExtension "SPV_KHR_cooperative_matrix"

; Declare a cooperative matrix type
; Parameters: component_type, scope, rows, columns, use
%coopmat_a = OpTypeCooperativeMatrixKHR %float16 %subgroup %16 %16 %MatrixAKHR
%coopmat_b = OpTypeCooperativeMatrixKHR %float16 %subgroup %16 %16 %MatrixBKHR
%coopmat_c = OpTypeCooperativeMatrixKHR %float32 %subgroup %16 %16 %MatrixAccumulatorKHR
```

The `Use` operand (MatrixAKHR, MatrixBKHR, MatrixAccumulatorKHR) constrains which positions in a multiply-accumulate the matrix may occupy — the driver rejects mismatched uses at pipeline creation time.

**Matrix multiply-accumulate:**

```spirv
; Result = (A × B) + C  (fused, saturating accumulation)
; Operand order of operations is implementation-defined —
; parallel reduction across subgroup invocations
%result = OpCooperativeMatrixMulAddKHR %coopmat_c %mat_a %mat_b %mat_c
```

**Load/store from buffer:**

```spirv
; Load from a pointer with a row stride
OpCooperativeMatrixLoadKHR %coopmat_a %ptr_a %stride %col_major
OpCooperativeMatrixStoreKHR %ptr_c %result %stride %col_major
```

The matrix is distributed across the subgroup: each invocation holds a fragment of the rows and columns. The distribution is implementation-defined, which is why direct element access is restricted — only load/store/MulAdd operations are specified.

**Hardware mapping:**

- NVIDIA Ada Lovelace (RTX 40 series): 16×16×16 and 32×16×16 FP16/BF16 tiles on Tensor Cores
- AMD RDNA3 (RX 7000 series): 16×16×16 FP16/BF16 matrix tiles via WMMA (Wave Matrix Multiply-Accumulate) instructions
- Intel Xe2 (Arc B-series): 8×16×16 XMX (Xe Matrix eXtensions) instructions

**Mesa support**: NVK supports `SPV_KHR_cooperative_matrix` on Turing and later (via `VK_KHR_cooperative_matrix`). RADV support arrived in Mesa 24.2 for RDNA3. ANV support was added in Mesa 24.1 for Xe2 hardware.

---

## 10. Shader Debugging with SPIR-V

### 10.1 Basic Debug Annotations

Even without extended debug info, SPIR-V provides three mechanisms for source correlation:

```spirv
; Associate a file and version with the module
OpSource GLSL 460 %file_id "// source here (optional)"

; Give a human name to an ID (appears in spirv-dis output and validator errors)
OpName %main "main"
OpName %ubo "UniformBlock"
OpMemberName %UBOType 0 "viewProj"
OpMemberName %UBOType 1 "model"

; Mark that the following instructions come from source line 42, column 5
OpLine %file_id 42 5
  %result = OpFAdd %float %a %b
OpNoLine   ; end of source correlation region
```

`spirv-dis` displays `OpLine` as `// File: shader.frag Line 42 Column 5` comments adjacent to the disassembly, providing a coarse mapping back to source without requiring extended debug extensions.

### 10.2 NonSemantic.Shader.DebugInfo.100

For full source-level debugging comparable to DWARF, SPIR-V uses the `NonSemantic.Shader.DebugInfo.100` extended instruction set, defined in `SPV_KHR_non_semantic_info`. NonSemantic instructions are invisible to the GPU — drivers may strip them at any point — but debuggers like RenderDoc parse them before submission.
[Source: NonSemantic.Shader.DebugInfo.100 spec](https://github.khronos.org/SPIRV-Registry/nonsemantic/NonSemantic.Shader.DebugInfo.100.html)

```spirv
OpExtension "SPV_KHR_non_semantic_info"
%dbg = OpExtInstImport "NonSemantic.Shader.DebugInfo.100"

; Declare a source file
%file = OpExtInst %void %dbg DebugSource %filename_string %source_text

; Declare a compile unit
%cu = OpExtInst %void %dbg DebugCompilationUnit %version %dwarfVersion %file %lang

; Declare a function scope
%fn_type = OpExtInst %void %dbg DebugTypeFunction %debug_none %void_type
%fn_info = OpExtInst %void %dbg DebugFunction
    %name_str %fn_type %file %line %col %cu %linkage_name %flags %line

; Declare a local variable
%var_info = OpExtInst %void %dbg DebugLocalVariable
    %name_str %var_type %file %line %col %fn_scope %flags

; Mark source location inside a function (replaces OpLine for Shader DebugInfo)
OpExtInst %void %dbg DebugLine %file %line_start %line_end %col_start %col_end
  %value = OpFAdd %float %a %b
OpExtInst %void %dbg DebugNoLine
```

**Compiler flags to emit NonSemantic debug info:**

```bash
# glslang: -gV emits NonSemantic.Shader.DebugInfo.100
glslangValidator -V -gV shader.frag -o shader.frag.spv

# DXC: -Zi -fspv-debug=vulkan-with-source
dxc -spirv -T ps_6_6 -E main -Zi -fspv-debug=vulkan-with-source shader.hlsl -Fo shader.spv

# slangc:
slangc -g2 shader.slang -target spirv -o shader.spv
```

For Vulkan devices, `VK_KHR_shader_non_semantic_info` must be enabled (promoted to core in Vulkan 1.3).

**RenderDoc source debugging**: RenderDoc parses the `NonSemantic.Shader.DebugInfo.100` instructions before submission and maps GPU invocation stepping back to source lines, variable names, and types — enabling true source-level shader debugging without device-side overhead.

### 10.3 Debug Printf in Shaders

`NonSemantic.DebugPrintf` extends the NonSemantic framework to allow `printf`-style logging from shader invocations:
[Source: Vulkan Debug Printf documentation](https://vulkan.lunarg.com/doc/view/1.4.321.1/windows/debug_printf.html)

In GLSL, enabled with the `GL_EXT_debug_printf` extension:

```glsl
#extension GL_EXT_debug_printf : require

void main() {
    vec4 pos = gl_Position;
    debugPrintfEXT("Invocation %d: pos = (%f, %f, %f, %f)\n",
                   gl_VertexIndex, pos.x, pos.y, pos.z, pos.w);
}
```

glslang emits:

```spirv
OpExtension "SPV_KHR_non_semantic_info"
%printf_ext = OpExtInstImport "NonSemantic.DebugPrintf"
; ...
OpExtInst %void %printf_ext 1 %fmt_str %invocation_idx %pos_x %pos_y %pos_z %pos_w
```

The Khronos validation layer intercepts these instructions and routes output to the debug callback or stdout. Requirements:

```c
// Enable in layer configuration
VkValidationFeatureEnableEXT enables[] = {
    VK_VALIDATION_FEATURE_ENABLE_DEBUG_PRINTF_EXT,
};
VkValidationFeaturesEXT features = {
    .sType                         = VK_STRUCTURE_TYPE_VALIDATION_FEATURES_EXT,
    .enabledValidationFeatureCount = 1,
    .pEnabledValidationFeatures    = enables,
};
// Attach to VkInstanceCreateInfo.pNext
```

Printf output arrives via `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT` in the debug messenger callback.

### 10.4 GPU-Assisted Validation

Beyond printf, the Khronos validation layer (`VK_LAYER_KHRONOS_validation`) supports GPU-Assisted Validation:

```c
VkValidationFeatureEnableEXT enables[] = {
    VK_VALIDATION_FEATURE_ENABLE_GPU_ASSISTED_EXT,
    VK_VALIDATION_FEATURE_ENABLE_GPU_ASSISTED_RESERVE_BINDING_SLOT_EXT,
};
```

GPU-AV instruments the SPIR-V at pipeline creation time, inserting bounds checks for descriptor array accesses, buffer address validity checks, and ray tracing acceleration structure validity. Violations are reported asynchronously after the offending draw or dispatch completes.

### 10.5 VK_EXT_shader_object for Debugging

`VK_EXT_shader_object` (Vulkan 1.3 extension) decouples shaders from pipeline state objects, allowing shader modules to be compiled and swapped independently. This is especially useful for debugging:

```c
VkShaderEXT shader;
VkShaderCreateInfoEXT shaderInfo = {
    .sType     = VK_STRUCTURE_TYPE_SHADER_CREATE_INFO_EXT,
    .stage     = VK_SHADER_STAGE_FRAGMENT_BIT,
    .nextStage = 0,
    .codeType  = VK_SHADER_CODE_TYPE_SPIRV_EXT,
    .codeSize  = spirv.size() * 4,
    .pCode     = spirv.data(),
    .pName     = "main",
};
vkCreateShadersEXT(device, 1, &shaderInfo, nullptr, &shader);
```

A RenderDoc shader replacement workflow: capture the SPIR-V via `MESA_SPIRV_DUMP_PATH`, modify it, recompile, and inject the new `VkShaderEXT` without rebuilding pipeline objects.

---

## 11. Integrations

**Ch14 — NIR: Mesa's Shader Intermediate Representation**: `spirv_to_nir` is the exclusive bridge between SPIR-V and Mesa's internal IR. Every SPIR-V capability and decoration discussed in this chapter has a corresponding handler in `src/compiler/spirv/`, and every output of that translation is a `nir_shader*` that Ch14's optimization passes consume.

**Ch24 — Vulkan for Application Developers**: All `VkShaderModule` objects are SPIR-V binaries. The binary format (§2), the type system, decorations, and layout rules (§2.2–§2.3) are the specification that Vulkan application developers must satisfy. `spirv-val` (§4.3) is the first-line tool for debugging `VK_ERROR_INVALID_SHADER_NV` and similar errors.

**Ch35 — Dawn and WebGPU**: Tint, Dawn's WGSL compiler, emits SPIR-V 1.3 that enters Mesa's `spirv_to_nir` path. Dawn also integrates `spirv-val` for validation and `spirv-opt` for optimization before submission. The WGSL → SPIR-V → NIR pipeline described across Ch35 and this chapter is one of the most commonly exercised shader paths on Linux desktops today.

**Ch61 — SPIR-V Ecosystem in Depth** (Part XIV): That chapter provides the book-level survey of SPIR-V as a Khronos standard. This chapter (Ch110) goes deep on tooling internals, CLI usage, and integration patterns. The two chapters are complementary: Ch61 covers the why and the spec-level design; Ch110 covers the how and the tool invocations.

**Ch77 — Shader Toolchain**: The broader shader compilation pipeline — offline SPIR-V compilation, pipeline cache interaction, `VkPipelineLibrary` — uses spirv-tools as the pre-submission validation and optimization layer. This chapter's coverage of `spirv-opt` (§5) and `spirv-val` (§4.3) fills in the SPIR-V-specific portion of that toolchain.

**Ch91 — MLIR/GPU Compilation**: MLIR's SPIR-V dialect (`mlir::spirv::ModuleOp`) emits SPIR-V binaries that Mesa's `spirv_to_nir` ingests. The mapping from MLIR's type system to SPIR-V opcodes closely mirrors the glslang AST → SPIR-V path.

**Ch104 — DXVK and VKD3D-Proton**: DXVK's `d3d11` layer translates DirectX shader bytecode (DXBC) to SPIR-V using its own DXBC parser and then feeds the result to Mesa. VKD3D-Proton does the same for DXIL (DX12). Both rely on `spirv-val` (via the Khronos validation layer) to catch translation errors. SPIRV-Cross (§6) is sometimes used in debugging these translations to produce readable GLSL for comparison with the original DXBC disassembly.

**Ch56 — Ray Tracing on Linux**: The `SPV_KHR_ray_tracing` extension (§9.1) is the SPIR-V surface of `VK_KHR_ray_tracing_pipeline`. Every ray tracing shader in Ch56 is a SPIR-V module with one of the six new execution models. The acceleration structure type (`OpTypeAccelerationStructureKHR`) and shader-call instructions covered in §9.1 are the shader-side view of the BVH traversal pipeline covered in Ch56.

---

*Sources referenced in this chapter:*
- [SPIR-V Specification (unified)](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html)
- [SPIRV-Tools GitHub](https://github.com/KhronosGroup/SPIRV-Tools)
- [SPIRV-Tools optimizer.hpp](https://github.com/KhronosGroup/SPIRV-Tools/blob/main/include/spirv-tools/optimizer.hpp)
- [SPIRV-Cross GitHub](https://github.com/KhronosGroup/SPIRV-Cross)
- [SPIRV-Reflect GitHub](https://github.com/KhronosGroup/SPIRV-Reflect)
- [glslang GitHub](https://github.com/KhronosGroup/glslang)
- [DXC GitHub](https://github.com/microsoft/DirectXShaderCompiler)
- [DXC SPIR-V CodeGen Wiki](https://github.com/microsoft/DirectXShaderCompiler/wiki/SPIR%E2%80%90V-CodeGen)
- [SPV_KHR_ray_tracing](https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_ray_tracing.html)
- [SPV_EXT_mesh_shader](https://github.khronos.org/SPIRV-Registry/extensions/EXT/SPV_EXT_mesh_shader.html)
- [SPV_KHR_cooperative_matrix](https://github.khronos.org/SPIRV-Registry/extensions/KHR/SPV_KHR_cooperative_matrix.html)
- [NonSemantic.Shader.DebugInfo.100](https://github.khronos.org/SPIRV-Registry/nonsemantic/NonSemantic.Shader.DebugInfo.100.html)
- [SPIRV-Guide: shader_debug_info](https://github.com/KhronosGroup/SPIRV-Guide/blob/main/chapters/shader_debug_info.md)
- [Vulkan Debug Printf](https://vulkan.lunarg.com/doc/view/1.4.321.1/windows/debug_printf.html)
- [Mesa SPIR-V Debugging](https://docs.mesa3d.org/spirv/index.html)
- [Mesa src/compiler/spirv/](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/compiler/spirv)
- [Vulkan Versions & Porting Guide](https://docs.vulkan.org/guide/latest/versions.html)
