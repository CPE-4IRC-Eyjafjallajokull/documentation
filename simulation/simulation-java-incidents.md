# Simulation Java — Incidents

Génère des incidents (type, localisation, gravité, évolution) pour alimenter le QG en mode test.

## Rôle
- Création périodique ou scénarisée d'incidents.
- Injection dans l'API FastAPI ou directement dans RabbitMQ selon le mode choisi.
- Permettre des campagnes de tests reproductibles (scriptées).

## Build & exécution
```bash
cd simulation-java-incidents
mvn clean package
# Exécution (main à préciser) : java -jar target/*.jar
```

## Points à préciser
- Format des messages envoyés (queues RabbitMQ / payload JSON pour l'API).
- Paramétrage des scénarios : fréquence, zones géographiques, types d'incidents.
- Commandes CLI disponibles (seed, mode continu vs batch).
