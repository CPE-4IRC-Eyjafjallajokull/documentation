# Application Terrain Micro:bit

Micro:bit embarqué dans les véhicules d'intervention pour transmettre les positions GPS et statuts par radio.

---

## 🎯 Rôle et objectifs

L'application terrain transforme une carte BBC Micro:bit en **passerelle radio** embarquée dans un véhicule d'intervention. Elle :

- 📡 **Reçoit via UART** : Positions GPS et statuts depuis un système embarqué (tablette, GPS)
- 📻 **Émet par radio** : Transmet les données chiffrées au centre de commandement (trame CPE 29 octets)
- 📥 **Reçoit par radio** : Affectations d'incidents depuis la centrale
- 💻 **Renvoie via UART** : Transmet les affectations au système embarqué

---

## ⚙️ Configuration technique

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| **Plateforme** | BBC Micro:bit v1 | Basé sur nRF51822 (ARM Cortex-M0) |
| **UART** | 115200 baud | Communication avec système embarqué |
| **Buffer UART RX** | 254 octets | Taille max buffer réception |
| **Groupe radio** | 42 | Identifiant réseau SDMIS |
| **Puissance radio** | 7 (maximum) | Portée ~150 m extérieur |
| **Chiffrement** | AES-128 CTR | Avec clé pré-partagée 16 octets |
| **Protocole CPE** | 29 octets | Header 9 + payload chiffré 20 octets |

### Clé cryptographique

```c
static const uint8_t CPE_KEY[16] = {
    0x21, 0x53, 0xB6, 0x09, 0x9A, 0xD2, 0x41, 0x7C,
    0xE4, 0x10, 0x5F, 0x3A, 0x77, 0xC8, 0x90, 0x0B
};
```

⚠️ **IMPORTANT** : Cette clé doit être **identique** sur :
- Toutes les cartes terrain
- La passerelle UART-Radio
- La passerelle RF centrale

---

## 🏗️ Architecture logicielle

```
┌─────────────────────────────────────────────────────────┐
│       Système embarqué (Tablette, GPS, Ordi)           │
│  Envoie CSV: vehicle_position,status,immat,lat,lon,ts  │
│          ou: vehicle_status,status,immat,0,0,ts         │
└───────────────────────┬─────────────────────────────────┘
                        │ UART 115200 baud
                        ↓
┌─────────────────────────────────────────────────────────┐
│            Application Terrain (main.cpp)               │
├─────────────────────────────────────────────────────────┤
│  • Lecture UART caractère par caractère                 │
│  • Buffer jusqu'à '\n' (ligne complète)                 │
│  • Parse CSV → appel SDMIS selon événement              │
│  • Poll radio → envoi UART si affectation               │
│  • Boucle 10 ms + pixel (4,4) fixe = système actif     │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│                Librairie SDMIS_RADIO                    │
├─────────────────────────────────────────────────────────┤
│  • sdmis_radio_send_position() → LED "T" ou "!"         │
│  • sdmis_radio_send_status() → LED "S" ou "!"           │
│  • sdmis_radio_poll() → détecte affectations            │
│  • Gestion ACK/Retry auto (3 × 200ms timeout)           │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│                   Protocole CPE                         │
├─────────────────────────────────────────────────────────┤
│  • Trame fixe 29 octets (header 9 + payload 20)        │
│  • Chiffrement AES-128 CTR                              │
│  • CRC-16 CCITT pour intégrité                          │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────┐
│              MicroBit Radio (2.4 GHz)                   │
└─────────────────────────────────────────────────────────┘
```

---

## 📱 Interface utilisateur

### Écran LED 5×5

| Affichage | Signification | Contexte |
|-----------|---------------|----------|
| Pixel (4,4) allumé | Système actif | Permanent (boucle principale) |
| **T** | Position transmise avec succès | ACK reçu après vehicle_position |
| **S** | Statut transmis avec succès | ACK reçu après vehicle_status |
| **!** | Échec de transmission | Pas d'ACK après 3 tentatives |
| **A** | Affectation d'incident reçue | Trame INCIDENT_AFFECT déchiffrée |

**Interface** : Pas de boutons. Contrôle 100% via UART (tablette, GPS, ordi embarqué).

