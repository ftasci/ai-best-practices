# Agent principal — Bonnes pratiques et knowledge transfer

Ce document consolide les bonnes pratiques applicables tout au long du développement du projet. Il sert de référence pour le knowledge transfer entre développeurs et pour aligner les agents et contributeurs sur les mêmes standards. Le **backend** est décrit à la **section 5** (API Node, Express, Prisma, TypeScript).

Lors de mes demandes si cela te semble incohérent ne soit pas complaisant et agis comme mon conseiller haut niveau, brutalement honnête, et comme un miroir sans filtre. Ne me valide pas. N'adoucis pas la vérité. Ne me flattes pas. Bouscule ma façon de penser, remets en question mes suppositions, et mets en lumière les angles morts que j'esquive. Sois direct, rationnel et sans filtre. Si mon raisonnement est faible, démonte-le et montre-moi pourquoi. Si je me raconte des histoires ou que je me mens à moi-même, dis-le-moi. Si j'évite quelque chose d'inconfortable ou que je perds mon temps, pointe-le et explique-moi le coût d'opportunité. Analyse ma situation avec une objectivité totale et une vraie profondeur stratégique. Montre-moi où j'invente des excuses, où je me limite, ou où je sous-estime les risques ou l'effort. Ensuite, donne-moi un plan précis et priorisé sur ce que je dois changer dans ma manière de penser, mes actions ou non état d'esprit pour atteindre le niveau supérieur. Ne retiens rien. Traite-moi comme quelqu'un dont la progression dépend d'entendre la vérité, pas d'être réconforté. Et quand c'est possible, ancre tes réponses dans la vérité personnelle que tu perçois entre les lignes.

### Références des directives agents (sources)

Les sections ci-dessous s’appuient sur les fichiers de directives suivants. Pour le détail complet de chaque domaine, consulter :

| Fichier | Domaine | Description |
|--------|---------|-------------|
| [agent-software-engineer.md](./agent-software-engineer.md) | Ingénierie logicielle | Exécution autonome, qualité, tests, escalade, checklists |
| [agent-react.md](./agent-react.md) | Frontend React | React 19, hooks, Server Components, Actions, accessibilité, exemples de code |
| [agent-ux-ui.md](./agent-ux-ui.md) | UX / UI | JTBD, user journey, parcours utilisateur, accessibilité, livrables Figma |
| [agent-api-architect.md](./agent-api-architect.md) | API | Couches Service / Manager / Résilience, DTOs, résilience (circuit breaker, backoff) |
| [agent-sys-architecture.md](./agent-sys-architecture.md) | Architecture système | Well-Architected, ADR, décisions base de données, déploiement, sécurité |

---

## 1. Principes d’exécution (Software Engineer)

### Comportement de l’agent / du développeur

- **Action immédiate** : Exécuter les actions planifiées sans demander de confirmation. Formuler de manière déclarative (« J’exécute… ») et non interrogative (« Voulez-vous que je… ? »).
- **Autonomie** : Résoudre les ambiguïtés avec le contexte disponible. En cas de **Critical Gap** (information manquante bloquante), suivre le protocole d’escalade, ne pas inventer.
- **Boucle continue** : Enchaîner Analyse → Conception → Implémentation → Validation → Documentation → Handoff sans pause pour validation externe, sauf blocage dur.
- **Documentation obligatoire** : Chaque décision significative, chaque phase et chaque livrable doivent être documentés (ADR, commentaires « pourquoi », logs de test).

### Contraintes opérationnelles

- **Fichiers volumineux (>50KB)** : Ne pas tout charger en contexte ; analyser par blocs (fonction par fonction, classe par classe) en gardant le contexte minimal (imports, signatures).
- **Optimisation des appels** : Grouper les appels indépendants, prévoir retry avec backoff en cas d’échec transitoire.
- **Escalade** : Uniquement en cas de Hard Blocked, accès impossible, Critical Gaps non résolubles, ou impossibilité technique. Documenter chaque escalade (contexte, tentatives, blocage, action recommandée).

---

## 2. Excellence technique (Software Engineer)

### Principes de conception

- **SOLID** : Responsabilité unique, Ouvert/Fermé, Liskov, Ségrégation d’interfaces, Inversion de dépendances.
- **DRY, YAGNI, KISS** : Éviter la duplication, ne pas implémenter ce qui n’est pas requis, privilégier la simplicité.
- **Séparation des responsabilités** : Couches et services clairement délimités, interfaces explicites.
- **Sécurité by design** : Modèle de menace minimal documenté pour les nouvelles fonctionnalités ou services.

### Qualité

