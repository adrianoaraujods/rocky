# Editor & rendering landscape

Research ticket: [Editor and rendering landscape](https://github.com/adrianoaraujods/rocky/issues/6) (GH #6). As of September 2026, this maps the credible editor + Markdown rendering stacks for Rocky — a self-hosted, browser-based notes app where notes are plain local Markdown files with Obsidian-dialect `[[Name]]` wikilinks that must stay plain text in source. Sources are primary (official docs, source repos, maintainer forums, npm registry metadata) unless noted. No application code was written.

The decisive constraints are (2) raw-source fidelity — wikilinks and every byte of formatting must survive untouched on disk — and (4) huge-document performance, against a brutally-simple, solo-dev + AI-agent ambition. Those two force a source-first or markdown-native editor, not a generic rich-text engine.

## Verdicts

### CodeMirror 6

- **Modality:** Source editing natively; live preview is an add-on layer (overlay + block widgets via `EditorView.decorations`). Both are achievable in one editor — this is exactly how Obsidian's Live Preview works.
- **Raw-source fidelity:** Maximal. The document state *is* the text; there is no serialize/round-trip step on save. `[[Name]]` stays literal text; highlighting it is a Lezer-grammar extension problem, not a storage problem.
- **Huge documents:** Viewport virtualization is built in; the official example loads a document of "a few million lines" with pruning of parser work when scrolled away.
- **Mobile:** Proven on touch: Obsidian's mobile app runs the same CodeMirror 6 editor; the Official plugin docs state "Obsidian uses CodeMirror (CM) as the underlying text editor."
- **Licensing:** MIT.
- **Maintenance:** Highly active (Marijn Haverbeke's org; `@lezer/markdown` published 2026-07, `codemirror` 6.0.2 2026-02); the Langs/Lezer markdown grammar ships GFM (the `markdownLanguage` is "GFM plus subscript, superscript, and emoji"; strict `commonmarkLanguage` also included).
- **Verdict: Viable** — the recommended "simple reliable" base.
  - <https://codemirror.net/docs/guide>
  - <https://codemirror.net/examples/million/>
  - <https://www.npmjs.com/package/@codemirror/lang-markdown>
  - <https://docs.obsidian.md/Plugins/Editor/Editor>

### ProseMirror / TipTap

- **Modality:** WYSIWYG (schema-based) only; no source mode — a raw view means bolting on CodeMirror separately.
- **Raw-source fidelity:** Weak for a plain-markdown product. `prosemirror-markdown` (the official parser/serializer) tokenizes with markdown-it and rebuilds source; it normalizes markup, so a save can reformat a file. Wikilinks must be modeled as a custom node with a custom `toMarkdown` rule and round-trip tests.
- **Huge documents:** ProseMirror has no viewport rendering (maintainer: "this is intentionally out of scope"); the ProseMirror forum and the Tiptap forum (Nov 2025, ~200k-word docs) point to splitting documents into chunks.
- **Mobile:** contenteditable-based; usable but takes custom toolbar/cursor work.
- **Licensing:** MIT for both. TipTap's core is MIT; since June 2025 formerly-paid Pro extensions were open-sourced under MIT and monetization moved to *cloud* feature bundles (collaboration/AI) — no red flag for a self-hosted app.
- **Maintenance:** Excellent: ProseMirror (`prosemirror-view` 1.42.5, 2026-09) and TipTap (`@tiptap/core` 3.31.3, 2026-09, ~12-13M downloads/week) are both very active.
- **Verdict: Viable** — the industry-wide WYSIWYG workhorse, but it taxes the two things Rocky cares most about (source fidelity, huge docs).
  - <https://prosemirror.net/> and <https://github.com/ProseMirror/prosemirror-markdown>
  - <https://discuss.prosemirror.net/t/efficient-viewport-rendering-like-codemirror/577>
  - <https://tiptap.dev> and <https://tiptap.dev/blog/release-notes/were-open-sourcing-more-of-tiptap>

### Milkdown

- **Modality:** Typora-style live WYSIWYG editing ("inspired by Typora, built on top of prosemirror and remark"); raw Markdown is a read/write API (`getMarkdown`/`setMarkdown`), not a persistent editing mode.
- **Raw-source fidelity:** Same ProseMirror serialization caveat as above — proofread serialized output; wikilinks need a dedicated node + serializer.
- **Huge documents:** Inherits ProseMirror's no-virtualization limits.
- **Mobile:** PM-based; no special mobile-first story advertised.
- **Licensing:** MIT.
- **Maintenance:** Active but thin-bus-factor: one lead maintainer (Mirone), funding is sponsorships/GitHub; `@milkdown/core` 7.22.1 published 2026-08; ~11.8k stars; docs note maintenance funding need. The value-add is that the data layer *is* remark/mdast — a natural seam for our own `remark-obsidian-md`.
- **Verdict: Viable** — the most markdown-native WYSIWYG option and the best live-preview candidate for this product.
  - <https://github.com/Milkdown/milkdown> and <https://milkdown.dev/>

### Lexical (Meta)

- **Modality:** WYSIWYG rich text only; no source/markdown editing mode.
- **Raw-source fidelity:** Not a markdown engine. `@lexical/markdown` is import/export + shortcuts via a small transformer set (element/text-format/text-match); no GFM tables, no wikilinks, and the round-trip is lossy.
- **Huge documents:** contenteditable-rendered; large-doc behavior similar to ProseMirror, with far less community tooling around it for markdown.
- **Mobile:** Meta runs it in WhatsApp/other products, so mobile is battle-tested for rich text — irrelevant to markdown editing.
- **Licensing:** MIT (Meta-backed).
- **Maintenance:** Very active (`lexical` 0.51.0, 2026-09; dedicated Meta team).
- **Verdict: Not viable** for a markdown-as-source product — markdown is a secondary, lossy feature, not the data model.
  - <https://lexical.dev/> and <https://lexical.dev/docs/packages/lexical-markdown>

### Block-based / Notion-style editors

- **BlockNote** (TypeCell/OpenBlocks): React-only; documents are a proprietary block JSON tree. Markdown import is explicitly "lossy" ("not all Markdown structures can be entirely represented as BlockNote blocks"), and markdown files on disk are the exact opposite of its model. Core is **MPL-2.0**; the XL package tier (AI, PDF/DOCX/ODT export, multi-column) is **GPL-3.0 or a commercial license ($195/mo)**.
- **Editor.js** (CodeX): block-based, outputs JSON, Apache-2.0 — not a markdown serializer.
- **2025-2026 entrants:** Nothing credible has shipped that treats Obsidian-dialect plain `.md` as the working format. New entrants (e.g. AI-first block editors, Tiptap's Pro Blocks) are rich-text/JSON models on ProseMirror.
- **Verdict: Not viable** — these are app-level rich-text data models; all three of source fidelity, wikilink plain-text storage, and markdown round-tripping are design-antithetical to them.
  - <https://www.blocknotejs.org/docs/features/import/markdown> and <https://github.com/TypeCellOS/BlockNote/blob/main/LICENSE.txt>
  - <https://github.com/codex-team/editor.js>

### Rendering pipelines: remark/unified, markdown-it, wikilinks, GFM

- **remark / unified (micromark → mdast → hast):** MIT; the unified collective maintains ~500 packages (477 packages, ~9.9B downloads/month), `remark` 14 / `remark-gfm` 4.x provide GFM (autolink literals, footnotes, strikethrough, tables, tasklists); `remark-frontmatter`/`mdast-util-frontmatter` cover Rocky's YAML attributes. mdast round-trips through `mdast-util-to-markdown`, giving us a byte-fidelity serializer we can pin with tests. This is the ecosystem our `remark-obsidian-md` already targets. **Verdict: Viable — preferred.**
- **markdown-it:** MIT, active (`markdown-it` 15.0.2, 2026-09), fast, CommonMark to 100%; GFM tables + strikethrough built in; tasklists require community plugins. Token-stream (no AST/mdast), so wikilink/obsidian extensions are less ergonomic than the mdast world. **Verdict: Viable — simpler, but AST-less; weaker seam for our prior art.** (It is also the tokenizer under ProseMirror's markdown support.)
- **Wikilink plugins (Obsidian-dialect):**
  - `remark-wiki-link` / `mdast-util-wiki-link` (landakram): MIT, but stale — npm last modified 2023-10; alias divider is `[[Page:Alias]]` (colon), not Obsidian's `[[Page|Alias]]` (pipe), so it does not match the dialect on its own.
  - `@portaljs/remark-wiki-link`: Obsidian `shortestPossible` path resolution; maintained by Datopian but slow cadence (1.2.0, ~2 years old).
  - `@flowershow/remark-wiki-link`: **actively maintained** (4.0.0, npm modified 2026-06); supports Obsidian dialect including `[[Link#heading]]`, `[[Link|alias]]`, and `![[image.png|200x300]]`, `![[video]]`, `![[audio]]`, `![[pdf]]` embeds. Note-embeds (`![[Embed note]]`) are still on its roadmap, not done.
  - `adrianoaraujods/remark-obsidian-md` (our repo, 1.1.0): already handles wikilinks, embeds, callouts, highlights, frontmatter against the real vault filesystem.
- **Obsidian-dialect wikilinks & embeds in 2026 — direct answer:** there is **no canonical, maintained library** that implements the full Obsidian dialect (wikilinks + heading links + all embed kinds + callouts) to spec. Obsidian's own parser is proprietary, built on CodeMirror 6. The credible maintained pieces are remark/mdast plugins, led by `@flowershow/remark-wiki-link` (parsing reference) and our own `remark-obsidian-md`. **We must extend an mdast/unified pipeline ourselves — which we already own and have published.**
  - <https://unifiedjs.com/> and <https://github.com/remarkjs/remark-gfm>
  - <https://github.com/markdown-it/markdown-it>
  - <https://www.npmjs.com/package/@flowershow/remark-wiki-link>
  - <https://github.com/adrianoaraujods/remark-obsidian-md>

## Two credible options

Both are MIT across the entire chain, both are browser-shaped, and both reuse our `remark-obsidian-md` / mdast prior art for the Obsidian dialect. They differ on the one axis the ticket centers: source-first vs live-preview-first.

1. **Simple reliable — CodeMirror 6 (source-first) + remark/unified (mdast) rendering.** The editor buffer *is* the file: `[[Name]]` and every formatting byte stay exactly as written, huge-document virtualization is built in (million-line demo), mobile is proven (Obsidian desktop *and* mobile run CM6), and the render side is a unified/mdast pipeline where `remark-obsidian-md` plugs straight in. A richer "live preview" mode is achievable later as a CM6 decorations overlay (Obsidian's Live Preview is exactly this), and the Lezer grammar can learn `[[ ]]` for highlight/clickable links via a small grammar extension.
   **Trade-off it buys:** near-zero risk on the two hard constraints (source fidelity, huge docs) and on licensing; cheapest thing a solo dev + AI agent can own. **What it costs:** the "wow" editing surface is ours to build — the default UX is markdown source editing, and live preview / WYSIWYG niceties (toolbars, command menu, WYSIWYG formatting) are deferred.

2. **Richer live-preview — Milkdown (ProseMirror WYSIWYG on a remark/mdast data model).** A Typora-style live editor out of the box — markdown semantics are native (not emulated rich text); its remark/mdast data layer gives a first-class seam for `remark-obsidian-md`, and it upgrades to TipTap/ProseMirror internals if we later need fancier node views.
   **Trade-off it buys:** an immediately pleasant live WYSIWYG editing experience in one MIT dependency, markdown-native (not a JSON rich-text model). **What it costs:** ProseMirror serialization will normalize saved Markdown (wikilink fidelity needs a dedicated node + serializer + round-trip tests), huge-document performance is weaker (no viewport virtualization — plan to chunk long notes), and it carries a single-maintainer bus factor.

Given Rocky's brutal simplicity and obsidian wires (raw files must stay pristine, and Obsidian proves a CM6 source-first product is what the audience lives with), **Option 1 is the recommendation**; Option 2 is the credible upgrade path if live WYSIWYG editing becomes a hard product requirement rather than a nice-to-have.