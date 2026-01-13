# Simulation Java

Deux simulateurs pour tester le systeme sans materiel terrain :
- **Incidents** : cree des incidents et phases automatiquement.
- **Vehicules** : simule les trajets, statuts et la telemetrie.

---

## Quand l'utiliser

- Demonstations et tests bout en bout.
- Tests de charge (flux incidents / vehicules).
- Environnement sans micro:bit.

---

## Demarrage rapide

### Incidents

```bash
cd simulation-java-incidents
mvn clean package
java -jar target/simulateur_java_incidents-1.0-SNAPSHOT.jar
```

### Vehicules

```bash
cd simulation-java-vehicles
mvn clean package
java -jar target/simulateur_java_vehicles-1.0-SNAPSHOT.jar
```

---

## Configuration (exemples)

Les deux simulateurs utilisent un fichier `.env` a la racine du projet.
Variables courantes :

- `SDMIS_API_BASE_URL` (API QG)
- `KEYCLOAK_ISSUER`, `KEYCLOAK_CLIENT_ID`, `KEYCLOAK_CLIENT_SECRET`
- `UART_PORT` et `UART_BAUD` (vehicules)

---

## Documentation detaillee

- Incidents : [incidents.md](incidents.md)
- Vehicules : [vehicles.md](vehicles.md)
