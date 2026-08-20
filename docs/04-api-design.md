# 04 — API Design

This document defines the conventions for JSON APIs. It applies to the JSON-backend + SPA pattern (`apps/api`) and to any server routes that expose data to clients.

## Default: REST

REST is the default API style. tRPC and GraphQL are acceptable alternatives only when the team explicitly chooses them and documents the decision; the validation and error conventions below still apply.

## Resources and verbs

- Model the API around **resources** (nouns), not actions.
- Use the standard verbs:

| Verb | Purpose | Idempotent |
| --- | --- | --- |
| `GET` | Read a collection or a single resource | Yes |
| `POST` | Create a resource or trigger a non-idempotent action | No |
| `PUT` | Replace a resource | Yes |
| `PATCH` | Partially update a resource | No |
| `DELETE` | Delete a resource | Yes |

- Collection endpoints: `GET /users`, `POST /users`.
- Item endpoints: `GET /users/:id`, `PUT /users/:id`, `PATCH /users/:id`, `DELETE /users/:id`.
- Nested resources use the parent path: `GET /users/:id/posts`.
- Actions that do not map to CRUD use a sub-resource: `POST /orders/:id/cancel` (not `POST /orders/cancel`).

## Status codes

| Code | When |
| --- | --- |
| `200` | Successful `GET`, `PUT`, `PATCH`, `DELETE` |
| `201` | Successful `POST` (include `Location` header) |
| `204` | Successful `DELETE` with no body |
| `400` | Malformed request body or query (validation failure) |
| `401` | Missing or invalid authentication |
| `403` | Authenticated but not authorized |
| `404` | Resource not found |
| `409` | Conflict (e.g. duplicate unique value) |
| `422` | Semantically invalid request (business rule violation) |
| `429` | Rate limited |
| `500` | Unexpected server error (never leak details) |

## Error envelope

Every error response uses the same shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request payload",
    "details": [
      { "path": "email", "message": "must be a valid email" }
    ]
  }
}
```

- `code` is a stable machine-readable string (e.g. `NOT_FOUND`, `UNAUTHORIZED`, `VALIDATION_ERROR`). It is part of the API contract and must not change without a version bump.
- `message` is a human-readable summary. It is developer-facing, not user-facing; the client maps `code` to an i18n key (see `docs/06-i18n.md`).
- `details` is optional and used for field-level validation errors.

## Validation at the boundary

- Every request body, query string, and path parameter is validated with Zod before it reaches business logic.
- Schemas live in `packages/shared` and are shared with the client so both sides agree on the contract.
- Validation failure returns `400` with the error envelope above.
- Never trust unvalidated input. `unknown` in, typed value out.

## Request/response conventions

- **JSON only.** Content-Type `application/json` on requests; responses are JSON.
- **Pagination** — list endpoints return a page envelope:

```json
{
  "data": [],
  "pagination": { "page": 1, "pageSize": 20, "total": 137 }
}
```

  - Query params: `page` (1-based) and `pageSize` (default 20, max 100).
  - Cursor-based pagination is preferred for large, append-only collections.

#### Cursor-based pagination contract

Cursor-based (keyset) pagination is the default for large or append-only collections. It is O(1) per page regardless of depth and does not skip or duplicate rows when the collection changes between requests. See `docs/07-performance.md` for when to choose it over offset pagination.

- **Request params:** `cursor` (opaque token) and `limit` (default 20, max 100). The first request omits `cursor`.
- **Response envelope:**

```json
{
  "data": [],
  "pagination": { "nextCursor": "eyJjcmVhdGVkQXQiOiIuLi4iLCJpZCI6Ii4uLiJ9", "hasMore": true }
}
```

  - `nextCursor` is present only when there are more rows; `hasMore` mirrors that for convenience.
  - When `hasMore` is `false`, the client stops requesting pages.

- **Cursor encoding:** the cursor is opaque to the client — a base64url-encoded tuple of the sort key and the row `id` (e.g. `(createdAt, id)`). Never expose raw database values or accept a client-supplied sort key.
- **Sorting:** cursor endpoints must be ordered by a stable, unique sort key — a timestamp plus the `id` as a tiebreaker (e.g. `ORDER BY createdAt DESC, id DESC`). The server derives the keyset predicate from the cursor; the client never constructs it.
- **No `total`:** cursor pagination does not return a total count. If a count is required, expose it as a separate endpoint or field, never by scanning the whole collection per page.
- **Backward compatibility:** `cursor`/`limit` are additive. An endpoint may support both offset and cursor modes, but a given endpoint should pick one and document it.

- **Filtering** — use query params: `GET /users?role=admin&status=active`.
- **Sorting** — use `sort` with a field and optional direction: `sort=-createdAt` (descending) or `sort=createdAt` (ascending).
- **Field selection** — use `fields` to limit returned fields when payloads are large: `fields=id,name,email`.
- **Dates** — always ISO 8601 UTC (`2026-08-19T12:00:00.000Z`). Never locale-dependent formats.

## Versioning

- Version the API with a URL prefix: `/api/v1/...`.
- Breaking changes (renamed fields, removed endpoints, changed error codes) require a new major version.
- Additive changes (new fields, new endpoints) do not require a version bump.

## Idempotency

- `PUT` and `DELETE` are idempotent by definition.
- For `POST` operations that create resources and may be retried (payments, orders), accept an `Idempotency-Key` header. If the key is seen again, return the original result instead of creating a duplicate.

## Authentication and authorization

- Authentication is handled at the framework level (sessions, JWT, or OAuth) and is validated before the route handler runs.
- Authorization is enforced per-route with explicit checks. Never rely on the client to hide data.
- All authorization failures return `403` with the error envelope.

## Rate limiting

- Public endpoints are rate limited. Exceeded limits return `429` with a `Retry-After` header.

## Logging

- Log structured JSON: request id, method, path, status, duration, and a correlation id.
- Never log request bodies, tokens, passwords, or other secrets.
- Errors are logged with their `code` and a stack trace at the server; the client only ever sees the error envelope.

## Testing the API

- Every endpoint has integration tests (Vitest + supertest or the framework's test client) covering success, validation failure, and authorization failure.
- E2E flows are covered by Playwright (see `docs/03-tooling.md`).
