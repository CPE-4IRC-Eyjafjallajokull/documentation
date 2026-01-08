# Passerelle Micro:bit UART ↔ Radio SDMIS

**Récepteur radio côté Simulation** : Reçoit les trames des véhicules terrain et les envoie en radio (bidirectionnel)

---

## 🎯 Vue d'ensemble

La carte **Micro:bit** fonctionne comme une passerelle bidirectionnelle entre un simulateur Java (via liaison série UART) et un réseau radio IoT terrain utilisant le protocole **SDMIS crypté avec acquittement**.

### Architecture du système

```
┌─────────────┐      UART       ┌───────────┐      Radio       ┌─────────────┐
│ Simulateur  │ ←─────────────→ │ Micro:bit │ ←──────────────→ │   Réseau    │
│    Java     │   115200 bps    │ Passerelle│   AES-128+ACK    │   SDMIS     │
└─────────────┘                 └───────────┘                  └─────────────┘
                                      ↕
                           Gestion ACK/Retries
                           Détection doublons
```

---

## ⚙️ Configuration technique

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| **Baudrate UART** | 115200 bps | Communication série avec le simulateur |
| **Buffer RX UART** | 254 octets | Taille maximale du buffer de réception |
| **Groupe radio** | 42 | Identifiant du réseau radio SDMIS |
| **Puissance radio** | 7 (maximum) | Portée ~150 m en extérieur |
| **Chiffrement** | AES-128 CTR | Avec clé pré-partagée de 16 octets |
| **Timeout ACK** | 200 ms | Attente de l'acquittement radio |
| **Tentatives de retransmission** | 3 maximum | Avant signalement d'échec |
| **Délai entre retries** | 10-40 ms | Backoff aléatoire pour éviter collisions |

### Clé cryptographique

```c
static const uint8_t CPE_KEY[16] = {
    0x21, 0x53, 0xB6, 0x09, 0x9A, 0xD2, 0x41, 0x7C,
    0xE4, 0x10, 0x5F, 0x3A, 0x77, 0xC8, 0x90, 0x0B
};
```

⚠️ **IMPORTANT** : Cette clé doit être identique sur toutes les unités du réseau SDMIS.

---

## 🔄 Fonctionnement

### 1️⃣ Communication UART → Radio (Java → Terrain)

Le simulateur Java envoie des positions de véhicules au format CSV via la liaison série.

#### Flux de traitement

```
┌──────────────────────────────────────────────────────────────────┐
│  1. Réception UART (caractère par caractère)                     │
│     └─ Buffer jusqu'à '\n' (fin de ligne)                        │
├──────────────────────────────────────────────────────────────────┤
│  2. Parsing CSV                                                  │
│     └─ Extraction: event, immat, lat, lon, timestamp             │
├──────────────────────────────────────────────────────────────────┤
│  3. Vérification événement = "vehicle_position"                  │
├──────────────────────────────────────────────────────────────────┤
│  4. Construction trame CPE                                       │
│     └─ Chiffrement AES-128 + numéro de séquence unique          │
├──────────────────────────────────────────────────────────────────┤
│  5. Transmission radio SDMIS                                     │
│     └─ Envoi avec attente ACK (200 ms timeout)                  │
├──────────────────────────────────────────────────────────────────┤
│  6. Gestion des retransmissions                                  │
│     ├─ Si ACK reçu: Succès → Affichage "T"                      │
│     └─ Si pas d'ACK après 3 tentatives: Échec → Affichage "!"   │
└──────────────────────────────────────────────────────────────────┘
```

#### Exemple de séquence

**Réception UART :**
```csv
vehicle_position,AB123CD,48.856614,2.352222,1736172600
```

**Actions :**
1. Parse les données : immat="AB123CD", lat=48856614 (×10⁶), lon=2352222 (×10⁶)
2. Génère nonce et séquence automatiquement
3. Construit trame CPE chiffrée de 29 octets
4. **Tentative 1** : Envoi radio → Attente ACK (200 ms)
5. Si pas d'ACK : Backoff 15 ms → **Tentative 2**
6. Si pas d'ACK : Backoff 28 ms → **Tentative 3**
7. Affichage : **"T"** si succès, **"!"** si échec

