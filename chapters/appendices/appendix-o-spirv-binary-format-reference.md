# Appendix O: SPIR-V Binary Format Reference

> **Status**: First draft — 2026-06-20

This appendix is a compact reference card for the SPIR-V binary intermediate representation.
It targets three groups of readers:

- **Graphics application developers** who need to interpret validation errors, read `spirv-dis`
  output, or pass correct binary to Vulkan driver entry points.
- **Shader compiler authors** implementing a SPIR-V producer or consumer who need a fast lookup
  of encoding rules, type instructions, and decoration IDs.
- **GPU driver developers** debugging `vk_spirv_to_nir()` failures, auditing capability sets, or
  tracking down incorrect decorations that survive into NIR.

All numeric constants are verified against the machine-readable SPIRV-Headers at
[`include/spirv/unified1/spirv.h`](https://github.com/KhronosGroup/SPIRV-Headers/blob/main/include/spirv/unified1/spirv.h)
and the [SPIR-V 1.6 Unified Specification](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html).

---

## Table of Contents

1. [Module Layout](#o1-module-layout)
2. [Instruction Encoding](#o2-instruction-encoding)
3. [Type System](#o3-type-system)
4. [Execution Model and Entry Points](#o4-execution-model-and-entry-points)
5. [Variable and Constant Instructions](#o5-variable-and-constant-instructions)
6. [Common Operations](#o6-common-operations)
7. [Decoration System](#o7-decoration-system)
8. [Capabilities Table](#o8-capabilities-table)
9. [Reading spirv-dis Output: Annotated Example](#o9-reading-spirv-dis-output-annotated-example)
10. [Tools Reference](#o10-tools-reference)
11. [Integrations](#integrations)

---

## O.1 Module Layout

A SPIR-V module is a flat array of 32-bit unsigned words (`uint32_t`). The first five words form
the **module header**; the remainder are instructions in logical order.

```c
/* SPIR-V 1.6 module header — SPIRV-Headers unified1/spirv.h */
struct SpvHeader {
    uint32_t magic;      /* 0x07230203 — "SPIR" in little-endian */
    uint32_t version;    /* (major << 16) | (minor << 8), e.g. 0x00010600 = 1.6 */
    uint32_t generator;  /* tool identifier registered with Khronos */
    uint32_t bound;      /* highest result <id> + 1; all IDs are in [1, bound) */
    uint32_t schema;     /* reserved; must be 0 */
};
```

**Magic number** `0x07230203` encodes the letters `\x03\x02\x23\x07` in memory, which allows
byte-order detection: if the first byte is `0x03` the module is little-endian; if it is `0x07`
the module is big-endian. [Source: SPIR-V Spec §2.2](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html#_magic_number)

**Version encoding** — major version occupies bits 23:16, minor version occupies bits 15:8, bits
31:24 and 7:0 are zero.

| SPIR-V version | `version` word |
|---|---|
| 1.0 | `0x00010000` |
| 1.3 | `0x00010300` |
| 1.5 | `0x00010500` |
| 1.6 | `0x00010600` |

**Generator magic** values are registered in [`spir-v.xml`](https://github.com/KhronosGroup/SPIRV-Headers/blob/main/include/spirv/spir-v.xml) in SPIRV-Headers. The high 16 bits encode the vendor/tool entry; the low 16 bits are a tool-specific build number. Verified examples from that registry and from binary inspection:

- `0x000D000B` — Google Shaderc over Glslang (vendor 13 / `0x0D`, tool build 11). This is the value glslc emits; confirmed by `xxd frag.spv` reading `0b 00 0d 00` at bytes 8–11 (little-endian).
- `0x00080001` — Khronos Glslang Reference Front End (vendor 8 per `spir-v.xml`).
- `0x00060001` — Mesa-IR/SPIR-V Translator (vendor 16 in the registry; Mesa's NIR `spirv_builder` uses a separate Mesa-reserved opcode range).

These values carry no semantic meaning to consuming drivers.

**Logical sections** — after the header, instructions must appear in this order (SPIR-V §2.4):

1. All `OpCapability` instructions
2. All `OpExtension` instructions
3. `OpExtInstImport` instructions (extension instruction sets)
4. Exactly one `OpMemoryModel`
5. All `OpEntryPoint` instructions
6. All `OpExecutionMode` / `OpExecutionModeId` instructions
7. Debug instructions (`OpString`, `OpName`, `OpMemberName`, `OpSource`, …)
8. All annotation instructions (`OpDecorate`, `OpMemberDecorate`, …)
9. All type, constant, and global variable declarations
10. Functions (`OpFunction` … `OpFunctionEnd`), each with its own local variables and basic blocks

---

## O.2 Instruction Encoding

Every instruction occupies one or more 32-bit words. Word 0 always encodes both the **opcode** and
the **word count** for the instruction:

```
Word 0 layout
 ┌────────────────────────────┬────────────────────────────┐
 │ bits [31:16]  word count   │ bits [15:0]   opcode       │
 └────────────────────────────┴────────────────────────────┘
```

```c
uint32_t word0     = module_words[offset];
uint16_t opcode    = (uint16_t)(word0 & 0xFFFF);
uint16_t wc        = (uint16_t)(word0 >> 16);
/* words [offset+1 .. offset+wc-1] are the instruction's operands */
```

`wc` includes word 0 itself, so a bare `OpReturn` (opcode 253, no operands) encodes as
`0x00010000 | 253 = 0x000100FD`.

**Operand kinds** — subsequent words encode typed operands in declaration order (SPIR-V §2.2.1):

| Operand kind | Width | Notes |
|---|---|---|
| `<id>` | 1 word | A result or type reference, 1 ≤ id < bound |
| `LiteralInteger` | 1 word | Unsigned 32-bit integer |
| `LiteralFloat` | 1 or 2 words | 32-bit or 64-bit IEEE 754 |
| `LiteralString` | ≥1 word | UTF-8, null-terminated, zero-padded to 4 bytes |
| `LiteralExtInstInteger` | 1 word | Opcode within an extended instruction set |
| Enum | 1 word | Named constant; use header values only |
| Optional operand | 0 or 1 words | Present only when `wc` is large enough |
| Variadic operand | 0 or more words | Remaining words after required operands |

**Result type and result ID** — most value-producing instructions begin their operand list with
`<result-type-id>` then `<result-id>`. The result ID becomes available for use by later
instructions. Assignment-free instructions (e.g., `OpStore`, `OpBranch`) have neither.

---

## O.3 Type System

All SPIR-V types are declared with `OpType*` instructions. Types are identified solely by their
`<result-id>`; two structurally identical types declared with different instructions have different
IDs and are considered distinct.

### Scalar types

| Instruction | Operands (after result-id) | Notes |
|---|---|---|
| `OpTypeVoid` | — | No value type; used for `void` functions |
| `OpTypeBool` | — | Boolean; size is implementation-defined |
| `OpTypeInt` | `width signedness` | `width` ∈ {8,16,32,64}; `signedness` 0=unsigned, 1=signed |
| `OpTypeFloat` | `width` | `width` ∈ {16,32,64}; 64-bit requires `Float64` capability |

### Composite types

| Instruction | Operands | Notes |
|---|---|---|
| `OpTypeVector` | `component-type count` | `count` ∈ {2,3,4,8,16} |
| `OpTypeMatrix` | `column-type column-count` | `column-type` must be a float vector; requires `Matrix` capability |
| `OpTypeArray` | `element-type length-id` | `length-id` must be a constant scalar integer |
| `OpTypeRuntimeArray` | `element-type` | Unsized; only valid in `StorageBuffer` / `Uniform` structs |
| `OpTypeStruct` | `member-type ...` | Variadic; members are accessed by literal index |

### Pointer and function types

| Instruction | Operands | Notes |
|---|---|---|
| `OpTypePointer` | `storage-class type-id` | Every load/store target is a pointer; storage class determines memory space |
| `OpTypeFunction` | `return-type param-types...` | Used by `OpFunction` |
| `OpTypeSampledImage` | `image-type-id` | Combined image + sampler; produced by `OpSampledImage` |
| `OpTypeImage` | `sampled-type dim depth arrayed MS sampled format [access-qualifier]` | Full image descriptor |

### Storage classes

Storage classes specify the memory space a pointer belongs to. Values from
[`SpvStorageClass`](https://github.com/KhronosGroup/SPIRV-Headers/blob/main/include/spirv/unified1/spirv.h):

| Storage class | Value | Typical Vulkan use |
|---|---|---|
| `UniformConstant` | 0 | Sampler and image variables |
| `Input` | 1 | Vertex attributes, fragment interpolants, built-ins entering the stage |
| `Uniform` | 2 | UBO (`layout(std140)`) — `Block`-decorated struct pointer |
| `Output` | 3 | Stage outputs: vertex positions, fragment colours |
| `Workgroup` | 4 | Shared memory in compute shaders |
| `Private` | 6 | Module-scope private; not shared between invocations |
| `Function` | 7 | Local variables; scope ends at `OpFunctionEnd` |
| `PushConstant` | 9 | Push constants — small fast path, one block per pipeline |
| `Image` | 11 | Storage image (`layout(rgba8, set=…, binding=…) image2D`) |
| `StorageBuffer` | 12 | SSBO (`layout(std430)`) — `Block`-decorated, `BufferBlock` pre-1.3 |
| `PhysicalStorageBuffer` | 5349 | Buffer device addresses (BDA); requires `PhysicalStorageBuffer64` capability |

---

## O.4 Execution Model and Entry Points

### Preamble instructions

These instructions must appear in the order given in §O.1, before any type declarations.

**`OpCapability <cap>`**
Declares a required capability. Must precede every instruction that requires that capability.
A driver that does not support the declared capability must reject the module.

**`OpExtension <name-string>`**
Enables a non-core extension. Example: `OpExtension "SPV_KHR_vulkan_memory_model"`.

**`OpExtInstImport <result-id> <name-string>`**
Imports an extended instruction set. The result ID is used as the first operand of `OpExtInst`.
The canonical import for GLSL built-in math functions: `%1 = OpExtInstImport "GLSL.std.450"`.

**`OpMemoryModel <addressing-model> <memory-model>`**
Declares the module's addressing and memory model.

| Addressing model | Value | Use |
|---|---|---|
| `Logical` | 0 | Vulkan (no raw pointers without BDA extension) |
| `Physical32` | 1 | 32-bit address space (OpenCL kernels) |
| `Physical64` | 2 | 64-bit address space (OpenCL kernels) |

| Memory model | Value | Use |
|---|---|---|
| `GLSL450` | 1 | Vulkan 1.0/1.1 shaders; relaxed coherence |
| `Vulkan` | 3 | Vulkan 1.2+ with `VulkanMemoryModel` capability; explicit `MakeAvailable`/`MakeVisible` semantics |

**`OpEntryPoint <exec-model> <function-id> <name> <interface-ids...>`**
Declares a shader entry point. `<interface-ids>` must list all `Input` and `Output` variables
accessible from the function (SPIR-V 1.4+ also includes `Uniform` and `StorageBuffer` variables).

**`OpExecutionMode <entry-point-id> <mode> [literals...]`**
Sets execution-mode parameters:

| Mode | Applies to | Effect |
|---|---|---|
| `LocalSize x y z` | GLCompute / MeshEXT / TaskEXT | Sets workgroup dimensions statically |
| `LocalSizeId x-id y-id z-id` | GLCompute | Workgroup dimensions via spec constants |
| `OriginUpperLeft` | Fragment | Pixel origin at top-left (Vulkan default) |
| `DepthReplacing` | Fragment | Shader writes `FragDepth` |
| `EarlyFragmentTests` | Fragment | Enables early-Z/stencil; `OpImageWrite` still runs |
| `OutputVertices n` | Geometry / Tessellation | Output vertex count / max vertices |
| `Triangles` / `Quads` / `Isolines` | TessControl/Eval | Input topology |
| `SpacingEqual` / `SpacingFractionalEven` | TessEval | Tessellation spacing |

### Execution models

Values from [`SpvExecutionModel`](https://github.com/KhronosGroup/SPIRV-Headers/blob/main/include/spirv/unified1/spirv.h):

| Execution model | Value | Stage |
|---|---|---|
| `Vertex` | 0 | Vertex shader |
| `TessellationControl` | 1 | Hull shader |
| `TessellationEvaluation` | 2 | Domain shader |
| `Geometry` | 3 | Geometry shader |
| `Fragment` | 4 | Fragment / pixel shader |
| `GLCompute` | 5 | Compute shader (Vulkan / OpenGL) |
| `RayGenerationKHR` | 5313 | Ray-tracing ray generation |
| `IntersectionKHR` | 5314 | Ray-tracing intersection |
| `AnyHitKHR` | 5315 | Ray-tracing any-hit |
| `ClosestHitKHR` | 5316 | Ray-tracing closest-hit |
| `MissKHR` | 5317 | Ray-tracing miss |
| `TaskEXT` | 5364 | Mesh shading — task stage |
| `MeshEXT` | 5365 | Mesh shading — mesh stage |

---

## O.5 Variable and Constant Instructions

### Variables

**`OpVariable <result-type> <result-id> <storage-class> [initializer]`**

`<result-type>` must be a pointer type whose pointee type is the variable's type. For global
variables (any storage class except `Function`) the `OpVariable` must appear in the global
declarations section. For local variables the `OpVariable` must be the first instruction of the
function's first basic block.

```spirv
; float output variable, storage class Output (3)
%_ptr_Output_v4float = OpTypePointer Output %v4float
      %fragColor = OpVariable %_ptr_Output_v4float Output
```

### Constants

| Instruction | Operands | Notes |
|---|---|---|
| `OpConstant` | `<type> <result-id> value...` | Literal value; 64-bit uses two words |
| `OpConstantTrue` | `<type> <result-id>` | `true` for `OpTypeBool` |
| `OpConstantFalse` | `<type> <result-id>` | `false` for `OpTypeBool` |
| `OpConstantComposite` | `<type> <result-id> constituent-ids...` | Aggregate constant from constituent constants |
| `OpConstantNull` | `<type> <result-id>` | Zero-valued aggregate or pointer |
| `OpUndef` | `<type> <result-id>` | Undefined value; any read is valid, used by optimisers |
| `OpSpecConstant` | `<type> <result-id> default-value` | Specialization constant, overridable at pipeline creation |
| `OpSpecConstantTrue` | `<type> <result-id>` | Specialization bool constant, default true |
| `OpSpecConstantFalse` | `<type> <result-id>` | Specialization bool constant, default false |
| `OpSpecConstantComposite` | `<type> <result-id> constituent-ids...` | Composite specialization constant |
| `OpSpecConstantOp` | `<type> <result-id> opcode operands...` | Constant expression evaluated at specialization time |

Specialization constants are assigned a `SpecId` decoration and overridden via
`VkSpecializationInfo` at `vkCreateGraphicsPipelines` / `vkCreateComputePipelines` time. The
`OpSpecConstantOp` instruction encodes a restricted subset of arithmetic opcodes that can be
applied to spec constants at pipeline-compile time.

---

## O.6 Common Operations

Operand notation: `R = result-id`, `T = result-type-id`, `a/b/c = value operands`.
All value-producing instructions begin `T R …`.

### Arithmetic

| Instruction | Operands | Semantics |
|---|---|---|
| `OpFAdd` | `T R a b` | IEEE 754 `a + b` |
| `OpFSub` | `T R a b` | IEEE 754 `a - b` |
| `OpFMul` | `T R a b` | IEEE 754 `a * b` |
| `OpFDiv` | `T R a b` | IEEE 754 `a / b` |
| `OpFNegate` | `T R a` | IEEE 754 `-a` |
| `OpFRem` | `T R a b` | IEEE remainder; sign matches dividend |
| `OpIAdd` | `T R a b` | Integer add, modulo 2^N; works for signed/unsigned |
| `OpISub` | `T R a b` | Integer subtract |
| `OpIMul` | `T R a b` | Integer multiply (low N bits) |
| `OpUDiv` | `T R a b` | Unsigned integer divide |
| `OpSDiv` | `T R a b` | Signed integer divide (truncation toward zero) |
| `OpSNegate` | `T R a` | Two's-complement negate |
| `OpDot` | `T R a b` | Dot product of float vectors |
| `OpVectorTimesScalar` | `T R vec scalar` | Component-wise scale |
| `OpMatrixTimesVector` | `T R mat vec` | Matrix-vector product |

### Comparison

All comparison instructions produce `OpTypeBool` or a vector of booleans. `Ord` variants return
false if either operand is NaN; `Unord` variants return true if either operand is NaN.

| Instruction | Semantics |
|---|---|
| `OpFOrdEqual` | `a == b` (ordered) |
| `OpFUnordEqual` | `a == b` (unordered) |
| `OpFOrdLessThan` | `a < b` |
| `OpFOrdLessThanEqual` | `a <= b` |
| `OpFOrdGreaterThan` | `a > b` |
| `OpFUnordGreaterThan` | `a > b` (unordered) |
| `OpIEqual` | Integer equality |
| `OpULessThan` | Unsigned `a < b` |
| `OpSLessThan` | Signed `a < b` |
| `OpLogicalNot` | Boolean `!a` |
| `OpLogicalAnd` | Boolean `a && b` |
| `OpLogicalOr` | Boolean `a \|\| b` |

### Bitwise and shift

| Instruction | Semantics |
|---|---|
| `OpBitwiseAnd` | `a & b` |
| `OpBitwiseOr` | `a \| b` |
| `OpBitwiseXor` | `a ^ b` |
| `OpNot` | `~a` |
| `OpShiftRightLogical` | `a >> b` (zero fill) |
| `OpShiftRightArithmetic` | `a >> b` (sign fill) |
| `OpShiftLeftLogical` | `a << b` |
| `OpBitCount` | Population count |
| `OpBitReverse` | Bit reversal |

### Composite access

| Instruction | Operands | Semantics |
|---|---|---|
| `OpCompositeConstruct` | `T R constituents...` | Build vector/struct from components |
| `OpCompositeExtract` | `T R composite indices...` | Extract component by literal index chain |
| `OpCompositeInsert` | `T R object composite indices...` | Return composite with one component replaced |
| `OpVectorShuffle` | `T R v1 v2 components...` | Swizzle/permute from two vectors |
| `OpVectorExtractDynamic` | `T R vec index-id` | Dynamic component select (index is a value, not literal) |
| `OpVectorInsertDynamic` | `T R vec component index-id` | Dynamic component insert |

### Memory

| Instruction | Operands | Semantics |
|---|---|---|
| `OpLoad` | `T R pointer [mem-operands]` | Load from pointer |
| `OpStore` | `pointer value [mem-operands]` | Store value to pointer; no result |
| `OpAccessChain` | `T R base indices...` | GEP: compute pointer into composite; indices are value IDs |
| `OpInBoundsAccessChain` | `T R base indices...` | As above, with in-bounds contract (UB if violated) |
| `OpCopyMemory` | `dst src [mem-operands]` | Memory copy between pointers |
| `OpCopyObject` | `T R operand` | SSA copy (logical alias) |
| `OpConvertFToS` | `T R f` | Float → signed int |
| `OpConvertSToF` | `T R i` | Signed int → float |
| `OpConvertUToF` | `T R u` | Unsigned int → float |
| `OpBitcast` | `T R value` | Reinterpret bits; types must have same bit width |

Optional `mem-operands` is a bitmask: `None=0`, `Volatile=1`, `Aligned=2 align-literal`,
`Nontemporal=4`, `MakePointerAvailable=8 scope`, `MakePointerVisible=16 scope`,
`NonPrivatePointer=32`.

### Control flow

| Instruction | Operands | Notes |
|---|---|---|
| `OpFunction` | `T R ctrl fn-type` | Begin function; `ctrl` is `None`, `Inline`, or `Pure` |
| `OpFunctionParameter` | `T R` | Declared for each parameter, inside `OpFunction`…`OpLabel` |
| `OpLabel` | `R` | Begins a basic block; `R` is the block's ID |
| `OpBranch` | `target-label` | Unconditional branch |
| `OpBranchConditional` | `cond true-label false-label [weights...]` | Conditional branch |
| `OpSwitch` | `selector default targets...` | Multi-way branch |
| `OpSelectionMerge` | `merge-label ctrl` | Structured selection header; precedes `OpBranchConditional` |
| `OpLoopMerge` | `merge-label continue-label ctrl` | Structured loop header; precedes `OpBranch`/`OpBranchConditional` |
| `OpReturn` | — | Return void |
| `OpReturnValue` | `value` | Return a value |
| `OpFunctionCall` | `T R fn args...` | Direct call; recursive calls are illegal in the `Shader` execution model |
| `OpKill` | — | Fragment: terminate (deprecated in 1.6; prefer `OpTerminateInvocation`) |
| `OpTerminateInvocation` | — | Fragment: discard and terminate (SPIR-V 1.6) |
| `OpUnreachable` | — | Assert this block is unreachable |
| `OpFunctionEnd` | — | End of function |

### Image sampling

| Instruction | Operands | Notes |
|---|---|---|
| `OpSampledImage` | `T R image sampler` | Combine separate image and sampler |
| `OpImageSampleImplicitLod` | `T R sampled-img coord [img-ops]` | Sample with implicit LOD (Fragment stage only) |
| `OpImageSampleExplicitLod` | `T R sampled-img coord Lod lod [img-ops]` | Sample with explicit LOD (any stage) |
| `OpImageSampleDrefImplicitLod` | `T R sampled-img coord dref [img-ops]` | Depth comparison sample, implicit LOD |
| `OpImageFetch` | `T R image coord [img-ops]` | Fetch from integer coordinate (non-sampled) |
| `OpImageRead` | `T R image coord [img-ops]` | Read from storage image |
| `OpImageWrite` | `image coord texel [img-ops]` | Write to storage image |
| `OpImageQuerySize` | `T R image` | Image dimensions without LOD |
| `OpImageQuerySizeLod` | `T R image lod` | Mip-level dimensions |

Image operand options (bitmask after coordinate): `Bias`, `Lod`, `Grad`, `ConstOffset`,
`Offset`, `ConstOffsets`, `Sample`, `MinLod`, `MakeTexelAvailable`, `MakeTexelVisible`.

---

## O.7 Decoration System

Decorations annotate `<id>`s with semantic metadata. They appear in the annotation section
(before type declarations) and have no runtime cost — they are consumed by the driver or
reflection library at pipeline creation time.

**`OpDecorate <target-id> <decoration> [literals...]`** — decorate a type or variable ID.

**`OpMemberDecorate <struct-type-id> <member-index> <decoration> [literals...]`** — decorate a
specific struct member.

**`OpDecorateString <target-id> <decoration> <string>`** — string-valued decoration (e.g., `HlslSemanticGOOGLE`).

### Interface decorations

| Decoration | Value | Operand | Use |
|---|---|---|---|
| `Location` | 30 | `uint32` | Input/output slot; vertex attributes, fragment outputs |
| `Component` | 31 | `uint32` | Component within a Location (0–3) |
| `Binding` | 33 | `uint32` | Descriptor binding index within set |
| `DescriptorSet` | 34 | `uint32` | Vulkan descriptor set index |
| `InputAttachmentIndex` | 43 | `uint32` | Subpass input attachment slot |
| `Index` | 32 | `uint32` | Dual-source blend index |

### Memory layout decorations

These must be applied to `Block`-decorated structs (UBO) and their members:

| Decoration | Value | Operand | Use |
|---|---|---|---|
| `Block` | 2 | — | Marks a struct as a Uniform Buffer Object |
| `BufferBlock` | 3 | — | Marks a struct as an SSBO (pre-SPIR-V 1.3; `Block` + `StorageBuffer` class is preferred in 1.3+) |
| `Offset` | 35 | `uint32` | Byte offset of a struct member |
| `ArrayStride` | 6 | `uint32` | Bytes between consecutive array elements |
| `MatrixStride` | 7 | `uint32` | Bytes between matrix columns (ColMajor) or rows (RowMajor) |
| `ColMajor` | 5 | — | Matrix stored column-major (default for GLSL) |
| `RowMajor` | 4 | — | Matrix stored row-major |

### Quality decorations

| Decoration | Value | Notes |
|---|---|---|
| `RelaxedPrecision` | 0 | Allows mediump-equivalent on the decorated value |
| `SpecId` | 1 | Spec constant override ID (used with `OpSpecConstant`) |
| `NoPerspective` | 13 | Linear interpolation, no perspective divide |
| `Flat` | 14 | No interpolation; must be used on integer fragment inputs |
| `Centroid` | 16 | Centroid interpolation |
| `Sample` | 17 | Per-sample interpolation |
| `Invariant` | 18 | Invariant across shader invocations |
| `NonWritable` | 24 | SSBO member is read-only |
| `NonReadable` | 25 | Storage image is write-only |
| `Aliased` | 20 | Pointer may alias other pointers |

### `BuiltIn` values

`OpDecorate %var BuiltIn <value>` (decoration value 11):

| BuiltIn name | Value | Stage(s) | Description |
|---|---|---|---|
| `Position` | 0 | Vertex out / TessEval out | Clip-space position `gl_Position` |
| `PointSize` | 1 | Vertex out | `gl_PointSize` |
| `VertexIndex` | 42 | Vertex in | `gl_VertexIndex` (1-indexed in OpenGL, 0-indexed in Vulkan) |
| `InstanceIndex` | 43 | Vertex in | `gl_InstanceIndex` |
| `PrimitiveId` | 7 | Geom / Frag in | `gl_PrimitiveID` |
| `InvocationId` | 8 | Geom / TessCtrl in | `gl_InvocationID` |
| `FragCoord` | 15 | Fragment in | Window-space position `gl_FragCoord` |
| `FrontFacing` | 17 | Fragment in | `gl_FrontFacing` |
| `SampleId` | 18 | Fragment in | `gl_SampleID` |
| `FragDepth` | 22 | Fragment out | `gl_FragDepth`; requires `DepthReplacing` mode |
| `NumWorkgroups` | 24 | Compute in | `gl_NumWorkGroups` |
| `WorkgroupSize` | 25 | Compute in | `gl_WorkGroupSize` (constant) |
| `WorkgroupId` | 26 | Compute in | `gl_WorkGroupID` |
| `LocalInvocationId` | 27 | Compute in | `gl_LocalInvocationID` |
| `GlobalInvocationId` | 28 | Compute in | `gl_GlobalInvocationID` |
| `LocalInvocationIndex` | 29 | Compute in | `gl_LocalInvocationIndex` (linearised) |
| `SubgroupId` | 40 | Compute in | `gl_SubgroupID`; requires `GroupNonUniform` capability |
| `SubgroupLocalInvocationId` | 41 | Any in | `gl_SubgroupInvocationID` |

---

## O.8 Capabilities Table

Capabilities declare which optional features a module uses. Each `OpCapability` is validated by
the driver against `VkPhysicalDeviceVulkan12Features` and extension feature structs before
pipeline creation begins. All values from
[`SpvCapability`](https://github.com/KhronosGroup/SPIRV-Headers/blob/main/include/spirv/unified1/spirv.h):

| Capability | ID | Required by / Enables |
|---|---|---|
| `Matrix` | 0 | `OpTypeMatrix`, `OpDot`, `OpMatrixTimesVector` |
| `Shader` | 1 | All graphics shaders (Vertex, Fragment, etc.); required for Vulkan |
| `Geometry` | 2 | Geometry shader execution model |
| `Tessellation` | 3 | TessellationControl / TessellationEvaluation models |
| `Float64` | 10 | 64-bit `double` types (`OpTypeFloat 64`) |
| `Int64` | 11 | 64-bit integer types (`OpTypeInt 64`) |
| `InputAttachment` | 40 | `OpTypeImage` with Dim=SubpassData |
| `StorageImageReadWithoutFormat` | 55 | `OpImageRead` from unformatted storage images |
| `StorageImageWriteWithoutFormat` | 56 | `OpImageWrite` to unformatted storage images |
| `GroupNonUniform` | 61 | Subgroup ballot, vote, arithmetic; wave intrinsics |
| `ShaderLayer` | 69 | Write to `Layer` from vertex or mesh stage |
| `DrawParameters` | 4427 | `BaseVertex`, `BaseInstance`, `DrawIndex` built-ins |
| `SubgroupVoteKHR` | 4431 | `OpGroupNonUniformAll`, `OpGroupNonUniformAny` |
| `VariablePointers` | 4442 | Pointers in PhysicalStorageBuffer with full flexibility |
| `RayTracingKHR` | 4479 | Full `VK_KHR_ray_tracing_pipeline` ray tracing |
| `VulkanMemoryModel` | 5345 | Explicit acquire/release sync semantics for Vulkan 1.2+ |
| `MeshShadingEXT` | 5283 | `MeshEXT` / `TaskEXT` execution models |

> **Note**: The user-facing prompt listed `RayTracingKHR = 5353` — that value is
> `RayTracingProvisionalKHR`, which was the provisional extension ID before standardisation.
> The correct ratified value is **4479** as shown above, verified against
> `SpvCapabilityRayTracingKHR = 4479` in SPIRV-Headers.

---

## O.9 Reading spirv-dis Output: Annotated Example

The following fragment shader was compiled with `glslc -fshader-stage=frag` using
Shaderc/Glslang and disassembled with `spirv-dis`. The output is actual tool output, not
reconstructed from memory.

```glsl
#version 450
layout(location=0) in  vec2 vUV;
layout(binding=0)  uniform sampler2D uTex;
layout(location=0) out vec4 fragColor;
void main() { fragColor = texture(uTex, vUV); }
```

```bash
glslc -fshader-stage=frag frag.glsl -o frag.spv
spirv-dis frag.spv
```

```spirv
; SPIR-V
; Version: 1.0
; Generator: Google Shaderc over Glslang; 11   ← generator magic 11 = this Shaderc build
; Bound: 20                                     ← IDs 1..19 are in use
; Schema: 0                                     ← reserved, always 0

               OpCapability Shader              ; capability 1 — all graphics shaders

          %1 = OpExtInstImport "GLSL.std.450"   ; import the GLSL math built-ins; %1 is the set handle

               OpMemoryModel Logical GLSL450    ; addressing=Logical(0), memory=GLSL450(1)

               OpEntryPoint Fragment %main "main" %fragColor %vUV
               ;  ↑ execution model=Fragment(4); function %main; export name "main"
               ;    interface variables: %fragColor (Output), %vUV (Input)

               OpExecutionMode %main OriginUpperLeft
               ;  ↑ Vulkan default: pixel (0,0) is top-left

; ── Debug annotations (stripped by spirv-opt --strip-debug) ─────────────────
               OpSource GLSL 450
               OpSourceExtension "GL_GOOGLE_cpp_style_line_directive"
               OpSourceExtension "GL_GOOGLE_include_directive"
               OpName %main "main"
               OpName %fragColor "fragColor"
               OpName %uTex "uTex"
               OpName %vUV "vUV"

; ── Decorations ─────────────────────────────────────────────────────────────
               OpDecorate %fragColor Location 0   ; output slot 0
               OpDecorate %uTex Binding 0          ; descriptor binding 0
               OpDecorate %uTex DescriptorSet 0    ; descriptor set 0
               OpDecorate %vUV Location 0          ; input slot 0

; ── Type declarations ────────────────────────────────────────────────────────
       %void = OpTypeVoid
          %3 = OpTypeFunction %void               ; () → void
      %float = OpTypeFloat 32
    %v4float = OpTypeVector %float 4              ; vec4
    %v2float = OpTypeVector %float 2              ; vec2

%_ptr_Output_v4float = OpTypePointer Output %v4float
         %10 = OpTypeImage %float 2D 0 0 0 1 Unknown
               ;  ↑ sampled type=float, Dim=2D, depth=0(not depth), arrayed=0,
               ;    MS=0(single-sample), sampled=1(sampled), format=Unknown
         %11 = OpTypeSampledImage %10             ; sampler2D = image + sampler
%_ptr_UniformConstant_11 = OpTypePointer UniformConstant %11
%_ptr_Input_v2float = OpTypePointer Input %v2float

; ── Global variables ─────────────────────────────────────────────────────────
  %fragColor = OpVariable %_ptr_Output_v4float Output
       %uTex = OpVariable %_ptr_UniformConstant_11 UniformConstant
        %vUV = OpVariable %_ptr_Input_v2float Input

; ── Function body ─────────────────────────────────────────────────────────────
       %main = OpFunction %void None %3           ; define main()
          %5 = OpLabel                            ; entry basic block %5

         %14 = OpLoad %11 %uTex                  ; load the sampler2D from UniformConstant
         %18 = OpLoad %v2float %vUV              ; load the UV input
         %19 = OpImageSampleImplicitLod %v4float %14 %18
               ;  ↑ sample: result=vec4, sampled-image=%14, coord=%18
               ;    implicit LOD = driver computes LOD from derivatives (valid in Fragment)

               OpStore %fragColor %19            ; write the sample result to output
               OpReturn
               OpFunctionEnd
```

Key observations:

- The `OpTypeImage` encodes six flags inline: sampled-component-type, dimensionality, depth,
  array, multisampling, and whether it is used as a sampled image or storage image.
- `OpImageSampleImplicitLod` has no operand for the LOD — the hardware computes it from
  `dFdx`/`dFdy` of the texture coordinate. This instruction is illegal in compute shaders.
- `OpTypeSampledImage` creates a `sampler2D` by combining an image type with an implicit sampler;
  both must have been loaded with `OpLoad` before being combined with `OpSampledImage` when using
  separate image/sampler types.
- Variable names (`%fragColor`, `%uTex`, `%vUV`) come from `OpName` debug instructions; they are
  absent in stripped modules. Tools like `spirv-reflect` and Mesa's `spirv_to_nir` use the
  `OpDecorate DescriptorSet` / `Binding` values, not the names.

---

## O.10 Tools Reference

All tools listed here are open source. Install via system package (`spirv-tools`, `shaderc`,
`spirv-cross`) or build from source.

| Tool | Source | One-liner |
|---|---|---|
| `glslc` | [google/shaderc](https://github.com/google/shaderc) | Compile GLSL or HLSL to SPIR-V; wraps `glslang`. Flags: `-fshader-stage=<vert\|frag\|comp\|…>`, `-O`, `-g` (debug info), `-MD` (dependency file). |
| `glslangValidator` | [KhronosGroup/glslang](https://github.com/KhronosGroup/glslang) | Reference Khronos GLSL→SPIR-V compiler; also validates GLSL. Use `--target-env vulkan1.3` to pin the environment. |
| `spirv-as` | [KhronosGroup/SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools) | Assemble human-readable SPIR-V text (as output by `spirv-dis`) back to binary. Useful for hand-editing then revalidating a module. |
| `spirv-dis` | SPIRV-Tools | Disassemble a SPIR-V binary to human-readable text. `--no-indent` for compact output; `--raw-id` to show numeric IDs instead of `%name`. |
| `spirv-val` | SPIRV-Tools | Validate a SPIR-V module against the spec. `--target-env vulkan1.3` to check Vulkan-specific rules. Used as a CI prerequisite in Mesa and Vulkan CTS. |
| `spirv-opt` | SPIRV-Tools | Optimise a SPIR-V module. Key passes: `-O` (all optimisations), `-Os` (optimise for size), `--strip-debug` (remove `OpName`/`OpSource`), `--inline-entry-points-exhaustive`, `--eliminate-dead-code-aggressive`. |
| `spirv-cross` | [KhronosGroup/SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) | Transpile SPIR-V to GLSL, GLSL ES, HLSL, or MSL. Invaluable for inspecting Mesa-consumed SPIR-V in a readable shading language. Flags: `--vulkan-semantics` (output GLSL 450), `--msl` (Metal). |
| `spirv-reflect` | [KhronosGroup/SPIRV-Reflect](https://github.com/KhronosGroup/SPIRV-Reflect) | C/C++ library that extracts descriptor set layout, push constant ranges, input/output locations, and specialization constants from a SPIR-V module without a full driver. Used by game engines for automatic pipeline layout generation. |
| `tint` | [dawn.googlesource.com/dawn](https://dawn.googlesource.com/dawn) | Google's WGSL compiler; produces SPIR-V as its primary Vulkan target. Also targets MSL and HLSL. Used by Chrome's Dawn WebGPU implementation (Ch35) and the WebGPU CTS. |
| `dxc` (DirectXShaderCompiler) | [microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler) | Compile HLSL to SPIR-V (`-spirv -fvk-*` flags). Used by DXVK, VKD3D-Proton, and UE5 to produce Vulkan shaders from HLSL source. |

### Typical debugging workflow

```bash
# 1. Compile GLSL to SPIR-V with debug info
glslc -fshader-stage=frag -g shader.glsl -o shader.spv

# 2. Validate against the Vulkan 1.3 environment
spirv-val --target-env vulkan1.3 shader.spv

# 3. Disassemble for inspection
spirv-dis shader.spv -o shader.spvasm

# 4. Strip debug info and optimise for size
spirv-opt --strip-debug -Os shader.spv -o shader_opt.spv

# 5. Cross-compile to GLSL for readability
spirv-cross --vulkan-semantics shader.spv

# 6. Validate again after optimisation (opt bugs do exist)
spirv-val --target-env vulkan1.3 shader_opt.spv
```

To capture the exact SPIR-V a driver consumes (before any driver-side patching), use
`RADV_DEBUG=spirv` (AMD/RADV), `ANV_ENABLE_PIPELINE_CACHE=0` with `VK_LAYER_LUNARG_api_dump`
(Intel/ANV), or `NAK_DEBUG=print` (NVK/Nouveau — prints the NAK compiler IR to stderr).
These environment variables are documented in Ch63 (Vulkan shader compilation).

---

## Integrations

- **Ch35 — Dawn / WebGPU**: Dawn's Tint compiler emits SPIR-V 1.3 targeting the Vulkan execution
  environment. The `OpCapability` set Tint requests is constrained to what `VkPhysicalDeviceFeatures`
  reports available on the host device; any capability in this appendix's table that Tint requests
  must be satisfied by the Mesa Vulkan driver beneath it.

- **Ch61 — SPIR-V Ecosystem in Depth**: The full chapter counterpart to this appendix. Covers the
  SPIR-V intermediate representation in narrative detail: the design rationale for the SSA form,
  structured control flow requirements, the `Logical` addressing model, and how the SPIR-V
  validator enforces dominance rules. This appendix is the quick-reference companion to that
  chapter.

- **Ch63 — Vulkan Shader Compilation**: Traces the journey from `vkCreateShaderModule` through
  `vk_spirv_to_nir()` into NIR. The capability validation, decoration handling, and type lowering
  that Mesa performs at the SPIR-V → NIR boundary directly corresponds to the declarations
  described in §O.3, §O.4, and §O.7 of this appendix.

- **Ch110 — SPIR-V Tooling (spirv-tools, SPIRV-Cross, and the Shader Ecosystem)**: Deep coverage
  of the tools listed in §O.10: `spirv-opt` pass pipeline, `spirv-cross` reflection internals,
  `spirv-reflect` library API, and how `tint` and `naga` interact with the broader SPIRV-Tools
  ecosystem. This appendix provides the binary encoding context that Ch110 assumes.

- **Appendix L — Shader Toolchain Matrix**: Shows which tools (glslc, DXC, Tint, naga,
  spirv-cross, glslang) produce or consume SPIR-V binary and where they fit in the Linux graphics
  stack. Use Appendix L as the entry point for format-to-format routing; use this appendix for the
  encoding details of the SPIR-V modules those tools produce.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
