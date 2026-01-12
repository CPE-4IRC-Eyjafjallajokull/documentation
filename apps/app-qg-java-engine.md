# App QG Java Engine - Moteur de décision d'affectation

## Vue d'ensemble

Le **QG Java Engine** est un service Java qui calcule automatiquement les meilleures affectations de véhicules aux incidents en temps réel. Il écoute les événements RabbitMQ, applique un algorithme de scoring, et propose des affectations optimisées à l'API.

## Architecture

```
RabbitMQ (sdmis_engine)
        ↓
   [Event Listener]
        ↓
   [IncidentHandler]
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
- **Écoute RabbitMQ** : Réagit aux événements `new_incident`
- **API SDMIS** : Récupère les données d'incidents, véhicules, planifications
- **PostgreSQL** : Connectivité et health check
- **Keycloak** : Authentification JWT pour l'API SDMIS

## Composants principaux

### DecisionEngine
Interface du moteur de décision :
```java
DecisionResult proposeAssignments(UUID incidentId)
```

**Implémentation** : `VehicleAssignmentDecisionEngine`
- Récupère la situation d'incident et le planning requis
- Liste les véhicules disponibles
- Calcule les candidats par type de véhicule
- Applique la stratégie de scoring
- Génère les propositions d'affectation

### VehicleScoringStrategy
Stratégie de scoring des véhicules candidats.

**Implémentation** : `DistanceEnergyScoringStrategy`
- **Score** = (distance × 0.4 + temps × 0.4 + énergie × 0.2) / somme_poids
- Favorise les véhicules proches, rapides et avec bonne énergie
- Gère les métriques manquantes (distance/temps indisponibles)

### DecisionDataSource
Source de données pour les décisions.

**Implémentation** : `SdmisDecisionDataSource`
- Requêtes vers l'API SDMIS (REST)
- Récupération : incidents, véhicules, planifications, phases
- Authentification Keycloak
- Cache et retry

### IncidentHandler
Handler d'événements pour les nouveaux incidents :
1. Reçoit événement `new_incident` depuis RabbitMQ
2. Extrait l'ID incident du payload
3. Appelle le moteur de décision
4. Publie les propositions sur queue `sdmis_api`

## Flux de décision

### 1. Réception événement
```json
{
  "event": "new_incident",
  "payload": {
    "incident_id": "uuid-123",
    ...
  }
}
```

### 2. Récupération données
- **Incident** : Position GPS, phases actives
- **Planning** : Requirements par phase et type de véhicule
- **Véhicules** : Liste complète avec positions, types, énergie

### 3. Calcul scoring
Pour chaque véhicule candidat :
1. Calcul distance euclidienne incident ↔ véhicule
2. Estimation temps de trajet (si disponible)
3. Récupération niveau d'énergie
4. Application de la formule de scoring
5. Classement par score décroissant

### 4. Sélection optimale
- Parcours des groupes de requirements par priorité
- Allocation des meilleurs véhicules disponibles
- Respect des quantités requises par type
- Évite les doublons (véhicule déjà alloué)

### 5. Génération proposition
```json
{
  "event": "vehicle_assignment_proposal",
  "payload": {
    "proposal_id": "uuid-456",
    "incident_id": "uuid-123",
    "generated_at": "2026-01-12T10:00:00Z",
    "proposals": [
      {
        "vehicle_id": "uuid-789",
        "score": 0.85,
        "rationale": "distance_km=2.5, time_min=3.2, energy=0.9"
      }
    ],
    "missing_by_vehicle_type": {
      "uuid-type-1": 2
    }
  }
}
```

## Configuration

Variables d'environnement (`.env`) :

```bash
# RabbitMQ
RABBITMQ_URI=amqp://user:pass@host:5672/
RABBITMQ_QUEUE_DURABLE=true

# PostgreSQL
POSTGRES_URL=jdbc:postgresql://host:5432/sdmis
POSTGRES_USER=user
POSTGRES_PASSWORD=password

# Keycloak
KEYCLOAK_REALM=sdmis
KEYCLOAK_AUTH_SERVER_URL=http://keycloak:8080
KEYCLOAK_CLIENT_ID=sdmis-api
KEYCLOAK_CLIENT_SECRET=secret

# API SDMIS
SDMIS_API_BASE_URL=http://api:8000

# Critères de décision (optionnel)
DECISION_MIN_ENERGY_LEVEL=0.2
DECISION_MAX_DISTANCE_KM=50.0
DECISION_MAX_TRAVEL_TIME_MIN=30.0
```

## Queues RabbitMQ

| Queue | Direction | Description |
|-------|-----------|-------------|
| `sdmis_engine` | Subscription | Reçoit les événements d'incidents |
| `sdmis_api` | Publication | Envoie les propositions d'affectation |

## Événements

### Consommés
- **`new_incident`** : Nouvel incident créé, déclenche le moteur

### Produits
- **`vehicle_assignment_proposal`** : Propositions d'affectation calculées

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
java -jar target/app-qg-java-engine.jar
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
├── App.java                    # Point d'entrée
├── auth/                       # Authentification Keycloak
├── config/                     # Configuration environnement
├── database/                   # Clients PostgreSQL
├── decision/                   # Moteur de décision
│   ├── api/                    # Interfaces
│   ├── impl/                   # Implémentations
│   └── model/                  # Modèles de données
├── events/                     # Gestion événements
├── handlers/                   # Handlers d'événements
│   └── IncidentHandler.java   # Handler incidents
├── messaging/                  # Client RabbitMQ
├── sdmis/                      # Client API SDMIS
│   └── dto/                    # DTOs API
└── service/                    # Services (health check)
```

## Tests

```bash
# Run tests
mvn test

# Tests avec couverture
mvn verify
```

## Monitoring

### Logs
```
INFO - Starting QG Java Engine...
INFO - RabbitMQ Config: uri=amqp://..., durableQueue=true
INFO - Connectivity check passed
INFO - Engine is running. Listening on queues [sdmis_engine]
INFO - Processing new incident message: {...}
INFO - Sent decision proposal to sdmis_api for incident uuid-123
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

- **Routing réel** : Intégration OSRM/GraphHopper pour distances/temps précis
- **Scoring avancé** : Prise en compte trafic, météo, historique performance
- **Machine Learning** : Apprentissage des patterns d'affectation optimaux
- **Multi-objectifs** : Optimisation équité vs efficacité
