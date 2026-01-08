# Protocole CPE - Documentation Complète

## Vue d'ensemble

Le protocole **CPE** (Cryptographic Positioning Exchange) est un protocole de communication sécurisé conçu pour l'échange d'informations de positionnement et de statut entre véhicules ou unités sur le terrain. Il utilise le chiffrement AES-128 en mode CTR pour garantir la confidentialité et l'intégrité des données transmises.

### Caractéristiques principales

- **Chiffrement** : AES-128 en mode CTR (Counter Mode)
- **Intégrité** : CRC-16 CCITT
- **Taille de trame fixe** : 29 octets
- **Version actuelle** : v1
- **Types de messages** : Position, Statut, Incident

---

## Structure de la trame

Une trame CPE fait exactement **29 octets** et se compose de trois parties :

```
┌─────────────┬──────────────────────┬─────────┐
│   En-tête   │   Payload chiffré    │  Total  │
│   9 octets  │     20 octets        │ 29 oct  │
└─────────────┴──────────────────────┴─────────┘
```

### 1. En-tête (Header) - 9 octets

L'en-tête n'est **pas chiffré** et contient les métadonnées de la trame :

| Offset | Taille | Champ      | Description                                    |
|--------|--------|------------|------------------------------------------------|
| 0      | 1      | `seq`      | Numéro de séquence (1-255, ne doit pas être 0) |
| 1      | 4      | `nonce`    | Nombre aléatoire unique (big-endian)           |
| 5      | 4      | `timestamp`| Horodatage Unix en secondes (big-endian)       |

### 2. Payload chiffré - 20 octets

Le payload contient les données métier chiffrées avec AES-128 CTR :

#### Structure avant chiffrement (18 octets de données + 2 octets CRC)

| Offset | Taille | Champ        | Description                                      |
|--------|--------|--------------|--------------------------------------------------|
| 0      | 1      | `ver/type`   | 4 bits version + 4 bits type de trame            |
| 1      | 8      | `immat`      | Immatriculation (identifiant) du véhicule        |
| 9      | 1      | `status`     | Code de statut (0-255)                           |
| 10     | 4      | `lat_e6`     | Latitude × 10⁶ (int32, big-endian)               |
| 14     | 4      | `lon_e6`     | Longitude × 10⁶ (int32, big-endian)              |
| 18     | 2      | `crc16`      | CRC-16 CCITT des 18 premiers octets              |

---

## Types de trames

Le protocole CPE définit trois types de trames, identifiés par les 4 bits inférieurs du premier octet du payload :

### 0x01 - `CPE_FT_VEH_POS` (Position du véhicule)

Transmet la position GPS complète et le statut d'un véhicule.

**Champs utilisés** :
- `immat` : Identifiant du véhicule (8 caractères)
- `status` : Code de statut
- `lat_e6` : Latitude en micro-degrés (latitude × 1 000 000)
- `lon_e6` : Longitude en micro-degrés (longitude × 1 000 000)

**Exemple** : Un véhicule "VEH00001" à Paris (48.8566° N, 2.3522° E) avec statut 5
- `lat_e6` = 48 856 600
- `lon_e6` = 2 352 200

### 0x02 - `CPE_FT_VEH_STATUS` (Statut du véhicule)

Transmet uniquement le statut d'un véhicule sans information de position.

**Champs utilisés** :
- `immat` : Identifiant du véhicule
- `status` : Nouveau code de statut
- `lat_e6` : Non utilisé (mis à 0)
- `lon_e6` : Non utilisé (mis à 0)

**Cas d'usage** : Changement de statut sans déplacement significatif

### 0x03 - `CPE_FT_INCIDENT_AFFECT` (Affectation à un incident)

Signale qu'un véhicule est affecté à un incident à une position donnée.

**Champs utilisés** :
- `immat` : Identifiant du véhicule affecté
- `status` : Non utilisé (mis à 0)
- `lat_e6` : Latitude de l'incident
- `lon_e6` : Longitude de l'incident

**Cas d'usage** : Dispatch d'une unité vers un incident

---

## Cryptographie

### Algorithme de chiffrement

Le protocole utilise **AES-128 en mode CTR** (Counter Mode) avec la bibliothèque TinyCrypt.

#### Vecteur d'initialisation (IV) - 16 octets

```
┌──────────┬─────┬───────────────┐
│  nonce   │ seq │   padding 0   │
│ 4 octets │ 1   │   11 octets   │
└──────────┴─────┴───────────────┘
```

- **Octets 0-3** : Valeur du nonce (big-endian)
- **Octet 4** : Numéro de séquence
- **Octets 5-15** : Zéros

### Calcul du CRC-16

