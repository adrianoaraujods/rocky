# Rust core crates landscape

Research ticket: [Rust core crates landscape](https://github.com/adrianoaraujods/rocky/issues/31) (GH #31), part of the map [Rocky in Rust](https://github.com/adrianoaraujods/rocky/issues/28) (GH #28). As of 2026-11-15, this picks the crates `rocky-core` (no UI) should build on, measured against the Markdown contract (`docs/markdown-contract.md`), identity/frontmatter (ADR-0003), the derived index (ADR-0004: watcher as accelerator, periodic snapshot-and-diff sweep, ~100k-file vault) and trash-based delete (issue #23; the brief cites ADR-0008, but no ADR-0008 exists on `main`, so the spec text in #23 is the reference used here).

Method: crate metadata from the crates.io API; feature claims checked against each crate's source, docs and changelog; advisories from [RustSec advisory-db](https://github.com/rustsec/advisory-db/tree/main/crates). Speed numbers come from a local harness (see [Benchmarks](#benchmarks)). Selection weights follow the map: crate maturity, **agent fluency** (all code is written by agents), AGPL-3.0 compatibility, and later mobile reach.

## Summary

| # | Area | Pick | Alternatives | Depends on UI framework? |
|---|------|------|--------------|--------------------------|
| 1 | Markdown parsing | **comrak** (0.59 now, 1.0 at rc.1) | pulldown-cmark (faster, fewer contract features); tree-sitter-md (editor highlighting only) | No for the core. The **editor crate** may want a separate incremental parser (see #30) |
| 2 | Frontmatter | **saphyr** to read + **yaml-edit** for lossless writes | serde_norway (typed serde); span-splice on saphyr's `MarkedYaml` as a fallback writer | No |
| 3 | File watching | **notify 8.2** (`RecommendedWatcher` = inotify), one non-recursive watch per non-ignored directory, *without* the debouncers; `PollWatcher` for network/FUSE | notify 9 (rc.9 today; move after it ships); notify-debouncer-full | No |
| 4 | Trash | **trash 5.3** (`freedesktop` feature, the default) | `portal` feature if Rocky ships as a Flatpak | No (packaging dependent, see #32) |
| 5 | Index persistence | **serde_json** single file, atomic swap, debounced save (ADR-0004 unchanged) | postcard (≈1.7x faster load); rusqlite if incremental persistence is ever needed | No |
| 6a | Syntax highlighting | **Webview:** highlight in the core with syntect (via two-face) or highlight in JS (Shiki). **Native:** syntect + two-face | arborium / tree-sitter-highlight | **Yes** |
| 6b | Math | **Webview:** KaTeX (JS) in the webview. **Native:** no mature Rust LaTeX typesetter; best route is mitex (LaTeX→Typst) + typst → SVG | pulldown-latex → MathML (webview only, young) | **Yes** |
| 7 | Fuzzy search | **nucleo** (the threaded front end) / **nucleo-matcher** | frizbee | No |
| 8 | Concurrency | **std threads + crossbeam-channel + rayon**, walk with **ignore**. No tokio in the core | jwalk; tokio only if the host forces it | Partly: the host adapts the core's sync API (Tauri commands run on tokio) |

## 1. Markdown parsing

The core needs, per note: front-matter bounds, every wikilink/embed (with its byte range), heading text + `[#id]` markers for header targets, and enough structure to render. The parser must never be the writer: Rocky edits source text, so "preserve unknown syntax" means **never re-serialize the AST**. Every candidate's writer normalizes (comrak's `format_commonmark` included), so all writes stay text splices at known offsets.

### comrak: recommended

- **Contract coverage is near-complete natively.** `ExtensionOptions` ships GFM (table, strikethrough, tasklist, autolink), `footnotes`, `math_dollars` (`$…$`, `$$…$$`), `front_matter_delimiter`, `wikilinks_title_after_pipe` (`[[target|label]]`, matching the contract's order), wikilink **embeds** `![[…]]` (0.57), `ofm_callouts` (Obsidian `> [!type]`, any type; 0.58), `ofm_comments` (`%%…%%`, inline and block; 0.57), `highlight` (`==…==`). Source: [`src/parser/options.rs`](https://github.com/kivikakk/comrak/blob/main/src/parser/options.rs), [release notes v0.57–v0.59](https://github.com/kivikakk/comrak/releases).
- **Gap: `## Heading [#id]`.** comrak's `header_attributes` uses `{…}` syntax, not `[#id]`. The marker survives as trailing text in the heading's last `Text` node, so a small post-parse pass extracts it (by node sourcepos) and validates it per contract §5. Auto-slugs can come from comrak's `header_id_prefix`/`HeadingAdapter`, or from our own slugger for exact control.
- **Gap: `[[Note#Section|display]]`** arrives as a wikilink URL `Note#Section`. Rocky splits on `#` itself. That is resolution logic and belongs in the core anyway.
- **Source positions:** every node carries `sourcepos` (line/column, byte columns by default, `sourcepos_chars` for char columns since 0.52). A line-start table converts these to byte offsets. Sourcepos accuracy has been an active focus: many fixes through 2026, and a dedicated sourcepos fuzzer landed in **1.0.0-rc.1 (2026-11-12)** ([release](https://github.com/kivikakk/comrak/releases/tag/v1.0.0-rc.1)).
- **Extensibility:** a real AST (arena of `RefCell` nodes) you can walk and mutate. Plugins cover code-fence rendering and syntax highlighting (`SyntaxHighlighterAdapter`, `CodefenceRendererAdapter`) and heading rendering. There is no inline-syntax plugin API: new syntax means an upstream extension or a post-pass over `Text` nodes. Upstream is receptive: the OFM features above came from outside contributors in 2026.
- **Speed:** 3.6 s single-threaded / **0.30 s on 24 threads** for 100k notes (281 MB). The upstream README is explicit: "Comrak is not and will not be the fastest" ([README §Benchmarking](https://github.com/kivikakk/comrak#benchmarking)).
- **Health:** BSD-2-Clause, 1.7k★, release cadence of weeks, maintained by kivikakk. Used by GitLab, docs.rs, crates.io. Advisories: two 2021 XSS ones, both long fixed ([RUSTSEC-2021-0026](https://rustsec.org/advisories/RUSTSEC-2021-0026), [-0063](https://rustsec.org/advisories/RUSTSEC-2021-0063)). Agent fluency: good (cmark-gfm port, stable API shape for years). Note that **1.0 is imminent**: start on 1.0 if it ships before the build begins.

### pulldown-cmark: strong alternative

- **Coverage (0.13.6, the latest release):** GFM, footnotes, `ENABLE_MATH`, `ENABLE_WIKILINKS` (embeds arrive as `Tag::Image` with `LinkType::WikiLink`), YAML metadata blocks. **Missing:** callouts are limited to GFM's five alert types (`BlockQuoteKind::{Note,Tip,Important,Warning,Caution}`), so `> [!abstract]` is a plain quote. No `%%comments%%`. `ENABLE_HIGHLIGHT` exists on `master` only, unreleased ([`src/lib.rs` @v0.13.6](https://github.com/pulldown-cmark/pulldown-cmark/blob/v0.13.6/pulldown-cmark/src/lib.rs) vs [master](https://github.com/pulldown-cmark/pulldown-cmark/blob/master/pulldown-cmark/src/lib.rs)).
- **Positions:** `into_offset_iter()` gives byte ranges per event directly, the best offset story of the four.
- **Extensibility:** none beyond transforming the event stream. No AST, so you build your own tree to render.
- **Speed:** fastest, at **1.27 s single-threaded / 0.10 s on 24 threads** for 100k notes.
- **Advisories, relevant here:** two 2026 DoS bugs in exactly the wikilink code Rocky would enable: exponential event growth ([RUSTSEC-2026-0414](https://github.com/rustsec/advisory-db/blob/main/crates/pulldown-cmark/RUSTSEC-2026-0414.md), fixed 0.13.5) and quadratic nested image wikilinks ([RUSTSEC-2026-0433](https://github.com/rustsec/advisory-db/blob/main/crates/pulldown-cmark/RUSTSEC-2026-0433.md), fixed 0.13.6). Both are fixed, but the extension is young.
- **When to prefer it:** if the index pass is ever measured as the bottleneck. Rocky would then need hand-rolled callout/comment/highlight handling, which means more custom code for agents to get right.

### markdown-rs (`markdown` crate): not recommended

100% CommonMark/GFM, an mdast AST with positions, and math/frontmatter. But it has **no wikilinks and no extension mechanism by design** ("It is not a goal of this project to support lots of different extensions", [readme](https://github.com/wooorm/markdown-rs/blob/main/readme.md)). Its last commit was **2025-04-23** (1.0.0), and it was the slowest here: ~50 s single-threaded extrapolated for 100k notes (5.0 s per 10k).

### tree-sitter-md: not for the core

The README states: "it is not recommended to use this parser where correctness is important. The main goal… is to provide syntactical information for syntax highlighting" ([README](https://github.com/tree-sitter-grammars/tree-sitter-markdown/blob/split_parser/README.md)). It uses two grammars (block + inline) wired via included ranges. Wiki links are a compile-time-env extension, with no math, callouts, `==`, or `%%`. Its one strength is **incremental reparsing**, which matters to the *editor crate* (Live Preview), not to the index. Defer that to #30.

## 2. Frontmatter (YAML)

`serde_yaml` is archived and marked `+deprecated` (0.9.34). The forks differ sharply in health:

| crate | state | note |
|---|---|---|
| serde_yml | **avoid**: [RUSTSEC-2025-0068](https://github.com/rustsec/advisory-db/blob/main/crates/serde_yml/RUSTSEC-2025-0068.md) "unsound and unmaintained", no patched version | |
| serde_yaml_ng 0.10 | last release 2024-05 | libyaml-based serde fork |
| serde_norway 0.9.42 | last release 2024-12, 11.7M recent dl | libyaml-based (`unsafe-libyaml-norway`) serde fork |
| yaml-rust2 0.13 | active (2026-09) | YAML 1.2, no serde |
| **saphyr 0.3.4** | active (2026-11-11), 341★ | YAML 1.2 "fully compliant" ([README](https://github.com/saphyr-rs/saphyr)). `MarkedYaml` nodes carry spans. Its serde layer `saphyr-serde` is still a 0.0.0 placeholder |
| **yaml-edit 0.3.6** | active (2026-10-31) | **lossless** rowan CST: "preserves all whitespace, comments, and original formatting" ([README](https://github.com/jelmer/yaml-edit)) |

**Round-trip test** (local harness, the frontmatter below, edit `difficulty` and add `status`):

```yaml
title: "Starting Seeds"   # display title
aliases: [Seedlings, 'Germination']
# free attributes below
difficulty: easy
version: 1.10
```

- **yaml-edit:** exactly one changed line (`difficulty: hard`) plus the appended `status: done`. Comments, quotes, flow list, and `1.10` are all intact.
- **serde_norway** (deserialize → mutate → serialize): drops both comments and all quoting, rewrites the flow list as a block list, and turns **`version: 1.10` into `1.1`**. That is silent data loss on a value the user typed.
- **saphyr emitter** (no edit at all): same losses as serde_norway, plus a leading `---` and `ok: "yes"` re-quoted.

**Recommendation: split read and write.** Read every note's frontmatter with **saphyr** (fast: 0.54 s per 100k blocks single-threaded, YAML 1.2, spans available) into Rocky's own `Frontmatter` struct (reserved keys case-insensitive per ADR-0003, free attributes as an ordered map). Write property edits through **yaml-edit** on just the frontmatter slice, then splice that slice back into the file, so the body bytes are never touched.

**Risk:** yaml-edit is young (12★, 29 reverse deps, mostly the author's Debian tooling such as lintian-brush), though it runs the YAML test suite. Mitigations: (a) keep it behind a `FrontmatterEditor` trait, (b) the fallback writer is a **span splice**: locate the key's value span through saphyr's `MarkedYaml`, then replace exactly those bytes. Rocky's edits are scalar or string-list changes on top-level keys, so the fallback stays small. Round-trip property tests (parse → edit → reparse, assert untouched bytes) should gate either path.

## 3. File watching

**notify 8.2.2** is the de facto crate (199M downloads; used by rust-analyzer, Zed, deno, watchexec, mdBook; CC0 core). **notify 9 is at rc.9 (released today, 2026-11-15)** with debouncers 0.8-rc. Its breaking changes include MSRV 1.91, `paths_mut` → `update_paths(Vec<PathOp>)`, and event-kind masks ([UPGRADING_V8_TO_V9](https://github.com/notify-rs/notify/blob/main/docs/UPGRADING_V8_TO_V9.md)). Build on 8.x and move once 9.0 is final, or start on 9 if it ships before the build tickets.

How it maps onto ADR-0004:

- **inotify cost is per directory, not per file.** notify's recursive inotify walk skips non-directories (`if !metadata.is_dir() { continue; }` in [`notify/src/inotify.rs`](https://github.com/notify-rs/notify/blob/main/notify/src/inotify.rs)). A 100k-file vault needs one watch per folder, typically a few thousand. Since Linux 5.11 the default `max_user_watches` scales with RAM up to 1,048,576 (8192 floor), so the ADR's `sysctl` advice is now an edge case. This machine reports 524288. Exhausting the limit surfaces as `ErrorKind::MaxFilesWatch` / ENOSPC ([notify crate docs, "Linux: Bad File Descriptor / No space left on device"](https://docs.rs/notify/latest/notify/#linux-bad-file-descriptor--no-space-left-on-device)).
- **Ignore patterns must apply before watches exist** (ADR-0004), but notify's recursive mode cannot skip subtrees: "To ignore a subtree, filter its events in the event handler" (v9 upgrade guide). So do **not** use `RecursiveMode::Recursive` on the vault root. Instead, the sweep's walker (which already applies ignore globs, §8) yields the directory set, and Rocky registers **one `NonRecursive` watch per non-ignored directory**, adding watches on `Create(Folder)` events and dropping them on removal. This is the ADR's "layered bottom-up" model in Rust terms, and it keeps `node_modules`/`.git` from eating watches.
- **Overflow → sweep.** On `IN_Q_OVERFLOW`, notify emits an event with `Flag::Rescan` (`Event::need_rescan()`). Handle it **before any path filtering** and trigger an immediate full sweep. This is exactly ADR-0004's "dropped events cost staleness until the next sweep", with the next sweep pulled forward.
- **Debouncers: skip them for the core.** notify-debouncer-full stitches renames and collapses duplicate events ([crate docs](https://docs.rs/notify-debouncer-full)), but it owns a thread and a timer, and its rename stitching duplicates what the reconciler already does. Treat raw events as **dirty-path hints**: coalesce them in a small `HashSet<PathBuf>` with a ~100–300 ms quiet window, then re-stat and reparse those paths (and their parents for renames). Rename correctness never depends on pairing `From`/`To` events, because the sweep diff is authoritative. debouncer-full is a fine alternative if the hint-coalescer turns out to be fiddly.
- **Network/FUSE mounts:** notify documents that NFS-style mounts "may not emit any events" and recommends `PollWatcher` ([crate docs](https://docs.rs/notify/latest/notify/)). For Rocky, polling mode is **just the sweep at a short interval**: the snapshot-and-diff reconciler already *is* a poller. So the `polling` workspace setting can skip `PollWatcher` entirely: turn off inotify and shorten the sweep interval. `PollWatcher::with_compare_contents` only matters for mtime-preserving writers.
- **Sweep snapshot** = `(path → size, mtime_ns, inode)` from the walk. Diff against the index, then reparse only changed entries. At 100k files a metadata-only walk costs `stat` calls, not file reads. Reparsing is bounded by §1's numbers.

## 4. Trash

**trash 5.3.0** (Byron/trash-rs, MIT, 255★, active 2026-11). Its freedesktop backend implements [Trash spec 1.0](https://specifications.freedesktop.org/trash-spec/trashspec-1.0.html). Checked against issue #23 in [`src/freedesktop.rs`](https://github.com/Byron/trash-rs/blob/master/src/freedesktop.rs):

- **`.trashinfo`:** written *before* the move, with `create_new` to avoid name races, containing `Path=` (URI-encoded **absolute** path) and `DeletionDate=` in local time. `DeletionDate` is emitted **only with the `chrono` feature**, which is on by default, so do not build with `default-features = false` without re-adding `chrono`, or #23's "timestamp preserved" breaks silently.
- **Volume-root fallback:** for a file on a different mount than `$XDG_DATA_HOME/Trash`, it tries `$topdir/.Trash/$uid` (only if `.Trash` passes the sticky/non-symlink validity check), then `$topdir/.Trash-$uid` (created if missing). If the volume trash is not writable, it **falls back to the home trash**.
- **Caveat (#23 "nothing permanently deleted"):** the home-trash fallback for a cross-device file goes through `rename`, which fails with `EXDEV`, so the crate **copies and then deletes** the source (`move_items_no_replace`). That is still a trash, not a destruction, but it is slow for a large folder and not atomic. Acceptable. Note it in #23's acceptance tests (a cross-device folder delete must leave the subtree in some trash).
- Directories are trashed whole (subtree → one trash entry), which matches "deleting a folder trashes the whole subtree".
- `Path=` is absolute even in per-volume trashes. The spec permits it, but a vault on a removable drive mounted elsewhere later restores to the old mount path. This is minor, and OS file managers handle it.
- **Flatpak:** the `portal` feature uses `org.freedesktop.portal.Trash` (no host filesystem trash access needed). It cannot trash symlinks (upstream portal issue #518). It is mutually exclusive with `freedesktop` at build time. This is a **packaging-dependent** choice, settled in #32.

## 5. Index persistence

The ADR-0004 entry shape is benchmarked with 100k entries, each carrying 12 out-links, 2 aliases and 1 parent:

| format | encode | decode | size | file write / read+parse |
|---|---:|---:|---:|---:|
| **serde_json** | 46 ms | 212 ms | 45.3 MB | 10 ms write+rename / 215 ms |
| postcard 1.1 | 19 ms | 128 ms | 26.5 MB | n/a |
| rusqlite 0.43 (bundled, WAL, one txn) | 212 ms insert | 107 ms select-all | 37.2 MB | n/a |
| redb 4.6 (postcard values) | 180 ms insert | 85 ms read-all | 71.3 MB | n/a |

**Pick: keep ADR-0004 as written, using serde_json.** A 0.2 s cold load is well under a full rebuild, which is dominated by reading 100k files: parse alone is 0.1–0.3 s parallel, plus I/O. The format is debuggable, and serde's `#[serde(default)]` handles additive schema changes. Mitigate the real cost, which is **rewriting 45 MB per change**, by **debouncing saves** (e.g. on idle ≥ 2 s, on close, and after a sweep). The index is derived, so a lost save costs one reconcile, not data.

- **postcard** is the drop-in upgrade if load time matters: ~1.7x faster, 40% smaller, MIT/Apache, active. Because it is not self-describing, any schema change requires a version bump plus a rebuild. That is acceptable for a derived file but loses inspectability.
- **bincode: avoid.** [RUSTSEC-2025-0141](https://github.com/rustsec/advisory-db/blob/main/crates/bincode/RUSTSEC-2025-0141.md) "Bincode is unmaintained" (2025-12-16). 3.0.0 is the tombstone release.
- **rkyv: avoid for this.** Zero-copy is overkill at 200 ms, and 2026 brought five advisories, including UAF and OOB reads on crafted archives (RUSTSEC-2026-0001, -0122, -0233, -0234, -0235; all fixed ≥ 0.8.17).
- **rusqlite** is the right answer *only* if Rocky later wants incremental persistence or queries (full-text search, graph). Pin a version that bundles SQLite ≥ the [RUSTSEC-2026-0459](https://github.com/rustsec/advisory-db/blob/main/crates/libsqlite3-sys/RUSTSEC-2026-0459.md) fix (libsqlite3-sys ≥ 0.39). **redb** is pure Rust and fast but produced the largest file. Neither earns its complexity for a rebuildable 100k-entry map.

## 6. Rendering helpers: framework dependent

### 6a. Syntax highlighting

- **syntect 5.3** (MIT, 2.5k★, 40M dl). Sublime `.sublime-syntax` grammars, emits HTML with classes or inline styles, or styled spans for native text layout. Pair it with **two-face** (bat's extra syntaxes and themes, 10.9M recent dl) for broad language coverage. Use the pure-Rust `fancy-regex` backend (since 4.0) to avoid building the Oniguruma C library ([README](https://github.com/trishume/syntect#pure-rust-fancy-regex-mode-without-onig)). Comrak has a built-in `SyntectAdapter` plugin. Agent fluency: high (bat and delta are built on it).
- **tree-sitter-highlight 0.27** (+ per-language grammar crates) or **arborium 2.18** (bundles ~70 permissively licensed tree-sitter grammars with an HTML renderer, MIT/Apache, [README](https://github.com/bearcove/arborium)). These give better accuracy than regex grammars, but each grammar is C and adds build time and binary size. arborium is the easy entry if tree-sitter is wanted. It is also the natural pick if the **editor** (#30) standardizes on tree-sitter, so the code-block highlighting matches.
- **Webview UI:** either highlight in the core (syntect → class-annotated HTML, theme via CSS) or in JS (Shiki, as ADR-0007 planned). Prefer the core, so highlighting stays host-agnostic and testable in Rust.
- **Native UI:** syntect's `(Style, &str)` spans map directly onto rich-text runs in iced/egui/GPUI.

### 6b. Math

- **Webview UI: KaTeX in the webview.** It is mature, the contract says "KaTeX math", and comrak already marks math nodes (`data-math-style`). A pure-Rust option is **pulldown-latex** (LaTeX → MathML, MIT, active 2026-08), since WebKitGTK renders MathML. But its own README says "not recommended for large scale production use as more robust testing is required" ([README](https://github.com/Carlosted/pulldown-latex)). Keep it as a fallback only.
- **Native UI: no mature Rust LaTeX typesetter exists.** The `katex` crate runs KaTeX inside an embedded JS engine and emits **HTML**, which is useless without a browser; it was last released 2023. The crates.io `ReX` name now points to an unrelated workflow-language project ([peterkelly/rex](https://github.com/peterkelly/rex)), so the old ReX typesetter is gone. The credible route is **mitex** (LaTeX → Typst, Apache-2.0, 646★; "correctly converted 32.5k equations from OI Wiki", [README](https://github.com/mitex-rs/mitex)) feeding **typst 0.15** (Apache-2.0) to lay out the formula as SVG/frames, then rasterizing (resvg) or drawing the paths natively. This works, but typst is a heavy dependency, and KaTeX-parity gaps (macros, `\newcommand` edge cases) will exist. **This is the single strongest framework-dependent cost in the core list.**

## 7. Fuzzy search

**nucleo / nucleo-matcher** (Helix's matcher, MPL-2.0, 1.5k★; MPL is file-level copyleft and compatible with AGPL-3.0). Measured: **32–45 ms single-threaded** for a query over 200k strings (100k names + 100k paths) with `nucleo-matcher` 0.3 and `Config::DEFAULT.match_paths()`. The high-level `nucleo` crate adds a lock-free, multi-threaded, incremental matcher with a snapshot API designed for interactive pickers, which suits the quick switcher and `[[` autocomplete. Score name, aliases, title and path as separate columns. Caveat: the last crates.io releases date from 2024 (repo active 2026-06). The API is stable, but watch for a 1.0.

- **frizbee 0.13** (MIT, SIMD Smith-Waterman, used by blink.cmp, active 2026-11) is the alternative if MPL is unwanted.
- **fuzzy-matcher: avoid**: the repository is **archived** (last release 2020).

## 8. Concurrency

**Recommendation: no async runtime in `rocky-core`.** All the work is local CPU + blocking filesystem I/O, with no network. Use:

- **ignore 0.4** (ripgrep's walker) for the vault walk: `WalkBuilder::build_parallel()` plus `overrides::OverrideBuilder` for Rocky's ignore globs, with `.rocky/` and `.git/` as built-in excludes. It gives parallelism, glob filtering *before descending* (which ADR-0004 needs for both the sweep and watch registration), and symlink-loop handling in one place ([docs](https://docs.rs/ignore)). Disable its gitignore defaults (`standard_filters(false)`) so Rocky's own ignore patterns are the only rule. jwalk is the alternative.
- **rayon** for parse fan-out: `par_iter` over changed paths took the 100k parse from 1.27 s → 0.10 s (pulldown-cmark) and 3.6 s → 0.30 s (comrak) on 24 threads.
- **std threads + crossbeam-channel** for the long-lived parts: a watcher thread (notify's callback → channel), one **index-owner thread** that serializes all mutations (apply reparsed entries, run sweeps, debounce saves), and snapshot reads published as `Arc<Index>` (swap on commit). Readers never block writers, and agents get a simple actor model with no lock ordering to get wrong.
- **tokio: no**, unless the host needs it. The core exposes a **sync API plus an event channel**. Hosts adapt it: Tauri's command layer runs on tokio ([`tauri::async_runtime`](https://docs.rs/tauri/latest/tauri/async_runtime/)), so wrap calls in `spawn_blocking` and forward the channel to frontend events. iced takes a `Subscription` over the channel. GPUI uses its `BackgroundExecutor`. This is the one **framework-touching seam**, and it lives in the app crate, not the core.

## Benchmarks

Harness: a throwaway Rust binary (not committed). The synthetic vault is 100k notes generated from `docs/example-note.md` (every contract construct) with 8 extra paragraphs each, giving 2.8 KB average and 281 MB total, all in memory with no disk I/O. Each note has 16 wikilinks/embeds, and **all three counting parsers agreed on 1.6M**. Release build, rustc 1.98.1, i7-13700KF (24 threads), 32 GB, Linux 7.2. Single runs, so treat the numbers as ±15%.

| task (100k notes unless stated) | 1 thread | rayon (24 threads) |
|---|---:|---:|
| pulldown-cmark 0.13.6 parse + offsets | 1271 ms | 99 ms |
| comrak 0.59 parse to AST + walk | 3569 ms | 305 ms |
| markdown-rs 1.0 to_mdast (10k notes) | 5036 ms | n/a |
| saphyr 0.3 load frontmatter | 538 ms | n/a |
| serde_norway 0.9 → Value | 661 ms | n/a |
| yaml-edit 0.3 lossless parse + print | 1597 ms | n/a |
| nucleo-matcher, 200k strings, 1 query | 4–45 ms | n/a |

Index persistence numbers are in §5.

## Sources

- crates.io API metadata (versions, dates, downloads) for every crate named, fetched 2026-11-15.
- comrak: [options.rs](https://github.com/kivikakk/comrak/blob/main/src/parser/options.rs), [releases](https://github.com/kivikakk/comrak/releases), [README](https://github.com/kivikakk/comrak).
- pulldown-cmark: [lib.rs @v0.13.6](https://github.com/pulldown-cmark/pulldown-cmark/blob/v0.13.6/pulldown-cmark/src/lib.rs); RustSec [2026-0414](https://github.com/rustsec/advisory-db/blob/main/crates/pulldown-cmark/RUSTSEC-2026-0414.md), [2026-0433](https://github.com/rustsec/advisory-db/blob/main/crates/pulldown-cmark/RUSTSEC-2026-0433.md).
- markdown-rs: [readme](https://github.com/wooorm/markdown-rs/blob/main/readme.md). tree-sitter-md: [README](https://github.com/tree-sitter-grammars/tree-sitter-markdown/blob/split_parser/README.md).
- YAML: [saphyr](https://github.com/saphyr-rs/saphyr), [yaml-edit](https://github.com/jelmer/yaml-edit), [RUSTSEC-2025-0068 (serde_yml)](https://github.com/rustsec/advisory-db/blob/main/crates/serde_yml/RUSTSEC-2025-0068.md).
- notify: [crate docs](https://docs.rs/notify/latest/notify/), [inotify.rs](https://github.com/notify-rs/notify/blob/main/notify/src/inotify.rs), [UPGRADING_V8_TO_V9](https://github.com/notify-rs/notify/blob/main/docs/UPGRADING_V8_TO_V9.md), [debouncer-full docs](https://docs.rs/notify-debouncer-full).
- trash: [freedesktop.rs](https://github.com/Byron/trash-rs/blob/master/src/freedesktop.rs), [lib.rs docs](https://docs.rs/trash), [FreeDesktop Trash spec 1.0](https://specifications.freedesktop.org/trash-spec/trashspec-1.0.html).
- Persistence: RustSec [bincode 2025-0141](https://github.com/rustsec/advisory-db/blob/main/crates/bincode/RUSTSEC-2025-0141.md), [rkyv advisories](https://github.com/rustsec/advisory-db/tree/main/crates/rkyv), [libsqlite3-sys 2026-0459](https://github.com/rustsec/advisory-db/blob/main/crates/libsqlite3-sys/RUSTSEC-2026-0459.md).
- Rendering: [syntect](https://github.com/trishume/syntect), [two-face](https://codeberg.org/CosmicHarper/two-face), [arborium](https://github.com/bearcove/arborium), [pulldown-latex](https://github.com/Carlosted/pulldown-latex), [mitex](https://github.com/mitex-rs/mitex), [typst](https://github.com/typst/typst), [katex-rs](https://github.com/xu-cheng/katex-rs).
- Search: [nucleo](https://github.com/helix-editor/nucleo), [frizbee](https://github.com/saghen/frizbee), [fuzzy-matcher (archived)](https://github.com/lotabout/fuzzy-matcher).
- Concurrency: [ignore docs](https://docs.rs/ignore), [rayon](https://github.com/rayon-rs/rayon), [crossbeam-channel](https://docs.rs/crossbeam-channel), [tauri::async_runtime](https://docs.rs/tauri/latest/tauri/async_runtime/).
