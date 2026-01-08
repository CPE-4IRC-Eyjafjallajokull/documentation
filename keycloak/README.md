# Keycloak – Authentification & Sécurité  
## Projet SDMIS – 4IRC

Ce dépôt/documentation décrit la mise en place de **Keycloak** comme serveur d’authentification centralisé pour le projet **SDMIS** (Système de gestion et de suivi des incidents).

Keycloak est utilisé afin de :
- centraliser l’authentification des utilisateurs,
- sécuriser l’accès aux API,
- gérer les rôles et les droits,
- répondre aux exigences de sécurité du cahier des charges (OWASP, accès restreints).



## Vue d’ensemble

L’architecture d’authentification repose sur :
- **OpenID Connect (OIDC)** et **OAuth 2.0**
- des **clients distincts** pour chaque composant (Front, API, services internes)
- une séparation claire entre :
  - authentification utilisateur,
  - authentification machine-to-machine,
  - logique métier applicative.



## Composants concernés

| Composant | Rôle |
|---------|------|
| Keycloak | Serveur d’identité (IdP) |
| Front QG | Interface web des opérateurs |
| Front Terrain | Application des équipes terrain |
| API QG | API REST protégée |
| Engine Java | Moteur décisionnel interne |
| RF Central | Passerelle radiofréquence |



## Modèle de sécurité

- **Frontends (QG / Terrain)**  
  - Authentification utilisateur via Keycloak  
  - Authorization Code Flow + **PKCE S256**

- **API REST**  
  - Validation de **Bearer Tokens**
  - Contrôle d’accès basé sur les rôles (RBAC)

- **Services internes (Engine / RF)**  
  - Authentification machine-to-machine
  - Clients confidentiels + Service Accounts
  - Flux *client credentials*

Aucun flux obsolète ou risqué (Implicit, Direct Access Grants) n’est activé.



## Contenu de la documentation

### 1️.  `keycloak-installation.md`
> Mise en place et déploiement de Keycloak

- Prérequis
- Accès à la console d’administration
- Environnement d’exécution (Raspberry / Docker)
- Points de base pour l’exploitation

---

### 2. `keycloak-configuration.md`
> Configuration détaillée du Realm et des clients

- Création du Realm `sdmis`
- Paramètres globaux du Realm
- Liste et configuration des clients :
  - `sdmis_front_qg`
  - `sdmis_front_terrain`
  - `sdmis_api_qg`
  - `sdmis_engine`
  - `sdmis_rf_central`
- Justification des choix de sécurité
- Bonnes pratiques appliquées

---

### 3️. `keycloak-architecture-securite.md`
> Vision architecture & sécurité (STG)

- Rôle de Keycloak dans l’architecture globale
- Flux d’authentification (utilisateur et services)
- Modèle RBAC (rôles)
- Mesures de sécurité mises en œuvre
- Limitation de la surface d’attaque
- Pistes d’amélioration (MFA, audit, séparation des environnements)


##  Choix techniques clés

- OpenID Connect (standard moderne)
- PKCE S256 pour les clients publics
- Clients dédiés par application/service
- Tokens JWT signés et vérifiés côté API
- Cloisonnement strict des responsabilités

Ces choix sont alignés avec :
- les bonnes pratiques OAuth2/OIDC,
- les recommandations OWASP,
- les exigences du cahier des charges du projet.


##  Évolutions possibles

- Activation de la MFA pour les comptes sensibles
- Mise en place de logs d’audit Keycloak
- Séparation des environnements (dev / prod)
- Rotation des secrets clients
- Renforcement des politiques de mot de passe



##  Contexte pédagogique

Cette documentation s’inscrit dans le cadre du **Projet Scientifique 4IRC**  
et participe aux livrables suivants :
- Spécifications Techniques Générales (STG)
- Documentation de sécurité
- Justification des choix d’architecture



## Équipe

Projet réalisé par l’équipe Eyjafjallajökull – 4IRC  
Année universitaire 2025–2026



Features Possible --> Ajout de DPoP côté Serveur Keycloack





