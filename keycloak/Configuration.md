
# Configuration Keycloak - Création d'un Realm

## Prérequis
- Container Keycloak installé sur le Raspberry de Mathis.
- Vérifier l'accessibilité à la page d'administration

## Créer un Realm

### 1. Accéder à la console d'administration
```
https://auth.mathislambert.fr
```

### 2. Création d'un Realm (sdmis)
1. Se connecter à l'interface d'administration à l'aide du compte administrateur fourni par défaut.
2. Cliquer sur la rubrique "Manage Realms" (présente dans le panneau de gauche).
3. Cliquer sur "**Create Realm**".
4. Stipuler un "**Realm Name**", dans notre cas on a utilisé "**sdmis**". Le realm permet de spécifier un espace isolé pour gérer les utilisateurs, clients et configurations d'authentification.

## Configuration de base du Realm

### Activer les fonctionnalités
1. Aller dans **Realm Settings**
2. Onglet **General**:
    - Vérifier que le Realm est activé
3. Onglet **Login** :
    - Activer "User registration"
    - Activer "Forgot password"

### Créer un Client

1. Avant de créer un client, il faut s'assurer que l'on se trouve dans le bon "**Realm**" visible en haut à gauche de la page.
2. Cliquer sur la rubrique **Clients** puis "**Create Client**".
3. Dans la partie "**General settings**" :
    - Sélectionner le type de client "**OpenID Connect**"
    - **Client ID** (champ unique permettant d'identifier l'application) :
        - "**sdmis_front_qg**"
        - "**sdmis_front_terrain**"
        - "**sdmis_api_qg**"
    - **Name** (affiché dans l'interface Keycloak) : entrer le nom du client
4. Cliquer sur "**Next**"
5. Pour la partie "**Capability config**" deux configurations différentes sont à appliquer :
    - **Pour la configuration des clients fronts :**
        - Dans un premier temps, il faut activer l'option "**Client authentification**" afin que le client puisse s'authentifier de manière sécurisée et qu'il ne soit pas exposé sur internet.
        - Il est nécessaire de laisser les choix proposés par défaut, avec le "**standard flow**" coché, celui-ci permet de gérer le flux d'authentification standard.
        - Pour la méthode "**PKCE Method**", permettant de sécuriser les applications publiques, nous avons sélectionné "**S256**"
        - Cliquer sur "**Next**"
    - **Pour la configuration du back :**
        - ----A FINALISER----

6. Pour la partie "**Login settings**", il faut renseigner les routes ci-dessous pour les champs "**Root URL**" et "**Home URL**" :
    - **API :** `https://api.sdmis.mathislambert.fr/`
    - **FRONT QG :** `https://sdmis.mathislambert.fr/`
    - **FRONT APP TERRAIN :** `https://terrain.sdmis.mathislambert.fr/`

    💡 Les routes sont à appliquer de façon unique par application

7. Configuration supplémentaire :
    - **Valid redirect URIs :** `http://localhost:3000/*`
    - **Web origins :** `http://localhost:3000`

### Créer un utilisateur
1. Aller dans **Users**
2. Cliquer sur "Add user"
3. Remplir les informations (username, email, etc.)
4. Onglet **Credentials**: définir un mot de passe
5. Cliquer sur "Set Password"
6. Entrer le mot de passe et confirmer
7. Désactiver "Temporary" si vous voulez que le mot de passe soit permanent
8. Cliquer sur "Set Password"

## Tester la configuration

### Accéder à la page de connexion
1. Naviguer vers `http://localhost:3000`
2. Cliquer sur le bouton de connexion
3. Vous devriez être redirigé vers Keycloak
4. Se connecter avec l'utilisateur créé

### Vérifier les tokens
1. Après connexion, ouvrir les DevTools (F12)
2. Onglet **Storage** → **Cookies**
3. Vérifier la présence du token de session

## Prochaines étapes
- Configurer les rôles et permissions
- Ajouter des providers d'authentification externes (Google, GitHub, etc.)
- Mettre en place les politiques de sécurité