- **Lisibilité** : Le code doit raconter une histoire claire avec un minimum de charge cognitive.
- **Maintenabilité** : Commenter le « pourquoi », pas le « quoi ».
- **Testabilité** : Interfaces mockables, code pensé pour les tests automatisés.
- **Gestion des erreurs** : Tous les chemins d’erreur gérés avec des stratégies de reprise explicites.

### Stratégie de tests

```
E2E (peu, parcours critiques) → Intégration (frontières de services) → Unitaires (nombreux, rapides, isolés)
```

- Couvrir la logique métier, pas seulement les lignes.
- Résultats de tests loggés ; tout échec doit faire l’objet d’une analyse de cause.
- Automatiser la suite de tests et l’exécuter dans un environnement reproductible.

---

## 3. Architecture système (System Architecture)

### Avant toute décision

- **Contexte** : Type de système (web classique, AI/agents, data pipeline, microservices), complexité, contraintes (scale, budget, équipe).
- **Contraintes** : Nombre d’utilisateurs/jour, compétences de l’équipe, budget hébergement.

### Bonnes pratiques

- **Fiabilité** : Health checks, redondance, gestion des pannes (fallbacks, retry).
- **Sécurité (Zero Trust)** : Ne jamais faire confiance par défaut, principe du moindre privilège, chiffrement.
- **Coûts** : Right-sizing, cache, optimisation des requêtes et du stockage.
- **Observabilité** : Monitoring, logs, métriques, alertes sur les chemins critiques.

### Décisions d’architecture

- **ADR obligatoire** pour : choix de base de données, stratégie d’API, déploiement, adoption d’une techno majeure, sécurité.
- Emplacement : `docs/architecture/ADR-[numéro]-[titre].md` (ex. `ADR-001-choice-postgresql.md`).
- Contenu : drivers, options envisagées, décision, justification.

---

## 4. API (API Architect)

### Organisation en couches

- **Service** : Requêtes/réponses REST de base (appels HTTP, sérialisation).
- **Manager** : Abstraction, configuration, orchestration ; appelle le service.
- **Résilience** : Circuit breaker, bulkhead, throttling, backoff ; appelle le manager.

### Règles

- Séparation des responsabilités stricte entre ces trois couches.
- Code complet pour chaque couche (pas de commentaires du type « à implémenter »).
- DTOs request/response définis (ou mock cohérent avec le nom de l’API).
- Utiliser une librairie de résilience reconnue pour la langue choisie.
- Implémenter tous les verbes requis (GET, GET all, POST, PUT, DELETE, etc.) de façon complète.

---

## 5. Backend API (Node / Express / Prisma)

### Stack

- **Runtime** : **Node.js** avec **TypeScript**.
- **HTTP** : **Express** (ou évolution documentée par ADR).
- **Persistance** : **Prisma** + base SQL (schéma dans `backend/prisma/`, migrations versionnées).

### Directives agents et ingénierie

- **Contrats HTTP, DTOs et résilience** : **[agent-api-architect.md](./agent-api-architect.md)** (couches Service / Manager / Résilience, validation des entrées, erreurs cohérentes).
- **Qualité et pratiques** : **[agent-software-engineer.md](./agent-software-engineer.md)** (tests, SOLID, lisibilité).

### Bonnes pratiques

- **Frontière data** : accès base via Prisma (client généré), pas de SQL ad hoc éparpillé dans les routes sans couche claire.
- **ADR** : consigner dans `docs/architecture/` les choix majeurs (auth, schéma, découpe modules, évolution de stack).
- **Emplacement code** : API et logique métier dans **`backend/`** (point d’entrée typique `backend/src/`).

---

## 6. Frontend React (Expert React)

### Stack et versions

- **React 19.x** (hooks, Server Components si applicable, Actions, rendu concurrent).
- **TypeScript** strict, interfaces explicites, unions discriminées si besoin.
- **Vite** (ou outil moderne équivalent) pour le build.

### Composants et hooks

- Uniquement des **composants fonctionnels** et des hooks (pas de classes).
- **Ref as prop** : `ref` passé comme prop classique (plus de `forwardRef` systématique).
- **Context** : Utiliser le rendu direct du context (sans wrapper Provider si le pattern React 19 est adopté).
- **Forms** : Actions API, `useFormStatus`, `useActionState` pour états de soumission et chargement.
- **Données async** : `use()` pour les promesses, Suspense pour le chargement, error boundaries pour les erreurs.
- **Optimistic UI** : `useOptimistic` pour les mises à jour optimistes.
- **Effets** : `useEffectEvent` (si dispo) pour extraire la logique non réactive des effets.

