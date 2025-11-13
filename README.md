# Système Intégré de Gestion et Suivi des Incidents — Documentation

Ce dépôt centralise l’ensemble de la documentation liée au projet : conception, architecture, protocoles IoT, modèles de données, décisions techniques et guides d’utilisation.
Il constitue la référence unique pour comprendre, maintenir et faire évoluer le système dans toutes ses composantes.

---

## 1. Présentation générale

Le système vise à fournir une plateforme complète permettant :

* la supervision en temps quasi réel des incidents, véhicules et interventions ;
* la gestion opérationnelle des ressources du SDMIS ;
* l’orchestration automatique (mais supervisée) des propositions d’affectation ;
* la réception et l’exploitation de messages IoT transmis depuis les véhicules ;
* la capacité à fonctionner avec une simulation complète pour les tests.

L’architecture repose sur trois blocs principaux :

1. **App QG** : interface opérateur, API REST et moteur décisionnel Java.
2. **IoT Terrain** : micro:bit embarqué, passerelle RF et protocole UART.
3. **Simulation Java** : génération d’incidents, véhicules et événements externes.

---

## 2. Architecture Applicative

Le schéma associé décrit les interactions entre les différents composants :

### Composants internes à l’App QG

* **Front — UI QG (React)**
  Interface web opérateur : carte, incidents, interventions, ressources.
  Consommation API REST, authentification via Keycloak, mise à jour continue via SSE.

* **Backend QG — API REST (FastAPI)**
  Point d’entrée unique des événements externes.
  Normalisation/validation des données, agrégation, exposition des endpoints front.
  Communication interne vers le moteur Java via RabbitMQ.

* **Core Engine — Java**
  Centralisation des états incidents/interventions/ressources.
  Application des règles métier et filtrages.
  Génération des propositions automatiques d’intervention.
  Communication bidirectionnelle avec : API FastAPI, simulation, base de données, IoT.

* **RabbitMQ (pub/sub)**
  Canal interne de routage des événements : incidents, positions GPS, demandes d'affectation, états d'interventions.

* **PostgreSQL**
  Stockage durable des incidents, véhicules, interventions, positions, journaux d’événements.

* **Keycloak**
  Authentification et gestion des rôles pour l’interface opérateur et l’API.

---

## 3. Sous-système IoT Terrain

* **App Terrain (micro:bit)**
  Transmission RF des messages suivants :
  localisation GPS, informations incident, renfort, fin d’intervention, disponibilité.

* **Centrale RF**
  Réception RF → transformation → émission UART vers le moteur Java.

* **Protocole UART**
  Trames normalisées, robustes et sécurisées.

---

## 4. Simulation Java

La simulation agit comme source d’événements externes :

* génération d’incidents (type, localisation, gravité, évolution) ;
* simulation des véhicules (trajets, états, positions) ;
* injection cohérente dans l’API QG ou dans RabbitMQ selon le mode choisi.

Elle permet de tester le système sans matériel réel.

---

## 5. Organisation des dépôts (repos)

Chaque composant applicatif possède son propre dépôt :

* `documentation` (celui-ci)
* `app-qg-front` (React)
* `app-qg-api` (FastAPI)
* `app-qg-java-engine` (moteur décisionnel Java)
* `simulation-java`
* `iot-terrain-microbit`
* `rf-central-gateway`
* `infrastructure-devops` (CI/CD, conteneurs, Keycloak, migrations BDD)

---

## 6. Contenu de ce dépôt

Ce dépôt rassemble :

### 6.1. Spécifications Fonctionnelles

Cas d’usage, parcours opérateur, scénarios incidents/interventions, exigences liées au QG et au terrain.

### 6.2. Spécifications Techniques Générales

Décomposition des services, API internes/externes, diagrammes d’architecture, choix techniques motivés.

### 6.3. Documentation IoT

Protocole RF, structure des trames UART, contraintes de sécurité, gestion des erreurs.

### 6.4. Modèles de données

Schéma SQL complet (PostgreSQL), contraintes, index, historiques.

### 6.5. API REST

Endpoints, schémas JSON, statuts, exemples de requêtes/réponses.

### 6.6. Guides pratiques

Installation, exécution locale, intégration CI/CD, déploiement, configuration Keycloak.

### 6.7. Annexes

Diagrammes, décisions techniques, glossaire, conventions de nommage.

---

## 7. Objectif du dépôt

Garantir une compréhension claire, durable et extensible du système complet, tout en offrant un support unique pour :

* la soutenance,
* les jalons d’architecture,
* la montée en compétence des nouveaux contributeurs,
* la maintenance technique.

Ce dépôt doit toujours refléter l’état réel du système et ses évolutions.
