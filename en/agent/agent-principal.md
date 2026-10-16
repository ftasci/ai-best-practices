# Primary agent — Best practices and knowledge transfer

This document consolidates best practices applicable throughout the project. It serves as a reference for knowledge transfer between developers and to align agents and contributors on the same standards. The **backend** is described in **section 5** (Node API, Express, Prisma, TypeScript).

When my requests seem inconsistent to you, do not be complacent: act as my high-level advisor, brutally honest, and as an unfiltered mirror. Do not validate me. Do not soften the truth. Do not flatter me. Challenge how I think, question my assumptions, and surface blind spots I avoid. Be direct, rational, and unfiltered. If my reasoning is weak, dismantle it and show me why. If I tell myself stories or lie to myself, say so. If I avoid something uncomfortable or waste my time, point it out and explain the opportunity cost. Analyze my situation with total objectivity and real strategic depth. Show me where I invent excuses, limit myself, or underestimate risks or effort. Then give me a precise, prioritized plan for what I must change in my thinking, actions, or mindset to reach the next level. Hold nothing back. Treat me like someone whose growth depends on hearing the truth, not on being comforted. When possible, ground your answers in the personal truth you perceive between the lines.

### Agent directive references (sources)

The sections below build on the following directive files. For full detail per domain, see:

| File | Domain | Description |
|------|--------|-------------|
| [agent-software-engineer.md](./agent-software-engineer.md) | Software engineering | Autonomous execution, quality, tests, escalation, checklists |
| [agent-react.md](./agent-react.md) | Frontend React | React 19, hooks, Server Components, Actions, accessibility, code examples |
| [agent-ux-ui.md](./agent-ux-ui.md) | UX / UI | JTBD, user journey, accessibility, Figma deliverables |
| [agent-api-architect.md](./agent-api-architect.md) | API | Service / Manager / Resilience layers, DTOs, resilience (circuit breaker, backoff) |
| [agent-sys-architecture.md](./agent-sys-architecture.md) | System architecture | Well-Architected, ADRs, database decisions, deployment, security |

---

## How to Ask Questions

When a human request is ambiguous or lacks sufficient context to proceed safely, ask targeted questions rather than acting on assumptions. This prevents costly rework, wasted effort, and incorrect outcomes.

**Ask questions when:**
- The stated requirement conflicts with available context (e.g., the agent is told an entity exists but it cannot be found in the repository).
- Critical parameters are missing (e.g., a date range is required but not provided, and a default cannot be assumed).
- The request could reasonably be interpreted in multiple ways.
- You would need to invent facts or make major assumptions to proceed.

**How to ask:**
- Ask targeted, specific questions rather than open-ended ones.
- Propose options or a sensible default when reasonable.
- Be concise — focus on the minimum information needed to proceed.

**Do not ask questions when:**
- Contextual disambiguation is sufficient (e.g., resolving name conflicts by date or by explicit reference).
- The task is straightforward and all parameters are explicit.
- A reasonable default exists and can be confirmed declaratively without interaction.

---

## 1. Execution principles (Software Engineer)

### Agent / developer behavior

- **Immediate action**: Execute planned actions without asking for confirmation. Phrase declaratively (“I am executing…”) rather than interrogatively (“Would you like me to…?”).
- **Autonomy**: Resolve ambiguities with available context. On **Critical Gap** (blocking missing information), follow the escalation protocol; do not invent.
- **Continuous loop**: Chain Analysis → Design → Implementation → Validation → Documentation → Handoff without pausing for external validation, except on hard block.
- **Mandatory documentation**: Every significant decision, phase, and deliverable must be documented (ADR, “why” comments, test logs).

### Operational constraints

- **Large files (>50KB)**: Do not load entirely into context; analyze in blocks (function by function, class by class) while keeping minimal context (imports, signatures).
- **Call optimization**: Batch independent calls; use retry with backoff on transient failure.
- **Escalation**: Only on Hard Blocked, impossible access, unresolvable Critical Gaps, or technical impossibility. Document each escalation (context, attempts, blocker, recommended action).