### 2️⃣ Communication Radio → UART (Terrain → Java)

Lorsqu'un message radio de type **affectation de véhicule à un incident** est reçu sur le réseau SDMIS.

#### Flux de traitement

```
┌──────────────────────────────────────────────────────────────────┐
│  1. Réception trame radio SDMIS (29 octets chiffrés)            │
├──────────────────────────────────────────────────────────────────┤
│  2. Déchiffrement automatique                                    │
│     └─ Validation CRC-16 et version protocole                    │
├──────────────────────────────────────────────────────────────────┤
│  3. Envoi immédiat ACK à l'émetteur                             │
│     └─ 1 octet contenant le numéro de séquence                  │
├──────────────────────────────────────────────────────────────────┤
│  4. Détection des doublons                                       │
│     └─ Vérification (seq + nonce) déjà traité                   │
│     └─ Si duplicata: Ignorer (mais ACK déjà envoyé)             │
├──────────────────────────────────────────────────────────────────┤
│  5. Vérification type = CPE_FT_INCIDENT_AFFECT                   │
├──────────────────────────────────────────────────────────────────┤
│  6. Extraction des données                                       │
│     └─ immat, lat_e6, lon_e6, timestamp                         │
├──────────────────────────────────────────────────────────────────┤
│  7. Conversion coordonnées (micro-degrés → décimales)            │
├──────────────────────────────────────────────────────────────────┤
│  8. Construction message CSV                                     │
│     └─ "vehicle_affectation,IMMAT,LAT,LON,TS"                   │
├──────────────────────────────────────────────────────────────────┤
│  9. Transmission UART vers simulateur Java                       │
│     └─ Affichage "A" (Affectation)                              │
└──────────────────────────────────────────────────────────────────┘
```

#### Exemple de séquence

**Réception radio :**
- Trame CPE chiffrée 29 octets : type=`INCIDENT_AFFECT`, immat="SD304FR", lat=45797200, lon=4847000

**Actions :**
1. Déchiffrement et validation CRC → OK
2. Envoi ACK immédiat avec seq=42
3. Vérification duplicata (seq=42, nonce=0x12345678) → Nouveau
4. Extraction : immat="SD304FR", lat_e6=45797200, lon_e6=4847000, ts=1736172600
5. Conversion : 45797200 → 45.797200°, 4847000 → 4.847000°
6. Construction CSV

**Envoi UART :**
```csv
vehicle_affectation,SD304FR,45.797200,4.847000,1736172600
```

**Affichage :** **"A"**

---

## 📊 Format des données

### Format CSV échangé sur UART

Structure en **cinq champs** séparés par des virgules :

```
événement,immatriculation,latitude_décimale,longitude_décimale,timestamp_unix
```

| Champ | Type | Format | Exemple |
|-------|------|--------|---------|
| **événement** | String | `vehicle_position` ou `vehicle_affectation` | `vehicle_position` |
| **immatriculation** | String | 8 caractères max | `AB123CD` |
| **latitude** | Float | Degrés décimaux (6 décimales) | `48.856614` |
| **longitude** | Float | Degrés décimaux (6 décimales) | `2.352222` |
| **timestamp** | Integer | Secondes depuis epoch Unix | `1736172600` |

### Exemples de messages

#### 📤 Envoi d'une position de véhicule (Java → Micro:bit)

```csv
vehicle_position,AB123CD,48.856614,2.352222,1736172600
```

**Signification :**
- Véhicule "AB123CD"
- Position : 48.856614° N, 2.352222° E (Tour Eiffel, Paris)
- Timestamp : 2026-01-06 18:50:00 UTC

#### 📥 Réception d'une affectation (Micro:bit → Java)

```csv
vehicle_affectation,SD304FR,45.797200,4.847000,1736172600
```

**Signification :**
- Véhicule "SD304FR" affecté à un incident
- Position incident : 45.797200° N, 4.847000° E (Vieux Lyon)
- Timestamp : 2026-01-06 18:50:00 UTC

### Taille des messages

