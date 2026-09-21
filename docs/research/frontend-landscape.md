# Frontend framework & build landscape

Research for the wayfinder map issue (GitHub #7): recommend, as of September 2026, the frontend framework + build tooling for a single self-hosted Node/TypeScript process that serves a browser frontend over plain local Markdown files on the server, with a derived vault index (file catalog + note-name registry + clone edges + link edges) recomputed reactively as the filesystem changes. Hard constraints: one process, one frontend, one auditable build, brutal simplicity (three prior attempts stalled on scope).

Every claim below cites a primary source (official release notes, docs, changelogs, source repos). Current facts were checked against npm registrations and official blogs dated through 2026-09-15.

## Verdicts

### Frontend frameworks

**React 19 + Vite — VERDICT: viable (the safest pick).** React 19 shipped stable 2024-12-05; the 19.2.x line is the current stable series with 19.3 shipped 2026-09-09, and the React team moved governance under the React Foundation (Linux Foundation). React is MIT-licensed. The official non-framework build path is Vite — React's own blog explicitly names it a first-class replacement for the deprecated Create React App route — so React 19 + Vite is a squarely supported combination, not a scratch setup. Doc quality and the size of the ecosystem-language corpus are unmatched; that matters because a solo dev + AI agent leans on examples. Fit for the reactivity model is average: the derived index is a normal object with updates pushed over SSE, so React works, but you must bolt on an external store (Zustand) — React has no native derived-value primitive at the app store level. One red flag to script around: React Server Components have had critical RCE and source-exposure/DOS advisories (Dec 2025) — fine, Rocky uses a client-only SPA and must not turn on RSC/SSR.

- React 19 stable: https://react.dev/blog/2024/12/05/react-19
- React 19.3 (2026-09-09): https://react.dev/blog/2026/09/09/react-19-3
- React Foundation / Linux Foundation governance: https://react.dev/blog/2026/02/24/the-react-foundation
- CRA sunset naming Vite, Parcel, Rspack: https://react.dev/blog/2025/02/14/sunsetting-create-react-app
- React Compiler 1.0 (optional): https://react.dev/blog/2025/10/07/react-compiler-1
- RSC advisories: https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components and https://react.dev/blog/2025/12/11/denial-of-service-and-source-code-exposure-in-react-server-components

**Svelte 5 (+ SvelteKit vs Vite-only SPA) — VERDICT: viable (the leanest pick).** Svelte 5 has been stable since 2024-10-22 and is at 5.57 with active minor-line maintenance as of Sept 2026 (MIT). Runes make reactivity an explicit first-class construct you can use in plain `.svelte.ts` modules: `$state` holds the vault index, and every derived view (tree overlay, backlink list, registry lookups) becomes a `$derived` over it — the derived-index model is literally the framework's native model, with no state library. Two modes exist: the plain `svelte-ts` Vite SPA template, and SvelteKit. For Rocky the right mode is the Vite SPA template, not SvelteKit: SvelteKit 3 is mid-churn (in Release Candidate as of 2026-08-13 with broad breaking surface like the `$lib` → `#lib` rewrite and Node 22 requirement), and its own docs warn that SPA fallback mode buys you routing/meta machinery you don't need for a single-process local app. Trade-off you accept: a smaller ecosystem and thinner agent-language coverage than React.

- Svelte 5 stable ("Svelte 5 is alive"): https://svelte.dev/blog/svelte-5-is-alive
- Sept 2026 maintenance state (5.57): https://svelte.dev/blog/whats-new-in-svelte-september-2026
- SvelteKit 3 Release Candidate (do not adopt mid-churn): https://next.svelte.dev/blog/sveltekit-3-release-candidate
- SvelteKit SPA mode (and why to avoid it unless locally served, which Rocky is): https://kit.svelte.dev/docs/single-page-apps

**Solid (+ SolidStart vs solid + Vite) — VERDICT: not viable for this product right now.** Solid's fine-grained signals (`createSignal`/`createStore`/`createMemo`) are, in pure reactivity terms, the single best fit for a derived-index store, and `solid-ts` is an official Vite template. But as of today the maintained tip of the ecosystem is a breaking major transition: Solid 2.0 hit Release Candidate 2026-08-13 (async-reactive core, new DOM model) with 1.9.15 still the npm stable; and SolidStart's docs remain flagged "in beta" more than two years after 1.0. For a brutally-simple solo AI-assisted build, you do not want to start on the eve of a rewrite you must then grok under a version-transition. MIT-licensed; simply a bad timing call.

- Solid 1.9.15 npm stable: https://www.npmjs.com/package/solid-js
- Solid 2.0 RC ("The Big \<Reveal>") 2026-08-13: https://www.solidjs.com/blog/solid-2-0-rc-the-big-reveal
- SolidStart 1.0 release + "docs in beta": https://www.solidjs.com/blog/solid-start-the-shape-frameworks-to-come

**Plain TS without an SPA framework — VERDICT: viable (baseline, not recommended as the build plan).** The `vanilla-ts` Vite template is official and gets you the minimum concept count: one server, one bundle, zero framework, zero state library, SSE deltas applied to a module-level object, DOM updated by hand. For Rocky's actual UI (a tree pane, a backlinks pane, search, note rendering all reading one shared reactive index) hand-rolled DOM sync is exactly the kind of re-implementation that previously stalled scope. Treat it as a fallback posture only.

- Official `vanilla-ts` template: https://vite.dev/guide/

### Build tooling

**Vite — VERDICT: viable; current and not superseded for the simple SPA case.** As of Sept 2026 Vite 8.x is current (8.3.0 in docs; 8.0 shipped 2026-03-12, 8.2 line carries regular patches, 7.3/8.1 get backports). Nothing has superseded it for the single-static-bundle SPA; the 2025→2026 ecosystem still converges on Vite — it is now the official scaffolding surface for react, svelte, solid, and plain-TS templates, and its own docs are the recommended starting point for every one of them. Vite 8 bundles with Rolldown (oxc-powered) and requires Node 20.19+/22.12+, one major above Express 5's floor, all within current Node LTS. One auditable production artifact (`vite build` → `dist/`) served by the app's own process; dev server only for development. MIT.

- Vite 7 (context / Node floor now carried into 8): https://vite.dev/blog/announcing-vite7
- Vite 8.x release/support lines: https://vite.dev/releases and https://github.com/vitejs/vite/blob/main/packages/vite/CHANGELOG.md
- Getting started, templates, Rolldown bundling: https://vite.dev/guide/

### State management

Heads-up framing: Rocky's client-side state is one server-authoritative derived index mutated by an SSE delta stream — not REST cache. That single fact sorts the options.

**Zustand 5 — VERDICT: viable (best React pick).** MIT, current at 5.0.x with active maintenance. A small external store you update from the SSE handler and select from in components is the natural shape for a push-updated index; no cache/refetch machinery to fight. This is the standard companion to the React+Vite route above.

- Zustand 5.0.15, MIT: https://www.npmjs.com/package/zustand and https://github.com/pmndrs/zustand

**TanStack Query 5 — VERDICT: viable but not the natural fit; skip for the index.** The premier server-state cache, driven by request/response lifecycle (keys, staleTime, refetch, invalidation). A push-based SSE stream has no fetch/refetch lifecycle, so Query's core machinery is overhead here; the one deserved use (per-note body fetches) is small enough to do with an ordinary fetch + local state. Not worth a dependency in a brutal-simplicity build.

- Current release line (5.10x), MIT: https://tanstack.com/query/latest and https://www.npmjs.com/package/@tanstack/react-query

**Plain signals/stores (Svelte runes / Solid signals / Zustand-less React) — VERDICT: viable, and zero-cost where the framework supplies them.** In Svelte 5 the index is literally `$state` + derived views are `$derived` in a `.svelte.ts` module — no library. Solid likewise (createStore/createMemo). In React there is no built-in app-store primitive, which is exactly why Zustand earns its slot.

- Svelte 5 runes incl. `.svelte.ts` universal reactivity: https://svelte.dev/blog/svelte-5-is-alive

**Nothing + component-local + SSE — VERDICT: viable at component scope only.** Fine for a throwaway UI; a mistake here because the vault index is read by the tree, backlinks, and search simultaneously — a shared, single-source reactive store is the simple thing, and signals/runes make it nearly free.

### Server framework

All four are MIT and fit one Node process; the question is which gives the least surface for static-serving + a few JSON routes + one SSE stream.

**Express 5 — VERDICT: viable (the boring pick), and the one with the most explicit LTS story.** v5.0 (2024-09) was made npm `latest` at 5.1.0 on 2025-03-31 with a published LTS timeline: 5.x ACTIVE through (no sooner than) 2026-04-01, EOL no sooner than 2027-04-01; v4 moves to EOL no sooner than 2026-10-01 — so you must start on v5, not v4. Boring API, `express.static` + a catch-all for the SPA + `res.write` SSE is the whole surface.

- Express 5 intro: https://expressjs.com/en/blog/2024-10-15-v5-release/
- v5.1.0 = npm latest + LTS timeline: https://expressjs.com/en/blog/2025-03-31-v5-1-latest-release/

**Fastify — VERDICT: viable but heavier than the job requires, and a major is beginning.** v5.12 is current (OpenJS Foundation, MIT, Node 20/22); excellent, but its plugin/schema encapsulation is machinery a two-route JSON + SSE app will never exercise, and v6.0 alphas have begun. Faster (1.6× Express in its own benchmark) in a way this single-user local app cannot feel.

- Current v5.12.x: https://www.npmjs.com/package/fastify and https://github.com/fastify/fastify/releases
- LTS policy: https://fastify.dev/docs/latest/Reference/LTS/

**Hono — VERDICT: viable (the leanest pick).** MIT, zero-dependency, Web-Standards API with a Node.js runtime adapter and, critically, a first-party `streamSSE` helper — SSE support that Express needs you to hand-roll. 4.13.x is current with very high activity/downloads. Its only wrinkle: it's multi-runtime (Workers/Deno/Bun/Node), so you're adopting a project whose center of gravity isn't only Node — but Node is an officially maintained adapter.

- Hono 4.13.x, MIT: https://www.npmjs.com/package/hono and https://hono.dev/
- First-party SSE helper: https://hono.dev/docs/helpers/streaming

**Bare Node `node:http` — VERDICT: technically viable, not recommended.** Node gains static serving and routing nothing out of the box; you'd re-implement mime/static/SPA-fallback for zero size benefit. Only worth it if you set a hard cap of ~0 runtime dependencies (realizable: Filer/Node static is ~20 lines) — a legitimate lean variant of the "no framework" baseline.

### PWA / offline

**vite-plugin-pwa + Workbox — VERDICT: feasible and cheap, but defer offline-first.** `vite-plugin-pwa` 1.3.0 (2026-05, MIT, actively maintained, framework-agnostic) + Workbox v7's `generateSW` gives a service worker + manifest with near-zero config inside Vite; it is the standard Vite PWA path and works with any of the frameworks above.

**Realism of offline-first for this product — not viable as a v1 goal.** Offline-first requires the client to work while the server is unreachable. Here the vault *is* the server's filesystem — the UI shell and a cached index snapshot can survive Wi-Fi dropouts, but the notes themselves are only as reachable as the server that owns them. True offline editing would mean replicating the vault into the browser, which collides with the single-process, files-are-the-store architecture. Right posture: installable shell + cached UI later (a small, later ticket); treat offline note editing as an out-of-scope surface.

- vite-plugin-pwa: https://github.com/vite-pwa/vite-plugin-pwa and https://www.npmjs.com/package/vite-plugin-pwa
- Workbox generateSW/injectManifest usage: https://vite-pwa-org.netlify.app/workbox/

## Two credible option stacks

The deciding question is where you spend your extremely scarce "brutal-simplicity" budget: on one extra state library but the planet's deepest talent pool (React), or on no state library at all but a smaller ecosystem (Svelte). Both stacks serve the same single Vite 8 bundle from the same Node process, both are MIT end-to-end, and both should defer PWA.

**Stack A — Safest / most-boring.** React 19 (client-only SPA, RSC left off) + Vite 8 (`react-ts` template) + Zustand 5 (SSE deltas pushed into one store) + Express 5 (static + JSON + `res.write` SSE). The trade-off it buys: lowest discovery risk for a solo dev *and* an AI agent — React has by far the largest good-examples corpus a code-model is trained on, Compiler v1.0 and the Linux-Foundation governance de-risk the 2026 health question, and Express's published LTS timeline is the most predictable maintenance line of all four servers. You pay for that in a heavier mental model: reactivity must be bolted on via Zustand, and you must consciously stay off RSC/SSR after its Dec-2025 advisories.

**Stack B — Leanest.** Svelte 5 (runes) + Vite 8 (`svelte-ts` SPA template; explicitly *not* SvelteKit) + no state library (the vault index is `$state`, views are `$derived`, in one `.svelte.ts` module) + Hono (first-party `streamSSE`). The trade-off it buys: the fewest moving parts possible without hand-rolling DOM sync — the derived-index model is the framework's native model, so "index recomputes when files change" is expressed almost literally, and the total production dependency surface is Svelte + Hono + Vite. You pay for that in a smaller ecosystem, thinner agent-language coverage, and the discipline to keep SvelteKit's v3 churn (and its Node-22-aligned breaking changes) out of the build by staying on the plain Vite mode.

Recommended default for this roadmap: **Stack A** — the agent-assisted build makes React's example density the decisive resource, and "which of two routes" questions (RSC vs no-RSC, store vs no-store) are exactly the scope-growth traps Rocky has stalled on before; largely resolved by pre-deciding "client-only SPA + Zustand + Express." Choose **Stack B** if the product brief is, definitively, that absolute fewest-dependencies beats familiarity.