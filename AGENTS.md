# Agent and contributor instructions

Before any non-trivial task, load context in this order:

1. **[`agent/agent-principal.md`](agent/agent-principal.md)** — project rules, stack, conventions, checklists.
2. **[`docs/README.md`](docs/README.md)** — documentation map (`docs/architecture/`, `docs/guides/`, `docs/features/`, `docs/business/` for product vision and requirements history).

Then open only the files listed in `docs/README.md` for the relevant scope (avoid loading the entire repository).

Domain-specific guidance lives under **`agent/`** (for example `agent-react.md`, `agent-api-architect.md`).

When a change affects behavior, the API, or an architecture decision: update the relevant documentation or state explicitly in the commit / PR why docs were not updated.
