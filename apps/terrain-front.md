# App Terrain Front (Next.js)

Interface terrain pour les equipes de vehicules : suivi des incidents,
statuts et actions rapides.

---

## Role

- Selection d'un vehicule terrain.
- Suivi de l'intervention en cours.
- Mise a jour des statuts et positions en temps reel (SSE).
- Consultation des incidents, casernes et ressources.

---

## Stack

- Next.js 16 (App Router) + TypeScript
- NextAuth v5 avec Keycloak
- MapLibre GL, shadcn/ui, Tailwind CSS

---

## Demarrage rapide

```bash
cd app-terrain-front
cp .env.example .env.local
npm install
npm run dev
```

> Front sur http://localhost:3002

---

## Configuration (.env.local)

```env
KEYCLOAK_ISSUER=http://localhost:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=app-terrain-front
KEYCLOAK_CLIENT_SECRET=secret

NEXTAUTH_URL=http://localhost:3002
NEXTAUTH_SECRET=replace-with-random-secret
AUTH_TRUST_HOST=false

API_URL=http://localhost:8000
```

---

## Pages principales

| Route | Description |
| --- | --- |
| `/` | Selection vehicule + dashboard terrain |
| `/vehicles` | Liste et details vehicules |
| `/incidents` | Liste incidents |
| `/fire-stations` | Casernes |

---

## SSE (temps reel)

- Proxy SSE : `GET /api/events` -> `GET /qg/live`
- Evenements ecoutes : positions, statuts, affectations

---

## Notes

L'app s'appuie sur les routes `/api/*` pour injecter le token Keycloak
lors des appels au backend.
