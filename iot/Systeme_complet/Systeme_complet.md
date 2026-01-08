# Système SDMIS IoT Terrain - Vue d'ensemble complète

Documentation du système complet de communication radio sécurisée pour véhicules d'intervention sur le terrain.

---

## 🎯 Objectif du système

Le système **SDMIS IoT Terrain** permet aux véhicules d'intervention (pompiers, SAMU, police) de communiquer leur position et leur statut en temps réel via un réseau radio sécurisé basé sur des cartes BBC Micro:bit.

### Fonctionnalités principales

✅ **Transmission sécurisée de positions GPS**  
✅ **Communication bidirectionnelle** (véhicules ↔ centre de commandement)  
✅ **Chiffrement AES-128** de toutes les communications  
✅ **Protocole avec acquittement** (garantie de livraison)  
✅ **Détection automatique des doublons**  
✅ **Intégration avec simulateur Java**  

---

## 🏗️ Architecture globale

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SYSTÈME SDMIS IoT TERRAIN                        │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐                                    ┌──────────────────┐
│   SIMULATEUR     │                                    │   VÉHICULES      │
│      JAVA        │                                    │    TERRAIN       │
│                  │                                    │                  │
│  • Visualisation │                                    │  • micro:bit v1  │
│  • Dispatch      │                                    │  • Module GPS    │
│  • Incidents     │                                    │  • Boutons ctrl  │
└────────┬─────────┘                                    └────────┬─────────┘
         │                                                       │
         │ UART                                                  │ Radio
         │ 115200 bps                                           │ 2.4 GHz
         │                                                       │
         ↓                                                       ↓
┌──────────────────┐        Radio SDMIS         ┌──────────────────┐
│   PASSERELLE     │ ←──────────────────────── →│   ÉMETTEURS      │
│   UART ↔ RADIO   │     AES-128 + ACK          │    TERRAIN       │
│                  │     Groupe 42              │                  │
│  • micro:bit v1  │                            │  • Envoi périod. │
│  • Chiffrement   │                            │  • Envoi manuel  │
│  • ACK/Retry     │                            │  • Statuts       │
└──────────────────┘                            └──────────────────┘

         │
         │ UART
         │ 115200 bps
         ↓
┌──────────────────┐
│   PASSERELLE     │
│   RF CENTRALE    │
│   (optionnelle)  │
│                  │
│  • micro:bit     │
│  • Gateway Python│
│  • API Backend   │
└──────────────────┘
         │
         │ HTTP/SSE
         ↓
