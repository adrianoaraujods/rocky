# Core architecture and library lock

ADR-0010 made Rocky a Rust desktop app and ADR-0011 chose GPUI with a native editor split into core and view. This ADR locks the engine underneath: crate boundaries, parsing, watching, persistence, threading, change delivery, math and performance budgets. It revises the *implementation* half of ADR-0004 (the index model stays). Decided by grilling in the "Core architecture and library lock" ticket, from the four Rust research reports.

## Decisions

- **Markdown (index):** **comrak** (start on 1.0 if it ships before the build begins) with a small post-pass for `## Heading [#id]`. Parsers only read. Every write is a text splice, never an AST re-serialize.
- **Frontmatter:** **saphyr** reads (YAML 1.2, spans), **yaml-edit** writes (lossless CST), behind a `FrontmatterEditor` trait. Fallback writer: span-splice on saphyr's `MarkedYaml`. Round-trip property tests gate both. Body bytes are never touched.
- **Index persistence:** keep one `.rocky/index.json` with **serde_json**, atomic swap, debounced saves (idle ≥ 2 s, on close, after a sweep). postcard is the drop-in upgrade if budgets demand it.
- **Watching:** **notify 8.2**, one non-recursive watch per non-ignored folder, no debouncers. Raw events are dirty-path hints coalesced over ~100–300 ms. `Rescan` (inotify overflow) triggers an immediate full sweep. The periodic sweep stays (default 15 min). "Polling mode" is a short sweep interval with the watcher off, never `PollWatcher`. Move to notify 9 once final.
- **Threading:** a synchronous core. std threads, crossbeam-channel, rayon, `ignore` for the parallel walk. No async runtime in `rocky-core`. The app adapts the sync API to GPUI's background tasks.
- **Core → UI (replaces ADR-0009's SSE):** `rocky-core` exposes a vault handle with an immutable index snapshot swapped atomically (`arc-swap`) and a subscription delivering **coarse change events** (what changed, by path/id). The UI re-reads from the snapshot.
- **Crate boundaries:** the editor core does not depend on `rocky-core`, and `rocky-core` does not depend on the editor. Only the `rocky` app depends on all. Rocky-specific behaviour enters the editor core through traits the app implements (`LinkResolver`, `EmbedProvider`, `MathRenderer`). Two Markdown parsers coexist (comrak for the index, the editor's own for decorations); shared fixture notes test their consistency.
- **Code highlighting:** **syntect + two-face** (pure-Rust `fancy-regex`), mapped to GPUI text runs.
- **Math:** KaTeX is not usable natively (HTML/CSS output needs a browser; the webview is ruled out). Typeset math is a **post-MVP milestone after Live Preview**. In the MVP `$…$` is styled source. The editor core exposes a widget seam from day one. The engine is **Typst** (native Typst math, **mitex** for LaTeX → Typst), rendering SVG, cached per formula, in its own crate behind `MathRenderer`, wired only by the app.
- **Math syntax:** `$…$` / `$$…$$` is **LaTeX by default** (Obsidian compatibility, ADR-0002). A **vault setting** (`.rocky/settings.json`) flips what `$…$` means to Typst. No new delimiter. The Markdown contract must record the setting.
- **Performance budgets** (targets, checked by a benchmark harness on a generated 100k-file vault, not CI gates at first): warm start window usable < 300 ms and index loaded < 500 ms in the background; cold rebuild < 10 s (warm disk cache, 8 cores); index memory < 250 MB RSS; file-open to first paint < 50 ms; keystroke-to-paint in Live Preview < 16 ms.

## Considered Options

- **pulldown-cmark** (3x faster): rejected, no callouts/comments/highlight, and a young wikilink extension with two 2026 DoS advisories.
- **serde-based YAML** (serde_norway etc.): rejected, drops comments and quoting, rewrites `1.10` to `1.1`.
- **SQLite/redb/postcard for the index:** rejected for now, a 0.2 s load doesn't earn them. Revisit for incremental persistence or search.
- **tokio in the core:** rejected, all work is local CPU and blocking file I/O.
- **Fine-grained deltas in change events:** rejected, coarse events plus a snapshot are simpler and lock-free for readers.
- **KaTeX via an embedded JS engine; math as MathML/pulldown-latex:** rejected, no native renderer for the output.
- **Typst-only math, or a new Typst delimiter (`$typ …$`):** rejected, it breaks Obsidian compatibility or invents a dialect.

## Consequences

- ADR-0004's index model stands. Its chokidar/Node implementation is replaced as above. The `usePolling` setting becomes the short-sweep mode.
- The "Math" part of the Markdown contract changes: LaTeX default, Typst via setting, typeset rendering after the MVP. The feature-cut ticket must reflect math as post-MVP.
- The Typst dependency (AGPL-compatible, Apache-2.0) arrives only with the math milestone, never in `rocky-core` or the editor core.
