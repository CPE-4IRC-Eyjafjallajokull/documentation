# Infrastructure DevOps

Point d'entrée unique pour lancer toute l'infrastructure du projet avec Docker.

---

## 🎯 Rôle

- Orchestrer tous les services (BDD, broker, auth, apps)
- Centraliser la configuration via un fichier `.env`
- Simplifier le développement local avec `make up`

---

## 🏗️ Structure

| Dossier | Contenu |
|---------|---------|
| `compose/local/` | Docker Compose pour le dev |
| `compose/prod/` | Docker Compose pour la prod |
| `env/` | Variables d'environnement |
| `data/` | Volumes persistants (postgres, rabbitmq, keycloak) |
| `database/` | Scripts SQL d'initialisation |
| `rabbitmq/` | Configuration RabbitMQ |
| `bin/` | Scripts helper (bootstrap, build) |

---

## 🚀 Démarrage rapide

```bash
cd infrastructure-devops
cp env/.env.local.example env/.env.local   # Créer le fichier de config
make up                                     # Démarrer tout
```

> Première exécution : les images sont buildées automatiquement.

---

## 📜 Commandes Make

| Commande | Action |
|----------|--------|
| `make up` | Démarrer la stack (build si nécessaire) |
| `make down` | Arrêter la stack |
| `make logs` | Voir les logs |
| `make databases` | Démarrer uniquement PostgreSQL + RabbitMQ |
| `make ps` | Lister les services en cours |
| `make clean` | Tout supprimer (⚠️ efface les données) |

---

## 🐳 Services inclus

| Service | Port | Description |
|---------|------|-------------|
| PostgreSQL | 5432 | Base de données |
| RabbitMQ | 5672 / 15672 | Messagerie + UI management |
| Keycloak | 8080 | Authentification |
| app-qg-api | 8000 | API backend |
| app-qg-front | 3000 | Frontend |
| app-qg-java-engine | - | Moteur décisionnel |

### Simulateurs (optionnels)

```bash
COMPOSE_PROFILES=sim make up
```

Active `simulation-java-incidents` et `simulation-java-vehicles`.

---

## 🔧 Configuration

### Fichier `.env.local`

```env
# PostgreSQL
POSTGRES_USER=pt_user
POSTGRES_PASSWORD=pt_password
POSTGRES_DB=pt

# RabbitMQ
RABBITMQ_DEFAULT_USER=pt_rabbit
RABBITMQ_DEFAULT_PASS=pt_rabbit_password

# Keycloak
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=admin

# Ports
API_HTTP_PORT=8000
FRONT_HTTP_PORT=3000
```

> Voir `env/.env.local.example` pour la liste complète.

---

## 📁 Prérequis

- Docker + Docker Compose v2
- Repos clonés au même niveau :
  ```
  projet-pompier/
  ├── infrastructure-devops/
  ├── app-qg-api/
  ├── app-qg-front/
  ├── app-qg-java-engine/
  └── ...
  ```

---

## �️ Base de données

Les scripts SQL dans `database/init/` sont exécutés automatiquement au premier démarrage :

| Fichier | Rôle |
|---------|------|
| `00-users.sql` | Création des rôles et BDD (Keycloak, SDMIS) |
| `10-sdmis.sql` | Charge le schéma SDMIS |
| `sdmis/01-tables.sql` | Tables SDMIS |
| `sdmis/02-constraints.sql` | Contraintes |
| `sdmis/03-data.sql` | Données de référence |
| `sdmis/04-business-data.sql` | Données métier (véhicules, incidents) |
| `sdmis/05-additional-vehicles.sql` | Véhicules supplémentaires |

---

## 🐰 RabbitMQ

Configuration dans `rabbitmq/` :
- `rabbitmq.conf` — Config principale
- `definitions.json.template` — Exchanges, queues, bindings pré-configurés

---

## 🔄 Ajout d'un nouveau service

1. Ajouter dans `compose/local/docker-compose.yml`
2. Ajouter dans `compose/prod/docker-compose.yml`
3. Compléter `env/.env.local.example`

---

## ⚠️ Notes

- Les données sont dans `data/` — supprimer pour repartir à zéro
- Secrets : ne jamais commiter `.env.local`, utiliser `.env.local.example`
- Prod : utiliser des images prébuilt via `REGISTRY` et `TAG`
- Prod : utiliser des images prébuilt via `REGISTRY` et `TAG`