┌──────────────────┐
│   API BACKEND    │
│                  │
│  • Base données  │
│  • REST API      │
│  • WebSocket/SSE │
└──────────────────┘
```

---

## 📦 Composants du système

### 1. Protocole CPE (Cryptographic Positioning Exchange)

**Rôle** : Couche de chiffrement et de structuration des trames

| Caractéristique | Valeur |
|-----------------|--------|
| Taille de trame | 29 octets fixes |
| Chiffrement | AES-128 en mode CTR |
| Intégrité | CRC-16 CCITT |
| Version | 1 |

**Types de messages :**
- `CPE_FT_VEH_POS` (0x01) : Position complète du véhicule
- `CPE_FT_VEH_STATUS` (0x02) : Statut uniquement
- `CPE_FT_INCIDENT_AFFECT` (0x03) : Affectation à un incident

📖 [Documentation complète du protocole CPE](PROTOCOLE_CPE.md)

---

### 2. Librairie SDMIS_RADIO

**Rôle** : Couche de fiabilité et d'abstraction radio

| Fonctionnalité | Description |
|----------------|-------------|
| ACK/Retry | Jusqu'à 3 tentatives avec backoff aléatoire |
| Anti-duplication | Détection des messages en double |
| Gestion nonce/seq | Génération automatique |
| API simple | Fonctions send_position(), send_status(), poll() |

📖 [Documentation complète SDMIS_RADIO](SDMIS_RADIO.md)

---

### 3. Passerelle UART ↔ Radio

**Rôle** : Pont entre simulateur Java et réseau radio terrain

```
Simulateur Java  ←─[UART]─→  Micro:bit  ←─[Radio]─→  Réseau SDMIS
```

**Fonctionnalités :**
- ✅ Conversion CSV ↔ Trames CPE
- ✅ Transmission bidirectionnelle
- ✅ Gestion ACK/Retry
- ✅ Indicateurs visuels (T/!/A)

📖 [Documentation complète Passerelle UART-Radio](PASSERELLE_UART_RADIO.md)

---

### 4. Émetteurs Terrain (micro:bit embarqués)

**Rôle** : Cartes embarquées dans les véhicules d'intervention

**Fonctionnalités :**
- 📍 Envoi périodique de positions GPS
- 📊 Transmission de statuts (disponible/en mission/etc.)
- 🔘 Déclenchement manuel d'événements (arrivée, renfort, etc.)
- 📻 Réception d'affectations d'incidents

📖 [Documentation complète App Terrain](APP_TERRAIN.md)

---

### 5. Passerelle RF Centrale (optionnelle)

**Rôle** : Réception des messages terrain et transmission vers backend

```
Véhicules  ─[Radio]→  Micro:bit  ─[UART]→  Gateway Python  ─[HTTP]→  API
```

**Fonctionnalités :**
- 📡 Réception de toutes les trames radio
- 🐍 Gateway Python pour interface API
- 📤 Envoi positions/statuts vers backend
- 📥 Réception affectations depuis API (SSE)

📖 [Documentation complète Passerelle RF Centrale](PASSERELLE_RF_CENTRALE.md)

---

## 🔐 Sécurité du système

### Chiffrement

| Aspect | Implémentation |
|--------|----------------|
| **Algorithme** | AES-128 en mode CTR |
| **Clé** | 128 bits (16 octets) pré-partagée |
| **IV** | Nonce (4 octets) + Seq (1 octet) + Padding |
| **Intégrité** | CRC-16 CCITT |

### Clé cryptographique

```c
// Exemple de clé (À CHANGER EN PRODUCTION !)
const uint8_t CPE_KEY[16] = {
    0x21, 0x53, 0xB6, 0x09, 0x9A, 0xD2, 0x41, 0x7C,
    0xE4, 0x10, 0x5F, 0x3A, 0x77, 0xC8, 0x90, 0x0B
};
```

⚠️ **CRITIQUE** : 
- Changer la clé en production
- Distribution sécurisée hors bande
- Rotation régulière (mensuelle recommandée)
- Une clé par réseau/mission

### Protection contre les attaques

| Attaque | Protection | Niveau |
|---------|-----------|--------|
| **Écoute passive** | Chiffrement AES-128 | ✅ Fort |
| **Rejeu** | Nonce unique + Timestamp | ⚠️ Partiel |
| **Man-in-the-Middle** | Clé pré-partagée | ⚠️ Moyen |
| **Modification** | CRC-16 | ⚠️ Faible |
| **Injection** | Nécessite la clé | ✅ Fort |

**Recommandations :**
1. Vérifier la fraîcheur du timestamp (rejeter si > 60 secondes)
2. Passer à AES-GCM pour authentification forte
3. Implémenter un système de clés de session

---

## 📊 Format d'échange CSV (UART)

### Structure générale

```
événement,immatriculation,latitude,longitude,timestamp
```

### Messages supportés

| Type | Direction | Format |
|------|-----------|--------|
| **Position** | Java → micro:bit | `vehicle_position,AB123CD,48.856614,2.352222,1736172600` |
| **Affectation** | micro:bit → Java | `vehicle_affectation,SD304FR,45.797200,4.847000,1736172600` |
| **Statut** | Java → micro:bit | `vehicle_status,AB123CD,5,1736172600` |

### Exemples détaillés

#### 📤 Envoi position (Simulateur → Terrain)

```csv
vehicle_position,VSAV304,48.856614,2.352222,1736172600
```

**Traitement :**
1. Réception UART par passerelle
2. Parsing et conversion en micro-degrés
3. Construction trame CPE chiffrée
4. Transmission radio avec ACK
5. Affichage "T" (succès) ou "!" (échec)

#### 📥 Réception affectation (Terrain → Simulateur)

```csv
vehicle_affectation,VSAV304,45.797200,4.847000,1736172600
```

**Traitement :**
1. Réception trame radio chiffrée
2. Déchiffrement et validation
3. Envoi ACK immédiat
4. Conversion en CSV
5. Transmission UART vers Java
6. Affichage "A"

---

## 🔄 Flux de communication complets

### Scénario 1 : Mise à jour de position périodique

```
┌─────────────┐                                           ┌─────────────┐
│ Simulateur  │                                           │   Véhicule  │
│    Java     │                                           │   Terrain   │
└──────┬──────┘                                           └──────┬──────┘
       │                                                         │
       │ 1. vehicle_position,AB123CD,48.85,2.35,TS              │
       ├──────────[UART 115200]────────►┌──────────┐            │
       │                                 │Passerelle│            │
       │                                 │UART-Radio│            │
       │                                 └────┬─────┘            │
       │                                      │                  │
       │                            2. Trame CPE chiffrée        │
       │                                      ├─────[Radio]─────►│
       │                                      │        ↓         │
       │                                      │   3. Déchiffre   │
       │                                      │        ↓         │
       │                                      │   4. Traite GPS  │
       │                                      │        ↓         │
       │                            5. ACK    │◄────────────────┤
       │                                      │                  │
       │                            6. "T" affiché               │
       │                                                         │
