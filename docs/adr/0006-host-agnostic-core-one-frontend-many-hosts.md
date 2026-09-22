# Host-agnostic core: one frontend, many hosts

Rocky's MVP is a single self-hosted Node/TS process serving one browser frontend, but the driver's stated future landscape spans a native desktop app, a native mobile app, a self-hosted web app with optional auth, and a Fumadocs/Next.js read-only publish template. Decided by grilling in issue #9: the MVP is built as **one frontend, many hosts** — a host-agnostic core (vault engine + renderer) shared by every future surface, wrapped in per-surface shells. The web app built now *is* the self-hosted version; it is not a throwaway stepping stone, and future native shells are thin hosts embedding the same core rather than reimplementations.

## Considered Options

- **Flat brutal-simplicity app**: build the engine and renderer inline with the Express/React shell, refactor when a native client actually lands. Cheapest today; rejected because the extraction later is meaningful refactoring cost, and the map's whole reproach is that scope growth happens when threading through a flat app.
- **Native-is-the-product**: web app as a must-ship marketing/prototype stepping stone, rebuilt bespoke later. Rejected — the driver chose the web app as the durable product surface.
- **A router-first frontend** (see routing below): rejected for the MVP as premature; see Consequences.

## Consequences

- **Module seam**: the vault engine (index build, tree-overlay derivation, link resolution) and the renderer (rewritten `remark-obsidian-md`) are separate TS modules with **no DOM or server imports**. The seam is in the MVP spec and the build ticket-set, not a later refactor.
- **Shared assets across surfaces**: contract + index format + resolution rules (ADR-0001/0002/0004), the vault engine, and the renderer-as-library. The renderer is a remark/mdast library, so the Fumadocs publish template can adopt it inside its own remark pipeline; the router is **not** a shared asset.
- **Routing stays in the shell as state**: the MVP route *is* the current note path in the Zustand store, synced to the URL (history API) for bookmarking/deep links. **Policy**: adopt a real router the moment a second real screen (graph view or advanced search) enters the build; the URL/params schema is designed then, not now. On embedded hosts (webview shells) routing-as-state is uniform; a router would be friction.
- **Server choice is local**: Express 5 is selected for this web surface; auth and multi-user are future add-ons riding whatever server the web version runs, not constraints that shape the seam.
- **PWA/offline**: deferred out of the MVP (vault files live on the server's filesystem; offline editing is not viable against that store). A cheap installable shell remains a post-MVP path.