# Passerelle Micro:bit UART ↔ Radio SDMIS
Récepteur radio côté Simulation : Recoit les trames des véhicules terrain et les envoies en radio (bidirectionnel)
## 🎯 Vue d'ensemble

La carte **Micro:bit** fonctionne comme une passerelle bidirectionnelle entre un simulateur Java (via liaison série UART) et un réseau radio IoT terrain utilisant le protocole **SDMIS crypté avec acquittement**.

---

## ⚙️ Configuration technique

| Paramètre | Valeur |
|-----------|--------|
| **Baudrate UART** | 115200 bps |
| **Groupe radio** | 42 |
| **Puissance radio** | 7 (maximum) |
| **Chiffrement** | AES-128 avec clé pré-partagée |
| **Timeout ACK** | 200 ms |
| **Tentatives de retransmission** | 3 maximum |

---

## 🔄 Fonctionnement

### 1️⃣ Communication UART → Radio (Java → Terrain)

Le simulateur Java envoie des positions de véhicules au format CSV contenant l'événement `vehicle_position`, suivi de l'immatriculation, de la latitude, de la longitude et du timestamp.

**Actions effectuées :**

1. Réception de la ligne CSV via UART
2. Analyse des données : événement, immatriculation, latitude, longitude, timestamp
3. Si l'événement est `vehicle_position` :
   - Chiffrement de la trame avec **numéro de séquence unique**
   - Transmission de la position via radio SDMIS
   - **Attente d'un ACK** pendant 200 ms maximum
   - Si aucun ACK reçu : **retransmission jusqu'à 3 tentatives** avec délai aléatoire (10-40 ms)
   - Affichage de **"T"** (Transmis avec ACK reçu) ou **"!"** (Erreur, aucun ACK après 3 tentatives) sur l'écran

### 2️⃣ Communication Radio → UART (Terrain → Java)

Lorsqu'un message radio de type affectation de véhicule à un incident est reçu sur le réseau SDMIS :

**Actions effectuées :**

1. Réception et déchiffrement automatique du message radio
2. **Envoi immédiat d'un ACK** à l'émetteur avec le numéro de séquence reçu
3. **Détection des doublons** (même séquence + même nonce) pour éviter le traitement multiple
4. Extraction des données (immatriculation, position GPS, timestamp)
5. Transmission au simulateur Java via UART d'un message CSV contenant l'événement `vehicle_affectation`, l'immatriculation, la latitude, la longitude et le timestamp
6. Affichage de **"A"** (Affectation) sur l'écran

---

## 📊 Format des données

### Format CSV échangé sur UART

Le format est structuré en **cinq champs** séparés par des virgules :
```
événement,immatriculation,latitude_décimale,longitude_décimale,timestamp_unix
```

### Exemples de messages

**📤 Envoi d'une position de véhicule (Java → Micro:bit) :**
```csv
vehicle_position,AB123CD,48.856614,2.352222,1736172600
```

**📥 Réception d'une affectation (Micro:bit → Java) :**
```csv
vehicle_affectation,SD304FR,45.797200,4.847000,1736172600
```

> **Taille typique :** ~59 octets par message transmis sur UART

---

## 🔐 Sécurité et fiabilité

### Chiffrement

- ✅ Tous les messages radio sont chiffrés en **AES-128**
- ✅ Authentification des messages par **CMAC** (Cipher-based Message Authentication Code)
- ✅ Clé cryptographique de **128 bits** pré-configurée dans le firmware

### Fiabilité de transmission

- ✅ **Protocole avec acquittement (ACK)** automatique
- ✅ Chaque trame possède un **numéro de séquence unique**
- ✅ **Retransmission automatique** (jusqu'à 3 tentatives) en cas d'absence d'ACK
- ✅ **Délai aléatoire** entre retransmissions (10-40 ms) pour éviter les collisions
- ✅ **Détection et élimination des doublons** via nonce + séquence
- ✅ Le système **garantit la livraison** ou notifie l'échec

---

## 💡 Indicateurs visuels

| Indicateur | Signification |
|------------|---------------|
| Pixel (4,4) allumé | Système actif et en fonctionnement |
| **T** | Position transmise avec succès et ACK reçu |
| **!** | Échec de transmission (aucun ACK reçu après 3 tentatives) |
| **A** | Affectation reçue, ACK envoyé et transmise au simulateur |

---

## 🔧 Déploiement

1. Connexion de la Micro:bit au PC via câble USB
2. Flash du firmware compilé sur la carte
3. Lancement du simulateur Java configuré sur le port série approprié
4. La passerelle assure automatiquement la communication bidirectionnelle entre le simulateur et le réseau radio

---

## 🏗️ Architecture système

Cette architecture permet au simulateur Java d'**envoyer des positions de véhicules** qui sont diffusées sur le réseau radio terrain **avec confirmation de réception**, et de **recevoir en temps réel** les affectations de véhicules à des incidents provenant du réseau SDMIS sécurisé. 

La Micro:bit agit comme un **pont transparent** gérant le chiffrement, le déchiffrement, la conversion de format et le **protocole d'acquittement** pour garantir la fiabilité des communications même en environnement radio perturbé.
```
┌─────────────┐      UART       ┌──────────┐      Radio       ┌─────────────┐
│ Simulateur  │ ←─────────────→ │ Micro:bit│ ←──────────────→ │   Réseau    │
│    Java     │   115200 bps    │ Passerelle│   AES-128+ACK   │   SDMIS     │
└─────────────┘                 └──────────┘                  └─────────────┘
                                      ↕
                           Gestion ACK/Retries
                           Détection doublons
```

---

# 📡 App Terrain micro:bit (émetteur)

Micro:bit embarqué dans le véhicule pour émettre les informations terrain par radio (positions, états, messages opérateur).

## 🎯 Rôle

- **Envoi périodique** : GPS + statut (disponible/en route/sur intervention)
- **Envoi manuel** : 
  - Arrivée sur site
  - Fin d'intervention
  - Demande de renfort
  - Message libre
- **Respect du protocole** : RF/UART décrit dans la section IoT (trames sécurisées + identifiants véhicule)

---

## 🔨 Build

### Prérequis

- **GNU Arm Embedded Toolchain** (`arm-none-eabi-*`)
- **CMake** 3.6+
- **Python** 3.x

### Compilation
```bash
cd iot-terrain-microbit
python3 build.py
# Génère MICROBIT.hex et MICROBIT.bin à la racine
```

---

## 📁 Arborescence
```
iot-terrain-microbit/
├── source/
│   ├── main.cpp           # Point d'entrée (scheduler CODAL) à enrichir
│   ├── proto/             # Helpers de trames (cohérence avec passerelle)
│   └── crypto/            # Implémentation AES-CMAC
├── build.py               # Script de compilation
└── [MICROBIT.hex/bin]     # Fichiers générés
```

### Fichiers clés

- `source/main.cpp` : Point d'entrée de la carte Micro:bit
- `source/proto/` : Helpers de formatage des trames
- `source/crypto/` : Implémentation AES-CMAC (cohérence avec passerelle centrale)

---

## 🚀 Utilisation terrain

1. **Initialisation** : Flash du firmware avec identifiant véhicule unique
2. **Démarrage automatique** : Envoi périodique des positions GPS
3. **Interactions manuelles** : Boutons pour signaler les événements opérationnels
4. **Monitoring** : LED et écran pour feedback visuel de l'état de communication

---

## 🔗 Intégration avec le système global

Le micro:bit terrain communique avec :
- **Passerelle centrale** (via radio groupe 42)
- **Simulateur Java** (indirectement via la passerelle)
