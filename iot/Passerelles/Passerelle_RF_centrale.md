# Passerelle RF Centrale

Récepteur radio côté QG : reçoit les trames des véhicules terrain et les retransmet vers l'API backend.

## 🎯 Vue d'ensemble

Le système se compose de deux parties complémentaires :

| Composant | Rôle | Technologie |
|-----------|------|-------------|
| **Firmware** | Réception radio, transmission UART | C/C++ (yotta) |
| **Gateway Python** | Lecture série, envoi API, écoute SSE | Python 3.9+ |

```
┌─────────────────┐     Radio      ┌─────────────────┐
│   Émetteurs     │ ─────────────► │   micro:bit     │
│   (véhicules)   │ ◄───────────── │   (récepteur)   │
└─────────────────┘   Groupe 42    └────────┬────────┘
                                            │ UART 115200
                                   ┌────────▼────────┐
                                   │   Gateway       │
                                   │   (Python)      │
                                   └────────┬────────┘
                                            │ HTTP/SSE
                                   ┌────────▼────────┐
                                   │   API Backend   │
                                   └─────────────────┘
```

---

## 📡 Firmware micro:bit (récepteur)

### Configuration radio

Identique à l'émetteur terrain (voir [iot-terrain-microbit](iot-terrain-microbit.md)) : groupe 42, puissance 7, AES-128 + CMAC.

### Flux de communication

**Radio → UART (terrain vers QG) :**
- Réception des positions : `vehicle_position,IMMAT,LAT,LON,TIMESTAMP`
- Réception des statuts : `vehicle_status,IMMAT,STATUS,TIMESTAMP`
- Format UART avec checksum : `$payload*XX\n`

**UART → Radio (QG vers terrain) :**
- Envoi des affectations : `vehicle_affectation,IMMAT,LAT,LON,TIMESTAMP`
- Transmission radio avec acquittement (3 tentatives max)

### Build & Flash

```bash
cd rf-central-gateway
make clean
make firmware        # Build via Docker
cp ./firmware/build/bbc-microbit-classic-gcc/source/rf-central-gateway-combined.hex /Volumes/MICROBIT/ # Flash sur MAC
```

---

## 🐍 Gateway Python

Passerelle bidirectionnelle entre le micro:bit et l'API backend.

### Flux de données

| Direction | Source | Destination | Endpoint API |
|-----------|--------|-------------|--------------|
| UART → API | Position véhicule | Backend | `POST /qg/vehicles/{immat}/position` |
| UART → API | Statut véhicule | Backend | `POST /qg/vehicles/{immat}/status` |
| SSE → UART | Affectation incident | micro:bit | Event `vehicle_assignment` |

### Installation & Lancement

```bash
cd rf-central-gateway

# Installation
make gateway

# Configuration
cp gateway/.env.example gateway/.env
# Éditer .env avec les paramètres

# Lancement
make run-gateway
```

### Configuration (.env)

| Variable | Description | Défaut |
|----------|-------------|--------|
| `API_BASE_URL` | URL de l'API | `https://api.sdmis.mathislambert.fr` |
| `API_TOKEN` | Token d'authentification | - |
| `SERIAL_PORT` | Port série | `/dev/ttyACM0` |
| `SERIAL_BAUD` | Baudrate | `115200` |

---

## 📊 Format des trames UART

Format avec checksum XOR : `$<payload>*<checksum_hex>\n`

| Type | Payload |
|------|---------|
| Position | `vehicle_position,AB123CD,48.856614,2.352222,1736172600` |
| Statut | `vehicle_status,AB123CD,1,1736172600` |
| Affectation | `vehicle_affectation,SD304FR,45.797200,4.847000,1736172600` |

---

## 💡 Indicateurs LED

| Position | Signification |
|----------|---------------|
| Pixel (4,4) fixe | Système actif |
| Pixel (0,0) blink | Position reçue/transmise |
| Pixel (2,0) blink | Statut reçu |
| Pixel (4,0) blink | Affectation envoyée (ACK reçu) |