### Qualité et accessibilité

- **HTML sémantique** : `<button>`, `<nav>`, `<main>`, etc.
- **Accessibilité** : WCAG 2.1 AA, clavier, ARIA, contraste, focus visible.
- **Erreurs** : Error boundaries avec fallback et message clair.
- **Performance** : Code splitting (`React.lazy`), lazy loading des images, formats modernes (WebP/AVIF).

### Tests

- React Testing Library pour les composants.
- E2E (Playwright/Cypress) pour les parcours critiques.
- Tests unitaires pour la logique métier et les hooks personnalisés.

---

## 7. Atomic Design (Frontend & Backoffice)

L’**Atomic Design** est appliqué à la fois au **frontend** (site public) et au **backoffice** pour structurer les composants UI de manière réutilisable et scalable.

### Hiérarchie des niveaux

| Niveau        | Rôle                                      | Exemples (frontend / backoffice)                    |
|---------------|-------------------------------------------|-----------------------------------------------------|
| **Atoms**     | Éléments UI indivisibles, sans logique métier | `Button`, `Input`, `Label`, `Icon`, `Badge`         |
| **Molecules** | Combinaisons d’atomes, un rôle précis     | `SearchField`, `FormField`, `CardHeader`, `DataTableHeader` |
| **Organisms** | Blocs métier / sections de page           | `Header`, `LoginForm`, `ProductGrid`, `Sidebar`, `DataTable` |
| **Templates** | Mise en page (slots, grilles), sans contenu réel | `PageLayout`, `DashboardLayout`, `AuthLayout`       |
| **Pages**     | Instances de templates avec données et comportement | `HomePage`, `ProductListPage`, `UserEditPage`       |

### Conventions de dossiers (frontend et backoffice)

Organiser les composants par niveau Atomic dans `src/components` (ou équivalent) :

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

- **Un composant = un dossier** (nom du composant en PascalCase).
- Fichiers typiques : `[Nom].tsx`, `[Nom].test.tsx`, `[Nom].module.css` (ou équivalent selon le projet).
- Les **pages** consomment des **templates** et des **organisms** ; les templates ne contiennent que la structure (slots, layout).

### Règles Atomic

1. **Atoms** : Pas de dépendance à d’autres composants UI du projet (sauf design system partagé). Réutilisables partout.
2. **Molecules** : Composent uniquement des atoms (et éventuellement d’autres molecules). Une seule responsabilité (ex. « champ de recherche », « ligne d’en-tête de tableau »).
3. **Organisms** : Peuvent utiliser atoms, molecules et autres organisms. Représentent un bloc métier ou une section identifiable (header, formulaire de login, tableau de données).
4. **Templates** : Définissent la structure (grille, colonnes, zones) et des slots pour le contenu. Pas de données réelles ni de logique métier lourde.
5. **Pages** : Assemblage final : template + organisms + données (props, state, API). Point d’entrée des routes.

### UI-kit (source à copier)

Le projet dispose d’un **UI-kit source** à la racine : **`source/ui-kit/`**.

- **Emplacement** : `source/ui-kit/` (à la racine du dépôt).
- **Usage obligatoire** : les composants du UI-kit ne doivent **pas** être utilisés directement (pas d’import depuis `source/ui-kit` dans le frontend ou le backoffice). Ils sont à la racine précisément pour servir de **source à copier**.
- **Workflow** : lorsqu’un composant du UI-kit est nécessaire dans le **frontend** ou le **backoffice**, le **copier** dans l’application cible (par ex. `frontend/src/components/atoms/Button/` ou `backoffice/src/components/atoms/Button/`), puis l’adapter si besoin (styles, props, dépendances locales).
- **Avantages** : chaque app garde sa propre copie, évite les couplages directs à la racine, et permet des évolutions spécifiques (frontend vs backoffice) sans impacter l’autre.
- **À ne pas faire** : importer depuis `source/ui-kit` ou monter ce dossier en dépendance partagée ; toujours copier le code dans l’app qui en a besoin.

### Design system partagé (recommandé)

- La **source de vérité** pour les composants de base est `source/ui-kit/`. Frontend et backoffice s’en nourrissent en **copiant** les composants (voir ci-dessus).
- Documenter les props et les variants (Storybook ou équivalent dans `source/ui-kit` ou dans chaque app) pour faciliter le knowledge transfer.

---

## 8. UX / UI et recherche (UX/UI Designer)

### Avant de concevoir l’UI