Le protocole utilise le **CRC-16 CCITT** avec les paramètres suivants :
- Polynôme : `0x1021`
- Valeur initiale : `0xFFFF`
- Calcul : sur les 18 premiers octets du payload (avant ajout du CRC)

Le CRC est ajouté **avant le chiffrement** et vérifié **après le déchiffrement**.

### Gestion de la clé

- Clé partagée de **16 octets** (128 bits)
- Initialisée avec `cpe_init(key)`
- Stockée en mémoire statique
- **Critique** : Tous les participants doivent partager la même clé

---

## API de programmation

### Initialisation

```c
void cpe_init(const uint8_t key[CPE_KEY_LEN]);
```

**Paramètres** :
- `key` : Clé AES-128 de 16 octets

**Description** : Doit être appelée avant toute utilisation du protocole. Configure la clé cryptographique partagée.

---

### Construction de trames

#### Trame de position

```c
void cpe_build_position_frame(
    const char immat[CPE_IMMAT_LEN],
    int32_t lat_e6,
    int32_t lon_e6,
    uint8_t status,
    uint32_t nonce,
    uint32_t timestamp,
    uint8_t seq,
    uint8_t out_frame[CPE_FRAME_LEN]
);
```

**Paramètres** :
- `immat` : Immatriculation (8 caractères max)
- `lat_e6` : Latitude × 10⁶
- `lon_e6` : Longitude × 10⁶
- `status` : Code de statut
- `nonce` : Nombre unique pour cette trame
- `timestamp` : Timestamp Unix (secondes)
- `seq` : Numéro de séquence (1-255)
- `out_frame` : Buffer de sortie (29 octets)

#### Trame de statut

```c
void cpe_build_status_frame(
    const char immat[CPE_IMMAT_LEN],
    uint8_t status,
    uint32_t nonce,
    uint32_t timestamp,
    uint8_t seq,
    uint8_t out_frame[CPE_FRAME_LEN]
);
```

**Paramètres** :
- `immat` : Immatriculation (8 caractères max)
- `status` : Nouveau code de statut
- `nonce` : Nombre unique pour cette trame
- `timestamp` : Timestamp Unix (secondes)
- `seq` : Numéro de séquence (1-255)
- `out_frame` : Buffer de sortie (29 octets)

#### Trame d'incident

```c
void cpe_build_incident_affect_frame(
    const char immat[CPE_IMMAT_LEN],
    int32_t lat_e6,
    int32_t lon_e6,
    uint32_t nonce,
    uint32_t timestamp,
    uint8_t seq,
    uint8_t out_frame[CPE_FRAME_LEN]
);
```

**Paramètres** :
- `immat` : Immatriculation du véhicule affecté
- `lat_e6` : Latitude de l'incident × 10⁶
- `lon_e6` : Longitude de l'incident × 10⁶
- `nonce` : Nombre unique pour cette trame
- `timestamp` : Timestamp Unix (secondes)
- `seq` : Numéro de séquence (1-255)
- `out_frame` : Buffer de sortie (29 octets)

---

### Analyse de trames

```c
int cpe_parse_frame(
    const uint8_t frame[CPE_FRAME_LEN],
    cpe_frame_type_t *type_out,
    char immat_out[CPE_IMMAT_LEN],
    uint8_t *status_out,
    int32_t *lat_e6_out,
    int32_t *lon_e6_out,
    uint32_t *nonce_out,
    uint32_t *timestamp_out,
    uint8_t *seq_out
);
```

**Paramètres** :
- `frame` : Trame reçue (29 octets)
- `type_out` : Type de trame (obligatoire)
- Tous les autres paramètres sont optionnels (peuvent être `NULL`)

**Retour** :
- `0` : Succès, trame valide
- `-1` : Erreur (CRC invalide, version incorrecte, type inconnu)

**Description** : 
- Déchiffre la trame
- Vérifie le CRC-16
- Extrait tous les champs
- Les pointeurs `NULL` sont ignorés (permet de ne récupérer que les champs souhaités)

---

## Sécurité

### Protection contre les attaques

1. **Rejeu (Replay Attack)**
   - Le nonce unique et le timestamp empêchent la réutilisation de trames
   - Le numéro de séquence incrémental aide à détecter les duplicatas
   - L'application doit vérifier la fraîcheur du timestamp

2. **Man-in-the-Middle**
   - Le chiffrement AES-128 protège la confidentialité
   - La clé partagée doit être distribuée de manière sécurisée

3. **Intégrité**
   - Le CRC-16 détecte les corruptions accidentelles
   - Le chiffrement authentique empêche la modification sans connaissance de la clé

### Limitations

⚠️ **Attention** :
- Pas d'authentification cryptographique forte (HMAC/GCM)
- Le CRC n'est pas un MAC cryptographique
- La clé partagée doit être changée périodiquement
- Pas de Perfect Forward Secrecy

