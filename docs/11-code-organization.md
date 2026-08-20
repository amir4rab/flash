# 11 — Code Organization

This document defines how code is organized **inside** each application. It is the pattern specification for both supported stacks (see `docs/10-stack-guidance.md`). Read it before creating or restructuring any module inside an app. For the step-by-step workflow of creating a feature slice, see the `add-feature-slice` skill.

## The organizing principle: vertical DDD

Code is organized **by feature/domain, not by technical layer**. Each feature is a vertical slice that owns everything it needs — its route/controller, service, repository, schemas, and (on the frontend) its components and hooks. The horizontal layering defined elsewhere (`routes → controllers → services → repositories`, see `docs/05-data-layer.md`) still exists, but it lives **inside** each slice, not as top-level folders.

This is the default for the backend and the frontend. It is what makes the codebase scalable and maintainable over the long horizon:

- **High cohesion.** Everything a feature needs is in one place; code is easy to find and change.
- **Independent evolution.** Features can be owned, tested, and eventually split into separate services without touching unrelated code.
- **A clean split path.** The A→B evolution (`docs/01-architecture.md`) is a refactor because each feature already isolates its data access behind interfaces.

### What vertical DDD is not

- It is not a ban on shared code. Cross-cutting concerns (auth, i18n, logging, UI primitives) live in shared folders or `packages/*` and are imported by features.
- It is not a strict DDD prescription. Tactical DDD patterns (aggregates, value objects, domain events) are used where they earn their keep; they are not mandatory for every feature.
- It is not an excuse for duplication. If two features need the same logic, extract it to a shared module or package (see `docs/01-architecture.md`).

## Backend (`apps/api`, and the server side of Pattern A)

### Feature slice layout

```
src/
├── features/
│   ├── users/
│   │   ├── route.ts          # HTTP route(s) for this feature
│   │   ├── controller.ts     # maps validated input → service calls → responses
│   │   ├── service.ts        # business logic; depends on the repository interface
│   │   ├── repository.ts     # the only module that touches Drizzle
│   │   ├── schemas.ts        # Zod schemas for this feature's inputs/outputs
│   │   └── index.ts          # public surface: exports only what other features may import
│   └── orders/
│       ├── route.ts
│       ├── controller.ts
│       ├── service.ts
│       ├── repository.ts
│       ├── schemas.ts
│       └── index.ts
├── shared/                   # cross-cutting, non-feature code
│   ├── auth/
│   ├── logging/
│   └── errors/
└── composition-root.ts       # wires interfaces to implementations (see below)
```

### Rules

- **One feature per folder.** A feature is a cohesive domain capability (e.g. `users`, `orders`, `billing`). Do not create folders per technical layer (`controllers/`, `services/`, `repositories/`).
- **Public surface via `index.ts`.** Each feature exports only what other features may import. Anything not exported is internal. No cross-feature imports of internals.
- **Interfaces at every boundary.** Services and repositories expose interfaces; consumers depend on the interface, never the implementation. This is the seam for testing and for the A→B split.
- **The repository is the only Drizzle-touching module** (see `docs/05-data-layer.md`). Controllers never query the database; services never touch Drizzle directly.
- **Schemas live with the feature** in Pattern A. In Pattern B, request/response schemas live in `packages/shared` (see `docs/10-stack-guidance.md`); the feature imports them.

### Interfaces

Every service and repository declares an interface and a concrete implementation:

```ts
// features/users/repository.ts
export interface UserRepository {
  findById(id: UserId): Promise<User | undefined>;
  findByEmail(email: string): Promise<User | undefined>;
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
  getProfile(id: UserId): Promise<User>;
}

export class UserServiceImpl implements UserService {
  constructor(private readonly users: UserRepository) {}
  // business logic; never touches the database directly
}
```

### Manual dependency injection + composition root

Dependencies are injected through constructors and wired in a **single composition root** at app bootstrap. There is no DI container library.

