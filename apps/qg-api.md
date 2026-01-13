# App QG API (FastAPI)

API REST centrale du systeme SDMIS. Elle gere les incidents, vehicules,
operateurs et diffuse les evenements temps reel.

---

## Role

- Centraliser les donnees (incidents, vehicules, victimes, points d'interet).
- Authentifier via Keycloak (JWT).
- Diffuser les evenements SSE vers les fronts.
- Orchestrer les messages RabbitMQ.

---

## Demarrage rapide

```bash
cd app-qg-api
uv sync
uv run uvicorn app.main:app --reload --app-dir src
```

> API sur http://localhost:8000

---

## Configuration principale

| Variable | Description | Defaut |
| --- | --- | --- |
| `POSTGRES_DSN` | Connexion PostgreSQL | `postgresql+asyncpg://...` |
| `RABBITMQ_DSN` | Connexion RabbitMQ | `amqp://guest:guest@localhost:5672/` |
| `KEYCLOAK_SERVER_URL` | URL Keycloak | `http://localhost:8080` |
| `KEYCLOAK_REALM` | Realm Keycloak | `master` |
| `KEYCLOAK_CLIENT_ID` | Client ID | `app-qg-api` |
| `APP_CORS_ORIGINS` | Origines CORS | `*` |

---

## Endpoints principaux

### Publics
- `GET /health`
- `GET /docs`

### Proteges (JWT)
- `GET /qg/live` : flux SSE
- `POST /qg/incidents/new` : creation incident
- `POST /qg/incidents/{id}/request-assignment` : demande d'affectation
- `GET /qg/vehicles` : vue QG des vehicules
- `GET /terrain/interest-points/{kind_id}` : points d'interet terrain
- `GET /geo/address/reverse` : geocodage inverse
- `POST /geo/route` : calcul d'itineraire

---

## SSE (evenements)

- `new_incident`
- `incident_status_update`
- `incident_phase_update`
- `assignment_request`
- `assignment_proposal`
- `assignement_proposal_accepted`
- `assignment_proposal_refused`
- `vehicle_assignment`
- `vehicle_position_update`
- `vehicle_status_update`

Details : [qg-api-sse.md](qg-api-sse.md)

---

## RabbitMQ (queues)

- `sdmis_api` (SUB) : propositions d'affectation
- `sdmis_engine` (PUB) : demandes d'affectation
- `vehicle_telemetry` (SUB) : positions/statuts
- `vehicle_assignments` (PUB) : affectations terrain
- `incident_telemetry` (SUB) : statuts incident

---

## Modeles principaux

- `Incident`, `IncidentPhase`, `PhaseType`
- `Vehicle`, `VehicleAssignment`, `VehicleStatus`
- `Operator`, `Casualty`, `InterestPoint`