| Type | Taille approximative |
|------|---------------------|
| `vehicle_position` | ~59 octets |
| `vehicle_affectation` | ~64 octets |

---

## 🔐 Sécurité et fiabilité

### Chiffrement AES-128

| Élément | Description |
|---------|-------------|
| **Algorithme** | AES-128 en mode CTR (Counter) |
| **Taille de clé** | 128 bits (16 octets) |
| **Vecteur d'initialisation** | Construit avec nonce (4 octets) + seq (1 octet) + padding |
| **Intégrité** | CRC-16 CCITT calculé avant chiffrement |
| **Authentification** | Implicite via validation CRC après déchiffrement |

**Avantages :**
- ✅ Confidentialité des communications
- ✅ Protection contre l'écoute passive
- ✅ Détection de corruption (CRC-16)

**Limitations :**
- ⚠️ Pas d'authentification cryptographique forte (HMAC/GCM)
- ⚠️ Clé symétrique partagée (compromission = tout le réseau)

### Protocole d'acquittement (ACK)

Le système implémente un mécanisme de fiabilité inspiré des réseaux sans fil :

#### Schéma du protocole

```
Émetteur (Passerelle)                    Récepteur (Terrain)
        │                                        │
        ├──────── Trame CPE (29 octets) ───────→│
        │                                        ├─ Déchiffre
        │                                        ├─ Valide CRC
        │            ⏱ 200 ms timeout            ├─ Traite
        │←──────────── ACK (seq=X) ──────────────┤
        │                                        │
     [Succès]                                 [Continue]
```

#### En cas d'échec

```
Tentative 1:  Envoi ──→ ⏱ 200 ms ──→ [Pas d'ACK]
              ↓
         Backoff 10-40 ms (aléatoire)
              ↓
Tentative 2:  Envoi ──→ ⏱ 200 ms ──→ [Pas d'ACK]
              ↓
         Backoff 10-40 ms (aléatoire)
              ↓
Tentative 3:  Envoi ──→ ⏱ 200 ms ──→ [Pas d'ACK]
              ↓
           [ÉCHEC]
```

#### Caractéristiques

| Paramètre | Valeur | Raison |
|-----------|--------|--------|
| **Timeout ACK** | 200 ms | Temps traitement + latence radio |
| **Tentatives max** | 3 | Compromis fiabilité/latence |
| **Backoff** | 10-40 ms aléatoire | Évite collisions synchronisées |
| **Format ACK** | 1 octet (numéro seq) | Minimal pour performance |

### Détection et élimination des doublons

Le système mémorise le dernier message traité pour éviter les duplicatas.

#### Méthode

```c
// Variables statiques globales
static uint8_t g_last_rx_seq = 0;
static uint32_t g_last_rx_nonce = 0;

// Vérification lors de la réception
if (frame.seq == g_last_rx_seq && frame.nonce == g_last_rx_nonce) {
    return; // Duplicata détecté, ignorer
}

// Mémorisation
g_last_rx_seq = frame.seq;
g_last_rx_nonce = frame.nonce;
```

**Avantages :**
- ✅ Évite le traitement multiple d'une même trame
- ✅ Gère les retransmissions de l'émetteur
- ✅ Économise les ressources CPU et UART

**Limitations :**
- ⚠️ Un seul couple (seq, nonce) mémorisé
- ⚠️ Fonctionne bien pour un émetteur à la fois
- ⚠️ Multi-émetteurs simultanés : risque de faux négatifs

---

## 💡 Indicateurs visuels

| Indicateur | Signification | Durée |
|------------|---------------|-------|
| Pixel (4,4) allumé en continu | Système actif et en fonctionnement | Permanent |
| **T** affiché | Position transmise avec succès et ACK reçu | 1 seconde |
| **!** affiché | Échec de transmission (aucun ACK après 3 tentatives) | 1 seconde |
| **A** affiché | Affectation reçue, ACK envoyé et transmise au simulateur | 1 seconde |

---

## 🔧 Déploiement

### Prérequis

- Micro:bit v1 ou v2
- Câble USB pour connexion série
- Simulateur Java compatible
- Docker (pour compilation) ou yotta installé localement

### Étapes d'installation

