# Shell SaaS - Monorepo

Application SaaS modulaire construite avec une architecture moderne en monorepo.

## 🏗️ Architecture

- **Frontend**: Vite 7 + React 19 + TypeScript + Tailwind CSS 4
- **Backend**: Node.js + Express + Prisma + TypeScript
- **Database**: PostgreSQL 17
- **Deployment**: Docker Compose

## 📦 Structure du projet

```
shell-saas/
├── frontend/          # Application React
├── backend/           # API Node.js
├── docker-compose.yml # Configuration Docker
└── package.json       # Configuration monorepo
```

## 🚀 Démarrage rapide

### Prérequis

- Node.js 20+
- Docker & Docker Compose
- npm ou yarn

### Installation

1. Cloner le repository
```bash
git clone <repository-url>
cd shell-saas
```

2. Copier le fichier d'environnement
```bash
cp .env.example .env
```

3. Installer les dépendances
```bash
npm run install:all
```

4. Démarrer avec Docker
```bash
docker-compose up
```

L'application sera accessible sur :
- Frontend : http://localhost:3000
- Backend API : http://localhost:3001
- PostgreSQL : localhost:5434

### Développement local (sans Docker)

1. Démarrer PostgreSQL
```bash
docker-compose up postgres -d
```

2. Dans un terminal - Backend
```bash
cd backend
npm install
npm run prisma:migrate
npm run dev
```

3. Dans un autre terminal - Frontend
```bash
cd frontend
npm install
npm run dev
```

## 🛠️ Scripts disponibles

### Racine du projet
- `npm run dev` - Démarre tous les services avec Docker
- `npm run install:all` - Installe toutes les dépendances
- `npm run build` - Build frontend et backend
- `npm run clean` - Nettoie les node_modules

### Frontend
- `npm run dev` - Mode développement
- `npm run build` - Build production
- `npm run preview` - Preview du build

### Backend
- `npm run dev` - Mode développement avec hot-reload
- `npm run build` - Compile TypeScript
- `npm run start` - Démarre en production
- `npm run prisma:migrate` - Applique les migrations Prisma
- `npm run prisma:studio` - Ouvre Prisma Studio

## 📚 Technologies

### Frontend
- **React 19** - Framework UI avec Server Components
- **Vite 7** - Build tool ultra-rapide
- **TypeScript** - Typage statique
- **Tailwind CSS 4** - Framework CSS utility-first
- **Lucide React** - Icônes modernes

### Backend
- **Node.js** - Runtime JavaScript
- **Express** - Framework web
- **Prisma 7.2** - ORM moderne
- **TypeScript** - Typage statique
- **PostgreSQL** - Base de données relationnelle

## 📝 Conventions de code

- Utiliser TypeScript strict mode
- Suivre les principes SOLID
- Tests unitaires et d'intégration
- Code review obligatoire

## 🐳 Docker

Les services sont orchestrés via Docker Compose :
- PostgreSQL sur le port 5434
- Backend sur le port 3001
- Frontend sur le port 3000

## 📄 License

MIT
