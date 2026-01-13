# Simulation Java — Incidents

Simulateur d'incidents pour alimenter le QG en environnement de test. Il génère automatiquement des incidents (incendies, accidents, secours à personnes...) de manière continue et les envoie à l'API SDMIS.

---

## 🎯 Objectif

Le simulateur permet de :
- **Créer des incidents automatiquement** selon des probabilités réalistes
- **Tester le système QG** sans intervention manuelle
- **Reproduire des scénarios** grâce à une graine (seed) configurable

---

## 🏗️ Architecture

Le projet est écrit en **Java 17** et suit les principes **SOLID** :

```
src/main/java/cpe/simulator/
├── SimulatorApplication.java    # Point d'entrée
├── SimulatorFactory.java        # Création des composants
├── api/                         # Interfaces (contrats)
├── config/                      # Configuration
├── core/                        # Logique métier (Simulator, IncidentGenerator)
├── domain/                      # Objets métier (Incident, Location, GeoZone)
└── infrastructure/              # Implémentations concrètes (HTTP, Keycloak)
```

---

## ⚙️ Fonctionnement

1. **Chargement de la configuration** depuis les variables d'environnement (ou fichier `.env`)
2. **Connexion à Keycloak** pour obtenir un token d'authentification
3. **Récupération des types de phases** via l'API (`GET /incidents/phase/types`)
4. **Boucle de simulation** :
   - Attente d'un délai aléatoire (basé sur `INCIDENTS_PER_HOUR`)
   - Tirage d'un type d'incident selon les probabilités
   - Génération de coordonnées aléatoires dans la zone configurée
   - Enrichissement de l'adresse via géocodage inverse
   - Envoi de l'incident à l'API (`POST /qg/incidents/new`)

---

## 📊 Types d'incidents simulés

Les incidents sont tirés selon des probabilités définies dans `incident-probabilities.json` :

| Code | Description | Probabilité |
|------|-------------|-------------|
| `SAP_MALAISE` | Malaise | 6% |
| `SAP_TRAUMA` | Traumatisme | 6% |
| `FIRE_APARTMENT` | Feu d'appartement | 4% |
| `FIRE_HABITATION` | Feu d'habitation | 4% |
| `ACC_ROAD` | Accident de la route | 3.5% |
| `SAP_CARDIAC_ARREST` | Arrêt cardiaque | 3.5% |
| `FIRE_PUBLIC_SPACE` | Feu espace public | 1.5% |
| ... | + 25 autres types | ... |

> **Note** : Le code `NO_INCIDENT` (59.4%) représente les périodes sans incident.

---

## 🔧 Configuration

### Variables d'environnement requises

| Variable | Description | Exemple |
|----------|-------------|---------|
| `KEYCLOAK_CLIENT_ID` | ID client Keycloak | `sdmis_engine` |
| `KEYCLOAK_CLIENT_SECRET` | Secret client | `your-secret` |

### Variables optionnelles

| Variable | Description | Défaut |
|----------|-------------|--------|
| `SDMIS_API_BASE_URL` | URL de l'API | `http://localhost:3001` |
| `KEYCLOAK_ISSUER` | URL du realm Keycloak | `http://localhost:8080/realms/sdmis` |
| `INCIDENTS_PER_HOUR` | Nombre moyen d'incidents/heure | `12` |
| `RNG_SEED` | Graine pour reproductibilité | `42` |
| `GEO_ZONE_NAME` | Zone géographique | `lyon_villeurbanne` |

### Exemple de fichier `.env`

```env
SDMIS_API_BASE_URL=http://localhost:3001
KEYCLOAK_ISSUER=http://localhost:8080/realms/sdmis
KEYCLOAK_CLIENT_ID=sdmis_engine
KEYCLOAK_CLIENT_SECRET=my-secret
INCIDENTS_PER_HOUR=12
RNG_SEED=42
GEO_ZONE_NAME=lyon_villeurbanne
```

---

## 🚀 Installation et exécution

### Prérequis

- Java 17+
- Maven 3.x

### Compilation

```bash
cd simulation-java-incidents
mvn clean package
```

### Exécution

```bash
# Avec fichier .env à la racine
java -jar target/simulateur_java_incidents-1.0-SNAPSHOT.jar

# Ou avec variables inline
KEYCLOAK_CLIENT_ID=client KEYCLOAK_CLIENT_SECRET=secret \
java -jar target/simulateur_java_incidents-1.0-SNAPSHOT.jar
```

### Tests

```bash
mvn test    # Lance les 34 tests unitaires
```

---

## 🌍 Zone géographique

La zone par défaut est **Lyon/Villeurbanne** :

```json
{
  "lyon_villeurbanne": {
    "latitude": { "min": 45.74, "max": 45.79 },
    "longitude": { "min": 4.82, "max": 4.90 }
  }
}
```

Pour ajouter une zone, modifiez `src/main/resources/geographic-zone.json`.

---

## 📡 API utilisées

| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/incidents/phase/types` | GET | Liste des types de phases |
| `/qg/incidents/new` | POST | Création d'un incident |
| `/geocode/reverse` | GET | Géocodage inverse (coordonnées → adresse) |

---

## 📁 Fichiers importants

| Fichier | Description |
|---------|-------------|
| `incident-probabilities.json` | Probabilités de chaque type d'incident |
| `geographic-zone.json` | Définition des zones géographiques |
| `.env.example` | Exemple de configuration |

---

## 🔄 Reproductibilité

Grâce à la variable `RNG_SEED`, les simulations sont **reproductibles** :
- Même seed = même séquence d'incidents
- Utile pour les tests et le débogage

Pour varier les scénarios, changez simplement la valeur de `RNG_SEED`.