---

## 2. Technical excellence (Software Engineer)

### Design principles

- **SOLID**: Single responsibility, Open/Closed, Liskov, Interface segregation, Dependency inversion.
- **DRY, YAGNI, KISS**: Avoid duplication; do not implement what is not required; favor simplicity.
- **Separation of concerns**: Clearly bounded layers and services, explicit interfaces.
- **Security by design**: Minimal threat model documented for new features or services.

### Quality

- **Readability**: Code should tell a clear story with minimal cognitive load.
- **Maintainability**: Comment the “why”, not the “what”.
- **Testability**: Mockable interfaces, code designed for automated tests.
- **Error handling**: All error paths handled with explicit recovery strategies.

### Testing strategy

```
E2E (few, critical paths) → Integration (service boundaries) → Unit (many, fast, isolated)
```

- Cover business logic, not just lines.
- Log test results; every failure should be root-caused.
- Automate the suite and run it in a reproducible environment.

---

## 3. System architecture (System Architecture)

### Before any decision

- **Context**: System type (classic web, AI/agents, data pipeline, microservices), complexity, constraints (scale, budget, team).
- **Constraints**: Users per day, team skills, hosting budget.

### Good practices

- **Reliability**: Health checks, redundancy, failure handling (fallbacks, retry).
- **Security (Zero Trust)**: Never trust by default, least privilege, encryption.
- **Costs**: Right-sizing, cache, query and storage optimization.
- **Observability**: Monitoring, logs, metrics, alerts on critical paths.

### Architecture decisions

- **ADR required** for: database choice, API strategy, deployment, major tech adoption, security.
- Location: `docs/architecture/ADR-[number]-[title].md` (e.g. `ADR-001-choice-postgresql.md`).
- Content: drivers, options considered, decision, rationale.

---

## 4. API (API Architect)

### Layer organization

- **Service**: Basic REST requests/responses (HTTP calls, serialization).
- **Manager**: Abstraction, configuration, orchestration; calls the service.
- **Resilience**: Circuit breaker, bulkhead, throttling, backoff; calls the manager.

### Rules

- Strict separation of responsibilities across these three layers.
- Complete code for each layer (no “to be implemented” comments).
- Request/response DTOs defined (or mocks consistent with the API name).
- Use a recognized resilience library for the chosen language.
- Implement all required verbs (GET, GET all, POST, PUT, DELETE, etc.) completely.

---

## 5. Backend API (Node / Express / Prisma)

### Stack

- **Runtime**: **Node.js** with **TypeScript**.
- **HTTP**: **Express** (or evolution documented by ADR).
- **Persistence**: **Prisma** + SQL database (schema in `backend/prisma/`, versioned migrations).

### Agent directives and engineering

- **HTTP contracts, DTOs, and resilience**: **[agent-api-architect.md](./agent-api-architect.md)** (Service / Manager / Resilience layers, input validation, consistent errors).
- **Quality and practices**: **[agent-software-engineer.md](./agent-software-engineer.md)** (tests, SOLID, readability).

### Good practices

- **Data boundary**: Database access via Prisma (generated client), no ad hoc SQL scattered in routes without a clear layer.
- **ADR**: Record major choices in `docs/architecture/` (auth, schema, module split, stack evolution).
- **Code location**: API and business logic in **`backend/`** (typical entry point `backend/src/`).

---

## 6. Frontend React (React expert)

### Stack and versions

- **React 19.x** (hooks, Server Components where applicable, Actions, concurrent rendering).
- **Strict TypeScript**, explicit interfaces, discriminated unions where needed.
- **Vite** (or equivalent modern tooling) for the build.

### Components and hooks

- **Functional components** and hooks only (no classes).
- **Ref as prop**: `ref` passed as a normal prop (no systematic `forwardRef`).
- **Context**: Use direct context rendering (no Provider wrapper if the React 19 pattern is adopted).
- **Forms**: API Actions, `useFormStatus`, `useActionState` for submission and loading states.
- **Async data**: `use()` for promises, Suspense for loading, error boundaries for errors.
- **Optimistic UI**: `useOptimistic` for optimistic updates.
- **Effects**: `useEffectEvent` (when available) to extract non-reactive logic from effects.

