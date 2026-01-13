# SSE — App QG API

L'API expose un flux SSE pour les mises a jour temps reel des fronts.

---

## Endpoint

- `GET /qg/live`
- Authentification : Bearer JWT (Keycloak)
- Type : `text/event-stream`

Filtrage optionnel :

```
GET /qg/live?events=new_incident&events=vehicle_status_update
```

---

## Format des messages

Chaque evenement respecte le format SSE standard :

```
event: <nom_evenement>
data: {"event":"<nom>","data":{...},"timestamp":"2026-01-12T12:34:56Z"}
```

L'API envoie aussi :
- `connected` : message initial
- `heartbeat` : keep-alive

---

## Evenements principaux

- `new_incident`
- `incident_status_update`
- `incident_phase_update`
- `assignment_request`
- `assignment_proposal`
- `assignement_proposal_accepted` (orthographe conforme au code)
- `assignment_proposal_refused`
- `vehicle_assignment`
- `vehicle_position_update`
- `vehicle_status_update`

---

## Exemples

```bash
curl -N -H "Authorization: Bearer <token>" \
  "http://localhost:8000/qg/live?events=new_incident&events=vehicle_position_update"
```

---

## Configuration SSE

Variables (prefixe `APP_`) :

| Variable | Description | Defaut |
| --- | --- | --- |
| `APP_EVENTS_PING_INTERVAL_SECONDS` | Intervalle heartbeat | `25` |
| `APP_EVENTS_QUEUE_SIZE` | Taille de la file par client | `100` |
| `APP_EVENTS_QUEUE_OVERFLOW_STRATEGY` | `drop_newest`, `drop_oldest`, `block` | `drop_newest` |