#### 1. Compilation du firmware

**Option A : Avec Docker (recommandé)**
```bash
cd iot-terrain-microbit
make build
```
Génère : `out/iot-terrain-microbit.hex`

**Option B : Avec yotta local**
```bash
cd iot-terrain-microbit
make yotta-build
```
Génère : `build/bbc-microbit-classic-gcc/source/microbit-samples-combined.hex`

#### 2. Flash du firmware

**macOS :**
```bash
cp out/iot-terrain-microbit.hex /Volumes/MICROBIT/
```

**Linux :**
```bash
cp out/iot-terrain-microbit.hex /media/$USER/MICROBIT/
```

**Windows :**
Copier le fichier `.hex` sur le lecteur `MICROBIT:` via l'explorateur

#### 3. Configuration du simulateur Java

1. Identifier le port série :
   - **Linux** : `/dev/ttyACM0` ou `/dev/ttyUSB0`
   - **macOS** : `/dev/tty.usbmodem*`
   - **Windows** : `COM3`, `COM4`, etc.

2. Configurer le baudrate : **115200 bps**

3. Lancer la connexion série

#### 4. Vérification

1. Le pixel (4,4) doit être allumé en permanence
2. Envoyer une trame de test via le simulateur :
   ```csv
   vehicle_position,TEST001,48.856614,2.352222,1736172600
   ```
3. Observer l'affichage : **"T"** = succès, **"!"** = échec

---

## 🏗️ Architecture logicielle

### Structure du code

```cpp
// Point d'entrée
int main() {
    // 1. Initialisation MicroBit
    uBit.init();
    
    // 2. Configuration UART (115200 bps, buffer 254 octets)
    uBit.serial.baud(115200);
    uBit.serial.setRxBufferSize(254);
    
    // 3. Initialisation SDMIS Radio
    sdmis_radio_init(&uBit, CPE_KEY, RADIO_GROUP, RADIO_POWER);
    
    // 4. Boucle principale
    while (true) {
        // Indicateur système actif
        uBit.display.image.setPixelValue(4, 4, 255);
        
        // Traitement UART → Radio
        check_uart();
        
        // Traitement Radio → UART
        sdmis_frame_t rx;
        while (sdmis_radio_poll(&rx)) {
            if (rx.type == CPE_FT_INCIDENT_AFFECT) {
                send_csv("vehicle_affectation", rx.immat, 
                        rx.lat_e6, rx.lon_e6, rx.timestamp);
                uBit.display.print("A");
            }
        }
        
        // Attente courte pour économiser CPU
        uBit.sleep(10);
    }
}
```

### Fonctions principales

#### `check_uart()`

**Rôle** : Lecture non-bloquante des données UART

```cpp
static void check_uart() {
    while (uBit.serial.rxBufferedSize() > 0) {
        char c = (char)uBit.serial.read(ASYNC);
        
        if (c == '\n') {
            // Ligne complète reçue
            uart_buf[uart_idx] = '\0';
            process_uart_line(uart_buf);
            uart_idx = 0;
        } else if (c != '\r' && uart_idx < sizeof(uart_buf) - 1) {
            // Accumulation caractère par caractère
            uart_buf[uart_idx++] = c;
        }
    }
}
```

**Caractéristiques :**
- Non-bloquant (traite tous les caractères disponibles)
- Buffer circulaire de 256 octets
- Détection fin de ligne (`\n`)
- Ignore les retours chariot (`\r`)

#### `process_uart_line()`

**Rôle** : Analyse et traitement d'une ligne CSV complète

```cpp
static void process_uart_line(char *line) {
    char event[32], immat[CPE_IMMAT_LEN];
    int32_t lat_e6, lon_e6;
    uint32_t ts;
    
    // Parsing CSV
    if (!parse_csv(line, event, immat, &lat_e6, &lon_e6, &ts)) 
        return;
    
    // Traitement selon l'événement
    if (strcmp(event, "vehicle_position") == 0) {
        bool ok = sdmis_radio_send_position(immat, lat_e6, lon_e6, 0x01, ts);
        uBit.display.print(ok ? "T" : "!");
    }
}
```

