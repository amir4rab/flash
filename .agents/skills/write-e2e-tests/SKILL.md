---
name: write-e2e-tests
description: >
  Workflow for writing Playwright end-to-end tests. Use whenever adding or
  modifying E2E tests for user-visible flows. Covers placement and naming,
  data-testid selectors, what to test, the CI environment, and multi-locale
  runs. Supplements docs/03-tooling.md and docs/06-i18n.md.
---

# Write E2E Tests

E2E tests prove the user-visible behavior of the whole system works. They
run against a real build in CI, never against dev servers, and they test
what the user does — not implementation details. This skill covers the
workflow for writing Playwright E2E tests correctly.

## 1. Pre-flight

Before writing tests, confirm:

- **Flows to cover** — which user-visible flows the change touches. E2E
  covers the critical journeys end to end; unit/integration tests (see the
  `write-tests` skill) cover the details underneath.
- **App level** — E2E tests live in `e2e/` at the app level, one directory
  per app.
- **Related skills** — if the flow involves user-facing text, load the
  `add-i18n-strings` skill; a new endpoint, `add-api-endpoint`.

## 2. Placement and naming

- Tests live in `e2e/` at the app level.
- Naming: `user can <action> when <context>`.

```ts
test("user can create an order when signed in", async ({ page }) => {
  // ...
});
```

## 3. Selectors

- Use `data-testid` attributes for selectors; never select by CSS class or
  by visible text that may change.
- Add the `data-testid` to the component if it does not exist; it is part of
  the change.

```ts
await page.getByTestId("checkout-submit").click();
```

## 4. What to test

- Test **user-visible behavior**, not implementation details. Assert on what
  the user sees and can do, not on internal state, URLs, or DOM structure.
- Cover the happy path and the key failure paths a user can hit (validation
  errors, empty states, authorization failures).
- Keep E2E flows focused; do not duplicate unit/integration coverage at the
  UI level.

## 5. Environment

- E2E tests run against a **real build** in a CI environment, never against
  dev servers.
- The build is produced by the app's build step before the E2E run.
- Tests must be deterministic: seed the data they depend on, and clean up
  after themselves.

## 6. i18n

- E2E tests run in at least the default locale and one additional locale;
  user-visible flows are exercised in both to catch missing translations
  (`docs/06-i18n.md`).
- Assert on i18n keys or translated output, never on hardcoded English
  strings.

## 7. Verification

- Run the E2E suite: `npm run test:e2e` at the root runs every workspace.
- Run the full pipeline before declaring the change done: format, lint,
  typecheck, test, build (`docs/03-tooling.md`).

## Checklist

Before considering the E2E tests done, verify:

- [ ] Tests live in `e2e/` at the app level.
- [ ] Naming follows `user can <action> when <context>`.
- [ ] Selectors use `data-testid`; no CSS-class or visible-text selectors.
- [ ] Tests assert user-visible behavior, not implementation details.
- [ ] Happy path and key failure paths are covered.
- [ ] Tests run against a real build in CI, not dev servers.
- [ ] Flows run in the default locale and at least one additional locale.
- [ ] Full verification pipeline passes (format, lint, typecheck, test,
      build).
