# Rust Markdown editor feasibility

Research ticket: [Rust Markdown editor feasibility](https://github.com/adrianoaraujods/rocky/issues/30) (GH #30), part of [Rocky in Rust](https://github.com/adrianoaraujods/rocky/issues/28) (GH #28), feeding the UI framework and editor route choice (GH #33). As of 2026-11-15.

The question: how feasible is a Rocky-agnostic, publishable **Rust-native** crate that reproduces the behaviour of the driver's CodeMirror 6 package `codemirror-obsidian-md` (v0.3.0, MIT), and how does it compare with running that package unchanged in a webview? This reopens [ADR-0007](../adr/0007-editor-stack-codemirror-source-first-with-live-preview.md).

Sources: the reference package read locally (`~/Projects/personal/obsidian-md/packages/codemirror-obsidian-md`, cited as `ref:<path>`), crates.io API metadata, docs.rs, upstream repositories and changelogs. Version and date figures come from the crates.io API on 2026-11-15. Cost figures are **estimates**, calibrated against the line counts of the reference, of CodeMirror 6 itself and of existing Rust editors. They are not measurements.

## Verdict in one paragraph

A Rust-native editor is **feasible but expensive**, and nearly all of the cost is in Live Preview's view layer, not in the editing behaviours. The reference is small (2,754 lines of TypeScript in `src/editor`) because it rests on about 30k lines of CodeMirror 6 (`@codemirror/view`, `state`, `commands`, `autocomplete`). A Rust port has to rebuild that engine first. No Rust framework ships an editor widget that covers the spec: the stock widgets in iced, egui and parley are single-cursor, and none can hide source ranges or embed widgets in the text flow. So every framework route means **writing the editor view ourselves**. Prior art shows it can be done. [writ](https://github.com/wilfreddenton/writ) (ropey + tree-sitter-md + parley + vello, about 30k lines of Rust) already does Obsidian-style hide-markers-unless-cursor Live Preview, including callouts and checkboxes. Byte-exact saves are easy in Rust: a rope *is* the file, with the same discipline the reference already uses. **Recommendation:** go Rust-native, with a framework-agnostic headless core crate plus one view crate, and gate the work: build Source mode first, then a time-boxed Live Preview spike. **Fallback:** the hybrid, the existing CM6 package in a Tauri webview, which reaches parity at the lowest cost but breaks "Rust everywhere".

## 1. Reference inventory

### What the package is, structurally

- **Size:** `src/editor` has 2,754 lines of TS plus 313 lines of CSS. The rest of the package's roughly 5.4k source lines are storage, save-flow, lock, manager and new-note, which Rocky's own vault layer replaces and which are out of scope for an editor crate. The parser dependency `remark-obsidian-md/src` adds 3,554 lines (wikilinks, embeds, callouts, highlights, comments, block IDs, inline footnotes, resolver, frontmatter).
- **Tests (the behaviour spec):** about 228 `it`/`test` cases in `tests/editor/*` under jsdom. The ones that matter for the editor: inline 33, editing-shortcuts 25, list-indent 21, list-continuation 19, auto-pair 16, picker 15, markdown 12, callout-embed 12, selection-wrap 12, pipeline 10, multi-cursor 8, plus a byte-identity corpus test (`ref:tests/editor/corpus.test.ts`).
- **Parsing is *not* Lezer.** The core (`ref:src/editor/state.ts`, `parse.ts`) runs **remark (mdast) over the whole Note**, debounced to idle (`DEFAULT_IDLE_MS = 250`). The resulting `Construct { type, from, to, node }` list is **mapped through edits** (`tr.changes.mapPos`) between re-parses. Decorations are rebuilt when a parse lands or the selection moves, and are only position-mapped while the user types. Frontmatter is cut off before parsing. `%%…%%` block comments and `^id` block IDs are recovered by post-passes over the text (`withBlockComments`, `blockIdParser`). Tables, code, math, footnote definitions and YAML stay plain source (`PLAIN_SOURCE_TYPES`).
  - Consequence: the port **does not need an incremental parser**. It needs an mdast-shaped parse with byte offsets, run off the input thread, plus position mapping. That is a much easier requirement.
- **Providers** are pluggable (`ConstructProvider { name, types, decorate(construct, {state, revealed}) }`), registered through a CM `Facet`. `revealed` means "the selection touches `[from, to]`" (`ref:src/editor/cursor.ts`).

### Live Preview providers and the CM6 primitives each one uses

| Provider (file) | Constructs | What it does | CM6 primitives |
|---|---|---|---|
| `wikiLinkProvider` (`providers/inline.ts`) | `wikiLink` | Hides `[[target\|` and `]]`, marks the label as `<a class=internal-link>`. Raw-styled mark when revealed. | `Decoration.replace({})` (hide), `Decoration.mark({tagName, attributes, class})` |
| `highlightProvider` | `obsidianHighlight` | Hides `==`, `<mark data-highlight>` with an emoji colour prefix. | replace, mark |
| `commentProvider` | `obsidianComment` | Hides the whole `%%…%%`. | replace |
| `footnoteProvider` | `obsidianFootnote` | Hides `^[`/`]`, `<sup>` content. | replace, mark |
| `blockIdProvider` | `blockId` | Hides `^id`. | replace |
| `markdownProvider` (`providers/markdown.ts`) | `heading` | Line class per depth, hides `# ` marker. | `Decoration.line`, replace, mark |
| | `strong`, `emphasis`, `inlineCode` | Styles content, hides markers. | mark, replace |
| | `listItem` | Line class. Bullet **widget** replaces `-`. Task **checkbox widget** replaces `- [ ] ` and toggles the status char on mousedown (`view.posAtDOM`, `dispatch`). Reveal uses only the first line. | `WidgetType` + `Decoration.replace({widget})`, `ignoreEvent`, `posAtDOM` |
| | `blockquote` | Line class per line, hides `>` markers. Nested quotes defer to the outermost. | line, replace |
| Callout (`providers/callout-embed.ts`) | `blockquote` whose first line is `> [!type]±` | Per-line callout classes with a `--callout-color` style. **Title widget** replaces the whole first line. On click it places the cursor inside the typed title via `caretPositionFromPoint`/`caretRangeFromPoint`. Hides `>` prefixes. | line (attributes), replace-with-widget, DOM hit-testing, `dispatch({selection})` |
| Embed (`providers/callout-embed.ts`) | `embed` | Image **widget** (`resolveAsset`, `|WxH` sizing) or an unresolved/note **chip** widget. | replace-with-widget |

Shared engine requirements: a **decoration range set** (mark, line, replace, widget) that the cursor treats as atomic, with replaced ranges skipped by motion. Also **viewport virtualization**, **line wrapping** (`EditorView.lineWrapping`), and **coordinate↔position hit-testing** across hidden ranges and widgets.

### Editing behaviours (`src/editor/editing/*`, `editing-shortcuts.ts`, `multi-cursor.ts`, `picker.ts`)

| Behaviour | Keys / trigger | CM6 primitives |
|---|---|---|
| Auto-pair (`auto-pair.ts`, `pairs.ts`) | Typed openers (brackets, quotes, backtick), `[[` → `[[]]`, step over the closer, Backspace deletes the pair. | `EditorView.inputHandler`, `changeByRange`, `keymap` |
| Selection wrap (`selection-wrap.ts`, `typing.ts`) | Typed wrap char over a selection. Each range decides independently, and each wrap is its own undo step. | inputHandler, `changeByRange`, `isolateHistory` annotation |
| Emphasis toggle (`emphasis.ts`) | `Mod-b` → `**`, `Mod-i` → `_`, detects markers around or at the edges, `***` handling. | `Command`, `changeByRange` |
| Link (`link.ts`) | `Mod-k` → `[text]()` / `[]()`. | Command, `EditorSelection` |
| List continuation (`list-continuation.ts`, `list-line.ts`) | Enter continues a bullet, number, task, quote or callout line. Backspace on an empty marker removes it. Tab/Shift-Tab indent. CRLF-aware, skips fenced code. | `keymap` + `Prec`, per-range decisions |
| List indent (`list-indent.ts`) | `Mod-]`/`Mod-[`, Tab/Shift-Tab, renumbering of ordered runs, minimal diff edits. | Command, `ChangeSpec` |
| Task toggle (`task-toggle.ts`) | `Mod-Enter` per cursor. | Command |
| Paste URL (`paste-url.ts`) | Pasting an http(s) URL over a selection → `[text](url)`. | `EditorView.domEventHandlers({paste})` |
| Composition guard (`composition.ts`) | Every behaviour is a no-op while the IME composes. Bound keys are claimed so stock bindings cannot fire mid-composition. | `view.composing`, `view.compositionStarted` |
| Multi-cursor (`multi-cursor.ts`) | Alt-click adds a cursor, Shift-Alt-drag or middle-drag makes a rectangle, `Mod-Alt-Up/Down` adds one above or below, Escape collapses. | `allowMultipleSelections`, `drawSelection`, `rectangularSelection`, `crosshairCursor`, `addCursorAbove/Below` |
| CRLF Enter (`state.ts`) | Enter inserts `\r\n` when every break in the Note is CRLF. The doc's `lineSeparator` is pinned to `\n`, so a `\r` stays an ordinary character. | `EditorState.lineSeparator`, keymap |
| `[[` picker (`picker.ts`) | Ranked Note candidates (exact, prefix, word start, substring, alias, path, subsequence), inserts a closed wikilink. | `@codemirror/autocomplete` |
| History (`mount.ts`) | Undo/redo with grouping. | `@codemirror/commands` `history` |

Takeaway: the editing behaviours are **pure functions over (text, selection ranges) → change set + new selection**. They port 1:1 to a headless Rust core and are cheap. What makes the reference cheap in TS is the **view** layer that CM6 provides.

## 2. Rust building blocks

| Need | Candidates (latest version, last update) | Assessment |
|---|---|---|
| Text buffer | **ropey** 1.6.1 (2025-08, 15M downloads), **crop** 0.4.3 (2025-04), xi-rope 0.3.0 (2019, dormant) | ropey: valid UTF-8 only, "all Unicode line endings are correctly tracked including CRLF", byte/char/line indexing ([README](https://github.com/cessen/ropey)). crop is a B-tree rope with cheap clones for snapshots ([README](https://github.com/nomad/crop)). Either works; ropey is the ecosystem default (Helix, writ, gpui-component). xi-rope is abandoned. |
| Shaping + layout | **parley** 0.13.0 (2026-11-12) + swash 0.2.10; **cosmic-text** 0.19.0 (2026-04) | parley has rich styled runs, **`InlineBox`** for inline widgets, and bidi. A `Layout` must be rebuilt when text or styles change ([docs](https://docs.rs/parley/latest/parley/)). Its `PlainEditor` is "a single style applied to the entire text", single selection, with IME compose/preedit support ([docs](https://docs.rs/parley/latest/parley/editing/struct.PlainEditor.html)): a useful reference but not our editor. cosmic-text has `Editor`/`SyntaxEditor`/`ViEditor`, a single cursor, and per-line `LineEnding` (Lf/CrLf/Cr/LfCr/None) ([docs](https://docs.rs/cosmic-text/latest/cosmic_text/), [source](https://github.com/pop-os/cosmic-text/blob/main/src/line_ending.rs)). |
| Markdown parse | **markdown-rs** (`markdown` 1.0.0), **pulldown-cmark** 0.13.6, **comrak** 0.59.0, **tree-sitter-md** 0.5.4 | **markdown-rs** is wooorm's micromark port, so it is the closest match to the reference's remark: an **mdast with byte offsets**, CommonMark-compliant, GFM, math, frontmatter. It has **no user extension API** ("not a goal") ([README](https://github.com/wooorm/markdown-rs)), so Obsidian syntax needs post-passes, which the reference already does for comments and block IDs. **pulldown-cmark** has `into_offset_iter` (event + source range) and native `ENABLE_WIKILINKS` ([docs](https://docs.rs/pulldown-cmark/latest/pulldown_cmark/struct.Parser.html)). **tree-sitter-md** is incremental (`MarkdownParser::parse_with_options(old_tree)`, `MarkdownTree::edit`), split into block and inline grammars. Wiki links and tags are **compile-time grammar options, off in the published crate**. The maintainer warns the output "contains many inaccuracies" and it is "not recommended where correctness matters", being meant for highlighting ([README](https://github.com/tree-sitter-grammars/tree-sitter-markdown)). |
| Undo/history | `undo` 0.52 (2025-03); Helix `helix-core/src/history.rs` + `transaction.rs` (MPL-2.0); cosmic-text `Change` | The reference needs grouping plus forced isolation, and history entries that carry selections. Helix's `Transaction` (ChangeSet + Selection) and `History` are the closest design match ([helix-core/src](https://github.com/helix-editor/helix/tree/master/helix-core/src)). Own implementation: about 300–500 lines. |
| IME | winit `Ime::{Enabled, Preedit(text, cursor), Commit, Disabled}` (0.30.13) ([docs](https://docs.rs/winit/latest/winit/event/enum.Ime.html)); GPUI's platform input handler; egui `ImeEvent::{Enabled, Preedit, Commit, Disabled}`; iced 0.14 "Input method support" (#2777) ([CHANGELOG](https://github.com/iced-rs/iced/blob/master/CHANGELOG.md)) | Linux IME works through these toolkits but is historically fragile. Zed needed `text_input_v3` on Wayland (#11712), a fcitx5 fix on X11 (#14508), XIM preedit fixes (#17373), Wayland scaled-display positioning (#37600) and a duplicate-input fix (#46560) ([umbrella issue #9207](https://github.com/zed-industries/zed/issues/9207)). fcitx's FAQ notes poor compositor support for Wayland IM protocols ([fcitx FAQ](https://fcitx-im.org/wiki/FAQ)). Budget real time for this; GPUI has paid most of that debt already. |

## 3. Prior art

| Project | Stack / licence | Relevance |
|---|---|---|
| **Zed editor** (GPUI) | `editor`, `text`, `rope`, `multi_buffer`, `markdown` crates are **GPL-3.0-or-later**. `gpui` and `sum_tree` are **Apache-2.0** (each crate's `Cargo.toml`). | The best Rust editor architecture. Its `DisplayMap` stacks `InlayMap → FoldMap → TabMap → WrapMap → BlockMap`, with `FoldPlaceholder`/`Crease` render callbacks and `BlockMap` custom blocks ([display_map.rs](https://github.com/zed-industries/zed/blob/main/crates/editor/src/display_map.rs)). That is a near one-to-one analogue of CM6 replace/widget/block decorations. Being GPL, it can be **studied, not copied**, into an MIT crate. `gpui` itself is "pre-1.0 … there will often be breaking changes" ([README](https://github.com/zed-industries/zed/tree/main/crates/gpui)). Linux X11 + Wayland. |
| **gpui-component** (Longbridge), crate `gpui-base` 0.9.1 (2026-11-09), 20.7k stars | Apache-2.0 | Ships a GPUI code editor (ropey, tree-sitter highlight, LSP, "stable performance at 200K lines") with a `DisplayMap = WrapMap → FoldMap` stack ([README](https://github.com/longbridge/gpui-component), [display_map README](https://github.com/longbridge/gpui-component/blob/main/crates/base/src/input/editor/display_map/README.md)). No inline replace or widget decorations, and multi-cursor is not documented. A permissively licensed base to learn from or fork for Source mode. |
| **Helix** (core only) | MPL-2.0 | ropey `Transaction`/`ChangeSet`/`Selection` (multi-range), `History`, `auto_pairs.rs`, `surround.rs`, `text_annotations.rs` (virtual text). The model for the headless core ([helix-core/src](https://github.com/helix-editor/helix/tree/master/helix-core/src)). |
| **Lapce / Floem** | MIT | Floem has an `editor-core` but "is still maturing"; the last crates.io release is 0.2.0 (2024-11) ([README](https://github.com/lapce/floem)). A weak candidate. |
| **iced `text_editor`** (0.14) | MIT | A `highlight_with(Highlighter)`, `key_binding`, `wrapping`, IME and `LineEnding` widget, but a **single cursor/selection** ([source](https://github.com/iced-rs/iced/blob/master/widget/src/text_editor.rs)). It cannot hide ranges or host widgets. |
| **egui `TextEdit`** (0.37) | MIT/Apache | `layouter` hook for highlighting, which "runs at least once per frame, cache it" ([docs](https://docs.rs/egui/latest/egui/widgets/text_edit/struct.TextEdit.html)). Single cursor, no inline widgets. |
| **Freya** 0.5.2 | MIT | Skia renderer, its own reactive core (no longer Dioxus since 0.4), a "code-editor" feature with rope + tree-sitter ([README](https://github.com/marc2332/freya)). Small user base (about 50k downloads). |
| **writ** 0.19.3 (2026-09) | winit + wgpu + **Vello + Parley + ropey + tree-sitter-md**. No licence file on GitHub, so **read, don't copy**. | **The closest working proof** of Rust Live Preview: markers hidden unless the cursor is on them, callouts with icons and folding, checkboxes, images, tables, a "per-line map between displayed text and buffer offsets" for hit-testing, viewport virtualization ([README](https://github.com/wilfreddenton/writ), [docs.rs](https://docs.rs/crate/writ/latest)). About 1.18 MB of Rust (roughly 30k lines). Gaps: setext headings unsupported; multi-cursor and IME are not mentioned. |
| **Velotype** 0.5 (GPUI) | Apache-2.0 | A WYSIWYG **block tree** that "is serialized back into canonical Markdown" on save, with IME listed as incomplete ([README](https://github.com/manyougz/velotype), [announcement](https://users.rust-lang.org/t/velotype-a-native-rust-markdown-editor-built-for-wysiwyg-writing/140046)). This is the **anti-pattern for the buffer-is-the-file rule**. About 2.8 MB of Rust. |
| **Ferrite** 0.3.1 (egui 0.34, comrak) | MIT | Separate Raw / Rendered / Split modes, not inline Live Preview. Known GNOME-Wayland keyboard issue ([README](https://github.com/OlaProeis/Ferrite)). |

Calibration: CodeMirror's `view` + `state` + `commands` + `autocomplete` add up to about 1.09 MB of TypeScript (GitHub language stats), roughly 30k lines. writ, at about 30k lines of Rust, is the same order of magnitude. **A Live Preview editor costs about one CodeMirror.**

## 4. Cost estimate

### Shape common to every Rust route

Split it the way the reference splits CM6 from its extensions:

1. **`*-core` (headless, framework-free, publishable):** rope buffer, `ChangeSet` with position mapping, multi-range `Selection`, `Transaction`, history (group/isolate), Markdown parse to `Construct`s (markdown-rs + Obsidian post-passes, or a port of `remark-obsidian-md`), construct mapping between debounced parses, a provider trait producing an abstract **decoration set** (mark / line / replace / replace-with-widget-id), and **every editing behaviour as a pure command**. All of the reference's tests port here as plain `cargo test` tables. This is the ideal surface for AI agents.
2. **`*-view-<framework>`:** layout per visible line (parley or the framework's text system), a display map that applies the decoration set (hidden ranges, inline boxes for widgets, line styles), hit-testing, caret and multi-selection painting, IME preedit, clipboard, keymaps, scrolling and virtualization.

Estimates are in lines of Rust including tests, then in relative effort. The core is the same for all frameworks.

| Piece | Lines (est.) | Notes |
|---|---|---|
| Core: buffer, changes, selection, history | 2–3k | Helix-shaped. |
| Core: parser + Obsidian post-passes + constructs | 2–4k | Low end with markdown-rs plus post-passes. High end if `remark-obsidian-md` (3.5k TS) is ported as a parser extension. |
| Core: editing behaviours + picker ranking | 2–3k | A direct port of about 1.3k TS plus tests. |
| **Core total** | **6–10k** | Framework-agnostic, highly testable. |

### (a) Source mode only (highlighting + editing behaviours + multi-cursor + IME)

| Framework | View work | Est. view lines | Overall |
|---|---|---|---|
| **GPUI** | Own editor element on GPUI's text shaping and input handler (IME already proven in Zed). Learn from gpui-component's editor (Apache-2.0); Zed's is GPL, study only. Multi-cursor painting is ours. | 4–6k | **Medium.** Best-supported Linux text input. API churn risk. |
| **iced** | Stock `text_editor` is single-cursor, so this means a custom `advanced::Widget` on cosmic-text or parley. | 5–7k | Medium-high |
| **egui** | Stock `TextEdit` is single-cursor and re-lays out every frame, so this means a custom widget with galley caching. | 5–7k | Medium-high. The immediate-mode model fights large documents. |
| **Freya / Floem** | Own widget. Small or stale ecosystems. | 5–7k | High (ecosystem risk) |
| **Custom winit + parley + vello** (writ-style) | Everything, including the window, scrolling and accessibility. | 7–10k | High, but full control. |

Total for (a): about **10–16k lines**. With AI agents writing against ported tests, that is weeks to a couple of months, not quarters.

### (b) Full Live Preview (inline markup compiled while editable, block widgets that snap back to source)

On top of (a): a display map with hidden ranges (cursor motion skips them atomically, reveal on touch), inline widgets (bullet, checkbox, chip, image) as parley `InlineBox`es or framework elements, a callout title widget that replaces a whole line with click-to-offset, line-level styling (heading sizes, quote and callout bars), widget hit-testing that dispatches doc changes (checkbox toggle), a relayout on every selection move (reveal changes layout), and IME preedit inside revealed constructs.

| Framework | Extra view lines (est.) | Notes |
|---|---|---|
| GPUI | 6–10k | Zed's DisplayMap is the blueprint (study only). GPUI elements can be embedded as blocks. |
| Custom parley/vello | 6–10k | writ proves the per-line display↔buffer map approach on exactly this stack. |
| iced / egui | 8–12k | Fighting stock text widgets. Inline widgets need custom layout and paint. |

Total for (b): about **20–30k lines**, consistent with writ (about 30k). This is the expensive, high-risk part. The hardest bugs are hit-testing and caret motion across hidden or replaced ranges under soft wrap, and IME composition interacting with reveal/relayout.

### (c) Hybrid: CodeMirror 6 (the existing package) in a Tauri/Dioxus webview

- **Work:** a Tauri 2 (2.12.5) app embedding the package. Dioxus desktop also renders through a webview, with WGPU/Blitz "experimental" ([README](https://github.com/DioxusLabs/dioxus)). Glue: Rust reads bytes → IPC → `createEditor({doc})` → `getText()` → IPC → write bytes. Wire the host resolver/picker candidates to the Rust vault index. Estimate: **1–3k lines of glue**. Every behaviour and all ~228 tests come for free, and Live Preview is already done.
- **Costs and risks:** the editor is TypeScript and the remark pipeline runs in JS, which breaks "Rust everywhere". Two toolchains. On Linux the webview is **WebKitGTK**, whose version depends on the distribution ([Tauri webview versions](https://v2.tauri.app/reference/webview-versions/)), so test CM6 on WebKitGTK. IME goes through GTK/WebKit, which is mature. Native look and performance are those of a webview. "Behaves like Obsidian" is literally true, because Obsidian is CM6 in Electron.

### Summary

| Route | Rust lines (est.) | Fidelity to the spec at completion | Risk |
|---|---|---|---|
| (a) Source only, native | 10–16k | Source mode plus all behaviours. **No Live Preview** (ADR-0007 made Live Preview the default mode). | Medium |
| (b) Live Preview, native | 20–30k | Full | High: view layer + IME |
| (c) Hybrid CM6 in a webview | 1–3k glue | Full on day one | Low technical risk; fails the "Rust everywhere" preference |

## 5. Byte-exact saves

The reference's rule (`ref:src/editor/state.ts`, `mount.ts`, `corpus.test.ts`): the document *is* the Note. `lineSeparator` is pinned to `\n`, so `\r` is an ordinary character and CRLF, lone CR and mixed endings round-trip. Enter inserts `\r\n` only for an all-CRLF Note. `getText()` is the save payload. A corpus test asserts byte identity for fixture notes plus CRLF, no trailing newline, odd or unclosed frontmatter, HTML, mixed endings and lone CR.

| Route | Guarantee | Caveats |
|---|---|---|
| Native, **ropey/crop core** | **Yes, by construction.** The rope holds the exact UTF-8. `\r\n` and lone `\r` are characters, so `rope.to_string()` equals the file. Port the corpus test as is. | **Non-UTF-8 files cannot enter a rope** ([ropey README](https://github.com/cessen/ropey)). Detect them on open and refuse or open read-only; never lossy-decode. A BOM survives as U+FEFF as long as nothing strips it. Line-ending-aware helpers must not "normalise" (mirror `contentEnd`/`stripCr`). |
| Native, framework stock widgets | **Not as given.** cosmic-text and iced store **lines + a per-line `LineEnding`**, which is reconstructible in principle but not "the buffer is the file". Prove it with the corpus test, or avoid it. egui `TextEdit` and parley `PlainEditor` hold a `String`, which is exact. | A further reason to own the buffer in the core crate and let the view only *render* it. |
| Block-model WYSIWYG (Velotype-style) | **No.** It serializes to "canonical Markdown". | Excluded, as ADR-0007 already concluded for ProseMirror/Milkdown. |
| Hybrid CM6 in a webview | **Yes, already proven** by the reference's corpus test. | Read bytes in Rust with strict `String::from_utf8` and pass a string over IPC. Do not use JS `TextDecoder` (it strips a BOM by default) or any lossy decode. Same non-UTF-8 rule. |

## Recommendation for the route choice (GH #33)

**Feasibility verdict:** feasible. Source mode is a moderate project. Full Live Preview is roughly a CodeMirror-sized project (20–30k lines), proven possible by writ, with the risk concentrated in the view layer (hit-testing across hidden ranges and widgets, IME).

**Recommended route: Rust-native, staged and gated.**

1. Build a publishable, framework-free **`obsidian-md-editor-core`-style crate** (name to be decided): ropey, Helix-shaped transactions and history, markdown-rs + Obsidian post-passes, constructs mapped between idle re-parses (the reference's own design, which needs no incremental parser), a provider trait emitting an abstract decoration set, and every editing command. Port the reference's tests and the byte-identity corpus test first. This is cheap, agent-friendly and useful whatever happens to the view.
2. Build **Source mode** in one view crate. On Linux, **GPUI** has the strongest editor-grade text input (IME paid for by Zed) and an Apache-2.0 editor to learn from (gpui-component). The cost is pre-1.0 churn; GH #33 owns the framework pick. A custom **parley/vello** view is the framework-neutral alternative with writ as precedent.
3. Run a **time-boxed Live Preview spike** against the hardest cases: hidden markers with caret motion, checkbox widget toggling, callout title line widget click-to-offset, image embed, and IME preedit inside a revealed construct, all under soft wrap. Go or no-go on whether the ported `inline`/`markdown`/`callout-embed` tests pass inside the box.

**Fallback: the hybrid.** Embed the existing `codemirror-obsidian-md` package in a Tauri 2 webview (WebKitGTK on Linux), keeping the vault, index and IO in Rust and the editor in TS. Take it if the spike fails, or if Live Preview must ship before the native view is ready. The core crate's tests keep the two editors behaviourally aligned. The Rust app can also ship the hybrid first and swap in the native editor later behind the same bytes-in/bytes-out seam.

## Sources

- Reference package (local): `~/Projects/personal/obsidian-md/packages/codemirror-obsidian-md` (`package.json` v0.3.0; `src/editor/**`; `tests/editor/**`), `~/Projects/personal/obsidian-md/packages/remark-obsidian-md/src`.
- crates.io API, `https://crates.io/api/v1/crates/<name>`, for ropey, crop, xi-rope, cosmic-text, parley, swash, tree-sitter, tree-sitter-md, pulldown-cmark, comrak, markdown, iced, egui, freya, floem, gpui, dioxus, tauri, wry, undo, xilem, vizia, slint (queried 2026-11-15).
- ropey: <https://github.com/cessen/ropey>; crop: <https://github.com/nomad/crop>
- parley: <https://docs.rs/parley/latest/parley/>, <https://docs.rs/parley/latest/parley/editing/struct.PlainEditor.html>, <https://github.com/linebender/parley/blob/main/CHANGELOG.md>
- cosmic-text: <https://docs.rs/cosmic-text/latest/cosmic_text/>, <https://github.com/pop-os/cosmic-text/blob/main/src/line_ending.rs>
- tree-sitter-md: <https://github.com/tree-sitter-grammars/tree-sitter-markdown> (README, `Cargo.toml`, `bindings/rust/parser.rs`, `tree-sitter-markdown-inline/grammar.js`)
- markdown-rs: <https://github.com/wooorm/markdown-rs>
- pulldown-cmark: <https://docs.rs/pulldown-cmark/latest/pulldown_cmark/struct.Parser.html>
- winit IME: <https://docs.rs/winit/latest/winit/event/enum.Ime.html>; egui IME: <https://docs.rs/egui/latest/egui/enum.ImeEvent.html>
- iced: <https://github.com/iced-rs/iced/blob/master/widget/src/text_editor.rs>, <https://github.com/iced-rs/iced/blob/master/CHANGELOG.md>
- egui TextEdit: <https://docs.rs/egui/latest/egui/widgets/text_edit/struct.TextEdit.html>
- Zed: <https://github.com/zed-industries/zed/blob/main/crates/editor/src/display_map.rs>, <https://github.com/zed-industries/zed/tree/main/crates/gpui>, per-crate `Cargo.toml` licences; IME history <https://github.com/zed-industries/zed/issues/9207> and PRs #11712, #14508, #17373, #37600, #46560
- fcitx FAQ: <https://fcitx-im.org/wiki/FAQ>
- gpui-component: <https://github.com/longbridge/gpui-component>, `crates/base/src/input/editor/display_map/README.md`
- Helix core: <https://github.com/helix-editor/helix/tree/master/helix-core/src>
- Floem: <https://github.com/lapce/floem>; Freya: <https://github.com/marc2332/freya>; Dioxus: <https://github.com/DioxusLabs/dioxus>
- writ: <https://github.com/wilfreddenton/writ>, <https://docs.rs/crate/writ/latest>
- Velotype: <https://github.com/manyougz/velotype>, <https://users.rust-lang.org/t/velotype-a-native-rust-markdown-editor-built-for-wysiwyg-writing/140046>
- Ferrite: <https://github.com/OlaProeis/Ferrite>
- Tauri webview versions: <https://v2.tauri.app/reference/webview-versions/>
- CodeMirror sizes: GitHub language stats for `codemirror/view`, `state`, `commands`, `autocomplete`.
