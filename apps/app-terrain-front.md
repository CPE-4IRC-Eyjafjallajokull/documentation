# App Terrain Front (Next.js)

Interface terrain (React/Next.js) pour les opérateurs sur le terrain : gestion des véhicules, visualisation des incidents et suivi en temps réel des interventions.

## Rôle
- Affichage en temps réel des véhicules, incidents et interventions via l'API QG.
- Gestion des statuts de véhicules (Disponible, Engagé, Sur intervention, etc.).
- Navigation entre incidents et véhicules avec liens bidirectionnels.
- Authentification Keycloak via NextAuth et mise à jour temps réel via SSE.
- Interface optimisée pour les opérateurs terrain avec actions rapides.

## Stack
- Next.js 16.1.1 (App Router) + TypeScript + Turbopack.
- React 19 avec hooks personnalisés (`useVehicles`, `useIncidents`, `useLiveEvent`).
- UI : shadcn/ui + Tailwind CSS 4 + Lucide icons.
- Authentification : NextAuth v5 avec Keycloak.
- Images optimisées : Next.js Image avec fallback.

## Structure
```
app/
  ├── vehicles/          # Page de gestion des véhicules
  ├── incidents/         # Page de gestion des incidents  
  ├── api/               # Routes API proxy vers le backend
  │   ├── vehicles/      # Proxy PATCH /vehicles/{id}
  │   └── incidents/     # Proxy incidents
  └── auth/              # Pages d'authentification

components/
  ├── vehicles/          # Composants véhicules (card, detail, filters)
  ├── incidents/         # Composants incidents (list, detail, engagement)
  ├── ui/                # Composants UI réutilisables (shadcn)
  └── auth-provider.tsx  # Provider d'authentification

lib/
  ├── vehicles/          # Services et types véhicules
  ├── incidents/         # Services et types incidents
  ├── api-proxy.ts       # Proxy générique vers l'API backend
  └── auth-redirect.ts   # Gestion auth et redirections

hooks/
  ├── useVehicles.ts     # Hook de gestion des véhicules
  ├── useIncidents.ts    # Hook de gestion des incidents
  └── useLiveEvent.ts    # Hook SSE temps réel

public/
  └── vehicles/          # Images des véhicules (vehicle_{CODE}.png)
```

## Démarrage rapide
```bash
cd app-terrain-front
npm install
npm run dev
# navigateur sur http://localhost:3000
```

## Configuration
Variables d'environnement principales :

**Publiques (NEXT_PUBLIC_)** :
- `NEXT_PUBLIC_API_URL` : URL de l'API backend

**Serveur** :
- `AUTH_SECRET` : Secret NextAuth
- `AUTH_KEYCLOAK_ID` : Client ID Keycloak
- `AUTH_KEYCLOAK_SECRET` : Client secret Keycloak
- `AUTH_KEYCLOAK_ISSUER` : URL Keycloak issuer
- `API_URL` : URL interne de l'API (pour SSR)

## Fonctionnalités principales

### Gestion des véhicules
- Liste complète avec filtres (recherche, station, statut, type)
- Détail véhicule avec informations complètes (type, énergie, position, affectation)
- **Changement de statut** : Interface de sélection avec 7 statuts
  - Code couleur : Rouge (Hors service), Bleu (Engagé), Vert (Disponible), Orange (Sur intervention), Violet (Retour), Bleu ciel (Transport), Gris (Indisponible)
  - API : `PATCH /api/vehicles/{vehicle_id}` avec `status_id`
  - Optimisation : refetch uniquement la liste des véhicules (pas tous les types/statuts/stations)
- Images des véhicules avec normalisation des codes (PC_Mobile → PC Mobile)
- Auto-sélection via URL param `?selected={immatriculation}`

### Navigation incidents ↔ véhicules
- Depuis un incident : clic sur véhicule engagé → page véhicules avec auto-sélection
- Depuis un véhicule : lien vers l'incident d'affectation (si `incident_id` disponible)

### Temps réel
- SSE pour mise à jour live des positions et statuts
- Refetch automatique après actions utilisateur
- Mise à jour du véhicule sélectionné quand les données changent

## Routes API (proxy vers backend)