- **Constructor injection only.** A class declares its dependencies as constructor parameters typed as interfaces. No service locator, no global singletons, no `@inject` decorators.
- **One composition root.** `src/composition-root.ts` is the only place that constructs implementations and wires them to interfaces. It is called once at startup and passes the wired graph into the framework (route registration, server functions, etc.).
- **No `new` outside the composition root.** Application code receives its dependencies; it does not construct them. The composition root is the exception.
- **Why manual:** it adds no runtime dependency, keeps the graph explicit and greppable, and works with strict TypeScript. If an app outgrows manual wiring, adopting a container is a review decision (see `docs/10-stack-guidance.md`).

```ts
// composition-root.ts
const userRepository: UserRepository = new DrizzleUserRepository(db);
const userService: UserService = new UserServiceImpl(userRepository);
// ...wire the rest, then register routes with the wired services
```

### Testing

- **Mock the interface, not the implementation.** Tests construct a fake `UserRepository` and pass it to the service under test (see `docs/03-tooling.md` and the `write-tests` skill).
- **Per-feature tests.** Test files live next to the code they test, inside the feature folder.
- **Repositories** are tested against a real database; **services** are tested with mocked repositories (see `docs/05-data-layer.md`).

For the step-by-step workflow of creating a feature slice and its tests, see the `add-feature-slice` skill.

## Frontend (SPA and the client side of Pattern A)

The frontend uses the same vertical organization. Each feature folder owns its UI and data access; cross-cutting concerns live in shared folders.

### Feature slice layout

```
src/
├── features/
│   ├── users/
│   │   ├── components/       # components for this feature
│   │   ├── hooks/            # feature-specific hooks
│   │   ├── queries.ts        # TanStack Query hooks for this feature
│   │   ├── api.ts            # typed API calls (Pattern B: typed client from packages/shared)
│   │   └── index.ts          # public surface
│   └── orders/
│       ├── components/
│       ├── hooks/
│       ├── queries.ts
│       ├── api.ts
│       └── index.ts
├── shared/
│   ├── ui/                   # shared UI primitives (or packages/ui)
│   ├── hooks/
│   └── lib/
└── routes/                   # router route definitions (TanStack Router)
```

### Rules

- **Feature folders mirror the backend.** A feature on the frontend corresponds to the same domain capability on the backend. Same mental model, easy to find code.
- **Data access is centralized per feature.** TanStack Query hooks live in `queries.ts`; typed API calls live in `api.ts`. Never scatter hand-typed `fetch` calls through components (see `docs/10-stack-guidance.md`).
- **Components stay thin.** Components render; hooks and queries own state and data fetching. Client state is minimal and local (see `docs/10-stack-guidance.md`).
- **No DI container on the client.** React context and hooks are sufficient for sharing dependencies. Do not introduce a DI container in the browser.
- **Shared UI primitives** live in `shared/ui` or `packages/ui` (see `docs/01-architecture.md`). Feature components are not shared; extract them only when a second consumer appears.

## SSR app (Pattern A)

The SSR app is both server and client in one codebase. It follows both patterns above:

- **Server side** (loaders, server functions, server routes): organized by feature, with services and repositories behind interfaces, wired in the composition root. Loaders and server functions call services — never the database directly.
- **Client side** (components, hooks, queries): organized by feature as described above.
- **Contract location:** in Pattern A, schemas live inside the app with the feature until a split is needed (see `docs/10-stack-guidance.md`).

## Cross-cutting rules

- **No cross-feature imports of internals.** Features communicate only through their public surface (`index.ts`). If two features need the same logic, extract it to `shared/` or a package.
- **Shared code is extracted, not duplicated.** Anything used by more than one feature or app moves to `shared/` or `packages/*` (see `docs/01-architecture.md`).
- **Interfaces are the seam.** Every boundary that may change (data access, external services, business rules) is behind an interface. This is what keeps the codebase swappable and testable.
- **The composition root is the only wiring point.** If you need a dependency, receive it through a constructor; do not construct it inline.
