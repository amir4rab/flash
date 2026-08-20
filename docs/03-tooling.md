# 03 — Tooling

This document defines the tooling every project must use and the verification pipeline every change must pass.

## Toolchain

| Concern | Tool |
| --- | --- |
| Package manager | npm (workspaces) |
| Language | TypeScript (strict) |
| Linting | ESLint with `typescript-eslint` strict config |
| Formatting | Prettier |
| Unit / integration tests | Vitest |
| End-to-end tests | Playwright |
| Runtime validation | Zod |
| Build | Framework-native (Next.js, Vite, etc.) |

## Linting

- Use `typescript-eslint` with the **strict** recommended config, plus `eslint-plugin-import` for import ordering and `eslint-plugin-perfectionist` (or equivalent) for consistent sorting.
- Rules that are non-negotiable:
  - `@typescript-eslint/no-explicit-any` — error.
  - `@typescript-eslint/no-unused-vars` — error.
  - `@typescript-eslint/consistent-type-imports` — error.
  - `@typescript-eslint/no-non-null-assertion` — error.
  - `@typescript-eslint/no-unnecessary-condition` — error.
- Lint must run on the whole workspace: `npm run lint` at the root lints every package.

## Formatting

- Prettier with a shared config in `packages/config`.
- Recommended settings: `semi: true`, `singleQuote: false`, `trailingComma: "all"`, `printWidth: 100`, `tabWidth: 2`.
- Formatting is enforced in CI (`prettier --check`), not just applied locally.

## Testing

### Vitest (unit / integration)

- Tests live next to the code they test: `src/foo.test.ts` or `src/foo.spec.ts`.
- Use `describe` / `it` / `expect` from Vitest.
- Mock at boundaries only (HTTP, database, time). Do not mock the code under test.
- Coverage is collected and reported. The team sets a coverage threshold; the default is 80% lines for `packages/*` and 70% for `apps/*`.

### Playwright (end-to-end)

- E2E tests live in `e2e/` at the app level.
- E2E tests run against a real build in a CI environment, not against dev servers.
- Test the user-visible behavior, not implementation details.
- Use data-testid attributes for selectors; never select by CSS class or text that may change.

### Test naming

- Unit tests: `should <expected behavior> when <condition>`.
- E2E tests: `user can <action> when <context>`.

## Runtime validation (Zod)

- Every external boundary is validated with Zod: API requests/responses, environment variables, config files, webhook payloads.
- Schemas live in `packages/shared` and are the single source of truth for the API contract.
- Never trust unvalidated input. `unknown` in, typed value out (see `docs/02-typescript.md`).

## Environment variables

- All environment variables are validated at startup with Zod (`z.object({ ... }).parse(process.env)`).
- A `.env.example` file documents every variable with a comment describing its purpose.
- Secrets are never committed, logged, or printed.

## Verification pipeline

Every change must pass, in order:

1. **Format** — `npm run format:check` (Prettier).
2. **Lint** — `npm run lint` (ESLint).
3. **Typecheck** — `npm run typecheck` (tsc `--noEmit` across the workspace).
4. **Test** — `npm run test` (Vitest) and `npm run test:e2e` (Playwright) where applicable.
5. **Build** — `npm run build` (framework-native build for each app).

### Root scripts

The root `package.json` exposes:

```jsonc
{
  "scripts": {
    "lint": "npm run lint --workspaces --if-present",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "npm run typecheck --workspaces --if-present",
    "test": "npm run test --workspaces --if-present",
    "test:e2e": "npm run test:e2e --workspaces --if-present",
    "build": "npm run build --workspaces --if-present"
  }
}
```

## CI

- CI runs the full verification pipeline on every push and pull request.
- CI fails on any lint error, type error, failing test, or build failure.
- CI runs on a clean checkout with a lockfile (`npm ci`), never `npm install`.
- CI caches `node_modules` and build outputs to keep runs fast.

## If tooling is not configured

If a project does not yet have lint, typecheck, tests, or build configured, say so explicitly and ask whether to set them up before proceeding. Do not silently skip verification.
