# Rust tooling, CI and Linux packaging

Research ticket: [Rust tooling, CI and Linux packaging](https://github.com/adrianoaraujods/rocky/issues/32) (GH #32), part of map [#28](https://github.com/adrianoaraujods/rocky/issues/28). As of 2026-11-15, this sets out the project tooling, quality gates, test strategy, CI and Linux distribution for Rocky as a Linux-first Rust desktop app. Rocky is a three-crate Cargo workspace:

- `rocky-core`: no UI.
- A publishable, Rocky-agnostic editor crate (MIT OR Apache-2.0, on crates.io).
- The `rocky` app (AGPL-3.0-or-later).

All code is written by AI agents. The driver runs Arch Linux (Omarchy). The UI framework is still open (Tauri / Dioxus / iced / egui / GPUI / Slint), so every framework-dependent point is marked **[FW]**.

Sources are primary: the Cargo, rustup and Clippy docs, tool READMEs and docs, crates.io API metadata, GitHub release metadata, Flathub and freedesktop specs. Versions were read from the crates.io and GitHub APIs on 2026-11-15. No application code was written.

## Version snapshot (2026-11-15)

| Tool | Latest | Released | Source |
|---|---|---|---|
| Rust stable | 1.100.0 | 2026-11-12 | https://github.com/rust-lang/rust/releases |
| cargo-nextest | 0.9.152 | 2026-11-13 | https://crates.io/crates/cargo-nextest |
| insta | 1.49.0 | 2026-10-03 | https://crates.io/crates/insta |
| proptest | 1.12.0 | 2026-10-10 | https://crates.io/crates/proptest |
| cargo-deny / cargo-deny-action | 0.20.2 / v2.1.1 | 2026-07 | https://crates.io/crates/cargo-deny, https://github.com/EmbarkStudios/cargo-deny-action |
| cargo-shear | 1.14.1 | 2026-11-13 | https://crates.io/crates/cargo-shear |
| cargo-machete | 0.9.2 | 2026-04-15 | https://crates.io/crates/cargo-machete |
| cargo-udeps | 0.1.61 | 2026-04-29 | https://crates.io/crates/cargo-udeps |
| Swatinem/rust-cache | v2.9.3 | 2026-10-20 | https://github.com/Swatinem/rust-cache/releases |
| sccache / sccache-action | 0.19.0 / v0.0.11 | 2026-10 / 2026-07 | https://github.com/mozilla/sccache, https://github.com/Mozilla-Actions/sccache-action |
| taiki-e/install-action | v2.90.5 | 2026-11-15 | https://github.com/taiki-e/install-action |
| release-plz | 0.4.3 | 2026-11-02 | https://github.com/release-plz/release-plz/releases |
| cargo-release | 1.1.7 | 2026-10-21 | https://crates.io/crates/cargo-release |
| cargo-semver-checks | 0.51.0 | 2026-10-03 | https://crates.io/crates/cargo-semver-checks |
| git-cliff | v2.14.2 | 2026-09-18 | https://github.com/orhun/git-cliff/releases |
| cargo-dist (`dist`) | 0.34.0 | 2026-11-02 | https://github.com/axodotdev/cargo-dist/releases |
| cargo-packager | 0.11.8 | **2025-11-27** (a year stale) | https://github.com/crabnebula-dev/cargo-packager |
| tauri-bundler | 2.10.4 | 2026-11-06 | https://crates.io/crates/tauri-bundler |
| cargo-deb | 3.8.0 | 2026-09-01 | https://crates.io/crates/cargo-deb |

## Naming collisions to resolve before anything ships

These were checked against the crates.io and AUR APIs. They affect workspace layout, release-plz config and packaging, so they need a decision early.

- **crates.io `rocky`** is taken: a 2015 web framework owned by another user. **crates.io `rocky-core`** is also taken, by an unrelated project (`BierDav/rocky`) publishing nightlies in November 2026. Neither matters while the app and core use `publish = false`. The **editor crate needs a unique, Rocky-agnostic name** anyway, which fits its intended design.
  - https://crates.io/api/v1/crates/rocky
  - https://crates.io/api/v1/crates/rocky-core
- **AUR `rocky`** is taken (a C++ map-rendering SDK). The AUR package needs another name, for example `rocky-notes` / `rocky-notes-bin` / `rocky-notes-git`. Source: https://aur.archlinux.org/rpc/v5/search/rocky?by=name
- **The desktop/Flatpak app ID** must use reverse DNS. The freedesktop spec says the desktop entry name "should start with a reversed DNS domain name controlled by the author of the application". Flathub requires the `io.github.` prefix for GitHub-hosted apps. Without a custom domain, the ID is therefore **`io.github.adrianoaraujods.Rocky`**. It must exactly match the MetaInfo `<id>`.
  - https://specifications.freedesktop.org/desktop-entry/latest/file-naming.html
  - https://docs.flathub.org/docs/for-app-authors/requirements

## 1. Workspace layout and conventions

**Layout.** Use a virtual workspace (no root `[package]`) with members under `crates/`:

```
Cargo.toml            # [workspace], [workspace.package], [workspace.dependencies], [workspace.lints]
rust-toolchain.toml
deny.toml  release-plz.toml  cliff.toml  .config/nextest.toml
crates/rocky-core/    # publish = false
crates/<editor>/      # publish = true, MIT OR Apache-2.0
crates/rocky/         # the app binary, publish = false, AGPL-3.0-or-later
packaging/            # PKGBUILD, .desktop, metainfo, icons, flatpak manifest
tests/fixtures/vaults/  # fixture vaults shared by core + app tests
```

**Inheritance.** Members can inherit any of `authors, categories, description, documentation, edition, exclude, homepage, include, keywords, license, license-file, publish, readme, repository, rust-version, version` from `[workspace.package]`. Use it for `edition`, `rust-version`, `repository` and `authors`. Do **not** inherit `license`, because the crates differ deliberately; set it per crate. Put every dependency version in `[workspace.dependencies]` and have members write `foo.workspace = true`. Two caveats:

- Workspace dependency entries cannot be `optional`.
- Member `features` add to the workspace's features. A member's `default-features = false` only overrides the workspace entry in Edition 2024 on Rust 1.99+. Before that, the workspace entry itself had to disable defaults.

Lints go in `[workspace.lints]`, and each member opts in with `[lints] workspace = true` (Rust 1.74+). A virtual workspace must set `resolver` explicitly; use `"3"`.

- https://doc.rust-lang.org/cargo/reference/workspaces.html

**Edition.** Use **2024**, stable since Rust 1.85 (Feb 2025). No 2027 edition has been announced or stabilized. Internals discussion still treats its timing and contents as open. Re-evaluate when one is announced.

- https://doc.rust-lang.org/edition-guide/
- https://internals.rust-lang.org/t/we-have-only-two-years-to-prepare-the-rust-2027-edition/22494

**MSRV policy.** The Cargo book lists these policy options:

- "latest only" ("the simplest policy")
- N-2
- even releases
- calendar-year

It says to pick one and document it. For the app and core, use **latest only**: they are not consumed as libraries, and agents benefit from new std APIs. For the published editor crate, use **N-2**, a common courtesy to downstream users. Both are expressed with `rust-version`. Clippy reads `rust-version` and does not fire suggestions that need a newer compiler (`incompatible_msrv`). Cargo's MSRV-aware resolver (resolver 3) picks dependency versions compatible with `rust-version`.

Within a workspace, all members must still load on the oldest supported toolchain. In practice, set workspace `rust-version` to the editor crate's N-2 value and let the app use newer std behind nothing more than that, or override `rust-version` on the app crate only. A CI job that builds the editor crate on its `rust-version` toolchain makes the claim checkable ("Verified" support level).

- https://doc.rust-lang.org/cargo/reference/rust-version.html
- https://raw.githubusercontent.com/rust-lang/rust-clippy/master/README.md

**Pinned toolchain.** Commit a `rust-toolchain.toml` with an exact version (`channel = "1.100.0"`) and `components = ["rustfmt", "clippy"]`. rustup honours this file automatically for the driver locally and on CI. `actions-rust-lang/setup-rust-toolchain` also reads it: "If a toolchain file … is found in the root of the repository … all items specified in the toolchain file will be installed". Pinning makes agents' local `cargo clippy` match CI exactly. Bumping is a one-line PR that a scheduled job (Renovate/Dependabot or a cron workflow) can open every six weeks.

- https://rust-lang.github.io/rustup/overrides.html
- https://raw.githubusercontent.com/actions-rust-lang/setup-rust-toolchain/main/README.md

**Caveat for Arch packaging.** Arch's Rust package guidelines build with the system `rust` package (`RUSTUP_TOOLCHAIN=stable` in PKGBUILDs). The PKGBUILD should override the pin, so the pin should never be newer than Arch's `extra/rust`. Arch tracks stable closely.

## 2. Quality gates

**rustfmt.** Run `cargo fmt --all --check` with a `rustfmt.toml` holding `style_edition = "2024"` and nothing exotic. Agents should run it before committing.

**Clippy.** Clippy's own groups are:

- `correctness`: deny by default.
- `suspicious`, `style`, `complexity`, `perf`: warn by default.
- `pedantic`, `restriction`, `nursery`, `cargo`: allow by default.

Clippy says "The `restriction` category should, *emphatically*, not be enabled as a whole." It describes pedantic as "rather strict or have occasional false positives". For an all-agent codebase, a strict but non-noisy baseline in `[workspace.lints]` is:

```toml
[workspace.lints.rust]
unsafe_code = "deny"            # forbid in the editor crate if it needs no FFI
missing_docs = "warn"           # editor crate is public API
unreachable_pub = "warn"

[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "warn", priority = -1 }
# cherry-picked restriction lints that catch typical agent shortcuts:
unwrap_used = "deny"            # allow in tests via clippy.toml allow-unwrap-in-tests
expect_used = "warn"
dbg_macro = "deny"
todo = "deny"
print_stdout = "warn"
undocumented_unsafe_blocks = "deny"
```

Then make warnings fatal on CI only, not locally, so agents can iterate. Use `cargo clippy --workspace --all-targets --all-features -- -D warnings`, or Cargo's new `build.warnings = "deny"` / `CARGO_BUILD_WARNINGS=deny` (respected as of Cargo 1.97). `setup-rust-toolchain` already defaults `build-warnings: deny`.

- https://raw.githubusercontent.com/rust-lang/rust-clippy/master/README.md
- https://doc.rust-lang.org/cargo/reference/config.html

**cargo-deny (licences, advisories, bans, sources).** Licence checking covers workspace members by default. Unpublished members are skipped only with `[licenses.private] ignore = true`. GNU licences match strictly, so `AGPL-3.0-or-later` must be written exactly. Recommended shape:

- `allow` = the permissive set only: MIT, Apache-2.0, Apache-2.0 WITH LLVM-exception, BSD-2/3-Clause, ISC, Zlib, Unicode-3.0, MPL-2.0 (file-level copyleft, acceptable for both).
- `exceptions = [{ crate = "rocky", allow = ["AGPL-3.0-or-later"] }]`, plus per-crate exceptions for any LGPL/GPL dependency that **only the app** uses.

This keeps copyleft out of the global allow list, so a copyleft dependency can never quietly enter the editor crate's graph. To make that guarantee explicit, add a second CI invocation scoped to the editor crate with a stricter config:

```
cargo deny --manifest-path crates/<editor>/Cargo.toml check licenses --config deny-editor.toml
```

Also enable:

- `advisories` (RustSec)
- `bans` with `multiple-versions = "warn"` (GUI stacks pull duplicate windowing crates; warn, don't fail)
- `sources` restricted to crates.io

Run it with `EmbarkStudios/cargo-deny-action@v2`.

- https://embarkstudios.github.io/cargo-deny/checks/licenses/cfg.html

**Unused dependencies: use cargo-shear, not udeps.** cargo-shear parses sources with rust-analyzer's parser on stable and has `--fix`. Its README describes the alternatives:

- cargo-machete "collects dependency usage by running regex patterns" and "does not detect all usages".
- cargo-udeps compiles the project on nightly, "does not work with cargo workspaces", and "no longer seems to work with recent versions of cargo".

cargo-shear is also the most actively released of the three (2026-11-13). cargo-machete stays a reasonable fallback, with `[package.metadata.cargo-machete] ignored = [...]` for false positives.

- https://raw.githubusercontent.com/Boshen/cargo-shear/main/README.md
- https://raw.githubusercontent.com/bnjbvr/cargo-machete/main/README.md

**Semver for the editor crate.** Use cargo-semver-checks, which release-plz runs automatically for library packages (see §6).

## 3. Testing

**Runner: nextest plus `cargo test --doc`.** nextest claims "Up to 3x as fast as cargo test" through process-per-test isolation. It also offers JUnit output, retries, partitioning, archiving and setup scripts. Its limit: "Doctests are currently not supported because of limitations in stable Rust." CI therefore runs `cargo nextest run --workspace --all-features` **and** `cargo test --doc --workspace`. Process-per-test isolation also helps with GUI and global-state tests (one event loop per process). Keep a `ci` profile in `.config/nextest.toml` with `retries = 0`, `fail-fast = false` and JUnit output. Don't retry: flaky tests should fail loudly so agents fix them.

- https://nexte.st/

**Snapshot tests (insta) for parser and index output.** Use `assert_snapshot!` / `assert_debug_snapshot!`, or `assert_json_snapshot!` with the `json` feature, on:

- the parsed Markdown AST / link extraction
- the serialized vault index (catalog, registry, clone edges, link edges)

`glob!` (the `glob` feature) "executes a closure for all input files matching a glob". That gives one snapshot per fixture file under `tests/fixtures/`, so adding a fixture adds a test with no new code. Use `redactions` for mtimes and absolute paths in index output. On CI, `INSTA_UPDATE=auto` behaves as `no` (no snapshot writes), so a missing or changed snapshot fails. Agents run `cargo insta test --review` / `--accept` locally, and the diff of `.snap` files is the reviewable artifact in the PR. That is especially valuable when code is AI-written.

- https://docs.rs/insta/latest/insta/

**Fixture vaults.** Commit small vaults under `tests/fixtures/vaults/<case>/`: wikilinks, clones, frontmatter edge cases, CRLF, non-UTF-8 names, deep trees. Tests copy a fixture into a `tempfile::TempDir` before mutating it. Never write in place: ADR-0001 makes files the truth, so tests must prove writes are byte-exact on real files. The same fixtures feed insta (`glob!`) and integration tests in `rocky-core`.

**Property tests (proptest) for byte-exact edits.** proptest 1.12 is current. The right properties are:

- *round-trip*: parse → serialize == input bytes, for arbitrary Markdown-ish strings plus a curated grammar strategy.
- *edit locality*: applying an edit changes only the bytes inside its range.
- *rename / link rewrite*: rewrite then reverse rewrite == original.

Commit `proptest-regressions/` files so shrunk failures become permanent regression cases.

- https://crates.io/crates/proptest
- https://proptest-rs.github.io/proptest/

**Headless GUI testing [FW].** The architectural rule that works for every framework: keep all behaviour in `rocky-core` and the editor crate's model (pure functions over buffers and indexes), and test it without any UI. That covers the bulk of the logic, and it is where agents get the fastest feedback. Framework-level options for the thin UI layer differ sharply:

| Framework | Headless test story | Source |
|---|---|---|
| egui | `egui_kittest` 0.37: AccessKit-driven `Harness`. Query widgets by label, click, run until settled. Optional wgpu image snapshots (`snapshot`+`wgpu` features) with per-OS tolerances in `kittest.toml`. Image snapshots need a GPU or software Vulkan on CI. | https://docs.rs/egui_kittest/latest/egui_kittest/ |
| iced | `iced_test` 0.14: "testing iced applications in headless mode". The `Simulator` has click/typewrite, selectors by text, snapshot comparison, and an `Emulator` for whole apps. | https://docs.rs/iced_test/latest/iced_test/ |
| Slint | `i-slint-backend-testing`: `init_no_event_loop()` for unit tests, `ElementHandle::find_by_accessible_label`, clicks. The API is documented as preliminary and needs `SLINT_EMIT_DEBUG_INFO=1`. | https://docs.rs/i-slint-backend-testing |
| Tauri | E2E through WebDriver (`tauri-driver`, Linux and Windows only; uses WebKitWebDriver on Linux), plus IPC mocking. The web frontend gets its own JS test stack. Heaviest option: needs a display (xvfb) on CI. | https://v2.tauri.app/develop/tests/webdriver/ |
| GPUI | `#[gpui::test]` with `TestAppContext` / `VisualTestContext` and a deterministic executor (`run_until_parked`). Documented mostly in Zed's source, not user docs. | https://docs.rs/crate/gpui/0.2.2/source/src/app/test_context.rs |
| Dioxus | Not researched in depth. Its desktop renderer is a webview (like Tauri), so E2E is WebDriver-shaped. | — |

egui and iced have the strongest first-party, in-process, headless story. Tauri's is weakest and splits testing across two languages.

## 4. CI on GitHub Actions

**Runners.** `ubuntu-latest` is Ubuntu 24.04. Ubuntu 26.04 (`ubuntu-26.04`, `ubuntu-26.04-arm`) is also available but is not yet what `-latest` points to. Pin `ubuntu-24.04` explicitly so a silent `-latest` flip cannot break the GUI system-package list.

- https://github.com/actions/runner-images

**Toolchain and caching.** `actions-rust-lang/setup-rust-toolchain` reads `rust-toolchain.toml` and wires `Swatinem/rust-cache` in automatically (`cache: true` by default). Alternatively, use `dtolnay/rust-toolchain` + `Swatinem/rust-cache@v2` directly. rust-cache:

- caches `~/.cargo` (registry and git; not `registry/src`) and dependency artifacts in `target/`, but not workspace crates
- sets `CARGO_INCREMENTAL=0`

Use `save-if: ${{ github.ref == 'refs/heads/main' }}` so PRs restore but don't churn the 10 GB repo cache, and `cache-on-failure: true`.

- https://raw.githubusercontent.com/Swatinem/rust-cache/master/README.md

**sccache is not part of the baseline.** sccache-action uses the GHA cache with `SCCACHE_GHA_ENABLED=true` and `RUSTC_WRAPPER=sccache`. It helps most when many jobs compile overlapping crates or when target caches are evicted. For one workspace with a handful of jobs, rust-cache alone is simpler, and stacking both doubles cache traffic. Revisit only if cold builds (GUI stacks like wgpu, webkit bindings and tauri compile hundreds of crates) exceed about 10 minutes.

- https://raw.githubusercontent.com/Mozilla-Actions/sccache-action/main/README.md

**Tool installs.** Use `taiki-e/install-action` for nextest, cargo-deny, cargo-shear, cargo-insta and cargo-semver-checks (prebuilt binaries, seconds instead of `cargo install` minutes).

- https://github.com/taiki-e/install-action

**System packages for a GUI build on Ubuntu [FW]:**

- **Tauri / Dioxus-desktop (WebKitGTK):** `libwebkit2gtk-4.1-dev build-essential curl wget file libxdo-dev libssl-dev libayatana-appindicator3-dev librsvg2-dev`. The Arch equivalent is `webkit2gtk-4.1 base-devel … libappindicator-gtk3 librsvg xdotool`. https://v2.tauri.app/start/prerequisites/
- **egui/eframe:** `libclang-dev libgtk-3-dev libxcb-render0-dev libxcb-shape0-dev libxcb-xfixes0-dev libxkbcommon-dev libssl-dev`. The GTK dependency is for native file dialogs via rfd. https://raw.githubusercontent.com/emilk/egui/main/README.md
- **iced (winit+wgpu):** runtime-loaded `libxkbcommon`, `vulkan-loader`, `wayland` (X11 fallback: `libX11`, `libXcursor`, `libXi`). These are mostly dlopened, so the build needs few `-dev` packages. https://raw.githubusercontent.com/iced-rs/iced/master/DEPENDENCIES.md
- **Slint / GPUI:** not verified here. Both need at least xkbcommon/wayland/fontconfig. Check their install docs once the framework is chosen.

Install them with a single `apt-get install` step, or cache the .debs with `awalsh128/cache-apt-pkgs-action`. Keep the package list in one file (`ci/apt-packages.txt`) so packaging and CI share it.

**Keeping CI fast.** Proposed job graph:

1. `fmt`: no compile, no cache, about 20 s.
2. `lint`: clippy + `cargo deny` + `cargo shear`. A `check`-profile build, sharing one rust-cache key.
3. `test`: `cargo nextest run` + `cargo test --doc`, plus `cargo insta test --check` semantics via `INSTA_UPDATE=no`.
4. `msrv`: `cargo check -p <editor>` on its `rust-version` toolchain, editor crate only, so no GUI deps.
5. `package`: only on tags or on `main`.

Further speed-ups:

- Use `concurrency: { group: ${{ github.workflow }}-${{ github.ref }}, cancel-in-progress: true }`.
- Gate GUI-heavy jobs with path filters: `rocky-core` and the editor crate must not depend on the UI framework, so a core-only change never compiles webkit or wgpu.
- Set `[profile.dev] debug = "line-tables-only"` and `[profile.dev.package."*"] opt-level = 1` for smaller caches.

**DCO.** CONTRIBUTING.md is referenced as requiring DCO sign-off, but it is not on `main` yet. Enforce sign-off with the DCO GitHub App or a `git log --format=%B | grep Signed-off-by` check in CI, and add CONTRIBUTING.md to the bootstrap ticket.

## 5. Linux distribution

**Tool landscape:**

- **cargo-dist (`dist` 0.34)** makes shell, powershell, npm, homebrew and msi installers only. There is no .deb, AppImage or AUR output, and Flatpak is listed as a "future installer". It is aimed at CLIs, so it is **not a fit** for a desktop app. https://axodotdev.github.io/cargo-dist/book/installers/index.html
- **cargo-packager** produces .deb, AppImage and pacman (`.tar.gz` + PKGBUILD), plus an updater crate. But its last release was 2025-11-27, a year ago, so treat it as **at risk**. https://github.com/crabnebula-dev/cargo-packager
- **Tauri's bundler [FW]** produces `deb`, `rpm` and `appimage` on Linux. Tauri also documents guides (not bundler outputs) for Flatpak/Flathub, Snap and AUR. This is the strongest option, but only if the app is Tauri. https://v2.tauri.app/reference/config/, https://v2.tauri.app/distribute/
- **cargo-deb 3.8** is a framework-agnostic .deb builder driven by `[package.metadata.deb]`. Use it if a .deb is wanted without Tauri. https://crates.io/crates/cargo-deb

**Recommended order for a Linux-first, Arch-driven app:**

1. **AUR first.** The driver is on Arch. Ship `packaging/arch/PKGBUILD` in-repo and publish `rocky-notes-git` (builds `main`) and later `rocky-notes` (tagged source tarball). AUR `rocky` is taken. Arch's Rust guidelines:
   - `prepare()`: `cargo fetch --locked --target "$(rustc -vV | sed -n 's/host: //p')"` (older snapshots use `"$CARCH-unknown-linux-gnu"`).
   - `build()`: `cargo build --frozen --release` (`--frozen` = `--locked --offline`).
   - `check()`: `cargo test --frozen`.
   - `export RUSTUP_TOOLCHAIN=stable` and `CARGO_TARGET_DIR=target`.
   - `makedepends=(cargo)` plus the [FW] system libraries as `depends`.

   Sources: https://wiki.archlinux.org/title/Rust_package_guidelines. The Arch wiki was bot-blocked when fetched for this research; the flags were confirmed through the `cargo-fetch(1)` man page at https://man.archlinux.org/man/cargo-fetch.1.en and Arch packaging commits. Verify the current wiki text when writing the PKGBUILD. A CI job can run `makepkg` in an `archlinux:base-devel` container to keep the PKGBUILD green.
2. **Flatpak/Flathub second**, for every other distro. It needs:
   - the app ID `io.github.adrianoaraujods.Rocky`
   - a desktop file, an SVG (or ≥256×256 PNG) icon, and a validated MetaInfo file whose `<id>` matches
   - an offline build ("There is no network access during the build process")

   For Rust, the offline build means generating `cargo-sources.json` from `Cargo.lock` with `flatpak-cargo-generator.py` and building with `org.freedesktop.Sdk.Extension.rust-stable` matching the runtime branch. A vault app needs filesystem access to arbitrary folders. Plan on the portal file chooser (`org.freedesktop.portal.FileChooser`) or a `--filesystem=home` permission, and justify it in the Flathub review. That is a real design constraint on the vault-open flow. Its interaction with the file watcher (ADR-0004) is not verified here.
   - https://docs.flathub.org/docs/for-app-authors/requirements
   - https://raw.githubusercontent.com/flatpak/flatpak-builder-tools/master/cargo/README.md
3. **AppImage / .deb: optional, and free if Tauri** (bundler targets). Otherwise skip them until someone asks. Flatpak covers non-Arch users with less maintenance.

**Desktop integration files** (framework-agnostic, kept in `packaging/`):

- `io.github.adrianoaraujods.Rocky.desktop`: reverse-DNS name per the Desktop Entry spec, with `Exec=rocky %U`, `MimeType=text/markdown;` and `Categories=Office;TextEditor;`.
- `io.github.adrianoaraujods.Rocky.svg`, installed to `/usr/share/icons/hicolor/scalable/apps/`, plus PNG sizes if wanted.
- `io.github.adrianoaraujods.Rocky.metainfo.xml`, installed to `/usr/share/metainfo/`, with `<project_license>AGPL-3.0-or-later</project_license>`.

Validate them in CI with `desktop-file-validate` and `appstreamcli validate`.

The Wayland `app_id` / X11 `WM_CLASS` the window sets must equal the desktop-file basename, or Hyprland/GNOME won't match the window to its icon and entry. How you set it is **[FW]**: winit `with_name`, a Tauri config field, or a GTK application ID.

- https://specifications.freedesktop.org/desktop-entry/latest/file-naming.html

## 6. Publishing the editor crate

**Use release-plz (CI-driven), not cargo-release (local CLI).** release-plz opens a release PR on every push to `main`. It derives version bumps from conventional commits, writes changelogs with git-cliff (Keep a Changelog format) and checks API breakage with cargo-semver-checks. On merge, it publishes, tags and creates GitHub releases. That matches agent-written code: the release is a reviewable PR, not a command someone has to remember. Per-package config does what Rocky needs:

```toml
[workspace]
changelog_update = false
git_release_enable = false
publish = false

[[package]]
name = "<editor>"
publish = true
changelog_update = true
git_release_enable = true
semver_check = true

[[package]]
name = "rocky"          # app: tag + GitHub release (packaging hooks off this), no crates.io
release = true
publish = false
changelog_update = true
git_release_enable = true
```

`publish = false` "skips `cargo publish` … but still creates git tags". `release = false` drops a package entirely (for `rocky-core`).

- https://release-plz.dev/docs
- https://release-plz.dev/docs/config

**Auth: crates.io Trusted Publishing.** Announced 2025-07-11, GitHub-Actions-only, via `rust-lang/crates-io-auth-action@v1` with OIDC (`permissions: id-token: write`), so no long-lived `CARGO_REGISTRY_TOKEN` secret. The first publish must be done manually, then the repo is registered as trusted on crates.io.

- https://blog.rust-lang.org/2025/07/11/crates-io-development-update-2025-07/

**Prerequisite: conventional commits.** release-plz's bumping depends on them. The repo already uses `docs:` / `chore:` prefixes. Make the rule explicit in AGENTS.md, and optionally lint PR titles. cargo-release 1.1.7 remains a fine manual fallback, but it adds no CI-native workflow.

## Recommended baseline

1. **Workspace:** virtual, `crates/{rocky-core,<editor>,rocky}`, `resolver = "3"`, edition 2024. Shared `[workspace.package]` / `[workspace.dependencies]` / `[workspace.lints]`, per-crate `license`.
2. **Toolchain:** `rust-toolchain.toml` pinned to `1.100.0` + rustfmt + clippy. Editor crate MSRV is N-2, checked in CI. App and core use latest-only.
3. **Gates:** `cargo fmt --check`; clippy with `all = deny`, `pedantic = warn` and cherry-picked restriction lints (`unwrap_used`, `dbg_macro`, `todo`, `undocumented_unsafe_blocks`), warnings fatal on CI only; `cargo deny` with a permissive allow list, an AGPL exception for `rocky` only and a stricter editor-only licence check; `cargo shear`.
4. **Tests:** nextest + `cargo test --doc`; insta + `glob!` over committed fixture vaults; proptest round-trip and edit-locality properties with committed regressions; nearly all behaviour tested UI-free in core and editor.
5. **CI:** GitHub Actions on `ubuntu-24.04`, `setup-rust-toolchain` (built-in rust-cache, save on `main` only), `taiki-e/install-action`, concurrency cancel, path-filtered GUI jobs. No sccache until cold builds hurt.
6. **Distribution:** in-repo PKGBUILD → AUR `rocky-notes-git` / `rocky-notes`, then Flathub `io.github.adrianoaraujods.Rocky`. Desktop file, SVG icon and MetaInfo validated in CI. Avoid cargo-dist (CLI-only) and cargo-packager (stale).
7. **Release:** release-plz with git-cliff and semver-checks. The editor crate is published via crates.io Trusted Publishing; the app is tagged and GitHub-released only. Requires conventional commits.
8. **Names:** pick an editor crate name now; `rocky` and `rocky-core` are taken on crates.io and `rocky` on the AUR.

## Parts that depend on the UI framework [FW]

- **Ubuntu/Arch system packages** for CI and PKGBUILD `depends`: WebKitGTK 4.1 + appindicator + librsvg (Tauri, Dioxus-desktop) versus GTK3/xkbcommon/xcb (egui) versus xkbcommon/wayland/vulkan-loader (iced).
- **Headless GUI test tool:** egui_kittest or iced_test (in-process, first-party, best), Slint testing backend (preliminary), GPUI `TestAppContext` (sparsely documented), Tauri/Dioxus WebDriver + xvfb (heaviest, two-language).
- **Packaging tool:** Tauri's bundler gives deb/rpm/AppImage for free. Every other framework needs cargo-deb or appimagetool by hand, or relies on AUR + Flatpak only (recommended anyway).
- **Window `app_id` / `WM_CLASS` wiring** to match the desktop-file ID.
- **Flatpak runtime choice:** GNOME runtime (WebKitGTK provided) for webview frameworks versus the plain freedesktop runtime for wgpu/winit frameworks. This affects bundle size and Flathub review.
- **CI build time,** and therefore whether sccache becomes worth adding: wgpu- and webkit-based stacks are the slowest cold builds.
- **Lint baseline:** Tauri adds a JS/TS frontend that needs its own lint, format and test toolchain alongside Rust. That doubles the gates agents must satisfy.
