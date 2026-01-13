# Applications

Synthese des composants applicatifs et liens vers leur documentation detaillee.

---

## Repartition

- **App QG API (FastAPI)** : [qg-api.md](qg-api.md)
- **App QG Front (Next.js)** : [qg-front/README.md](qg-front/README.md)
- **Moteur decisionnel Java** : [qg-java-engine.md](qg-java-engine.md)
- **App Terrain Front (Next.js)** : [terrain-front.md](terrain-front.md)

---

## SSE (temps reel)

- Demarrage rapide : [sse-quickstart.md](sse-quickstart.md)
- Front QG (hooks + proxy) : [qg-front/sse.md](qg-front/sse.md)
- API QG (endpoint /qg/live) : [qg-api-sse.md](qg-api-sse.md)

---

## Authentification

- Setup Keycloak + NextAuth : [auth/keycloak-nextauth-setup.md](auth/keycloak-nextauth-setup.md)
- Integration detaillee : [auth/keycloak-nextauth-integration.md](auth/keycloak-nextauth-integration.md)

---

## A retenir

- **RabbitMQ** pour la messagerie interne.
- **PostgreSQL** pour la persistance des donnees.
- **Keycloak** comme IDP pour les fronts et l'API.