#### `send_csv()`

**Rôle** : Formatage et envoi CSV sur UART

```cpp
static void send_csv(const char *event, const char *immat, 
                    int32_t lat_e6, int32_t lon_e6, uint32_t ts) {
    // Conversion micro-degrés → degrés décimaux
    int32_t lat_deg = lat_e6 / 1000000, lat_dec = lat_e6 % 1000000;
    int32_t lon_deg = lon_e6 / 1000000, lon_dec = lon_e6 % 1000000;
    
    // Gestion des valeurs négatives
    if (lat_dec < 0) lat_dec = -lat_dec;
    if (lon_dec < 0) lon_dec = -lon_dec;
    
    // Construction et envoi
    char csv[100];
    snprintf(csv, sizeof(csv), "%s,%s,%ld.%06ld,%ld.%06ld,%lu\n",
             event, immat, lat_deg, lat_dec, lon_deg, lon_dec, ts);
    uBit.serial.send(csv);
}
```

---

## 📈 Performances

### Latences typiques

| Opération | Latence | Notes |
|-----------|---------|-------|
| Réception UART | < 10 ms | Dépend de la taille du message |
| Parsing CSV | < 1 ms | Traitement léger |
| Chiffrement CPE | ~5 ms | AES-128 CTR |
| Transmission radio | ~10 ms | Sans attente ACK |
| Attente ACK (succès) | 10-30 ms | Temps de traitement du récepteur |
| Attente ACK (timeout) | 200 ms | Par tentative |
| **Latence totale (succès)** | **30-50 ms** | UART → Radio avec ACK |
| **Latence totale (3 échecs)** | **~650 ms** | Avec 3 timeouts + backoffs |

### Débit maximal

| Direction | Débit théorique | Débit pratique |
|-----------|-----------------|----------------|
| UART → Radio | ~19 messages/s | ~15 messages/s |
| Radio → UART | Illimité en réception | ~10 messages/s |

**Facteurs limitants :**
- Attente ACK (200 ms par message)
- Temps de chiffrement/déchiffrement
- Backoff en cas de retransmission

### Consommation électrique

| Mode | Courant | Description |
|------|---------|-------------|
| Actif (RX+TX) | ~13 mA | Radio en écoute + transmission |
| Actif (RX seul) | ~11 mA | Radio en écoute |
| Sleep (non utilisé) | ~1 µA | Radio désactivée |

**Autonomie typique :**
- Batterie 3V CR2032 (220 mAh) : ~17 heures en continu
- Batterie USB (5V, 1000 mAh) : ~70 heures en continu

---

## 🐛 Dépannage

### Problème : Aucun message reçu du simulateur

**Symptômes :**
- Pas d'affichage "T" ou "!" sur la micro:bit
- Le pixel (4,4) est allumé

**Vérifications :**
1. ✓ Port série correct (`/dev/ttyACM0`, `COM3`, etc.)
2. ✓ Baudrate = 115200 bps
3. ✓ Format CSV correct (5 champs séparés par virgules)
4. ✓ Terminaison par `\n` (line feed)
5. ✓ Simulateur connecté et actif

**Test manuel :**
```bash
# Linux/macOS
echo "vehicle_position,TEST001,48.856614,2.352222,1736172600" > /dev/ttyACM0
```

### Problème : Transmission radio échoue (affichage "!")

**Symptômes :**
- Réception UART OK
- Affichage "!" systématique
- Pas d'ACK reçu

**Causes possibles :**
1. Aucun récepteur à portée
2. Clé cryptographique différente
3. Groupe radio différent (doit être 42)
4. Interférences radio (WiFi 2.4 GHz, micro-ondes)
5. Portée dépassée (> 150 m)

**Solutions :**
1. Vérifier qu'un récepteur est allumé et à portée
2. Re-flasher les firmwares avec la même clé
3. Vérifier `RADIO_GROUP = 42`
4. Réduire la distance émetteur/récepteur
5. Augmenter la puissance radio (`RADIO_POWER = 7`)

### Problème : Messages en double côté Java

**Symptômes :**
- Même message `vehicle_affectation` reçu plusieurs fois
- Timestamps identiques

