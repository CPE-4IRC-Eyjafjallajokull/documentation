# Flux incidents

Ce flux decrit comment un incident est cree, traite et suivi jusqu'a la fin
d'une intervention.

---

## 1. Creation de l'incident

Sources possibles :
- **Operateur QG** via l'interface.
- **Simulation incidents** (tests et demo).

Traitement :
- L'API enregistre l'incident et sa phase initiale.
- L'API emet un evenement SSE `new_incident` pour rafraichir le front.

---

## 2. Demande d'affectation

Quand un incident demande des ressources :
- L'API construit la liste des besoins (types de vehicules, quantites).
- L'API publie un message RabbitMQ `assignment_request` sur la queue `sdmis_engine`.
- L'API emet un evenement SSE `assignment_request`.

---

## 3. Proposition automatique (moteur Java)

- Le moteur Java consomme `assignment_request`.
- Il interroge l'API (incidents, vehicules, routage).
- Il calcule un classement (distance, temps, energie).
- Il publie une proposition `assignment_proposal` sur `sdmis_api`.

---

## 4. Validation ou refus

- L'API recoit la proposition et la stocke.
- L'operateur accepte ou refuse.
- L'API emet :
  - `assignement_proposal_accepted` (orthographe conforme au code), ou
  - `assignment_proposal_refused`.

---

## 5. Affectation et suivi

- L'API cree les affectations vehicule <-> incident.
- Les fronts recoivent les mises a jour :
  - `vehicle_assignment`
  - `incident_status_update`
  - `incident_phase_update`

---

## Evenements SSE utilises

- `new_incident`
- `assignment_request`
- `assignment_proposal`
- `assignement_proposal_accepted`
- `assignment_proposal_refused`
- `vehicle_assignment`
- `incident_status_update`
- `incident_phase_update`
