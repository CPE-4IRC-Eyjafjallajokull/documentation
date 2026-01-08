# Librairie SDMIS_RADIO - Documentation Complète

## Vue d'ensemble

La librairie **SDMIS_RADIO** est une couche d'abstraction haut niveau pour la communication radio sur BBC micro:bit. Elle encapsule le protocole CPE et fournit une interface simple pour l'envoi et la réception de messages sécurisés entre unités sur le terrain.

### Caractéristiques principales

- **Fiabilité** : Mécanisme d'acquittement (ACK) avec retransmission automatique
- **Sécurité** : Intégration transparente du protocole CPE (chiffrement AES-128)
- **Simplicité** : API haut niveau pour envoyer/recevoir des messages
- **Gestion automatique** : Nonce, séquence, déduplication
- **Non-bloquant** : Reception par polling

---

## Architecture

```
┌─────────────────────────────────────────┐
│         Application micro:bit           │
└───────────────┬─────────────────────────┘
                │
                ↓
┌─────────────────────────────────────────┐
│        SDMIS_RADIO (couche haut)        │
│  • Gestion ACK/Retry                    │
│  • Anti-duplication                     │
│  • Gestion nonce/séquence               │
└───────────────┬─────────────────────────┘
                │
                ↓
┌─────────────────────────────────────────┐
│      Protocole CPE (cryptographie)      │
│  • Chiffrement AES-128                  │
│  • Validation CRC-16                    │
└───────────────┬─────────────────────────┘
                │
                ↓
┌─────────────────────────────────────────┐
│    MicroBit Radio (couche physique)     │
│  • Transmission 2.4 GHz                 │
└─────────────────────────────────────────┘
```

---

## Structure de données

### `sdmis_frame_t`

Structure représentant une trame SDMIS complète après déchiffrement :

```cpp
typedef struct {
    cpe_frame_type_t type;      // Type de trame (VEH_POS, VEH_STATUS, INCIDENT)
    char immat[CPE_IMMAT_LEN + 1]; // Immatriculation (9 octets avec \0)
    uint8_t status;             // Code de statut
    int32_t lat_e6;             // Latitude × 10⁶
    int32_t lon_e6;             // Longitude × 10⁶
    uint32_t nonce;             // Nonce de la trame
    uint32_t timestamp;         // Timestamp Unix
    uint8_t seq;                // Numéro de séquence
} sdmis_frame_t;
```

**Champs** :
- `type` : Un des trois types de trame CPE
- `immat` : Identifiant du véhicule (terminé par `\0`)
- `status` : Code de statut applicatif
- `lat_e6` / `lon_e6` : Coordonnées GPS en micro-degrés
- `nonce` : Valeur unique de la trame
- `timestamp` : Horodatage de la trame
- `seq` : Numéro de séquence

---

## API de programmation

### Initialisation

```cpp
void sdmis_radio_init(
    MicroBit *uBit,
    const uint8_t key[CPE_KEY_LEN],
    uint8_t radio_group,
    uint8_t tx_power
);
```

**Paramètres** :
- `uBit` : Pointeur vers l'instance MicroBit
- `key` : Clé de chiffrement AES-128 (16 octets)
- `radio_group` : Groupe radio (0-255) pour segmenter le réseau
- `tx_power` : Puissance d'émission (0-7, voir documentation micro:bit)

**Description** :
- Initialise le protocole CPE avec la clé fournie
- Configure la radio micro:bit
- Initialise le générateur de nonces avec une seed aléatoire
- Réinitialise les compteurs de séquence
- Vide les buffers de réception

**Exemple** :
```cpp
MicroBit uBit;
uint8_t key[16] = {0x01, 0x02, ..., 0x10};

sdmis_radio_init(&uBit, key, 42, 6);
```

---

### Réception de trames

```cpp
bool sdmis_radio_poll(sdmis_frame_t *out_frame);
```

**Paramètres** :
- `out_frame` : Pointeur vers une structure pour stocker la trame reçue

**Retour** :
- `true` : Une nouvelle trame a été reçue et stockée dans `out_frame`
- `false` : Aucune nouvelle trame disponible

**Description** :
- Fonction non-bloquante
- Doit être appelée régulièrement (dans la boucle principale)
- Traite jusqu'à 8 paquets radio par appel
- Filtre automatiquement les duplicatas
- Envoie automatiquement les ACK
- Ne retourne qu'une seule trame à la fois

