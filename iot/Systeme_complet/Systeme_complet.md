# Système Complet - Architecture et Implémentation

## Architecture détaillée

Le système IoT est composé de 4 couches principales :

1. **Couche Terrain** : micro:bit sur véhicules (émetteurs)
2. **Couche Radio** : Communication 2.4 GHz avec protocole CPE
3. **Couche Passerelle** : micro:bit centrale + Gateway Python
4. **Couche Backend** : RabbitMQ + API

## 1. Applications micro:bit

### 1.1 Application Terrain (iot-terrain-microbit)

**Rôle** : Équipe les véhicules, émet position/statut, reçoit affectations.

**Architecture** :
```
[Système externe/GPS] 
        ↓ UART CSV
   [main.cpp]
        ├─ Parser CSV
        ├─ SDMIS_RADIO (émission)
        └─ SDMIS_RADIO (réception)
        ↓ Radio CPE
   [Passerelle]
```

**Fonctionnalités** :
- **Émission** : position, statut véhicule, statut incident
- **Réception** : affectations d'incidents
- **Feedback visuel** : LEDs (colonnes 0-4) indiquent succès/échec ACK

**Configuration** :
```cpp
RADIO_GROUP = 9
RADIO_POWER = 7
UART_BAUD = 115200
```

**Format UART émis** (vers système externe) :
```csv
vehicle_affectation,0,AB123CD,48.858859,2.294481,1705140000
```

**Format UART reçu** (du système externe) :
```csv
vehicle_position,1,AB123CD,48.858859,2.294481,1705140000
vehicle_status,2,AB123CD,0,0,1705140000
incident_status,3,AB123CD,0,0,1705140000
```