```

### Scénario 2 : Affectation à un incident

```
┌─────────────┐                                           ┌─────────────┐
│ Simulateur  │                                           │   Véhicule  │
│    Java     │                                           │   Terrain   │
└──────┬──────┘                                           └──────┬──────┘
       │                                                         │
       │                                    1. Incident détecté  │
       │                                                    ┌────┴────┐
       │                                                    │ Bouton  │
       │                                                    │ pressé  │
       │                                                    └────┬────┘
       │                                  2. Trame INCIDENT      │
       │                            ┌──────────┐   AFFECT        │
       │                            │Passerelle│◄────[Radio]─────┤
       │                            │UART-Radio│        ↓         │
       │                            └────┬─────┘   3. ACK envoyé │
       │                                 │                        │
       │                       4. Déchiffre + Parse              │
       │                                 │                        │
       │ 5. vehicle_affectation,AB,45.8,4.8,TS                   │
       │◄─────────[UART]─────────────────┤                       │
       │                                 │                        │
       │ 6. Affiche sur carte            │         7. "A" affiché│
       │                                                         │
```

---

## 📈 Performances du système

### Latences

| Opération | Latence typique | Notes |
|-----------|-----------------|-------|
| **UART → Radio** | 30-50 ms | Avec ACK reçu au 1er essai |
| **UART → Radio (échec)** | ~650 ms | Après 3 timeouts |
| **Radio → UART** | 10-20 ms | Déchiffrement + transmission |
| **End-to-end (Java → Terrain)** | 40-70 ms | Incluant traitements |

### Débit

| Flux | Débit max théorique | Débit pratique |
|------|---------------------|----------------|
| Positions terrain | ~15 msg/s | ~5 msg/s recommandé |
| Affectations | Illimité en RX | ~10 msg/s |

### Portée radio

| Environnement | Portée | Conditions |
|---------------|--------|------------|
| **Intérieur** | 20-50 m | Murs, obstacles |
| **Extérieur dégagé** | 100-150 m | Puissance max (7) |
| **Urbain dense** | 50-80 m | Immeubles, interférences |

**Facteurs d'amélioration :**
- Augmenter puissance TX (paramètre 0-7)
- Position en hauteur des antennes
- Éviter obstacles métalliques
- Éloigner sources WiFi 2.4 GHz

---

## 🛠️ Installation et déploiement

### Prérequis matériels

| Composant | Quantité | Usage |
|-----------|----------|-------|
| BBC Micro:bit v1 | 1+ | Passerelle UART-Radio |
| BBC Micro:bit v1 | N | Émetteurs terrain (1 par véhicule) |
| Module GPS | N | Optionnel si position simulée |
| Câble USB | 1+ | Programmation et liaison série |
| Batteries | N | Alimentation terrain |

### Prérequis logiciels

| Outil | Version | Usage |
|-------|---------|-------|
| **Docker** | 20.10+ | Compilation recommandée |
| **GNU Arm Toolchain** | - | Compilation alternative |
| **CMake** | 3.6+ | Build système |
| **Python** | 3.9+ | Gateway RF (optionnel) |
| **Java** | 11+ | Simulateur |

### Étapes de déploiement

#### 1. Cloner le dépôt

```bash
git clone https://github.com/votre-org/iot-terrain-microbit.git
cd iot-terrain-microbit
```

#### 2. Configurer la clé cryptographique

**Générer une nouvelle clé :**
```bash
# Linux/macOS
openssl rand -hex 16
# Exemple: 2153b6099ad2417ce4105f3a77c8900b
```

**Éditer la clé dans le code :**
```cpp
// source/main.cpp ou fichier de config
static const uint8_t CPE_KEY[16] = {
    0x21, 0x53, 0xB6, 0x09, 0x9A, 0xD2, 0x41, 0x7C,
    0xE4, 0x10, 0x5F, 0x3A, 0x77, 0xC8, 0x90, 0x0B
};
```

⚠️ **La même clé doit être présente sur toutes les cartes !**

#### 3. Compiler les firmwares

**Passerelle UART-Radio :**
```bash
cd iot-terrain-microbit
make clean && make build
# Génère: out/iot-terrain-microbit.hex
```

**Émetteurs terrain :**
```bash
cd iot-terrain-microbit
# Modifier source/main.cpp pour mode terrain
make clean && make build
# Génère: out/iot-terrain-microbit.hex
```

#### 4. Flasher les cartes

**macOS :**
```bash
cp out/iot-terrain-microbit.hex /Volumes/MICROBIT/
```

**Linux :**
```bash
cp out/iot-terrain-microbit.hex /media/$USER/MICROBIT/
```

**Windows :**
- Glisser-déposer le `.hex` sur le lecteur `MICROBIT:`

#### 5. Configuration du simulateur Java

```java
// Configuration port série
String port = "/dev/ttyACM0";  // Linux
// String port = "/dev/tty.usbmodem*";  // macOS
// String port = "COM4";  // Windows

