# 07 — Performance

This document defines the performance expectations every change must respect. Performance is a design concern, not a post-launch activity.

## Principles

1. **Measure before optimizing.** Never optimize without a measurement. Use profiling, bundle analysis, and load tests to find real bottlenecks.
2. **Optimize the critical path.** Focus on what the user experiences: initial load, interaction latency, and perceived responsiveness.
3. **Avoid premature optimization.** Write clear, correct code first; apply the techniques below where they matter.

## Frontend

### Bundle size

- Set a bundle budget (e.g. 200 KB gzipped initial JS for the main route) and enforce it in CI with a bundle analyzer.
- **Code splitting** — split by route and by heavy dependency. Load non-critical code lazily.
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
- **Index everything you filter or sort by.** Verify with `EXPLAIN` on slow queries.
- **Paginate all list queries.** Never return unbounded result sets.
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
- Performance regressions are caught in CI where possible (bundle size budgets, load tests on critical routes).

## When to stop and ask

If a change requires a performance trade-off (e.g. caching that risks stale data, or dropping a feature to meet a budget), call it out in the plan and confirm with the user before proceeding.
