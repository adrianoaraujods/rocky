# Derived vault index: flat JSON, watcher as accelerator

The vault index design, locked in issue #3: a single `.rocky/index.json` — a versioned, flat map of **every file** keyed by vault-relative path (`name`, `ext`, `kind`, `size`, `createdAt`/`updatedAt`, `title`, `aliases`, `parents`, `tag`, `folderNote`, `outLinks`) plus a name/alias **registry** for O(1) link resolution. It stays 100% derived (ADR-0001): rebuilt by a full walk + reparse, atomically swapped, deletable, never a store of user truth. The physical tree and the backlink inverse (from `outLinks`) are computed in memory and **never persisted**.

## Considered Options

- **SQLite (better-sqlite3/WAL)**: rejected for the MVP — because the index is fully rebuildable, crash safety is a wash; single-JSON rewrite at 10k files is ~1–2MB, a few-ms CPU burst. Revisit when real loads justify it.
- **LMDB**: rejected — OpenLDAP Public License distribution burden and a non-portable mmap format buy nothing here.
- **Nested JSON tree / storing the overlay**: rejected — the filesystem layout already encodes nesting, and the overlay is a pure function of `files[]` + `parents`; storing either would duplicate derivable state and drift.
- **inotify walls**: recursive `chokidar` per-directory watches hit per-user limits (8192 default on older kernels); watchman adds a persistent daemon with cookie/settle semantics; **fanotify needs `CAP_SYS_ADMIN`**. All rejected.
- **Model** (chosen): **layered bottom-up `fs.watch`/chokidar as a latency accelerator + a periodic full snapshot-and-diff sweep as the reconciler.** Ignore patterns (settings) apply before watches are created and before the index builds, so ignored paths appear nowhere. Sweep interval is a workspace setting (1–60 min slider + `never`, default 15). Setup docs bump `fs.inotify.max_user_watches=524288` for large vaults; `usePolling` is offered for network/FUSE mounts.

## Consequences

- Events dropped by inotify overruns cost staleness only, until the next sweep — mapping to Obsidian's posture (chokidar + lazy metadata-cache rebuild + "rebuild vault cache").
- `kind = note | file` now, with `drawing`/`spreadsheet`/`diagram` reserved for the deferred rich built-ins; the frontmatter `tag` flag (`tag: true`) is catalog-mirrored on note entries.
- Workspace settings move to `.rocky/settings.json` — authoritative user config, clearly outside the derived index.
- The frontmatter reserved set lost `type` in favor of `tag` (§ADR-0003), so the catalog carries kind-on-disk and tag-ness as independent axes.