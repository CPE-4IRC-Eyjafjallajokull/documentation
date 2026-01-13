# Specifications fonctionnelles (synthese)

Ce document presente une vue fonctionnelle claire du systeme SDMIS.

---

## 1. Objectifs

- Superviser incidents, interventions et ressources en temps quasi reel.
- Assister la decision d'affectation des vehicules.
- Coordonner les equipes terrain via IoT.
- Permettre la simulation complete pour tests et demos.

---

## 2. Acteurs

- **Operateur QG** : cree et suit les incidents, valide les propositions.
- **Equipe terrain** : envoie position/statut, recoit les affectations.
- **Simulation** : injecte des incidents et vehicules.
- **Systeme** : API + moteur de decision.

---

## 3. Donnees gerees

- Incidents, phases, statuts.
- Vehicules, positions, energies, affectations.
- Victimes, transports, points d'interet.
- Historique d'evenements et journaux.

---

## 4. Parcours principaux

### 4.1 Incident
1. Creation (operateur ou simulation).
2. Demande d'affectation.
3. Proposition automatique (moteur Java).
4. Validation/refus par l'operateur.
5. Suivi temps reel jusqu'a cloture.

### 4.2 Telemetrie terrain
1. Positions/statuts remontee via RF.
2. API met a jour la base.
3. SSE diffuse vers les fronts.

### 4.3 Affectation terrain
1. API publie une affectation.
2. Passerelle RF transmet vers micro:bit.
3. Vehicule applique l'affectation.

---

## 5. Fonctionnalites par composant

### App QG Front
- Carte temps reel (incidents, vehicules, points d'interet).
- Gestion incidents et phases.
- Validation des propositions d'affectation.
- Suivi des vehicules (statut, position).

### App QG API + Moteur Java
- Validation et normalisation des donnees.
- Publication/consommation RabbitMQ.
- Calcul des propositions d'affectation.
- Diffusion SSE vers les fronts.

### App Terrain Front
- Selection et suivi du vehicule.
- Suivi de l'intervention en cours.
- Consultation d'incidents et casernes.

### IoT (micro:bit + passerelle)
- Envoi de positions/statuts.
- Reception d'affectations.
- Communication radio securisee (CPE).

### Simulation
- Generation d'incidents (API).
- Simulation de vehicules (UART/route).

---

## 6. Exigences non-fonctionnelles

- **Temps reel** : SSE pour les mises a jour visibles immediatement.
- **Securite** : authentification Keycloak (JWT).
- **Robustesse** : files RabbitMQ pour absorber les pics.
- **Traçabilite** : logs et historique des evenements.

---

## 7. Roles et autorisations (Keycloak)

- `qg-operator` : acces complet QG.
- `qg-engine` : acces moteur.
- `qg-vehicles` : telemetrie vehicules.
- `qg-viewer` : lecture seule.
