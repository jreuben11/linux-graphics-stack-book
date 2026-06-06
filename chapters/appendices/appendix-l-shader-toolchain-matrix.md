# Appendix L: Shader Toolchain Matrix

> **Status**: Stub — content to be populated during final editing pass

This appendix is a quick-reference matrix showing which tools read and write which shader formats, and where each tool fits in the Linux graphics stack.

---

## L.1 Format Overview

| Format | Description | Primary consumers |
|---|---|---|
| GLSL | OpenGL Shading Language | Mesa (NIR front end via `glsl_to_nir`) |
| GLSL ES | OpenGL ES Shading Language | ANGLE, Mesa GLES drivers |
| HLSL | High-Level Shader Language (Microsoft) | DXVK, VKD3D-Proton, UE5, Unity, DXC |
| WGSL | WebGPU Shading Language | Dawn/Tint, wgpu/naga |
| SPIR-V | Khronos binary IR | All Mesa Vulkan drivers via `vk_spirv_to_nir()` |
| GDShader/GLSL | Godot shaders → GLSL → SPIR-V | Godot 4 (Ch41) |
| SkSL | Skia shading language | Skia Ganesh and Graphite (Ch37) |
| MSL | Metal Shading Language | macOS/iOS only |
| DXBC/DXIL | DirectX bytecode / LLVM IR | Windows only (D3D11/D3D12) |

---

## L.2 Tool Matrix

| Tool | Source | Target(s) | Notes |
|---|---|---|---|
| `glslang` / `shaderc` | GLSL, GLSL ES | SPIR-V | Reference Khronos compiler; used by ANGLE, many apps |
| `DXC` (DirectXShaderCompiler) | HLSL | SPIR-V, DXIL | Used by UE5, Unity (`#pragma use_dxc`), VKD3D-Proton |
| `Tint` | WGSL | SPIR-V, MSL, HLSL, GLSL | Dawn's shader compiler (Ch35) |
| `naga` | WGSL, SPIR-V, GLSL | SPIR-V, GLSL, MSL, HLSL | wgpu's Rust shader translator (Ch40) |
| `spirv-cross` | SPIR-V | GLSL, HLSL, MSL, GLSL ES | Cross-compilation for inspection or non-Vulkan backends |
| `spirv-val` | SPIR-V | — (validation) | Khronos SPIR-V validator; Mesa CI prerequisite |
| `spirv-opt` | SPIR-V | SPIR-V | SPIRV-Tools optimiser; useful for isolating driver bugs |
| `spirv-dis` / `spirv-as` | SPIR-V ↔ text | SPIR-V / text | Disassembler and assembler for inspection |
| `vk_spirv_to_nir()` | SPIR-V | NIR | Mesa's SPIR-V front end; entry point for all Vulkan drivers |
| `glsl_to_nir()` | GLSL | NIR | Mesa's GLSL front end; used by GL state tracker (Gallium) |
| `ACO` | NIR | GCN/RDNA ISA | RADV's shader compiler (Ch15) |
| `IGC` (Intel Graphics Compiler) | SPIR-V / LLVM IR | GEN ISA | Intel's GPU compiler; used by ANV and compute-runtime |
| `HLSLcc` | DXBC | GLSL / SPIR-V-compat GLSL | Unity's legacy shader cross-compiler |
| `glslangValidator` | GLSL, HLSL | SPIR-V, AST dump | Developer tool for shader validation |

---

## L.3 Key Paths in the Linux Stack

```
WGSL (browser/wgpu)
  → Tint or naga → SPIR-V
    → vk_spirv_to_nir() → NIR → ACO/LLVM → GPU ISA

GLSL (native OpenGL / Vulkan)
  → glslang → SPIR-V (Vulkan path)
  → glsl_to_nir() → NIR → pipe driver (OpenGL path)

HLSL (DXVK / VKD3D-Proton / UE5)
  → DXC → SPIR-V
    → vk_spirv_to_nir() → NIR → ACO/LLVM → GPU ISA

GDShader (Godot 4)
  → GLSL (internal transpile) → glslang → SPIR-V
    → vk_spirv_to_nir() → NIR → GPU ISA
```

*[Full cross-reference table and version notes to be added during editing pass.]*

---

## References

- [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools) — spirv-val, spirv-opt, spirv-dis, spirv-as
- [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) — SPIR-V reflection and cross-compilation
- [glslang](https://github.com/KhronosGroup/glslang) — Reference GLSL/HLSL compiler
- [DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler) — DXC SPIR-V backend
- [Mesa vk_spirv_to_nir](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/compiler/spirv/vtn_private.h) — Mesa SPIR-V front end

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
