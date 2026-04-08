# Agent and contributor instructions

**Default:** use the **English** tree under **`en/`**. Use **`fr/`** when the task or follow-up is in French (see below).

Before any non-trivial task, load context in this order:

1. **[`en/agent/agent-principal.md`](en/agent/agent-principal.md)** — project rules, stack, conventions, checklists.
2. **[`en/docs/README.md`](en/docs/README.md)** — documentation map (`en/docs/architecture/`, `en/docs/guides/`, `en/docs/features/`, `en/docs/business/` for product vision and requirements history).

Then open only the files listed in `en/docs/README.md` for the relevant scope (avoid loading the entire repository).

Domain-specific guidance lives under **`en/agent/`** (for example `agent-react.md`, `agent-api-architect.md`).

When a change affects behavior, the API, or an architecture decision: update the relevant documentation or state explicitly in the commit / PR why docs were not updated.

---

## French (`fr/`) — prompts and follow-up in French

If the **task**, **issue/PR discussion**, or **documentation** you are working on is in **French**, load context from **`fr/`** instead:

1. [`fr/agent/agent-principal.md`](fr/agent/agent-principal.md)
2. [`fr/docs/README.md`](fr/docs/README.md)
3. Only the files listed in `fr/docs/README.md` for the current scope.

Domain agents: **`fr/agent/`** (same file names as under `en/agent/`).

Use **`fr/`** for French-language **suivi** (tracking, reviews, product docs) so decisions stay aligned with the French documentation set.
