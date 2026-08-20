# 07 — Performance

This document defines the performance expectations every change must respect. Performance is a design concern, not a post-launch activity.

## Principles

1. **Measure before optimizing.** Never optimize without a measurement. For the audit workflow and tooling, see the `performance-review` skill.
2. **Optimize the critical path.** Focus on what the user experiences: initial load, interaction latency, and perceived responsiveness.
3. **Avoid premature optimization.** Write clear, correct code first; apply the techniques below where they matter.

## High-impact, low-complexity techniques

The techniques below give the largest performance wins for the least added complexity. They are framework-native or standard library features — no custom infrastructure required. Prefer them before reaching for anything more elaborate.

### Cursor-based (keyset) pagination

Offset pagination (`page`/`pageSize`) degrades as the collection grows: each deep page re-scans and discards `OFFSET` rows, so cost grows linearly with page depth. Cursor-based pagination filters on a stable sort key instead, so every page is O(1) — the database seeks directly to the cursor.

- **When to use:** large or append-only collections, deep pagination, and any list where rows are added while users page through it (offset pagination can skip or duplicate rows in that case).
- **When offset is fine:** small collections, admin tables, or any list where the user needs to jump to an arbitrary page number.
- **How it works:** the client sends an opaque cursor; the server filters with a keyset predicate on a stable, unique sort key. The `id` tiebreaker guarantees a total order even when sort keys collide. See the `performance-review` skill for the implementation.
- **Requirements:** an index on the sort key (see `docs/05-data-layer.md`), and a stable sort order. The cursor contract lives in `docs/04-api-design.md`.
- **Complexity note:** this is a query change plus an opaque token — no new infrastructure.

### Route lazy-loading for SPAs

Split the client bundle by route so the initial load ships only the code for the current route; sibling routes load on navigation. This is a framework-native feature (dynamic `import()` wired into the router), not custom code.

- **When to use:** any SPA with more than a couple of routes. It is the default, not an optimization.
- **Trade-off:** each route pays a small per-navigation fetch and a loading state; the initial bundle shrinks dramatically, which is usually the bigger win.
- **Combine with prefetching:** prefetch the next likely route on hover or on visibility (e.g. link hover) so navigation feels instant without shipping everything up front.
- **Complexity note:** one `import()` per route plus a loading fallback. Do not hand-roll a loader; use the router's built-in mechanism.

### Other low-complexity wins

- **Code-splitting by heavy dependency** — load large third-party libraries (charts, editors, date pickers) only where they are used, not in the shared bundle.
- **Bundle budget in CI** — a hard budget (e.g. 200 KB gzipped initial JS) enforced by a bundle analyzer catches regressions automatically.
- **Lazy-load images and below-the-fold components** — native `loading="lazy"` for images, and defer heavy components until they are near the viewport.
- **Parallel data fetching** — use `Promise.all` for independent requests; never `await` sequentially when operations do not depend on each other.

## Frontend

### Bundle size

- Set a bundle budget (e.g. 200 KB gzipped initial JS for the main route) and enforce it in CI with a bundle analyzer.
- **Code splitting** — split by route and by heavy dependency. Load non-critical code lazily (see "Route lazy-loading for SPAs" above).
- **Tree shaking** — import only what is used; avoid barrel files that pull in whole libraries.
- **Dependency discipline** — a new dependency is a review decision. Prefer small, focused packages; avoid pulling in a large library for a small feature.

### Rendering

- Use the framework's recommended rendering strategy (SSR, static generation, or client-side) per route. Do not render everything client-side by default.
- **Streaming / progressive rendering** where the framework supports it, so the user sees content before the full page is ready.
- **Lazy loading** for images, below-the-fold components, and heavy third-party widgets.
- **Virtualization** for long lists (e.g. `@tanstack/react-virtual`).

### Caching

- Cache static assets with immutable cache headers and content hashes.
- Cache API responses at the appropriate layer (CDN, HTTP cache, or in-memory) with correct `Cache-Control` headers.
- Never cache personalized data without explicit invalidation.

## Backend

### Database

- **Avoid N+1 queries.** Use joins or eager loading (see `docs/05-data-layer.md`).
- **Index everything you filter or sort by.** Verify slow queries with `EXPLAIN` (see the `performance-review` skill).
- **Paginate all list queries.** Never return unbounded result sets. Prefer cursor-based pagination for large collections (see "Cursor-based (keyset) pagination" above).
- **Select only needed columns.**
- Use connection pooling; never open a new connection per request.

### API

- Keep payloads small: field selection, pagination, and compression (gzip/brotli).
- Cache read-heavy endpoints that are safe to cache.
- Use `ETag` / `If-None-Match` for conditional requests on resources.
- Offload slow, non-critical work to background jobs instead of blocking the request.

### Concurrency

- Use `Promise.all` for independent parallel work; never `await` sequentially when the operations do not depend on each other.
- Avoid blocking the event loop: no synchronous I/O, no heavy CPU work in request handlers.

## SSR specifics

- Minimize server-side work per request: cache rendered fragments where safe, avoid redundant data fetches.
- Keep the server bundle lean; do not import client-only code into server code.
- Use the framework's data-fetching primitives (server components, loaders) instead of client-side waterfalls.

## Monitoring

- Every app exposes metrics for: request latency (p50/p95/p99), error rate, and throughput.
- Frontend: track Core Web Vitals (LCP, INP, CLS) in production.
- Performance regressions are caught in CI where possible (bundle size budgets, load tests on critical routes). See the `performance-review` skill for the audit workflow.

## When to stop and ask

If a change requires a performance trade-off (e.g. caching that risks stale data, or dropping a feature to meet a budget), call it out in the plan and confirm with the user before proceeding.
