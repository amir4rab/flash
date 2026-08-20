# 08 — Agent Workflow

This document defines the mandatory workflow for every task an agent performs. It exists to prevent the two failure modes that destroy long-horizon codebases: **guessing** and **unverified changes**.

## The loop

Every task follows this loop. Do not skip steps.

```
Clarify → Plan → Implement → Verify → Report
```

### 1. Clarify

Read the request and identify:

- **Ambiguities** — terms, scope, or behavior that could be read multiple ways.
- **Open-ended decisions** — design choices with multiple reasonable answers and no stated preference.
- **Missing context** — information needed to do the work that is not in the request.
- **Contradictions** — requirements that conflict with each other or with existing conventions.

If any of these exist, **ask the user before proceeding**. Do not plan around an assumption you have not confirmed.

Good clarifying questions are specific and offer options:

> "The request says 'add user management'. Should this be (a) a full CRUD admin UI, (b) a read-only user list, or (c) just the API endpoints?"

> "You mentioned 'fast'. Is there a specific performance target, or should I follow the standard conventions in `docs/07-performance.md`?"

### 2. Plan

Write a short plan and confirm it before making changes. A good plan contains:

- **Scope** — what will change and what will not.
- **Files to touch** — specific paths.
- **Approach** — the design decisions and which conventions apply.
- **Risks** — anything that could break, and how it will be caught.
- **Verification** — how the change will be tested.

Keep the plan short. It is a contract for the work, not a design document.

### 3. Implement

- Follow the conventions in `docs/`. Read the relevant doc before touching an area (see the table in `AGENTS.md`).
- Make the smallest change that satisfies the plan.
- Do not add unrelated refactors, renames, or "drive-by" fixes. If you find something that needs fixing, note it and ask.

### 4. Verify

Run the full verification pipeline before declaring the work done:

1. Format check
2. Lint
3. Typecheck
4. Tests (unit/integration, and e2e where applicable)
5. Build

Fix everything that fails. If a failure reveals a flaw in the plan, go back to the plan step and confirm the revised approach.

### 5. Report

Summarize:

- What changed (files and behavior).
- How it was verified (which commands passed).
- Anything left undone or deferred.

## When to stop and ask

Stop and ask the user when any of the following is true:

- The requirement is ambiguous or contradictory.
- There are multiple valid designs and no stated preference.
- The change would break an existing convention or boundary.
- The change affects the public API contract, data model, or i18n keys.
- The scope is larger than the request implies.
- The change requires a trade-off the user has not approved (performance, security, UX).

## What agents must never do

- **Never guess.** If you are unsure, ask.
- **Never skip verification.** A change that has not passed lint, typecheck, tests, and build is not done.
- **Never use `any`** or weaken types to make a change compile.
- **Never hardcode user-facing strings.**
- **Never commit secrets** or log sensitive data.
- **Never make changes outside the agreed scope** without asking.

## Working with this framework

- This repository is itself a template. Changes to `AGENTS.md` or `docs/` affect every project built on it — treat them as public-contract changes and confirm before making them.
- When adding a new convention, add it to the relevant `docs/` file and to the summary table in `AGENTS.md` if it is a new area.
