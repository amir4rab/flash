---
name: add-package
description: >
  Workflow for creating a new package in the monorepo. Use whenever a new
  package is needed in packages/, or when deciding whether to extract shared
  code out of an app. Covers the second-consumer gate, package type selection
  (shared/ui/config), naming, the public exports surface, dependency
  direction, and workspace wiring (tsconfig, lint/typecheck/test/build).
  Supplements docs/01-architecture.md.
---

# Add Package

Create a package only when the second-consumer gate passes; scaffold it with a
declared public surface; wire it into the workspace so the root verification
pipeline picks it up. The gate comes first — never scaffold before confirming
the package is warranted.

## 1. Pre-flight

Before writing anything, confirm:

- **Package type** — `shared` / `ui` / `config`. If the request is ambiguous,
  ask the user before proceeding.
- **Org scope** — the `@<org>` prefix. Derive it from existing packages in the
  workspace; ask if it is not derivable.
- **Contract impact** — if the new package is `packages/shared` and will hold
  or modify API contract schemas, load the `change-shared-contract` skill
  first.
- **i18n impact** — if the package carries user-facing text (e.g. `ui`), load
  the `add-i18n-strings` skill.

## 2. The second-consumer gate

Create a package only when all three criteria hold:

1. **Two or more apps need the same code.**
2. **The public API is stable enough to be depended on.**
3. **The code has a clear, single responsibility.**

Count the consumers:

- **1 consumer** — keep the code in the app (in `shared/` per
  `docs/11-code-organization.md`). Do not create a package.
- **2+ consumers** — extract into a package.

**Pattern A carve-out.** The API contract lives inside the app until a split
is actually needed; it is not extracted to `packages/shared` speculatively. The
A→B evolution path is the trigger for extracting the contract.

If the gate fails, stop and report — do not scaffold.

## 3. Choose the package type

| Type | Contents | Rules |
| --- | --- | --- |
| `packages/shared` | Zod schemas for every API request/response, shared domain types and constants, i18n key definitions | Dependency-free or depends only on other `packages/*`; usable by both server and client (no Node-only imports unless clearly marked) |
| `packages/ui` | Shared UI components | Explicit props interfaces; all user-facing text is i18n keys, never hardcoded strings; must not import from `apps/*` |
| `packages/config` | Shared tooling configs (ESLint, Prettier, TypeScript base configs) | See `docs/03-tooling.md` |

Choose: contract/domain/i18n → `shared`; React components → `ui`; tooling
configs → `config`. `ui` and `config` are optional — only create them when
actually needed.

## 4. Name the package

- Package names are `@<org>/<name>` scoped, e.g. `@acme/shared`.
- Directories are lowercase, kebab-case.
- Derive `<org>` from existing packages in the workspace; derive `<name>` from
  the package's single responsibility. The directory under `packages/` is the
  kebab-case name.

## 5. Declare the public API surface

Each package declares a public API surface via its `exports` in `package.json`;
anything not exported is internal and must not be imported by other packages.

- Set `exports` with `types` and `import`/`require` conditions.
- Point at a barrel `index.ts` that re-exports only the public surface.
- Keep internals in non-exported modules.
- Type-only exports use `import type` (`verbatimModuleSyntax`, see
  `docs/02-typescript.md`).

## 6. Wire into the workspace

- Confirm the package is a workspace member (npm workspaces monorepo; root
  `package.json` workspace definition).
- Add the same scripts the root expects, so the root
  `--workspaces --if-present` scripts pick the package up: `lint`,
  `typecheck`, `test`, `build` (and `test:e2e` where applicable).
- Create a `tsconfig.json` that **extends the shared base config**
  (`docs/02-typescript.md`). Framework-specific settings (JSX, DOM libs) go
  in the app's own tsconfig, never in the shared base.
- Use the shared Prettier config from `packages/config`.
- Coverage threshold for `packages/*` is 80% lines (`docs/03-tooling.md`).
- If the package is `config`, it *is* the source of the shared configs —
  place the base configs there.

## 7. Dependency direction

- **One-way dependency.** `apps/*` may depend on `packages/*`. `packages/*`
  must never depend on `apps/*`.
- Packages may depend on other packages only when the dependency is acyclic.
- No cross-app imports; apps communicate only through the contract in
  `packages/shared`.
- `packages/shared` must be dependency-free or depend only on other
  `packages/*`.

After scaffolding, grep the new package's imports for `apps/*` and for cycles;
both must be absent.

## 8. Verify

Run the full pipeline in order and fix everything that fails:

1. Format — `npm run format:check`
2. Lint — `npm run lint`
3. Typecheck — `npm run typecheck`
4. Test — `npm run test`
5. Build — `npm run build`

If the package is new, confirm the root scripts actually pick it up (the
workspace glob includes it).

## Checklist

Before considering the package done, verify:

- [ ] The second-consumer gate passed: two or more consumers exist, the public
      API is stable, and the code has a single responsibility.
- [ ] Package type is correct (`shared` / `ui` / `config`) and matches the
      package's contents.
- [ ] Name is `@<org>/<name>`; directory is lowercase kebab-case.
- [ ] Public surface declared via `exports` in `package.json`; internals are
      not importable by other packages.
- [ ] Barrel `index.ts` re-exports only the public surface.
- [ ] No `any`; type-only imports use `import type`.
- [ ] Dependency direction holds: no `apps/*` imports, no cycles, `shared` is
      dependency-free or packages-only.
- [ ] `tsconfig.json` extends the shared base config.
- [ ] Package scripts (`lint`, `typecheck`, `test`, `build`) match the root
      `--workspaces --if-present` pattern.
- [ ] Shared Prettier config from `packages/config` is used.
- [ ] Tests cover the package's public surface; 80% line coverage met.
- [ ] Full verification pipeline passes (format, lint, typecheck, test,
      build).
