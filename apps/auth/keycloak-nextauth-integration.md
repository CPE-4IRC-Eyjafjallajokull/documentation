# Keycloak + NextAuth Integration

Integration Keycloak pour les fronts (QG et Terrain).
L'utilisateur doit se connecter avant d'acceder aux pages protegees.

---

## Flux d'authentification

```
Utilisateur -> /auth/signin
  -> redirection Keycloak
  -> callback /api/auth/callback/keycloak
  -> session NextAuth
  -> retour sur l'app
```

---

## Variables d'environnement

Exemple (app-qg-front) :

```env
KEYCLOAK_ISSUER=http://localhost:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=app-qg-front
KEYCLOAK_CLIENT_SECRET=secret

NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=replace-with-random-secret
AUTH_TRUST_HOST=false

API_URL=http://localhost:8000
```

Pour app-terrain-front : ajuster `NEXTAUTH_URL` (3002) et le `CLIENT_ID`.

---

## Fichiers principaux

```
auth.ts
middleware.ts
app/api/auth/[...nextauth]/route.ts
app/auth/signin/page.tsx
app/auth/error/page.tsx
components/auth-provider.tsx
```

---

## Acces aux tokens

```tsx
"use client";

import { useSession } from "next-auth/react";

export function MyComponent() {
  const { data: session } = useSession();
  return <div>Token: {session?.accessToken}</div>;
}
```

---

## Appels API via proxy

Les fronts utilisent des routes `/api/*` qui ajoutent le token et appellent l'API QG.
Cela evite de gerer le JWT cote navigateur.
