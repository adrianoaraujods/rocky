# Rocky Markdown contract

The on-disk file contract for Rocky notes. This is the thing **Quit-App-Still-Works** hangs on: every rule below keeps the vault readable as plain Markdown outside the app. Decided in issue #2 (Markdown contract and link syntax).

## 1. Dialect floor

A note is a `.md` file whose content must render correctly in the MVP:

- **CommonMark + GFM core**: paragraphs, emphasis, headings, lists, tables, strikethrough, task lists (`- [ ]`), autolinks, fenced code.
- **Rocky/Obsidian-style extensions**:
  - Wikilinks `[[…]]` and embeds `![[…]]`
  - Header references `[[Note#Section]]`
  - Custom heading ids `## Heading [#id]` (see §5)
  - Callouts `> [!type]`
  - KaTeX math `$…$` and `$$…$$`
  - Footnotes `[^1]` and their definitions
  - Highlight `==text==`
  - Comments `%% … %%`
- **HTML passes through raw** — rendered, never stripped or rewritten.
- Unknown syntax **must not corrupt or delete anything**; the parser detects and preserves, never rejects the file.
- Mermaid/diagram code fences are **raw-preserved only** — never rendered inline by the MVP. A dedicated Mermaid note type belongs to the rich-built-ins effort (post-MVP); whether it coexists with the inline fence is undecided fog.
- Block references `[[Note#^blockid]]` are **not recognized**. A trailing `^blockid` anchor is ordinary text.
- Namespaces reserved by the app stay raw in the file; the contract never requires rewriting user text.

## 2. Frontmatter

- **Optional.** When present, it is a leading `---`-fenced YAML block before any other content.
- **Reserved keys** (lowercase in the file — a case-insensitive collision with a reserved key may not be used as a custom attribute):

  | key          | meaning                                                                                                   |
  | ------------ | --------------------------------------------------------------------------------------------------------- |
  | `title`      | Display title; overrides the filename in the tree and UI. Never a link target.                            |
  | `aliases`    | YAML list of alternate link targets. Alias collisions show a disambiguation picker. Never auto-hidden.    |
  | `parent`     | YAML list of vault-relative folder paths; each entry creates a **clone** of the note under that folder in the tree overlay. Cycles rejected. |
  | `type`       | `tag` marks a tag note (see §6). Reserved for future typed notes.                                         |
  | `cssclasses` | YAML list; applied as CSS classes when the note renders (Obsidian-compatible). Loads a custom theme/CSS if present. |
  | `foldernote` | `foldernote: false` opts a `X/X.md` file out of folder-note recognition (see §7).                        |

- **Custom attributes**: any other key, Trilium-style key/value or key/list (e.g. `status: done`, `tracking: [a, b]`). The user chooses the casing; anything that is not reserved is metadata.
- **Display**: the properties UI renders reserved keys capitalized (`Parent`, `CSS Classes`, …). Reserved keys auto-hide unless the properties element is focused; a workspace setting disables the auto-hide, and another disables the capitalized rendering. `aliases` always shows first and is never hidden.

## 3. Note identity and uniqueness

- A note's identity is its **filename** minus `.md`. There is no `name` frontmatter key.
- Filenames may contain spaces. They must not contain `[`, `]`, `|`, `#`, or newlines — Rocky blocks creating or renaming to such names.
- The filesystem is the truth: **duplicate names are possible**. Uniqueness is a workspace setting applied to the whole opened root folder:
  - default `warn` — warn when a note is created/renamed with a name already in use;
  - optional `enforce` — refuse duplicates;
  - it can also be disabled entirely.
- Duplicate names are **never autocompleted by name**; a specific file is reachable through a path link (`[[/folder/file.md]]`).
- `[[Name]]` with several matches renders a picker.

## 4. Link syntax

Supported forms:

| form                                  | meaning                          |
| ------------------------------------- | -------------------------------- |
| `[[Name]]`                            | note by name or alias            |
| `[[Name|display]]`                    | note with overridden link text   |
| `[[Note#Section]]`                    | header reference                 |
| `[[Note#id]]`                         | header reference by explicit id  |
| `[[Note#Section|display]]`            | header reference with text       |
| `[[path/to/file.ext]]`                | path link to any file            |
| `![[Name]]` / `![[Note#Section]]`     | embed on any resolvable target   |
| `![[path/image.png]]`                 | inline image / media embed       |

- Literal brackets are escaped `\[\[literal\]\]`.
- A display part may contain spaces and any characters except `[` and `]`.

## 5. Header anchors and custom heading ids

- Every heading gets an **auto-slug** id, derived from its text — the default `[[Note#slug]]` target.
- `## Heading [#custom-id]` declares an explicit id that **overrides** the slug.
- An explicit id must not contain `[`, `]`, or `|`; otherwise the marker is treated as literal text.
- The `[#id]` marker is **hidden in rendering** (a contract artifact, like Obsidian's block anchors) but stays in the file.
- Links by explicit id survive later heading-text edits; slug-targeted links break when the text changes.

## 6. Tag notes

- A tag is a Markdown note marked `type: tag`.
- Other notes tag it with an ordinary wikilink `[[Name]]` — no inline `#tag` syntax.
- The tag note's backlinks are the tagged notes. Nothing else is special; the `type` mechanism is reserved for future typed notes.

## 7. Folder notes

- A Markdown file `X/X.md` — inside a folder and sharing its name — is that folder's **folder note**: auto-recognized at index time, opt-out with `foldernote: false`.
- The folder note represents the folder in the tree; `[[X]]` resolves to it.
- The decision to opt out lives **in the file**, never in the index, so it survives any index rebuild.
- A non-Markdown file `X/promo.png` sharing the folder's name is an ordinary file, not a folder note.

## 8. Resolution algorithm

1. **Leading `/`** → vault-relative path link, resolved against the filesystem. Unique by construction; resolves notes and non-note files alike.
2. **Otherwise** → registry lookup by **filename or declared alias**:
   - exactly one match → resolve;
   - several matches → autocomplete shows all, `[[Name]]` renders unresolved-until-picked (picker on click);
   - no match → **unresolved** (broken-link marker).
3. **`title` is never a link target** — display only.
4. Header targets: explicit `[#id]` wins, else auto-slug.

## 9. Renames, moves, external edits

- The vault index is **derived** from the filesystem (ADR-0001) and rebuilt on every change; there is no persistent state that can disagree with the disk, so nothing gets corrupted.
- **Name-based links survive** renames and moves by construction.
- **Path links break** when their target is deleted or renamed: rendered with an unresolved marker and a jump-to-autocomplete affordance. No auto-repointing of moved files in the MVP.
- **External edits** (file added/removed/renamed outside the app) are picked up by the watcher; vanished names render as broken links.

## 10. Acceptance sample

`docs/example-note.md` is the reference sample: frontmatter (reserved + custom), wikilink, alias link, custom display text, path link, embed, header reference by explicit id and by auto-slug, callout, math, footnote, highlight, comment, a raw-preserved Mermaid fence, and a literal-bracket escape.