---

## 🔄 Fonctionnement

### Format CSV sur UART

**Structure** : `event,status,immat,lat,lon,timestamp`

| Champ | Type | Format | Exemple |
|-------|------|--------|----------|
| event | String | `vehicle_position` ou `vehicle_status` | `vehicle_position` |
| status | uint8 | 0=Disponible, 1=Engagé, 2=Sur intervention, 3=Transport, 4=Retour, 5=Indisponible, 6=Hors service | `1` |
| immat | String | 8 caractères max | `AB123CD` |
| lat | Float | Degrés décimaux (6 décimales) | `48.856614` |
| lon | Float | Degrés décimaux (6 décimales) | `2.352222` |
| timestamp | uint32 | Secondes depuis epoch Unix | `1736172600` |

### 1️⃣ Envoi de position (UART → Radio)

**Réception UART :**
```csv
vehicle_position,1,AB123CD,48.856614,2.352222,1736172600
```

**Flux :**
```
┌──────────────────────────────────────────────────────────┐
│  1. Réception UART caractère par caractère               │
│     └─ Buffer jusqu'à '\n'                               │
├──────────────────────────────────────────────────────────┤
│  2. Parse CSV: event, status, immat, lat, lon, ts       │
│     └─ Vérifie event == "vehicle_position"               │
├──────────────────────────────────────────────────────────┤
│  3. Conversion: 48.856614 → 48856614 (×10⁶)             │
├──────────────────────────────────────────────────────────┤
│  4. Appel sdmis_radio_send_position()                    │
│     ├─ Génère nonce et séquence automatiquement         │
│     ├─ Construit trame CPE 29 octets                    │
│     ├─ Chiffre AES-128 CTR + CRC-16                     │
│     └─ Envoie avec attente ACK (3 tentatives)           │
├──────────────────────────────────────────────────────────┤
│  5. Affichage LED: "T" si ACK, "!" si échec              │
└──────────────────────────────────────────────────────────┘
```

### 2️⃣ Envoi de statut (UART → Radio)

**Réception UART :**
```csv
vehicle_status,2,AB123CD,0,0,1736172600
```
_(status 2 = Sur intervention)_

**Flux :**
```
┌──────────────────────────────────────────────────────────┐
│  1. Parse CSV: event, status, immat, _, _, ts           │
│     └─ Vérifie event == "vehicle_status"                 │
├──────────────────────────────────────────────────────────┤
│  2. Appel sdmis_radio_send_status()                      │
│     └─ Trame CPE sans coordonnées (lat=0, lon=0)        │
├──────────────────────────────────────────────────────────┤
│  3. Affichage LED: "S" si ACK, "!" si échec              │
└──────────────────────────────────────────────────────────┘
```

**Codes de statut :**
| Code | Signification |
|------|---------------|
| 0 | Disponible |
| 1 | Engagé |
| 2 | Sur intervention |
| 3 | Transport |
| 4 | Retour |
| 5 | Indisponible |
| 6 | Hors service |

### 3️⃣ Réception d'affectation (Radio → UART)

**Flux :**
```
┌──────────────────────────────────────────────────────────┐
│  1. sdmis_radio_poll() détecte trame radio              │
├──────────────────────────────────────────────────────────┤
│  2. Déchiffrement et validation automatiques             │
│     └─ ACK envoyé immédiatement                          │
├──────────────────────────────────────────────────────────┤
│  3. Vérifie type == CPE_FT_INCIDENT_AFFECT               │
├──────────────────────────────────────────────────────────┤
│  4. Extraction: immat, status, lat_e6, lon_e6, ts       │
├──────────────────────────────────────────────────────────┤
│  5. Conversion: 48856614 → 48.856614 (÷10⁶)             │
├──────────────────────────────────────────────────────────┤
│  6. Construction CSV et envoi UART                       │
├──────────────────────────────────────────────────────────┤
│  7. Affichage LED "A"                                    │
└──────────────────────────────────────────────────────────┘
```

