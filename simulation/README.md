# Simulation Java

Modules de simulation pour alimenter le système sans matériel terrain. Deux briques distinctes : incidents (génération d'événements) et véhicules (trajectoires + états).

## Répartition
- [Incidents](simulation-java-incidents.md) — création d'incidents, escalade et injection API/RabbitMQ.
- [Véhicules](simulation-java-vehicles.md) — trajets, positions GPS et transitions d'état simulés.

## Objectifs
- Tester bout en bout l'API et le moteur décisionnel sans équipement réel.
- Rejouer des scénarios déterministes pour les démonstrations et les tests automatiques.
- Pousser des volumes réalistes pour éprouver la persistance et la messagerie.
