# App QG API (FastAPI)

API REST centrale du SDMIS. Gère les incidents, véhicules, opérateurs et diffuse des événements temps réel.

---

## 🎯 Rôle

- Centraliser les données (incidents, véhicules, victimes)
- Authentifier via Keycloak (JWT)
- Diffuser des événements temps réel (SSE)
- Communiquer avec RabbitMQ pour les messages asynchrones

---

## 🏗️ Architecture

Projet **Python 3.12+** avec **FastAPI** (async).

| Dossier | Contenu |
|---------|---------|
| `src/app/main.py` | Point d'entrée + lifespan |
| `src/app/core/` | Config, logging, sécurité Keycloak |
| `src/app/api/routes/` | Endpoints HTTP |
| `src/app/models/` | Modèles SQLAlchemy |
| `src/app/schemas/` | Schémas Pydantic |
| `src/app/services/` | DB, SSE, RabbitMQ, géocodage |

---

## 🚀 Démarrage rapide

```bash
cd app-qg-api
uv sync                                              # Installation
uv run uvicorn app.main:app --reload --app-dir src   # Lancement
```

> API disponible sur http://localhost:8000

---

## 🔧 Configuration

### Variables principales

| Variable | Description | Défaut |
|----------|-------------|--------|
| `POSTGRES_DSN` | Connexion PostgreSQL | `postgresql+asyncpg://...` |
| `RABBITMQ_DSN` | Connexion RabbitMQ | `amqp://guest:guest@localhost:5672/` |
| `KEYCLOAK_SERVER_URL` | URL Keycloak | `http://localhost:8080` |
| `KEYCLOAK_REALM` | Realm Keycloak | `master` |
| `KEYCLOAK_CLIENT_ID` | Client ID | `app-qg-api` |
| `AUTH_DISABLED` | Désactiver l'auth (dev) | `false` |
| `APP_LOG_LEVEL` | Niveau de log | `INFO` |
| `APP_CORS_ORIGINS` | Origines CORS | `*` |

### Exemple `.env`

```env
POSTGRES_DSN=postgresql+asyncpg://postgres:postgres@localhost:5432/sdmis
KEYCLOAK_SERVER_URL=http://localhost:8080
KEYCLOAK_REALM=sdmis
KEYCLOAK_CLIENT_ID=app-qg-api
```

> Voir `.env.example` pour toutes les options.

---

## 📡 Endpoints principaux

### Publics

| Route | Description |
|-------|-------------|
| `/health` | Healthcheck |
| `/docs` | Documentation Swagger |

### Protégés (JWT requis)

| Route | Description |
|-------|-------------|
| `/qg/live` | Flux SSE temps réel |
| `/qg/incidents` | Incidents côté QG |
| `/qg/vehicles` | Véhicules côté QG |
| `/incidents` | CRUD incidents |
| `/incidents/phase/types` | Types de phases |
| `/vehicles` | CRUD véhicules |
| `/vehicles/assignments` | Affectations |
| `/casualties` | Victimes |
| `/operators` | Opérateurs |
| `/interest-points` | Casernes, hôpitaux |
| `/geo/address/reverse` | Géocodage inverse |

---

## 📊 Événements SSE

Connexion au flux `/qg/live` :

```bash
curl -N -H "Authorization: Bearer <token>" http://localhost:8000/qg/live
```

| Événement | Description |
|-----------|-------------|
| `new_incident` | Nouvel incident créé |
| `incident_status_update` | Changement statut incident |
| `incident_phase_update` | Mise à jour phase |
| `assignment_request` | Demande d'affectation |
| `assignment_proposal` | Proposition d'affectation |
| `vehicle_assignment` | Véhicule affecté |
| `vehicle_position_update` | Position GPS véhicule |
| `vehicle_status_update` | Changement statut véhicule |

---

## 🗄️ Modèles de données

| Modèle | Description |
|--------|-------------|
| `Incident` | Incident déclaré |
| `IncidentPhase` | Phase d'un incident |
| `PhaseType` | Type de phase (incendie, SAP...) |
| `Vehicle` | Véhicule de secours |
| `VehicleType` | Type (VSAV, FPT...) |
| `VehicleAssignment` | Affectation véhicule → incident |
| `Operator` | Opérateur QG |
| `Casualty` | Victime |
| `InterestPoint` | Caserne, hôpital |

---

## 🐳 Docker

```bash
docker compose --profile dev up   # Développement (avec services)
docker compose up -d              # Production
```

---

## 🧪 Tests

```bash
uv run pytest
```

---

## 🌍 Géocodage inverse

`GET /geo/address/reverse?lat=<lat>&lon=<lon>`

- Source : Nominatim (OpenStreetMap)
- Rate limit : 1 req/s
- Cache : 10 minutes
