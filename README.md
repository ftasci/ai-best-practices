# Best Practices — AI Coding (solo & multi-agent)

This repository is a **best-practices reference** for building with AI agents, whether you work alone or with multiple contributors, with clear and versioned documentation.

The goal is straightforward: keep a shared framework for code quality, architecture decisions, product documentation, and handoff between humans and AI agents.

## How this project is set up

Documentation and agent rules are split by language:

- **`en/`** (default): `en/agent/` — operating rules; `en/docs/` — knowledge base (vision, ADRs, guides, features).
- **`fr/`**: `fr/agent/` and `fr/docs/` — same scope in French (follow-up, French prompts).
- `.gitignore`: files excluded from version control.

See **[`AGENTS.md`](AGENTS.md)** for which tree to load. Current layout (English):

```text
.
├── en/
│   ├── agent/
│   │   ├── agent-principal.md
│   │   ├── agent-software-engineer.md
│   │   ├── agent-api-architect.md
│   │   ├── agent-react.md
│   │   ├── agent-sys-architecture.md
│   │   ├── agent-ux-ui.md
│   │   └── agent-github.md
│   └── docs/
│       ├── README.md
│       ├── architecture/
│       ├── business/
│       ├── features/
│       └── guides/
└── fr/
    ├── agent/
    └── docs/
```

## Recommended reading order

Before any non-trivial task, load context in this order:

1. `en/agent/agent-principal.md`
2. `en/docs/README.md`
3. Targeted documents for the current scope (`en/docs/architecture`, `en/docs/guides`, `en/docs/features`, `en/docs/business`)

This sequence avoids loading the whole repository unnecessarily and improves consistency across sessions.

## Solo / multi-contributor workflow

- **Solo**: use this repo as your single source of truth for technical and product decisions.
- **Multi (humans + AI)**: track every meaningful decision and change in the right documents to enable clean handoff.
- **Async**: versioned docs let another agent/contributor pick up work without oral context.

## Documentation rules

When behavior, API, or architecture changes:

- update the relevant documentation in the same change, or
- explicitly state in the commit/PR why docs are unchanged.

Main references (English):

- Major technical decisions: `en/docs/architecture/ADR-00N-*.md`
- Implementation guides: `en/docs/guides/Guide-*.md`
- Functional specifications: `en/docs/features/*`
- Vision / glossary / requirements history: `en/docs/business/*`

## Suggested workflow

1. Load minimal context (reading order above).
2. Implement with domain conventions (`en/agent/*.md`).
3. Validate (tests/lint for the affected scope).
4. Update impacted documentation.
5. Commit with a clear message focused on the *why*.

## Contribution conventions

- Prefer atomic and traceable changes.
- Avoid documentation duplication: one source document, others link to it.
- Name new documentation files with existing conventions.

---

If you are just getting started, begin with `en/agent/agent-principal.md`, then `en/docs/README.md`. For French, use `fr/agent/agent-principal.md` and `fr/docs/README.md`.

## Version francaise

# Best Practices — Coding avec IA (solo & multi-agents)

Ce depot sert de **referentiel de bonnes pratiques** pour developper avec des agents IA, en solo ou a plusieurs, avec une base documentaire claire et versionnee.

L'objectif est simple : garder un cadre commun pour la qualite du code, les decisions d'architecture, la documentation produit, et le handoff entre contributeurs humains et IA.

## Comment le projet est organisé

La doc et les règles agents sont séparées par langue. Voir **[`AGENTS.md`](AGENTS.md)** (par défaut : `en/` ; suivi ou prompts en français : `fr/`).

- **`fr/agent/`** et **`fr/docs/`** : même périmètre que l’anglais, en français.
- **`.gitignore`** : fichiers exclus du versioning.

Extrait de structure côté français :

```text
fr/
├── agent/
│   ├── agent-principal.md
│   └── …
└── docs/
    ├── README.md
    ├── architecture/
    ├── business/
    ├── features/
    └── guides/
```

## Ordre de lecture recommande

Pour rester aligne avant toute tache non triviale :

1. `fr/agent/agent-principal.md`
2. `fr/docs/README.md`
3. Les documents cibles du perimetre concerne (`fr/docs/architecture`, `fr/docs/guides`, `fr/docs/features`, `fr/docs/business`)

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

- Decisions techniques structurantes : `fr/docs/architecture/ADR-00N-*.md`
- Guides d'implementation : `fr/docs/guides/Guide-*.md`
- Specifications fonctionnelles : `fr/docs/features/*`
- Vision / vocabulaire / historique besoins : `fr/docs/business/*`

## Workflow conseille

1. Charger le contexte minimal (ordre de lecture ci-dessus).
2. Implementer la tache avec les conventions du domaine (`fr/agent/*.md`).
3. Valider (tests/lint selon le perimetre).
4. Mettre a jour la documentation impactee.
5. Commit avec un message clair sur le *pourquoi*.

## Convention de contribution

- Privilegier des changements atomiques et tracables.
- Eviter la duplication documentaire : un document source, les autres referencent.
- Nommer les nouveaux fichiers de doc selon les conventions deja en place.

---

Si tu demarres en français, commence par `fr/agent/agent-principal.md` puis `fr/docs/README.md`. Par défaut en anglais : `en/agent/agent-principal.md` et `en/docs/README.md`.
