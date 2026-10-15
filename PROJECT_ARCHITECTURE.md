# GridCraft — Project Architecture Overview

## 1. Executive Summary

GridCraft is an open-source, clean-room, Rust-native spreadsheet application engineered for Microsoft Excel parity and high-performance execution. It is architected around an **engine-first, command-driven design** where core spreadsheet data structures, recalculation graphs, and file parsers are strictly decoupled from the presentation layer.

GridCraft runs natively across macOS, Windows, Linux, and BSD via native executables, and in web browsers via WebAssembly (WASM).

---

## 2. Rust GUI Framework & Platform Shell

### 2.1 Framework Used
* **Primary GUI Toolkit:** [`egui`](https://github.com/emilk/egui) (version **0.36**), an immediate-mode GUI framework written in pure Rust.
* **Application Shell & Windowing Runner:** [`eframe`](https://github.com/emilk/egui/tree/master/crates/eframe) (version **0.36**), which wraps [`winit`](https://github.com/rust-windowing/winit) for window creation, event handling, and display scaling.
* **Rendering Hardware Backend:** [`wgpu`](https://wgpu.rs) (WebGPU abstraction layer) as the primary renderer across desktop platforms:
  * **macOS:** Metal backend.
  * **Windows:** DirectX 12 by default (with automated fallback to OpenGL/WARP).
  * **Linux / BSD:** Vulkan and OpenGL backends (with Wayland and X11 support).
  * **Web (WASM):** WebGPU with fallback to WebGL2 (Glow) via `trunk`.
* **Accessibility:** Integrated through [`accesskit`](https://accesskit.dev).

### 2.2 Framework Decoupling & Thin UI Principle
The UI crate (`crates/ui-egui`) sits at layer L6 and depends only on `crates/engine` and `crates/chart`. Layering is validated by `cargo xtask layers`. **Nothing below L6 depends on `egui`, `eframe`, `winit`, or `rfd`**, making the GUI frontend completely swappable (e.g., capable of being replaced with Slint, Iced, or native platform toolkits in the future).

---

## 3. Framework Modifications, Workarounds & Overrides

GridCraft does not simply use standard `egui` widgets. It overrides several aspects of `egui` and `eframe` to achieve desktop spreadsheet fidelity, crash resilience, and automation capabilities:

### 3.1 Pre-Validated Font Engine with Dynamic Fallback Chains
* **Crash Prevention with `skrifa`:** In standard `egui`, calling `Context::set_fonts` with a corrupt, truncated, or unreadable font file panics inside `epaint` during character map parsing. GridCraft pre-validates every system font file using Google's [`skrifa`](https://github.com/googlefonts/fontations) parser (`crates/ui-egui/src/theme.rs`, `font_parses()`) before passing it to `egui`. Any unparseable faces are dropped gracefully.
* **Background Font Discovery:** Rather than stalling startup to scan disk fonts, `crates/ui-egui/src/fonts.rs` runs a background thread to index installed system fonts. Once scanned, fonts are bound dynamically without freezing the frame loop.
* **CJK Han Face Order & Multi-Script Fallback:** Fallback glyph chains prioritize Han glyphs depending on the selected UI language (Japanese vs. Simplified Chinese vs. Korean) to preserve proper regional glyph variants, alongside dedicated fallbacks for Thai, Hebrew, and mathematical/UI symbols (`DejaVu Sans`, `Segoe UI Symbol`).
* **One-Frame Font Rebinding Gate:** When font definitions change, `SheetApp` skips painting for exactly one frame to allow `egui` to compile the new font atlas cleanly.

### 3.2 Viewport & Window Chrome Overrides
* **Custom Titlebar & Client-Area Extension:** On macOS, `viewport.with_fullsize_content_view(true).with_titlebar_shown(false).with_title_shown(false)` is enabled in `apps/gridcraft/src/main.rs`. GridCraft draws its own custom Excel-style title bar with a integrated Quick Access Toolbar (QAT), window title, and window control margins.
* **Native OS Menu Integration (`muda`):** On macOS, native AppKit menus are mounted using [`muda`](https://github.com/tauri-apps/muda), completely bypassing egui's in-window menus to provide a system menu bar (GridCraft, File, Edit, View, Insert, Format, Tools, Window, Help).
* **Apple Events Interception:** `apps/gridcraft/src/apple_events.rs` uses `fmv-macos-events` to intercept `kAEOpenDocuments` events (e.g., double-clicking a `.xlsx` in Finder or dragging onto the Dock) which `winit 0.30` does not natively dispatch.

### 3.3 Keyboard Zoom Hijack Suppression
* Standard `egui` intercepts `Ctrl/Cmd + Plus / Minus / 0` to scale the entire egui interface. This swallows standard spreadsheet shortcuts (such as inserting/deleting cells). GridCraft disables egui's default behavior via `ctx.options_mut(|o| o.zoom_with_keyboard = false)` and implements its own stepped zoom ladder (`WHEEL_ZOOM_LADDER`) with smooth exponential gliding.

### 3.4 GPU Backend Driver Safety on Windows
* In `apps/gridcraft/src/graphics.rs`, GridCraft probes DirectX 12 hardware adapters before `eframe` initializes. If a machine lacks DX12 GPU drivers, it falls back to OpenGL or WARP software rasterization, explicitly avoiding Vulkan by default to prevent crashes observed with legacy Intel UHD drivers (`igvk64.dll`).

### 3.5 Synthetic Input Injection for Agent Automation
* GridCraft overrides `eframe::App::raw_input_hook` to inject queued batches of synthetic `egui::Event` objects (`SheetApp.synthetic`). This enables end-to-end automated control via the localhost JSON-RPC control protocol (`apps/gridcraft/src/control_server.rs`) and headless UI testing (`egui_kittest`).

### 3.6 Cross-Platform Desktop Theme Synchronization
* Because Linux / X11 / Wayland often lack standard system theme notifications in `winit`, `crates/ui-egui/src/desktop_theme.rs` spawns a background thread that communicates with the XDG Desktop Portal via D-Bus (`zbus`) to reactively detect dark/light mode switches.

---

## 4. Application Core Segmentation & Layering

The codebase is organized into an 8-layer Cargo workspace enforced by `cargo xtask layers`. Lower layers cannot depend on higher layers.

```
+-------------------------------------------------------------------------+
| L7: Applications                                                        |
|     apps/gridcraft (Desktop) | apps/gridcraft-cli | apps/gridcraft-web  |
+-------------------------------------------------------------------------+
                                    |
+-------------------------------------------------------------------------+
| L6: Presentation & External Automation                                 |
|     crates/ui-egui (Excel-style frontend) | crates/mcp (Agent Protocol) |
+-------------------------------------------------------------------------+
                                    |
+-------------------------------------------------------------------------+
| L5: Application Engine & Facade                                         |
|     crates/engine (Session, Command Registry, Undo/Redo, IO, Clipboard) |
+-------------------------------------------------------------------------+
                                    |
+-------------------------------------------------------------------------+
| L3: Visualization                                                       |
|     crates/chart (Toolkit-agnostic Chart Layout & Rasterizer)           |
+-------------------------------------------------------------------------+
                                    |
+-------------------------------------------------------------------------+
| L2: Document Model, Recalculation & File Formats                       |
|     crates/model (COW Cell Store, Styles, Sheet, Workbook)             |
|     crates/calc  (Dependency Graph Recalculator, Dynamic Array Spill)   |
|     crates/xlsx  (Lenient XLSX, XLSB, CSV, TSV, ODS Reader & Writer)    |
|     crates/pdf   (PDF 1.7 Exporter & Print Generator)                   |
+-------------------------------------------------------------------------+
                                    |
+-------------------------------------------------------------------------+
| L1: Formulas & Standard Functions                                       |
|     crates/formula   (Pratt Parser, AST, Reference Adjustment)          |
|     crates/functions (400+ Pure Excel Functions, Dynamic Array Lifting) |
+-------------------------------------------------------------------------+
                                    |
+-------------------------------------------------------------------------+
| L0: Core Primitives & Number Formatting                                 |
|     crates/core   (CellRef, RangeRef, Value, CellError, Coercion)       |
|     crates/numfmt (Excel Number Format Codes, Sections, Conditions)     |
+-------------------------------------------------------------------------+
```

### Detailed Crate Responsibilities

| Layer | Crate | Primary Responsibilities |
|---|---|---|
| **L0** | `crates/core` | Cell coordinates (`CellRef`, `RangeRef`, A1 & R1C1 notation up to 1,048,576 rows × 16,384 columns), `Value` enum, `CellError`, Excel type coercion rules, 1900/1904 date serial conversions. |
| **L0** | `crates/numfmt` | Full Excel format code parsing and formatting (positive/negative/zero/text sections, bracketed conditions `[>100]`, colors `[Red]`, currency, fractions, scientific notation, elapsed time, `TEXT()` function). |
| **L1** | `crates/formula` | Tokenizer, Pratt recursive-descent parser, Abstract Syntax Tree (AST), formula printer (A1 and R1C1), and reference rewriting for copy/cut/fill and structural row/column insertions. |
| **L1** | `crates/functions` | Over 400 Excel-compatible pure functions (`SUM`, `XLOOKUP`, `INDEX`, `MATCH`, `LET`, `LAMBDA`, `FILTER`, `SORT`, etc.) with signature registries (`FnSpec`) and automatic array lifting. |
| **L2** | `crates/model` | Workbook and sheet state; interned style tables (fonts, borders, fills, alignments, number formats); conditional formatting rules; data validation; comments; shapes, drawings, and images; copy-on-write cell storage. |
| **L2** | `crates/calc` | Recalculation engine (`Calc`), directed dependency graph with range nodes, topological evaluation, dynamic array spilling (`#SPILL!`), and circular reference detection. |
| **L2** | `crates/xlsx` | High-performance OpenXML (XLSX), XLSB, CSV/TSV, and ODS import/export; DrawingML charts; operates on raw byte slices (no disk dependency, fully WASM-safe). |
| **L2** | `crates/pdf` | Minimalist PDF 1.7 engine generating vector drawing instructions and standard fonts for document printing and PDF export. |
| **L3** | `crates/chart` | Toolkit-independent chart layout and rendering primitives (bar, column, line, pie, scatter, area, sparklines). Generates abstract vector shapes without binding to any GUI framework. |
| **L5** | `crates/engine` | Central `Session` manager: open workbook lifecycle, active selection, undo/redo COW snapshots, system clipboard bridge, fill operations, sorting, filtering, PivotTable cache calculation, and the 300+ command catalog (`cmd/*`). |
| **L6** | `crates/ui-egui` | Excel-style desktop UI: virtualized spreadsheet grid, ribbon, formula bar, sheet tabs, backstage view, dialogs, task panes, and the control channel. |
| **L6** | `crates/mcp` | Model Context Protocol (MCP) server over stdio for AI agent automation (headless or bridged to a live window). |
| **L7** | `apps/*` | Binaries for desktop execution (`gridcraft`), headless command line / MCP tools (`gridcraft-cli`), and WebAssembly browser deployment (`gridcraft-web`). |

---

## 5. Concurrency, Multi-Threading & State Management

### 5.1 Recalculation Threading Model
* **Recalculation is Single-Threaded:** Formula evaluation in `crates/calc` currently runs on a **single thread**. Dependency graphs are resolved in topological order on the calling thread. Parallel recalculation of independent dependency subgraphs is planned on the roadmap (documented in `docs/hardware-parity.md`), but not yet implemented.
* **Targeted Background Worker Threads:** Multi-threading is intentionally applied to non-blocking I/O and operating system integration:
  1. **Font Discovery:** Background thread (`std::thread::spawn` in `crates/ui-egui/src/fonts.rs`) parses system fonts at launch.
  2. **Desktop Appearance Watcher:** Background thread in `crates/ui-egui/src/desktop_theme.rs` monitors OS dark/light mode via D-Bus on Linux.
  3. **Control Server:** Background TCP listener thread (`apps/gridcraft/src/control_server.rs`) accepts JSON-RPC agent commands and forwards them to the UI thread via `std::sync::mpsc::channel`.
  4. **Print Worker:** Desktop print jobs are spooled to worker threads via `craft-print` so print job submission does not stall UI rendering.

### 5.2 State Management & Sparse Copy-on-Write (COW) Storage
GridCraft achieves instant undo/redo and lightweight snapshots through structural sharing:

1. **64-Row Banded Cell Store (`crates/model/src/store.rs`):**
   * Cells are stored inside `CellStore` divided into horizontal bands of 64 rows:
     ```rust
     struct Band {
         rows: BTreeMap<u32, BTreeMap<u32, Cell>>,
     }
     pub struct CellStore {
         bands: BTreeMap<u32, Arc<Band>>,
         len: usize,
     }
     ```
   * Each 64-row band is wrapped in an `Arc`.
   * Taking a snapshot of the entire spreadsheet (for an undo point) is an $O(\text{bands})$ operation that only clones `Arc` pointers—no cell data is duplicated.
   * When a cell is modified, `Arc::make_mut` clones *only* that specific 64-row band, leaving all other bands untouched.

2. **Session Document State & History (`crates/engine/src/lib.rs`):**
   * Each open workbook is managed by a `DocState`:
     * `wb: Arc<Workbook>` (current state)
     * `saved: Arc<Workbook>` (state on disk; dirty checking is a single pointer comparison: `!Arc::ptr_eq(&self.wb, &self.saved)`)
     * `undo: Vec<HistoryEntry>` and `redo: Vec<HistoryEntry>` (snapshots containing `Arc<Workbook>` and `Selection`)
     * `calc: Calc` (dependency graph and cached cell values)

3. **Command-Driven Mutation:**
   * Every edit passes through `Session::execute(command_id, json_params)`.
   * Before running an undoable command, `Session` captures `before = (d.uid, d.wb.clone(), d.selection.clone())`.
   * If the workbook pointer changes after execution (`!Arc::ptr_eq(&d.wb, &wb)`), an entry is pushed to the undo stack (capped at 100 entries).

4. **Panic Isolation & Never-Crash Guarantee:**
   * In compliance with the repository's strict "Never crash" rule, `Session::execute` wraps command execution in:
     ```rust
     std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| (spec.run)(self, &params)))
     ```
   * If any command encounters an unhandled panic, `Session` intercepts it, restores `d.wb` and `d.selection` from the `before` snapshot, rebuilds `d.calc`, and returns `Err(EngineError::Internal)`. The application remains alive, uncorrupted, and reports the error to the user via a modal dialog or toast.

---

## 6. Component Creation & Rendering Pipeline

GridCraft follows an **immediate-mode presentation model**. There are no persistent widget objects for rows, columns, or cells. The entire interface is evaluated frame-by-frame:

```
SheetApp::logic(ctx)
   ├── Poll control channel (JSON-RPC)
   ├── Poll async file imports & printing status
   ├── Check theme & interface zoom changes
   └── Verify font initialization

SheetApp::ui(ui)
   ├── ribbon::title_bar(...)
   ├── backstage::show(...) [if File tab open]
   ├── ribbon::show(...)
   ├── widgets::notice_bar(...)
   ├── formula_bar::show(...)
   ├── tabs::status_bar(...)
   ├── tabs::sheet_tabs(...)
   ├── pivot_pane::show(...) & panes::show(...)
   └── egui::CentralPanel -> grid::show(...)
         ├── Viewport geometry & scroll calculation (Geo)
         ├── Zoom gliding & wheel handling
         └── Custom Virtualized Grid Painting (paint_quadrant)
```

### 6.1 Virtualized Grid Rendering (`crates/ui-egui/src/grid.rs`)
The grid handles up to 1,048,576 rows and 16,384 columns at a steady 60+ FPS:
* **No Widget Allocation for Cells:** GridCraft does not use `egui::Grid` or `egui_extras::Table`. It directly allocates a single `Rect` covering the spreadsheet area and draws onto `egui::Painter`.
* **Viewport Culling (`Geo`):**
  * Grid geometry calculates which row range (`r0..=r1`) and column range (`c0..=c1`) intersect the visible bounding box.
  * Only cells within this visible range are retrieved from the `CellStore`.
* **Four-Quadrant Freeze Panes:**
  * When freeze panes are active, the screen is split into four distinct quadrants (Top-Left frozen, Top frozen header, Left frozen column, and the main scrollable quadrant).
  * `paint_quadrant(...)` is invoked for each active pane, clipping render boundaries cleanly.
* **Direct Primitive Drawing:**
  * **Gridlines:** Drawn with `p.line_segment(...)`.
  * **Cell Backgrounds:** Painted with `p.rect_filled(...)`.
  * **Gradient Conditional Data Bars:** Constructed as custom vertex meshes using `egui::Mesh` and `p.add(mesh)`.
  * **Borders:** Rendered with `paint_border(...)` supporting thin, medium, thick, dashed, dotted, and double lines.
  * **Text & Galleys:** Formatted values are laid out with `p.layout_no_wrap(...)` and cached.
  * **Text Spilling / Overflow:** If text exceeds cell width, GridCraft inspects adjacent cells; if they are unmerged and empty, text is allowed to spill horizontally across column boundaries (matching Excel's behavior).
  * **Selection Marquee & Fill Handle:** Painted with a 2px accent stroke, corner fill square, and animated marching-ants dashed strokes for clipboard selections.

### 6.2 Floating Canvas Objects
Charts, pictures, shapes, slicers, and freehand ink strokes are rendered in a separate overlay pass (`paint_overlays`):
* Floating objects are anchored to sheet coordinates (`CellRef` + offset `dx, dy`).
* The renderer translates sheet coordinates through `Geo` into screen coordinates, supporting interactive dragging, border selection, and corner resize handles.

---

## 7. Styling, Themes & Formatting Features

### 7.1 Application Theme Tokens (`crates/ui-egui/src/theme.rs`)
All UI colors and metrics are centralized in `theme::Tokens`:
* **Palettes:** Distinct light (`Tokens::light()`) and dark (`Tokens::dark()`) palettes matching modern spreadsheet aesthetics (fluent green accents, neutral gray ribbons, tinted selection fills).
* **Application to egui:** `theme::apply(ctx, dark)` configures `egui::Visuals`:
  * `v.panel_fill` and `v.window_fill` set to token backgrounds.
  * `v.selection.bg_fill` and `v.selection.stroke` configured with soft accent washes.
  * `v.window_corner_radius` and `v.menu_corner_radius` customized for modern rounded corners.
  * Text styles mapped to platform UI fonts (`ui_font(13.0)`, `ui_bold(18.0)`, etc.).

### 7.2 Spreadsheet Cell Styling Engine (`crates/model/src/style.rs`)
Cell styles in GridCraft mirror OpenXML specifications:
* **Interned Styles:** Styles are deduplicated and interned into a workbook-level style table (`Workbook.styles: BTreeMap<u32, Style>`). Cells reference styles by a compact `u32` index.
* **Style Properties:**
  * **Font:** Family name, point size, bold, italic, underline (single/double/accounting), strikethrough, color (`Color::Rgb` or `Color::Theme`).
  * **Fill:** Solid color, pattern types (`PatternType::Solid`, `LightGray`, `MediumGray`, etc.), or gradient fills.
  * **Borders:** Independent top, bottom, left, right, and diagonal borders with 14 distinct line styles (`Thin`, `Medium`, `Dashed`, `Double`, `Hair`, `MediumDashed`, etc.) and custom colors.
  * **Alignment:** Horizontal (`General`, `Left`, `Center`, `Right`, `Fill`, `Justify`), vertical (`Top`, `Center`, `Bottom`), text wrap, and indent level.
  * **Number Format:** Numeric format ID referring to built-in or custom format strings.

### 7.3 Number Formatting Engine (`crates/numfmt`)
GridCraft includes a standalone parser and renderer for Excel number format strings:
* Fully supports standard Excel format strings: `#,##0.00;[Red](#,##0.00);"zero";@`.
* Supports section splitting (positive, negative, zero, text), conditional brackets (`[>1000]`), date/time tokens (`YYYY-MM-DD HH:MM:SS`), fraction approximations (`# ??/??`), scientific notation (`0.00E+00`), and currency prefixes/suffixes.

### 7.4 Conditional Formatting (`crates/engine/src/cf.rs`)
* Evaluates dynamic rules against cell ranges:
  * **Cell Value Rules:** Between, equal, greater than, less than.
  * **Expression / Formula Rules:** Custom formulas evaluating to boolean results.
  * **Visual Formats:** Color scales (2-color and 3-color heatmaps), Data bars (gradient or solid fills with min/max bounds), and Icon sets (directional arrows, traffic lights, ratings).

### 7.5 Localization & Internationalization (`i18n`)
* **Interface Localization:** Translated strings for 8 languages: English (`en`), Spanish (`es`), Simplified Chinese (`zh`), Japanese (`ja`), Korean (`ko`), Russian (`ru`), Portuguese (`pt`), and Czech (`cs`).
* **Formula Localization:** Localized function names and list separators (e.g., German/French `;` vs. English `,` based on formula locale) handled seamlessly via `crates/ui-egui/src/formula_locale.rs`.
