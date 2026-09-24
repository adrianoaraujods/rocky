# MVP feature cut: solve the smallest useful loop, delete to system trash

The driver's "simplify or it won't finish" constraint (three stalled prior attempts party) demands an explicit MVP boundary before any build ticket is sliced. Grilled and locked in issue #5: the MVP is a **single self-hosted server, one vault, one user** supporting browse/read → search → author, with **Live Preview + Source toggle** (ADR-0007) as the editing mode. Milestones: M1 vault core → M2 browse & find → M3 author → M4 ship.

## Considered Options

- **Clones in the MVP**: rejected. The clone model (ADR-0005) is design-complete but the machinery (overlay injection, duplicate badges, `parent:` editing) is deferred; `parent:` is **reserved-but-inert** in the MVP — ignored by the overlay, file passed through untouched. Clone machinery is the first post-MVP add-on.
- **Backlinks panel in the MVP**: rejected at near-zero cost because "simplify or it won't finish" governs; inLinks are already derived, so the panel is also a first post-MVP add-on.
- **Permanent delete with type-to-confirm**: rejected. Delete **moves to the system trash** (XDG: `$XDG_DATA_HOME/Trash/files` + `.trashinfo`, original path + timestamp preserved for OS-file-manager restore; volume-root `.Trash` fallback for cross-device vaults). A one-click confirm dialog stays (type-to-confirm avoided entirely); the confirm becomes a toggleable workspace setting when settings land post-MVP.
- **Settings UI in the MVP**: rejected — engine defaults only; `.rocky/settings.json` remains the mechanism without forms.
- **Full-text search**: rejected — the catalog index (ADR-0004) is metadata-only; MVP search is note-name/alias/title/path matching.
- **Rich built-in note types, plugins, theming, quick switcher, graph view, Reading View, multi-vault, PWA/offline, native clients, auth/multi-user**: rejected or deferred per #8/#9 and the map's out-of-scope section.

## Consequences

- A V1 user's loop: open the one configured vault, browse the physical folder tree, read any note (full ADR-0002 contract rendered), search by name/alias/title/path, create/rename/move/delete notes and folders, edit in Live Preview or Source mode, edit properties/attributes, make tag notes.
- Delete is never destructive from inside the app; a trashed path leaves the tree on the next index sweep and a restored file re-enters it automatically (derived index, ADR-0004) — Quit-App-Still-Works improves to "the app can't destroy your files".
- The four milestones (core → browse+find → author → ship) are the skeleton the map's destination (the sequenced build ticket-set) is sliced from.
- Folder rename/move/delete ARE in the MVP (contra the initial cut): renaming/moving never rewrites links (name-based resolution, ADR-0002) and the index is derived, so these are cheap; delete is the trash-based guard.