**Indicateurs LED** :
- LED 0 : Erreur (pas d'ACK)
- LED 1 : Position envoyée avec ACK
- LED 2 : Statut véhicule envoyé avec ACK
- LED 3 : Statut incident envoyé avec ACK
- LED 4 : Affectation reçue (pixel 4,4 toujours allumé = en ligne)

### 1.2 Passerelle Centrale (rf-central-gateway/firmware)

**Rôle** : Reçoit toutes les trames radio, transmet à la gateway Python.

**Architecture** :
```
[Véhicules terrain]
        ↓ Radio CPE
   [main.cpp]
        ├─ SDMIS_RADIO (réception)
        ├─ SDMIS_RADIO (émission)
        └─ Parser CSV
        ↓ UART CSV
   [Gateway Python]
```

**Fonctionnalités** :
- **Réception** : positions, statuts véhicules, statuts incidents
- **Émission** : affectations d'incidents
- **Conversion** : Protocole CPE ↔ CSV

**Configuration** :
```cpp
RADIO_GROUP = 9
RADIO_POWER = 7
UART_BAUD = 115200
```

**Format UART émis** (vers gateway Python) :
```csv
vehicle_position,1,AB123CD,48.858859,2.294481,1705140000
vehicle_status,2,AB123CD,0,0,1705140000
incident_status,3,AB123CD,0,0,1705140000
```

**Format UART reçu** (de gateway Python) :
```csv
vehicle_affectation,0,AB123CD,48.858859,2.294481,1705140000
```

**Indicateurs LED** :
- LED 1 : Position reçue
- LED 2 : Statut véhicule reçu
- LED 3 : Statut incident reçu
- LED 4 : Affectation émise avec ACK
- Pixel 4,4 : Toujours allumé = en ligne

### 1.3 Gestion UART commune

Les deux applications partagent la même logique UART :
- **Buffer circulaire** : 192 octets pour réception
- **Drain callback** : Vidange UART pendant les attentes radio (évite overflow)
- **Parsing robuste** : Gère les lignes trop longues sans crash
- **Format strict** : Validation des CSV (6 champs obligatoires)

## 2. Gateway Python (rf-central-gateway/gateway)

### 2.1 Architecture

```
┌─────────────────────────────────────┐
│         SerialReader                │
│  Lecture UART non-bloquante         │
│  Buffer interne + extraction lignes │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│         FrameParser                 │
│  Parsing CSV → Modèles Pydantic     │
│  Validation: immat, coords, temps   │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│         Gateway (main)              │
│  Boucle UART → RabbitMQ             │
│  Thread RabbitMQ → UART             │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│       RabbitMQClient                │
│  Publisher: télémétrie              │
│  Consumer: affectations             │
└─────────────────────────────────────┘
```

### 2.2 Composants

#### SerialReader (`io/serial_reader.py`)
- **Mode** : Non-bloquant (timeout=0)
- **Buffer interne** : Accumule les octets reçus
- **Extraction** : Yield les lignes complètes (`\n`)
- **Sécurité** : Limite taille ligne (256 octets)

#### FrameParser (`core/parser.py`)
- **Validation** : Pydantic avec regex, ranges, timestamps
- **Types supportés** :
  - `PositionFrame` → `VehiclePosition`
  - `StatusFrame` → `VehicleStatus`
  - `IncidentStatusFrame` → `IncidentStatus`
- **Formatage** : Immatriculation standardisée (XX000XX)

#### RabbitMQClient (`io/rabbitmq_client.py`)
- **Publishing** : 3 queues (positions, statuts véhicules, statuts incidents)
- **Consuming** : Queue affectations (thread séparé)
- **Robustesse** : Reconnexion automatique, logging des erreurs
- **Message format** : JSON structuré

#### Gateway (`core/gateway.py`)
- **Thread principal** : UART → RabbitMQ (polling 1ms)
- **Thread secondaire** : RabbitMQ → UART (callback)
- **Bidirectionnel** : Flux simultanés dans les 2 sens
- **Lifecycle** : Connexion, boucle, cleanup

### 2.3 Modèles de données

```python
# Modèles UART (CSV)
PositionFrame(event, status, immat, lat, lon, timestamp)
StatusFrame(event, status, immat, _, _, timestamp)
IncidentStatusFrame(event, status, immat, _, _, timestamp)

# Modèles domaine (validation enrichie)
VehiclePosition(immat, latitude, longitude, timestamp)
VehicleStatus(immat, status, status_label, timestamp)
IncidentStatus(immat, status, timestamp)

# Modèles RabbitMQ (affectations)
VehicleAssignment(immat, lat, lon, incident_id, timestamp)
```

### 2.4 Configuration

Variables d'environnement (`.env`) :
```bash
SERIAL_PORT=/dev/ttyACM0
SERIAL_BAUD=115200
RABBITMQ_DSN=amqp://user:pass@host:5672/
RABBITMQ_QUEUE_TELEMETRY=vehicle.telemetry
RABBITMQ_QUEUE_ASSIGNMENTS=vehicle.assignments
RABBITMQ_QUEUE_INCIDENT_TELEMETRY=incident.telemetry
LOG_LEVEL=INFO
```

### 2.5 Installation et démarrage

```bash
# Installation
cd rf-central-gateway/gateway
pip install -r requirements.txt

# Configuration
cp .env.example .env
# Éditer .env avec vos paramètres

# Démarrage
python -m rf_gateway

# Ou avec Docker
docker compose up
```

## 3. Flux de données détaillés

### 3.1 Position véhicule (Terrain → API)

1. **GPS externe** génère CSV avec coordonnées
2. **micro:bit terrain** :
   - Parse CSV
   - Encode en trame CPE (type `VEH_POS`)
   - Émet par radio avec retry/ACK
3. **micro:bit centrale** :
   - Reçoit trame radio
   - Décode protocole CPE
   - Émet CSV sur UART
4. **Gateway Python** :
   - Lit ligne UART
   - Parse en `PositionFrame`
   - Valide (immat, coordonnées, timestamp)
   - Crée `VehiclePosition`
   - Publie JSON sur RabbitMQ (`vehicle.telemetry`)
5. **API** :
   - Consomme message RabbitMQ
   - Stocke en base PostgreSQL
   - Diffuse SSE aux clients web

### 3.2 Affectation incident (API → Terrain)

1. **API** :
   - Reçoit ordre d'affectation
   - Publie JSON sur RabbitMQ (`vehicle.assignments`)
2. **Gateway Python** :
   - Thread consumer reçoit message
   - Crée `VehicleAssignment`
   - Génère CSV
   - Émet sur UART
3. **micro:bit centrale** :
   - Lit ligne UART
   - Parse CSV
   - Encode en trame CPE (type `INCIDENT_AFFECT`)
   - Émet par radio avec retry/ACK
4. **micro:bit terrain** :
   - Reçoit trame radio
   - Décode protocole CPE
   - Émet CSV sur UART
5. **Système externe véhicule** :
   - Affiche l'affectation

## 4. Sécurité et fiabilité

### 4.1 Couche radio
- **Chiffrement** : AES-128-CTR (confidentialité)
- **CRC double** : CRC8 header + CRC16 payload (intégrité)
- **ACK/Retry** : 3 tentatives avec timeout 200ms
- **Anti-duplication** : Filtrage (seq, nonce)
- **Backoff aléatoire** : 10-40ms entre retries

### 4.2 Couche UART
- **Validation stricte** : Format CSV, taille champs
- **Buffer overflow** : Protection via drain callback
- **Timestamps** : Fenêtre de 10 secondes max
- **Coordonnées** : Validation ranges GPS

### 4.3 Couche Gateway
- **Non-bloquant** : Polling 1ms, pas de perte de données
- **Validation Pydantic** : Regex, types, ranges
- **Error handling** : Logs détaillés, pas de crash
- **Reconnexion** : Automatique sur perte RabbitMQ

## 5. Performances

### 5.1 Débits
- **Radio** : ~30 trames/seconde théorique (trame 30 octets)
- **UART** : 115200 bauds (~11 KB/s)
- **Pratique** : ~5-10 positions/seconde par véhicule (avec ACK)

### 5.2 Latences
- **Radio** : 5-50ms (selon retry)
- **UART** : <1ms
- **Gateway Python** : <5ms (parsing + RabbitMQ)
- **Total terrain→API** : 10-60ms

### 5.3 Portée radio
- **Théorique** : 100m en intérieur, 300m en extérieur
- **Pratique** : 50-150m selon environnement
- **Puissance** : Niveau 7/7 (maximum)

## 6. Debugging et monitoring

### 6.1 LEDs micro:bit
Indicateurs visuels en temps réel (voir sections 1.1 et 1.2).

### 6.2 Logs Gateway Python
```bash
# Niveau INFO (par défaut)
INFO - UART → RabbitMQ loop
INFO - Parsed frame: PositionFrame(...)
INFO - Published position for AB123CD

# Niveau DEBUG
DEBUG - Read UART line: vehicle_position,1,...
DEBUG - RabbitMQ publish success
```

### 6.3 Commandes utiles

```bash
# Tester connexion UART
ls -l /dev/ttyACM*
screen /dev/ttyACM0 115200

# Vérifier RabbitMQ
docker exec -it rabbitmq rabbitmqctl list_queues

# Monitorer trafic
python -m rf_gateway --log-level DEBUG
```

## 7. Évolutions futures

### 7.1 Améliorations possibles
- **Multi-hop** : Relais entre micro:bit pour étendre portée
- **Compression** : Réduction taille trames (delta encoding)
- **Mesh network** : Réseau maillé auto-réparant
- **OTA updates** : Mise à jour firmware à distance

### 7.2 Scalabilité
- **Actuel** : ~10 véhicules simultanés
- **Limite radio** : ~30 véhicules (saturation canal)
- **Solution** : Multiplexage temporel ou canaux multiples
