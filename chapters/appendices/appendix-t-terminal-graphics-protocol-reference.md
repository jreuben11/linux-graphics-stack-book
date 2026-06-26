# Appendix T: Terminal Graphics Protocol Reference

> **Reference baseline**: Sixel (DEC VT340 / xterm 389) / Kitty Graphics Protocol 1.5.0 / iTerm2 Inline Images Protocol 3.0 / June 2026
> **Primary sources**: [Kitty Protocol spec](https://sw.kovidgoyal.net/kitty/graphics-protocol/) | [xterm Sixel](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html#h3-Sixel-Graphics) | [iTerm2 Inline Images](https://iterm2.com/documentation-images.html) | [libsixel](https://github.com/libsixel/libsixel)

**Audience**: Terminal and TUI developers implementing image display in terminal emulators or client applications. See Chapters 67–70 (terminal rendering stack) for architecture; see Chapter 67 (Sixel), Chapter 68 (Kitty Protocol), and Chapter 70 (Ghostty/GPU rendering) for implementation detail.

---

## Table of Contents

1. [Protocol Comparison Matrix](#1-protocol-comparison-matrix)
2. [Sixel Protocol](#2-sixel-protocol)
3. [Kitty Graphics Protocol](#3-kitty-graphics-protocol)
4. [iTerm2 Inline Image Protocol](#4-iterm2-inline-image-protocol)
5. [Terminal Support Matrix](#5-terminal-support-matrix)
6. [Detection and Capability Query](#6-detection-and-capability-query)
7. [Common Client Patterns](#7-common-client-patterns)

---

## 1. Protocol Comparison Matrix

| Feature                       | Sixel             | Kitty             | iTerm2            |
|-------------------------------|-------------------|-------------------|-------------------|
| **Escape sequence type**      | DCS (`\eP`)       | APC (`\e_G`)      | OSC 1337 (`\e]1337`) |
| **Pixel colour depth**        | 256 colours (palette) | Full 24-bit RGB + alpha | Full 24-bit RGB + alpha |
| **Alpha / transparency**      | No                | Yes (RGBA)        | No (PNG alpha via base64) |
| **Transfer encoding**         | Inline binary     | base64 or direct fd/shm | base64          |
| **Chunked transfer**          | Implicit (stream) | Yes (m=1/0)       | No                |
| **Placement positioning**     | Cursor position   | Cursor or absolute (col,row) | Cursor position |
| **Virtual image placement**   | No                | Yes (image ID + placement ID) | No          |
| **Shared memory transfer**    | No                | Yes (`t=s`, POSIX shm) | No            |
| **File descriptor transfer**  | No                | Yes (`t=f`)       | No                |
| **Resize/scale**              | Client-side       | Protocol (rows/cols) | Protocol (width/height px or %) |
| **Delete/update image**       | No                | Yes (action=d/t)  | No                |
| **Cursor advance**            | Yes (cell rows)   | Configurable      | Yes               |
| **Animate / frames**          | No                | Yes (z= frame index) | No            |
| **In-band pixel query**       | No                | Yes (`a=q`)       | No                |
| **Standard body**             | ECMA-48 / DEC     | Kitty project     | iTerm2 project    |

---

## 2. Sixel Protocol

### DCS Wrapper

Sixel is wrapped in a Device Control String (DCS):

```
ESC P <params> q <sixel-data> ESC \
DCS  = \x1bP  or  \033P
ST   = \x1b\\ or  \033\\  (String Terminator)
```

Full sequence:
```
\x1bP<P1>;<P2>;<P3>q<color-introductions><sixel-band-data>\x1b\\
```

| Parameter | Position | Meaning                                           | Common values           |
|-----------|----------|---------------------------------------------------|-------------------------|
| P1        | 1st      | Pixel aspect ratio (deprecated in xterm)          | `0` = 2:1, `1` = 5:1, `7` = 1:1 |
| P2        | 2nd      | Background colour handling                        | `0`/`2` = use bg; `1` = transparent |
| P3        | 3rd      | Horizontal grid size (deprecated)                 | `0` = default           |

### Colour Introduction

```
#<register>;<type>;<p1>;<p2>;<p3>
```

| Parameter   | Value | Meaning                          |
|-------------|-------|----------------------------------|
| `<register>`| 0–255 | Colour register number           |
| `<type>`    | 1     | HLS (hue 0–360, luma 0–100, sat 0–100) |
| `<type>`    | 2     | RGB (r,g,b in 0–100 percent)     |

Example: register 3 = red (RGB 100%, 0%, 0%):
```
#3;2;100;0;0
```

### Sixel Band Data

Each Sixel character encodes 6 vertical pixels as a bitmask. Pixel rows are grouped into "bands" of 6 pixels high.

```
Sixel char = 0x3F + bitmask    (range 0x3F '?' to 0x7E '~')
Bit 0 (LSB) = top pixel of the 6-pixel column
Bit 5 (MSB) = bottom pixel
```

**Control characters within Sixel data:**

| Character | ASCII | Meaning                                                |
|-----------|-------|--------------------------------------------------------|
| `#`       | 0x23  | Colour introduction (followed by register/definition)  |
| `!`       | 0x21  | Repeat: `!<count><char>` — repeat char N times         |
| `-`       | 0x2D  | Graphics New Line — advance to next 6-pixel band       |
| `$`       | 0x24  | Graphics Carriage Return — return to start of band     |

### Minimal Example (2×6 pixel red/blue image)

```
\x1bP0;1;0q
#0;2;0;0;100
#1;2;100;0;0
#0!2?
-
#1!2?
\x1b\\
```

### libsixel Quick Reference

```c
#include <sixel.h>
sixel_encoder_t *enc;
sixel_encoder_new(&enc, NULL);
sixel_encoder_setopt(enc, SIXEL_OPTFLAG_WIDTH,   "320");
sixel_encoder_setopt(enc, SIXEL_OPTFLAG_HEIGHT,  "240");
sixel_encoder_setopt(enc, SIXEL_OPTFLAG_COLORS,  "256");
sixel_encoder_encode(enc, "image.png");
sixel_encoder_unref(enc);
```

CLI:
```bash
img2sixel -w 320 -h 240 image.png        # encode to sixel
img2sixel -w auto image.png > out.six    # auto-size to terminal columns
```

---

## 3. Kitty Graphics Protocol

### APC Escape Wrapper

All Kitty protocol messages use the APC escape:

```
ESC _ G <key=value pairs, comma-separated> ; <base64-payload> ESC \
APC = \x1b_  (0x1B 0x5F)
ST  = \x1b\\ (0x1B 0x5C)
```

### Key-Value Reference

Keys are single ASCII letters. Order is not significant. Required keys depend on action.

#### Action (`a=`)

| Value | Action          | Description                                              |
|-------|-----------------|----------------------------------------------------------|
| `t`   | transmit        | Send image data; display immediately                     |
| `T`   | transmit+display| Send and display (default when `a` omitted)              |
| `p`   | put             | Display a previously transmitted image                   |
| `q`   | query           | Query support; terminal responds with `OK` or error      |
| `f`   | frame           | Add an animation frame                                   |
| `d`   | delete          | Delete image or placement                                 |
| `c`   | compose         | Compose frame onto animation                             |

#### Transmission Type (`t=`)

| Value | Type             | Payload format                                            |
|-------|------------------|-----------------------------------------------------------|
| `d`   | direct (default) | Raw pixel data, base64-encoded                            |
| `f`   | file             | File path, base64-encoded                                 |
| `t`   | temporary file   | Temp file path, base64-encoded (terminal deletes after)  |
| `s`   | shared memory    | POSIX shm name, base64-encoded                            |

#### Image Format (`f=`)

| Value | Format          |
|-------|-----------------|
| `32`  | RGBA (default when `f` omitted for direct) |
| `24`  | RGB             |
| `100` | PNG (auto-detects dimensions)             |

#### Key Reference Table

| Key | Name              | Type   | Description                                               |
|-----|-------------------|--------|-----------------------------------------------------------|
| `a` | action            | char   | Action to perform (see above)                             |
| `t` | transmission type | char   | How data is sent (see above)                              |
| `f` | format            | int    | Pixel format: 24 (RGB), 32 (RGBA), 100 (PNG)             |
| `i` | image ID          | uint32 | Unique image ID (0 = auto-assign)                         |
| `I` | image number      | uint32 | Monotonic number; terminal assigns IDs sequentially       |
| `p` | placement ID      | uint32 | Placement ID for virtual placement                        |
| `q` | quiet             | int    | 0 = report OK+error; 1 = report errors only; 2 = silent  |
| `m` | more chunks       | int    | 1 = more data follows; 0 = last chunk (for large images)  |
| `w` | width (pixels)    | int    | Image width in pixels (required for raw formats)          |
| `h` | height (pixels)   | int    | Image height in pixels (required for raw formats)         |
| `c` | columns           | int    | Display width in terminal columns (0 = auto)              |
| `r` | rows              | int    | Display height in terminal rows (0 = auto)                |
| `x` | src x             | int    | Source crop x offset in pixels                            |
| `y` | src y             | int    | Source crop y offset in pixels                            |
| `s` | src width         | int    | Source crop width in pixels                               |
| `v` | src height        | int    | Source crop height in pixels                              |
| `X` | dst x offset      | int    | Destination x offset within cell (pixels, 0–cell_width)   |
| `Y` | dst y offset      | int    | Destination y offset within cell (pixels, 0–cell_height)  |
| `z` | z-index / frame   | int    | Z-index for placement; frame number for animation         |
| `o` | compression       | char   | `z` = zlib-compressed payload                             |
| `S` | data size         | int    | Payload size in bytes (for shm/file)                      |
| `O` | data offset       | int    | Byte offset into file/shm                                 |
| `d` | delete target     | char   | What to delete: `a`=all; `i`=by ID; `c`=at cursor; `p`=placement |
| `C` | cursor movement   | int    | 0 = don't move cursor; 1 = move (default)                 |
| `U` | unicode placeholder | int  | 1 = use Unicode placeholder rendering method              |

### Transmission Examples

**Single chunk PNG (simplest):**
```bash
printf '\x1b_Ga=T,f=100,q=2;'$(base64 -w0 image.png)'\x1b\\'
```

**Chunked transmission (large images, base64 split at 4096 chars):**
```bash
# First chunk: m=1 (more follows)
printf '\x1b_Ga=T,f=32,w=1920,h=1080,m=1;'${chunk1}'\x1b\\'
# Middle chunks
printf '\x1b_Gm=1;'${chunk2}'\x1b\\'
# Last chunk: m=0
printf '\x1b_Gm=0;'${chunkN}'\x1b\\'
```

**Shared memory (zero-copy for large images):**
```bash
# Host writes RGBA pixels to /dev/shm/myimage, then:
printf '\x1b_Ga=T,t=s,f=32,w=800,h=600,S=1920000;'$(printf '/dev/shm/myimage' | base64)'\x1b\\'
```

**Query support:**
```bash
printf '\x1b_Ga=q,i=31,s=1,v=1,a=q;AAAA\x1b\\'
# Terminal responds: \x1b_Gi=31;OK\x1b\\ (or error message)
```

**Virtual placement (image at arbitrary cell position):**
```bash
# Transmit with ID (no display yet):
printf '\x1b_Ga=t,i=42,f=100;'$(base64 -w0 img.png)'\x1b\\'
# Display at specific position (move cursor first, or use p=):
tput cup 5 10   # row 5, col 10
printf '\x1b_Ga=p,i=42,c=20,r=10\x1b\\'
```

**Delete image by ID:**
```bash
printf '\x1b_Ga=d,d=i,i=42\x1b\\'
```

---

## 4. iTerm2 Inline Image Protocol

### OSC 1337 Wrapper

```
ESC ] 1337 ; <command> BEL
or
ESC ] 1337 ; <command> ESC \
```

BEL = `\x07` (preferred for compatibility). `ESC \` also accepted.

### File= Subcommand

```
ESC ] 1337 ; File=[arguments] : <base64-encoded-file-data> BEL
```

Arguments are semicolon-separated `key=value` pairs:

| Argument         | Values               | Default      | Description                                  |
|------------------|----------------------|--------------|----------------------------------------------|
| `name`           | base64(filename)     | —            | Filename hint (displayed in some terminals)  |
| `size`           | int                  | —            | File size in bytes (recommended)             |
| `width`          | `Npx` / `N%` / `N` / `auto` | `auto` | Display width (px, %, chars, or auto)  |
| `height`         | `Npx` / `N%` / `N` / `auto` | `auto` | Display height (same units as width)   |
| `preserveAspectRatio` | `0` / `1`      | `1`          | Maintain aspect ratio when scaling           |
| `inline`         | `0` / `1`            | `0`          | `1` = display inline; `0` = download only   |
| `doNotMoveCursor`| `0` / `1`            | `0`          | `1` = cursor stays at image start position  |

**Minimal example:**
```bash
printf '\x1b]1337;File=inline=1;size=%d:%s\x07' \
    $(wc -c < image.png) \
    $(base64 -w0 image.png)
```

**With explicit dimensions:**
```bash
printf '\x1b]1337;File=inline=1;width=320px;height=240px;preserveAspectRatio=0:%s\x07' \
    $(base64 -w0 image.png)
```

### Other OSC 1337 Commands

| Command                           | Description                              |
|-----------------------------------|------------------------------------------|
| `SetMark`                         | Add a shell integration mark             |
| `CurrentDir=<path>`               | Set current working directory            |
| `SetUserVar=<name>=<base64>`      | Set a user variable                      |
| `ReportCellSize`                  | Query cell size in pixels (response via OSC 1337) |
| `Copy=:<base64>`                  | Copy text to clipboard                   |

---

## 5. Terminal Support Matrix

| Terminal          | Sixel  | Kitty  | iTerm2 | Notes                                              |
|-------------------|--------|--------|--------|----------------------------------------------------|
| Kitty             | No     | Full   | No     | Reference implementation; all Kitty features       |
| Ghostty           | No     | Full   | No     | GPU-accelerated; implements Kitty protocol 1.5+    |
| WezTerm           | Yes    | Full   | Yes    | All three protocols; GPU-accelerated               |
| foot              | Yes    | No     | No     | Wayland-native; best Sixel performance             |
| xterm             | Yes    | No     | No     | Reference Sixel; requires `-ti 340` or `sixelScrolling` |
| mlterm            | Yes    | No     | No     | Full Sixel including scrollback                    |
| iTerm2 (macOS)    | Yes    | No     | Full   | Reference iTerm2 implementation                    |
| Alacritty         | No     | No     | No     | No graphics protocol support as of mid-2026        |
| GNOME Terminal    | No     | No     | No     | VTE-based; no graphics protocol                    |
| Konsole (KDE)     | Yes    | No     | No     | Sixel via libsixel; partial support                |
| tmux              | Yes¹   | Yes¹   | No     | Pass-through required; `allow-passthrough on`      |
| screen            | No     | No     | No     | No graphics support                                |

¹ tmux pass-through: wrap sequences in `\ePtmux;\e<sequence>\x1b\\` for DCS/APC inside tmux.

---

## 6. Detection and Capability Query

### Kitty Query (most reliable)

```bash
# Send query; read response with timeout
printf '\x1b_Ga=q,i=31,s=1,v=1,a=q;AAAA\x1b\\'
# Read response (e.g. with stty + read or a proper terminal library)
# OK response: \x1b_Gi=31;OK\x1b\\
# Unsupported: no response or garbled output
```

### Sixel: Query via Device Attributes

```bash
printf '\x1b[c'   # Primary Device Attributes
# Response: \x1b[?<params>c
# Sixel supported if param list contains '4': e.g. \x1b[?64;4;6c
```

Parse `4` in the `?...c` parameter list:
```bash
IFS=';' read -ra params <<< "${response//[^0-9;]/}"
for p in "${params[@]}"; do [[ "$p" == "4" ]] && echo "sixel supported"; done
```

### iTerm2 / General: `$TERM_PROGRAM`

```bash
case "$TERM_PROGRAM" in
    iTerm.app)   echo "iTerm2 protocol available" ;;
    WezTerm)     echo "WezTerm: all three protocols" ;;
    ghostty)     echo "Kitty protocol available" ;;
esac
```

### Cell Size Query (for pixel positioning)

```bash
# XTWINOPS: query cell size in pixels
printf '\x1b[16t'
# Response: \x1b[6;<cell_height>;<cell_width>t
```

---

## 7. Common Client Patterns

### Check Protocol Support at Runtime

```python
import os, sys, select, termios, tty

def kitty_supported() -> bool:
    fd = sys.stdin.fileno()
    old = termios.tcgetattr(fd)
    try:
        tty.setraw(fd)
        sys.stdout.write('\x1b_Ga=q,i=31,s=1,v=1,a=q;AAAA\x1b\\')
        sys.stdout.flush()
        r, _, _ = select.select([fd], [], [], 0.3)
        return bool(r) and b'OK' in os.read(fd, 100)
    finally:
        termios.tcsetattr(fd, termios.TCSADRAIN, old)
```

### Kitty: Chunked RGBA Transmission (Rust sketch)

```rust
fn send_kitty_image(rgba: &[u8], width: u32, height: u32) {
    use base64::{engine::general_purpose::STANDARD, Engine};
    const CHUNK: usize = 4096;
    let encoded = STANDARD.encode(rgba);
    let chunks: Vec<&str> = encoded.as_bytes().chunks(CHUNK)
        .map(|c| std::str::from_utf8(c).unwrap()).collect();
    let n = chunks.len();
    for (i, chunk) in chunks.iter().enumerate() {
        let more = if i < n - 1 { 1 } else { 0 };
        if i == 0 {
            print!("\x1b_Ga=T,f=32,w={width},h={height},m={more};{chunk}\x1b\\");
        } else {
            print!("\x1b_Gm={more};{chunk}\x1b\\");
        }
    }
}
```

### Sixel: tmux Pass-Through Wrapper

```bash
# Wrap a Sixel DCS in tmux pass-through (so tmux forwards it to the outer terminal)
sixel_data=$(img2sixel image.png)
printf '\x1bPtmux;\x1b%s\x1b\\' "$sixel_data"
# Note: DCS inside tmux pass-through must have each ESC doubled (\x1b → \x1b\x1b)
```

### Protocol Selection Priority

```python
def best_protocol():
    if os.environ.get('TERM_PROGRAM') in ('ghostty',) or kitty_query_ok():
        return 'kitty'
    if os.environ.get('TERM_PROGRAM') == 'iTerm.app':
        return 'iterm2'
    if '4' in da1_params():   # sixel bit in DA1
        return 'sixel'
    return None
```

---

## Cross-References

- **Chapter 67** — Sixel: DEC VT340 history, xterm implementation, libsixel encoder pipeline
- **Chapter 68** — Kitty Graphics Protocol: APC framing, virtual placements, shared-memory transfer, GPU compositor integration
- **Chapter 69** — iTerm2 and WezTerm: OSC 1337 implementation, GPU-accelerated terminal rendering
- **Chapter 70** — Ghostty: GPU-accelerated Kitty protocol implementation; libghostty embedding API
- **Appendix J** — Debugging Quick Reference: diagnosing garbled image output, tmux pass-through issues

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
