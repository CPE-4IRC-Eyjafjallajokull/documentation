# Keycloak + NextAuth Integration - Quick Summary

## 5 Étapes principales

### 1️⃣ **Configuration Keycloak**

Sur votre instance Keycloak (http://localhost:8080):

1. Créer un **Realm** (ex: `qg-realm`)
2. Créer un **Client** `app-qg-front`
3. Configurer les **Redirect URIs**:
   - `http://localhost:3000/api/auth/callback/keycloak`
4. Copier le **Client Secret** dans l'onglet Credentials
5. Noter l'**Issuer URL**: `http://localhost:8080/realms/qg-realm`

### 2️⃣ **Configuration Frontend (.env.local)**

```env
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=<generate-random-string>
KEYCLOAK_CLIENT_ID=app-qg-front
KEYCLOAK_CLIENT_SECRET=<from-keycloak>
KEYCLOAK_ISSUER=http://localhost:8080/realms/qg-realm
NEXT_PUBLIC_API_URL=http://localhost:8000
```

### 3️⃣ **Fichiers créés**

| Fichier | Rôle |
|---------|------|
| `auth.ts` | Configuration NextAuth + Keycloak provider |
| `middleware.ts` | Protège les routes côté serveur |
| `app/api/auth/[...nextauth]/route.ts` | Handlers NextAuth |
| `app/auth/signin/page.tsx` | Page de connexion |
| `app/auth/error/page.tsx` | Page d'erreur auth |
| `components/auth-provider.tsx` | SessionProvider wrapper |
| `components/protected-route.tsx` | Composant de protection |
| `components/header.tsx` | Header avec user info + logout |

### 4️⃣ **Modifications existantes**

- **`app/layout.tsx`** - Enveloppé avec `<AuthProvider>`
- **`.env.local`** - Ajout variables Keycloak

### 5️⃣ **Flux utilisateur**

```
User → /  
  ↓  
Redirigé vers /auth/signin (pas authentifié)  
  ↓  
Click "Sign in with Keycloak"  
  ↓  
Redirect vers Keycloak login  
  ↓  
Utilisateur rentre identifiants  
  ↓  
Keycloak envoie le code à /api/auth/callback/keycloak  
  ↓  
NextAuth échange le code pour un token  
  ↓  
Session créée et stockée  
  ↓  
Redirect vers home (/)  
  ↓  
Utilisateur connecté ✅
```

## 🚀 Démarrage rapide

### Avec Keycloak local (Docker)

```bash
# Démarrer Keycloak
docker run -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  keycloak/keycloak:latest \
  start-dev

# Puis configurer manuellement depuis le dashboard admin
```

### Sans Keycloak (test basique)

Vous pouvez tester avec un autre provider en modifiant `auth.ts`:
```typescript
// Remplacer KeycloakProvider par GithubProvider, GoogleProvider, etc.
import GitHubProvider from "next-auth/providers/github";

providers: [
  GitHubProvider({
    clientId: process.env.GITHUB_ID || "",
    clientSecret: process.env.GITHUB_SECRET || "",
  }),
],
```

## 📝 Code clés

### Accéder à la session utilisateur

```typescript
"use client";
import { useSession } from "next-auth/react";

export default function MyComponent() {
  const { data: session, status } = useSession();
  
  if (status === "loading") return <p>Loading...</p>;
  if (status === "unauthenticated") return <p>Not signed in</p>;
  
  return <p>Welcome {session?.user?.name}</p>;
}
```

### Appeler l'API avec token

```typescript
const response = await fetch("http://localhost:8000/api/endpoint", {
  headers: {
    Authorization: `Bearer ${session?.accessToken}`,
  },
});
```

### Protéger une page

La page est automatiquement protégée via le **middleware**. Si utilisateur n'est pas connecté, redirect vers `/auth/signin`.

## ✅ Checklist de configuration

- [ ] Keycloak instance running
- [ ] Realm créé dans Keycloak
- [ ] Client créé dans Keycloak
- [ ] Redirect URIs configurées dans Keycloak
- [ ] Client Secret copié
- [ ] `.env.local` rempli avec valeurs Keycloak
- [ ] `NEXTAUTH_SECRET` généré
- [ ] `npm install` exécuté
- [ ] Frontend lancé avec `npm run dev`
- [ ] Test: accéder à `http://localhost:3000`

## 🐛 Troubleshooting

**Erreur: "Invalid redirect_uri"**
- Vérifiez que `http://localhost:3000/api/auth/callback/keycloak` est dans Keycloak Redirect URIs

**Erreur: "Invalid client id or secret"**
- Vérifiez `KEYCLOAK_CLIENT_ID` et `KEYCLOAK_CLIENT_SECRET` dans `.env.local`

**Session non persistée**
- Vérifiez `NEXTAUTH_SECRET` est défini
- Vérifiez `NEXTAUTH_URL` = `http://localhost:3000`

**Token expiré**
- NextAuth renouvelle automatiquement via le refresh token
- Vérifiez que Keycloak envoie le `refresh_token` au client

---

**Documentation complète**: `documentation/apps/keycloak-nextauth-integration.md`
