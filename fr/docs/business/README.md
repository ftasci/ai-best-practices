# Contexte métier et produit

Ce dossier est la **source de vérité** pour le *pourquoi* du produit (hors détails d’implémentation). Il est **versionné dans Git** : toute machine ou session d’agent qui clone le dépôt récupère le même contexte.

## Ordre de lecture recommandé (humain ou IA)

| Ordre | Fichier | Rôle |
|-------|---------|------|
| 1 | [`VISION.md`](VISION.md) | Objectif global, utilisateurs, périmètre, critères de succès |
| 2 | [`GLOSSAIRE.md`](GLOSSAIRE.md) | Termes métier et définitions partagées |
| 3 | [`HISTORIQUE-BESOINS.md`](HISTORIQUE-BESOINS.md) | Journal des évolutions de besoins (le plus récent en haut) |
| 4 | Puis | `docs/features/`, ADR, guides selon la tâche (`docs/README.md`) |

## Règles de maintenance

- **Vision ou glossaire** : mise à jour quand la compréhension du produit change (même si le code n’est pas encore là).
- **Historique** : à chaque **changement notable** de besoin (priorité, périmètre, règle métier), ajouter une entrée datée dans `HISTORIQUE-BESOINS.md` ; lier la PR ou le ticket si possible.
- **Éviter la duplication** : les fiches `docs/features/` décrivent une feature ; la vision reste courte et globale ; l’historique enregistre *ce qui a changé* et *pourquoi*.

## Travail multi‑agent / multi‑machine

En début de session, indiquer explicitement les chemins consultés (ex. « contexte : `docs/business/VISION.md`, `docs/business/HISTORIQUE-BESOINS.md` »). Après une évolution métier, **commit + push** pour que les autres postes et agents restent alignés.
