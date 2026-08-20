# 05 — Data Layer

This document defines the conventions for database access. The default ORM is **Drizzle**.

## ORM: Drizzle

- Drizzle is the default ORM. It provides typed queries and a migration workflow without hiding SQL.
- Do not introduce a second ORM. If a specific need is not met by Drizzle, discuss it before adding another tool.

## Schema

- The schema is defined in TypeScript with Drizzle's schema builder, in a dedicated module per domain (e.g. `src/db/schema/users.ts`).
- Every table has:
  - A primary key (`id` as `uuid` or `serial`).
  - `createdAt` and `updatedAt` timestamps.
  - Explicit column types; no implicit `any`.
- Foreign keys are declared in the schema so Drizzle can type the relations.
- Table and column names are `snake_case`; the TypeScript identifiers are `camelCase` via Drizzle's `camelCase` naming option.

### Example

```ts
import { pgTable, uuid, text, timestamp } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

## Migrations

- All schema changes go through Drizzle migrations. Never alter the database by hand.
- Workflow:
  1. Change the schema in `src/db/schema/`.
  2. Generate a migration: `npm run db:generate`.
  3. Review the generated SQL before applying it.
  4. Apply it: `npm run db:migrate`.
- Migrations are committed to the repository and are part of the code review.
- Destructive changes (dropping columns, changing types) require a plan for data migration and must be called out in the plan phase.

## Typed queries

- Use Drizzle's query builder for reads and writes. It returns fully typed results.
- Prefer the relational query API (`db.query.users.findMany({ with: { posts: true } })`) for nested reads; it is typed and avoids manual joins.
- Raw SQL is allowed only when the query builder cannot express the query, and must be wrapped in a typed function with a documented reason.

## Repository pattern

- Database access is isolated behind **repositories** — one module per aggregate/domain.
- Repositories expose typed methods (`findById`, `findByEmail`, `create`, `update`, `delete`) and never leak Drizzle internals to callers.
- Business logic lives in **services**; services depend on repositories, not on the database directly.
- This keeps the data layer swappable and testable.

### Example

```ts
export interface UserRepository {
  findById(id: UserId): Promise<User | undefined>;
  findByEmail(email: string): Promise<User | undefined>;
  create(input: CreateUserInput): Promise<User>;
  update(id: UserId, patch: UpdateUserInput): Promise<User>;
}
```

## Transactions

- Multi-step operations that must be atomic run inside a transaction.
- Repositories expose a way to run a unit of work in a transaction; services compose them.
- Never leave a transaction open across an HTTP request boundary.

## Query performance

- Avoid N+1 queries: use Drizzle's `with` relations or explicit joins.
- Add indexes for columns used in `WHERE`, `ORDER BY`, and `JOIN` clauses. Indexes are declared in the schema.
- Use `limit` on all list queries; pagination is mandatory (see `docs/04-api-design.md`).
- Select only the columns you need; avoid `select *` in production code.
- See `docs/07-performance.md` for more.

## Testing the data layer

- Repositories are tested against a real database (a test database or a containerized Postgres), not mocked.
- Services are tested with mocked repositories to verify business logic in isolation.
- Use a transaction-per-test pattern: wrap each test in a transaction and roll back, or truncate tables between tests.
- Never run tests against the production database.

## Environment

- Database connection settings come from validated environment variables (see `docs/03-tooling.md`).
- Connection pooling is configured for the deployment target (e.g. PgBouncer for serverless).
