---
name: change-shared-contract
description: >
  Workflow for modifying the API contract in packages/shared — schemas,
  fields, error codes, or endpoint paths. Use whenever changing a Zod schema
  that an API and its client share. Covers classifying additive vs breaking
  changes, the confirmation gate, and updating both sides atomically.
  Supplements docs/04-api-design.md and docs/10-stack-guidance.md.
---

# Change Shared Contract

`packages/shared` holds the Zod schemas that define the API contract. The
backend validates requests with them; the frontend types its API client from
them. A schema change is a **public-contract change**: follow this skill so
the two sides never disagree.

## 1. Classify the change

Before editing any schema, classify the change. When in doubt, treat it as
breaking.

| Change | Classification |
| --- | --- |
| New optional response field | Additive |
| New endpoint / new schema | Additive |
| New optional request field with a default | Additive |
| New **required** request or response field | Breaking |
| Renaming or removing a field | Breaking |
| Narrowing a type (`string` → email format) | Breaking |
| Widening a type (`string` → `string \| number`) | Breaking |
| Changing an error `code` | Breaking |
| Changing an endpoint path or method | Breaking |
| Changing the pagination/error envelope shape | Breaking |

Additive changes do not require a version bump. Breaking changes require a
new major API version (see step 3).

## 2. Confirmation gate

Breaking changes — and any rename of error codes or i18n keys derived from
them — **must be confirmed with the user before implementation**. This is a
stop-and-ask situation per `docs/08-agent-workflow.md`.

When asking, present:

- What breaks and for whom (which clients/endpoints consume the schema).
- The migration options: version bump, dual-shape response, or removal date.
- The recommended option.

Do not start editing schemas while a breaking change is unconfirmed.

## 3. Versioning

- Breaking changes go to a new URL prefix: `/api/v1/...` → `/api/v2/...`.
- Keep the old version's schemas and routes until the migration plan is
  agreed; removal is its own confirmed change.
- Additive changes land in the current version with no bump.
- If the change stems from a data-model change, load the `drizzle-migration`
  skill and run its destructive-change gate in the same plan.

## 4. Same-change rule

Backend and frontend update **together, in one change**:

1. Edit the schema in `packages/shared`.
2. Update the backend route/controller/service to the new shape.
3. Update the frontend API client and every consumer of the inferred type.
4. Update both sides' tests in the same change.

Never merge one side without the other. `npm run typecheck` across the
workspace is the guardrail: a stale consumer fails typecheck, not runtime.

## 5. Verification

Beyond the full pipeline (format, lint, typecheck, test, build):

- **Rejection paths** — integration tests assert the new schema rejects
  invalid input with `400` + the error envelope, not just that valid input
  passes.
- **Backward compatibility** (additive changes) — a test parses a payload in
  the old shape (without the new optional field) and succeeds.
- **Error codes** — if codes changed, client-side i18n mappings are updated
  in the same change (`docs/06-i18n.md`).
- **Docs** — contract-level changes are reflected in the relevant `docs/`
  and in `README` references where applicable.

## Checklist

Before declaring the contract change done, verify:

- [ ] Change classified as additive or breaking; breaking classified
      correctly.
- [ ] Breaking change confirmed with the user before implementation.
- [ ] Version bumped (`/api/v2`) or no-bump justified (additive).
- [ ] Backend validation, frontend client types, and all consumers updated
      in the same change.
- [ ] Tests updated: success, rejection, and (if additive) old-shape
      compatibility.
- [ ] Error-code-to-i18n mappings updated if codes changed.
- [ ] Documentation updated where the contract is described.
- [ ] Full verification pipeline passes across the whole workspace.
