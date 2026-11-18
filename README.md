# Best Practices — AI Coding (solo & multi-agent)

This repository is a **best-practices reference** for building with AI agents, whether you work alone or with multiple contributors, with clear and versioned documentation.

The goal is straightforward: keep a shared framework for code quality, architecture decisions, product documentation, and handoff between humans and AI agents.

## How this project is set up

- **`agent/`**: operating rules for agents (principal + domain-specific).
- **`docs/`**: knowledge base (vision, ADRs, guides, features).
- `.gitignore`: files excluded from version control.

See **[`AGENTS.md`](AGENTS.md)** for how agents should load context. Current layout:

```text
.
├── AGENTS.md
├── README.md
├── agent/
│   ├── agent-principal.md
│   ├── agent-software-engineer.md
│   ├── agent-api-architect.md
│   ├── agent-react.md
│   ├── agent-sys-architecture.md
│   ├── agent-ux-ui.md
│   └── agent-github.md
└── docs/
    ├── README.md
    ├── architecture/
    ├── business/
    ├── features/
    └── guides/
```

## Recommended reading order

Before any non-trivial task, load context in this order:

1. `agent/agent-principal.md`
2. `docs/README.md`
3. Targeted documents for the current scope (`docs/architecture`, `docs/guides`, `docs/features`, `docs/business`)

This sequence avoids loading the whole repository unnecessarily and improves consistency across sessions.

## Solo / multi-contributor workflow

- **Solo**: use this repo as your single source of truth for technical and product decisions.
- **Multi (humans + AI)**: track every meaningful decision and change in the right documents to enable clean handoff.
- **Async**: versioned docs let another agent/contributor pick up work without oral context.

## Documentation rules

When behavior, API, or architecture changes:

- update the relevant documentation in the same change, or
- explicitly state in the commit/PR why docs are unchanged.

Main references:

- Major technical decisions: `docs/architecture/ADR-00N-*.md`
- Implementation guides: `docs/guides/Guide-*.md`
- Functional specifications: `docs/features/*`
- Vision / glossary / requirements history: `docs/business/*`

## Suggested workflow

1. Load minimal context (reading order above).
2. Implement with domain conventions (`agent/*.md`).
3. Validate (tests/lint for the affected scope).
4. Update impacted documentation.
5. Commit with a clear message focused on the *why*.

## Contribution conventions

- Prefer atomic and traceable changes.
- Avoid documentation duplication: one source document, others link to it.
- Name new documentation files with existing conventions.

---

If you are just getting started, begin with `agent/agent-principal.md`, then `docs/README.md`.
