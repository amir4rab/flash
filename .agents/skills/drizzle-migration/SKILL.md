---
name: drizzle-migration
description: >
  Workflow for changing a Drizzle schema and generating/applying migrations.
  Use whenever adding or modifying a table, column, index, or relation in a
  Drizzle schema, or when running db:generate / db:migrate. Covers SQL
  review, the destructive-change gate, index discipline, and repository
  impact. Supplements docs/05-data-layer.md.
---

# Drizzle Migration

All schema changes go through Drizzle migrations. Never alter the database by
hand. The workflow is fixed: schema → generate → **review the SQL** → apply
→ commit. The destructive-change gate sits between generate and apply.

## 1. Schema rules recap

Edit the schema in `src/db/schema/<domain>.ts`. Full rules live in
`docs/05-data-layer.md`; the essentials:

- Every table: primary key (`id` uuid or serial), `createdAt` and
  `updatedAt` timestamps.
- Column names `snake_case`, TypeScript identifiers `camelCase` (Drizzle's
  `camelCase` naming option).
- Foreign keys declared in the schema so relations type correctly.
- Explicit column types everywhere; no implicit `any`.

```ts
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

## 2. Workflow

Run in this order; do not skip step 3:

```bash
# 1. Edit the schema in src/db/schema/
npm run db:generate   # 2. Generate the migration
# 3. READ the generated SQL (see below)
npm run db:migrate    # 4. Apply it
# 5. Commit the migration file together with the schema change
```

### Reviewing the generated SQL

Open the generated migration file and check for:

- **Unexpected drops** — a renamed column generates a drop + add, losing
  data. A rename must be written as a migration that preserves data.
- **Type changes** — an in-place type change can fail or truncate on
  existing data.
- **NOT NULL additions** — fail on tables with existing rows unless a
  default or backfill is included.
- **Missing index** — if the change adds a column used in `WHERE`,
  `ORDER BY`, or `JOIN`, the index belongs in this same migration.

If any of these appear and were not planned, stop and apply the
destructive-change gate below.

## 3. Destructive-change gate

Dropping columns or tables, changing types, and adding NOT NULL constraints
**require a data-migration plan confirmed with the user before applying** —
never silently.

When asking, present:

- What is destroyed or at risk, and on which tables.
- The plan: backfill (migrate data first), staged rollout (add nullable →
  backfill → enforce), or explicit data-loss sign-off.
- The recommended option.

Only run `db:migrate` after the user confirms. Destructive changes must also
be called out in the plan phase before any code is written.

## 4. Index discipline

- Every column used in `WHERE`, `ORDER BY`, or `JOIN` clauses gets an index,
  declared in the schema — added in the same migration as the column.
- Verify slow queries with `EXPLAIN` before and after.
- New relations used with Drizzle's `with` (eager loading) should not
  introduce N+1; check the query plan (`docs/07-performance.md`).

## 5. Repository and service impact

A schema change is not done at the migration step:

- Update the repository interface and its typed methods
  (`findById`, `create`, ...) — Drizzle internals never leak past the
  repository.
- Update services only through repository signatures.
- Select only needed columns; keep `limit` on list queries.
- Raw SQL only when the query builder cannot express the query, wrapped in a
  typed function with a documented reason.

If the change alters API response shapes, load the `change-shared-contract`
skill — the shared Zod schemas may be a breaking contract change requiring
confirmation.

## 6. Testing

- Repositories are tested against a real database (test or containerized
  Postgres), **never mocked and never production**.
- Use transaction-per-test (rollback) or truncate between tests.
- New/changed columns get explicit repository test coverage, including
  failure paths (unique violations, missing rows).
- Services are tested with mocked repositories, unaffected by the migration.

## Checklist

Before declaring the migration done, verify:

- [ ] Schema change follows the schema rules (PK, timestamps, naming, FKs).
- [ ] Generated SQL was read and reviewed, not just applied.
- [ ] Destructive change (drop/type/NOT NULL) had a confirmed
      data-migration plan.
- [ ] Indexes for new filter/sort/join columns included in the same
      migration.
- [ ] Migration file committed together with the schema change.
- [ ] Repository interfaces and typed queries updated.
- [ ] Repository tests updated and passing against a real test database.
- [ ] Full verification pipeline passes.
