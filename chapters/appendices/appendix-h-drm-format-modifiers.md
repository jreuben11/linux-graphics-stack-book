&lt;!-- Reference baseline: Linux kernel 6.12 / Mesa 25.1 / June 2026 --&gt;
&lt;!-- Primary source: include/uapi/drm/drm_fourcc.h (Linux 6.12) --&gt;
&lt;!-- https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_fourcc.h --&gt;

# Appendix H: DRM Format Modifier Reference

**Audience**: All readers — driver developers, graphics application developers, and browser/web platform engineers who need to identify, understand, or negotiate DRM format modifiers in production code.

DRM format modifiers are 64-bit tokens embedded in every GPU buffer descriptor that crosses an API boundary — from a `gbm_bo` allocation to a Vulkan image export to a KMS framebuffer. They encode the exact memory layout: whether a surface is linearly strided, stored in a hardware tiling pattern, or decorated with a vendor-specific lossless compression auxiliary surface. Getting the modifier wrong means either silent visual corruption (when a display engine scans out a tiled buffer expecting linear) or a needless GPU copy (when a compositor falls back to blitting instead of direct scanout because the modifier was not pre-negotiated). This appendix consolidates every production-relevant modifier into a single vendor-grouped reference, adds a step-by-step negotiation flowchart showing how GBM, EGL, `zwp_linux_dmabuf_feedback_v1`, and `VK_EXT_image_drm_format_modifier` converge on a shared modifier, and provides a cross-driver sharing compatibility matrix so developers can determine at a glance which modifier combinations permit zero-copy DMA-BUF hand-off between render driver, display engine, and media decoder.

---

## Table of Contents

