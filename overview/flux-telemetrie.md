# Flux terrain et telemetrie

Ce flux decrit comment les positions et statuts vehicules remontent au QG,
et comment les affectations redescendent vers le terrain.

---

## 1. Remontee des donnees terrain

Sources possibles :
- **Micro:bit terrain** (vehicules reels).
- **Simulation vehicules** (UART emule).

Cheminement :
1. Micro:bit terrain envoie une trame radio (protocole CPE).
2. Micro:bit centrale recoit et transmet en UART.
3. La passerelle RF (Python) convertit en JSON et publie sur RabbitMQ.
4. L'API consomme la queue `vehicle_telemetry` (et `incident_telemetry`).
5. L'API met a jour PostgreSQL.
6. L'API diffuse en SSE :
   - `vehicle_position_update`
   - `vehicle_status_update`
   - `incident_status_update`

---

## 2. Descente des affectations

1. L'API publie une affectation sur RabbitMQ (`vehicle_assignments`).
2. La passerelle RF lit la queue et envoie en UART.
3. La micro:bit centrale envoie la trame radio.
4. La micro:bit terrain affiche/consomme l'affectation.

---

## Format UART (simplifie)

```
event,status,immatriculation,lat,lon,timestamp
```

Exemples :
- `vehicle_position,1,AB123CD,45.76401,4.83572,1749821234`
- `vehicle_status,2,AB123CD,0,0,1749821234`
- `vehicle_affectation,0,AB123CD,45.76123,4.83456,1749821234`
