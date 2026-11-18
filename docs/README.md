# Documentation — knowledge map

This file is the **entry point** for humans and agents (AI): it shows **where to find what** and in which order to load context. Paths are stable for sharing context across sessions and contributors.

---

## 1. Recommended reading order

| Priority | File / folder | Role |
|----------|---------------|------|
| 1 | [`../agent/agent-principal.md`](../agent/agent-principal.md) | Global project rules, stack, conventions, checklists |
| 2 | **This map** (`docs/README.md`) | Index of business and technical docs |
| 3 | [`business/`](business/) — vision, glossary, requirements history | Shared *product* context (Git), before technical detail |
| 4 | Docs for the **task scope** (ADR, guide, feature — see below) | Targeted context; avoids loading everything |

For a specific task: read the specialized agent from `agent/` if needed (React, API, UX, etc.), then **only** the documents listed for the area touched.

---

## 2. Business / product context (vision & requirements history)

Location: **`docs/business/`** — see [`business/README.md`](business/README.md).

| Document | Role |
|----------|------|
| [`business/VISION.md`](business/VISION.md) | Overall goal, users, scope, success criteria |
| [`business/GLOSSARY.md`](business/GLOSSARY.md) | Business terms and definitions |
| [`business/REQUIREMENTS-HISTORY.md`](business/REQUIREMENTS-HISTORY.md) | Dated log of requirement changes (most recent first) |

**Rule**: any notable evolution of the business *what* or *why* → entry in `REQUIREMENTS-HISTORY.md` + update `VISION.md` or the glossary if needed. Deliverable detail stays in `docs/features/` (and technical *why* in ADRs).

**Multi-machine / multi-agent**: this folder is versioned; after editing, **commit + push** to propagate context. In AI sessions, cite paths read (convention already in “Agent directives”).

---

## 3. Agent directives (behavior & quality)

Location: **`agent/`** at the repository root.

| File | Usage |
|------|--------|
| `agent-principal.md` | Main reference; links to the others |
| `agent-software-engineer.md` | Execution, tests, quality |
| `agent-react.md` | Frontend React |
| `agent-api-architect.md` | API layers, DTOs, resilience |
| `agent-sys-architecture.md` | System architecture, ADRs |
| `agent-ux-ui.md` | UX, journeys, accessibility |
| `agent-github.md` | Git / PR (if used) |

**Multi-agent convention**: at the start of a task, state explicitly which paths were already read (e.g. “context: `agent/agent-principal.md`, `docs/business/VISION.md`, `docs/architecture/ADR-002-…`”) so the next contributor is aligned.

---

## 4. Architecture decisions (ADR)

Location: **`docs/architecture/`**

| Document | Topic |
|----------|--------|
| `ADR-001-pagination-implementation.md` | Offset pagination, defaults, response shape |
| `ADR-002-filters-and-sorting-system.md` | Generic filters / sorting, security, operators |

**Rule**: any structural decision (stack, auth, major schema, API pattern) → **new numbered ADR** here, not only a code comment.

---

## 5. Technical guides (developer how-to)

Location: **`docs/guides/`** — naming: `Guide-<Topic>.md`

| Document | Topic |
|----------|--------|
| [`guides/Guide-Pagination.md`](guides/Guide-Pagination.md) | Middleware, Prisma, response format |
| [`guides/Guide-Filters.md`](guides/Guide-Filters.md) | Query params, model config, routes |

**Rule**: the guide describes **how**; the ADR describes **why**. If code changes, update the guide **in the same change** when it affects API consumers.

---

## 6. Product features

Location: **`docs/features/`** — keep existing naming (`BO-Feature-*.md`, etc.)

| Document | Topic |
|----------|--------|
| [`features/BO-Feature-SSO-Backoffice.md`](features/BO-Feature-SSO-Backoffice.md) | SSO / OIDC Keycloak — spec, env, endpoints |
| [`features/BO-Feature-Media.md`](features/BO-Feature-Media.md) | Media (backoffice) |
| [`features/SSO_IMPLEMENTATION_GUIDE.md`](features/SSO_IMPLEMENTATION_GUIDE.md) | SSO implementation in the repo (complements the SSO feature doc) |

**Rule**: significant new feature → **`docs/features/BO-Feature-<NAME>.md`** (goal, scope, data, endpoints, out of scope). Split a long implementation guide into a separate file if the feature doc grows too large.

---

## 7. Where to put future documents

| Type | Location |
|------|----------|
| ADR | `docs/architecture/ADR-00N-title-kebab-case.md` |
| Technical guide | `docs/guides/Guide-<Topic>.md` |
| Backoffice feature | `docs/features/BO-Feature-<NAME>.md` |
| Public frontend feature | `docs/features/FR-Feature-<NAME>.md` |
| JTBD / UX journeys | `docs/ux/<feature>-jtbd.md`, `-journey.md`, `-flow.md` (create `docs/ux/` if needed) |
| Vision / glossary / requirements history | `docs/business/` (`VISION.md`, `GLOSSARY.md`, `REQUIREMENTS-HISTORY.md`) |
| Stack / version report | `docs/report/` (maintained or regenerated intentionally) |

Avoid **duplication**: one **reference** document; others **link** with a Markdown link.

---

## 8. “Minimal context” checklist (human or agent)

Before implementing a change:

- [ ] `agent/agent-principal.md` refreshed in your reading if the task touches stack or conventions
- [ ] `docs/business/` (vision, glossary, history) if the task touches product scope or a business rule
- [ ] Relevant ADR(s) if you change a decision already recorded (or plan a new ADR)
- [ ] Guide (`docs/guides/`) or `docs/features/` updated if behavior or API changes
- [ ] At end of task: one sentence in the PR or commit message on what was documented (or “docs unchanged, no impact”)

---

## 9. Living file

This map should be **adjusted** when new document categories appear (e.g. published OpenAPI, ops runbooks). Prefer **editing this README** rather than multiplying “indexes” elsewhere.
