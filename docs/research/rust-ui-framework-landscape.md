# Rust desktop UI framework landscape

Research for wayfinder ticket [#29](https://github.com/adrianoaraujods/rocky/issues/29), part of the map [#28](https://github.com/adrianoaraujods/rocky/issues/28). Question: as of November 2026, which Rust desktop UI frameworks can credibly carry Rocky, an Obsidian-like app with a ~100k-file tree, a Markdown Live Preview editor, a properties panel, search and a settings UI? Linux (Wayland + X11) is the MVP target, mobile reach is a selection criterion, and every line of code is written by agents.

Every claim cites a primary source: official release notes, READMEs, docs, crates.io metadata or the framework's own repository. Versions, dates, download counts and licences were read from the crates.io API and the GitHub API on 2026-11-15. Judgements (labelled as such) are mine.

## Headline

- **No Rust-native framework ships a Live-Preview-grade rich-text editor.** Every native route means Rocky builds its own editor view (text layout, selection, IME, inline widgets) on the framework's text primitives. GPUI is the only one whose primitives already carry a production code editor (Zed).
- **The webview route is the only one that can reuse the behaviour reference directly.** `codemirror-obsidian-md` is a CodeMirror 6 package; in a webview it runs as is. Everywhere else it is a spec to re-implement.
- **Three credible options:** (A) Tauri 2 with a webview UI, (B) GPUI with gpui-component, (C) egui. Iced, Slint, Dioxus Native/Blitz, Freya, Makepad and Xilem are not credible for Rocky today, for the reasons below.

## Verdict table

| Framework | Latest (date) | Licence | Verdict |
|---|---|---|---|
| Tauri 2 (webview) | 2.12.5 (2026-11-06); 3.0 in alpha | MIT OR Apache-2.0 | **Credible (option A)** |
| Dioxus (webview desktop) | 0.7.10; 0.8 in alpha | MIT OR Apache-2.0 | Viable variant of A, more churn |
| Dioxus Native / Blitz | 0.7.10 / blitz-dom 0.2.4 | MIT OR Apache-2.0 | Not yet: beta engine |
| Leptos / Yew in Tauri | Leptos 0.8.23 (0.9 RC); Yew 0.23.0 | MIT; MIT OR Apache-2.0 | Viable as A's UI layer (Leptos), Yew weaker |
| GPUI (+ gpui-component) | gpui 0.2.2 (crates.io) / git; gpui-component 0.9.1 (2026-11-09) | Apache-2.0 | **Credible (option B)** |
| egui | 0.37.0 (2026-10-15) | MIT OR Apache-2.0 | **Credible (option C)** |
| iced | 0.14.0 (2025-12-07) | MIT | Not credible: no a11y, no mobile, slow releases |
| Slint | 1.18.1 (2026-09-21) | GPL-3.0-only OR royalty-free OR commercial | Not credible: plain-text editing, licence friction for the editor crate |
| Freya | 0.5.2 (2026-11-11) | MIT | Watch: small, no stated IME |
| Makepad | makepad-widgets 1.0.0 (2025-05-13) | MIT OR Apache-2.0 | Not credible: crates stale, weekly-rebased dev branch |
| Xilem / Masonry | 0.4.0 (2025-10-29) | Apache-2.0 | Not credible: self-declared experimental |

Sources for the table: crates.io API (`/api/v1/crates/<name>`) and each repo's releases page, read 2026-11-15.

## Per-framework findings

### Tauri 2 (webview UI) — credible

- **Status.** Tauri 2.0 stable shipped 2024-10-02; the 2.x line is at 2.12.5, with 2.12.0 on 2026-09-26 and 2.12.3 on 2026-10-10. Tauri is the most-used crate in this survey (≈18.6M downloads in the last 90 days; 112k GitHub stars). Tauri 3 is in alpha (3.0.0-alpha.6, 2026-11-06).
  - https://v2.tauri.app/blog/tauri-20/
  - https://v2.tauri.app/release/tauri/v2.12.0/ and https://v2.tauri.app/release/tauri/v2.12.3/
  - https://github.com/tauri-apps/tauri/releases
- **Text editing.** The UI is HTML/CSS/JS, so the editor is CodeMirror 6 and Rocky's behaviour reference runs unchanged. IME, selection, inline widgets and accessibility come from the browser engine. This is the only route where Live Preview is integration, not construction.
- **Scale.** A 100k-entry tree is a virtualised DOM list; long documents are CodeMirror's viewport rendering. Both are solved problems in the web stack (judgement).
- **Wayland and Linux.** On Linux the webview is WebKitGTK (`webkit2gtk`), whose version depends on the distro. Tauri's own docs carry a "Linux Graphics Issues" page: blank or white windows, the Wayland "Error 71" crash and DMABUF renderer failures, worked around with environment variables (`WEBKIT_DISABLE_DMABUF_RENDERER=1`, `__NV_DISABLE_EXPLICIT_SYNC=1`), some of which cost rendering speed. This is the main Linux risk of the route.
  - https://v2.tauri.app/reference/webview-versions/
  - https://v2.tauri.app/develop/debug/linux-graphics/
- **Escape hatch in progress.** Tauri 3.0.0-alpha.0 makes the webview runtime a build-time choice: `tauri-runtime-wry` (system webview, GTK 3) or `tauri-runtime-cef` (Chromium Embedded Framework, GTK 4). The CEF runtime pins CEF 154. A bundled Chromium would remove the WebKitGTK variance at the cost of binary size, but it is alpha.
  - https://github.com/tauri-apps/tauri/releases/tag/tauri-v3.0.0-alpha.0
  - https://github.com/tauri-apps/tauri/releases/tag/tauri-runtime-cef-v3.0.0-alpha.7
- **Mobile.** Android and iOS are first-class targets in 2.x (Android uses the system Chromium WebView, iOS uses WKWebView). The 2.0 announcement says mobile DX is "not completely" where the team wants it and not every plugin supports mobile. 2.12.0 still touches the Android Gradle scripts, so the mobile path is maintained.
- **Desktop features.** Multiple windows, native menus, tray, dialogs (plugin) and bundling to deb/rpm/AppImage are core features. Tabs and split panes are the web UI's job.
- **Agent fluency.** Highest of the survey: Tauri + a web frontend draws on the whole web corpus, and the 2.x API has been stable for two years (judgement). Rust is used for the backend (`rocky-core`), commands and IPC.
- **"Rust everywhere" cost.** The UI is TypeScript unless a Rust-to-wasm framework is used (Leptos is listed in Tauri's frontend guides; see below). The editor crate would be a CodeMirror package, not a Rust crate.
  - https://v2.tauri.app/start/

### Dioxus (webview desktop, mobile, Native/Blitz) — variant of A; Native not ready

- **Status.** Dioxus 0.7 shipped 2025-09-08; 0.7.10 is current; 0.8.0-alpha.3 landed 2026-10-16 with breaking changes (e.g. keyboard-types 0.8 moves `Key::Enter` to `Key::Named(NamedKey::Enter)`). Every minor (0.5, 0.6, 0.7) has shipped a migration guide, so agent knowledge decays across releases (judgement).
  - https://dioxuslabs.com/blog/release-070
  - https://github.com/DioxusLabs/dioxus/releases
- **Agent fluency.** 0.7 ships a first-party `llms.txt` and optional agent prompts in templates. Downloads ≈1.4M per 90 days, a tenth of Tauri's.
- **Desktop/mobile renderer.** Desktop and mobile render through wry (the same webview layer as Tauri: WebKitGTK on Linux; 0.8-alpha.3 moved to wry 0.57). So the Linux risks and the "reuse CodeMirror via JS interop" benefit are the same as option A, with the UI in Rust RSX.
- **Dioxus Native / Blitz.** A WGPU-painted HTML/CSS renderer without a webview. The 0.7 post calls Blitz "still considered a 'work in progress'" with missing CSS features and layout quirks. The Blitz README says it is "currently in a beta state", with "many bugs and missing features", and lists form controls and AccessKit as things it "intends to support". There is no JavaScript, so CodeMirror cannot run on it; there is no `contenteditable` story. Not a home for a Live Preview editor today.
  - https://github.com/DioxusLabs/blitz

### Leptos / Yew inside Tauri — viable UI layer for A

- **Leptos.** 0.8.23 (2026-10-31); 0.9.0-rc1 the same day. The 0.9 RC notes say most 0.8 apps upgrade without code changes. Tauri's docs list Leptos among supported frontend setups (with Trunk). ≈1.9M downloads per 90 days.
  - https://github.com/leptos-rs/leptos/releases/tag/v0.9.0-rc0
  - https://v2.tauri.app/start/
- **Yew.** 0.23.0 (2026-03-10); ≈0.45M downloads per 90 days, slower cadence than Leptos. Not listed in Tauri's frontend guides.
- **Fit.** Either keeps the UI in Rust while the editor stays CodeMirror via `wasm-bindgen` interop. It adds a wasm build and a JS-interop seam that agents must handle (judgement).

### GPUI (+ gpui-component) — credible

- **Status.** GPUI is Zed's UI framework. Its README says it "is still in active development … and is still pre-1.0" and warns "There will often be breaking changes between versions" and that the best way to learn it is to read Zed's source or ask on Discord. crates.io has `gpui` 0.2.2 (2025-10-22); Zed itself and gpui-component consume a pinned git snapshot. Licence Apache-2.0. Zed ships weekly (v1.28.1, 2026-11-11).
  - https://github.com/zed-industries/zed/blob/main/crates/gpui/README.md
- **gpui-component (GPUI Kit).** Longbridge's component layer, 0.9.1 (2026-11-09), ≈0.5M downloads per 90 days, 20.7k stars, used by Longbridge Pro. It claims 75+ components including a code editor (tree-sitter, LSP hooks) "stable at 200K lines", virtual lists with variable item sizes, virtualised tables "with hundreds of thousands of rows", a serialisable dock layout with draggable tabs and nested splits, Markdown/HTML rendering and AccessKit. Minor versions bump weekly (0.8.0 → 0.9.1 in three weeks). These are vendor claims, unbenchmarked.
  - https://github.com/longbridge/gpui-kit
- **Text editing.** Strongest native primitives in the survey: Zed's editor (multi-buffer, inline blocks, inlay hints, folds) is built on them. IME is supported on Wayland/X11 but still being hardened: open 2026 PRs/issues cover Wayland preedit segment highlighting, Ctrl+K swallowing IME input and Backspace repeat under CJK IME on GNOME.
  - https://github.com/zed-industries/zed/issues/65302
  - https://github.com/zed-industries/zed/pull/65847
  - https://github.com/zed-industries/zed/pull/65583
- **Licence trap.** Zed's `editor` crate is GPL-3.0-or-later (its `Cargo.toml`), so it cannot be copied into Rocky's MIT/Apache editor crate. GPUI and gpui-component are Apache-2.0 and can.
- **Wayland and Linux.** Native Wayland and X11 backends selected by feature (`wayland`, `x11`); Zed's Linux build is the proof at scale.
- **Mobile.** An experimental `gpui_ios` backend merged 2026-10-14 (UIKit, Metal, a `UITextInput` bridge for native text input). The Android PR (#61057) was closed unmerged on 2026-08-08. A `gpui_web` backend is also active. Mobile reach is therefore iOS-experimental, Android-absent.
  - https://github.com/zed-industries/zed/pull/63068
  - https://github.com/zed-industries/zed/pull/61057
- **Agent fluency.** Weak on docs, strong on code: Zed (95k stars) and gpui-component are large, real-world corpora, but the API churns and is learned from source (judgement).

### egui — credible

- **Status.** 0.37.0 (2026-10-15), roughly one minor every two to three months; ≈9.9M downloads per 90 days; MIT OR Apache-2.0. The README warns "New releases will have breaking changes" and "if you want something that doesn't break when you upgrade it, egui isn't for you (yet)". The 0.37 notes list a page of breaking changes (e.g. `Ui::id` → `Ui::scope_id`, `WidgetType` → `accesskit::Role`).
  - https://github.com/emilk/egui
  - https://github.com/emilk/egui/releases/tag/0.37.0
- **Agent fluency.** Large corpus and the most agent-oriented tooling in the survey: 0.35 added an inspection protocol and `egui_mcp`, an MCP server that lets an agent read a running app's AccessKit tree and drive it; `egui_kittest` runs headless UI tests and (0.37) records them to video.
  - https://newreleases.io/project/github/emilk/egui/release/0.35.0 (mirror of the GitHub release notes)
- **Text editing.** `TextEdit` is a plain/`LayoutJob`-styled text field. 0.35 improved IME composition display; 0.37 added double/triple-click-drag selection, X11/Wayland PRIMARY selection, AccessKit `SetValue`, and rendering of any system glyph including colour emoji. A Live Preview editor (hidden syntax, inline widgets, per-block rendering) must be custom-built on `Galley`/`LayoutJob` (judgement).
- **Scale.** Immediate mode re-runs UI code each frame; the README names layout and CPU cost on "very complex" GUIs as the drawbacks. `ScrollArea::show_rows` virtualises uniform-height lists, which suits a flattened 100k-row tree.
- **Mobile.** eframe targets Web, Linux, macOS, Windows and Android per the README; iOS is not listed.
- **Theming and desktop features.** Multi-viewport (multiple OS windows) exists; native menus do not (egui draws its own). A class-based styling system is experimental (`StyleProvider`, 0.37). Docking/tabs come from third-party crates (judgement: `egui_dock`, `egui_tiles`).

### iced — not credible for Rocky

- 0.14.0 (2025-12-07) is still the latest release eleven months on; the README says "Iced is currently experimental software" and lists Windows, macOS, Linux and the Web, not mobile, and says nothing on accessibility. 0.14 added input-method support, `text_editor` improvements, hot reloading and headless testing.
  - https://github.com/iced-rs/iced/releases/tag/0.14.0
  - https://github.com/iced-rs/iced
- COSMIC (System76) is the strongest Wayland proof, but it runs on libcosmic's iced-based toolkit with accessibility marked "Experimental", and System76's roadmap puts an iced rebase and IME/fcitx work in Epoch 2.
  - https://github.com/pop-os/libcosmic
  - https://blog.system76.com/post/cosmic-epoch-2-and-3-roadmap/
- No accessibility and no mobile rule it out against Rocky's criteria.

### Slint — not credible for Rocky

- 1.18.1 (2026-09-21); steady releases (1.15 Feb, 1.16 Apr, 1.17 Jun, 1.18 Sep 2026). Android and iOS are supported (iOS a tech preview since 1.12, Rust only).
  - https://slint.dev/blog/slint-1.18-released
  - https://docs.slint.dev/latest/docs/slint/guide/platforms/mobile/ios/
- Slint's own "Making Slint Desktop-Ready" post (2025-10-09) lists rich text, shortcuts, global drag-and-drop, modal dialogs and tooltips as missing; 1.16 added Markdown-styled *display* text and 1.17 tooltips and drag-and-drop. `TextEdit` remains a plain text control, with no inline widgets.
  - https://slint.dev/blog/making-slint-desktop-ready
- Licence: GPL-3.0-only OR Slint royalty-free OR commercial. AGPL-3.0 Rocky can use the GPL option, but an MIT/Apache editor crate built on Slint would in practice inherit Slint's terms. The `.slint` DSL is also a smaller corpus for agents than Rust (judgement).
  - https://slint.dev/pricing

### Freya — watch

- 0.5.0 (2026-11-05), 0.5.2 (2026-11-11), MIT, ≈12k downloads per 90 days, 3.4k stars, essentially one maintainer. Skia rendering; since 0.4 it has its own reactive core instead of Dioxus. 0.5 adds multiline inputs, wrapped-line navigation, per-item-size virtual scroll, multi-window contexts, AccessKit work, an Android dev shell and an experimental web backend. Neither the release notes nor the 0.5 post mention IME.
  - https://github.com/marc2332/freya
  - https://freyaui.dev/posts/0.5

### Makepad — not credible for Rocky

- makepad-widgets 1.0.0 was published 2025-05-13 and nothing since on crates.io; development happens on a `work` branch that "is squashed onto `main` and re-pointed about weekly". Strong platform reach (Linux X11/Wayland, Android, iOS, web) and an `AGENTS.md`, but a moving git target with a custom live DSL and thin docs is the worst case for agent fluency (judgement).
  - https://github.com/makepad/makepad

### Xilem / Masonry — not credible for Rocky

- 0.4.0 (2025-10-29). The README: "Xilem and Masonry provide an experimental high-level architecture." Linebender's text stack (Parley 0.13, 2026-11-12; Vello 0.12) is however the shared text/rendering layer under Blitz and Masonry, and a candidate building block for a native editor.
  - https://github.com/linebender/xilem

## Criteria matrix (judgement, from the findings above)

| | Tauri 2 (A) | GPUI (B) | egui (C) |
|---|---|---|---|
| Agent fluency | Best (web + Tauri corpus, stable 2.x) | Weak docs, large real code, high churn | Good corpus, agent tooling (`egui_mcp`), regular breaks |
| Live Preview editor | Reuse CodeMirror 6 + reference package | Build on Zed-grade primitives | Build from scratch on `Galley` |
| IME / a11y | Browser engine | Works, Wayland IME still hardening; AccessKit | Basic IME; AccessKit |
| 100k tree | Virtualised DOM | Virtual list built in | `show_rows` |
| Wayland | WebKitGTK quirks; CEF runtime in v3 alpha | Native, proven by Zed | Native via winit |
| Mobile | Android + iOS stable | iOS experimental, no Android | Android, no iOS |
| Desktop features | Full (menus, windows, tray, bundler) | Windows, dock/tabs; native menus macOS-centric | Own menus, multi-viewport, 3rd-party docking |
| Rust everywhere | No (TS UI) unless Leptos | Yes | Yes |

## The 2–3 credible options

1. **Tauri 2 + webview UI (TypeScript, or Leptos for Rust UI) with CodeMirror 6.** Buys: the cheapest and lowest-risk Live Preview (the behaviour reference *is* the implementation), browser-grade IME/a11y, the best agent fluency, and Android + iOS today. Costs: the UI and editor are not Rust (the editor crate becomes a CodeMirror package), WebKitGTK rendering quirks on Wayland until the CEF runtime matures, and an IPC seam between `rocky-core` and the UI. Dioxus desktop is the same trade with a Rust RSX UI and more API churn.
2. **GPUI + gpui-component.** Buys: Rust everywhere, native Wayland/X11 proven by Zed, the strongest native text primitives, a ready virtual list and dock/tab layout, Apache-2.0 throughout. Costs: pre-1.0 with frequent breaking changes and thin docs (agents learn from Zed's source), the Live Preview editor built in-house (Zed's GPL editor cannot be reused in an MIT crate), and mobile reach limited to an experimental iOS backend with no Android.
3. **egui.** Buys: Rust everywhere, the simplest mental model for agents, a large corpus plus `egui_mcp`/`egui_kittest` for agent-driven testing, Android support, and an MIT/Apache stack. Costs: an immediate-mode UI that is harder to make look and feel like Obsidian, the largest editor build (Live Preview from scratch on a plain text widget), no iOS, no native menus, and breaking changes every release.

A Rust-native route (2 or 3) is the one that needs the throwaway Live Preview prototype flagged in #28 before the route is locked; route 1 does not.
