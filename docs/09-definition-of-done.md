# 09 — Definition of Done

A change is **done** only when every item below is satisfied. If any item cannot be satisfied, say so explicitly and explain why before declaring the work complete.

## Checklist

### Correctness

- [ ] The change does what the request asked for, as confirmed in the plan.
- [ ] Edge cases are handled (empty input, missing data, invalid input, concurrency).
- [ ] Errors are handled explicitly and typed; nothing is silently swallowed.

### Typing

- [ ] No `any` anywhere in the change.
- [ ] All new interfaces, function signatures, and boundaries are explicitly typed.
- [ ] No untyped `as` casts; casts are justified and documented.
- [ ] `tsc --noEmit` passes with strict settings.

### Style and structure

- [ ] The change follows the conventions in `docs/` for the areas it touches.
- [ ] Code is formatted (Prettier) and linted (ESLint) with zero errors.
- [ ] The change is scoped: no unrelated refactors or drive-by fixes.
- [ ] New shared code lives in the right package; no duplication across apps.

### Testing

- [ ] Unit/integration tests cover the new behavior, including failure paths.
- [ ] E2E tests cover user-visible flows where applicable.
- [ ] The full test suite passes.
- [ ] Coverage thresholds are met (see `docs/03-tooling.md`).

### Validation

- [ ] All external boundaries are validated with Zod (API, env vars, config).
- [ ] Validation failures produce the standard error shape.

### i18n

- [ ] No hardcoded user-facing strings; all user-facing text uses i18n keys.
- [ ] New keys are added to the default locale and all other locale files.
- [ ] Locale key sets are consistent across locales.

### Performance

- [ ] No obvious performance regressions (N+1 queries, unbounded lists, missing indexes).
- [ ] Bundle budget is respected where applicable.

### Security

- [ ] No secrets committed or logged.
- [ ] Authorization is enforced server-side; the client is not trusted.
- [ ] Input is validated before use.

### Documentation

- [ ] Public API contract, data model, or i18n key changes are reflected in the relevant docs.
- [ ] New environment variables are documented in `.env.example`.

### Verification pipeline

- [ ] `format:check` passes.
- [ ] `lint` passes.
- [ ] `typecheck` passes.
- [ ] `test` passes (and `test:e2e` where applicable).
- [ ] `build` passes.

## If something cannot be satisfied

If a checklist item cannot be met (e.g. a test cannot be written, or a convention conflicts with the request), stop and ask the user. Do not silently skip items.
