---
name: add-api-endpoint
description: >
  End-to-end workflow for adding or modifying a REST endpoint, route handler,
  or API resource. Use whenever creating a new endpoint, changing an existing
  one, or exposing data to a client. Covers the Zod contract in
  packages/shared, layering, status codes, the error envelope, authorization,
  and required tests. Supplements docs/04-api-design.md.
---

# Add API Endpoint

Create REST endpoints that satisfy the API contract, validation, and testing
conventions in one pass. The order of steps matters: contract first, then
layers, then tests.

## 1. Pre-flight

Before writing code, confirm:

- **Resource name** — the endpoint is modeled around a noun (`/users`,
  `/orders/:id`), not a verb.
- **Pattern** — is this a Pattern A route handler (Next.js route handler,
  SvelteKit `+server.ts`, Nitro route, Remix resource route) or a Pattern B
  `apps/api` route (Fastify/Hono)? See `docs/10-stack-guidance.md`.
- **Resource vs action** — CRUD maps to the standard verbs. An action that is
  not CRUD is a sub-resource: `POST /orders/:id/cancel`, never
  `POST /orders/cancel`.
- **Version** — new endpoints go under the current version prefix
  (`/api/v1/...`).

If the endpoint requires a new table or relation, load the
`drizzle-migration` skill first.

## 2. Contract first

Define the contract in `packages/shared` before touching app code. It is the
single source of truth for both sides of the API.

- A Zod schema for the request (body, query, path params) and one for the
  response, per endpoint.
- Export inferred types so both server and client use them:
  `export type CreateUserRequest = z.infer<typeof CreateUserRequest>;`
- Error `code`s are stable machine-readable strings (`NOT_FOUND`,
  `VALIDATION_ERROR`) — they are part of the contract.
- Pagination on list endpoints: the page envelope
  (`{ data, pagination: { page, pageSize, total } }`), `page` 1-based,
  `pageSize` default 20, max 100.

If this modifies an existing schema, field, or error code, load the
`change-shared-contract` skill and classify the change before proceeding.

## 3. Layers

Code is organized by feature (vertical DDD, see `docs/11-code-organization.md` and the `add-feature-slice` skill).
The endpoint's route, controller, service, and repository live **inside the
feature slice** (`src/features/<feature>/`), not in top-level layer folders.

**Pattern B (`apps/api`)** follows the structure
route → controller → service → repository within the feature slice:

- **Route** — HTTP concerns only: parse, validate, delegate, set status.
- **Controller** — maps validated input to service calls and results to
  responses.
- **Service** — business logic; depends on the repository **interface**, never
  on the database directly.
- **Repository** — the only layer that touches Drizzle; typed methods, no
  Drizzle internals leaked upward. See `docs/05-data-layer.md`.

Services and repositories declare interfaces; implementations are wired in the
composition root (`src/composition-root.ts`). Never `new` a service or
repository inside a route or controller — receive it through the constructor.

**Pattern A** keeps the same discipline inside the framework's route
handler: validate with Zod, call a service, never query the database from
client components.

## 4. Conventions

### Verbs and status codes

| Verb | Purpose | Success code |
| --- | --- | --- |
| `GET` | Read collection or item | `200` |
| `POST` | Create / non-idempotent action | `201` + `Location` header |
| `PUT` | Replace | `200` |
| `PATCH` | Partial update | `200` |
| `DELETE` | Delete | `204` (no body) |

Failure codes: `400` validation, `401` unauthenticated, `403` unauthorized,
`404` not found, `409` conflict, `422` business rule, `429` rate limited,
`500` unexpected (never leak details).

### Error envelope

Every error response uses the same shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request payload",
    "details": [{ "path": "email", "message": "must be a valid email" }]
  }
}
```

`message` is developer-facing; the client maps `code` to an i18n key — never
render `message` to users.

### Request/response rules

- JSON only; dates always ISO 8601 UTC.
- Validation failure of body/query/params returns `400` with the envelope —
  validate before any business logic runs.
- Filtering via query params, sorting via `sort=-createdAt`, field selection
  via `fields=id,name,email`.

## 5. Cross-cutting concerns

- **Authorization is enforced per-route on the server.** Never rely on the
  client to hide data. Failures return `403` with the envelope.
- **Rate limit** public endpoints; exceeded limits return `429` with
  `Retry-After`.
- **Idempotency** — retryable resource-creating POSTs (payments, orders)
  accept an `Idempotency-Key` header; a repeated key returns the original
  result.
- **Logging** — structured JSON (request id, method, path, status, duration).
  Never log request bodies, tokens, or secrets.
- **Performance** — paginate all lists, select only needed columns, watch for
  N+1 (`docs/07-performance.md`).

## 6. Required tests

Every endpoint has integration tests (Vitest + supertest or the framework's
test client) covering:

1. **Success** — the happy path returns the expected status and shape.
2. **Validation failure** — invalid input returns `400` with the error
   envelope and field-level `details`.
3. **Authorization failure** — missing/insufficient credentials return
   `401`/`403`.

Test naming: `should <expected behavior> when <condition>`. E2E coverage for
user-visible flows goes to Playwright (`docs/03-tooling.md`).

## Checklist

Before considering the endpoint done, verify:

- [ ] Zod schemas for request and response live in `packages/shared`.
- [ ] Both server and client types derive from those schemas (`z.infer`).
- [ ] Endpoint is under `/api/v1`, modeled around a resource.
- [ ] Route, controller, service, and repository live inside the feature slice
      (`src/features/<feature>/`), not in top-level layer folders.
- [ ] Service and repository expose interfaces; wired in the composition root.
- [ ] Correct success status code (`201` + `Location` for POST, `204` for
      DELETE).
- [ ] All error responses use the standard envelope with stable codes.
- [ ] Input validated with Zod before reaching business logic.
- [ ] Authorization enforced server-side on the route.
- [ ] List endpoints paginate; queries select only needed columns.
- [ ] Integration tests cover success, validation failure, and authorization
      failure.
- [ ] No secrets or request bodies logged.
- [ ] Full verification pipeline passes (format, lint, typecheck, test,
      build).
