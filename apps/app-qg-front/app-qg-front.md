# App QG Front (Next.js)

Interface opérateur pour la gestion des incidents et véhicules en temps réel.

---

## 🎯 Rôle

- Visualiser incidents et véhicules sur une carte interactive
- Recevoir les événements temps réel via SSE
- Gérer les affectations de véhicules
- Administrer les données de référence (casernes, types...)

---

## 🏗️ Stack technique

| Technologie | Usage |
|-------------|-------|
| Next.js 16 (App Router) | Framework React |
| TypeScript | Typage |
| NextAuth v5 | Authentification Keycloak |
| MapLibre GL | Carte interactive |
| Radix UI + Tailwind | Composants UI |
| SWR | Fetching et cache |

---

## 🚀 Démarrage rapide

```bash
cd app-qg-front
pnpm install
pnpm dev
```

> Application sur http://localhost:3000

---

## 🔧 Configuration

Créer `.env.local` :

```env
# Keycloak
KEYCLOAK_ISSUER=http://localhost:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=frontend-client
KEYCLOAK_CLIENT_SECRET=your-secret

# NextAuth
NEXTAUTH_SECRET=random-secret-32-chars
NEXTAUTH_URL=http://localhost:3000

# API Backend
API_URL=http://localhost:3001
```

---

## 📁 Structure

| Dossier | Contenu |
|---------|---------|
| `app/` | Pages et routes API (App Router) |
| `app/api/` | Proxy vers l'API backend |
| `components/qg/` | Composants métier (carte, incidents) |
| `components/admin/` | Interface d'administration |
| `components/ui/` | Composants UI réutilisables |
| `hooks/` | Hooks React personnalisés |
| `lib/` | Services et utilitaires |
| `types/` | Types TypeScript |

---

## 📡 Pages principales

| Route | Description |
|-------|-------------|
| `/` | Dashboard QG (carte + incidents) |
| `/admin` | Administration des données |
| `/metrics` | Tableau de bord statistiques |
| `/auth/signin` | Connexion Keycloak |

---

## 🗺️ Carte interactive

- **Bibliothèque** : MapLibre GL + react-map-gl
- **Centre par défaut** : Lyon
- **Marqueurs** : Incidents, véhicules, points d'intérêt
- **Interactions** : Clic pour créer incident/point d'intérêt

---

## 🔐 Authentification

- NextAuth v5 avec provider Keycloak
- Token JWT transmis à l'API via proxy
- Routes protégées automatiquement

---

## 🐳 Docker

```bash
docker compose up --build
```

> ⚠️ `API_URL` est embarqué au build. Reconstruire l'image si modifié.

---

## 📜 Scripts

| Commande | Action |
|----------|--------|
| `pnpm dev` | Développement |
| `pnpm build` | Build production |
| `pnpm start` | Lancer le build |
| `pnpm lint` | Linter |
