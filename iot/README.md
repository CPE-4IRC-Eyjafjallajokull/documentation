# Documentation IoT - Système de Communication Radio

## Vue d'ensemble

Ce système IoT permet la communication bidirectionnelle entre des véhicules de terrain équipés de micro:bit et l'API centrale via une passerelle RF. Il utilise le protocole CPE sécurisé (AES-128) pour transmettre les positions, statuts et affectations d'incidents.

## Architecture globale

```
┌───────────────────────┐
│  Véhicules terrain    │
│  (micro:bit #1, #2..) │
│  iot-terrain-microbit │
└──────────┬────────────┘
           │ Radio 2.4GHz
           │ (Protocole CPE)
           ↓
┌───────────────────────┐
│  Passerelle centrale  │
│  (micro:bit RX)       │
│  rf-central-gateway   │
│     └─ firmware       │
└──────────┬────────────┘
           │ UART 115200
           │ (CSV)
           ↓
┌───────────────────────┐
│  Gateway Python       │
│  rf_gateway/          │
│  └─ Serial ↔ RabbitMQ │
└──────────┬────────────┘
           │ RabbitMQ
           ↓
┌───────────────────────┐
│  API Backend          │
│  (FastAPI)            │
└───────────────────────┘
```

## Composants

### 1. Protocole CPE
Le protocole de communication sécurisé utilisé pour les échanges radio.
- **Fichier** : [Protocole/Protocole_CPE.md](Protocole/Protocole_CPE.md)
- **Code** : `firmware/source/proto/cpe/`
- **Chiffrement** : AES-128-CTR
- **Taille trame** : 30 octets

### 2. Librairie SDMIS_RADIO
Couche d'abstraction pour la communication radio sur micro:bit.
- **Fichier** : [Librairie/SDMIS_radio.md](Librairie/SDMIS_radio.md)
- **Code** : `firmware/source/lib/`
- **Fonctions** : ACK/Retry, déduplication, buffer circulaire

### 3. Applications micro:bit
Deux applications C++ pour micro:bit v1 :
- **Terrain** (`iot-terrain-microbit`) : Émetteur sur véhicules
- **Centrale** (`rf-central-gateway/firmware`) : Récepteur passerelle

### 4. Gateway Python
Service Python pour la conversion UART ↔ RabbitMQ.
- **Code** : `rf-central-gateway/gateway/`
- **Langage** : Python 3.11+
- **Librairies** : pyserial, pika, pydantic

## Flux de données

### Terrain → API (Télémétrie)
1. **Véhicule** envoie position/statut via UART CSV
2. **micro:bit terrain** encode en protocole CPE et transmet par radio
3. **micro:bit centrale** reçoit, décode et transmet en UART CSV
4. **Gateway Python** parse le CSV et publie sur RabbitMQ
5. **API** consomme les messages et met à jour la base de données

### API → Terrain (Affectations)
1. **API** publie une affectation d'incident sur RabbitMQ
2. **Gateway Python** consomme le message et envoie en UART CSV
3. **micro:bit centrale** encode en protocole CPE et transmet par radio
4. **micro:bit terrain** reçoit, décode et transmet en UART CSV
5. **Véhicule** affiche l'affectation

## Formats de données

### UART (CSV)
Format d'échange entre micro:bit et systèmes externes :
```
event,status,immat,lat,lon,timestamp
```

**Exemples** :
```csv
vehicle_position,1,AB123CD,48.858859,2.294481,1705140000
vehicle_status,2,AB123CD,0,0,1705140000
incident_status,3,AB123CD,0,0,1705140000
vehicle_affectation,0,AB123CD,48.860000,2.300000,1705140000
```

### Radio (Protocole CPE)
Trames binaires chiffrées de 30 octets transmises en 2.4 GHz.

### RabbitMQ (JSON)
Messages JSON structurés échangés avec l'API.

## Configuration

### Clé de chiffrement
Clé AES-128 partagée (identique sur tous les micro:bit) :
```c
const uint8_t CPE_KEY[16] = {
    0x21, 0x53, 0xB6, 0x09, 0x9A, 0xD2, 0x41, 0x7C,
    0xE4, 0x10, 0x5F, 0x3A, 0x77, 0xC8, 0x90, 0x0B
};
```

### Paramètres radio
- **Groupe terrain** : 9 (iot-terrain-microbit)
- **Groupe centrale** : 42 (rf-central-gateway)
- **Puissance** : 7 (maximum)
- **UART** : 115200 bauds

## Sécurité

- **Confidentialité** : Chiffrement AES-128 des données utiles
- **Intégrité** : Double CRC (header + payload)
- **Anti-replay** : Nonce et séquence uniques par trame
- **Validation** : Timestamps, format d'immatriculation, coordonnées GPS

## Documentation détaillée

Pour plus d'informations sur chaque composant, consultez :
- **[Système complet](Systeme_complet.md)** : Documentation détaillée de l'ensemble du système
- **[Protocole CPE](Protocole/Protocole_CPE.md)** : Format des trames radio
- **[Librairie SDMIS](Librairie/SDMIS_radio.md)** : API de communication micro:bit
