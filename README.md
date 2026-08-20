# Flash

A framework / template for accelerating agentic software development with LLM agents (e.g. OpenCode).

This repository does not contain an application. It contains the conventions, standards, and workflows that agents must follow when building TypeScript applications on top of it. The goal is **long-horizon development**: codebases that stay clean, typed, tested, and maintainable over time.

## What this framework provides

- **Agent workflow** — a mandatory Clarify → Plan → Implement → Verify loop that prevents agents from guessing and keeps changes reviewable.
- **Strict TypeScript standards** — no `any`, explicit typing everywhere, strict compiler settings.
- **Tooling conventions** — ESLint, Prettier, Vitest, Playwright, and Zod wired into a verification pipeline every change must pass.
- **Architecture conventions** — npm workspaces monorepo, with support for both SSR frameworks and JSON-backend + SPA splits.
- **Enterprise-grade design patterns** — REST API design, Drizzle data layer, error handling, performance, and i18n requirements defined up front.

## Getting started

1. Read `AGENTS.md` — it is the mandatory entry point for any agent working in this repository.
2. Read the relevant `docs/` file before touching an area (see the table in `AGENTS.md`).
3. Follow the workflow in `docs/08-agent-workflow.md` for every task.

## Documentation index

| Doc | Purpose |
| --- | --- |
| `docs/01-architecture.md` | Monorepo layout, SSR vs SPA+backend patterns, package boundaries |
| `docs/02-typescript.md` | Strict typing rules, no `any`, tsconfig settings |
| `docs/03-tooling.md` | Linting, formatting, testing, validation, CI gates |
| `docs/04-api-design.md` | REST conventions, error envelope, validation at boundaries |
| `docs/05-data-layer.md` | Drizzle, migrations, typed queries, repository pattern |
| `docs/06-i18n.md` | Internationalization requirements |
| `docs/07-performance.md` | Performance best practices |
| `docs/08-agent-workflow.md` | The mandatory agent workflow |
| `docs/09-definition-of-done.md` | Checklist every change must satisfy |
| `docs/10-stack-guidance.md` | Per-framework guidance for supported stacks |
| `docs/11-code-organization.md` | Vertical DDD, feature slices, interfaces, DI |

## License

See `LICENSE` if present; otherwise this template is provided for internal use.