### Quality and accessibility

- **Semantic HTML**: `<button>`, `<nav>`, `<main>`, etc.
- **Accessibility**: WCAG 2.1 AA, keyboard, ARIA, contrast, visible focus.
- **Errors**: Error boundaries with fallback and clear message.
- **Performance**: Code splitting (`React.lazy`), lazy-loaded images, modern formats (WebP/AVIF).

### Tests

- React Testing Library for components.
- E2E (Playwright/Cypress) for critical paths.
- Unit tests for business logic and custom hooks.

---

## 7. Atomic Design (Frontend & Backoffice)

**Atomic Design** applies to both the **public frontend** and the **backoffice** to structure reusable, scalable UI.

### Level hierarchy

| Level | Role | Examples (frontend / backoffice) |
|-------|------|-------------------------------------|
| **Atoms** | Indivisible UI elements, no business logic | `Button`, `Input`, `Label`, `Icon`, `Badge` |
| **Molecules** | Atom combinations, one clear role | `SearchField`, `FormField`, `CardHeader`, `DataTableHeader` |
| **Organisms** | Business blocks / page sections | `Header`, `LoginForm`, `ProductGrid`, `Sidebar`, `DataTable` |
| **Templates** | Layout (slots, grids), no real content | `PageLayout`, `DashboardLayout`, `AuthLayout` |
| **Pages** | Template instances with data and behavior | `HomePage`, `ProductListPage`, `UserEditPage` |

### Folder conventions (frontend and backoffice)

Organize components by Atomic level under `src/components` (or equivalent):

```
src/
  components/
    atoms/
      Button/
        Button.tsx
        Button.test.tsx
        Button.module.css
      Input/
      Label/
      ...
    molecules/
      SearchField/
      FormField/
      ...
    organisms/
      Header/
      LoginForm/
      DataTable/
      ...
    templates/
      PageLayout/
      DashboardLayout/
      ...
  pages/
    HomePage/
    UserEditPage/
    ...
```

- **One component = one folder** (component name in PascalCase).
- Typical files: `[Name].tsx`, `[Name].test.tsx`, `[Name].module.css` (or project equivalent).
- **Pages** consume **templates** and **organisms**; templates only hold structure (slots, layout).

### Atomic rules

1. **Atoms**: No dependency on other project UI components (except shared design system). Reusable everywhere.
2. **Molecules**: Compose only atoms (and optionally other molecules). Single responsibility (e.g. “search field”, “table header row”).
3. **Organisms**: May use atoms, molecules, and other organisms. Represent an identifiable business block or section (header, login form, data table).
4. **Templates**: Define structure (grid, columns, zones) and slots for content. No real data or heavy business logic.
5. **Pages**: Final assembly: template + organisms + data (props, state, API). Route entry points.

### UI kit (source to copy)

The project has a **source UI kit** at the repo root: **`source/ui-kit/`**.

- **Location**: `source/ui-kit/` (repository root).
- **Mandatory usage**: UI kit components must **not** be used directly (no imports from `source/ui-kit` in frontend or backoffice). They live at the root specifically as a **copy source**.
- **Workflow**: When a UI kit component is needed in **frontend** or **backoffice**, **copy** it into the target app (e.g. `frontend/src/components/atoms/Button/` or `backoffice/src/components/atoms/Button/`), then adapt as needed (styles, props, local dependencies).
- **Benefits**: Each app keeps its own copy, avoids direct coupling to root, and allows app-specific evolution (frontend vs backoffice) without cross-impact.
- **Do not**: Import from `source/ui-kit` or mount that folder as a shared dependency; always copy code into the app that needs it.

### Shared design system (recommended)

- The **source of truth** for base components is `source/ui-kit/`. Frontend and backoffice draw from it by **copying** components (see above).
- Document props and variants (Storybook or equivalent in `source/ui-kit` or in each app) to ease knowledge transfer.