int baudrate = 115200;
```

#### 6. Lancement

1. Connecter passerelle UART-Radio en USB
2. Lancer simulateur Java
3. Allumer émetteurs terrain
4. Vérifier LED pixel (4,4) allumée

---

## 🧪 Tests et validation

### Test 1 : Communication UART

**Objectif** : Vérifier la liaison série

```bash
# Terminal 1 : Moniteur série
screen /dev/ttyACM0 115200

# Terminal 2 : Envoi test
echo "vehicle_position,TEST001,48.856614,2.352222,1736172600" > /dev/ttyACM0
```

**Résultat attendu :**
- Affichage "T" sur la micro:bit si récepteur à portée
- Affichage "!" si aucun récepteur

### Test 2 : Communication radio

**Objectif** : Valider le lien radio avec ACK

1. Programmer 2 micro:bits avec le même firmware et la même clé
2. Une en mode passerelle (connectée USB)
3. Une en mode terrain (sur batterie)
4. Envoyer position via UART
5. Observer "T" → ACK reçu

### Test 3 : Chiffrement

**Objectif** : Vérifier que la clé est nécessaire

1. Programmer émetteur avec clé A
2. Programmer récepteur avec clé B (différente)
3. Envoyer message
4. Résultat : Pas d'ACK, échec de validation CRC

### Test 4 : Portée radio

**Objectif** : Mesurer la portée effective

1. Placer émetteur fixe
2. Éloigner récepteur progressivement
3. Noter distance maximale avec ACK
4. Recommencer avec puissance TX = 4, 5, 6, 7

**Résultats typiques :**
- Intérieur : 30-50 m
- Extérieur : 100-150 m

---

## 🐛 Diagnostic et résolution de problèmes

### Problème : Aucune communication

| Symptôme | Cause probable | Solution |
|----------|---------------|----------|
| LED (4,4) éteinte | Firmware non flashé | Re-flasher la carte |
| Pas d'affichage T/!/A | Pas de données UART | Vérifier connexion série |
| Affichage "!" permanent | Aucun récepteur | Vérifier récepteur allumé + portée |
| Messages non reçus | Clé différente | Re-flasher avec même clé |
| Timeouts fréquents | Interférences radio | Éloigner WiFi, changer de canal |

### Debug mode

Activer les logs UART dans le code :

```cpp
static const bool DEBUG_UART_ECHO = true;  // Dans main.cpp
```

Affiche :
```
GOT:vehicle_position,AB123CD,48.856614,2.352222,1736172600
```

### Monitoring avancé

```bash
# Linux : Capture trafic série
cat /dev/ttyACM0 | tee uart_log.txt

