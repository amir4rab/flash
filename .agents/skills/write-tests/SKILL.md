---
name: write-tests
description: >
  Workflow for writing Vitest unit and integration tests. Use whenever adding
  or modifying tests for services, repositories, API endpoints, or components.
  Covers test placement and naming, mocking discipline, layer-specific
  guidance, failure paths, coverage thresholds, and i18n assertions.
  Supplements docs/03-tooling.md.
---

# Write Tests

Tests are part of the change, not an afterthought. A change without tests for
its new behavior — including failure paths — is not done. This skill covers
the workflow for writing Vitest unit and integration tests correctly: right
placement, right naming, mocking only at boundaries, and coverage of the
paths that matter.

## 1. Pre-flight

Before writing tests, confirm:

- **Layer under test** — service, repository, API endpoint, or component.
  Each has its own guidance (section 4). If the change spans layers, test
  each layer at its own level; do not test everything through one endpoint.
- **Unit vs integration** — unit tests exercise one unit in isolation
  (mocked boundaries); integration tests exercise real boundaries (a real
  test database, the framework's test client). Both are Vitest.
- **Coverage thresholds** — 80% lines for `packages/*`, 70% for `apps/*`
  (`docs/03-tooling.md`). New behavior must not drag coverage below the
  threshold.
- **Related skills** — if the tests cover a new endpoint, load the
  `add-api-endpoint` skill; a schema change, `change-shared-contract`; a
  migration, `drizzle-migration`; user-facing text, `add-i18n-strings`.

## 2. Placement and naming

- Tests live next to the code they test: `src/foo.test.ts` or
  `src/foo.spec.ts`. No separate test directory. In feature-organized apps,
  tests live inside the feature slice (`src/features/<feature>/`), see
  `docs/11-code-organization.md`.
- Naming: `should <expected behavior> when <condition>`.

```ts
describe("UserService", () => {
  it("should return the user when the id exists", async () => {
    // ...
  });

  it("should throw a typed error when the user is not found", async () => {
    // ...
  });
});
```

- Use `describe` / `it` / `expect` from Vitest. No custom assertion
  frameworks.

## 3. Mocking discipline

- **Mock at boundaries only** — HTTP, database, time. Never mock the code
  under test.
- Services are tested with **mocked repositories** to verify business logic
  in isolation.
- Repositories are tested against a **real database** (test or containerized
  Postgres), never mocked and never production.
- Use a transaction-per-test pattern (wrap each test in a transaction and
  roll back) or truncate tables between tests.
- Mock time with Vitest's fake timers when a test depends on `Date` or
  `setTimeout`; restore after each test.

## 4. Layer-specific guidance

### Services

- Mock the repository **interface**, not the database or the implementation.
- Cover the business rules: happy path, each branch, and each typed error.
- Assert on the returned value and on what the service called on the
  repository (e.g. `create` called with the mapped input).

### Repositories

- Run against a real test database; never mock the database.
- Cover success, missing rows, and failure paths (unique violations,
  constraint errors).
- Assert on the returned shape, not on SQL.

### API endpoints

- Use Vitest + supertest or the framework's test client.
- Every endpoint is covered by three tests at minimum:
  1. **Success** — expected status and response shape.
  2. **Validation failure** — invalid input returns `400` with the error
     envelope and field-level `details`.
  3. **Authorization failure** — missing/insufficient credentials return
     `401`/`403`.
- See the `add-api-endpoint` skill for the full endpoint contract.

### Components

- Assert on i18n keys or rendered translated output, never on hardcoded
  English strings.
- Cover the component's states: default, loading, empty, error, and any
  interactive behavior.

## 5. Failure paths and edge cases

Every new behavior gets failure-path coverage, not just the happy path:

- Empty input and missing data.
- Invalid input (validation failures).
- Concurrency where relevant (duplicate creation, conflicting updates).
- Errors are asserted as typed errors; nothing is silently swallowed.

## 6. i18n in tests

- Tests assert on i18n keys or rendered translated output, never on
  hardcoded English strings.
- The locale-parity test verifies every locale file has the same key set as
  the default locale. Extend it if new namespaces are added
  (`docs/06-i18n.md`).

## 7. Verification

- Run the test suite: `npm run test` at the root runs every workspace.
- Run the full pipeline before declaring the change done: format, lint,
  typecheck, test, build (`docs/03-tooling.md`).

## Checklist

Before considering the tests done, verify:

- [ ] Tests live next to the code they test (`src/foo.test.ts`).
- [ ] Naming follows `should <expected behavior> when <condition>`.
- [ ] Mocks are at boundaries only; the code under test is never mocked.
- [ ] Services tested with mocked repositories; repositories tested against
      a real test database.
- [ ] Endpoint tests cover success, validation failure, and authorization
      failure.
- [ ] Failure paths and edge cases are covered, not just the happy path.
- [ ] Tests assert on i18n keys or translated output, not English strings.
- [ ] Coverage thresholds are met (80% `packages/*`, 70% `apps/*`).
- [ ] Full verification pipeline passes (format, lint, typecheck, test,
      build).
