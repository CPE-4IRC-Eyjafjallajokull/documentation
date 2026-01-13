# Keycloak + NextAuth — Configuration rapide

Cette configuration s'applique a **app-qg-front** et **app-terrain-front**.

---

## 1. Cote Keycloak

### 1.1 Realm

1. Creer un **Realm** (ex: `sdmis`).
2. Recuperer l'**Issuer URL** : `http://localhost:8080/realms/<realm>`.

### 1.2 Roles (realm roles)

Creer les **Realm Roles** suivants (utilises par l'API) :
- `qg-operator` : acces complet
- `qg-engine` : acces complet (services)
- `qg-vehicles` : acces vehicules + lecture reste
- `qg-viewer` : lecture seule

Assigner ces roles aux utilisateurs ou aux groupes.

### 1.3 Clients

#### Fronts (NextAuth)

Creer 2 clients **confidential** :
- `app-qg-front`
- `app-terrain-front`

Parametres recommandes :
- **Client authentication** : ON (pour avoir un `Client Secret`)
- **Standard flow** : ON (Authorization Code)
- **Direct access grants** : OFF
- **Service accounts** : OFF
- **Valid Redirect URIs** :
  - `http://localhost:3000/api/auth/callback/keycloak` (QG)
  - `http://localhost:3002/api/auth/callback/keycloak` (Terrain)
- **Web Origins** :
  - `http://localhost:3000`
  - `http://localhost:3002`

Recuperer pour chaque front : **Client ID** + **Client Secret**.

#### API (audience)

Creer un client pour l'API (par defaut `app-qg-api`).

Parametres recommandes :
- **Client authentication** : ON
- **Standard flow** : OFF
- **Direct access grants** : OFF
- **Service accounts** : OFF (a activer uniquement si vous voulez du client credentials)

Ce client sert a l'**audience** dans le token, pas a l'authentification NextAuth.

### 1.4 Ajouter l'audience dans le token

L'API valide le claim `aud` (par defaut `app-qg-api`). Il faut donc que le token contienne cette audience.

Option simple (Keycloak >= 20) :
1. **Client Scopes** -> **Create** (ex: `qg-api-audience`).
2. Ajouter un **Protocol Mapper** de type **Audience** :
   - **Included Client Audience** : `app-qg-api`
   - **Add to access token** : ON
   - **Add to ID token** : OFF
3. Assigner ce scope comme **Default** sur les clients front (`app-qg-front`, `app-terrain-front`).

Alternative : ajouter directement un mapper Audience sur chaque client front.

### 1.5 Utilisateurs

- Creer les utilisateurs (ou groupes) et leur assigner les **realm roles**.
- Verifier dans un token que `realm_access.roles` contient les roles attendus.

---

## 2. Cote Front (.env.local)

Exemple pour **app-qg-front** :

```env
KEYCLOAK_ISSUER=http://localhost:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=app-qg-front
KEYCLOAK_CLIENT_SECRET=secret

NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=replace-with-random-secret
AUTH_TRUST_HOST=false

API_URL=http://localhost:8000
```

Pour **app-terrain-front**, adaptez `NEXTAUTH_URL` (port 3002) et le `CLIENT_ID`.

---

## 3. Cote API (app-qg-api)

Exemple d'environnement (les valeurs doivent matcher le realm/client) :

```env
KEYCLOAK_SERVER_URL=http://localhost:8080
KEYCLOAK_REALM=sdmis
KEYCLOAK_CLIENT_ID=app-qg-api
KEYCLOAK_AUDIENCE=app-qg-api
```

`KEYCLOAK_AUDIENCE` est optionnel (si absent, l'API utilise `KEYCLOAK_CLIENT_ID`).

---

## 4. Demarrage

```bash
npm install
npm run dev
```

---

## 5. Checklist

- [ ] Keycloak demarre
- [ ] Realm cree
- [ ] Roles (realm roles) crees et assignes
- [ ] Clients front crees + secrets recuperes
- [ ] Client API cree (`app-qg-api`)
- [ ] Audience ajoutee dans le token (`aud` contient `app-qg-api`)
- [ ] Redirect URIs corrects
- [ ] `.env.local` rempli
- [ ] Front accessible (3000 ou 3002)