1. [Modifier Encoding Structure](#1-modifier-encoding-structure)
2. [`DRM_FORMAT_MOD_LINEAR` — The Baseline Modifier](#2-drm_format_mod_linear--the-baseline-modifier)
3. [AMD Format Modifiers](#3-amd-format-modifiers)
   - [3a. AMD Modifier Encoding](#3a-amd-modifier-encoding)
   - [3b. AMD Modifier Entries](#3b-amd-modifier-entries)
4. [Intel Format Modifiers](#4-intel-format-modifiers)
   - [4a. Intel Modifier Entries (selected)](#4a-intel-modifier-entries-selected)
   - [4b. Intel Modifier Quick-Reference Table (complete)](#4b-intel-modifier-quick-reference-table-complete)
5. [NVIDIA Format Modifiers](#5-nvidia-format-modifiers)
   - [5a. NVIDIA Modifier Encoding](#5a-nvidia-modifier-encoding)
   - [5b. NVIDIA Modifier Entries](#5b-nvidia-modifier-entries)
6. [ARM AFBC Format Modifiers](#6-arm-afbc-format-modifiers)
   - [6a. AFBC Modifier Encoding](#6a-afbc-modifier-encoding)
   - [6b. AFBC Modifier Entries](#6b-afbc-modifier-entries)
7. [Modifier Negotiation Flowchart](#7-modifier-negotiation-flowchart)
8. [Cross-Driver DMA-BUF Sharing Compatibility Matrix](#8-cross-driver-dma-buf-sharing-compatibility-matrix)
9. [Cross-Reference Index](#9-cross-reference-index)
10. [Integrations](#integrations)
11. [References](#references)

---

## 1. Modifier Encoding Structure

All DRM format modifiers are 64-bit unsigned integers. The encoding is defined by the `fourcc_mod_code` macro in `include/uapi/drm/drm_fourcc.h` ([source](https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_fourcc.h)):

```c
/* include/uapi/drm/drm_fourcc.h */
#define fourcc_mod_code(vendor, val) \
    ((((___u64)DRM_FORMAT_MOD_VENDOR_##vendor) << 56) | ((val) & 0x00ffffffffffffffULL))
```

This places the vendor code in the high 8 bits (bits 63–56) and the vendor-defined payload in the low 56 bits (bits 55–0):

```
Bits 63–56 (8 bits):  DRM_FORMAT_MOD_VENDOR_* code
Bits 55–0  (56 bits): vendor-defined payload
```

The complete set of registered vendor codes from `drm_fourcc.h` (Linux 6.12):

| Constant | Value | Used by |
|---|---|---|
| `DRM_FORMAT_MOD_VENDOR_NONE` | `0x00` | `DRM_FORMAT_MOD_LINEAR` |
| `DRM_FORMAT_MOD_VENDOR_INTEL` | `0x01` | i915, xe |
| `DRM_FORMAT_MOD_VENDOR_AMD` | `0x02` | amdgpu |
| `DRM_FORMAT_MOD_VENDOR_NVIDIA` | `0x03` | nouveau, nvidia-open |
| `DRM_FORMAT_MOD_VENDOR_SAMSUNG` | `0x04` | Exynos |
| `DRM_FORMAT_MOD_VENDOR_QCOM` | `0x05` | msm |
| `DRM_FORMAT_MOD_VENDOR_VIVANTE` | `0x06` | etnaviv |
| `DRM_FORMAT_MOD_VENDOR_BROADCOM` | `0x07` | vc4, v3d |
| `DRM_FORMAT_MOD_VENDOR_ARM` | `0x08` | Panfrost, Panthor, display controllers |
| `DRM_FORMAT_MOD_VENDOR_ALLWINNER` | `0x09` | sun4i |
| `DRM_FORMAT_MOD_VENDOR_AMLOGIC` | `0x0a` | meson |
| `DRM_FORMAT_MOD_VENDOR_MTK` | `0x0b` | MediaTek |
| `DRM_FORMAT_MOD_VENDOR_APPLE` | `0x0c` | AGX |

A critical distinction governs how modifier values are used in practice. Intel modifiers and `DRM_FORMAT_MOD_LINEAR` are **fixed constants** — each named macro expands to a single, invariant 64-bit value that can be hardcoded in compatibility tables. AMD, NVIDIA, and ARM AFBC modifiers, by contrast, are **bitfield-packed values computed by macros**. An AMD modifier carries hardware topology information (pipe count, render-backend count) that varies per GPU instance; no single hex constant represents "AMD GFX9 DCC." The tables in this appendix therefore list the macro signature and key field values rather than a single hex literal for these three vendors. Only Intel modifiers and `DRM_FORMAT_MOD_LINEAR` have entries in the "fixed hex" style.

Cross-reference: Ch4 §7 covers the authoritative modifier architecture explanation; Ch2 §3 covers ADDFB2 modifier advertisement via KMS.

---

## 2. `DRM_FORMAT_MOD_LINEAR` — The Baseline Modifier

| Field | Value |
|---|---|
| **Macro** | `DRM_FORMAT_MOD_LINEAR` = `fourcc_mod_code(NONE, 0)` |
| **Modifier value** | `0x0000000000000000` (fixed constant) |
| **Vendor** | NONE (vendor code `0x00`) |
| **GPU generation** | All |
| **Use cases** | Import/export baseline; CPU readback; cross-vendor DMA-BUF sharing; display scanout when hardware supports linear stride; software renderers |
| **Memory layout** | Rows stored left-to-right, top-to-bottom with a uniform stride; no tiling, no compression, no auxiliary surfaces. CPU-accessible directly via `mmap` at the DRM buffer offset. |
| **Chapter introduction** | Ch4 §7 |
| **Known limitations** | No hardware compression; scanout performance degraded on tiled-native GPUs that must detile on the fly; some display engines require minimum stride alignment (usually 64 bytes). Not used by default for GPU rendering on any discrete GPU after GCN/Gen8. Used by Wayland clients that allocate shared memory buffers (`wl_shm`) with `WL_SHM_FORMAT_*` and import via linux-dmabuf. |

`DRM_FORMAT_MOD_LINEAR` is the lingua franca of DMA-BUF sharing. Every driver that advertises modifier support must accept linear as a valid import format. It is the fallback position in every negotiation flowchart: if no tiled or compressed modifier is acceptable to all parties, linear is the guaranteed path to a zero-error import, at the cost of a GPU copy from the native tiled allocation. Compositors commonly maintain two buffer pools — a tiled render pool in the driver's native layout, and a linear export pool — and blit between them when direct scanout is not available.

---

## 3. AMD Format Modifiers

### 3a. AMD Modifier Encoding

AMD modifiers are constructed by OR-ing field values into the `AMD_FMT_MOD` base value. The payload is a packed bitfield defined in `include/uapi/drm/drm_fourcc.h`. Each field has a corresponding `AMD_FMT_MOD_SET(field, value)` helper macro that shifts and masks the value into the correct bit range. The complete field table (Linux 6.12):

| Field constant | Bit range | Description |
|---|---|---|
| `AMD_FMT_MOD_TILE_VERSION` | bits 0–7 (mask `0xFF`) | GFX family version (see constants below) |
| `AMD_FMT_MOD_TILE` | bits 8–12 (mask `0x1F`) | Swizzle mode index matching `AMDGPU_TILING_SWIZZLE_MODE_*` |
| `AMD_FMT_MOD_DCC` | bit 13 | Delta Color Compression enabled |
| `AMD_FMT_MOD_DCC_RETILE` | bit 14 | DCC surface uses a separate display-compatible retiled copy |
| `AMD_FMT_MOD_DCC_PIPE_ALIGN` | bit 15 | DCC metadata aligned per-pipe |
| `AMD_FMT_MOD_DCC_INDEPENDENT_64B` | bit 16 | Each 64-byte block independently decompressible |
| `AMD_FMT_MOD_DCC_INDEPENDENT_128B` | bit 17 | Each 128-byte block independently decompressible |
| `AMD_FMT_MOD_DCC_MAX_COMPRESSED_BLOCK` | bits 18–19 | Maximum compressed block size: 0=64 B, 1=128 B, 2=256 B |
| `AMD_FMT_MOD_DCC_CONSTANT_ENCODE` | bit 20 | Solid-colour DCC encoding (GFX11+) |
| `AMD_FMT_MOD_PIPE_XOR_BITS` | bits 21–23 (mask `0x7`) | Pipe XOR bits for bank-swizzle |
| `AMD_FMT_MOD_BANK_XOR_BITS` | bits 24–26 (mask `0x7`) | Bank XOR bits for bank-swizzle |
| `AMD_FMT_MOD_PACKERS` | bits 27–29 (mask `0x7`) | Number of DCC packers (GFX10+) |
| `AMD_FMT_MOD_RB` | bits 30–32 (mask `0x7`) | Render back-end count (GFX9 DCC) |
| `AMD_FMT_MOD_PIPE` | bits 33–35 (mask `0x7`) | Pipe count (GFX9 DCC) |

The `TILE_VERSION` constants that select the GPU family (Linux 6.12, verified against `drm_fourcc.h`):

```c
/* include/uapi/drm/drm_fourcc.h */
#define AMD_FMT_MOD_TILE_VER_GFX9           1   /* Vega, Raven APU */
#define AMD_FMT_MOD_TILE_VER_GFX10          2   /* RDNA 1 — Navi 10/14 */
#define AMD_FMT_MOD_TILE_VER_GFX10_RBPLUS   3   /* RDNA 2 — Navi 21/22/23 */
#define AMD_FMT_MOD_TILE_VER_GFX11          4   /* RDNA 3 — Navi 31/32/33 */
#define AMD_FMT_MOD_TILE_VER_GFX12          5   /* RDNA 4 */
```

**Critical**: `GFX11 = 4`, not `3`. The value `3` is occupied by `GFX10_RBPLUS` (RDNA 2), which adds the additional render-backend-plus pipe configuration found on higher-end RDNA 2 products such as the Navi 21 (RX 6800 XT) but absent from mid-range RDNA 2 parts. Any code that hard-codes `TILE_VERSION == 3` for "GFX11" is incorrect; `TILE_VERSION == 4` must be used.

Because the modifier is a packed bitfield encoding hardware topology discovered at driver init time, there is no single canonical hex value for, say, "GFX9 DCC with 4 pipes and 2 render backends." The value depends on the actual GPU instance. The amdgpu kernel driver discovers the hardware configuration in `amdgpu_device_init()` and uses it to build the modifier list advertised to userspace via `amdgpu_dm_plane_format_mod_supported()` in `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_plane.c`. The RADV Vulkan driver queries these modifiers at device initialisation in `src/amd/vulkan/radv_image.c` (Mesa repository).

GFX12 support (RDNA 4, introduced with the RX 9070 series) landed in the amdgpu kernel driver during the 6.13/6.14 merge window. The `amdgpu_dm_add_gfx12_modifiers()` helper adds GFX12-specific swizzle modes to the plane's modifier list; refer to the current `amdgpu_dm_plane.c` for the exact bitfield combinations, as GFX12 does not use the DCC retile mechanism from older generations.

### 3b. AMD Modifier Entries

**Entry 1: AMD tiled render target (GFX9 64K_R_X, no DCC)**

| Field | Value |
|---|---|
| **Macro** | `AMD_FMT_MOD \| AMD_FMT_MOD_SET(TILE_VERSION, AMD_FMT_MOD_TILE_VER_GFX9) \| AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_R_X)` with `DCC = 0` |
| **Modifier value** | Computed — see field table above |
| **Vendor** | AMD (`0x02`) |
| **GPU generation** | GFX9 (Vega, Raven) and later |
| **Use cases** | Rendering; scanout via PRIME (display controller reads tiled); no compression overhead for small targets |
| **Memory layout** | 64 KB tiles with R-XOR swizzle pattern; improves L2 cache locality relative to linear but CPU-accessible after detile. No auxiliary metadata surface. |
| **Chapter introduction** | Ch5 §8 |
| **Known limitations** | Cannot be exported to non-AMD drivers; display engine requires tiling-aware programming. Not valid for DMA-BUF import on non-amdgpu devices. |

**Entry 2: AMD GFX9 DCC (Delta Color Compression, no retile)**

| Field | Value |
|---|---|
| **Macro** | `AMD_FMT_MOD` with `TILE_VERSION = GFX9`, `DCC = 1`, `DCC_RETILE = 0`, hardware-specific `PIPE`, `RB`, `PIPE_XOR_BITS`, `BANK_XOR_BITS` |
| **Modifier value** | Computed from hardware topology; driver enumerates valid combinations via `amdgpu_dm_plane_format_mod_supported()` |
| **Vendor** | AMD (`0x02`) |
| **GPU generation** | GFX9 (Vega, Raven APU) |
| **Use cases** | Rendering and scanout on the same GPU; lossless compression saves bandwidth on high-resolution render targets |
| **Memory layout** | 64 KB swizzled colour surface paired with a separate DCC metadata surface; metadata encodes the per-64-byte-block compression state (clear/compressed/uncompressed). Both surfaces must be allocated contiguously and included in any DMA-BUF export. |
| **Chapter introduction** | Ch5 §8 |
| **Known limitations** | DCC metadata surface is not importable on another vendor's driver. Scanout requires a display controller that understands DCC (DCN1 and later on Vega). GFX9 DCC metadata format differs from GFX10; modifier encoding reflects hardware topology (pipe/RB counts) and is therefore hardware-specific, not portable. |

**Entry 3: AMD GFX10/GFX11 DCC with retile (`DCC_RETILE = 1`)**

| Field | Value |
|---|---|
| **Macro** | `AMD_FMT_MOD` with `TILE_VERSION` set to `GFX10`, `GFX10_RBPLUS`, or `GFX11`; `DCC = 1`; `DCC_RETILE = 1`; `DCC_INDEPENDENT_64B`, `DCC_INDEPENDENT_128B`, `DCC_CONSTANT_ENCODE` fields set per hardware configuration |
| **Modifier value** | Computed; see `amdgpu_modifier_gfx10()`, `amdgpu_modifier_gfx10_rbplus()`, and `amdgpu_modifier_gfx11()` helpers in `drivers/gpu/drm/amd/amdgpu/amdgpu_display.c` |
| **Vendor** | AMD (`0x02`) |
| **GPU generation** | GFX10 (RDNA 1 — Navi 10/14), GFX10_RBPLUS (RDNA 2 — Navi 21/22/23), GFX11 (RDNA 3 — Navi 31/32/33) |
| **Use cases** | Rendering and scanout with maximum compression; `DCC_RETILE = 1` indicates that a display-compatible DCC copy is stored alongside the render-optimised copy, enabling DCN2+ to scan out the compressed surface directly |
| **Memory layout** | Two swizzle patterns in one allocation: a pipe-aligned render-optimised 64 KB-swizzle surface and a pipe-aligned display-compatible copy; DCC metadata covers both. GFX11 adds `DCC_CONSTANT_ENCODE` for solid-colour block encoding where pixel data need not be stored. |
| **Chapter introduction** | Ch5 §8 |
| **Known limitations** | Larger allocation footprint due to the retile copy. Not importable on non-AMD hardware. A retile blit is required when render and display topology differ. |

---

## 4. Intel Format Modifiers

Intel modifiers are fixed constants of the form `fourcc_mod_code(INTEL, N)` where `N` is a small integer. The Intel vendor code is `0x01`, so the high byte of all Intel modifiers is `0x01`, giving a base value of `0x0100000000000000`. Unlike AMD, NVIDIA, or ARM AFBC, there are no bitfield-packed variants — each named constant corresponds to exactly one memory layout, deterministic across all hardware instances.

This property makes Intel modifiers uniquely suitable for static compatibility tables: a modifier received from one i915 instance is byte-for-byte identical on any other i915 or xe instance. The `intel_modifier_info[]` table in `drivers/gpu/drm/i915/display/intel_fb.c` maps each constant to the display engine programming parameters, and `anv_image.c` in Mesa (`src/intel/vulkan/anv_image.c`) drives modifier selection for the ANV Vulkan driver.

### 4a. Intel Modifier Entries (selected)

**Entry 1: `I915_FORMAT_MOD_X_TILED`**

| Field | Value |
|---|---|
| **Macro** | `I915_FORMAT_MOD_X_TILED` = `fourcc_mod_code(INTEL, 1)` |
| **Modifier value** | `0x0100000000000001` |
| **Vendor** | Intel (`0x01`) |
| **GPU generation** | Gen6 (Sandy Bridge) through current; legacy path |
| **Use cases** | Scanout and rendering on pre-Gen8 hardware; some legacy media paths; accepted for import on all i915/xe hardware |
| **Memory layout** | Surface stored in 512-byte-wide, 8-row-tall tiles (X-tile pattern); major-order tiling with good horizontal locality. No compression metadata. |
| **Chapter introduction** | Ch5 §9 |
| **Known limitations** | No compression support. Deprecated for new code; Y-tiled and Tile4 preferred on Gen8+. |

**Entry 2: `I915_FORMAT_MOD_Y_TILED`**

| Field | Value |
|---|---|
| **Macro** | `I915_FORMAT_MOD_Y_TILED` = `fourcc_mod_code(INTEL, 2)` |
| **Modifier value** | `0x0100000000000002` |
| **Vendor** | Intel (`0x01`) |
| **GPU generation** | Gen6–Gen11 (Sandy Bridge through Ice Lake) |
| **Use cases** | Rendering and scanout; default tiling mode for Gen8–Gen11 render targets |
| **Memory layout** | 128-byte-wide, 32-row-tall tiles (Y-tile); better vertical cache locality than X-tile; 4 KB tile aligned. No compression. |
| **Chapter introduction** | Ch5 §9 |
| **Known limitations** | Deprecated from Gen12 onward, replaced by Tile4 (`I915_FORMAT_MOD_4_TILED`). Not valid for CCS on Gen12+. |

**Entry 3: `I915_FORMAT_MOD_Y_TILED_CCS`**

| Field | Value |
|---|---|
| **Macro** | `I915_FORMAT_MOD_Y_TILED_CCS` = `fourcc_mod_code(INTEL, 4)` |
| **Modifier value** | `0x0100000000000004` |
| **Vendor** | Intel (`0x01`) |
| **GPU generation** | Gen9–Gen11 (Skylake through Ice Lake) |
| **Use cases** | Lossless render compression on Gen9–Gen11; required for full GPU bandwidth on these generations |
| **Memory layout** | Y-tiled main surface paired with a CCS (Color Control Surface) auxiliary surface allocated immediately following the main surface; the CCS stores per-4 KB-tile clear/compressed state. Both surfaces must be included in any DMA-BUF export that carries the modifier. |
| **Chapter introduction** | Ch5 §9 |
| **Known limitations** | Cannot be exported via DMA-BUF to another vendor's driver — the CCS metadata format is Intel-private. A resolved surface (produced by a BLORP resolve pass) must be exported as `I915_FORMAT_MOD_Y_TILED` or `DRM_FORMAT_MOD_LINEAR` for cross-driver sharing. Cannot be scanned out directly from Gen12+ hardware. |

**Entry 4: `I915_FORMAT_MOD_4_TILED`**

| Field | Value |
|---|---|
| **Macro** | `I915_FORMAT_MOD_4_TILED` = `fourcc_mod_code(INTEL, 9)` |
| **Modifier value** | `0x0100000000000009` |
| **Vendor** | Intel (`0x01`) |
| **GPU generation** | Gen12 (Tiger Lake, Alder Lake, Raptor Lake) and Xe (Arc Alchemist, Battlemage) |
| **Use cases** | Default render target and scanout tiling for Gen12+; replaces Y-tiling for all new code |
| **Memory layout** | 4 KB tiles with 64-byte cache-line-aligned rows inside each tile (Tile4 format); better suited to the 128-byte L3 cache lines on Xe architecture. No compression metadata. |
| **Chapter introduction** | Ch5 §9 |
| **Known limitations** | Not compatible with Gen11 or older hardware. Cross-driver import requires the importer to understand the Tile4 layout (not universally supported outside the Intel driver stack). |

**Entry 5: `I915_FORMAT_MOD_4_TILED_DG2_RC_CCS`**

| Field | Value |
|---|---|
| **Macro** | `I915_FORMAT_MOD_4_TILED_DG2_RC_CCS` = `fourcc_mod_code(INTEL, 10)` |
| **Modifier value** | `0x010000000000000a` |
| **Vendor** | Intel (`0x01`) |
| **GPU generation** | DG2 / Xe_HPG (Arc Alchemist A-series) and later discrete |
| **Use cases** | Render compression on Xe discrete GPUs; required for full render bandwidth on Arc A-series |
| **Memory layout** | Tile4 main surface with a CCS auxiliary surface stored in a **separate GEM object** (unlike Gen12 iGPU, where the CCS is appended to the main surface); the separate-object model changes the DMA-BUF export to carry two file descriptors. The DG2 CCS format is updated relative to the Gen12 CCS. |
| **Chapter introduction** | Ch5 §9 |
| **Known limitations** | CCS metadata is Intel-private; cannot be exported to another vendor's driver. Media decode via VA-API requires resolve to `I915_FORMAT_MOD_4_TILED` before passing surfaces to non-Intel decode hardware. Meteor Lake (`I915_FORMAT_MOD_4_TILED_MTL_RC_CCS`), Lunar Lake (`I915_FORMAT_MOD_4_TILED_LNL_CCS`), and Battlemage (`I915_FORMAT_MOD_4_TILED_BMG_CCS`) each have their own distinct CCS formats; they are not interchangeable. |

### 4b. Intel Modifier Quick-Reference Table (complete)

All Intel modifier hex values share the prefix `0x010000000000`. The abbreviated notation `...XXXX` drops this prefix; the full value is `0x010000000000XXXX`.

| Modifier | Hex suffix | Generation | Compression | Scanout |
|---|---|---|---|---|
| `I915_FORMAT_MOD_X_TILED` | `0001` | Gen6+ | None | Yes |
| `I915_FORMAT_MOD_Y_TILED` | `0002` | Gen6–Gen11 | None | Yes |
| `I915_FORMAT_MOD_Yf_TILED` | `0003` | Gen9–Gen11 | None | No (render only) |
| `I915_FORMAT_MOD_Y_TILED_CCS` | `0004` | Gen9–Gen11 | RC (private) | Partial |
| `I915_FORMAT_MOD_Yf_TILED_CCS` | `0005` | Gen9–Gen11 | RC (private) | No |
| `I915_FORMAT_MOD_Y_TILED_GEN12_RC_CCS` | `0006` | Gen12 iGPU | RC (private) | Yes (DCN) |
| `I915_FORMAT_MOD_Y_TILED_GEN12_MC_CCS` | `0007` | Gen12 iGPU | MC (private) | Yes |
| `I915_FORMAT_MOD_Y_TILED_GEN12_RC_CCS_CC` | `0008` | Gen12 iGPU | RC+CC | Yes |
| `I915_FORMAT_MOD_4_TILED` | `0009` | Gen12+ / Xe | None | Yes |
| `I915_FORMAT_MOD_4_TILED_DG2_RC_CCS` | `000a` | DG2 / Alchemist | RC (private) | No† |
| `I915_FORMAT_MOD_4_TILED_DG2_MC_CCS` | `000b` | DG2 / Alchemist | MC (private) | No |
| `I915_FORMAT_MOD_4_TILED_DG2_RC_CCS_CC` | `000c` | DG2 / Alchemist | RC+CC | No |
| `I915_FORMAT_MOD_4_TILED_MTL_RC_CCS` | `000d` | Meteor Lake | RC (private) | Yes |
| `I915_FORMAT_MOD_4_TILED_MTL_MC_CCS` | `000e` | Meteor Lake | MC (private) | Yes |
| `I915_FORMAT_MOD_4_TILED_MTL_RC_CCS_CC` | `000f` | Meteor Lake | RC+CC | Yes |
| `I915_FORMAT_MOD_4_TILED_LNL_CCS` | `0010` | Lunar Lake | RC (private) | Yes |
| `I915_FORMAT_MOD_4_TILED_BMG_CCS` | `0011` | Battlemage | RC (private) | Yes |

Abbreviations: RC = render compression, MC = media compression, CC = clear colour encoding.

† DG2 discrete scanout policy: cross-check with the xe driver's display engine documentation at publication time; discrete display scanout of compressed surfaces depends on display engine capability exposed via the KMS plane's supported modifier list.

`I915_FORMAT_MOD_4_TILED_LNL_CCS` (`fourcc_mod_code(INTEL, 16)`) and `I915_FORMAT_MOD_4_TILED_BMG_CCS` (`fourcc_mod_code(INTEL, 17)`) were introduced in Linux 6.10 and became officially supported (i.e., the `force_probe` requirement was dropped) in Linux 6.12, which enabled Lunar Lake and Battlemage as fully supported platforms for the xe driver. Verify that CCS scanout is fully operational and not gated behind feature flags at the reference kernel before citing these as production-ready modifiers.

---

## 5. NVIDIA Format Modifiers

### 5a. NVIDIA Modifier Encoding

NVIDIA modifiers use the `DRM_FORMAT_MOD_NVIDIA_BLOCK_LINEAR_2D` macro family. The production path for Tegra (K1 through Orin) and desktop discrete (Fermi through Ada Lovelace) is:

```c
/* include/uapi/drm/drm_fourcc.h */
#define DRM_FORMAT_MOD_NVIDIA_BLOCK_LINEAR_2D(c, s, g, k, h) \
    fourcc_mod_code(NVIDIA, (0x10 | \
                             ((h) & 0xf) | \
                             (((k) & 0xff) << 12) | \
                             (((g) & 0x3) << 20) | \
                             (((s) & 0x1) << 22) | \
                             (((s) & 0x6) << 25) | \
                             (((c) & 0x7) << 23)))
```

The five parameters encode:

- **`h`** — log₂(GOB height / 8): `0` = 1 GOB (8 rows), `1` = 16 rows, `2` = 32 rows, `3` = 64 rows, `4` = 128 rows. This is the most important parameter for display compatibility.
- **`k`** — NVIDIA surface kind field (hardware-generation-specific; `0` = generic block-linear, `0xfe` = pitch/linear, `0xb5` = ARGB8 block-linear on certain generations).
- **`g`** — compression generation: `0` = uncompressed, `1` = generic compression, `2` = sector layout variant.
- **`s`** — sector layout parameter. Note that the three-bit `s` field is intentionally split across two non-contiguous positions in the macro body: `(s & 0x1) << 22` and `(s & 0x6) << 25`. This split is not a typo; it reflects the way the hardware descriptor encodes the field.
- **`c`** — compression kind.

The simplified helper for the most common Tegra and desktop discrete layout uses all-zero compression and kind parameters, exposing only the GOB height:

```c
/* include/uapi/drm/drm_fourcc.h */
#define DRM_FORMAT_MOD_NVIDIA_16BX2_BLOCK(v) \
    DRM_FORMAT_MOD_NVIDIA_BLOCK_LINEAR_2D(0, 0, 0, 0, (v))
```

The `16BX2` name refers to the 16-byte-wide × 2-row GOB cell that forms the building block of NVIDIA's block-linear tile. The `v` parameter selects the number of GOB rows stacked vertically per tile.

The NVIDIA modifier space is significantly more complex than AMD or Intel at the hardware level. The `k` (kind) parameter maps to NVIDIA's internal surface kind table, which is hardware-generation-specific and only partially documented outside NVIDIA's own SDK. The kernel-visible modifiers described here are the subset observable at the DRM/GBM level as documented in the kernel source. Proprietary surface kinds used internally by the NVIDIA user-mode driver are out of scope.

### 5b. NVIDIA Modifier Entries

**Entry 1: `DRM_FORMAT_MOD_NVIDIA_16BX2_BLOCK(0)` — single-GOB height (Tegra)**

| Field | Value |
|---|---|
| **Macro** | `DRM_FORMAT_MOD_NVIDIA_16BX2_BLOCK(0)` |
| **Modifier value** | Computed: `fourcc_mod_code(NVIDIA, 0x10 | 0)` |
| **Vendor** | NVIDIA (`0x03`) |
| **GPU generation** | Tegra K1 (T124) and later Tegra SoCs; Fermi discrete for single-GOB surfaces |
| **Use cases** | Tegra display scanout; single-layer 2D surfaces on Tegra; GBM allocation for `wl_drm` on Tegra-based systems |
| **Memory layout** | 16×2 GOB tiles; each GOB is 64 bytes wide × 8 rows; height = 1 GOB (8 rows). Successive GOBs are laid out in Z-order within 512-byte-aligned 2D tiles. |
| **Chapter introduction** | Ch10 §3 |
| **Known limitations** | GOB height must match what the display controller expects; a height mismatch causes display corruption or a blank screen. Not importable on non-NVIDIA hardware. Tegra display engine only accepts specific GOB heights matched to the SoC generation. |

**Entry 2: `DRM_FORMAT_MOD_NVIDIA_16BX2_BLOCK(4)` — 16-GOB height (Turing/Ampere/Ada desktop)**

| Field | Value |
|---|---|
| **Macro** | `DRM_FORMAT_MOD_NVIDIA_16BX2_BLOCK(4)` |
| **Modifier value** | Computed: `fourcc_mod_code(NVIDIA, 0x10 | 4)` |
| **Vendor** | NVIDIA (`0x03`) |
| **GPU generation** | Turing (RTX 2000 series), Ampere (RTX 3000), Ada Lovelace (RTX 4000) |
| **Use cases** | Render target allocation by NVK (nouveau Vulkan) and nvidia-open proprietary WSI; the default block-linear layout for desktop discrete NVIDIA |
| **Memory layout** | 16×2 GOB tiles with GOB height = 16 GOBs × 8 rows = 128 rows total. Z-order layout within the tile improves GPU L2 cache utilisation for 2D spatial access patterns. No compression (`c = 0`). |
| **Chapter introduction** | Ch10 §3 |
| **Known limitations** | Not directly importable on amdgpu, i915/xe, or Panfrost — block-linear is NVIDIA-proprietary. Cross-device sharing requires re-encoding to `DRM_FORMAT_MOD_LINEAR` before DMA-BUF export. NVK's linux-dmabuf negotiation path (see Section 7 flowchart) falls back to linear when a non-NVIDIA consumer is detected. The list of modifiers advertised by nouveau is maintained in `drivers/gpu/drm/nouveau/nouveau_display.c`; NVK modifier selection is in `src/nouveau/vulkan/nvk_image.c` (Mesa). |

---

## 6. ARM AFBC Format Modifiers

### 6a. AFBC Modifier Encoding

ARM Frame Buffer Compression (AFBC) modifiers use a two-level encoding defined in `include/uapi/drm/drm_fourcc.h`. The outer wrapper is:

```c
/* include/uapi/drm/drm_fourcc.h */
#define DRM_FORMAT_MOD_ARM_AFBC(__afbc_mode) \
    DRM_FORMAT_MOD_ARM_CODE(DRM_FORMAT_MOD_ARM_TYPE_AFBC, __afbc_mode)
```

where `__afbc_mode` is constructed by OR-ing exactly one block size constant with zero or more feature bits.

**Block size constants** (exactly one must be present; the block size governs the superblock pixel footprint):

| Constant | Value | Tile shape |
|---|---|---|
| `AFBC_FORMAT_MOD_BLOCK_SIZE_16x16` | `1ULL` | 16×16 pixels per superblock |
| `AFBC_FORMAT_MOD_BLOCK_SIZE_32x8` | `2ULL` | 32×8 pixels per superblock |
| `AFBC_FORMAT_MOD_BLOCK_SIZE_64x4` | `3ULL` | 64×4 pixels per superblock (video decode, AFBC v1.3+) |
| `AFBC_FORMAT_MOD_BLOCK_SIZE_32x8_64x4` | `4ULL` | Mixed 32×8 and 64×4 (AFBC v1.3+) |

**Feature bits** (zero or more may be set; each adds a specific codec or layout property):

| Constant | Value | Meaning |
|---|---|---|
| `AFBC_FORMAT_MOD_YTR` | `1ULL << 4` | Luma-chroma colour space transform; improves compression of YCbCr content |
| `AFBC_FORMAT_MOD_SPLIT` | `1ULL << 5` | Block-split: each 16×16 superblock split into 4×4 sub-blocks for better small-region decode |
| `AFBC_FORMAT_MOD_SPARSE` | `1ULL << 6` | Sparse layout: superblocks may be non-contiguous in memory, enabling partial updates |
| `AFBC_FORMAT_MOD_CBR` | `1ULL << 7` | Copy-block restrict: limits inter-superblock copy references |
| `AFBC_FORMAT_MOD_TILED` | `1ULL << 8` | Superblock 8×8 macro-tiled layout for GPU rendering (Bifrost/Valhall render targets) |
| `AFBC_FORMAT_MOD_SC` | `1ULL << 9` | Solid-colour blocks: entirely solid-colour superblocks stored in the header with no body data |
| `AFBC_FORMAT_MOD_DB` | `1ULL << 10` | Double-buffer: two frame copies for zero-copy display update |
| `AFBC_FORMAT_MOD_BCH` | `1ULL << 11` | Buffer content hints: per-block metadata hints for decoder optimisation |
| `AFBC_FORMAT_MOD_USM` | `1ULL << 12` | Uncompressed storage mode: stores incompressible blocks without compression overhead |

The ARM AFBC Architecture Specification (ARM DDI 0591) [Ref 1] is the authoritative source for the superblock layout, header format, and feature bit semantics. The `AFBC_FORMAT_MOD_BCH` and `AFBC_FORMAT_MOD_USM` bits are defined in `drm_fourcc.h` but may not be advertised by any upstream Panfrost/Panthor driver at the Linux 6.12 baseline; verify which bits are live in `panfrost_format_modifier_supported()` before citing them as production-ready.

Note that **Lima (Mali Utgard, pre-Bifrost)** does not support AFBC. Mali Utgard predates the AFBC standard; the Lima open-source driver (`drivers/gpu/drm/lima/`) supports only `DRM_FORMAT_MOD_LINEAR`. AFBC is a capability of Bifrost (G31, G52, G72, G76) and Valhall (G57, G610, G710, G720) render GPUs driven by Panfrost/Panthor.

### 6b. AFBC Modifier Entries

**Entry 1: `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_16x16)` — baseline AFBC**

| Field | Value |
|---|---|
| **Macro** | `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_16x16)` = `DRM_FORMAT_MOD_ARM_AFBC(1ULL)` |
| **Modifier value** | Computed: `DRM_FORMAT_MOD_ARM_CODE(DRM_FORMAT_MOD_ARM_TYPE_AFBC, 1ULL)`; vendor ARM = `0x08` |
| **Vendor** | ARM (`0x08`) |
| **GPU generation** | Bifrost (G31, G52, G72, G76); Valhall (G57, G610, G710, G720); also supported by SoC display controllers including Mali-DP500/650, Rockchip VOP2, MediaTek RDMA, Qualcomm MDP |
| **Use cases** | Render target compression on Bifrost/Valhall; scanout by display controllers that decode AFBC natively; reduces memory bandwidth 30–50% on typical UI workloads |
| **Memory layout** | Surface divided into 16×16-pixel superblocks; each superblock has a 16-byte header (stored contiguously in a header region at the start of the allocation) followed by compressed pixel data at a body offset encoded in the header. A linear scan of the header region allows the decoder to locate any superblock's body data. |
| **Chapter introduction** | Ch6 §4 |
| **Known limitations** | Not importable on amdgpu, i915/xe, or NVIDIA drivers. Display controller must explicitly support AFBC. Qualifier: `YES(AFBC)` in the compatibility matrix requires both endpoints to implement AFBC with a matching feature set; not all SoC display engines support all feature bits. |

**Entry 2: `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_16x16 | AFBC_FORMAT_MOD_TILED | AFBC_FORMAT_MOD_SC)` — tiled AFBC with solid colour (Valhall render target)**

| Field | Value |
|---|---|
| **Macro** | `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_16x16 \| AFBC_FORMAT_MOD_TILED \| AFBC_FORMAT_MOD_SC)` |
| **Modifier value** | Computed: `DRM_FORMAT_MOD_ARM_AFBC(1ULL \| (1ULL << 8) \| (1ULL << 9))` |
| **Vendor** | ARM (`0x08`) |
| **GPU generation** | Valhall (G57, G610, G720); Panthor-driven hardware |
| **Use cases** | Default render compression on Panthor; the 8×8 macro-tile layout gives better GPU texture cache utilisation during rendering |
| **Memory layout** | Same 16×16-pixel superblock structure as baseline AFBC, but superblocks are further grouped into 8×8 macro-tiles optimised for GPU L2 cache access patterns. Superblocks that contain a solid colour (for example, a cleared colour attachment) are represented entirely in the header with no body payload (`SC` bit). |
| **Chapter introduction** | Ch6 §4 |
| **Known limitations** | The `TILED` bit is typically incompatible with display scanout on most current SoC display controllers that expect non-tiled AFBC. Panthor must resolve to non-tiled AFBC before hand-off to a display engine that lacks tiled AFBC decode. The Panfrost modifier selection logic is in `src/gallium/drivers/panfrost/pan_resource.c` (Mesa). |

**Entry 3: `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_32x8 | AFBC_FORMAT_MOD_YTR)` — wide-block YCbCr AFBC for video**

| Field | Value |
|---|---|
| **Macro** | `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_32x8 \| AFBC_FORMAT_MOD_YTR)` |
| **Modifier value** | Computed: `DRM_FORMAT_MOD_ARM_AFBC(2ULL \| (1ULL << 4))` |
| **Vendor** | ARM (`0x08`) |
| **GPU generation** | Bifrost and Valhall; also used by MediaTek display and Rockchip VOP2 for video output surfaces |
| **Use cases** | Video decode output surfaces; 32×8 superblocks align to 4:2:0 macroblock geometry; the YTR colour-space transform improves compression ratio on YCbCr content |
| **Memory layout** | 32×8-pixel superblocks with luma-chroma colour-space transform applied before compression. Header + body structure identical to 16×16 AFBC but with larger superblocks. The YTR transform is applied to each superblock independently before entropy coding. |
| **Chapter introduction** | Ch6 §4; Ch26 §5 |
| **Known limitations** | `YTR` requires a YCbCr pixel format; applying `YTR` to an RGB surface produces incorrect output. The `32x8` block size is incompatible with the `TILED` feature bit on most current hardware implementations. |

---

## 7. Modifier Negotiation Flowchart

The diagram below describes the end-to-end path by which a modifier is negotiated between a GPU render driver, a Wayland compositor, and the KMS display engine. The same modifier token must survive every API boundary from allocation to scanout; any substitution forces a GPU copy.

```
MODIFIER NEGOTIATION FLOW
─────────────────────────────────────────────────────────────────────────────

[1] KMS / DRM
     │  Compositor queries supported (format, modifier) pairs per KMS plane:
     │    DRM_IOCTL_MODE_GET_PLANE_RES → DRM_MODE_OBJECT_PLANE
     │    → plane IN_FORMATS blob → supported_modifiers[]
     ▼

[2] Wayland Compositor
     │  Compositor sends feedback to Wayland clients:
     │    zwp_linux_dmabuf_feedback_v1.format_table   ← mmap'd fd of (fmt,mod) pairs
     │    zwp_linux_dmabuf_feedback_v1.tranche_formats ← index array into table
     │    (introduced in linux-dmabuf-unstable-v1 version 4)
     ▼

[3] Client / GBM path
     │  Client calls:
     │    gbm_surface_create_with_modifiers2(dev, w, h, fmt, modifiers[], count)
     │    GBM selects first modifier the driver supports for rendering.
     │
     │  ┌─ Driver supports requested modifier? ──────────────┐
     │  │ YES                                          NO    │
     │  ▼                                             fall back to DRM_FORMAT_MOD_LINEAR
     │
[4] GBM ↔ Mesa DRI
     │  gbm_device_create() opens DRI driver
     │  Modifier support queried via __DRI_IMAGE.createImageWithModifiers
     │  Allocated gbm_bo carries selected modifier
     ▼

[5] EGL
     │  eglCreateWindowSurface(display, config, gbm_surface, NULL)
     │  EGL driver records modifier from gbm_bo_get_modifier()
     │  for later DMA-BUF export
     ▼

[6] Render + Export
     │  eglSwapBuffers() → wl_surface.commit
     │  Compositor calls:
     │    gbm_bo_get_fd()     → dma-buf fd
     │    gbm_bo_get_modifier() → modifier token
     │    zwp_linux_dmabuf_v1.create_immed(fd, width, height, format, modifier)
     ▼

         ─── ALTERNATIVE: Vulkan WSI path ───
[7] Vulkan driver
     │  vkGetPhysicalDeviceFormatProperties2(
     │      format, &VkDrmFormatModifierPropertiesListEXT)
     │    → enumerate driver-supported (format, modifier) pairs
     │  Intersect with compositor modifiers from step [2]
     │  vkCreateImage with VkImageDrmFormatModifierListCreateInfoEXT
     │    + VkExternalMemoryImageCreateInfo
     │  Export via VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT
     ▼

[8] KMS Import
     │  Compositor: drmPrimeFDToHandle(fd) → gem_handle
     │  DRM_IOCTL_MODE_ADDFB2 with (format + modifier)
     │
     │  ┌─ KMS ADDFB2 with modifier succeeds? ───────────────┐
     │  │ YES                                       NO       │
     │  ▼ direct scanout                     GPU composite → scanout BO
```

**Step-by-step narrative:**

**Step 1 — Compositor queries KMS.** Before advertising any modifiers to clients, the Wayland compositor must discover which `(format, modifier)` pairs the kernel display engine can scan out directly. The compositor calls `DRM_IOCTL_MODE_GET_PLANE_RES` to enumerate KMS planes and then retrieves the `IN_FORMATS` blob property from each plane object. This blob contains the set of pixel formats and, for each format, the list of modifiers the hardware display pipeline can accept. This query is the ground truth for direct scanout; any modifier not in this set will fail `DRM_IOCTL_MODE_ADDFB2` at step 8.

**Step 2 — Compositor advertises via linux-dmabuf feedback.** The compositor aggregates the KMS-supported modifiers and sends them to every connected Wayland client via the `zwp_linux_dmabuf_feedback_v1` protocol, introduced in version 4 of the linux-dmabuf protocol. The compositor sends a `format_table` event carrying a file descriptor that the client can `mmap` to access a tightly packed array of 16-byte `(uint32_t format, uint32_t padding, uint64_t modifier)` records. It then sends one or more `tranche_formats` events carrying arrays of 16-bit indices into the table, grouped into tranches of descending preference. The first tranche typically contains modifiers that allow the compositor to pass the buffer directly to KMS without a copy; later tranches contain modifiers requiring compositing. The feedback protocol replaces the simpler but less expressive `zwp_linux_dmabuf_v1.modifier` event from earlier protocol versions.

**Step 3 — Client allocates via GBM.** The client intersects the compositor-advertised modifier list with any application-side constraints and calls `gbm_surface_create_with_modifiers2(dev, width, height, format, modifiers[], count)`, passing the negotiated modifier list. GBM iterates the list and selects the first modifier that the underlying DRI driver supports for rendering. If no modifier in the list is supported for rendering, GBM falls back to `DRM_FORMAT_MOD_LINEAR`.

**Step 4 — GBM delegates to Mesa DRI.** The `gbm_device_create()` call opens the DRI driver and exposes modifier support via the `__DRI_IMAGE` extension's `createImageWithModifiers` entrypoint (`src/gbm/backends/dri/gbm_dri.c` in Mesa). The allocated `gbm_bo` carries the selected modifier in its metadata; it can be retrieved later with `gbm_bo_get_modifier()`.

**Step 5 — EGL wraps the GBM surface.** `eglCreateWindowSurface(display, config, gbm_surface, NULL)` creates an EGL surface backed by the GBM surface. The EGL Wayland platform driver (`src/egl/drivers/dri2/platform_wayland.c` in Mesa) queries the GBM BO's modifier and records it for DMA-BUF export.

**Step 6 — Render and export.** After the GPU completes rendering, `eglSwapBuffers()` presents the current back buffer and signals the compositor via `wl_surface.commit`. The compositor acquires the DMA-BUF file descriptor via `gbm_bo_get_fd()` and the modifier via `gbm_bo_get_modifier()`, then calls `zwp_linux_dmabuf_v1.create_immed` with all four parameters: fd, width, height, format, and modifier.

**Step 7 — Vulkan WSI path (alternative to GBM/EGL).** A native Vulkan application follows a parallel path. Before creating any image, the application queries `vkGetPhysicalDeviceFormatProperties2` with a `VkDrmFormatModifierPropertiesListEXT` structure chained onto `VkFormatProperties2`; this returns the list of modifiers the Vulkan driver supports for the given format. The application intersects this list with the modifiers received from the compositor at step 2. It creates a `VkImage` with `VkImageDrmFormatModifierListCreateInfoEXT` (chained onto `VkImageCreateInfo`) specifying the negotiated modifiers and `VkExternalMemoryImageCreateInfo` with `VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT`. The Vulkan driver allocates with the best available modifier from the list and later exports the memory as a DMA-BUF fd. The Vulkan WSI modifier path is implemented in `src/vulkan/wsi/wsi_common_drm.c` (Mesa) and the extension is specified at [Ref 3].

**Step 8 — Compositor imports and schedules KMS.** The compositor imports the DMA-BUF via `drmPrimeFDToHandle()` to obtain a GEM handle, then attempts `DRM_IOCTL_MODE_ADDFB2` specifying the format and modifier. If the modifier matches an entry in the KMS plane's supported modifier list from step 1, direct scanout is attempted and the buffer is scheduled for the next vblank without a GPU copy. If `ADDFB2` fails (because the modifier is not in the supported list, or because a hardware constraint such as stride alignment is violated), the compositor falls back to compositing the incoming buffer onto a scanout-compatible framebuffer via a GPU blit, then scheduling the composited result for KMS display.

The modifier selected at allocation time (step 3) must survive every API boundary intact. If a compositor imports a buffer with a modifier different from what was advertised, or if the Vulkan application exports a DMA-BUF with a modifier that was not included in the original `VkImageDrmFormatModifierListCreateInfoEXT`, the result is either a failed `ADDFB2` or silent display corruption.

---

## 8. Cross-Driver DMA-BUF Sharing Compatibility Matrix

The following table shows whether a buffer produced by the exporter can be consumed by the importer without an intermediate GPU copy. Rows are exporters (the render driver that produced the buffer); columns are importers (the driver consuming the buffer for display, media decode, or compute). For display importers, "consume" means direct KMS scanout via `ADDFB2`; for VA-API and compute, it means importing the DMA-BUF fd without a re-encode.

Status codes:
- **YES(*modifier*)** — sharing works natively with the named modifier; no copy needed.
- **LINEAR** — sharing requires re-encoding to `DRM_FORMAT_MOD_LINEAR` first (one GPU copy needed).
- **RESOLVE** — sharing requires a vendor-specific resolve pass before export (resolve + optional copy).
- **NO** — sharing not supported; no path exists even via linear (e.g., capability gap or security boundary).

| Exporter → Importer | amdgpu display (DCN) | i915/xe display | NVIDIA display (discrete) | Panfrost/Panthor display | VA-API decode (Mesa) | VA-API decode (NVIDIA) | ROCm/CUDA peer |
|---|---|---|---|---|---|---|---|
| **amdgpu (AMD_FMT_MOD DCC)** | YES(AMD_FMT_MOD) | LINEAR | LINEAR | LINEAR | LINEAR | NO | YES(AMD_FMT_MOD, same GPU) |
| **amdgpu (DRM_FORMAT_MOD_LINEAR)** | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) |
| **i915/xe (I915_FORMAT_MOD_4_TILED)** | LINEAR | YES(4_TILED) | LINEAR | LINEAR | YES(4_TILED, same iGPU) | NO | NO |
| **i915/xe (I915_FORMAT_MOD_4_TILED_DG2_RC_CCS)** | RESOLVE→LINEAR | YES(4_TILED_DG2_RC_CCS, same dGPU) | NO | NO | RESOLVE→LINEAR | NO | NO |
| **i915/xe (DRM_FORMAT_MOD_LINEAR)** | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | YES(LINEAR) | NO |
| **NVIDIA block-linear** | LINEAR | LINEAR | YES(block-linear, same GPU) | LINEAR | LINEAR | YES(block-linear, same GPU) | YES(block-linear, NVLink/same bus) |
| **ARM AFBC (16x16)** | LINEAR | LINEAR | LINEAR | YES(AFBC, compatible display) | LINEAR | NO | NO |
| **DRM_FORMAT_MOD_LINEAR** | YES | YES | YES | YES | YES | YES | Conditional† |

† ROCm/CUDA peer-to-peer import of a linear DMA-BUF from another vendor requires P2P DMA support (`pci_p2p_dma_supported()`), which is not guaranteed on all PCIe topologies. See Ch25 §6 and Ch4 §6 for the PRIME/P2P architecture.

**Key observations:**

1. **`DRM_FORMAT_MOD_LINEAR` is the only universally importable modifier.** Any cross-vendor pipeline should negotiate linear as the lowest-common-denominator fallback. The extra GPU blit from tiled to linear is almost always preferable to a protocol failure.

2. **CCS and DCC metadata are vendor-private.** All Intel CCS modifiers (`I915_FORMAT_MOD_*_CCS`) and AMD DCC modifiers (`AMD_FMT_MOD` with `DCC=1`) require a vendor-specific resolve pass before any cross-vendor import. There is no standardised metadata format for lossless compression auxiliary surfaces. Attempting to import a CCS or DCC surface on a different vendor's driver will produce incorrect output or an import error.

3. **AFBC is an ARM-ecosystem modifier.** AFBC is natively understood by ARM display controllers (Mali-DP, Rockchip VOP2, MediaTek RDMA) and ARM render GPUs (Bifrost/Valhall), but there is no import path on x86 drivers (amdgpu, i915, NVIDIA). Any pipeline that crosses the ARM/x86 boundary must transit through linear. The `YES(AFBC, compatible display)` entry in the Panfrost/Panthor column is qualified: it requires both the render GPU (Panfrost/Panthor) and the display controller to be on the same SoC with AFBC decode support.

4. **The amdgpu→i915 display sharing case** — common in multi-GPU laptops with an AMD iGPU or dGPU rendering to an Intel iGPU display engine — **requires linear or a separate scanout buffer.** The amdgpu driver cannot advertise its AMD_FMT_MOD tiled modifiers in the KMS plane's IN_FORMATS blob of the Intel display engine; the compositor handles this via the PRIME display path (Ch4 §5), maintaining a linear scanout surface on the display-side engine.

5. **NVIDIA block-linear is always opaque to non-NVIDIA importers.** Any Wayland compositor running on an NVIDIA render GPU must present via DMA-BUF linear (or a Vulkan WSI copy path) to a non-NVIDIA display engine. The NVK Vulkan driver's linux-dmabuf modifier negotiation path falls back to `DRM_FORMAT_MOD_LINEAR` when the compositor's feedback tranches do not include a NVIDIA block-linear modifier.

6. **Same-GPU sharing optimisations.** Several `YES` entries in the matrix are qualified "same GPU" or "same iGPU." For example, `YES(4_TILED, same iGPU)` for i915→VA-API decode means that the media decode accelerator on the same Intel iGPU can import a Tile4 surface directly, bypassing a linear conversion, because both the render engine and the media engine share the same GGTT and CCS topology. This optimisation is not available when render and decode are on different physical devices.

---

## 9. Cross-Reference Index

The following index maps each modifier group to the chapters where it is discussed in depth. Chapter and section numbers reflect the first edition chapter plan; update if the book is renumbered.

```
AMD DCC modifiers (AMD_FMT_MOD with DCC=1)
    Ch5 §8 — amdgpu tiling modes and DCC pipeline
    Ch18 §3 — RADV modifier selection and allocation
    Appendix H §3

ARM AFBC modifiers (DRM_FORMAT_MOD_ARM_AFBC)
    Ch6 §4 — Panfrost/Panthor tiling and AFBC render
    Ch20 §6 — linux-dmabuf modifier negotiation
    Appendix H §6

DRM_FORMAT_MOD_LINEAR
    Ch4 §7 — modifier architecture overview
    Ch20 §6 — cross-driver linear sharing
    Appendix H §2

Intel CCS modifiers (I915_FORMAT_MOD_*_CCS)
    Ch5 §9 — Intel display tiling and CCS
    Ch18 §4 — ANV modifier selection and CCS surface allocation
    Appendix H §4

Intel tiling modifiers (X-tiled, Y-tiled, Tile4)
    Ch5 §9 — Intel tiling generations
    Appendix H §4

Modifier negotiation (GBM / EGL / Vulkan WSI)
    Ch20 §6 — linux-dmabuf-feedback and EGL platform
    Ch24 §4 — Vulkan WSI modifier negotiation
    Appendix H §7

NVIDIA block-linear modifiers
    Ch10 §3 — NVK buffer allocation and block-linear
    Appendix H §5

Cross-driver DMA-BUF sharing
    Ch4 §4–5 — DMA-BUF and PRIME architecture
    Ch20 §6 — compositor DMA-BUF handling
    Appendix H §8

fourcc_mod_code / modifier encoding
    Ch4 §7 — authoritative encoding explanation
    Appendix H §1
```

---

## Integrations

This appendix draws on and extends material from the following chapters:

- **Ch2 §3** — KMS `ADDFB2` modifier advertisement: the kernel API pathway by which a modifier is declared when creating a KMS framebuffer object.
- **Ch4 §4–5** — DMA-BUF and PRIME: the buffer-sharing substrate that carries modifier tokens between drivers; the PRIME offload path that handles cross-vendor modifier incompatibility.
- **Ch4 §7** — Modifier architecture: the authoritative treatment of the `fourcc_mod_code` encoding, the `DRM_FORMAT_MOD_INVALID` sentinel, and the kernel-side infrastructure for modifier advertisement.
- **Ch5 §8–9** — AMD and Intel hardware-specific tiling and compression: the hardware mechanisms that AMD DCC and Intel CCS modifiers encode.
- **Ch6 §4** — Panfrost/Panthor AFBC render: the driver-level AFBC allocation and the Valhall tiling heap required for compressed render targets.
- **Ch10 §3** — NVK buffer allocation: NVIDIA block-linear tile geometry and the NVK modifier selection path.
- **Ch18 §3–4** — RADV and ANV modifier selection: how the two principal Vulkan drivers on Linux choose and advertise modifiers at image creation time.
- **Ch20 §6** — linux-dmabuf feedback protocol and EGL Wayland platform: the full Wayland-side modifier negotiation flow of which the flowchart in §7 is a condensed summary.
- **Ch24 §4** — Vulkan WSI modifier negotiation: `VK_EXT_image_drm_format_modifier` in the context of a full Vulkan WSI present chain.
- **Ch25 §6** — GPU compute interop: ROCm/CUDA peer-to-peer DMA-BUF import and the PCIe topology constraints on cross-vendor linear sharing.
- **Ch26 §5** — VA-API surface allocation: how video decode surfaces interact with AFBC and linear modifiers for zero-copy video pipelines.

---

## References

1. ARM. *ARM Frame Buffer Compression (AFBC) Architecture Specification*, ARM DDI 0591. https://developer.arm.com/documentation/102543/latest/
2. Linux kernel, `include/uapi/drm/drm_fourcc.h` (Linux 6.12). https://github.com/torvalds/linux/blob/master/include/uapi/drm/drm_fourcc.h — Primary source for all modifier macro definitions, vendor codes, and field layouts.
3. Khronos Group. *VK_EXT_image_drm_format_modifier* extension specification. https://registry.khronos.org/vulkan/specs/latest/man/html/VK_EXT_image_drm_format_modifier.html
4. Wayland protocols. *linux-dmabuf-unstable-v1*, `zwp_linux_dmabuf_feedback_v1` interface. https://wayland.app/protocols/linux-dmabuf-v1
5. Linux kernel, `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_plane.c` — `amdgpu_dm_plane_format_mod_supported()` and AMD modifier enumeration helpers.
6. Linux kernel, `drivers/gpu/drm/i915/display/intel_fb.c` — `intel_modifier_info[]` table and Intel modifier display parameters.
7. Mesa, `src/amd/vulkan/radv_image.c` — RADV modifier selection for render targets and scanout surfaces.
8. Mesa, `src/intel/vulkan/anv_image.c` — ANV CCS surface allocation and Intel modifier selection.
9. Mesa, `src/nouveau/vulkan/nvk_image.c` — NVK modifier selection and block-linear allocation.
10. Mesa, `src/gallium/drivers/panfrost/pan_resource.c` — Panfrost modifier selection logic.
11. Mesa, `src/gbm/backends/dri/gbm_dri.c` — `gbm_surface_create_with_modifiers2()` implementation and DRI modifier query.
12. Mesa, `src/egl/drivers/dri2/platform_wayland.c` — EGL Wayland platform: modifier advertisement and `EGL_EXT_image_dma_buf_import_modifiers`.
13. Mesa, `src/vulkan/wsi/wsi_common_drm.c` — Vulkan WSI modifier query and DMA-BUF export.
14. AMD GPU open. *Display Core Next Architecture*. https://gpuopen.com/amd-display-architecture/
15. Phoronix. *Intel Enables Xe2 Lunar Lake & Battlemage Graphics By Default With Linux 6.12*. https://www.phoronix.com/news/Linux-6.12-Intel-Xe2-Stable
16. amd-gfx mailing list. *[PATCH 12/13] drm/amdgpu/display: add all gfx12 modifiers*. https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg108855.html
