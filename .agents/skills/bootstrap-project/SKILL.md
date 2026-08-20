---
name: bootstrap-project
description: >
  Workflow for initializing a new project on this framework. Use whenever a
  project is started or scaffolded. Reads the project declaration (type and
  goal), applies the type's conventions from docs/12-project-types.md, and
  tailors the docs/ folder — keeping, modifying, and removing docs to fit the
  project and limit context size. Supplements docs/12-project-types.md.
---

# Bootstrap Project

Initialize a new project by turning the user's declaration into a tailored
framework. The goal is a `docs/` folder and `AGENTS.md` that describe exactly
this project — nothing more, nothing less — so future agent sessions carry
minimal context.

## 1. Read the declaration

The user states the project type and goal, for example:

> "This project is a contracted software for a third party with the goal of
> developing an internal dashboard with X and Y features."

Extract from the declaration:

- **Type** — first-party or contracted. If it is not stated or is ambiguous,
  ask before proceeding. Do not guess.
- **Goal** — what the project does and who uses it.
- **Features** — the capabilities implied (dashboard, API, auth, billing, ...).
- **Stack signals** — anything that implies or rules out a pattern, database,
  UI, or API (see `docs/10-stack-guidance.md`).

## 2. Apply the type's conventions

Read `docs/12-project-types.md` and apply the conventions for the declared
type:

- **First-party** — long-horizon maintainability: strict convention adherence,
  scalability and evolution paths, package extraction discipline, freedom to
  evolve internal APIs, internal dependencies allowed.
- **Contracted** — handover readiness: handover documentation set, self-
  containment rules, versioning and changelog discipline, licensing and
  ownership notes. The handover checklist must also include the private-IP
  removal checklist and the git-history decision (see
  `docs/12-project-types.md`).

## 3. Tailor the `docs/` folder

Decide for each doc whether to **keep**, **modify**, or **remove** it. The
decision is driven by the project type **and** its technical characteristics.
Remove unneeded docs to limit context size; modify docs that need project-
specific tailoring.

### Decision table

| Doc | Keep when | Remove when |
| --- | --- | --- |
| `01-architecture.md` | Always — every project is a monorepo | Never |
| `02-typescript.md` | Always — strict typing is non-negotiable | Never |
| `03-tooling.md` | Always — the verification pipeline applies to every project | Never |
| `04-api-design.md` | The project exposes a REST API | No API (e.g. a pure internal library or CLI) |
| `05-data-layer.md` | The project has a database | No database |
| `06-i18n.md` | The project has user-facing UI | No UI (e.g. backend-only or library) |
| `07-performance.md` | Always — performance rules apply broadly | Never |
| `08-agent-workflow.md` | Always — the workflow is mandatory | Never |
| `09-definition-of-done.md` | Always — the DoD applies to every change | Never |
| `10-stack-guidance.md` | The project uses one of the supported stacks | A custom stack is chosen (then modify it) |
| `11-code-organization.md` | The project has app code organized by feature | No app code (e.g. a pure library) |
| `12-project-types.md` | Always — but **trim it** to the declared type (step 4) | Never |

### Modifying a doc

When a doc needs project-specific tailoring, edit it in place rather than
adding a parallel doc. Examples:

- `10-stack-guidance.md` — pin the chosen pattern (A or B) and remove the
  other pattern's section.
- `04-api-design.md` — remove endpoint conventions for resources the project
  does not have.
- `05-data-layer.md` — remove database engines the project does not use.

Do not remove a doc's non-negotiable rules (strict typing, verification
pipeline, i18n where UI exists). If a removal would break a non-negotiable
rule, keep the doc and ask the user.

## 4. Trim `12-project-types.md`

Since the project type is rarely changed after initialization, trim
`docs/12-project-types.md` to only the declared type's conventions:

- Remove the other type's section.
- Keep the "Bootstrapping a project" section only if it is still useful as
  reference; otherwise remove it.
- Keep the "When to stop and ask" section.

## 5. Update `AGENTS.md`

After tailoring, update the project's `AGENTS.md`:

- Remove rows from the "Before touching an area, read its doc" table for docs
  that were removed.
- Keep the table consistent with the remaining `docs/` files.
- Add a **Project type** line near the top recording the declared type and the
  tailoring decisions, so future sessions have the context without re-reading
  the full docs.

## 6. Record the declaration

Record the declared type, goal, and tailoring decisions in the project's
README and `AGENTS.md`. This is the context future agent sessions rely on.

## Checklist

Before considering the bootstrap done, verify:

- [ ] Project type was stated or confirmed with the user; not guessed.
- [ ] The type's conventions from `docs/12-project-types.md` are applied.
- [ ] `docs/` folder is tailored: unneeded docs removed, needed docs kept,
      project-specific docs modified in place.
- [ ] `12-project-types.md` is trimmed to the declared type only.
- [ ] `AGENTS.md` table matches the remaining `docs/` files.
- [ ] Declared type, goal, and tailoring decisions recorded in README and
      `AGENTS.md`.
- [ ] No non-negotiable rules (strict typing, verification pipeline, i18n)
      were removed.
