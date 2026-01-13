# Passerelle RF centrale (rf-central-gateway)

La passerelle RF fait le lien entre les micro:bit (radio) et le backend (RabbitMQ).
Elle tourne sur un poste connecte en USB a une micro:bit centrale.

---

## Rôle

- **UART -> RabbitMQ** : positions et statuts vehicules.
- **RabbitMQ -> UART** : affectations vers le terrain.

---

## Démarrage rapide

Depuis la racine du projet :

```bash
cd rf-central-gateway
make gateway
cp gateway/.env.example gateway/.env
make run-gateway
```

---

## Configuration essentielle

Variables (.env) principales :

| Variable | Role | Exemple |
| --- | --- | --- |
| `RABBITMQ_DSN` | Connexion RabbitMQ | `amqp://sdmis:sdmis@localhost:5672/sdmis` |
| `RABBITMQ_QUEUE_TELEMETRY` | Queue telemetrie | `vehicle_telemetry` |
| `RABBITMQ_QUEUE_ASSIGNMENTS` | Queue affectations | `vehicle_assignments` |
| `SERIAL_PORT` | Port serie micro:bit | `/dev/ttyACM0` |
| `SERIAL_BAUD` | Vitesse UART | `115200` |

Mode CPE (radio chiffre) :
- `CPE_ENABLED=true`
- `CPE_KEY_HEX=<cle AES 128 en hex>`

---

## Format UART (CSV)

```
EVENT,STATUS,IMMAT,LAT,LON,TIMESTAMP
```

Exemples :
- `vehicle_position,1,AB123CD,45.76401,4.83572,1749821234`
- `vehicle_affectation,0,AB123CD,45.76123,4.83456,1749821234`

---

## Documentation associee

- Vue IoT globale : [README.md](README.md)
- Protocole CPE : [protocole/protocole-cpe.md](protocole/protocole-cpe.md)
