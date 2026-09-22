# Clone model: home plus extra positions

Rocky's folders-on-disk tree needs an answer to Trilium's clones that keeps the filesystem the single source of truth (ADR-0001). Decided in issue #4: a note's file has exactly one **home** — its physical vault-relative path, with the vault root a legal home — and zero or more **clone positions** declared by `parent:` frontmatter entries. Clones are strictly extra positions derived into the tree overlay; the overlay is a pure function and the disk is never mutated for a clone.

## Considered Options

- **Trilium-style symmetric positions**: every parent equal, none special; deleting the last position leaves a detached note. Rejected — a note IS a file (ADR-0001); a file always lives somewhere, so detached notes are impossible without inventing a hidden holding area, and making the disk location incidental turns every file move into bookkeeping.
- **Detached notes via a hidden store or root-hiding**: hiding vault-root files behind an "unfiled" tab was mooted. The root being a legal home already captures the unfiled pattern; the hiding/tab idea is post-MVP fog.
- **Position-symmetric single delete verb**: delete anywhere removes that position and promotes a remaining clone to home. Rejected — it smuggles a move into a delete and surprises the disk layout.
- **Folder clones in the MVP**: subtree-mirror semantics (each descendant file gains its mirrored-folder parent) are decided and reusable, but folder cloning is ruled out of the MVP's scope (cross-checked by the MVP cut, #5).

## Consequences

- **Two delete verbs**: *remove clone* — rewrite a `parent:` entry (home stays) — is available on a clone position whenever ≥2 positions exist; *delete note* — from the home — removes the physical file and with it every position, since the file is one thing.
- **Links never address clone positions**: `[[Name]]` resolves through the registry, `/path` links target the physical file, autocomplete lists each note once; clones exist only in the tree overlay.
- **Re-homing dedupes**: moving a file into a folder that is already a `parent:` target drops the now-redundant clone entry with no dialog.
- A note's **title is note-level**: one `title:` per file, shown at every position; you cannot give a clone its own display name.
- Uniqueness (workspace setting, contract §3) gates names, never clone positions; `parent:` cycles are rejected (contract §2).