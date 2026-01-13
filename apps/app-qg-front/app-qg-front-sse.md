# SSE - App QG Front

Intégration Server-Sent Events pour les mises à jour temps réel.

---

## 🎯 Principe

```
Frontend  ──►  /api/live (proxy)  ──►  API /qg/live
                    │
            Ajoute le token JWT
```

Le proxy Next.js injecte automatiquement le token d'authentification.

---

## 🏗️ Architecture

### Provider global

```tsx
// app/layout.tsx
<LiveEventsProvider>
  {children}
</LiveEventsProvider>
```

Gère la connexion SSE, la reconnexion automatique et distribue les événements.

### Composants

| Composant | Rôle |
|-----------|------|
| `LiveEventsProvider` | Contexte React, connexion SSE globale |
| `useLiveEvent(event, handler)` | Hook pour s'abonner à un événement |
| `/api/live` | Proxy Next.js, injecte le JWT |

---

## 📊 Événements disponibles

| Événement | Description |
|-----------|-------------|
| `new_incident` | Nouvel incident créé |
| `incident_status_update` | Changement de statut incident |
| `incident_phase_update` | Mise à jour d'une phase |
| `vehicle_position_update` | Nouvelle position GPS |
| `vehicle_status_update` | Changement de statut véhicule |
| `vehicle_assignment` | Affectation véhicule |
| `assignment_proposal` | Proposition d'affectation |

---

## 💡 Utilisation

### Écouter un événement

```tsx
import { useLiveEvent } from "@/hooks/useLiveEvent";

useLiveEvent("new_incident", (event) => {
  console.log("Nouvel incident:", event.data);
});
```

### Exemple complet avec état

```tsx
"use client";

import { useLiveEvent } from "@/hooks/useLiveEvent";
import { useState } from "react";

export function VehicleTracker() {
  const [positions, setPositions] = useState(new Map());

  useLiveEvent("vehicle_position_update", (event) => {
    const { vehicle_id, latitude, longitude } = event.data;
    setPositions(prev => new Map(prev).set(vehicle_id, { latitude, longitude }));
  });

  return <div>{positions.size} véhicules suivis</div>;
}
```

---

## 🔧 Configuration

| Variable | Description |
|----------|-------------|
| `API_URL` | URL du backend (pour le proxy) |

---

## ⚠️ Notes

- Connexion SSE nécessite une session authentifiée
- Le proxy `/api/live` injecte automatiquement le token JWT
- Types définis dans `lib/sse/types.ts`
- Reconnexion automatique en cas de coupure
