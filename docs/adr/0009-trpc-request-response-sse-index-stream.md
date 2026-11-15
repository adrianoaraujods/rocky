# tRPC for the web shell's request/response surface, SSE for the index stream

The MVP web surface (ADR-0006) is one Node/TypeScript process: an Express 5 server serving a React 19 / Vite 8 SPA, with the host-agnostic core (vault engine + renderer) on the other side of the ADR-0006 seam. The shell's client state is one server-authoritative derived vault index (ADR-0004) pushed to the app — the frontend-landscape research verdicts that milestone state around Zustand 5 updated from an SSE delta stream, and explicitly skipped TanStack Query as "not worth a dependency in a brutal-simplicity build". That leaves the shell a long-lived push channel for the index and a spare, request/response surface: configure/open the vault, read settings, fetch a note body, run name/alias/title/path search, and the authoring mutations of M3. The research recommended hand-rolling that as "a few JSON routes". Decided in the #12 bootstrap grilling: the request/response surface is built on **tRPC** instead, and the index stream **stays SSE** (tRPC subscriptions are not used).

## Considered Options

- **A few hand-rolled JSON routes** (frontend-landscape "Stack A"): rejected as the *chosen* shape. It works, but the request/response surface is where the server↔app contract is written by hand twice (schema on each side), exactly the seam a solo dev + agent benefits from leaving to a single typed source — and M3 authoring brings ~8 mutation procedures to maintain that way.
- **tRPC for request/response, SSE for the stream** (chosen): `@trpc/server`/`@trpc/client` (a recent major, TS ≥5.7 only as peer) behind an Express adapter, with the shared `AppRouter` type as the single cross-side contract. The index remains push-based over SSE — React-Query-free.
- **tRPC with subscriptions replacing SSE**: rejected. The index is one high-frequency delta stream; tRPC's WebSocket subscription machinery adds reconnect/lifecycle surface the boring `res.write` SSE route already covers, and the research's push-SSE framing still holds. Kept for the future only if a truly RPC-shaped live need appears.
- **`@trpc/react-query` for the client**: rejected for the MVP. It drags in `@tanstack/react-query` transitively — the exact dependency the frontend-landscape verdicts deprioritized. The **vanilla client** (`createTRPCClient`) is called from inside Zustand store actions, keeping pull calls out of component rendering and React-Query out of the dependency tree.

## Consequences

- `@trpc/server` + `@trpc/client` join the shell package; core stays clean (tRPC never crosses the ADR-0006 seam — the router adapter calls into the engine).
- `packages/shell/src/shared/api.ts` holds the `AppRouter` type; the server router and the app client are the two consumers, so a procedure name or its shape can't drift across the wire.
- The index delta stream remains a plain SSE route on Express (`res.write`), per the Zustand verdicts; the watcher/sweep reconciler (ADR-0004, issue #14) is untouched by this decision.
- First procedures are zero-input and need no zod; input schemas are added per procedure when they get input (e.g. `vault.open(path)` in #13), not loaded up front.
- The bootstrap (#12) proves the pattern with a `health.ping` procedure; #13's "shell reports vault open with vault root and file count" becomes a typed `vault` procedure; M3 authoring mutations are added as plain procedures.