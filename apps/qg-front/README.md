# App QG Front (Next.js)

Interface operateur pour la gestion des incidents et vehicules en temps reel.

---

## Role

- Visualiser incidents et vehicules sur une carte.
- Valider les propositions d'affectation.
- Administrer les donnees de reference.
- Recevoir les evenements temps reel (SSE).

---

## Stack technique

| Technologie | Usage |
| --- | --- |
| Next.js 16 (App Router) | Framework React |
| TypeScript | Typage |
| NextAuth v5 | Authentification Keycloak |
| MapLibre GL | Carte interactive |
| Radix UI + Tailwind | UI |
| SWR | Fetching et cache |

---

## Demarrage rapide

```bash
cd app-qg-front
cp .env.example .env.local
npm install
npm run dev
```

> Front sur http://localhost:3000

---

## Configuration (.env.local)

```env
KEYCLOAK_ISSUER=http://localhost:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=app-qg-front
KEYCLOAK_CLIENT_SECRET=secret

NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=replace-with-random-secret
AUTH_TRUST_HOST=false

API_URL=http://localhost:8000
```

---

## Structure

| Dossier | Contenu |
| --- | --- |
| `app/` | Pages + routes API (proxy) |
| `components/qg/` | Composants metier |
| `hooks/` | Hooks react (SSE, data) |
| `lib/` | Services et utilitaires |
| `types/` | Types TypeScript |

---

## Pages principales

| Route | Description |
| --- | --- |
| `/` | Dashboard QG (carte + incidents) |
| `/admin` | Administration |
| `/metrics` | Statistiques |
| `/auth/signin` | Connexion Keycloak |

---

## SSE (temps reel)

- Proxy SSE : `GET /api/live` -> `GET /qg/live`
- Provider global : `LiveEventsProvider`
- Evenements principaux :
  - `new_incident`
  - `vehicle_position_update`
  - `vehicle_status_update`
  - `incident_status_update`
  - `incident_phase_update`
  - `assignment_proposal`
  - `assignement_proposal_accepted`
  - `assignment_proposal_refused`

Details : [sse.md](sse.md)

---

## Docker

```bash
docker compose up --build
```

> `API_URL` est embarque au build : reconstruire si modifie.
