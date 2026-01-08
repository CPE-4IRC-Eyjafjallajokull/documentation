
# Configuration Keycloak - Création d'un Realm

## 1. Prérequis
- Container Keycloak installé sur le Raspberry de Mathis.
- Vérifier l'accessibilité à la page d'administration



## 2. Créer un Realm

### 2.1 Accéder à la console d'administration
```
https://auth.mathislambert.fr
```

### 2.2 Création d'un Realm `sdmis`
1. Se connecter à l'interface d'administration avec le compte administrateur.
2. Cliquer sur la rubrique **Manage Realms**
3. Cliquer sur "**Create Realm**".
4. Renseigner le **Realm Name** : `sdmis`

Le Realm permet de définir un espace isolé pour la gestion :
- des utilisateurs,
- des clients,
- des règles d’authentification.



## 3. Configuration de base du Realm

### 3.1 Paramètres généraux

1. Aller dans **Realm Settings**
2. Onglet **General**
   - Vérifier que le Realm est activé

### 3.2 Paramètres de connexion

1. Onglet **Login**
   - Activer **User registration**
   - Activer **Forgot password**




## 4. Création et configuration des clients

Dans le Realm **sdmis**, plusieurs clients ont été créés afin de séparer clairement les responsabilités entre :
- les interfaces utilisateur (Front-end),
- les services backend,
- les composants techniques internes.

Chaque client correspond à une application ou un service distinct, ce qui permet :
- un cloisonnement des accès,
- une gestion fine des droits,
- une meilleure sécurité globale.



## 4.1 Vue d’ensemble des clients

| Client ID | Description | Type | Usage principal |
|---------|-------------|------|----------------|
| `sdmis_front_qg` | Front – UI QG | Public | Interface web du QG |
| `sdmis_front_terrain` | Front – App Terrain | Public | Interface terrain |
| `sdmis_api_qg` | API REST – Backend QG | Confidential | API sécurisée |
| `sdmis_engine` | Engine Java SDMIS | Confidential | Moteur décisionnel |
| `sdmis_rf_central` | Passerelle RF centrale | Confidential | Communication RF |
| `security-admin-console` | Console admin Keycloak | Interne | Administration |



### 4.2 Détail des clients

---

### 4.2.1 `sdmis_front_qg`

**Description**  
Client représentant l’interface web utilisée par les opérateurs du QG SDMIS.

**Type de client**
- Public
- OpenID Connect

**Configuration principale**
- Name : Front - UI QG
- Client Authentication : On désactivé
- Standard Flow : activé
- PKCE Method : `S256`
- Root URL : `https://sdmis.mathislambert.fr/`
- Home URL : `https://sdmis.mathislambert.fr/`
- Valid Redirect URIs : `https://sdmis.mathislambert.fr/*`

**Justification des choix**

- Le **Standard Flow (Authorization Code Flow)** est le flux recommandé pour les applications web.
- L’utilisation de **PKCE (S256)** permet de :
  - protéger contre les attaques par interception de code,
  - renforcer la sécurité même en cas d’exposition du client.
- Les flux **Implicit** et **Direct Access Grants** sont volontairement désactivés car :
  - obsolètes ou déconseillés,
  - plus vulnérables dans un contexte web moderne.
- L’Authorization Services de Keycloak n’est pas utilisée à ce stade, les autorisations étant gérées via :
  - rôles,
  - contrôles côté API.

**Résumé sécurité**

- Authentification sécurisée via OpenID Connect
- Flux standard uniquement
- Protection PKCE activée
- Pas de flux legacy ou risqués
- Client isolé fonctionnellement des autres composants

---

### 4.2.2 `sdmis_front_terrain`

**Description**  
Client représentant l’application utilisée par les équipes du SDMIS sur le terrain.

**Type de client**
- Public
- OpenID Connect

**Configuration principale**
- Name : Front – App Terrain
- Client Authentication : On désactivé
- Standard Flow : activé
- PKCE Method : `S256`
- Root URL : `https://terrain.sdmis.mathislambert.fr/`
- Home URL : `https://terrain.sdmis.mathislambert.fr/`
- Valid Redirect URIs : `https://terrain.sdmis.mathislambert.fr/*`

**Justification des choix**

- Le **Standard Flow (Authorization Code Flow)** est utilisé afin de permettre une authentification utilisateur interactive et sécurisée.
- L’utilisation de **PKCE (S256)** renforce la sécurité lors de l’échange du code d’autorisation, ce qui est particulièrement important pour une application exposée.
- Les flux **Implicit** et **Direct Access Grants** sont désactivés afin de :
  - limiter la surface d’attaque,
  - respecter les bonnes pratiques OAuth2/OpenID Connect.
- Les mécanismes d’autorisation avancés de Keycloak ne sont pas activés, les droits étant gérés via :
  - les rôles utilisateurs,
  - les contrôles d’accès implémentés côté API.

**Résumé sécurité**