**Cause :**
- Retransmissions radio légitimes (duplicatas)
- Filtre anti-duplication limité

**Solution :**
Implémenter un filtre côté Java :
```java
Map<String, Long> lastReceived = new HashMap<>();

if (lastReceived.get(immat) != null && 
    lastReceived.get(immat) == timestamp) {
    // Duplicata, ignorer
    return;
}
lastReceived.put(immat, timestamp);
```

### Problème : Buffer UART overflow

**Symptômes :**
- Messages tronqués ou perdus
- Comportement erratique

**Cause :**
- Débit UART trop élevé
- Traitement trop lent

**Solutions :**
1. Réduire la fréquence d'envoi côté simulateur
2. Augmenter `sleep(10)` dans la boucle principale
3. Vérifier que le buffer RX = 254 octets max

---

## 🔒 Recommandations de sécurité

### Distribution de la clé

⚠️ **CRITIQUE** : La clé AES-128 doit être protégée :

**À FAIRE :**
- ✅ Changer la clé par défaut en production
- ✅ Utiliser un générateur cryptographiquement sûr
- ✅ Distribuer la clé de manière sécurisée (hors bande)
- ✅ Changer régulièrement la clé (rotation mensuelle)
- ✅ Une clé différente par réseau/mission

**À NE PAS FAIRE :**
- ❌ Laisser la clé d'exemple en production
- ❌ Transmettre la clé en clair par email/chat
- ❌ Réutiliser la même clé pour plusieurs missions
- ❌ Publier le firmware avec la clé sur GitHub

### Authentification des messages

**Limitation actuelle :**
- Le système n'a pas d'authentification forte
- Un attaquant avec la clé peut injecter des messages

**Améliorations possibles :**
1. Utiliser AES-GCM au lieu de AES-CTR + CRC
2. Ajouter un HMAC-SHA256 sur chaque message
3. Implémenter un système de clés de session

### Protection contre le rejeu

**Protection actuelle :**
- Nonce unique par message
- Détection des duplicatas immédiats

**Amélioration recommandée :**
```c
// Vérifier la fraîcheur du timestamp
uint32_t now = (uint32_t)(uBit.systemTime() / 1000);
if (abs(now - frame.timestamp) > 60) {
    // Message trop ancien ou futur, rejeter
    return;
}
```

---

## 📚 Fichiers source

| Fichier | Rôle |
|---------|------|
| [source/main.cpp](../source/main.cpp) | Point d'entrée et boucle principale |
| [source/lib/sdmis_radio.h](../source/lib/sdmis_radio.h) | API de la librairie SDMIS |
| [source/lib/sdmis_radio.cpp](../source/lib/sdmis_radio.cpp) | Implémentation SDMIS |
| [source/proto/cpe/cpe.h](../source/proto/cpe/cpe.h) | API du protocole CPE |
| [source/proto/cpe/cpe.c](../source/proto/cpe/cpe.c) | Implémentation CPE |

---

## 🚀 Améliorations futures

### Court terme
- [ ] Mode debug activable par bouton (echo UART)
- [ ] Compteur de messages transmis/reçus
- [ ] Affichage du taux de perte sur l'écran
- [ ] Support des messages `vehicle_status`

### Moyen terme
- [ ] Buffer circulaire pour mémoriser N duplicatas
- [ ] Validation timestamp (rejet messages trop anciens)
- [ ] Compression des coordonnées GPS
- [ ] Mode économie d'énergie (sleep radio)

### Long terme
- [ ] Chiffrement authentifié (AES-GCM)
- [ ] Négociation de clés dynamique (Diffie-Hellman)
- [ ] Support multi-canal radio (frequency hopping)
- [ ] Interface configuration via Bluetooth

---

## 📄 Licence

Voir le fichier [LICENSE](../LICENSE) à la racine du projet.

---

## 📞 Support

- **Projet** : IoT Terrain Micro:bit
- **Date** : Janvier 2026
- **Documentation associée** :
  - [Protocole CPE](PROTOCOLE_CPE.md)
  - [Librairie SDMIS_RADIO](SDMIS_RADIO.md)