**Comportement** :
1. Poll les paquets radio disponibles
2. Déchiffre et valide les trames CPE
3. Envoie un ACK immédiatement
4. Rejette les duplicatas (même seq + nonce)
5. Stocke la première nouvelle trame valide
6. Retourne `true` si une trame est disponible

**Exemple** :
```cpp
sdmis_frame_t frame;

while (true) {
    if (sdmis_radio_poll(&frame)) {
        // Nouvelle trame reçue
        if (frame.type == CPE_FT_VEH_POS) {
            double lat = frame.lat_e6 / 1000000.0;
            double lon = frame.lon_e6 / 1000000.0;
            uBit.display.scroll(frame.immat);
        }
    }
    uBit.sleep(50); // Attente entre les polls
}
```

---

### Envoi de position

```cpp
bool sdmis_radio_send_position(
    const char immat[CPE_IMMAT_LEN],
    int32_t lat_e6,
    int32_t lon_e6,
    uint8_t status,
    uint32_t timestamp
);
```

**Paramètres** :
- `immat` : Immatriculation du véhicule (8 caractères max)
- `lat_e6` : Latitude × 10⁶
- `lon_e6` : Longitude × 10⁶
- `status` : Code de statut
- `timestamp` : Timestamp Unix (secondes)

**Retour** :
- `true` : Trame envoyée avec succès (ACK reçu)
- `false` : Échec après 3 tentatives (pas d'ACK)

**Description** :
- Construit une trame CPE de type `CPE_FT_VEH_POS`
- Génère automatiquement le nonce et la séquence
- Envoie avec mécanisme de retry (jusqu'à 3 tentatives)
- Fonction **bloquante** (attend l'ACK ou timeout)

**Exemple** :
```cpp
uint32_t now = (uint32_t)uBit.systemTime() / 1000;
bool ok = sdmis_radio_send_position(
    "VEH00042",
    48856600,  // Paris
    2352200,
    5,         // Statut "en mission"
    now
);

if (!ok) {
    uBit.display.print("X"); // Échec d'envoi
}
```

---

### Envoi de statut

```cpp
bool sdmis_radio_send_status(
    const char immat[CPE_IMMAT_LEN],
    uint8_t status,
    uint32_t timestamp
);
```

**Paramètres** :
- `immat` : Immatriculation du véhicule (8 caractères max)
- `status` : Nouveau code de statut
- `timestamp` : Timestamp Unix (secondes)

**Retour** :
- `true` : Trame envoyée avec succès (ACK reçu)
- `false` : Échec après 3 tentatives

**Description** :
- Construit une trame CPE de type `CPE_FT_VEH_STATUS`
- N'inclut pas de coordonnées GPS
- Plus léger qu'un envoi de position
- Même mécanisme de fiabilité (ACK/retry)

**Cas d'usage** :
- Changement de statut sans déplacement
- Économie de batterie (pas de GPS)
- Mise à jour rapide de disponibilité

**Exemple** :
```cpp
uint32_t now = (uint32_t)uBit.systemTime() / 1000;

// Bouton A : Disponible
if (uBit.buttonA.isPressed()) {
    sdmis_radio_send_status("VEH00042", 1, now);
}

// Bouton B : Occupé
if (uBit.buttonB.isPressed()) {
    sdmis_radio_send_status("VEH00042", 5, now);
}
```

---

### Envoi d'affectation à un incident

```cpp
bool sdmis_radio_send_incident_affect(
    const char immat[CPE_IMMAT_LEN],
    int32_t lat_e6,
    int32_t lon_e6,
    uint32_t timestamp
);
```

**Paramètres** :
- `immat` : Immatriculation du véhicule affecté
- `lat_e6` : Latitude de l'incident × 10⁶
- `lon_e6` : Longitude de l'incident × 10⁶
- `timestamp` : Timestamp Unix (secondes)

**Retour** :
- `true` : Trame envoyée avec succès (ACK reçu)
- `false` : Échec après 3 tentatives

**Description** :
- Construit une trame CPE de type `CPE_FT_INCIDENT_AFFECT`
- Utilisée pour dispatcher un véhicule vers un incident
- Pas de champ `status` (mis à 0)

**Cas d'usage** :
- Centre de dispatch envoie une mission
- Affectation automatique d'un incident
- Guidage vers un point d'intervention

**Exemple** :
```cpp
// Centre de commande reçoit un incident à Lyon
int32_t incident_lat = 45764000; // Lyon
int32_t incident_lon = 4836000;
uint32_t now = (uint32_t)uBit.systemTime() / 1000;

// Affecte le véhicule le plus proche
sdmis_radio_send_incident_affect("VEH00008", incident_lat, incident_lon, now);
```

---

## Mécanisme de fiabilité

### Acquittement (ACK)

La librairie implémente un système d'acquittement pour garantir la réception :

```
Émetteur                        Récepteur
   │                                │
   ├──── Trame CPE (29 octets) ───→│
   │                                ├─ Déchiffre
   │                                ├─ Valide CRC
   │                                ├─ Traite
   │←───── ACK (1 octet: seq) ─────┤
   │                                │
```

**Format de l'ACK** :
- 1 octet contenant le numéro de séquence de la trame acquittée
- Envoyé immédiatement après validation réussie
- Pas de chiffrement pour l'ACK

### Retransmission

En cas de perte d'ACK, l'émetteur retransmet automatiquement :

**Paramètres** :
- `SDMIS_RETRY_COUNT = 3` : Nombre maximum de tentatives
- `SDMIS_ACK_TIMEOUT_MS = 200` : Timeout d'attente de l'ACK (200 ms)
- Backoff aléatoire : 10-40 ms entre les retransmissions

**Algorithme** :
```cpp
pour chaque tentative (max 3) :
    envoyer trame
    attendre ACK pendant 200 ms
    si ACK reçu :
        retourner succès
    sinon :
        attendre 10-40 ms (jitter aléatoire)
retourner échec
```

---

## Déduplication

### Protection contre les duplicatas

La librairie filtre automatiquement les trames en double :

**Méthode** :
- Mémorisation du dernier `(seq, nonce)` reçu
- Comparaison avec chaque nouvelle trame
- Rejet silencieux des duplicatas (mais ACK envoyé quand même)

**Avantages** :
- Évite le traitement multiple d'une même trame
- Gère les retransmissions de l'émetteur
- L'ACK est toujours envoyé (même pour un duplicata)

**Limitation** :
- Un seul couple `(seq, nonce)` mémorisé
- Fonctionne bien pour un émetteur à la fois
- Multi-émetteurs : risque de faux négatifs

---

## Gestion automatique

### Numéro de séquence

La librairie gère automatiquement les numéros de séquence :

```cpp
static uint8_t g_seq = 1;

static uint8_t next_seq(void) {
    uint8_t seq = g_seq;
    g_seq = (uint8_t)(g_seq + 1);
    if (g_seq == 0) {
        g_seq = 1;  // Ne jamais utiliser 0
    }
    return seq;
}
```

**Caractéristiques** :
- Démarre à 1
- Incrémente à chaque envoi
- Reboucle après 255 → 1
- Jamais 0 (réservé/invalide)

### Nonce

Le nonce est généré automatiquement à chaque envoi :

```cpp
static uint32_t g_nonce = 0;

// Initialisation avec seed aléatoire
uint32_t seed = (uint32_t)uBit.systemTime();
uBit.seedRandom(seed);
g_nonce = seed ^ (uint32_t)uBit.random(0x7fffffff);

// Génération
static uint32_t next_nonce(void) {
    return g_nonce++;
}
```

**Caractéristiques** :
- Initialisé avec source d'entropie
- Incrémenté à chaque envoi
- Combinaison timestamp + random
- Unicité garantie sur une session

---

## Configuration et constantes

```cpp
#define SDMIS_ACK_TIMEOUT_MS 200   // Timeout d'attente ACK
#define SDMIS_RETRY_COUNT 3        // Nombre de retransmissions
#define SDMIS_POLL_SLICE_MS 5      // Intervalle de poll radio
#define SDMIS_POLL_MAX 8           // Max de paquets traités par poll
```

### Réglage de la puissance d'émission

La puissance TX du micro:bit (paramètre `tx_power`) :

| Valeur | Puissance | Portée approximative |
|--------|-----------|----------------------|
| 0      | -30 dBm   | ~2 m                 |
| 1      | -20 dBm   | ~5 m                 |
| 2      | -16 dBm   | ~10 m                |
| 3      | -12 dBm   | ~20 m                |
| 4      | -8 dBm    | ~30 m                |
| 5      | -4 dBm    | ~50 m                |
| 6      | 0 dBm     | ~100 m               |
| 7      | +4 dBm    | ~150 m               |

**Recommandations** :
- Intérieur : `tx_power = 4-5`
- Extérieur : `tx_power = 6-7`
- Tests rapprochés : `tx_power = 1-2`

---

## Exemple complet d'utilisation

### Unité sur le terrain (émission de position périodique)

```cpp
#include "MicroBit.h"
#include "lib/sdmis_radio.h"

MicroBit uBit;

int main() {
    uBit.init();
    
    // Clé partagée (doit être la même pour toutes les unités)
    uint8_t key[16] = {
        0x2b, 0x7e, 0x15, 0x16, 0x28, 0xae, 0xd2, 0xa6,
        0xab, 0xf7, 0x15, 0x88, 0x09, 0xcf, 0x4f, 0x3c
    };
    
    // Initialisation SDMIS
    sdmis_radio_init(&uBit, key, 42, 6);
    
    const char *mon_immat = "VEH00042";
    uint8_t status = 1; // Disponible
    
    while (true) {
        // Simulation GPS (à remplacer par vraies coordonnées)
        int32_t lat = 48856600 + uBit.random(1000);
        int32_t lon = 2352200 + uBit.random(1000);
        
        // Timestamp actuel
        uint32_t now = (uint32_t)(uBit.systemTime() / 1000);
        
        // Envoi de la position
        bool ok = sdmis_radio_send_position(mon_immat, lat, lon, status, now);
        
        if (ok) {
            uBit.display.print("✓");
        } else {
            uBit.display.print("✗");
        }
        
        // Changement de statut avec les boutons
        if (uBit.buttonA.isPressed()) {
            status = 1; // Disponible
            sdmis_radio_send_status(mon_immat, status, now);
        }
        if (uBit.buttonB.isPressed()) {
            status = 5; // En mission
            sdmis_radio_send_status(mon_immat, status, now);
        }
        
        uBit.sleep(5000); // Envoi toutes les 5 secondes
    }
}
```

### Centre de commande (réception de positions)

```cpp
#include "MicroBit.h"
#include "lib/sdmis_radio.h"

MicroBit uBit;

int main() {
    uBit.init();
    
    uint8_t key[16] = {
        0x2b, 0x7e, 0x15, 0x16, 0x28, 0xae, 0xd2, 0xa6,
        0xab, 0xf7, 0x15, 0x88, 0x09, 0xcf, 0x4f, 0x3c
    };
    
    sdmis_radio_init(&uBit, key, 42, 6);
    
    sdmis_frame_t frame;
    
    while (true) {
        // Poll des messages
        if (sdmis_radio_poll(&frame)) {
            // Traitement selon le type
            switch (frame.type) {
                case CPE_FT_VEH_POS:
                    // Affichage de l'immatriculation
                    uBit.display.scroll(frame.immat);
                    
                    // Conversion coordonnées
                    double lat = frame.lat_e6 / 1000000.0;
                    double lon = frame.lon_e6 / 1000000.0;
                    
                    // Envoi vers console série ou affichage
                    uBit.serial.printf("Position: %s @ %.6f, %.6f (status=%d)\r\n",
                                      frame.immat, lat, lon, frame.status);
                    break;
                    
                case CPE_FT_VEH_STATUS:
                    uBit.serial.printf("Statut: %s = %d\r\n", 
                                      frame.immat, frame.status);
                    break;
                    
                case CPE_FT_INCIDENT_AFFECT:
                    // (Normalement reçu par le véhicule, pas le centre)
                    break;
            }
        }
        
        // Dispatch d'un incident avec le bouton A
        if (uBit.buttonA.isPressed()) {
            uint32_t now = (uint32_t)(uBit.systemTime() / 1000);
            sdmis_radio_send_incident_affect("VEH00042", 48900000, 2400000, now);
            uBit.display.print("!");
        }
        
        uBit.sleep(50); // Poll toutes les 50 ms
    }
}
```

---

## Performances et limitations

### Débit

- **Taille de trame** : 29 octets + overhead radio
- **Temps d'émission** : ~10 ms
- **Temps ACK** : ~210 ms (200 ms timeout + traitement)
- **Débit max théorique** : ~4-5 trames/seconde (mode fiable)

### Portée

- Dépend de la puissance TX et de l'environnement
- Intérieur : 20-50 m (murs, interférences)
- Extérieur dégagé : 100-150 m
- Obstacles métalliques : réduction significative

### Consommation

- **Émission** : ~13 mA (TX power 6)
- **Réception** : ~11 mA (radio en veille active)
- **Sleep** : ~1 µA (radio désactivée)

**Optimisations** :
- Augmenter l'intervalle entre les envois
- Désactiver la radio entre les transmissions
- Réduire la puissance TX si possible

---

## Dépannage

### Aucune trame reçue

**Vérifications** :
1. ✓ Même clé sur émetteur et récepteur
2. ✓ Même groupe radio (`radio_group`)
3. ✓ Appel régulier de `sdmis_radio_poll()`
4. ✓ Portée radio suffisante
5. ✓ Pas d'obstacles majeurs

### ACK non reçus (échecs d'envoi)

**Causes possibles** :
- Récepteur surchargé (ne poll pas assez souvent)
- Interférences radio
- Portée limite
- Récepteur non initialisé

**Solutions** :
- Augmenter la fréquence de poll
- Réduire la distance
- Augmenter `SDMIS_ACK_TIMEOUT_MS`

### Duplicatas reçus

**Normal** : Le filtre anti-duplication ne mémorise qu'un couple `(seq, nonce)`
**Solution** : Implémenter un buffer circulaire côté application si nécessaire

---

## Sécurité

### Distribution de la clé

⚠️ **CRITIQUE** : La clé AES-128 doit être distribuée de manière sécurisée :

- **Ne pas** coder en dur dans le code source (visible)
- Utiliser un mécanisme de provisionnement sécurisé
- Changer régulièrement la clé
- Une clé par réseau/mission

### Attaques possibles

1. **Rejeu** : Partiellement protégé par nonce/timestamp
   - ⚠️ Implémenter une vérification de fraîcheur du timestamp
   
2. **Brouillage** : Pas de protection (couche physique)
   - Solution : Saut de fréquence, étalement de spectre

3. **Clé compromise** : Tout le trafic est déchiffrable
   - Solution : Rotation fréquente, clés de session

---

## Intégration avec le protocole CPE

La librairie SDMIS_RADIO est une **surcouche** au protocole CPE :

| Couche       | Responsabilités                                    |
|--------------|---------------------------------------------------|
| SDMIS_RADIO  | ACK/Retry, Anti-duplication, Nonce, Séquence      |
| CPE          | Chiffrement, CRC, Format de trame                 |
| MicroBit     | Transmission radio physique                       |

**Avantages** :
- Séparation des préoccupations
- CPE réutilisable dans d'autres contextes
- SDMIS_RADIO spécifique à l'environnement micro:bit

---

## Fichiers sources

- **Header** : [sdmis_radio.h](../source/lib/sdmis_radio.h)
- **Implémentation** : [sdmis_radio.cpp](../source/lib/sdmis_radio.cpp)
- **Protocole sous-jacent** : [cpe.h](../source/proto/cpe/cpe.h), [cpe.c](../source/proto/cpe/cpe.c)

---

## Améliorations futures

### Court terme
- [ ] Buffer circulaire pour mémoriser N couples (seq, nonce)
- [ ] Validation du timestamp (rejet si trop ancien)
- [ ] Statistiques de transmission (taux de perte, latence)

### Moyen terme
- [ ] Mode broadcast (sans ACK pour diffusion)
- [ ] Priorités de messages
- [ ] Compression des données GPS

### Long terme
- [ ] Mesh networking (relais multi-sauts)
- [ ] Chiffrement authentifié (AES-GCM)
- [ ] Gestion de clés dynamique

---

## Support et contribution

- **Projet** : IoT Terrain Micro:bit
- **Date** : Janvier 2026
- **Dépendances** :
  - MicroBit DAL (Device Abstraction Layer)
  - Protocole CPE
  - TinyCrypt (via CPE)

---

## Licence

Voir le fichier [LICENSE](../LICENSE) à la racine du projet.