- Authentification sécurisée via OpenID Connect
- Flux standard uniquement
- Protection PKCE activée
- Aucun flux obsolète ou risqué activé
- Client dédié exclusivement à l’application terrain

---

### 4.2.3 `sdmis_api_qg`

**Description**  
Client représentant l’API REST du backend QG SDMIS.  
Cette API est utilisée par les applications front-end afin d’accéder aux données et aux fonctionnalités métier du système.

**Type de client**
- Public
- OpenID Connect

**Configuration principale**
- Name : API REST – Backend QG
- Client Authentication : Off
- Standard Flow : désactivé
- PKCE Method : non configuré
- Authorization : désactivé
- Root URL : `https://api.sdmis.mathislambert.fr/`
- Home URL : `https://api.sdmis.mathislambert.fr/`
- Valid Redirect URIs : `https://api.sdmis.mathislambert.fr/*`

**Justification des choix**

- Ce client ne correspond pas à une application interactive mais à une **API exposant des endpoints REST**.
- Aucun flux d’authentification utilisateur (**Standard Flow**) n’est activé, car l’API ne gère pas de connexion directe.
- L’API se base sur des **tokens d’accès (Bearer tokens)** émis par Keycloak lors de l’authentification des utilisateurs via les clients front-end.
- L’API se contente de :
  - vérifier la validité des tokens,
  - contrôler les rôles et permissions présents dans ceux-ci.
- Les mécanismes d’autorisation avancés de Keycloak ne sont pas utilisés, les règles d’accès étant appliquées côté backend.

**Résumé sécurité**

- API protégée par des tokens OpenID Connect
- Aucun flux d’authentification exposé
- Pas de stockage de secret côté API
- Validation systématique des tokens côté serveur
- Séparation claire entre authentification (Keycloak) et logique métier (API)

---

### 4.2.4 `sdmis_engine`

**Description**  
Client représentant le moteur Java du système SDMIS.  
Ce composant interne est chargé de la logique métier avancée, notamment l’analyse des incidents et la proposition d’affectation des ressources.

**Type de client**
- Confidential
- OpenID Connect

**Configuration principale**
- Name : Engine JAVA of SDMIS App
- Client Authentication : On
- Service accounts roles : activé
- Standard Flow : désactivé
- PKCE Method : non configuré
- Authorization : désactivé

**Justification des choix**

- Ce client correspond à un **service backend interne**, sans interaction directe avec un utilisateur.
- L’activation de **Client Authentication** permet l’utilisation d’un **client secret**, stocké de manière sécurisée côté serveur.
- Le moteur s’authentifie auprès de Keycloak via un **Service Account**, en utilisant le flux *client credentials*.
- Aucun flux utilisateur (Standard Flow, Implicit Flow) n’est activé, car ce composant ne gère pas de connexion interactive.
- Les autorisations avancées de Keycloak ne sont pas utilisées, les contrôles d’accès étant appliqués via :
  - les rôles associés au service account,
  - la logique métier du moteur.

**Résumé sécurité**

- Authentification machine-to-machine sécurisée
- Utilisation d’un client confidentiel avec secret
- Aucun flux utilisateur exposé
- Rôles techniques dédiés au moteur
- Cloisonnement strict entre services internes et applications utilisateur

---

### 4.2.5 `sdmis_rf_central`

**Description**  
Client représentant le composant central de communication radiofréquence (RF) du système SDMIS.  
Il assure la réception et la transmission des données issues des dispositifs RF ou de leur simulation vers le reste du système.

**Type de client**
- Confidential
- OpenID Connect

**Configuration principale**
- Name : RF Central
- Client Authentication : On
- Service accounts roles : activé
- Standard Flow : désactivé
- PKCE Method : `S256`
- Authorization : désactivé

**Justification des choix**

- Ce client correspond à un **service technique interne**, sans interaction directe avec un utilisateur.
- L’activation de **Client Authentication** permet l’utilisation d’un **client secret**, garantissant une authentification sécurisée côté serveur.
- Le composant RF central s’authentifie auprès de Keycloak via un **Service Account**, selon un modèle *machine-to-machine*.
- Aucun flux d’authentification utilisateur (Standard Flow, Implicit Flow) n’est activé, car ce service ne propose pas d’interface de connexion.
- Le mécanisme **PKCE (S256)** est activé par cohérence de configuration, bien que le service utilise principalement un flux technique.
- Les services d’autorisation avancés de Keycloak ne sont pas utilisés, les contrôles d’accès étant gérés :
  - par les rôles associés au service account,
  - par la logique applicative des composants consommateurs.

**Résumé sécurité**

- Authentification machine-to-machine sécurisée
- Client confidentiel avec secret stocké côté serveur
- Utilisation d’un service account dédié
- Aucun flux utilisateur exposé
- Cloisonnement strict des composants techniques RF


---------------------------------------------------------------------------------------------------------------------------------------