**Envoi UART :**
```csv
vehicle_affectation,0,SD304FR,45.797200,4.847000,1736172600
│     ├─ Latitude de l'incident                           │
│     ├─ Longitude de l'incident                          │
│     └─ Timestamp                                        │
├─────────────────────────────────────────────────────────┤
│  5. Affichage:                                          │
│     ├─ Icône → (affectation)                            │
│     ├─ Scroll de l'immatriculation                      │
│     └─ Vibration (si disponible)                        │
├─────────────────────────────────────────────────────────┤
│  6. Changement automatique statut = "En route"          │
└─────────────────────────────────────────────────────────┘
```

---

## 🔌 Intégration GPS

### Module GPS externe (recommandé)

**Connexions :**
| Pin Micro:bit | Pin GPS | Signal |
|---------------|---------|--------|
| P0 | TX | UART RX |
| P1 | RX | UART TX |
| GND | GND | Masse commune |
| 3V | VCC | Alimentation |

**Protocole NMEA :**
```c
// Parsing trame $GPGGA
void parse_gps_nmea(const char *nmea) {
    // Extraction latitude/longitude
    // Format: $GPGGA,hhmmss,DDMM.MMMM,N,DDDMM.MMMM,E,...
    // Conversion en degrés décimaux
}
```

### Mode simulation (sans GPS)

Pour les tests ou démonstrations sans module GPS :

```cpp
// Position fixe simulée (Tour Eiffel)
int32_t sim_lat = 48856614;  // 48.856614° N
int32_t sim_lon = 2352222;   // 2.352222° E

// Ou mouvement aléatoire autour d'un point
int32_t lat = base_lat + uBit.random(-5000, 5000);  // ±5 km
int32_t lon = base_lon + uBit.random(-5000, 5000);
```

---

## 💻 Code source type

### Structure minimale

```cpp
#include "MicroBit.h"
#include "lib/sdmis_radio.h"

MicroBit uBit;

// Configuration
static const uint8_t RADIO_GROUP = 42;
static const uint8_t RADIO_POWER = 7;
static const uint8_t CPE_KEY[16] = { /* ... */ };

// État du véhicule
static char MY_IMMAT[] = "VSAV304";
static uint8_t current_status = 1;  // Disponible

int main() {
    uBit.init();
    
    // Initialisation SDMIS Radio
    sdmis_radio_init(&uBit, CPE_KEY, RADIO_GROUP, RADIO_POWER);
    
    uint32_t last_send = 0;
    const uint32_t SEND_INTERVAL = 5000;  // 5 secondes
    
    while (true) {
        // Indicateur système actif
        uBit.display.image.setPixelValue(4, 4, 255);
        
        // === ENVOI PÉRIODIQUE ===
        uint32_t now = (uint32_t)uBit.systemTime();
        if (now - last_send >= SEND_INTERVAL) {
            // Acquisition GPS (ou simulation)
            int32_t lat = get_gps_lat();
            int32_t lon = get_gps_lon();
            uint32_t ts = now / 1000;
            
            // Envoi position
            bool ok = sdmis_radio_send_position(
                MY_IMMAT, lat, lon, current_status, ts
            );
            
            uBit.display.print(ok ? "T" : "!");
            last_send = now;
        }
        
        // === BOUTON A : CHANGEMENT STATUT ===
        if (uBit.buttonA.isPressed()) {
            current_status = (current_status % 5) + 1;
            uint32_t ts = (uint32_t)(uBit.systemTime() / 1000);
            
            bool ok = sdmis_radio_send_status(MY_IMMAT, current_status, ts);
            uBit.display.print(ok ? "T" : "!");
            
            // Affichage icône
            display_status_icon(current_status);
            
            uBit.sleep(500);  // Debounce
        }
        
        // === BOUTON B : ÉVÉNEMENT TERRAIN ===
        if (uBit.buttonB.isPressed()) {
            int32_t lat = get_gps_lat();
            int32_t lon = get_gps_lon();
            uint32_t ts = (uint32_t)(uBit.systemTime() / 1000);
            
            // Statut spécial "arrivée sur site"
            bool ok = sdmis_radio_send_position(
                MY_IMMAT, lat, lon, 10, ts
            );
            
            uBit.display.print(ok ? "T" : "!");
            uBit.sleep(500);  // Debounce
        }
        
        // === RÉCEPTION AFFECTATIONS ===
        sdmis_frame_t rx;
        while (sdmis_radio_poll(&rx)) {
            if (rx.type == CPE_FT_INCIDENT_AFFECT) {
                // Vérifier que c'est pour nous
                if (strcmp(rx.immat, MY_IMMAT) == 0) {
                    uBit.display.print(">");
                    
                    // Passer en statut "En route"
                    current_status = 2;
                    
                    // Optionnel: Afficher coordonnées incident
                    // display_coordinates(rx.lat_e6, rx.lon_e6);
                }
            }
        }
        
        uBit.sleep(10);  // 10 ms
    }
}
```

