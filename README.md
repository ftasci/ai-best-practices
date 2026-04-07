# Best Practices — AI Coding (solo & multi-agent)

This repository is a **best-practices reference** for building with AI agents, whether you work alone or with multiple contributors, with clear and versioned documentation.

The goal is straightforward: keep a shared framework for code quality, architecture decisions, product documentation, and handoff between humans and AI agents.

## How this project is set up

- `agent/`: operating rules for agents and contributors (quality, API, React, architecture, UX, GitHub).
- `docs/`: project knowledge base (product vision, ADRs, guides, feature specs).
- `.gitignore`: files excluded from version control.

Current structure:

```text
.
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

## Version francaise

# Best Practices — Coding avec IA (solo & multi-agents)

Ce depot sert de **referentiel de bonnes pratiques** pour developper avec des agents IA, en solo ou a plusieurs, avec une base documentaire claire et versionnee.

L'objectif est simple : garder un cadre commun pour la qualite du code, les decisions d'architecture, la documentation produit, et le handoff entre contributeurs humains et IA.

## Comment le projet est "set"

- `agent/` : regles de travail pour les agents et contributeurs (qualite, API, React, architecture, UX, GitHub).
- `docs/` : base de connaissance projet (vision produit, ADR, guides, specs de features).
- `.gitignore` : fichiers a exclure du versioning.

Structure actuelle :

```text
.
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

## Ordre de lecture recommande

Pour rester aligne avant toute tache non triviale :

1. `agent/agent-principal.md`
2. `docs/README.md`
3. Les documents cibles du perimetre concerne (`docs/architecture`, `docs/guides`, `docs/features`, `docs/business`)

Cette sequence evite de charger tout le repo inutilement et ameliore la coherence entre sessions.

## Fonctionnement solo / multi-contributeurs

- **Solo** : tu utilises ce repo comme source de verite pour tes choix techniques et produit.
- **Multi (humains + IA)** : chaque decision et changement important est trace dans les bons documents pour permettre un handoff propre.
- **Asynchrone** : la doc versionnee permet a un autre agent/contributeur de reprendre sans contexte oral.

## Regles de documentation

Quand le comportement, l'API ou l'architecture evolue :

- mettre a jour la doc concernee dans le meme changement, ou
- indiquer explicitement dans le commit / la PR pourquoi la doc ne change pas.

Reperes principaux :

- Decisions techniques structurantes : `docs/architecture/ADR-00N-*.md`
- Guides d'implementation : `docs/guides/Guide-*.md`
- Specifications fonctionnelles : `docs/features/*`
- Vision / vocabulaire / historique besoins : `docs/business/*`

## Workflow conseille

1. Charger le contexte minimal (ordre de lecture ci-dessus).
2. Implementer la tache avec les conventions du domaine (`agent/*.md`).
3. Valider (tests/lint selon le perimetre).
4. Mettre a jour la documentation impactee.
5. Commit avec un message clair sur le *pourquoi*.

## Convention de contribution

- Privilegier des changements atomiques et tracables.
- Eviter la duplication documentaire : un document source, les autres referencent.
- Nommer les nouveaux fichiers de doc selon les conventions deja en place.

---

Si tu demarres, commence par `agent/agent-principal.md` puis `docs/README.md`.
