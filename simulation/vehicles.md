# Simulation Java — Véhicules

Simulateur de véhicules pour le SDMIS. Il simule le déplacement des véhicules de secours, leurs changements d'état et communique avec le système IoT terrain via UART.

---

## 🎯 Objectif

Le simulateur permet de :
- **Simuler les déplacements** des véhicules de pompiers en temps réel
- **Gérer les affectations** aux incidents (réception via UART)
- **Calculer les trajets** via l'API de routage SDMIS
- **Communiquer avec les Micro:bit** terrain via liaison série (UART)

---

## 🏗️ Architecture

Le projet est écrit en **Java 21** et utilise la bibliothèque **jSerialComm** pour la communication série.

```
src/main/java/cpe/simulator/vehicles/
├── SimulatorApplication.java    # Point d'entrée
├── SimulatorFactory.java        # Création des composants
├── api/                         # Interfaces (contrats)
├── config/                      # Configuration
├── core/                        # Logique métier
│   ├── VehicleSimulator.java    # Boucle principale
│   ├── Fleet.java               # Gestion de la flotte
│   ├── MovementModel.java       # Calcul des déplacements
│   └── RoutePlan.java           # Suivi des trajets
├── domain/                      # Objets métier (GeoPoint, VehicleStatus)
├── infrastructure/              # Implémentations
│   ├── http/                    # Client HTTP + Keycloak
│   ├── sdmis/                   # Services API SDMIS
│   └── uart/                    # Communication série
└── uart/                        # Parsing des messages UART
```

---

## ⚙️ Fonctionnement

### Cycle de vie d'un véhicule

```
┌─────────────┐     Affectation      ┌─────────┐     Arrivée      ┌──────────────────┐
│ DISPONIBLE  │ ──────────────────▶  │ ENGAGE  │ ──────────────▶  │ SUR_INTERVENTION │
└─────────────┘                      └─────────┘                  └──────────────────┘
       ▲                                                                    │
       │                           ┌─────────┐                              │
       └────── Arrivée caserne ─── │ RETOUR  │ ◀── Fin intervention ───────┘
                                   └─────────┘
```

### Boucle de simulation

1. **Chargement des véhicules** depuis l'API SDMIS
2. **Ouverture du port UART** pour communication avec la gateway RF
3. **Boucle principale** (tick configurable ~200ms) :
   - Avancement de chaque véhicule sur son trajet
   - Détection des arrivées (intervention / caserne)
   - Envoi des positions via UART
   - Écoute des affectations entrantes

---

## 📊 États des véhicules

| Code | État | Description |
|------|------|-------------|
| 0 | `DISPONIBLE` | À la caserne, prêt à intervenir |
| 1 | `ENGAGE` | En route vers l'incident |
| 2 | `SUR_INTERVENTION` | Arrivé sur les lieux |
| 3 | `TRANSPORT` | Transport de victime |
| 4 | `RETOUR` | Retour à la caserne |
| 5 | `INDISPONIBLE` | Non disponible temporairement |
| 6 | `HORS_SERVICE` | Véhicule en maintenance |

---

## 📡 Communication UART

### Format des messages

Les messages sont échangés en **CSV** sur liaison série :

```
event,status,immatriculation,latitude,longitude,timestamp
```

### Types d'événements

| Événement | Direction | Description |
|-----------|-----------|-------------|
| `vehicle_position` | Sortie | Position GPS du véhicule |
| `vehicle_status` | Sortie | Changement d'état |
| `vehicle_affectation` | Entrée | Nouvelle affectation reçue |
| `incident_status` | Sortie | Fin d'intervention |

### Intervalles d'envoi

| Situation | Intervalle |
|-----------|------------|
| Véhicule à la caserne | 60 secondes |
| Véhicule en mouvement | 500 ms |
| Changement de statut | 5 secondes |

---

## 🔧 Configuration

### Variables d'environnement requises

| Variable | Description | Exemple |
|----------|-------------|---------|
| `KEYCLOAK_CLIENT_ID` | ID client Keycloak | `simulation` |
| `KEYCLOAK_CLIENT_SECRET` | Secret client | `your-secret` |

### Variables optionnelles

| Variable | Description | Défaut |
|----------|-------------|--------|
| `SDMIS_API_BASE_URL` | URL de l'API | `http://localhost:3001` |
| `KEYCLOAK_ISSUER` | URL du realm Keycloak | `http://localhost:8080/realms/sdmis` |
| `UART_PORT` | Port série | `/dev/ttyACM0` |
| `UART_BAUD` | Vitesse (bauds) | `115200` |
| `SIM_TICK_MS` | Intervalle de simulation | `200` |
| `VEHICLE_SPEED_MPS` | Vitesse véhicule (m/s) | `13.89` (~50 km/h) |
| `ON_SITE_DURATION_MS` | Durée sur intervention | `60000` (1 min) |

### Exemple de fichier `.env`

```env
# Keycloak
KEYCLOAK_ISSUER=http://localhost:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=simulation
KEYCLOAK_CLIENT_SECRET=my-secret

# API
SDMIS_API_BASE_URL=http://localhost:3001

# Simulation
SIM_TICK_MS=200
VEHICLE_SPEED_MPS=13.89
ON_SITE_DURATION_MS=60000

# UART
UART_PORT=/dev/ttyACM0
UART_BAUD=115200
UART_BASE_SEND_INTERVAL_MS=60000
UART_MOVING_SEND_INTERVAL_MS=500
```

---

## 🚀 Installation et exécution

### Prérequis

- Java 21+
- Maven 3.x
- Port série accessible (ex: `/dev/ttyACM0`)

### Compilation

```bash
cd simulation-java-vehicles
mvn clean package
```

### Exécution

```bash
# Avec fichier .env à la racine
java -jar target/simulateur_java_vehicles-1.0-SNAPSHOT.jar

# Ou avec variables inline
KEYCLOAK_CLIENT_ID=client KEYCLOAK_CLIENT_SECRET=secret \
UART_PORT=/dev/ttyACM0 \
java -jar target/simulateur_java_vehicles-1.0-SNAPSHOT.jar
```

### Tests

```bash
mvn test    # Lance les tests unitaires
```

---

## 📡 API utilisées

| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/vehicles` | GET | Liste des véhicules |
| `/vehicles/{id}` | GET | Détails d'un véhicule |
| `/route` | POST | Calcul d'itinéraire |
| `/assignments` | GET | Affectations en cours |

---

## 🔌 Connexion matérielle

Le simulateur communique avec la **gateway RF centrale** via USB :

```
┌──────────────────┐      USB/UART      ┌─────────────────┐      RF 2.4GHz      ┌───────────┐
│  Simulateur Java │ ◀───────────────▶  │  Gateway RF     │ ◀────────────────▶  │ Micro:bit │
│  (ce projet)     │                    │  (Raspberry Pi) │                     │  terrain  │
└──────────────────┘                    └─────────────────┘                     └───────────┘
```

---

## 📁 Fichiers importants

| Fichier | Description |
|---------|-------------|
| `.env.example` | Exemple de configuration |
| `pom.xml` | Dépendances Maven (jSerialComm, Jackson) |

---

## 🔄 Calcul des trajets

Les trajets sont calculés via l'API de routage SDMIS :
- **Aller** : Position actuelle → Incident
- **Retour** : Incident → Caserne d'origine

Le véhicule suit les points du trajet à vitesse constante (`VEHICLE_SPEED_MPS`).
