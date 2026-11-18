# Business requirements history

**Append-only** log of requirement changes (priorities, scope, business rules). **Most recent entries first.**

Recommended format per entry:

```markdown
## YYYY-MM-DD — Short title

- **Change**: …
- **Motivation / context**: …
- **Impact**: (features, roles, data, API…)
- **References**: PR #…, ticket …, link …
```

---

## 2026-04-04 — Business documentation foundation

- **Change**: Created `docs/business/` (vision, glossary, history) to anchor product context in the repository.
- **Motivation / context**: The project lacked shared, traceable business context across machines and AI sessions.
- **Impact**: None on code; future requirements should be reflected here and in `docs/features/` when relevant.
- **References**: —
