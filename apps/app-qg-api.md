# App QG API (FastAPI)

API REST centrale (FastAPI async) servant le front, la simulation et le moteur Java. Gère la normalisation des données entrantes et publie des événements temps réel (SSE).

## Rôle
- Point d'entrée HTTP unique (incidents, interventions, flux simulation/IOT).
- Connecteurs gérés en lifespan vers MongoDB, PostgreSQL et RabbitMQ (stockés sur `app.state`).
- Diffusion temps réel via endpoint SSE `/events`.

## Layout rapide
- `src/app/main.py` : factory FastAPI + wiring des connecteurs.
- `src/app/core/config.py` : réglages Pydantic (prefix env `APP_`).
- `src/app/api/routes/` : routes `health` et `events` (SSE) prêtes à étendre.
- `src/app/services/db` et `src/app/services/messaging` : helpers Mongo/Postgres/RabbitMQ.
- `tests/` : tests de fumée (health + events).

## Démarrage rapide
```bash
cd app-qg-api
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
uvicorn app.main:app --reload --app-dir src
```
Endpoints utiles :
- `GET /health` : OK JSON.
- `GET /events` : flux SSE (ping keepalive configurable).

## Configuration
Variables d'environnement (préfixe `APP_`) principales :
- `APP_DEBUG`, `APP_LOG_LEVEL`, `APP_APP_NAME`.
- `APP_MONGO_DSN`, `APP_POSTGRES_DSN`, `APP_RABBITMQ_DSN`.
- `APP_EVENTS_PING_INTERVAL_SECONDS`.

## À documenter/compléter
- Schémas des payloads incidents/interventions et mapping RabbitMQ.
- Modèle de données cible PostgreSQL + migrations.
- Intégration Keycloak (middlewares d'auth) et règles de rôles.
