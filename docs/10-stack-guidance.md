# 10 — Stack Guidance

This document provides concrete conventions for the supported stacks. It supplements the general docs; where there is a conflict, the general docs win.

## Choosing a pattern

| Need | Pattern |
| --- | --- |
| SEO-critical, content-heavy, or server-rendered app | **A — SSR framework** |
| Highly interactive app with a separate API consumed by multiple clients | **B — JSON backend + SPA** |
| Team wants one codebase, full-stack TypeScript, minimal API surface | **A — SSR framework** |

Document the choice in the app's README.

---

## Pattern A — SSR frameworks

### Next.js (App Router)

- Use the App Router. Route handlers and server components are the default; client components are opt-in with the `"use client"` directive.
- Data fetching happens in server components or route handlers; the client never queries the database.
- Validate route params, search params, and request bodies with Zod at the boundary.
- i18n: `next-intl` with locale routing (`/en/...`, `/fr/...`).
- API routes follow `docs/04-api-design.md` when they expose data to external clients.

### SvelteKit

- Use `+page.server.ts` loaders for server-side data; `+page.ts` for universal loads.
- Form actions for mutations; validate with Zod before use.
- i18n: SvelteKit's i18n routing with `paraglide` or `svelte-i18n`.
- API routes (`+server.ts`) follow `docs/04-api-design.md`.

### Nuxt

- Use Nitro server routes for backend logic; `useFetch`/`useAsyncData` for data.
- i18n: `@nuxtjs/i18n`.
- API routes follow `docs/04-api-design.md`.

### Remix / React Router

- Use loaders and actions; validate with Zod before use.
- i18n: `react-i18next` or `remix-i18next`.
- Resource routes follow `docs/04-api-design.md`.

### Astro

- Use server islands for interactive parts; static generation for content.
- i18n: Astro's built-in i18n routing.
- API endpoints follow `docs/04-api-design.md`.

---

## Pattern B — JSON backend + SPA

### Backend (`apps/api`)

- **Framework:** Fastify or Hono (typed, fast, schema-aware). Express is acceptable only if already established.
- **Structure:** routes → controllers → services → repositories (see `docs/05-data-layer.md`).
- **Validation:** Zod schemas from `packages/shared` at every route boundary.
- **API:** REST per `docs/04-api-design.md`.
- **Database:** Drizzle per `docs/05-data-layer.md`.
- **Auth:** framework-level session/JWT handling; per-route authorization checks.

### Frontend (`apps/web`)

- **Framework:** React (Vite), Vue, or Svelte.
- **Data fetching:** a typed client generated from the shared Zod schemas; never hand-typed fetch calls scattered through components.
- **State:** server state in a cache (TanStack Query / SWR); client state minimal and local.
- **i18n:** `i18next` + `react-i18next` (or the framework's equivalent), with typed keys from `packages/shared`.
- **Routing:** framework-native router with lazy-loaded routes (see `docs/07-performance.md`).

### The contract

- `packages/shared` holds the Zod schemas for every request and response.
- The backend validates incoming requests with these schemas.
- The frontend types its API client from the same schemas (`z.infer`).
- A schema change is a public-contract change: confirm before making it, and update both sides in the same change.

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
