# Keycloak + NextAuth Integration

## 📋 Vue d'ensemble

Cette intégration fournit une authentification sécurisée via Keycloak pour l'application QG Front. Les utilisateurs doivent se connecter avant d'accéder à l'application.

## 🔐 Flux d'authentification

```
┌─────────────┐
│   Browser   │
│             │
│ /auth/signin├─────────────────┐
│             │                 │
└─────────────┘                 │
      │                         │
      │ Click "Sign in"         │
      │                         │
      └────────────────────────→┌──────────────────┐
                                │   Keycloak       │
                                │   Login Page      │
                                └──────────────────┘
                                         │
                                User logs in
                                         │
                                         ↓
                        ┌────────────────────────┐
                        │  OAuth2 Callback       │
                        │ /api/auth/callback     │
                        └────────────────────────┘
                                         │
                                         ↓
                        ┌────────────────────────┐
                        │  Session Created       │
                        │  Access Token stored   │
                        └────────────────────────┘
                                         │
                                         ↓
                        ┌────────────────────────┐
                        │  Redirect to /         │
                        │  User authenticated    │
                        └────────────────────────┘
```

## ⚙️ Configuration Keycloak

### Étape 1: Créer un Realm (si nécessaire)
1. Connectez-vous à Keycloak Admin Console
2. Créez un nouveau realm (ex: `qg-realm`)

### Étape 2: Créer un Client
1. Allez à Clients → Create
2. **Client ID**: `app-qg-front`
3. **Client Protocol**: `openid-connect`
4. **Access Type**: `confidential`

### Étape 3: Configurer le Client
Dans l'onglet **Settings**:
- **Valid Redirect URIs**: 
  ```
  http://localhost:3000/api/auth/callback/keycloak
  https://yourdomain.com/api/auth/callback/keycloak
  ```
- **Web Origins**: 
  ```
  http://localhost:3000
  https://yourdomain.com
  ```
- **Valid Post Logout Redirect URIs**:
  ```
  http://localhost:3000
  ```

### Étape 4: Récupérer les Credentials
Dans l'onglet **Credentials**:
- Copier **Client Secret**

### Étape 5: Récupérer l'Issuer URL
Format: `http://localhost:8080/realms/your-realm-name`

Ou via l'endpoint de découverte:
```
http://localhost:8080/realms/your-realm-name/.well-known/openid-configuration
```

## 🔧 Configuration Frontend

### Variables d'environnement (`.env.local`)

```env
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:8000

# NextAuth Configuration
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=generate-a-random-secret-key

# Keycloak Configuration
KEYCLOAK_CLIENT_ID=app-qg-front
KEYCLOAK_CLIENT_SECRET=your-client-secret-from-keycloak
KEYCLOAK_ISSUER=http://localhost:8080/realms/your-realm-name
```

**Pour générer NEXTAUTH_SECRET:**
```bash
openssl rand -base64 32
```

## 📁 Structure des fichiers créés

```
app-qg-front/
├── auth.ts                           # Configuration NextAuth
├── middleware.ts                     # Protection des routes
├── .env.local                        # Variables d'environnement
├── app/
│   ├── api/auth/[...nextauth]/
│   │   └── route.ts                 # Handlers NextAuth
│   ├── auth/
│   │   ├── signin/page.tsx          # Page de connexion
│   │   └── error/page.tsx           # Page d'erreur
│   ├── layout.tsx                   # Enveloppe AuthProvider
│   ├── page.tsx                     # Accueil (protégé)
│   └── events/page.tsx              # Events (protégé)
└── components/
    ├── auth-provider.tsx             # SessionProvider wrapper
    ├── protected-route.tsx           # Composant de protection
    └── header.tsx                    # Header avec user info
```

## 🚀 Utilisation

### 1. Protéger une page entière

Page `app/events/page.tsx` est déjà protégée via le middleware.

### 2. Protéger un composant

```typescript
"use client";

import { useSession } from "next-auth/react";
import { ProtectedRoute } from "@/components/protected-route";

export default function MyPage() {
  return (
    <ProtectedRoute>
      <div>Content visible only to authenticated users</div>
    </ProtectedRoute>
  );
}
```

### 3. Accéder aux infos utilisateur

```typescript
"use client";

import { useSession } from "next-auth/react";

export default function MyComponent() {
  const { data: session } = useSession();

  return (
    <div>
      <p>Welcome, {session?.user?.name}!</p>
      <p>Email: {session?.user?.email}</p>
      <p>Access Token: {session?.accessToken}</p>
    </div>
  );
}
```

### 4. Utiliser le token d'accès pour les appels API

```typescript
"use client";

import { useSession } from "next-auth/react";

export default function MyComponent() {
  const { data: session } = useSession();

  const fetchData = async () => {
    const response = await fetch("http://localhost:8000/api/data", {
      headers: {
        "Authorization": `Bearer ${session?.accessToken}`,
      },
    });
    // ...
  };

  return <button onClick={fetchData}>Fetch Data</button>;
}
```

## 🔄 Flux de Token Refresh

La configuration NextAuth gère automatiquement le refresh du token si celui-ci expire. Le callback `jwt` vérifie l'expiration et renouvelle le token si nécessaire.

## 🛡️ Routes protégées vs publiques

### Routes protégées (authentification requise)
- `/` (accueil)
- `/events` (SSE streaming)
- `/dashboard` (ajouté à titre d'exemple)

### Routes publiques
- `/auth/signin` (page de connexion)
- `/auth/error` (page d'erreur)

### Routes API publiques
- `/api/auth/*` (NextAuth endpoints)

## 🧪 Test en développement

1. **Démarrer l'app**:
   ```bash
   npm run dev
   ```

2. **Accéder à l'app**:
   ```
   http://localhost:3000
   ```

3. **Vous serez redirigé vers**:
   ```
   http://localhost:3000/auth/signin
   ```

4. **Cliquer sur "Sign in with Keycloak"** et vous serez redirigé vers Keycloak

5. **Après connexion, vous reviendrez à** `http://localhost:3000`

## ⚠️ Notes importantes

### Développement
- `NEXTAUTH_SECRET` peut être n'importe quoi en développement
- Keycloak doit être accessible sur `http://localhost:8080`

### Production
- Générer un `NEXTAUTH_SECRET` sécurisé
- Utiliser HTTPS partout
- Configurer les URLs Keycloak avec le domaine de production
- Utiliser les variables d'environnement du serveur (pas de `.env.local`)

## 🔐 Sécurité

- Les tokens sont stockés dans les cookies HTTP-only (sécurisé contre XSS)
- Le refresh token est géré par NextAuth
- Les routes protégées redirigent automatiquement vers `/auth/signin`
- Les erreurs d'authentification sont capturées et affichées

## 📚 Ressources

- [NextAuth.js Documentation](https://next-auth.js.org/)
- [NextAuth Keycloak Provider](https://next-auth.js.org/providers/keycloak)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [OpenID Connect](https://openid.net/connect/)

---

**Last Updated**: December 11, 2025