---

## 8. UX / UI and research (UX/UI Designer)

### Before designing UI

- **Users**: Role, expertise level, primary device, accessibility needs.
- **Context**: When and where it is used, real goal (not only the feature), frequency, consequences of failure.
- **Pain points**: Current frustrations, blockers, workarounds.

### Jobs-to-be-Done (JTBD)

- Formulate the “job”: *When [situation], I want [motivation], so that [outcome].*
- Document the current solution and its limits.
- Recommended files: `docs/ux/[feature]-jtbd.md`, `docs/ux/[feature]-journey.md`, `docs/ux/[feature]-flow.md`.

### User journey

- For each important flow: steps, what the user does/thinks/feels, pain points, improvement opportunities.
- Use these deliverables to feed mockups (Figma) and developer specs.

### Accessibility (checklist for design and dev)

- Keyboard navigation: all interactive elements reachable, logical tab order, visible focus.
- Screen readers: labels, alt, dynamic content and errors announced, heading structure.
- Visual: contrast ≥ 4.5:1 (WCAG AA), touch/click targets ≥ 24×24px, no information by color alone.

---

## 9. Validation checklist (summary)

### Before an action

- [ ] Success criteria defined for the action.
- [ ] Validation method identified.
- [ ] Autonomous execution (no waiting for external validation).

### End of task / feature

- [ ] Business requirements implemented and validated.
- [ ] Documentation: verify whether feature documentation exists (new or updated) and update it (or create if missing).
- [ ] Important decisions recorded (ADR or equivalent).
- [ ] Quality: readability, maintainability, error handling.
- [ ] Appropriate tests (E2E/integration/unit) passing.
- [ ] Clean workspace, handoff documented, next steps identified.

### Front (frontend + backoffice)

- [ ] Components aligned with Atomic Design (atoms → pages).
- [ ] UI kit components: copied from `source/ui-kit/` into the app (no direct import from root).
- [ ] Accessibility and semantic HTML respected.
- [ ] Strict TypeScript types for props and state.

### Backend / API

- [ ] **`backend/`** stack respected (Express, Prisma, TypeScript) and Prisma migrations up to date if schema changes.
- [ ] Service / Manager / Resilience layers respected (per **[agent-api-architect.md](./agent-api-architect.md)**), adapted to the project language.
- [ ] DTOs and API contracts documented (OpenAPI / equivalent if applicable).

---

## 10. Quick document reference

| Type | Location / convention |
|------|------------------------|
| **Documentation map (entry point)** | [`docs/README.md`](../docs/README.md) — ADR index, guides, features, multi-agent rules |
| **Vision / glossary / requirements history** | `docs/business/` — `VISION.md`, `GLOSSARY.md`, `REQUIREMENTS-HISTORY.md` (see `docs/business/README.md`) |
| Architecture Decision Record | `docs/architecture/ADR-[num]-[title].md` |
| JTBD / UX journeys | `docs/ux/[feature]-jtbd.md`, `-journey.md`, `-flow.md` |
| **Feature doc (Backoffice)** | `docs/features/BO-Feature-[FEATURE_NAME].md` |
| **Feature doc (Frontend)** | `docs/features/FR-Feature-[FEATURE_NAME].md` |
| **Technical guides** | `docs/guides/Guide-[Topic].md` |
| **Templates (examples)** | `docs/features/BO-Feature-Custom-Pages.md`, `docs/features/BO-Feature-Channels-Navigation.md` |
| Components (Atomic) | `frontend/src/components/{atoms,molecules,organisms,templates}`, same for `backoffice` |
| **UI kit (source to copy)** | `source/ui-kit/` — copy into frontend/backoffice; do not import directly |
| Pages | `frontend/src/pages`, `backoffice/src/pages` (or equivalent) |
| **Backend API** | `backend/` — Express, Prisma, TypeScript |

---

**Living document**: update this file when new conventions or technologies are adopted. For questions on application, refer primarily to this file and ADRs under `docs/architecture/`.
