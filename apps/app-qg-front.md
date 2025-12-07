# App QG Front (Next.js)

Interface opérateur (React/Next.js) pour visualiser incidents, interventions et ressources en temps réel.

## Rôle
- Affichage carte + listes (incidents, interventions, véhicules) alimentées par l'API QG.
- Authentification Keycloak (à brancher) et rafraîchissement continu via SSE/WebSocket ou polling.
- Actions opérateur : création/validation d'interventions, suivi des messages terrain.

## Stack
- Next.js (App Router) + TypeScript.
- UI composants dans `components/` et logique partagée dans `lib/`.
- Assets publics dans `public/`.

## Démarrage rapide
```bash
cd app-qg-front
npm install
npm run dev
# navigateur sur http://localhost:3000
```

## Points à compléter
- Intégration des endpoints `/health` et `/events` du backend (SSE) + sécurisation Keycloak.
- Cartographie (lib choisie) et conventions de typage des payloads API.
- Guidelines UI/UX (états de chargement, accessibilité, thèmes) à ajouter ici.
