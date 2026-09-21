# Local files are the truth

Storage model for Rocky, locked in charting: the vault is a plain folder tree whose notes are real Markdown files; the `.rocky/` directory holds only a **derived** index (file catalog, note-name registry, clone edges, link edges) rebuilt from the filesystem, never authoritative. This replaces both Obsidian's path-fragile linking and Trilium's database: users get Trilium's tree/clones/attributes with files that survive the app dying. Context: three prior Rocky attempts (rocky_v0/v1/v2) stalled; the deciding constraint is brutal simplicity and "my files still work if I quit".

## Considered Options

- **Trilium's database storage**: rejected — files must own the content.
- **Obsidian's folders + duplicate names + path links**: rejected — a free-form tree with unique note names is the product.
- **Flat pool + JSON tree (approach (b))**: rejected — the hierarchy would live only in app-owned JSON, violating Quit-App-Still-Works; also rejects broad-folder vaults.
- **Symlink clones**: rejected with data — Android blocks symlinks in user storage (FUSE/EPERM), iOS has no API to create them in a picked folder, and Dropbox/OneDrive/Google Drive/Nextcloud dereference or drop links, silently turning clones into diverging copies. git on Windows checks symlinks out as plain text.
- **Clones as index edges**: chosen — this is literally Trilium's model (a `branches` table) moved out of the database into a portable JSON file.

## Consequences

- Links are name-based, but the filesystem is truth: note-name uniqueness is a workspace setting (warn by default, optional enforce), not an app guarantee — see ADR-0003. The index records duplicates and alias collisions so colliding `[[Name]]` links resolve via a picker.
- Any external edit to the folder tree needs a rescan to refresh the index; index staleness surfaces as re-routable broken links, not corruption.
- The tree overlay (folders + clone edges) is a UI composition, not new filesystem structure.