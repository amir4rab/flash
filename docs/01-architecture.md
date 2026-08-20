# 01 — Architecture

This document defines the repository layout and the supported application patterns. Read it before creating or restructuring any package.

## Monorepo

All projects built on this framework are **npm workspaces monorepos**. There is no orchestration layer beyond npm workspaces (no Turborepo, no Nx).

### Layout

```
/
├── apps/          # Deployable applications (one per entry point)
│   ├── web/       # SSR application or SPA frontend
│   └── api/       # JSON backend (only when the SPA+backend pattern is used)
├── packages/      # Shared, non-deployable libraries
│   ├── shared/    # Shared types, schemas, and constants
│   ├── ui/        # Shared UI components (optional)
│   └── config/    # Shared tooling configs (optional)
├── docs/          # This framework's documentation
└── package.json   # Root workspace definition
```

### Rules

- **One deployable per `apps/` entry.** Do not bundle multiple services into one application.
- **Shared code lives in `packages/`.** Anything used by more than one app must be extracted into a package. Do not duplicate types or utilities across apps.
- **Dependency direction is one-way.** `apps/*` may depend on `packages/*`. `packages/*` must never depend on `apps/*`. Packages may depend on other packages only when the dependency is acyclic.
- **No cross-app imports.** `apps/web` must never import from `apps/api` and vice versa. They communicate only through the API contract defined in `packages/shared`.
- **Explicit boundaries.** Each package declares a public API surface (its `exports`). Anything not exported is internal and must not be imported by other packages.

## Supported application patterns

The framework supports two patterns. Pick one per application and document the choice in the app's README.

### Pattern A — SSR framework

A single application using a server-rendered framework: Next.js, SvelteKit, Nuxt, Remix, or Astro.

- Server and client live in one app; the framework owns routing and rendering.
- Data access happens on the server (server components / loaders / server routes).
- The client never talks to the database directly.
- See `docs/10-stack-guidance.md` for per-framework conventions.

### Pattern B — JSON backend + SPA

Two applications: a JSON API backend and a client-side SPA.

- `apps/api` exposes a REST API (see `docs/04-api-design.md`).
- `apps/web` is a SPA (React, Vue, or Svelte) that consumes the API.
- The API contract (request/response schemas) lives in `packages/shared` and is the single source of truth for both sides.
- The SPA must not import backend internals; it consumes the API over HTTP.

## Package boundaries

### `packages/shared`

The contract package. Contains:

- Zod schemas for every API request/response (see `docs/04-api-design.md`).
- Shared domain types and constants.
- i18n key definitions (see `docs/06-i18n.md`).

Rules:

- Must be dependency-free or depend only on other `packages/*`.
- Must be usable by both server and client (no Node-only imports unless clearly marked).

### `packages/ui` (optional)

Shared UI components. Rules:

- Components are typed with explicit props interfaces.
- All user-facing text is i18n keys, never hardcoded strings.
- Components must not import from `apps/*`.

### `packages/config` (optional)

Shared tooling configs (ESLint, Prettier, TypeScript base configs). See `docs/03-tooling.md`.

## When to create a new package

Create a new package when:

- Two or more apps need the same code.
- A package's public API is stable enough to be depended on.
- The code has a clear, single responsibility.

Do not create packages speculatively. Start in the app that needs the code and extract when the second consumer appears.

## Naming

- Package names are `@<org>/<name>` scoped (e.g. `@acme/shared`).
- Apps are named after their role: `web`, `api`, `admin`, `worker`.
- Directories are lowercase, kebab-case.