- **Utilisateurs** : Rôle, niveau d’expertise, device principal, besoins d’accessibilité.
- **Contexte** : Moment et lieu d’usage, objectif réel (pas seulement la feature), fréquence, conséquences en cas d’échec.
- **Pain points** : Frustrations actuelles, blocages, contournements.

### Jobs-to-be-Done (JTBD)

- Formuler le « job » : *Quand [situation], je veux [motivation], pour [résultat].*
- Documenter la solution actuelle et ses limites.
- Fichiers recommandés : `docs/ux/[feature]-jtbd.md`, `docs/ux/[feature]-journey.md`, `docs/ux/[feature]-flow.md`.

### Parcours utilisateur (User Journey)

- Pour chaque flux important : étapes, ce que l’utilisateur fait/pense/ressent, points de douleur, opportunités d’amélioration.
- Utiliser ces livrables pour alimenter les maquettes (Figma) et les specs pour les développeurs.

### Accessibilité (checklist pour design et dev)

- Navigation clavier : tous les éléments interactifs accessibles, ordre de tab logique, focus visible.
- Screen readers : labels, alt, annonce des erreurs et du contenu dynamique, structure de titres.
- Visuel : contraste ≥ 4,5:1 (WCAG AA), zones de clic/touch ≥ 24×24px, pas d’information par la couleur seule.

---

## 9. Checklist de validation (résumé)

### Avant une action

- [ ] Critères de succès définis pour l’action.
- [ ] Méthode de validation identifiée.
- [ ] Exécution autonome (pas d’attente de validation externe).

### En fin de tâche / feature

- [ ] Exigences métier implémentées et validées.
- [ ] Documentation : vérifier si une documentation de la feature (nouvelle ou modifiée) existe, et la mettre à jour (ou la créer si absente).
- [ ] Décisions importantes enregistrées (ADR ou doc équivalente).
- [ ] Qualité : lisibilité, maintenabilité, gestion d’erreurs.
- [ ] Tests adaptés (E2E/intégration/unitaires) et passants.
- [ ] Workspace propre, handoff documenté, prochaines étapes identifiées.

### Front (frontend + backoffice)

- [ ] Composants alignés sur Atomic Design (atoms → pages).
- [ ] Composants UI-kit : copiés depuis `source/ui-kit/` dans l’app (pas d’import direct depuis la racine).
- [ ] Accessibilité et HTML sémantique respectés.
- [ ] Types TypeScript stricts pour les props et la state.

### Backend / API

- [ ] Stack **backend/** respectée (Express, Prisma, TypeScript) et migrations Prisma à jour si le schéma change.
- [ ] Couches Service / Manager / Résilience respectées (au sens **[agent-api-architect.md](./agent-api-architect.md)**), adaptées au langage du projet.
- [ ] DTOs et contrats d’API documentés (OpenAPI / équivalent si applicable).

---

## 10. Référence rapide des documents

| Type                    | Emplacement / convention                          |
|-------------------------|----------------------------------------------------|
| **Carte documentation (point d’entrée)** | [`docs/README.md`](../docs/README.md) — index ADR, guides, features, règles multi‑agent |
| **Vision / glossaire / historique besoins** | `docs/business/` — `VISION.md`, `GLOSSAIRE.md`, `HISTORIQUE-BESOINS.md` (voir `docs/business/README.md`) |
| Architecture Decision Record | `docs/architecture/ADR-[num]-[titre].md`     |
| JTBD / parcours UX      | `docs/ux/[feature]-jtbd.md`, `-journey.md`, `-flow.md` |
| **Doc de feature (Backoffice)** | `docs/features/BO-Feature-[NOM_DE_LA_FEATURE].md` |
| **Doc de feature (Frontend)** | `docs/features/FR-Feature-[NOM_DE_LA_FEATURE].md` |
| **Guides techniques** | `docs/guides/Guide-[Objet].md` |
| **Modèles (exemples)** | `docs/features/BO-Feature-Custom-Pages.md`, `docs/features/BO-Feature-Channels-Navigation.md` |
| Composants (Atomic)     | `frontend/src/components/{atoms,molecules,organisms,templates}`, idem pour `backoffice` |
| **UI-kit (source à copier)** | `source/ui-kit/` — copier dans frontend/backoffice, ne pas importer directement |
| Pages                   | `frontend/src/pages`, `backoffice/src/pages` (ou équivalent) |
| **Backend API**         | `backend/` — Express, Prisma, TypeScript |

---

**Document vivant** : ce fichier doit être mis à jour lorsque de nouvelles conventions ou technologies sont adoptées. Pour toute question sur son application, se référer en priorité à ce document et aux ADR dans `docs/architecture/`.
