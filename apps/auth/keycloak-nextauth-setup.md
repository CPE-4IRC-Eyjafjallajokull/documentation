# Keycloak + NextAuth — Configuration rapide

Cette configuration s'applique a **app-qg-front** et **app-terrain-front**.

---

## 1. Cote Keycloak

1. Creer un **Realm** (ex: `sdmis`).
2. Creer un **Client** (confidential).
3. Configurer les **Redirect URIs** :
   - `http://localhost:3000/api/auth/callback/keycloak` (QG)
   - `http://localhost:3002/api/auth/callback/keycloak` (Terrain)
4. Configurer les **Web Origins** :
   - `http://localhost:3000`
   - `http://localhost:3002`
5. Recuperer :
   - **Client Secret**
   - **Issuer URL** : `http://localhost:8080/realms/<realm>`

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

## 3. Demarrage

```bash
npm install
npm run dev
```

---

## 4. Checklist

- [ ] Keycloak demarre
- [ ] Realm + Client crees
- [ ] Redirect URIs corrects
- [ ] `.env.local` rempli
- [ ] Front accessible (3000 ou 3002)
