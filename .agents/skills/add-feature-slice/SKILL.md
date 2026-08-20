---
name: add-feature-slice
description: >
  Workflow for creating a new feature slice (vertical DDD) in an app. Use
  whenever adding a new domain capability — a feature folder with its route,
  controller, service, repository, schemas, and public surface — on the
  backend, frontend, or SSR app. Covers naming, Pattern A vs B contract
  location, slice layout, interfaces at boundaries, composition-root wiring,
  and required tests. Supplements docs/11-code-organization.md.
---

# Add Feature Slice

Every feature is a vertical slice that owns everything it needs — its route,
controller, service, repository, schemas, and (on the frontend) its components
and hooks. The horizontal layering still exists, but it lives **inside** the
slice. The order of steps matters: name → pattern → files → interfaces →
wiring → tests.

## 1. Pre-flight

Before writing any code, confirm:

- **Feature name** — a cohesive domain capability, plural noun (`users`,
  `orders`, `billing`), lowercase kebab-case directory. Never a technical layer
  (`controllers/`, `services/`, `repositories/`). If the name is ambiguous or
  the capability is really a technical layer, ask the user before proceeding.
- **Pattern A vs B** — confirm the app's pinned pattern from its README and
  `docs/10-stack-guidance.md`. This decides where schemas live: **in-slice**
  for Pattern A, **`packages/shared`** for Pattern B (the feature imports
  them).
- **Feature vs shared code** — if the capability already exists or is
  cross-cutting (auth, i18n, logging, UI primitives), do not create a slice.
  Put it in `shared/` or `packages/*` instead. Shared code is extracted, not
  duplicated.
- **Related skills** — if the feature needs a new table or relation, load the
  `drizzle-migration` skill first. If it exposes endpoints, load
  `add-api-endpoint`. If it changes a Pattern B contract, load
  `change-shared-contract`.

## 2. Backend slice layout

Create `src/features/<feature>/` in this order:

```
src/features/<feature>/
├── schemas.ts      # Zod schemas for this feature's inputs/outputs
├── repository.ts   # the only module that touches Drizzle
├── service.ts      # business logic; depends on the repository interface
├── controller.ts   # maps validated input → service calls → responses
├── route.ts        # HTTP route(s) for this feature
└── index.ts        # public surface: exports only what other features may import
```

- **`schemas.ts`** — Zod schemas for inputs/outputs. In Pattern B, import from
  `packages/shared` instead of defining them here.
- **`repository.ts`** — interface + Drizzle implementation. Typed methods; no
  Drizzle internals leak upward. This is the only Drizzle-touching module.
- **`service.ts`** — interface + implementation. Business logic; depends on the
  repository **interface**, never on the database directly.
- **`controller.ts`** — maps validated input to service calls and results to
  responses.
- **`route.ts`** — HTTP route(s). See `add-api-endpoint` for endpoint
  conventions.
- **`index.ts`** — exports only the public surface. Anything not exported is
  internal; no cross-feature imports of internals.

## 3. Interfaces at boundaries

Every service and repository declares an interface and a concrete
implementation. Declare the interface first, then the implementation:

```ts
// features/users/repository.ts
export interface UserRepository {
  findById(id: UserId): Promise<User | undefined>;
  create(input: CreateUserInput): Promise<User>;
}

export class DrizzleUserRepository implements UserRepository {
  // the only module that touches Drizzle
}
```

```ts
// features/users/service.ts
export interface UserService {
  register(input: RegisterUserInput): Promise<User>;
}

export class UserServiceImpl implements UserService {
  constructor(private readonly users: UserRepository) {}
  // business logic; never touches the database directly
}
```

Naming: `XRepository` / `DrizzleXRepository`, `XService` / `XServiceImpl`.
Consumers depend on the interface, never the implementation. This is the seam
for testing and for the A→B split.

## 4. Composition-root wiring

Dependencies are injected through constructors and wired in a **single
composition root** at app bootstrap. There is no DI container library.

- **Constructor injection only.** A class declares its dependencies as
  constructor parameters typed as interfaces.
- **One composition root.** `src/composition-root.ts` is the only place that
  constructs implementations and wires them to interfaces.
- **No `new` outside the composition root.** Application code receives its
  dependencies; it does not construct them.

After creating the slice, open `src/composition-root.ts`, construct the new
implementations, wire interface → implementation, and pass the wired services
into route registration:

```ts
// composition-root.ts
const userRepository: UserRepository = new DrizzleUserRepository(db);
const userService: UserService = new UserServiceImpl(userRepository);
// ...wire the rest, then register routes with the wired services
```

## 5. Frontend slice

The frontend uses the same vertical organization. Create
`src/features/<feature>/` in this order:

```
src/features/<feature>/
├── api.ts          # typed API calls (Pattern B: typed client from packages/shared)
├── queries.ts      # TanStack Query hooks for this feature
├── hooks/          # feature-specific hooks
├── components/     # components for this feature
└── index.ts        # public surface
```

- **Feature folders mirror the backend.** Same domain capability, same mental
  model.
- **Data access is centralized per feature.** TanStack Query hooks live in
  `queries.ts`; typed API calls live in `api.ts`. Never scatter hand-typed
  `fetch` calls through components.
- **Components stay thin.** Components render; hooks and queries own state and
  data fetching.
- **No DI container on the client.** React context and hooks are sufficient.
- **Shared UI primitives** live in `shared/ui` or `packages/ui`. Feature
  components are not shared; extract them only when a second consumer appears.

## 6. SSR app (Pattern A)

The SSR app is both server and client in one codebase. It follows both
patterns above:

- **Server side** (loaders, server functions, server routes): apply §2–§4.
  Loaders and server functions call services — never the database directly.
- **Client side** (components, hooks, queries): apply §5.
- **Contract location:** schemas live inside the app with the feature until a
  split is actually needed. Do not extract them speculatively.

## 7. Required tests

- **Mock the interface, not the implementation.** Tests construct a fake
  `UserRepository` and pass it to the service under test.
- **Per-feature tests.** Test files live next to the code they test, inside
  the feature folder.
- **Repositories** are tested against a real database (test or containerized
  Postgres), **never mocked and never production**. Use transaction-per-test
  (rollback) or truncate between tests.
- **Services** are tested with mocked repositories.
- **Controller/route** integration tests via the framework's test client.
- Test naming: `should <expected behavior> when <condition>`.

See the `write-tests` skill for the full test workflow.

## Checklist

Before considering the feature slice done, verify:

- [ ] Feature name is a cohesive domain capability (plural noun, kebab-case),
      not a technical layer.
- [ ] Pattern (A vs B) confirmed; schemas live in the right place (in-slice
      for A, `packages/shared` for B).
- [ ] Slice folder `src/features/<feature>/` holds route, controller, service,
      repository, schemas, and `index.ts`.
- [ ] Repository is the only Drizzle-touching module; typed methods, no
      Drizzle internals leaked upward.
- [ ] Service and repository declare interfaces; consumers depend on the
      interface, never the implementation.
- [ ] Implementations wired in the composition root; no `new` outside it.
- [ ] `index.ts` exports only the public surface; no cross-feature imports of
      internals.
- [ ] Frontend slice (if any) mirrors the backend: `api.ts`, `queries.ts`,
      `hooks/`, `components/`, `index.ts`.
- [ ] SSR loaders/server functions call services, never the database.
- [ ] Tests: repository against a real test DB, service with a mocked
      repository, endpoint integration tests.
- [ ] Tests live next to the code, named `should <behavior> when <condition>`.
- [ ] Full verification pipeline passes (format, lint, typecheck, test,
      build).
