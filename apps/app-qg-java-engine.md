# Moteur décisionnel Java

Service Java centralisant l'état incidents/interventions/ressources et appliquant les règles métier de proposition d'affectation.

## Rôle
- Maintien de la source de vérité : incidents, interventions, véhicules (états + positions).
- Application des règles de filtrage/affectation et génération de propositions automatiques.
- Échanges bidirectionnels :
  - Consommation des flux IoT/simulation via RabbitMQ ou API.
  - Publication des mises à jour vers l'API QG / front / base de données.

## Build & exécution (base)
```bash
cd app-qg-java-engine
mvn clean package
java -jar target/qg_engine-1.0-SNAPSHOT.jar
```

## Intégrations attendues
- Messaging : RabbitMQ (files à documenter) pour incidents, positions GPS, états interventions.
- Persistance : PostgreSQL (schéma à préciser) et éventuellement cache.
- API : endpoints internes pour pilotage et supervision (health, stats décisions, etc.).

## À documenter
- Diagrammes de flux entre API FastAPI, moteur et simulation.
- Règles métiers (priorisation, distances, statuts véhicules) et paramètres ajustables.
- Contrats de messages (JSON/Protobuf) et stratégies de relecture/tolérance aux pannes.
