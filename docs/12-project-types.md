# 12 — Project Types

This document defines the two project types this framework supports and the conventions that apply to each. Read it when starting a new project and before making changes that are affected by the project's type.

## The two project types

Every project built on this framework is one of two types. The type is declared at project initialization and is **rarely changed afterwards** — treat it as fixed once set.

### First-party tools

Developed **by us, for us**. We are the only users and the only maintainers.

- The customer is internal; there is no external party to hand the code to.
- We own the roadmap, the API surface, and the maintenance burden indefinitely.
- Internal knowledge is assumed; no handover or customer-facing documentation is required.

### Contracted projects

Built **at the request of a third party**, developed by us but used by them. The source code may be **transferred to the customer** at the end of the engagement.

- The customer is external; they may take ownership of the codebase.
- The code must be understandable and runnable by people who were not involved in building it.
- The API surface and behavior are part of a deliverable; stability matters.

## Conventions by type

### First-party — long-horizon maintainability

Because we maintain these projects for years, the emphasis is on staying clean, scalable, and cheap to evolve:

- **Strict convention adherence.** Every rule in `docs/` applies in full. There is no external deadline that justifies shortcuts; debt compounds over the long horizon.
- **Scalability and evolution.** Follow the A→B evolution path (`docs/01-architecture.md`) and the vertical DDD structure (`docs/11-code-organization.md`). Design for growth even when the current scale does not require it.
- **Package extraction discipline.** Extract shared code to `packages/*` as soon as a second consumer appears. Do not duplicate logic across apps.
- **Freedom to evolve.** Internal APIs can change without external notice. Refactoring and renames are encouraged when they improve the codebase.
- **Internal dependencies allowed.** Relying on internal-only infrastructure, packages, or services is acceptable — we control them.

### Contracted — handover readiness

Because ownership may transfer to the customer, the emphasis is on documentation, self-containment, and stability:

- **Handover documentation is required.** The project must be understandable and runnable by a team that did not build it. At minimum:
  - Architecture overview (how the pieces fit, the patterns used).
  - Local setup guide (prerequisites, install, run, test).
  - Deployment guide (environments, infrastructure, release process).
  - API reference (endpoints, contracts, error codes).
  - Data model documentation (tables, relations, migrations).
  - Environment variable documentation (`.env.example` with comments).
  - Decision records (ADRs) for non-obvious choices.
- **Self-containment.** The project must not depend on anything the customer cannot access:
  - No internal-only infrastructure, services, or credentials.
  - No internal-only npm packages; everything must be reproducible outside our org.
  - Secrets and configuration must be documented and replaceable by the customer.
- **Versioning and changelog discipline.** External consumers depend on stability:
  - Strict semantic versioning for the public contract (`packages/shared`).
  - A changelog and release notes for every release.
  - Breaking changes to the API contract are a review decision and must be called out in the plan phase (see `docs/04-api-design.md` and the `change-shared-contract` skill).
- **Licensing and ownership.** Clarify and document the ownership and licensing terms up front:
  - License headers where applicable.
  - Ownership-transfer notes: what is delivered, what is excluded, and what the customer may do with it.
  - A handover checklist that is kept current as the project evolves, including the private-IP removal checklist and the git-history decision below.

### Private IP removal (handover scrubbing)

Before the codebase is transferred, remove everything that is internal to our org and not part of the deliverable. The customer must receive only what they need to run and maintain the product.

- **Agent tooling.** Remove `.agents/` (skills, agent configs), `AGENTS.md`, and any agent-specific configuration (`.opencode/`, `opencode.json`). These encode our internal workflows, not the product.
- **Internal process docs.** Remove or trim agent-process docs that do not describe the delivered system: `08-agent-workflow.md`, `09-definition-of-done.md`, and `12-project-types.md` (or the trimmed project-type record). Keep only docs that help the customer run and maintain the product.
- **Internal references.** Remove internal-only URLs, credentials, service names, and infrastructure references from code, configs, and docs.
- **CI/CD configs.** Remove or rewrite CI/CD configs that reference internal infrastructure, credentials, or services the customer cannot access.
- **Internal naming.** Replace internal org scopes and package names (e.g. `@acme/...`) with names the customer owns.
- **ADRs.** Keep only decision records that explain product-relevant choices; remove those that reveal internal process or trade-offs not relevant to the customer.
- **Framework residue.** Remove framework docs and conventions that do not apply to the delivered project (see the `bootstrap-project` skill for tailoring).

### Git history

By default, deliver a **clean repository**: a fresh repo containing the final state, or a squashed history. The full development history is not part of the deliverable and typically reveals internal process, timelines, and mistakes.

- **Default — clean repo.** The customer receives the final state without the development history.
- **Full history only when the contract requires it.** If the contract demands auditability or traceability, the history must be scrubbed first (e.g. with `git filter-repo`) — secrets and internal references persist in history even after they are removed from the working tree.
- **Record the decision.** The handover notes must state which option was chosen and, if history is delivered, how it was scrubbed.

## Bootstrapping a project

At project initialization the user states the type and the goal, for example:

> "This project is a contracted software for a third party with the goal of developing an internal dashboard with X and Y features."

The agent then:

1. **Identifies the type** (first-party or contracted) from the declaration.
2. **Applies the type's conventions** from this document.
3. **Tailors the `docs/` folder** to the project — keeping the docs that apply, modifying ones that need project-specific tailoring, and removing unneeded ones to limit context size. See the `bootstrap-project` skill for the full workflow.
4. **Trims this document** to only the declared type's conventions, since the type is rarely changed after initialization.
5. **Records the declared type and the tailoring decisions** in the project's README and `AGENTS.md` so future agent sessions have the context.

## When to stop and ask

- If the project type is not stated or is ambiguous, ask before proceeding.
- If a project appears to change type after initialization, confirm with the user before re-applying conventions.
- If a contracted project's handover requirements conflict with a convention, ask which wins.
