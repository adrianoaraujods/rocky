# Rocky

Rocky is an open-source, self-hosted web app for a personal knowledge base: Trilium Notes' structure (a free-form tree, clones, attributes) stored the way Obsidian users expect — plain Markdown files that stay yours even if the app goes away.

## Language

**Vault**:
A root directory opened in Rocky. The folder tree on disk IS the hierarchy. A vault can be a broad folder — projects, documents, media, code — so notes can link to almost anything on the filesystem.
_Avoid_: Workspace, project, database

**Note**:
A single piece of authored content made of one Markdown file (`.md`) inside the vault. The note's identity is its name. Several files may share a name on disk (the filesystem is the truth); unique names are a workspace setting.
_Avoid_: Node, document, entry

**Note name**:
The identifier of a note, equal to its filename (minus `.md`). Uniqueness across the vault is a workspace setting (default: warn on collision); duplicate names are reachable through path links only. Links type the name, not a path or an ID.
_Avoid_: Title (a title is displayed text; a note name is identity), slug

**Title**:
The `title` frontmatter property, display text that overrides the filename in the tree and UI. A title is never a link target and never part of identity.
_Avoid_: Note name (a note name is identity; a title is how it's shown)

**Alias**:
An additional name a note may expose through the `aliases` frontmatter list. Aliases are allowed to conflict with each other; a conflicting link is disambiguated by the user at click time. The first property shown, never auto-hidden.
_Avoid_: Nickname, synonym

**Clone**:
A note referenced from two or more places in the tree. A clone is not a copy — the note keeps one identity and one file, and appears under several parents. Clones are expressed as the `parent` frontmatter property (list of folder paths) and/or index edges, never as filesystem symlinks.
_Avoid_: Symlink, copy, duplicate, alias

**Tag**:
A Markdown note marked `type: tag`. Other notes tag it with an ordinary wikilink `[[Name]]` — there is no inline `#tag` syntax. A tag note's backlinks are the tagged notes.
_Avoid_: Label (a label is a custom attribute), embedded tag

**Folder note**:
The Markdown file `X/X.md` inside a folder and sharing its name, auto-recognized at index time as that folder's representative (opt-out: `foldernote: false`). `[[X]]` resolves to it.
_Avoid_: Index page (a folder note is a file, not an app construct)

**Path link**:
A `[[path/to/file.ext]]`-style reference to a file that is not a note (an image, a video, code, a document). Path links address the file system directly and are inherently unique.
_Avoid_: Attachment, resource

**Wikilink**:
A `[[Name]]` or `[[Name|display]]` reference to a note by its note name. Wikilinks are plain text in the source file, human-readable outside the app; they resolve through the note-names registry, not through paths, so a note can move without breaking its links.
_Avoid_: Backlink (a backlink is a derived view, not a stored reference)

**Attribute**:
A key/value (or key/string-list) field stored in a note's frontmatter, following the Trilium attribute concept. Any frontmatter key that is not a reserved property is a free attribute.
_Avoid_: Property (a reserved property is a key with engine semantics; an attribute is free-form), tag (a tag is a type of note, linked to, not a property)

**Reserved property**:
A frontmatter key with engine semantics: `title`, `aliases`, `parent`, `type`, `cssclasses`, `foldernote`. Lowercase in the file; rendered capitalized in the properties UI. Reserved keys auto-hide when the properties element is unfocused (a workspace setting), except `aliases`. A key that collides case-insensitively with a reserved property may not be used as an attribute.
_Avoid_: Attribute (see separate entry)

**Vault index**:
A derived, rebuildable catalog in the `.rocky/` directory that records every file in the vault, the note-name registry, clone edges, and wikilink/path-link edges. The filesystem remains the source of truth; the index is a cache and can be regenerated at any time.
_Avoid_: Database, store (the files are the store), cache

**Tree overlay**:
The app's view of the vault: the real folder tree illustrated with clone edges injected as extra parents, so clones appear inline in the hierarchy while the disk keeps a single file.
_Avoid_: Virtual tree

**Heading id**:
The link target of a header reference `[[Note#id]]`. Default is an auto-slug of the heading text; `## Heading [#id]` overrides it explicitly. The marker is hidden in rendering and stays in the file.
_Avoid_: Anchor (block anchors `^blockid` are out of the contract)

**Workspace setting**:
A preference scoped to the opened root folder (the vault), not global. Examples: the note-name uniqueness rule, reserved-property auto-hide, capitalized property display.
_Avoid_: Global setting (these are per-vault), config file in `.rocky/` (settings ride in the vault)

## Architecture

**Local markdown store**: Notes and their hierarchy live as real files and folders; a `.rocky/` directory holds the derived vault index and app config.