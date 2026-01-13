# SSE Quick Start

Objectif : verifier que les evenements temps reel circulent entre l'API et les fronts.

---

## Option 1 — Stack complete (recommande)

```bash
cd infrastructure-devops
cp env/.env.local.example env/.env.local
make up
```

Ensuite :
1. Configurez Keycloak (voir [auth/keycloak-nextauth-setup.md](auth/keycloak-nextauth-setup.md)).
2. Ouvrez le front QG : http://localhost:3000
3. Connectez-vous puis verifiez l'indicateur **Live**.

---

## Option 2 — Manuel (API + Front)

### API
```bash
cd app-qg-api
uv sync
uv run uvicorn app.main:app --reload --app-dir src
```

### Front QG
```bash
cd app-qg-front
cp .env.example .env.local
# Renseigner KEYCLOAK_* et API_URL
npm install
npm run dev
```

---

## Test simple avec curl

```bash
curl -N -H "Authorization: Bearer <token>" http://localhost:8000/qg/live
```

Vous devez voir :
- `connected`
- `heartbeat`
- puis les evenements du systeme.
