# App QG Java Engine - Moteur de décision d'affectation

## Vue d'ensemble

Le **QG Java Engine** est un service Java qui calcule automatiquement les meilleures affectations de véhicules aux incidents en temps réel. Il écoute les événements RabbitMQ, applique un algorithme de scoring, et propose des affectations optimisées à l'API.

## Architecture

```
RabbitMQ (sdmis_engine)
        ↓
   [Event Listener]
        ↓
   [AssignmentRequestHandler]
        ↓
   [Decision Engine]
        ├─ Data Source (API SDMIS)
        ├─ Scoring Strategy
        └─ Criteria
        ↓
   [Proposals]
        ↓
RabbitMQ (sdmis_api)
        ↓
   API Backend
```

## Fonctionnalités

### 1. Moteur de décision
- **Calcul automatique** : Propose les véhicules optimaux pour chaque incident
- **Multi-critères** : Distance, temps de trajet, niveau d'énergie
- **Scoring pondéré** : 40% distance + 40% temps + 20% énergie
- **Gestion des contraintes** : Respect des requirements par type de véhicule

### 2. Intégration système
- **Écoute RabbitMQ** : Réagit aux événements `assignment_request`
- **API SDMIS** : Récupère les données d'incidents, véhicules, estimations de trajet
- **PostgreSQL** : Connectivité et health check
- **Keycloak** : Authentification JWT pour l'API SDMIS

## Composants principaux

### DecisionEngine
Interface du moteur de décision :
```java
DecisionResult proposeAssignments(AssignmentRequest request)
```

**Implémentation** : `VehicleAssignmentDecisionEngine`
- Agrège les besoins par phase d'incident
- Récupère la position de l'incident et les véhicules disponibles
- Filtre les véhicules (assignés, en proposition, hors critères)
- Calcule distance/temps via API de routing (`/geo/route`)
- Applique la stratégie de scoring
- Génère les propositions avec rang et géométrie de route

### VehicleScoringStrategy
Stratégie de scoring des véhicules candidats.

**Implémentation** : `DistanceEnergyScoringStrategy`
- **Score** = (distance × 0.4 + temps × 0.4 + énergie × 0.2) / somme_poids
- Favorise les véhicules proches, rapides et avec bonne énergie
- Gère les métriques manquantes dynamiquement

### DecisionDataSource
Source de données pour les décisions.

**Implémentation** : `SdmisDecisionDataSource`
- `GET /qg/incidents/{id}/situation` : Position de l'incident
- `GET /qg/vehicles` : Liste des véhicules avec positions
- `POST /geo/route` : Calcul de distance, temps et géométrie de route
- Authentification Keycloak automatique

### AssignmentRequestHandler
Handler d'événements pour les demandes d'affectation :
1. Reçoit événement `assignment_request` depuis RabbitMQ
2. Parse le payload contenant `incident_id` et `vehicles_needed`
3. Appelle le moteur de décision avec les besoins explicites
4. Publie les propositions sur queue `sdmis_api`

## Flux de décision

### 1. Réception événement
```json
{
  "event": "assignment_request",
  "payload": {
    "incident_id": "uuid-123",
    "vehicles_needed": [
      {
        "incident_phase_id": "uuid-phase",
        "vehicle_type_id": "uuid-type",
        "quantity": 2
      }
    ]
  }
}
```

### 2. Récupération données
- **Besoins** : Liste des véhicules requis par phase (`vehicles_needed`)
- **Véhicules** : Liste complète avec positions, types, énergie (via API SDMIS)