| Route Frontend                      | Méthode | Route Backend           | Usage                          |
| ----------------------------------- | ------- | ----------------------- | ------------------------------ |
| `/api/vehicles`                     | GET     | `qg/vehicles`           | Liste tous les véhicules       |
| `/api/vehicles/{vehicle_id}`        | PATCH   | `vehicles/{vehicle_id}` | Mise à jour véhicule (statut)  |
| `/api/vehicles/types`               | GET     | `vehicles/types`        | Types de véhicules             |
| `/api/vehicles/statuses`            | GET     | `vehicles/statuses`     | Statuts disponibles            |
| `/api/incidents`                    | GET     | `qg/incidents`          | Liste des incidents            |
| `/api/incidents/{id}/engagements`   | GET     | `qg/incidents/{id}/...` | Véhicules engagés sur incident |

## Hooks personnalisés

### useVehicles
```typescript
const {
  vehicles,              // Liste complète
  filteredVehicles,      // Filtrée par critères
  vehicleTypes,          // Types disponibles
  vehicleStatuses,       // Statuts disponibles
  stations,              // Casernes pour filtres
  isLoading,
  error,
  filters,
  selectedVehicle,       // Véhicule sélectionné
  setFilters,
  resetFilters,
  selectVehicle,
  refetch,               // Recharge tout
  refetchVehicles,       // Recharge uniquement véhicules (optimisé)
} = useVehicles();
```

**Optimisations** :
- `refetchVehicles()` au lieu de `refetch()` après changement de statut (plus rapide)
- Auto-update du `selectedVehicle` quand les données changent
- Filtrage avec `useMemo` pour performances

### useLiveEvent
```typescript
const { status, error } = useLiveEvent({
  events: ['vehicle_position', 'vehicle_status'],
  onEvent: (event, data) => {
    // Traitement événement temps réel
  }
});
```

## Architecture technique

### API Proxy Pattern
- Routes `/api/*` proxifiées vers le backend via `proxyApiRequest()`
- Gestion auth avec `fetchWithAuth()` (injection token session)
- Parsing centralisé des réponses avec `parseResponseBody()`

### Gestion d'état
- Hooks personnalisés avec `useState`, `useCallback`, `useMemo`
- Pas de Redux/Zustand : état local aux composants/hooks
- Optimisations : éviter re-renders inutiles avec `useCallback`

### Images véhicules
- Path : `/vehicles/vehicle_{CODE}.png`
- Normalisation : espaces et tirets → underscores
- Fallback : image par défaut si inexistante
- Next.js Image avec `unoptimized` pour assets statiques

## Conventions de code

### Composants
- Client components : `"use client"` en haut
- Props typées avec TypeScript
- Déstructuration des props pour clarté

### Services
- Fonctions async/await
- Gestion erreurs avec try/catch
- Types retour explicites

### Styling
- Tailwind CSS avec classes utilitaires
- shadcn/ui pour composants de base
- Classes conditionnelles avec `clsx` ou `cn()`

## Problèmes résolus

### Changement de statut véhicule
**Problème initial** : Erreur 404/502, route inexistante
**Solution** :
- Utilisation de `PATCH /api/vehicles/{vehicle_id}` (route backend existante)
- Passage de `status_id` au lieu de `status_label`
- Route proxy créée : `/api/vehicles/[vehicleId]/route.ts`

### Performance refetch
**Problème** : Tout recharger après changement de statut (lent)
**Solution** :
- Nouveau hook `refetchVehicles()` qui recharge uniquement la liste
- `refetch()` gardé pour boutons "Actualiser" (recharge tout)
- `refetchVehicles()` utilisé après changement de statut

### Mise à jour selectedVehicle
**Problème** : Véhicule sélectionné pas mis à jour après refetch
**Solution** :
- `useEffect` dans `useVehicles` qui détecte changement de `vehicles[]`
- Trouve et remplace `selectedVehicle` par version mise à jour

## Points à compléter
- Documentation détaillée des événements SSE disponibles
- Guide de contribution et standards de code
- Tests unitaires et e2e (Playwright/Jest)
- Documentation des types TypeScript complexes
- Guide de déploiement et CI/CD
