# Editor stack: CodeMirror 6 source-first with Live Preview

Rocky's file contract (ADR-0002) demands readable `[[Name]]` wikilinks and byte-pristine Markdown on disk, against a browser-based self-hosted app where the driver wants a rendered-feeling editing experience (explicitly the reason Trilium Notes is disliked). Research (issue #6) mapped the 2026 editor/rendering landscape; the choice was grilled and locked in issue #8: **CodeMirror 6 as the editor (the source buffer *is* the file) + remark/unified (mdast) rendering, client-side**, with **Live Preview as the MVP's default editing mode** — overturning the research report's "live preview deferred" trade-off, which positions CM6's render-decoration layer as the edit surface from day one.

## Considered Options

- **Milkdown (ProseMirror WYSIWYG on a remark/mdast model)**: Typora-style live editing out of the box, but its serializer normalizes saved Markdown (threatening the readable-source contract without dedicated node + round-trip tests), it has no viewport virtualization for the ~100k-file vault's long notes, and it is a single-maintainer bus factor.
- **Full Typora-style WYSIWYG (hidden source, contenteditable body)**: rejected outright — it requires the source buffer to be *replaced* by a rendered body, which CodeMirror cannot do without becoming a ProseMirror-shaped editor, breaking "the buffer is the file" (ADR-0002).
- **Block-based editors (BlockNote, Editor.js) and Lexical**: rejected — markdown-as-source is design-antithetical to their JSON/AST data models.

## Consequences

- **Editing model**: Obsidian-style live markup — inline markup (bold, links, wikilinks, images, checkboxes) is directly editable while compiled; block constructs (headings, callouts, math, code fences, tables) render as final-looking widgets that snap to their source on interaction. A **Source-mode toggle** ships as the fidelity escape hatch.
- **Render pipeline**: client-side, resolving wikilinks/embeds against the vault-index registry (not the live filesystem). **KaTeX** for math, **Shiki** for code highlight, **no in-editor Mermaid** — ` ```mermaid ` fences render as plain code blocks; the deferred Diagram note type embeds only its **rendered** view (no in-editor editing of any rich built-in).
- **Obsidian dialect**: bridged by **rewriting our own `remark-obsidian-md`** to match this project (client-side, index-based resolution) — no third-party wikilink plugin is adopted.
- **Reading View** (a distinct read-only surface) is deferred post-MVP; expected to reuse Live Preview's renderer.
- **MVP feature cut (#5)**: Live Preview + Source toggle are MVP editor features, feeding the cut's complexity budget.