---

## Constantes et définitions

```c
#define CPE_KEY_LEN 16           // Taille de la clé AES
#define CPE_VERSION 1            // Version du protocole
#define CPE_IMMAT_LEN 8          // Longueur de l'immatriculation
#define CPE_HEADER_LEN 9         // Taille de l'en-tête
#define CPE_PAYLOAD_LEN 18       // Taille du payload (avant CRC)
#define CPE_CRC_LEN 2            // Taille du CRC
#define CPE_PLAINTEXT_LEN 20     // Payload + CRC
#define CPE_FRAME_LEN 29         // Taille totale de la trame
```

---

## Exemple d'utilisation

```c
#include "proto/cpe/cpe.h"

// Initialisation
uint8_t key[16] = {0x01, 0x02, 0x03, ..., 0x10};
cpe_init(key);

// Construction d'une trame de position
uint8_t frame[CPE_FRAME_LEN];
cpe_build_position_frame(
    "VEH00001",      // Immatriculation
    48856600,        // Latitude (Paris)
    2352200,         // Longitude (Paris)
    5,               // Statut
    0x12345678,      // Nonce
    1704672000,      // Timestamp
    1,               // Séquence
    frame            // Sortie
);

// Transmission de la trame (via radio, série, etc.)
// ...

// Réception et parsing
cpe_frame_type_t type;
char immat[CPE_IMMAT_LEN];
uint8_t status;
int32_t lat, lon;
uint32_t nonce, timestamp;
uint8_t seq;

int result = cpe_parse_frame(
    frame, &type, immat, &status,
    &lat, &lon, &nonce, &timestamp, &seq
);

if (result == 0) {
    // Trame valide
    if (type == CPE_FT_VEH_POS) {
        printf("Position: %.6f, %.6f\n", 
               lat / 1000000.0, lon / 1000000.0);
    }
}
```

---

## Recommandations d'implémentation

### Gestion du nonce
- Utiliser un compteur incrémental ou une valeur aléatoire
- Ne **jamais** réutiliser la même combinaison (nonce, seq) avec la même clé
- Initialiser avec une source d'entropie (timestamp, random, etc.)

### Gestion de la séquence
- Commencer à 1 (jamais 0)
- Incrémenter après chaque envoi
- Reboucler après 255 : `seq = (seq % 255) + 1`

### Gestion du timestamp
- Utiliser le temps Unix (secondes depuis 1970-01-01)
- Synchroniser les horloges entre les unités
- Rejeter les trames trop anciennes (> 60 secondes par exemple)

### Conversion des coordonnées GPS
```c
// Conversion degrés décimaux → micro-degrés
int32_t lat_e6 = (int32_t)(latitude_degrees * 1000000.0);
int32_t lon_e6 = (int32_t)(longitude_degrees * 1000000.0);

// Conversion micro-degrés → degrés décimaux
double latitude = lat_e6 / 1000000.0;
double longitude = lon_e6 / 1000000.0;
```

---

## Format de trame détaillé (exemple)

Exemple de trame **CPE_FT_VEH_POS** pour "VEH00001" à Paris avec statut 5 :

### Avant chiffrement

```
En-tête (9 octets):
  00: 01                    // seq = 1
  01: 12 34 56 78           // nonce = 0x12345678
  05: 65 92 FC 40           // timestamp = 1704672000

Payload clair (20 octets):
  00: 11                    // version=1, type=1 (VEH_POS)
  01: 56 45 48 30 30 30 30 31  // "VEH00001"
  09: 05                    // status = 5
  0A: 02 E9 2D 38           // lat = 48856600
  0E: 00 23 E8 28           // lon = 2352200
  12: XX XX                 // CRC-16 calculé
```

### Après chiffrement

```
Trame complète (29 octets):
  00: 01                    // seq = 1
  01: 12 34 56 78           // nonce = 0x12345678
  05: 65 92 FC 40           // timestamp = 1704672000
  09: [20 octets chiffrés]  // Payload + CRC chiffrés
```

---

## Dépendances

Le protocole CPE nécessite :
- **TinyCrypt** : Pour AES-128 CTR
  - `tinycrypt/aes.h`
  - `tinycrypt/ctr_mode.h`
- **string.h** : Pour les opérations mémoire
- **stdint.h** : Pour les types entiers

---

## Licence et références

- Fichiers : [cpe.h](../source/proto/cpe/cpe.h), [cpe.c](../source/proto/cpe/cpe.c)
- Projet : IoT Terrain Micro:bit
- Date : Janvier 2026

---

## Notes de version

### Version 1 (actuelle)
- Support des trames de position, statut et incident
- Chiffrement AES-128 CTR
- CRC-16 CCITT pour l'intégrité
- Trame fixe de 29 octets
