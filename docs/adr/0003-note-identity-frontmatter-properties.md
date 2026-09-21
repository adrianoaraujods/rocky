# Note identity, frontmatter and properties

The identity and metadata model, locked in issue #2: a note's identity is its filename minus `.md` (no `name` key), and the filesystem is the truth — so duplicate names are legal and **unique note names become a workspace setting** (warn by default, optional enforce), revising ADR-0001. Frontmatter carries a fixed set of lowercase reserved keys — `title`, `aliases`, `parent`, `type`, `cssclasses`, `foldernote` — rendered capitalized in the UI; any other key is a free Trilium-style attribute. Folder notes are `X/X.md` inside folder `X`, auto-recognized with an in-file opt-out; tags are notes marked `type: tag`, linked by ordinary wikilinks.

## Considered Options

- **Enforced unique names per vault (ADR-0001)**: revised during grilling — links are name-based but the filesystem already permits collisions; uniqueness becomes a setting, duplicates are reachable only by path link, and colliding `[[Name]]` shows a picker.
- **`name` frontmatter key for identity (Trilium style)**: rejected — the filename is the file's truth; files stay self-describing.
- **Obsidian `tags:` frontmatter and inline `#tag`**: rejected — tags are links to `type: tag` notes; one mechanism instead of two, and `type` stays reusable for future typed notes.
- **Capitalized reserved keys on disk**: revised — keys are lowercase in the file; the UI renders them capitalized (a disablable setting), keeping the raw file unobtrusive. `aliases` is exempt from the auto-hide of the other reserved keys.
- **Index/config flag for folder-note opt-out**: rejected — `foldernote: false` rides in the file so the opt-out survives any index rebuild (ADR-0001's derived-index principle).

## Consequences

- Duplicate filenames may exist in a vault; name autocomplete excludes them, path links reach them, and the index must record duplicates and alias collisions for the picker.
- `parent` lists become clone edges in the tree overlay; cycles are rejected at index time.
- Frontmatter parsing/schema must treat reserved keys case-insensitively as reserved; a `Title` custom attribute is forbidden.
- Filenames containing `[ ] | #` or newlines can't exist as notes (Renames etc. are refused) and link-by-name only when within the grammar.