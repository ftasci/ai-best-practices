# Business and product context

This folder is the **source of truth** for product *why* (outside implementation detail). It is **versioned in Git**: any machine or agent session that clones the repo gets the same context.

## Recommended reading order (human or AI)

| Order | File | Role |
|-------|------|------|
| 1 | [`VISION.md`](VISION.md) | Overall goal, users, scope, success criteria |
| 2 | [`GLOSSARY.md`](GLOSSARY.md) | Shared business terms and definitions |
| 3 | [`REQUIREMENTS-HISTORY.md`](REQUIREMENTS-HISTORY.md) | Log of requirement changes (most recent first) |
| 4 | Then | `docs/features/`, ADRs, guides per task (`docs/README.md`) |

## Maintenance rules

- **Vision or glossary**: update when product understanding changes (even if code is not there yet).
- **History**: on every **notable** requirement change (priority, scope, business rule), add a dated entry in `REQUIREMENTS-HISTORY.md`; link the PR or ticket when possible.
- **Avoid duplication**: `docs/features/` describes a feature; vision stays short and global; history records *what changed* and *why*.

## Multi-agent / multi-machine work

At the start of a session, state explicitly which paths were read (e.g. “context: `docs/business/VISION.md`, `docs/business/REQUIREMENTS-HISTORY.md`”). After a business change, **commit + push** so other workstations and agents stay aligned.
