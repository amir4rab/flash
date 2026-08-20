---
name: performance-review
description: >
  Workflow for auditing performance and applying fixes. Use whenever a change
  touches performance-sensitive code, a review flags a slow query or route, or
  a page/endpoint needs a performance pass. Covers measure-first, N+1
  detection, index review, pagination, bundle analysis, Core Web Vitals, and
  the concrete techniques (cursor pagination, route lazy-loading,
  code-splitting, caching, Promise.all, connection pooling). Supplements
  docs/07-performance.md.
---

# Performance Review

`docs/07-performance.md` is the declarative reference — principles, when-to-use
decisions, and trade-offs. This skill is the ordered workflow for finding and
fixing bottlenecks. The audit runs: measure → backend → frontend → SSR → fix →
re-measure.

## 1. Pre-flight

Before auditing, confirm:

- **Scope** — is the review a single route/query, a page, or the whole app?
- **Baseline** — does a measurement already exist, or do we measure first
  (see §2)?
- **Trade-off gate** — if a fix requires a performance trade-off (e.g. caching
  that risks stale data, or dropping a feature to meet a budget), call it out
  in the plan and confirm with the user before proceeding.
- **Skill routing** — if the fix will add an index or column, load the
  `drizzle-migration` skill. If it changes the pagination contract in
  `packages/shared`, load `change-shared-contract`. If it adds an endpoint,
  load `add-api-endpoint`.

## 2. Measure first

Never optimize without a measurement. Establish a baseline before changing
anything:

- **Backend** — profile the slow endpoint: request latency (p50/p95/p99),
  `EXPLAIN ANALYZE` on the query.
- **Frontend** — run the bundle analyzer; pull Core Web Vitals from
  production.
- **Load tests** on critical routes.

**Record the baseline** so §6 can prove the fix helped. If no measurement is
possible, say so and ask before optimizing.

## 3. Backend audit

### 3.1 N+1 queries

Detect: repository/service calls inside loops (`forEach`/`map` over a parent
collection); Drizzle relational reads done per-row instead of with `with`.

Fix: use joins or eager loading (Drizzle `with` relations, see
`docs/05-data-layer.md`).

### 3.2 Index review

For every column in `WHERE`, `ORDER BY`, or `JOIN`, confirm an index exists.
Run `EXPLAIN` and look for sequential scans on large tables. Cursor pagination
needs an index on the sort key `(createdAt, id)`.

Schema changes go through the `drizzle-migration` skill.

### 3.3 Pagination

Detect: list queries without `limit`; offset pagination on large or
append-only collections.

Apply cursor-based (keyset) pagination:

- Keyset predicate on a stable, unique sort key:
  `WHERE (createdAt, id) < (cursor)` ordered by `createdAt DESC, id DESC`.
  The `id` tiebreaker guarantees a total order.
- Opaque cursor (base64url-encoded); the contract lives in
  `docs/04-api-design.md`.
- No `total` count; use a `hasMore` / `nextCursor` envelope.
- Additive `cursor` / `limit` params.

Offset pagination is fine for small collections, admin tables, or lists where
the user needs to jump to an arbitrary page.

### 3.4 Payload and column selection

- Select only needed columns; no `select *`.
- Keep payloads small: field selection and compression (gzip/brotli).

### 3.5 Caching

- Cache read-heavy endpoints that are safe to cache.
- Use `ETag` / `If-None-Match` for conditional requests on resources.
- Never cache personalized data without explicit invalidation.
- Choose the layer (CDN / HTTP cache / in-memory) and set correct
  `Cache-Control` headers.
- Flag stale-data risk to the user (§1 trade-off gate).

### 3.6 Concurrency

- Use `Promise.all` for independent parallel work; never `await` sequentially
  when operations do not depend on each other.
- Avoid blocking the event loop: no synchronous I/O, no heavy CPU work in
  request handlers.

### 3.7 Connection pooling

- Use connection pooling; never open a new connection per request.
- Verify the pool is configured for the deployment target (e.g. PgBouncer for
  serverless).

## 4. Frontend audit

### 4.1 Bundle analysis

- Run the bundle analyzer and compare against the budget (e.g. 200 KB gzipped
  initial JS for the main route).
- Confirm the CI gate exists and fails on regressions.

### 4.2 Route lazy-loading

- Split the client bundle by route: one dynamic `import()` per route wired
  into the router; sibling routes load on navigation.
- Add a loading fallback.
- Prefetch the next likely route on hover or visibility so navigation feels
  instant.
- Do not hand-roll a loader; use the router's built-in mechanism.

### 4.3 Code-splitting and dependencies

- Split heavy dependencies (charts, editors, date pickers) out of the shared
  bundle; load them only where used.
- Tree shake: import only what is used; avoid barrel files that pull in whole
  libraries.
- Dependency discipline: a new dependency is a review decision. Prefer small,
  focused packages.

### 4.4 Rendering

- Use the framework's recommended rendering strategy per route (SSR, static
  generation, or client-side). Do not render everything client-side by
  default.
- Use streaming / progressive rendering where the framework supports it.
- Lazy-load images (`loading="lazy"`) and below-the-fold components.
- Virtualize long lists (e.g. `@tanstack/react-virtual`).

### 4.5 Caching

- Cache static assets with immutable cache headers and content hashes.
- Cache API responses at the appropriate layer with correct `Cache-Control`
  headers.
- Never cache personalized data without explicit invalidation.

### 4.6 Core Web Vitals

- Track LCP, INP, and CLS in production.
- Identify the worst metric and trace its cause: LCP → largest element /
  render-blocking resources; INP → long tasks / event handlers; CLS → layout
  shifts.
- Tie the fix back to §4.1–§4.5.

## 5. SSR specifics

- Minimize server-side work per request: cache rendered fragments where safe,
  avoid redundant data fetches.
- Keep the server bundle lean; do not import client-only code into server
  code.
- Use the framework's data-fetching primitives (server components, loaders)
  instead of client-side waterfalls.

## 6. Apply the fix and verify

- Make the smallest change that satisfies the plan.
- **Re-measure against the baseline from §2.** The fix must show an
  improvement; if it does not, stop and reassess.
- Run the full verification pipeline: format, lint, typecheck, tests, build.
- Add or adjust tests where behavior changed (see the `write-tests` and
  `write-e2e-tests` skills).

## Checklist

Before considering the performance review done, verify:

- [ ] Baseline measured before any change (latency, bundle size, CWV).
- [ ] No N+1 queries in the reviewed paths; joins or eager loading used.
- [ ] Every WHERE/ORDER BY/JOIN column is indexed; slow queries verified
      with EXPLAIN.
- [ ] All list queries paginate; large/append-only collections use cursor
      pagination with an opaque cursor and no total count.
- [ ] Queries select only needed columns; no `select *`.
- [ ] Read-heavy endpoints cached safely; personalized data never cached
      without explicit invalidation.
- [ ] Independent requests use Promise.all; no sequential awaits; event loop
      not blocked.
- [ ] Connection pooling in use; no per-request connections.
- [ ] Bundle within budget; CI enforces it; routes lazy-loaded; heavy
      dependencies code-split.
- [ ] Core Web Vitals tracked in production; worst metric identified and
      addressed.
- [ ] SSR minimizes server work; server bundle is lean.
- [ ] Fix re-measured against the baseline; improvement confirmed.
- [ ] Trade-offs (stale data, dropped features) confirmed with the user.
- [ ] Full verification pipeline passes (format, lint, typecheck, test,
      build).
