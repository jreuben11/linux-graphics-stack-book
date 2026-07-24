# Chapter 192: Ratatui — The Rust TUI Application Framework

*Target audience: Terminal and TUI developers building interactive Rust applications; systems developers who want to understand how a high-level widget abstraction maps onto the VT escape-sequence layer and terminal pixel-graphics protocols; embedded-systems developers interested in running a TUI widget system on LCD or e-ink display hardware; web-platform engineers exploring WebAssembly-based terminal UIs in the browser.*

---

## Table of Contents

1. [Why Ratatui](#1-why-ratatui)
   - [1.1 What is a Terminal User Interface (TUI)?](#11-what-is-a-terminal-user-interface-tui)
   - [1.2 What is Ratatui?](#12-what-is-ratatui)
   - [1.3 What is Immediate-Mode Rendering?](#13-what-is-immediate-mode-rendering)
2. [Core Architecture: Terminal, Backend, Buffer, Frame](#2-core-architecture-terminal-backend-buffer-frame)
3. [The Cell Model and Diff-Based Rendering](#3-the-cell-model-and-diff-based-rendering)
4. [The Widget System: Widget and StatefulWidget](#4-the-widget-system-widget-and-statefulwidget)
5. [Layout and Constraints](#5-layout-and-constraints)
6. [Built-in Widgets Survey](#6-built-in-widgets-survey)
7. [Event Loop Integration via Crossterm](#7-event-loop-integration-via-crossterm)
8. [Pixel Graphics with ratatui-image](#8-pixel-graphics-with-ratatui-image)
9. [Visual Effects with Tachyonfx](#9-visual-effects-with-tachyonfx)
10. [Ratatui on Embedded Hardware: The Mousefood Backend](#10-ratatui-on-embedded-hardware-the-mousefood-backend)
11. [Ratatui in the Browser: Ratzilla and WebAssembly](#11-ratatui-in-the-browser-ratzilla-and-webassembly)
12. [Ecosystem Comparison: Ratatui vs Rich, Textual, Bubble Tea, and ncurses](#12-ecosystem-comparison-ratatui-vs-rich-textual-bubble-tea-and-ncurses)
13. [Roadmap](#13-roadmap)
14. [Integrations](#14-integrations)

---

## 1. Why Ratatui

Terminal user interfaces occupy a peculiar design space. On one hand, they must operate within the rigid character-cell model inherited from 1970s hardware — every output is ultimately a sequence of VT escape codes that position a cursor and emit styled Unicode code points into a grid. On the other hand, modern terminal emulators (kitty, Ghostty, WezTerm, foot) are GPU-accelerated Wayland clients with glyph atlases, Sixel decoding pipelines, and Kitty Graphics Protocol image caches — anything but simple text boxes. A TUI framework must bridge these worlds: expose a composable widget abstraction to the application developer while mapping faithfully onto the escape-sequence layer below.

**Ratatui** ([github.com/ratatui/ratatui](https://github.com/ratatui/ratatui), [docs.rs/ratatui](https://docs.rs/ratatui)) is the community-maintained fork of `tui-rs` (forked 2023 when `tui-rs` went unmaintained). It is written entirely in Rust, compiles to no-std environments, and supports multiple backends through a trait abstraction that — as this chapter explores — extends well beyond traditional terminal emulators to embedded display hardware and WebAssembly browser targets. The crate's 2026 workspace structure splits compilation into sub-crates to reduce build times and stabilise the API surface, while the public API surface available at `ratatui::prelude::*` remains stable across minor versions.

The key insight that makes Ratatui practical is its **immediate-mode, diff-based rendering model**: applications describe what the screen should look like on every frame; the framework computes the delta against the previous frame and emits only the changed cells as VT sequences. This avoids the overheads of a retained-mode scene graph while still keeping terminal output proportional to change rate rather than absolute frame content.

### 1.1 What is a Terminal User Interface (TUI)?

A Terminal User Interface (TUI) is an interactive application that runs inside a terminal emulator and renders its UI entirely through character-cell output — rows and columns of styled Unicode text — rather than through a native windowing system or a bitmap display. TUIs communicate with the terminal emulator via VT escape sequences: control strings that position the cursor, set foreground and background colors, apply text attributes (bold, italic, underline), and manipulate the alternate screen buffer. The terminal emulator interprets these sequences and maps them onto GPU-accelerated glyph rendering, Wayland surface composition, or — on embedded targets — direct framebuffer writes.

TUIs sit above the terminal emulator in the Linux graphics stack: the application writes to a pseudoterminal (PTY) master, the kernel's PTY layer connects it to a terminal emulator process (kitty, Ghostty, foot, or an xterm derivative), and that emulator handles escape-sequence parsing and final pixel rendering. This architecture means a single TUI application binary can run identically in a local Wayland terminal, over an SSH connection, or on a UART-attached serial console — portability that native GUI frameworks cannot match without significant additional work. The downside is that the character-cell grid imposes strict layout constraints: widgets snap to integer cell boundaries, font metrics are fixed by the terminal's glyph atlas, and pixel-accurate graphics require out-of-band protocols such as Sixel or the Kitty Graphics Protocol.

### 1.2 What is Ratatui?

Ratatui is an open-source Rust crate that provides a widget-based abstraction layer over terminal escape-sequence output. Its primary role in the Linux graphics stack is to translate application-level widget descriptions (paragraphs, lists, tables, charts, progress bars) into a compact grid of styled Unicode cells, and from there into VT escape sequences emitted to a terminal backend. Ratatui originated as a community fork of the `tui-rs` crate when that project went unmaintained in 2023 and is now the canonical Rust TUI framework maintained at `github.com/ratatui/ratatui` with an active release cadence.

The crate targets `no_std` environments (with `alloc`), enabling deployment on microcontrollers and embedded displays, as well as WebAssembly targets via the Ratzilla companion crate. Its workspace separates `ratatui-core` (the `Buffer`, `Widget`, and `Layout` machinery) from backend implementations and optional widget sets, reducing compilation costs for downstream crates that embed only the framework core. The public API is exported through `ratatui::prelude::*` and is stable across minor versions. Ratatui integrates with the broader terminal ecosystem through the `crossterm` backend (the default, cross-platform choice) and optional backends for `termion`, the WezTerm terminal library `termwiz`, and the `TestBackend` used in unit tests. [Source: ratatui/ratatui/src/backend/mod.rs](https://github.com/ratatui/ratatui/blob/main/ratatui/src/backend/mod.rs)

### 1.3 What is Immediate-Mode Rendering?

Immediate-mode rendering is a UI architecture in which the application re-describes the entire desired screen state on every frame rather than maintaining a persistent tree of widget objects that are updated incrementally. In a retained-mode GUI (GTK, Qt, or a browser DOM), widgets are long-lived objects; the application mutates their properties and the framework schedules repaints selectively. In immediate-mode, every draw cycle creates widget values from scratch, passes them to the renderer, and discards them — there is no widget lifecycle, no object identity, and no explicit invalidation.

Ratatui adopts immediate-mode rendering at the `Widget` trait level: the `render(self, area, buf)` signature consumes the widget value, making it illegal to retain a widget across frames. The framework's efficiency comes not from skipping work on unchanged widgets but from the diff step that follows rendering: after all widgets have written their cells into the new `Buffer`, the framework compares it against the previous frame's buffer and emits VT sequences only for changed cells. This combination — full redraw into an in-memory buffer followed by diff-based terminal output — gives the simplicity of immediate-mode with output volume proportional to actual change, a property that matters on SSH sessions and low-bandwidth connections where terminal byte counts are constrained.

---

## 2. Core Architecture: Terminal, Backend, Buffer, Frame

```
Application
  │  terminal.draw(|frame| { ... })
  ▼
Terminal<B: Backend>
  │  calls draw() → builds a Buffer of Cells
  ▼
Backend trait
  │  flush() → emits VT escape sequences
  ▼
Terminal emulator (crossterm / termion / wezterm-termwiz / ...)
```

**`Terminal<B>`** is the root type. Parameterised over a `Backend`, it owns two `Buffer` instances (current and previous), manages cursor state, and coordinates the draw/flush cycle.

```rust
use ratatui::{Terminal, backend::CrosstermBackend};
use std::io::stdout;

let backend = CrosstermBackend::new(stdout());
let mut terminal = Terminal::new(backend)?;
```

**`Backend` trait** abstracts over the output mechanism. Implementations ship with the crate:

| Backend | Crate | Use case |
|---|---|---|
| `CrosstermBackend` | `crossterm` | Primary; cross-platform terminal on Windows/macOS/Linux |
| `TermionBackend` | `termion` | UNIX-only; no external C dependency |
| `TermwizBackend` | `termwiz` (WezTerm) | WezTerm's own terminal library |
| `TestBackend` | (built-in) | Unit tests; renders to a string |

[Source: ratatui/src/backend/mod.rs](https://github.com/ratatui/ratatui/blob/main/ratatui/src/backend/mod.rs)

**`Frame`** is the ephemeral rendering context passed to the closure in `terminal.draw()`. It exposes `frame.area()` (the full terminal `Rect`) and `frame.render_widget()` / `frame.render_stateful_widget()`. The closure is called once per draw cycle; after it returns, `Terminal` performs the diff and calls `Backend::flush()`.

```rust
terminal.draw(|frame| {
    let area = frame.area();
    frame.render_widget(my_widget, area);
})?;
```

`Terminal::draw()` returns a `CompletedFrame` containing a reference to the final buffer, useful for testing with `TestBackend`.

---

## 3. The Cell Model and Diff-Based Rendering

### Buffer and Cell

The fundamental unit is `Buffer` — a `Vec<Cell>` indexed by `(x, y)` coordinates within a `Rect`. Each `Cell` holds:

```rust
pub struct Cell {
    pub symbol: CompactString, // UTF-8, usually 1 code point but can be multi-char grapheme
    pub fg: Color,
    pub bg: Color,
    pub underline_color: Color,
    pub modifier: Modifier,    // bitflags: BOLD | ITALIC | UNDERLINED | REVERSED | ...
    pub skip: bool,            // true = compositor owns this cell (e.g. wide-char continuation)
}
```

[Source: ratatui/src/buffer/cell.rs](https://github.com/ratatui/ratatui/blob/main/ratatui/src/buffer/cell.rs)

`Color` supports ANSI named colours (16 values), indexed 256-colour (`Color::Indexed(u8)`), and 24-bit true colour (`Color::Rgb(u8, u8, u8)`). The framework emits the appropriate SGR sequences (`ESC[38;2;r;g;bm` for true colour) based on what the backend and terminal support.

**Wide characters** (CJK ideographs, emoji) occupy two cells. Ratatui tracks this via the `unicode-width` crate and sets `skip: true` on the continuation cell; backends skip continuation cells when emitting cursor-movement sequences, preventing broken interleaving.

### Diff Algorithm

On each `draw()` call:

1. Widgets render into the **new** `Buffer`.
2. `Terminal::flush()` iterates over all positions, comparing new vs. previous cell.
3. Changed cells are collected in screen order; contiguous runs on the same row are batched into a single cursor-move + style-set + content emission.
4. The previous buffer is replaced with the new buffer.

This is an O(N) scan over the terminal grid (where N = rows × columns) but in practice is extremely fast because modern terminals are 80–220 columns × 24–50 rows — under 12 000 cells even on a large monitor. The diff is the only point where VT escape sequences are generated; the widget rendering phase writes only into the in-memory `Buffer` with no I/O.

---

## 4. The Widget System: Widget and StatefulWidget

### The `Widget` Trait

```rust
pub trait Widget {
    fn render(self, area: Rect, buf: &mut Buffer);
}
```

Widgets are **consumed** by `render()` — they are cheap, stack-allocated value types created fresh each frame. This is the direct analogy of immediate-mode UI: there is no retained widget tree, no object lifetime management, no invalidation mechanism. Applications create widgets from their own state and pass them to `frame.render_widget()`.

```rust
let paragraph = Paragraph::new("status: ok")
    .block(Block::bordered().title("Health"))
    .style(Style::default().fg(Color::Green));

frame.render_widget(paragraph, area);
```

### `WidgetRef` and Borrowed Widgets

`WidgetRef` allows widgets to borrow their data:

```rust
pub trait WidgetRef {
    fn render_ref(&self, area: Rect, buf: &mut Buffer);
}
```

Useful when widget data lives in application state that must not be moved out. `WidgetRef` is blanket-implemented for `&T where T: Widget` and for boxed trait objects, enabling `Box<dyn WidgetRef>` for dynamic dispatch in heterogeneous widget collections.

### `StatefulWidget`

Some widgets carry scroll positions, selection state, or other runtime state that persists between frames:

```rust
pub trait StatefulWidget {
    type State;
    fn render(self, area: Rect, buf: &mut Buffer, state: &mut Self::State);
}
```

The canonical example is `List` with `ListState`:

```rust
let mut state = ListState::default().with_selected(Some(0));

// in the draw closure:
let list = List::new(["Item 1", "Item 2", "Item 3"])
    .block(Block::bordered())
    .highlight_style(Style::new().bold());

frame.render_stateful_widget(list, area, &mut state);
```

`ListState` persists across frames (typically stored in the application's top-level struct). Event handling code mutates `state.select(Some(new_idx))` in response to arrow-key events; the next frame picks up the change automatically.

---

## 5. Layout and Constraints

Ratatui's layout engine divides a `Rect` into sub-rects using a `Constraint`-based solver inspired by CSS Flexbox. The engine (using the `cassowary` linear constraint solver or, in recent versions, a custom simpler solver) handles constraint propagation without a full-weight layout pass.

```rust
use ratatui::layout::{Layout, Constraint, Direction};

let chunks = Layout::default()
    .direction(Direction::Vertical)
    .constraints([
        Constraint::Length(3),      // header: exactly 3 rows
        Constraint::Min(0),          // body: remaining space
        Constraint::Length(1),      // status bar: exactly 1 row
    ])
    .split(frame.area());
```

`split()` returns a `Vec<Rect>` with one element per constraint. Nesting layouts — splitting chunks[1] again horizontally — is the standard pattern for multi-pane UIs.

### Constraint Variants

| Variant | Meaning |
|---|---|
| `Length(u16)` | Exact fixed size in cells |
| `Percentage(u16)` | Percentage of parent dimension |
| `Ratio(u32, u32)` | Ratio (e.g. `Ratio(1,3)` = one third) |
| `Min(u16)` | At least this many cells; expands to fill available space |
| `Max(u16)` | At most this many cells; shrinks as needed |
| `Fill(u16)` | Like CSS `flex-grow`; weighted expansion among Fill siblings |

`Fill` was added in Ratatui 0.26 and is the recommended replacement for the common `Min(0)` trick that forces a widget to take all remaining space. With multiple `Fill(n)` constraints, space is distributed proportionally to `n`.

```rust
// Three-column layout with proportional widths
let cols = Layout::horizontal([
    Constraint::Fill(1),   // left: narrow
    Constraint::Fill(3),   // centre: wide
    Constraint::Fill(1),   // right: narrow
]).split(area);
```

[Source: ratatui/src/layout/constraint.rs](https://github.com/ratatui/ratatui/blob/main/ratatui/src/layout/constraint.rs)

---

## 6. Built-in Widgets Survey

Ratatui ships a comprehensive set of widgets under `ratatui::widgets`:

### Block

The fundamental container with optional border, title, and padding:

```rust
Block::bordered()
    .title("My Panel")
    .title_alignment(Alignment::Center)
    .border_style(Style::new().fg(Color::Blue))
    .border_type(BorderType::Rounded)
```

`BorderType` options include `Plain`, `Rounded`, `Double`, `Thick`, `QuadrantInside`, `QuadrantOutside` (using block-drawing Unicode characters for higher visual density).

### Paragraph

Styled text with word-wrap, alignment, and scroll support:

```rust
let text = Text::from(vec![
    Line::from(vec![
        Span::styled("Error: ", Style::new().red().bold()),
        Span::raw("connection refused"),
    ]),
]);

Paragraph::new(text)
    .wrap(Wrap { trim: true })
    .scroll((self.scroll_offset, 0))
```

`Text` → `Line` → `Span` is the styling hierarchy. `Line` can carry a `Style` applied to all its spans as a base, with per-span overrides. The `Style` type combines foreground colour, background colour, underline colour, and `Modifier` bitflags.

### List and Table

`List` renders a vertical sequence of items with optional selection highlight. `Table` renders a grid with column constraints (using the same `Constraint` enum as `Layout`). Both are `StatefulWidget`s with `ListState` / `TableState` carrying the scroll offset and selected row/column index.

### Canvas

`Canvas` provides a coordinate-system drawing API (points, lines, rectangles, circles, maps) using Braille Unicode characters (`⠀`–`⣿`) as a pixel grid at 2×4 sub-cell resolution. One terminal cell ≈ 2×4 Braille dots, yielding an effective pixel grid of `2*width × 4*height` with no image-protocol dependency.

```rust
Canvas::default()
    .x_bounds([0.0, 100.0])
    .y_bounds([0.0, 50.0])
    .paint(|ctx| {
        ctx.draw(&Line { x1: 10.0, y1: 10.0, x2: 90.0, y2: 40.0,
                          color: Color::Yellow });
    })
```

### Chart and Sparkline

`Chart` renders a line graph with axis labels and dataset legends using Unicode box-drawing and Braille characters. `Sparkline` renders a single-row histogram — common in `btop`/`htop`-style dashboards for CPU/network metrics.

### Gauge and BarChart

`Gauge` renders a horizontal progress bar with percentage label; `BarChart` renders a vertical or horizontal bar chart. Both accept `Style` for colour customisation.

---

## 7. Event Loop Integration via Crossterm

A Ratatui application's event loop is entirely application-managed. The framework provides no built-in event dispatch; instead, the application alternates between polling for input events and calling `terminal.draw()`. The most common pattern uses `crossterm::event::poll()` with a timeout:

```rust
use crossterm::event::{self, Event, KeyCode, KeyEventKind};
use std::time::Duration;

loop {
    // Render
    terminal.draw(|frame| render(frame, &mut app_state))?;

    // Event polling with 16ms timeout (≈60 FPS cap)
    if event::poll(Duration::from_millis(16))? {
        match event::read()? {
            Event::Key(key) if key.kind == KeyEventKind::Press => {
                match key.code {
                    KeyCode::Char('q') => break,
                    KeyCode::Up => app_state.scroll_up(),
                    KeyCode::Down => app_state.scroll_down(),
                    _ => {}
                }
            }
            Event::Resize(w, h) => {
                // Terminal::autoresize() handles this if called before draw(),
                // but explicit handling is needed if layout depends on size.
                app_state.update_size(w, h);
            }
            _ => {}
        }
    }
}
```

### SIGWINCH and TIOCGWINSZ

When the terminal window is resized, the kernel sends `SIGWINCH` to the foreground process group (see Ch178 for the PTY/TTY mechanics). Crossterm on Linux registers a `SIGWINCH` handler via `libc::signal()` that sets a flag; the next `event::poll()` call synthesises an `Event::Resize(cols, rows)` by calling `ioctl(TIOCGWINSZ)` on the terminal file descriptor.

Ratatui's `Terminal::autoresize()` — called internally by `draw()` before the user closure runs — queries the backend for the current size and reallocates the buffer pair if it has changed:

```rust
// internal to Terminal::draw()
let size = self.backend.size()?;
if size != self.viewport.area {
    self.resize(size)?;
}
```

[Source: ratatui/src/terminal/mod.rs](https://github.com/ratatui/ratatui/blob/main/ratatui/src/terminal/mod.rs)

Applications do not need to handle `Event::Resize` manually unless their layout logic depends on knowing the new size before the next draw cycle. The buffer reallocation in `autoresize()` preserves the previous buffer contents for the diff on the first post-resize frame, avoiding a full-screen redraw.

### Mouse Event Support

Crossterm exposes `Event::Mouse(MouseEvent)` when mouse tracking is enabled via `crossterm::execute!(stdout, EnableMouseCapture)`. `MouseEvent` carries the kind (`Down`, `Up`, `Drag`, `Moved`, `ScrollDown`, `ScrollUp`), button, column, row, and modifier keys. The application maps `(column, row)` to widget areas from the last layout to implement hit-testing.

---

## 8. Pixel Graphics with ratatui-image

`ratatui-image` ([github.com/ratatui/ratatui-image](https://github.com/ratatui/ratatui-image), [docs.rs/ratatui-image](https://docs.rs/ratatui-image)) bridges Ratatui's widget system to the three pixel-graphics protocols described in Chapter 43: Sixel, Kitty Graphics Protocol, and iTerm2 Inline Images. It also provides a `Halfblocks` fallback using Unicode upper/lower half-block characters (`▀`/`▄`) that works in any colour terminal with no image-protocol support.

### Protocol Detection: Picker

The `Picker` type automates terminal capability detection at startup. It queries the running terminal's response to secondary DA (`ESC[c`) and issues terminal-specific OSC queries to determine font cell dimensions — necessary for pixel-accurate image scaling.

```rust
use ratatui_image::picker::Picker;

// Queries terminal interactively: sends escape sequences, reads responses.
// Must be called before entering raw mode or before terminal.draw() loop.
let picker = Picker::from_query_stdio()?;

println!("Protocol: {:?}", picker.protocol_type());
println!("Font size: {:?}", picker.font_size());
```

`Picker::protocol_type()` returns a `ProtocolType` enum: `Sixel`, `Kitty`, `Iterm2`, or `Halfblocks`. The picker interrogates the terminal in priority order: Kitty first (most capable), then iTerm2, then Sixel (requires `\033[c` response to include `4` in the capability list for Sixel support), falling back to `Halfblocks`.

`Picker::font_size()` returns `(width_px, height_px)` per cell. It issues `ESC[14t` (report text area size in pixels) and `ESC[18t` (report terminal size in characters) and divides. This is critical for `Resize::Fit` — without accurate cell dimensions, images are scaled to wrong aspect ratios.

### Image Widgets

Two widget types handle image rendering:

**`Image`** (stateless, `Widget`):

```rust
use ratatui_image::{Image, Resize, picker::Picker};
use image::DynamicImage;

let dyn_img: DynamicImage = image::open("photo.png")?;
let protocol = picker.new_protocol(dyn_img, area, Resize::Fit(None))?;
let image_widget = Image::new(&protocol);
frame.render_widget(image_widget, area);
```

`new_protocol()` performs the encoding (quantization for Sixel, base64 for Kitty/iTerm2) at construction time, so `render()` is non-blocking. This is appropriate when the image is known at draw time and the encoding latency is acceptable in the calling thread.

**`StatefulImage`** (stateful, `StatefulWidget`):

```rust
use ratatui_image::{StatefulImage, protocol::StatefulProtocol};

let mut proto: Box<dyn StatefulProtocol> = picker.new_resize_protocol(dyn_img);

// in draw closure:
frame.render_stateful_widget(StatefulImage::default(), area, &mut proto);
```

`StatefulImage` re-encodes the image lazily at render time, adapting to the current `area`. This is the correct choice for images whose display area changes (e.g. images in resizable panes). Blocking encoding should be offloaded to a background thread via `ThreadProtocol`:

```rust
use ratatui_image::protocol::thread::ThreadProtocol;

let thread_proto = ThreadProtocol::new(&picker, dyn_img);
// Spawns an OS thread; poll thread_proto.is_ready() or handle the notification.
```

### Resize Enum

| Variant | Behaviour |
|---|---|
| `Resize::Fit(None)` | Scale to fit area preserving aspect ratio |
| `Resize::Fit(Some((w, h)))` | Scale to fit within explicit pixel bounds |
| `Resize::Crop(alignment)` | Crop to fill area without scaling |
| `Resize::Scale(w, h)` | Force scale to exact pixel dimensions |

### Protocol Details

Under the hood, `ratatui-image` renders images into Ratatui `Buffer` cells differently per protocol:

- **Sixel**: quantizes to ≤256 colours (Wu quantizer), encodes DCS bands, stores the entire escape sequence as a special cell value that the backend emits verbatim. The `Cell::symbol` field is set to the full Sixel DCS string — bypassing Ratatui's normal per-character escape-sequence generation for those cells.
- **Kitty**: transmits chunked base64-encoded RGBA/PNG via APC sequences with image ID allocation; stores the `a=p` placement command in the cell. The image data is transmitted once and re-placed each frame using only the placement APC.
- **Halfblocks**: renders two pixels per cell using `▀` (upper half block) with `bg=lower_pixel_color` and `fg=upper_pixel_color`. Effective resolution is `2*width × 2*height` (not 4-row Braille) but requires only standard SGR styling.

---

## 9. Visual Effects with Tachyonfx

**Tachyonfx** ([github.com/ratatui/tachyonfx](https://github.com/ratatui/tachyonfx), [docs.rs/tachyonfx](https://docs.rs/tachyonfx)) adds shader-like post-processing effects to Ratatui applications. Effects operate on an already-rendered `Buffer` — they are applied *after* widgets have written their cells, transforming cell colours, characters, or positions to produce animations, transitions, and visual feedback.

### The Effect Model

Effects are stateful objects that consume elapsed time and transform buffer cells. The core application point is:

```rust
use tachyonfx::{EffectRenderer, fx};
use std::time::Duration;

// Create an effect
let mut effect = fx::dissolve(Duration::from_millis(800));

// In the draw closure, after widget rendering:
frame.render_effect(&mut effect, area, elapsed);
```

`render_effect()` (provided by the `EffectRenderer` trait, implemented for `Frame`) applies the effect to every cell in `area`, advancing the effect's internal timer by `elapsed`. Effects return a boolean from their internal `render()` call: `true` means the effect is still running; `false` means it has completed. `EffectManager` tracks multiple effects and removes completed ones automatically.

### Effect Categories

**Colour transitions:**

```rust
// Fade in from black over 500ms
let fade_in = fx::fade_from(Color::Black, Duration::from_millis(500));

// Gradually saturate foreground colours
let saturate = fx::saturate_fg(1.0, Duration::from_millis(300));

// HSL hue rotation
let hue_cycle = fx::hsl_shift_fg((360.0, 0.0, 0.0), Duration::from_secs(2));
```

**Text materialisation and dissolution:**

```rust
// Characters coalesce from random positions
let appear = fx::coalesce(Duration::from_millis(600));

// Characters dissolve into space
let disappear = fx::dissolve(Duration::from_millis(400));

// Characters evolve through a symbol set before settling
let glitch = fx::evolve(vec!["@#$%&*", "░▒▓", "abc"], Duration::from_millis(500));
```

**Directional motion:**

```rust
// Slide content in from the left
let slide = fx::slide_in(Direction::Left, 0, Color::Black, Duration::from_millis(300));

// Sweep a colour band across the area
let sweep = fx::sweep_in(Direction::LeftToRight, 10, Color::Blue, Duration::from_millis(400));
```

**Particle effects:**

```rust
// Cells explode outward from the centre
let explode = fx::explode(Duration::from_millis(500));
```

### Composition

Effects can be combined with sequencing and parallelism combinators:

```rust
use tachyonfx::fx;

// Run two effects simultaneously
let parallel = fx::parallel(&[
    fx::fade_from(Color::Black, Duration::from_millis(400)),
    fx::coalesce(Duration::from_millis(600)),
]);

// Run effects in sequence
let sequence = fx::sequence(&[
    fx::sweep_in(Direction::LeftToRight, 5, Color::Black, Duration::from_millis(300)),
    fx::fade_from(Color::Black, Duration::from_millis(200)),
]);

// Loop an effect
let pulsing = fx::repeat(fx::hsl_shift_fg((360.0, 0.0, 0.0), Duration::from_secs(3)),
                          RepeatCount::Infinite);

// Delay before starting
let delayed = fx::delay(Duration::from_millis(200), fx::coalesce(Duration::from_millis(400)));
```

### Spatial Patterns

Effects support **spatial patterns** that modulate when different cells begin their transition, creating wave-like or radial-sweep animations:

```rust
use tachyonfx::fx;
use tachyonfx::fx::effect_fn;
use tachyonfx::SpatialPattern;

// Dissolve starting from the centre, spreading outward
let radial_dissolve = fx::dissolve(Duration::from_millis(800))
    .with_pattern(SpatialPattern::radial());

// Diagonal wave
let diagonal_sweep = fx::coalesce(Duration::from_millis(600))
    .with_pattern(SpatialPattern::diagonal());

// Checkerboard stagger
let checker = fx::fade_from(Color::Black, Duration::from_millis(400))
    .with_pattern(SpatialPattern::checkerboard());
```

Available patterns: `radial()`, `diagonal()`, `checkerboard()`, `spiral()`, `wave()`. Patterns control a per-cell *start delay offset* — cells further from the pattern's origin start their animation later, creating smooth spatial propagation with a single `Effect` instance.

### Custom Shaders via the `Shader` Trait

Applications can define custom effects by implementing the `Shader` trait:

```rust
use tachyonfx::{Shader, ShaderContext, Cell, CellIterator};

pub struct RainbowShader { cycle: f32 }

impl Shader for RainbowShader {
    fn render(&mut self, ctx: &ShaderContext, mut cells: CellIterator<'_>) -> bool {
        let t = ctx.elapsed().as_secs_f32() + self.cycle;
        for (pos, cell) in cells {
            let hue = (pos.x as f32 * 0.1 + t * 120.0) % 360.0;
            cell.fg = Color::from_hsl(hue, 1.0, 0.5);
        }
        !ctx.timer().done()
    }
}

let rainbow = RainbowShader { cycle: 0.0 }.into_effect(Duration::from_secs(5));
```

`into_effect()` wraps any `Shader` implementor into an `Effect` value usable with `fx::parallel()`, `EffectManager`, and `frame.render_effect()`.

### EffectManager

`EffectManager` coordinates multiple named effects, allowing per-effect cancellation and replacement:

```rust
use tachyonfx::EffectManager;

let mut fx_manager = EffectManager::default();

// Push an effect with a label
fx_manager.push("transition", fx::dissolve(Duration::from_millis(600)));
fx_manager.push("highlight", fx::fade_from(Color::Yellow, Duration::from_millis(200)));

// In the draw closure:
fx_manager.process_effects(elapsed, frame.buffer_mut(), frame.area());

// Cancel by label
fx_manager.cancel("highlight");
```

---

## 10. Ratatui on Embedded Hardware: The Mousefood Backend

**Mousefood** ([github.com/ratatui/mousefood](https://github.com/ratatui/mousefood), [docs.rs/mousefood](https://docs.rs/mousefood)) demonstrates the reach of Ratatui's `Backend` abstraction: it implements the trait not for a terminal emulator but for **embedded-graphics** display hardware — microcontrollers driving LCD, OLED, and e-ink panels. The name reflects the crate's position in the Ratatui ecosystem as a feeding layer for `embedded-graphics` displays.

This is architecturally significant. The same `Widget`, `Layout`, and `StatefulWidget` abstractions used in a desktop TUI dashboard can run on an RP2040 driving an SSD1306 OLED, on an ESP32-S3 driving an ILI9341 LCD, or in a simulation window on a development machine — with no application code changes, only backend substitution.

### Rendering Pipeline Contrast

```
Standard Ratatui:
  Widgets → Buffer → CrosstermBackend → VT escape sequences → Terminal emulator → GPU → Display

Mousefood:
  Widgets → Buffer → EmbeddedBackend<D> → embedded-graphics primitives → Display driver → Hardware
```

`EmbeddedBackend<D>` translates Ratatui's `Buffer` — a grid of `Cell` values with Unicode characters and `Style` attributes — into `embedded-graphics` drawing operations. Each cell becomes a glyph drawn at the corresponding pixel position using a configurable font. The font selection determines whether Unicode box-drawing characters (used by `Block::bordered()`) and Braille characters (used by `Canvas` and `Sparkline`) render correctly: Mousefood uses `embedded-graphics-unicodefonts` by default, as standard embedded-graphics fonts cover only ASCII.

### Setup

```toml
# Cargo.toml
[dependencies]
mousefood = "0.1"
embedded-graphics = "0.8"
ratatui = "0.29"

# For RP2040 + SSD1306 OLED
ssd1306 = "0.9"
```

```rust
use mousefood::{EmbeddedBackend, EmbeddedBackendConfig};
use ratatui::{Terminal, widgets::{Block, Paragraph}};
use ssd1306::{Ssd1306, DisplaySize128x64, I2CDisplayInterface, mode::BufferedGraphicsMode};

let interface = I2CDisplayInterface::new(i2c);
let mut display = Ssd1306::new(interface, DisplaySize128x64, DisplayRotation::Rotate0)
    .into_buffered_graphics_mode();
display.init().unwrap();

let config = EmbeddedBackendConfig::default();
let backend = EmbeddedBackend::new(&mut display, config);
let mut terminal = Terminal::new(backend)?;

terminal.draw(|frame| {
    let area = frame.area();
    frame.render_widget(
        Paragraph::new("Hello from Ratatui!")
            .block(Block::bordered().title("RP2040")),
        area,
    );
})?;
```

### EmbeddedBackendConfig

Configuration options include font selection, colour-theme mapping, cursor style, blink support, and a framebuffer mode for displays that require a full-screen upload before flush:

| Config field | Purpose |
|---|---|
| Font | `UnicodeFont` variant; controls glyph rendering quality and Unicode coverage |
| `color_theme` | Maps ANSI colours to display-specific RGB values (ANSI theme, Tokyo Night theme, custom) |
| `cursor_style` | `Inverse`, `Underline`, `Outline`, `Japanese` (uses katakana cursor glyph) |
| `blink` (feature) | Enables blinking cell support via timer |
| Framebuffer | For display drivers requiring `display.flush()` after each frame |

### Supported Hardware

| MCU | Display driver | Notes |
|---|---|---|
| ESP32, ESP32-S3, ESP32-C6 | ILI9341, ST7735 | Color LCD common on dev boards |
| RP2040, RP2350 | SSD1306 | I²C OLED; 128×64 mono |
| STM32 | ST7735 | SPI LCD |
| Any | Waveshare EPD, WeAct EPD | E-ink; partial refresh for low-power dashboards |
| Any (dev) | `embedded-graphics-simulator` | SDL2 window for development |

### Development with the Simulator

The `simulator` feature flag activates `embedded-graphics-simulator`, which opens an SDL2 window mimicking the target display:

```rust
#[cfg(feature = "simulator")]
{
    use embedded_graphics_simulator::{SimulatorDisplay, OutputSettingsBuilder, Window};
    let mut sim_display = SimulatorDisplay::<Rgb888>::new(Size::new(128, 64));
    let backend = EmbeddedBackend::new(&mut sim_display, config);
    // ... same Terminal::new() and draw() code as on hardware
}
```

This allows developing and testing Ratatui-based embedded UIs on a Linux workstation before flashing to hardware — the same `Widget` tree renders identically in both environments.

---

## 11. Ratatui in the Browser: Ratzilla and WebAssembly

**Ratzilla** ([github.com/ratatui/ratzilla](https://github.com/ratatui/ratzilla), [docs.rs/ratzilla](https://docs.rs/ratzilla)) enables Ratatui applications to run in the browser via WebAssembly, replacing the terminal backend with a browser-native rendering surface. The result is a terminal-themed web application: familiar TUI widgets (tables, lists, progress bars, charts) rendered in a browser tab, without requiring a terminal emulator or SSH session.

The connection to the Linux graphics stack discussed in Part X is direct: Ratzilla's rendering ultimately goes through the browser's GPU compositor — the same Chromium Viz/OOP-D pipeline (Ch33/Ch36) or Firefox WebRender (Ch52) that composites any other WebGL or Canvas element.

### Rendering Backends

Ratzilla provides three browser rendering backends:

**`DomBackend`**: Renders each `Cell` as a `<span>` element in a `<div>` grid. Accessible (screen readers see the text), but slow for complex UIs due to DOM diffing overhead. Best for applications where accessibility matters more than animation performance.

**`CanvasBackend`**: Renders cells using the HTML5 Canvas 2D API (`fillText`, `fillRect`). Faster than DOM for dense grids; no accessibility. Suitable for most interactive TUI applications.

**`WebGL2Backend`**: Uses WebGL2 for high-performance rendering with a configurable font atlas. Accepts a `FontAtlasConfig` enum controlling atlas dimensions and font size. Best for animation-heavy applications using `tachyonfx`. On Linux/Chromium, the `gl` calls go through ANGLE (Ch34) → Mesa/RADV or ANV → kernel DRM.

```toml
# Cargo.toml
[dependencies]
ratzilla = "0.1"
ratatui = "0.29"

[lib]
crate-type = ["cdylib"]
```

### Application Structure

```rust
use ratzilla::{WebTerm, backend::DomBackend};
use ratatui::widgets::{Block, Paragraph};
use ratatui::prelude::*;

#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    let backend = DomBackend::new()?;
    let mut term = WebTerm::new(backend)?;

    term.draw_web(|frame| {
        let area = frame.area();
        frame.render_widget(
            Paragraph::new("Hello from Ratzilla!")
                .block(Block::bordered().title("Browser TUI"))
                .alignment(Alignment::Center),
            area,
        );
    })?;

    Ok(())
}
```

`WebTerm` wraps the Ratatui `Terminal` type with browser-specific lifecycle management. `draw_web()` behaves identically to `terminal.draw()` — the same `Frame`, `render_widget()`, and `Layout::split()` calls work unchanged.

### WebRenderer and WebEventHandler Traits

The `WebRenderer` trait abstracts over the three backend rendering implementations:

```rust
pub trait WebRenderer {
    fn draw(&mut self, buffer: &Buffer) -> Result<(), Error>;
    fn clear(&mut self) -> Result<(), Error>;
    fn size(&self) -> Result<(u16, u16), Error>;
}
```

`WebEventHandler` bridges browser input events (keyboard, mouse, resize) into the `crossterm::event` types that Ratatui applications already handle:

```rust
pub trait WebEventHandler {
    fn on_key(&self, event: KeyboardEvent) -> Option<crossterm::event::Event>;
    fn on_mouse(&self, event: MouseEvent) -> Option<crossterm::event::Event>;
    fn on_resize(&self, w: u32, h: u32) -> Option<crossterm::event::Event>;
}
```

This means an existing Ratatui event-loop — written against `crossterm::event::Event` — runs unchanged in the browser. The `Event::Resize` handling that responds to `SIGWINCH` / `TIOCGWINSZ` in the terminal context (Ch178) instead fires when the browser window resizes, because `WebEventHandler` synthesises the same `Event::Resize` variant from a browser `resize` DOM event.

### WASM Compilation and Deployment

```bash
# Install Trunk (WASM bundler)
cargo install trunk

# Add WASM target
rustup target add wasm32-unknown-unknown

# Build and serve
trunk serve --release
# → opens http://localhost:8080, auto-rebuilds on file changes

# Production build
trunk build --release
# → generates dist/ with index.html, pkg/*.wasm, pkg/*.js
```

Trunk handles `wasm-bindgen` code generation, asset copying, and the HTML wrapper. The output is a static site deployable to any web server or CDN.

### CellSized Trait

`CellSized` ensures consistent character dimensions across backends:

```rust
pub trait CellSized {
    fn cell_width(&self) -> f64;
    fn cell_height(&self) -> f64;
}
```

`CanvasBackend` and `WebGL2Backend` use `cell_width`/`cell_height` to position glyphs on the Canvas/WebGL surface. The `DomBackend` uses CSS `ch`/`em` units instead. Applications that need pixel-perfect alignment (e.g. overlaying a `ratatui-image` Kitty-protocol widget in the browser) must account for the rendered cell size when computing pixel positions — the same concern as `ratatui-image`'s `FontSize` query in a real terminal, but obtained from the browser's font metrics API rather than a terminal escape response.

### Combining Ratzilla with Tachyonfx

Because Tachyonfx operates on `ratatui::Buffer` (the in-memory cell grid), it is fully compatible with Ratzilla — effects applied via `frame.render_effect()` modify cells before `WebRenderer::draw()` serialises them to DOM/Canvas/WebGL. A Ratzilla app with `tachyonfx` effects uses the browser's `requestAnimationFrame` loop for timing:

```rust
use ratzilla::animation::AnimationFrame;
use tachyonfx::{EffectManager, fx};

let mut effects = EffectManager::default();
effects.push("intro", fx::coalesce(Duration::from_millis(800)));

AnimationFrame::new(move |elapsed| {
    term.draw_web(|frame| {
        frame.render_widget(my_widget, frame.area());
        effects.process_effects(elapsed, frame.buffer_mut(), frame.area());
    }).unwrap();
}).run();
```

On Linux/Chromium, this rendering pipeline runs as: JavaScript `requestAnimationFrame` → Ratzilla draw → WebGL2 draw calls → ANGLE (GLES2→Vulkan translation) → Mesa RADV/ANV → kernel DRM → KMS scanout — the full chain described across Ch33–Ch36 for any WebGL application.

---

## 12. Ecosystem Comparison: Ratatui vs Rich, Textual, Bubble Tea, and ncurses

TUI frameworks exist in every major systems language. This section compares Ratatui against the four most commonly encountered alternatives, focusing on the architectural decisions that matter when choosing a framework for a Linux graphics-adjacent terminal application.

### Python Rich

**Rich** (by Will McGugan) is a Python library for rich text and beautiful formatting in the terminal. It is not a full TUI framework — it has no event loop, no widget state model, and no input handling. Its domain is **output formatting**: syntax highlighting, tables, progress bars, markdown rendering, tracebacks, and log formatting, all streamed to stdout.

```python
from rich.console import Console
from rich.table import Table

console = Console()
table = Table(title="GPU Adapters")
table.add_column("Device", style="cyan")
table.add_column("VRAM", justify="right")
table.add_row("Radeon RX 7900 XTX", "24 GB")
table.add_row("RTX 4090", "24 GB")
console.print(table)
```

Rich's rendering model is **streaming**: it writes escape sequences to stdout one line at a time. There is no off-screen buffer, no diff algorithm, no full-screen repaint — Rich's `Console.print()` is essentially a sophisticated `print()`. This makes it excellent for CLI tools that produce formatted output, but unsuitable for interactive full-screen applications.

| Aspect | Ratatui | Rich |
|---|---|---|
| Paradigm | Immediate-mode full-screen TUI | Streaming formatted output |
| Event loop | Application-owned (crossterm/termion) | None |
| Input handling | Via crossterm `event::read()` | None |
| Screen model | Double-buffer diff | Single-pass stdout write |
| Primary use | Interactive TUI apps | CLI output, logging, progress |
| Language | Rust | Python |
| Pixel graphics | Via ratatui-image | Via `rich.console` image (limited) |

[Source: Rich documentation](https://rich.readthedocs.io/)

### Python Textual

**Textual** (also by Will McGugan, built on Rich) is a full TUI application framework for Python. Unlike Rich, it has an event loop, widget composition, CSS-like layout, reactive state, and mouse support — making it the closest Python equivalent to Ratatui.

```python
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, DataTable

class GPUMonitor(App):
    CSS = "DataTable { height: 1fr; }"

    def compose(self) -> ComposeResult:
        yield Header()
        yield DataTable()
        yield Footer()

    def on_mount(self) -> None:
        table = self.query_one(DataTable)
        table.add_columns("Device", "Utilisation", "VRAM")
        table.add_rows([("RX 7900 XTX", "87%", "18/24 GB")])

app = GPUMonitor()
app.run()
```

Textual uses a **CSS-inspired layout engine** (flexbox-like, with Textual's own CSS dialect) rather than Ratatui's constraint-based `Layout`. Widgets are class instances with persistent state (retained-mode), reactive attributes that trigger re-renders, and an async message-passing system for inter-widget communication. Ratatui is immediate-mode (widgets are stateless render functions; state lives in the application); Textual is retained-mode (widgets own their state).

| Aspect | Ratatui | Textual |
|---|---|---|
| Language | Rust | Python |
| Rendering model | Immediate-mode (stateless widgets) | Retained-mode (stateful widget objects) |
| Layout | Constraint solver (`Length`, `Percentage`, `Min`, `Max`, `Ratio`) | CSS-like (flexbox, grid) |
| State management | Application-owned, manual | Reactive attributes (`reactive`, `watch`) |
| Async integration | Via `tokio`/`async-std` manually | Built-in async event loop (asyncio) |
| Styling | `Style` struct (fg, bg, modifiers) | Full CSS subset with theming |
| Testing | `TestBackend` + `Buffer::assert_eq` | `App.run_test()` async test harness |
| Pixel graphics | ratatui-image (Kitty/Sixel/iTerm2) | None (terminal images not supported) |
| Startup time | ~1ms (Rust binary) | ~200ms+ (Python startup + imports) |
| Deployment | Single static binary | Python + virtualenv |

Textual's CSS layout and retained-mode widgets make complex layouts more ergonomic to write; Ratatui's immediate-mode model is more predictable for performance-sensitive rendering loops (no hidden re-render triggers from reactive attribute chains). [Source: Textual documentation](https://textual.textualize.io/)

### Go Bubble Tea

**Bubble Tea** (by Charm, written in Go) is a TUI framework based on **The Elm Architecture**: a pure functional update model where the application is a `Model` struct, a `Update(msg) (Model, Cmd)` function, and a `View(Model) string` function. There is no mutable state — each event produces a new model.

```go
package main

import (
    tea "github.com/charmbracelet/bubbletea"
    "fmt"
)

type model struct { count int }

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        if msg.String() == "+" { return model{m.count + 1}, nil }
        if msg.String() == "q" { return m, tea.Quit }
    }
    return m, nil
}

func (m model) View() string {
    return fmt.Sprintf("Count: %d\n\nPress + to increment, q to quit.", m.count)
}

func main() {
    tea.NewProgram(model{}).Run()
}
```

Bubble Tea's `View()` returns a **string** of escape sequences — not a structured widget tree. Layout is performed by string concatenation or the companion **Lip Gloss** library (CSS-like string styling). The framework re-renders by calling `View()` on every update and diffing the resulting strings line-by-line to emit minimal escape sequences. Ratatui, by contrast, diffs at the **cell level** (per character with attributes), which is more precise but requires a structured widget system rather than string rendering.

| Aspect | Ratatui | Bubble Tea |
|---|---|---|
| Language | Rust | Go |
| Architecture | Immediate-mode widget tree | Elm Architecture (Model/Update/View) |
| Render unit | Cell (char + style) | String (escape sequences) |
| Diff granularity | Per-cell (precise) | Per-line (coarser) |
| Layout | Constraint solver | Lip Gloss string composition |
| Concurrency | Rust ownership model | Go goroutines + `tea.Cmd` |
| Async I/O | Tokio / async-std | Goroutines via `tea.Cmd` |
| Ecosystem | ratatui-image, tachyonfx, ratzilla | Bubble components (list, spinner, viewport…) |
| Binary size | ~3–5 MB (static) | ~8–12 MB (Go runtime embedded) |
| Pixel graphics | Kitty/Sixel/iTerm2 via ratatui-image | None |

The Elm Architecture in Bubble Tea makes unit-testing straightforward (pure functions, no side effects in `Update`), but complex applications with many event sources can become difficult to manage with a single monolithic `Update` function. Ratatui's immediate-mode model defers architectural decisions about state management to the application, enabling patterns from simple `match event` loops to full actor systems via Tokio channels. [Source: Bubble Tea repository](https://github.com/charmbracelet/bubbletea)

### ncurses

**ncurses** (and its successor **ncursesw** for Unicode) is the oldest and most widely deployed TUI library, providing a C API that abstracts terminal capabilities via the **terminfo** database. It is the substrate beneath applications like `vim`, `htop`, `mc` (Midnight Commander), and `less`.

```c
#include <ncurses.h>

int main(void) {
    initscr();
    cbreak();
    noecho();
    keypad(stdscr, TRUE);

    WINDOW *win = newwin(10, 40, 5, 10);
    box(win, 0, 0);
    mvwprintw(win, 1, 2, "GPU: RX 7900 XTX");
    mvwprintw(win, 2, 2, "VRAM: 18/24 GB");
    wrefresh(win);

    getch();
    endwin();
    return 0;
}
```

ncurses uses a **retained-mode window model** (`WINDOW*` structs) with an explicit `refresh()`/`wrefresh()` call to flush changes. The diff between the virtual screen and the physical screen is computed by ncurses internally and emitted as minimal escape sequences. It reads terminal capabilities from `terminfo` (e.g. `/usr/share/terminfo/x/xterm-256color`) rather than hardcoding sequences, making it portable across terminal types.

| Aspect | Ratatui | ncurses |
|---|---|---|
| Language | Rust | C (bindings in many languages) |
| Rendering model | Immediate-mode, full buffer diff | Retained-mode `WINDOW*`, explicit `refresh()` |
| Terminal capabilities | Hardcoded ANSI + crossterm detection | terminfo database |
| Unicode | Full (Rust `char` = Unicode scalar) | ncursesw (wide-char extension required) |
| Mouse support | crossterm mouse events | `mousemask()` + `MEVENT` |
| Thread safety | Rust ownership (safe by construction) | Not thread-safe without wrappers |
| Pixel graphics | ratatui-image (Kitty/Sixel/iTerm2) | None |
| Memory model | Stack-allocated `Buffer` | Heap `WINDOW*` graph |
| Binary dependency | No dynamic libs (crossterm is pure Rust) | `libncurses.so` (dynamic link) |
| Maturity | ~2019 (tui-rs) / 2023 (ratatui fork) | 1993 (ncurses) / 1986 (curses) |

ncurses's terminfo portability is its key advantage: it works correctly on VT100, xterm, screen, and exotic terminals without special-casing. Ratatui targets modern ANSI terminals and assumes 256-colour or truecolour support, trading portability for a simpler programming model and Rust memory safety. For new applications targeting Linux desktop terminals (kitty, foot, Ghostty, WezTerm), Ratatui's assumptions hold universally. [Source: ncurses documentation](https://invisible-island.net/ncurses/)

### Gap-Closing Roadmap

Ratatui's active development explicitly targets several gaps identified against its competitors.

**Retained-mode ergonomics (gap vs Textual)**: The core impedance mismatch with Textual is that Ratatui widgets are stateless — the application holds all state and re-renders everything on each frame, whereas Textual widgets own their state reactively. The `WidgetRef` stabilisation (targeting 0.30–0.31) and the upcoming `StatefulWidget` trait improvements reduce this friction by allowing widget state structs to carry update methods, approaching the retained-mode feel without abandoning the immediate-mode render contract. A formally proposed `Component` abstraction (issue tracker RFC, 2025) would allow pre-built widget trees to re-use state between frames without full application ownership — closing the most common complaint against the immediate-mode model.

**Async-native rendering (gap vs Textual's asyncio)**: Ratatui currently requires `ThreadProtocol` to prevent the Kitty image encoding step from blocking the `tokio` event loop. Native `async` encoding via `ratatui-image`'s planned `async-std`/`tokio` integration (§13, near-term) removes this workaround, bringing Ratatui's async model closer to Textual's where widget updates are naturally asynchronous.

**Inline (non-full-screen) mode (gap vs Rich)**: Rich's primary use case — printing formatted output to stdout in a scrolling terminal — has no equivalent in Ratatui, which always occupies the full alternate screen. An **inline viewport** mode has been under discussion since 2024: rather than entering the alternate screen with `EnterAlternateScreen`, the application renders into a fixed-height region at the cursor position, scrolling with normal terminal output. This would allow Ratatui widgets (progress bars, tables, spinners) to appear in CLI tool output alongside unstructured text, directly competing with Rich for that use case.

**Elm-style testability (gap vs Bubble Tea)**: Bubble Tea's pure-function `Update(msg) → (Model, Cmd)` model makes unit-testing trivial with no test backend required. Ratatui addresses this differently: the `TestBackend` + `Buffer::assert_eq!()` integration test approach verifies rendered cells without a real terminal. A planned `EventSimulator` utility (tracking in the ratatui-org repository) would let callers inject synthetic `Event::Key` and `Event::Mouse` events and assert on resulting buffer state, giving a comparable isolated-test story without requiring the Elm architectural constraint.

**terminfo portability (gap vs ncurses)**: Ratatui does not plan to add terminfo lookup — this would conflict with the goal of zero system-library dependencies. Instead, `crossterm`'s runtime detection expands: colour support queries via `COLORTERM` and `$TERM`, `DECSCUSR` cursor-shape fallbacks, and mouse protocol detection via `DA1` all improve compatibility with non-standard terminals without reaching for the full terminfo database.

**CSS/design-system layout (gap vs Textual)**: No CSS layout engine is planned for Ratatui core — this is a deliberate architectural decision. The Tachyonfx and Ratzilla ecosystems partially address the visual-design gap for effects and browser targets respectively, but the constraint solver is intended to remain the layout primitive.

### Summary Comparison

| Framework | Language | Model | Layout | Pixel graphics | Best for |
|---|---|---|---|---|---|
| **Ratatui** | Rust | Immediate-mode | Constraint solver | Kitty/Sixel/iTerm2 | High-performance TUI, Rust ecosystem |
| **Rich** | Python | Streaming output | None (inline) | Limited | CLI output formatting, logging |
| **Textual** | Python | Retained-mode | CSS-like | None | Complex Python TUI apps |
| **Bubble Tea** | Go | Elm Architecture | Lip Gloss strings | None | Go services, Charm ecosystem tools |
| **ncurses** | C | Retained WINDOW* | Manual placement | None | Portable C/C++ TUI, legacy terminals |

**When to choose Ratatui**: Rust codebase; need pixel graphics (Kitty/Sixel); performance-critical render loop (diff at cell level, no GC pauses); embedding in a terminal emulator or graphics-adjacent tool; need WebAssembly target (Ratzilla).

**When to choose Textual**: Python codebase; need CSS layout expressiveness; prefer retained-mode reactive state; development speed matters more than binary startup time.

**When to choose Bubble Tea**: Go codebase; value Elm Architecture's testability; building tools in the Charm ecosystem (`gum`, `soft-serve`, `mods`).

**When to choose ncurses**: C/C++ codebase; must run on legacy terminals; need the terminfo portability guarantee; maintaining existing ncurses application.

---

## 13. Roadmap

### Near-term (6–12 months)

**Ratatui workspace modularisation** is the dominant near-term architectural change. The monolithic `ratatui` crate is being split into feature-focused sub-crates (`ratatui-core`, `ratatui-widgets`, `ratatui-crossterm`, etc.) to reduce compile times and allow applications to depend only on the subset they use. A `WidgetRef` trait stabilisation and boxed-widget ergonomics improvement are targeted in the 0.30–0.31 release window.

**ratatui-image Kitty protocol unicode placeholder support** is in development — the mechanism (Ch43) that allows Kitty images to move with text during scroll is not yet implemented in ratatui-image; once landed, terminal multiplexer compatibility (tmux, zellij) will improve significantly. Async encoding via `tokio`/`async-std` integration is also planned, removing the need for `ThreadProtocol` in tokio-based applications.

**Tachyonfx shader DSL** — the `EffectDsl` compile-from-string API — is being expanded to support composition operators (`parallel`, `sequence`, `delay`) inline, enabling runtime-configurable effect pipelines loaded from configuration files.

**Mousefood display-driver coverage** is expanding: PRs for Waveshare 7-color e-paper and the GC9A01 circular LCD (common on round smartwatch-style displays) are in review. The crate maintainer intends to publish a `mousefood-drivers` meta-crate bundling common HAL configurations.

### Medium-term (1–3 years)

**Ratzilla WebGPU backend**: The current `WebGL2Backend` will likely gain a `WebGPUBackend` once `wgpu`'s WebGPU path stabilises in browsers. This would allow Tachyonfx compute-shader effects (not yet implemented but architecturally possible) to run via `wgpu` on both native Linux and in-browser WASM targets.

**Ratatui animation primitives**: Native frame-timing support — analogous to `wp_presentation` timestamps in a real terminal (Ch45) — is under discussion for the core crate. Currently applications must manage elapsed-time tracking externally; a built-in `AnimationState` associated with `StatefulWidget` would standardise this.

**Mousefood async embedded**: Integration with `embassy-rs` (the async Rust embedded runtime) would allow `mousefood`-backed Ratatui apps to run on `async` executor loops on STM32 and RP2040, replacing the current blocking draw cycle with cooperative multitasking.

### Long-term

- **Universal image widget tier**: A merged `ratatui-image` + `ratatui-video` widget covering Kitty animated sequences, piped frame streams (from `ffmpeg`), and WebCodecs-sourced frames in Ratzilla — bridging TUI image display and video playback into a single widget API.
- **Font atlas sharing with terminal emulators**: If `libghostty` (Ch44) stabilises its C ABI and exposes a glyph atlas handle, a Ratatui backend could share the terminal's existing GPU atlas rather than re-uploading fonts via escape sequences — eliminating the encoding layer entirely for embedded terminal applications.
- **Wasm Component Model**: Ratzilla components packaged as WASM components (under the W3C Component Model proposal) would allow Ratatui TUI panels to be embedded as web components in standard HTML pages, composited by the browser's normal DOM layout engine.

---

## 14. Integrations

**Chapter 43 — Terminal Pixel Protocols (Sixel, Kitty, iTerm2)**: `ratatui-image` is the primary application-layer consumer of the protocols described in that chapter. The `Picker::from_query_stdio()` detection stack and the `Resize::Fit` / `Resize::Crop` encoding paths map directly onto the escape-sequence semantics of Sixel DCS, Kitty APC, and iTerm2 OSC 1337 documented there. Readers implementing custom image widgets in Ratatui must understand the Kitty image ID / placement lifecycle to avoid ID exhaustion in long-running applications.

**Chapter 44 — Terminal GPU Rendering Architectures**: The pixel-protocol capability matrix established there (which terminals support Kitty, Sixel, Halfblocks, animated Kitty) is the foundation for `ratatui-image`'s `Picker` decision tree. WezTerm's wgpu backend (Ch44) is the primary target for ratatui-image Kitty streaming; Ghostty's libghostty VT parser (Ch44) is the most likely host for future shared-atlas Ratatui backends.

**Chapter 178 — PTY/TTY Kernel Layer**: The `SIGWINCH` → `TIOCGWINSZ` resize chain that Chapter 178 traces at the kernel level surfaces in Ratatui as `crossterm::event::Event::Resize` and in `Terminal::autoresize()`. The throughput limits of the N_TTY line discipline (4096-byte buffer, character-by-character processing) constrain how fast `ratatui-image` can push Sixel data through the PTY pipe — large Sixel payloads can stall the event loop until flushed.

**Chapter 45 — Terminal Compositor Integration**: When a Ratatui application runs inside kitty, WezTerm, or Ghostty and uses `ratatui-image` to display Kitty-protocol images, the image textures ultimately reside in the terminal's GPU texture pool (Ch44) and are composited by the terminal's Wayland submission pipeline (Ch45). The zero-copy path — `ratatui-image` using `Resize::Fit` with a Kitty terminal, which stores the image server-side and re-places it each frame — is the TUI equivalent of the DMA-BUF zero-copy scan-out discussed in Chapter 45.

**Chapter 98 — WebAssembly and WebGPU**: Ratzilla's `wasm32-unknown-unknown` target connects directly to the WASM runtime described in Chapter 98. The `WebGL2Backend` rendering path traverses the same ANGLE translation layer (Ch34) and Viz compositing pipeline (Ch36) as any other WebGL application in Chromium.

**Chapter 33 — Chromium GPU Architecture** and **Chapter 34 — ANGLE and WebGL**: Ratzilla's `CanvasBackend` and `WebGL2Backend` are Chromium renderer-process clients; their draw calls enter the `cc::LayerTreeHost` pipeline (Ch36), are submitted to the GPU process via Mojo, and execute via ANGLE against Mesa on Linux. Tachyonfx effects applied inside a Ratzilla `draw_web()` closure run as buffer transformations before the `WebGL2Backend` serialises cells to the GPU — the same sequence as native Ratatui, but with the GPU path going through the browser's GPU process rather than directly to the DRM render node.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
