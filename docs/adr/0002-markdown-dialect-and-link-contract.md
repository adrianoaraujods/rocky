# Markdown dialect and link contract

The on-disk file contract the MVP hangs on (Quit-App-Still-Works), locked in issue #2: a note is CommonMark + GFM (tables, strikethrough, task lists, autolinks) plus the Obsidian-flavoured extensions made first-class — wikilinks `[[Name]]`, embeds `![[…]]`, header references `[[Note#Section]]`, callouts, KaTeX math, footnotes, highlight, comments. HTML passes through raw; Mermaid/diagram fences are raw-preserved only. Full spec in `docs/markdown-contract.md`.

## Considered Options

- **Full Obsidian-dialect compatibility**: rejected — block references `[[Note#^blockid]]` are excluded (their `^anchor` markers stay literal text) and Mermaid is not rendered inline; block-id tracking and diagram rendering don't earn their complexity for the MVP.
- **Strict minimal dialect (GFM only)**: rejected — "most Obsidian flavours" is the product's editing feel, not a future add-on.
- **Explicit heading anchors**: chosen — `## Heading [#custom-id]` overrides the auto-slug so `[[Note#id]]` links survive heading-text edits; the marker is hidden in rendering but preserved in the file.

## Consequences

- The renderer (editor ticket #8) must implement callouts, math, footnotes, highlight, and comments on top of a CommonMark parser.
- `[#id]` markers must be stripped from rendering but never from the source.
- Unknown syntax is preserved, never mangled or rejected — the parser contract.
- Mermaid fence-vs-dedicated-note-type coexistence stays fog for the rich-built-ins effort.