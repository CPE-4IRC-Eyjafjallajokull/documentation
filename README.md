# Systeme Integre de Gestion et Suivi des Incidents — Documentation

Ce dossier regroupe la documentation fonctionnelle et technique du projet SDMIS.
L'objectif est d'expliquer simplement comment le systeme fonctionne, comment il se demarre
et comment les composants dialoguent entre eux.

---

## Lire en premier

- Vue d'ensemble du systeme : [overview/README.md](overview/README.md)
- Flux des incidents : [overview/flux-incidents.md](overview/flux-incidents.md)
- Flux terrain et telemetrie : [overview/flux-telemetrie.md](overview/flux-telemetrie.md)

---

## Demarrage rapide (dev)

- Stack Docker complete (BDD, RabbitMQ, Keycloak, apps) : [infrastructure/README.md](infrastructure/README.md)
- Details des applications : [apps/README.md](apps/README.md)
- Simulation (incidents/vehicules) : [simulation/README.md](simulation/README.md)
- IoT (micro:bit + passerelle RF) : [iot/README.md](iot/README.md)

---

## Architecture (resume)

```
[Front QG] ----\
                >-- API (FastAPI) -- PostgreSQL
[Front Terrain]-/
                    |
                    | SSE (temps reel)
                    v
                 Navigateurs

API <-> RabbitMQ <-> Java Engine (propositions d'affectation)
IoT micro:bit -> RF Gateway -> RabbitMQ -> API
Simulation -> API / RabbitMQ
```

---

## Documentation par theme

### Vue d'ensemble
- [overview/README.md](overview/README.md)
- [overview/flux-incidents.md](overview/flux-incidents.md)
- [overview/flux-telemetrie.md](overview/flux-telemetrie.md)

### Applications
- [apps/README.md](apps/README.md)
- API FastAPI : [apps/qg-api.md](apps/qg-api.md)
- Front QG : [apps/qg-front/README.md](apps/qg-front/README.md)
- Front Terrain : [apps/terrain-front.md](apps/terrain-front.md)
- Moteur Java : [apps/qg-java-engine.md](apps/qg-java-engine.md)
- SSE : [apps/sse-quickstart.md](apps/sse-quickstart.md)
- Auth Keycloak/NextAuth : [apps/auth/keycloak-nextauth-setup.md](apps/auth/keycloak-nextauth-setup.md)

### IoT
- [iot/README.md](iot/README.md)
- Terrain micro:bit : [iot/terrain-microbit.md](iot/terrain-microbit.md)
- Passerelle RF : [iot/rf-central-gateway.md](iot/rf-central-gateway.md)
- Protocole : [iot/protocole/protocole-cpe.md](iot/protocole/protocole-cpe.md)

### Simulation
- [simulation/README.md](simulation/README.md)
- Incidents : [simulation/incidents.md](simulation/incidents.md)
- Vehicules : [simulation/vehicles.md](simulation/vehicles.md)

### Specifications et annexes
- Specifications fonctionnelles : [specs/functional.md](specs/functional.md)
- Matrice des flux applicatifs : [references/matrice-des-flux.xlsx](references/matrice-des-flux.xlsx)
- Securite CI : [security/](security/)

---

## Objectif

Fournir une reference unique, claire et maintenable pour comprendre le systeme,
faciliter la prise en main, et aligner l'implementation avec les choix techniques.
