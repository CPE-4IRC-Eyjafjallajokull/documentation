# Keycloak – Architecture & Sécurité (SDMIS)

## 1. Objectif

Keycloak est utilisé comme serveur d’identité (IdP) pour centraliser :
- l’authentification des utilisateurs (QG et Terrain),
- l’émission de tokens (OpenID Connect),
- la gestion des rôles et des droits d’accès (RBAC).

L’objectif est de répondre à l’exigence **“accès restreints : authentification (mise en place d’un serveur Keycloak)”**
et de renforcer la sécurité globale de la plateforme (OWASP, séparation des responsabilités, contrôle d’accès).



## 2. Périmètre et composants concernés

### 2.1 Composants applicatifs

- **Keycloak** : serveur d’authentification (Realm `sdmis`)
- **Front QG** (`sdmis_front_qg`) : interface web opérateurs
- **Front Terrain** (`sdmis_front_terrain`) : interface équipes terrain
- **API QG** (`sdmis_api_qg`) : API REST protégée (resource server)
- **Engine Java** (`sdmis_engine`) : service interne (M2M)
- **RF Central** (`sdmis_rf_central`) : service interne (M2M)

### 2.2 Protocoles

- **OpenID Connect (OIDC)** sur OAuth 2.0
- Échanges sécurisés via **HTTPS** (TLS)



## 3. Modèle de sécurité retenu

### 3.1 Authentification utilisateur (Front QG / Front Terrain)

- Les utilisateurs s’authentifient via Keycloak.
- Les applications front utilisent le **Standard Flow (Authorization Code Flow)**.
- **PKCE S256** est activé pour réduire les risques d’interception du code d’autorisation.

**Pourquoi ?**
- Recommandé pour les applications web modernes
- Réduit les risques d’attaque type “authorization code interception”
- Évite l’usage de flux obsolètes (Implicit)

---

### 3.2 Protection de l’API (API QG)

- L’API ne fait pas de login utilisateur.
- Elle valide les **Bearer tokens** envoyés par les front-end (Access Token).
- Les droits sont contrôlés à partir des **rôles** contenus dans le token (RBAC).

**Rôle de l’API :**
- vérifier signature + expiration du token,
- vérifier audience / issuer,
- vérifier rôles/permissions nécessaires.

---

### 3.3 Authentification machine-to-machine (Engine / RF Central)

- Les services internes utilisent un **Client confidentiel** avec **Service Account**.
- Authentification via flux **Client Credentials**.
- Les services obtiennent un token “technique” pour appeler des endpoints internes.

**Pourquoi ?**
- Secret stockable côté serveur
- Aucun flux utilisateur exposé
- Cloisonnement strict des composants techniques



## 4. Flux d’authentification (séquence)

### 4.1 Login utilisateur (Front → Keycloak)

1. L’utilisateur clique “Se connecter”
2. Redirection vers Keycloak
3. Authentification (mot de passe)
4. Redirection vers le front avec un code
5. Échange code → tokens (Access + Refresh)

### 4.2 Appel API (Front → API)

1. Front appelle l’API avec `Authorization: Bearer <access_token>`
2. API valide le token (signature, exp, issuer)
3. API autorise/refuse selon les rôles

### 4.3 Appel M2M (Engine/RF → Keycloak → API)

1. Service s’authentifie via client credentials
2. Keycloak retourne un access token technique
3. Service appelle l’API avec ce token



## 5. Gestion des droits (RBAC)

Le contrôle d’accès est basé sur des **rôles** associés :
- aux utilisateurs (QG / Terrain),
- aux services (Engine / RF).

Exemples (à adapter à votre projet) :
- `ROLE_QG_OPERATOR`
- `ROLE_QG_ADMIN`
- `ROLE_TERRAIN_AGENT`
- `ROLE_ENGINE`
- `ROLE_RF_CENTRAL`

Ces rôles sont injectés dans les tokens et utilisés côté API pour appliquer les règles d’accès.



## 6. Mesures de sécurité appliquées

### 6.1 Réduction de la surface d’attaque
- Implicit flow désactivé
- Direct access grants désactivé (pas de login via mot de passe en API)
- Authorization Services Keycloak non utilisés (simplicité, maîtrise)

### 6.2 Bonnes pratiques sur les tokens
- Tokens courts (access token)
- Refresh token pour éviter les reconnexions fréquentes (selon besoin)
- Validation stricte côté API (issuer, audience, exp, signature)

### 6.3 Cloisonnement
- 1 client par application/service
- rôles dédiés par périmètre
- services internes isolés (M2M)

---

## 7. Points de vigilance / améliorations possibles

- Activer des politiques de mot de passe (longueur, complexité)
- Activer la MFA (TOTP) pour les comptes admin
- Séparer environnements (dev / prod) via realms distincts
- Mettre en place des logs d’audit (connexion, erreurs, admin events)
- Rotation des secrets des clients confidentiels



## 8. Conclusion

L’architecture Keycloak retenue permet :
- une authentification centralisée et standard (OIDC),
- une protection robuste de l’API par tokens,
- une séparation nette entre utilisateurs et services internes,
- une base saine pour la gestion des rôles et des permissions.
