# Apps QG

Synthèse des composants applicatifs du QG et liens vers leur documentation détaillée.

## Répartition
- [App QG API (FastAPI)](app-qg-api.md) — point d'entrée REST/SSE, orchestration des connecteurs et du moteur décisionnel.
- [App QG Front (Next.js)](app-qg-front.md) — interface opérateur sur navigateur.
- [Moteur décisionnel Java](app-qg-java-engine.md) — logique métier centralisée et échanges RabbitMQ/DB.

## À retenir
- Tous les services QG s'appuient sur RabbitMQ pour la messagerie et PostgreSQL pour la persistance.
- Keycloak est l'IDP commun pour l'auth des opérateurs/simulations.
- Les docs ci-dessous indiquent les commandes de démarrage rapides et les zones de configuration à compléter.
