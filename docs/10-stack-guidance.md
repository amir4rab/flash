# 10 — Stack Guidance

This document defines the two supported stacks. It supplements the general docs; where there is a conflict, the general docs win.

## Choosing a pattern

| Need | Pattern |
| --- | --- |
| SEO-critical, content-heavy, or server-rendered app | **A — SSR (TanStack Start)** |
| Highly interactive app with a separate API consumed by multiple clients | **B — JSON backend + SPA** |
| Team wants one codebase, full-stack TypeScript, minimal API surface | **A — SSR (TanStack Start)** |

Document the choice in the app's README.

**Pattern A can evolve into Pattern B.** When the data layer outgrows the app, or a second client needs the API, the SSR app can split into a rendering server plus a separate API backend. The conventions below are designed so that split is a refactor, not a rewrite.

---

## Pattern A — SSR: TanStack Start

A single application in `apps/web` using TanStack Start (TanStack Router with SSR). Server and client live in one app; the framework owns routing and rendering.

- **Routing:** file-based routing via TanStack Router. Route params, search params, and server-function inputs are validated with Zod at the boundary.
- **Data fetching:** TanStack Query wired into router loaders. Server-side data is fetched in loaders; the client never queries the database directly.
- **Mutations:** server functions for mutations; validate inputs with Zod before use.
- **API routes:** follow `docs/04-api-design.md` when they expose data to external clients.
- **i18n:** `i18next` with locale routing (`/en/...`, `/fr/...`).
- **Maturity:** TanStack Start is pre-1.0. Its API may shift; pin versions and review upgrades.

### Contract location

The API contract (Zod schemas, shared types) lives **inside the SSR app** (`apps/web`), not in `packages/shared`, until a split is actually needed. Do not extract it speculatively.

### Evolution path: rendering server

The stack is designed so the app can grow from full-stack into a pure rendering server that delegates all data to a separate API backend. To keep that split a refactor rather than a rewrite:

- **Data access is isolated from day one.** Database access lives behind services and repositories (see `docs/05-data-layer.md`), never inline in route handlers or server functions. When the split happens, that layer moves wholesale into a new `apps/api`.
- **Split mechanics.** `apps/web` becomes the rendering server (SSR + TanStack Query on the client), a new `apps/api` owns the data layer, and the contract moves to `packages/shared`. The web app consumes the API over HTTP via a typed client generated from the shared schemas. This converges on Pattern B's contract model.

---

## Pattern B — JSON backend + SPA

Two applications: a JSON API backend and a client-side SPA.

### Backend (`apps/api`)

- **Framework:** Hono or Fastify (typed, fast, schema-aware). Express is acceptable only if already established.
  - **Hono** — lightweight, TypeScript-first, edge-ready, minimal footprint. Prefer it when the API is small or may run on edge runtimes.
  - **Fastify** — richer plugin ecosystem and more established. Prefer it when the API is large or needs mature plugins.
- **Structure:** routes → controllers → services → repositories (see `docs/05-data-layer.md`).
- **Validation:** Zod schemas from `packages/shared` at every route boundary.
- **API:** REST per `docs/04-api-design.md`.
- **Database:** Drizzle per `docs/05-data-layer.md`.
- **Auth:** framework-level session/JWT handling; per-route authorization checks.

### Frontend (`apps/web`)

- **Framework:** React with Vite.
- **Routing:** TanStack Router with lazy-loaded routes (see `docs/07-performance.md`).
- **Data fetching:** TanStack Query with a typed client generated from the shared Zod schemas; never hand-typed fetch calls scattered through components.
- **State:** server state in the TanStack Query cache; client state minimal and local.
- **i18n:** `i18next` + `react-i18next`, with typed keys from `packages/shared`.
- **Animation:** `motion` (formerly Framer Motion) for animations.

### The contract

- `packages/shared` holds the Zod schemas for every request and response.
- The backend validates incoming requests with these schemas.
- The frontend types its API client from the same schemas (`z.infer`).
- A schema change is a public-contract change: confirm before making it, and update both sides in the same change.

---

## Library ecosystem

The TanStack ecosystem is the default for data and routing concerns:

| Concern | Library |
| --- | --- |
| Data fetching / server state | TanStack Query (both stacks) |
| Routing (SPA) | TanStack Router |
| SSR framework | TanStack Start |
| Animation | `motion` |

A new dependency is a review decision. Prefer small, focused packages; avoid pulling in a large library for a small feature (see `docs/07-performance.md`).

## Third-party services

Services must have the least reliance on third-party services, since they can cause issues in the future.

- **Self-host by default.** Infrastructure such as databases, object storage, auth, and email is self-hosted (e.g. Postgres, S3-compatible storage such as MinIO, SMTP).
- **Managed services only when self-hosting is impractical.** A managed service is a review decision and must be called out in the plan phase.
- **Dependency discipline.** Prefer small, focused npm packages; avoid pulling in a large library for a small feature. A new dependency is a review decision.

---

## Shared conventions across all stacks

- **Monorepo layout** per `docs/01-architecture.md`.
- **Strict TypeScript** per `docs/02-typescript.md`.
- **Tooling** per `docs/03-tooling.md`.
- **i18n requirements** per `docs/06-i18n.md`.
- **Performance** per `docs/07-performance.md`.

## Adding a new framework

If a team wants a framework not listed here, add a section to this document describing:

- The framework's data-fetching and rendering primitives.
- Its i18n solution.
- How it maps to the general conventions.

Treat this as a public-contract change and confirm before adding it.