---

## 🔨 Compilation et déploiement

### Prérequis

- **GNU Arm Embedded Toolchain** (`arm-none-eabi-*`)
- **CMake** 3.6+
- **Python** 3.x
- **Docker** (recommandé pour build) ou **yotta** installé localement

### Étapes de build

#### Option A : Build avec Docker (recommandé)

```bash
cd iot-terrain-microbit
make clean && make build
```

**Sortie :**
```
out/iot-terrain-microbit.hex
```

#### Option B : Build avec yotta local

```bash
cd iot-terrain-microbit

# Si yotta n'est pas activé
source /path/to/yotta/bin/activate

# Build
make yotta-build
```

**Sortie :**
```
build/bbc-microbit-classic-gcc/source/microbit-samples-combined.hex
```

### Flash du firmware

#### macOS

```bash
cp out/iot-terrain-microbit.hex /Volumes/MICROBIT/
```

#### Linux

```bash
cp out/iot-terrain-microbit.hex /media/$USER/MICROBIT/
```

#### Windows

Glisser-déposer le fichier `.hex` sur le lecteur `MICROBIT:` dans l'explorateur.

### Personnalisation par véhicule

**Modifier l'immatriculation avant compilation :**

```cpp
// source/main.cpp
static char MY_IMMAT[] = "VSAV304";  // Changer ici
```

**Ou via #define :**

```bash
```

---

## 📁 Arborescence du projet

```
iot-terrain-microbit/
├── source/
│   ├── main.cpp                # Point d'entrée application terrain
│   ├── lib/
│   │   ├── sdmis_radio.h       # API librairie SDMIS
│   │   └── sdmis_radio.cpp     # Implémentation SDMIS
│   ├── proto/
│   │   └── cpe/
│   │       ├── cpe.h           # API protocole CPE
│   │       └── cpe.c           # Implémentation CPE
│   └── crypto/
│       └── tinycrypt/          # Bibliothèque AES-128
│           ├── aes_encrypt.c
│           ├── cmac_mode.c
│           ├── ctr_mode.c
│           └── utils.c
├── build.py                    # Script de compilation
├── Makefile                    # Cibles de build
├── module.json                 # Configuration yotta
├── config.json                 # Configuration micro:bit
└── out/
    └── iot-terrain-microbit.hex  # Firmware compilé
```

---

## 🧪 Tests et validation

### Test 1 : Transmission basique

**Objectif** : Vérifier que les messages sont envoyés

1. Flasher le firmware
2. Placer un récepteur à portée (passerelle ou autre terrain)
3. Alimenter la carte
4. Observer affichage périodique de "✓"

**Résultat attendu :** ✓ toutes les 5 secondes

### Test 2 : Boutons

**Objectif** : Valider les interactions utilisateur

1. Appuyer sur bouton A
2. Observer affichage ✓ puis icône de statut
3. Appuyer sur bouton B
4. Observer affichage ✓ immédiat

**Résultat attendu :** Réponse immédiate aux appuis

### Test 3 : Réception affectation

**Objectif** : Tester le flux incident → terrain

1. Configurer une passerelle UART-Radio
2. Envoyer via Java : `vehicle_affectation,VSAV304,45.797200,4.847000,1736172600`
3. Observer affichage "→" sur la carte terrain

**Résultat attendu :** Affichage de l'affectation

### Test 4 : Portée radio

**Objectif** : Mesurer la portée effective

1. Position émetteur fixe
2. Éloigner progressivement
3. Noter distance max avec ✓
4. Tester avec obstacles (murs, véhicules)