# Analyse
grep "vehicle_position" uart_log.txt | wc -l  # Compter messages
```

---

## 📚 Documentation technique complète

| Document | Contenu |
|----------|---------|
| [Protocole CPE](PROTOCOLE_CPE.md) | Spécification chiffrement et trames |
| [Librairie SDMIS_RADIO](SDMIS_RADIO.md) | API haut niveau, ACK, anti-duplication |
| [Passerelle UART-Radio](PASSERELLE_UART_RADIO.md) | Interface simulateur ↔ radio |
| [App Terrain](APP_TERRAIN.md) | Émetteur embarqué véhicules |
| [Passerelle RF Centrale](PASSERELLE_RF_CENTRALE.md) | Gateway backend (optionnel) |

---

## 🚀 Roadmap et améliorations

### Phase 1 : Stabilisation (actuel)
- [x] Protocole CPE avec chiffrement
- [x] Librairie SDMIS_RADIO
- [x] Passerelle UART-Radio
- [x] Support affectations

### Phase 2 : Fiabilisation
- [ ] Validation timestamp (anti-rejeu)
- [ ] Buffer circulaire anti-duplication
- [ ] Statistiques de transmission
- [ ] Mode économie d'énergie

### Phase 3 : Fonctionnalités
- [ ] Support messages `vehicle_status`
- [ ] Messages opérateur libres
- [ ] Demande de renfort
- [ ] Accusés de réception applicatifs

### Phase 4 : Sécurité avancée
- [ ] AES-GCM (authentification forte)
- [ ] Clés de session (échange Diffie-Hellman)
- [ ] Rotation automatique de clés
- [ ] Audit trail cryptographique

### Phase 5 : Scalabilité
- [ ] Mesh networking (relais multi-sauts)
- [ ] Gestion multi-canal radio
- [ ] Support 100+ véhicules simultanés
- [ ] Gateway haute performance (C++)

---

## 📄 Licence et contributions

**Licence** : Voir [LICENSE](../LICENSE)

**Contributions** :
- Issues : GitHub Issues
- Pull requests : Bienvenues avec tests

**Contact** : 
- Projet : IoT Terrain Micro:bit
- Date : Janvier 2026

---

## ⚠️ Avertissements

### Usage professionnel

Ce système est conçu pour un usage professionnel en environnement contrôlé :

- ✅ Exercices et simulations
- ✅ Tests et développement
- ⚠️ Déploiement opérationnel : validation sécurité requise

### Limitations connues

| Limitation | Impact | Mitigation |
|-----------|--------|------------|
| Portée radio limitée | ~150 m max | Déployer relais intermédiaires |
| Clé symétrique | Compromission = tout le réseau | Rotation régulière |
| Pas d'authentification forte | Injection possible avec clé | Passer à AES-GCM |
| CRC-16 faible | Pas de protection cryptographique | Implémenter HMAC |
| Un seul groupe radio | Scalabilité limitée | Multi-canal + mesh |

### Conformité réglementaire

- 📡 Bande 2.4 GHz ISM : Usage libre en Europe/USA
- 🔒 Chiffrement : Conforme réglementation française
- ⚠️ Usage militaire/sécurité : Homologation requise

---

**Version du document** : 1.0  
**Dernière mise à jour** : Janvier 2026
