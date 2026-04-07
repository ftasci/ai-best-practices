# Best Practices — Coding avec IA (solo & multi-agents)

Ce dépôt sert de **référentiel de bonnes pratiques** pour développer avec des agents IA, en solo ou à plusieurs, avec une base documentaire claire et versionnée.

L'objectif est simple : garder un cadre commun pour la qualité du code, les décisions d'architecture, la documentation produit, et le handoff entre contributeurs humains et IA.

## Comment le projet est "set"

- `agent/` : règles de travail pour les agents et contributeurs (qualité, API, React, architecture, UX, GitHub).
- `docs/` : base de connaissance projet (vision produit, ADR, guides, specs de features).
- `.gitignore` : fichiers à exclure du versioning.

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

## Ordre de lecture recommandé

Pour rester aligné avant toute tâche non triviale :

1. `agent/agent-principal.md`
2. `docs/README.md`
3. Les documents ciblés du périmètre concerné (`docs/architecture`, `docs/guides`, `docs/features`, `docs/business`)

Cette séquence évite de charger tout le repo inutilement et améliore la cohérence entre sessions.

## Fonctionnement solo / multi-contributeurs

- **Solo** : tu utilises ce repo comme source de vérité pour tes choix techniques et produit.
- **Multi (humains + IA)** : chaque décision et changement important est tracé dans les bons documents pour permettre un handoff propre.
- **Asynchrone** : la doc versionnée permet à un autre agent/contributeur de reprendre sans contexte oral.

## Règles de documentation

Quand le comportement, l'API ou l'architecture évolue :

- mettre à jour la doc concernée dans le même changement, ou
- indiquer explicitement dans le commit / la PR pourquoi la doc ne change pas.

Repères principaux :

- Décisions techniques structurantes : `docs/architecture/ADR-00N-*.md`
- Guides d'implémentation : `docs/guides/Guide-*.md`
- Spécifications fonctionnelles : `docs/features/*`
- Vision / vocabulaire / historique besoins : `docs/business/*`

## Workflow conseillé

1. Charger le contexte minimal (ordre de lecture ci-dessus).
2. Implémenter la tâche avec les conventions du domaine (`agent/*.md`).
3. Valider (tests/lint selon le périmètre).
4. Mettre à jour la documentation impactée.
5. Commit avec un message clair sur le *pourquoi*.

## Convention de contribution

- Privilégier des changements atomiques et traçables.
- Éviter la duplication documentaire : un document source, les autres référencent.
- Nommer les nouveaux fichiers de doc selon les conventions déjà en place.

---

Si tu démarres, commence par `agent/agent-principal.md` puis `docs/README.md`.