**Résultats typiques :**
- Intérieur : 30-50 m
- Extérieur dégagé : 100-150 m
- Urbain dense : 50-80 m

---

## 📊 Performances

### Consommation électrique

| Mode | Courant | Autonomie (CR2032 220mAh) |
|------|---------|---------------------------|
| **TX périodique (5s)** | Moy. 13 mA | ~17 heures |
| **TX + RX actif** | Moy. 15 mA | ~15 heures |
| **Sleep (non impl.)** | ~1 µA | ~25 000 heures |

**Optimisations possibles :**
- Augmenter intervalle d'envoi (10s, 30s)
- Désactiver radio entre transmissions
- Utiliser mode low-power du nRF51

### Latence de transmission

| Opération | Latence |
|-----------|---------|
| Bouton → Transmission | 20-50 ms |
| Acquisition GPS | 100-1000 ms |
| Attente ACK (succès) | 10-30 ms |
| Attente ACK (échec 3×) | ~650 ms |

---

## 🐛 Dépannage

### Problème : Affichage ✗ permanent

**Causes possibles :**
1. ✗ Aucun récepteur à portée
2. ✗ Clé cryptographique différente
3. ✗ Groupe radio différent
4. ✗ Interférences fortes

**Solutions :**
1. ✓ Vérifier récepteur allumé et à portée (< 100 m)
2. ✓ Re-flasher avec même clé que récepteur
3. ✓ Vérifier `RADIO_GROUP = 42` partout
4. ✓ Éloigner sources WiFi/Bluetooth

### Problème : Pas d'affectation reçue

**Causes possibles :**
1. ✗ Immatriculation incorrecte
2. ✗ Groupe radio différent
3. ✗ Boucle poll() pas appelée

**Solutions :**
1. ✓ Vérifier `MY_IMMAT` correspond au message
2. ✓ Vérifier groupe = 42
3. ✓ Assurer `sdmis_radio_poll()` dans loop

### Problème : LED (4,4) éteinte

**Cause :** Firmware non démarré

**Solutions :**
1. Re-flasher le firmware
2. Vérifier alimentation (> 3V)
3. Appuyer sur bouton RESET au dos

---

## 🔒 Sécurité terrain

### Recommandations opérationnelles

| Aspect | Recommandation |
|--------|----------------|
| **Protection physique** | Boîtier étanche IP65+ |
| **Fixation** | Montage antivibration |
| **Alimentation** | Batterie + USB backup |
| **Visibilité** | Indicateur LED externe |
| **Accessibilité** | Boutons accessibles conducteur |

### Confidentialité

⚠️ **Les positions GPS sont chiffrées mais :**
- La clé doit rester confidentielle
- Ne pas flasher les cartes en présence de tiers
- Changer la clé entre missions sensibles

---

## 🚀 Améliorations futures

### Court terme
- [ ] Support GPS NMEA complet
- [ ] Messages opérateur libres (texte court)
- [ ] Affichage distance vers incident
- [ ] Boussole/cap vers destination

### Moyen terme
- [ ] Mode économie d'énergie intelligent
- [ ] Enregistrement positions hors ligne
- [ ] Interface configuration Bluetooth
- [ ] Support accéléromètre (détection choc)

### Long terme
- [ ] Micro:bit v2 (plus de RAM/Flash)
- [ ] écran OLED externe
- [ ] Intégration CAN Bus véhicule
- [ ] Géofencing automatique

---

## 📚 Documentation associée

| Document | Lien |
|----------|------|
| Protocole CPE | [PROTOCOLE_CPE.md](PROTOCOLE_CPE.md) |
| Librairie SDMIS | [SDMIS_RADIO.md](SDMIS_RADIO.md) |
| Passerelle UART | [PASSERELLE_UART_RADIO.md](PASSERELLE_UART_RADIO.md) |
| Système complet | [SYSTEME_COMPLET.md](SYSTEME_COMPLET.md) |

---

## 📄 Licence

Voir [LICENSE](../LICENSE) à la racine du projet.

---

**Projet** : IoT Terrain Micro:bit  
**Composant** : Application Terrain (Émetteur)  
**Version** : 1.0  
**Date** : Janvier 2026
