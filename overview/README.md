# Vue d'ensemble du systeme

Le systeme SDMIS combine une application QG (supervision), une application terrain,
un moteur de decision, une chaine IoT micro:bit et des simulateurs pour tester sans
materiel reel.

---

## Composants principaux

- **App QG Front** : interface operateur (carte, incidents, affectations).
- **App QG API** : API REST centrale + SSE, expose les donnees et orchestre les flux.
- **Moteur Java** : calcule les propositions d'affectation via RabbitMQ.
- **App Terrain Front** : interface terrain pour vehicules/interventions.
- **IoT** : micro:bit terrain + passerelle RF (UART/RabbitMQ).
- **Simulation** : generateurs d'incidents et de vehicules.

---

## Schema simplifie

```
Utilisateurs
  | (web)
  v
[Front QG]   [Front Terrain]
      \       /
       \     /
        v   v
        [API FastAPI] ---- PostgreSQL
             |
             | RabbitMQ
             v
         [Java Engine]

IoT micro:bit -> RF Gateway -> RabbitMQ -> API
Simulation --------> API / RabbitMQ

SSE : API -> Fronts (temps reel)
```

---

## Flux a connaitre

- Flux incidents : [flux-incidents.md](flux-incidents.md)
- Flux terrain/telemetrie : [flux-telemetrie.md](flux-telemetrie.md)

---

## Demarrage

Pour lancer l'ensemble en local (BDD, RabbitMQ, Keycloak, apps) :
[infrastructure/README.md](../infrastructure/README.md)
