# Chapter 105: Font Rendering — FreeType2, HarfBuzz, and the Text Pipeline

> **Part**: Part VI — The Display Stack
> **Status**: First draft — 2026-06-19

**Target audiences:** Application developers, UI engineers, browser developers, and compositor authors who need to render text correctly on Linux. This chapter is relevant to anyone building on the GNOME/GTK or KDE/Qt stacks, embedding a browser engine, writing a terminal emulator, or implementing a Wayland compositor that handles text rendering. A working knowledge of C and basic familiarity with GPU rendering concepts (texture atlases, draw calls) is assumed; deep OpenType or Unicode knowledge is not required — those topics are introduced as needed.

---

## Table of Contents

1. [Introduction — From Codepoints to Pixels](#1-introduction--from-codepoints-to-pixels)
2. [Font Formats — TrueType, OpenType, Variable, and Color](#2-font-formats--truetype-opentype-variable-and-color)
3. [fontconfig — Font Discovery and Matching](#3-fontconfig--font-discovery-and-matching)
4. [FreeType2 — Rasterization](#4-freetype2--rasterization)
5. [HarfBuzz — Text Shaping](#5-harfbuzz--text-shaping)
6. [Pango — Paragraph Layout](#6-pango--paragraph-layout)
7. [Cairo — Compositing Text to a Surface](#7-cairo--compositing-text-to-a-surface)
8. [GPU-Accelerated Text Rendering](#8-gpu-accelerated-text-rendering)
9. [Subpixel Rendering and Hinting Trade-offs](#9-subpixel-rendering-and-hinting-trade-offs)
10. [Emoji and Color Fonts](#10-emoji-and-color-fonts)
11. [Integrations](#11-integrations)

---

## 1. Introduction — From Codepoints to Pixels

Text rendering is deceptively complex. A programmer typing `cairo_show_text(cr, "Hello")` triggers a pipeline that spans a half-dozen libraries, two levels of Unicode processing, at least one on-disk font-file parse, a rasterization pass that applies hinting, gamma correction, and subpixel filtering, and finally a compositing operation that blends antialiased glyph bitmaps onto a GPU surface with per-pixel alpha. For Latin text the complexity is manageable. For Arabic, Devanagari, Tibetan, or a paragraph containing all four simultaneously, additional stages — bidirectional analysis, script itemization, and OpenType GSUB/GPOS table interpretation — become mandatory.

The Linux text pipeline has five conceptual layers:

```
Unicode codepoints
        │
        ▼
fontconfig  ──── font discovery: which file on disk?
        │
        ▼
HarfBuzz   ──── shaping:  codepoints → glyph IDs + positions
        │                 (applies GSUB substitution, GPOS kerning)
        ▼
FreeType2  ──── rasterization: glyph ID → 8-bit or subpixel bitmap
        │
        ▼
Pango / text-layout library  ──── paragraph layout: wrapping, BiDi
        │
        ▼
Cairo / Skia / GPU path  ──── compositing bitmap onto surface
```

Notice that **shaping precedes rasterization**: HarfBuzz must decide which glyph IDs result from a sequence of codepoints — including all ligature substitutions and contextual alternates — *before* FreeType knows which outlines to rasterize. HarfBuzz uses FreeType only for glyph metric queries during shaping; the actual pixel output comes from FreeType's renderer, invoked afterwards with the resolved glyph IDs.

This chapter traces each layer in detail, from font format structures through API calls to GPU texture uploads, including the patent history of subpixel rendering, the mechanics of color emoji, and the HiDPI trade-offs that make hinting irrelevant at retina densities.

---

## 2. Font Formats — TrueType, OpenType, Variable, and Color

### 2.1 TrueType (.ttf)

TrueType was developed by Apple in 1987 as a competitor to PostScript Type 1. Its outlines use **quadratic Bézier splines** (B-splines with on-curve and off-curve control points), making them cheaper to rasterize than PostScript's cubic curves. The binary structure is a collection of named tables inside a SFNT wrapper; every table is identified by a 4-byte tag.

Key TrueType tables relevant to Linux rendering:

| Table | Purpose |
|-------|---------|
| `glyf` | Glyph outline data (quadratic B-splines) |
| `cmap` | Character-to-glyph-ID mapping (multiple subtables for Unicode, legacy encodings) |
| `hmtx` | Horizontal metrics: advance width, left side bearing |
| `kern` | Legacy pairwise kerning (superseded by GPOS in OpenType) |
| `head`, `hhea` | Global font metrics |
| `fpgm`, `prep`, `cvt ` | TrueType bytecode hinting programs |

The `cmap` table is how a font maps a Unicode codepoint to an internal *glyph ID* (GID). FreeType exposes this via `FT_Get_Char_Index(face, codepoint)`. The TrueType hinting bytecode (`fpgm`/`prep`/`cvt`) is a stack-based VM that can move glyph contour points to pixel grid boundaries, improving appearance at small sizes. Executing this bytecode requires FreeType's **bytecode interpreter** (`TT_CONFIG_OPTION_BYTECODE_INTERPRETER`). The interpreter was enabled by default after the relevant TrueType bytecode patents expired (~2010, FreeType 2.4.x era). FreeType 2.7.0 (September 2016) introduced the **v40 interpreter mode** — which ignores horizontal hinting instructions to produce DirectWrite/ClearType-like rendering — and made it the default. [Source: FreeType v40 subpixel hinting](https://freetype.org/freetype2/docs/hinting/subpixel-hinting.html)

### 2.2 OpenType (.otf)

OpenType is a joint Microsoft/Adobe extension of TrueType, standardized in 1996 and now maintained as [OpenType 1.9+](https://docs.microsoft.com/en-us/typography/opentype/spec/). It adds:

- **CFF outlines** (`CFF ` table): cubic Bézier splines from PostScript, preferred for print-quality output at large sizes.
- **GSUB** (Glyph Substitution): allows the shaper to replace one or more glyphs with others. Used for ligatures (`fi`, `fl`), contextual alternates, Arabic initial/medial/final/isolated forms, Devanagari conjunct consonants, stylistic sets.
- **GPOS** (Glyph Positioning): provides precise x/y adjustments for pairs or groups of glyphs. Enables optical kerning (more accurate than the legacy `kern` table), mark-to-base positioning (diacritics above base glyphs), and mark-to-mark positioning.
- **GDEF** (Glyph Class Definitions): classifies glyphs as base, ligature, mark, or component, which GSUB/GPOS rules reference.

GSUB lookups are typed (e.g., LookupType 1 = Single Substitution, LookupType 4 = Ligature Substitution, LookupType 6 = Context Substitution). HarfBuzz's shaping engine interprets these tables to produce the correct glyph sequence for any given Unicode input and enabled feature set. [Source: OpenType GSUB Specification](https://docs.microsoft.com/en-us/typography/opentype/spec/gsub)

### 2.3 Variable Fonts (OpenType 1.8+)

OpenType 1.8 (2016) introduced font variations, reviving a concept from Apple's GX Typography (1994). A variable font contains a continuous design space parameterized by *variation axes*, each identified by a 4-letter tag:

| Axis Tag | Registered Name | Typical Range |
|----------|----------------|---------------|
| `wght` | Weight | 100–900 |
| `wdth` | Width | 75–125 (%) |
| `ital` | Italic | 0–1 |
| `opsz` | Optical Size | 6–144 (pt) |
| `slnt` | Slant | -90–90 (degrees) |

The `fvar` table defines axes and named instances (e.g., "Regular", "Bold", "Light Italic"). For TrueType outlines, the `gvar` table stores per-glyph delta tuples that shift control-point coordinates as a function of axis coordinates. For CFF outlines the equivalent is `CFF2`. FreeType exposes the variation API since version 2.7: `FT_Set_Var_Design_Coordinates()` accepts a `wght` value of, say, 650 and interpolates the glyph outlines before rasterization. [Source: OpenType Font Variations Overview](https://docs.microsoft.com/en-us/typography/opentype/spec/otvaroverview)

### 2.4 Color Font Tables

Four competing color font formats exist in OpenType:

- **COLR/CPAL** (v0): Microsoft's layered colored glyphs. `CPAL` stores named palettes (arrays of RGBA colors). `COLR` maps each glyph to a list of (base-glyph, color-index) layers painted on top of each other. Flat colors only. Used by Microsoft's Segoe UI Emoji.
- **COLRv1** (OpenType 1.9, 2021): Extends COLR with gradients (linear, radial, sweep), Porter-Duff compositing, and variation support. Google published a COLRv1 build of Noto Color Emoji around 2022, reducing the font from ~10 MB (CBDT bitmap build) to ~1.85 MB in COLRv1+WOFF2 form. However, as of 2026 most Linux distributions still ship the CBDT bitmap build (`NotoColorEmoji.ttf`) by default; Fedora is an early adopter of the COLRv1 variant (`Noto-COLRv1.ttf`). COLRv1 glyphs are scalable to any size with no quality loss. [Source: COLRv1 Color Gradient Vector Fonts](https://developer.chrome.com/blog/colrv1-fonts/)
- **CBDT/CBLC**: Google's bitmap emoji format. CBLC is the index (analogous to `bloc`), CBDT stores compressed PNG bitmaps. Size-specific; no scaling quality. Used by older Android Noto emoji and still widely deployed.
- **SVG table**: Mozilla's proposal; embeds full SVG 1.1 documents per glyph. Rich but large; limited toolchain support. Used by some Twemoji and Firefox OS fonts.
- **sbix** (Apple): Per-glyph PNG/JPEG/TIFF bitmaps with resolution hints. Supported by macOS and recent FreeType versions.

FreeType reports color font capability via `FT_HAS_COLOR(face)`. Since FreeType 2.10.0 (2019), `FT_LOAD_COLOR` in the load flags instructs the library to return color bitmap data when available. [Source: FreeType FT_LOAD_COLOR](https://freetype.org/freetype2/docs/reference/ft2-base_interface.html)

### 2.5 WOFF2

WOFF2 (Web Open Font Format 2.0, W3C Recommendation 2018) is a compressed container around OpenType/TrueType data. It applies Brotli compression to the raw SFNT tables, reducing file size by ~30% versus WOFF1 and ~50% versus raw .ttf for typical fonts. FreeType reads WOFF2 natively since version 2.10.2 (2020), when Nikhil Ramakrishnan's GSoC 2019 contribution landed; the `sfnt` driver gained built-in Brotli/woff2 decompression at that release. [Source: W3C WOFF2](https://www.w3.org/TR/WOFF2/)

---

## 3. fontconfig — Font Discovery and Matching

fontconfig is the library that maps an abstract font description (family, weight, slant, size) to a concrete file path and index on disk. Every GTK application, every Cairo text call that isn't given an explicit FT_Face, and every `pango_font_description_from_string()` call goes through fontconfig. [Source: fontconfig Developer Reference](https://fontconfig.pages.freedesktop.org/fontconfig/fontconfig-devel/)

### 3.1 Font Discovery

fontconfig builds an in-memory database of available fonts by scanning directories listed in its configuration. The default scan paths on a typical Linux system are:

```
/usr/share/fonts/          # system-wide fonts (distro packages)
/usr/local/share/fonts/    # locally installed fonts
~/.local/share/fonts/      # per-user fonts (XDG base dir spec)
~/.fonts/                  # legacy per-user location
```

Additional directories may be specified in configuration files. fontconfig reads all `.conf` files from `/etc/fonts/conf.d/` in sorted order — filenames conventionally begin with two-digit priority numbers (`10-hinting-slight.conf`, `45-latin.conf`, `65-nonlatin.conf`) so that rules stack in predictable priority order. System defaults live in `/etc/fonts/fonts.conf`; per-user overrides go in `~/.config/fontconfig/fonts.conf`.

The result of scanning is a set of `FcFontSet` objects containing one `FcPattern` per discovered font file/face. The scan is expensive (it must parse every font file for metadata), so fontconfig caches the results in `/var/cache/fontconfig/` as binary `.cache-N` files keyed by directory mtime and UUID. Cache invalidation happens automatically when a font directory's mtime changes (e.g., after installing a font package); the `fc-cache -fv` command forces a rebuild. [Source: fontconfig User Reference](https://www.freedesktop.org/software/fontconfig/fontconfig-user.html)

### 3.2 FcPattern and Font Properties

An `FcPattern` is fontconfig's universal descriptor: a typed property list where each name (an `FcObjectType`) has one or more values. Roughly 60 standard property names are defined:

| Property | Type | Example Value |
|----------|------|---------------|
| `FC_FAMILY` | String | "DejaVu Sans" |
| `FC_STYLE` | String | "Bold Italic" |
| `FC_WEIGHT` | Integer | 200 (FC_WEIGHT_BOLD) |
| `FC_SLANT` | Integer | 0 (FC_SLANT_ROMAN), 100 (ITALIC) |
| `FC_SIZE` | Double | 12.0 (points) |
| `FC_FILE` | String | "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf" |
| `FC_INDEX` | Integer | 0 (face index within file) |
| `FC_PIXEL_SIZE` | Double | 16.0 |
| `FC_DPI` | Double | 96.0 |
| `FC_SPACING` | Integer | 0 (proportional), 100 (monospace) |
| `FC_LANG` | LangSet | "en:fr:de" |

```c
/* Example: build a request pattern for 12pt Bold DejaVu Sans */
FcPattern *pat = FcPatternCreate();
FcPatternAddString(pat, FC_FAMILY, (FcChar8 *)"DejaVu Sans");
FcPatternAddInteger(pat, FC_WEIGHT, FC_WEIGHT_BOLD);
FcPatternAddDouble(pat, FC_SIZE, 12.0);
```

[Source: fontconfig Datatypes](https://www.freedesktop.org/software/fontconfig/fontconfig-devel/x31.html)

### 3.3 Font Matching

The matching pipeline requires three steps to produce correct results:

```c
FcConfigSubstitute(config, pat, FcMatchPattern);  /* apply config rules */
FcDefaultSubstitute(pat);                          /* fill missing defaults */
FcResult result;
FcPattern *matched = FcFontMatch(config, pat, &result);
```

`FcConfigSubstitute()` applies substitution rules from `fonts.conf` — for example, it may expand `FC_FAMILY = "sans-serif"` to `FC_FAMILY = "DejaVu Sans:Noto Sans:..."`. `FcDefaultSubstitute()` fills properties the caller left unspecified (DPI defaults to 75, hinting defaults to `true`, etc.). Only after both substitutions is `FcFontMatch()` accurate; calling it on a bare pattern yields unpredictable results.

`FcFontMatch()` scores every font in the database against the request pattern using a priority-weighted distance function. The priority order (highest first) is: charset coverage, then family name, then pixel size, then weight, then slant, then other properties. The font with the lowest total distance is returned. The returned `FcPattern` includes `FC_FILE` and `FC_INDEX`, which can be passed directly to `FT_New_Face()`.

### 3.4 Font Aliasing

fontconfig provides generic family aliases so that applications can request `"sans-serif"` and receive a locale-appropriate font. Default aliases on most Linux distributions:

| Generic Name | Typical Mapping |
|-------------|----------------|
| `sans-serif` | DejaVu Sans, then Noto Sans |
| `serif` | DejaVu Serif, then Noto Serif |
| `monospace` | DejaVu Sans Mono, then Noto Sans Mono |

The alias expansion is implemented as `<alias>` rules in `65-nonlatin.conf` and `65-latin.conf`. Users can override aliases in `~/.config/fontconfig/fonts.conf`:

```xml
<alias>
  <family>sans-serif</family>
  <prefer>
    <family>Inter</family>
    <family>Noto Sans</family>
    <family>DejaVu Sans</family>
  </prefer>
</alias>
```

### 3.5 CLI Tools

```bash
fc-list                          # list all available fonts (family, style, file)
fc-list :spacing=100             # filter: monospace fonts only
fc-match sans-serif:weight=bold  # show which font would be selected
fc-scan /path/to/font.ttf        # show all properties fontconfig reads from a file
fc-cache -fv                     # rebuild cache verbosely
```

---

## 4. FreeType2 — Rasterization

FreeType2 is the C library that reads a font file, parses the glyph outline for a given glyph ID, applies hinting transformations, and rasterizes the result into a bitmap. It supports TrueType, CFF/OpenType, Type 1, PCF (X11 bitmap fonts), BDF, and several other formats. [Source: FreeType2 API Reference](https://freetype.org/freetype2/docs/reference/)

### 4.1 Core Types

```c
FT_Library   lib;     /* one per process; owns the font module table */
FT_Face      face;    /* one per font file/index; holds parsed tables */
FT_GlyphSlot slot;    /* face->glyph; reused per FT_Load_Glyph() call */
FT_Bitmap    bm;      /* rasterized pixel data; field of slot->bitmap */
```

`FT_Library` is not thread-safe — a single library handle must not be used concurrently from multiple threads. Applications that need concurrent rendering (e.g., a browser compositor loading many fonts) should either create one `FT_Library` per thread or serialize access via a mutex. Each `FT_Face` is similarly per-thread-safe only when accessed from one thread at a time.

### 4.2 The Rendering Pipeline

```c
/* 1. Initialize the library */
FT_Library lib;
FT_Init_FreeType(&lib);

/* 2. Load a face from a file (face index 0 = first face in a TTC/collection) */
FT_Face face;
FT_New_Face(lib, "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 0, &face);

/* 3. Set the pixel size (26.6 fixed-point: 16 << 6 = 16px, or use FT_Set_Pixel_Sizes) */
FT_Set_Pixel_Sizes(face, 0, 16);   /* width=0: derive from height */

/* 4. Get glyph ID from codepoint (after HarfBuzz shaping in a real pipeline) */
FT_UInt glyph_index = FT_Get_Char_Index(face, 0x0041);  /* 'A' */

/* 5. Load the glyph: parse outline, apply hinting */
FT_Load_Glyph(face, glyph_index, FT_LOAD_DEFAULT);

/* 6. Rasterize to bitmap */
FT_Render_Glyph(face->glyph, FT_RENDER_MODE_NORMAL);

/* 7. Access the bitmap */
FT_GlyphSlot slot = face->glyph;
FT_Bitmap *bm = &slot->bitmap;
/* bm->buffer: pixel data, row-major, bm->pitch bytes per row */
/* bm->rows: height in pixels */
/* bm->width: width in pixels */
/* slot->bitmap_left: horizontal bearing (pen origin to left edge) */
/* slot->bitmap_top: vertical bearing (pen baseline to top edge, positive up) */
/* slot->advance.x: advance width in 26.6 fixed-point (>> 6 to get pixels) */
```

In a real shaping pipeline, `glyph_index` is the value from `hb_glyph_info_t.codepoint` (which after `hb_shape()` holds the glyph ID, not a Unicode codepoint) and the pen position is accumulated from `hb_glyph_position_t.x_advance`.

### 4.3 Render Modes

`FT_Render_Glyph()` accepts one of four render modes:

| Mode | Output | Bits/Pixel | Use |
|------|--------|------------|-----|
| `FT_RENDER_MODE_NORMAL` | 8-bit grayscale | 8 | Standard AA; single channel |
| `FT_RENDER_MODE_MONO` | 1-bit bitmap | 1 | Monochrome displays, old X11 |
| `FT_RENDER_MODE_LCD` | RGB horizontal subpixel | 8×3 (24) | LCD subpixel rendering, horizontal stripes |
| `FT_RENDER_MODE_LCD_V` | BGR vertical subpixel | 8×3 (24) | Vertical subpixel (rare displays) |
| `FT_RENDER_MODE_SDF` | Signed distance field | 8 (float encoded) | Since FreeType 2.11; scalable rendering |

For `FT_RENDER_MODE_LCD`, the output bitmap has `width * 3` bytes per row; R, G, and B sub-components are stored consecutively. The caller composite each 3-byte column onto an RGB framebuffer pixel. FreeType applies a 5-tap FIR filter during LCD rendering to reduce color fringing (see §9).

### 4.4 Load Flags and Hinting

`FT_Load_Glyph()` takes an integer bitmask of load flags:

```c
/* Common combinations: */
FT_LOAD_DEFAULT            /* apply hinting, load scaled outline */
FT_LOAD_NO_HINTING         /* skip hinting; preserve letterform exactly */
FT_LOAD_TARGET_NORMAL      /* hint for grayscale AA rendering */
FT_LOAD_TARGET_LIGHT       /* autohinter: preserve letterform, fit lightly */
FT_LOAD_TARGET_LCD         /* hint optimally for horizontal LCD subpixel */
FT_LOAD_TARGET_LCD_V       /* hint for vertical LCD subpixel */
FT_LOAD_FORCE_AUTOHINT     /* use FreeType's autohinter even if TT bytecode present */
FT_LOAD_NO_BITMAP          /* ignore embedded bitmaps; always use outline */
FT_LOAD_COLOR              /* return color bitmap for color fonts (CBDT, COLR) */
```

The autohinter uses a heuristic algorithm that analyses glyph geometry (stem widths, blue zones) to align points to the pixel grid. It is less aggressive than TrueType bytecode hinting, producing glyphs that remain closer to the designer's intent while still improving grid alignment at small sizes. At HiDPI (≥192 DPI), the sub-pixel size of hinting corrections becomes invisible and hinting is commonly disabled entirely.

### 4.5 FT_Bitmap Structure

After `FT_Render_Glyph()`, the `face->glyph->bitmap` struct has:

```c
typedef struct FT_Bitmap_ {
  unsigned int    rows;       /* number of rows (height in pixels) */
  unsigned int    width;      /* number of columns (width in pixels) */
  int             pitch;      /* bytes per row; negative = bottom-up */
  unsigned char  *buffer;     /* pixel data; size = |pitch| * rows */
  unsigned short  num_grays;  /* 256 for FT_PIXEL_MODE_GRAY */
  unsigned char   pixel_mode; /* FT_PIXEL_MODE_GRAY, _MONO, _LCD, etc. */
  /* ... */
} FT_Bitmap;
```

`pitch` may differ from `width` due to alignment padding. For LCD mode, `width` is the logical glyph width and `pitch ≥ width * 3`. Always use `|pitch|` as the row stride when iterating `buffer`. [Source: FreeType FT_Bitmap](https://freetype.org/freetype2/docs/reference/ft2-basic_interface.html)

---

## 5. HarfBuzz — Text Shaping

Text shaping is the process of converting a sequence of Unicode codepoints into a sequence of positioned glyph IDs, with all script-specific typographic rules applied. HarfBuzz is the industry-standard shaper: it is used by Chrome (V8/Blink), Firefox (Gecko), Android, LibreOffice, GIMP, and the entire GTK/Pango stack. [Source: HarfBuzz Manual](https://harfbuzz.github.io/)

### 5.1 Why Shaping Is Necessary

For Latin text the relationship between codepoints and glyphs is nearly 1:1. Shaping is still needed for kerning (GPOS lookup type 2) and ligatures (`fi`, `fl`, `ffi` from GSUB lookup type 4), but could theoretically be skipped with a loss of typographic quality. For other scripts, shaping is non-negotiable:

- **Arabic**: Characters have four contextual forms (initial, medial, final, isolated). U+0628 (ب) becomes glyph ID 200 in initial position but glyph ID 203 in final position. Mandatory ligatures (lam-alef: لا → single glyph) require GSUB substitutions that the Unicode character stream does not encode.
- **Devanagari**: Consonant conjuncts (the combination क + ् + ष → क्ष) require GSUB half-form substitutions and reordering. The visual order of pre-base matras differs from the logical Unicode order.
- **Hebrew**: Bidirectional text and niqqud (vowel diacritics) require GPOS mark positioning.
- **Emoji ZWJ sequences**: U+1F468 (man) + U+200D (ZWJ) + U+1F469 (woman) + U+200D + U+1F467 (girl) must become a single "family" glyph via GSUB.

HarfBuzz handles all of these through its Universal Shaping Engine (USE, since 2018) and dedicated shaping plans for Arabic, Indic, Hangul, and other complex scripts. [Source: What Does HarfBuzz Do?](https://harfbuzz.github.io/what-does-harfbuzz-do.html)

### 5.2 Core API

```c
#include <hb.h>
#include <hb-ft.h>   /* FreeType integration */

/* 1. Create a font from an FT_Face (reference-counted; ft_face stays alive) */
hb_font_t *hb_font = hb_ft_font_create_referenced(ft_face);

/* 2. Create and populate a buffer */
hb_buffer_t *buf = hb_buffer_create();
hb_buffer_add_utf8(buf, text_utf8, -1, 0, -1);  /* -1: NUL-terminated */

/* 3. Set buffer properties (or use hb_buffer_guess_segment_properties) */
hb_buffer_set_direction(buf, HB_DIRECTION_LTR);         /* or RTL, TTB, BTT */
hb_buffer_set_script(buf, HB_SCRIPT_LATIN);             /* HB_SCRIPT_ARABIC, etc. */
hb_buffer_set_language(buf, hb_language_from_string("en", -1));

/* 4. Shape: applies GSUB + GPOS from the font's tables */
hb_feature_t features[] = {
    { HB_TAG('l','i','g','a'), 1, 0, (unsigned int)-1 },  /* enable ligatures */
    { HB_TAG('k','e','r','n'), 1, 0, (unsigned int)-1 },  /* enable kerning */
};
hb_shape(hb_font, buf, features, 2);

/* 5. Retrieve shaped glyph data */
unsigned int glyph_count;
hb_glyph_info_t     *glyph_info = hb_buffer_get_glyph_infos(buf, &glyph_count);
hb_glyph_position_t *glyph_pos  = hb_buffer_get_glyph_positions(buf, &glyph_count);

for (unsigned int i = 0; i < glyph_count; i++) {
    uint32_t glyph_id  = glyph_info[i].codepoint;  /* NOT Unicode codepoint anymore! */
    uint32_t cluster   = glyph_info[i].cluster;     /* byte offset in original UTF-8 */
    int32_t  x_advance = glyph_pos[i].x_advance;   /* in font units (26.6 if hb-ft) */
    int32_t  y_advance = glyph_pos[i].y_advance;
    int32_t  x_offset  = glyph_pos[i].x_offset;    /* placement correction */
    int32_t  y_offset  = glyph_pos[i].y_offset;
}
```

After `hb_shape()`, `hb_glyph_info_t.codepoint` no longer holds a Unicode codepoint — it holds the **glyph ID** that should be passed to `FT_Load_Glyph()`. The field name is a historical artifact of the API. The `cluster` field maps each output glyph back to the byte offset of its originating character(s) in the UTF-8 input, enabling bidirectional caret positioning and selection hit-testing.

### 5.3 Positions and Units

When `hb_ft_font_create_referenced()` is used, HarfBuzz sets the font scale to match FreeType's 26.6 fixed-point format: 1 font unit = 1/64th pixel at the current pixel size. To convert to integer pixels, right-shift by 6: `(x_advance + 32) >> 6` (the +32 rounds to nearest). The y_advance is non-zero only for vertical text (`HB_DIRECTION_TTB`).

### 5.4 OpenType Feature Tags

HarfBuzz feature tags correspond directly to OpenType feature name tags, which are 4-byte identifiers like `liga` (standard ligatures), `calt` (contextual alternates), `kern` (kerning), `smcp` (small capitals), `frac` (fractions), `ss01`–`ss20` (stylistic sets), and `salt` (stylistic alternates). Applications can expose feature toggles to end users, allowing, for example, the disabling of automatic ligatures in code editors. [Source: HarfBuzz Feature API](https://harfbuzz.github.io/harfbuzz-hb-common.html)

### 5.5 Script Itemization

A real text layout engine does not pass a whole paragraph of mixed-script text to a single `hb_shape()` call. Instead, it *itemizes* the input: splits the input into *runs* where each run has a uniform combination of (script, direction, language, font, font-size, color). Pango performs this itemization via `pango_itemize()` before calling HarfBuzz. The Unicode Bidirectional Algorithm (Unicode TR#9) determines run direction; the Unicode Script Determination Algorithm (Unicode TR#24) determines script. HarfBuzz itself does not itemize — it shapes one homogeneous run at a time.

---

## 6. Pango — Paragraph Layout

Pango is the text layout engine used by GTK (and by Cairo when higher-level text layout is needed). It sits above HarfBuzz and FreeType, providing paragraph-level services: line breaking, bidirectional ordering, script itemization, font fallback, alignment, and justification. [Source: Pango Documentation](https://docs.gtk.org/Pango/)

### 6.1 Architecture

Pango's design is modular: a platform-specific backend provides font enumeration and glyph rendering, while the core layout logic is backend-neutral. Since **Pango 1.44** (2019), HarfBuzz is the sole shaping engine on all platforms — the previous platform-specific shapers (Pango/CoreText on macOS, Uniscribe on Windows) were removed. The fontconfig/FreeType backend is the default on Linux.

### 6.2 PangoFontDescription

`PangoFontDescription` describes a logical font by family, style, weight, variant, stretch, and size:

```c
PangoFontDescription *desc = pango_font_description_from_string("Inter Bold 14");
/* or build programmatically: */
PangoFontDescription *desc2 = pango_font_description_new();
pango_font_description_set_family(desc2, "DejaVu Serif");
pango_font_description_set_weight(desc2, PANGO_WEIGHT_SEMIBOLD);  /* 600 */
pango_font_description_set_size(desc2, 12 * PANGO_SCALE);  /* PANGO_SCALE = 1024 */
```

Sizes in Pango are specified in **Pango units**: 1 point = `PANGO_SCALE` (= 1024) Pango units. Device pixels depend on the DPI context. `pango_layout_get_pixel_size()` returns dimensions after DPI scaling has been applied.

### 6.3 PangoLayout

`PangoLayout` represents an entire paragraph (or multi-paragraph block) of text with all layout decisions resolved:

```c
PangoContext *ctx = pango_cairo_create_context(cr);
PangoLayout  *layout = pango_layout_new(ctx);

pango_layout_set_text(layout, "مرحبا بالعالم", -1);  /* Arabic: RTL text */
pango_layout_set_font_description(layout, desc);
pango_layout_set_width(layout, 400 * PANGO_SCALE);   /* word-wrap at 400px */
pango_layout_set_alignment(layout, PANGO_ALIGN_RIGHT);

/* Measure */
int text_width, text_height;
pango_layout_get_pixel_size(layout, &text_width, &text_height);

PangoRectangle ink_rect, logical_rect;
pango_layout_get_pixel_extents(layout, &ink_rect, &logical_rect);
/* ink_rect: tightest bounding box of actual painted pixels */
/* logical_rect: layout-reserved space (includes leading, descenders) */
```

The distinction between ink and logical extents matters for: clipping (use ink rect), spacing/margins (use logical rect), and cursor positioning (use logical baseline).

### 6.4 Pango Markup

Pango markup is an HTML-like syntax for mixing text attributes inline:

```c
pango_layout_set_markup(layout,
    "Hello <b>world</b>, <span foreground='red' size='14pt'>attention!</span>",
    -1);
```

Supported tags: `<b>` (bold), `<i>` (italic), `<u>` (underline), `<s>` (strikethrough), `<tt>` (monospace), `<big>`, `<small>`, and the general-purpose `<span>` with attributes `font`, `foreground`, `background`, `size`, `weight`, `style`, `variant`, `stretch`, `letter_spacing`, `rise`, `underline`, `strikethrough`, `font_features`.

### 6.5 PangoAttrList for Fine-Grained Control

For programmatic attribute control (e.g., syntax highlighting), `PangoAttrList` applies typed attribute objects to byte ranges of the text:

```c
PangoAttrList *attrs = pango_attr_list_new();
PangoAttribute *bold = pango_attr_weight_new(PANGO_WEIGHT_BOLD);
bold->start_index = 6;    /* byte offset in UTF-8 string */
bold->end_index   = 11;
pango_attr_list_insert(attrs, bold);
pango_layout_set_attributes(layout, attrs);
pango_attr_list_unref(attrs);
```

### 6.6 PangoItem: Shaped Run

Internally, `pango_itemize()` breaks the input into `PangoItem` objects, each representing a run with uniform script, direction, font, and language. Each `PangoItem` is shaped independently by calling `pango_shape()`, which calls HarfBuzz's `hb_shape()` under the hood. The `PangoGlyphString` output of `pango_shape()` maps directly to the HarfBuzz position arrays. Applications rarely need to touch `PangoItem` directly; `PangoLayout` manages the whole pipeline.

### 6.7 Rendering via Cairo

```c
/* Render PangoLayout to a Cairo context at (x, y) */
cairo_move_to(cr, x, y);
pango_cairo_show_layout(cr, layout);

/* Or render just one line: */
pango_cairo_show_layout_line(cr, pango_layout_get_line_readonly(layout, 0));
```

`pango_cairo_show_layout()` iterates the layout's glyph runs, calls `cairo_show_glyphs()` for each run, and Cairo composites the glyphs using the current source and operator.

---

## 7. Cairo — Compositing Text to a Surface

Cairo is the 2D vector graphics library that underlies GTK's rendering (on non-GL paths) and Pango's output target. Its text API has two layers: a "toy" API for simple use cases and a low-level glyph API for production use. [Source: Cairo Text API](https://www.cairographics.org/manual/cairo-text.html)

### 7.1 The Toy Text API

```c
cairo_select_font_face(cr, "sans-serif", CAIRO_FONT_SLANT_NORMAL, CAIRO_FONT_WEIGHT_BOLD);
cairo_set_font_size(cr, 14.0);
cairo_move_to(cr, 10.0, 30.0);
cairo_show_text(cr, "Hello, world!");
```

The toy API uses fontconfig internally to select a font file and FreeType to rasterize it. It is sufficient for simple one-line labels but lacks BiDi support and does not expose OpenType features. For internationalized text, use PangoLayout.

### 7.2 Low-Level Glyph Rendering

```c
/* cairo_glyph_t: one per shaped glyph */
cairo_glyph_t glyphs[] = {
    { .index = 36,  .x = 10.0, .y = 30.0 },  /* glyph ID 36 at pen position */
    { .index = 72,  .x = 21.5, .y = 30.0 },
    { .index = 104, .x = 32.0, .y = 30.0 },
};
cairo_show_glyphs(cr, glyphs, 3);
```

`cairo_glyph_t.index` is the glyph ID (same as what HarfBuzz outputs), and `x`, `y` are the pen position on the Cairo surface coordinate system. This is exactly how `pango_cairo_show_layout()` operates internally: it converts HarfBuzz positions to Cairo glyph coordinates and calls `cairo_show_glyphs()`.

### 7.3 FreeType Font Faces in Cairo

Cairo can use a FreeType face directly, bypassing fontconfig:

```c
#include <cairo-ft.h>

cairo_font_face_t *ft_face_c =
    cairo_ft_font_face_create_for_ft_face(ft_face, FT_LOAD_DEFAULT);
cairo_set_font_face(cr, ft_face_c);
cairo_set_font_size(cr, 14.0);
cairo_show_text(cr, "Direct FreeType rendering");
cairo_font_face_destroy(ft_face_c);
```

The load flags passed to `cairo_ft_font_face_create_for_ft_face()` are used in all subsequent `FT_Load_Glyph()` calls Cairo makes for this face. [Source: Cairo FreeType Fonts](https://www.cairographics.org/manual/cairo-FreeType-Fonts.html)

### 7.4 Font Options and Antialiasing

`cairo_font_options_t` controls the rendering quality:

```c
cairo_font_options_t *opts = cairo_font_options_create();
cairo_font_options_set_antialias(opts, CAIRO_ANTIALIAS_SUBPIXEL);
cairo_font_options_set_subpixel_order(opts, CAIRO_SUBPIXEL_ORDER_RGB);
cairo_font_options_set_hint_style(opts, CAIRO_HINT_STYLE_SLIGHT);
cairo_font_options_set_hint_metrics(opts, CAIRO_HINT_METRICS_ON);
cairo_set_font_options(cr, opts);
```

`CAIRO_ANTIALIAS_SUBPIXEL` enables FreeType's LCD filter path. `CAIRO_SUBPIXEL_ORDER_RGB` matches most modern LCD panels (red subpixel on the left); `CAIRO_SUBPIXEL_ORDER_BGR` covers some panels and most vertical stripe OLED. Cairo on Wayland typically reads subpixel order from the compositor's `wl_output` metadata.

### 7.5 Alpha Compositing

Text is alpha-composited onto the destination surface using `CAIRO_OPERATOR_OVER` (the default):

```
destination_out = alpha * source + (1 - alpha) * destination_in
```

For grayscale-antialiased text, `alpha` is the 8-bit coverage value from `FT_Bitmap.buffer`. For subpixel text, each RGB channel has its own coverage value, and Cairo applies per-channel compositing to avoid color fringing on the destination.

---

## 8. GPU-Accelerated Text Rendering

On a GPU-composited desktop or in a browser, rendering text through Cairo's software rasterizer for every frame is too slow. Modern toolkits — Skia (Chrome, Flutter), Qt, WebRender (Firefox) — use GPU texture atlases to cache rasterized glyphs and draw text with minimal draw calls.

### 8.1 The Glyph Texture Atlas

The canonical approach:

1. **Rasterize on CPU**: For each new (font, size, glyph-ID) triple, invoke FreeType to produce a bitmap.
2. **Upload to GPU**: Pack the bitmap into a shelf-allocated region of a large GPU texture (typically `GL_R8` for grayscale or `GL_RGBA8` for color glyphs), typically 2048×2048 or 4096×4096 RGBA texels.
3. **Draw with quads**: For each glyph in a text run, emit a quad (2 triangles) whose UV coordinates point into the atlas at the glyph's packed position.
4. **Blend in shader**: The fragment shader reads the coverage value from the atlas and multiplies by text color, then alpha-blends onto the framebuffer.

The key insight is that glyphs for a given font/size combination are rendered once and cached across frames. A scrolling text view of 10,000 characters may need only 80–120 distinct glyph bitmaps; the GPU draws all 10,000 with one or two batched draw calls into the atlas texture.

### 8.2 Skia's Text Path

Skia uses `SkTextBlob` as the unit of cached text:

```cpp
/* Build a text blob (can be cached and reused across frames) */
SkFont font(typeface, 14.0f);
font.setEdging(SkFont::Edging::kSubpixelAntiAlias);
font.setSubpixel(true);

SkTextBlobBuilder builder;
const auto &run = builder.allocRunPosH(font, glyph_count, baseline_y);
memcpy(run.glyphs, glyph_ids, glyph_count * sizeof(uint16_t));
memcpy(run.pos,    x_positions, glyph_count * sizeof(float));
sk_sp<SkTextBlob> blob = builder.make();

/* Draw it */
canvas->drawTextBlob(blob.get(), 0, 0, paint);
```

Skia's glyph cache is managed by `SkStrikeCache`, which holds `SkStrike` objects — one per (font, size, device-transform) combination. Each `SkStrike` stores rasterized glyph masks in a per-font hash table. The GPU backend (`SkGpuDevice`) uploads glyph masks to a `GrAtlasManager` that maintains one or more GL/Vulkan texture atlases, separate atlases for `A8` (grayscale), `A565` (LCD), and `RGBA` (color glyphs). [Source: Skia SkTextBlob API](https://api.skia.org/classSkTextBlob.html)

### 8.3 WebRender's GPU Text

Mozilla's WebRender (the GPU compositor in Firefox since Firefox 67 on Windows, and progressively rolled out to Linux since Firefox 80) renders text via a dedicated glyph cache in the GPU process:

- Glyph rasterization happens on the **content process** using FreeType/HarfBuzz/Pango.
- Rasterized bitmaps are transmitted to the **GPU process** via shared memory.
- The GPU process packs them into texture atlas tiles using a shelf allocator (added to Firefox ~2021 to reduce atlas fragmentation). [Source: Mozilla WebRender Atlas Allocation](https://mozillagfx.wordpress.com/2021/02/04/improving-texture-atlas-allocation-in-webrender/)
- Text draw calls are encoded as WebRender `PrimitiveKind::TextRun` display list items, processed by the frame builder, and rendered via Vulkan/OpenGL draw calls.

### 8.4 Signed Distance Field (SDF) Text

An alternative to bitmap atlases is the **signed distance field** approach, popularized by Valve's 2007 SIGGRAPH paper:

- For each glyph, compute the signed distance from each texel center to the nearest glyph contour edge (positive inside, negative outside).
- Store as an 8-bit or float32 texture at a moderate resolution (e.g., 64×64 per glyph).
- In the fragment shader, threshold the SDF value at 0.5 to reconstruct the glyph edge: `alpha = smoothstep(0.5 - spread, 0.5 + spread, texture(sdf_atlas, uv).r)`.

**Advantage**: A single SDF texture supports arbitrary scaling — the same 64×64 SDF produces acceptable quality from 8px to 200px. **Disadvantage**: Corners and thin features in the glyph may be rounded or distorted.

**Multi-channel SDF (MSDF)** (Chlumský, 2016) encodes three independent signed distances in R, G, B channels, each measuring distance to a different subset of edges. The shader takes the *median* of the three channels to reconstruct a sharper edge that preserves corners. MSDF is used by production engines that need scalable text on GPU (game UIs, real-time dashboards). [Source: msdfgen project](https://github.com/Chlumsky/msdfgen)

### 8.5 Qt Text Rendering

Qt's text path:
- `QPainter::drawText()` → `QTextLayout` (layout engine, analogous to PangoLayout)
- `QTextLayout` → `QFont` → `QFontEngine` (backend: FreeType on Linux via `QFontEngineFT`)
- `QFontEngineFT` calls FreeType to rasterize each glyph and uploads the result to a `QFontEngineGlyphCache`
- On Qt's OpenGL/Vulkan/Metal backends, `QFontEngineGlyphCache` uploads to a GL texture; text is drawn with a quad-based shader similar to Skia's.
- `QRawFont` provides direct access to glyph IDs and advance widths without going through the string API, useful when HarfBuzz-shaped glyph IDs need to be passed to Qt for rendering.

---

## 9. Subpixel Rendering and Hinting Trade-offs

### 9.1 Subpixel Rendering: The Physics

Modern LCD panels are arrays of red, green, and blue subpixels arranged in rows. A "pixel" as measured in software actually comprises three color elements in a horizontal row (for standard RGB stripe panels). Subpixel rendering exploits this: by treating the three subpixels as independently addressable samples, the effective horizontal resolution is tripled.

FreeType implements ClearType-style horizontal subpixel rendering via `FT_RENDER_MODE_LCD`. The rasterizer computes per-subpixel coverage (a separate grayscale value for R, G, B), which must then be filtered before compositing to prevent color fringing on sharp edges.

### 9.2 FreeType's LCD Filter: A History

**Early 2000s–2010: Patent lock-in.** Microsoft held patents on ClearType subpixel rendering, including US Patent 6,219,025 ("Mapping image data samples to pixel sub-components on a striped display device," expiring July 2019) and related patents covering the FIR filtering step. FreeType implemented LCD rendering but distribution builds disabled it by default to avoid patent exposure — distributors shipped FreeType without `FT_CONFIG_OPTION_SUBPIXEL_RENDERING` defined, forcing users to recompile or install a patched build.

**2017: Harmony mode.** FreeType 2.8.1 (2017) introduced the "Harmony" LCD rendering technique, developed as a patent workaround. Harmony shifts three separate monochrome rendering passes of the same outline by ±1/3 pixel, then combines the three coverage bitmaps into R, G, and B channels. Because Harmony does not use the patented FIR filter step, it could be distributed freely. The quality is comparable to filtered ClearType at most sizes. [Source: FreeType Harmony commit, 2017](https://lists.nongnu.org/archive/html/freetype-commit/2017-08/msg00020.html)

**2019: Patent expiry.** The key ClearType patents expired in the United States around 2019. Additionally, Microsoft joined the Open Invention Network (OIN) in 2018, granting Linux-community royalty-free licenses to Microsoft's patent portfolio. The legal barrier to ClearType-style LCD rendering on Linux evaporated.

**2020: Default on.** FreeType 2.10.3 (October 2020) changed the default: **LCD filtering is now enabled by default** with `FT_LCD_FILTER_DEFAULT`. Applications no longer need to call `FT_Library_SetLcdFilter()` explicitly to get subpixel rendering — it is active whenever `FT_RENDER_MODE_LCD` is requested. The five-tap FIR filter with weights `[0x08, 0x4D, 0x56, 0x4D, 0x08]` (normalized, beveled, color-balanced) is the default. [Source: FreeType LCD Rendering Reference](https://freetype.org/freetype2/docs/reference/ft2-lcd_rendering.html)

Applications that need a different filter can still call:

```c
FT_Library_SetLcdFilter(lib, FT_LCD_FILTER_LIGHT);   /* 3-tap, softer */
FT_Library_SetLcdFilter(lib, FT_LCD_FILTER_NONE);    /* no filter; fringing visible */
FT_Library_SetLcdFilter(lib, FT_LCD_FILTER_DEFAULT); /* default 5-tap [0x08 4D 56 4D 08] */
```

### 9.3 Hinting Trade-offs

Font hinting moves glyph contour control points to align with the pixel grid, producing sharper glyphs at small sizes at the cost of distorting the letterform away from the designer's intent.

| Hinting Style | FreeType Flag | Result |
|--------------|--------------|--------|
| Full (bytecode) | `FT_LOAD_DEFAULT` | Sharp stems; letterform may be significantly distorted |
| Light (autohint) | `FT_LOAD_TARGET_LIGHT` | Horizontal alignment preserved; vertical stems float |
| No hinting | `FT_LOAD_NO_HINTING` | Exact letterform; may appear blurry at ≤14px on 96 DPI |
| LCD-optimized | `FT_LOAD_TARGET_LCD` | Bytecode hints aligned to subpixel grid |

For system UI text at 10–14px on 96 DPI, slight hinting (`FT_LOAD_TARGET_LIGHT`) is a good compromise: it preserves vertical rhythm (baseline, cap-height, x-height align consistently) while keeping the letterform close to the designer's intent.

### 9.4 HiDPI and the Death of Hinting

At HiDPI densities (192 DPI or "2×", typical of 4K displays at arm's reach, or "Retina" MacBook panels), one CSS/logical pixel spans 2×2 physical pixels. At 14pt text on 192 DPI, the rendered glyph is approximately 37×37 physical pixels — large enough that the maximum possible hinting error (±0.5 physical pixel) corresponds to a 1/37 change in glyph height, imperceptible to most viewers.

Best practices for HiDPI on Linux:

1. **Disable hinting**: Set `FT_LOAD_NO_HINTING` or `CAIRO_HINT_STYLE_NONE`.
2. **Use grayscale AA only**: Subpixel rendering is less beneficial at HiDPI (the gains are sub-perceptible and the fringing risk remains). Use `FT_RENDER_MODE_NORMAL`.
3. **Rely on fractional positioning**: At HiDPI, sub-pixel pen positions significantly improve rendering quality without LCD filtering. Use floating-point x positions in `cairo_glyph_t`.

### 9.5 Wayland DPI Negotiation

Wayland exports the physical DPI of each monitor through `wl_output::geometry` (since protocol version 1) with the `physical_width` and `physical_height` fields in millimeters. From physical size and pixel dimensions, the client computes DPI and adjusts font rendering accordingly.

Wayland protocol `wp_fractional_scale_v1` (added to `staging/fractional-scale/` in 2022) allows compositors to advertise non-integer scale factors (e.g., 1.25×, 1.5×, 1.75×) to accommodate displays between standard 96 DPI and 192 DPI. With fractional scaling, font rendering at the native (pre-scale) resolution produces higher-quality results than rendering at the rounded integer scale then downsampling. GTK 4 and Qt 6 both support `wp_fractional_scale_v1`.

GNOME Shell and KDE Plasma both propagate per-monitor DPI information through their compositor implementations to font rendering backends. GNOME uses GSettings keys (`org.gnome.desktop.interface.text-scaling-factor`) as a user-accessible override.

---

## 10. Emoji and Color Fonts

### 10.1 The Emoji Pipeline

Emoji characters are Unicode codepoints in the Miscellaneous Symbols (U+2600–U+26FF), Emoticons (U+1F600–U+1F64F), Supplemental Symbols (U+1F900–U+1FAFF), and other blocks. Most modern emoji are encoded as multi-codepoint sequences:

- **Skin tone modifiers**: U+1F44B (waving hand) + U+1F3FD (medium skin tone modifier) → medium-skin waving hand
- **Gender sequences**: U+1F575 (detective) + ZWJ + U+2642 (male sign) → male detective
- **Family ZWJ sequences**: U+1F468 + ZWJ + U+1F469 + ZWJ + U+1F467 → 👨‍👩‍👧 family emoji
- **Country flags**: regional indicator letters (U+1F1E6–U+1F1FF) pair to form ISO 3166-1 flags

HarfBuzz handles these sequences through its emoji shaping plan: it recognizes ZWJ sequences and modifier sequences, looks them up in the font's GSUB table (or falls back to per-codepoint lookup), and outputs a single glyph ID per sequence when the font provides a composed glyph. [Source: Unicode emoji sequences](https://www.unicode.org/emoji/charts/emoji-zwj-sequences.html)

### 10.2 Color Font Formats in Practice

**Noto Color Emoji** (Google): Originally used CBDT/CBLC (PNG bitmaps at 16, 32, 64, 128, and 512 px). Since the 2022 release, Noto Color Emoji uses **COLRv1**, providing vector color glyphs with gradients that render crisply at any size. COLRv1 glyph descriptions use a paint graph (a tree of paint operations: PaintSolid, PaintLinearGradient, PaintRadialGradient, PaintGlyph, PaintComposite). [Source: COLRv1 Font Spec](https://docs.microsoft.com/en-us/typography/opentype/spec/colr)

**Twemoji** (Twitter): Uses the `SVG` table, with full SVG 1.1 documents per glyph. Large file sizes (~10 MB) and limited renderer support (Firefox, some versions of Pango).

**Apple Color Emoji**: Uses the `sbix` table with per-size PNG images. Supported by FreeType since 2.8 via `FT_LOAD_COLOR`.

### 10.3 FreeType Color Glyph Support

FreeType detects color capabilities with `FT_HAS_COLOR(face)` (returns non-zero if the face contains a color table). To render:

```c
/* Load with color flag to get CBDT or sbix bitmap */
FT_Load_Glyph(face, glyph_id, FT_LOAD_COLOR | FT_LOAD_DEFAULT);
FT_Render_Glyph(face->glyph, FT_RENDER_MODE_NORMAL);
/* face->glyph->bitmap.pixel_mode == FT_PIXEL_MODE_BGRA: BGRA 32bpp */
```

For COLRv1 glyphs, FreeType 2.13 added support for the COLRv1 paint graph via `FT_Get_Color_Glyph_Paint()`, but the caller must traverse the paint tree and execute each paint operation itself. HarfBuzz 6.0+ provides a higher-level color glyph API: `hb_color_glyph_paint_funcs_t` allows the caller to register callbacks for each paint type, and HarfBuzz dispatches through the paint graph.

### 10.4 Emoji in Terminals

Terminal emulators face a specific challenge: they operate on a fixed-cell grid where each cell is *n* × *m* pixels. Unicode assigns East Asian Width (`W` = Wide) to most emoji, meaning they should occupy two terminal cells. A correctly implemented terminal:

1. Calls `wcwidth()` or consults Unicode East_Asian_Width data to determine cell span.
2. Allocates a 2-cell region for wide emoji.
3. Uses color glyph rendering (CBDT, COLR, etc.) to paint the emoji into those cells.
4. Clips correctly when the emoji spans a line boundary or a scrollback boundary.

**foot** terminal (Wayland-native) supports color emoji via FreeType's `FT_LOAD_COLOR` path, rendering COLRv1 glyphs since foot 1.14 (2023). [Source: foot changelog](https://codeberg.org/dnkl/foot/src/branch/master/CHANGELOG.md)

**Kitty** terminal implements full-color emoji via its own compositing engine, using FreeType color bitmaps uploaded to GPU textures. ZWJ sequences in Kitty are shaped using HarfBuzz, and multi-codepoint emoji yield the correct single-glyph output. [Source: Kitty source](https://github.com/kovidgoyal/kitty)

**Ghostty** (libghosty, since 2024) uses HarfBuzz for text shaping including emoji ZWJ sequences, and renders color glyphs to the GPU using its Vulkan/Metal backend. [Source: Ghostty source](https://github.com/ghostty-org/ghostty)

### 10.5 Emoji Width in GTK and GNOME

GTK's emoji picker (added in GTK 3.22) uses Pango's emoji shaping support. Emoji in `GtkLabel` or `GtkTextView` are shaped by HarfBuzz (which recognizes ZWJ sequences via its emoji plan), and the color glyph is rendered by Cairo's FreeType backend using `FT_LOAD_COLOR`. Width measurement for wrapping purposes uses the logical advance from `hb_glyph_position_t.x_advance`, which for a 2-unit-wide emoji glyph is approximately 2em.

---

## Roadmap

### Near-term (6–12 months)

- **HarfBuzz WASM shaper stabilization**: The WebAssembly shaper introduced experimentally in HarfBuzz 8.0 (2023) allows fonts to embed custom WASM programs in a `Wasm` table, bypassing the standard OpenType/AAT/Graphite shaping paths for entirely font-controlled glyph substitution logic. As of 2026 it remains opt-in at build time; the near-term goal is to harden the sandbox, settle the API surface, and enable it by default. [Source: HarfBuzz WASM shaper docs](https://github.com/harfbuzz/harfbuzz/blob/main/docs/wasm-shaper.md)
- **Incremental Font Transfer (IFT) client-side integration**: The W3C IFT specification allows progressive loading of font subsets over HTTP, critical for large CJK fonts on the web. HarfBuzz has shipped subsetting speedups (up to 88% faster for TTF/OTF subsets) as groundwork; the near-term work involves integrating IFT patch-merging into browser font loaders and Pango's font cache. [Source: HarfBuzz 8.0 release notes](https://github.com/harfbuzz/harfbuzz/releases/tag/8.0.0)
- **Fontconfig 2.18.x font-matching regression fix**: Fontconfig 2.18.0 introduced a scoring regression that ranks dynamically-added fonts (via `FcConfigAppFontAddFile`) below system fonts when resolving generic families, breaking applications that register fonts at runtime. Fixes are being tracked upstream. Note: needs verification on exact fix timeline.
- **FreeType 2.14.x maintenance and COLRv1 paint traversal improvements**: FreeType 2.14.3 (March 2026) continues the 2.14.x stable series; near-term work focuses on improving the COLRv1 paint graph API (`FT_Get_Color_Glyph_Paint()`) ergonomics so callers need less manual traversal boilerplate. [Source: FreeType project](https://freetype.org/)
- **Pango variable-font optical sizing**: Pango 1.56.x (current stable) gained COLRv1 and Unicode 16 support; a near-term improvement is automatic `opsz` axis activation based on rendered point size, so text at 8 pt and 72 pt draws from the correct part of the optical-size design space without application intervention. Note: needs verification on exact release.

### Medium-term (1–3 years)

- **COLRv1 + variable font intersection (animated and parametric emoji)**: The combination of COLRv1 gradients with variable-font axes (`wght`, custom axes) enables animated or parametric color emoji. Google's Noto COLRv1 emoji already varies color stops; full integration with CSS `font-variation-settings` and HarfBuzz variation handling is an active area. [Source: COLRv1 Spec](https://docs.microsoft.com/en-us/typography/opentype/spec/colr)
- **GPU-native glyph rasterization replacing FreeType**: Projects including cosmic-text (used in the COSMIC desktop) and browser text engines are exploring moving glyph rasterization fully to the GPU (via signed-distance fields or GPU curve rendering), eliminating the CPU FreeType rasterization + atlas upload round-trip. HarfBuzz would still handle shaping; only the rasterization back-end changes. [Source: cosmic-text GitHub](https://github.com/pop-os/cosmic-text)
- **Wide-gamut and HDR-aware subpixel rendering**: FreeType's LCD filters assume sRGB primaries. On wide-gamut displays the R/G/B subpixel color fringing changes; per-display subpixel filter coefficients calibrated to the display's primaries are a proposed direction. This is coupled to the compositor's color management work (HDR Wayland protocols). Note: needs verification — no formal RFC found as of 2026.
- **OpenType MATH table support in Pango/HarfBuzz**: Mathematical typesetting (TeX-quality layout of MATH-table fonts such as STIX Two and Latin Modern Math) is handled by specialized engines (HarfBuzz has `hb-ot-math.h` querying APIs). Integration into Pango's layout engine for native math layout without an external TeX renderer is a long-discussed design goal. [Source: HarfBuzz OT-MATH API](https://harfbuzz.github.io/harfbuzz-hb-ot-math.html)
- **fontconfig ML-assisted font matching**: Research prototypes using glyph-similarity models to augment fontconfig's heuristic family/weight/style scoring have appeared in the academic literature; practical integration into fontconfig's scoring pipeline remains speculative but discussed. Note: needs verification — no upstream proposal found as of 2026.

### Long-term

- **Unified shaping + layout API**: The current pipeline requires application authors to orchestrate fontconfig → HarfBuzz → FreeType → Cairo/Skia manually or through Pango. Long-term proposals envision a single high-level "text rendering" library that encapsulates the entire pipeline (discovery, shaping, rasterization, atlas management) with a single API surface, reducing the integration burden for terminal emulators, GUI toolkits, and browsers. Note: speculative; no formal proposal as of 2026.
- **Full Unicode script coverage in terminals**: As terminal emulators adopt HarfBuzz for shaping (foot, Kitty, Ghostty already do), the remaining challenge is full complex-script layout within the fixed-cell grid model — Devanagari, Tibetan, and Indic scripts that produce glyphs wider or taller than one cell. Standardized escape sequences or protocol extensions for variable-width terminal cells are a long-term direction. Note: needs verification on specific protocol proposals.
- **Color-managed emoji compositing at the compositor level**: As Wayland compositors adopt ICC/HDR color management, color fonts (COLRv1 sRGB gradients, CBDT sRGB bitmaps) will need compositor-level gamut mapping when displayed on wide-gamut or HDR outputs, rather than the current approach of treating glyph colors as sRGB regardless of display gamut. This requires coordination between FreeType/HarfBuzz, the toolkit, and the compositor's color pipeline. Note: speculative architectural direction.

---

## 11. Integrations

This chapter describes the font rendering pipeline that is consumed by nearly every other chapter covering application-layer rendering on Linux. Key cross-references:

**Chapter 37 (Skia):** Skia's GPU text path, described in §8.2, is the primary text rendering mechanism in Chrome, Chromium-based Electron apps, and Flutter on Linux. Skia bypasses Pango entirely; it calls FreeType and HarfBuzz directly through its own abstraction (`SkTypeface` → `SkFontHost_FreeType` → FreeType, `SkShaper` → HarfBuzz). `SkGpuDevice::drawTextBlob()` drives `GrAtlasManager` for glyph upload described in §8.2.

**Chapter 39 (GTK and Qt):** GTK's text rendering is built entirely on this chapter's pipeline: fontconfig for font lookup, HarfBuzz (since Pango 1.44) for shaping, FreeType for rasterization, Cairo for compositing. The GTK 4 GL renderer (`gsk-gl`) uploads FreeType glyphs to a GPU atlas using GDK's GL/Vulkan backend, providing the GPU-accelerated path described in §8. Qt's `QFontEngineFT` uses the same FreeType base for Linux rendering.

**Chapter 50 (Firefox / Gecko):** Firefox on Linux uses HarfBuzz for all text shaping (including complex scripts, emoji ZWJ sequences, and variable fonts), fontconfig for font lookup, and FreeType for rasterization. Gecko implements color font support internally rather than delegating to FreeType's color bitmap path, giving it control over COLRv1 paint graph traversal and SVG glyph rendering.

**Chapter 52 (WebRender):** WebRender's GPU text pipeline, as described in §8.3, relies on FreeType/HarfBuzz rasterization in the Firefox content process, cross-process bitmap transfer, and GPU atlas allocation in the render process. The shelf allocator used since 2021 is analogous to the glyph atlas packing strategy described in §8.1.

**Chapter 68 (Ghostty terminal):** Ghostty's text rendering uses HarfBuzz for shaping (including complex scripts and emoji), FreeType for rasterization, and a GPU atlas for fast cell rendering, following the GPU text pipeline of §8. The terminal-specific cell-grid constraints (cell width, baseline alignment, wide-character handling) for emoji are described in §10.4.

**Chapter 69 (foot terminal):** foot is a Wayland-native terminal that implements HarfBuzz shaping for complex script support and FreeType color glyph rendering for emoji. foot's architecture (direct Wayland surface compositing, no toolkit layer) makes the font rendering pipeline visible without GTK/Qt abstraction: FreeType bitmaps are uploaded to a `wl_shm` buffer and composited by the Wayland compositor.

**Chapter 101 (Color Science and ICC Profiles):** Subpixel rendering (§9) and color glyph rendering (§10) are both color-managed operations. The LCD filter in FreeType assumes sRGB output; on wide-gamut displays, the subpixel color fringing characteristics change with the display's primary gamut. Additionally, COLRv1 gradient colors specified as linear sRGB values need ICC color management applied by the compositor when the display operates in a wider color space. The per-channel blend operations described in §7.5 assume sRGB gamma; the linear-light blending discussed in Chapter 101 produces different results for antialiased glyph edges.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