### 3. Calcul scoring
Pour chaque véhicule candidat :
1. Résolution position (GPS temps réel ou base d'affectation)
2. Calcul distance/temps via API de routing (fallback: formule Haversine)
3. Récupération niveau d'énergie
4. Application de la formule de scoring
5. Classement par score décroissant, puis temps, puis distance

### 4. Filtrage des véhicules
Un véhicule est **exclu** si :
- Déjà assigné à un incident (`activeAssignment != null`)
- Référencé dans une proposition en attente (`referencedInPendingProposal`)
- Énergie insuffisante (< `DECISION_MIN_ENERGY_LEVEL`)
- Distance trop grande (> `DECISION_MAX_DISTANCE_KM`)

### 5. Sélection optimale
- Parcours des groupes de requirements par priorité
- Allocation des meilleurs véhicules disponibles
- Respect des quantités requises par type
- Évite les doublons (véhicule déjà alloué)

### 6. Génération proposition
```json
{
  "event": "assignment_proposal",
  "payload": {
    "proposal_id": "uuid-456",
    "incident_id": "uuid-123",
    "generated_at": "2026-01-12T10:00:00Z",
    "vehicles_to_send": [
      {
        "incident_phase_id": "uuid-phase",
        "vehicle_id": "uuid-789",
        "distance_km": 2.5,
        "estimated_time_min": 3.2,
        "route_geometry": { "type": "LineString", "coordinates": [...] },
        "energy_level": 0.9,
        "score": 0.85,
        "rank": 1
      }
    ],
    "missing": [
      {
        "incident_phase_id": "uuid-phase",
        "vehicle_type_id": "uuid-type",
        "missing_quantity": 2
      }
    ]
  }
}
```

## Configuration

Variables d'environnement (`.env`) :

```bash
# Logging
LOG_LEVEL=INFO

# PostgreSQL
POSTGRES_URL=jdbc:postgresql://host:5432/sdmis
POSTGRES_USER=user
POSTGRES_PASSWORD=password
POSTGRES_POOL_SIZE=5
POSTGRES_CONNECTION_TIMEOUT_MS=30000

# RabbitMQ
RABBITMQ_URI=amqp://user:pass@host:5672/vhost
RABBITMQ_QUEUE_DURABLE=true

# Keycloak
KEYCLOAK_ISSUER=http://keycloak:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=sdmis-api
KEYCLOAK_CLIENT_SECRET=secret
KEYCLOAK_TIMEOUT_MS=3000
KEYCLOAK_TOKEN_EXPIRY_SKEW_SECONDS=30

# API SDMIS
SDMIS_API_BASE_URL=http://api:8000
SDMIS_API_TIMEOUT_MS=5000

# Critères de décision
DECISION_MAX_DISTANCE_KM=15
DECISION_MIN_ENERGY_LEVEL=0.3
```

## Queues RabbitMQ

| Queue | Direction | Description |
|-------|-----------|-------------|
| `sdmis_engine` | Subscription | Reçoit les événements d'incidents |
| `sdmis_api` | Publication | Envoie les propositions d'affectation |

## Événements

### Consommés
- **`assignment_request`** : Demande d'affectation de véhicules pour un incident

### Produits
- **`assignment_proposal`** : Propositions d'affectation calculées avec scores et routes

## Démarrage

### Prérequis
- Java 21+
- Maven 3.9+
- RabbitMQ accessible
- PostgreSQL accessible
- API SDMIS accessible

### Installation
```bash
# Clone et build
cd app-qg-java-engine
mvn clean package

# Configuration
cp .env.example .env
# Éditer .env avec vos paramètres

# Exécution
java -jar target/app-qg-java-engine-1.0-SNAPSHOT-jar-with-dependencies.jar
```

### Docker
```bash
# Build image
docker build -t app-qg-java-engine .

# Run avec docker-compose
docker compose up
```

## Structure du code

```
src/main/java/cpe/qg/engine/
├── App.java                        # Point d'entrée
├── auth/                           # Authentification Keycloak
├── config/                         # Configuration environnement
├── database/                       # Clients PostgreSQL
├── decision/                       # Moteur de décision
│   ├── api/                        # Interfaces
│   ├── impl/                       # Implémentations
│   └── model/                      # Modèles de données
├── events/                         # Gestion événements
├── handlers/                       # Handlers d'événements
│   └── AssignmentRequestHandler.java  # Handler principal
├── logging/                        # Provider de logs
├── messaging/                      # Client RabbitMQ
├── sdmis/                          # Client API SDMIS
│   └── dto/                        # DTOs API
└── service/                        # Services (health check)
```

## Tests

```bash
# Run tests
mvn test
```

## Monitoring

### Logs
```
INFO - Starting QG Java Engine...
INFO - RabbitMQ Config: uri=amqp://..., durableQueue=true
INFO - Connectivity check passed
INFO - Engine is running. Listening on queues [sdmis_engine] for events [assignment_request]
INFO - Processing assignment request message: {...}
INFO - Assignment proposals generated for incident uuid-123: 3 vehicle(s), 1 missing entries
INFO - Sent assignment proposal to sdmis_api for incident uuid-123
```

### Health Check
Au démarrage, le service vérifie :
- ✅ Connexion PostgreSQL
- ✅ Connexion RabbitMQ
- ✅ Déclaration des queues

## Algorithme de scoring

### Formule
```
score = (distance_score × 0.4 + time_score × 0.4 + energy_score × 0.2) / poids_total
```

Où :
- **distance_score** = 1 / (1 + distance_km)
- **time_score** = 1 / (1 + time_min)
- **energy_score** = clamp(energy_level, 0, 1)

### Exemple
Véhicule à 5 km, 8 min, énergie 85% :
- distance_score = 1 / (1 + 5) = 0.167
- time_score = 1 / (1 + 8) = 0.111
- energy_score = 0.85
- **score final** = (0.167×0.4 + 0.111×0.4 + 0.85×0.2) / 1.0 = **0.281**

## Évolutions futures

- **Scoring avancé** : Prise en compte trafic temps réel, météo, historique performance
- **Machine Learning** : Apprentissage des patterns d'affectation optimaux
- **Multi-objectifs** : Optimisation équité vs efficacité
- **Cache de routing** : Mise en cache des estimations de trajet fréquentes
