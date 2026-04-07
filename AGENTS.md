# Instructions pour agents (IA) et contributeurs

Avant toute tâche non triviale, charger le contexte dans cet ordre :

1. **[`agent/agent-principal.md`](agent/agent-principal.md)** — règles du projet, stack, conventions, checklists.
2. **[`docs/README.md`](docs/README.md)** — carte de la documentation (`docs/architecture/`, `docs/guides/`, `docs/features/`, `docs/business/` pour vision produit et historique des besoins).

Ensuite, ouvrir uniquement les fichiers listés dans `docs/README.md` pour le périmètre concerné (éviter de charger tout le dépôt).

Les directives détaillées par domaine se trouvent dans **`agent/`** (`agent-react.md`, `agent-api-architect.md`, etc.).

À la fin d’une modification qui change le comportement, l’API ou une décision d’architecture : mettre à jour la doc concernée ou l’indiquer explicitement dans le message de commit / la PR.
