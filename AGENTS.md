# AGENTS.md

This file is the **mandatory entry point** for any agent (human or LLM) working in this repository. Read it in full before doing anything else. It defines the contract every change must satisfy.

## Purpose

This repository is a **framework / template** for accelerating agentic software development. It does not contain an application. It contains the conventions, standards, and workflows that agents must follow when building TypeScript applications on top of it. The goal is long-horizon development: codebases that stay clean, typed, tested, and maintainable over time.

## Non-negotiable rules

1. **Clarify before planning.** If a requirement is ambiguous, open-ended, or has multiple reasonable solutions, ask the user for clarification first. Do not guess. Do not plan around an assumption you have not confirmed. See `docs/08-agent-workflow.md`.
2. **No `any`.** The `any` type is banned. Use `unknown` with narrowing, generics, branded types, or discriminated unions. See `docs/02-typescript.md`.
3. **Strict typing everywhere.** Every interface, function signature, and boundary is explicitly typed. No implicit `any`, no untyped `as` casts.
4. **Lint, typecheck, test, build.** Every change must pass the full verification pipeline before it is considered done. See `docs/03-tooling.md`.
5. **No hardcoded user-facing strings.** All user-facing text goes through the i18n system. See `docs/06-i18n.md`.
6. **Follow the Definition of Done.** A change is not complete until it satisfies every item in `docs/09-definition-of-done.md`.

## Mandatory workflow

Every task follows this loop. Do not skip steps.

1. **Clarify** — read the request, identify ambiguities and open-ended decisions, ask the user before proceeding.
2. **Plan** — write a short plan (files to touch, approach, risks) and confirm it before making changes.
3. **Implement** — make the change following the conventions in `docs/`.
4. **Verify** — run lint, typecheck, tests, and build. Fix everything that fails.
5. **Report** — summarize what changed and how it was verified.

## Before touching an area, read its doc

| Area | Read first |
| --- | --- |
| Repo layout, package boundaries | `docs/01-architecture.md` |
| TypeScript rules | `docs/02-typescript.md` |
| Linting, formatting, testing, validation | `docs/03-tooling.md` |
| REST APIs, error handling | `docs/04-api-design.md` |
| Database, Drizzle, migrations | `docs/05-data-layer.md` |
| Internationalization | `docs/06-i18n.md` |
| Performance | `docs/07-performance.md` |
| Agent workflow | `docs/08-agent-workflow.md` |
| Definition of Done | `docs/09-definition-of-done.md` |
| Framework-specific guidance | `docs/10-stack-guidance.md` |

## When to stop and ask

Stop and ask the user when any of the following is true:

- The requirement is ambiguous or contradictory.
- There are multiple valid designs and no stated preference.
- The change would break an existing convention or boundary.
- The change affects the public API contract, data model, or i18n keys.
- The scope is larger than the request implies.

## Verification commands

The exact commands live in `docs/03-tooling.md`. At minimum, before finishing any change, run:

- the linter
- the typechecker
- the test suite
- the build

If any of these are not configured yet, say so explicitly and ask whether to set them up before proceeding.
