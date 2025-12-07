# Simulation Java — Véhicules

Simule les véhicules terrain : positions GPS, vitesses, changements d'état et messages opérationnels.

## Rôle
- Génération de trajets (point A → incident → caserne) avec positions périodiques.
- Simulation des transitions d'état (disponible, en route, sur intervention, terminé).
- Envoi des messages terrain (arrivée, fin d'intervention, demande de renfort) vers le moteur/ API.

## Build & exécution
```bash
cd simulation-java-vehicles
mvn clean package
# Exécution (main à préciser) : java -jar target/*.jar
```

## Points à préciser
- Protocoles de sortie : RabbitMQ (queues), API HTTP ou fichiers replay.
- Modèle de véhicule utilisé (capacités, vitesse moyenne, immobilisation).
- Synchronisation avec la simulation incidents pour des scénarios cohérents.
